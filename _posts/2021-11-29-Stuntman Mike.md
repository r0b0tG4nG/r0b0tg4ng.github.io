---
title: Stuntman Mike Writeup PwnTillDawn
author: r0b0tG4nG
date: 2021-11-29 12:25:00 
categories: [Blogging, PwnTillDawn]
tags: [writeups, exploit-db, ThinVnc, rdp, powershell, nishang]
math: true
mermaid: true
---

(image)<br><br>

**> Information Gathering**<br>
Is there a way to attack a system without gathering information? As usual we nee dto fire nmap on the target to find out what ports are open and what services are running on these ports. There are only two ports 22 (ssh) & 8089 (splunkd)<br>
(image nmap)<br>

On port `8089` (splunkd), we found nothing juicy. Tried a couple of stufs but none worked. Well it's confusing when the usual attack surface doesn't produce any results.<br>
(web image)<br>

There is something unusual about this box. Innormal circumstances, attackers hardly check port 22 unless they have credentials, but on this box, since we are out of options on port `8089` let's try a test login through ssh. `ssh test@10.150.150.166`.<br>
(ssh test)<br>

Wait! did you see that?? in the banner ssh connection showed there is a hint there. It's written in the banner that `You are attempting to login to stuntman mike's server `. A username is leaked in there. The username is either `stuntman` or `mike`. What can we do with this infromation? The best option is to call on the `3 headed dragon` in short `hydra`.<br>

What is hydra?? `Hydra is a parallelized network login cracker` in short, hydra is a tool for bruteforcing networking protocal with a wordlist. It's obvious hydra is the best tool to use in this situation. To start hydra, type `hydra -l mike -P /usr/share/wordlists/rockyou.txt 10.150.150.166 ssh`. We waited for a few minutes and hydra found `Mike's` passsword `username: mike` & `password:babygirl`.<br>
(image hrydra)<br>

We can connect to the ssh service using this credential. `ssh mike@10.150.150.166`, enter the password to get access. You can also login with a tool call `sshpass`.<br>
(image ssh)<br> 

**> Post Exploitation**<br>

On linux environments, whenever i have a shell, there are 2 main things i check. First is the `id` command. This command will tell you who you are on the system and what groups you are part of. The next command is `sudo -l`, What this command does is to chec what permissions / rights the system admin (root) has assigned to you in the `/etc/sudoers` file. typing `sudo -l` revealed that mike has the `(ALL : ALL) ALL` permissions.<br>
(sudo -l )<br>

Since Mike user has `(ALL : ALL) ALL` permissions, the easiest way to elevelate privileges on this target is to type `sudo su root`. Simple and easy we are root user on the target. <br>
(root img)<br>

Referrence: <a href="https://online.pwntilldawn.com/">PwnTillDawn Online battlefield</a>