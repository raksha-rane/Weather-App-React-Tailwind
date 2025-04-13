# Zeotap – Salesforce Service Cloud Contact Integration PRD

---

## 1. Overview

### 1.1 Use Case and Goals
The objective of this integration is to synchronize customer profile data from Zeotap to Salesforce Service Cloud, ensuring the **Contact** object in Salesforce always reflects the latest and most accurate customer information.

**Business Value:**
- Maintain real-time, accurate, and complete contact records in Salesforce.
- Improve customer service and personalization by providing agents with the latest customer data.
- Reduce manual updates and potential errors by automating data synchronization.
- Enable data-driven decisions by leveraging unified customer profiles across systems.

---

## 2. Data Flow Overview

### 2.1 Triggers
- New contact is created in Zeotap.
- Existing contact is updated in Zeotap.

### 2.2 Flow Steps
1. Zeotap detects a new or updated contact.
2. Zeotap checks Salesforce for an existing contact using unique identifiers (e.g., Email).
3. If contact exists, a **PATCH** request is sent to update; otherwise, a **POST** request is sent to create.
4. Salesforce processes the request and responds with success or error.
5. Zeotap logs responses and handles retries or alerts on failure.

*Note: Include data flow diagram in final submission if required.*

---

## 3. Authentication Method

### 3.1 Type
**OAuth 2.0 JSON Web Token (JWT) Bearer Flow**

This method allows backend systems to access Salesforce data without requiring interactive user login, making it ideal for automated processes.

### 3.2 Prerequisites
- **Connected App Configuration in Salesforce:**
  - Must be admin-approved.
  - OAuth scopes should include: `Full access`, `Access and manage your data`.
  - An X.509 certificate must be associated with the connected app.

### 3.3 JWT Setup
- JWT is constructed and signed using RSA SHA256.
- Includes the following claims:
  - `iss`: client_id of the connected app
  - `sub`: Salesforce username
  - `aud`: `https://login.salesforce.com`
  - `exp`: Expiration timestamp in UTC

### 3.4 Token Request
**Endpoint:** `POST https://login.salesforce.com/services/oauth2/token`

**Parameters:**
| Parameter   | Value                                              |
|-------------|----------------------------------------------------|
| grant_type  | `urn:ietf:params:oauth:grant-type:jwt-bearer`     |
| assertion   | `<signed JWT>`                                    |

### 3.5 Sample Token Response
```json
{
  "access_token": "00Dxx0000001gPz!AQYAQ...",
  "instance_url": "https://instance.salesforce.com",
  "token_type": "Bearer"
}
```

---

## 4. Key Endpoints

| Operation        | Endpoint                                                                 |
|------------------|--------------------------------------------------------------------------|
| Create Contact   | `POST /services/data/v63.0/sobjects/Contact/`                           |
| Update Contact   | `PATCH /services/data/v63.0/sobjects/Contact/{ContactId}`              |
| Query Contact    | `GET /services/data/v63.0/query/?q=SELECT+Id+FROM+Contact+WHERE+Email='{email}'` |

**Base URL:** `https://<instance>.salesforce.com`

---

## 5. Sample Requests and Responses

### 5.1 Create Contact
```http
POST https://<instance>.salesforce.com/services/data/v63.0/sobjects/Contact/
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "FirstName": "Jane",
  "LastName": "Doe",
  "Email": "jane.doe@example.com",
  "Phone": "+1234567890",
  "MailingCity": "Berlin"
}
```
**Response:**
```json
{
  "id": "0032x00001ABCdE",
  "success": true,
  "errors": []
}
```

### 5.2 Update Contact
```http
PATCH https://<instance>.salesforce.com/services/data/v63.0/sobjects/Contact/0032x00001ABCdE
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "Phone": "+1987654321",
  "MailingCity": "Munich"
}
```
**Response:** `HTTP/1.1 204 No Content`

### 5.3 Retrieve Contact by ID
```http
GET https://<instance>.salesforce.com/services/data/v63.0/sobjects/Contact/0032x00001ABCdE
Authorization: Bearer <access_token>
```
**Response:**
```json
{
  "Id": "0032x00001ABCdE",
  "FirstName": "Jane",
  "LastName": "Doe",
  "Email": "jane.doe@example.com",
  "Phone": "+1234567890",
  "MailingCity": "Berlin"
}
```

