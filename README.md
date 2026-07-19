<div align="center">

```
██╗    ██╗███████╗██████╗     ██╗   ██╗ █████╗ ██████╗ ████████╗
██║    ██║██╔════╝██╔══██╗    ██║   ██║██╔══██╗██╔══██╗╚══██╔══╝
██║ █╗ ██║█████╗  ██████╔╝    ██║   ██║███████║██████╔╝   ██║
██║███╗██║██╔══╝  ██╔══██╗    ╚██╗ ██╔╝██╔══██║██╔═══╝    ██║
╚███╔███╔╝███████╗██████╔╝     ╚████╔╝ ██║  ██║██║        ██║
 ╚══╝╚══╝ ╚══════╝╚═════╝       ╚═══╝  ╚═╝  ╚═╝╚═╝        ╚═╝
```

### WEB APPLICATION SECURITY FIELD NOTES

`recon` · `enumeration` · `web` · `api` · `appsec`

</div>

---

> A collection of commands and testing notes I've put together while working on web application security assessments and labs.

```text
$ tree

web-vapt-cheatsheet
│
├── recon.md
├── api-vapt.md
├── authentication-testing.md
├── authorization-testing.md
├── cors-testing.md
├── security-headers.md
└── burp-suite.md
```

## INDEX

**01 // RECON**

[Web Reconnaissance →](recon.md)

Subdomain discovery, live host identification, crawling, endpoint enumeration, JavaScript analysis and attack-surface mapping.

---

**02 // API**

`work in progress`

API discovery and manual API security testing notes.

---

**03 // AUTH**

`work in progress`

Authentication, session management and authorization testing.

---

**04 // WEB CONFIG**

`work in progress`

CORS, HTTP methods, security headers and common web misconfigurations.

---

**05 // BURP**

`work in progress`

Burp Suite workflow and manual testing notes.

---

## WORKFLOW

```text
[ SCOPE ]
    │
    ├── RECON
    │     ├── subdomains
    │     ├── live hosts
    │     └── technologies
    │
    ├── ENUMERATION
    │     ├── endpoints
    │     ├── parameters
    │     ├── javascript
    │     └── APIs
    │
    ├── TESTING
    │     ├── authentication
    │     ├── authorization
    │     ├── input validation
    │     └── business logic
    │
    └── VALIDATION
          ├── reproduce
          ├── determine impact
          └── document
```

---

### NOTE

These are personal field notes and quick-reference commands, not a replacement for a complete testing methodology.

Use only against systems you own or have explicit authorization to test.
