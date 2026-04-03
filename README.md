# 🛡️ ModSecurity WAF + DVWA – Web Attack Detection Lab

![ModSecurity](https://img.shields.io/badge/ModSecurity_WAF-CC0000?style=for-the-badge)
![DVWA](https://img.shields.io/badge/DVWA-8B0000?style=for-the-badge)
![Wazuh](https://img.shields.io/badge/Wazuh-005DAC?style=for-the-badge&logo=wazuh&logoColor=white)
![Kali Linux](https://img.shields.io/badge/Kali_Linux-557C94?style=for-the-badge&logo=kalilinux&logoColor=white)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE_ATT%26CK-FF0000?style=for-the-badge)

A hands-on web attack simulation lab using DVWA as the target application and ModSecurity WAF as the defensive layer. Real attack payloads were sent from Kali Linux, WAF blocking behavior was confirmed, and all detections were verified through ModSecurity error logs and Wazuh SIEM.

---

## 🖥️ Lab Environment

| Component | Role |
|---|---|
| DVWA | Deliberately vulnerable web application, attack target |
| ModSecurity WAF | Web application firewall sitting in front of DVWA |
| Apache 2.4.58 | Web server hosting DVWA on Ubuntu (192.168.1.100:8080) |
| Kali Linux | Attacker machine (192.168.1.101), Burp Suite, browser |
| Wazuh | SIEM monitoring all events from KaliLinux and Ubuntu agents |

---

## 📂 What Was Attacked and Detected

---

### 🔴 SQL Injection – WAF Block Confirmed

Submitted a SQL injection payload directly via the DVWA SQLi vulnerability page URL.

**Payload used:**
```
http://192.168.1.100:8080/DVWA/vulnerabilities/sqli/?id='+OR+'1'%3D1&Submit=Submit#
```

**WAF response – 403 Forbidden returned by Apache/ModSecurity, payload blocked:**

![1773477883010](https://github.com/user-attachments/assets/593e7b64-c0b7-4a17-957e-cfe182dfebe7)

<br>

- ModSecurity intercepted the SQL injection payload in the URL parameter
- Apache returned HTTP 403 Forbidden – the request never reached DVWA
- WAF blocking confirmed end-to-end – attacker received no data from the database
- 📌 MITRE: `T1190` Exploit Public-Facing Application · `T1083` File and Directory Discovery

---

### 🔴 XSS Attack – OWASP CRS Rules Fired

Submitted reflected XSS payloads against DVWA XSS vulnerability page from Kali Linux browser.

**ModSecurity error log showing XSS detection and blocking – OWASP CRS rules 941100, 941160, 949110, 980130 all firing:**

![1773477902062](https://github.com/user-attachments/assets/3caa7a1d-d39c-4b01-a9ef-a8046adb96aa)

<br>

**OWASP CRS rules triggered:**

| Rule ID | Description | Severity |
|---|---|---|
| 941100 | XSS Attack Detected via libinjection | CRITICAL |
| 941160 | XSS Filter – Category 2: Event Handler Vector | CRITICAL |
| 949110 | Inbound Anomaly Score Exceeded (Total Score: 18) | CRITICAL |
| 980130 | Inbound Anomaly Score Exceeded – event correlation (SQLI=0, XSS=15) | CRITICAL |

- Payload `<img src=x onerror=alert(1)>` matched rule 941100 via libinjection detection
- XSS score accumulated to 15, pushing total inbound anomaly score to 18
- Rule 949110 fired when anomaly score exceeded threshold – access denied with 403
- Rule 980130 confirmed event correlation across multiple rule hits – systematic attack pattern confirmed
- 📌 MITRE: `T1059.007` JavaScript · `T1190` Exploit Public-Facing Application

---

### 🔴 Brute Force Attack Against DVWA Login – Burp Suite Intruder

Used Burp Suite Intruder to brute force the DVWA login page with username and password wordlists.

**Burp Suite Intruder results – POST requests to /DVWA/vulnerabilities/brute/, testing username/password combinations, all returning HTTP 200:**

![1773477900382](https://github.com/user-attachments/assets/dccb2bb1-301b-4c87-99c8-27fb716f091e)

<br>

- Intruder attack targeting `POST /DVWA/vulnerabilities/brute/ HTTP/1.1`
- Usernames tested: tony, rohith, rocket, admin, root, john
- All responses returned HTTP 200 with consistent response length of 5484 bytes – indicating login page was returned each time without successful authentication
- Cookie `security=impossible` visible in request – DVWA security level set to impossible during testing
- 📌 MITRE: `T1110.001` Password Guessing · `T1078` Valid Accounts

---

### 🔴 Wazuh Detection – KaliLinux Agent Monitoring

Wazuh was monitoring the KaliLinux agent during all attack activity, capturing file integrity and system events generated during the attack simulation.

**Wazuh Threat Hunting dashboard for KaliLinux agent – 7,959 total alerts, top groups: syscheck, syscheck_file, syscheck_entry events:**

![1773477900613](https://github.com/user-attachments/assets/f143357c-e1f8-4422-a2e1-ed2294b08665)

<br>

**Wazuh KaliLinux event list – integrity checksum changed (rule 550), file deleted (rule 553), 7,959 hits across 531 pages:**

![1773477900319](https://github.com/user-attachments/assets/3b96b6e7-c915-47d8-958b-188ff882971c)

<br>

**Wazuh Discover view – 8,281 hits, AppArmor DENIED events (rule 52002) firing repeatedly during attack period:**

![1773477899277](https://github.com/user-attachments/assets/037bae99-ae53-44b8-92ec-94ed2dc010de)

<br>

- 7,959 alerts generated on KaliLinux agent during the attack window
- Integrity checksum changes (rule 550) and file deletions (rule 553) captured by Wazuh FIM
- AppArmor DENIED events (rule 52002) firing — kernel-level application control blocking unauthorized access attempts
- 📌 MITRE: `T1046` Network Service Scanning · `T1565.001` Stored Data Manipulation

---

## 🧾 IOCs

| Type | Value | Source |
|---|---|---|
| Attacker IP | 192.168.1.101 | ModSecurity logs, Wazuh |
| Target | 192.168.1.100:8080 | DVWA/Apache |
| SQLi payload | `'+OR+'1'%3D1` | Browser URL bar |
| XSS payload | `<img src=x onerror=alert(1)>` | ModSecurity error log |
| OWASP Rules fired | 941100, 941160, 949110, 980130 | ModSecurity error log |
| WAF response | HTTP 403 Forbidden | Apache/ModSecurity |

---

## 📁 Repository Structure

```
ModSecurity-WAF-DVWA-Lab/
└── screenshots/
    ├── sqli-blocked-403.png
    ├── modsecurity-xss-logs.png
    ├── burpsuite-intruder-brute.png
    ├── wazuh-kali-dashboard.png
    ├── wazuh-kali-events.png
    └── wazuh-discover-apparmor.png
```

---

⬅️ [Back to SOC Home Lab](https://github.com/rohithbaggu56-dot/Home-SOC-Lab-Detection-Log-Analysis) · [Back to Portfolio](https://github.com/rohithbaggu56-dot)
