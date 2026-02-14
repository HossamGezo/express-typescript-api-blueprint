# Phase 01: Professional Environment & Infrastructure Setup 🏗️

## 1. Initializing the Repository

Create a new repository on GitHub with:

- **Visibility:** Public.
- **License:** MIT License.
- **Gitignore:** Node.js Template.

---

## 2. Refining the Git Strategy (.gitignore) 🧹

Add these custom rules to the end of your `.gitignore` to ensure a clean repository:

```text
# --- Custom Project Ignored Files ---

# MacOS internal display files
.DS_Store

# Husky internal directory
.husky/_/

# Tailwind CSS generated output
public/styles/output.css
```

---

## 3. Full Dependency Installation (Compatibility Stack) 📦

Run these commands to install the exact versions used in this blueprint to ensure 100% compatibility:

### Step 1: Install Production Dependencies

```bash
npm install bcryptjs compression cors dotenv ejs express express-async-handler express-rate-limit helmet hpp jsonwebtoken mongoose multer nodemailer swagger-jsdoc swagger-ui-express zod
```

### Step 2: Install Development Dependencies (DX)

```bash
npm install -D typescript @eslint/js@^9.39.2 @tailwindcss/cli @types/compression @types/connect-livereload @types/cors @types/ejs @types/express @types/hpp @types/jsonwebtoken @types/livereload @types/multer @types/node @types/nodemailer @types/swagger-jsdoc @types/swagger-ui-express concurrently connect-livereload eslint@^9.39.2 husky livereload nodemon prettier tailwindcss tsx typescript-eslint@^8.55.0
```

---

## 4. TypeScript Configuration (tsconfig.json) 🛡️

Run `tsc --init` then replace the content with this **S-Tier configuration**:

```json
{
  "compilerOptions": {
    // File Layout
    "rootDir": "./src",
    "outDir": "./dist",

    // Environment Settings
    "module": "nodenext",
    "target": "esnext",
    "types": ["node"],

    // Other Outputs
    "sourceMap": true,
    "declaration": true,
    "declarationMap": true,

    // Stricter Typechecking Options
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,

    // Style Options
    "noImplicitReturns": true,
    "noImplicitOverride": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    // Recommended Options
    "strict": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "noUncheckedSideEffectImports": true,
    "moduleDetection": "force",
    "skipLibCheck": true
  }
}
```

---

## 5. The Complete package.json Configuration 🚀

This setup includes all necessary metadata and automated scripts:

```json
{
  "name": "ts-bookstore-api-v2",
  "version": "2.0.0",
  "description": "A high-performance Bookstore API built with Node.js, TypeScript, and Docker, featuring Clean Architecture and Full CI/CD Pipeline.",
  "main": "dist/index.js",
  "scripts": {
    // --- Core Execution & Build ---
    "start": "node dist/index.js",
    "build": "npm run tailwind:build && tsc",
    "prepare": "husky", // Automatically setup Git Hooks after install

    // --- Integrated Development Workflows ---
    "dev": "concurrently \"npm run server\" \"npm run check-types:watch\"",
    "mvc": "concurrently \"npm run server\" \"npm run check-types:watch\" \"npm run tailwind:dev\"",

    // --- Development Sub-Services ---
    "server": "nodemon --exec tsx src/index.ts",
    "check-types": "tsc --noEmit", // One-time type check
    "check-types:watch": "tsc --noEmit --watch", // Real-time type monitoring
    "tailwind:dev": "npx @tailwindcss/cli -i ./src/styles/input.css -o ./public/styles/output.css --watch",
    "tailwind:build": "npx @tailwindcss/cli -i ./src/styles/input.css -o ./public/styles/output.css",

    // --- Quality Control & Utilities ---
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "tree": "tree -I 'node_modules|dist'",
    "gen-secret": "node -e \"console.log(require('crypto').randomBytes(32).toString('hex'))\"", // Generate 32-byte JWT secret

    // --- Infrastructure Management ---
    "mongo:start": "brew services start mongodb-community",
    "mongo:stop": "brew services stop mongodb-community",

    // --- Data Management (Seeders) ---
    "seed:users": "tsx src/modules/user/user.seeder.ts -seed",
    "delete:users": "tsx src/modules/user/user.seeder.ts -delete",
    "seed:authors": "tsx src/modules/author/author.seeder.ts -seed",
    "delete:authors": "tsx src/modules/author/author.seeder.ts -delete",
    "seed:books": "tsx src/modules/book/book.seeder.ts -seed",
    "delete:books": "tsx src/modules/book/book.seeder.ts -delete"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/HossamGezo/ts-bookstore-api-v2.git"
  },
  "keywords": [
    "nodejs",
    "typescript",
    "express",
    "mongodb",
    "docker",
    "clean-architecture",
    "rest-api",
    "swagger",
    "ci-cd"
  ],
  "author": "Hossam Gezo <ha2ghossam10@gmail.com> (https://github.com/HossamGezo)",
  "license": "MIT",
  "type": "module",
  "bugs": {
    "url": "https://github.com/HossamGezo/ts-bookstore-api-v2/issues"
  },
  "homepage": "https://github.com/HossamGezo/ts-bookstore-api-v2#readme",
  "devDependencies": {},
  "dependencies": {}
}
```

