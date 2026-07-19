<div align="center">

# `API VAPT // APPLICATION PROGRAMMING INTERFACE SECURITY`

### WEB VAPT FIELD NOTES

`DISCOVER` → `UNDERSTAND` → `AUTHENTICATE` → `AUTHORIZE` → `VALIDATE` → `REPORT`

<br>

**An API endpoint is not secure simply because it is difficult to discover.**

</div>

---

<div align="center">

[← HOME](README.md) &nbsp; • &nbsp; [RECON](Recon.md) &nbsp; • &nbsp; **API VAPT** &nbsp; • &nbsp; AUTH &nbsp; • &nbsp; ACCESS CONTROL &nbsp; • &nbsp; CORS &nbsp; • &nbsp; HEADERS

</div>

---

## `// WHAT IS API VAPT?`

API VAPT is the process of assessing an authorized API for security weaknesses in areas such as authentication, authorization, input handling, business logic, and security configuration.

A typical API assessment tries to answer:

```text
What API endpoints exist?
        ↓
Which endpoints are public?
        ↓
Which endpoints require authentication?
        ↓
What roles can access each endpoint?
        ↓
Can users access only their own objects?
        ↓
Can users perform only permitted actions?
        ↓
Are sensitive properties protected?
        ↓
Is user-controlled input validated?
        ↓
Are abuse protections implemented?
        ↓
Does the API expose unnecessary information?
```

API testing should not consist of sending random payloads to every endpoint.

First understand what the API is designed to do.

Then verify whether its security controls enforce that design.

> **Important**
>
> Use these techniques only on systems you own or have explicit authorization to test. Follow the defined scope, rules of engagement, test-account requirements, automation restrictions, and rate limits.

---

# `// BEFORE YOU START`

Throughout this guide, the following environment variables are used.

### `$TARGET`

The base URL of the authorized API.

```bash
export TARGET="https://api.target.com"
```

### `$ENDPOINT`

A specific endpoint being tested.

```bash
export ENDPOINT="$TARGET/api/v1/profile"
```

### `$TOKEN`

A valid test-account authorization token.

```bash
export TOKEN="YOUR_AUTHORIZED_TEST_TOKEN"
```

Do not commit real tokens, passwords, API keys, session cookies, or client secrets to GitHub.

Verify the variables:

```bash
echo "$TARGET"
echo "$ENDPOINT"
```

Avoid printing sensitive tokens into shared terminal logs.

---

# `01 // IDENTIFY API ENDPOINTS`

### What are we doing?

Identifying URLs that appear to expose API functionality.

If you already completed reconnaissance, search the collected endpoints:

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

### Common API patterns

```text
/api/
/api/v1/
/api/v2/
/rest/
/graphql
/swagger
/openapi
```

### What to look for

Pay attention to endpoints associated with:

```text
Users
Accounts
Profiles
Orders
Documents
Payments
Files
Administrative actions
Authentication
Password recovery
Search
Reports
```

> **Field Note**
>
> Discovering an API endpoint does not mean the endpoint is vulnerable. It simply expands your understanding of the application's attack surface.

---

# `02 // UNDERSTAND THE API REQUEST`

### What are we doing?

Understanding the structure of an API request before testing its security controls.

A request typically contains:

```text
METHOD
   │
   ├── URL / PATH
   │
   ├── QUERY PARAMETERS
   │
   ├── HEADERS
   │
   ├── AUTHENTICATION
   │
   └── REQUEST BODY
```

Example:

```http
GET /api/v1/profile HTTP/1.1
Host: api.target.com
Authorization: Bearer <TEST_TOKEN>
Accept: application/json
```

The response may contain:

```text
Status Code
Headers
Content Type
JSON / XML Data
Error Information
```

Before testing an endpoint, understand:

- What is the endpoint supposed to do?
- Is authentication expected?
- Which role should access it?
- Does it operate on a specific object?
- Does it modify application state?

