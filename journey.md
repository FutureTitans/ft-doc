# Future Titans — User Journey Flowchart

## Platform Overview

Future Titans is a multi-platform innovation ecosystem with **3 user roles** across **3 surfaces**:

| Role | Surface | Entry Point |
|------|---------|-------------|
| **Student** | Flutter Mobile App | Signup → Pay → Learn |
| **Admin** | Next.js Web Panel | `/admin` |
| **School POC** | Next.js Web Panel | `/school-poc/login` |

---

## 1. Complete User Journey — High Level

```mermaid
flowchart TD
    START(["🌐 Landing Page / App Store"])

    START --> ROLE{"User Role?"}

    ROLE -->|Student| S_SIGNUP["Signup (with optional School Slug)"]
    ROLE -->|Admin| A_LOGIN["Admin Login (Web)"]
    ROLE -->|School POC| POC_LOGIN["School POC Login (Web)"]

    %% ─── STUDENT FLOW ───
    S_SIGNUP --> S_LOGIN["Login"]
    S_LOGIN --> S_DASH["📊 Dashboard"]
    S_DASH --> PAID{"Subscription Active?"}

    PAID -->|No| PAYMENT["💳 Razorpay Payment<br/>(Discounted if School Slug)"]
    PAYMENT --> VERIFY["Payment Verification"]
    VERIFY --> S_DASH

    PAID -->|Yes| MODULES["📚 Learning Modules"]
    MODULES --> CHAPTER["Chapter Player"]
    CHAPTER --> AI_CHAT["🤖 Zunova AI Chat"]
    AI_CHAT --> FINISH["Finish Chapter"]
    FINISH --> SSI["SSI Score Updated"]
    SSI --> MODULES

    S_DASH --> GLOBAL_CHAT["🌍 Global SURGE Chat"]
    S_DASH --> SUBMISSION["📝 Submit Startup Idea"]
    SUBMISSION --> DRAFT["Save Draft"]
    SUBMISSION --> SUBMIT_FINAL["Final Submission<br/>(PDF + Video)"]
    S_DASH --> PROFILE["👤 Profile"]
    PROFILE --> EDIT_PROFILE["Edit Profile"]
    PROFILE --> HELP["Help & Support"]

    %% ─── ADMIN FLOW ───
    A_LOGIN --> A_DASH["🛡️ Admin Dashboard"]
    A_DASH --> MANAGE_STUDENTS["Manage Students"]
    A_DASH --> MANAGE_MODULES["Manage Modules & Chapters"]
    A_DASH --> MANAGE_SLUGS["Manage School Slugs"]
    A_DASH --> ANALYTICS["Analytics & Export"]
    A_DASH --> REVIEW_SUBS["Review & Score Submissions"]
    A_DASH --> SETTINGS["Settings<br/>(SMTP, Chat Limits)"]

    %% ─── SCHOOL POC FLOW ───
    POC_LOGIN --> POC_DASH["🏫 School POC Dashboard"]
    POC_DASH --> VIEW_STUDENTS["View School Students<br/>& Progress"]

    style START fill:#D4AF37,stroke:#B8952E,color:#fff
    style S_DASH fill:#F5EDD6,stroke:#D4AF37
    style A_DASH fill:#1A1A1A,stroke:#D4AF37,color:#F5D76E
    style POC_DASH fill:#1A1A1A,stroke:#D4AF37,color:#F5D76E
    style PAYMENT fill:#F5D76E,stroke:#B8952E
    style AI_CHAT fill:#15803d,stroke:#14532d,color:#fff
    style GLOBAL_CHAT fill:#15803d,stroke:#14532d,color:#fff
```

---

## 2. Student Authentication Flow

