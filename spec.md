# ContextForge — Technical Specification (v1)

> **One-liner:** A guided AI tool that helps everyday users craft better prompts by asking smart
> clarifying questions, learning preferences over time, and organizing context into reusable spaces.

---

## 1. Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                    FRONTEND (React + TypeScript)          │
│                                                          │
│   Landing Page ──► Auth ──► Dashboard ──► Guided Flow    │
│                                  │             │         │
│                           Context Spaces    Rate Result   │
│                                  │             │         │
│                            Admin Analytics     │         │
└──────────────────────────┬───────────────────────────────┘
                           │  API calls
                           ▼
┌──────────────────────────────────────────────────────────┐
│                 BACKEND (Next.js API Routes)              │
│                                                          │
│   /api/auth/*          User authentication               │
│   /api/spaces/*        CRUD for context spaces           │
│   /api/sessions/*      Guided flow sessions              │
│   /api/generate/*      Gemini API orchestration          │
│   /api/ratings/*       Thumbs up/down feedback           │
│   /api/analytics/*     Usage tracking (admin only)       │
│   /api/profile/*       User profile management           │
└──────────────────────────┬───────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
┌──────────────────────┐  ┌──────────────────────────┐
│   Database (Prisma)  │  │   Gemini API (Free Tier)  │
│   SQLite → Postgres  │  │   gemini-2.5-flash-lite   │
└──────────────────────┘  └──────────────────────────┘
```

**Hosting:** Vercel (free tier)
**Domain:** Your choice (~$12/year)

---

## 2. Data Models (Prisma Schema)

These are the database tables that store everything in your app.

```prisma
// schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"       // Switch to "postgresql" when deploying
  url      = env("DATABASE_URL")
}

// ─── USER ───────────────────────────────────────────────
// Stores each registered user
model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  // General preferences that apply across all spaces
  profile       Profile?

  // Everything the user owns
  spaces        Space[]
  sessions      Session[]
  ratings       Rating[]
}

// ─── PROFILE ────────────────────────────────────────────
// Minimal user preferences. Most fields are LOCAL ONLY (used by the app UI,
// never sent to the AI). Only communicationStyle gets sent to Gemini, and
// only during prompt generation (API call 2) — never during question
// generation (API call 1). We intentionally keep AI payloads lean to
// protect user privacy and reduce unnecessary token usage.
model Profile {
  id                  String   @id @default(cuid())
  userId              String   @unique
  user                User     @relation(fields: [userId], references: [id])

  displayName         String?              // LOCAL ONLY — shown in app UI, never sent to AI
  communicationStyle  String?              // SENT TO AI (call 2 only) — "Casual", "Professional", "Friendly"

  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt
}

// ─── SPACE ──────────────────────────────────────────────
// A category of context that builds up over time
// Examples: "Cooking", "Job Search", "Home Projects", "Parenting"
model Space {
  id          String    @id @default(cuid())
  userId      String
  user        User      @relation(fields: [userId], references: [id])

  name        String                  // "Cooking", "Work Emails", etc.
  icon        String?                 // Emoji or icon identifier
  description String?                 // User's own description of this space

  // Learned context stored as JSON
  // This grows over time as the user interacts
  // Example: { "diet": "vegetarian", "allergies": ["nuts"],
  //            "householdSize": 2, "preferredCuisine": "Mediterranean" }
  learnedContext  String  @default("{}")  // JSON string

  sessionCount    Int     @default(0)     // How many times this space has been used
  lastUsedAt      DateTime?

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  sessions        Session[]

  @@index([userId])
}

// ─── SESSION ────────────────────────────────────────────
// One complete interaction: user asks something → gets questions → gets prompt
model Session {
  id          String    @id @default(cuid())
  userId      String
  user        User      @relation(fields: [userId], references: [id])

  spaceId     String?                // Which space this belongs to (null if uncategorized)
  space       Space?    @relation(fields: [spaceId], references: [id])

  // What the user originally typed
  rawInput    String

  // The clarifying questions the AI generated (stored as JSON array)
  // Example: [
  //   { "id": "q1", "question": "What tone do you want?", "answer": "Professional" },
  //   { "id": "q2", "question": "Who is the recipient?", "answer": "My landlord" }
  // ]
  questions   String    @default("[]")   // JSON string

  // The final assembled prompt
  generatedPrompt  String?

  // Status tracking
  status      String    @default("started")  // started | questions_shown | completed

  createdAt   DateTime  @default(now())
  completedAt DateTime?

  rating      Rating?

  @@index([userId])
  @@index([spaceId])
}

// ─── RATING ─────────────────────────────────────────────
// User feedback on how well the generated prompt worked
model Rating {
  id          String    @id @default(cuid())

  sessionId   String    @unique
  session     Session   @relation(fields: [sessionId], references: [id])

  userId      String
  user        User      @relation(fields: [userId], references: [id])

  score       Int                     // 1 = thumbs down, 2 = thumbs up
  comment     String?                 // Optional: what went wrong / right

  createdAt   DateTime  @default(now())

  @@index([userId])
}

// ─── ANALYTICS EVENT ────────────────────────────────────
// Lightweight event tracking for your admin dashboard
model AnalyticsEvent {
  id          String    @id @default(cuid())

  event       String                  // "session_started", "prompt_generated",
                                      // "prompt_copied", "rating_given"
  userId      String?                 // Null for anonymous events
  metadata    String?                 // JSON string for extra data

  createdAt   DateTime  @default(now())

  @@index([event])
  @@index([createdAt])
}
```

### How the Data Connects (Plain English)

- A **User** has one **Profile** (their general info) and many **Spaces**
- A **Space** is like a folder of learned context ("Cooking", "Job Search")
- Each time someone uses the app, it creates a **Session** linked to a Space
- The Session stores the raw question, the AI-generated questions + answers, and the final prompt
- After using the prompt, the user can leave a **Rating**
- Every meaningful action logs an **AnalyticsEvent** so you can track usage

---

## 3. Screen-by-Screen Flow

### Screen 1: Landing Page (`/`)

**Purpose:** Explain what the app does and get signups.

**What's on it:**
- Headline: Something like "Ask AI anything. We'll make sure it understands you."
- 3-step visual: (1) Type your question → (2) Answer a few quick questions → (3) Get a perfect prompt
- "Get Started Free" button → goes to signup
- Brief section showing before/after: a vague prompt vs. an enriched one
- Footer with basic links

**Technical notes:**
- Static page, server-rendered by Next.js for SEO
- No authentication required


### Screen 2: Sign Up / Log In (`/auth`)

**Purpose:** Create account or log in.

**What's on it:**
- Email + password signup (use NextAuth.js)
- Google OAuth option (easy to add with NextAuth)
- Simple, clean form — nothing intimidating

**Technical notes:**
- NextAuth.js handles all auth logic
- Creates User record in database on first signup
- Redirects to Dashboard after login


### Screen 3: Dashboard (`/dashboard`)

**Purpose:** Home base. See your spaces, start a new session.

**What's on it:**
- **Primary action (top and center):** Big input field — "What do you want to ask AI today?"
  This is the most important element on the page. It should feel inviting and obvious.
- **Your Spaces (below):** Grid of cards showing saved spaces
  Each card shows: icon + name, how many times used, last used date
  Clicking a space shows its learned context (what the app knows about you for this topic)
- **Recent Sessions:** Simple list of last 5-10 sessions with the raw question and rating
- **Profile link:** Small link to edit your general profile info

**Layout concept:**
```
┌─────────────────────────────────────────────────┐
│  ContextForge                    [Profile] [Logout] │
├─────────────────────────────────────────────────┤
│                                                 │
│    ┌─────────────────────────────────────────┐   │
│    │ What do you want to ask AI today?  [→]  │   │
│    └─────────────────────────────────────────┘   │
│                                                 │
│  Your Spaces                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ 🍳       │  │ 💼       │  │ 🏠       │      │
│  │ Cooking  │  │ Job      │  │ Home     │      │
│  │ 12 uses  │  │ Search   │  │ Projects │      │
│  │          │  │ 5 uses   │  │ 3 uses   │      │
│  └──────────┘  └──────────┘  └──────────┘      │
│                                                 │
│  Recent Sessions                                │
│  • "Help me write an email to..." ── 👍        │
│  • "What should I cook for..."    ── 👎        │
│  • "How do I fix my leaky..."     ── (no rating)│
│                                                 │
└─────────────────────────────────────────────────┘
```


### Screen 4: Guided Flow (`/session/new`)

**Purpose:** The core experience. This is where the magic happens.

**Step 4a — Space Selection**
After user types their question and hits enter:
- App checks if the question matches an existing space (simple keyword matching)
- Shows suggestion: "This looks like a Cooking question. Use your Cooking context?" [Yes] [No, new space] [No space]
- If new space: prompt for a name and emoji
- If no match: ask "Want to save this to a space?" with option to name one or skip

**Step 4b — Clarifying Questions**
- App sends the raw question + any existing context from the selected space to Gemini API
- Gemini returns 2-5 clarifying questions
- Questions display one at a time or as a short form (test both, see what feels better)
- User answers each one
- "Skip" option on each question (not everything is relevant every time)

**Step 4c — Prompt Generation**
- App sends everything to Gemini: raw question + space context + profile context + answers
- Gemini returns the enriched, well-structured prompt
- Display the prompt in a nice formatted box
- Big "Copy to Clipboard" button
- "Regenerate" button if they want a different version

**Step 4d — Post-Use Rating**
- Below the generated prompt: "After you use this prompt, let us know how it went!"
- Two buttons: 👍 and 👎
- Optional text field: "What could have been better?"
- This screen can also be accessed later from the dashboard's recent sessions

**Layout concept for Step 4c:**
```
┌─────────────────────────────────────────────────┐
│  ← Back to Dashboard          🍳 Cooking Space  │
├─────────────────────────────────────────────────┤
│                                                 │
│  Your question:                                 │
│  "I need a dinner recipe for tonight"           │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  YOUR ENRICHED PROMPT                   │    │
│  │                                         │    │
│  │  I'm looking for a vegetarian dinner    │    │
│  │  recipe for two people. I have about    │    │
│  │  30 minutes to cook. I prefer           │    │
│  │  Mediterranean cuisine and have a nut   │    │
│  │  allergy. I'd like something with       │    │
│  │  pasta as the base. Please include      │    │
│  │  exact measurements and step-by-step    │    │
│  │  instructions suitable for an           │    │
│  │  intermediate home cook.                │    │
│  │                                         │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  [📋 Copy to Clipboard]  [🔄 Regenerate]        │
│                                                 │
│  ── How did it go? ──                           │
│  [👍 Great]  [👎 Not great]                     │
│  [Optional: tell us more _________________]     │
│                                                 │
└─────────────────────────────────────────────────┘
```

**What happens behind the scenes after rating:**
- If thumbs up: the answers from the clarifying questions get merged into the
  space's learnedContext (so they won't be asked again next time)
- If thumbs down + comment: flag this session for your own review later;
  don't auto-merge context since it didn't work well


### Screen 5: Space Detail (`/spaces/[id]`)

**Purpose:** View and manage what the app has learned about you for a specific topic.

**What's on it:**
- Space name and icon (editable)
- "What I know about you" section showing the learned context in plain language
  Example: "You're vegetarian, cook for 2, prefer Mediterranean, have a nut allergy"
- Edit/delete individual context items
- Session history for this space (list of past sessions with ratings)
- "Start new session in this space" button
- Delete space option

**Why this matters:** Gives users transparency and control. They can see exactly what the app
has learned and correct anything that's wrong. Builds trust, especially for non-technical users.


### Screen 6: Profile (`/profile`)

**Purpose:** Edit general preferences. Kept intentionally minimal.

**What's on it:**
- Display name (used in the app UI only — never sent to AI)
- Communication style preference (dropdown: Casual, Professional, Friendly, etc.)
- Save button

**Technical notes:**
- Only communicationStyle gets sent to Gemini, and only during prompt
  generation (API call 2). Display name is purely for the app UI.
- We intentionally keep this slim. The real personalization lives in
  context spaces, not the profile. Less data sent to AI = better privacy.


### Screen 7: Admin Analytics (`/admin`)

**Purpose:** YOUR dashboard to track usage. Not visible to regular users.

**What's on it:**
- Total users (all time)
- Sessions today / this week / this month / all time
- Prompts generated (total)
- Prompts copied (how many people actually used what they generated)
- Average rating (what % thumbs up vs thumbs down)
- Simple line chart showing sessions per day over time
- Most popular spaces (what topics are people using most)

**Technical notes:**
- Protected route — only accessible by your account (hardcode your user ID initially)
- Queries the AnalyticsEvent table
- Use a simple charting library (Recharts works great with React)

```
┌─────────────────────────────────────────────────┐
│  Admin Dashboard                                │
├─────────────────────────────────────────────────┤
│                                                 │
│  Total Users: 142    Sessions Today: 38         │
│  Total Prompts: 1,847  Copy Rate: 89%           │
│  Avg Rating: 78% positive                       │
│                                                 │
│  Sessions Over Time                             │
│  ┌─────────────────────────────────────────┐    │
│  │          ╱╲    ╱╲                       │    │
│  │    ╱╲  ╱    ╲╱    ╲   ╱╲               │    │
│  │  ╱    ╲              ╲╱  ╲  ╱           │    │
│  │╱                          ╲╱            │    │
│  └─────────────────────────────────────────┘    │
│  Mar 1 ─────────────────────────── Mar 30       │
│                                                 │
│  Top Spaces: Cooking (23%), Work Email (18%),   │
│  Job Search (12%), Home (9%), Health (7%)        │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## 4. API Routes

### Authentication
```
POST   /api/auth/signup        Create new account
POST   /api/auth/login         Log in
POST   /api/auth/logout        Log out
GET    /api/auth/session       Get current session
```
(NextAuth.js handles most of this automatically)

### Profile
```
GET    /api/profile            Get current user's profile
PUT    /api/profile            Update profile
```

### Spaces
```
GET    /api/spaces             List all spaces for current user
POST   /api/spaces             Create new space
GET    /api/spaces/[id]        Get space detail + learned context
PUT    /api/spaces/[id]        Update space (name, icon, edit context)
DELETE /api/spaces/[id]        Delete space
```

### Sessions (the guided flow)
```
POST   /api/sessions           Create new session (with rawInput)
POST   /api/sessions/[id]/questions    Generate clarifying questions (calls Gemini)
POST   /api/sessions/[id]/answers      Submit answers to questions
POST   /api/sessions/[id]/generate     Generate the final prompt (calls Gemini)
GET    /api/sessions           List recent sessions for current user
GET    /api/sessions/[id]      Get session detail
```

### Ratings
```
POST   /api/ratings            Submit a rating for a session
```

### Analytics (admin only)
```
GET    /api/analytics/overview        Summary stats
GET    /api/analytics/sessions-over-time   Time series data
GET    /api/analytics/top-spaces      Most used space categories
```

---

## 5. Gemini API Integration

### Privacy-First Data Policy

We minimize what gets sent to the AI. Here's the full picture:

| API Call | What IS sent | What is NOT sent |
|----------|-------------|-----------------|
| Call 1: Generate questions | Raw question, space context | Name, email, profile, communication style |
| Call 2: Generate prompt | Raw question, Q&A answers, space context, communication style | Name, email, any other profile data |

The user's name and email NEVER leave your server. They exist only in your
database for authentication and your app's UI. Communication style is the
only profile field that touches the AI, and only during prompt generation.

### System Prompt for Generating Clarifying Questions

**DATA SENT TO AI:** Space context + raw question ONLY.
**NOT sent:** User's name, email, profile, or any personal info.

```
You are a context-gathering assistant inside an app called ContextForge.

The user wants to ask an AI assistant a question, but their question may lack
important detail. Your job is to generate 2-5 clarifying questions that would
make their prompt dramatically more effective.

RULES:
- Ask only questions that would meaningfully improve the AI's response
- Keep questions simple and conversational (the user may not be tech-savvy)
- Don't ask for information you already have (check the existing context below)
- Each question should have a clear purpose
- Phrase questions so they're easy to answer quickly
- Return ONLY valid JSON, no other text

EXISTING SPACE CONTEXT (what we already know about this topic):
{spaceContext}

USER'S QUESTION:
{rawInput}

Respond with this exact JSON format:
{
  "suggestedSpace": "Name of suggested space category",
  "questions": [
    {
      "id": "q1",
      "question": "The clarifying question text",
      "why": "Brief reason this question matters (shown as helper text)",
      "inputType": "text" | "select",
      "options": ["option1", "option2"]  // only if inputType is "select"
    }
  ]
}
```

### System Prompt for Generating the Final Prompt

**DATA SENT TO AI:** Space context + raw question + Q&A answers +
communication style preference ONLY.
**NOT sent:** User's name, email, or any other personal info.

```
You are a prompt engineer inside an app called ContextForge.

Your job is to take a user's raw question, their answers to clarifying questions,
and their saved context, and produce a rich, well-structured prompt they can
paste into any AI assistant (ChatGPT, Claude, Gemini, etc.) to get excellent results.

RULES:
- Write the prompt in first person, as if the user is speaking
- Include all relevant context naturally (don't make it feel like a form)
- Structure it so any AI model would understand exactly what's needed
- Include specific instructions about format, length, or tone when relevant
- Match the communication style specified below
- Keep it conversational but thorough
- Do NOT include any preamble or explanation — output ONLY the prompt text

COMMUNICATION STYLE PREFERENCE:
{communicationStyle}

SPACE CONTEXT:
{spaceContext}

ORIGINAL QUESTION:
{rawInput}

CLARIFYING Q&A:
{questionsAndAnswers}

Generate the enriched prompt now.
```

### Context Merging Logic (After Thumbs Up)

When a user rates a session thumbs up, extract new facts from their
clarifying question answers and merge them into the space's learnedContext:

```
// Pseudocode
if (rating === thumbsUp) {
  const currentContext = JSON.parse(space.learnedContext)
  const newFacts = extractFactsFromAnswers(session.questions)
  const mergedContext = { ...currentContext, ...newFacts }
  await updateSpace(space.id, { learnedContext: JSON.stringify(mergedContext) })
}
```

For v1, this merge can be simple key-value overwriting. Later, you could
use Gemini to do a smarter merge that resolves conflicts and infers patterns.

---

## 6. Build Order (What to Code First)

### Phase 1 — Foundation (Days 1-3)
1. Scaffold Next.js project with TypeScript
2. Set up Prisma with SQLite, run initial migration
3. Set up NextAuth.js (email + Google login)
4. Create basic layout component (nav bar, page wrapper)
5. Deploy skeleton to Vercel (get a live URL immediately)

### Phase 2 — Core Flow (Days 4-8)
6. Build the Dashboard page (input field + empty states)
7. Build the Session creation API route
8. Integrate Gemini API — get clarifying questions working
9. Build the question display UI (Step 4b)
10. Build the prompt generation API route + display (Step 4c)
11. Add copy-to-clipboard functionality

### Phase 3 — Context & Memory (Days 9-12)
12. Build Space creation (auto-suggest from Gemini + manual)
13. Build Space detail page — show learned context
14. Implement context merging on thumbs-up rating
15. Wire up space context into Gemini API calls
16. Build Profile page (only communicationStyle gets wired into prompt generation call)

### Phase 4 — Polish & Analytics (Days 13-16)
17. Build rating system (thumbs up/down + optional comment)
18. Build admin analytics dashboard
19. Add analytics event logging throughout the app
20. Polish UI — loading states, error handling, empty states
21. Build landing page
22. Write a basic privacy policy and terms page

### Phase 5 — Launch Prep (Days 17-18)
23. Test everything end-to-end
24. Set up a custom domain on Vercel
25. Final UI polish pass
26. Deploy and share!

---

## 7. File Structure

```
contextforge/
├── prisma/
│   └── schema.prisma              # Database schema (from Section 2)
├── src/
│   ├── app/                       # Next.js App Router
│   │   ├── layout.tsx             # Root layout (nav, global styles)
│   │   ├── page.tsx               # Landing page
│   │   ├── auth/
│   │   │   └── page.tsx           # Login / Signup
│   │   ├── dashboard/
│   │   │   └── page.tsx           # Main dashboard
│   │   ├── session/
│   │   │   └── [id]/
│   │   │       └── page.tsx       # Guided flow (questions → prompt)
│   │   ├── spaces/
│   │   │   └── [id]/
│   │   │       └── page.tsx       # Space detail view
│   │   ├── profile/
│   │   │   └── page.tsx           # Edit profile
│   │   └── admin/
│   │       └── page.tsx           # Analytics dashboard
│   ├── api/                       # API Routes
│   │   ├── auth/
│   │   │   └── [...nextauth]/
│   │   │       └── route.ts       # NextAuth config
│   │   ├── profile/
│   │   │   └── route.ts
│   │   ├── spaces/
│   │   │   ├── route.ts           # GET list, POST create
│   │   │   └── [id]/
│   │   │       └── route.ts       # GET, PUT, DELETE
│   │   ├── sessions/
│   │   │   ├── route.ts           # GET list, POST create
│   │   │   └── [id]/
│   │   │       ├── questions/
│   │   │       │   └── route.ts   # POST: generate questions
│   │   │       ├── answers/
│   │   │       │   └── route.ts   # POST: submit answers
│   │   │       └── generate/
│   │   │           └── route.ts   # POST: generate prompt
│   │   ├── ratings/
│   │   │   └── route.ts
│   │   └── analytics/
│   │       └── route.ts
│   ├── components/                # Reusable React components
│   │   ├── ui/                    # shadcn/ui components
│   │   ├── SpaceCard.tsx
│   │   ├── QuestionCard.tsx
│   │   ├── PromptDisplay.tsx
│   │   ├── RatingWidget.tsx
│   │   ├── SessionList.tsx
│   │   └── AnalyticsChart.tsx
│   ├── lib/                       # Utility functions
│   │   ├── prisma.ts              # Prisma client singleton
│   │   ├── gemini.ts              # Gemini API wrapper
│   │   ├── auth.ts                # Auth helpers
│   │   ├── context-merger.ts      # Logic for merging context after ratings
│   │   └── analytics.ts           # Event logging helper
│   └── types/                     # TypeScript type definitions
│       └── index.ts
├── public/                        # Static assets
├── .env.local                     # Environment variables (API keys, DB URL)
├── tailwind.config.ts
├── tsconfig.json
├── package.json
└── next.config.js
```

---

## 8. Environment Variables

```env
# .env.local (never commit this file)

# Database
DATABASE_URL="file:./dev.db"

# NextAuth
NEXTAUTH_SECRET="generate-a-random-secret-here"
NEXTAUTH_URL="http://localhost:3000"

# Google OAuth (for login)
GOOGLE_CLIENT_ID="your-google-oauth-client-id"
GOOGLE_CLIENT_SECRET="your-google-oauth-client-secret"

# Gemini API
GEMINI_API_KEY="your-gemini-api-key"
```

---

## 9. Key Dependencies

```json
{
  "dependencies": {
    "next": "^14",
    "@prisma/client": "^5",
    "next-auth": "^4",
    "@google/generative-ai": "^0.21",
    "tailwindcss": "^3",
    "recharts": "^2",
    "lucide-react": "^0.300",
    "zod": "^3"
  },
  "devDependencies": {
    "prisma": "^5",
    "typescript": "^5",
    "@types/react": "^18",
    "@types/node": "^20"
  }
}
```

**What each one does:**
- `next` — The framework (React + routing + API routes)
- `@prisma/client` — Talks to your database with TypeScript types
- `next-auth` — Handles login/signup/sessions
- `@google/generative-ai` — Google's official SDK for calling Gemini
- `tailwindcss` — Utility CSS framework (style without writing CSS files)
- `recharts` — Charts for your analytics dashboard
- `lucide-react` — Clean icon library
- `zod` — Validates data coming into your API routes (prevents bad data)
