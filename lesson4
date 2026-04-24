## Lesson #4: Insecure Cloud Configuration (Unrestricted File Upload)

---


## 🚀 Setup and Deployment

Before attempting the exploit, the DVSA environment must be correctly deployed.

1.  **AWS Deployment**: Deploy the official DVSA CloudFormation stack in a **non-production** AWS account.
2.  **Access the Frontend**: Identify the CloudFront or S3 website URL for your deployment.
3.  **Local Tools**: Ensure the following tools are installed on your local machine:
    * **ngrok**: For creating a public tunnel to your local listener.
    * **netcat (nc)**: To listen for incoming exfiltrated data.
    * **CyberChef**: To decode the Base64 payloads exfiltrated by the exploit.

---

## 🔍 Vulnerability Replication: Step-by-Step

The vulnerability exists because the application allows users to upload files via the "Contact" page without validating the file extension or content. This can be abused to execute commands or exfiltrate environment variables.

### **1. Set Up Your Listeners**
First, prepare your local environment to receive data from the vulnerable AWS Lambda function:
* **Open a terminal** and start an `ngrok` tunnel on port 8080:
    ```bash
    ngrok http 8080
    ```
* **Open a second terminal** and start a `netcat` listener:
    ```bash
    nc -lvp 8080
    ```

### **2. Prepare the Malicious Payload**
Create a file named `cat.png.python3`. Inside this file, insert a Python command designed to capture environment variables (like AWS keys and session tokens) and send them to your `ngrok` URL.

> **Note:** Redact your specific `ngrok` URL if sharing screenshots of this payload.

### **3. Trigger the Exploit**
1.  Navigate to the **Contact** page of your DVSA deployment.
2.  Fill in the "Your Name", "Your Email", and "Subject" fields with arbitrary data.
3.  Click **Attach File** and select your `cat.png.python3` file.
4.  Click **Send Feedback**.

### **4. Capture and Decode Data**
1.  Check your `netcat` terminal. You should see a `POST` request containing a long Base64 string.
2.  Copy this string and paste it into **CyberChef**.
3.  Apply the **"From Base64"** recipe.
4.  [cite_start]You will now see the Lambda function's environment variables, including `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN`[cite: 123, 232].

---

## 🛡️ Remediation & Patching

The root cause is a lack of input validation on the uploaded file's metadata. The fix involves implementing a server-side check to ensure only safe file types are processed.

### **1. Modify the Lambda Function**
1.  Log in to the **AWS Management Console** and navigate to **AWS Lambda**.
2.  Open the `DVSA-FEEDBACK-UPLOADS` function.
3.  In the code editor, locate the commented-out helper function named `is_safe()`.

### **2. Apply the Technical Fix**
Uncomment the `is_safe()` function and the logic within the main handler that calls it. This function checks the filename for dangerous characters or unauthorized extensions.

**Before (Vulnerable):**
```python
# The is_safe check was bypassed or commented out
# allows any file to trigger downstream processing
```

**After (Patched):**
```python
def is_safe(filename):
    # Ensure the filename does not contain command injection characters
    if ".." in filename or ";" in filename:
        return False
    return filename.endswith(('.png', '.jpg', '.jpeg'))

# In the handler:
if not is_safe(uploaded_file_name):
    return {"status": "err", "message": "Invalid file type"}
```

4.  [cite_start]Click **Deploy** to apply the changes[cite: 234].

---

## ✅ Verification After Fix

1.  **Re-upload Payload**: Attempt to upload the same `cat.png.python3` file on the Contact page.
2.  **Observe Frontend**: The application may still show a success message ("Thank you hacker"), but you must verify the backend behavior.
3.  **Check Listener**: Observe your `netcat` and `ngrok` terminals. **No data should be received**.
4.  [cite_start]**Confirm logs**: Check **Amazon CloudWatch** logs for the `DVSA-FEEDBACK-UPLOADS` function to confirm the `is_safe` logic correctly identified and blocked the malicious file[cite: 121, 416].

---

## ⚠️ Security Best Practices
* [cite_start]**Principle of Least Privilege**: Ensure the Lambda execution role does not have permissions to access sensitive S3 buckets unless absolutely necessary[cite: 104].
* [cite_start]**Redaction**: Never commit actual AWS keys or session tokens to your repository[cite: 532].

---
*This repository is for educational purposes only.*