---

# `03 // BASELINE REQUEST`

### What are we doing?

Creating a known-good baseline request.

This gives us something to compare against when security controls are tested later.

### Unauthenticated request

```bash
curl -ski "$ENDPOINT"
```

### Authenticated request

```bash
curl -ski "$ENDPOINT" \
-H "Authorization: Bearer $TOKEN"
```

### Request JSON

```bash
curl -sk "$ENDPOINT" \
-H "Authorization: Bearer $TOKEN" \
-H "Accept: application/json"
```

### Save the response

```bash
curl -sk "$ENDPOINT" \
-H "Authorization: Bearer $TOKEN" \
-o baseline_response.json
```

### What to record

Document:

```text
HTTP Method
Endpoint
Authentication State
Test Account / Role
Status Code
Content Type
Response Size
Expected Behavior
Actual Behavior
```

This baseline makes later comparisons more reliable.

---

# `04 // HTTP METHOD REVIEW`

### What are we doing?

Understanding which HTTP methods an endpoint accepts.

Common methods include:

| Method | Typical Purpose |
|---|---|
| `GET` | Retrieve data |
| `POST` | Create or submit data |
| `PUT` | Replace or update data |
| `PATCH` | Partially update data |
| `DELETE` | Delete data |
| `HEAD` | Retrieve headers |
| `OPTIONS` | Describe communication options |

### Check common methods

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

### How to interpret this

A supported HTTP method is not automatically a vulnerability.

The important question is:

```text
Can this method perform an action
the current user should not be allowed to perform?
```

State-changing methods should be tested only where explicitly authorized.

---

# `05 // AUTHENTICATION CHECKS`

### What are we doing?

Verifying that endpoints requiring authentication actually enforce it.

Start with the known-good authenticated request.

Then compare it with an unauthenticated request.

### Without authentication

```bash
curl -ski "$ENDPOINT"
```

### With authorized test credentials

```bash
curl -ski "$ENDPOINT" \
-H "Authorization: Bearer $TOKEN"
```

Compare:

```text
Status Code
Response Body
Response Size
Returned Data
```

### Expected behavior

Protected resources will commonly reject unauthenticated requests.

Possible responses include:

```text
401 Unauthorized
403 Forbidden
```

The exact expected response depends on the API design.

### What to investigate

Ask:

```text
Can a protected endpoint be accessed without authentication?

Does removing authentication still return sensitive information?

Are authentication requirements consistently enforced across API versions?
```

> A public endpoint returning public information is not an authentication vulnerability.

---

# `06 // OBJECT-LEVEL AUTHORIZATION — BOLA / IDOR`

### What are we doing?

Checking whether authenticated users are restricted to objects they are authorized to access.

Object references may appear in:

```text
/api/v1/users/{id}
/api/v1/orders/{id}
/api/v1/documents/{id}
/api/v1/accounts/{id}
```

### Safe testing approach

Use **two authorized test accounts** and objects specifically created for testing.

```text
TEST ACCOUNT A
      │
      └── OBJECT A

TEST ACCOUNT B
      │
      └── OBJECT B
```

Expected:

```text
Account A → Object A → ALLOWED
Account B → Object B → ALLOWED

Account A → Object B → DENIED
Account B → Object A → DENIED
```

### What to verify

Determine whether changing an object reference allows one test account to:

```text
Read another test account's object
Modify another test account's object
Delete another test account's object
Perform actions against another account's object
```

> **Important**
>
> Use only controlled test accounts and test data when performing authorization testing. Do not access or modify real users' data.

A numeric ID in a URL is not automatically an IDOR.

The vulnerability exists when authorization checks fail.

---

# `07 // FUNCTION-LEVEL AUTHORIZATION`

### What are we doing?

Checking whether users can access API functions that should be restricted to another role.

Example role model:

