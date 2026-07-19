# Web VAPT Cheatsheet

A practical command-line cheatsheet for web application security testing, reconnaissance, endpoint discovery, and basic security validation.

> **Disclaimer:** Use these commands only on systems you own or have explicit authorization to test. Always follow the applicable scope and rules of engagement.

---

## 1. Create a Dedicated Workspace

Always create a separate folder for each target to prevent output files from different assessments from being overwritten.

```bash
mkdir -p ~/vapt/target.com
cd ~/vapt/target.com
```

Set the target:

```bash
export DOMAIN="target.com"
```

Verify:

```bash
echo $DOMAIN
pwd
```

---

## 2. Subdomain Enumeration

Discover subdomains using Subfinder:

```bash
subfinder -d "$DOMAIN" -silent | sort -u > subdomains.txt
```

Check results:

```bash
wc -l subdomains.txt
cat subdomains.txt
```

---

## 3. Identify Live Hosts

Probe discovered subdomains:

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

---

## 4. Identify Interesting Subdomains

Search for development, staging, API, authentication, and administrative systems:

```bash
grep -Ei 'api|dev|test|qa|stage|stg|sandbox|internal|admin|auth|login|portal|vpn|jenkins|jira|git|nexus' \
subdomains.txt | sort -u > interesting_subdomains.txt
```

View results:

```bash
cat interesting_subdomains.txt
```

---

## 5. Crawl a Selected Host

Set the host you want to investigate:

```bash
export TARGET="https://app.target.com"
```

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

---

## 6. Find Interesting Endpoints

Filter potentially security-relevant paths:

```bash
grep -Ei 'api|login|logout|forgot|reset|password|user|users|account|admin|auth|profile|member|customer|employee|upload|download|document|file|report|search|config|token|payment|invoice' \
endpoints_unique.txt \
| sort -u > interesting_endpoints.txt
```

View results:

```bash
cat interesting_endpoints.txt
```

---

## 7. Probe Discovered Endpoints

Check HTTP status codes, content types, sizes, and redirects:

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

---

## 8. Collect Historical URLs

Using gau:

```bash
gau "$DOMAIN" \
| sort -u > gau_urls.txt
```

Using Waybackurls:

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

---

## 9. Find Parameterized URLs

Extract URLs containing query parameters:

```bash
grep '=' all_endpoints.txt \
| sort -u > parameterized_urls.txt
```

View sample:

```bash
head -50 parameterized_urls.txt
```

---

## 10. Find API Endpoints

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

---

## 11. Find High-Value Paths

```bash
grep -Ei 'admin|login|auth|account|user|employee|customer|password|reset|forgot|upload|download|document|report|payment|invoice|token|config|internal|debug|backup' \
all_endpoints.txt \
| sort -u > high_value_endpoints.txt
```

View results:

```bash
head -100 high_value_endpoints.txt
```

---

## 12. Inspect JavaScript Files

Find JavaScript URLs:

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

---

## 13. Inspect Page Source

Download a page:

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

---

## 14. Security Header Check

```bash
curl -skI "$TARGET" \
| tee security_headers.txt
```

Review headers such as:

- Content-Security-Policy
- Strict-Transport-Security
- X-Content-Type-Options
- X-Frame-Options
- Referrer-Policy
- Permissions-Policy
- Access-Control-Allow-Origin
- Server
- X-Powered-By

---

## 15. HTTP vs HTTPS Check

```bash
curl -v --max-time 10 \
"http://$DOMAIN/" \
-o /dev/null 2>&1 \
| tee http_check.txt
```

Verify whether HTTP redirects to HTTPS or whether port 80 is unavailable.

---

## 16. Common Exposure Checks

Create a list of common diagnostic and configuration paths:

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

Probe them:

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

A `200` response does not automatically mean a vulnerability. Manually verify the returned content and assess whether sensitive information is actually exposed.

---

## 17. CORS Header Check

Check how the application responds to a cross-origin request:

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

Look for:

```text
Access-Control-Allow-Origin
Access-Control-Allow-Credentials
Access-Control-Allow-Methods
Access-Control-Allow-Headers
Vary: Origin
```

`Access-Control-Allow-Origin: *` alone is not necessarily a vulnerability. Determine whether sensitive authenticated data can actually be read cross-origin.

---

## 18. HTTP Method Check

For an authorized endpoint:

```bash
export ENDPOINT="$TARGET/api/endpoint"
```

Check common methods:

```bash
for method in GET HEAD OPTIONS POST PUT PATCH DELETE; do
  curl -sk --max-time 10 \
    -X "$method" \
    -o /dev/null \
    -w "$method -> HTTP %{http_code} | Type: %{content_type} | Size: %{size_download}\n" \
    "$ENDPOINT"
done
```

Do not assume that an allowed HTTP method is automatically a vulnerability. Verify whether unauthorized state changes or access are possible.

---

## 19. Technology Fingerprinting

```bash
httpx -u "$TARGET" \
-silent \
-status-code \
-title \
-tech-detect
```

Technology disclosure can help identify the application stack, but version information alone is generally informational unless it leads to a demonstrable security impact.

---

## 20. Quick Results Review

List generated files:

```bash
ls -lh
```

Search collected results for potentially interesting responses:

```bash
grep -Ei '\[200\]|\[201\]|\[204\]|\[301\]|\[302\]|\[401\]|\[403\]' \
*.txt 2>/dev/null
```

---

## Suggested Workflow

```text
1. Confirm authorization and scope
        ↓
2. Create dedicated target folder
        ↓
3. Enumerate subdomains
        ↓
4. Identify live hosts
        ↓
5. Prioritize interesting hosts
        ↓
6. Crawl selected applications
        ↓
7. Collect historical URLs
        ↓
8. Extract API and high-value endpoints
        ↓
9. Inspect JavaScript and page source
        ↓
10. Check security headers
        ↓
11. Review common exposures
        ↓
12. Validate CORS behavior
        ↓
13. Review HTTP methods
        ↓
14. Manually validate potential findings
        ↓
15. Document evidence and impact
```

---

## Important Testing Principle

A security misconfiguration or unusual response is not automatically a reportable vulnerability.

Always validate:

- What is exposed?
- Is authentication required?
- Can unauthorized users access sensitive information?
- Can the behavior be exploited?
- What is the real security impact?
- Is the affected asset explicitly in scope?

Document the exact request, response, affected endpoint, evidence, impact, and recommended remediation for every confirmed finding.

---

## Tools

- Subfinder
- httpx
- Katana
- gau
- waybackurls
- curl
- grep
- awk
- sort

---

## Notes

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

---

**For authorized security testing and educational purposes only.**
