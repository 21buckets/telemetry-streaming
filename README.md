# Implementing Telemetry Streaming

This README is to assist in implementing Telemetry Streaming and provide relevant declarations and code snippets. Starting out with a Splunk based implementation for now, and will add to it over time



## PreRequisites. 
* Download Telemetry streaming rpm from [GitHub Telemetry Streaming releases](https://github.com/F5Networks/f5-telemetry-streaming/releases) and install on active/standby devices (iApps > Package Management LX > import)
* (optional) Download AS3 rpm from [GitHub AS3 releases](https://github.com/F5Networks/f5-appsvcs-extension/releases) and install on active/standby devices (iApps > Package Management LX > import)
  * This is optional as you can configure the required listeners and logging profiles via any method (tmsh, gui, api commands etc)




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