```text
NORMAL USER
     │
     ├── View own profile
     └── Update own profile

ADMINISTRATOR
     │
     ├── View users
     ├── Manage users
     └── Administrative actions
```

Testing asks whether the server enforces these role boundaries.

### Controlled test

Using authorized accounts with different roles, compare access to the same endpoint.

```bash
curl -ski "$ENDPOINT" \
-H "Authorization: Bearer $TOKEN"
```

### What to verify

Ask:

```text
Can a lower-privileged test account access an administrative endpoint?

Can the user perform a privileged operation directly through the API?

Is authorization enforced server-side?
```

Hiding an administrative button in the frontend is not sufficient authorization.

The API itself must enforce permissions.

---

# `08 // PROPERTY-LEVEL AUTHORIZATION`

### What are we doing?

Checking whether users can read or modify object properties that should be restricted.

An API object might contain:

```json
{
  "id": 1001,
  "name": "Test User",
  "email": "test@example.com",
  "role": "user"
}
```

Not every property should necessarily be modifiable by the user.

### What to verify

Review whether the API correctly controls access to sensitive properties such as:

```text
role
permissions
accountStatus
ownerId
isAdmin
internalFlags
```

Questions to ask:

```text
Does the response expose properties the user should not see?

Can the user modify properties that should be server-controlled?

Does the backend enforce an allowlist of writable properties?
```

Use only authorized test accounts and test objects.

---

# `09 // INPUT VALIDATION`

### What are we doing?

Checking whether the API validates user-controlled input according to its expected format and business rules.

Consider fields such as:

```text
Name
Email
Date
Quantity
Object ID
File Name
Search Input
Pagination
```

### Example request structure

```bash
curl -ski -X POST "$ENDPOINT" \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-d '{
  "name": "Test User"
}'
```

### Validation questions

Ask:

```text
Are required fields enforced?

Are data types validated?

Are length limits enforced?

Are invalid values rejected?

Are unexpected fields ignored or rejected?

Are server-side business rules enforced?
```

Client-side validation alone is not sufficient.

Security-relevant validation should occur on the server.

---

# `10 // MASS ASSIGNMENT REVIEW`

### What are we doing?

Checking whether the API automatically binds user-controlled JSON properties to sensitive server-side object properties.

Suppose the intended request is:

```json
{
  "name": "Test User",
  "email": "test@example.com"
}
```

The application should define which properties the user is permitted to modify.

### What to review

Look for sensitive fields such as:

```text
role
isAdmin
permissions
status
ownerId
accountType
```

### Security question

```text
Does the server accept properties
that should be controlled only by the application?
```

Use controlled test accounts and test records.

The presence of a property name in JavaScript or an API response does not itself prove mass assignment.

The server must actually accept an unauthorized property change for there to be security impact.

---

# `11 // RATE LIMITING & RESOURCE CONTROLS`

### What are we doing?

Checking whether sensitive or resource-intensive API operations have appropriate abuse protections.

Potentially sensitive operations include:

```text
Login
Password recovery
OTP verification
Account creation
Search
Expensive reports
File processing
```

### What to observe

Review response headers:

```bash
curl -skI "$ENDPOINT"
```

Look for potential rate-limit information:

```text
RateLimit
RateLimit-Limit
RateLimit-Remaining
RateLimit-Reset
Retry-After
```

Not every API must expose these headers.

### Questions to ask

```text
Is abuse protection appropriate for the endpoint?

Does the application respond consistently when limits are reached?

Are expensive operations appropriately controlled?
```

> Do not perform high-volume stress testing or denial-of-service testing unless it is explicitly authorized in the rules of engagement.

---

# `12 // FILE UPLOAD API REVIEW`

### What are we doing?

Reviewing API endpoints that accept file uploads.

First determine:

```text
Which file types are expected?
What is the maximum file size?
Where are files stored?
Who can retrieve uploaded files?
```

### Authorized test upload

For a harmless test file:

