---
title: Elmariachi-PC Writeup PwnTillDawn
author: r0b0tG4nG
date: 2021-11-28 10:25:00 +0000
categories: [Blogging, PwnTillDawn]
tags: [writeups, exploit-db, ThinVnc, rdp, powershell, nishang]
math: true
mermaid: true
---

(image)<br><br>

**> Information Gathering**<br>
Fire nmap as usual, and from the scan we can see port 445 (smb), 3389 (rdp) & 60000 (ThinVnc). The first two ports are common on windows environments but the other `port 60000` is something one hardly come across on default windows environments.<br>
(nmap)<br>

When we laubched our browser and navigate to the high port `http://10.150.150.69:60000/`, an `http-login` prompt appeared on the browser. On the login form, there is message which is found right above the `username` field. The message says `The site says: "ThinVnc"`.<br>
(image)<br>

Whenever i see something strange, the fiest thing i do perform a quick google search. searching for default credentials on `ThinVnc` lead to the discovery of an `authentication bypass` vulnerability which affects this application.<br>
(google)<br>

Reading the blog from <a href="https://redteamzone.com/ThinVNC/">Red Team Zone</a> shows how we can manually grab credentials and bypass the http login form<br>
(image red team)<br>

To exploit this, we need to capture a request to the index page in burpsuite. Once the request is captured, we will forward it to `repeater` in burpsuite. <br>
(image burp)<br>

From the blog we read at <a href="https://redteamzone.com/ThinVNC/">Red Team Zone</a>, it explains well that we need to send a request to `/admin/../../ThinVnc.ini`. ThinVnc stores login credentials in the `ThinVnc.ini` file and if we are able to reach this file, we will grab login credentials. After seninding the request, we found valid credentials `User=desperado & Password=TooComplicatedToGuessMeAhahahahahahahh`.<br>
(image creds)<br>

We have a valid credentials, what can we do next??. Lets logon to the `ThinVnc` application with `User=desperado & Password=TooComplicatedToGuessMeAhahahahahahahh`<br>
(image login)<br>

Credentials worked and we have acces sto the `ThinVnc` dashboard. unfortunately there isn't much we can do there. Looks like we hit a road block.<br>
(image dashboard)<br>

Going back to nmap, remember port `3389` (rdp) was open. What's rdpp?? `Remote Desktop Protocol or RDP software provides access to a desktop or application hosted on a remote host. It allows you to connect, access, and control data and resources on a remote host as if you were doing it locally.` Why dont we try to connect to rdp using the credentials we found.<br>

To connect to rdp, i typed the command `xfreerdp /u:desperado /v:10.150.150.69 /p:TooComplicatedToGuessMeAhahahahahahahh  +clipboard /dynamic-resolution`.<br>
(rdp connect)<br>

**> Post Exploitation**<br>

This target does not need post exploitation but to challenge yourself, why dont you try to spawn a reverse shell on the target and find your way up to `NT AUTHORITY\SYSTEM`. Good luck researcher...:)<br>

Referrence: <a href="https://online.pwntilldawn.com/">PwnTillDawn Online battlefield</a>