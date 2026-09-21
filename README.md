# HTB-interpreter

<img src="Images/01-Banner.png" width="600">

**Difficulty:** Medium

**OS:** Linux

**IP:** 10.129.244.184

**Date:** 21/02/2026

We began this machine with a standard Nmap scan which led to the discovery of a webserver on port 443. The web server contained the Mirth Connect engine and we identified its version by downloading and analysing webstart.jnlp. A search for related web exploits returned CVE-2023-43208 which we leveraged to obtain a shell on the server as mirth. Further investigation into files related to the underlying Mirth Engine revealed database credentials in the mirth.properties file. By logging into MariaDB, we obtained an encrypted password and identified its algorithm using AI, we then recovered its plaintext value using hashcat, and were able to log into the machine as Sedric using SSH.

As Sedric, we dumped the running root processes on the machine and discovered notif.py was running on localhost. Analysis of this file revealed that it was using a dangerous Python Eval statement. As a result, we forwarded port 54321 and sent a request to test the dangerous function, which proved successful. After some time, we were able to come up with a working exploit that returned a reverse shell to our attacker listener as root and we had completed the machine.


**Key Findings**

| Finding         | Severity     | Impact
------------------------------------------------------------------------------------------------
| CVE-2023-43208 | Critical 9.8 | Remote Code Execution on a remote server running Mirth Connect.


Finding: Python Eval code execution.<br>
Severity: Critical: 9.9<br>
Impact: Remote Code Execution leading to privilege escalation on local machine.<br><br>

