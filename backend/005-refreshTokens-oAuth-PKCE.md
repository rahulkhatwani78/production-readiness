# Refresh Tokens & OAuth 2.0 Integration with PKCE

Building upon the concepts of stateless authentication (JWT), this guide covers advanced authentication patterns: implementing robust **Refresh Token** strategies and delegating authentication using **OAuth 2.0** and OpenID Connect (OIDC). All examples use Node.js, Express, and TypeScript.

---

## 1. Refresh Tokens

When using stateless JWTs for authentication, security best practices dictate that **Access Tokens must be short-lived** (e.g., 10-15 minutes). This limits the window of opportunity for an attacker if a token is stolen. However, making the user log in every 15 minutes provides a terrible user experience.

Enter the **Refresh Token**.

### How it works:

1.  User logs in with credentials.
2.  Server returns two tokens:
    - **Access Token:** Short-lived JWT (e.g., 15 minutes) used to access protected routes. Usually stored in memory on the client side.
    - **Refresh Token:** Long-lived opaque string or JWT (e.g., 7 days) used _only_ to obtain new Access Tokens. Usually stored securely in an `HttpOnly`, `Secure` cookie.
3.  Client uses the Access Token for API requests.
4.  When the Access Token expires, the API returns a `401 Unauthorized`.
5.  Client catches the `401`, silently sends the Refresh Token to a `/api/refresh` endpoint.
6.  Server validates the Refresh Token (checking against a database or Redis to ensure it hasn't been revoked), generates a new Access Token, and returns it.
7.  Client retries the original failed request with the new Access Token.

### Security Enhancements: Refresh Token Rotation

To mitigate the risk of a stolen Refresh Token, implement **Refresh Token Rotation**:

- Every time a Refresh Token is used to get a new Access Token, the server issues a _new_ Refresh Token and invalidates the old one.
- **Reuse Detection:** If an invalidated (old) Refresh Token is presented again, it strongly implies a token has been compromised. The server should immediately invalidate _all_ refresh tokens associated with that user family, forcing them to re-authenticate completely.

### Example: Implementing Refresh Tokens (Express + TypeScript)

```typescript
// auth.controller.ts
import express, { Request, Response } from "express";
import jwt from "jsonwebtoken";
import { v4 as uuidv4 } from "uuid";
import cookieParser from "cookie-parser";

const app = express();
app.use(express.json());
app.use(cookieParser());

// Mock Database for Refresh Tokens
const refreshTokensDB: Record<string, string> = {}; // { refreshToken: userId }

const ACCESS_TOKEN_SECRET =
  process.env.ACCESS_TOKEN_SECRET || "your-access-secret";

// 1. Login Endpoint
app.post("/api/login", (req: Request, res: Response) => {
  const { username, password } = req.body;

  // Validate user (mocked)
  if (username === "test" && password === "password") {
    const userId = "user_123";

    // Generate Tokens
    const accessToken = jwt.sign({ id: userId }, ACCESS_TOKEN_SECRET, {
      expiresIn: "15m",
    });
    const refreshToken = uuidv4(); // Using opaque token instead of JWT for easy revocation

    // Store Refresh Token in DB (In production, use Redis or Postgres with expiry)
    refreshTokensDB[refreshToken] = userId;

    // Send Refresh Token as HttpOnly Cookie
    res.cookie("jwt", refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === "production",
      sameSite: "strict",
      maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
    });

    // Send Access Token in response body
    return res.json({ accessToken });
  }

  return res.status(401).json({ error: "Invalid credentials" });
});

// 2. Refresh Endpoint
app.post("/api/refresh", (req: Request, res: Response) => {
  const cookies = req.cookies;

  if (!cookies?.jwt) {
    return res.status(401).json({ error: "Unauthorized" });
  }

  const refreshToken = cookies.jwt;

  // Verify Refresh Token exists in DB (not revoked)
  const userId = refreshTokensDB[refreshToken];
  if (!userId) {
    return res.status(403).json({ error: "Forbidden: Invalid refresh token" });
  }

  // REFRESH TOKEN ROTATION: Invalidate the old one, create a new one
  delete refreshTokensDB[refreshToken];

  const newRefreshToken = uuidv4();
  refreshTokensDB[newRefreshToken] = userId;

  // Issue new Access Token
  const newAccessToken = jwt.sign({ id: userId }, ACCESS_TOKEN_SECRET, {
    expiresIn: "15m",
  });

  // Set new Refresh Token Cookie
  res.cookie("jwt", newRefreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "strict",
    maxAge: 7 * 24 * 60 * 60 * 1000,
  });

  return res.json({ accessToken: newAccessToken });
});

// 3. Logout Endpoint
app.post("/api/logout", (req: Request, res: Response) => {
  const cookies = req.cookies;
  if (!cookies?.jwt) return res.sendStatus(204); // No content

  const refreshToken = cookies.jwt;

  // Revoke token in DB
  delete refreshTokensDB[refreshToken];

  // Clear cookie
  res.clearCookie("jwt", { httpOnly: true, secure: true, sameSite: "strict" });
  return res.status(200).json({ message: "Logged out" });
});
```

---

## 2. OAuth 2.0 & OpenID Connect (OIDC)

### What is OAuth 2.0?

OAuth 2.0 is an **Authorization** protocol, not an Authentication protocol. It allows a user (Resource Owner) to grant a third-party application (Client) limited access to their protected resources on another server (Resource Server), without exposing their credentials (password).

Example: "Allow Application X to read your Google Contacts."

### What is OpenID Connect (OIDC)?

Since OAuth 2.0 is for authorization, **OpenID Connect (OIDC)** was built _on top_ of OAuth 2.0 to provide **Authentication**. OIDC introduces the **ID Token** (a JWT), which contains information about the authenticated user (e.g., name, email), standardizing how apps log users in via third parties (e.g., "Sign in with Google").

### Common OAuth 2.0 Flows (Grant Types)

1.  **Authorization Code Flow (with PKCE):** The most secure and common flow for server-side web apps, mobile apps, and SPAs. It involves an exchange of an authorization code for tokens.
2.  **Client Credentials Flow:** Used for machine-to-machine (M2M) communication where no user is involved (e.g., a backend cron job accessing an API).

### The Authorization Code Flow (High Level)

1.  **User Initiation:** User clicks "Sign in with Google".
2.  **Redirect:** Your server redirects the user's browser to Google's Authorization Server with your `client_id`, `redirect_uri`, and requested `scopes` (e.g., `openid profile email`).
3.  **Consent:** User logs into Google and grants permission to your app.
4.  **Callback:** Google redirects the user back to your `redirect_uri` with a temporary `Authorization Code`.
5.  **Token Exchange:** Your backend server makes a secure server-to-server request to Google, exchanging the `Authorization Code` and your `client_secret` for an `Access Token`, an `ID Token`, and possibly a `Refresh Token`.
6.  **Session Creation:** Your server validates the `ID Token`, extracts user info, finds or creates the user in your database, and establishes a session (or issues your own JWTs to the client).

### Example: Implementing "Sign in with Google" (Using raw HTTP)

Instead of using heavy libraries like Passport.js, understanding the raw flow via HTTP requests clarifies the protocol.

```bash
npm install axios
npm install -D @types/axios
```

```typescript
// oauth.controller.ts
import express, { Request, Response } from "express";
import axios from "axios";
import jwt from "jsonwebtoken";

const app = express();

const GOOGLE_CLIENT_ID = process.env.GOOGLE_CLIENT_ID || "your-client-id";
const GOOGLE_CLIENT_SECRET =
  process.env.GOOGLE_CLIENT_SECRET || "your-client-secret";
const REDIRECT_URI = "http://localhost:3000/api/auth/google/callback";
const JWT_SECRET = process.env.JWT_SECRET || "your-app-jwt-secret";

// 1. Initiate Flow: Redirect to Google
app.get("/api/auth/google", (req: Request, res: Response) => {
  const scopes = [
    "https://www.googleapis.com/auth/userinfo.profile",
    "https://www.googleapis.com/auth/userinfo.email",
  ];
  const authUrl =
    `https://accounts.google.com/o/oauth2/v2/auth?` +
    `client_id=${GOOGLE_CLIENT_ID}&` +
    `redirect_uri=${REDIRECT_URI}&` +
    `response_type=code&` +
    `scope=${scopes.join(" ")}&` +
    `access_type=offline`; // offline gets a refresh token

  res.redirect(authUrl);
});

