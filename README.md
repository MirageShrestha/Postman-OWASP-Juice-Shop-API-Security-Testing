# 🔐 API Security Testing — OWASP Juice Shop

A hands-on security testing project using **Postman** and **Newman** to identify and document real-world API vulnerabilities on the OWASP Juice Shop application. Covers SQL Injection, Cross-Site Scripting (XSS), and Broken Access Control.

Newman Summary Report: https://api-security-testing-postman.vercel.app/

---

## 📌 Project Overview

| Field | Details |
|---|---|
| **Target App** | OWASP Juice Shop (intentionally vulnerable app) |
| **Testing Tool** | Postman + Newman CLI |
| **Report Format** | HTML (newman-reporter-htmlextra) |
| **Local URL** | `http://localhost:3000` |
| **Vulnerabilities Tested** | SQL Injection · XSS · Broken Access Control |
| **Total Test Cases** | 14+ requests · 25+ assertions |
| **Training Institute** | Broadway Infosys |

---

## 🛠️ Tech Stack

- **Postman** — API request building, scripting, and test assertions
- **Newman** — CLI runner for Postman collections
- **newman-reporter-htmlextra** — HTML report generation
- **Docker** — Running OWASP Juice Shop locally
- **OWASP Juice Shop** — Vulnerable target application

---

## ⚙️ Setup & Installation

### 1. Run Juice Shop via Docker

```bash
docker pull bkimminich/juice-shop
docker run -d -p 3000:3000 bkimminich/juice-shop
```

Open `http://localhost:3000` in your browser to confirm it's running.

### 2. Install Newman and HTML Reporter

```bash
npm install -g newman
npm install -g newman-reporter-htmlextra
```

### 3. Clone this Repository

```bash
git clone https://github.com/your-username/juiceshop-security-tests.git
cd juiceshop-security-tests
```

### 4. Run the Collection

```bash
newman run juiceshop-security.json \
  -e juiceshop-env.json \
  -r cli,htmlextra \
  --reporter-htmlextra-export reports/security-report.html \
  --reporter-htmlextra-title "Juice Shop Security Test Report"
```

---

## 📁 Project Structure

```
juiceshop-security-tests/
│
├── juiceshop-security.json      # Postman collection (exported)
├── juiceshop-env.json           # Postman environment variables
├── reports/
│   └── security-report.html    # Generated Newman HTML report
└── README.md
```

---

## 🧪 Test Cases

### 🔴 Module 1 — SQL Injection (SQLi)

**Endpoint:** `POST /rest/user/login` · `GET /rest/products/search`

| Test | Payload | Expected | Actual | Finding |
|---|---|---|---|---|
| Basic auth bypass | `' OR '1'='1'--` | 401 | 200 + token | 🔴 Vulnerable |
| Admin account bypass | `admin@juice-sh.op'--` | 401 | 200 + token | 🔴 Vulnerable |
| UNION data extraction | `')) UNION SELECT id,email,password,role,'5','6','7','8','9' FROM Users--` | 400 | 200 + 24 user records | 🔴 Critical |

**Impact of UNION injection:** Full Users table dumped — all emails, MD5 hashed passwords, and roles (admin/customer/deluxe) exposed in product search results.

**Assertions written:**
```js
pm.test("SQLi bypass should return 401", () => {
  pm.response.to.have.status(401);
});

pm.test("User emails must not appear in product search", () => {
  pm.expect(pm.response.text()).to.not.include("@juice-sh.op");
});

pm.test("Password hashes must not be exposed", () => {
  pm.expect(pm.response.text()).to.not.match(/[a-f0-9]{32}/);
});
```

---

### 🟠 Module 2 — Cross-Site Scripting (XSS)

**Endpoint:** `PUT /rest/products/1/reviews` · `GET /rest/products/1/reviews`

| Test | Payload | Expected | Actual | Finding |
|---|---|---|---|---|
| Store script tag | `<script>alert('XSS')</script>` | 400 rejected | 200 stored | 🔴 Stored XSS |
| Retrieve payload | GET reviews | Encoded output | Raw `<script>` tag | 🔴 No sanitization |
| img onerror vector | `<img src=x onerror=alert(1)>` | 400 rejected | 200 stored | 🔴 Filter bypass |

