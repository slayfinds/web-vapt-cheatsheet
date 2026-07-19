<div align="center">

# `RECON // ATTACK SURFACE DISCOVERY`

### WEB VAPT FIELD NOTES

`DISCOVER` → `PROBE` → `CRAWL` → `ENUMERATE` → `ANALYZE`

<br>

**Understand what is exposed before deciding what to test.**

</div>

---

<div align="center">

[← HOME](README.md) &nbsp; • &nbsp; **RECON** &nbsp; • &nbsp; API &nbsp; • &nbsp; AUTH &nbsp; • &nbsp; ACCESS CONTROL &nbsp; • &nbsp; CORS &nbsp; • &nbsp; HEADERS

</div>

---

## `// WHAT IS RECON?`

Reconnaissance is the process of discovering and mapping the externally accessible attack surface of an authorized target.

Before testing for vulnerabilities, we first want to understand:

```text
What domains exist?
        ↓
Which hosts are reachable?
        ↓
What technologies are running?
        ↓
What pages and endpoints exist?
        ↓
Are APIs exposed?
        ↓
What parameters accept user input?
        ↓
Which areas deserve deeper testing?
```

Recon is not about blindly running as many tools as possible.

The goal is to collect information, understand the application, and use each result to decide what to investigate next.

> **Important**
>
> Use these commands only on systems you own or have explicit authorization to test. Always follow the applicable scope and rules of engagement.

---

# `// BEFORE YOU START`

Throughout this guide, two environment variables are used.

### `$DOMAIN`

The root domain being assessed.

```bash
export DOMAIN="target.com"
```

### `$TARGET`

A specific application or host you want to investigate.

```bash
export TARGET="https://app.target.com"
```

Think of it like this:

```text
DOMAIN
│
└── target.com
      │
      ├── www.target.com
      ├── api.target.com
      ├── app.target.com    ← TARGET
      └── staging.target.com
```

`$DOMAIN` is generally used for broad discovery.

`$TARGET` is used when investigating a specific application.

---

# `01 // CREATE A DEDICATED WORKSPACE`

### What are we doing?

Creating a separate directory for each authorized assessment.

This prevents output files from one target from overwriting results collected for another target.

### Run

```bash
mkdir -p ~/vapt/target.com
cd ~/vapt/target.com
```

Set the target domain:

```bash
export DOMAIN="target.com"
```

Verify:

```bash
echo $DOMAIN
pwd
```

### Example Structure

```text
vapt/
│
├── target-one.com/
│   ├── subdomains.txt
│   ├── live.txt
│   ├── endpoints.txt
│   └── security_headers.txt
│
└── target-two.com/
    ├── subdomains.txt
    ├── live.txt
    ├── endpoints.txt
    └── security_headers.txt
```

> **Field Note**
>
> Always confirm that you are inside the correct target directory before starting an assessment.

### Next

Discover subdomains associated with the target.

---

# `02 // SUBDOMAIN ENUMERATION`

### What are we doing?

Discovering subdomains associated with the target domain.

A company may have more than just:

```text
www.target.com
```

It may also have:

```text
api.target.com
dev.target.com
staging.target.com
admin.target.com
```

These systems can represent different parts of the authorized attack surface.

### Run

Discover subdomains using Subfinder:

```bash
subfinder -d "$DOMAIN" -silent | sort -u > subdomains.txt
```

Check results:

```bash
wc -l subdomains.txt
cat subdomains.txt
```

### What to look for

Pay attention to names that may indicate:

```text
api
dev
test
qa
stage
stg
sandbox
admin
auth
portal
vpn
```

A subdomain name alone is **not a vulnerability**.

It simply helps us understand the target's infrastructure.

### Next

Check which discovered hosts are reachable.

---

# `03 // IDENTIFY LIVE HOSTS`

### What are we doing?

Probing discovered subdomains to identify reachable web applications.

### Run

