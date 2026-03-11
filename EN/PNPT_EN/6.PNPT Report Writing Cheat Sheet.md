This is a report writing guide for the PNPT (Practical Network Penetration Tester) certification by TCM Security.

Even if you successfully compromise the Domain Controller, a poor-quality report will result in a failing grade.

Past exam candidates repeatedly report: "I failed my first attempt because of the report."

### [Report Writing Overall Flow]

The goal is to document the entire penetration test in a way that both technical and non-technical readers can understand.

```bash
[Domain Controller compromised]

Phase 1. Design Report Structure
  Base on TCM Security sample report template
  Review all sections and fill in missing screenshots

Phase 2. Write Executive Summary
  Non-technical summary for C-level executives
  Current vulnerability status, security strengths/weaknesses

Phase 3. Write Technical Findings
  Document each vulnerability in its own detailed section
  Order: Critical → High → Medium → Low → Info

Phase 4. Write Attack Chain
  From OSINT → external penetration → internal network → DA
  Narrate the full attack flow with screenshots

Phase 5. Write Remediation Recommendations
  Provide specific fixes for each vulnerability

Phase 6. Review and Submit
  Final check for typos and missing screenshots

[Report submitted → 15-minute debrief → Pass/Fail]
```

### [TCM Security Sample Template]

TCM Security publishes an official sample report template on GitHub.

Passing candidates consistently report using or referencing this template.

※ TCM Security sample template: https://github.com/hmaverickadams/TCM-Security-Sample-Pentest-Report

**Full Report Structure:**

```bash
[Cover Page]
  Client name, report title, date, author, classification level

[Table of Contents]
  Include page numbers

[1] Disclaimer
  - This test represents a point-in-time snapshot
  - Only vulnerabilities found within the contracted period are included
  - No liability for new vulnerabilities discovered after testing ends

[2] Assessment Overview
  - Based on NIST penetration testing methodology
  - Pentest lifecycle (Reconnaissance → Scanning → Exploitation → Post-Exploitation → Reporting)

[3] Assessment Components
  - External Penetration Test description
  - Internal Penetration Test / AD attack description

[4] Findings Severity Summary
  - CVSS v3-based severity classification table
  - Total vulnerability count / chart by severity

[5] Scope
  - List of target IPs/domains
  - Exclusions (DoS, Social Engineering, etc.)
  - Client-provided support

[6] Executive Summary
  - Written in non-technical language
  - Testing period and scope
  - Security strengths / weaknesses
  - Vulnerability count chart by severity

[7] Technical Findings
  - Detailed sections per vulnerability (Critical → Info order)

[8] Attack Chain
  - Full attack flow narrative

[9] Appendix
  - List of tools used
  - References
```

###

### [Phase 1: Cover Page and Table of Contents]

The goal is to create a cover page that makes a strong first impression and a navigable table of contents.

**Writing Example:**

```bash
[Cover Page]

Report Title : Internal and External Network Penetration Test
               Findings Report
Client       : Acme Corp
Author       : [Your Name]
Date         : March 5, 2025
Classification: CONFIDENTIAL

--------------------------------------------------------------

[Table of Contents]

Recommended: use the automatic TOC feature in Word / LibreOffice
  - Assign Heading 1, Heading 2 styles to sections, then auto-generate
  - Page numbers are required
  - Update TOC last, after all sections are complete
```

### [Phase 2: Disclaimer]

The goal is to define the test's limitations and scope of liability to prevent legal disputes.

**Writing Example:**

```bash
This report documents the results of a penetration test conducted
from [start date] to [end date].

This test represents a point-in-time snapshot. We accept no
responsibility for vulnerabilities that arise or change after
the conclusion of the testing period.

Only vulnerabilities discovered within the contracted scope are
included in this report. Due to time constraints, we cannot
guarantee that all vulnerabilities have been identified.
```

### [Phase 3: Assessment Overview]

The goal is to explain what penetration testing is and which methodology was followed.

※ Methodology selection is flexible (PTES, OWASP Testing Guide, NIST SP 800-115, etc.), as long as it aligns with the scope and exclusions in the Rules of Engagement document.

**Writing Example:**

```bash
This assessment was conducted based on the NIST SP 800-115
Technical Guide and consists of the following phases:

1. Reconnaissance     - Collect publicly available and target system information
2. Scanning           - Enumerate ports and services
3. Exploitation       - Leverage vulnerabilities to gain initial access
4. Post-Exploitation  - Lateral movement and privilege escalation
5. Reporting          - Document findings and provide remediation guidance
```

