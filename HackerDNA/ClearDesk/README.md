# ClearDesk

## 1. Overview

ClearDesk is a web-based IT helpdesk laboratory focused on web application security and access control.

The application provides a session-based authentication mechanism and exposes several API endpoints for tickets and user information. The main security issue is that authentication is implemented without proper authorization checks at the object level.

By enumerating ticket identifiers, it was possible to access tickets belonging to other users. One of the exposed tickets contained administrative credentials, allowing authentication as an administrator.

The administrative panel then exposed a log viewer that accepted a user-controlled filename. This functionality was vulnerable to Path Traversal, allowing arbitrary files on the server to be read and ultimately exposing `/root/flag-root.txt`.

The main security concepts involved were:

* Broken Access Control
* IDOR / BOLA
* API enumeration
* Sensitive information disclosure
* Credential exposure
* Privilege escalation
* Path Traversal
* Arbitrary File Read
* Vulnerability chaining

---

## 2. Objective

The objective of the laboratory was to compromise the ClearDesk application by:

1. Enumerating the exposed API and application functionality.
2. Identifying an authorization vulnerability.
3. Accessing resources belonging to other users.
4. Obtaining information that could be used to escalate privileges.
5. Accessing the administrative interface.
6. Identifying a file-reading vulnerability.
7. Exploiting Path Traversal to access `/root/flag-root.txt`.
8. Obtain the final root flag.

---

## 3. Environment

* **Platform:** HackerDNA
* **Lab:** ClearDesk
* **Target:** ClearDesk web application
* **Target hostname:** `ec2-3-252-89-37.eu-west-1.compute.amazonaws.com`
* **Target IP:** Dynamic; the IP changed when the laboratory was restarted
* **Attacker machine:** Kali Linux
* **Target operating system:** Linux
* **Web server:** nginx/1.28.2
* **Web application:** ClearDesk v2.1
* **Protocol:** HTTP
* **Tools:** Nmap, Gobuster, cURL, web browser

The target exposed the following relevant TCP services:

```text
53/tcp   domain
80/tcp   http
```

---

## 4. Reconnaissance

The first step was to identify the services exposed by the target.

A port and service scan was performed using Nmap:

```bash
nmap -sV -sC <TARGET_IP>
```

The scan identified:

```text
53/tcp   domain
80/tcp   http
```

The HTTP service was running behind:

```text
nginx/1.28.2
```

The target was identified as a Linux system.

### Web enumeration

The web application was then enumerated for commonly exposed directories and resources.

The following directories were discovered:

```text
/admin
/login
/api
```

No relevant files were discovered during the initial file enumeration.

`robots.txt` did not reveal additional resources.

### HTTP behavior

An HTTP request to the application returned a redirect:

```http
HTTP/1.1 302 FOUND
Server: nginx/1.28.2
Location: /
Vary: Cookie
```

The `Vary: Cookie` header was an early indication that the application was using session-based behavior.

---

## 5. Enumeration

### 5.1 Login page

The initial page presented the ClearDesk login interface:

```text
ClearDesk / IT Helpdesk Portal
```

No public account registration functionality was available.

However, the page provided demonstration credentials:

```text
Username: guest
Password: [REDACTED]
```

These credentials were used to authenticate to the application.

The application identified the authenticated employee as:

```text
Name: Alex Kim
Employee ID: 3
Department: IT Support
Role: Intern
```

---

### 5.2 Dashboard

After authentication, the dashboard displayed the user's tickets.

The two tickets initially visible were:

```text
#2  WiFi not working in Conference Room B
#3  Request: New monitor for workstation
```

Both tickets belonged to:

```text
owner_id = 3
```

which corresponded to the authenticated user.

The dashboard also exposed an API link.

---

### 5.3 API enumeration

Accessing:

```text
/api
```

revealed the application's API structure:

```json
{
  "authentication": "Session-based. Log in via /login first.",
  "endpoints": {
    "/api/ticket/<id>": "Get ticket by ID",
    "/api/tickets": "List your tickets",
    "/api/user/<id>": "Get user by ID",
    "/api/users": "List all users"
  },
  "service": "ClearDesk API",
  "version": "1.0"
}
```

This was an important discovery because several endpoints accepted an object identifier directly through the URL.

The most interesting endpoints were:

```text
/api/ticket/<id>
/api/user/<id>
/api/users
```

---

### 5.4 Initial ticket requests

The user's own ticket could be accessed through:

```text
/api/ticket/2
```

which returned:

