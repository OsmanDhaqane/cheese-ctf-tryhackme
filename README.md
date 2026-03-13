# Cheese CTF – TryHackMe Walkthrough

## Overview
Cheese CTF is a Linux-based TryHackMe room that includes web enumeration, authentication bypass, local file inclusion, remote code execution, gaining user access, and privilege escalation to root.

## Objectives
- Gain initial access to the target
- Capture the user flag
- Escalate privileges to root
- Capture the root flag

## Tools Used
- Nmap
- ffuf
- curl
- php_filter_chain_generator
- netcat
- SSH
- MySQL
- systemctl
- xxd

## Target Information
- Target IP: `10.x.x.x`

## Walkthrough Sections
## 1. Reconnaissance
I started with an Nmap SYN scan against the target to identify open ports and potential services.

```bash
nmap -sS 10.x.x.x
```
<img width="531" height="319" alt="recon" src="https://github.com/user-attachments/assets/e6dcca80-6b30-4937-af8b-777dd4434adb" />

The scan reported a very large number of open ports. This suggested that the result might be noisy or intentionally misleading, so I decided not to focus on every reported service.

To continue enumeration, I shifted attention to the web application and browsed the target over HTTP. I also performed directory and file enumeration to identify exposed pages and interesting resources.
```bash
feroxbuster -u http://10.x.x.x -w /usr/share/dirb/wordlists/common.txt -x php,txt,html
```
<img width="1120" height="668" alt="enumeration" src="https://github.com/user-attachments/assets/959911d9-cd38-49fa-b162-74b1eeea88f8" />
Next, I visited the main page in the browser and found a visible Login link.

Opening the login page confirmed that the web application was likely the main attack surface for initial access.
<img width="879" height="662" alt="the_web_page" src="https://github.com/user-attachments/assets/0df413ff-3359-481e-9cbd-1e1e9b841926" />
<img width="538" height="365" alt="login_page" src="https://github.com/user-attachments/assets/878603e3-fe10-4dde-954d-8612277e3a4a" />


## 2. Login Bypass (SQL Injection)

After identifying `login.php` as the main entry point, I tested the login form for basic injection behavior. A simple single-quote test returned the normal login failure message, so I continued with more focused testing.

To identify unusual responses, I used `ffuf` with a SQL injection payload list against the `username` parameter.

```bash
ffuf -w /usr/share/seclists/Fuzzing/login_bypass.txt \
-u http://10.x.x.x/login.php \
-X POST \
-d "username=FUZZ&password=test" \
-H "Content-Type: application/x-www-form-urlencoded" \
-mc all
```

<img width="802" height="642" alt="ffuf-login-bypass-1" src="https://github.com/user-attachments/assets/c000d026-c96c-4cf5-b7c9-13c35819e1d5" />

Most responses returned the same status code and size, but a few payloads produced a different result. In particular, I observed 302 redirects, which suggested a successful authentication bypass.

One working payload was:
```
' OR 'x'='x'#;
```
<img width="708" height="28" alt="ffuf-login-bypass-2" src="https://github.com/user-attachments/assets/ca3a4915-5ffe-4f50-b5f5-af5a50ee0606" />

I entered that payload into the username field and used any value for the password.

- Username: ' OR 'x'='x'#;
- Password: test
<img width="473" height="321" alt="sqli-payload-login" src="https://github.com/user-attachments/assets/19ad244c-0d07-4acf-ba31-93dc80015379" />

This successfully bypassed authentication and redirected me into the application, confirming that the login form was vulnerable to SQL injection in the username field.

<img width="939" height="322" alt="post-login-redirect" src="https://github.com/user-attachments/assets/3a974e03-52fd-431b-ba84-29d3006abd8e" />

After the redirect, I landed on the admin panel page at:

`secret-script.php?file=supersecretadminpanel.html`

At first glance, the panel did not reveal much useful information. The visible sections were very minimal, and some pages appeared to contain little or no actionable content. For example, the **Users** page only displayed a simple heading, and the **Messages** area did not immediately expose anything obvious. This suggested that the real value was not in the page content itself, but in how the application was loading files through the `file=` parameter.


## 3. Local File Inclusion (LFI)

After bypassing the login page, I noticed that the application loaded content through the `file=` parameter in `secret-script.php`.

The URL looked like this:

```text
http://10.x.x.x/secret-script.php?file=supersecretadminpanel.html
```
Because the application was directly referencing files through a GET parameter, I tested for Local File Inclusion (LFI) by replacing the value with a path traversal sequence targeting /etc/passwd.
```text
http://10.x.x.x/secret-script.php?file=../../../../../../etc/passwd
```
<img width="1912" height="258" alt="lfi-etc-passwd" src="https://github.com/user-attachments/assets/230a0b5a-b200-4717-83b4-40881d36ca48" />

The application returned the contents of /etc/passwd, which confirmed that the file= parameter was vulnerable to Local File Inclusion.

This was an important finding because it meant I could read local files from the server and potentially inspect the source code of the vulnerable application.


## 4. Reading PHP Source Code

After confirming Local File Inclusion, I used the PHP stream wrapper `php://filter` to read the source code of the vulnerable script instead of executing it directly.

I requested the file through Base64 encoding:

```text
http://10.x.x.x/secret-script.php?file=php://filter/convert.base64-encode/resource=secret-script.php
```


5. Remote Code Execution (RCE)
6. Reverse Shell
7. Credential Discovery
8. User Access via SSH Key Injection
9. User Flag
10. Privilege Escalation to Root
11. Root Flag
12. Lessons Learned