```bash
curl -ski -X POST "$ENDPOINT" \
-H "Authorization: Bearer $TOKEN" \
-F "file=@test.txt"
```

### What to verify

Review:

```text
File type validation
File size restrictions
Filename handling
Storage location
Authorization on file retrieval
Content-Disposition behavior
Content-Type handling
```

Use only harmless test files during routine validation.

---

# `13 // CORS REVIEW`

### What are we doing?

Reviewing how the API handles requests originating from another website.

### Send a test Origin

```bash
curl -ski "$ENDPOINT" \
-H 'Origin: https://example.com' \
-o /dev/null \
-D - \
| grep -iE 'HTTP/|access-control|vary:'
```

### Preflight request

```bash
curl -ski -X OPTIONS "$ENDPOINT" \
-H 'Origin: https://example.com' \
-H 'Access-Control-Request-Method: POST' \
-H 'Access-Control-Request-Headers: authorization,content-type'
```

### Look for

```text
Access-Control-Allow-Origin
Access-Control-Allow-Credentials
Access-Control-Allow-Methods
Access-Control-Allow-Headers
Vary: Origin
```

### Important

```text
Access-Control-Allow-Origin: *
```

alone does not automatically indicate a vulnerability.

Determine:

```text
Is the endpoint sensitive?

Does it require authentication?

Are credentials involved?

Can an unauthorized origin actually read protected data?
```

Validate real impact before reporting.

---

# `14 // ERROR HANDLING & INFORMATION DISCLOSURE`

### What are we doing?

Checking whether invalid requests cause the API to reveal unnecessary internal information.

### What to look for

Error responses should generally avoid exposing:

```text
Stack traces
Internal file paths
Database errors
SQL queries
Internal IP addresses
Framework debugging information
Secrets
Credentials
API keys
```

### Example inspection

```bash
curl -ski "$ENDPOINT"
```

Review:

```text
Response headers
Response body
Error structure
Server information
Framework information
```

Detailed errors are not always vulnerabilities.

Assess whether the disclosed information meaningfully helps an attacker or exposes sensitive data.

---

# `15 // API VERSIONING`

### What are we doing?

Checking whether multiple API versions exist and whether security controls are consistently applied.

Common patterns:

```text
/api/v1/
/api/v2/
/api/v3/
```

Search collected URLs:

```bash
grep -Eio '/api/v[0-9]+/' all_endpoints.txt \
| sort -u
```

### What to investigate

If multiple authorized API versions exist, compare:

```text
Authentication requirements
Authorization controls
Returned properties
Deprecated functionality
Security headers
```

> An old API version is not automatically vulnerable.
>
> The security concern arises when deprecated endpoints remain accessible with weaker security controls or expose functionality that should no longer be available.

---

# `16 // API DOCUMENTATION DISCOVERY`

### What are we doing?

Checking whether API documentation is intentionally exposed on an authorized target.

Possible locations include:

```text
/swagger
/swagger-ui
/api-docs
/openapi.json
/swagger.json
```

If discovered during recon, manually review whether the documentation is intended to be public.

### What to look for

Documentation can help identify:

```text
Available endpoints
Expected parameters
Authentication mechanisms
Request schemas
Response schemas
API versions
```

Public API documentation is not automatically a vulnerability.

The security impact depends on whether sensitive internal-only information is unintentionally exposed.

---

# `17 // GRAPHQL REVIEW`

### What are we doing?

Identifying whether the application uses GraphQL.

Common endpoints include:

```text
/graphql
/api/graphql
```

### Basic identification

```bash
grep -Ei 'graphql' all_endpoints.txt
```

### What to review

For an authorized GraphQL API, understand:

```text
Authentication
Object-level authorization
Field-level authorization
Query complexity controls
Error handling
Sensitive field exposure
```

GraphQL is not inherently insecure.

Security depends on how authentication, authorization, schema exposure, and resource controls are implemented.

---

# `18 // RESPONSE COMPARISON`

