---
layout: post
title:  "security.txt implementation @ Assurance Maladie"
date:   2021-09-22 13:33:19 +0200
author: cbrocas
categories: secops
---
**TL;DR:** Finding how to report vulnerabilities detected on web sites and web services has always been challenging for security researchers. But it has also been a challenge for companies and blue teams to publish contact informations in a consistent way accross their sites and services. Let's see if `security.txt` initiative can help us!

{% include toc %}

# Security vulnerabilities reporting: the problem(s)

* **Security researchers side:** they want to find a way to report their findings. Until recently, it is only possible in a manual and unconsistant way: 
  * check if any contact are available on the website, 
  * use one of the email addresses commonly available inside companies (abuse, admin etc) but without having any assurance that their report will be processed (correctly).
* **Blue teams side:** inside companies, choosing the good way to setup a proper reporting channel and overcoming all sorts of resistance (cultural, organization, technical) are not without challenges. 

**Impacts:**
* For companies, it generates a **loss of vulnerabilities reports** coming from external security researchers.
* For security researchers, **this situation is a hassle** that would gain a lot to be streamlined.

# Challenges encountered at Assurance Maladie

Like many big companies, we provide a lot of websites and webservices to our different types of users (citizens, health professionals and employers in our case).

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

Security.txt is a proposal to standardize the way companies document, on each of their websites, how they want to receive the vulnerability reports and how they will handle them.

