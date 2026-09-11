1. Overview
Include Me is a web exploitation lab from HackerDNA focused on identifying and exploiting a Local File Inclusion (LFI) vulnerability in a PHP application.
The application uses a page HTTP GET parameter to determine which file should be included by the server. Because this parameter is passed directly to PHP's include() function without validation or access controls, an attacker can manipulate it to read arbitrary local files.
The lab demonstrates several important web security concepts:
Web reconnaissance
HTTP parameter analysis
PHP include() behavior
Local File Inclusion (LFI)
Directory Traversal
PHP stream wrappers
php://filter
Base64 encoding
Source-code disclosure
Sensitive file enumeration

2. Objective
The objective of the lab was to identify the vulnerability in the web application and exploit it to obtain the flag.
The main goals were:
Enumerate the target.
Identify suspicious parameters in the web application.
Determine how the page parameter was processed.
Confirm the existence of an LFI vulnerability.
Read local files from the target system.
Retrieve the application's source code.
Locate and read the flag.txt file.

3. Environment
Platform: HackerDNA
Lab: Include Me
Target: http://<TARGET-IP>/
Attacker machine: Linux
Web Server: Apache HTTP Server 2.4.57
Target OS: Debian Linux
Main technologies: PHP
Tools: Nmap, Gobuster, cURL/browser, Base64 utilities

4. Reconnaissance
The first step was to identify the services exposed by the target.
Nmap
The initial scan was performed with:
```bash
nmap -sC -sV -Pn <TARGET-IP>
```

The scan identified the following relevant services:
```text
53/tcp   open
80/tcp   open
```

Port 80 was running:
```text
Apache httpd 2.4.57 (Debian)
```

The target was identified as a Linux/Debian system.
The HTTP service was therefore the main focus of the investigation.
Initial Web Application
Accessing the web application revealed the following URL structure:
```http
http://<TARGET-IP>/index.php?page=about.html
```

The parameter:
```text
page=about.html
```

was immediately interesting because it appeared to determine which page the application loaded.
This led to the initial hypothesis that the application could be using the page parameter in a PHP file inclusion operation.

5. Enumeration
Web Directory Enumeration
Gobuster was used to enumerate common web resources:
```bash
gobuster dir -u http://<TARGET-IP> \
-w /usr/share/wordlists/dirb/common.txt
```

The relevant results were:
```text
/index.php       (Status: 302)
/server-status   (Status: 403)
```

The redirect from index.php pointed to:
```text
index.php?page=about.html
```

No additional useful directories were discovered during the initial enumeration.
Source Code Inspection
The page source did not initially reveal useful information.
Because the page parameter was suspicious, the next step was to manually manipulate it.
Testing the page Parameter
A nonexistent value was supplied:
```text
?page=test
```

The application returned a PHP warning similar to:
Warning: include(/usr/local/lib/php/test): Failed to open stream:
No such file or directory

and:
```text
Warning: include(): Failed opening 'test' for inclusion
(include_path='.:/usr/local/lib/php')
```

This was an important discovery.
The error revealed that the application was using PHP's:
include()

function.
It also showed that the value supplied through page was being used as the file to include.

6. Vulnerability Identification
Local File Inclusion
The vulnerable functionality was identified in the page parameter.
The application effectively performed:
```php
include($_GET["page"]);
```

without validating or restricting the supplied filename.
This creates a Local File Inclusion (LFI) vulnerability.
The vulnerability exists because attacker-controlled input is directly passed to a file inclusion function.
A secure application should never allow arbitrary user input to determine which local file is included.
Confirming the LFI
To verify whether arbitrary local files could be accessed, a Directory Traversal payload was used:
```text
?page=../../../../etc/passwd
```

The application returned the contents of:
```text
/etc/passwd
```

The response contained entries such as:
```text
root:x:0:0:root:/root:/bin/bash
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
```

This confirmed that arbitrary local files could be included.
At this point the LFI vulnerability was definitively confirmed.

7. Exploitation
Step 1 — Confirming File Inclusion
The first test was:
?page=test

The resulting PHP warning confirmed that the application was attempting to include the supplied value.
This established the initial vulnerability hypothesis.

Step 2 — Reading /etc/passwd
The next test used Directory Traversal:
```text
?page=../../../../etc/passwd
```

The ../ sequences instruct the filesystem to move up one directory.
For example:
/var/www/html
      ↓ ../
/var/www
      ↓ ../
/var
      ↓ ../
/

This allowed the request to reach:
/etc/passwd

The contents of the file were successfully returned.
This demonstrated that the application could be abused to read local files.

Step 3 — Attempting to Read index.php
Because the vulnerable application was located at:
/var/www/html/index.php

the next logical step was attempting:
?page=index.php

However, this produced a memory exhaustion error:
Fatal error: Allowed memory size exhausted

The reason was recursive inclusion.
The application attempted:
```text
index.php
   ↓
include(index.php)
   ↓
include(index.php)
   ↓
include(index.php)
   ↓
…
```

Eventually the PHP process exhausted its allocated memory.
Therefore, directly including the PHP source file was not an effective way to read its contents.

Step 4 — Using php://filter
PHP provides stream wrappers that can alter how files are accessed.
The php://filter wrapper can apply a filter to a file before it is returned.
The following filter was used:
convert.base64-encode

The resulting request was:
```text
http://<TARGET-IP>/index.php?page=php://filter/convert.base64-encode/resource=index.php
```

