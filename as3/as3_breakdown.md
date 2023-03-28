# Breakdown of AS3 components 

In the main [README](https://github.com/21buckets/telemetry-streaming) in this repository for configuring Telemetry Streaming, there is a sample AS3 declaration provided to configure the required virtual servers, pools, logging profiles etc.
This document breaks each component down, and provides:
* overview of the intent of the class
* link to relevant AS3 schema for the class
* description of what is actually created on the Big-IP itself


The easiest way to tell what AS3 is creating when looking at a declaration is to look at the `Class` property. For most objects, the class value will be a recognisable name, i.e.
* `class: Pool` - Creates an LTM Pool
* `class: iRule` - Creates an LTM iRule

Virtual servers are an obvious exception to this, but will make sense:
* `class: Service_TCP` - creates a virtual server for a TCP listener. Think of this as a template for what a TCP based virtual server would require. A lot of default values are set behind the scenes, making the declaration a lot simpler.  

Other examples for virtual servers would be `Service_HTTPS`. This class sets things like a HTTP profile by default, and makes the clientssl profile a mandatory field.

## Resources

Some useful resources to help understand how AS3 works:  
* [User Guide](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/) - contains everything you need to know, but it will put you to sleep
* [FAQ](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/faq.html)
* [Schema Reference](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/schema-reference.html) - Contains all the possible classes, properties, and data types
* [Declaration using all Big-IP AS3 Properties](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/declarations/all-properties.html) - Handy to find examples of AS3 usage



## Telemetry Streaming AS3 details

## Overview

The AS3 declaration will create the following objects in the `/Common` partition:
* iRule 
* Virtual Server
* Pool
* Log Destination x2
* Log Publisher
* Security Logging Profile x2 (one for ASM, one for AFM - these could be combined)
* LTM request logging profile




### Boilerplate

The below snippet is the boilerplate required for any AS3 declaration. There are a couple of useful properties to know about:
* `action` Can be set to either 'deploy' or 'dry-run'. Dry-runs are very useful for validating the schema without actually making any changes to the Big-IP
* `schemaVersion` As new versions of AS3 are released, you may want to update the schema the declaration is being validated against to allow for new features.

[Schema documentation](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/schema-reference.html#as3)

```json
{
    "class":"AS3",
    "action":"deploy",
    "persist":true,
    "declaration":{
        "class":"ADC",
        "schemaVersion":"3.42.0",
     }
 }
```


### Tenant Information

* This sets up the Big-IP partition your configuration is being deployed into, see `System > Users > Partition List` (or look for the dropdown on the top right)
* The declaration below will be creating objects in the `Common` partition. 
* Generally speaking, each new AS3 declaration should go into a new partition (i.e. one that doesnt exist already) to avoid accidentally deleting previously exisitng objects.
* The Common partition is special, and should only be used if you want to share objects between other partitions. Read up on this in the [FAQ](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/faq.html), including information on the `Shared` application and Template.


```json
     "Common":{
            "class":"Tenant",
            "Shared":{
                "class":"Application",
                "template":"shared",
             }
       }

```

Using `Common` for the tenant (partion in Big-IP language) and `Shared` will create objects in `/Common/Shared` as per below

<img width="932" alt="image" src="https://user-images.githubusercontent.com/39548246/228107695-e6a5bee3-8ccf-42d6-be19-244815c74494.png">



### iRule

* This creates an iRule with the name `telemetry_local_rule`, with the iRule contents being the value of `telemetry_local_rule.iRule`
*  Can be found under `Local Traffic > iRules > iRule List`
* Rather than storing the contents of the iRule directly in the declaration here, you can reference a HTTP endpoint with the iRule configuration (recommended for more advanced iRules)

[Schema documentation](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/schema-reference.html#irule)

```json
"telemetry_local_rule": {
   "remark": "Only required when TS is a local listener",
   "class": "iRule",
   "iRule": "when CLIENT_ACCEPTED {\n  node 127.0.0.1 6514\n}"
 }
 ```
 
 Note below the iRule is also created in `/Common/Shared`
 
![image](https://user-images.githubusercontent.com/39548246/228109001-ff09462b-171a-40c6-9d0f-5d3323fc42f2.png)


### Virtual Servers

* This creates a virtual server `telemetry_local` listening on port `6514` to be used as a local listener for Telemetry Streaming
* Can be found under `Local Traffic > Virtual Servers > Virtual Servers List`
* It has the previously mentioned iRule attached to it that will send connection requests to the Big-IP device itself
* `Service_TCP` works as a template for creating a TCP based virtual server with most properties set as defaults by AS3 itself, so the JSON required by the user is drastically reduced

[Schema Documentation](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/schema-reference.html#service-tcp)

```json
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
}
```

![image](https://user-images.githubusercontent.com/39548246/228110435-60552155-edd5-4c8f-ad41-dd9465d285b6.png)


### Pool

* This creates an LTM pool called `telemetry`
* Can be found under `Local Traffic > Pools > Pool List`
* The pool node/member is the local VS listener previously created
* It is required by the logging profile we will be creating further on
* Any property not set inherits default values that can be found in the schema documentation

[Schema Documentation](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/schema-reference.html#pool)


```json
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
}
 ```

![image](https://user-images.githubusercontent.com/39548246/228110946-69d56325-b645-4e78-afb4-022d1375884e.png)
![image](https://user-images.githubusercontent.com/39548246/228110980-a65a4399-d9ba-4e75-9d60-d95d6f994358.png)


### Log Destination

* Creates the log destination found in `System > Logs > Configuration > Log Destinations`
* Sends logs to the pool we created (that sends it on to the local listener, that in turn sends it to local device on port `6514` via the iRule) - This is where the Telemetry Streaming application is actually running
* Note the use of `telemetry_hsl.pool.use` This syntax is telling AS3 to use the pool defined within the overarching declaration. If you wanted to refrence an already existing pool, you would use the `bigip` syntax (see [Pointer_BIGIP](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/schema-reference.html#pointer-bigip)), i.e. `telemetry_hsl.pool.bigip: "/Common/existing_pool"


[Schema Documentation](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/schema-reference.html#log-destination)



```json
"telemetry_hsl": {
   "class": "Log_Destination",
   "type": "remote-high-speed-log",
   "protocol": "tcp",
   "pool": {
       "use": "telemetry"
   }
}
```

![image](https://user-images.githubusercontent.com/39548246/228136433-8fc18c95-5a73-4359-b4fe-069641a1b20e.png)


This declaration also has a log destination of type `Splunk`, which forwards to the above log destination... F5 can be weird... I am not exactly sure why this is required - I am going to assume the `Splunk` log destination adds some metadata to help with splunk ingestion. I should find the documentation and link it here...

### Log Publisher

* Creates the log publisher named `telemetry_publisher` under `System > Logs > Configuration > Log Publishers`
* Consumed by the logging profile to send logs to the aforementioned Splunk log destination (denoted by the `use` syntax)
* For all this work, note that the security logging profiles (for ASM and AFM) dont actually use these log publishers, it is only the `Traffic_log_profile` that does.

```json
 "telemetry_publisher": {
   "class": "Log_Publisher",
   "destinations": [{
      "use": "telemetry_formatted"
   }]
}
```

![image](https://user-images.githubusercontent.com/39548246/228136761-04918e97-937f-436d-a23d-b10286137d84.png)

### LTM Logging Profile

* Creates the LTM request logging profile named `telemetry_traffic_log_profile` under `Local Traffic > Profiles > Other > Request Logging`
* Used to send HTTP request logs to the Splunk endpoint
* Formats in a key=value format
* TODO: Find documentation on supported templates (not really an AS3 thing)


```json
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
}

```

![image](https://user-images.githubusercontent.com/39548246/228143564-e31d86e2-b69c-4725-9fa8-a9436eb63c60.png)


### Security Logging Profile (asm/awaf)

* Creates the security logging profile named `telemetry_asm_security_log_profile` found under `Security > Event Logs > Logging Profiles`
* The log destination is directly configurable in this profile, rather than using the log publisher/destination configuration mentioned previously. Likely due to ASM being an acquired product (a long time ago) so is integrated slightly differently.
* The `remoteStorage` key refers to the `Logging Format` option in the GUI
* Note the log destination is still our Telemetry Streaming listener
* `storageFilter.requestType` is saying to send all requests to Telemetry Streaming, not just the illegal ones. I'd recommend changing this to `illegal-including-staged-signatures` (see [documentation](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/schema-reference.html#security-log-profile-protocol-sip-storageformat)) once you're happy log ingestion is working properly. No point sending all requests to Splunk unless absolutely necessary.

**Schema Documentation:** 
* This is a special class type. The class is `Security_Log_Profile` which has a schema definition [here](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/schema-reference.html#security-log-profile)
* However, the configuration is under a key called `application`. This is telling you it is an "application" type logging profile, so you need to look at the Security_Log_Profile_Application [schema definition](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/refguide/schema-reference.html#security-log-profile)

```json
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
}
```
![image](https://user-images.githubusercontent.com/39548246/228140146-5d33cbb4-a4db-42d9-9411-112c1492c28c.png)

                
