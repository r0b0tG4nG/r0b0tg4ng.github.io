---
title: Stuntman Mike Writeup PwnTillDawn
author: r0b0tG4nG
date: 2021-11-29 13:02:00 
categories: [Blogging, PwnTillDawn]
tags: [writeups, ssh, hydra, sudo]
math: true
mermaid: true
---

![image](https://user-images.githubusercontent.com/67085453/143872562-f60e8306-a5d1-4b8a-a830-f2ca304ac945.png)<br><br>

**> Information Gathering**<br>
Is there a way to attack a system without gathering information? As usual we need to fire nmap on the target to find out what ports are open and what services are running on these ports. There are only two ports 22 (ssh) & 8089 (splunkd)<br>
![image](https://user-images.githubusercontent.com/67085453/143872579-f4cf8d1e-bd2d-4da6-8875-aff02d034c99.png)<br>

On port `8089` (splunkd), we found nothing juicy. Tried a couple of stuffs but none worked. Well, it's confusing when the usual attack surface doesn't produce any results.<br>
![image](https://user-images.githubusercontent.com/67085453/143872595-f683fda1-c088-45df-ac77-12e0d7e992ea.png)<br>

There is something unusual about this box. In normal circumstances, attackers hardly check port 22 unless they have credentials, but on this box, since we are out of options on port `8089` let's try a test login through ssh. `ssh test@10.150.150.166`.<br>
![image](https://user-images.githubusercontent.com/67085453/143872613-bc780fcc-289c-4b5a-a14c-309a1b593110.png)<br>

Wait! did you see that?? in the banner ssh connection showed there is a hint there. It's written in the banner that `You are attempting to login to stuntman mike's server `. A username is leaked in there. The username is either `stuntman` or `mike`. What can we do with this information? The best option is to call on the `3 headed dragon` in short `hydra`.<br>

What is hydra?? `Hydra is a parallelized network login cracker` in short, hydra is a tool for brute forcing networking protocol with a wordlist. It's obvious hydra is the best tool to use in this situation. To start hydra, type `hydra -l mike -P /usr/share/wordlists/rockyou.txt 10.150.150.166 ssh`. We waited for a few minutes and hydra found `Mike's` password `username: mike` & `password:babygirl`.<br>
![image](https://user-images.githubusercontent.com/67085453/143872635-62649da4-96cd-42c7-9d6c-548c9b82f003.png)<br>

We can connect to the ssh service using this credential. `ssh mike@10.150.150.166`, enter the password `babygirl` to get access. You can also login with a tool call `sshpass`.<br>
![image](https://user-images.githubusercontent.com/67085453/143872677-c4edbdef-add4-49f6-bfd3-ca029e06771a.png)<br> 

**> Post Exploitation**<br>

On linux environments, whenever i have a shell, there are 2 main things i check. First is the `id` command. This command will tell you who you are on the system and what groups you are part of. The next command is `sudo -l`, What this command does is to check what permissions / rights the system admin (root) has assigned to you in the `/etc/sudoers` file. typing `sudo -l` revealed that mike has the `(ALL : ALL) ALL` permissions.<br>
![image](https://user-images.githubusercontent.com/67085453/143872705-3a6b7a62-5c2c-44ad-88d1-fc4510532e88.png)<br>

Since Mike user has `(ALL : ALL) ALL` permissions, the easiest way to elevate privileges on this target is to type `sudo su root`. Simple and easy we are root user on the target. <br>
![image](https://user-images.githubusercontent.com/67085453/143872727-d2523b02-94ab-403a-8cfc-64c4952abcb7.png)<br>

Reference: <a href="https://online.pwntilldawn.com/">PwnTillDawn Online battlefield</a>
