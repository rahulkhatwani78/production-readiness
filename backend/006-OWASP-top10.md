# OWASP Top 10 Security Vulnerabilities (Node.js & Express Focus)

The **OWASP Top 10** is a standard awareness document for developers and web application security. It represents a broad consensus about the most critical security risks to web applications. Ensuring your Node.js and Express backend is protected against these vulnerabilities is a critical step in achieving production readiness.

This guide explores the OWASP Top 10 (2021 edition) and provides actionable mitigation strategies specifically tailored for Node.js and Express.

---

## A01:2021 - Broken Access Control

Access control enforces policies such that users cannot act outside of their intended permissions. Failures typically lead to unauthorized information disclosure, modification, or destruction of all data.

### Node.js Context:
*   **Insecure Direct Object References (IDOR):** A user modifying the URL parameter from `/api/users/123` to `/api/users/124` and successfully viewing another user's profile.
*   **Missing Role-Based Access Control (RBAC):** An ordinary user accessing admin endpoints (e.g., `POST /api/admin/delete-user`).

### Mitigation in Express:
1.  **Implement robust Authorization middleware:** As discussed in the Authorization guide, explicitly check user roles.
2.  **Verify Ownership:** Before modifying or returning a resource, verify that the currently authenticated user actually owns it or has permission to view it.
3.  **Deny by Default:** Ensure all API endpoints require authentication by default, except for specific public routes (like login/register).

---

## A02:2021 - Cryptographic Failures

This category focuses on failures related to cryptography, which often lead to sensitive data exposure (e.g., passwords, credit card numbers, health records).

### Node.js Context:
*   Storing passwords in plaintext or using weak hashing algorithms like MD5 or SHA1.
*   Transmitting sensitive data over unencrypted HTTP.
*   Using weak, hardcoded, or easily guessable secrets for JWTs or session cookies.

### Mitigation in Express:
1.  **Use `bcrypt` or `argon2`:** Always hash passwords with a strong, slow algorithm with a salt.
2.  **Enforce HTTPS:** Ensure your application is served over TLS/SSL. In production, configure your load balancer or reverse proxy (like Nginx) to redirect HTTP traffic to HTTPS.
3.  **Use Strong Environment Variables:** Never hardcode secrets. Ensure `JWT_SECRET` and `SESSION_SECRET` are long, random, cryptographically secure strings.
4.  **Secure Cookies:** Always set `Secure: true` and `HttpOnly: true` on authentication cookies.

---

## A03:2021 - Injection

Injection flaws occur when untrusted data is sent to an interpreter as part of a command or query. The attacker's hostile data can trick the interpreter into executing unintended commands or accessing data without proper authorization.

### Node.js Context:
*   **SQL/NoSQL Injection:** Concatenating user input directly into database queries.
*   **Command Injection:** Passing user input to `child_process.exec()`.
*   **Code Injection:** Passing user input to `eval()`, `setTimeout()`, or `setInterval()`.

### Mitigation in Express:
1.  **Use ORMs/ODMs or Parameterized Queries:** Always use parameterized queries (e.g., `pg` library with `$1`) or an ORM/Query Builder (like Prisma, TypeORM, or Knex). Never use string concatenation for queries.
2.  **Input Validation:** Strictly validate and sanitize all user input using libraries like `zod` or `joi`.
3.  **Avoid Execution Functions:** Never use `eval()`. Be extremely cautious with `exec()`—use `execFile()` or `spawn()` instead if you must run system commands.

---

## A04:2021 - Insecure Design

This category focuses on risks related to design flaws. It highlights the need for threat modeling, secure design patterns, and reference architectures. An insecure design cannot be fixed by a perfect implementation.

### Node.js Context:
*   Allowing unlimited password reset attempts without rate limiting.
*   Designing an API that returns excessive data, relying on the frontend to filter it out (Mass Assignment/Data Exposure).

### Mitigation in Express:
1.  **Rate Limiting & Abuse Prevention:** Protect sensitive flows (login, OTP generation, password reset) using tools like `express-rate-limit`.
2.  **Avoid Mass Assignment:** Never blindly pass `req.body` to your database update functions. Explicitly pick the fields you want to update.
3.  **Data Minimization:** Only return the data the client absolutely needs. Do not send entire user objects (including hashed passwords or internal IDs) to the client.

---

## A05:2021 - Security Misconfiguration

This is highly common and results from insecure default settings, incomplete or ad hoc configurations, open cloud storage, misconfigured HTTP headers, and verbose error messages containing sensitive information.

### Node.js Context:
*   Running Express in "development" mode in production.
*   Leaking stack traces to users on errors.
*   Missing security headers.
*   Exposing the `X-Powered-By: Express` header, which informs attackers about your stack.

### Mitigation in Express:
1.  **Set `NODE_ENV=production`:** Express behaves differently in production (caches templates, hides detailed errors).
2.  **Use `helmet`:** Add the `helmet` middleware to automatically set secure HTTP headers (e.g., Content-Security-Policy, X-Frame-Options, HSTS).
3.  **Disable `X-Powered-By`:** `app.disable('x-powered-by');` (Helmet does this automatically).
4.  **Generic Error Messages:** Ensure your global error handler never leaks stack traces or internal DB errors to the client in production.

