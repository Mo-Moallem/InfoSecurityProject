# Security Analysis: Broken Authentication in DVSA

---

## 1. Setup & Deployment
To replicate this analysis, you must deploy the DVSA environment on AWS.

1.  **Deployment:** Use the AWS CloudFormation or SAM templates provided in the DVSA repository to provision the infrastructure (API Gateway, Lambda, Cognito).
2.  **AWS Console:** Navigate to the **AWS Lambda** service and locate the `DVSA-ORDER-MANAGER` function.
3.  **Frontend Access:** Open the DVSA frontend URL and create at least two user accounts (Attacker and Victim) to verify cross-user data access.

---

## 2. Vulnerability Replication (Discovery)
### The Root Cause
The vulnerability exists because the backend blindly trusts the identity claims (`sub` and `username`) found in the JWT payload without verifying the cryptographic signature. An attacker can decode the token, change the user identity, and re-encode it to impersonate any user.

### Step-by-Step Exploitation (Video 1)
1.  **Capture Legitimate Token:**
    * Log into the DVSA application as the "Attacker" user.
    * Open **Burp Suite** and ensure the Proxy Intercept is **ON**.
    * Navigate to "My Orders" to trigger a `POST` request to the `/order` endpoint.
    * In Burp Suite, locate the `Authorization: Bearer <TOKEN>` header.

2.  **Decode the JWT:**
    * Copy the JWT and paste it into [jwt.io](https://jwt.io) or **CyberChef** (using "JWT Decode").
    * Identify the `payload` section containing the `sub` (UUID) and `username`.

3.  **Forge the Identity:**
    * Replace the `sub` and `username` values with the details of the "Victim" user.
    * *Note: Since the signature is not checked, you do not need to provide a valid secret key.*

4.  **Inject and Send:**
    * Re-encode the modified payload (Base64URL) and replace the original token in the Burp Suite request.
    * Send the request via **Burp Repeater**.

5.  **Observe the Result:**
    * The server responds with `200 OK`, returning the full order history (Order IDs, totals, and dates) belonging to the victim.

> **[Insert Screenshot of Burp Suite showing unauthorized data retrieval]**

---

## 3. Remediation & Patching
### The Fix Strategy: Cryptographic Verification
To secure the application, we implement a strict verification process. The Lambda function will now fetch the **JSON Web Key Set (JWKS)** from Amazon Cognito's public endpoint and use it to verify the JWT signature before trusting any claims.

### Technical Implementation (Video 2)
1.  **Access Lambda Console:** Locate the `DVSA-ORDER-MANAGER` function.
2.  **Modify Source Code:** Open `order-manager.js` and replace the insecure decoding logic.

#### Crucial Code Changes:
**Vulnerable Code (REMOVED):**
```javascript
// Insecure: Directly decoding without signature check
var auth_header = headers.Authorization || headers.authorization;
var token_sections = auth_header.split('.');
var auth_data = jose.util.base64url.decode(token_sections[1]);
var token = JSON.parse(auth_data);
var user = token.username;
```

**Remediated Code (ADDED):**
```javascript
// Secure: Cryptographic verification using public keys
const claims = await verifyCognitoJwt(rawToken); // Custom function to verify signature
const user = claims["cognito:username"] || claims["username"];
```

3.  **Deploy Changes:** Click the **Deploy** button in the AWS Lambda editor to apply the patch.

---

## 4. Verification After Fix
1.  **Re-run the Exploit:** Attempt to send the forged token again using Burp Suite Repeater.
2.  **Observe Failure:** The API now returns an **HTTP 400 Bad Request** with a `NotAuthorizedException` error: `"Could not verify signature for token"`.
3.  **Check Legitimate Access:** Use a valid, unmodified token for the logged-in user. The system should still return their own order history correctly, proving that integrity and authentication are now enforced.

> **[Insert Screenshot of HTTP 400 error after patch]**

---

## Security Best Practices
* **Never Trust Client Data:** Treat all parts of a JWT (Header, Payload, and Signature) as untrusted until verified.
* **Use SDKs:** Always use official AWS SDKs or well-maintained cryptographic libraries (like `aws-jwt-verify`) to handle token validation.
* **Enforce Signature Verification:** Ensure the `alg` (algorithm) header in the JWT is validated against an allowlist (e.g., `RS256`) to prevent "None" algorithm attacks.

---