```bash
httpx -l subdomains.txt \
-silent \
-status-code \
-title \
-tech-detect \
-follow-redirects \
-o live.txt
```

View results:

```bash
cat live.txt
```

Extract only URLs:

```bash
awk '{print $1}' live.txt | sort -u > liveurls.txt
```

### How to read the results

A result may look similar to:

```text
https://app.target.com     [200] [Application]
https://api.target.com     [401] [API]
https://admin.target.com   [403] [Forbidden]
```

Common status codes:

| Status | General Meaning |
|:---:|---|
| `200` | Request succeeded |
| `301 / 302` | Redirect |
| `401` | Authentication required |
| `403` | Request understood but access denied |
| `404` | Resource not found |
| `405` | HTTP method not allowed |
| `429` | Too many requests / rate limiting |
| `500` | Server-side error |

> **Field Note**
>
> A `200` response is not automatically a vulnerability, and a `403` response does not automatically mean an application is secure. Status codes are clues that help determine what to investigate next.

### Next

Prioritize potentially interesting hosts.

---

# `04 // IDENTIFY INTERESTING SUBDOMAINS`

### What are we doing?

Filtering the discovered subdomains for names commonly associated with development, staging, authentication, APIs, and administrative systems.

### Run

```bash
grep -Ei 'api|dev|test|qa|stage|stg|sandbox|internal|admin|auth|login|portal|vpn|jenkins|jira|git|nexus' \
subdomains.txt | sort -u > interesting_subdomains.txt
```

View results:

```bash
cat interesting_subdomains.txt
```

### What to look for

Potentially interesting naming patterns include:

```text
api.target.com
dev.target.com
qa.target.com
staging.target.com
auth.target.com
portal.target.com
```

> **Remember**
>
> "Interesting" does not mean "vulnerable." Always verify that the host is explicitly within the authorized testing scope before investigating it.

### Next

Select an authorized host and map its application routes.

---

# `05 // CRAWL A SELECTED HOST`

### What are we doing?

Crawling a specific application to discover pages, routes, JavaScript files, and endpoints.

Set the host you want to investigate:

```bash
export TARGET="https://app.target.com"
```

### Run

Crawl using Katana:

```bash
katana -u "$TARGET" \
-d 3 \
-js-crawl \
-known-files all \
-silent \
-o endpoints.txt
```

Remove duplicates:

```bash
sort -u endpoints.txt > endpoints_unique.txt
```

Check endpoint count:

```bash
wc -l endpoints_unique.txt
```

### What to look for

The crawler may discover:

```text
/login
/api/v1/users
/account/profile
/password/reset
/assets/app.js
/download/report
```

Review discovered paths before performing additional testing.

### Next

Filter the collected endpoints for security-relevant functionality.

---

# `06 // FIND INTERESTING ENDPOINTS`

### What are we doing?

Filtering discovered URLs to identify paths that may handle sensitive or security-relevant functionality.

### Run

```bash
grep -Ei 'api|login|logout|forgot|reset|password|user|users|account|admin|auth|profile|member|customer|employee|upload|download|document|file|report|search|config|token|payment|invoice' \
endpoints_unique.txt \
| sort -u > interesting_endpoints.txt
```

View results:

```bash
cat interesting_endpoints.txt
```

### What to look for

Examples include functionality related to:

- Authentication
- Password recovery
- User accounts
- Administrative interfaces
- File uploads
- File downloads
- Documents
- Reports
- Payments
- API endpoints

Finding an endpoint does not mean it is vulnerable.

The purpose is to identify areas that may deserve further authorized testing.

### Next

Check how the discovered endpoints respond.

---

# `07 // PROBE DISCOVERED ENDPOINTS`

### What are we doing?

Checking endpoint response metadata including status codes, content types, response sizes, redirects, and page titles.

### Run

```bash
httpx -l interesting_endpoints.txt \
-silent \
-status-code \
-content-type \
-content-length \
-location \
-title \
-o endpoint_results.txt
```

View results:

```bash
cat endpoint_results.txt
```