---

## A06:2021 - Vulnerable and Outdated Components

Modern Node.js applications rely heavily on third-party `npm` packages. If an attacker exploits a vulnerability in a dependency, your application is compromised.

### Node.js Context:
*   Using an old version of a library with known Remote Code Execution (RCE) or Prototype Pollution vulnerabilities.

### Mitigation in Express:
1.  **Audit Regularly:** Run `npm audit` frequently to identify known vulnerabilities in your dependency tree.
2.  **Automated Updates:** Use tools like GitHub Dependabot, Snyk, or Renovate to automatically create PRs for dependency updates.
3.  **Minimize Dependencies:** Don't install a massive library for a single utility function. The fewer dependencies, the smaller the attack surface.

---

## A07:2021 - Identification and Authentication Failures

When authentication and session management are implemented incorrectly, attackers can compromise passwords, keys, or session tokens to assume other users' identities.

### Node.js Context:
*   Permitting weak passwords.
*   Session IDs or JWTs that don't expire or are vulnerable to session fixation.
*   Vulnerability to credential stuffing attacks.

### Mitigation in Express:
1.  **Implement Multi-Factor Authentication (MFA):** Where possible, add MFA to critical accounts.
2.  **Strong Password Policies:** Enforce length and complexity requirements.
3.  **Session Security:** Implement Refresh Token Rotation, short-lived Access Tokens, and secure cookie attributes.
4.  **Rate Limit Login:** Aggressively rate-limit the `/api/login` route to prevent brute-force attacks.

---

## A08:2021 - Software and Data Integrity Failures

This relates to code and infrastructure that does not protect against integrity violations. This includes insecure CI/CD pipelines or deserialization of untrusted data.

### Node.js Context:
*   Accepting unsigned or weakly signed JWTs (e.g., the `none` algorithm vulnerability).
*   Deserializing untrusted data (e.g., using `node-serialize` unsafely).

### Mitigation in Express:
1.  **Strict JWT Verification:** Ensure your JWT library strictly verifies the signature and the algorithm (e.g., enforcing `RS256` or `HS256` and rejecting `none`).
2.  **Avoid Unsafe Deserialization:** Prefer standard JSON parsing (`JSON.parse()`) over custom or complex serialization formats that might execute code upon deserialization.
3.  **Verify Package Integrity:** Use `package-lock.json` or `npm-shrinkwrap.json` to ensure the exact same dependencies are installed in production as in development.

---

## A09:2021 - Security Logging and Monitoring Failures

Without logging and monitoring, breaches cannot be detected. Attackers rely on a lack of monitoring to achieve their goals without being noticed.

### Node.js Context:
*   Not logging failed login attempts.
*   Logs stored locally on the server without being shipped to a centralized monitoring system.
*   Logging sensitive data (like passwords or full credit card numbers) in plain text.

### Mitigation in Express:
1.  **Structured Logging:** Use a robust logger like `Winston` or `Pino` and log in JSON format.
2.  **Audit Trails:** Log critical security events: successful/failed logins, password changes, authorization failures (403s), and critical data modifications.
3.  **Redact Sensitive Data:** Ensure your logging middleware strips out passwords, tokens, and PII from request bodies before writing to logs.
4.  **Centralize Logs:** Stream logs to a centralized service (e.g., ELK stack, Datadog, AWS CloudWatch) for alerting and analysis.

---

## A10:2021 - Server-Side Request Forgery (SSRF)

SSRF flaws occur whenever a web application is fetching a remote resource without validating the user-supplied URL. It allows an attacker to coerce the application to send a crafted request to an unexpected destination, even when protected by a firewall, VPN, or another type of network ACL.

### Node.js Context:
*   An endpoint like `/api/fetch-image?url=<user_provided_url>` where the server makes an HTTP request to the provided URL. An attacker could provide `http://localhost:27017` or an internal AWS metadata IP (`http://169.254.169.254`) to port-scan or steal cloud credentials.

### Mitigation in Express:
1.  **Validate and Allowlist:** If your server must fetch remote resources based on user input, strictly validate the URL. Use an allowlist of permitted domains if possible.
2.  **Disable Internal Routing:** Ensure the HTTP client (like `axios` or `node-fetch`) cannot route to internal IP addresses (e.g., `127.0.0.1`, `10.0.0.0/8`, `169.254.169.254`).
3.  **Use Dedicated Services:** If fetching untrusted content, do it in an isolated container or serverless function with no access to your internal network.

---

## Summary Checklist for Express

- [ ] Use `helmet` for secure HTTP headers.
- [ ] Enforce `NODE_ENV=production`.
- [ ] Implement rate limiting (`express-rate-limit`).
- [ ] Validate all input strictly (`zod`, `joi`).
- [ ] Use parameterized queries or ORMs.
- [ ] Store secrets in environment variables.
- [ ] Hash passwords with `bcrypt` or `argon2`.
- [ ] Use `HttpOnly` and `Secure` flags for auth cookies.
- [ ] Implement a global error handler that hides stack traces.
- [ ] Log securely using `winston` or `pino` (redact sensitive info).
- [ ] Run `npm audit` and update dependencies regularly.
