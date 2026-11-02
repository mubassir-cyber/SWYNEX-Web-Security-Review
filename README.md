# 🔐 Web Security Review

**A defensive security review of an intentionally vulnerable training application, performed in an isolated local lab**

![Skill](https://img.shields.io/badge/Skill-Cybersecurity-404040?style=flat-square&labelColor=C00000) ![Target](https://img.shields.io/badge/Target-OWASP%20Juice%20Shop-E87500?style=flat-square&labelColor=000000) ![Tool](https://img.shields.io/badge/Tool-Burp%20Suite-0070C0?style=flat-square&labelColor=000000) ![Skill](https://img.shields.io/badge/Skill-Linux-404040?style=flat-square&labelColor=C00000) ![Port](https://img.shields.io/badge/Target%20Port-3000-238F89?style=flat-square&labelColor=000000) ![Web App Security](https://img.shields.io/badge/Web%20App%20Security-C00000?style=flat-square&labelColor=000000) ![Skill](https://img.shields.io/badge/Skill-OWASP%20Top%2010-404040?style=flat-square&labelColor=C00000) ![GitHub](https://img.shields.io/badge/GitHub-404040?style=flat-square&labelColor=0070C0&logo=github&logoColor=white) ![SWYNEX](https://img.shields.io/badge/SWYNEX%20Technologies-404040?style=flat-square&labelColor=C00000) ![Ethical Hacking](https://img.shields.io/badge/Ethical%20Hacking-E87500?style=flat-square&labelColor=000000) ![Author](https://img.shields.io/badge/Author-%5BYour%20Name%5D-C00000?style=flat-square)

---

## 📌 Project Overview

This is my **Task 3 — Web Security Review** submission for the **SWYNEX Technologies** internship.

The project performs a **defensive security review** of a web application running entirely inside my own Kali Linux environment. I used **OWASP Juice Shop**, an intentionally vulnerable training application, as the authorized target, and documented every finding with evidence, severity, and remediation guidance.

> **Scope Note:** All testing was performed exclusively against a local, authorized training application (`localhost:3000`). No third-party or production systems were accessed, scanned, or targeted at any point.

---

## 🎯 Objectives

- Deploy OWASP Juice Shop locally as an authorized, isolated test target.
- Perform reconnaissance using `nmap`, `nikto`, and `curl`.
- Set up Burp Suite as an intercepting proxy to capture and inspect live traffic.
- Identify and manually validate common web vulnerabilities (OWASP Top 10).
- Triage automated scan output and discard false positives.
- Document each finding with location, evidence, severity, and remediation.
- Produce a structured, reviewable security report.

---

## 🛡️ Purpose of the Review

This review gave me hands-on practice with the full defensive assessment workflow, including:

- Reconnaissance and service fingerprinting
- Proxy-based traffic interception and request manipulation
- Manual exploitation of common injection and access-control flaws
- Distinguishing real findings from scanner noise/false positives
- Mapping findings to the OWASP Top 10 and writing remediation guidance

⚠️ **Important:** This lab was tested only on an application I deployed myself for training purposes. Never use these techniques against real, unauthorized systems.

---

## 🏗️ Lab Architecture

```
            Kali Linux (Host)
                   |
         Docker Engine (local)
                   |
   OWASP Juice Shop Container
        127.0.0.1:3000  (NAT'd, not exposed externally)
                   |
      Burp Suite Community (proxy, 127.0.0.1:8080)
                   |
         Firefox (manual testing)
```

The target container was never exposed beyond `localhost`, keeping the entire review self-contained on one machine.

---

## ⚙️ Lab Configuration

| 🧩 Component | ⚙️ Configuration |
|---|---|
| 🖥️ Host OS | Kali Linux |
| 🎯 Target Application | OWASP Juice Shop |
| 🐳 Deployment | Docker container (`bkimminich/juice-shop`) |
| 🌐 Target URL | `http://localhost:3000` |
| 🔍 Recon Tools | nmap, nikto, curl |
| 🕵️ Proxy Tool | Burp Suite Community Edition |
| 🌍 Browser | Firefox |

---

# 🪜 Review Procedure

## Step 1. Deploy the target application

Juice Shop was run as a Docker container, bound only to localhost:

```bash
sudo docker run -d -p 3000:3000 --name juice-shop bkimminich/juice-shop
curl -I http://127.0.0.1:3000
```

## Step 2. Reconnaissance — port and service scan

```bash
nmap -sV -p 3000 127.0.0.1
```

![nmap port scan result](screenshots/01-nmap-port-scan.png)

## Step 3. Reconnaissance — HTTP response headers

```bash
curl -I http://localhost:3000
```

![HTTP response headers](screenshots/02-http-response-headers.png)

## Step 4. Reconnaissance — automated misconfiguration scan

```bash
nikto -h http://localhost:3000
```

![nikto scan results](screenshots/03-nikto-scan-results.png)

> Nikto also reported ~150 "potentially interesting backup/cert file" hits (e.g. `/backup.tar`, `/database.jks`). These were manually verified and triaged as **false positives**, since the target is a single-page app that returns `HTTP 200` for almost any path. Only the genuine, manually confirmed findings below were kept.

## Step 5. Intercept traffic with Burp Suite

Burp Suite Community Edition was configured as the browser's proxy to capture and inspect live requests before manual exploitation began.

![Burp Suite capturing the login request](screenshots/04-burp-proxy-sqli-capture.png)

## Step 6. Manually test and validate each finding

Each finding below was reproduced manually and confirmed with evidence (Steps detailed in the **Findings** section).

---

# 🔎 Findings Summary

| # | Finding | OWASP Category | Severity |
|---|---------|-----------------|----------|
| 1 | SQL Injection — Authentication Bypass | A03:2021 – Injection | **Critical** |
| 2 | Reflected Cross-Site Scripting (XSS) | A03:2021 – Injection | **Medium** |
| 3 | Broken Access Control — Sensitive File Exposure | A01:2021 – Broken Access Control | **High** |
| 4 | Security Misconfiguration | A05:2021 – Security Misconfiguration | **Medium** |

---

## Finding 1: SQL Injection — Authentication Bypass

**Location:** `POST /rest/user/login` (login form, email field)

**Description:** The login form does not sanitize user input before using it in a backend database query. Submitting a SQL injection payload in the email field bypasses authentication entirely and logs the attacker in as the administrator, with no valid credentials.

**Steps to Reproduce:**
1. On the login page, enter `' or 1=1--` in the **Email** field.
2. Enter any value in **Password**.
3. Click **Log in**.

**Evidence:**

![SQLi payload in login form](screenshots/05-sqli-login-payload.png)
![Admin login confirmed](screenshots/06-sqli-admin-login-confirmed.png)

The Burp Suite capture in Step 5 above shows the raw `POST /rest/user/login` request and response, including a valid session token issued for `admin@juice-sh.op`.

**Severity:** Critical — complete authentication bypass with no valid credentials, granting full administrative access.

**Remediation:**
- Use parameterized queries / prepared statements.
- Apply strict server-side input validation on authentication fields.
- Use an ORM with built-in query parameterization.

---

## Finding 2: Reflected Cross-Site Scripting (XSS)

**Location:** `GET /#/search?q=` (search bar)

**Description:** User input in the search feature is reflected back into the page without output encoding, allowing arbitrary HTML/JavaScript execution.

**Steps to Reproduce:**
1. In the search bar, enter: `<iframe src="javascript:alert('xss')">`
2. Press Enter.

**Evidence:**

![Reflected XSS alert](screenshots/07-xss-reflected-alert.png)

**Severity:** Medium — could enable session hijacking or phishing if combined with a social-engineering vector.

**Remediation:**
- Encode all user-supplied output before rendering into HTML context.
- Implement a strict Content Security Policy (CSP).
- Avoid unsafe DOM sinks (e.g. `innerHTML`) for user input.

---

## Finding 3: Broken Access Control — Sensitive File Exposure

**Location:** `GET /ftp/`

**Description:** The `/ftp/` directory is publicly reachable with no authentication and exposes a directory listing of internal files, including what appears to be a password database and confidential business documents.

**Steps to Reproduce:**
1. Note the `/ftp/` entry disclosed in `robots.txt`.
2. Navigate to `http://localhost:3000/ftp`.

**Evidence:**

![FTP directory listing exposed](screenshots/08-ftp-sensitive-files-exposed.png)

Files visible without authentication include `incident-support.kdbx`, `acquisitions.md`, `legal.md`, `announcement_encrypted.md`, and several `.bak` backup files.

**Severity:** High — unauthenticated disclosure of apparently confidential business and credential-related files.

**Remediation:**
- Disable directory listing on the web server.
- Move non-public files outside the web root or restrict with proper authentication.
- Audit all publicly reachable paths for sensitive content before deployment.

---

## Finding 4: Security Misconfiguration

**Location:** HTTP response headers; server fingerprint

**Description:** Reconnaissance surfaced several configuration weaknesses:
- `Access-Control-Allow-Origin: *` — an overly permissive CORS policy.
- `X-Recruiting: /#/jobs` — a non-standard header disclosing an internal path.
- `robots.txt` discloses the sensitive `/ftp/` path rather than the path being restricted server-side.

**Evidence:** See Steps 2–4 above (`screenshots/01-nmap-port-scan.png`, `screenshots/02-http-response-headers.png`, `screenshots/03-nikto-scan-results.png`).

**Severity:** Medium — individually low-impact, but weakens the application's overall security posture.

**Remediation:**
- Restrict `Access-Control-Allow-Origin` to a known allow-list.
- Remove non-standard headers that disclose internal details.
- Don't rely on `robots.txt` as a substitute for real access control.

---

# 🐞 Problems Encountered & Solutions

## Problem 1. Docker port 3000 already allocated

When starting the Juice Shop container, `docker run` failed with `Bind for 0.0.0.0:3000 failed: port is already allocated`.

**Cause:** A Juice Shop container from an earlier session was already running on port 3000, and two more `docker run` attempts had been left in a `Created` (not running) state.

**Solution:**
1. Ran `docker ps -a` to see all containers and `ss -tulpn | grep 3000` to confirm what held the port.
2. Found the original container (`juice-shop`) was already up and serving correctly.
3. Removed the two failed, non-running containers with `docker rm -f`.
4. Verified the app with `curl -I http://127.0.0.1:3000`.

**Result:** A single, known-good Juice Shop container served the entire review with no port conflicts.

---

# 💡 What I Learned

### 1. The full assessment workflow
I practiced the complete defensive review cycle: recon → proxy setup → manual validation → evidence capture → severity rating → remediation.

### 2. Triage over blind trust in scanners
Automated tools like Nikto produce false positives on single-page apps. Manually confirming each finding before reporting it is essential to a credible review.

### 3. How common OWASP Top 10 flaws actually work
Reproducing SQL injection, XSS, and broken access control hands-on made the underlying mechanics far clearer than reading about them.

### 4. Using Burp Suite as an interception proxy
I learned to capture, inspect, and read raw HTTP requests/responses to confirm exploitation rather than relying on the UI alone.

### 5. Documentation as part of the job
Recording scope, steps, evidence, and remediation in a structured report is as much a part of security work as finding the bugs.

---

# 🔐 Security & Ethical Use

This review was performed for **education and internship assessment purposes only**, entirely inside a local, isolated environment on a device I own. I only tested a training application I deployed myself, and no third-party or production systems were accessed at any point.

---

# 🔗 Tools & Resources

- **OWASP Juice Shop:** https://owasp.org/www-project-juice-shop/
- **Burp Suite Community Edition:** https://portswigger.net/burp/communitydownload
- **nmap:** https://nmap.org/
- **Nikto:** https://cirt.net/Nikto2
- **OWASP Top 10:** https://owasp.org/www-project-top-ten/

---

# 👤 Author

**[Your Name]**
Cybersecurity Intern, SWYNEX Technologies

---

## 📌 Project Information

**Program Name:** Cybersecurity Internship at SWYNEX Technologies | **Task:** Task 3 | **Project:** Web Security Review | **Repository:** GitHub
