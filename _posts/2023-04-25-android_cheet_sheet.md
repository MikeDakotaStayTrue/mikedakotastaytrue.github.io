---
layout: post
title: Android cheet sheet
date: 2023-10-21 10:14:00-0400
description: Check list of android application for AppSec-engineers
tags: android appsec cheetsheet
categories: cheetsheet
giscus_comments: true
related_posts: false
toc:
  sidebar: left
---

{% include figure.html path="assets/img/android.jpg" title="android_cheetsheet" class="img-fluid rounded z-depth-1" %}


### Introduction
{:data-toc-text="Introduction"}
This post contains a list of checks that must be performed when testing the cybersecurity of Android-based applications.\
Each described check, if possible, includes tools that can be used and commands.
<br/><br/>

### No SSL-pinning
{:data-toc-text="No SSL-Pinning"}
Lack of SSL-pinning significantly simplify MITM-attacks.
Just set third-party certificate to device, and try to proxy traffic.

Example of adb commands for installing certificate:
```bash
adb root
adb remount
adb push 9a5ba575.0 /system/etc/security/cacerts/
adb shell chmod 664 /system/etc/security/cacerts/9a5ba575.0
```

Let's skip details about this vulnerability, because there are too many articles and information on it on the internet.

---

### Storing sensitive data
Android application can local store various sensitive data like access tokens, passwords, cookies and e.t.c.


#### sdcard
Check if Android.Manifest.xml contains next line:
```xml

<uses-permission android:name="android.permission. WRITE_EXTERNAL_STORAGE"/>

```
If it exists, then do recursive search in /sdcard directory.

#### logcat
For comfortable start to capture log before interact with application.

```bash
adb logcat > log.txt
```
Recommended to maximum interact parts of application which use user data to leak it in logcat. After, grep these sensitive data in saved part of logs.

#### cache
Check at the backend's caching permission - the Cache-Control HTTP-Header does not contain the no-cache, no-store, must-revalidate directive for authorization requests and responses.\
If it is, grep for various sensitive data in /data/data/android_app/cache.

---

### Tapjacking
Analogue of web-vulnerability clickjacking for Android. \
More detailed information and recommendations for fixing is [here](https://developer.android.com/topic/security/risks/tapjacking).\
Check it by installing [APK](https://github.com/dzmitry-savitski/tapjacker) and try that tapjacker application obscuring the activity of the testing application.

---

### AndroidManifest.xml misconfig
Check backup is not allowed (__android:allowBackup=true__) \
Check debug mode is turned off (__android:debuggable=true__) \
Check unsecured traffic not permitted (__android:usesCleartextTraffic=true__)

---

### network_security_config.xml misconfig
Config file path - resources/res/xml/network_security_config.xml. \
If network_security_config is specified in the AndroidManifest.xml, then it is worth checking for the presence of the __cleartextTrafficPermitted="true"__ setting and the trust-anchors tag.

Example of vulnerable config:
```xml
<domain-config cleartextTrafficPermitted="true">
    <domain includeSubdomains="true">testdomain.com
    </domain>
</domain-config>
...
<trust-anchors>
        <certificates src="user" />
        <certificates src="system"/>
</trust-anchors>
```

---

### Weak Crypto

#### Algorithms
Check the keywords by code or scanners and see if the data is used by the algorithm.\
Vulnerable encryption algorithms: DES, 3DES, RC2, RC4, BLOWFISH\
Vulnerable hashing algorithms: MD5, MD4, SHA1 \
Recommended: AES and SHA-256

#### Random function
Bad decision for generating numbers of sensetive data - __java.util.Random__\
Good one - __java.security.SecureRandom__

---

### Insecure biometric authentication
Code example of biometric key generation by alias:
```java
private SecretKey genSecretKeyByAlias(String alias) {

  KeyGenerator keyGenerator = KeyGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_AES, 
    KEYSTORE);

  KeyGenParameterSpec.Builder builder = new KeyGenParameterSpec.Builder(
    alias, 
    KeyProperties.PURPOSE_ENCRYPT | KeyProperties.PURPOSE_DECRYPT);

  builder.setKeySize(AesConstants.KEY_LENGTH);
  builder.setBlockModes(AesConstants.BLOCK_MODE);
  builder.setEncryptionPaddings(AesConstants.PADDING);
  builder.setUserAuthenticationRequired(true);
  builder.setRandomizedEncryptionRequired(false);
  keyGenerator.init(builder.build());
  return keyGenerator.generateKey();
  ...
```

It is necessary to check that the __setInvalidatedByBiometricEnrollment__ option is not set to false. \
Look at the entire biometrics verification flow: generating a key, adding it to the keystore, interacting with share_prefs, creating a CryptoObject, checking all cryptographic algorithms that are used, e.t.c.\
Useful video is [here](https://www.youtube.com/watch?v=uN6IocCyIF0&t=1207s).

---

### ProGuard turned off
Check file build.gradle in repository to determine, if ProGuard is used.\
Option minifyEnabled must be set in true.

Example of secure configured gradle file:
```grandle
buildTypes {
        debug {
            firebaseCrashlytics {
                mappingFileUploadEnabled false
                nativeSymbolUploadEnabled false
}
} release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.
txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.release
}
...
```

---

### Deeplink exploitation
Testing deeplinks similar to searching web client-side vulnerabilities (csrf, session hijacking and e.t.c).\
Check custom schemes in intent-filter tag in AndroidManifest.xml. \
Example below can let to use deeplinks like __myapp://user/somedata/__ or __mysuperapp://user/somedata__.
```xml  
<intent-filter android:label="@string/filter_view_myapp_account">
  <data android:host="user" android:pathPrefix="/" android:scheme="myapp"/>
  <data android:host="user" android:pathPrefix="/" android:scheme="mysuperapp"/>
</intent-filter>
```

Adb command to trigger deeplink in Android:
```bash
adb shell am start -W -a android.intent.action.VIEW -d "myapp://user"
```

---

### Insecure pin-code authentication
Just as in the case of biometrics, check the entire flow: generating a key, adding it to the keystore, interacting with share_prefs, creating a CryptoObject, checking all cryptographic algorithms that are used, e.t.c.
1. It is necessary to check whether the key can be restored using the values from shared_prefs.
2. Is EncryptedSharedPreferences used for sensitive data

---

### Timeout no reauthentication
Applications that handle sensitive data should, as a good security practice, re-prompt for a PIN or another authentication after some time of inactivity.\
The blocking time should be within reasonable limits.

Possible steps to check:
* Disable auto-lock screen
* Log in to the application
* Check the PIN code request when the application is open
  + Leave the application open for some time (for example 15 minutes)
  + If the application does not ask you to enter the PIN code again, then the application is vulnerable 
* Check the PIN code request when the application is minimized
  + Minimize the application and leave it for a while
  + We return to the application and check the appearance of the PIN code entry window

---

### Keyboard cache
Check text fields that can process sensitive data for the presence of the __textNoSuggestions__ flag, as in the code below:
```xml
<EditText android:id="@+id/KeyBoardCache" android:inputType="textNoSuggestions"/>
```

### Emulator detection

---

### Root user detection

---