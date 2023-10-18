---
layout: post
title: Android cheet sheet
date: 2023-10-18 10:14:00-0400
description: check list of android application for appsec-engineers
tags: android appsec cheetsheet
categories: sample-posts
giscus_comments: true
related_posts: false
toc:
  sidebar: left
---

### Introduction
{:data-toc-text="Introduction"}
This post contains a list of checks that must be performed when testing the cybersecurity of Android-based applications
Each described check, if possible, includes tools that can be used and commands
<br/><br/>

### No SSL-Pinning
{:data-toc-text="No SSL-Pinning"}

---

### Storing sensitive data
Android application can local store various sensitive data like access tokens, passwords, cookies and e.t.c


#### sdcard
Check if Android.Manifest.xml contains next line:
```xml

<uses-permission android:name="android.permission. WRITE_EXTERNAL_STORAGE"/>

```
If it exists, then do recursive search in /sdcard directory

#### logcat
For comfortable start to capture log before interact with application.

```bash
adb logcat > log.txt
```
Recommended to maximum interact parts of application which use user data to leak it in logcat. After, grep these sensitive data in saved part of logs

#### cache
Check at the backend's caching permission - the Cache-Control HTTP-Header does not contain the no-cache, no-store, must-revalidate directive for authorization requests and responses.

If it is, grep for various sensitive data in /data/data/android_app/cache

#### shared prefs
---


### Tapjacking
Analogue of web-vulnerability clickjacking for Android. 
More detailed information and recommendations for fixing is [here](https://developer.android.com/topic/security/risks/tapjacking).\
Check it by installing [APK] (https://github.com/dzmitry-savitski/tapjacker) and try that tapjacker application obscuring the activity of the testing application.