```json
{
  "created": "2026-03-09T11:30:00Z",
  "description": "The WiFi in Conference Room B has been intermittent since Monday. Multiple team members have reported disconnections during meetings. Access point may need replacement.",
  "id": 2,
  "owner_id": 3,
  "priority": "medium",
  "status": "open",
  "subject": "WiFi not working in Conference Room B"
}
```

The second ticket was available through:

```text
/api/ticket/3
```

and also belonged to:

```text
owner_id = 3
```

At this point, the application appeared to behave normally.

---

### 5.5 User enumeration

The `/api/users` endpoint was queried.

It returned four users:

```text
ID 1 → Sarah Chen
ID 2 → Mike Roberts
ID 3 → Alex Kim
ID 4 → Jordan Lee
```

Relevant information included:

```text
Sarah Chen
ID: 1
Department: IT Administration
Role: IT Director

Mike Roberts
ID: 2
Department: Engineering
Role: Senior Developer

Alex Kim
ID: 3
Department: IT Support
Role: Intern

Jordan Lee
ID: 4
Department: Design
Role: UI Designer
```

This exposed information about other employees to the authenticated low-privileged user.

---

### 5.6 Ticket enumeration

The API endpoint:

```text
/api/ticket/<id>
```

accepted arbitrary ticket identifiers.

Instead of restricting access to the authenticated user's tickets, sequential ticket IDs were tested:

```text
/api/ticket/1
/api/ticket/2
/api/ticket/3
/api/ticket/4
...
/api/ticket/8
```

Tickets from IDs 1 through 8 were accessible.

This was the first major security finding.

---

## 6. Vulnerability Identification

### 6.1 IDOR — Unauthorized Access to Another User's Ticket

After logging in using Alex Kim's account, an account with an **Intern** role, I began testing the API endpoints responsible for retrieving support tickets.

The application exposed the following endpoint:

```text
GET /api/ticket/<id>
```

The expected behavior was that Alex Kim should only be able to access tickets belonging to his own account. However, by manually modifying the ticket ID, it was possible to retrieve tickets belonging to other users.

For example:

```text
/api/ticket/2
/api/ticket/3
```

returned tickets associated with Kim's account. By sequentially testing other ticket IDs, however, the application also returned tickets belonging to other employees.

One of the accessible tickets belonged to the IT Director, Sarah Chen. Since Kim's account did not have permission to access this resource, this demonstrated an **IDOR (Insecure Direct Object Reference)** vulnerability, also referred to as **BOLA (Broken Object Level Authorization)**.

The application successfully verified that the user was authenticated, but failed to verify whether the authenticated user was authorized to access the requested ticket.

The exploitation can be summarized as follows:

```text
Login as Alex Kim (Intern)
        ↓
Access the ticket API
        ↓
Modify the ticket ID
        ↓
Access a ticket belonging to Sarah Chen
        ↓
Read the ticket without authorization
        ↓
Discover sensitive information and the user flag
```

The important aspect of this vulnerability was that Sarah Chen's credentials were not required. The low-privileged Kim account was already sufficient to access a resource belonging to another, more privileged user.

This demonstrated a clear separation between **authentication** and **authorization**: Kim was successfully authenticated, but the application failed to enforce whether he was authorized to access the requested object.

### 6.2 Technical Analysis of the IDOR

The vulnerable functionality was identified in:

```text
/api/ticket/<id>
```

The authenticated user was:

```text
Alex Kim
Employee ID: 3
Role: Intern
```

The application correctly allowed access to tickets associated with:

```text
owner_id = 3
```

However, changing the ticket ID allowed access to tickets owned by other users.

For example:

```text
/api/ticket/1
```

could be accessed even though the ticket belonged to another employee.

Conceptually, the vulnerable implementation could resemble:

```python
ticket = Ticket.query.get(ticket_id)

return jsonify(ticket)
```

The problem is that the application retrieves the ticket based only on the user-supplied object ID. No authorization check is performed to determine whether the requested ticket belongs to the authenticated user.

A secure implementation should enforce authorization when retrieving the object:

```python
ticket = Ticket.query.filter_by(
    id=ticket_id,
    owner_id=current_user.id
).first()
```

If the requested ticket does not belong to the authenticated user, the application should deny access rather than return the object's contents. Depending on the application's design, this could result in a `403 Forbidden` or a `404 Not Found` response.

The fundamental security requirement is:

```text
Authentication:
"Is the user logged in?"

Authorization:
"Is this user allowed to access this specific ticket?"
```

The application enforced the first check but failed to properly enforce the second.

### 6.3 Sensitive Information Disclosure

The IDOR vulnerability exposed tickets belonging to other employees.

One of the accessible tickets belonged to the IT Director and contained sensitive information, including a temporary administrative credential.

The ticket also referenced:

```text
/root/flag-root.txt
```

and described an issue involving a backup job and permissions associated with that file.

This information was significant because the IDOR was not merely an isolated information disclosure issue. The exposed ticket provided information that could be used to continue the exploitation chain toward administrative access.

Therefore, the attack chain can be summarized as:

```text
Low-privileged account
        ↓
IDOR / BOLA
        ↓
Unauthorized ticket access
        ↓
Sensitive information disclosure
        ↓
Information enabling further exploitation
        ↓
Administrative access
```

### 6.4 Credential exposure and privilege escalation

The exposed ticket contained administrative credentials.

The password is intentionally omitted from this write-up:

```text
Username: admin
Password: [REDACTED]
```

These credentials were then used to authenticate as the administrative user.

The privilege transition was therefore:

```text
Intern
   ↓
IDOR
   ↓
Access to administrator's ticket
   ↓
Administrative credentials
   ↓
Administrator account
```

This is an example of vulnerability chaining.

---

### 6.5 Path Traversal / Arbitrary File Read

After authenticating as an administrator, the /admin endpoint became accessible.

The administrator panel exposed a log viewer:

/admin/logs?file=app.log
/admin/logs?file=error.log
/admin/logs?file=access.log

The application accepted the filename through the file parameter.

Legitimate log files could be requested:

app.log
error.log
access.log

An invalid filename returned:

{
  "error": "Log file 'test.log' not found"
}

This behavior indicated that the application was processing user-controlled input as a file path rather than strictly restricting access to a predefined set of log files.

The file parameter was therefore tested for a Path Traversal vulnerability.

By supplying directory traversal sequences, it was possible to access files outside the intended log directory.

For example:

/admin/logs?file=../../../../root/flag-root.txt

The application processed the traversal sequence and returned the contents of the requested file.

This demonstrated an Arbitrary File Read vulnerability caused by insufficient validation of the user-controlled file parameter.

The exploitation can be summarized as:

Administrator access
        ↓
Access /admin/logs
        ↓
Control the "file" parameter
        ↓
Insert "../" traversal sequences
        ↓
Escape the intended log directory
        ↓
Read an arbitrary file
        ↓
Access /root/flag-root.txt

The fundamental issue was that the application trusted the filename supplied by the client without securely constraining the resulting filesystem path.

A vulnerable implementation could conceptually resemble:

file_path = os.path.join(LOG_DIR, request.args.get("file"))

with open(file_path, "r") as f:
    return f.read()

If the supplied filename contains traversal sequences such as ../, the resulting path can escape the intended directory.

A secure implementation should avoid directly trusting user-controlled filesystem paths. Where possible, the application should use a strict allowlist of permitted log files and map user-supplied identifiers to predefined files.

For example:

allowed_logs = {
    "app": "/var/log/app.log",
    "error": "/var/log/error.log",
    "access": "/var/log/access.log"
}

log = request.args.get("file")

if log not in allowed_logs:
    return {"error": "Invalid log file"}, 404

with open(allowed_logs[log], "r") as f:
    return f.read()

This prevents the client from directly controlling the filesystem path.

### 6.6 Complete Exploitation Chain

The complete attack chain consisted of multiple vulnerabilities and security weaknesses:

Low-privileged Intern account
        ↓
IDOR / BOLA
        ↓
Unauthorized access to another user's ticket
        ↓
Sensitive information disclosure
        ↓
Exposure of administrative credentials
        ↓
Administrator authentication
        ↓
Access to /admin/logs
        ↓
Path Traversal
        ↓
Arbitrary File Read
        ↓
Access to /root/flag-root.txt

The most important security lesson from this chain is that individual vulnerabilities can become significantly more impactful when combined.

The initial IDOR provided access to information that should have been restricted to another employee. That information exposed administrative credentials, which enabled access to the administrator interface. The administrator interface then contained a second vulnerability that allowed arbitrary files to be read from the server.

Therefore, the overall compromise was not dependent on a single vulnerability, but on the chaining of multiple security weaknesses:

Broken Object-Level Authorization → Credential Exposure → Privilege Escalation → Path Traversal → Arbitrary File Read.

---

## 7. Exploitation

### 7.1 Obtaining the initial session

The application provided demonstration credentials on the login page.

These credentials were used to authenticate as the low-privileged user.

The resulting identity was:

```text
Alex Kim
Employee ID: 3
Role: Intern
```

---

### 7.2 Discovering the API

The API documentation exposed:

```text
/api/ticket/<id>
/api/tickets
/api/user/<id>
/api/users
```

The ticket endpoint was particularly interesting because it used a predictable numerical identifier.

---