### What to look for

Review differences in:

```text
Status Code
Content Type
Content Length
Redirect Location
Page Title
```

Unexpected differences can help identify endpoints worth manually reviewing.

> **Field Note**
>
> Response size can be useful when comparing multiple endpoints. Two requests returning the same status code may still return completely different content.

---

# `08 // COLLECT HISTORICAL URLS`

### What are we doing?

Collecting URLs previously observed by public URL archives and indexing sources.

Historical URLs may reveal older application routes that are not easily discovered through current crawling.

### Using gau

```bash
gau "$DOMAIN" \
| sort -u > gau_urls.txt
```

### Using Waybackurls

```bash
echo "$DOMAIN" \
| waybackurls \
| sort -u > wayback_urls.txt
```

Combine results:

```bash
cat endpoints_unique.txt gau_urls.txt wayback_urls.txt 2>/dev/null \
| sort -u > all_endpoints.txt
```

Check total:

```bash
wc -l all_endpoints.txt
```

### What to look for

Historical results may contain:

```text
Old API routes
Legacy pages
Deprecated endpoints
Old parameters
JavaScript files
Previous application paths
```

> Historical URLs are leads, not proof that the resources still exist. Validate them against the current authorized target before drawing conclusions.

---

# `09 // FIND PARAMETERIZED URLS`

### What are we doing?

Finding URLs containing query parameters.

Parameters represent user-controlled input and may become useful during later manual security testing.

### Run

```bash
grep '=' all_endpoints.txt \
| sort -u > parameterized_urls.txt
```

View sample:

```bash
head -50 parameterized_urls.txt
```

### Example

```text
https://target.com/search?q=value
https://target.com/product?id=123
```

### What to look for

Identify parameters related to:

```text
IDs
Search values
File names
Redirects
User references
Object references
Filters
```

At this stage, we are identifying input points—not assuming they are vulnerable.

---

# `10 // FIND API ENDPOINTS`

### What are we doing?

Searching the collected attack surface for URLs that appear to expose APIs or API documentation.

### Run

```bash
grep -Ei '/api/|/api$|graphql|swagger|openapi' \
all_endpoints.txt \
| sort -u > api_endpoints.txt
```

Check results:

```bash
wc -l api_endpoints.txt
head -50 api_endpoints.txt
```

### What to look for

Common patterns include:

```text
/api/
/api/v1/
/graphql
/swagger
/openapi
```

API endpoints can later be assessed for authentication, authorization, object-level access control, input validation, and other API-specific security controls.

---

# `11 // FIND HIGH-VALUE PATHS`

### What are we doing?

Filtering the collected URL set for functionality that may handle sensitive operations or information.

### Run

```bash
grep -Ei 'admin|login|auth|account|user|employee|customer|password|reset|forgot|upload|download|document|report|payment|invoice|token|config|internal|debug|backup' \
all_endpoints.txt \
| sort -u > high_value_endpoints.txt
```

View results:

```bash
head -100 high_value_endpoints.txt
```

### What to look for

Prioritize paths associated with:

- Authentication
- Account management
- Administrative functionality
- Password recovery
- File handling
- Documents
- Reports
- Configuration
- Debug functionality

Always manually validate the endpoint and its actual behavior.

---

# `12 // INSPECT JAVASCRIPT FILES`

### What are we doing?

Reviewing client-side JavaScript for application routes and API references.

JavaScript used by web applications may contain references to endpoints that were not discovered during normal crawling.

### Find JavaScript URLs

```bash
grep -Ei '\.js([?#]|$)' all_endpoints.txt \
| sort -u > javascript_files.txt
```

View:

```bash
head -50 javascript_files.txt
```

For a specific JavaScript file:

```bash
curl -sk "$TARGET/path/to/app.js" -o app.js
```

Extract API-style paths:

```bash
grep -Eoi '(/api/[A-Za-z0-9_?&=./{}:-]+)' app.js \
| sort -u > js_api_paths.txt
```

