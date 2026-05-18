# Securing Express with Helmet and CORS

When deploying a Node.js and Express application to production, securing the HTTP headers and managing cross-origin requests are fundamental steps. Two of the most critical middlewares for this purpose are **Helmet** and **CORS**.

This guide covers what they are, why they are necessary, and how to configure them securely for a production environment.

---

## Helmet: Securing HTTP Headers

Helmet is a collection of middleware functions that set security-related HTTP response headers. By default, Express does not set many secure headers, leaving your application vulnerable to well-known web exploits like Cross-Site Scripting (XSS), Clickjacking, and MIME sniffing.

### Why is Helmet Important?
*   **Information Leakage:** Express sends an `X-Powered-By: Express` header by default, which tells attackers exactly what technology stack you are using.
*   **Clickjacking:** If your app can be embedded in an `<iframe>` on an attacker's site, users might be tricked into clicking things they didn't intend to.
*   **MIME Sniffing:** Browsers might try to guess the content type of a file, which can lead to executing malicious scripts disguised as other file types.

### Key Helmet Protections
Helmet acts as a wrapper around 15 smaller middlewares. Some of the most important include:
*   `helmet.hidePoweredBy()`: Removes the `X-Powered-By` header.
*   `helmet.frameguard()`: Sets the `X-Frame-Options` header to prevent Clickjacking.
*   `helmet.xssFilter()`: Adds some small XSS protections (setting `X-XSS-Protection`).
*   `helmet.noSniff()`: Sets `X-Content-Type-Options` to `nosniff`, preventing browsers from MIME-sniffing a response away from the declared content-type.
*   `helmet.hsts()`: Sets the `Strict-Transport-Security` header to enforce secure (HTTP over SSL/TLS) connections to the server.
*   `helmet.contentSecurityPolicy()`: Sets the `Content-Security-Policy` header to prevent XSS and other cross-site injections.

### Mitigation and Implementation in Express
Implementing Helmet is straightforward but configuration is key for production.

```typescript
import express from 'express';
import helmet from 'helmet';

const app = express();

// 1. Basic Usage (Recommended for most apps)
// This enables a sensible set of default headers.
app.use(helmet());

// 2. Custom Configuration (Strict Production)
app.use(helmet({
  // Content Security Policy (CSP) restricts where resources can be loaded from
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'trusted-cdn.com'"],
      objectSrc: ["'none'"],
      upgradeInsecureRequests: [],
    },
  },
  // Ensure HSTS is enabled to force HTTPS
  hsts: {
    maxAge: 31536000, // 1 year in seconds
    includeSubDomains: true,
    preload: true
  },
  // Customize Frameguard if you need to allow framing from specific origins
  // frameguard: { action: 'sameorigin' } // (default)
}));
```

---

## CORS: Cross-Origin Resource Sharing

CORS is a browser security feature that restricts cross-origin HTTP requests. An origin consists of a protocol, domain, and port (e.g., `https://example.com:443`). If your frontend is hosted at `https://my-app.com` and your backend API is at `https://api.my-app.com`, the browser will block frontend requests to the API unless the backend explicitly allows it via CORS headers.

### Why is CORS Important?
CORS protects users from malicious websites reading data from your API. Without CORS, a user logged into their banking app could visit an attacker's site, and the attacker's script could make a request to the banking API on behalf of the user, stealing data.

### Node.js Context & Risks
*   **Overly Permissive Configuration:** Setting `Access-Control-Allow-Origin: *` allows *any* website to access your API. This is extremely dangerous for APIs that use cookie-based authentication.
*   **Reflected Origins:** Dynamically taking the `Origin` header from the request and echoing it back in the `Access-Control-Allow-Origin` response header. This effectively bypasses CORS.

### Mitigation and Implementation in Express
Using the `cors` package in Express allows you to configure these rules securely.

```typescript
import express from 'express';
import cors from 'cors';

const app = express();

// 1. Development (Permissive - DO NOT USE IN PROD)
// app.use(cors()); // Allows all origins

// 2. Production (Strict Origin Allowlist)
const allowedOrigins = [
  'https://www.my-production-app.com',
  'https://my-production-app.com',
  'http://localhost:3000' // Allow local dev
];

const corsOptions: cors.CorsOptions = {
  origin: (origin, callback) => {
    // Allow requests with no origin (like mobile apps or curl requests)
    if (!origin) return callback(null, true);
    
    if (allowedOrigins.indexOf(origin) !== -1) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true, // IMPORTANT: Required if you are sending cookies (e.g., HttpOnly session/refresh tokens)
  optionsSuccessStatus: 200 // some legacy browsers (IE11, various SmartTVs) choke on 204
};

app.use(cors(corsOptions));
```

### Key CORS Configuration Options:
*   `origin`: The most important setting. Defines which domains are allowed. Always use an allowlist in production.
*   `credentials`: Set to `true` if your frontend needs to send cookies (like an HttpOnly refresh token) or authorization headers to the backend. If `credentials: true`, the `origin` cannot be `*`.
*   `methods`: Restrict the allowed HTTP methods (e.g., block `TRACE` or `OPTIONS` if not needed).
*   `allowedHeaders`: Specify which headers the client is allowed to send.

---

## Summary Checklist for Production

- [ ] Install and enable `helmet`.
- [ ] Configure `Content-Security-Policy` (CSP) via Helmet, restricting script and object sources.
- [ ] Verify `X-Powered-By` header is completely removed.
- [ ] Install and configure `cors`.
- [ ] Define a strict allowlist for the `origin` in CORS configuration. Never use `*` in production for sensitive APIs.
- [ ] Set `credentials: true` in CORS only if handling cookies or authorization headers, and ensure `origin` is explicit.
- [ ] Restrict CORS `methods` and `allowedHeaders` to only what your application requires.
