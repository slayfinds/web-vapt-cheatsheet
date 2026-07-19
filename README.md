<div align="center">

# `WEB // VAPT`

### WEB APPLICATION SECURITY FIELD NOTES

`RECON` &nbsp;•&nbsp; `API` &nbsp;•&nbsp; `AUTH` &nbsp;•&nbsp; `ACCESS CONTROL` &nbsp;•&nbsp; `CORS` &nbsp;•&nbsp; `WEB`

<br>

> **Learn the methodology. Understand the commands. Know what to look for.**

<br>

A practical collection of **Web Application Vulnerability Assessment and Penetration Testing (VAPT)**  
commands, workflows, methodologies, and field notes.

Built to be useful as both a **beginner learning resource** and a **quick reference during authorized security assessments**.

</div>

---

## `// START HERE`

New to Web VAPT?

The basic idea is simple:

```text
                    ┌──────────────────┐
                    │      SCOPE       │
                    │ What can I test? │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │      RECON       │
                    │ What is exposed? │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │   ENUMERATION    │
                    │ What is running? │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ ATTACK SURFACE   │
                    │ What can I test? │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ SECURITY TESTING │
                    │ Is it vulnerable?│
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │    VALIDATION    │
                    │ What is impact?  │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │    REPORTING     │
                    │ Document it.     │
                    └──────────────────┘
```

The goal is not to run every tool or command you know.

The goal is to **understand the application**, identify its attack surface, test relevant security controls, validate potential vulnerabilities, and document findings clearly.

---

# `// FIELD NOTES`

Each guide explains:

`WHAT IT IS` → `WHY WE TEST IT` → `HOW TO TEST` → `READ THE OUTPUT` → `WHAT TO LOOK FOR` → `NEXT STEP`

<br>

<table>
<tr>
<td width="50%" valign="top">

### `01 // RECON`

#### Attack Surface Discovery

Start here to discover what systems and applications are exposed.

Learn:

- Subdomain discovery
- Live host identification
- Technology fingerprinting
- URL collection
- Web crawling
- Endpoint discovery
- JavaScript analysis
- Interesting asset identification

<br>

**[OPEN RECON FIELD NOTES →](Recon.md)**

</td>

<td width="50%" valign="top">

### `02 // API`

#### API Security Testing

Understand how to discover and assess application APIs.

Learn:

- API endpoint discovery
- HTTP method testing
- Authentication checks
- Authorization testing
- BOLA / IDOR
- Parameter analysis
- Response analysis
- Rate-limit testing

<br>

`STATUS // COMING SOON`

</td>
</tr>

<tr>
<td width="50%" valign="top">

### `03 // AUTH`

#### Authentication Testing

Test how an application verifies user identity.

Learn:

- Login mechanisms
- Account enumeration
- Password policies
- Password reset flows
- MFA implementation
- Session handling
- Cookie security

<br>

`STATUS // COMING SOON`

</td>

<td width="50%" valign="top">

### `04 // ACCESS CONTROL`

#### Authorization Testing

Check whether users can access resources or actions they should not be able to access.

Learn:

- IDOR / BOLA
- Horizontal access control
- Vertical access control
- Role validation
- Privilege escalation
- Object-level authorization

<br>

`STATUS // COMING SOON`

</td>
</tr>

<tr>
<td width="50%" valign="top">

### `05 // CORS`

#### Cross-Origin Security

Understand how applications control requests coming from other origins.

Learn:

- CORS basics
- Origin validation
- Preflight requests
- Credential handling
- Wildcard origins
- CORS misconfigurations

<br>

`STATUS // COMING SOON`

</td>

<td width="50%" valign="top">

### `06 // HEADERS`

#### HTTP Security Headers

Review HTTP response headers and browser-side security controls.

Learn:

- Content Security Policy
- HSTS
- Clickjacking protection
- Content-Type protection
- Referrer Policy
- Permissions Policy

<br>

`STATUS // COMING SOON`

</td>
</tr>
</table>

---

# `// HOW THE GUIDES WORK`

Every guide follows the same structure so you know exactly what you're doing before running a command.

### `01` — Understand

> What is this security check?

A simple explanation of the concept.

### `02` — Purpose

> Why are we checking this?

Understand what security issue or attack surface the test helps identify.

### `03` — Run

```bash
# Example command
security-tool -d target.com
```

### `04` — Read the Output

```text
[200]  https://app.target.com
[401]  https://api.target.com
[403]  https://admin.target.com
```

Understand what the result actually means.

### `05` — Look For

Identify interesting behavior, unexpected exposure, or potential security weaknesses.

### `06` — Next Step

Use the result from one test to decide what to investigate next.

---

# `// CORE TOOLBOX`

| Purpose | Tools |
|---|---|
| Subdomain Discovery | `subfinder` `assetfinder` `amass` |
| Live Host Discovery | `httpx` |
| Web Crawling | `katana` `hakrawler` |
| Historical URLs | `gau` `waybackurls` |
| Port Scanning | `nmap` `naabu` |
| HTTP Testing | `curl` `httpx` |
| Vulnerability Scanning | `nuclei` |
| DNS Analysis | `dig` `dnsx` |
| Manual Web Testing | `Burp Suite` |
| API Testing | `Burp Suite` `curl` |

> Tools help automate parts of the assessment. They do not replace manual analysis or understanding of the application's behavior.

---

# `// HTTP STATUS QUICK REFERENCE`

| Status | What it usually means |
|:---:|---|
| `200` | Request succeeded |
| `201` | Resource was created |
| `301 / 302` | Redirect |
| `400` | Invalid request |
| `401` | Authentication required |
| `403` | Request understood but access denied |
| `404` | Resource not found |
| `405` | HTTP method not allowed |
| `429` | Too many requests / rate limiting |
| `500` | Server-side error |
| `502 / 503` | Upstream or service availability issue |

> HTTP status codes are clues, not vulnerabilities by themselves. Always examine the response body, headers, authentication state, and application behavior.

---

# `// TESTING MINDSET`

```text
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│   Don't ask:  "Which command should I run next?"              │
│                                                               │
│   Ask:        "What did I learn from the last result?"        │
│                                                               │
│               "What security assumption can I test next?"     │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

Good security testing is a process of forming and validating hypotheses.

```text
DISCOVER
    │
    ▼
UNDERSTAND
    │
    ▼
QUESTION
    │
    ▼
TEST
    │
    ▼
VALIDATE
    │
    ▼
DOCUMENT
```

---

# `// REPOSITORY MAP`

```text
web-vapt-cheatsheet/
│
├── README.md
│      └── You are here
│
├── Recon.md
│      └── Reconnaissance & attack-surface discovery
│
├── API-VAPT.md
│      └── API security testing
│
├── Authentication.md
│      └── Authentication security
│
├── Authorization.md
│      └── Access-control testing
│
├── CORS.md
│      └── Cross-origin security
│
├── Security-Headers.md
│      └── HTTP security headers
│
└── Burp-Suite.md
       └── Manual testing workflows
```

---

<div align="center">

### `DISCOVER` ── `UNDERSTAND` ── `TEST` ── `VALIDATE` ── `DOCUMENT`

<br>

This repository is intended for **educational purposes and authorized security testing only**.

Only test systems you own or have explicit permission to assess.  
Always follow the defined scope, rules of engagement, and applicable laws.

<br>

`slayfinds // web application security field notes`

</div>