### What are we doing?

Comparing API responses between different controlled testing conditions.

For example:

```text
Authenticated
vs
Unauthenticated

User Role
vs
Admin Role

Object Owner
vs
Non-Owner

Valid Input
vs
Invalid Input
```

Save responses:

```bash
curl -sk "$ENDPOINT" \
-H "Authorization: Bearer $TOKEN" \
-o response_authenticated.json
```

Without authentication:

```bash
curl -sk "$ENDPOINT" \
-o response_unauthenticated.json
```

Compare:

```bash
diff response_authenticated.json response_unauthenticated.json
```

### What to compare

Review differences in:

```text
Status Code
Response Size
Returned Fields
Error Messages
Sensitive Data
Application Behavior
```

Response comparison is often more useful than looking at individual status codes in isolation.

---

# `19 // RESPONSE HEADER REVIEW`

### What are we doing?

Inspecting API response headers for security controls and unnecessary technology disclosure.

### Run

```bash
curl -skI "$ENDPOINT"
```

Review:

```text
Content-Type
Cache-Control
Strict-Transport-Security
Access-Control-Allow-Origin
Access-Control-Allow-Credentials
Server
X-Powered-By
```

For sensitive API responses, also consider whether caching behavior is appropriate.

Header findings should always be assessed in context.

---

# `20 // TEMPLATE-BASED CHECKS`

### What are we doing?

Using automated templates to identify potential issues that deserve manual review.

If automated scanning is permitted by the rules of engagement:

```bash
nuclei -u "$TARGET" \
-severity info,low,medium,high,critical \
-o api_nuclei_results.txt
```

Review:

```bash
cat api_nuclei_results.txt
```

### Important

```text
NUCLEI MATCH
      │
      ▼
POTENTIAL FINDING
      │
      ▼
MANUAL REPRODUCTION
      │
      ▼
IMPACT VALIDATION
      │
      ▼
CONFIRMED FINDING
```

Never report an automated scanner result without manual validation.

---

# `21 // MANUAL VALIDATION`

Before reporting an API vulnerability, answer:

```text
Can I reproduce it consistently?

Is the affected endpoint in scope?

What authentication level is required?

Which user role is affected?

Can unauthorized data actually be accessed?

Can unauthorized actions actually be performed?

Is the impact demonstrable?

Is this expected application behavior?
```

A suspicious API response is not automatically a vulnerability.

---

# `22 // DOCUMENT THE FINDING`

For every confirmed API security issue, document:

```text
TITLE
  │
  ├── Affected Asset
  ├── Endpoint
  ├── HTTP Method
  ├── Authentication Required
  ├── Required Role
  ├── Vulnerability Description
  ├── Steps to Reproduce
  ├── Request
  ├── Response
  ├── Security Impact
  ├── Evidence
  └── Recommended Remediation
```

Avoid including unnecessary real user information in reports.

Use test-account identifiers where possible.

---

# `// API TESTING WORKFLOW`

```text
┌─────────────────────────────────────┐
│  01. CONFIRM SCOPE & AUTHORIZATION  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  02. DISCOVER API ENDPOINTS         │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  03. UNDERSTAND REQUEST / RESPONSE  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  04. CREATE A KNOWN-GOOD BASELINE   │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  05. TEST AUTHENTICATION            │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  06. TEST OBJECT AUTHORIZATION      │
│      BOLA / IDOR                    │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  07. TEST FUNCTION AUTHORIZATION    │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  08. REVIEW PROPERTY AUTHORIZATION  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  09. REVIEW INPUT & BUSINESS RULES  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  10. REVIEW RESOURCE CONTROLS       │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  11. REVIEW CORS & CONFIGURATION    │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  12. MANUALLY VALIDATE RESULTS      │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  13. DOCUMENT EVIDENCE & IMPACT     │
└─────────────────────────────────────┘
```

---

# `// OWASP API SECURITY MAPPING`

