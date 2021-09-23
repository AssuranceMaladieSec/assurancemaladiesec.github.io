---
layout: post
title:  "security.txt implementation @ Assurance Maladie"
date:   2021-09-22 13:33:19 +0200
author: cbrocas
categories: secops
---

{% include toc %}

# Security vulnerabilities reporting: the problem(s)

**TL;DR:** Finding how to report vulnerabilities found on web sites and web services has always been challenging for security researchers. But it has also been a challenge for companies and blue teams to publish contact informations in a consistent way.

**Detailled:**

* **Security researchers side:** they want to find a way to report their findings. Until recently, it is only possible in a manual and unconsistant way: 
  * check if any contact are available on the website, 
  * use one of the email addresses commonly available inside companies (abuse, admin etc) but without having any certainty that their report will be .
* **Blue teams side:** inside companies, choosing the good way to setup a proper reporting channel and overcoming all sorts of resistance (cultural, organization, technical) are not without challenges. 

# Challenges encountered at Assurance Maladie

Like many big companies, we provide a lot of Internet websites and webservices to our different types of users (citizens, health professionals and employers in our case).

These services are:
* hosted/operated in different locations by different providers (on premise, SaaS, cloud etc)
* powered by very various software stacks
* **important:** managed by a lot of different projects management structures.

**Faced challenges:**
* **service coverage:** how to display the vulnerability reporting instructions on (most of) all the services?
* **up to date instructions:** how to be sure to display an up to date version of these vulnerability reporting instructions on (most of) all services?
* **management acceptance of publication on their services:** how to convince projects managers of these services to modify their services contents to display the vulnerability reporting instructions? 
* **consistent display between services:** if we manage to convince them, we have to be sure they display it in a consistent way between services.

# The proposal: `security.txt`

What it is? 
* It is an [Internet Draft](https://datatracker.ietf.org/doc/html/draft-foudil-securitytxt)
* This RFC defines among others things a well-known URI `.well-known/security.txt` (and fallback URI `/security.txt`)
* These URI point to the security.txt file.
* The security.txt file contains several standardized records allowing websites owners to set contact infos, reporting policy, GPG key to use and so on.

<u>Example:</u> we can see on this image the contents of the [Deepl website](https://www.deepl.com/)  security.txt file. Its content is displayed through the Firefox extension security.txt :

![security.txt example](/images/posts/securitytxt-example.png)

All the details of the security.txt initiative (URI, records etc) are documented on [https://securitytxt.org/](https://securitytxt.org/).

# Assurance Maladie implementation of security.txt

## The naive way
A simple yet naive way of deploy a security.txt file is to ask all services managers to get the file and incorporate into their service file system.

![security.txt N copies](/images/posts/n-copies-of-security-txt.png)

Your security.txt TODO will fastly become like:
* **Task 1: try to convince & hope for exhaustivity.** The Security team will have to fight to convince a majority of the services managers.
* **Task 2: wait & depend.** The Security team waits until the file is deployed. Each service will deploy at its own rythm and with its own agenda. 
* **Task 3: ask for update & pray.** You will have to redo the same entire job at each update of the file content!

As you can see, one of the Security team darkest nightmares (aka _"Security depending from projects approval"_) is not far from us!

## A (little) clever way

![security.txt N copies](/images/posts/centralized-security-txt.png)

As a lot of our important services are behind load balancers/reverse proxies, we decided that using these 
