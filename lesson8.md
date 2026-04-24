# Security Analysis: Logic Vulnerability (Race Condition) in DVSA

## Project Overview
**Course:** ICS-344: Information Security  
**Institution:** King Fahd University of Petroleum and Minerals (KFUPM)  
**Target Application:** Damn Vulnerable Serverless Application (DVSA)  
**Vulnerability Focus:** Lesson #8 - Logic Vulnerabilities (Race Conditions) targeting the Order-Processing workflow.

This repository documents the identification and remediation of a business logic flaw in the DVSA order management system. The vulnerability allows an attacker to exploit a race condition between payment initiation and order finalization, resulting in the acquisition of multiple items for the price of one.

---

## Repository Structure
A clean organization is maintained to ensure all artifacts are easily accessible for review and replication:

```text
├── src/
│   ├── order_shipping_vulnerable.py # Original Lambda code with the logic flaw
│   └── order_shipping_patched.py    # Remediated code with atomic operations
├── config/
│   └── iam_policies.json            # Required IAM permissions for DynamoDB
├── scripts/
│   └── exploit_payload.json         # Sample JSON payload for Burp Suite Repeater
├── screenshots/                     # Proof of exploit and successful verification
└── README.md                        # Project documentation (this file)
```

---

## 1. Setup & Deployment
To deploy the DVSA environment for testing, follow these steps:

1.  **Clone the DVSA Repository:** Obtain the source from the official DVSA GitHub.
2.  **AWS Configuration:** Ensure your AWS CLI is configured with the necessary credentials.
3.  **Deployment:** Use the provided `deploy.sh` script or AWS CloudFormation console to deploy the stack.
    * *Services deployed:* API Gateway, AWS Lambda (including `DVSA-ORDER-SHIPPING`), and Amazon DynamoDB (`OrdersTable`).
4.  **Frontend Access:** Once the stack is deployed, navigate to the generated S3/CloudFront URL to access the DVSA storefront.

---

## 2. Vulnerability Replication (Exploitation)
### The Root Cause
The vulnerability stems from a **"Read-Modify-Write"** pattern in the `order_shipping.py` function. The code first reads the order status from DynamoDB, checks if it is already paid in the application logic, and then performs an update. Because these are two separate operations, a "race window" exists where a second request can modify the cart after the price is locked but before the order is marked as "paid."


### Step-by-Step Replication
1.  **Select Item:** Add a single item (e.g., Pac-Man for $33) to your cart and proceed to checkout.
2.  **Shipping Details:** Enter the required shipping information and proceed to the payment page.
3.  **Intercept Traffic:** Open **Burp Suite**, ensure the Intercept is **ON**, and submit the payment.
4.  **Send to Repeater:** Locate the intercepted `POST /order` request (where `action: billing`). Right-click and select **"Send to Repeater"**.
5.  **Prepare Exploit:**
    * In the Repeater tab, copy the request.
    * Modify the payload in the second request to include more items or change the order details.
6.  **Execute Race Condition:** Send the original billing request and the modified update request **concurrently** (rapidly one after the other).
7.  **Verify:** Observe that the backend returns `200 OK` for both. Check your orders in the UI; you will see multiple items listed, but the total price remains $33.

---

## 3. Remediation & Patching
### The Fix Strategy: Atomic Updates
To fix this, we move the state check from the application code directly into the database operation. By using a **DynamoDB Condition Expression**, we ensure that the update only occurs if the `orderStatus` is still below the "locked" threshold (e.g., `< 200`) at the exact moment of the write.

### Technical Implementation
1.  **Access Lambda Console:** Navigate to the `DVSA-ORDER-SHIPPING` function.
2.  **Modify `order_shipping.py`:** Remove the separate `get_item` check and implement the following atomic update:

#### Crucial Patch Code:
```python
import botocore

# ... inside the handler ...
try:
    response = table.update_item(
        Key={
            "orderId": order_id,
            "userId": user_id
        },
        UpdateExpression="SET #address = :address",
        # This condition ensures the write only happens if the order is not yet locked/paid
        ConditionExpression="attribute_exists(orderId) AND attribute_exists(userId) AND orderStatus < :locked_status",
        ExpressionAttributeNames={
            "#address": "address"
        },
        ExpressionAttributeValues={
            ":address": address,
            ":locked_status": 200 # Status 200 represents 'Paid/Locked'
        },
        ReturnValues="UPDATED_NEW"
    )
except botocore.exceptions.ClientError as e:
    if e.response['Error']['Code'] == "ConditionalCheckFailedException":
        return {"status": "error", "msg": "Too late to update order; order already processed."}
    raise e
```

3.  **Deploy Changes:** Click **Deploy** in the Lambda console.

---

## 4. Verification After Fix
1.  **Repeat Exploit:** Re-run the Burp Suite Repeater attack using the same concurrent requests.
2.  **Observe Failure:** The first request (payment) will succeed, but the second (cart modification) will now return a `400 Bad Request` or the custom error message: `"Too late to update order"`.
3.  **Data Integrity:** Check the database or UI to confirm that the order contents remained unchanged after the payment was initiated.

---

## Security Best Practices
* **Avoid Client-Side Trust:** Never assume that the state checked at the beginning of a function will remain the same by the end.
* **Database Atomicity:** Always use database-level locks or conditional expressions (`ConditionExpression` in DynamoDB, `WHERE` clauses in SQL) to handle state transitions.
* **Fail Securely:** Ensure that if a condition fails, the system returns a clear error and does not proceed with the restricted operation.

---
> **Note:** This analysis was performed in a controlled environment for educational purposes. No real secrets or AWS keys were exposed during this documentation process.