```mermaid
flowchart TD
    ENTRY(["App Opens"])
    ENTRY --> CHECK{"Auth Token<br/>Exists?"}

    CHECK -->|Yes| VALIDATE["Validate Token"]
    VALIDATE -->|Valid| DASH["Dashboard"]
    VALIDATE -->|Expired| REFRESH["Refresh Token"]
    REFRESH -->|Success| DASH
    REFRESH -->|Fail| LOGIN

    CHECK -->|No| LOGIN["Login Screen"]
    LOGIN --> CREDS["Enter Email & Password"]
    CREDS --> AUTH_API["POST /api/auth/login"]
    AUTH_API -->|Success| DASH
    AUTH_API -->|Fail| ERROR["Show Error"]
    ERROR --> LOGIN

    LOGIN --> FORGOT["Forgot Password"]
    FORGOT --> RESET_EMAIL["POST /api/auth/forgot-password"]
    RESET_EMAIL --> EMAIL_SENT["📧 Reset Email Sent"]
    EMAIL_SENT --> RESET_PAGE["Reset Password Page (Web)"]
    RESET_PAGE --> NEW_PASS["POST /api/auth/reset-password"]
    NEW_PASS --> LOGIN

    LOGIN --> SIGNUP["Go to Signup"]
    SIGNUP --> SLUG{"Has School<br/>Slug URL?"}
    SLUG -->|Yes| SLUG_SIGNUP["Signup with School Slug<br/>(Discounted Price)"]
    SLUG -->|No| NORMAL_SIGNUP["Normal Signup<br/>(Full Price ₹999)"]
    SLUG_SIGNUP --> AUTH_API2["POST /api/auth/signup"]
    NORMAL_SIGNUP --> AUTH_API2
    AUTH_API2 --> LOGIN

    style DASH fill:#F5EDD6,stroke:#D4AF37
    style AUTH_API fill:#D4AF37,stroke:#B8952E,color:#fff
    style AUTH_API2 fill:#D4AF37,stroke:#B8952E,color:#fff
```

---

## 3. Student Learning Flow (Mobile App)

```mermaid
flowchart TD
    DASH["📊 Dashboard"]

    DASH --> MOD_LIST["📚 Modules List<br/>GET /api/modules"]
    MOD_LIST --> MOD_DETAIL["Module Detail<br/>GET /api/modules/:id"]
    MOD_DETAIL --> CHAPTER["▶️ Chapter Player<br/>GET /api/modules/:moduleId/chapters/:chapterId"]

    CHAPTER --> TIME_TRACK["Track Time<br/>POST /api/modules/:moduleId/track-time"]
    CHAPTER --> CHAT["🤖 Zunova AI Chat<br/>POST /api/chat/:moduleId/chapters/:chapterId/message"]
    CHAT --> CHAT_HISTORY["View Chat History<br/>GET .../history"]
    CHAT --> FINISH_CHAT["Finish Chapter Chat<br/>POST .../finish"]
    FINISH_CHAT --> COMPLETE["Complete Chapter<br/>POST .../complete"]
    COMPLETE --> SSI_UPDATE["SSI Score Recalculated"]
    SSI_UPDATE --> AWARDS["🏆 Achievement Unlocked?<br/>GET /api/achievements"]

    COMPLETE --> NEXT{"More Chapters?"}
    NEXT -->|Yes| CHAPTER
    NEXT -->|No| MOD_COMPLETE["Module Complete ✅"]
    MOD_COMPLETE --> MOD_LIST

    DASH --> GLOBAL["🌍 Global SURGE Chat<br/>POST /api/chat/global/message"]
    DASH --> SUBMIT["📝 Submission Screen"]
    SUBMIT --> SAVE_DRAFT["Save Draft<br/>POST /api/submission/draft"]
    SUBMIT --> FINAL_SUBMIT["Submit Idea<br/>(PDF + Video Upload)<br/>POST /api/submission/submit"]
    FINAL_SUBMIT --> WAIT_REVIEW["⏳ Awaiting Admin Review"]

    style DASH fill:#F5EDD6,stroke:#D4AF37
    style CHAT fill:#15803d,stroke:#14532d,color:#fff
    style GLOBAL fill:#15803d,stroke:#14532d,color:#fff
    style FINAL_SUBMIT fill:#D4AF37,stroke:#B8952E,color:#fff
```

---

## 4. Payment Flow

