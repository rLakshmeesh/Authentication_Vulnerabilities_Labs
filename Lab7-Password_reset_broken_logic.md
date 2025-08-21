# Lab 7: Password reset brocken logic

## Vulnerability Type
  - Broken Password Reset Functionality
  - Improper Validation of Password Reset Token
  - Insecure Direct Object Reference (IDOR)-like issue (because attacker can change the username and reset another user’s password).

### OWASP Top 10 Mapping
  - A01:2021 – Broken Access Control
  - A07: 2021 – Identification and Authentication Failures

---

## Description
In this lab, the application implements a flawed password reset mechanism.  
When a user initiates a password reset, the server issues a temporary token (`temp-forgot-password-token`) which is included in both the **URL** and the **request body**.  

However, the application fails to properly validate that this token is strictly tied to the requesting account.  
By intercepting the request and modifying the **username field** to that of a victim, while keeping or tampering with the token, an attacker can successfully reset another user’s password.  

This results in an HTTP **302 Found** response indicating the reset was accepted.  
The attacker can then log in to the victim’s account using the newly set password.

---

## Tools Used
- **Burp Suite** (Proxy, Intruder, Grep Match/Extract)
- **Hydra** *(optional: requires explicit written permission)*

---

## Step-by-Step Exploitation

1. **Prepare & capture**
  - Open the target site with **Burp Proxy** on.
  - Click **Forgot password** and go through the normal flow for **your own account** so you can observe the traffic.

2. **Identify the reset request**
  - In **Proxy → HTTP history**, find the password reset **submission** request ( POST like `/forgot-password`).
  - Confirm that a `temp-forgot-password-token` appears **both**:
    - In the **URL/query** (e.g., `.../reset?temp-forgot-password-token=abc123`)
    - In the **body** (e.g., `temp-forgot-password-token=abc123&username=youruser&new-password=...`)

3. **Send to Repeater**
  - Right-click the reset submission request → **Send to Repeater**.

4. **Craft the malicious reset**
  - In **Repeater**, change:
    - `username=` from **your username** to the **victim’s username** (e.g., `carlos`).
    - **Token tampering**: either **blank out** the token value in **both** places or make them **mismatched/empty**:
      - URL: `temp-forgot-password-token=`
      - Body: `temp-forgot-password-token=`
  - Set a **new password** you control in `new-password` (e.g., `NewPass!123`).
  
### 🔎 Burp Request — Before (example)
```http
POST /forgot-password?temp-forgot-password-token=abc123 HTTP/1.1
Host: vulnerable-app.com
Content-Type: application/x-www-form-urlencoded

username=wiener&temp-forgot-password-token=abc123&new-password=MySecurePass1!
```
### 🔎 Burp Request — after (example)
```http
POST /forgot-password?temp-forgot-password-token= HTTP/1.1
Host: vulnerable-app.com
Content-Type: application/x-www-form-urlencoded

username=carlos&temp-forgot-password-token=&new-password=NewPass!123
```


5. **Send the request**
  - Click **Send** in Repeater.

6. **Confirm success via response**
  - Look for **`302 Found`** (Location redirect) indicating the password was reset.
  - If using Burp, hit **Response** to view the landing page and messages.

7. **Verify by logging in as the victim**
- In the browser, log in using:
  - `username = victim` (e.g., `carlos`)
  - `password = NewPass!123` (the one you set)
- You should gain access to the victim’s account page.

8. **Document impact**
  - State clearly: **anyone with a captured/blank token can reset arbitrary users’ passwords by changing `username`, due to        missing server-side binding between token and account**.


---

## Mitigation

1. **Bind reset tokens to specific accounts**  
   - Each password reset token must be generated uniquely per user and stored securely on the server side.  
   - The token should only be valid for the intended account and cannot be reused for other usernames.  

2. **Validate tokens strictly**  
   - The server should always validate the reset token against the correct user.  
   - Reject any request where the token is missing, blank, expired, or does not match the username.  

3. **Expire tokens quickly**  
   - Reset tokens should have short lifetimes (e.g., 10–15 minutes).  
   - Once used, they must be invalidated immediately.  

4. **Do not allow multiple reset parameters**  
   - The reset token should not appear in both URL and body.  
   - Keep it only in a secure channel (e.g., in the request body or as a signed link).  

5. **Rate limit reset attempts**  
   - Implement throttling to prevent brute-force or automated abuse of the reset functionality.  

6. **Audit and monitoring**  
   - Log all password reset requests and monitor for suspicious activity such as resets for multiple accounts from the same IP.  

---

✅ **Secure Implementation Example:**  
  - User requests reset → Server generates random, cryptographically secure token → Token stored against that user in DB → Link sent via email → On submission, server validates both the token *and* username match before allowing password change.  

---

## Real-World Relevance

Broken password reset flows are a **common real-world vulnerability** because password reset functionality is present in almost every web application. If implemented incorrectly, attackers can reset passwords of other users without authorization.

### Examples in the wild:
  1. **TalkTalk (2016)** – A broken password reset system allowed attackers to hijack accounts by manipulating reset tokens.  
  2. **GitHub (2014)** – A flaw in GitHub’s password reset logic could let attackers reuse reset tokens, potentially hijacking accounts.  
  3. **Many smaller SaaS platforms** – Security researchers frequently report bugs where reset tokens are not properly validated against the intended user, allowing account takeover.  

### Why it matters:
  - Attackers can take over **critical accounts (admin, financial, healthcare, or cloud accounts)**.  
  - It leads to **identity theft, fraud, and data breaches**.  
  - Exploits often require **no user interaction beyond requesting a reset**, making them highly dangerous.  
  - Automated attacks can **reset thousands of accounts quickly** if proper controls are missing.  
