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

There are machines i've added to "machines to do" list:

* Blocky (Easy)
* Help (Easy)
* Schooled (Medium)
* Magic (Medium)
* JSON (Medium)
* Popcorn (Medium)
* Unattended (Medium)
* Celestial (Medium)
* Vault (Medium)
* Unobtainium (Hard)
* Zipper (Hard)
* Falafel (Hard)
* Monitors (Hard)
* Sink (Insane)

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

---