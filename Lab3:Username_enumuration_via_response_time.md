# Lab : Username Enumeration via response time

## Vulnerability Type
Authentication Flaw

### OWASP Top 10 Mapping
A07: 2021 – Identification and Authentication Failures

---

## Description
This vulnerability occurs when the time taken by the application to respond to login requests reveals whether a username is 
valid. In this lab, if an attacker submits a login request with an invalid username, the server quickly rejects the request.
However, when a valid username is supplied with an incorrect password, the server takes longer to respond because it proceeds 
to check the password before rejecting the attempt. This difference in response times allows attackers to reliably enumerate 
valid usernames, even without visible differences in error messages.

The application also included an IP blocking mechanism that **temporarily blocked** login attempts from the same IP address for 
30 minutes after repeated failures. While this adds a layer of protection, it is *not foolproof*. Attackers can bypass this 
defense by distributing requests across multiple **IP addresses** *(by adding X-Forworded-for: header in the request)*, using botnets 
or proxy networks, or by slowing down the rate of their requests to avoid triggering the block. Once valid usernames are 
identified, attackers can still perform targeted **brute-force or credential stuffing attacks**, significantly reducing the effort 
needed to compromise user accounts.

---

## Tools Used
- **Burp Suite** (Proxy, Intruder, Grep Match/Extract)
- **Hydra** *(optional: requires explicit written permission)*

---

## Step-by-Step Exploitation


1) **Scope & intercept** 
   - In Burp, set the target host in **Scope**.  
   - Intercept a normal login attempt and **Send to Repeater** and **Intruder**.

2) **Establish timing baselines in Repeater** 
   - Send a few requests with a clearly **invalid username** (e.g., `notarealuser123`) and a dummy password that is lengthy so we can
     see an visible time difference. Note the **Time** shown by Repeater (top right).  
   - Send a few with a **candidate username** from your list (or any suspected valid name) and the same wrong password.  
   - You should observe: **invalid username ⇒ faster rejection**, **valid username ⇒ slower response** (server checks password).
     Keep rough averages in mind.
   - This website also have the IP blocking website as we do multiple login attempt it blocks us from login for 30 min. So we also
     need to find any way to bypass this blocking

3) **Prepare the Intruder positions** 
   - In the captured **POST /login** request, highlight the **username** value and mark it as a payload position: `username=§candidate§`.  
   - Choose a single constant wrong password (e.g., `Password1!`) to keep password cost stable for timing.  
   - **Add a header** line manually under existing headers:  
     ```
     X-Forwarded-For: §IP§
     ```
     This will be the second payload position for rotating “source IPs”.

4) **Choose attack type**  
   - Set Intruder **Attack type = Pitchfork** (so payload set #1 = usernames, payload set #2 = IPs line up one-to-one).
   - We can generate the series of number in burp suit itself by selecting paylod position and go through the option in by changing
     simple list to other. 
   - Make sure your **IP list is longer** than your username list to avoid running out.

5) **Configure payload set #1 (usernames)**  
   - **Payload type:** Simple list (load your username wordlist).

6) **Configure payload set #2 (rotating IPs via `X-Forwarded-For`)**  
   - **Payload type:** Numbers.  
   - **From:** 10, **To:** 250, **Step:** 1 (or any range big enough).  
   - In **Payload Processing**, add a **Prefix** like `12.34.56.` so requests use `12.34.56.10`, `12.34.56.11`, …  
     > You can also use ranges like `198.51.100.` / `203.0.113.`—the app typically only parses the string, not whether it’s a
     real IP.  
   - Result: each request carries a **different `X-Forwarded-For` IP**, mitigating the 30-minute per-IP lock.

7) **Request engine & throttling**  
   - In **Intruder → Options → Request engine**, start modestly (e.g., 2–5 req/s) to keep noise down and avoid other rate-limits.

8) **Make timing visible in results**  
   - In Intruder’s results table, **enable the “Time” column** (right-click a column header → Columns → Time).  
   - Optionally add **Grep – Extract** for any token/marker in the response to confirm the same code path, but timing is the
     key signal here.

9) **Run the attack & analyze**  
   - Start the attack.  
   - **Sort by Time**. Entries with **consistently higher response times** across repeats are likely **valid usernames**
     (server performed a password check).  
   - Sanity-check a few candidates back in **Repeater** (send several times to reduce jitter). Look for a stable gap
     (e.g., ≥200–300 ms) versus invalids.

10) **Understand and handle IP blocking (30 minutes)**  
   - If you hammer from one IP, the app may return **429/403** (or similar) and block further attempts for **~30 minutes**.  
   - Because we’re rotating `X-Forwarded-For`, the server’s **per-IP limiter** sees each request as coming from a **different
     client**, so the block is much harder to trigger.  
   - If the app ignores `X-Forwarded-For`, try `X-Real-IP` or `Forwarded: for=§IP§`, or reduce rate / add delays.

11) **Move to password phase (optional)**  
   - For confirmed valid usernames, rerun Intruder with:  
     - **Position 1:** password list,  
     - **Position 2:** `X-Forwarded-For` rotating IPs (Pitchfork again),  
     - Keep username fixed.  
   - Watch for **status/length/timing** changes (e.g., redirect to dashboard or different content length).

### ℹ️ About `X-Forwarded-For` (XFF)
`X-Forwarded-For` is a de-facto header used by proxies/load balancers to tell the upstream app the **original client IP**. 
Many rate-limiting and security controls **trust** this header. If the app trusts XFF **without validation**, an attacker can 
**spoof a new IP each request**, sidestepping **per-IP lockouts** like the 30-minute block in this lab.


---

## Mitigation 

- Implement **uniform response times** so valid and invalid usernames take the same time to process.  
- Use **generic error messages** like *“Invalid username or password”* without revealing details.  
- **Do not trust user-controlled headers** (`X-Forwarded-For`, `X-Real-IP`, etc.) for rate limiting or security.  
- Apply **rate limiting beyond IP-based checks**, such as:  
  - Progressive delays after repeated failures.  
  - Account-based lockouts.  
  - Device/browser fingerprinting.  
  - CAPTCHA after suspicious activity.  
- Enforce **Multi-Factor Authentication (MFA)** to reduce the impact of brute-force or credential stuffing.  
- Set up **monitoring and alerting** to detect unusual login patterns (e.g., rapid IP rotation, distributed failures).  
  

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
