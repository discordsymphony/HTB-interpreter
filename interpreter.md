<img src="Images/01-Banner.png" width="600">

**Difficulty:** Medium

**OS:** Linux

**IP:** 10.129.244.184

**Date:** 21/02/2026

We will begin this machine by using Nmap to scan the server for all open TCP ports:

<img src="Images/02-Nmap.png" width="600">

Visiting the webserver on port 443, we find a Mirth Connect instance.  Clicking the Launch Mirth Connect Administrator initiates a download of the file webstart.jnlp.

<img src="Images/03-Landing-Page.png" width="600">

Viewing webstart.jnlp using vim, reveals that we are working with Mirth Connect Administrator 4.4.0:
 
<img src="Images/04-Webstart.png" width="600">

Searching for exploits related to this version of Mirth Connect returns the following exploit:

<img src="Images/05-Google-Search.png" width="600">

Preparing and executing the payload provides us with a shell on the server:

<img src="Images/06-Payload-Prep.png" width="600">

<img src="Images/07-Listener-Shell.png" width="600">


We then use the following commands to stabilize the shell:

python3 -c ‘import pty;pty.spawn(“/bin/bash”)’

export TERM=xterm

ctrl-z

stty raw -echo; fg

At this point we want to identify any related files of interest for the underlying Mirth Connect technology. After some research we discover the following file which contains database credentials:

<img src="Images/08-Database-Crds.png" width="600">

Using ss, we discover that there is a running SQL server on port 3306:

<img src="Images/09-Netstat.png" width="600">

With this information, we connect to the SQL server using MariaDB:

<img src="Images/10-Database-Connect.png" width="600">

Next, we enumerate the database and discover an encrypted password:

<img src="Images/11-Databases.png" width="400">

[...]

<img src="Images/12-Tables.png" width="400">

<img src="Images/13-Database-Password.png" width="600">

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
