# Production-Ready Error Handling in Node.js (TypeScript)

Proper error handling is crucial for building resilient, production-ready Node.js applications. Unhandled errors can crash your server, leave it in an inconsistent state, or leak sensitive system details to users.

## 1. Operational vs. Programmer Errors

Before diving into code, it's essential to understand the two main types of errors:

*   **Operational Errors:** These represent run-time problems experienced by correctly written programs. They are expected to happen (e.g., failed to connect to database, request timeout, user provided invalid input, file not found). You should handle these gracefully.
*   **Programmer Errors:** These are bugs in the code (e.g., trying to read a property of `undefined`, syntax errors, failing to handle an asynchronous rejection). The application cannot recover from these; it should crash and be restarted by a process manager (like PM2 or Kubernetes).

## 2. Centralized Error Handling

Instead of handling errors randomly throughout your application, use a centralized approach. This ensures consistency in how errors are logged and how responses are formatted.

### Step 1: Create a Custom `AppError` Class

Extend the built-in `Error` class to include useful metadata like HTTP status codes and a flag to distinguish operational errors from programmer errors.

```typescript
// src/utils/AppError.ts
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly isOperational: boolean;

  constructor(message: string, statusCode: number, isOperational = true) {
    super(message);
    
    this.statusCode = statusCode;
    this.isOperational = isOperational;

    // Maintain proper stack trace for where our error was thrown (only available on V8)
    Error.captureStackTrace(this, this.constructor);
  }
}
```

### Step 2: Implement a Global Error Handler Middleware (Express.js Example)

In Express, a middleware function with four arguments is automatically treated as an error handler.

```typescript
// src/middlewares/errorHandler.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../utils/AppError';
// Assuming you have a logger set up (like Winston or Pino)
import logger from '../utils/logger'; 

export const errorHandler = (
  err: Error | AppError,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  let { statusCode, message } = err as AppError;
  
  // Default to 500 for unknown errors
  if (!statusCode) statusCode = 500;

  // Log the error
  logger.error(err.message, { stack: err.stack, path: req.path, method: req.method });

  // Do not leak stack traces to the client in production!
  const isProduction = process.env.NODE_ENV === 'production';

  res.status(statusCode).json({
    status: 'error',
    statusCode,
    message: isProduction && !((err as AppError).isOperational) 
      ? 'Internal Server Error' // Hide details of programmer errors in prod
      : message,
    stack: isProduction ? undefined : err.stack,
  });
};
```

### Step 3: Use the Middleware in your App

```typescript
// src/app.ts
import express from 'express';
import { errorHandler } from './middlewares/errorHandler';
import { AppError } from './utils/AppError';

const app = express();

// ... your routes ...
app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await findUser(req.params.id);
    if (!user) {
      // Throw operational error
      throw new AppError('User not found', 404);
    }
    res.json(user);
  } catch (error) {
    // Pass to global error handler
    next(error); 
  }
});

// 404 Handler for undefined routes
app.all('*', (req, res, next) => {
  next(new AppError(`Can't find ${req.originalUrl} on this server!`, 404));
});

// The global error handler must be the VERY LAST middleware
app.use(errorHandler);
```

## 3. Handling Asynchronous Errors cleanly

Using `try...catch` blocks in every Express route can become verbose. You can create a wrapper function to catch asynchronous errors and automatically pass them to `next()`.

```typescript
// src/utils/catchAsync.ts
import { Request, Response, NextFunction } from 'express';

export const catchAsync = (fn: Function) => {
  return (req: Request, res: Response, next: NextFunction) => {
    fn(req, res, next).catch(next);
  };
};
```

**Usage:**

```typescript
import { catchAsync } from '../utils/catchAsync';

app.get('/users/:id', catchAsync(async (req, res, next) => {
  const user = await findUser(req.params.id);
  if (!user) {
    throw new AppError('User not found', 404);
  }
  res.json(user);
}));
```

## 4. Catching Unhandled Rejections and Exceptions

Node.js processes will eventually exit on unhandled promise rejections. You must listen for these process-level events to log the error and perform a graceful shutdown.

```typescript
// src/server.ts
import app from './app';
import logger from './utils/logger';

const server = app.listen(3000, () => {
  logger.info('Server running on port 3000');
});

// Catch synchronous errors outside of express (e.g., typos in initialization code)
process.on('uncaughtException', (err: Error) => {
  logger.error('UNCAUGHT EXCEPTION! 💥 Shutting down...');
  logger.error(err.name, err.message, err.stack);
  
  // Exit immediately as the process is in an unclean state
  process.exit(1); 
});

// Catch asynchronous errors outside of express (e.g., failed DB connection on startup)
process.on('unhandledRejection', (err: Error) => {
  logger.error('UNHANDLED REJECTION! 💥 Shutting down...');
  logger.error(err.name, err.message, err.stack);
  
  // Graceful shutdown: close server, then exit
  server.close(() => {
    process.exit(1);
  });
});
```

## Best Practices Checklist

1.  **Use a Custom Error Class:** Always throw standard `AppError` objects to maintain consistency.
2.  **Centralize Error Handling:** Use a global Express error middleware.
3.  **Differentiate Error Types:** Understand Operational vs. Programmer errors.
4.  **Graceful Shutdown:** Listen to `uncaughtException` and `unhandledRejection` to log errors and exit safely.
5.  **Hide Details in Production:** Never expose stack traces or DB syntax errors to the client in a production environment.
6.  **Log Everything Relevant:** Use Winston or Pino to log the error message, stack trace, and request context (path, user ID) before sending the response.