```mermaid
flowchart TD
    BANNER["Dashboard Payment Banner<br/>'Pay ₹price'"]
    PROFILE_PAY["Profile → Subscription & Payments"]

    BANNER --> PAY_SCREEN
    PROFILE_PAY --> PAY_SCREEN

    PAY_SCREEN["Payment Screen<br/>GET /api/payment/status"]
    PAY_SCREEN --> STATUS{"Already Paid?"}
    STATUS -->|Yes| DONE["✅ You're All Set!"]
    STATUS -->|No| SHOW_PRICE["Show Price<br/>(School Slug = Discounted)"]
    SHOW_PRICE --> TAP_PAY["Tap 'Pay ₹price Now'"]
    TAP_PAY --> INITIATE["POST /api/payment/initiate"]
    INITIATE --> RAZORPAY["🔐 Razorpay Checkout"]
    RAZORPAY --> RESULT{"Payment Result"}
    RESULT -->|Success| VERIFY["POST /api/payment/verify<br/>(orderId, paymentId, signature)"]
    VERIFY --> DONE2["✅ Payment Verified<br/>Full Access Unlocked"]
    RESULT -->|Failed| RETRY["Show Error → Retry"]
    RETRY --> TAP_PAY

    style PAY_SCREEN fill:#F5D76E,stroke:#B8952E
    style RAZORPAY fill:#1A1A1A,stroke:#D4AF37,color:#F5D76E
    style DONE2 fill:#15803d,color:#fff
```

---

## 5. Admin Flow (Web Portal)

```mermaid
flowchart TD
    A_LOGIN["Admin Login (Web)<br/>POST /api/auth/login"]
    A_LOGIN --> A_DASH["🛡️ Admin Dashboard<br/>GET /api/admin/dashboard"]

    A_DASH --> STUDENTS["👥 Student Management"]
    STUDENTS --> LIST_S["List / Search / Filter Students<br/>GET /api/admin/students"]
    STUDENTS --> DETAIL_S["Student Detail<br/>GET /api/admin/students/:id"]
    STUDENTS --> TAG_S["Tag Student<br/>POST /api/admin/students/:id/tag"]
    STUDENTS --> DELETE_S["Delete Student<br/>DELETE /api/admin/students/:id"]
    STUDENTS --> EXPORT["Export CSV<br/>GET /api/admin/export/students"]

    A_DASH --> MODULES["📚 Module Management"]
    MODULES --> CREATE_M["Create Module<br/>POST /api/modules"]
    MODULES --> EDIT_M["Edit Module<br/>PUT /api/modules/:id"]
    MODULES --> PUBLISH_M["Publish Module<br/>POST /api/modules/:id/publish"]
    MODULES --> CHAPTERS["Chapter Management<br/>Create / Edit / Delete / Reorder"]

    A_DASH --> SLUGS["🏫 School Slug Management"]
    SLUGS --> CREATE_SLUG["Create Slug (with discounted price)"]
    SLUGS --> EDIT_SLUG["Edit Slug"]
    SLUGS --> VIEW_SLUG_STUDENTS["View Students by Slug"]

    A_DASH --> SUBS["📝 Submission Review"]
    SUBS --> ALL_SUBS["List All Submissions<br/>GET /api/submission/admin/all"]
    SUBS --> SCORE["Score Submission<br/>PUT /api/submission/admin/:id/score"]
    SUBS --> DOWNLOAD["Download PDF / Video"]

    A_DASH --> ANALYTICS["📊 Analytics<br/>GET /api/admin/analytics"]
    A_DASH --> SETTINGS["⚙️ Settings"]
    SETTINGS --> SMTP["SMTP Configuration"]
    SETTINGS --> CHAT_LIMITS["Chat Message Limits & Reset"]

    style A_DASH fill:#1A1A1A,stroke:#D4AF37,color:#F5D76E
    style SLUGS fill:#F5D76E,stroke:#B8952E
```

---

## 6. School POC Flow (Web Portal)

