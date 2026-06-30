# 🌙 ShamiLearn (Learn-Shami)
A full-stack web app for learning Shami (Levantine) Arabic through flashcards, quizzes, AI-generated audio, and SM-2 spaced repetition.
---
## 🛠️ Tech Stack
- Next.js 15 (App Router, Turbopack dev server) + React 19 + TypeScript
- MongoDB (native driver via a shared `clientPromise`, no Mongoose)
- NextAuth.js v4 (JWT sessions, credentials provider, MongoDB adapter)
- OpenAI API (`tts-1` text-to-speech) for generating lesson pronunciation audio (offline script, not a live endpoint)
- Tailwind CSS 3 + Framer Motion + shadcn/ui-style components (Radix primitives, `class-variance-authority`)
- PapaParse for CSV → lesson data ingestion
---
## ✅ Prerequisites
- Node.js 18+
- A MongoDB connection string (Atlas or local)
- An OpenAI API key (only needed if you want to regenerate audio via the `generate-audio` scripts — not required to run the app against the existing `public/audio` files)
---
## 🚀 Setup
### 1. Clone the repo
```bash
git clone https://github.com/yourusername/shamilearn.git
cd shamilearn
```
### 2. Install dependencies
```bash
npm install
```
### 3. Configure environment variables
Create a `.env.local` file in the project root:
```
MONGODB_URI=your_mongodb_connection_string
MONGODB_DB=learn-shami
NEXTAUTH_SECRET=your_super_secret_key_at_least_32_characters
NEXTAUTH_URL=http://localhost:3000
OPENAI_API_KEY=your_openai_api_key
```
`MONGODB_DB` is optional — every script and route falls back to the database name `learn-shami` if it's not set. `.env*` is already in `.gitignore`.
### 4. Seed the database
The app reads lessons, users, and progress entirely from MongoDB — there's no auto-seeding on first run, so populate it manually:
```bash
npm run setup-users      # creates demo + admin accounts (see Notes below)
node scripts/seedLessons.js   # parses lessons/*.csv + *.json into the `lessons` collection (all 5 lessons)
```
`npm run populate-lessons` only seeds **lesson 1** — prefer `node scripts/seedLessons.js` to load all five lessons at once.
### 5. Run the dev server
```bash
npm run dev
```
Open [http://localhost:3000](http://localhost:3000).

Other scripts: `npm run build`, `npm run start`, `npm run test-mongo` (sanity-checks the Mongo connection via `scripts/test-connection.js`), `npm run generate-audio` / `generate-wav-audio` (regenerate pronunciation clips with OpenAI TTS).
---
## 📡 API Endpoints
### 🔑 `POST /api/auth/register`
Registers a new learner account. Always created with `role: "learner"`.

**Auth required:** None.

**Request body:**
```json
{ "email": "jane@example.com", "password": "secret1", "name": "Jane Doe" }
```

**Response `200`:**
```json
{ "success": true, "user": { "id": "...", "email": "jane@example.com", "name": "Jane Doe", "role": "learner" } }
```

**Errors:** `400` missing fields or password under 6 characters · `409` email already registered · `500` server error.
---
### 🔑 `GET / POST /api/auth/[...nextauth]`
NextAuth.js catch-all route (`/api/auth/signin`, `/api/auth/session`, `/api/auth/signout`, etc.). Uses the `credentials` provider — email + bcrypt-compared password — and issues a 30-day JWT session containing `id` and `role`.

**Auth required:** None to sign in; this *is* the auth mechanism.
---
### 📖 `GET /api/lessons/[id]`
Fetches one lesson's full metadata and flashcard/quiz data by numeric ID.

**Auth required:** None.

**Response `200`:**
```json
{
  "lessonId": 1,
  "title": "Greetings",
  "description": "Learn common Shami greetings for daily interactions.",
  "difficulty": "Beginner",
  "tags": ["shami", "arabic", "greetings"],
  "totalItems": 20,
  "data": [
    { "id": "greet_0001", "topic": "greetings", "type": "word", "arabic": "مرحبا", "transliteration": "marHaba", "english": "Hello", "source": "Seed", "notes": "Common informal hello", "audioUrl": "" }
  ],
  "unit": 1,
  "order": 1,
  "estimatedTime": "5 minutes"
}
```

**Errors:** `400` lesson ID isn't numeric · `404` lesson not found in the `lessons` collection · `500` server error.
---
### 📈 `GET /api/progress`
Returns **every** lesson combined with the current user's completion progress for each.

**Auth required:** Session cookie (any authenticated user).

**Response `200`:** an array, one entry per lesson:
```json
[
  { "lessonId": 1, "title": "Greetings", "description": "...", "totalItems": 20, "completedItems": ["greet_0001"], "progress": { "userId": "jane@example.com", "lessonId": 1, "completedItems": ["greet_0001"], "updatedAt": "..." } }
]
```

**Errors:** `401` not authenticated · `500` server error.
---
### 📈 `POST /api/progress`
Marks one flashcard item as completed for the current user (lesson ID passed in the body, not the URL).

**Auth required:** Session cookie.

**Request body:**
```json
{ "lessonId": 1, "itemId": "greet_0001" }
```

**Response `200`:** the resulting progress document. Also auto-creates a spaced-repetition `reviews` record for the item (SM-2 defaults: `interval: 1`, `easeFactor: 2.5`, `repetitions: 0`, `nextReview: now`) if one doesn't already exist.

**Errors:** `401` not authenticated · `400` missing or wrong-typed `lessonId`/`itemId` · `500` server error.
---
### 📈 `GET /api/progress/[lessonId]`
Same as `GET /api/progress` but scoped to a single lesson (lesson ID in the URL instead of the body).

**Auth required:** Session cookie.

**Response `200`:** `{ userId, lessonId, completedItems: [...], updatedAt }` (empty `completedItems` if nothing started yet).

**Errors:** `401` not authenticated · `400` lesson ID isn't numeric · `500` server error.
---
### 📈 `POST /api/progress/[lessonId]`
Same behavior as `POST /api/progress`, with `lessonId` taken from the URL path and only `itemId` in the body.

**Auth required:** Session cookie.

**Request body:**
```json
{ "itemId": "greet_0001" }
```

**Response `200`:** the resulting progress document (also creates a `reviews` record, same as above).

**Errors:** `401` not authenticated · `400` missing/wrong-typed `itemId` · `500` server error.
---
### 📈 `DELETE /api/progress/[lessonId]`
Un-marks a single item as completed (removes it from `completedItems`) and deletes its spaced-repetition `reviews` record entirely.

**Auth required:** Session cookie.

**Request body:**
```json
{ "itemId": "greet_0001" }
```

**Response `200`:** the updated progress document, or an empty-progress shape if none existed.

**Errors:** `401` not authenticated · `400` missing/wrong-typed `itemId` · `500` server error.

> ⚠️ Note the duplication: progress can be read/written both via `/api/progress` (body-based lesson ID) and `/api/progress/[lessonId]` (path-based lesson ID), with nearly identical logic implemented twice in separate files.
---
### 🔁 `GET /api/review`
Lists all spaced-repetition items currently due for review (`nextReview <= now`) for the current user, soonest first.

**Auth required:** Session cookie.

**Response `200`:** an array of review documents:
```json
[ { "userId": "jane@example.com", "lessonId": 1, "itemId": "greet_0001", "nextReview": "...", "interval": 1, "easeFactor": 2.5, "repetitions": 0, "updatedAt": "..." } ]
```

**Errors:** `401` not authenticated · `500` server error.
---
### 🔁 `POST /api/review`
Submits a recall grade for one review item and recalculates its next review date using the **SM-2** algorithm. Also updates the user's daily learning streak.

**Auth required:** Session cookie.

**Request body:**
```json
{ "itemId": "greet_0001", "grade": 4 }
```
`grade` must be an integer `0`–`5` (SM-2 scale: 0–2 = fail/reset, 3–5 = pass with increasing ease).

**Response `200`:** the updated review document (`repetitions`, `interval`, `easeFactor`, `nextReview` recalculated). As a side effect, upserts the `user_stats` collection: if the user already reviewed something today the streak is unchanged, a review exactly one UTC day after the last one increments the streak, and any longer gap resets it to `1`.

**Errors:** `401` not authenticated · `400` missing `itemId` or `grade` outside `0`–`5` · `404` no existing review record for that `itemId` (must complete the item via `/api/progress` first, which auto-creates the review record) · `500` server error.
---
### 📊 `GET /api/stats`
Returns a dashboard summary for the current user: total items learned, items due today, current streak, and a per-lesson breakdown of due counts.

**Auth required:** Session cookie.

**Response `200`:**
```json
{
  "totalLearned": 12,
  "dueToday": 3,
  "streak": 5,
  "lastReviewDate": "2026-06-29T00:00:00.000Z",
  "reviewsDoneToday": 1,
  "perLessonDue": [ { "lessonId": 1, "dueCount": 2 } ]
}
```

**Errors:** `401` not authenticated · `500` server error.
---
### 🧪 `GET /api/test-mongo`
Debug-only endpoint that verifies the MongoDB connection by reading a lesson, inserting a throwaway test document, then deleting it.

**Auth required:** None — **this should not ship to production** (see Notes below).

**Response `200`:**
```json
{ "message": "MongoDB connection working", "lessonFound": true, "testInserted": true, "testId": "...", "lessonCount": 5 }
```

**Errors:** `500` if the connection or test write fails.
---
## 🗃️ Data Models
All collections live in MongoDB and are accessed via the plain `mongodb` driver (no Mongoose schemas) — shapes below are inferred from the seed scripts and route handlers.

### `users`
```ts
{
  _id: ObjectId,
  email: string,         // lowercased
  name: string,
  password: string,       // bcrypt hash, cost factor 12
  role: 'learner' | 'admin', // registration always creates 'learner'
  createdAt: Date,
  updatedAt: Date
}
```
---
### `lessons`
```ts
{
  id: number,            // unique index
  title: string,
  description: string,
  difficulty: string,     // e.g. "Beginner"
  tags: string[],
  csv: string,             // source CSV filename
  unit: number,
  order: number,
  estimatedTime: string,   // e.g. "5 minutes"
  data: Array<{            // parsed from lessons/*.csv at seed time
    id: string,            // e.g. "greet_0001"
    topic: string,
    type: string,           // e.g. "word"
    arabic: string,
    transliteration: string,
    english: string,
    source: string,
    notes: string,
    audioUrl: string
  }>,
  createdAt: Date, updatedAt: Date
}
```
---
### `progress`
```ts
{
  _id: ObjectId,
  userId: string,        // session.user.email
  lessonId: number,
  completedItems: string[], // item IDs from a lesson's `data` array
  updatedAt: Date
}
```
---
### `reviews` (SM-2 spaced repetition)
```ts
{
  userId: string,         // session.user.email
  lessonId: number,
  itemId: string,
  nextReview: Date,
  interval: number,        // days until next review
  easeFactor: number,      // SM-2 ease factor, min 1.3, starts at 2.5
  repetitions: number,     // consecutive successful reviews
  updatedAt: Date
}
```
---
### `user_stats`
```ts
{
  userId: string,          // session.user.email
  streak: number,          // consecutive days with a review
  lastReviewDate: Date     // UTC midnight of the last review day
}
```
---
## 📝 Notes for Developers
- **`GET /api/test-mongo` is unauthenticated and writes to the database** (inserts then deletes a test document) on every call. It's a debugging leftover — remove it or gate it behind an admin check before deploying publicly.
- **Progress logic is duplicated** across `/api/progress` (lesson ID in the request body) and `/api/progress/[lessonId]` (lesson ID in the URL) — both implement near-identical create/update/delete logic against the same `progress` and `reviews` collections. Consolidating these would reduce the chance of the two drifting apart.
- **`reviews` records are created implicitly**, not explicitly: marking a flashcard "completed" via either progress endpoint auto-creates its SM-2 review entry. `POST /api/review` will 404 if you try to grade an item that was never marked completed first.
- **`role: "admin"` exists in the `users` schema and `setup-users.js` script, but no route in this API currently checks for it** — there's no admin panel or admin-gated endpoint in this codebase (unlike the sister `Supplier-Ecommerce` project). The role field appears to be reserved for future use.
- **`npm run populate-lessons` only seeds lesson 1.** To load all five lessons (greetings, introductions, numbers, phrases, family), run `node scripts/seedLessons.js` instead, which also creates the unique index on `lessons.id`.
- **Audio files are pre-generated, not served dynamically.** `audioUrl` in the CSV/lesson data is largely empty; actual `.wav` files live under `public/audio/lesson{N}/` and are referenced by the frontend player components directly. Regenerating them requires `OPENAI_API_KEY` and the `generate-audio`/`generate-wav-audio` scripts.
- **Demo credentials** (created by `npm run setup-users`): `demo@email.com` / `demo123` (role `learner`) and `admin@example.com` / `password` (role `admin`, though no admin-only routes consume it yet). Don't run this script against a production database with these defaults.
- **No Next.js middleware route protection** (no `middleware.ts`) — auth gating for pages like `/lessons`, `/review`, `/profile` is handled client-side via the `ProtectedRoute` component (`components/auth/protected-route.tsx`), not at the routing layer. API routes each check `getServerSession` individually.
