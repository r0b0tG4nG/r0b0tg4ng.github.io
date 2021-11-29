---
title: Elmariachi-PC Writeup PwnTillDawn
author: r0b0tG4nG 10:25:00AM 0000
date: 2021-11-29 
categories: [Blogging, PwnTillDawn]
tags: [writeups, exploit-db, ThinVnc, rdp, powershell, nishang]
math: true
mermaid: true
---

![image](https://user-images.githubusercontent.com/67085453/143863482-9f43d331-efb8-49b3-b189-7d6e408e3229.png)<br><br>

**> Information Gathering**<br>
Fire nmap as usual, and from the scan we can see port 445 (smb), 3389 (rdp) & 60000 (ThinVnc). The first two ports are common on windows environments but the other `port 60000` is something one hardly come across on default windows environments.<br>
![image](https://user-images.githubusercontent.com/67085453/143863906-c12b92ff-b008-45ef-a742-9dcad8b7cb3b.png)<br>

When we laubched our browser and navigate to the high port `http://10.150.150.69:60000/`, an `http-login` prompt appeared on the browser. On the login form, there is message which is found right above the `username` field. The message says `The site says: "ThinVnc"`.<br>
![image](https://user-images.githubusercontent.com/67085453/143863559-8299cb2a-fbb4-4eba-870f-2c5b8f4d94e5.png)<br>

Whenever i see something strange, the fiest thing i do perform a quick google search. searching for default credentials on `ThinVnc` lead to the discovery of an `authentication bypass` vulnerability which affects this application.<br>
![image](https://user-images.githubusercontent.com/67085453/143863586-872d8183-5fd5-4f68-8ff3-03644f4d8d1a.png)<br>

Reading the blog from <a href="https://redteamzone.com/ThinVNC/">Red Team Zone</a> shows how we can manually grab credentials and bypass the http login form<br>
![image](https://user-images.githubusercontent.com/67085453/143863617-f40b90bd-4533-42a8-99ae-f5080bb57c82.png)<br>

To exploit this, we need to capture a request to the index page in burpsuite. Once the request is captured, we will forward it to `repeater` in burpsuite. <br>
![image](https://user-images.githubusercontent.com/67085453/143863643-f17d6ed2-5666-4da7-9677-f4fd511aa88e.png)<br>

From the blog we read at <a href="https://redteamzone.com/ThinVNC/">Red Team Zone</a>, it explains well that we need to send a request to `/admin/../../ThinVnc.ini`. ThinVnc stores login credentials in the `ThinVnc.ini` file and if we are able to reach this file, we will grab login credentials. After seninding the request, we found valid credentials `User=desperado & Password=TooComplicatedToGuessMeAhahahahahahahh`.<br>
![image](https://user-images.githubusercontent.com/67085453/143863662-8430271a-db2f-468c-9895-c8db6651a46e.png)<br>

We have a valid credentials, what can we do next??. Lets logon to the `ThinVnc` application with `User=desperado & Password=TooComplicatedToGuessMeAhahahahahahahh`<br>
![image](https://user-images.githubusercontent.com/67085453/143863684-1cd7f483-0a9b-43ec-b479-8683b1e8e83f.png)<br>

Credentials worked and we have acces sto the `ThinVnc` dashboard. unfortunately there isn't much we can do there. Looks like we hit a road block.<br>
![image](https://user-images.githubusercontent.com/67085453/143863697-0d6fc358-b679-48b1-b179-5e6c7f54e770.png)<br>

Going back to nmap, remember port `3389` (rdp) was open. What's rdpp?? `Remote Desktop Protocol or RDP software provides access to a desktop or application hosted on a remote host. It allows you to connect, access, and control data and resources on a remote host as if you were doing it locally.` Why dont we try to connect to rdp using the credentials we found.<br>

To connect to rdp, i typed the command `xfreerdp /u:desperado /v:10.150.150.69 /p:TooComplicatedToGuessMeAhahahahahahahh  +clipboard /dynamic-resolution`.<br>
![image](https://user-images.githubusercontent.com/67085453/143863719-03d27681-d071-4f3a-9292-83cba9d4e85b.png)<br>

**> Post Exploitation**<br>

This target does not need post exploitation but to challenge yourself, why dont you try to spawn a reverse shell on the target and find your way up to `NT AUTHORITY\SYSTEM`. Good luck researcher...:)<br>

Referrence: <a href="https://online.pwntilldawn.com/">PwnTillDawn Online battlefield</a>
