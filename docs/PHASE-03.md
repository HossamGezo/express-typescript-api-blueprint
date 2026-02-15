# Phase 03: Modular Architecture & Shared Core Logic 🏛️🧠

## 1. The Modular Structure Overview

In this phase, we move from a traditional layered structure to a **Modular (Feature-based) Architecture**. This ensures that each feature (e.g., Books, Authors) is self-contained, making the codebase easier to navigate and scale.

### 🌳 The src Directory Tree:

```text
src/
├── index.ts               # App Entry Point (Refer to Phase 02)
├── models/                # Centralized Mongoose Models
├── shared/                # Global Core (Helpers, Middlewares, Types)
├── modules/               # Feature Modules (The Heart of Business Logic)
│   ├── auth/              # Login & Register logic
│   ├── author/            # Author CRUD + Seeders
│   ├── book/              # Book CRUD + Seeders
│   ├── user/              # User Management
│   ├── password/          # Reset Password (MVC)
│   └── upload/            # Image Upload logic
└── styles/                # Tailwind CSS Input
```

### 📦 Anatomy of a Module:

Every folder inside `src/modules` follows a standardized structure to ensure consistency:

- **`.controller.ts`**: Handles HTTP requests/responses.
- **`.service.ts`**: Contains the business logic and DB queries.
- **`.routes.ts`**: Defines the endpoints.
- **`.validation.ts`**: Zod schemas for request body.
- **`.seeder.ts` & `.data.ts`**: For database seeding.
- **`docs/*.yaml`**: Swagger documentation for the module.

---

## 2. The Shared Layer: Global Infrastructure 🛠️

The `src/shared` directory contains tools that are used across all modules. This prevents code duplication and enforces a "Single Source of Truth".

```text
shared/config/
├── db.ts           # [Explained in Phase-02] - DB Connection & Plugins
└── swagger.ts      # [Explained in Phase-02] - API Documentation Metadata
```

### 📋 2.1 The Query Helper (`shared/helpers/query.helper.ts`)

This is the most critical helper in the project. It provides a **Generic Engine** for Pagination, Filtering, and Sorting for any Mongoose model.

```text
shared/helpers/
├── livereload.helper.ts   # [Explained in Phase-02] - Development Auto-refresh
├── mongoose.helper.ts     # [Explained in Phase-02] - Global Data Transformation
├── query.helper.ts        # [Explained Below] - Generic Pagination & Filtering Engine
└── response.helper.ts     # [Explained Below] - Standardized Success/Failure Factory
```

```typescript
import { Model, type PopulateOptions } from "mongoose";

interface QueryOptions {
  page: number;
  limit?: number;
  filter?: any;
  sort?: any;
  populate?: PopulateOptions | PopulateOptions[];
  select?: string;
}

/**
 * @desc A generic helper to handle complex database queries
 * @template T - The Mongoose Model Type
 */
export const queryOperations = async <T>(model: Model<T>, options: QueryOptions) => {
  const { page, limit = 2, filter = {}, sort = { createdAt: -1 }, populate, select } = options;
  const skip = (page - 1) * limit;

  // Execute data fetching and count in parallel for maximum performance
  const [items, totalItems] = await Promise.all([
    model
      .find(filter)
      .sort(sort)
      .skip(skip)
      .limit(limit)
      .populate((populate as any) || [])
      .select(select || ""),
    model.countDocuments(filter),
  ]);

  const totalPages = Math.ceil(totalItems / limit);

  // Business Rule: Return 404 if the requested page exceeds total pages
  if (page > totalPages && totalItems > 0) {
    return {
      success: false,
      statusCode: 404,
      message: `Page ${page} not found. Total pages available: ${totalPages}`,
    };
  }

  return {
    success: true,
    data: { items, totalItems, currentPage: page, totalPages },
  };
};
```

---

### 📋 2.2 The Response Factory (`shared/helpers/response.helper.ts`)

Unifies the structure of all API responses, making it easy for Frontend developers to handle data.