// 2. Callback Endpoint
app.get("/api/auth/google/callback", async (req: Request, res: Response) => {
  const code = req.query.code as string;

  if (!code) return res.status(400).send("No code provided");

  try {
    // 3. Exchange code for tokens
    const tokenResponse = await axios.post(
      "https://oauth2.googleapis.com/token",
      {
        client_id: GOOGLE_CLIENT_ID,
        client_secret: GOOGLE_CLIENT_SECRET,
        code,
        redirect_uri: REDIRECT_URI,
        grant_type: "authorization_code",
      },
    );

    const { access_token, id_token } = tokenResponse.data;

    // 4. Decode ID token to get user info
    // Note: In production, verify the ID token signature using Google's public keys!
    const decodedIdToken = jwt.decode(id_token) as any;
    const { email, name, sub: googleId } = decodedIdToken;

    // 5. App Logic: Find or create user in YOUR database
    // let user = await db.users.findByGoogleId(googleId);
    // if (!user) user = await db.users.create({ email, name, googleId });
    const userId = "user_999"; // Mocked DB lookup

    // 6. Issue YOUR app's tokens (e.g., Access + Refresh token as discussed in Section 1)
    const appAccessToken = jwt.sign({ id: userId }, JWT_SECRET, {
      expiresIn: "15m",
    });

    // Redirect user to frontend with token, or set HttpOnly cookie
    res.cookie("app_auth", appAccessToken, { httpOnly: true });
    res.redirect("http://localhost:5173/dashboard"); // Redirect to frontend app
  } catch (error) {
    console.error("OAuth Error:", error);
    res.status(500).send("Authentication failed");
  }
});
```

---

## 3. PKCE (Proof Key for Code Exchange)

### What is PKCE and why is it needed?

In the standard Authorization Code Flow, the `client_secret` is used to prove the identity of the application requesting the tokens. However, in **Single Page Applications (SPAs)** or **Mobile Apps**, the source code is public or easily decompiled. You cannot securely store a `client_secret` on the client side.

If an attacker intercepts the `Authorization Code` (e.g., via malicious browser extensions or app link hijacking), they could exchange it for tokens if there's no `client_secret` protecting the exchange endpoint.

**PKCE (Proof Key for Code Exchange)** solves this by dynamically creating a unique "secret" for _every single authorization request_.

### How PKCE Works

1.  **Code Verifier:** The client generates a secure random string (the `code_verifier`).
2.  **Code Challenge:** The client hashes the `code_verifier` (usually with SHA-256) and base64-url encodes it to create a `code_challenge`.
3.  **Authorization Request:** The client redirects the user to the Authorization Server, passing the `code_challenge` and the hashing method (`code_challenge_method=S256`).
4.  **Authorization Server stores Challenge:** The Authorization Server stores the `code_challenge` temporarily and returns the `Authorization Code` after the user grants consent.
5.  **Token Exchange:** The client sends the `Authorization Code` to the token endpoint, along with the _original, unhashed_ `code_verifier` (instead of a `client_secret`).
6.  **Verification:** The Authorization Server hashes the provided `code_verifier` and compares it to the previously stored `code_challenge`. If they match, it proves the client exchanging the code is the exact same client that requested it. Tokens are issued.

### Example: Generating PKCE variables (Node.js/TypeScript)

```typescript
import crypto from "crypto";