The purpose was to make PHP read the source code and encode it as Base64 before the include() processed it.
The server returned a Base64 string:
```text
PD9waHAKCiAgICBpZiAoIWlzc2V0KCRfR0VUWyJwYWdlIl0p…
```

The Base64 data was then decoded locally:
```bash
echo '<BASE64_DATA>' | base64 -d
```

This revealed the actual source code of the vulnerable application.

Step 5 — Analyzing the Source Code
The decoded source code was:
```php
<?php

if (!isset($_GET["page"])) {
    header("Location: index.php?page=about.html");
    die();
} else {
    include($_GET["page"]);
}
```

The vulnerable line was:
include($_GET["page"]);

There was no:
input validation;
whitelist;
filename restriction;
path validation;
sanitization.
This confirmed the root cause of the LFI.

Step 6 — Reading about.html
The application originally loaded:
about.html

The file was also read through the LFI using the same php://filter technique.
The decoded file contained only the application's static "About Us" page and did not reveal additional useful information.
This ruled out about.html as the source of the flag.

Step 7 — Enumerating Server Configuration
Because the application's DocumentRoot was already known as:
/var/www/html

Apache configuration files were examined through the LFI.
The relevant configuration confirmed:
DocumentRoot /var/www/html

It also showed the Apache log locations:
ErrorLog ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined

This confirmed the web application's filesystem location and provided additional information about the server configuration.

Step 8 — Locating the Flag
After confirming the LFI and understanding the application's filesystem location, the next step was to look for the challenge's flag file.
The flag was located in:
flag.txt

It was accessed through the vulnerable page parameter using the appropriate filesystem path.
The server returned the contents of the file, confirming successful exploitation of the vulnerability and completion of the lab.

8. Proof of Compromise
The objective was successfully achieved by retrieving the challenge's:
flag.txt

through the vulnerable page parameter.
The successful retrieval of the flag demonstrates that arbitrary local file inclusion was possible and that the attacker could access files outside the intended web application content.
The actual flag value is intentionally omitted from this write-up.

9. Attack Chain
The complete attack chain was:
Reconnaissance
      ↓
Nmap identifies Apache/PHP web service
      ↓
Web Enumeration with Gobuster
      ↓
Discovery of index.php?page=about.html
      ↓
Parameter Manipulation
      ↓
PHP include() error discovered
      ↓
Directory Traversal
      ↓
/etc/passwd successfully read
      ↓
Local File Inclusion confirmed
      ↓
php://filter discovered
      ↓
index.php source code disclosed
      ↓
Vulnerable include($_GET["page"]) confirmed
      ↓
Filesystem enumeration
      ↓
flag.txt located
      ↓
Flag retrieved


10. Root Cause
The root cause was the use of attacker-controlled input directly inside PHP's include() function:
```php
include($_GET["page"]);
```

The application trusted the value supplied by the user and did not restrict which files could be included.

This allowed an attacker to manipulate the parameter with values such as:
../../../../etc/passwd

and access files outside the intended application directory.
The application also exposed detailed PHP error messages, which disclosed:
filesystem paths;
PHP configuration information;
the location of the vulnerable source code;
the use of the include() function.
This information significantly simplified exploitation.

11. Mitigation
The primary mitigation is to never pass unrestricted user input directly to include() or require().
Instead of:
```php
include($_GET["page"]);
```

the application should use a strict allowlist.
For example:
```php
$pages = [
    "about" => "about.html",
    "contact" => "contact.html"
];

if (isset($pages[$_GET["page"]])) {
    include($pages[$_GET["page"]]);
} else {
    http_response_code(404);
}
```

This ensures that the user can only select files explicitly authorized by the application.
Additional security measures include:
Disable detailed PHP errors in production.
Avoid exposing filesystem paths in HTTP responses.
Apply proper input validation.
Use a strict allowlist for dynamic page selection.
Run the web server with the minimum required filesystem permissions.
Prevent access to sensitive files from the web application user.
Review PHP configuration and filesystem permissions.
Monitor application logs for Directory Traversal attempts.

12. Lessons Learned
This laboratory demonstrated how a seemingly simple URL parameter can lead to significant security consequences.
The main lessons learned were:
1. Parameters deserve attention
The parameter:
?page=about.html

looked simple, but it controlled a sensitive server-side operation.
Parameters that determine files, paths, commands, templates, or database queries should always be treated as potentially dangerous.
2. Error messages are valuable during enumeration
The PHP warning immediately revealed:
include()

and exposed filesystem information.
Detailed errors can provide attackers with valuable information about the application's internal implementation.
3. LFI can expose sensitive files
The ability to read:
/etc/passwd

demonstrated that the vulnerability was not limited to the application's intended files.
4. php://filter can be used to analyze PHP source code
Directly including a PHP file causes PHP to execute it.
Using:
```php
php://filter/convert.base64-encode/
```

allowed the source to be retrieved in encoded form instead, making it possible to inspect the vulnerable code.
5. Directory Traversal and LFI can work together
The ../ sequences allowed navigation outside the intended application directory.
Combined with the vulnerable include(), this enabled arbitrary local file access.
6. Source-code disclosure makes exploitation easier
Once the source code was obtained, the exact vulnerability became obvious:
```php
include($_GET["page"]);
```

This eliminated the need to guess how the application processed the parameter.
7. Methodology is more important than memorizing payloads
The most important part of the laboratory was not the final payload.
The exploitation followed a logical process:
Observe
  ↓
Hypothesize
  ↓
Test
  ↓
Analyze response
  ↓
Confirm vulnerability
  ↓
Enumerate
  ↓
Exploit

This methodology can be applied to many other web application security challenges.
