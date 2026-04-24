# Security Analysis: Vulnerable Dependencies (RCE) in DVSA


This repository documents the identification, exploitation, and remediation of a critical vulnerability within the `DVSA-ORDER-MANAGER` Lambda function. The application relies on outdated third-party libraries (`node-serialize` and `node-jose`) that allow an attacker to execute arbitrary code within the serverless runtime.

---


## 1. Setup & Deployment
Before replicating the vulnerability, ensure the DVSA environment is deployed on AWS.

1.  **Deployment:** Deploy the DVSA stack using the AWS CloudFormation console or the Serverless Framework as instructed in the main project setup.
2.  **Environment:** Ensure you have access to the AWS Management Console and the **AWS Lambda** service.
3.  **Identity:** Locate the `DVSA-ORDER-MANAGER` function in the Lambda list. This function handles the primary order-routing logic and is the entry point for this analysis.

---

## 2. Vulnerability Replication (Discovery & Exploitation)
### The Root Cause
The `node-serialize` library (version 0.0.4) contains a fundamental flaw where it can reconstruct executable JavaScript objects from strings. If the string contains the prefix `_$$ND_FUNC$$_`, the library treats the following string as a function and executes it.


### Step 1: Identifying the Vulnerability (Video 1)
1.  **Audit Dependencies:** Open the `DVSA-ORDER-MANAGER` function in the AWS Lambda console. Open the `package.json` file.
2.  **Check Versions:** Observe that `node-serialize` is set to `0.0.4`.
    > **Note:** This version is publicly documented to be vulnerable to Remote Code Execution.
3.  **Locate the "Sink":** Open `order-manager.js`. Look for where the library is used. You will find:
    ```javascript
    const serialize = require('node-serialize');
    // ...
    const req = serialize.unserialize(event.body); 
    ```
    This line is highly dangerous because it takes untrusted user input (`event.body`) and passes it directly into the `unserialize()` function.


### Step 2: Crafting and Executing the Exploit
1.  **Craft Payload:** An attacker creates a JSON payload containing an **Immediately Invoked Function Expression (IIFE)**. This tells the server to run the code the moment it is parsed.
    ```json
    {
      "rce": "_$$ND_FUNC$$_function(){ require('child_process').exec('ls /tmp', function(error, stdout, stderr) { console.log(stdout) }); }() "
    }
    ```
2.  **Trigger the Attack:** Use `curl` or a browser-based tool to send this payload to the API Gateway endpoint associated with the Order Manager.
3.  **Verify via Logs:** Open **Amazon CloudWatch Logs** for the Lambda function. You will see the output of the `ls /tmp` command, proving that the runtime executed your injected code.

---

## 3. Remediation & Patching (Video 2)
### The Fix Strategy
The remediation strategy follows the principle of **minimizing the attack surface**. We remove the dangerous third-party libraries and replace them with native Node.js primitives that treat input strictly as data, not code.

### Step 1: Update the Dependency Manifest
1.  Open `package.json` in the Lambda editor.
2.  Delete the lines for `node-serialize` and `node-jose`.
3.  Click **Deploy**.

### Step 2: Patching the Source Code
Modify `order-manager.js` to use safe, built-in parsing and cryptography modules.

**1. Remove the vulnerable requirements:**
```javascript
// DELETE THESE LINES
const serialize = require('node-serialize');
const jose = require('node-jose');

// ADD NATIVE CRYPTO
const crypto = require("crypto");
```

**2. Implement Safe Parsing:**
Replace the `unserialize` call with `JSON.parse()`. Unlike `node-serialize`, `JSON.parse()` only interprets data and cannot execute functions.
```javascript
// PATCHED LOGIC
const req = typeof event.body === "string" ? JSON.parse(event.body) : (event.body || {});
```


### Step 3: Deployment and Verification
1.  **Deploy:** Click the **Deploy** button in the AWS Console to apply the changes to the live Lambda function.
2.  **Verify Neutralization:** Re-send the malicious `_$$ND_FUNC$$_` payload.
3.  **Observe Results:** The application will now treat the payload as a literal string. The `JSON.parse()` method will either parse it as a standard string or throw a syntax error if the JSON is malformed, but it will **not** execute the code. Check CloudWatch logs to confirm no unauthorized commands were executed.

---

## Security Best Practices
* **Dependency Auditing:** Regularly use tools like `npm audit` or Snyk to identify vulnerable packages in your project.
* **Favor Native Primitives:** Use built-in modules (like `JSON.parse` or the `crypto` module) instead of complex third-party libraries for basic tasks.
* **Strict Input Validation:** Never treat untrusted user input as executable logic. Always validate and sanitize input at the boundary of your application.

---
