**CWE:** CWE-639 – Authorization Bypass Through User-Controlled Key  
**CVSS v3.1:** 7.1 (High)
## Description  
  
Insecure Direct Object Reference (IDOR) vulnerabilities occur when an application exposes internal object identifiers and fails to verify whether the authenticated user is authorized to access the requested resource.  
  
During testing, the order management functionality was found to rely on predictable numeric identifiers. By modifying the order identifier within a legitimate request, it was possible to access order records belonging to other users without authorization.  
  
## Vulnerable Instance  
  
```http  
GET /workshop/api/shop/orders/{orderId}  
```

![522](Attachments/Pasted%20image%2020260602132543.png)
## Proof of Concept  
  
### Step 1 – Access the Shop Orders Functionality  
  
After authenticating to the application, navigate to the **Shop** section and purchase a product.  
  
Open the order details page and intercept the request using Burp Suite Proxy.  
  
Example request:  
  
```http  
GET /workshop/api/shop/orders/16 HTTP/1.1  
Host: localhost:8888  
Authorization: Bearer [JWT_TOKEN]  
```
![580](Attachments/Pasted%20image%2020260602132706.png)
The server returns the details associated with the authenticated user's order, including personal and payment-related information.
### Step 2 – Send the Request to Burp Intruder  
  
Send the intercepted request to **Burp Intruder** and mark the order identifier as a payload position.  
  
Example:  
  
```http  
GET /workshop/api/shop/orders/§16§ HTTP/1.1  
Host: localhost:8888  
Authorization: Bearer [JWT_TOKEN]  
```
![567](Attachments/Pasted%20image%2020260602132803.png)
Configure a numeric payload list to iterate through multiple order identifiers
**Payload Configuration:**  
  
- Type: Numbers  
- Range: 1–100  
- Step: 1
![371](Attachments/Pasted%20image%2020260602132814.png)

### Step 3 – Enumerate Order Identifiers  
  
Execute the Intruder attack and analyze the responses.  
  
Several responses return different content lengths and response sizes, indicating the existence of additional order records.  
  

![564](../Pasted%20image%2020260602132928.png)One response revealed another user's order information:  
  
```json  
{  
"order": {  
"id": 5,  
"user": {  
"email": "admin@example.com"  
},  
"product": {  
"name": "Seat",  
"price": "10.00"  
}  
}  
}  
```  
  
The response confirms that order records belonging to other users can be accessed simply by modifying the order identifier.

### Step 4 – Verify Unauthorized Data Access  
  
By continuing to enumerate order IDs, additional records belonging to different users become accessible.  
  
The application fails to validate object ownership before returning sensitive order information, resulting in unauthorized access to customer data.  
  
**Result:** The endpoint is vulnerable to **Insecure Direct Object Reference (IDOR)**.

![539](Attachments/Pasted%20image%2020260602133434.png)


## Impact  
  
- Unauthorized access to sensitive customer information  
- Disclosure of personal and financial data  
- Exposure of order and transaction history  
- Potential privilege escalation opportunities  
- Business logic abuse through unauthorized resource access  
- Increased risk of account compromise and privacy violations  
  
## Mitigation  
  
- Implement server-side authorization checks for every object request.  
- Verify resource ownership before returning data.  
- Enforce Role-Based Access Control (RBAC) or Attribute-Based Access Control (ABAC).  
- Replace predictable numeric identifiers with UUIDs where appropriate.  
- Use user-context endpoints such as:  
  
```http  
GET /me/orders  
```  
  
instead of:  
  
```http  
GET /users/{id}/orders  
```  
  
- Apply the principle of least privilege.  
- Log and monitor excessive object enumeration attempts.  
  
## References  
  
- OWASP API Security Top 10 – API1:2023 Broken Object Level Authorization  
- OWASP Authorization Cheat Sheet  
- CWE-639: Authorization Bypass Through User-Controlled Key