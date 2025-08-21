# Lab 6: Brute-forcing a stay-logged-in cookiee

## Vulnerability Type
- Predictable Cookie Value
- Insecure Password Hashing
- Authentication Bypass via Cookie Manipulation

### OWASP Top 10 Mapping
- A02: 2021 - Cryptographic Failures
- A07: 2021 – Identification and Authentication Failures

---
## Observation
- Normal login → Only `session` cookie is issued.
- Login with **"Stay logged in"** checked → Two cookies:
  - `session`
  - `stay-logged-in` (longer value, persistent)
- `stay-logged-in` cookie value is **the same every time** for the same user credentials.
- This indicates that the cookie is generated in a deterministic way from **username + password**.

---

## Cookie Structure Hypothesis
The format appears to be: base64(username:md5(password))

**Example:**
- Username: `wiener`
- Password: `peter`

1. `md5(peter)` → `21bd12dc183f740ee76f27b78eb39c8d`  
2. String: `wiener:21bd12dc183f740ee76f27b78eb39c8d`  
3. Base64 encode → `d2llbmVyOjIxYmQxMmRjMTgzZjc0MGVlNzZmMjdiNzhlYjM5Yzhk`

This matches the `stay-logged-in` cookie.

---

## Description
In this lab, the vulnerability lies in the insecure implementation of the "stay-logged-in" cookie. When the "Remember me" option is selected, the application issues an additional cookie whose value is derived from the username and password using a weak hashing scheme (`base64(username:md5(password))`). Since the cookie is predictable and not tied to secure server-side session management, an attacker can brute-force or craft valid cookies for other users. This allows bypassing the normal login process and directly impersonating victims without knowing their actual credentials.

---

## Tools Used
- **Burp Suite** (Proxy, Intruder, Grep Match/Extract)
- **Hashcat**
- **John the Ripper**
- **Hash-identifier**
- **Crackstation** *(optional: not preffered to use as we should not paste the tokens here. Insted use offline tools.)*

---

## Step-by-Step Exploitation

1. **Recon: observe cookies**
    - Log in normally → only `session` cookie is set.
    - Log in with **“Stay logged in”** checked → `session` **and** `stay-logged-in` are set.
    - Log out/in a few times with the same creds and note the **stay cookie is identical** each time → it’s deterministic.

2. **Derive the cookie scheme (using your own account)**
    - Copy your `stay-logged-in` value and **Base64-decode** it.
    - You should see something like: `wiener:21bd12dc183f740ee76f27b78eb39c8d`.
    - Compute `md5(your_password)` and confirm it matches the hash string.
    - Hypothesis confirmed:  
      **`stay-logged-in = base64(username:md5(password))`**

3. **Pick the target**
    - Target username: **carlos**.

4. **Choose the request to brute-force**
    - Use a request that **accepts cookie auth** without a form (e.g., `GET /my-account?id=wiener` then follow redirects).
    - **Remove the `session` cookie** entirely so only `stay-logged-in` is used.
    - Final Cookie header template (with a payload slot):  
    Cookie: stay-logged-in=§PAYLOAD§ session=


5. **Set up Burp Intruder**
    - Send the chosen request to **Intruder**.
    - Attack type: **Sniper** (single payload position on the stay cookie value).
    - Load a **password wordlist** as the payload set (for carlos).

6. **Payload Processing (build the cookie on-the-fly)**
    - In **Payload Processing**, add these rules **in order**:
        1. **Hash → MD5** (applies to each password candidate)
        2. **Add prefix**: `carlos:`
        3. **Encode → Base64**
        - (Optional) If the app is picky about cookie chars, add **URL-encode** after Base64.

7. **Prevent interference**
    - In Intruder **Options**:
    - Disable/ignore **Cookie jar updates** so Burp doesn’t re-add a `session` cookie.
    - Keep **Follow redirects = off** so you can see 302 vs 200 differences.

8. **Success markers**
    - Add **Grep – Match/Extract** for indicators of success, e.g.Update email:
    - Text like `Update email`, `carlos`, `My account`, or a unique element on the logged-in page.
    - **Status**/**Length** changes (success often returns 200 with larger length; invalid cookie usually 302 → /login).

9. **Run attack & analyze**
    - Start the attack and **sort by Status/Length** or check the **Grep Extract** column.
    - The payload that yields a **Update email** is the correct password for **carlos** (and thus the valid stay cookie).

10. **Verify in Repeater**
    - Take the winning payload value (already Base64 of `carlos:md5(pass)`).
    - In **Repeater**, send:

(ensure **no `session` cookie**)
- `GET /my-account` should return **carlos’s account page**.

### Notes
- If you prefer pre-building payloads, you can generate files of:
base64("carlos:" + md5(candidate_password))

---

## Mitigation

1. **Do not store credentials in cookies**  
   - Avoid embedding username/password (even in hashed or encoded form) inside cookies.  
   - Use random, securely generated session tokens instead.

2. **Use strong hashing algorithms**  
   - Replace weak/fast hashing like MD5 with adaptive algorithms such as bcrypt, scrypt, or Argon2.  
   - Apply salting to prevent precomputed attacks.

3. **Tie cookies to server-side sessions**  
   - Store only a session identifier in the cookie.  
   - Keep user authentication data on the server side.

4. **Implement cookie expiration and rotation**  
   - Set short lifetimes for “stay-logged-in” cookies.  
   - Rotate tokens periodically to reduce replay risk.

5. **Enable cookie security flags**  
   - Set cookies with `HttpOnly`, `Secure`, and `SameSite` attributes to protect against theft.

6. **Apply rate limiting**  
   - Limit the number of login and cookie validation attempts to prevent brute force.  

7. **Use Multi-Factor Authentication (MFA)**  
   - Add MFA to sensitive accounts, reducing the impact if a cookie is compromised.
  

---

## Real-World Relevance

In real-world applications, some websites implement "remember me" or "stay logged in" features by storing user credentials or weakly hashed values in cookies.  
If an attacker can predict or brute-force these cookies (e.g., `base64(username:md5(password))`), they can impersonate users without knowing the actual password.  

This vulnerability has been observed in poorly designed authentication systems of e-commerce sites, forums, and legacy applications, leading to account takeover and unauthorized access.

