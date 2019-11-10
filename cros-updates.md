# Getting Notified of Chrome OS Updates

I own too many Chrome OS devices (*Courage!*) . I want to know when there are updates available, so I tried to find a way to get notified whenever updates were posted for my devices. I found a couple ways. Maybe it's ironic that I'm using AWS to do this, but then again I work for AWS, so...

## CrOS Omaha
Google has a service where you can look up the current versions of Chrome OS for all supported devices at [https://cros-omahaproxy.appspot.com](https://cros-omahaproxy.appspot.com/viewer). This service takes a *long* time (about 25s) to respond, presumably because it's pulling data every time you hit it without any caching. But whatever, it works and yields accurate data.

You can get the raw data as a CSV file from [https://cros-omahaproxy.appspot.com/all](https://cros-omahaproxy.appspot.com/all). I tried to get it to yield less data by some other URL but couldn't find anything. If you find a way, please let me know!

## Update Service
A better way is to query Google's update service directly, the same way your Chromebook does. Getting the right data from your Chromebook is a bit trickier, but the latency is a lot better. To discover how your chromebook is polling for updates, have a look at `file:///var/log/update_engine.log` where you'll see something like this buried in there:

```log
[1109/122036.268126:INFO:action_processor.cc(51)] ActionProcessor: starting OmahaRequestAction
[1109/122036.270252:INFO:omaha_request_action.cc(444)] Posting an Omaha request to https://tools.google.com/service/update2
[1109/122036.270303:INFO:omaha_request_action.cc(445)] Request: <?xml version="1.0" encoding="UTF-8"?>
<request requestid="33041542-1d01-475b-84fd-ccb6e8e4e3cb" sessionid="deea5477-4296-41ed-b627-2dd1063d311a" protocol="3.0" updater="ChromeOSUpdateEngine" updaterversion="0.1.0.0" installsource="scheduler" ismachine="1">
    <os version="Indy" platform="Chrome OS" sp="12499.51.0_x86_64"></os>
    <app appid="{BD7F7139-CC18-49C1-A847-33F155CCBCA8}" cohort="1:c/1f:" cohortname="nocturne_stable_canaryrelease_12499.51.0" version="12499.51.0" track="stable-channel" lang="en-US" board="nocturne-signed-mpkeys" hardware_class="NOCTURNE D5B-A5F-B47-H6A-A5L" delta_okay="true" fw_version="" ec_version="" installdate="4683" >
        <updatecheck></updatecheck>
    </app>
</request>

[1109/122036.273080:INFO:libcurl_http_fetcher.cc(141)] Starting/Resuming transfer
[1109/122036.279691:INFO:libcurl_http_fetcher.cc(160)] Using proxy: no
[1109/122036.280139:INFO:libcurl_http_fetcher.cc(300)] Setting up curl options for HTTPS
[1109/122036.409431:INFO:metrics_reporter_omaha.cc(550)] Uploading 0 for metric UpdateEngine.CertificateCheck.UpdateCheck
[1109/122036.410456:INFO:metrics_reporter_omaha.cc(550)] Uploading 0 for metric UpdateEngine.CertificateCheck.UpdateCheck
[1109/122036.411256:INFO:metrics_reporter_omaha.cc(550)] Uploading 0 for metric UpdateEngine.CertificateCheck.UpdateCheck
[1109/122036.497188:INFO:libcurl_http_fetcher.cc(471)] HTTP response code: 200
[1109/122036.497724:INFO:libcurl_http_fetcher.cc(578)] Transfer completed (200), 566 bytes downloaded
[1109/122036.497789:INFO:omaha_request_action.cc(840)] Omaha request response: <?xml version="1.0" encoding="UTF-8"?><response protocol="3.0" server="prod"><daystart elapsed_days="4695" elapsed_seconds="44437"/><app appid="{BD7F7139-CC18-49C1-A847-33F155CCBCA8}" cohort="1:c/1f:" cohortname="nocturne_stable_canaryrelease_12499.51.0" status="ok"><updatecheck _firmware_version_0="1.1" _firmware_version_1="1.1" _firmware_version_2="1.1" _firmware_version_3="1.1" _firmware_version_4="1.1" _kernel_version_0="1.1" _kernel_version_1="1.1" _kernel_version_2="1.1" _kernel_version_3="1.1" _kernel_version_4="1.1" status="noupdate"/></app></response>
```

Trial and error allowed me to work out a smaller, more generic request body that will probably work for a few years yet until backwards compatibility is dropped:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<request protocol="3.0" ismachine="1">
  <app appid="{BD7F7139-CC18-49C1-A847-33F155CCBCA8}" track="stable-channel" board="nocturne-signed-mpkeys" hardware_class="NOCTURNE D5B-A5F-B47-H6A-A5L" delta_okay="false">
    <updatecheck/>
  </app>
</request>
```

Parameters for your Chromebook can be obtained by going to `chrome://system` on your device.

|Attribute|chrome://system Value|
|--|--|
|`appid`|`CHROMEOS_RELEASE_APPID`|
|`track`|`CHROMEOS_RELEASE_TRACK`|
|`board`|`CHROMEOS_RELEASE_BOARD`|
|`hardware_class`|`HWID`|

`POST`ing this query to `https://tools.google.com/service/update2` yields a response like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<response protocol="3.0" server="prod">
  <daystart elapsed_days="4696" elapsed_seconds="30722"/>
  <app appid="{BD7F7139-CC18-49C1-A847-33F155CCBCA8}" cohort="1:c/1f:" cohortname="nocturne_stable_canaryrelease_12499.51.0" status="ok">
    <updatecheck _firmware_version="1.1" _firmware_version_0="1.1" _firmware_version_1="1.1" _firmware_version_2="1.1" _firmware_version_3="1.1" _firmware_version_4="1.1" _kernel_version="1.1" _kernel_version_0="1.1" _kernel_version_1="1.1" _kernel_version_2="1.1" _kernel_version_3="1.1" _kernel_version_4="1.1" status="ok">
      <urls>
        <url codebase="http://dl.google.com/chromeos/nocturne/12499.51.0/stable-channel/"/>
        <url codebase="https://dl.google.com/chromeos/nocturne/12499.51.0/stable-channel/"/>
      </urls>
      <manifest version="12499.51.0">
        <actions>
          <action event="install" run="chromeos_12499.51.0_nocturne_stable-channel_full_mp.bin-e52a7b4317aacd1689bc610656a9bcfb.signed"/>
          <action ChromeOSVersion="12499.51.0" ChromeVersion="78.0.3904.92" IsDeltaPayload="false" MaxDaysToScatter="14" MetadataSignatureRsa="okC/hvhqmQnerQ33y4AWPYFI6yGLHKIPOmzKzb/ri4odvKEmr1KKMWvgXLxzTFTorBpl2I/Wrx634E61cMQSssQKPUQ9hAFXdSorIuO60kEgGZivQVMR4kktETka84SCuORgOzum9VN27V9MQyG3+CIS+C1BflPPGXPd6zw35FTh4LI4HkX6cIy6kTldxZt9V7XywEdLuZpQZmC2PI3kr1Nf9B+scgTwdHaoq9g2hCmbsxq+ivPKVjfVRrWNwVUVnERJs5WfK+27qmuf6a8piC2wl3ApyqzYda4iY/QLsWTuROYVNbf7YWKrPQF1QpzeWLmgDtuAThS0oLkFuGZwNw==" MetadataSize="65824" event="postinstall" sha256="MSAexfVkSoPRVl3sGQJGlY4Gl6pJRoXDXy1+BsHNpdg="/>
        </actions>
        <packages>
          <package fp="1.31201ec5f5644a83d1565dec190246958e0697aa494685c35f2d7e06c1cda5d8" hash_sha256="31201ec5f5644a83d1565dec190246958e0697aa494685c35f2d7e06c1cda5d8" name="chromeos_12499.51.0_nocturne_stable-channel_full_mp.bin-e52a7b4317aacd1689bc610656a9bcfb.signed" required="true" size="1133479208"/>
        </packages>
      </manifest>
    </updatecheck>
  </app>
</response>
```

The piece we're interested in is `ChromeVersion` from this `action` element:

```xml
<action ChromeOSVersion="12499.51.0" ChromeVersion="78.0.3904.92" IsDeltaPayload="false" MaxDaysToScatter="14" MetadataSignatureRsa="okC/hvhqmQnerQ33y4AWPYFI6yGLHKIPOmzKzb/ri4odvKEmr1KKMWvgXLxzTFTorBpl2I/Wrx634E61cMQSssQKPUQ9hAFXdSorIuO60kEgGZivQVMR4kktETka84SCuORgOzum9VN27V9MQyG3+CIS+C1BflPPGXPd6zw35FTh4LI4HkX6cIy6kTldxZt9V7XywEdLuZpQZmC2PI3kr1Nf9B+scgTwdHaoq9g2hCmbsxq+ivPKVjfVRrWNwVUVnERJs5WfK+27qmuf6a8piC2wl3ApyqzYda4iY/QLsWTuROYVNbf7YWKrPQF1QpzeWLmgDtuAThS0oLkFuGZwNw==" MetadataSize="65824" event="postinstall" sha256="MSAexfVkSoPRVl3sGQJGlY4Gl6pJRoXDXy1+BsHNpdg="/>
```

We can get at that element with the XPath query `.//action[@ChromeVersion]`. A fairly minimal Python script to fetch the current version for a Chromebook looks like this:

```python
#!/usr/bin/env python3
# Copyright (c) Andrew Jorgensen. All rights reserved.
# SPDX-License-Identifier: MIT-0
from urllib.request import urlopen
from xml.etree import ElementTree

AUSERVER = "https://tools.google.com/service/update2"
REQUEST = """<?xml version="1.0" encoding="UTF-8"?>
<request protocol="3.0" ismachine="1">
  <app appid="{appid}" track="{track}" board="{board}" hardware_class="{hardware_class}" delta_okay="false">
    <updatecheck/>
  </app>
</request>"""
VERSION_ATTRIB = "ChromeVersion"
VERSION_XPATH = ".//action[@{}]".format(VERSION_ATTRIB)


def chrome_version(appid, track, board, hardware_class):
    """Get the Chrome version for a Chromebook"""
    request = REQUEST.format(
        appid=appid, track=track, board=board, hardware_class=hardware_class
    ).encode()
    with urlopen(AUSERVER, data=request) as response:
        root = ElementTree.parse(response).getroot()
        for action in root.findall(VERSION_XPATH):
            return action.attrib[VERSION_ATTRIB]


if __name__ == "__main__":
    from sys import argv

    print(chrome_version(*argv[1:]))
```

## Getting Notified
To actually get notified whenever one of my Chromebooks gets an update, I've create an AWS Lambda function that queries the update service and stores the result in DynamoDB. When the queried version differs from the stored version, I post a message to an SNS topic. Since I've subscribed my phone to that SNS topic, I get a text message every time the update service gives a new response.

The code for this is in GitHub, as it's more than I want to include in a blog post, and I'll improve it over time.