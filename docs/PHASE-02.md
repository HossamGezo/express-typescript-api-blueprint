# Phase 02: The Entry Point & Middleware Orchestration 🧠

## 📑 Table of Contents

- [Phase 02: The Entry Point \& Middleware Orchestration 🧠](#phase-02-the-entry-point--middleware-orchestration-)
  - [📑 Table of Contents](#-table-of-contents)
  - [1. Dependency Management \& Environment Setup](#1-dependency-management--environment-setup)
  - [2. Layer 02: App Initialization \& Optimized Environment Logic](#2-layer-02-app-initialization--optimized-environment-logic)
    - [📂 Deep Dive: The LiveReload Helper Architecture](#-deep-dive-the-livereload-helper-architecture)
      - [🧐 Why this approach? (The "Missing DevDependency" Trap)](#-why-this-approach-the-missing-devdependency-trap)
      - [❌ The Incorrect Way (Static Import):](#-the-incorrect-way-static-import)
      - [✅ The Professional Way (Dynamic CommonJS):](#-the-professional-way-dynamic-commonjs)
      - [📋 Helper Code Implementation:](#-helper-code-implementation)
      - [⚠️ The "Async Import" Pitfall:](#️-the-async-import-pitfall)
      - [🛠️ Library Breakdown:](#️-library-breakdown)
  - [3. Layer 03: Request Parsing \& The Security Shield 🛡️](#3-layer-03-request-parsing--the-security-shield-️)
    - [📋 Request Parsing Middlewares](#-request-parsing-middlewares)
    - [📋 Security Middlewares](#-security-middlewares)
  - [4. Layer 04: Assets, UI Engine, Documentation, and Error Management 🎨📚](#4-layer-04-assets-ui-engine-documentation-and-error-management-)
    - [4.1 Request Logging (`shared/middlewares/logger.middleware.ts`)](#41-request-logging-sharedmiddlewaresloggermiddlewarets)
    - [4.2 Static Assets \& View Engine Setup (MVC Layer)](#42-static-assets--view-engine-setup-mvc-layer)
    - [4.3 Interactive API Documentation (`shared/config/swagger.ts`)](#43-interactive-api-documentation-sharedconfigswaggerts)
      - [🧐 Deep Dive: The Swagger Configuration Logic](#-deep-dive-the-swagger-configuration-logic)
    - [4.4 Application Routing Architecture](#44-application-routing-architecture)
    - [4.5 Global Error Handling (`shared/middlewares/errors.middleware.ts`)](#45-global-error-handling-sharedmiddlewareserrorsmiddlewarets)
      - [🧐 Deep Dive: The Logic of Centralized Error Management](#-deep-dive-the-logic-of-centralized-error-management)
  - [5. Layer 05: The Server Bootstrap \& Database Resilience 🚀🗄️](#5-layer-05-the-server-bootstrap--database-resilience-️)
    - [5.1 The Startup Sequence (`index.ts`)](#51-the-startup-sequence-indexts)
    - [5.2 Database Logic \& Health Monitoring (`shared/config/db.ts`)](#52-database-logic--health-monitoring-sharedconfigdbts)
    - [🧐 Deep Dive: The Global Schema Transformation Plugin](#-deep-dive-the-global-schema-transformation-plugin)
      - [📋 The Helper Code (`shared/helpers/mongoose.helper.ts`)](#-the-helper-code-sharedhelpersmongoosehelperts)

---

## 1. Dependency Management & Environment Setup

In this layer, we load all essential tools and initialize our environment variables. Proper ordering here is critical to prevent "undefined" configuration errors.

```typescript
// --- External Libraries ---
import express from "express";
import { config } from "dotenv";
import helmet from "helmet";
import cors from "cors";
import path from "path";

/**
 * Library: swagger-ui-express & swagger-jsdoc
 * Purpose: Provides the UI and generator for interactive API documentation.
 * Enables developers to test endpoints directly from the browser.
 */
import { serve, setup } from "swagger-ui-express";

/**
 * Library: compression
 * Purpose: Decreases the response body size using Gzip.
 * Improves API performance and reduces data transfer costs.
 */
import compression from "compression";

/**
 * Library: hpp (HTTP Parameter Pollution)
 * Purpose: Protects against malicious query string manipulation (e.g., ?id=1&id=2).
 * It ensures only a single value is processed, preventing logic errors.
 */
import hpp from "hpp";

/**
 * Library: express-rate-limit
 * Purpose: Safeguards the server from DDoS and Brute-force attacks by
 * limiting the number of requests a single IP can make within a timeframe.
 */
import { rateLimit } from "express-rate-limit";

// --- Load Environment Variables ---
// Must be invoked immediately after imports to ensure all subsequent
// files (like database config) have access to process.env values.
config();

// --- Local Configurations & Helpers ---
import connectToDB from "./shared/config/db.js";
import swaggerSpec from "./shared/config/swagger.js";
import { setupLiveReload } from "./shared/helpers/livereload.helper.js";

// --- Global Middlewares ---
import logger from "./shared/middlewares/logger.middleware.js";
import { errorHandler, notFound } from "./shared/middlewares/errors.middleware.js";

// --- Router Files ---
import AuthRouter from "./modules/auth/auth.routes.js";
import PasswordRouter from "./modules/password/password.routes.js";
import UserRouter from "./modules/user/user.routes.js";
import AuthorRouter from "./modules/author/author.routes.js";
import BookRouter from "./modules/book/book.routes.js";
import UploadRouter from "./modules/upload/upload.routes.js";
```

---

## 2. Layer 02: App Initialization & Optimized Environment Logic

This layer initializes the Express engine and implements advanced logic to handle the differences between Development and Production environments.

```typescript
// 1. Initialize the Express application instance
const app = express();

/**
 * 2. Feature: Trust Proxy
 * Order: MUST be the first setting.
 * Purpose: REQUIRED for Render/Cloud deployments.
 * It tells Express it is behind a proxy (Load Balancer). This allows the
 * server to identify the real user's IP address instead of the proxy IP,
 * which is critical for the Rate Limiter to function correctly.
 */
app.set("trust proxy", 1);

/**
 * 3. Feature: setupLiveReload(app)
 * Order: Invoked immediately after app creation.
 * Purpose: Automatically refreshes the browser when EJS/CSS files change.
 * Placement: Needs to be early to "hook" into the response cycle before
 * other middlewares or routes interfere.
 */
setupLiveReload(app);

/**
 * 4. Feature: Compression Middleware
 * Logic: Strictly enabled in Production ONLY.
 * ⚠️ Why avoid it in Development?
 * During UI development, 'compression' can cause "ERR_CONTENT_DECODING_FAILED"
 * when rendering large EJS files (like welcome.ejs) alongside LiveReload.
 * Disabling it in DEV ensures a smooth design experience.
 */
if (process.env.NODE_ENV === "production") {
  app.use(compression());
}
```

### 📂 Deep Dive: The LiveReload Helper Architecture

To maintain a clean and production-ready `index.ts`, the LiveReload logic is isolated in a helper file: `src/shared/helpers/livereload.helper.ts`.

#### 🧐 Why this approach? (The "Missing DevDependency" Trap)

If we use a standard `import` at the top of the file, the Production Image will **CRASH** because `devDependencies` (like `livereload`) are removed during the multi-stage Docker build.

#### ❌ The Incorrect Way (Static Import):

```typescript
import livereload from "livereload"; // This will throw 'Module Not Found' in production!
```

#### ✅ The Professional Way (Dynamic CommonJS):

We use `createRequire` to load these libraries only when `NODE_ENV` is `development`. We avoided dynamic `import()` because it is asynchronous and often causes complex TypeScript safety issues during the middleware injection.

#### 📋 Helper Code Implementation:

```typescript
// src/shared/helpers/livereload.helper.ts

import type { Application } from "express";
import path from "path";
// --- Enable CommonJS imports inside ESM ---
import { createRequire } from "module";

const require = createRequire(import.meta.url);

/**
 * @desc Setup LiveReload for development environment
 */
export const setupLiveReload = (app: Application) => {
  if (process.env.NODE_ENV === "development") {
    // Dynamically require libraries to avoid production crashes
    const livereload = require("livereload");
    const connectLivereload = require("connect-livereload");

    // Initialize the server to watch for UI changes (EJS/CSS)
    const liveReloadServer = livereload.createServer({
      exts: ["ejs", "css", "js", "png", "jpg"],
      debug: false, // Set to true only for troubleshooting
    });

    // Watch specific asset directories
    liveReloadServer.watch([
      path.join(process.cwd(), "public"),
      path.join(process.cwd(), "views"),
    ]);

    // Inject the livereload script into the browser response automatically
    app.use(connectLivereload());
  }
};
```

#### ⚠️ The "Async Import" Pitfall:

You might be tempted to use dynamic ESM imports like this:

```typescript
// ❌ INCORRECT & PROBLEMATIC
const livereload = await import("livereload");
```

**Why avoid this?**

- **TypeScript Type Issues:** Dynamic imports of legacy CommonJS packages often result in `unknown` types. This forces you to use `as any` and lose all the benefits of TypeScript safety.
- **Race Conditions:** Using `await` inside the setup function forces the entire chain to be asynchronous. If not handled perfectly, the server might start listening before the LiveReload middleware is fully injected.
- **The Solution:** Using `createRequire` provides a **synchronous** and **type-safe** way to bridge the gap between ESM and CJS during app startup.

#### 🛠️ Library Breakdown:

- **`livereload`**: Creates a WebSocket server that monitors file changes.
- **`connect-livereload`**: A middleware that injects a small script into your HTML/EJS so the browser knows when to refresh.

---

## 3. Layer 03: Request Parsing & The Security Shield 🛡️

This layer is responsible for cleaning incoming data and protecting the server from common web vulnerabilities.

### 📋 Request Parsing Middlewares

Before incoming data can be processed by our controllers, it must be parsed into a readable JavaScript object. This layer handles both API (JSON) and Web Form data.

```typescript
// 1. Parse incoming JSON payloads (standard for REST APIs)
app.use(express.json());

/**
 * 2. Feature: express.urlencoded
 * Purpose: Allows Express to parse data from standard HTML forms (MVC/EJS).
 * Without this, req.body would be empty when submitting the "Forgot Password" form.
 *
 * Logic of { extended: false }:
 * - false: Uses the built-in Node.js "querystring" library. It is lightweight and
 *          perfect for simple forms containing basic key-value pairs.
 * - true:  Uses the "qs" library, which supports parsing complex nested objects
 *          (e.g., user[name]=hossam&user[age]=25).
 *
 * We use 'false' here for better performance as our forms use simple data structures.
 */
app.use(express.urlencoded({ extended: false }));
```

---

### 📋 Security Middlewares

```typescript
/**
 * Library: hpp (HTTP Parameter Pollution)
 * Purpose: Prevents attackers from sending multiple query parameters with the same name.
 *
 * Example:
 * Request: GET /api/books?price=10&price=500
 * Without HPP: req.query.price becomes an array ['10', '500'], which might crash your logic.
 * With HPP: It sanitizes the query and only keeps the last value ('500'), ensuring your code receives a string.
 */
app.use(hpp());

/**
 * Library: Helmet
 * Purpose: Sets various HTTP headers to secure the app.
 *
 * Logic:
 * In Development: We use { contentSecurityPolicy: false, crossOriginEmbedderPolicy: false }.
 * - CSP (Content Security Policy): A security layer that detects and mitigates certain types of attacks (like XSS).
 *   We disable it in DEV because the default policy blocks inline scripts required by Swagger UI and LiveReload.
 * - COEP (Cross-Origin Embedder Policy): We disable it to allow the browser to load assets (like images or scripts)
 *   from different origins during development.
 */
if (process.env.NODE_ENV === "development") {
  app.use(
    helmet({ contentSecurityPolicy: false, crossOriginEmbedderPolicy: false }),
  );
} else {
  app.use(helmet());
}

/**
 * Library: CORS (Cross-Origin Resource Sharing)
 * Purpose: Restricts which domains can interact with your API.
 */
app.use(
  cors({
    origin: process.env.CLIENT_URL, // Only allow your frontend domain
    methods: ["GET", "POST", "PUT", "DELETE"],
  }),
);

/**
 * Library: express-rate-limit
 * Purpose: The "Bodyguard" of the server. It prevents DDoS and Brute-force attacks.
 *
 * Key Concept:
 * windowMs (Window Milliseconds): The timeframe for which requests are checked (e.g., 15 minutes).
 * limit: The maximum number of requests allowed from a single IP during the windowMs.
 */
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes timeframe
  limit: 100, // Max 100 requests per IP
  message: {
    message: "Too many requests from this IP, please try again after 15 minutes",
  },
});

// Apply rate limiting to protect sensitive API and Password routes
app.use("/api", limiter);
app.use("/password", limiter);
```

---

## 4. Layer 04: Assets, UI Engine, Documentation, and Error Management 🎨📚

This layer transforms our backend into a full-featured hybrid application (MVC + REST API) and establishes the final safeguards for the system.

### 4.1 Request Logging (`shared/middlewares/logger.middleware.ts`)

The logger is our first point of visibility. It must be placed before routers to track every incoming request.

```typescript
import type { Request, Response, NextFunction } from "express";

/**
 * @desc Logger middleware to track HTTP requests
 */
const logger = (req: Request, _res: Response, next: NextFunction) => {
  const date = new Date().toISOString(); // High-precision ISO timestamp
  const method = req.method;
  const url = `${req.protocol}://${req.get("host")}${req.originalUrl}`;

  console.log(`[${date}] ${method} ${url}`);
  next();
};

export default logger;
```

**Why this approach?**

- **Dynamic Host Detection:** Using `req.get("host")` ensures logs remain accurate whether the app is on `localhost:5001` or a production domain like `render.com`.
- **Visibility:** Tracking requests before they hit controllers allows us to debug middleware failures (like Auth errors) effectively.

---

### 4.2 Static Assets & View Engine Setup (MVC Layer)

We configure Express to serve physical assets and render dynamic HTML pages using the EJS template engine.

```typescript
/**
 * Feature: Static Folder & View Engine Setup
 *
 * 1. express.static: Serves files like CSS, Images, and Client-side JS.
 * 2. process.cwd(): Points to the project root. Essential for ESM and Docker consistency.
 * 3. View Engine: Configures EJS to render files from the /views directory.
 */
app.use(express.static(path.join(process.cwd(), "public")));
app.set("views", path.join(process.cwd(), "views"));
app.set("view engine", "ejs");
```

---

### 4.3 Interactive API Documentation (`shared/config/swagger.ts`)

We use Swagger (OpenAPI 3.0) to provide a living, interactive documentation dashboard.

```typescript
// --- Swagger UI Middleware ---
app.use("/api-docs", serve, setup(swaggerSpec));
```

#### 🧐 Deep Dive: The Swagger Configuration Logic

To maintain high-quality documentation, we centralize the metadata and reusable components in `src/shared/config/swagger.ts`.

**The Structure breakdown:**

- **`openapi`**: Defines the version of the specification.
- **`info`**: Contains the title, version, and a rich description (including test credentials).
- **`servers`**: Lists the environments where the API can be tested (Local vs Production).
- **`components`**: The "Warehouse" of the documentation. It stores reusable `schemas` (data structures) and `responses` (error/success messages) to avoid duplication.

```typescript
import swaggerJsdoc, {type Options} from "swagger-jsdoc";

const options: Options = {
  definition: {
    openapi: "3.0.0",
    info: {
      title: "Bookstore API",
      version: "2.0.0",
      description:
        "A professional, high-performance Bookstore system featuring a hybrid **RESTful API** and **MVC architecture**, built with Node.js, Express, and TypeScript. \n\n" +
        "--- \n" +
        "🔑 **Testing Admin Features:** \n" +
        "To test Admin-only routes (Create/Update/Delete), please log in with: \n" +
        "- **Email:** admin@gmail.com \n" +
        "- **Password:** Password@123 \n" +
        "*(Make sure to copy the returned token and paste it into the 'Authorize' button above)*",
      contact: {
        name: "Hossam Gezo",
        url: "https://github.com/HossamGezo",
        email: "ha2ghossam10@gmail.com",
      },
    },
    servers: [
      {
        url: "https://bookstore-api-0fy2.onrender.com",
        description: "Production server",
      },
      {
        url: "http://localhost:5001",
        description: "Development server",
      },
    ],
    tags: [
      {name: "Auth", description: "Authentication operations"},
      {name: "User", description: "User profile and administration"},
      {name: "Password", description: "Password recovery"},
      {name: "Author", description: "Author details and profiles"},
      {name: "Book", description: "Book catalog management"},
      {name: "Upload", description: "File and image upload services"},
    ],
    components: {
      securitySchemes: {
        tokenAuth: {
          type: "apiKey",
          in: "header",
          name: "token",
          description: "Enter your JWT token",
        },
      },
      schemas: {
        // Here we define all our Data Models (User, Author, Book, etc.)
        // This allows us to use $ref in our routes for clean documentation.
        User: {},
        Book: {},
        Author: {},
        ErrorResponse: {},
        // ... and more
      },
      responses: {
        // Pre-defined error responses (400, 401, 403, 404)
        BadRequestError: {},
        NotFoundError: {},
        // ... and more
      },
    },
  },
  // Automated scanning for documentation in YAML files
  apis: ["./src/modules/**/*.swagger.yaml", "./src/modules/**/*.swagger.yml"],
};

const swaggerSpec = swaggerJsdoc(options);
export default swaggerSpec;
```

---

### 4.4 Application Routing Architecture

We separate our application into "Data API" routes and "MVC/Workflow" routes for better organization.

```typescript
/**
 * Route: Home/Welcome Page (MVC)
 */
app.get("/", (_req, res) => {
  res.render("home/welcome");
});

/**
 * Routers: Modular Service Injection
 */
app.use("/api/auth", AuthRouter);
app.use("/password", PasswordRouter);
app.use("/api/users", UserRouter);
app.use("/api/authors", AuthorRouter);
app.use("/api/books", BookRouter);
app.use("/api/upload", UploadRouter);
```

---

### 4.5 Global Error Handling (`shared/middlewares/errors.middleware.ts`)

Error handlers **MUST** be the last middlewares in the file. They act as the final safety net for the entire application.

#### 🧐 Deep Dive: The Logic of Centralized Error Management

- **The Two-Step Filter:** We first catch missing routes using `notFound`, then handle all errors (thrown manually or by libraries) in the `errorHandler`.
- **Database Error Translation:** We specifically intercept MongoDB errors:
  - `CastError`: Occurs when an invalid ID format is provided.
  - `11000`: Occurs when a duplicate unique field (like Email) is entered.
- **Multer Integration:** Automatically catches file upload errors.
- **Standardized Response:** We use our `failureResponse` helper to ensure the error JSON structure is consistent across the entire API.

```typescript
import type { Request, Response, NextFunction } from "express";
import { failureResponse } from "../helpers/response.helper.js";

/**
 * @desc Handle 404 Routes (Page/Route not found)
 */
export const notFound = (req: Request, res: Response, next: NextFunction) => {
  const error = new Error(`Not found ${req.originalUrl}`);
  res.status(404);
  next(error);
};

/**
 * @desc Global Error Handler (The Final Guard)
 */
export const errorHandler = (err: any, _req: Request, res: Response, _next: NextFunction) => {
  let statusCode = res.statusCode === 200 ? 500 : res.statusCode;
  let message = err.message || "Something went wrong!";

  // --- MongoDB Bad ObjectId (CastError) ---
  if (err.name === "CastError") {
    statusCode = 400;
    message = "Invalid ID Format";
  }

  // --- MongoDB Duplicate Key Error (e.g. Email exists) ---
  if (err.code === 11000) {
    statusCode = 400;
    message = "Duplicate field value entered";
  }

  // --- Multer File Upload Errors ---
  if (err.name === "MulterError") {
    statusCode = 400;
  }

  // --- Format the failure message using our standardized helper ---
  const result = failureResponse({ statusCode, message });

  res.status(statusCode).json({
    ...result,
    // Hide the technical stack trace in production for security
    stack: process.env.NODE_ENV === "production" ? null : err.stack,
  });
};
```

---

## 5. Layer 05: The Server Bootstrap & Database Resilience 🚀🗄️

This final layer is responsible for "igniting" the engine. We follow a **Database-First** startup strategy, ensuring the API never accepts requests until the database connection is fully established.

### 5.1 The Startup Sequence (`index.ts`)

Instead of starting the server immediately, we use a Promise-based approach.

```typescript
// --- Server Startup Logic ---
const PORT = process.env.PORT || 8000;

// Connect to the database first, then start the server
connectToDB().then(() =>
  app.listen(PORT, () =>
    console.log(
      `✅ Server is running in ${process.env.NODE_ENV || "development"} mode on port ${PORT}`
    )
  )
);
```

**Why this approach?**

- **Prevention of "Zombies":** If the database is down, the server should not start. This prevents the application from entering a "broken" state where it's live but cannot serve data.

---

### 5.2 Database Logic & Health Monitoring (`shared/config/db.ts`)

Our database configuration implements environment-aware URI selection and real-time health monitoring.

```typescript
import mongoose from "mongoose";
import { globalSchemaPlugin } from "../helpers/mongoose.helper.js";

// 1. Activate the Global Transformation Plugin
// Must be called BEFORE any model is loaded to ensure global application.
mongoose.plugin(globalSchemaPlugin);

const connectToDB = async () => {
  // 2. Conditional URI Selection based on Environment
  const mongoUri = process.env.NODE_ENV === "development"
      ? process.env.MONGO_URI_DEV
      : process.env.MONGO_URI_PRO;

  if (!mongoUri) {
    console.error("❌ MONGO_URI is missing! Check your .env file.");
    process.exit(1); // Kill the process if infrastructure is missing
  }

  try {
    // 3. Initial connection attempt
    await mongoose.connect(mongoUri);
    console.log("✅ Connected to MongoDB...");

    /**
     * 4. EVENT LISTENERS: Monitoring DB health after startup
     */
    // Tracks technical issues while the server is running
    mongoose.connection.on("error", (err) => {
      console.error(`🛑 MongoDB runtime connection error: ${err}`);
    });

    // Detects when the connection drops (Mongoose will auto-reconnect)
    mongoose.connection.on("disconnected", () => {
      console.warn("⚠️ MongoDB disconnected! Attempting to reconnect...");
    });

  } catch (error) {
    console.error("❌ Failed to connect to MongoDB during startup!", error);
    process.exit(1);
  }
};

export default connectToDB;
```

---

### 🧐 Deep Dive: The Global Schema Transformation Plugin

By default, MongoDB uses `_id` and includes an internal version field `__v`. To make our API **"Frontend Friendly"**, we use a global plugin to transform the data structure automatically.

#### 📋 The Helper Code (`shared/helpers/mongoose.helper.ts`)

```typescript
/**
 * @desc Transforms MongoDB documents to a clean JSON structure
 * 1. Converts '_id' to 'id' for easier frontend usage.
 * 2. Removes the internal '__v' field.
 */
export const globalSchemaPlugin = (schema: any) => {
  const transform = (_doc: any, ret: any) => {
    ret.id = ret._id; // Add a friendly 'id' field
    delete ret._id;   // Remove the original MongoDB '_id'
    delete ret.__v;   // Remove the internal version key
    return ret;
  };

  // Apply the transformation whenever the document is converted to JSON or an Object
  schema.set("toJSON", { transform });
  schema.set("toObject", { transform });
};
```

**Why is this S-Tier?**

1.  **Automation:** You write this logic **once**, and it applies to `User`, `Author`, `Book`, and any future models automatically.
2.  **Consistency:** Every single endpoint in your API will return a consistent ID format, improving the **Developer Experience (DX)** for frontend engineers.
3.  **Encapsulation:** The transformation logic is isolated from the database connection and the business logic.

---