```typescript
interface FailureResponseParams {
  statusCode?: number;
  message: string;
}

export interface FailureResponse {
  success: false;
  statusCode: number;
  message: string;
}

/**
 * @desc Standard failure response generator
 */
export const failureResponse = ({ statusCode = 400, message }: FailureResponseParams): FailureResponse => {
  return { success: false, statusCode, message };
};

/**
 * @desc Specialized 404 Not Found response
 */
export const notFoundResponse = (target: string): FailureResponse => {
  const formattedTarget = target.charAt(0).toUpperCase() + target.slice(1).toLowerCase();
  return failureResponse({ statusCode: 404, message: `${formattedTarget} not found` });
};

/**
 * @desc Standard success response wrapper using Generics <T>
 */
export const successResponse = <T>(data: T) => {
  return {
    success: true as const, // Uses 'as const' for literal type discrimination
    data,
  };
};
```

---

## 3. Advanced Middlewares 🛡️📸

```text
shared/middlewares/
├── errors.middleware.ts             # [Explained in Phase-02] - Centralized Error Handling
├── logger.middleware.ts             # [Explained in Phase-02] - Request Tracking
├── upload.middleware.ts             # [Explained Below] - Multer Image Configuration
├── validateObjectId.middleware.ts   # [Explained Below] - Request Sanitization (ID Check)
└── verifyToken.middleware.ts        # [Explained Below] - Authentication & RBAC Guard
```

### 📋 3.1 Image Upload (`shared/middlewares/upload.middleware.ts`)

Configures **Multer** for secure file storage and filename sanitization.

```typescript
import multer, { type FileFilterCallback } from "multer";
import path from "path";

const storage = multer.diskStorage({
  destination: (_req, _file, cb) => {
    // Uses process.cwd() to ensure correct pathing in both Local and Docker environments
    cb(null, path.join(process.cwd(), "public/images"));
  },
  filename: (_req, file, cb) => {
    const uniqueSuffix = new Date().toISOString().replace(/:/g, "-");
    // Remove spaces and special characters for URL safety
    const cleanFileName = file.originalname.toLowerCase().replace(/\s+/g, "_").replace(/[^\w.-]/g, "");
    cb(null, `${uniqueSuffix}-${cleanFileName}`);
  },
});

const fileFilter = (_req: any, file: Express.Multer.File, cb: FileFilterCallback) => {
  if (file.mimetype.startsWith("image")) {
    cb(null, true);
  } else {
    cb(new Error("Unsupported file format! Please upload an image.") as any, false);
  }
};

export const upload = multer({
  storage,
  fileFilter,
  limits: { fileSize: 2 * 1024 * 1024 }, // Limit size to 2MB
});
```

---

### 📋 3.2 Authentication & Authorization Guard (`shared/middlewares/verifyToken.middleware.ts`)

The security core that decodes JWTs and handles Role-Based Access Control (RBAC).

```typescript
import type { NextFunction, Request, Response } from "express";
import jwt from "jsonwebtoken";

/**
 * @desc Verify JWT Token Middleware
 */
export const verifyToken = (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  // --- Check Existence
  /**
   * FUTURE_SECURITY_UPGRADE & STANDARDS:
   * Currently, we are using a custom 'token' header for simplicity.
   * In a production-grade environment, consider these two industry standards:
   *
   * 1. AUTHORIZATION HEADER (Bearer Token):
   *    - Format: Authorization: Bearer <token>
   *    - Usage: Best for Mobile Apps, Third-party integrations, and Server-to-Server communication.
   *    - Implementation: const token = req.headers.authorization?.split(" ")[1];
   *
   * 2. HTTP-ONLY COOKIES (Secure Storage):
   *    - Usage: Best for Web Applications (React/Next.js) to prevent XSS attacks.
   *    - Security: Browsers manage cookies automatically and JS cannot access them.
   *    - Implementation: const token = req.cookies.token; (requires cookie-parser)
   */
  const token = req.headers.token;
  if (!token || typeof token !== "string") {
    res.status(401).json({ message: "No token provided or invalid format" });
    return;
  }

  // --- Ensure JWT_SECRET_KEY exists (Typescript safety check)
  if (!process.env.JWT_SECRET_KEY) {
    throw new Error("JWT_SECRET_KEY is missing!");
  }

  // --- Decoded Token
  try {
    const decoded = jwt.verify(
      token,
      process.env.JWT_SECRET_KEY,
    ) as UserPayload;
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: `Invalid or expired token, ${error}` });
  }
};

/**
 * @desc Verify Token And Authorization Middleware
 */
export const verifyTokenAndAuthorization = (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  verifyToken(req, res, () => {
    // --- Ensure that token belongs to user
    if (req.user?.id !== req.params.id && !req.user?.isAdmin) {
      res.status(403).json({
        message: "Access denied. You are not authorized to perform this action",
      });
      return;
    }
    next();
  });
};

/**
 * @desc Verify Token And Admin Middleware
 */
export const verifyTokenAndAdmin = (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  verifyToken(req, res, () => {
    // --- Ensure that token belongs Admin
    if (!req.user?.isAdmin) {
      res.status(403).json({
        message: "You are not allowed, only admin allowed",
      });
      return;
    }
    next();
  });
};
```

