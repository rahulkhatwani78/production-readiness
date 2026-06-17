# Production-Ready Logging in Node.js (TypeScript)

Logging is a critical component of any production application. It provides visibility into the application's behavior, helps in debugging issues, and can be used for monitoring and alerting. In the Node.js ecosystem, `console.log` is insufficient for production. You need a structured logging library. Two of the most popular choices are **Winston** and **Pino**.

## Why Structured Logging?
In production, logs are often shipped to centralized log management systems (like ELK stack, Datadog, or AWS CloudWatch). These systems parse logs better when they are in JSON format. Structured logging means writing logs as JSON objects containing key-value pairs, making them easily searchable and indexable.

## 1. Winston

Winston is a versatile logging library with support for multiple transports (where logs are sent: console, file, HTTP, etc.). It's highly configurable and has a massive community.

### Installation
```bash
npm install winston
npm install -D @types/winston # Not needed in newer versions as winston includes types, but good to know
```

### TypeScript Implementation

Create a `logger.ts` file:

```typescript
import winston from 'winston';

// Define custom log levels if needed, or use the default npm levels:
// error: 0, warn: 1, info: 2, http: 3, verbose: 4, debug: 5, silly: 6

const levels = {
  error: 0,
  warn: 1,
  info: 2,
  http: 3,
  debug: 4,
};

const level = () => {
  const env = process.env.NODE_ENV || 'development';
  const isDevelopment = env === 'development';
  return isDevelopment ? 'debug' : 'info';
};

const colors = {
  error: 'red',
  warn: 'yellow',
  info: 'green',
  http: 'magenta',
  debug: 'white',
};

winston.addColors(colors);

// Format for development (human-readable)
const consoleFormat = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss:ms' }),
  winston.format.colorize({ all: true }),
  winston.format.printf(
    (info) => `${info.timestamp} ${info.level}: ${info.message}`,
  ),
);

// Format for production (JSON)
const jsonFormat = winston.format.combine(
  winston.format.timestamp(),
  winston.format.json()
);

const logger = winston.createLogger({
  level: level(),
  levels,
  // Use JSON format by default
  format: jsonFormat,
  transports: [
    // Write all logs to console
    new winston.transports.Console({
      format: process.env.NODE_ENV === 'production' ? jsonFormat : consoleFormat,
    }),
    // You can add File transports here as well
    // new winston.transports.File({
    //   filename: 'logs/error.log',
    //   level: 'error',
    // }),
  ],
});

export default logger;
```

### Usage

```typescript
import logger from './logger';

logger.info('Server started on port 3000');
logger.error('Failed to connect to database', { error: 'Connection timeout' });
```

---

## 2. Pino

Pino focuses on speed and extremely low overhead. It is significantly faster than Winston because it is designed to log JSON inherently and minimizes serialization overhead. 

### Installation
```bash
npm install pino
npm install -D @types/pino pino-pretty
```
*Note: `pino-pretty` is a separate package used to format JSON logs into a human-readable format for development.*

### TypeScript Implementation

Create a `logger.ts` file:

```typescript
import pino from 'pino';

const isProduction = process.env.NODE_ENV === 'production';

const logger = pino({
  level: isProduction ? 'info' : 'debug',
  // Redact sensitive information
  redact: ['password', 'req.headers.authorization', 'user.email'],
  // Format the time
  timestamp: pino.stdTimeFunctions.isoTime,
  
  // Use pino-pretty only in development
  ...(isProduction ? {} : {
    transport: {
      target: 'pino-pretty',
      options: {
        colorize: true,
        translateTime: 'SYS:standard',
        ignore: 'pid,hostname', // Keep the output clean
      },
    },
  }),
});

export default logger;
```

### Usage

```typescript
import logger from './logger';

logger.info({ port: 3000 }, 'Server is up and running');
logger.error({ err: new Error('DB connection failed') }, 'Database error');
```

## Winston vs Pino: Which one to choose?

| Feature | Winston | Pino |
| :--- | :--- | :--- |
| **Performance** | Good, but heavier | **Excellent** (Extremely fast) |
| **Flexibility** | **High** (Multiple transports built-in) | Medium (Relies on external workers/transports) |
| **JSON Logging** | Supported via formatters | **Native** and default |
| **Learning Curve**| Slightly steeper | Simple and straightforward |

**Recommendation:** 
- If your application is under high load and performance/low overhead is critical, choose **Pino**.
- If you have complex logging requirements, need multiple different outputs (transports) directly from the app (e.g., to a file, to Loggly, to the console simultaneously), and want a highly customizable format, choose **Winston**.

## Best Practices
1. **Never use `console.log`**: Always use your configured logger.
2. **Contextual Logging**: Attach useful metadata (User ID, Request ID, Error stack).
3. **Redaction**: Never log passwords, tokens, or PII (Personally Identifiable Information). Pino has native redaction; Winston requires custom formatters.
4. **Log Levels**: Use appropriate levels (`error` for failures, `warn` for unexpected situations that don't stop the flow, `info` for general lifecycle events, `debug` for troubleshooting).