### [Phase 4: Assessment Components]

The goal is to clearly state what types of testing were performed.

**Writing Example:**

```bash
This assessment consisted of two components:

External Penetration Test
  Evaluated entry paths from the public internet targeting
  externally accessible services, assessing the feasibility
  of gaining internal network access.

Internal Penetration Test
  Following internal network access, evaluated the Active
  Directory environment for lateral movement, privilege
  escalation, and Domain Controller compromise.
```

### [Phase 5: Findings Severity Summary]

The goal is to present all discovered vulnerabilities at a glance, organized by severity.

※ Severity can be calculated using the CVSS calculator or assigned using your own judgment.

**Writing Example:**

```bash
[CVSS v3 Severity Criteria]

Critical  (9.0 ~ 10.0)  : 2 findings
High      (7.0 ~ 8.9)   : 3 findings
Medium    (4.0 ~ 6.9)   : 2 findings
Low       (0.1 ~ 3.9)   : 1 finding
Info      (0.0)         : 1 finding
------------------------------
Total                   : 9 findings

--------------------------------------------------------------

[Severity Criteria]

Severity is determined using CVSS v3 scoring based on
Likelihood and Impact:

- Likelihood: attack complexity, authentication requirements, public exploit availability
- Impact: confidentiality (C), integrity (I), availability (A) impact
```

※ Inserting a pie chart or bar chart in Word / LibreOffice lets reviewers grasp the picture instantly and earns bonus points.

###

### [Phase 6: Scope]

The goal is to clearly document what was tested and what was excluded.

**Writing Example:**

```bash
[In-Scope]

IP / Domain         Description
-----------------------------------------
10.10.10.0/24      Full internal network
192.168.1.50       External web server
target.com         External domain

--------------------------------------------------------------

[Out-of-Scope]

- DoS / DDoS attacks
- Social Engineering (employee phishing, etc.)
- Physical security testing
- Systems outside the listed IPs/domains

--------------------------------------------------------------

[Client-Provided Support]

- None (black-box test)
or
- VPN credentials provided
- One internal user account provided (grey-box test)
```

---

### [Phase 7: Executive Summary]

The goal is to write a summary that allows a C-level executive with no technical background to understand the full situation in 10 minutes.

**Writing Structure:**

```bash
[Test Overview]

Example:
"At the request of [Company Name], an external and internal network
penetration test was conducted from [date] to [date]. The purpose
of this test was to identify security vulnerabilities in [Company
Name]'s network infrastructure and provide recommendations for
improvement."

--------------------------------------------------------------

[Testing Summary]

- What attacks were performed (without tool names)
- What was discovered (focus on High / Critical)
- What was obtained (Domain Admin access, etc.)

Example:
"During testing, credential vulnerabilities were identified in
external services, enabling access to the internal network. Once
inside, the full Active Directory environment including the Domain
Controller was completely compromised."

--------------------------------------------------------------

[Security Strengths]

- Note what was done well (firewall config, patches applied, etc.)
- If nothing stands out, note that, but don't leave it blank

--------------------------------------------------------------

[Security Weaknesses]

- MFA not implemented
- Weak password policy
- No login failure lockout
- Unpatched systems
- Insufficient internal network segmentation
```

※ Executive Summary must be readable by someone with no technical background.

※ Use phrases like "intercepted network communications" instead of "SMB Relay Attack."

### [Phase 8: Technical Findings]

The goal is to document each discovered vulnerability in a reproducible and verifiable format.

Vulnerabilities must be ordered Critical → High → Medium → Low → Informational.

**Each vulnerability is written in the following format:**

```bash
[Vulnerability Title]  e.g., LLMNR Poisoning
Severity: Critical
CVSS Score: 9.8

--------------------------------------------------------------

[Description]

LLMNR (Link-Local Multicast Name Resolution) is a protocol used to
resolve hostnames on a local network when DNS lookup fails.
An attacker can respond to these requests and capture the victim's
NTLMv2 hash.

--------------------------------------------------------------

[Impact]

An attacker can obtain NTLMv2 hashes from domain users on the
internal network. The captured hashes can be cracked offline to
recover plaintext passwords, or used directly in Pass-the-Hash
attacks to authenticate to other systems without a password.

--------------------------------------------------------------

[Steps to Reproduce]

1. Run Responder on attacker machine
2. Internal network victim attempts to access a non-existent host
3. Responder intercepts the LLMNR query and captures NTLMv2 hash
4. Offline cracking with hashcat

[Screenshot — Responder hash capture]
[Screenshot — hashcat cracking success]

--------------------------------------------------------------

[Remediation]

1. Disable LLMNR via GPO
   Computer Configuration → Administrative Templates
   → Network → DNS Client → Turn off multicast name resolution: Enabled
2. Disable NBT-NS
3. Enforce strong password policy to increase cracking difficulty
```