### 7.3 Exploiting the IDOR

The legitimate tickets were:

```text
/api/ticket/2
/api/ticket/3
```

Both belonged to employee ID 3.

The ticket identifier was then changed manually:

```text
/api/ticket/1
/api/ticket/4
/api/ticket/5
...
/api/ticket/8
```

The server returned the requested tickets even when they belonged to other employees.

This confirmed the IDOR/BOLA vulnerability.

The application did not enforce object-level authorization.

---

### 7.4 Obtaining administrative credentials

One of the unauthorized tickets belonged to the administrator and contained a temporary administrator credential.

The sensitive credential is not reproduced here.

The important exploitation step was:

```text
Unauthorized ticket access
        ↓
Administrator ticket
        ↓
Credential disclosure
```

The credential was then used to authenticate to the application as `admin`.

---

### 7.5 Accessing the Admin Panel

Before the privilege escalation, requesting:

```text
/admin
```

returned:

```json
{
  "error": "Admin access required"
}
```

After authenticating using the recovered administrative account, `/admin` became accessible.

The panel identified the user as:

```text
Sarah Chen
```

and exposed the following functionality:

```text
Log Viewer
User Management
```

The Log Viewer exposed:

```text
/admin/logs?file=app.log
/admin/logs?file=error.log
/admin/logs?file=access.log
```

---

### 7.6 Testing the Log Viewer

The legitimate log files were requested first.

For example:

```text
/admin/logs?file=app.log
```

returned the contents of `app.log`.

Similarly:

```text
/admin/logs?file=error.log
/admin/logs?file=access.log
```

returned the corresponding log contents.

A nonexistent file was then tested:

```text
/admin/logs?file=test.log
```

The server returned:

```json
{
  "error": "Log file 'test.log' not found"
}
```

This indicated that the parameter was being used to select a file.

---

### 7.7 Exploiting Path Traversal

The `file` parameter was then manipulated using parent-directory traversal:

```text
../
```

The final payload used was:

```text
/admin/logs?file=../../../../root/flag-root.txt
```

The `../` sequences instruct the filesystem to move to the parent directory.

Conceptually:

```text
/log-directory/
      ↓ ../
/parent/
      ↓ ../
/parent/
      ↓ ../
/
      ↓
/root/flag-root.txt
```

Because the application did not properly restrict the resolved path to the intended log directory, it was possible to escape the expected directory and access the root-owned file.

The server returned the contents of:

```text
/root/flag-root.txt
```

This confirmed an **Arbitrary File Read** vulnerability caused by insufficient Path Traversal protection.

---

## 8. Privilege Escalation

Privilege escalation was part of the exploitation chain, although it occurred primarily at the application level.

### Initial privilege

The initial authenticated account was:

```text
Alex Kim
Role: Intern
Employee ID: 3
```

This account was not authorized to access the administrator panel.

### Escalation mechanism

The IDOR vulnerability allowed access to tickets belonging to other users.

An administrator-owned ticket disclosed administrative credentials.

Those credentials were used to authenticate as:

```text
admin
```

This provided access to:

```text
/admin
```

and the administrator's Log Viewer.

### Final access

The Log Viewer could be abused through Path Traversal to read:

```text
/root/flag-root.txt
```

This was not a traditional Linux:

```text
user → root shell
```

privilege escalation.

Instead, the attacker abused a privileged server-side application to perform a file read that the original low-privileged user should never have been able to perform.

The application effectively acted as a privileged intermediary.

---

## 9. Proof of Compromise

The final objective was successfully achieved by reading:

```text
/root/flag-root.txt
```

through the vulnerable endpoint:

```text
/admin/logs?file=../../../../root/flag-root.txt
```

The endpoint returned the contents of the root-protected file, including the laboratory flag.

The flag obtained during the exercise was:

```text
e8ef8880-66dc-4873-899f-df293f8b72bd
```

Administrative credentials are intentionally not included in this section.

---

## 10. Attack Chain

The complete exploitation path was:

```text
Reconnaissance
      ↓
Web Enumeration
      ↓
Discover /api
      ↓
Enumerate API endpoints
      ↓
Authenticate as guest
      ↓
Identify Alex Kim / Employee ID 3
      ↓
Enumerate ticket IDs
      ↓
IDOR / BOLA
      ↓
Access tickets belonging to other users
      ↓
Administrator ticket exposed
      ↓
Administrative credentials disclosed
      ↓
Authenticate as administrator
      ↓
Access /admin
      ↓
Discover Log Viewer
      ↓
file=<filename>
      ↓
Path Traversal
      ↓
Arbitrary File Read
      ↓
/root/flag-root.txt
      ↓
ROOT FLAG
```

