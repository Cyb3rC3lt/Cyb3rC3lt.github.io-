---
layout: post
title: "RBCD on Local Machine via Socks Proxy"
permalink: "cme"
date: 02-09-2023
categories: Blog
---

I recently got inspired by a neat trick by @flodari on Twitter where he managed to carry out a RBCD LPE on a Windows machine with just some regular domain creds.

There is nothing new in these techniques but I hadn't seen this idea before so I decided to see if it could also be done via a C2. Relaying authentications is also my favourite thing to do! 

### Step 1

1st part of the attack is to open a socks proxy on port 1080 using CStrike on the compromised workstation (Elish) to allow us to tunnel our Kali tools through it.
Then reverse port fwd 8888 to 80 on our localhost. This will catch any auth on 8888 & pass it to ntlmrelayx on 80

<img src="/assets/img/CSProxy.jpg" width="200"/>

### Step 2

Next up is to add a machine account (evilpc) via CME to allow us to use it for delegation. You could also just let ntlmrelayx do this automatically but I get to showcase the add-computer module I ported to CME from impacket😊

<img src="/assets/img/CMEAdd.jpg" width="200"/>


### Step 3

We then trigger authentication with PetitPotam over proxychains but the key is to specify 127.0.0.1 so PetitPotam acts on the Elish$ host which then calls out to port 8888 on itself which we have port forwarded in Step 1 back to port 80 which is our listening ntlmrelayx.

<img src="/assets/img/PetitPotam.jpg" width="200"/>

### Step 4

In the last step Ntlmrelayx then gets a hit from Elish$ as shown and adds the delegation rights to our evilpc$ allowing use control over the Elish$ machine we have our beacon on. Workstation takeover of Elish$ can then be achieved using the normal http://getST.py approach

<img src="/assets/img/Result.jpg" width="200"/>


### Final Thoughts

I thought this was a nice little way to easily get control of a machine without too much of an effort assuming you have credentials. Hope this helps others too.