View:

```bash
cat js_api_paths.txt
```

### What to look for

JavaScript may reference:

```text
API routes
Authentication endpoints
Application paths
Feature flags
Service URLs
```

> **Important**
>
> A string found inside JavaScript is not automatically an active endpoint. Verify whether the route exists and is within scope.

---

# `13 // INSPECT PAGE SOURCE`

### What are we doing?

Reviewing the application's HTML source for forms, scripts, API references, and framework-specific endpoints.

### Download a page

```bash
curl -sk "$TARGET" -o page.html
```

Search for interesting references:

```bash
grep -Ein \
'action=|href=|src=|api|login|forgot|reset|password|ajax|graphql|swagger|webmethod|\.aspx|\.ashx|\.asmx' \
page.html \
| tee page_interesting.txt
```

### What to look for

Look for references to:

```text
Form actions
JavaScript sources
API endpoints
Login functionality
Password reset functionality
AJAX calls
GraphQL
Swagger
ASP.NET endpoints
```

The source code can help explain how the frontend communicates with backend services.

---

# `14 // SECURITY HEADER CHECK`

### What are we doing?

Reviewing HTTP response headers for browser security controls and information disclosure.

### Run

```bash
curl -skI "$TARGET" \
| tee security_headers.txt
```

### Review headers such as

| Header | Purpose |
|---|---|
| `Content-Security-Policy` | Controls allowed content sources |
| `Strict-Transport-Security` | Enforces HTTPS usage |
| `X-Content-Type-Options` | Helps prevent MIME-type sniffing |
| `X-Frame-Options` | Helps protect against clickjacking |
| `Referrer-Policy` | Controls referrer information |
| `Permissions-Policy` | Controls browser features |
| `Access-Control-Allow-Origin` | Defines allowed cross-origin access |
| `Server` | May reveal server technology |
| `X-Powered-By` | May reveal application technology |

> Missing headers or technology disclosure should be assessed in context. Not every missing header is automatically a reportable vulnerability.

---

# `15 // HTTP VS HTTPS CHECK`

### What are we doing?

Checking how the target handles unencrypted HTTP connections.

### Run

```bash
curl -v --max-time 10 \
"http://$DOMAIN/" \
-o /dev/null 2>&1 \
| tee http_check.txt
```

### What to look for

Determine whether:

```text
HTTP redirects to HTTPS
HTTP remains accessible
Port 80 is unavailable
```

The expected behavior depends on the application's architecture and deployment.

---

# `16 // COMMON EXPOSURE CHECKS`

### What are we doing?

Checking a small set of commonly known diagnostic, metadata, and configuration-related paths.

### Create the list

```bash
cat > exposure_checks.txt <<EOF
$TARGET/robots.txt
$TARGET/sitemap.xml
$TARGET/.well-known/security.txt
$TARGET/web.config
$TARGET/Web.config
$TARGET/trace.axd
$TARGET/elmah.axd
$TARGET/Global.asax
$TARGET/App_Data/
$TARGET/bin/
EOF
```

### Probe them

```bash
httpx -l exposure_checks.txt \
-silent \
-status-code \
-content-type \
-content-length \
-location \
-title \
-o exposure_results.txt
```

View:

```bash
cat exposure_results.txt
```

### How to interpret the result

A `200` response does not automatically mean a vulnerability.

Manually verify:

```text
What content was returned?
Is the information sensitive?
Is authentication required?
Was actual configuration data exposed?
Does the exposure create security impact?
```

A redirect to a login page may indicate that the resource is protected rather than publicly accessible.

---

# `17 // CORS HEADER CHECK`

### What are we doing?

Performing an initial review of how the application responds to cross-origin requests.

### Basic Origin Check

```bash
curl -ski "$TARGET" \
-H 'Origin: https://example.com' \
-o /dev/null \
-D - \
| grep -iE 'HTTP/|access-control|vary:'
```

For an authorized API endpoint, inspect preflight behavior:

