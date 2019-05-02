*Note: This document does not describe the github > jenkins > ec2 pipeline. It describes how to inject jmeter results into Splunk server post the ec2 spin up*

# Setting up the Splunk Forwarder

*Note: The following setup of splunk forwarder should be part of the AMI creation which means that when a ec2 instance is spin up from the AMI, spunk forwarder will be ready listening for jmeter results*

## Download the Splunk's Universal Forwarder
wget -O splunkforwarder-7.0.1-2b5b15c4ee89-Linux-x86_64.tgz 'https://www.splunk.com/bin/splunk/DownloadActivityServlet?architecture=x86_64&platform=linux&version=7.0.1&product=universalforwarder&filename=splunkforwarder-7.0.1-2b5b15c4ee89-Linux-x86_64.tgz&wget=true'

*Note: This method downloads splunk universal forwarder version 7.0.1 from Splunk's website. A presribed archive may also be downloaded from Disney's repositories.*

## Untar the downloaded archive
tar xvzf splunkforwarder-7.0.1-2b5b15c4ee89-Linux-x86_64.tgz -C /opt

## Start the forwarder
/opt/splunkforwarder/bin/splunk start --accept-license

## Change the default password
/opt/splunkforwarder/bin/splunk edit user admin -password D1$nEy#123 -auth admin:changeme (choose appropriate password)

*Note: The default password for admin is changeme*

## Set the forward server
/opt/splunkforwarder/bin/splunk add forward-server localhost:9997 -auth admin:D1$nEy#123 (replace with appropriate hostname and port for the forward server)

## Add monitor (the Jmeter log to be monitored)
/opt/splunkforwarder/bin/splunk add monitor /jmeter/results/results.jtl -index wdat-ee-ra-pe -sourcetype jmeter

# Setting Jmeter
The following jmeter setup is a pre-requisite. This setup enables grouping of metrics in splunk easier.

![alt text](https://github.disney.com/PE/splunkjmeter/blob/master/jmeter_setup_for_sample_variables.png "jmeter setup for sample variables")

# Running Jmeter

```java
./jmeter -n \
-t /jmeter/scripts/script.jmx \
-l /jmeter/results/results.jtl \
-JapplicationName=dcl \
-JtestDesc=dcl \
-JapplicationVersion=4.5 \
-Jjmeter.save.saveservice.print_field_names=false \
-Jjmeter.save.saveservice.timestamp_format="yyyy-MM-dd HH:mm:ss" \
-Jjmeter.save.saveservice.autoflush=true \
-Jsample_variables=conversationId,applicationName,testId,testDesc,applicationVersion,ipAddress \
-Jjmeter.save.saveservice.response_data.on_error=true \
-Jjmeter.save.saveservice.output_format=xml
```

*Note: Change the values for the variables applicationName, testDesc and applicationVersion accordingly*

# Suggested Metrics for the Splunk Dashboard
![alt text](https://github.disney.com/PE/splunkjmeter/blob/master/splunkjmeter_dash.jpg "splunk jmeter dash")

# Queries for the Splunk Dashboard

## Table of KPIs - grouped by labels
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| rex field=_raw "lb=\"(?<label>\w+)\""
| rex field=_raw "\bt=\"(?<elapsedTime>\d+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| stats count, min(elapsedTime) as min, avg(elapsedTime) as avg, stdev(elapsedTime) as stdev, perc90(elapsedTime) as p90, perc95(elapsedTime) as p95, perc99(elapsedTime) as p99, max(elapsedTime) as max by label
| eval avg=round(avg,3)
| eval stdev=round(stdev,3)
```
## Table of latencies - grouped by labels
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| rex field=_raw "lb=\"(?<label>\w+)\""
| rex field=_raw "\bt=\"(?<elapsedTime>\d+)\""
| rex field=_raw "lt=\"(?<latency>\d+)\""
| rex field=_raw "ct=\"(?<connectTime>\d+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| stats count, avg(connectTime) as connectTime, avg(latency) as latency, avg(elapsedTime) as elapsedTime by label
| eval connectTime=round(connectTime,3)
| eval latency=round(latency,3)
| eval elapsedTime=round(elapsedTime,3)
```
## Table of failed transactions
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| rex field=_raw "lb=\"(?<label>\w+)\""
| rex field=_raw "rc=\"(?<responseCode>[^\"]+)\""
| rex field=_raw "conversationId=\"(?<conversationId>[^\"]+)\""
| rex field=_raw "\bs=\"(?<isSuccess>\w+)\""
| search isSuccess=false applicationName=$applicationName$ testId=$testId$ applicationVersion=$applicationVersion$
| rename label as Label responseCode as ResponseCode isSuccess as isSuccess? conversationId as ConversationId
| table Label, ResponseCode, isSuccess?, ConversationId
```
## Threads executing - grouped by labels
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| rex field=_raw "lb=\"(?<label>\w+)\""
| rex field=_raw "ng=\"(?<threadsInThisGroup>\d+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| timechart span=10s avg(threadsInThisGroup) by label limit=20
```
## Total threads executing for the test
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| rex field=_raw "lb=\"(?<label>\w+)\""
| rex field=_raw "na=\"(?<threadsInAllGroups>\d+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| timechart span=10s avg(threadsInAllGroups)
```
## Failures over time - grouped by labels
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| rex field=_raw "lb=\"(?<label>\w+)\""
| rex field=_raw "\bs=\"(?<isSuccess>\w+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| timechart  count(eval(isSuccess != "true")) as failedTransactions by label
```
## Total bytes sent and received for the test
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| rex field=_raw "by=\"(?<receivedBytes>\w+)\""
| rex field=_raw "sby=\"(?<sentBytes>\w+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| timechart avg(sentBytes) as sentBytes, avg(receivedBytes) as receivedBytes
```
## Percentage of successful transactions
```perl
index=main sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| rex field=_raw "lb=\"(?<label>\w+)\""
| rex field=_raw "\bs=\"(?<isSuccess>\w+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| stats count(eval(isSuccess == "true")) as successfulTransactions, count as totalTransactions
| eval percentageSuccess=((successfulTransactions/totalTransactions)*100)
| eval percentageSuccess=round(percentageSuccess,0)
| table percentageSuccess
```
## Average elapsed time for the test
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| rex field=_raw "\bt=\"(?<elapsedTime>\d+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| stats avg(elapsedTime) as elapsedTime
| eval elapsedTime=round(elapsedTime,0)
```
## Average TPS for the test
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| timechart span=1s count as totalTransactions
| stats avg(totalTransactions) as avgTPS
| eval avgTPS=round(avgTPS,0)
```
## Total concurrent users for the test
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| rex field=_raw "na=\"(?<threadsInAllGroups>\d+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| stats avg(threadsInAllGroups)
```
## Count of response codes
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "applicationName=\"(?<applicationName>\w+)\""
| rex field=_raw "testId=\"(?<testId>\w+)\""
| rex field=_raw "testDesc=\"(?<testDesc>\w+)\""
| rex field=_raw "applicationVersion=\"(?<applicationVersion>\w+)\""
| rex field=_raw "rc=\"(?<responseCode>[^\"]+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| stats count by responseCode
```
## Users location
```perl
index=wdat-ee-ra-pe sourcetype=jmeter httpSample
| rex field=_raw "ipAddress=\"(?<ipAddress>\w+)\""
| search applicationName=dcl testId=20180110 applicationVersion=4.5
| iplocation ipAddress 
| geostats count by ipAddress
```
