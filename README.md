# graylog2_iis
Configuration to get IIS logs into [Graylog2](https://github.com/Graylog2/graylog2-server) using [Graylog2 collector-sidecar](https://github.com/Graylog2/collector-sidecar) (FileBeats), after which all fields are extracted from the message into searchable fields.
In this example, the field log_timestamp is also converted from unmarked UTC to Europe/Stockholm.

## Currently missing or not working

- I haven't handled `C:/Windows/System32/LogFiles/HTTPERR/*.log` yet. 
- - [Error logging in HTTP APIs](https://support.microsoft.com/en-us/help/820729/error-logging-in-http-apis) 

# Configure IIS to get the logs in the correct format

Set the correct formatting of the log-files
```
%windir%\system32\inetsrv\appcmd.exe set config -section:system.applicationHost/sites -siteDefaults.logfile.logFormat:"W3C"
```

Add the necessary fields
```
%windir%\system32\inetsrv\appcmd.exe set config -section:system.applicationHost/sites -siteDefaults.logfile.logExtFileFlags:Date,Time,ClientIP,UserName,SiteName,ComputerName,ServerIP,Method,UriStem,UriQuery,HttpStatus,Win32Status,BytesSent,BytesRecv,TimeTaken,ServerPort,UserAgent,Cookie,Referer,ProtocolVersion,Host,HttpSubStatus
```

If you wish to change the path to where the logs are stored, edit the command below and run it
```
%windir%\system32\inetsrv\appcmd.exe set config -section:system.applicationHost/sites -siteDefaults.logfile.directory:"%SystemDrive%\inetpub\logs\LogFiles"
```

# Graylog

## Input

For this configuration to work, we'll need to setup a Beats-input.
Select __Beats__ and click __Launch new input__ if you don't already have one you want to use.
You will not need to enter any special configuration here to get it running, but keep track of what port the Input is listening on.
You might need to set a __bind_adress__ _(what IP this Input will be listening on)_.

### Manage extractors

If we want to creat our extactor through the GUI, we will need events in our stream first.
You can however import extractors before that if you have an export of the extractor you want to import.  
Below, I've described how to do it both ways.  
  
* _Before we can create our extractor, we need some events in our input_, so return to this step when you have recieved your first message and click __Manage extractors__.  
Then click on __Get started__ to create the extractor described in the table below.  
* Or you can import the extractor by clicking on __Actions__ and then __Import extractors__, then copy the JSON-formatted message below this table, _then you don't have to wait until a message has been harvested/saved in Graylog/Elasticsearch._

|Setting|Choice
|-|-|
|Extractor type|Grok pattern
|Source field|message
|Grok pattern|`%{TIMESTAMP_ISO8601:log_timestamp} %{WORD:S-SiteName} %{HOSTNAME:S-ComputerName} %{IPORHOST:S-IP} %{WORD:CS-Method} %{URIPATH:CS-URI-Stem} %{NOTSPACE:CS-URI-Query} %{NUMBER:S-Port} %{NOTSPACE:CS-Username} %{IPORHOST:C-IP} %{NOTSPACE:CS-Version} %{NOTSPACE:CS-UserAgent} %{NOTSPACE:CS-Cookie} %{NOTSPACE:CS-Referer} %{NOTSPACE:CS-Host} %{NUMBER:SC-Status} %{NUMBER:SC-SubStatus} %{NUMBER:SC-Win32-Status} %{NUMBER:SC-Bytes} %{NUMBER:CS-Bytes} %{NUMBER:Time-Taken}`
|Condition|Only attempt extraction if field matches regular expression
|Field matches regular expression|`W3SVC`
|Extraction strategy|Copy
|Extractor title|IIS-WC3 Full

_The JSON-formatted importable extractor_
```json
{
  "extractors": [
    {
      "title": "IIS-WC3 Full",
      "extractor_type": "grok",
      "converters": [],
      "order": 0,
      "cursor_strategy": "copy",
      "source_field": "message",
      "target_field": "",
      "extractor_config": {
        "grok_pattern": "%{TIMESTAMP_ISO8601:log_timestamp} %{WORD:serviceName} %{SERVERNAME:serverName} %{IP:serverIP} %{WORD:method} %{URIPATH:uriStem} %{NOTSPACE:uriQuery} %{NUMBER:port} %{NOTSPACE:username} %{IPORHOST:clientIP} %{NOTSPACE:protocolVersion} %{NOTSPACE:userAgent} %{NOTSPACE:cookie} %{NOTSPACE:referer} %{NOTSPACE:requestHost} %{NUMBER:response} %{NUMBER:subresponse} %{NUMBER:win32response} %{NUMBER:bytesSent} %{NUMBER:bytesReceived} %{NUMBER:timetaken}",
        "named_captures_only": false
      },
      "condition_type": "string",
      "condition_value": "W3SVC"
    }
  ],
  "version": "2.4.6"
}
```

## Collector

### Beats Input

|Setting|Value
|-|-|
|Path to logfile|`['C:\inetpub\logs\LogFiles\*\*.log']`
|Encoding|`utf-8`
|Type of input file|`iis`
|Lines that you want Filebeat to exclude|`['^#']`

## Pipelines

In this example, I'm using two rules in two stages;
1. The first one modifies the value log_timestamp so it's stored as a datetime object containing also containing timezone, I also change the timezone to my local timezone to make it easier to read
1. The second pipeline uses the now corrected log_timestamp as a source to set the field timestamp, otherwise timestamp will be referring to when Filebeats sent the event to Graylog and not when it actually happened. Since IIS can wait a while before writing to it's logfile, this means that the data we get will be slightly unreliable (and/or if Filebeat for whatever reason wasn't running for a while, all of the new events that it'll send later will have the same timestamp, even though the events could be from different days!)

### Set correct time for log_timestamps

For the the first __Stage__ _(Stage 0)_, we will set the log_timestamp with the time converted to our own timezone _(Europe/Stockolm)_.

*For this rule to work, __Pipeline Processor__ must run after __Message Filter Chain__ (edit this under __System - Configuration__ by pressing __Update__ below the table showing the current order and move __Pipeline Processor__ to after __Message Filter Chain__*

This rule will 
- read all messages that are of type __iis__ and contains the field __log_timestamp__
- parse the field to add it's correct timezone (UTC)
- save the field with "Europe/Stockholm" as timezone in the format: yyyy-MM-dd HH:mm:ss +0100 (non-DST, +0200 when DST is in effect)

```
rule "Set correct IIS log_timestamp"
when
	contains(
            value: to_string($message.type), 
            search: "iis",
            ignore_case: true)
	AND has_field(field: "log_timestamp")
then
	let set_timezone = parse_date(
				value: to_string($message.log_timestamp),
				pattern: "yyyy-MM-dd HH:mm:ss",
				timezone: "UTC");
	let correct_timestamp = format_date(
					value: set_timezone,
					format: "yyyy-MM-dd HH:mm:ss Z",
					timezone: "Europe/Stockholm");
    set_field("log_timestamp", correct_timestamp);
end
```

### Set timestamp to the timestamp from the logfile

For the the second __Stage__ _(Stage 1)_, we will set the timestamp we just corrected in Stage 0 _(Graylog will store it in UTC-format)_

*For this rule to work, __Pipeline Processor__ must run after __Message Filter Chain__ (edit this under __System - Configuration__ by pressing __Update__ below the table showing the current order and move __Pipeline Processor__ to after __Message Filter Chain__*  

This rule will:
- Use the field log_timestamp as a source for the correct time
- Convert/parse log_timestamp into a format Graylog2 can store
- And finallly store the data.

```
rule "Set timestamp to timestamp from IIS own log-file"
when
	contains(
            value: to_string($message.type), 
            search: "iis",
            ignore_case: true)
	AND has_field(field: "log_timestamp")
then
	let log_timestamp = $message.log_timestamp;
	let new_timestamp = parse_date( value: to_string(log_timestamp),
					pattern: "yyyy-MM-dd HH:mm:ss Z");						
	set_field("timestamp", new_timestamp);
end
```

# Info

 http://www.nsi.bg/nrnm/Help/iisHelp/iis/htm/core/iiintlg.htm 
 https://docs.microsoft.com/en-us/windows/desktop/http/w3c-logging 

|Swedish name|Field-name according to logfile|GROK
|-|-|-|
|Datum|date|(date+time is handled by the rule below)
|Tid|time|%{TIMESTAMP_ISO8601:log_timestamp}
|Servicenamn|s-sitename|%{WORD:S-SiteName}
|Servernamn|s-computername|%{HOSTNAME:S-ComputerName}
|IP-adress för server|s-ip|%{IPORHOST:S-IP}
|Metod|cs-method|%{WORD:CS-Method}
|URI-stam|cs-uri-stem|%{URIPATH:CS-URI-Stem}
|URI-fråga|cs-uri-query|%{NOTSPACE:CS-URI-Query}
|Server-port|s-port|%{NUMBER:S-Port}
|Användarnamn|cs-username|%{NOTSPACE:CS-Username}
|IP-adress för klient|c-ip|%{IPORHOST:C-IP}
|Protokollversion|cs-version|%{NOTSPACE:CS-Version}
|Användaragent|cs(User-Agent)|%{NOTSPACE:CS-UserAgent}
|Cookie|cs(Cookie)|%{NOTSPACE:CS-Cookie}
|Referenssida|cs(Referer)|%{NOTSPACE:CS-Referer}
|Värd|cs-host|%{NOTSPACE:CS-Host}
|Protokollstatus|sc-status|%{NUMBER:SC-Status}
|Understatus för protokoll|sc-substatus|%{NUMBER:SC-SubStatus}
|Win32-Status|sc-win32-status|%{NUMBER:SC-Win32-Status}
|Skickade bytes|sc-bytes|%{NUMBER:SC-Bytes}
|Mottagna bytes|cs-bytes|%{NUMBER:CS-Bytes}
|Tidsåtgång|time-taken|%{NUMBER:Time-Taken}