### [8-2: Screenshot Requirements]

**Required screenshots:**

```bash
1. Initial access achieved
   - Reverse shell capture (with whoami output)

2. Each vulnerability exploit succeeded
   - Hash capture, credential theft, code execution, etc.

3. Internal network access established
   - Internal host access after pivoting

4. Privilege escalation
   - Before/after comparison: standard user → admin/root

5. Domain Controller compromised
   - whoami or hostname on DC
   - Domain Admins group membership confirmed
   - secretsdump or ntds.dit dump success

6. Sensitive data accessed
   - Critical file or credential file access
```

**What makes a good screenshot:**

```bash
- Command and result visible in the same frame
- IP address and hostname visible to identify which system
- Timestamp included (terminal prompt or system clock)
- Sufficient resolution to read all text
```

**Common mistakes:**

```bash
- Screenshots too small to read text
- No way to tell which system the screenshot is from
- Missing "before" screenshot (can't show contrast)
- Taking all screenshots at the end (can't recreate after exam environment closes)
```

※ Candidate Fajao reported failing the first attempt due to insufficient screenshots and missing information. Take far more than you think you need, at every step.

### [Phase 9: Attack Chain]

The goal is to narrate the full attack flow from OSINT to Domain Controller compromise as a single coherent story.

This is the most critically evaluated section in the PNPT exam.

**Writing Structure:**

```bash
[Phase 1 — OSINT]

Narrative:
"OSINT was performed against the target domain (target.com),
collecting the following information:"

Collected:
- Email address list
- Leaked credentials
- Subdomain list
- Employee LinkedIn profiles

[Screenshot — email collection results]
[Screenshot — leaked credentials found]

--------------------------------------------------------------

[Phase 2 — External Penetration]

Narrative:
"Using the collected emails and leaked passwords, a password
spray was attempted against the O365 portal."

- Services found: OWA (10.10.10.1), VPN portal (10.10.10.2)
- Spray result: user@target.com / Password2023! authenticated successfully

[Screenshot — password spray success]
[Screenshot — credentials obtained]

--------------------------------------------------------------

[Phase 3 — Initial Access (Foothold)]

Narrative:
"Using the obtained credentials, VPN access was established
and internal network access was confirmed."

[Screenshot — VPN connection success]
[Screenshot — internal host scan results]

--------------------------------------------------------------

[Phase 4 — Internal Enumeration and AD Attacks]

Narrative:
"Inside the internal network, a domain user hash was captured
via LLMNR Poisoning, and a service account password was obtained
through Kerberoasting."

- LLMNR Poisoning → jdoe NTLMv2 hash captured → cracked → Password1234
- Kerberoasting → svc_sql ticket captured → cracked → SQLService123

[Screenshot — Responder hash collection]
[Screenshot — cracking success]

--------------------------------------------------------------

[Phase 5 — Privilege Escalation and DC Compromise]

Narrative:
"Using the obtained service account credentials, the DC was
accessed and the full domain hash was dumped via DCSync."

[Screenshot — DC access confirmed]
[Screenshot — DCSync / secretsdump results]
[Screenshot — whoami /all or hostname on DC]
[Screenshot — Domain Admins group confirmed]
```

### [Phase 10: Remediation Recommendations]

The goal is to provide specific, actionable fixes for each discovered vulnerability.

**Common vulnerability remediation:**

