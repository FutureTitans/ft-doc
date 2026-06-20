# Technical Manual


> Production domain: **youngpreneurs.ai**

---

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [Development Setup](#2-development-setup)
3. [Backend — Express.js Monolith](#3-backend--expressjs-monolith)
4. [Frontend — Next.js 14](#4-frontend--nextjs-14)
5. [Microservices](#5-microservices)
6. [Database — MongoDB Models](#6-database--mongodb-models)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [AI Engine — SURGE Framework & SSI Scoring](#8-ai-engine--surge-framework--ssi-scoring)
9. [Payments — Razorpay Integration](#9-payments--razorpay-integration)
10. [Email Service](#10-email-service)
11. [File Uploads — Vercel Blob](#11-file-uploads--vercel-blob)
12. [Call Service — ElevenLabs + Plivo](#12-call-service--elevenlabs--plivo)
13. [Caching & Rate Limiting](#13-caching--rate-limiting)
14. [Deployment — Vercel](#14-deployment--vercel)
15. [Testing](#15-testing)
16. [API Reference — All Endpoints](#16-api-reference--all-endpoints)

---

## 1. System Architecture

### High-Level Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           CLIENTS                                        │
│  Next.js Frontend (port 3000)         Flutter Mobile App                 │
└────────────────┬─────────────────────────────────────────────────────────┘
                 │  4 separate Axios instances
                 │
    ┌────────────┼──────────────┬────────────────┬─────────────────┐
    ▼            ▼              ▼                ▼                 ▼
┌────────┐ ┌──────────┐ ┌────────────┐ ┌────────────┐ ┌────────────────┐
│Backend │ │Auth Svc   │ │Core Svc    │ │Chat Svc    │ │Call Svc        │
│:5006   │ │:3002      │ │:3003       │ │:5003       │ │:3004           │
│        │ │           │ │            │ │            │ │                │
│Express │ │Express    │ │Express     │ │Express     │ │Express + WS    │
│Monolith│ │Auth only  │ │Modules,    │ │AI Chat,    │ │Plivo + 11Labs  │
│        │ │           │ │Admin,      │ │SSI Scoring │ │Voice AI        │
│All     │ │Login,     │ │Achievements│ │            │ │                │
│routes  │ │Signup,    │ │Associations│ │Gemini API  │ │WebSocket       │
│        │ │Profile    │ │Blogs,      │ │            │ │streaming       │
│        │ │           │ │School POC  │ │            │ │                │
└───┬────┘ └─────┬─────┘ └──────┬─────┘ └──────┬─────┘ └───────┬────────┘
    │            │              │               │               │
    └────────────┴──────────────┴───────────────┴───────────────┘
                              │
                    ┌─────────▼──────────┐
                    │    MongoDB Atlas    │
                    │    (shared DB)      │
                    └────────────────────┘
```

### Service Responsibilities

| Service | Port | Purpose |
|---------|------|---------|
| `backend/` | 5006 | Primary monolith — contains ALL routes. Handles payments, submissions, newsletters, and duplicates all microservice routes |
| `auth-service/` | 3002 | Login, signup, token refresh, profile, password reset |
| `core-service/` | 3003 | Modules, admin, achievements, associations, school POC, blogs |
| `chat-service/` | 5003 | AI chat conversations, SSI scoring, global SURGE chat |
| `call-service/` | 3004 | AI phone calls via Plivo + ElevenLabs, WebSocket audio streaming |

The backend monolith **duplicates all microservice routes**. The microservices exist for independent scaling. When changing a controller or model, always check if the same file exists in the relevant microservice and update both.

### Module System

All services use **ES6 modules** (`"type": "module"` in their package.json) — use `import`/`export`, never `require`. The root `package.json` is CommonJS (`"type": "commonjs"`) for tooling scripts.

---

## 2. Development Setup

### Prerequisites

- Node.js 18+
- MongoDB (local or Atlas URI)
- npm

### Environment Variables

Each service requires its own `.env` file.

**Backend `.env`:**
```
MONGODB_URI=mongodb+srv://...
NODE_ENV=development
PORT=5006
JWT_SECRET=<secret>
JWT_REFRESH_SECRET=<secret>
JWT_EXPIRE=15m
JWT_REFRESH_EXPIRE=7d
RAZORPAY_KEY_ID=<key>
RAZORPAY_KEY_SECRET=<secret>
GEMINI_API_KEY=<key>
FRONTEND_URL=http://localhost:3000
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_USER=<email>
SMTP_PASS=<app-password>
SMTP_FROM=<email>
SMTP_FROM_NAME=Future Titans
```

**Frontend `.env.local`:**
```
NEXT_PUBLIC_API_URL=http://localhost:5006/api
NEXT_PUBLIC_AUTH_API_URL=http://localhost:3002/api
NEXT_PUBLIC_CORE_API_URL=http://localhost:3003/api
NEXT_PUBLIC_CHAT_API_URL=http://localhost:5003/api
```

**Call Service `.env`:**
```
MONGODB_URI=mongodb+srv://...
ELEVENLABS_AGENT_ID=<agent-id>
ELEVENLABS_API_KEY=<key>
PORT=3004
```

### Running Services

```bash
# Backend (monolith)
cd backend && npm install && npm run dev     # nodemon, port 5006

# Frontend
cd frontend && npm install && npm run dev    # Next.js, port 3000

# Microservices
cd auth-service && npm install && npm run dev    # nodemon, port 3002
cd core-service && npm install && npm run dev    # nodemon, port 3003
cd chat-service && npm install && npm run dev    # nodemon, port 5003
cd call-service && npm install && npm run dev    # node --watch, port 3004

# Utility scripts
cd backend && npm run seed:users              # Seed test user data
cd backend && npm run migrate:chapters        # Migrate chapter AI prompts
cd backend && npm run load:test               # Load testing
```

### Building for Production

```bash
cd frontend && npm run build    # Next.js production build
cd frontend && npm run lint     # ESLint
```

---

## 3. Backend — Express.js Monolith

### Directory Structure

```
backend/src/
├── config/
│   └── database.js          # MongoDB connection (Mongoose)
├── controllers/
│   ├── achievementController.js
│   ├── adminController.js
│   ├── aiChatController.js
│   ├── associationController.js
│   ├── authController.js
│   ├── blogController.js
│   ├── moduleController.js
│   ├── newsletterController.js
│   ├── outreachController.js
│   ├── paymentController.js
│   ├── schoolPocController.js
│   └── submissionController.js
├── middleware/
│   ├── auth.js               # JWT verification + role guards
│   ├── errorHandler.js       # Global error handler + asyncHandler
│   ├── multer.js             # File upload (in-memory storage)
│   └── rateLimiter.js        # Rate limiting (edge-deferred, passthrough)
├── models/                   # Mongoose schemas (14 models)
├── routes/                   # Express route definitions (11 files)
├── scripts/
│   ├── seed-test-users.js
│   ├── migrate-chapters-aiPrompt.js
│   └── load-test.js
├── services/
│   ├── aiEngine.js           # Gemini AI + SURGE framework
│   ├── emailService.js       # Nodemailer SMTP
│   └── paymentService.js     # Razorpay integration
├── utils/
│   ├── cache.js              # In-memory TTL cache
│   └── chatRateLimit.js      # Per-student chat rate limiting
└── index.js                  # Entry point
```

### Request Flow

```
Client Request
  → helmet() (security headers)
  → cors() (origin allowlist)
  → express.json()
  → apiLimiter (rate limiting — currently edge-deferred, passthrough)
  → ensureDB middleware (awaits MongoDB connection)
  → Route handler
  → Controller
  → Service (business logic, Gemini API, Razorpay)
  → Model (Mongoose)
  → errorHandler (catches all errors)
```

### Middleware Stack (applied in order in `index.js`)

1. **`helmet()`** — Sets security headers (CSP, X-Frame-Options, etc.)
2. **`cors()`** — Origin allowlist:
   - `http://localhost:3000`
   - `https://youngpreneurs.ai` and subdomains (`*.youngpreneurs.ai`)
   - `*.vercel.app` (preview deployments)
   - `FRONTEND_URL` env var
   - Non-browser requests (no `Origin` header) are always allowed
3. **`express.json()`** — Parse JSON request bodies
4. **`apiLimiter`** — Rate limiting (currently passthrough; handled at edge layer)
5. **`ensureDB`** — Applied to `/api/*` routes; awaits MongoDB connection before handling requests

### Error Handler (`middleware/errorHandler.js`)

Global Express error middleware. Handles:

| Error Type | Status | Response |
|-----------|--------|----------|
| Mongoose `ValidationError` | 400 | Human-readable field-level messages |
| Mongoose `CastError` | 400 | `"Invalid ID format"` |
| `MongoServerError` code 11000 (duplicate key) | 409 | `"<field> already exists"` |
| All other errors | `err.status` or 500 | Full message in dev, `"Internal server error"` in production |

Also exports `asyncHandler(fn)` — wraps async route handlers to forward errors to the error middleware.

### Database Configuration (`config/database.js`)

Mongoose connection with these pool settings:

| Setting | Value |
|---------|-------|
| `maxPoolSize` | 50 |
| `minPoolSize` | 5 |
| `maxIdleTimeMS` | 30,000ms |
| `serverSelectionTimeoutMS` | 5,000ms |
| `socketTimeoutMS` | 45,000ms |
| `readPreference` | `secondaryPreferred` |
| `w` | `majority` |
| `autoIndex` | `false` in production |

Uses a **singleton promise pattern** — `connectDB()` caches the connection promise and returns it on subsequent calls. The `ensureDB` middleware guarantees the connection is ready before any API handler runs. On disconnect, the promise is nullified so the next request triggers a fresh connection.

### Swagger API Docs

Auto-generated from JSDoc annotations in `routes/*.js`. Served at `/api-docs` (only accessible on the backend monolith).

---

## 4. Frontend — Next.js 14

### Tech Stack

| Library | Purpose |
|---------|---------|
| Next.js 14 | App Router, SSR/SSG |
| Tailwind CSS 3.3 | Styling |
| Radix UI | Primitives (Dialog, Dropdown, Progress, Tabs) |
| Zustand | State management |
| Axios | HTTP client (4 instances) |
| Framer Motion | Animations |
| Recharts | Charts and data visualization |
| face-api.js | Biometric face verification |
| react-quill | Rich text editor (admin blog editor) |
| react-markdown + markdown-it | Markdown rendering |
| js-cookie | Cookie management |
| jsPDF + jspdf-autotable | PDF generation |
| Lucide React | Icons |
| @vercel/analytics | Analytics |

### Route Structure

```
frontend/app/
├── layout.js                     # Root layout
├── page.js                       # Landing page (public)
├── globals.css                   # Global styles
├── robots.js                     # SEO robots.txt
├── sitemap.js                    # SEO sitemap
│
├── login/page.js                 # Student/Admin login
├── signup/page.js                # Student registration
├── forgot-password/page.js       # Password reset request
├── reset-password/page.js        # Password reset form
│
├── student/
│   ├── dashboard/page.js         # Student dashboard
│   ├── modules/page.js           # Module listing + chapter views
│   ├── submission/page.js        # Idea submission form
│   └── profile/page.js           # Student profile
│
├── admin/
│   ├── layout.js                 # Admin layout with sidebar
│   ├── page.js                   # Admin dashboard
│   ├── students/page.js          # Student CRM
│   ├── modules/page.js           # Module builder
│   ├── submissions/page.js       # Submission review
│   ├── analytics/page.js         # Analytics dashboard
│   ├── settings/page.js          # Platform settings (SMTP, chat limits)
│   ├── schools/page.js           # School slug management
│   ├── associations/page.js      # Association management
│   ├── blogs/page.js             # Blog editor
│   └── grant-simulation/page.js  # Grant simulation tool
│
├── association/
│   ├── login/page.js             # Association login
│   └── dashboard/page.js         # Association dashboard + outreach
│
├── school-poc/
│   ├── layout.js                 # School POC layout
│   ├── login/page.js             # School POC login
│   └── dashboard/page.js         # School POC student overview
│
├── blog/
│   ├── page.js                   # Blog listing
│   └── [slug]/page.js            # Blog detail
│
├── outreach/[slug]/page.js       # Public outreach form
├── demo-access/ft-demo-2026x/page.js  # Demo access page
│
├── about-us/page.js              # Marketing pages
├── academy/page.js               #   ↓
├── contact/page.js               #   ↓
├── for-parents/page.js           #   ↓
├── for-schools/page.js           #   ↓
├── future-titans/page.js         #   ↓
├── media/page.js                 #   ↓
├── success-stories/page.js       #   ↓
├── team/page.js                  #   ↓
│
└── api/upload/                   # Next.js API route for file uploads
```

### Components

```
frontend/components/
├── shared/
│   ├── Navbar.js                 # Authenticated navigation bar
│   ├── PublicNavbar.js           # Public marketing navbar
│   ├── PublicFooter.js           # Public footer
│   ├── ErrorBoundary.js          # React error boundary
│   ├── LoadingSpinner.js         # Loading indicator
│   ├── FreezeOverlay.js          # Account freeze overlay
│   ├── RichTextEditor.js         # react-quill wrapper
│   ├── FaceGuardWrapper.js       # Face verification gate for protected pages
│   ├── FaceLoginVerification.js  # Face check during login
│   ├── FaceRegistration.js       # Initial face enrollment
│   ├── FaceVerificationProvider.js # Context provider for face auth state
│   └── ModuleFaceGuard.js        # Per-module face verification
│
├── student/
│   ├── AIChat.js                 # Chapter-specific AI chat component
│   ├── GlobalAIChat.js           # Module-agnostic SURGE chat
│   ├── ChapterQuiz.js            # Chapter quiz component
│   └── ZunnovaAvatar.js          # AI persona avatar
│
└── home/                         # Landing page sections
```

### API Layer (`lib/api.js`)

Four Axios instances connect to four different backends:

| Instance | Base URL | Timeout | Target Service |
|----------|----------|---------|----------------|
| `api` | `NEXT_PUBLIC_API_URL` (default: `localhost:5006/api`) | 15s | Backend monolith |
| `authApi` | `NEXT_PUBLIC_AUTH_API_URL` (default: `localhost:3002/api`) | 15s | Auth service |
| `coreApi` | `NEXT_PUBLIC_CORE_API_URL` (default: `localhost:3003/api`) | 15s | Core service |
| `chatApi` | `NEXT_PUBLIC_CHAT_API_URL` (default: `localhost:5003/api`) | 30s | Chat service |

**Request interceptors**: All four instances attach the JWT Bearer token from `getAuthToken()`.

**Response interceptors**: All four instances unwrap `response.data` on success. On 403 errors, they attempt a token refresh via the auth endpoint, then retry the original request. On refresh failure, they clear auth state and redirect to the portal-specific login page based on the current URL path (`/association/*` → `/association/login`, `/admin/*` → `/admin/login`, else → `/login`).

**Exported API namespaces:**

| Namespace | Service Instance | Functions |
|-----------|-----------------|-----------|
| `auth` | `authApi` | `signup`, `login`, `logout`, `refreshToken`, `getProfile`, `checkCompletionStatus`, `updateProfile`, `forgotPassword`, `resetPassword`, `saveFaceDescriptor`, `getFaceDescriptor` |
| `payment` | `api` | `initiatePayment`, `verifyPayment`, `getPaymentStatus` |
| `modules` | `coreApi` | `getAll`, `getById`, `getChapter`, `trackTime`, `create`, `update`, `delete`, `publish`, `createChapter`, `updateChapter`, `deleteChapter`, `reorderChapters` |
| `aiChat` | `chatApi` | `sendMessage`, `getChatHistory`, `finishChapterChat`, `completeChapter`, `getSSI`, `getReport`, `getStudentChats`, `overrideSSI`, `sendGlobalMessage`, `getGlobalHistory`, `getRateLimitStatus` |
| `submission` | `api` | `saveDraft`, `submit`, `get`, `update`, `getAll`, `getDetails`, `score`, `downloadFile`, `reject` |
| `admin` | `coreApi` | `getDashboard`, `getStudents`, `getStudentDetail`, `updateStudentStatus`, `tagStudent`, `deleteStudent`, `getAnalytics`, `exportStudents`, `getSchoolSlugs`, `createSchoolSlug`, `updateSchoolSlug`, `deleteSchoolSlug`, `getStudentsBySlug`, `reviewSchoolProof`, `getSMTPConfig`, `updateSMTPConfig`, `testSMTPConfig`, `getChatSettings`, `updateChatSettings` |
| `achievements` | `coreApi` | `getAll` |
| `schoolPoc` | `coreApi` | `login`, `getDashboard` |

### Auth Token Storage (`lib/auth.js`)

**Dual storage strategy** — tokens are stored in both `js-cookie` and `localStorage` for redundancy:

| Key | Cookie Expiry | Storage |
|-----|---------------|---------|
| `future_titans_token` | 0.625 days (~15 min) | Cookie + localStorage |
| `future_titans_refresh_token` | 7 days | Cookie + localStorage |
| `future_titans_user` | N/A | localStorage only (JSON) |

Read functions prefer cookie, fall back to localStorage. On logout, both are cleared, plus face verification session data in `sessionStorage`.

**Utility functions:**
- `getAuthToken()` / `setAuthToken(token)` — JWT access token
- `getRefreshToken()` / `setRefreshToken(token)` — JWT refresh token
- `getUser()` / `setUser(user)` — User object
- `removeAuthToken()` — Clears all auth data
- `isAuthenticated()` / `isAdmin()` / `isStudent()` — Role checks

### State Management — Zustand (`store/useAuthStore.js`)

```javascript
{
  user: null,            // User object or null
  isLoading: false,
  error: null,
  faceVerified: false,   // Face verification state
  isFrozen: false,       // Account freeze state

  // Methods:
  hydrateUser()          // Load user from storage (call in useEffect only!)
  setUser(user)          // Update user in store + persist
  setTokens(access, refresh)
  setFaceVerified(bool)
  setFrozen(bool)
  logout()               // Clear everything, call backend logout
}
```

**SSR safety**: `hydrateUser()` must be called inside `useEffect` to avoid SSR/client hydration mismatches. Never read auth state during server render.

### Design System

**Apple-inspired aesthetic with glassmorphism. No emojis in UI text.**

#### Color Palette (Tailwind classes)

| Token | Hex | Usage |
|-------|-----|-------|
| `cream` | `#FAF8F3` | Page backgrounds |
| `cream-dark` | `#F0E8D4` | Subtle background |
| `gold` | `#D4AF37` | Primary accent, CTAs |
| `gold-dark` | `#B8952E` | Hover states |
| `gold-light` | `#F5D76E` | Highlights |
| `primary-red` | `#DC2626` | Brand primary |
| `primary-dark-red` | `#991B1B` | Contrast/hover |
| `primary-light-red` | `#FEE2E2` | Light backgrounds |
| `accent-gold` | `#D97706` | Secondary accent |
| `neutral-dark` | `#1A1A1A` | Text primary |
| `neutral-medium` | `#6B6B6B` | Text secondary |
| `semantic-success` | `#10B981` | Success states |
| `semantic-warning` | `#F59E0B` | Warning states |
| `semantic-error` | `#EF4444` | Error states |
| `semantic-info` | `#3B82F6` | Info states |

#### Custom Shadows

| Class | Description |
|-------|-------------|
| `shadow-glass` | Standard glassmorphism |
| `shadow-glass-lg` | Large glassmorphism |
| `shadow-gold` | Gold glow effect |
| `shadow-gold-lg` | Large gold glow |

#### Custom Gradients

| Class | Description |
|-------|-------------|
| `bg-gold-gradient` | Gold 135deg gradient |
| `bg-cream-gradient` | Radial gold/cream background |

#### Custom Fonts

| Class | Font |
|-------|------|
| `font-sans` | Inter + system stack |
| `font-edo` | Edo |
| `font-roca` | roca-two (serif) |
| `font-heading-now` | Heading Now |

#### Custom Animations

| Class | Effect |
|-------|--------|
| `animate-fade-in` | Fade in + translate up |
| `animate-slide-up` | Slide up from 20px |
| `animate-slide-down` | Slide down from 20px |
| `animate-scale-in` | Scale from 0.95 |
| `animate-float` | Infinite float up/down |
| `animate-glow` | Infinite gold glow pulse |

#### Design Token File (`styles/design-system.js`)

Exports JS objects for use in inline styles or dynamic theming:
- `colors` — Primary, accent, neutral, semantic, gradients
- `typography` — Font families, sizes, weights, letter spacing
- `spacing` — xs (4px) through 4xl (64px)
- `borderRadius` — sm (4px) through full (9999px)
- `shadows` — sm through 2xl
- `transitions` — fast (0.15s), base (0.3s), slow (0.5s)
- `breakpoints` — xs (320px) through 2xl (1536px)
- `zIndex` — dropdown (1000) through tooltip (1060)
- `componentStyles` — Button (primary/secondary/ghost), card, input presets

### Face Verification System

Uses `face-api.js` for biometric identity verification:

1. **Registration** (`FaceRegistration.js`): Student enrolls face via webcam. Face descriptor (128-float array) is extracted and stored on the User model via `auth.saveFaceDescriptor()`.

2. **Verification** (`FaceGuardWrapper.js`, `ModuleFaceGuard.js`): Before accessing protected content, the student's live face is compared against their stored descriptor. Uses Euclidean distance threshold.

3. **Provider** (`FaceVerificationProvider.js`): Context provider that wraps protected pages. Manages verification state in `sessionStorage` (`ft_face_verified`, `ft_face_descriptor`, `ft_face_threshold`).

4. **Login check** (`FaceLoginVerification.js`): Face verification during login flow.

### Next.js Configuration (`next.config.js`)

**Security headers** (all routes):
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Permissions-Policy: camera=(self), microphone=()`

**Remote image patterns**: `*.vercel-storage.com`, `*.blob.vercel-storage.com`, `future-titans-backend-gamma.vercel.app`, `localhost`

**Webpack**: Client bundle disables `fs`, `path`, `crypto` fallbacks.

---

## 5. Microservices

### Architecture Pattern

Each microservice (`auth-service`, `core-service`, `chat-service`) mirrors the backend's directory structure:

```
<service>/src/
├── config/database.js
├── controllers/
├── middleware/
├── models/
├── routes/
├── services/
├── utils/
└── index.js
```

They share the same MongoDB database. Models are duplicated (not shared as a package). When changing a model or controller, update both the backend monolith and the relevant microservice.

### Route Ownership

| Service | Routes Owned | Frontend API Instance |
|---------|-------------|----------------------|
| `auth-service` | `/api/auth/*` — login, signup, refresh, logout, profile, password reset, face descriptor | `authApi` |
| `core-service` | `/api/modules/*`, `/api/admin/*`, `/api/achievements/*`, `/api/association/*`, `/api/school-poc/*`, `/api/blogs/*` | `coreApi` |
| `chat-service` | `/api/chat/*` — AI conversations, SSI scoring, global SURGE chat, rate limit status | `chatApi` |
| `call-service` | `/api/plivo/*` — incoming call webhook; `/stream` — WebSocket audio | N/A (phone calls only) |
| `backend` | All of the above + `/api/payment/*`, `/api/submission/*`, `/api/newsletter/*` | `api` |

**Outreach routes** are nested under `/api/association/outreach/*`, not a separate namespace. The public outreach form submission (`POST /api/association/outreach/submit/:slug`) requires no authentication.

### Vercel Configuration

Each service has a `vercel.json`:

```json
{
  "functions": { "src/index.js": { "maxDuration": 60, "memory": 512 } },
  "routes": [
    { "src": "/api/(.*)", "dest": "src/index.js" },
    { "src": "/health", "dest": "src/index.js" }
  ],
  "crons": [{ "path": "/health", "schedule": "*/5 * * * *" }]
}
```

The cron job hits `/health` every 5 minutes to keep the Lambda warm (~5-15 minute warm window on Vercel).

---

## 6. Database — MongoDB Models

### User

The central model. Supports 4 roles: `student`, `admin`, `association`, `school_poc`.

| Field | Type | Details |
|-------|------|---------|
| `name` | String | Required |
| `studentId` | String | Unique, sparse |
| `email` | String | Required, unique, lowercase |
| `phone` | String | Required |
| `school` | String | Required |
| `schoolSlug` | String | Optional school cohort |
| `class` | String | Default: `'Unassigned'` |
| `section` | String | Default: `''` |
| `rollNumber` | String | Default: `''` |
| `city` | String | Required |
| `country` | String | Required |
| `password` | String | Required, auto-hashed with bcrypt (salt 10) |
| `isOnline` | Boolean | Live tracking |
| `role` | String (enum) | `student` / `admin` / `school_poc` / `association` |
| `isPaid` | Boolean | Payment status |
| `razorpayOrderId` | String | Razorpay order ID |
| `razorpayPaymentId` | String | Razorpay payment ID |
| `paymentDate` | Date | When payment was made |
| `ssiScore` | Number | Overall SSI score (0-100) |
| `ssiBreakdown` | Object | `{ selfAwareness, understandingOpportunities, resilience, growthExecution, entrepreneurialLeadership }` — each 0-100 |
| `associationId` | ObjectId → User | For school_poc: which association they belong to |
| `commissionPercentage` | Number | For association: revenue share percentage |
| `profilePicture` | String | URL to profile image |
| `bio` | String | Max 500 chars |
| `profileTheme` | String (enum) | `default`, `ocean`, `sunset`, `forest`, `cosmic`, `royal` |
| `modulesProgress[]` | Array | `{ moduleId, completedChapters[], completionPercentage, timeSpent (seconds), lastAccessedAt }` |
| `ideaSubmission` | ObjectId → IdeaSubmission | Reference to competition submission |
| `faceDescriptor` | [Number] | Float array for face verification |
| `lastLogin` | Date | |
| `resetPasswordToken` | String | Password reset token |
| `resetPasswordExpiry` | Date | Token expiry |

**Indexes**: `email` (unique), `role+isPaid`, `schoolSlug`, `school`, `city+country`, `ssiScore`, `createdAt` (descending)

**Hooks**: `pre('save')` — auto-hashes password if modified.

**Methods**: `comparePassword(enteredPassword)` — bcrypt comparison.

### Module

| Field | Type | Details |
|-------|------|---------|
| `title` | String | Required |
| `description` | String | Required |
| `difficulty` | String (enum) | `beginner`, `intermediate`, `advanced` |
| `coverImage` | String | URL |
| `estimatedCompletionTime` | Number | Minutes |
| `aiInteractionEnabled` | Boolean | Default: true |
| `aiQuestionsPerChapter` | Number | Default: 10 |
| `chapters[]` | [ObjectId → Chapter] | Ordered chapter references |
| `isPublished` | Boolean | Default: false |
| `createdBy` | ObjectId → User | Required |
| `mentorName` | String | Instructor name |
| `mentorProfilePicture` | String | Instructor avatar URL |

**Indexes**: `isPublished+createdAt`, `createdBy`

### Chapter

| Field | Type | Details |
|-------|------|---------|
| `moduleId` | ObjectId → Module | Required |
| `title` | String | Required |
| `description` | String | |
| `order` | Number | Required — sort order |
| `content.type` | String (enum) | `text`, `video`, `audio`, `pdf` |
| `content.text` | String | For text content |
| `content.videoUrl` | String | For video content |
| `content.audioUrl` | String | For audio content |
| `content.pdfUrl` | String | For PDF content |
| `subtitles` | String | Used as AI context |
| `aiInteractionEnabled` | Boolean | Default: true |
| `aiPrompt` | String | Required — custom prompt for this chapter's AI interactions |
| `surgeDimensionFocus` | String | Which SURGE dimension to emphasize |
| `maxMessagesPerConversation` | Number | Default: 10 |
| `isPublished` | Boolean | Default: false |

**Indexes**: `moduleId+order`, `moduleId`, `isPublished`

### AIChat

Stores per-chapter or global AI conversation sessions.

| Field | Type | Details |
|-------|------|---------|
| `studentId` | ObjectId → User | Required |
| `chapterId` | ObjectId → Chapter | Null for global chats |
| `moduleId` | ObjectId → Module | Null for global chats |
| `conversation[]` | Array | `{ role: 'user'/'assistant', message, timestamp }` |
| `ssiScore` | Object | `{ overall, selfAwareness, understandingOpportunities, resilience, growthExecution, entrepreneurialLeadership }` |
| `interactionMetrics` | Object | `{ creativeScore, clarityScore, resilienceIndicators, decisionMakingQuality, communicationQuality }` |
| `ssiHistory[]` | Array | Per-turn SSI snapshot: `{ turnNumber, scores{}, surgeStageScores{}, timestamp }` |
| `currentSurgeStage` | String (enum) | `spot`, `unbundle`, `reframe`, `generate`, `evaluate` |
| `gapAnalysis` | Object | `{ weakestDimension, weakestSurgeStage, suggestedMission, gapScores{} }` |
| `isCompleted` | Boolean | Default: false |
| `startedAt` | Date | |
| `completedAt` | Date | |
| `timeSpent` | Number | Seconds |

**Indexes**: `studentId+chapterId+isCompleted`, `studentId+moduleId`, `studentId+startedAt`, `chapterId`, `isCompleted`

### IdeaSubmission

Competition entry with multi-field form + file uploads.

| Field | Type | Details |
|-------|------|---------|
| `studentId` | ObjectId → User | Required |
| `teamMembers[]` | Array | `{ name, email, role }` |
| `projectTitle` | String | |
| `primaryCategory` | String | |
| `secondaryCategory` | String | |
| `elevatorPitch` | String | Max 1000 chars |
| `problemStatement` | String | Max 2000 chars |
| `existingSolutions` | String | |
| `solutionOverview` | String | Max 3000 chars |
| `prototypeDescription` | String | |
| `userTypes` | String | |
| `reachStrategy` | String | |
| `impactDescription` | String | |
| `businessModel` | String | |
| `biggestCosts[]` | [String] | |
| `teamSkills` | String | |
| `wantToLearn` | String | |
| `improvementPlan` | String | |
| `inspiration` | String | Max 500 chars |
| `previousWork` | String | |
| `pdfFile` | String | Vercel Blob URL |
| `videoFile` | String | Vercel Blob URL |
| `submissionStatus` | String (enum) | `draft`, `submitted`, `reviewed`, `shortlisted`, `rejected` |
| `adminNotes` | String | |
| `adminScore` | Number | |
| `submittedAt` | Date | |

**Indexes**: `studentId`, `submissionStatus+submittedAt`, `primaryCategory`, `teamMembers.email`, `projectTitle` (text)

### SchoolSlug

Represents a school portal with POC credentials and approval workflow.

| Field | Type | Details |
|-------|------|---------|
| `name` | String | Required |
| `slug` | String | Required, unique, lowercase |
| `description` | String | |
| `price` | Number | Required — INR amount |
| `requiresPayment` | Boolean | Default: true |
| `isActive` | Boolean | Default: true |
| `pocName` | String | Point of Contact name |
| `pocEmail` | String | POC login email |
| `pocPassword` | String | Auto-hashed with bcrypt |
| `associationId` | ObjectId → User | Which association created this |
| `lockUntil` | Date | Expiry for lock |
| `proofOfAcceptance` | String | Document URL |
| `approvalStatus` | String (enum) | `PendingProof`, `PendingApproval`, `Approved`, `Rejected` |
| `googlePlaceId` | String | Google Maps entity ID |
| `targetStudentCount` | Number | Expected registrations |

**Hooks**: `pre('save')` — auto-hashes `pocPassword` if modified.

**Methods**: `comparePocPassword(enteredPassword)` — bcrypt comparison.

### AppSettings (Singleton)

Platform-wide configuration stored in DB. Only one document (key: `app_settings`).

| Field | Type | Default | Details |
|-------|------|---------|---------|
| `chatMessageLimit` | Number | 200 | Max messages per student per window |
| `chatResetHours` | Number | 24 | Window duration in hours |
| `updatedBy` | ObjectId → User | | Last admin who changed settings |

**Static method**: `getSettings()` — Returns the singleton document, creates with defaults if missing.

### Other Models

| Model | Purpose |
|-------|---------|
| `AssociationCode` | Time-limited codes for school onboarding. Fields: `code` (unique), `schoolName`, `associationId`, `isUsed`, `usedBy`, `expiresAt` |
| `Achievement` | Gamification badges. 16 types (first_chapter, ssi_milestone_50, streak_7_days, etc.). Rarity: common/rare/epic/legendary. Indexed by `studentId+type` |
| `Blog` | CMS blog posts. Fields: `title`, `slug` (unique), `coverImage`, `content` (markdown/HTML), `author`, `tags[]`, `status` (draft/published) |
| `CompetitionForm` | Form template definitions. Fields: `questionNumber`, `questionTitle`, `type` (text/number/pdf_upload/video_upload), `wordLimit`, `isRequired`, `allowedFormats[]` |
| `OutreachForm` | Association outreach forms with email templates. One per association. Contains `responses[]` array with school contact submissions. Has `emailYesTemplate` and `emailNoTemplate` with subject, body, attachment |
| `NewsletterSubscriber` | Email subscribers. Fields: `email` (unique), `isActive`, `source` (default: 'footer') |
| `SMTPConfig` | Dynamic SMTP configuration. Single active config. Fields: `host`, `port`, `secure`, `auth.user`, `auth.pass`, `from`, `fromName`, `isActive`. Static method: `getActive()` |

---

## 7. Authentication & Authorization

### JWT Token Flow

```
1. Login → POST /api/auth/login
   ← { accessToken (15min), refreshToken (7d), user }

2. Every request → Authorization: Bearer <accessToken>

3. Token expired (403) → POST /api/auth/refresh { refreshToken }
   ← { accessToken (new) }

4. Refresh failed → Clear auth, redirect to login
```

### Middleware Guards

Five middleware functions in `backend/src/middleware/auth.js`:

| Guard | Behavior |
|-------|----------|
| `authenticateToken` | Verifies JWT Bearer token. Sets `req.user` from decoded payload. Returns 401 if missing, 403 if invalid/expired |
| `authorizeAdmin` | Checks `req.user.role === 'admin'`. Returns 403 if not |
| `authorizeStudent` | Checks `req.user.role === 'student'`. Returns 403 if not |
| `authorizeAssociation` | Checks `req.user.role === 'association'`. Returns 403 if not |
| `authenticateSchoolPOC` | Separate JWT decode path. Verifies token AND checks `role === 'school_poc'`. Sets `req.pocUser` (not `req.user`) |

### Usage Patterns

```javascript
// Public route (no auth)
router.get('/blogs', getBlogs);

// Authenticated (any role)
router.get('/profile', authenticateToken, getProfile);

// Student only
router.post('/draft', authenticateToken, authorizeStudent, saveDraft);

// Admin only
router.get('/dashboard', authenticateToken, authorizeAdmin, getDashboard);

// Association only
router.post('/school', authenticateToken, authorizeAssociation, createSchool);

// School POC (separate decode path)
router.get('/dashboard', authenticateSchoolPOC, getDashboard);
```

---

## 8. AI Engine — SURGE Framework & SSI Scoring

### Overview

The AI engine (`backend/src/services/aiEngine.js`) implements the proprietary **SURGE Framework** and **SSI (Solution-Seeking Index) Scoring Engine** using Google Gemini API (`gemini-2.5-flash`).

### SURGE Framework Stages

| Stage | Turns | Focus |
|-------|-------|-------|
| **S**pot | 1-2 | Identify and narrow down a real-world problem |
| **U**nbundle | 3-4 | Break down the problem analytically. Ask why, who, what has been tried |
| **R**eframe | 5-6 | Apply creative constraints to force radical rethinking |
| **G**enerate | 7-8 | Rapid prototyping, MVPs, action over perfection |
| **E**valuate | 9-10 | Test, learn, iterate. Reframe failures as learning opportunities |

Turn index maps to stages: `SURGE_STAGES_BY_TURN = ['spot', 'spot', 'unbundle', 'unbundle', 'reframe', 'reframe', 'generate', 'generate', 'evaluate', 'evaluate']`

### SSI Dimensions (5 dimensions, 0-100 each)

| Dimension | What It Measures |
|-----------|-----------------|
| Self-Awareness | Understanding problems, personal strengths, limitations, biases |
| Understanding Opportunities | Spotting ideas, market gaps, needs, potential value |
| Resilience | Handling failure, adapting, persisting, bouncing back |
| Growth Execution | Implementing solutions, taking action, building, experimenting |
| Entrepreneurial Leadership | Leading teams, strategic thinking, inspiring others, decision-making |

### AI Chat Flow

```
1. Student sends message
2. Add message to conversation history
3. Evaluate scores via separate Gemini call (AI judge)
4. Perform gap analysis (compare to ideal blueprint, find weakest area)
5. Build system prompt with:
   - Chapter-specific custom prompt (from Chapter.aiPrompt)
   - Chapter content/context
   - Student profile + gap analysis
   - Current SURGE stage + target SSI dimension
6. Send to Gemini with conversation history (last 20 messages)
7. Save AI response + scores to AIChat document
8. Fire-and-forget: update User's aggregate SSI from all their chats
```

### Gemini Concurrency Control

A semaphore-based queue limits concurrent Gemini API calls:
- `MAX_CONCURRENT_GEMINI`: default 5 (configurable via `GEMINI_MAX_CONCURRENT` env var)
- Requests exceeding the limit are queued and processed FIFO
- 30-second timeout per Gemini call with one automatic retry

### Score Caching

In-memory TTL cache for scoring evaluations:
- **TTL**: 5 minutes
- **Max size**: 200 entries (LRU eviction)
- **Cache key**: Based on last 3 messages + turn number

### Gap Analysis Engine

Compares student scores against ideal benchmarks (80 for all dimensions/stages):

```
1. Calculate gap scores = ideal - actual for each SSI dimension
2. Find weakest SSI dimension (largest gap)
3. Find weakest SURGE stage (largest gap)
4. Generate suggested mission based on weaknesses:
   - Reframe/Understanding weak → "Force radical rethinking"
   - Generate/Growth weak → "Push for rapid prototyping"
   - Evaluate/Resilience weak → "Challenge with failure scenario"
   - Unbundle weak → "Break problem into 3 root causes"
```

### Fallback Scoring

If Gemini scoring fails (API error, JSON parse failure), a heuristic provides progressive scores:
- Base progress = `min(60, turnNumber * 5)`
- Each dimension gets `previous + 2`, clamped to 100
- Ensures students see forward progress even during API issues

### Student Report Generation

`generateStudentReport(studentId)` aggregates SSI scores across all of a student's AIChat sessions, averaging each dimension.

### User SSI Update

`updateUserSSIFromChats(studentId)` recomputes the student's aggregate SSI on the User model from all their chats. Called fire-and-forget after each chat message.

---

## 9. Payments — Razorpay Integration

### Service (`services/paymentService.js`)

| Function | Description |
|----------|-------------|
| `createOrder(amount, studentId, studentEmail)` | Creates a Razorpay order. Amount in INR (converted to paise internally: `amount * 100`). Receipt is `student_<id>_<timestamp>` truncated to 40 chars. Currency: INR |
| `verifyPayment(orderId, paymentId, signature)` | HMAC-SHA256 signature verification: `sha256(orderId + '\|' + paymentId, RAZORPAY_KEY_SECRET)`. Returns boolean |
| `refundPayment(paymentId, amount)` | Issues partial/full refund via Razorpay API. Amount in INR (converted to paise) |

### API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/payment/initiate` | JWT (any) | Create Razorpay order for the authenticated user |
| POST | `/api/payment/verify` | None | Webhook — verify payment signature and mark user as paid |
| GET | `/api/payment/status` | JWT (any) | Check current user's payment status |

### Flow

```
1. Frontend calls payment.initiatePayment()
2. Backend creates Razorpay order → returns order_id
3. Frontend opens Razorpay checkout with order_id
4. On success, frontend calls payment.verifyPayment({ orderId, paymentId, signature })
5. Backend verifies HMAC signature
6. Backend updates User: isPaid=true, razorpayOrderId, razorpayPaymentId, paymentDate
```

---

## 10. Email Service

### Configuration Priority

1. **Environment variables** (checked first): `SMTP_HOST`, `SMTP_USER`, `SMTP_PASS`, `SMTP_PORT`, `SMTP_FROM`, `SMTP_FROM_NAME`
2. **Database** (fallback): `SMTPConfig` model — admin-configurable via the settings page

### Initialization

Called at startup after DB connects:
```javascript
connectDB().then(async () => {
  const { initializeEmailService } = await import('./services/emailService.js');
  await initializeEmailService();
});
```

Verifies the connection with `transporter.verify()`. If verification fails, `transporter` is set to null and email sending will fail gracefully.

### Functions

| Function | Description |
|----------|-------------|
| `initializeEmailService()` | Creates and verifies Nodemailer transporter |
| `sendPasswordResetEmail(email, resetLink)` | Sends branded HTML email with reset link (1-hour expiry) |
| `sendTestEmail(testEmail)` | Tests SMTP config by sending a verification email |

### Admin SMTP Management

Admins can configure, test, and update SMTP settings via:
- `GET /api/admin/smtp-config` — Get current config
- `PUT /api/admin/smtp-config` — Update config
- `POST /api/admin/smtp-config/test` — Send test email

---

## 11. File Uploads — Vercel Blob

### Multer Configuration (`middleware/multer.js`)

- **Storage**: In-memory (`multer.memoryStorage()`) — files are buffered in memory, not written to disk
- **Allowed MIME types**: `application/pdf`, `video/mp4`, `image/jpeg`, `image/png`, `image/jpg`, `image/webp`
- **Max file size**: 50MB (or `MAX_FILE_SIZE` env var in bytes)

### Upload Flow

```
1. Client sends multipart/form-data with file
2. Multer validates type + size, buffers in memory
3. Controller uploads buffer to Vercel Blob
4. Vercel Blob URL saved to the model (User.profilePicture, IdeaSubmission.pdfFile, etc.)
```

### Upload Points

| Route | Field(s) | Max Count |
|-------|----------|-----------|
| `PUT /api/auth/profile` | `profilePicture` | 1 |
| `POST /api/modules` | `mentorProfilePicture`, `coverImage` | 1 each |
| `PUT /api/modules/:id` | `mentorProfilePicture`, `coverImage` | 1 each |
| `PUT /api/association/outreach/templates` | `yesAttachment`, `noAttachment` | 1 each |

The frontend also has a Next.js API route at `app/api/upload/` for client-side uploads.

---

## 12. Call Service — ElevenLabs + Plivo

### Architecture

The call service bridges **Plivo** (telephony) and **ElevenLabs** (AI voice) via WebSocket audio streaming.

```
Incoming Phone Call
  → Plivo
  → POST /api/plivo/incoming (webhook)
  ← Plivo XML: connect to bidirectional WebSocket at /stream
  → WebSocket connection established

  Plivo WS ←→ Call Service ←→ ElevenLabs WS
  (mulaw 8kHz)    (transcode)    (PCM 16kHz)
```

### Entry Point (`src/app.js`)

- Uses `http.createServer()` (not just Express) to support WebSocket upgrade
- MongoDB pool: max 10, min 1
- Mounts Plivo webhook routes at `/api/plivo`
- Sets up WebSocket server at `/stream` path

### Audio Transcoding

ElevenLabs outputs **PCM 16-bit signed LE at 16kHz**. Plivo telephony expects **mulaw 8-bit at 8kHz**.

The `callManager.js` performs real-time transcoding:
1. Decode base64 PCM buffer
2. Downsample 16kHz → 8kHz (skip every other sample)
3. Compress linear PCM → mulaw (ITU-T G.711)
4. Encode as base64 and send to Plivo

### ElevenLabs Client Tools

The service implements server-side tool execution for ElevenLabs agent tools:

| Tool Name | Parameters | Action |
|-----------|-----------|--------|
| `get_student_info` | `phone_number` | Lookup student by phone (last 10 digits) → name, email, school, status |
| `get_payment_status` | `phone_number` | Lookup payment status by phone |
| `get_module_progress` | `phone_number` | Lookup completed/in-progress modules + SSI score |
| `get_faq_answer` | `category` | Return FAQ data by category (from `faqData.js`) |
| `get_technical_support` | `topic` | Return technical support info (from `faqData.js`) |

### WebSocket Message Flow

**Plivo → Call Service:**
- `start` — Stream initiated, captures `streamId`
- `media` — Audio chunks forwarded to ElevenLabs
- `stop` — Stream ended, cleanup

**ElevenLabs → Call Service:**
- `conversation_initiation_metadata` — Connection confirmed
- `audio` — AI voice audio, transcoded and forwarded to Plivo
- `interruption` — Clear audio buffer on Plivo
- `user_transcript` / `agent_response` — Logged for debugging
- `client_tool_call` — Execute local tool and return result
- `ping` → respond with `pong`

---

## 13. Caching & Rate Limiting

### In-Memory Cache (`utils/cache.js`)

Lightweight Map-based cache with TTL. Works within a single Vercel Lambda instance (~5-15 minute warm window).

```javascript
import { getCached, invalidateCache, invalidateByPrefix, clearCache } from '../utils/cache.js';

// Read-through pattern:
const modules = await getCached('all_modules', 60, () => Module.find());

// Invalidate after write:
invalidateCache('all_modules');
invalidateByPrefix('module_');
```

### Chat Rate Limiting (`utils/chatRateLimit.js`)

Per-student message limit across all AI chats.

| Setting | Default | Source |
|---------|---------|--------|
| Message limit | 200 | `AppSettings.chatMessageLimit` |
| Window | 24 hours | `AppSettings.chatResetHours` |
| Demo limit | 10 | Hardcoded for `demo@futuretitans.com` |

Settings are cached for 60 seconds to avoid DB lookups on every message.

`checkChatRateLimit(studentId)` returns:
```javascript
{
  used: 45,           // Messages sent in current window
  remaining: 155,     // Messages left
  limit: 200,         // Current limit
  limitReached: false, // Whether limit hit
  resetAt: Date,      // When oldest message expires (only if limitReached)
  windowHours: 24     // Window duration
}
```

### API Rate Limiting

The `rateLimiter.js` middleware is currently **passthrough** (no-op). Rate limiting is deferred to the edge layer (Vercel/Cloudflare firewall rules).

```javascript
// Currently all passthroughs:
export const authLimiter = (req, res, next) => next();
export const apiLimiter = (req, res, next) => next();
export const uploadLimiter = (req, res, next) => next();
```

---

## 14. Deployment — Vercel

### Service Deployments

| Service | Platform | Entry Point | Warm-Up |
|---------|----------|-------------|---------|
| Backend | Vercel Serverless | `src/index.js` | Cron `/health` every 5 min |
| Auth Service | Vercel Serverless | `src/index.js` | Cron `/health` every 5 min |
| Core Service | Vercel Serverless | `src/index.js` | Cron `/health` every 5 min |
| Chat Service | Vercel Serverless | `src/index.js` | Cron `/health` every 5 min |
| Call Service | Railway | `src/app.js` | Always-on (needs WebSocket) |
| Frontend | Vercel | `next build` | N/A |

### Production URLs

| Service | URL |
|---------|-----|
| Frontend | `youngpreneurs.ai` |
| Backend | `future-titans-backend-gamma.vercel.app` |

### Lambda Configuration

All Vercel functions:
- **Max duration**: 60 seconds
- **Memory**: 512 MB
- **Trust proxy**: Enabled (`app.set('trust proxy', 1)`) for correct IP detection

### Important Deployment Notes

1. **File uploads**: Must use Vercel Blob (in-memory → Blob), never local disk (ephemeral in serverless)
2. **In-memory cache**: Only persists within a single Lambda warm instance (~5-15 min)
3. **WebSocket**: Not supported on Vercel Serverless — call-service deploys to Railway
4. **Auto-index**: Disabled in production (`autoIndex: process.env.NODE_ENV !== 'production'`)

---

## 15. Testing

### End-to-End Tests (Playwright)

Location: `frontend/e2e/`

```bash
cd frontend
npm run test:e2e          # Headless Chromium
npm run test:e2e:ui       # Interactive UI mode
```

**Configuration** (`playwright.config.js`):
- Test directory: `./e2e`
- Timeout: 30 seconds per test
- Retries: 1
- Browser: Chromium (headless)
- Screenshots: Only on failure
- Traces: On first retry
- Auto-starts dev server on port 3000 if not running

**Test Suites:**

| File | Tests |
|------|-------|
| `auth.spec.js` | Invalid login shows error, empty fields validation, signup page has required fields, forgot-password page loads, login links to signup |
| `health.spec.js` | Homepage loads (200 status), login page loads, signup page loads, navigation links present |
| `protected-routes.spec.js` | Unauthenticated access to `/student/dashboard`, `/student/profile`, `/admin`, `/admin/students`, `/admin/analytics` redirects to login |
| `public-pages.spec.js` | `/about-us`, `/contact`, `/blog`, `/for-schools`, `/for-parents` load successfully (< 400 status). Contact page has form or contact info |

### Backend Tests (Jest)

```bash
cd backend && npm test
```

Jest is configured but no test files currently exist.

---

## 16. API Reference — All Endpoints

### Auth Routes (`/api/auth`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/signup` | None (rate-limited) | Register new user |
| POST | `/login` | None (rate-limited) | Login, returns JWT + refresh token |
| POST | `/refresh` | None | Refresh access token |
| POST | `/logout` | JWT | Logout user |
| GET | `/profile` | JWT | Get authenticated user's profile |
| GET | `/completion-status` | JWT | Check profile completion status |
| PUT | `/profile` | JWT | Update profile (supports `profilePicture` file upload) |
| POST | `/forgot-password` | None | Request password reset email |
| POST | `/reset-password` | None | Reset password with token |

### Payment Routes (`/api/payment`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/initiate` | JWT | Create Razorpay order |
| POST | `/verify` | None | Verify payment webhook (HMAC signature) |
| GET | `/status` | JWT | Get current user's payment status |

### Module Routes (`/api/modules`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | JWT | List all modules |
| GET | `/:id` | JWT | Get module by ID |
| GET | `/:moduleId/chapters/:chapterId` | JWT | Get chapter content |
| POST | `/:moduleId/track-time` | JWT | Track time spent on module |
| POST | `/` | JWT + Admin | Create module (with `mentorProfilePicture`, `coverImage` uploads) |
| PUT | `/:id` | JWT + Admin | Update module (with uploads) |
| DELETE | `/:id` | JWT + Admin | Delete module |
| POST | `/:id/publish` | JWT + Admin | Publish/unpublish module |
| POST | `/:moduleId/chapters` | JWT + Admin | Create chapter |
| PUT | `/:moduleId/chapters/:chapterId` | JWT + Admin | Update chapter |
| DELETE | `/:moduleId/chapters/:chapterId` | JWT + Admin | Delete chapter |
| POST | `/:moduleId/reorder-chapters` | JWT + Admin | Reorder chapters |

### AI Chat Routes (`/api/chat`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/:moduleId/chapters/:chapterId/message` | JWT + Student | Send message in chapter-specific AI chat |
| GET | `/:moduleId/chapters/:chapterId/history` | JWT + Student | Get chapter chat history |
| POST | `/:moduleId/chapters/:chapterId/finish` | JWT + Student | Finish chapter chat (finalize SSI) |
| POST | `/:moduleId/chapters/:chapterId/complete` | JWT + Student | Mark chapter as completed |
| GET | `/ssi` | JWT + Student | Get student's SSI scores |
| GET | `/report` | JWT + Student | Get student's SSI report |
| POST | `/global/message` | JWT + Student | Send message in global SURGE chat |
| GET | `/global/history` | JWT + Student | Get global chat history |
| GET | `/rate-limit-status` | JWT + Student | Check chat rate limit status |
| GET | `/admin/student/:studentId/chats` | JWT + Admin | Get student's AI chats (admin view) |
| PUT | `/admin/chat/:chatId/override-ssi` | JWT + Admin | Override SSI score on a chat |

### Submission Routes (`/api/submission`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/draft` | JWT + Student | Save submission draft |
| POST | `/submit` | JWT + Student | Submit idea |
| GET | `/` | JWT + Student | Get own submission |
| PUT | `/:submissionId` | JWT + Student | Update own submission |
| GET | `/admin/all` | JWT + Admin | Get all submissions (with filters) |
| GET | `/admin/:submissionId` | JWT + Admin | Get submission details |
| PUT | `/admin/:submissionId/score` | JWT + Admin | Score a submission |
| GET | `/admin/:submissionId/download/:fileType` | JWT + Admin | Download submission file |
| DELETE | `/admin/:submissionId` | JWT + Admin | Delete submission |

### Admin Routes (`/api/admin`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/dashboard` | JWT + Admin | Dashboard stats |
| GET | `/students` | JWT + Admin | List students (with filters) |
| GET | `/students/:studentId` | JWT + Admin | Student detail |
| PUT | `/students/:studentId` | JWT + Admin | Update student status |
| POST | `/students/:studentId/tag` | JWT + Admin | Tag a student |
| DELETE | `/students/:studentId` | JWT + Admin | Delete student |
| GET | `/analytics` | JWT + Admin | Analytics data |
| GET | `/export/students` | JWT + Admin | Export student data |
| GET | `/slugs` | JWT + Admin | List school slugs |
| POST | `/slugs` | JWT + Admin | Create school slug |
| PUT | `/slugs/:slugId` | JWT + Admin | Update school slug |
| DELETE | `/slugs/:slugId` | JWT + Admin | Delete school slug |
| GET | `/slugs/:slug/students` | JWT + Admin | Get students by school slug |
| PUT | `/slugs/:slugId/proof` | JWT + Admin | Review school proof |
| GET | `/smtp-config` | JWT + Admin | Get SMTP configuration |
| PUT | `/smtp-config` | JWT + Admin | Update SMTP configuration |
| POST | `/smtp-config/test` | JWT + Admin | Send test email |
| GET | `/chat-settings` | JWT + Admin | Get chat rate limit settings |
| PUT | `/chat-settings` | JWT + Admin | Update chat rate limit settings |
| GET | `/associations` | JWT + Admin | List associations |
| POST | `/associations` | JWT + Admin | Create association |
| PUT | `/associations/:id` | JWT + Admin | Update association |

### Achievement Routes (`/api/achievements`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | JWT | Get all achievements for authenticated student |

### School POC Routes (`/api/school-poc`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/login` | None | Login as School POC |
| GET | `/dashboard` | School POC JWT | Get school dashboard with student data |

### Association Routes (`/api/association`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/dashboard` | JWT + Association | Dashboard stats |
| POST | `/school` | JWT + Association | Create school for the association |
| POST | `/school/upload-proof` | JWT + Association | Upload proof of school acceptance |
| GET | `/places/autocomplete` | JWT + Association | Google Places search proxy |
| GET | `/school/check` | JWT + Association | Check school name availability |
| POST | `/outreach/form` | JWT + Association | Create or get outreach form |
| GET | `/outreach/responses` | JWT + Association | Get form and responses |
| PUT | `/outreach/templates` | JWT + Association | Update email templates (with file attachments) |
| POST | `/outreach/send-email` | JWT + Association | Send outreach email |
| DELETE | `/outreach/responses/:responseId` | JWT + Association | Delete a response |
| POST | `/outreach/submit/:slug` | **None (public)** | Submit outreach response (school fills this) |

### Blog Routes (`/api/blogs`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/preview-link` | None | Get link preview metadata |
| GET | `/` | None | List all blogs |
| GET | `/:slug` | None | Get blog by slug |
| POST | `/` | None* | Create blog (*should have admin auth) |
| PUT | `/:id` | None* | Update blog (*should have admin auth) |
| DELETE | `/:id` | None* | Delete blog (*should have admin auth) |

> **Note**: Blog create/update/delete routes currently lack auth middleware in the backend monolith. The core-service may have its own auth setup.

### Newsletter Routes (`/api/newsletter`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/subscribe` | None | Subscribe to newsletter |
| GET | `/subscribers` | None* | List all subscribers (*should have admin auth) |

### Call Service Routes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/plivo/incoming` | None (Plivo webhook) | Returns Plivo XML to start bidirectional WebSocket stream |
| WS | `/stream` | None | WebSocket endpoint for bidirectional audio streaming |
| GET | `/health` | None | Health check |

---

## Appendix: Key File Paths Quick Reference

| What | Path |
|------|------|
| Backend entry point | `backend/src/index.js` |
| Frontend entry point | `frontend/app/layout.js` |
| API client config | `frontend/lib/api.js` |
| Auth utilities | `frontend/lib/auth.js` |
| Zustand store | `frontend/store/useAuthStore.js` |
| Design tokens | `frontend/styles/design-system.js` |
| Tailwind config | `frontend/tailwind.config.js` |
| Next.js config | `frontend/next.config.js` |
| Playwright config | `frontend/playwright.config.js` |
| E2E tests | `frontend/e2e/*.spec.js` |
| AI engine | `backend/src/services/aiEngine.js` |
| Payment service | `backend/src/services/paymentService.js` |
| Email service | `backend/src/services/emailService.js` |
| DB config | `backend/src/config/database.js` |
| In-memory cache | `backend/src/utils/cache.js` |
| Chat rate limiter | `backend/src/utils/chatRateLimit.js` |
| Auth middleware | `backend/src/middleware/auth.js` |
| Error handler | `backend/src/middleware/errorHandler.js` |
| File upload config | `backend/src/middleware/multer.js` |
| Call service entry | `call-service/src/app.js` |
| WebSocket manager | `call-service/src/streams/callManager.js` |
| Call service tools | `call-service/src/services/tools.js` |