---

### 📋 3.3 Request Sanitization (`shared/middlewares/validateObjectId.middleware.ts`)

This middleware acts as a "Fast-Fail" guard. It ensures that any ID passed in the URL is a valid MongoDB ObjectId **before** it even reaches the Controller or the Database.

```typescript
import type { Request, Response, NextFunction } from "express";
import mongoose from "mongoose";

/**
 * @desc Validate MongoDB ObjectId
 * @purpose To prevent Mongoose "CastError" and unnecessary database trips.
 */
export const validateObjectId = (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  const id = req.params.id;

  // Check if the ID exists and follows the 24-character hex format
  if (!id || !mongoose.Types.ObjectId.isValid(id)) {
    res.status(400).json({ message: "Invalid ID Format" });
    return; // Stop the request cycle here
  }

  next(); // ID is safe, proceed to the next middleware or controller
};
```

#### 🧐 Why is this S-Tier Logic?

1.  **Database Protection:** Without this middleware, if a user sends a malformed ID (e.g., `/api/books/123`), Mongoose will attempt to query the database and then throw a `CastError`. Our middleware stops this at the entry point.
2.  **Performance Optimization:** Rejecting an invalid request in the middleware takes microseconds. A database round-trip takes significantly longer. By filtering junk requests early, we save valuable server resources.
3.  **HTTP Semantics:** It allows us to clearly distinguish between a **400 Bad Request** (Malformed ID) and a **404 Not Found** (Valid ID but record doesn't exist).

---

## 4. Global Typing & Validation Strategy 🧬

```text
shared/types/
├── express.d.ts     # [Explained Below] - Global Express Namespace Extension
└── service.ts       # [Explained Below] - Unified ServiceResult Contract
```

```text
shared/validations/
└── query.validation.ts   # [Explained Below] - Centralized URL Query Validation
```

### 📋 4.1 Global Express Augmentation (`shared/types/express.d.ts`)

To allow `req.user` to be recognized by TypeScript across all controllers.

```typescript
// eslint-disable-next-line @typescript-eslint/no-unused-vars
import type { Request } from "express";

/**
 * 🧐 Why 'eslint-disable'?
 * We import 'Request' only to trigger the type augmentation.
 * Since it's not used as a variable, ESLint would flag it as unused.
 */
declare global {
  interface UserPayload { id: string; isAdmin: boolean; }
  namespace Express {
    interface Request { user?: UserPayload; }
  }
}
```

### 📋 4.2 Generic Service Interface (`shared/types/service.ts`)

Ensures all services return a predictable structure.

```typescript
export interface ServiceResult<T = any> {
  success: boolean;
  statusCode?: number;
  message?: string;
  data?: T;
}
```

### 📋 4.3 Base Query Validation (`shared/validations/query.validation.ts`)

Centralizes Zod validation for search and pagination.

```typescript
import { z } from "zod";

const BaseQuerySchema = z.object({
  page: z
    .string()
    .trim()
    .regex(/^\d+$/, { message: "Page number must be a number" })
    .transform(Number)
    .optional(),
  search: z.string().trim().optional(),
  sortBy: z.string().trim().optional(),
});

export const BookQuerySchema = BaseQuerySchema.extend({
  minPrice: z
    .string()
    .trim()
    .regex(/^\d+$/, { message: "Minimum Price must be a number" })
    .transform(Number)
    .optional(),
  maxPrice: z
    .string()
    .trim()
    .regex(/^\d+$/, { message: "Maximum Price must be a number" })
    .transform(Number)
    .optional(),
});

export const AuthorQuerySchema = BaseQuerySchema;
export const UserQuerySchema = BaseQuerySchema;

export type BookQueryDto = z.infer<typeof BookQuerySchema>;
export type AuthorQueryDto = z.infer<typeof AuthorQuerySchema>;
export type UserQueryDto = z.infer<typeof UserQuerySchema>;

```

---
