# Lab 1: Username Enumeration via different response

## Vulnerability Type
Authentication Flaw

### OWASP Top 10 Mapping
A07: 2021 – Identification and Authentication Failures

---

## Description
This vulnerability occurs when the website doesn't avoid brut forcing against it and also revels weather a username 
exists through different responses such as error messages, response length, status code or response time. By 
analyzing these difference an attacker can enumurate valid username first and then try to crack the password.

---

## Tools Used
- **Burp Suite** (Proxy, Intruder, Grep Match/Extract)
- **Hydra** *(optional: requires explicit written permission)*

---

## Attack Path Diagram
```plaintext
[Attacker]
    |
    v
[Login Page]
    |
    |-- Invalid Username --> Response: "Invalid username or password"
    |
[Username Brute-Force]
    |
    |-- Valid Username + Wrong Password --> Response: "Invalid password" or longer response time
    |
    v
[Password Brute-Force]
    |
    v
[Successful Login]
```
## Step-by-Step Exploitation


1. **Step 1 - Reconnaissance / Intercept Request**
  - Configure your browser to use **Burp Proxy** (FoxyProxy or Burp’s built-in browser).
  - Submit an obviously wrong **username/password** on the login form.
  - In **Proxy → HTTP history**, note the baseline **status code**, **response body text**, **response length**, and **time**  


2. **Step 2 - Enumeration / Identification**
  - Send the request to **Burp Intruder**.  
  - Set payload positions at the vulnerable field (username).  
  - Add payload list (usernames, emails, tokens, etc.).  
  - Configure **Grep-Match / Grep-Extract** for the message recived from the server to highlight differences.  
  - Run the attack (Sniper attack) and identify valid input based on:
    - Response length  
    - Response time  
    - Response message / status code
    - Grep extract 
  - There will be an difference in respons time because after the usernam was correct then only the server try to check for
    password and is responsible for the respons time difference.


3. **Step 3 - Exploitation**  
  - Now we have got the username and do the same for the password field.
  - Set a payload position for it and and the payload items and start the attack.  
  - Analyze successful response status code.  


4. **Cluster Bombing**
  - We can also use cluster bombing method to solve this in which we can set multiple payload position and give multiple
    payloads and the attack can be started.
  - This will have a lot of possible combinations and we can get both the password and username at one scan.


5. **Step 5 - Tool Alternatives (Optional)**
  - If allowed, demonstrate with command-line tool like **Hydra**  
  - Needed information for Hydra (Authentication flaws) to start the attack:  
    - **Target server IP:** `<IP>`  
    - **Login URL:** `<login endpoint>`  
    - **Username/Password payloads:** `<user.txt> / <pass.txt>`  
    - **Failure string:** `<error message>`
  
    - Hydra Command template: 
        hydra -L <username_list.txt> -P <password_list.txt> <protocol>://<target_ip>[:port]/<path> \

    - Hydra command outputs valid credentials.

---

## Mitigation
- Ensure the application returns the same response for both valid and invalid usernames
- Implement measures to normalize response times so attackers cannot differentiate valid usernames based on time delays.
- The website should build logic in such a way that it should not allow brut forcing the password or usernames.  
- Block a user if multiple login failed  

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