Use the OWASP API Security Top 10 as a framework for organizing your assessment.

Typical areas include:

| Security Area | What You Are Reviewing |
|---|---|
| Object-Level Authorization | Access to individual objects and records |
| Authentication | Identity and session/token controls |
| Object Property Authorization | Access to sensitive object fields |
| Resource Consumption | Rate and resource controls |
| Function-Level Authorization | Access to privileged functionality |
| Sensitive Business Flows | Automated abuse of important workflows |
| Server-Side Request Handling | Handling of user-supplied remote resources |
| Security Configuration | API and infrastructure configuration |
| API Inventory | Versions, hosts, and undocumented APIs |
| Third-Party API Consumption | Trust boundaries with external APIs |

> Use the current OWASP API Security Top 10 documentation when performing a formal assessment, as categories and guidance may evolve over time.

---

# `// IMPORTANT TESTING PRINCIPLE`

<div align="center">

### `STATUS CODE ≠ SECURITY DECISION`

</div>

Do not judge API security from a status code alone.

For example:

```text
200 OK
```

may return public information and be completely expected.

```text
403 Forbidden
```

may indicate correctly enforced authorization.

The important question is:

```text
WHAT SHOULD THIS USER BE ALLOWED TO DO?
                 │
                 ▼
WHAT DID THE SERVER ACTUALLY ALLOW?
                 │
                 ▼
IS THERE A SECURITY IMPACT?
```

Always understand the intended authorization model before declaring a vulnerability.

---

# `// API TESTING CHECKLIST`

Before completing an authorized API assessment, review:

- [ ] API endpoints identified
- [ ] API versions identified
- [ ] Authentication requirements understood
- [ ] Unauthenticated access reviewed
- [ ] Object-level authorization reviewed
- [ ] Function-level authorization reviewed
- [ ] Property-level authorization reviewed
- [ ] Input validation reviewed
- [ ] Mass assignment risks reviewed
- [ ] Resource and rate controls reviewed
- [ ] File handling reviewed where applicable
- [ ] CORS configuration reviewed
- [ ] Error handling reviewed
- [ ] Sensitive information exposure reviewed
- [ ] API documentation exposure reviewed
- [ ] GraphQL reviewed where applicable
- [ ] Automated findings manually validated
- [ ] Confirmed findings documented

---

# `// TOOLBOX`

| Tool | Primary Use |
|---|---|
| `Burp Suite` | Intercepting and manually testing API requests |
| `Postman` | API request creation and collection management |
| `curl` | Command-line HTTP requests |
| `httpx` | Endpoint probing |
| `katana` | Endpoint discovery |
| `gau` | Historical URL collection |
| `waybackurls` | Archived URL discovery |
| `nuclei` | Template-based security checks |
| `jq` | JSON parsing and formatting |
| `grep` | Filtering endpoints and responses |
| `diff` | Comparing API responses |

---

# `// FINAL NOTES`

API security testing is primarily about understanding **trust boundaries**.

Think in terms of:

```text
WHO
 │
 ▼
is making the request?

WHAT
 │
 ▼
resource are they accessing?

WHICH
 │
 ▼
action are they performing?

SHOULD
 │
 ▼
they be allowed to do it?

DID
 │
 ▼
the server enforce that decision?
```

Tools help discover and send requests.

The tester still needs to understand the application's intended behavior, validate the security impact, and distinguish genuine vulnerabilities from expected functionality.

---

<div align="center">

### `DISCOVER` ── `UNDERSTAND` ── `TEST` ── `COMPARE` ── `VALIDATE`

<br>

**For authorized security testing and educational purposes only.**

Only test systems you own or have explicit permission to assess.  
Always follow the defined scope and rules of engagement.

<br>

[← BACK TO HOME](README.md) &nbsp; • &nbsp; [← RECON](Recon.md)

<br>

`slayfinds // web application security field notes`

</div>