---

## 11. Root Cause

The laboratory contained two primary security design failures.

### 11.1 Missing object-level authorization

The API authenticated users but did not consistently verify whether they were authorized to access the requested resource.

The vulnerable design was effectively:

```text
User authenticated?
       ↓
      YES
       ↓
Return requested object
```

instead of:

```text
User authenticated?
       ↓
      YES
       ↓
Is user authorized for this object?
       ↓
   ┌───┴───┐
  YES      NO
   ↓        ↓
Return    403
object
```

This allowed an authenticated low-privileged user to enumerate and access other users' tickets.

---

### 11.2 Unsafe file path handling

The Log Viewer accepted a user-controlled filename:

```text
/admin/logs?file=<filename>
```

without adequately restricting the resulting filesystem path.

The application should have restricted access to the intended log directory and prevented traversal outside that directory.

Because this protection was missing or incorrectly implemented, sequences such as:

```text
../../../../
```

could be used to escape the intended directory.

---

## 12. Mitigation

### 12.1 Fix IDOR / BOLA

Every object request should perform an authorization check.

For example:

```python
ticket = Ticket.query.filter_by(
    id=ticket_id,
    owner_id=current_user.id
).first()

if not ticket:
    abort(403)
```

Authorization should be enforced **server-side**, never relying on the frontend to hide unauthorized objects.

---

### 12.2 Restrict user enumeration

The `/api/users` endpoint should only expose information that the authenticated user is authorized to access.

If ordinary users do not require a complete employee directory, the endpoint should not expose all internal accounts.

Sensitive information such as credentials should never be included in tickets or API responses.

---

### 12.3 Never store credentials in tickets

Administrative credentials should never be stored in helpdesk tickets.

If credentials must be distributed, a dedicated secrets-management mechanism should be used.

Passwords should also be:

* Strong
* Unique
* Hashed when stored
* Rotated when temporary
* Never exposed through application content

---

### 12.4 Prevent Path Traversal

The application should not directly concatenate user input with filesystem paths.

Instead, use a strict allowlist:

```python
ALLOWED_LOGS = {
    "app.log",
    "error.log",
    "access.log"
}
```

Only predefined files should be accessible.

The application should also canonicalize and validate paths before opening them.

The final resolved path must remain inside the intended log directory.

Conceptually:

```text
Requested path
      ↓
Normalize / resolve
      ↓
Is path inside allowed directory?
      ↓
   ┌──┴──┐
  YES    NO
   ↓      ↓
Allow   Deny
```

---

### 12.5 Apply least privilege

The web application process should run with the minimum filesystem privileges required for its operation.

A logging feature should not need permission to read:

```text
/root/*
```

Restricting filesystem permissions would reduce the impact of an arbitrary file read vulnerability.

---

### 12.6 Improve session and administrative security

Administrative accounts should use:

* Strong unique passwords
* Secure session management
* Multi-factor authentication where possible
* Short-lived sessions
* Appropriate authorization checks
* Credential rotation

---

## 13. Lessons Learned

This laboratory demonstrated that authentication alone does not make an application secure.

The most important distinction was between:

```text
Authentication
```

and:

```text
Authorization
```

The application correctly identified the authenticated user, but failed to consistently determine which resources that user was allowed to access.

The IDOR vulnerability demonstrated how predictable object identifiers can become dangerous when authorization checks are missing.

Another important lesson was the impact of **vulnerability chaining**.

The IDOR did not directly provide root access. Instead:

```text
IDOR
 ↓
Unauthorized ticket
 ↓
Credential disclosure
 ↓
Administrative account
 ↓
Admin functionality
 ↓
Path Traversal
 ↓
Arbitrary File Read
 ↓
Root flag
```

This demonstrates why vulnerabilities should not always be evaluated in isolation.

A seemingly moderate information-disclosure vulnerability can become critical when the disclosed information enables authentication as a privileged user.

The laboratory also demonstrated the danger of filesystem operations based on user-controlled input. A parameter that appears harmless, such as:

```text
file=app.log
```

can become extremely dangerous when it is used directly to access files on the server.

Finally, the lab reinforced an important pentesting methodology:

```text
Do not immediately look for an exploit.
First understand the application.
```

The successful exploitation came from progressively understanding:

1. The available services.
2. The web application.
3. The API.
4. The authenticated user's privileges.
5. The object identifiers.
6. The authorization behavior.
7. The information exposed by unauthorized objects.
8. The functionality available after privilege escalation.
9. The parameters controlling filesystem access.

This approach made it possible to build the complete attack chain rather than relying on blind exploitation.