```bash
[LLMNR / NBT-NS Poisoning]
- Disable LLMNR via GPO
- Disable NBT-NS
- Enforce strong password policy (minimum 12 characters, complexity required)

--------------------------------------------------------------

[Password Spray / Brute-Force]
- Implement account lockout policy (30-minute lockout after 5 failures)
- Enforce MFA (especially required for external-facing services)
- Apply password filter to check against leaked password databases

--------------------------------------------------------------

[Kerberoasting]
- Change service account passwords to 25+ character random strings
- Adopt Group Managed Service Accounts (gMSA)
- Enable AES encryption (disable RC4)

--------------------------------------------------------------

[Pass-the-Hash]
- Deploy LAPS (Local Administrator Password Solution)
- Enable SMB Signing
- Use Protected Users security group

--------------------------------------------------------------

[AS-REP Roasting]
- Enable Kerberos Pre-Authentication
  (Uncheck "Do not require Kerberos preauthentication" in account properties)

--------------------------------------------------------------

[DCSync]
- Remove DCSync permissions (Replicating Directory Changes)
  from any account outside Domain Admins and Enterprise Admins
- Monitor for changes to this permission (Event ID 4662)

--------------------------------------------------------------

[Vulnerable Software Versions]
- Establish a regular patch and update management process

--------------------------------------------------------------

[Insufficient Internal Network Segmentation]
- Segment VLANs for user / server / management networks
- Apply Principle of Least Privilege
```

### [Phase 11: Appendix]

The goal is to organize supplemental reference information that is useful but too detailed for the main body.

**Writing Example:**

```bash
[Tools Used]

Tool              Purpose
-----------------------------------------
Nmap              Port scanning and service enumeration
Responder         LLMNR/NBT-NS poisoning
Hashcat           Offline password cracking
CrackMapExec      SMB enumeration and credential validation
BloodHound        AD privilege relationship visualization
Impacket          DCSync, Kerberoasting
Ligolo-ng         Pivoting tunnel

--------------------------------------------------------------

[References]

- NIST SP 800-115: Technical Guide to Information Security Testing
- MITRE ATT&CK Framework: https://attack.mitre.org
- TCM Security Sample Report: https://github.com/hmaverickadams/TCM-Security-Sample-Pentest-Report
```

### [Phase 12: Debrief Preparation]

The goal is to present the attack flow to a TCM Security reviewer for 15 minutes based on the submitted report.

**Preparation:**

```bash
[Presentation Method]
  - Screen share the report directly (most passing candidates use this)
  - Prepare a separate PowerPoint (optional, not required)

--------------------------------------------------------------

[Presentation Order (15 minutes)]
  - Brief self-introduction (1 minute)
  - Executive Summary recap (2 minutes)
  - Top 3-5 key findings summary (5 minutes)
  - Full Attack Chain walkthrough (5 minutes)
  - Key remediation recommendations summary (2 minutes)

--------------------------------------------------------------

[Common Reviewer Questions]
  - "How did you discover this vulnerability?"
  - "Why did you classify this as Critical?"
  - "What would a real attacker do next?"
  - "Which remediation should be prioritized?"
```

**Important notes:**

```bash
- Findings without screenshots are hard to explain
  (This is where the importance of in-exam screenshots comes back)
- The reviewer already knows all the techniques —
  focus on "why this is a problem" rather than technical explanations
- If you don't know the answer to a question, say so honestly
```

### [Core Report Writing Principles]

```bash
[Absolute Rules]

1. Take screenshots immediately at every stage during the exam
   Backfilling later is impossible. Take them now.

2. Write each vulnerability so it can be reproduced
   The reviewer must be able to follow along and replicate it.

3. Executive Summary in non-technical language
   "Captured user authentication credentials via network interception"
   NOT "Stole NTLMv2 hash using Responder"

4. Include a remediation recommendation for every vulnerability
   Don't write vague statements like "apply patch"
   Specify exact GPO settings, service disablement steps, etc.

5. Strongly recommended: use the TCM Security sample template
   https://github.com/hmaverickadams/TCM-Security-Sample-Pentest-Report

--------------------------------------------------------------

[Recommended Tools]

Report writing : Microsoft Word / LibreOffice Writer (.docx)
Note-taking    : CherryTree / Obsidian / KeepNote
Screenshots    : Flameshot (Linux) / Greenshot (Windows)

--------------------------------------------------------------

[In-Exam Note Template]

## [Hostname / IP]
- Discovery time:
- Services:
- Vulnerability:
- Commands used:
- Result:
- Screenshot filename:
- Remediation:
```

※ Even if you technically compromise everything perfectly, a low-quality report means a failing grade. Dedicate both full exam days to the report.

→This is a sample report simply created with Claude.

[Download PNPT Sample Report_EN](https://github.com/jaejun835/hacking-notes/blob/main/PNPT_Pentest_Report_EN.pdf)

[Download PNPT Sample Report_KR](https://github.com/jaejun835/hacking-notes/blob/main/PNPT_%EC%B9%A8%ED%88%AC%ED%85%8C%EC%8A%A4%ED%8A%B8_%EB%B3%B4%EA%B3%A0%EC%84%9C_KR.pdf)