Security.txt in detail:
* It is an [Internet Draft](https://datatracker.ietf.org/doc/html/draft-foudil-securitytxt)
* This RFC defines among others things a well-known URI `.well-known/security.txt` (and fallback URI `/security.txt`)
* These URI point to the `security.txt` file.
* The `security.txt` file contains several standardized records allowing companies owners to publish their contact infos, their reporting policy, an eventual GPG key to use and so on.

<u>Example:</u> we can see on this screenshot the contents of the [Deepl website](https://www.deepl.com/)  `security.txt` file. Its content is displayed through the Firefox extension security.txt :

![security.txt example](/images/posts/securitytxt-example.png)

All the details of the `security.txt` initiative (URI, records etc) are documented on [https://securitytxt.org/](https://securitytxt.org/).

# Assurance Maladie implementation of `security.txt`

Let's see how we handle it at Assurance Maladie IT Security team. 

## Initial situation

As many companies, we had not any consistent informations on our web sites and services to notify security vulnerabilities reporters how to report their findings. Even worse, in the past, we tried to display on our main web site our SOC email address in case people has a security problem with our web site and guess what? Our SOC has been overflowed by not security related notifications but only business oriented questions. Rollback has been done in a few hours. The _"Learning by failing mode"_ was ON ;-)

So when emerged the `security.txt` initiative, we looked at it with a very interested attention.

## The naive way
A simple yet naive way to deploy a `security.txt` file is to ask all services managers to get the file and incorporate into their service file system.

![security.txt N copies](/images/posts/n-copies-of-security-txt.png)

Your `security.txt` TODO will fastly become like:
* **Task 1: try to convince & hope for exhaustivity.** The Security team will have to fight to convince a majority of the services managers.
* **Task 2: wait & depend.** The Security team waits until the file is deployed. Each service will deploy at its own rythm and with its own agenda. 
* **Task 3: ask for update & pray.** You will have to redo the same entire job at each update of the file content!

As you can see, one of the Security team darkest nightmares (aka _"Security depending from projects approval"_) is not far from us!

## A (little) clever way

![security.txt N copies](/images/posts/centralized-security-txt.png)

We host some of the biggest public web sites in France (our main service has 40M reular users, millions connections on a daily basis).

So our services are behind load balancers/reverse proxies that **we manage**. Due to this fact, we decided that using these **central** IT components would be interesting.

More precisely, we decided to:
* **intercept** requests asking the 2 URI `/security.txt` and `/.well-known/security.txt` for all services behind the LB/reverse proxies.
* **get the content of the `security.txt` file** from a centralized server (OK, from 2 for HA) and send it back to clients.

A lot of advantages came up with this decision:
* **Zero impact on websites/services**. No deployment required, no update to manage on websites servers etc.
* **Security deployment not depending from services project managers approval**. And _"it is a Good thing(tm)!"_.
* **One file to rule them all**. You manage the vulnerabilities reporting information you provide through a single copy of a file. First deployment and next iterations will be easy and under your fully and own control.

**Bonus point:** 
* **Medium: digital signature.** As we provide a GnuPG public key to reporters, we decided to digitally sign our `security.txt` file with the corresponding private key. Effectively, compromised files are [part of the threat model](https://datatracker.ietf.org/doc/html/draft-foudil-securitytxt#section-6.1) included in the RFC.  
* **Impact: providing a secured way to publish `security.txt` on externally managed sites.** We can then ask our external services providers that operate services for us to deploy our `security.txt` file (example for [www.ameli.fr](https://www.ameli.fr/security.txt)) without worrying about silent file alteration as we are able to check the file integrity at any moment.

## Implementation details

According to our existing reporting workflow, we decided to use the following records inside our `security.txt` file:
* **`Contact:`** through this record, we push the abuse@ email address as main contact point.
* **`Preferred-Languages:`** the languages we speak/understand.
* **`Encryption:`** URL of the GnuPG public key of the abuse@ email address.
* **`Policy:`** URL of our vulnerability reporting policy.
* **`Acknowledgments:`** URL of the page where we thank/acknowledge all the reporters that responsibily disclose vulnerabilities with us.

**To host the ressources** pointed by the last 3 records, **we decided to use Github** because it is an external hosting (can be useful in case of massive compromission locally). 

It is also a sure external hosting as we have decided to **sign all of our commits**, not only for code we published but also for static ressources like reporting policy web page or GnuPG public key file.

## Final Benefits on the reporting side

At time of writing this post (end of September 2021), we are one year and a half after the deployment of `security.txt` file on most of our sites/services which occured during the first quarter of 2020.

And finally, the question is: _"You have done all of this **for what kind of results**, mate?"_

* **Before the deployment:** we have received **very few if any external reports** during many years.

* **One year and a half after the deployment: we have received a dozen of security vulnerabilities reports** through this new announcement channel. Some of them have notably led us to focus on not previously investigated grey areas of our services (sofware stack components, way to expose some services and so on). Another interesting point: we have received no "noise" through this channel. No fake reports, no out of context sollicitations. Thank to all reporters for that!

* **Why it is so effective?** After asking the reporters (who are usually bug bounty hunters or infosec profesionals), it is because `security.txt` is used by them on a daily basis as the main way to reach out Security teams during reporting phase of their findings.

* **Security researchers acknowledgement:** After validation, and fix if necessary, of these vulnerabilities, **we credit the security vulnerabilities reporters** on [our acknowledgement page](https://assurancemaladiesec.github.io/abuse/thanks/).

## Limitations

Of course, this centralized deployment does not provide 100% service coverage as we also have SaaS or external hosted services. But as we do not have a lot of them, we can reach a good global coverage without a not to big effort of `security.txt` deployment among these external providers.

And, even if you want to use `security.txt`, **it is not a silver bullet!**

Before using it, you will have to set up a well defined vulnerability management workflow inside your company. Without it, no magic will happen. 

In our case, putting down things in a publicly available reporting policy help us to clarify the internal reporting channel, roles or how we want to acknowledge reporters for example.

# Conclusion

`Security.txt` intiative is a simple and cost effective solution to improve a cybersecurity challenge online: the vulnerability reporting channel announcement. 

Using _cleverly_ your technical stack to deploy it across most of your services will allow you to avoid the main usual (mainly organizational!) caveats that Security faced inside companies when it comes to apply Security to corporate websites and webservices.

# Resources
* Our security.txt file: [https://assure.ameli.fr/.well-known/security.txt](https://assure.ameli.fr/.well-known/security.txt)
* The security.txt initiative website: [https://securitytxt.org/](https://securitytxt.org/)
* The security.txt RFC: [https://datatracker.ietf.org/doc/html/draft-foudil-securitytxt-12](https://datatracker.ietf.org/doc/html/draft-foudil-securitytxt-12)
* All schemas have been done using [draw.io](https://app.diagrams.net/).