// 1. Generate Code Verifier (random string, 43-128 chars)
function generateCodeVerifier(): string {
  return crypto.randomBytes(32).toString("base64url");
}

// 2. Generate Code Challenge from Verifier (SHA-256 + Base64URL)
function generateCodeChallenge(verifier: string): string {
  return crypto.createHash("sha256").update(verifier).digest("base64url");
}

const codeVerifier = generateCodeVerifier();
const codeChallenge = generateCodeChallenge(codeVerifier);

console.log("Verifier:", codeVerifier);
// Client must store the verifier temporarily (e.g., in sessionStorage or an HttpOnly cookie)

console.log("Challenge:", codeChallenge);
// Send the challenge in the initial authorization URL:
// ?client_id=...&response_type=code&code_challenge=${codeChallenge}&code_challenge_method=S256
```

---

## 4. Production Readiness Checklist for Refresh Tokens & OAuth

1.  **State Parameter (OAuth):** Always pass a random `state` string in the initial OAuth request and verify it in the callback to prevent CSRF attacks.
2.  **PKCE (Proof Key for Code Exchange):** If implementing OAuth in a mobile app or SPA where client secrets cannot be stored securely, you _must_ use Authorization Code Flow with PKCE.
3.  **Token Storage:** Never store refresh tokens in `localStorage`. Always use `HttpOnly` cookies to protect against XSS.
4.  **Database Backed Refresh Tokens:** Store refresh token hashes or identifiers in your database so you can proactively revoke them if a user's account is compromised or they log out from all devices.
5.  **Refresh Token Rotation:** Crucial for detecting token theft.
6.  **Scope Minimization:** Only request the OAuth scopes you absolutely need. Users are more likely to abandon sign-in if an app requests excessive permissions.
