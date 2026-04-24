# Security Analysis: Over-Privileged Function (IAM) in DVSA

---

## 1. Setup & Deployment
To deploy the DVSA application for testing:

1.  **Clone the DVSA Repo:** `git clone https://github.com/m6000/dvsa`
2.  **Infrastructure as Code (IaC):** Use the AWS CloudFormation or SAM templates provided in the original repository to spin up the infrastructure.
3.  **Identify Target:** Ensure the `DVSA-SEND-RECEIPT-EMAIL` and `DVSA-ORDER-MANAGER` functions are active in your AWS console.
4.  **Configuration:** Verify that the `DVSA-ORDERS-DB` DynamoDB table and a Cognito User Pool have been created.

---

## 2. Vulnerability Replication (Exploitation)
### The Root Cause
The vulnerability exists because the Lambda's **Execution Role** is attached to an AWS Managed Policy (e.g., `AdministratorAccess` or `AmazonS3FullAccess`) or contains `Resource: "*"` for sensitive actions. This violates the **Principle of Least Privilege**.

### Step-by-Step Discovery
1.  **Examine the Role:** Navigate to **Lambda > Functions > DVSA-SEND-RECEIPT-EMAIL**. Under **Configuration > Permissions**, click on the Execution Role name.
2.  **Identify Broad Permissions:** In the IAM console, observe policies that grant access to resources unrelated to sending emails (e.g., `cognito-idp:*` or `dynamodb:Scan` on all resources).
3.  **Extract STS Credentials:** Use an existing entry point (such as an Event Injection vulnerability) to execute code that leaks environment variables.
    ```bash
    # Payload sent via curl to the vulnerable endpoint
    {
      "receipt": "'; require('os').environ; //"
    }
    ```
4.  **Harvest Tokens:** Capture the following values from the logs or your listener:
    * `AWS_ACCESS_KEY_ID`
    * `AWS_SECRET_ACCESS_KEY`
    * `AWS_SESSION_TOKEN`

### Exploitation (Assuming the Identity)
1.  **Local Configuration:** Export the stolen credentials into your local terminal:
    ```bash
    export AWS_ACCESS_KEY_ID="[STOLEN_KEY]"
    export AWS_SECRET_ACCESS_KEY="[STOLEN_SECRET]"
    export AWS_SESSION_TOKEN="[STOLEN_TOKEN]"
    ```
2.  **Verify Masquerade:** Run `aws sts get-caller-identity`. The output will confirm you are now acting with the Lambda's role.
3.  **Cross-Service Data Theft:** List all users in the Cognito pool, an action the email function should never be allowed to do:
    ```bash
    aws cognito-idp list-users --user-pool-id [POOL_ID]
    ```

---

## 3. Remediation & Patching
### The Fix Strategy: Least Privilege
To mitigate this, we replace the broad AWS Managed policies with a **Customer Managed Policy** that explicitly defines the required actions and restricts them to specific Resource ARNs.

### Technical Implementation
1.  **Create Restricted Policy:** In the IAM console, create a new policy (e.g., `dvsa-restricted-lambda-policy`).
2.  **Apply Scoped JSON:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ses:SendEmail",
                "ses:SendRawEmail"
            ],
            "Resource": "arn:aws:ses:us-east-1:123456789012:identity/*"
        },
        {
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": "arn:aws:lambda:us-east-1:123456789012:function:DVSA-ORDER-*"
        }
    ]
}
```
3.  **Detach/Attach:** Remove the old over-privileged policies from the role and attach this new restricted policy.

---

## 4. Verification After Fix
1.  **Attempt Exploit:** Re-export fresh STS tokens (if required) and attempt to list Cognito users again.
2.  **Observe Failure:** The command should now return an `AccessDeniedException`.
    ```text
    An error occurred (AccessDeniedException) when calling the ListUsers operation: 
    User is not authorized to perform: cognito-idp:ListUsers on resource...
    ```
3.  **Functional Check:** Trigger a legitimate order checkout. Confirm the `DVSA-SEND-RECEIPT-EMAIL` still works for its intended purpose, proving that the restricted policy allows necessary traffic while blocking malicious actions.

---

## Security Best Practices
* **Zero Trust IAM:** Never use `Resource: "*"` unless absolutely necessary.
* **IAM Policy Simulator:** Use this AWS tool during development to test what a role can and cannot do before deployment.
* **Service Control Policies (SCPs):** Use SCPs at the AWS Organization level to provide guardrails that prevent roles from ever gaining administrative access in production environments.

---
