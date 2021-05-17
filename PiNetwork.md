# Pi Network's heel of Achilles
After hearing the news from [Ryan](https://twitter.com/0dayctf/status/1383494676141903877) on Twitter about Pi Network, it seemed like he worked with the iOS version of the app and it was only a announcement and isn't really detailed so we decided to do our own analysis on the latest Android version of the Pi Network app ([version 1.30.3](https://play.google.com/store/apps/details?id=com.blockchainvault)).

While doing our own thorough investigation, we encountered many difficulties such as SSL Pinning preventing us from just strait up sliding in Burp Suite and reading the data so the steps taken to address this are the followings.

## Preparing the environment
Tools that we used:

* Burp Suite
* Genymotion
* Frida
* adb

Since we're going to work with Pi Network, the first step would be to actually get it running somewhere (preferably a virtual environment that we can dispose of later). I prefer Genymotion myself but you're free to choose what ever works for you, just make sure that it is rooted (this is needed for the SSL Pinning bypass later).

## Installing Pi Network and bypassing SSL Pinning
Since we don't already have the app, we'd have to install it via the official Play Store.

To get started with the SSL Pinning Bypass, we'll have to install Frida.

```sh
python -m pip install Frida objection frida-tools
```

Then prepare the injection script by saving [this](https://codeshare.frida.re/@akabe1/frida-multiple-unpinning/) script as ``sslbypass.js``ss

Then download the frida-server binary your device from [here](https://github.com/frida/frida/releases/) (it's ``frida-server-14.2.18-android-x86.xz`` in my case).

Decompressing the .xz file:

```sh
xz --decompress frida-server-14.2.18-android-x86.xz
```


I highly suggest renaming the binary to ``frida-server`` just to 
make it easier.

```sh
mv frida-server-14.2.18-android-x86 frida-server
```

Next up we'll push the server binary onto the device
```sh
adb push frida-server /data/local/tmp
```

And give it permission
```sh
adb shell chmod 755 /data/local/tmp/frida-server
```

After generating the certificate from Burpsuite, push it onto the device and install it.	You can check out [this](https://distributedcompute.com/2019/08/15/tech-note-installing-burp-certificate-on-android-9/) blog post for it.

Starting up frida-server:

```sh
adb shell /data/local/tmp/frida-server &
```

To check if Frida is running:

```sh
frida-ps -U
```

Finally to start PiNetwork and bypass it's SSL Pinning
```
frida -U -f com.blockchainvault -l sslbypass.js --no-pause
```


Voilà, we successfully bypassed SSL Pinning mechanism of the app.

![](https://i.imgur.com/op3WYR3.png)

## Pi Network truly free?
After installing and registering for Pi Network, monitoring the app's traffic reveals this:

```
POST /mediation?adUnit=3&sessionId=0ce81eb8-b562-4f48-8462-10aa272399b4&appKey=b3495a45&compression=false HTTP/1.1
Content-Type: application/json
Content-Length: 1130
User-Agent: Dalvik/2.1.0 (Linux; U; Android 7.0; Google Nexus 9 B
uild/NBD92Y)
Host: outcome-ssp.supersonicads.com
Connection: close
Accept-Encoding: gzip, deflate

{"userIdType":"userGenerated","userId":"demoapp","appKey":"b3495a45","connectionType":"wifi","isLimitAdTrackingEnabled":false,"gmtMinutesOffset":-240,"sessionId":"23547525-5108-492c-9636-b832af30933f","bundleId":"com.blockchainvault","mobileCarrier":"Android","jb":"true","internalFreeMemory":11295,"advertisingIdType":"GAID","appVersion":"1.30.3","sdkVersion":"7.1.1","deviceOEM":"unknown","firstSession":"true","osVersion":"24(7.0)","deviceModel":"Google Nexus 9","advertisingId":"0ce81eb8-b562-4f48-8462-10aa272399b4","language":"en","deviceOS":"Android","externalFreeMemory":11295,"battery":100,"abt":"A","groupIdRV":"1295821","is_coppa":"false","groupIdIS":"1885138","groupIdBN":"1295825","internalTestId":"{}","timestamp":1621254220016,"adUnit":3,"events":[{"provider":"Mediation","eventSessionId":"23547525-5108-492c-9636-b832af30933f","eventId":52,"timestamp":1621254219346},{"provider":"Mediation","ext1":"appLanguage=Kotlin,kiag,androidx=true","sessionDepth":1,"eventSessionId":"23547525-5108-492c-9636-b832af30933f","connectionType":"wifi","firstSessionTimestamp":1621254219352,"eventId":14,"timestamp":1621254219368}]}
```

As you can see, this request contains many information on the device that you install the app on such as:

* The name of the device (Google Nexus 9)
* Android version (7.0)
* Android API version (24)
* Connection type (wifi)
* Internal Free Memory (11295)

The app is designed with an affiliate system designed under Earning Team > Invite > Invite your contacts (which requires the permission to read your Contacts). We saw that after granting the app the permission to read our contacts, there was a request going out to Pi Network's server which looks like the following:

```
POST /api/contacts/upload_list HTTP/1.1
Host: socialchain.app
Connection: close
Content-Length: 2235
Accept: application/json, text/plain, /
Origin: https://app-cdn.minepi.com
Authorization: Bearer BaxgJZlhTwtzHopK1Pwy9Zw9Qy9zsI6wKNFUML1Kbjg
User-Agent: Mozilla/5.0 (Linux; Android 7.0; Google Nexus 9 Build/NBD92Y; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/74.0.3729.186 Safari/537.36
Content-Type: application/json;charset=UTF-8
Referer: https://app-cdn.minepi.com/mobile-app-ui/invite
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
X-Requested-With: com.blockchainvault

{"contacts":[{"postalAddresses":[],"emailAddresses":[],"urlAddresses":[],"note":"","department":"","company":"","familyName":"1","displayName":"contact 1","givenName":"contact","rawContactId":"1","recordID":"1","phoneNumbers":[{"id":"2","label":"mobile","number":"(038) 888-4448"}],"thumbnailPath":"","hasThumbnail":false,"jobTitle":"","suffix":null,"prefix":null,"middleName":null},{"postalAddresses":[],"emailAddresses":[],"urlAddresses":[],"note":"","department":"","company":"","familyName":"2","displayName":"contact 2","givenName":"contact","rawContactId":"2","recordID":"2","phoneNumbers":[{"id":"8","label":"mobile","number":"(038) 845-6789"}],"thumbnailPath":"","hasThumbnail":false,"jobTitle":"","suffix":null,"prefix":null,"middleName":null},{"postalAddresses":[],"emailAddresses":[],"urlAddresses":[],"note":"","department":"","company":"","familyName":"3","displayName":"Contact 3","givenName":"Contact","rawContactId":"3","recordID":"3","phoneNumbers":[{"id":"14","label":"mobile","number":"(038) 812-3456"}],"thumbnailPath":"","hasThumbnail":false,"jobTitle":"","suffix":null,"prefix":null,"middleName":null},{"postalAddresses":[],"emailAddresses":[],"urlAddresses":[],"note":"","department":"","company":"","familyName":"4","displayName":"contact 4","givenName":"contact","rawContactId":"4","recordID":"4","phoneNumbers":[{"id":"20","label":"mobile","number":"(038) 812-3211"}],"thumbnailPath":"","hasThumbnail":false,"jobTitle":"","suffix":null,"prefix":null,"middleName":null},{"postalAddresses":[],"emailAddresses":[],"urlAddresses":[],"note":"","department":"","company":"","familyName":"5","displayName":"contact 5","givenName":"contact","rawContactId":"5","recordID":"5","phoneNumbers":[{"id":"26","label":"mobile","number":"1 234-567-89"}],"thumbnailPath":"","hasThumbnail":false,"jobTitle":"","suffix":null,"prefix":null,"middleName":null},{"postalAddresses":[],"emailAddresses":[],"urlAddresses":[],"note":"","department":"","company":"","familyName":"6","displayName":"Contact 6","givenName":"Contact","rawContactId":"6","recordID":"6","phoneNumbers":[{"id":"32","label":"mobile","number":"1 234-560-000"}],"thumbnailPath":"","hasThumbnail":false,"jobTitle":"","suffix":null,"prefix":null,"middleName":null}]}
```

And the response:

```
HTTP/1.1 200 OK
content-type: application/json; charset=utf-8
status: 200 OK
cache-control: max-age=0, private, must-revalidate
access-control-allow-origin: https://app-cdn.minepi.com
vary: Origin
referrer-policy: strict-origin-when-cross-origin
x-permitted-cross-domain-policies: none
access-control-max-age: 7200
x-xss-protection: 1; mode=block
x-request-id: 520179dd-a1f2-46d0-a499-cd29560b569b
access-control-allow-credentials: true
access-control-allow-methods: GET, POST, OPTIONS, PUT, DELETE
x-download-options: noopen
etag: W/"adf50f26c361b0882bfd089adef2c5c8"
x-frame-options: SAMEORIGIN
x-runtime: 0.123402
x-content-type-options: nosniff
date: Mon, 17 May 2021 14:09:06 GMT
x-powered-by: Phusion Passenger 6.0.6
server: nginx/1.14.0 + Phusion Passenger 6.0.6
connection: close
Content-Length: 205

{"id":182902471,"user_id":339408592,"dto":{"hash":-2128208535025350508},"created_at":"2021-05-17T14:09:05.000Z","updated_at":"2021-05-17T14:09:06.000Z","processing_completed_at":"2021-05-17T14:09:06.000Z"}
```

We saw that every time the user accesses this particular part of the app, it autonomously sends new entries in your contacts list to Pi Network's server.

Finally, we also found issues with the session management system of the app which allows anyone to get the account's information including: user data, contact entries, etc using the old Bearer Token even after logging out or DELETING their own account (which SHOULD'VE erased all information from the server promptly).

![Deleting the account](https://i.imgur.com/Ruwldq6.png)

For example, after we've deleted our account, we tried the following request several times. At first it didn't work and returned the status code 401 but after that, it started giving us the data back.

![](https://i.kym-cdn.com/entries/icons/original/000/027/475/Screen_Shot_2018-10-25_at_11.02.15_AM.png)	

The request:
```
GET /api/contacts/processed_list HTTP/1.1
Host: socialchain.app
Connection: close
Accept: application/json, text/plain, /
Origin: https://app-cdn.minepi.com
Authorization: Bearer BaxgJZlhTwtzHopK1Pwy9Zw9Qy9zsI6wKNFUML1Kbjg
User-Agent: Mozilla/5.0 (Linux; Android 7.0; Google Nexus 9 Build/NBD92Y; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/74.0.3729.186 Safari/537.36
Referer: https://app-cdn.minepi.com/mobile-app-ui/invite
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
X-Requested-With: com.blockchainvault
If-None-Match: W/"9f62fd28f30bf506a89642950b47dc8a"
```

And lo a behold, the fucking data that should've been erased off the face of the earth.
```
HTTP/1.1 200 OK
content-type: application/json; charset=utf-8
status: 200 OK
cache-control: max-age=0, private, must-revalidate
access-control-allow-origin: https://app-cdn.minepi.com
vary: Origin
referrer-policy: strict-origin-when-cross-origin
x-permitted-cross-domain-policies: none
access-control-max-age: 7200
x-xss-protection: 1; mode=block
x-request-id: 375ff958-cac4-4d8d-ad84-1856637e9917
access-control-allow-credentials: true
access-control-allow-methods: GET, POST, OPTIONS, PUT, DELETEx-download-options: noopen
etag: W/"25adc1a0835e258417663183c98fcc97"
x-frame-options: SAMEORIGIN
x-runtime: 0.078783
x-content-type-options: nosniff
date: Mon, 17 May 2021 15:49:05 GMT
x-powered-by: Phusion Passenger 6.0.6
server: nginx/1.14.0 + Phusion Passenger 6.0.6
connection: close
Content-Length: 1395

{"processed_contacts":{"created_contacts":[{"id":15388648311,"user_id":339408592,"first_name":"contact","last_name":"1","phone_record_id":"1","existing_user_id":null,"created_at":"2021-05-17T13:21:47.000Z","updated_at":"2021-05-17T13:21:47.000Z"},{"id":15388648345,"user_id":339408592,"first_name":"contact","last_name":"2","phone_record_id":"2","existing_user_id":204603621,"created_at":"2021-05-17T13:21:47.000Z","updated_at":"2021-05-17T13:25:28.000Z"},{"id":15388648367,"user_id":339408592,"first_name":"Contact","last_name":"3","phone_record_id":"3","existing_user_id":220411953,"created_at":"2021-05-17T13:21:47.000Z","updated_at":"2021-05-17T13:25:28.000Z"},{"id":15389264594,"user_id":339408592,"first_name":"contact","last_name":"4","phone_record_id":"4","existing_user_id":null,"created_at":"2021-05-17T13:34:58.000Z","updated_at":"2021-05-17T13:34:58.000Z"},{"id":15389753378,"user_id":339408592,"first_name":"contact","last_name":"5","phone_record_id":"5","existing_user_id":null,"created_at":"2021-05-17T13:45:12.000Z","updated_at":"2021-05-17T13:45:12.000Z"},{"id":15390950941,"user_id":339408592,"first_name":"Contact","last_name":"6","phone_record_id":"6","existing_user_id":null,"created_at":"2021-05-17T14:09:05.000Z","updated_at":"2021-05-17T14:09:05.000Z"}],"matching_users":[{"id":220411953,"username":"mammd123","display_name":"Mam Map","trusted":false}]},"status":"ready"}
```

So there you have it, the answer to this section's title.

## Conclusion
Just to be clear, this is just our security research on Pi Network and we hope that people are aware of it's problems and ask for an explanation from it.

We are ManhNho and Cu64, wear a mask, peace out.