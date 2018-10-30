# graylog2_iis
Configuration to get IIS logs into Graylog using Graylog collector-sidecar (FileBeats), after which all fields are extracted from the message into searchable fields.
In this example, the field log_timestamp is also converted from unmarked UTC to Europe/Stockholm.

## Currently missing or not working

- I haven't handled `C:/Windows/System32/LogFiles/HTTPERR/*.log` yet. 
- - [Error logging in HTTP APIs](https://support.microsoft.com/en-us/help/820729/error-logging-in-http-apis) 
- For whatever reason, I'm unable to overwrite the field timestamp

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

Before we continue to another part of the configuration, click __Manage extractors__ and create the rule below.

|Setting|Choice
|-|-|
|Extractor type|Grok pattern
|Source field|message
|Grok pattern|`%{TIMESTAMP_ISO8601:log_timestamp} %{WORD:S-SiteName} %{HOSTNAME:S-ComputerName} %{IPORHOST:S-IP} %{WORD:CS-Method} %{URIPATH:CS-URI-Stem} %{NOTSPACE:CS-URI-Query} %{NUMBER:S-Port} %{NOTSPACE:CS-Username} %{IPORHOST:C-IP} %{NOTSPACE:CS-Version} %{NOTSPACE:CS-UserAgent} %{NOTSPACE:CS-Cookie} %{NOTSPACE:CS-Referer} %{NOTSPACE:CS-Host} %{NUMBER:SC-Status} %{NUMBER:SC-SubStatus} %{NUMBER:SC-Win32-Status} %{NUMBER:SC-Bytes} %{NUMBER:CS-Bytes} %{NUMBER:Time-Taken}`
|Condition|Only attempt extraction if field matches regular expression
|Field matches regular expression|`W3SVC`
|Extraction strategy|Copy
|Extractor title|IIS-WC3 Full

## Collector

### Beats Input

|Setting|Value
|-|-|
|Path to logfile|`['C:\inetpub\logs\LogFiles\*\*.log']`
|Encoding|`utf-8`
|Type of input file|`iis`
|Lines that you want Filebeat to exclude|`['^#']`

## Pipelines

### Correct log_timestamps

*For this rule to work, __Pipeline Processor__ must run after __Message Filter Chain__ (edit this under __System - Configuration__ by pressing __Update__ below the table showing the current order and move __Pipeline Processor__ to after __Message Filter Chain__*

This rule will 
- read all messages that are of type __iis__ and contains the field __log_timestamp__
- parse the field to add it's correct timezone (UTC)
- save the field with "Europe/Stockholm" as timezone in the format: yyyy-MM-dd HH:mm:ss +0100 (non-DST, +0200 when DST is in effect)

```
rule "correct IIS-log timestamp"
when
    contains(
            value: to_string($message.type), 
            search: "iis",
            ignore_case: true)
    AND has_field(field: "log_timestamp")
then
    let new_date = format_date(
                        value: parse_date(
                            value: to_string($message.log_timestamp),
                            pattern: "yyyy-MM-dd HH:mm:ss",
                            timezone: "UTC"),
                        format: "yyyy-MM-dd HH:mm:ss Z",
                        timezone: "Europe/Stockholm");
    set_field("log_timestamp", new_date);
end
```

### Set timestamps to timestamps from log, not from time of harvest

The rule doesn't give any errors anymore, but it doesn't change the value for the field __timestamp__ for some reason.

```
rule "replace harvesting-timestamp with actual timestamp from log"
when
    contains(
            value: to_string($message.type), 
            search: "iis",
            ignore_case: true)
then
    set_field("timestamp", format_date(
				value: parse_date(
						value: to_string($message.log_timestamp),
						pattern: "yyyy-MM-dd HH:mm:ss Z"),
				format:"yyyy-MM-dd'T'hh:mm:ss.SSSZ"));		
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
