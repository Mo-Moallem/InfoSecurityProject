This `README.md` is designed to be a definitive, step-by-step guide for a beginner to replicate your work on **Lesson #10: Unhandled Exceptions**. [cite_start]It follows the specific repository requirements for the **ICS-344** course at **KFUPM**[cite: 4, 140].

---

# 🛡️ DVSA Vulnerability Analysis: Unhandled Exceptions
## Project: OWASP Damn Vulnerable Serverless Application (DVSA)
[cite_start]**Course:** ICS-344: Information Security [cite: 1]  
[cite_start]**Institution:** King Fahd University of Petroleum and Minerals (KFUPM) [cite: 4]

---

## 📂 Repository Organization
[cite_start]This repository is organized to allow for easy replication of the security audit[cite: 474, 475]:
* **`/remediation`**: Contains the patched `index.js` for the Lambda function.
* **`/evidence/exploit`**: Screenshots showing the leaked stack traces and file paths.
* **`/evidence/verification`**: Screenshots showing the sanitized backend responses after the fix.
* **`/payloads`**: Sample JSON payloads used to trigger the vulnerability.

---

## 🛠️ Prerequisites & Setup
Before starting, ensure you have the following tools and environment ready:
1.  [cite_start]**AWS Account**: A non-production account where DVSA is deployed[cite: 20].
2.  **Burp Suite**: Community or Professional edition for intercepting and modifying HTTP traffic.
3.  [cite_start]**DVSA Deployment**: Ensure the stack is deployed and you have the URL of your S3-hosted frontend[cite: 15, 50].

---

## 🔍 Part 1: Finding the Vulnerability (The Exploit)
[cite_start]The goal of this phase is to demonstrate how malformed requests can cause the backend to leak internal system details[cite: 344].

### **Step 1: Intercept Legitimate Traffic**
1.  Open your DVSA website and navigate to the **"My Orders"** page.
2.  Open **Burp Suite** and ensure "Intercept is ON" in the Proxy tab.
3.  Refresh the "My Orders" page. Burp will catch the `POST` request sent to the API Gateway.

### **Step 2: Send to Repeater**
1.  Right-click the intercepted request in Burp and select **"Send to Repeater"**.
2.  Go to the **Repeater** tab to manually modify the JSON payload.

### **Step 3: Trigger the Exception**
1.  Locate the JSON body in the request. It should look like this:
    ```json
    { "action": "orders" }
    ```
2.  Change the value of `"action"` to something the backend does not expect, such as `"get"`.
3.  Click **Send**.

### **Step 4: Analyze the Data Leak**
* [cite_start]**The Vulnerability**: The server returns a `500 Internal Server Error` containing a massive `errorStack`[cite: 345].
* [cite_start]**What is Leaked?**: You will see internal file paths (e.g., `/var/task/get_order.py`), specific line numbers, and the logic sequence of the Lambda function[cite: 345]. [cite_start]This information helps an attacker understand the backend architecture for more advanced attacks[cite: 79, 345].

---

## 🛡️ Part 2: Patching the Vulnerability (The Remediation)
[cite_start]The goal is to implement centralized error handling to ensure the backend only returns "client-safe" messages[cite: 346].

### **Step 1: Open the AWS Lambda Console**
1.  Log in to the **AWS Management Console**.
2.  Navigate to **Lambda** > **Functions**.
3.  Find and click on the function named `DVSA-ORDER-MANAGER`.

### **Step 2: Implement the Try-Catch Block**
1.  In the **Code** tab, locate the main handler function.
2.  Wrap the entire logic inside a `try...catch` block.
3.  [cite_start]Modify the `catch` block to log the error to **CloudWatch** (for your eyes only) and return a generic response to the user[cite: 346].

**Original (Vulnerable) Logic:**
```javascript
exports.handler = async (event) => {
    // Logic here crashes when action is "get" and returns a raw stack trace
    let result = await processAction(event.action);
    return result;
};
```

**Patched (Secure) Logic:**
```javascript
exports.handler = async (event) => {
    try {
        let result = await processAction(event.action);
        return result;
    } catch (error) {
        [cite_start]// 1. Log the real error to CloudWatch for debugging [cite: 116]
        console.error("Internal Error Details:", error);

        [cite_start]// 2. Return a sanitized, generic error to the client [cite: 346]
        return {
            statusCode: 400,
            body: JSON.stringify({
                "status": "err",
                "message": "Invalid request or missing parameters."
            })
        };
    }
};
```

### **Step 3: Deploy and Save**
1.  Click the **Deploy** button to update the function in the AWS Cloud.

---

## ✅ Part 3: Verification After Fix
[cite_start]To ensure the fix works and didn't break the app[cite: 416]:

1.  **Repeat the Exploit**: Go back to **Burp Suite Repeater** and send the malformed `"action": "get"` request again.
2.  [cite_start]**Observe the Change**: You should now receive a clean, short JSON error message without any stack traces or file paths[cite: 419].
3.  **Confirm Legitimate Use**: Change the action back to `"orders"` and click Send. [cite_start]The application should still return your order history correctly[cite: 418].

---

## 📌 Key Security Takeaway
[cite_start]In a serverless environment, **Observability** should be handled internally via tools like **Amazon CloudWatch**, while external users should never see backend crashes[cite: 116, 346]. [cite_start]This follows the principle of **Defense in Depth**[cite: 16].

---
*Note: This documentation is part of an academic project. [cite_start]Do not use these techniques on systems you do not own[cite: 131].*
