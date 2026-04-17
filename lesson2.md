This guide outlines the process of exploiting **Broken Authentication** within the Damn Vulnerable Serverless Application (DVSA). We'll manipulate a JSON Web Token (JWT) to perform an **Insecure Direct Object Reference (IDOR)** attack, allowing us to view orders belonging to another user.

---

## 🛠️ Prerequisites

Before starting, ensure you have the following tools ready:
* **Burp Suite:** For intercepting and repeating HTTP requests.
* **CyberChef:** To handle Base64 decoding and encoding.
* **JWT.io:** To inspect and debug the token structure.
* **DVSA Instance:** Access to the vulnerable web application.

---

## 🔓 Step-by-Step Reproduction

### 1. Capture the Valid Token
First, we need a legitimate session to work with. Log in to the application as **User B** (the attacker).

1.  Open your browser's **DevTools** (F12) and navigate to the **Application** tab.
2.  In the left sidebar, look under **Local Storage** or **Cookies**. Alternatively, use **Burp Suite** to intercept a request.
3.  Locate the `Authorization` header or a stored token. This is typically a long string of characters (the JWT).

### 2. Identify the Target
To impersonate another user, you need their unique identifier (`sub`). 

> [!TIP]
> In a real-world scenario, you might find these IDs through public profiles, URL parameters, or administrative leaks. In this environment, you can find the target's `sub` (User ID) via the AWS CloudFormation outputs or the user management dashboard.

* **Target User (User C) ID:** `4b803468-b881-707a-f8f4-debd1534a78c` (example from video).

### 3. Decode and Forge the Token
Now, we will modify our own token to point to the target user.

1.  **Decode:** Copy your token into **CyberChef**. Use the `JWT Decode` or `From Base64` recipes.
2.  **Edit Payload:** Locate the JSON payload. You will see fields like `sub` and `username`.
3.  **Modify:** Replace your `sub` and `username` with the values for **User C**.
4.  **Re-encode:** Use the `To Base64` recipe in CyberChef to convert the modified JSON back into a string.


```json
// Original Payload Snippet
{
  "sub": "your-uuid-here",
  "username": "User_B",
  ...
}

// Forged Payload Snippet
{
  "sub": "4b803468-b881-707a-f8f4-debd1534a78c",
  "username": "User_C",
  ...
}
```

### 4. Execute the Attack
We will now use the forged token to request sensitive data.

1.  In **Burp Suite**, go to the **Proxy** tab and ensure **Intercept is ON**.
2.  In the DVSA web app, click on the **Orders** section.
3.  Intercept the `POST` or `GET` request to `/orders`.
4.  Right-click the request and select **Send to Repeater**.
5.  In the **Repeater** tab, replace the original `Authorization` token with your **forged token**.
6.  Click **Send**.

### 5. Verify the Vulnerability
Observe the **Response** pane in Burp Suite.

* **Success:** If the response contains order details (Order ID, Date, Total) for User C, the vulnerability is confirmed.
* **The Flaw:** The server is trusting the `sub` and `username` claims provided by the client without properly verifying the JWT signature against a secret key.

---

## 🛡️ Remediation Strategies

To fix this vulnerability, the following software architecture principles should be applied:

* **Signature Verification:** The backend **must** verify the JWT signature using a secure, server-side secret key or public key (RS256). If the signature doesn't match the payload, the request must be rejected.
* **Strict Access Control:** Never rely solely on client-side identifiers. The server should cross-reference the authenticated user's identity (validated via the token) with the requested resource.
* **Use Standard Libraries:** Avoid manual Base64 manipulation for auth logic; use established libraries (like `jsonwebtoken` for Node.js or `System.IdentityModel.Tokens.Jwt` for .NET) that handle validation automatically.

Would you like to explore how to implement a secure JWT validation middleware in a specific language?
