# DVSA Security Analysis: Information Disclosure via Unhandled Exceptions

This repository contains a comprehensive security analysis and remediation guide for the **Damn Vulnerable Serverless Application (DVSA)**, specifically focusing on **Lesson 10: Unhandled Exceptions**. 


---

## ## Setup & Deployment
To replicate this environment, follow these steps to deploy the DVSA infrastructure:

1. **Prerequisites:** An active AWS Account and the [DVSA Source Code](https://github.com/m6000/dvsa).
2. **Deployment:**
   * Log in to the **AWS Management Console**.
   * Deploy the stack using the provided CloudFormation template or the Serverless Framework (`sls deploy`).
   * Ensure the **DVSA-ORDER-MANAGER** and **DVSA-ORDER-GET** Lambda functions are active.
3. **Access:** Open the DVSA frontend URL provided in the stack outputs.

---

## ## Vulnerability Discovery & Replication
This section describes how to trigger the **Information Disclosure** vulnerability.

### 1. Intercepting the Request
* Open **Burp Suite** and enable the Intercept feature.
* On the DVSA "My Orders" page, perform a standard action (like refreshing orders).
* Intercept the `POST` request sent to the `/orders` endpoint.

### 2. Manipulating the Payload
* Send the intercepted request to **Burp Repeater** (Ctrl+R).
* Modify the JSON body to provide an incomplete request. We will provide a valid action (`get`) but omit the required `order-id`.

**Malformed Payload:**
```json
{
  "action": "get"
}
```

### 3. Execution & Result
* Click **Send** in Burp Suite.
* **Observe the Response:** The server returns a `500 Internal Server Error` or a `200 OK` containing a raw Python `KeyError`.

> **Vulnerability Evidence:** The response includes a `stackTrace` leaking internal paths like `/var/task/get_order.py` and specific code logic fragments.


---

## ## Remediation & Patching
The goal is to implement **Input Validation** at the entry point (Order Manager) to prevent downstream crashes.

### 1. Modifying the Lambda Function
Navigate to the **AWS Lambda Console**, select the **DVSA-ORDER-MANAGER** function, and update the `order_manager.js` file.

#### Step A: Add a Validation Helper
Add this function at the top of your script to check for required fields:

```javascript
function requireFields(obj, fields) { 
   const missing = []; 
   for (const field of fields) { 
       const value = obj[field]; 
       if (value === undefined || value === null || value === "") { 
           missing.push(field); 
       } 
   } 
   return missing; 
}
```

#### Step B: Apply Validation to the 'Get' Case
Update the switch statement to verify input before routing the request:

```javascript
case "get": { 
   const missing = requireFields(req, ["order-id"]); 
   if (missing.length > 0) { 
       // Return a clean, generic error instead of crashing
       return callback(null, badRequest("Missing required field(s)", missing)); 
   } 
   payload = { 
       user: user, 
       orderId: req["order-id"], 
       isAdmin: isAdmin 
   }; 
   functionName = "DVSA-ORDER-GET"; 
   break; 
}
```

### 2. Deployment
Click **Deploy** in the Lambda console to apply the changes.

---

## ## Post-Fix Verification
To ensure the patch is effective, re-run the exploit attempt:

1. Go back to **Burp Suite Repeater**.
2. Resend the malformed payload: `{"action": "get"}`.
3. **Observe the New Response:**
   * **Status:** `err`
   * **Message:** `Missing required field(s)`
   * **Details:** `["order-id"]`

The system now fails gracefully. No internal file paths or stack traces are disclosed to the user.


---

## ## Security Best Practices
* **Centralized Error Handling:** Always use `try-catch` blocks to capture unexpected errors and return generic messages.
* **Input Validation:** Never trust client-side data. Validate all required fields at the API boundary.
* **Least Privilege:** Ensure Lambda execution roles only have access to the specific resources they need to minimize the impact of a potential leak.