```bash
curl -ski -X OPTIONS \
"$TARGET/api/endpoint" \
-H 'Origin: https://example.com' \
-H 'Access-Control-Request-Method: POST' \
-H 'Access-Control-Request-Headers: content-type'
```

### Look for

```text
Access-Control-Allow-Origin
Access-Control-Allow-Credentials
Access-Control-Allow-Methods
Access-Control-Allow-Headers
Vary: Origin
```

> `Access-Control-Allow-Origin: *` alone is not necessarily a vulnerability.
>
> Determine whether sensitive authenticated data can actually be accessed by an unauthorized origin before considering the behavior security-relevant.

---

# `18 // HTTP METHOD CHECK`

### What are we doing?

Checking how an authorized endpoint responds to different HTTP request methods.

Set the endpoint:

```bash
export ENDPOINT="$TARGET/api/endpoint"
```

### Run

```bash
for method in GET HEAD OPTIONS POST PUT PATCH DELETE; do
  curl -sk --max-time 10 \
    -X "$method" \
    -o /dev/null \
    -w "$method -> HTTP %{http_code} | Type: %{content_type} | Size: %{size_download}\n" \
    "$ENDPOINT"
done
```

### Example

```text
GET     -> HTTP 200
HEAD    -> HTTP 200
OPTIONS -> HTTP 200
POST    -> HTTP 405
PUT     -> HTTP 405
PATCH   -> HTTP 405
DELETE  -> HTTP 405
```

### How to interpret the result

An allowed HTTP method is not automatically a vulnerability.

Verify whether the method allows:

```text
Unauthorized data access
Unauthorized resource creation
Unauthorized modification
Unauthorized deletion
```

The security impact depends on what the endpoint actually allows the requester to do.

---

# `19 // TECHNOLOGY FINGERPRINTING`

### What are we doing?

Identifying technologies that appear to be used by the application.

### Run

```bash
httpx -u "$TARGET" \
-silent \
-status-code \
-title \
-tech-detect
```

### What to look for

Technology fingerprinting may identify:

```text
Web servers
Application frameworks
JavaScript frameworks
CDNs
WAFs
Reverse proxies
```

Technology disclosure can help understand the application stack.

Version information alone is generally informational unless it contributes to a demonstrable security impact.

---

# `20 // CONTENT DISCOVERY WITH FEROXBUSTER`

### What are we doing?

Discovering directories, files, and application routes that may not be linked directly from the application's visible pages.

### Run

```bash
feroxbuster -u "$TARGET" \
-d 2 \
-o ferox_results.txt
```

### What to look for

Potentially interesting discovered paths may include:

```text
/admin
/api
/login
/uploads
/downloads
/backup
/debug
```

> **Field Note**
>
> A discovered path is not automatically a vulnerability. Confirm that the path is within scope and manually review its actual content and access controls.
>
> Directory discovery can generate significant traffic. Always follow the target's rules of engagement and permitted request-rate limits.

---

# `21 // TEMPLATE-BASED CHECKS WITH NUCLEI`

### What are we doing?

Running template-based checks against authorized live targets to identify potential exposures, misconfigurations, and known security issues that deserve manual investigation.

### Run

```bash
nuclei -l liveurls.txt \
-severity info,low,medium,high,critical \
-o nuclei_results.txt
```

For a specific target:

```bash
nuclei -u "$TARGET" \
-severity info,low,medium,high,critical \
-o nuclei_target_results.txt
```

### Review results

```bash
cat nuclei_results.txt
```

### What to look for

Nuclei may identify potential:

```text
Exposed panels
Security misconfigurations
Information disclosure
Known vulnerable components
Publicly accessible files
Missing security controls
```

> **Important**
>
> A Nuclei match is not automatically a confirmed vulnerability.
>
> Manually reproduce and validate every result before reporting it. Also review the program's rules before running automated scanners, because some authorized testing programs restrict automation or request rates.


---

