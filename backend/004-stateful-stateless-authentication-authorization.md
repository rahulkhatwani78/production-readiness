# Stateful vs. Stateless Authentication and Authorization

In modern web applications, securing routes and identifying users is critical. This guide explores the two primary approaches to authentication: **Stateful (Session-based)** and **Stateless (Token-based)**, along with authorization strategies. All examples use Node.js, Express, and TypeScript.

---

## 1. Authentication vs. Authorization

Before diving deep, it's important to distinguish between these two concepts:

*   **Authentication (AuthN):** Who are you? (e.g., Logging in with username and password).
*   **Authorization (AuthZ):** What are you allowed to do? (e.g., Only admins can delete a post).

---

## 2. Stateful Authentication (Session-Based)

In stateful authentication, the server keeps a record of the authenticated user's session.

### How it works:
1.  Client sends credentials (username/password).
2.  Server verifies credentials, creates a session in memory or a database (e.g., Redis), and generates a unique `Session ID`.
3.  Server sends the `Session ID` back to the client, usually in a `Set-Cookie` header.
4.  Subsequent requests from the client automatically include the cookie with the `Session ID`.
5.  Server looks up the `Session ID` in its store to identify the user.

### Pros:
*   **Easy to revoke:** You can instantly invalidate a session on the server.
*   **Secure against XSS:** If using `HttpOnly` cookies, JavaScript cannot access the session ID.
*   **Simple state management:** The server knows exactly how many active sessions exist.

### Cons:
*   **Scalability:** Requires centralized session storage (like Redis) if you have multiple server instances.
*   **Overhead:** Database/Memory lookup required on every authenticated request.
*   **CORS issues:** Managing cookies across different domains can be tricky.

### Example: Express + TypeScript + Redis

First, install dependencies:
```bash
npm install express express-session connect-redis redis
npm install -D @types/express @types/express-session
```

```typescript
// server.ts
import express, { Request, Response, NextFunction } from 'express';
import session from 'express-session';
import RedisStore from 'connect-redis';
import { createClient } from 'redis';

const app = express();
app.use(express.json());

// Initialize Redis Client
const redisClient = createClient({ url: 'redis://localhost:6379' });
redisClient.connect().catch(console.error);

// Configure Session Middleware
app.use(
  session({
    store: new RedisStore({ client: redisClient }),
    secret: process.env.SESSION_SECRET || 'super-secret-key',
    resave: false,
    saveUninitialized: false,
    cookie: {
      secure: process.env.NODE_ENV === 'production', // true if https
      httpOnly: true, // Prevents client-side JS from reading the cookie
      maxAge: 1000 * 60 * 60 * 24, // 1 day
      sameSite: 'lax', // CSRF protection
    },
  })
);

// Augment express-session types
declare module 'express-session' {
  interface SessionData {
    userId: string;
    role: string;
  }
}

// Authentication Route
app.post('/api/login', (req: Request, res: Response) => {
  const { username, password } = req.body;
  
  // In production, verify against DB and check password hash!
  if (username === 'admin' && password === 'password') {
    req.session.userId = 'user_123';
    req.session.role = 'admin';
    return res.status(200).json({ message: 'Logged in successfully' });
  }
  
  return res.status(401).json({ error: 'Invalid credentials' });
});

// Logout Route
app.post('/api/logout', (req: Request, res: Response) => {
  req.session.destroy((err) => {
    if (err) return res.status(500).json({ error: 'Could not log out' });
    res.clearCookie('connect.sid');
    return res.status(200).json({ message: 'Logged out' });
  });
});

// Protected Route Middleware
const requireAuth = (req: Request, res: Response, next: NextFunction) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
};

app.get('/api/profile', requireAuth, (req: Request, res: Response) => {
  res.json({ message: `Welcome user ${req.session.userId}` });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## 3. Stateless Authentication (Token-Based / JWT)

In stateless authentication, the server does not store any session state. Instead, it signs a token (usually a JSON Web Token - JWT) containing user information and sends it to the client.

### How it works:
1.  Client sends credentials.
2.  Server verifies credentials and creates a JWT signed with a secret key.
3.  Server sends the JWT to the client.
4.  Client stores the JWT (in memory, localStorage, or an HttpOnly cookie).
5.  Client includes the JWT in subsequent requests (e.g., in the `Authorization: Bearer <token>` header).
6.  Server verifies the token's signature mathematically without querying a database.

### Pros:
*   **Highly Scalable:** No central session store needed. Any server with the secret key can verify the token.
*   **Cross-Domain:** Easily works across different APIs and domains (if sent via headers).
*   **Decoupled:** The auth server can be completely separate from the API servers.

### Cons:
*   **Hard to Revoke:** You cannot invalidate a JWT before it expires without maintaining a "blacklist" (which defeats the purpose of being stateless).
*   **Token Size:** JWTs are generally larger than Session IDs, increasing network payload.
*   **Security Risks:** If stored in `localStorage`, tokens are vulnerable to XSS.

### Security Best Practices for JWTs:
1.  **Short Expiration:** Keep Access Tokens short-lived (e.g., 15 minutes).
2.  **Refresh Tokens:** Use an opaque, longer-lived Refresh Token (stored in an HttpOnly cookie) to get new Access Tokens.
3.  **Storage:** Prefer storing Access Tokens in memory (React state) and Refresh Tokens in `HttpOnly` cookies to mitigate XSS and CSRF.

### Example: Express + TypeScript + JWT

Install dependencies:
```bash
npm install jsonwebtoken cookie-parser
npm install -D @types/jsonwebtoken @types/cookie-parser
```

```typescript
// server.ts
import express, { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import cookieParser from 'cookie-parser';

const app = express();
app.use(express.json());
app.use(cookieParser());

const JWT_SECRET = process.env.JWT_SECRET || 'super-secret-jwt-key';

// Augment Express Request type
declare global {
  namespace Express {
    interface Request {
      user?: { id: string; role: string };
    }
  }
}

// Authentication Route
app.post('/api/login', (req: Request, res: Response) => {
  const { username, password } = req.body;

  // In production, verify against DB
  if (username === 'user' && password === 'password') {
    const payload = { id: 'user_456', role: 'user' };
    
    // Create Access Token
    const token = jwt.sign(payload, JWT_SECRET, { expiresIn: '15m' });

    // Approach A: Send in response body (Client must attach to headers later)
    // return res.json({ token });

    // Approach B (More secure for browsers): Store in HttpOnly cookie
    res.cookie('token', token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 15 * 60 * 1000, // 15 mins
    });

    return res.status(200).json({ message: 'Logged in' });
  }

  return res.status(401).json({ error: 'Invalid credentials' });
});

