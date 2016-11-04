## Overview

The security model chosen for the open source version of Parse is to require any type of push functionality to be implemented as Parse Cloud code executed only on the server side.  Unlike Parse's hosted service, **you cannot implement this type of code** on the actual client:

```java
// Note: This does NOT work with Parse Server at this time
ParsePush push = new ParsePush();
push.setChannel("mychannel");
push.setMessage("this is my message");
push.sendInBackground();
```

You will likely see this error in the API response:

```json
{
    "code": 115,
    "error": "Master key is invalid, you should only use master key to send push"
}
```

Instead, you need to write your own server-side Parse code and have the client invoke it. 

### Parse Server Setup

The instructions below apply to the open source version of Parse, not hosted Parse.  

#### Obtain GCM Sender ID and API Key

First, you will need to obtain a Google Cloud Messaging Sender ID and API Key.  You can follow only step 1 of [this guide](https://github.com/codepath/android_guides/wiki/Google-Cloud-Messaging/b7ab0d3329898f147b2fe7a32c731f9ce251893c#step-1-register-with-google-developers-console) to obtain the Sender ID (equivalent to the Project Number) and API Key.  You do not need to follow the other steps because Parse provides much of code to handle GCM registration for you.  Remember the GCM Sender and API key provided.

#### Update Parse Server Config

Modify your `index.js` to add support for GCM.  See [this example](https://github.com/codepath/parse-server-example/blob/master/index.js#L15-L18):

```javascript
if (process.env.GCM_SENDER_ID && process.env.GCM_API_KEY) {
   pushConfig['android'] = { 
   senderId: process.env.GCM_SENDER_ID || '',
   apiKey: process.env.GCM_API_KEY || ''};
}
```

This repo has some additional environment variables configurations added that help facilitate sending push    notifications (i.e. see `GCM_SENDER_ID`, and `GCM_API_KEY` in [index.js](https://github.com/codepath/parse-server-example/blob/master/index.js#L37-L40)).  You need to pass this configuration to the instantiation of the Parse server:

```javascript
var api = new ParseServer({
          // other variables here
              push: pushConfig,
          });
```

#### Setup environment variables

If you are using Heroku to configure the server, make sure to set the following environment variables:

   * Set `GCM_SENDER_KEY` and `GCM_API_KEY` environment variables to correspond to the Sender ID and API Key in the previous step.  

      <img src="http://imgur.com/KgU2S2y.png"/>

   * Confirm the `SERVER_URL` environment variable is set to the URL and Parse mount location (i.e. `http://yourappname.herokuapp.com/parse`).  

   * Verify that `cloud/main.js` is the default value of `CLOUD_CODE_MAIN` environment variable.  

#### Add Parse Cloud Code

Modify [cloud/main.js](https://github.com/codepath/parse-server-example/blob/master/cloud/main.js) yourself to add custom code to send Push notifications.  See [these examples](https://github.com/ParsePlatform/parse-server/issues/401#issuecomment-183767065) for other ways of sending too.  

#### Redeploy Code

Next, redeploy the code.  If you are using Heroku, you need to connect your own forked repository and redeploy.  

<img src="http://i.imgur.com/OmxXc6s.png"/>

### Parse Client Setup

Before getting started, make sure you have Google Play installed on the emulator or device, since push notifications via [Google Cloud Messaging](Google-Cloud-Messaging) (GCM) will only work for devices and emulators that have Google Play installed.  

#### Add GCM Sender ID

Set the GCM Sender ID inside `res/values/strings.xml`:

```xml
<string name="gcm_sender_id">id:1234567</string>
```

Add a `meta-data` with the Sender ID in your AndroidManifest.xml.  

```xml
<application>  
   <!-- Add this INSIDE the application node... -->
   <meta-data android:name="com.parse.push.gcm_sender_id"
              android:value="@string/gcm_sender_id"/>
```
          
NOTE: Make sure the `id:` is used as the prefix (Android treats any metadata value that has a string of digits as an integer so Parse prefixes this value).  If you forget this step, Parse will register with its own Sender ID but you will see `SenderID mismatch` errors when trying to issue push notifications.

#### Add Push Permissions

Add the necessary permissions to your `Android Manifest.xml` file.  These permissions should live outside the `<application>` tag:

```xml
<manifest package="com.codepath.mapdemo">
  <uses-permission android:name="android.permission.INTERNET" />
  <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
  <uses-permission android:name="android.permission.WAKE_LOCK" />
  <uses-permission android:name="android.permission.GET_ACCOUNTS" />
  <uses-permission android:name="android.permission.VIBRATE" />
  <uses-permission android:name="com.google.android.c2dm.permission.RECEIVE" />

  <!-- ${packageName} is substituted for the package name declared above.  -->
  <permission android:protectionLevel="signature" android:name="${packageName}.permission.C2D_MESSAGE" />
  <uses-permission android:name="${packageName}.permission.C2D_MESSAGE" />

  <!-- These permissions should live outside the application tag -->

  <application>
     <!-- Other stuff here -->        
  </application>
</manifest>
```

**NOTE**: The Parse Android SDK permissions are different than those in the official Firebase Cloud Messaging documentation.   In particular, the permissions in the manifest file that you need to set are checked by the Parse Android SDK, so you should **must** use the old permissions as provided.   There is still work underway to upgrade the Parse Android SDK to the latest version of Firebase Cloud Messaging (see [discussion issue](https://github.com/ParsePlatform/Parse-SDK-Android/pull/452)).  

Use the `${packageName}` instead of specifying the package name yourself.  If you mistype this information, push notifications may not be enabled by the Parse Android SDK.

#### Create a Gcm Broadcast Receiver

You will also need to add a Gcm Broadcast receiver for handling GCM messages in your `AndroidManifest.xml` file:

```xml
<application>
    <receiver android:name="com.parse.GcmBroadcastReceiver"
                  android:permission="com.google.android.c2dm.permission.SEND">
        <intent-filter>
            <action android:name="com.google.android.c2dm.intent.RECEIVE" />
            <action android:name="com.google.android.c2dm.intent.REGISTRATION" />

            <category android:name="${packageName}" />
        </intent-filter>
    </receiver>
</application>
```

#### Create a Parse-specific broadcast receiver

Declare a Parse-specific broadcast receiver `AndroidManifest.xml` file:

```xml
<application>
    <service android:name="com.parse.PushService" />
    <receiver android:name="com.parse.ParsePushBroadcastReceiver"
                    android:exported="false">
        <intent-filter>
            <action android:name="com.parse.push.intent.RECEIVE" />
            <action android:name="com.parse.push.intent.DELETE" />
            <action android:name="com.parse.push.intent.OPEN" />
        </intent-filter>
    </receiver>
</application>
```

If you use Parse's default [ParsePushBroadcastReceiver](https://github.com/ParsePlatform/Parse-SDK-Android/blob/master/Parse/src/main/java/com/parse/ParsePushBroadcastReceiver.java#L155-L160), using either `alert` or `title` as a key/value pair will trigger a notification message. See [this section](http://parseplatform.github.io/docs/android/guide/#receiving-pushes) of the Parse documentation.

You can also create your own custom receiver as shown in [this example](https://github.com/codepath/ParsePushNotificationExample/blob/master/app/src/main/java/com/test/MyCustomReceiver.java).

#### Executing Parse Cloud Functions

Assuming the function is named `pushChannelTest`, modify your Android code to invoke this function by using the `callFunctionInBackground()` call.  Any parameters should be passed as a `HashMap`:

```java

HashMap<String, String> test = new HashMap<>();
test.put("channel", "testing");

ParseCloud.callFunctionInBackground("pushChannelTest", test);
```

#### Save GCM Token

Make sure to register the GCM token to the server:

```java
Parse.initialize(...);

// Need to register GCM token
ParseInstallation.getCurrentInstallation().saveInBackground();
```

### Receiving Pushes on Android

After following the steps outlined above, be sure to check out the following resources for more information:

 * [ParsePlatform Push Docs](http://parseplatform.github.io/docs/android/guide/#push-notifications)
 * [ParsePlatform Push QuickStart](https://github.com/ParsePlatform/parse-server/wiki/Push#quick-start)
 * [CodePath Parse Push Example](https://github.com/codepath/ParsePushNotificationExample) 

Refer to these resources for more information on Parse Server push. 

### Troubleshooting 

* If you are using Facebook's Stetho library with your Android client, you can see the LogCat statements and verify that GCM tokens are being registered by API calls to the `/parse/classes/_Installation` endpoint:

```
: Url : http://192.168.3.116:1337/parse/classes/_Installation
```

You should be able to se the `deviceToken`, `installationId`, and `appName` registered:

```
03-02 03:17:27.859 9362-9596/com.test I/ParseLogInterceptor:
Body : {
   "pushType": "gcm",
   "localeIdentifier": "en-US",
   "deviceToken": XXX,
   "appVersion": "1.0",
   "deviceType": "android",
   "appIdentifier": "com.test",
   "installationId": "XXXX",
   "parseVersion": "1.13.1",
   "appName": "PushNotificationDemo",
   "timeZone": "America\/New_York"
}
```

* Make sure you are on latest open source Parse version: [![npm version](https://img.shields.io/npm/v/parse-server.svg?style=flat)](https://www.npmjs.com/package/parse-server)  You will want to verify what version is set in your `package.json` file (i.e. https://github.com/ParsePlatform/parse-server-example/blob/master/package.json#L15).  Make sure to update this file and redeploy.

* If GCM is fully setup, your app if properly configured should register itself with your Parse server.  Check your `_Installation` table to verify that the entries were being saved. Clear your app cache or uninstall the app if an entry in the  `_Installation` table hasn't been added.

* Inside your `AndroidManifest.xml` definition within the application node, make sure your `gcm_sender_id` is prefixed with `id:` (i.e. `id:123456`).  Parse needs to begin with an `id:` to work correctly.
  * Make sure to add the GCM metadata **inside of the `<application>` node** or this won't be picked up properly. 

* You can use this curl command with your application key and master key to send a push to all Android devices:

```bash
 curl -X POST -H "X-Parse-Application-Id: myAppId" -H "X-Parse-Master-Key: myMasterKey" \
-H "Content-Type: application/json" \
-d '{"where": {"deviceType": "android"}, "data": {"action": "com.example.UPDATE_STATUS", "newsItem": "Man bites dog", "name": "Vaughn", "alert": "Ricky Vaughn was injured during the game last night!"}}' \
https://parse-testing-port.herokuapp.com/parse/push/
```

* Use `heroku logs --app <app_name>` to see what is happening on the server side:

```bash
2016-03-03T09:26:50.032609+00:00 app[web.1]: #### PUSH OK
```

If you see `Can not find sender for push type android`, it means you forgot to set the environment variables `GCM_SENDER_ID` and `GCM_API_KEY`.

* Make sure you have **not** included `com.google.android.gms:play-services-gcm:8.4.0` in your Gradle configuration.  Parse's Android SDK library already includes code to deal with the GCM registration.

* Verify that you have all the permissions for GCM setup in your `AndroidManifest.xml` file and that you have the correct receivers configured.

