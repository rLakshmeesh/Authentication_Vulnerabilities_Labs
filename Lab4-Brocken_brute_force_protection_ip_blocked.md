# Lab4 : Broken Brute-Force Protection, IP Block

## Vulnerability Type
Authentication Flaw

### OWASP Top 10 Mapping
A07: 2021 – Identification and Authentication Failures

---

## Description
In this lab, the application attempts to block brute-force login attempts by counting failed authentication tries per IP address. After a certain number of failures, the system temporarily blocks further attempts.  

However, we discovered a **flaw in the implementation**:  
- The counter resets if a successful login attempt occurs in between.  
- This means an attacker can try **two wrong passwords**, then submit the correct password (to reset the counter), and repeat the cycle.  

This design oversight enables attackers to **brute-force passwords indefinitely** without triggering IP block protection, rendering the brute-force defense ineffective.  

---

## Tools Used
- **Burp Suite** (Proxy, Intruder, Grep Match/Extract)
- **Hydra** *(optional: requires explicit written permission)*

---

## Step-by-Step Exploitation

1 **Scope & capture**
- Add the target to **Burp → Target → Scope**.
- Intercept a normal **POST /login** and **Send to Repeater** and **Intruder**.
- Identify the parameters (e.g., `username`, `password`) and note any success indicators (redirect to `/my-account`, success banner, different content length/status).

2 **Prove the reset flaw (manual)**
- From **Repeater**, send **two wrong passwords** for the same `username`. Observe the “failed attempts” counter rising (e.g., different message or nearing IP block threshold).
- Now send the **correct password** for that username. Confirm successful login (redirect / status / content length).
- Immediately send another **wrong password**. You should see that the protection **did not trigger** (counter has reset to 0 after the success).  
  > Conclusion: The app **resets the brute-force/IP counter after any successful login from the same IP**.

3 **Prepare the automated attack (pattern: 2 wrong + 1 correct)**
- Goal: test many candidate passwords while **inserting one known good password** every third request to keep the counter at zero.
- In **Intruder**, keep a single payload position around the password value:  
  `password=§PAYLOAD§`  
  (Keep `username` fixed to the target account.)

4 **Build the interleaved password list**
- Create a wordlist that **repeats the pattern** using python script or using AI:  
  `wrong1`  
  `wrong2`  
  `KNOWN_GOOD_PASSWORD`  
  `wrong3`  
  `wrong4`  
  `KNOWN_GOOD_PASSWORD`  
  … and so on.
- Do same for the username also.
- Use this list as the **only payload set** (Intruder **Cluster bombing** is fine since there’s two position).
  - Tip: If your candidate list is large, generate the interleaved list with a quick script (or text editor macro) so every 2 candidates are followed by the known good password.

5 **Configure Intruder**
- **Attack type:** Cluster bombing (two position).
- **Payloads:** load your **interleaved** list.
- **Options → Grep – Match / Extract:** add a marker for login **success** (e.g., `Location: /my-account`, “Welcome”, or a session cookie pattern) so you can see when the reset happened.
- **Columns:** enable **Status**, **Length**, and **Time** to spot behavioral changes you can also use grep extract message.
- **Request Engine:** keep rate modest (e.g., 2–5 req/s) to avoid other noisy rate limits.

6 **(Optional) Avoid per-IP lockouts entirely**
- If the app still blocks aggressively per IP despite your 2W+1C pattern, add a header **only if the app trusts it**:


---

## Mitigation 

- Enforce strict account lockout after a defined number of failed attempts (e.g., 5 attempts).  
- Do not reset the failed login attempt counter immediately after a successful login.  
- Implement both IP-based and account-based throttling to prevent distributed attacks.  
- Use exponential backoff to gradually increase login delay after repeated failures.  
- Log and monitor all failed login attempts for detection of brute-force patterns.  
- Alert security teams on suspicious login behavior (multiple attempts, different accounts, etc.).  
- Require Multi-Factor Authentication (MFA) to add an extra security layer.  
- Avoid relying solely on headers like `X-Forwarded-For` for IP blocking unless from trusted proxies.  
- Notify users after multiple failed login attempts or suspicious activity.  
  

---

## Real-World Relevance

**1. Credential Stuffing Attacks**  
      - Attackers often use leaked username/password lists (from breaches) and test them on multiple websites.  
      - Username enumeration makes it easier to validate which accounts actually exist, speeding up attacks.  

**2. Phishing & Social Engineering**  
      - Knowing valid usernames allows attackers to craft **targeted phishing emails** (e.g., “Dear John, we noticed a login attempt…”).  

**3. Account Takeover (ATO)**  
      - If attackers can confirm valid usernames, they can launch **brute-force or password-spray attacks** to hijack accounts.  

**4. Regulatory & Compliance Risks**  
      - Applications that expose usernames may violate **security standards** such as OWASP ASVS, PCI DSS, or GDPR (leak of PII like emails/user IDs).  

**5. High-Profile Breaches**  
      - Many real-world data breaches (e.g., LinkedIn, Dropbox, Yahoo) started with attackers confirming valid usernames before escalating to full account compromise.  