### 5.4 Search Contact by Email
```http
GET https://<instance>.salesforce.com/services/data/v63.0/query?q=SELECT+Id+FROM+Contact+WHERE+Email='jane.doe@example.com'
Authorization: Bearer <access_token>
```
**Response:**
```json
{
  "totalSize": 1,
  "done": true,
  "records": [
    { "Id": "0032x00001ABCdE" }
  ]
}
```

### 5.5 Handle Invalid Data
```http
POST https://<instance>.salesforce.com/services/data/v63.0/sobjects/Contact/
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "FirstName": "Jane",
  "LastName": "Doe",
  "Email": "invalid-email",
  "Phone": "+1234567890",
  "MailingCity": "Berlin"
}
```
**Response:**
```json
{
  "message": "MALFORMED_ID: invalid-email",
  "errorCode": "MALFORMED_ID"
}
```

---

## 6. Common Errors, Rate Limits, and Required Fields

### 6.1 Required Fields
| Field Name | Required For   | Description                                  |
|------------|----------------|----------------------------------------------|
| LastName   | Create, Update | Mandatory for Salesforce Contact object.     |
| Email      | Sync Logic     | Used as a unique identifier by Zeotap.       |

### 6.2 Common API Errors
| Error Code | Message                   | Cause                                      |
|------------|---------------------------|--------------------------------------------|
| 400        | Required fields missing   | Missing fields like LastName               |
| 401        | Invalid session ID        | Invalid/expired access token               |
| 403        | Insufficient access       | Insufficient permissions                   |
| 404        | Resource not found        | Invalid ContactId                          |
| 429        | Request limit exceeded    | API rate limit exceeded                    |
| 500        | Internal server error     | Server-side issue                          |
| 503        | Service unavailable       | Scheduled maintenance or outage            |

### 6.3 Rate Limits
| Limit Type                          | Value                                |
|------------------------------------|--------------------------------------|
| Daily API Requests                 | 15,000 – 1,000,000+ (license-based)  |
| Concurrent REST API Calls (per org)| 25 (default)                         |

---

## 7. Assumptions and Edge Cases

### 7.1 Assumptions
| Assumption               | Description                                                                 |
|--------------------------|-----------------------------------------------------------------------------|
| Email as Primary ID      | Used for identifying and matching records                                  |
| Salesforce Access Scope  | Zeotap has necessary API access                                            |
| Pre-validation by Zeotap | Data is pre-validated and schema matched                                   |
| Network/Auth Stability   | Stable token generation and authentication                                 |
| Idempotent Design        | Duplicate requests don’t create duplicates                                |

### 7.2 Edge Cases
| Edge Case                         | Handling Strategy                                                                 |
|----------------------------------|----------------------------------------------------------------------------------|
| Multiple Contacts with Same Email| Return error, log conflict, and flag for manual review                           |
| Missing Required Fields          | Reject request before sending to Salesforce                                      |
| Deleted Contact in Salesforce    | Skip sync and log                                                                |
| Downtime or Rate Limit Breach    | Implement retry with backoff; queue failed updates                              |
| Invalid Contact ID               | Log and optionally retry by re-searching email                                  |
| Partial Updates                  | Use PATCH to avoid overwriting unaffected fields                                |
| API Token Expiration             | Handle 401 errors with auto-refresh via JWT                                     |

---

## 8. Testing Approach

**Tools:** Postman, Salesforce Developer Org

**Steps:**
1. **Authenticate** using JWT Bearer Flow in Postman.
2. **Create Contact:** Test valid POST request and validate via GET.
3. **Update Contact:** Test PATCH request with changed fields.
4. **Negative Tests:**
   - Missing fields
   - Invalid ID
   - Rate limits
5. **Edge Case Tests:**
   - Duplicate emails
   - Non-existing contacts
   - Partial updates

---

## 9. References
- Salesforce Spring '25 (v63.0) REST API Guide
- Salesforce JWT OAuth 2.0 Bearer Token Flow
- Salesforce Contact Object Documentation

---

**Prepared by:** Raksha Rane  
**Document Version:** 1.0  
**Date:** April 12, 2025