# `22 // QUICK RESULTS REVIEW`

### What are we doing?

Reviewing the files generated during reconnaissance and quickly identifying potentially interesting responses.

### List generated files

```bash
ls -lh
```

Search collected results:

```bash
grep -Ei '\[200\]|\[201\]|\[204\]|\[301\]|\[302\]|\[401\]|\[403\]' \
*.txt 2>/dev/null
```

> This is a quick triage step. Always return to the original output and manually review interesting results.

---

# `// RECON WORKFLOW`

```text
┌─────────────────────────────────────┐
│  01. CONFIRM AUTHORIZATION & SCOPE  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  02. CREATE TARGET WORKSPACE        │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  03. ENUMERATE SUBDOMAINS           │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  04. IDENTIFY LIVE HOSTS            │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  05. PRIORITIZE INTERESTING HOSTS   │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  06. CRAWL SELECTED APPLICATIONS    │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  07. COLLECT HISTORICAL URLS        │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  08. FIND APIs & HIGH-VALUE PATHS   │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  09. INSPECT JAVASCRIPT & SOURCE    │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  10. REVIEW INITIAL SECURITY        │
│      CONFIGURATION                  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  11. MANUALLY VALIDATE RESULTS      │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  12. DOCUMENT EVIDENCE & IMPACT     │
└─────────────────────────────────────┘
```

---

# `// IMPORTANT TESTING PRINCIPLE`

<div align="center">

### `AN INTERESTING RESPONSE ≠ A VULNERABILITY`

</div>

A security misconfiguration or unusual response is not automatically a reportable vulnerability.

Always validate:

- **What is exposed?**
- **Is authentication required?**
- **Can unauthorized users access sensitive information?**
- **Can the behavior be exploited?**
- **What is the real security impact?**
- **Is the affected asset explicitly in scope?**

For every confirmed finding, document:

```text
Affected Asset
      ↓
Endpoint
      ↓
Request
      ↓
Response
      ↓
Evidence
      ↓
Security Impact
      ↓
Recommended Remediation
```

---

# `// TOOLBOX`

| Tool | Primary Use |
|---|---|
| `subfinder` | Subdomain discovery |
| `dnsx` | DNS resolution and validation |
| `httpx` | HTTP probing and fingerprinting |
| `naabu` | Port discovery |
| `katana` | Web crawling |
| `gau` | Historical URL collection |
| `waybackurls` | Archived URL discovery |
| `feroxbuster` | Content and directory discovery |
| `ffuf` | Content and parameter fuzzing |
| `nuclei` | Template-based security checks |
| `wafw00f` | WAF detection and fingerprinting |
| `gowitness` | Web application screenshots |
| `uro` | URL cleaning and deduplication |
| `unfurl` | URL parsing and analysis |
| `gf` | Pattern-based URL filtering |
| `anew` | Append new and unique results |
| `curl` | Manual HTTP requests |
| `grep` | Filtering collected data |
| `awk` | Text processing |
| `sort` | Sorting and deduplication |

---

# `// FINAL NOTES`

Keep every assessment in a dedicated directory:

```text
vapt/
├── target-one.com/
│   ├── subdomains.txt
│   ├── live.txt
│   ├── endpoints.txt
│   └── security_headers.txt
│
└── target-two.com/
    ├── subdomains.txt
    ├── live.txt
    ├── endpoints.txt
    └── security_headers.txt
```

This prevents results from one target from overwriting another.

Most importantly, do not run commands simply because they appear in a checklist.

Use the results from each stage to decide what makes sense to investigate next.

---

<div align="center">

### `DISCOVER` ── `UNDERSTAND` ── `ENUMERATE` ── `ANALYZE` ── `VALIDATE`

<br>

**For authorized security testing and educational purposes only.**

Only test systems you own or have explicit permission to assess.  
Always follow the defined scope and rules of engagement.

<br>

[← BACK TO HOME](README.md)

<br>

`slayfinds // web application security field notes`

</div>
