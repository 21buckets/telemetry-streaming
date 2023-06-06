# Implementing Telemetry Streaming

This README is to assist in implementing Telemetry Streaming and provide relevant declarations and code snippets. Starting out with a Splunk based implementation for now, and will add to it over time



## PreRequisites. 
* Download Telemetry streaming rpm from [GitHub Telemetry Streaming releases](https://github.com/F5Networks/f5-telemetry-streaming/releases) and install on active/standby devices (iApps > Package Management LX > import)
* (optional) Download AS3 rpm from [GitHub AS3 releases](https://github.com/F5Networks/f5-appsvcs-extension/releases) and install on active/standby devices (iApps > Package Management LX > import)
  * This is optional as you can configure the required listeners and logging profiles via any method (tmsh, gui, api commands etc)

## To Do:
* Provide POSTMAN collection for Telemetry Streaming and AS3 declarations
* Add steps for generating an API token for the Big-IP
* Remove code snippets from README and move to specific files


## Preparing Splunk

I am using a [free trial](https://www.splunk.com/en_us/download.html) of Splunk Cloud for this demonstration. The free trial lasts for 14 days, requires no credit card and allows for up to 5GB of data per day.

###  Data Input

First, let's create an indexer so all data ingested from the Big-IP will be searchable with this index. 

1. Go to `Settings > Indexers`
2. Select `New Index`
3. Provide a name: `f5` (I like lowercase everything)
4. The Index Data Type is `Events`
5. I've set the Maximum Data Size to `0` (unlimited). I'm sure Splunk administrators will have a view on what this _should_ be.
6. Retention period is set to the duration of my cloud trial (14 days). Again, Splunk administrators will have a better opinion on this.


Now we set up the HTTP Event Collector (HEC). This will be the endpoint used by Telemetry Streaming to send data to Splunk without having to run Syslog and Heavy/Universal Forwarders. 

1. Go to `Settings > Data Inputs`
2. Select `HTTP Event Collector` and `New Token`
3. Give the token a Name. I've gone with `f5_telemetry_streaming`
4. Make sure Indexer Acknowledgement is disabled. I had issues getting `400` HTTP response codes when I had this enabled. 
5. Leave the `source type` as `automatic`. Splunk does a pretty good job of categorising the data from Telemetry Streaming. (Or perhaps TS is doing some magic behind the scenes with the Splunk details we will provide it later)
6. On the next page, add the `f5` index, and any others if necessary (There is probably some best practice here)
7. Review and Submit the token, and take a copy of it as this is what Big-IP/Telemetry Streaming will use to authenticate to the Splunk endpoint.


Now Splunk is ready to go, let's configure Telemetry Streaming...


## Configuring Telemetry Streaming

You can find all the required Telemetry Streaming documentation [here](https://clouddocs.f5.com/products/extensions/f5-telemetry-streaming/latest/), so I won't spend any time talking about it. I'll just provide the configuration and describe the relevant parts.


We are setting up a `Push Consumer`, that is, Telemetry Streaming will push metrics/events to the Splunk endpoint, rather than having Splunk pull from Big-IP, although Big-IP does support `Pull Consumers` as well.

To set up TS, POST the below declaration to your Big-IP. Note, you will need to have an authentication token.
Read though this [document](https://clouddocs.f5.com/products/extensions/f5-telemetry-streaming/latest/quick-start.html) for a detailed explanation of the various components in the below declaration

`https://{{bigip_mgmt}}/mgmt/shared/telemetry/declare`



```
{
    "class": "Telemetry",
    "Controls":{
        "class":"Controls",
        "logLevel":"debug",
        "debug":true
    },
    "My_System": {
        "class": "Telemetry_System",
        "systemPoller": {
            "interval": 60
        }
    },
    "My_Listener": {
        "class": "Telemetry_Listener",
        "port": 6514
    },
    "My_Consumer": {
        "class": "Telemetry_Consumer",
        "type": "Splunk",
        "host": "prd-p-xxxx.splunkcloud.com",
        "protocol": "https",
        "port": 8088,
        "allowSelfSignedCert": true,
        "passphrase": {
            "cipherText": "******-****-****-****-****"
        }
    }
}

```

Note:
* Remove the controls class completely to reduce the log spam, or look up the TS Schema to set the logLevel to something less spammy.
* You can change the frequency TS will poll for metrics by increasing `My_System.systemPoller.interval`. The minimum value is 60 seconds.
* Update `My_Consumer.host` to the hostname of your Splunk instance
* Update `My_Consumer.passphrase.cipherText` to the value of the Splunk token you generated earlier


The declaration is all that is required to send Big-IP metrics to the Splunk endpoint. It won't do anything for application logs/metrics from LTM/AFM/ASM etc, as this capability is still handled by the logging profile construct. You can configure the logging profiles to point to the telemetry streaming endpoint to send logs off to Splunk, which is covered below.

## Configuring Big-IP listeners and logging

As previously mentioned, Telemetry Streaming does not handle application level logging from LTM/ASM/APM/AVR etc. This is all handled through the standard logging profiles. We can (and will) integrate this with AS3. Logging profiles need a log destination to point to, so this next part exposes the Telemetry Streaming endpoint via the use of a virtual server.

I am going to use AS3 to create the required logging profiles, virtual server listeners, pools, and iRules. You can do all of this manually, but AS3 makes it a very simple single declaration. I recommend reading up on AS3 before using though.

This declaration is creating objects in the `Common` partition, which is unusual for AS3, but it is supported. See "When does BIG-IP AS3 write to the Common partition for LTM configurations?" in the [FAQ](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/faq.html)


POSTing the below declaration to `https://{{bigip_mgmt}}/mgmt/shared/appsvcs/declare` will configure the required ltm objects. Essentially what we are doing here is creating log destinations and Security logging profiles (AFM and AWAF) to be applied to your application virtual server. The `log destination` is a pool on the F5 that contains a local listener for Telemetry Streaming running on port 6514. When there are security events, the logging profile sends the event to Telemetry Streaming which in turn sends the event to Splunk (as per the above TS declaration)

```
{
    "class":"AS3",
    "action":"deploy",
    "persist":true,
    "declaration":{
        "class":"ADC",
        "schemaVersion":"3.42.0",
        "Common":{
            "class":"Tenant",
            "Shared":{
                "class":"Application",
                "template":"shared",
                "telemetry_local_rule": {
                    "remark": "Only required when TS is a local listener",
                    "class": "iRule",
                    "iRule": "when CLIENT_ACCEPTED {\n  node 127.0.0.1 6514\n}"
                },
                "telemetry_local": {
                    "remark": "Only required when TS is a local listener",
                    "class": "Service_TCP",
                    "virtualAddresses": [
                        "255.255.255.254"
                    ],
                    "virtualPort": 6514,
                    "iRules": [
                        "telemetry_local_rule"
                    ]
			    },
                "telemetry": {
                    "class": "Pool",
                    "members": [{
                        "enable": true,
                        "serverAddresses": [
                            "255.255.255.254"
                        ],
                        "servicePort": 6514
                    }],
                    "monitors": [{
                        "bigip": "/Common/tcp"
                    }]
                },
                "telemetry_hsl": {
                    "class": "Log_Destination",
                    "type": "remote-high-speed-log",
                    "protocol": "tcp",
                    "pool": {
                        "use": "telemetry"
                    }
                },
                "telemetry_formatted": {
                    "class": "Log_Destination",
                    "type": "splunk",
                    "forwardTo": {
                        "use": "telemetry_hsl"
                    }
                },
                "telemetry_publisher": {
                    "class": "Log_Publisher",
                    "destinations": [{
                        "use": "telemetry_formatted"
                    }]
                },
                "telemetry_traffic_log_profile": {
                    "class": "Traffic_Log_Profile",
                    "requestSettings": {
                        "requestEnabled": true,
                        "requestProtocol": "mds-tcp",
                        "requestPool": {
                            "use": "telemetry"
                        },
                        "requestTemplate": "event_source=\"request_logging\",hostname=\"$BIGIP_HOSTNAME\",client_ip=\"$CLIENT_IP\",server_ip=\"$SERVER_IP\",http_method=\"$HTTP_METHOD\",http_uri=\"$HTTP_URI\",virtual_name=\"$VIRTUAL_NAME\",event_timestamp=\"$DATE_HTTP\""
                    },
                    "responseSettings": {
                        "responseEnabled": true,
                        "responseProtocol": "mds-tcp",
                        "responsePool": {
                            "use": "telemetry"
                        },
                        "responseTemplate": "event_source=\"response_logging\",hostname=\"$BIGIP_HOSTNAME\",client_ip=\"$CLIENT_IP\",server_ip=\"$SERVER_IP\",http_method=\"$HTTP_METHOD\",http_uri=\"$HTTP_URI\",virtual_name=\"$VIRTUAL_NAME\",event_timestamp=\"$DATE_HTTP\",http_statcode=\"$HTTP_STATCODE\",http_status=\"$HTTP_STATUS\",response_ms=\"$RESPONSE_MSECS\""
                    }
                },

                "telemetry_asm_security_log_profile": {
                    "class": "Security_Log_Profile",
                    "application": {
                        "localStorage": false,
                        "remoteStorage": "splunk",
                        "servers": [{
                            "address": "255.255.255.254",
                            "port": "6514"
                        }],
                        "storageFilter": {
                            "requestType": "all"
                        }
                    }
		    	},
                "telemetry_afm_security_log_profile": {
                    "class": "Security_Log_Profile",
                    "application": {
                        "localStorage": false,
                        "remoteStorage": "splunk",
                        "protocol": "tcp",
                        "servers": [
                            {
                                "address": "255.255.255.254",
                                "port": "6514"
                            }
                        ],
                        "storageFilter": {
                            "requestType": "illegal-including-staged-signatures"
                        }
                    },
                    "network": {
                        "publisher": {
                            "use": "telemetry_publisher"
                        },
                        "logRuleMatchAccepts": true,
                        "logRuleMatchRejects": true,
                        "logRuleMatchDrops": true,
                        "logIpErrors": true,
                        "logTcpErrors": true,
                        "logTcpEvents": true
                    }
                }               
            }
        }
    }
}


```


### Applying the logging profile to your application VS

This part is no different to adding logging to an AWAF enabled virtual server. 

1. Find the application virtual server
2. Select `Security > Policies`
3. Add the `telemetry_asm_security_log_profile` Log Profile to the virtual server
