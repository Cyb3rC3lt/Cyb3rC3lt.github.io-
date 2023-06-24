---
layout: post
title: "Azure Scoping Overview"
permalink: "azurescoping"
date: 24-06-2023
categories: Blog
---

Recently I participated in my first Azure pen test. In the lead up to the test when I was scoping out how we would access Azure and what creds would be required, it became apparent that these weren't as simple questions as they would be for an on-prem test.

To help with this I created this scoping flow chart to assist with the decision making around the test.

<img src="/assets/img/AzureScoping.png"/>

As you can see there are a few decision points.

First one being whether a basic CIS Benchmarks audit of the platform is required versus an actual Pentest.

Then whether there is a Domain involved or not, so things like Kerberos would still exist and it would require domain creds like a more of a traditional internal test.

Then finally whether the internal infrastructure like Virtual Machines are required to be tested or whether it will only be the Azure platform itself.

Some of the options make more sense than others so in my opinion. In the orange branch of the chart I probably wouldnt choose Case E or F option 2 as there isn't much to gain putting your own VMs on the network when VPN or using the clients own VPN deployment mechanisms including using Bastion would probably be a simpler option for everyone.

Likewise in the blue branch of the chart it would be preferable to get a device on the network, given that the use of Kerberos or NTLM is still in place. I find it is easier to trigger NTLM relaying and other trafic manipulation tasks over VM rather than VPN. Sometimes over VPN the VPN itself can restrict the type of traffic that is visible.

Hopefully you find this useful and please get in touch if you spot any issues with it because I'm still just a novice cloud pen tester.