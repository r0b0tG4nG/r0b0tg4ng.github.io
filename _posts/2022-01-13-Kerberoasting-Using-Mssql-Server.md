---
title: Kerberoasting Using Mssql Server [ Abusing GenericAll ] 
author: r0b0tG4nG
date: 2022-01-13 13:33:00 +0000
categories: [Blogging, Exploiting]
tags: [writeups, mssql, impacket-mssqlclient, hashcat, xp_cmdshell, powerview, evil-winrm, kerberoast, bloodhound, nmap]
math: true
mermaid: true
---

![image](https://user-images.githubusercontent.com/67085453/149357946-9fd46ceb-e42d-4d76-9b0d-bc2a474ec637.png)
<br><br>

**> Information Gathering**<br>

From The previous writeup on <a href="https://r0b0tg4ng.github.io/posts/Abusing-MSSQL-Linked-Servers/">Abusing MSSQL Linked Servers</a>, we found credentials on the target. In this chapter, we will look at how we can perform kerberoast attacks through `MSSQL Server`. During our previous engagement, we found credentials using <a href="https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-execute-external-script-transact-sql?view=sql-server-ver15">sp_execute_external_script</a>.<br>
```web.config
        <authentication mode="Forms">
            <forms name="login" loginUrl="/admin">
                <credentials passwordFormat = "Clear">
                    <user 
                        name="Administrator" 
                        password="EverybodyWantsToWorkAtP.O.O."
                    />
                </credentials>
            </forms>
        </authentication>
```
<br>

The question is what can we do with this credential?? Well, we are not out of options. I checked the ipaddress of the target `MSSQL Server` is running on using `sp_execute_external_script` command `EXEC sp_execute_external_script @language = N'Python', @script = N'import os;os.system("ipconfig");'; `. Looking at `IPV4` IP address, there was no service that we could connect to remotely with the credentials we have. So, i decided to dive deep into the `IPV6` address.<br>
![image](https://user-images.githubusercontent.com/67085453/149357971-62ddb9d1-f1a5-467f-93ac-af21677c3a47.png)<br>

A quick nmap scan was performed on the `IPV6` IP address. Port 5985 (Windows Remote Management) was open. Good news at last all we need to do is add the `IPV6` IP address to `/etc/hosts` file `dead:beef::1001  targetipv6`. Henceforth, when i type `targetipv6` i am referring to `dead:beef::1001` address.<br>
![image](https://user-images.githubusercontent.com/67085453/149357995-64d905cf-e354-4354-99ad-3a9775f413bd.png)<br>

I connected to `WinRM` using <a href="https://github.com/Hackplayers/evil-winrm">evilwinrm</a> tool. The command executed to connect to `WinRM` was `evil-winrm -i targetipv6 -u Administrator -p 'EverybodyWantsToWorkAtP.O.O.'`<br>
![image](https://user-images.githubusercontent.com/67085453/149358092-a9615c4f-e560-451f-96b4-ae37e846a737.png)<br>

Digging around using <a href="https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS">winpeas</a> and some few post enumeration tools didn’t yield any positive results. So, i decided to enumerate the domain to find privilege escalation paths but i realized we can't query the domain from a
local administrator account from the errors i encountered.<br>
![image](https://user-images.githubusercontent.com/67085453/149358253-3b39b099-9097-4658-8f09-e777400dfa3c.png)<br>

However, the `SQL service account` can be use instead since service accounts
automatically impersonate the computer account, which are members of the domain and effectively a special type of user account. I logged into `MSSQL` using `impacket-mssqlclient` with the `SA` User we added in the <a href="https://r0b0tg4ng.github.io/posts/Abusing-MSSQL-Linked-Servers/">previous post</a> and enabled `xp_cmdshell` to query domain users with command `xp_cmdshell net users /domain`. <br>
![image](https://user-images.githubusercontent.com/67085453/149358283-48722ed2-2d42-4c46-9b73-80e9c1d1678f.png)<br>

`BloodHound` is an application developed with one purpose: to find relationships within an `Active Directory (AD) domain` to discover attack paths. With this said, we will grab `SharpHound.exe` from <a href="https://github.com/BloodHoundAD/BloodHound/tree/master/Collectors">BloodHoundAD GitHub page</a>. Create the `C:\temp\` directory first then upload the `SharpHound.exe` to the target using our evil-winrm session.<br>
![image](https://user-images.githubusercontent.com/67085453/149358328-2fe15870-4bbe-43f8-bc1e-9e50ad550337.png)<br>

Then the next step is to collect all information about the `Active Directory (AD) domain` through mssql using the command `xp_cmdshell C:\temp\SharpHound.exe -C All --outputdirectory C:/temp`.<br>
![image](https://user-images.githubusercontent.com/67085453/149358361-d447e6d1-62e4-4e3c-98f3-78da446f5abc.png)<br>

Once information about the target `Active Directory (AD) domain` is collected and saved into a zip file, we then download the zip file using evil-winrm download command.<br>
![image](https://user-images.githubusercontent.com/67085453/149358379-23531855-f919-42be-abd0-7901ccaa0c76.png)<br>

Install `BloodHound GUI` and upload the zip file you downloaded. *Note: you can install bloodhound GUI from* <a href="https://bloodhound.readthedocs.io/en/latest/data-analysis/bloodhound-gui.html">BloodHound Docs</a>.<br>
![image](https://user-images.githubusercontent.com/67085453/149358407-1dbcbc96-34e6-4b1f-9135-f80f70e22561.png)<br>

After uploading the zip file to bloodhound UI for visualization. We can use pre-built queries to find any quick `privilege escalation vectors`. Clicking on the `Shortest Paths to Domain Admins from Kerberoastable Users` entry shows the following.<br>
![image](https://user-images.githubusercontent.com/67085453/149358427-1558189b-5f46-4e29-adc6-79af19030266.png)<br>

The user `p00_adm` is a member of `help desk` and has `GenericAll privileges` on the `Domain Admins group`. This means that we can add any user to `Domain Admins` if we have `p00_adm credentials`. To accomplish this, we need to kerberoast the `p00_adm` user and try to crack his hash. We can use the `Invoke-Kerberoast.ps1` from <a href="https://github.com/EmpireProject/Empire/blob/master/data/module_source/credentials/Invoke-Kerberoast.ps1">EmpireProject GitHub page</a>.<br>

**> What is Kerberoasting??:** Kerberoasting is a post-exploitation attack that extracts service account credential hashes from Active Directory for offline cracking. It is used to crack a Kerberos (encrypted password) hash using brute force techniques. If successful, it can crack NTLM hashes in a few hours and provide the adversary with a clear-text password<br>

Download the `Invoke-Kerberoast.ps1` script, upload it to `C:\temp\` directory using `Evil-winrm` session like we did to the `SharpHHound.exe` binary. Once we have the `.ps1` uploaded, we can invoke our `kerberoast attack` through `MSSQL` using the command `xp_cmdshell powershell.exe -c "Import-Module C:\temp\Invoke-Kerberoast.ps1; Invoke-Kerberoast -OutputFormat hashcat"`<br>
![image](https://user-images.githubusercontent.com/67085453/149358471-724a69a4-5f19-4db5-be2e-87cdd9ea9f9b.png)<br>
![image](https://user-images.githubusercontent.com/67085453/149358487-e840b805-3f39-42e8-868b-9830dc3fd182.png)<br>

We found two hashes `p00_hr` & `p00_adm`. Keep in mind that we are after `p00_adm` user hash therefore i copied the `p00_adm` user hash to a file and made sure that i deleted the spaces and empty lines from the hash. `p00_adm`. The hash can be cracked using Hashcat command `hashcat -m 13100 hashes /usr/share/seclists/Passwords/Keyboard-Combinations.txt --force`. Lucky us! we were able to crack the password. *Note: this took a very long time since i had to try a variety of wordlist until i got the  password* The password for `p00_adm` is `ZQ!5t4r`.<br>
![image](https://user-images.githubusercontent.com/67085453/149358516-37aeb491-2817-4814-b520-3069a62812aa.png)<br>

What else now?? well we have `p00_adm` user plaintext password and we can `impersonate` the `p00_adm` with the help of `PowerView`. How do we achieve this? first let's grab `PowerView.ps1` from <a href="https://github.com/PowerShellEmpire/PowerTools/tree/master/PowerView">PowerShellEmpire GitHub page</a> and upload it to the target using evil-winrm. Once The `PowerView.ps1` script is uploaded, the next thing to do is to import the file so we will have access to `PowerView` extended commands like `Add-DomainGroupMember`, `Get-DomainComputer`.....etc.<br>

All done?? what's next?? We need to use a credential object to impersonate `p00_adm` user. Execute this command<br>
```shell
$user='p00_adm'
$pass='ZQ!5t4r'
$p= ConvertTo-SecureString -AsPlainText $pass -force
$cred=New-Object System.Management.Automation.PSCredential -ArgumentList $user,$p
```

![image](https://user-images.githubusercontent.com/67085453/149358621-d2409d18-8cbb-497a-86ed-434add3a3487.png)<br>

After impersonating `p00_adm` user, i still had issues logging in so i searched for `Domain Admins` on the target using the command `get-netuser -DomainController dc -Credential $cred`. From this, i found out the user `mr3ks` is part of the `Domain Admins Group`.<br>
![image](https://user-images.githubusercontent.com/67085453/149358652-d8e5ee90-c4c7-4715-a148-6354cc374103.png)<br>

**> Abusing GenericALL**<br>

Looking back at the `BloodHound GUI`, to abuse `GenericAll`, i right-clicked on the `GenericAll` line, selected `help` and then `Abuse`. This Feature of `BloodHound GUI` explains how we can abuse this feature.<br>
![image](https://user-images.githubusercontent.com/67085453/149358680-5e81be5e-41ea-4e0e-8124-93c2e4e41b20.png)<br>

What can we do now? well, we can change the credentials for `mr3ks` user. The question is how do we do that?? remember i just said `BloodHound GUI` explains everything. I executed this command<br>
```shell
$Username = 'p00_adm'
$Password = 'ZQ!5t4r'
$Connect = ConvertTo-SecureString -AsPlainText $Password -Force
$Auth = New-Object System.Management.Automation.PSCredential -ArgumentList $Username,$Connect
Set-DomainUserPassword -Identity mr3ks -Password $Connect -Credential $Auth
```

![image](https://user-images.githubusercontent.com/67085453/149358712-4a44c04c-4ece-40de-8acf-b1a96dd91615.png)<br>

We encountered no errors that's good news to the eyes. Let's try to connect to the target through the target `IPV6` `Win-RM` service.  To achieve this, i executed `evil-winrm -i targetipv6 -u mr3ks -p 'ZQ!5t4r'`. Hehehe we are logged in and we have fully compromised a `Domain Admin` account.<br>
![image](https://user-images.githubusercontent.com/67085453/149358731-8f998768-a5e4-4d5a-982b-bc62e01cb4ef.png)<br>


I hope you enjoyed the ride and also discovered something new. Kindly subscribe to my <a href="https://www.youtube.com/channel/UCSY-pfwuYspZFlRsO7vBfIQ"> YouTube channel</a> for more contents<br>