---

## 6. ESLint Configuration (eslint.config.mjs) 💎

Create `eslint.config.mjs` to maintain high code standards:

```javascript
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";

export default [
  // Use standard recommended rules for general JavaScript
  eslint.configs.recommended,

  // Use recommended rules for TypeScript specific code
  ...tseslint.configs.recommended,

  {
    // Apply these rules only to TypeScript files
    files: ["**/*.ts"],
    rules: {
      // Show a warning if 'any' type is used (better to use specific types)
      "@typescript-eslint/no-explicit-any": "warn",

      // Show an error if a variable is created but not used
      "@typescript-eslint/no-unused-vars": [
        "error",
        {
          // Ignore variables or arguments starting with an underscore (e.g., _req)
          argsIgnorePattern: "^_",
          varsIgnorePattern: "^_",
        },
      ],

      // Allow using console.log/error without warnings
      "no-console": "off",
    },
  },

  {
    // Tell ESLint to skip these folders (don't check them)
    ignores: ["dist/", "node_modules/", "public/"],
  },
];
```

---

## 7. Automated Pre-commit Guard (Husky) 🐶

1. Run `npm run prepare`.
2. Inside `.husky/pre-commit`, add:

```bash
# Ensure formatting, linting and type safety before every commit
npm run format && npm run lint && npm run check-types
```

---

## 8. Environment Orchestration (.env & .env.example) 🔐

Setup your environment templates:

```env
# APPLICATION CONFIGURATION
PORT=5001
NODE_ENV=development
BASE_URL=http://localhost:5001

# DATABASE CONFIGURATION
MONGO_URI_DEV=mongodb://localhost/bookStoreDB
MONGO_URI_PRO=

# SECURITY & AUTHENTICATION
JWT_SECRET_KEY=
JWT_EXPIRES_IN=30d
PASSWORD_RESET_EXPIRES_IN=10m

# FRONTEND INTEGRATION (CORS)
CLIENT_URL=http://localhost:3000

# PAGINATION SETTINGS (Business Logic)
BOOKS_PER_PAGE=2
AUTHORS_PER_PAGE=2
USERS_PER_PAGE=5

# NODEMAILER CONFIGURATION (Gmail/SMTP)
USER_EMAIL=
USER_PASS=
```

---

## 9. Modular Architecture & Complete Project Skeleton 📂

To set up the full professional foundation instantly, run the following command in your terminal. This will create the entire directory tree and all core configuration files, including folders for assets and CI/CD workflows:

### 🚀 One-Command Full Setup:
```bash
mkdir -p src/shared/{config,helpers,middlewares,types,validations} src/models src/modules src/styles views/home public/images assets .github/workflows && touch src/index.ts src/styles/input.css src/shared/config/{db.ts,swagger.ts} src/shared/helpers/{livereload.helper.ts,mongoose.helper.ts,query.helper.ts,response.helper.ts} src/shared/middlewares/{errors.middleware.ts,logger.middleware.ts,upload.middleware.ts,validateObjectId.middleware.ts,verifyToken.middleware.ts} src/shared/types/express.d.ts src/shared/validations/query.validation.ts views/home/welcome.ejs .env .env.example Dockerfile docker-compose.yml .github/workflows/deploy.yml
```

---

### 🌳 Complete Initial Structure:
After running the command, your project skeleton will look like this:

```text
.
├── .github/
│   └── workflows/
│       └── deploy.yml      # CI/CD Pipeline configuration
├── assets/                 # Screenshots and documentation media
├── src/
│   ├── shared/             # Global core (The Heart of the System)
│   │   ├── config/         # db.ts, swagger.ts
│   │   ├── helpers/        # livereload, mongoose, query, response helpers
│   │   ├── middlewares/    # logger, errors, upload, auth & validation guards
│   │   ├── types/          # express.d.ts (Global type extensions)
│   │   └── validations/    # query.validation.ts
│   │
│   ├── models/             # centralized Mongoose Models
│   ├── modules/            # Feature-based modules (Auth, User, etc.)
│   ├── styles/             # input.css (Tailwind source)
│   └── index.ts            # Main application entry point
│
├── views/                  # EJS templates (MVC Layer)
│   └── home/               # welcome.ejs
│
├── public/                 # Static assets directory
│   └── images/             # Directory for uploaded images (Persistent)
│
├── .env                    # Local secrets (ignored)
├── .env.example            # Environment template
├── Dockerfile              # Container recipe
└── docker-compose.yml      # Service orchestration
```

---

### 💡 Why this complete setup?
*   **Documentation Ready:** The `assets` folder allows you to store diagrams and UI screenshots for your README from day one.
*   **Automation First:** Including the `.github/workflows` directory ensures that your CI/CD mindset is established before you even write your first route.
*   **Total Organization:** By creating the `shared` sub-folders immediately, you force yourself to write clean, reusable code instead of cluttering the root.

---