// Protected Route Middleware
const requireAuth = (req: Request, res: Response, next: NextFunction) => {
  // Check cookie OR Authorization header
  const token = req.cookies?.token || req.headers.authorization?.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'Unauthorized: No token provided' });
  }

  try {
    const decoded = jwt.verify(token, JWT_SECRET) as { id: string; role: string };
    req.user = decoded; // Attach user to request
    next();
  } catch (error) {
    return res.status(403).json({ error: 'Forbidden: Invalid or expired token' });
  }
};

app.get('/api/profile', requireAuth, (req: Request, res: Response) => {
  res.json({ message: `Welcome user ${req.user?.id}`, role: req.user?.role });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## 4. Authorization Patterns

Once a user is authenticated, you must determine what they can access.

### Role-Based Access Control (RBAC)
Users are assigned roles (e.g., `admin`, `editor`, `user`). Permissions are associated with these roles.

**Express/TypeScript Example for RBAC:**

```typescript
// RBAC Middleware
export const requireRole = (allowedRoles: string[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    // Ensure the user object exists (requires Auth Middleware to run first)
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Forbidden: Insufficient permissions' });
    }

    next();
  };
};

// Route only accessible by Admins
app.delete('/api/users/:id', requireAuth, requireRole(['admin']), (req: Request, res: Response) => {
  res.json({ message: `User ${req.params.id} deleted successfully.` });
});

// Route accessible by Admins and Editors
app.put('/api/posts/:id', requireAuth, requireRole(['admin', 'editor']), (req: Request, res: Response) => {
  res.json({ message: `Post updated.` });
});
```

### Attribute-Based Access Control (ABAC)
More granular. Policies use attributes of the user, resource, and environment (e.g., "User can edit a document ONLY IF they are the author AND the document is in draft state"). Usually implemented via policy engines (like CASL, OPA, or custom logic inside controllers).

---

## 5. Production Readiness Checklist for Auth

When deploying authentication to production, ensure the following:

1.  **Always use HTTPS:** Never send credentials, session IDs, or tokens over unencrypted HTTP.
2.  **Secure Password Hashing:** Never store plaintext passwords. Use robust algorithms like `bcrypt` or `argon2`.
    ```typescript
    import bcrypt from 'bcrypt';
    const hashedPassword = await bcrypt.hash('plain-password', 12);
    const isMatch = await bcrypt.compare('plain-password', hashedPassword);
    ```
3.  **Rate Limiting:** Protect your login, registration, and password reset endpoints against brute-force attacks.
4.  **Security Headers:** Use packages like `helmet` to set appropriate HTTP security headers.
5.  **Refresh Token Rotation:** If using JWTs, implement Refresh Token Rotation. When a refresh token is used, invalidate it and issue a new one. If an invalidated refresh token is used again, revoke all tokens for that user (indicates potential token theft).
6.  **Cookie Security:** Ensure all auth cookies have `HttpOnly`, `Secure: true`, and `SameSite: strict` (or `lax` if cross-site navigation is required).
7.  **Input Validation:** Use libraries like `zod` or `joi` to validate login payloads to prevent injection attacks.