```mermaid
flowchart TD
    POC_LOGIN["School POC Login<br/>POST /api/school-poc/login"]
    POC_LOGIN --> POC_DASH["🏫 School POC Dashboard<br/>GET /api/school-poc/dashboard"]

    POC_DASH --> VIEW["View All Students<br/>Under This School Slug"]
    VIEW --> METRICS["Student Progress & Metrics"]
    VIEW --> PAYMENT_STATUS["Payment Status per Student"]

    style POC_DASH fill:#1A1A1A,stroke:#D4AF37,color:#F5D76E
```

---

## 7. Mobile App Navigation Structure

```mermaid
flowchart TD
    APP(["📱 App Launch"])
    APP --> AUTH_CHECK{"Authenticated?"}

    AUTH_CHECK -->|No| AUTH_FLOW["Auth Flow"]
    AUTH_FLOW --> LOGIN_S["/login"]
    AUTH_FLOW --> SIGNUP_S["/signup"]
    AUTH_FLOW --> FORGOT_S["/forgot-password"]

    AUTH_CHECK -->|Yes| SHELL["Shell (Bottom Nav)"]

    SHELL --> TAB1["📊 Dashboard<br/>/dashboard"]
    SHELL --> TAB2["📚 Modules<br/>/modules"]
    SHELL --> TAB3["👤 Profile<br/>/profile"]

    TAB2 --> MOD_D["/modules/:id"]
    MOD_D --> CHAP["/modules/:id/chapter/:chapterId"]
    CHAP --> CHAT_S["/modules/:id/chapter/:chapterId/chat"]

    TAB3 --> EDIT_P["/profile/edit"]
    TAB3 --> PAY_P["/profile/payment"]
    TAB3 --> HELP_P["/profile/help"]

    SHELL --> GLOBAL_C["🌍 /global-chat"]
    SHELL --> SUBMIT_I["📝 /submit-idea"]

    SHELL --> ZUNOVA["🤖 Zunova Avatar<br/>(Floating on all tabs)"]
    ZUNOVA --> GLOBAL_C

    style APP fill:#D4AF37,stroke:#B8952E,color:#fff
    style SHELL fill:#F5EDD6,stroke:#D4AF37
    style ZUNOVA fill:#15803d,stroke:#14532d,color:#fff
```

---

## API Endpoint Summary

| Category | Endpoint | Method | Auth |
|----------|----------|--------|------|
| **Auth** | `/api/auth/signup` | POST | — |
| | `/api/auth/login` | POST | — |
| | `/api/auth/refresh` | POST | — |
| | `/api/auth/profile` | GET/PUT | Token |
| | `/api/auth/forgot-password` | POST | — |
| | `/api/auth/reset-password` | POST | — |
| **Modules** | `/api/modules` | GET | Token |
| | `/api/modules/:id` | GET | Token |
| | `/api/modules/:moduleId/chapters/:chapterId` | GET | Token |
| | `/api/modules/:moduleId/track-time` | POST | Token |
| **AI Chat** | `/api/chat/:moduleId/chapters/:chapterId/message` | POST | Student |
| | `/api/chat/:moduleId/chapters/:chapterId/finish` | POST | Student |
| | `/api/chat/:moduleId/chapters/:chapterId/complete` | POST | Student |
| | `/api/chat/global/message` | POST | Student |
| | `/api/chat/ssi` | GET | Student |
| **Payment** | `/api/payment/status` | GET | Token |
| | `/api/payment/initiate` | POST | Token |
| | `/api/payment/verify` | POST | — |
| **Submission** | `/api/submission/draft` | POST | Student |
| | `/api/submission/submit` | POST | Student |
| **Achievements** | `/api/achievements` | GET | Token |
| **Admin** | `/api/admin/dashboard` | GET | Admin |
| | `/api/admin/students` | GET | Admin |
| | `/api/admin/slugs` | CRUD | Admin |
| | `/api/admin/analytics` | GET | Admin |
| **School POC** | `/api/school-poc/login` | POST | — |
| | `/api/school-poc/dashboard` | GET | POC Token |
