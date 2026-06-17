# Rate Limiting Strategies and Validations

In production-ready applications, protecting your APIs from abuse, managing traffic, and ensuring data integrity are critical. This document covers common rate-limiting strategies and robust validation techniques.

---

## Part 1: Rate Limiting Strategies

Rate limiting controls the rate of traffic sent or received by a network interface controller. It is used to prevent DoS attacks, limit web scraping, and enforce API usage quotas.

### 1. Token Bucket Algorithm

**Analogy:** Imagine a bucket of a fixed capacity filled with tokens at a constant rate. Every time a request arrives, it must take a token from the bucket to be processed. If the bucket is empty, the request is dropped (or queued).

**How it works:**
- A bucket has a maximum capacity of `N` tokens.
- Tokens are added to the bucket at a fixed rate `R` (e.g., 10 tokens per second).
- When a request arrives, it consumes 1 token.
- If there are enough tokens, the request goes through. If not, the request is rejected with a `429 Too Many Requests` status.

**Pros:**
- Memory efficient.
- Allows bursts of traffic (up to the bucket capacity).

**Cons:**
- Bursts can still overwhelm backend services if the bucket size is large.

**Example (Redis + Node.js concept):**
```typescript
// Conceptual Token Bucket
class TokenBucket {
  capacity: number;
  tokens: number;
  refillRate: number; // tokens per second
  lastRefillTime: number;

  constructor(capacity: number, refillRate: number) {
    this.capacity = capacity;
    this.tokens = capacity;
    this.refillRate = refillRate;
    this.lastRefillTime = Date.now();
  }

  allowRequest(): boolean {
    this.refill();
    if (this.tokens >= 1) {
      this.tokens -= 1;
      return true;
    }
    return false;
  }

  refill() {
    const now = Date.now();
    const elapsedTime = (now - this.lastRefillTime) / 1000;
    const tokensToAdd = elapsedTime * this.refillRate;
    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefillTime = now;
  }
}
```

### 2. Leaky Bucket Algorithm

**Analogy:** Imagine a bucket with a small hole at the bottom. Water (requests) is poured into the bucket. It leaks out at a constant rate. If water is poured in faster than it leaks out, the bucket overflows, and the extra water is spilled (requests dropped).

**How it works:**
- Requests are added to a queue (the bucket) of a fixed size.
- A background worker processes requests from the queue at a constant rate.
- If the queue is full, new incoming requests are dropped.

**Pros:**
- Smooths out bursts of traffic, resulting in a stable outflow rate.
- Good for use cases where a constant processing rate is required.

**Cons:**
- A burst of traffic can fill up the queue with old requests, starving newer requests if not handled properly.

### 3. Fixed Window Counter

**How it works:**
- Time is divided into fixed windows (e.g., 12:00:00 to 12:01:00).
- A counter is maintained for each window.
- Each request increments the counter.
- If the counter exceeds a threshold within the window, subsequent requests are dropped.

**Pros:**
- Easy to understand and implement.
- Memory efficient.

**Cons:**
- **Spike at the edges:** Traffic bursts at the edges of the time windows can let twice the allowed limit through. For example, if the limit is 100 requests/minute, a user could send 100 requests at 12:00:59 and another 100 at 12:01:01, totaling 200 requests in 2 seconds.

### 4. Sliding Window Log

**How it works:**
- Keeps a log of timestamps for every request a user makes.
- When a new request comes in, remove all timestamps older than the time window (e.g., older than 1 minute).
- If the number of remaining timestamps is less than the limit, allow the request and log the new timestamp. Otherwise, drop it.

**Pros:**
- Very accurate. Eliminates the edge spikes of the Fixed Window.

**Cons:**
- Memory intensive because it stores a timestamp for *every* request.

### 5. Sliding Window Counter

**How it works:**
- A hybrid approach combining Fixed Window Counter and Sliding Window Log.
- Instead of keeping individual timestamps, it keeps counters for smaller sub-windows (e.g., 1-second windows within a 1-minute limit) and calculates a weighted sum based on the current time.

**Pros:**
- Smooths out traffic spikes.
- More memory efficient than the Sliding Window Log.

---

## Part 2: Validations

Validating incoming data is the first line of defense against malicious input and guarantees that your application processes only expected data formats.

### 1. Types of Validation

1.  **Sanitization:** Cleaning the input (e.g., trimming whitespace, escaping HTML to prevent XSS).
2.  **Schema/Format Validation:** Ensuring the data structure, types, and required fields match expectations (e.g., `email` is a valid email string, `age` is an integer).
3.  **Business Logic Validation:** Checking if the data is valid in the context of your application state (e.g., checking if a username is already taken, or if a user has sufficient balance).

### 2. Schema Validation using Zod

[Zod](https://zod.dev/) is a TypeScript-first schema declaration and validation library. It is highly recommended for modern Node.js applications.

**Why Zod?**
- Eliminates duplicated type declarations (infers TypeScript types from the schema).
- Rich ecosystem and chaining methods.
- Excellent error reporting.

**Example Implementation:**

```typescript
import { z } from 'zod';
import express, { Request, Response, NextFunction } from 'express';

// 1. Define the Schema
const createUserSchema = z.object({
  body: z.object({
    username: z.string().min(3).max(20),
    email: z.string().email(),
    password: z.string().min(8).regex(/[A-Z]/, "Password must contain an uppercase letter"),
    age: z.number().int().positive().optional(),
  })
});

// Infer the TypeScript type
type CreateUserRequest = z.infer<typeof createUserSchema>;

// 2. Create a generic validation middleware
const validate = (schema: z.AnyZodObject) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      // Parse the request against the schema
      await schema.parseAsync({
        body: req.body,
        query: req.query,
        params: req.params,
      });
      next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          message: "Validation Failed",
          errors: error.errors.map(e => ({
            field: e.path.join('.'),
            message: e.message
          }))
        });
      }
      next(error);
    }
  };
};

const app = express();
app.use(express.json());

// 3. Apply the middleware to the route
app.post('/users', validate(createUserSchema), (req: Request, res: Response) => {
  // At this point, req.body is fully validated and typed!
  const { username, email } = req.body;
  
  // Business logic here...
  res.status(201).json({ message: "User created", user: { username, email } });
});
```

### 3. Combining Rate Limiting and Validation in Express

A production route should typically flow like this:

1.  **Rate Limiter Middleware:** Reject fast if the user is over quota.
2.  **Authentication Middleware:** Ensure the user is who they say they are.
3.  **Validation Middleware:** Ensure the payload is correct.
4.  **Controller (Business Logic):** Execute the actual logic.

```typescript
// Example Route Setup
router.post(
  '/api/v1/resource',
  apiLimiter,            // 1. Rate Limiting (e.g., express-rate-limit)
  authenticateUser,      // 2. Authentication
  validate(schema),      // 3. Validation (Zod/Joi)
  resourceController     // 4. Controller
);
```

### Key Takeaways
- **Rate limiting** is about *when* and *how often* a client can call an API. Choose the algorithm that fits your resource constraints and burst requirements.
- **Validation** is about *what* a client can send to an API. Always validate at the boundary of your application using a robust schema library like Zod or Joi to prevent malformed data from reaching your business logic.