**Assertions written:**
```js
pm.test("Script tags must not appear raw in response", () => {
  pm.expect(pm.response.text()).to.not.include("");
});

pm.test("Event handler attributes must not be stored", () => {
  pm.expect(pm.response.text()).to.not.include("onerror=");
  pm.expect(pm.response.text()).to.not.include("onload=");
});
```

---

### 🟢 Module 3 — Broken Access Control (BAC)

**Endpoints:** `/rest/basket/:id` · `/api/Users/` · `/api/Users/:id`

| Test | Endpoint | Expected | Actual | Finding |
|---|---|---|---|---|
| IDOR — other user's basket | `GET /rest/basket/1` | 403 | 200 + basket data | 🔴 IDOR |
| No token on protected route | `GET /api/Users/` | 401 | Varies | 🟡 Check report |
| Normal user on admin route | `GET /api/Users/` with user token | 403 | 200 + all users | 🔴 Privilege escalation |
| Self-assign admin role | `PUT /api/Users/:id` with `{"role":"admin"}` | 403 | 200 role changed | 🔴 Mass assignment |

**Assertions written:**
```js
pm.test("IDOR — cannot access other user's basket", () => {
  pm.expect(pm.response.code).to.equal(403);
});

pm.test("Regular user cannot list all users", () => {
  pm.response.to.have.status(403);
});

pm.test("Cannot self-assign admin role", () => {
  pm.expect(pm.response.code).to.be.oneOf([400, 403]);
});
```

---

## 📊 Newman Report

After running Newman, open `reports/security-report.html` in any browser.

The report includes:
- Total requests run / passed / failed
- Each request with full URL, headers, body, and response
- Every `pm.test()` with ✓ pass or ✗ fail status
- Response time per request
- Environment variable state at end of run

> **Note:** Failing assertions on Juice Shop are **expected and intentional** — they confirm the vulnerability was found, not that the tests are wrong.

---

## 🔍 Key Findings Summary

| # | Vulnerability | Severity | Endpoint | Impact |
|---|---|---|---|---|
| 1 | UNION-based SQL Injection | 🔴 Critical | `GET /rest/products/search` | 24 user records + password hashes dumped |
| 2 | SQLi Auth Bypass | 🔴 High | `POST /rest/user/login` | Login without valid credentials |
| 3 | Stored XSS | 🔴 High | `PUT /rest/products/:id/reviews` | Script tag stored and returned unescaped |
| 4 | IDOR | 🔴 High | `GET /rest/basket/:id` | Access any user's private basket |
| 5 | Privilege Escalation | 🔴 High | `GET /api/Users/` | Regular user reads full user list |
| 6 | Mass Assignment | 🔴 High | `PUT /api/Users/:id` | User can self-promote to admin role |

---

## 🛡️ Recommended Fixes

| Vulnerability | Fix |
|---|---|
| SQL Injection | Use **parameterized queries** / prepared statements — never concatenate user input into SQL |
| XSS | **Sanitize and encode** all user input before storing; use libraries like DOMPurify on output |
| IDOR | Validate that the **authenticated user owns** the resource being requested |
| Privilege Escalation | Enforce **role-based access control (RBAC)** on every protected endpoint server-side |
| Mass Assignment | **Whitelist** only allowed fields in update operations — never accept `role` from user input |

---

## ⚠️ Disclaimer

This project was conducted entirely on **OWASP Juice Shop** — an intentionally vulnerable application built for security training. All testing was performed locally in a controlled environment.

**Never run these tests against real production systems without written authorization.**

---

## 👤 Author

**Mirage Shrestha**

---

## 📚 References

- [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Newman Docs](https://learning.postman.com/docs/collections/using-newman-cli/command-line-integration-with-newman/)
- [newman-reporter-htmlextra](https://github.com/DannyDainton/newman-reporter-htmlextra)
