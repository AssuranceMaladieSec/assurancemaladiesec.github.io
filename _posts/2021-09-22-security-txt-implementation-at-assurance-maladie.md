---
layout: post
title:  "security.txt implementation @ Assurance Maladie"
date:   2021-09-22 13:33:19 +0200
author: cbrocas
categories: secops
---

**The problem:** Finding how to report vulnerabilities found on web sites and web services has always been challenging for serurity researchers. But it has also been a challenge for companies and blue teams to publish contact informations in a consistent way.

**The (last) proposal:** `security.txt`

What it is? 
* an [Internet Draft](https://datatracker.ietf.org/doc/html/draft-foudil-securitytxt)
* defining a well-known URI `.well-known/security.txt` (and fallback URI `/security.txt`)
* pointing to a file
* containing several records allowing websites owners to set contact infos, reporting policy, GPG key to use and so on.


