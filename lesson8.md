This guide provides a detailed walkthrough for reproducing a **Race Condition** logic vulnerability within the Damn Vulnerable Serverless Application (DVSA). This specific exploit allows an attacker to purchase multiple items for the price of one by manipulating the order state between the billing and finalization phases.

---

# Lesson 8: Exploiting Race Conditions in Order Logic

A **Race Condition** occurs when a system's behavior depends on the sequence or timing of uncontrollable events. In web applications, this often happens when multiple requests are processed simultaneously, allowing an attacker to modify data (like a cart) after the price has been calculated but before the transaction is finalized.


## Prerequisites
* **Burp Suite** (Community or Professional)
* Access to the **DVSA** web application
* A browser configured to proxy traffic through Burp Suite

---

## Step-by-Step Reproduction

### 1. Initialize the Cart
1.  Navigate to the DVSA store page.
2.  Select the **Pac-Man (Atari 2600)** product.
3.  Click **Add to Cart**. Ensure the quantity is set to **1** (Price: **$33**).
4.  Navigate to the cart and click **Check Out**.

### 2. Enter Shipping & Card Details
1.  Fill in the **Shipping Details** (Name, Address, etc.) and click Submit.
2.  On the **Enter Card Details** page, input a test card number (e.g., `4242 4242 4242 4242`).
3.  **Stop!** Before clicking Submit, ensure Burp Suite is open.

### 3. Intercept the Billing Request
1.  In Burp Suite, go to the **Proxy** tab and turn **Intercept is on**.
2.  Return to the browser and click **Submit**.
3.  Burp will catch the `POST /dvsa/order` request. 
    * Verify the body contains `"action":"billing"`.
    * Note the `order-id` (e.g., `7a50939d-714e-4ba6-9ff4-f3fdaa3f3c0a`).

### 4. Prepare the Exploit (Repeater Setup)
1.  Right-click the intercepted request and select **Send to Repeater**.
2.  Switch to the **Repeater** tab.
3.  Modify the JSON body of the request to change the `action` and `items` count. This request will "update" the cart quantity while the server is still processing the initial payment.

**Original Payload:**
```json
{
    "action":"billing",
    "order-id":"YOUR_ORDER_ID",
    "data":{
        "ccn":"4242424242424242",
        "exp":"09/29",
        "cvv":"999"
    }
}
```

**Modified Payload for Repeater:**
```json
{
    "action":"update",
    "order-id":"YOUR_ORDER_ID",
    "items": {
        "1013": 5
    }
}
```

### 5. Execute the Race Condition
This is the critical step. You must send the billing request and the update request in rapid succession so the update occurs during the billing processing window.

1.  In the **Proxy** tab, prepare to click **Forward** on the intercepted "billing" request.
2.  In the **Repeater** tab, prepare to click **Send** on the "update" request.
3.  Click **Forward** in Proxy, then **immediately** switch to Repeater and click **Send**.
4.  If successful, the Repeater response should show:
    ```json
    {
        "status":"ok",
        "msg":"cart updated"
    }
    ```

### 6. Verify the Results
1.  Go back to the **Proxy** tab and **Forward** any remaining queued requests (like the final order confirmation).
2.  The application will redirect you to the receipt page.
3.  **The Result:** The "Total Price" will still display **$33**, but the quantity of items purchased will reflect the updated amount (e.g., **5 items**).

---

## Technical Summary

| Component | Detail |
| :--- | :--- |
| **Vulnerability Type** | Logic Flaw / Race Condition |
| **Target Endpoint** | `POST /dvsa/order` |
| **Impact** | Financial loss; unauthorized inventory acquisition |
| **Remediation** | Implement atomic transactions and state locking to prevent cart modifications once the billing process has initiated. |

> **Note:** Successful reproduction requires precise timing. If the order completes before the update request is processed, the quantity will remain at 1. If the update happens too early, the price may recalculate correctly. The goal is to hit the "window" between authorization and finalization.
