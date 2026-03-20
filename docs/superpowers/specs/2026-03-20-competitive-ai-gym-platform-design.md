# Competitive AI Gym Platform — Design Specification

**Date**: 2026-03-20
**Status**: Approved
**Target Users**: Competitive athletes who already track workouts

## Overview

A competitive fitness tracking social platform where athletes log workouts, earn scores based on consistency/volume/intensity, compete on category-specific leaderboards, get personalized AI coaching and nutrition advice through a conversational RAG-powered chat, and share progress on a social feed.

## Feature Decomposition

The platform is too large for a single implementation cycle. It is decomposed into 6 features, each with its own spec/plan/implement cycle, built in dependency order:

| # | Feature | Deliverable | Phase |
|---|---------|-------------|-------|
| 001 | User Auth & Profiles | Registration, JWT login, athlete profiles with category, body stats | Backend |
| 002 | Workout Logging & Exercise Library | Exercise database, custom templates, manual logging, workout history | Backend |
| 003 | Scoring Engine & Leaderboards | Category-specific scoring algorithm, global leaderboards | Backend + ML |
| 004 | AI Coach & Nutritionist (RAG) | Conversational AI chat, RAG knowledge base, adaptive recommendations | Backend + ML |
| 005 | Social Feed & Interactions | Follow system, workout posts, likes, comments | Backend |
| 006 | Frontend | React app — dashboard, logging, leaderboard, coach chat, feed | Full Stack |

### Why this order

- 001 is foundational — everything needs users
- 002 is the core data — can't score or coach without workouts
- 003 makes it competitive — the differentiator
- 004 adds AI value — needs workout data to be meaningful
- 005 adds social — needs all the above to have content worth sharing
- 006 ties it all together — backend-first per constitution Principle X

## Architecture

### Why a monolith

The platform has tightly coupled data — a user's workout log feeds the scoring engine, which feeds the leaderboard, which shows on the social feed, and the AI coach needs access to all of it. Splitting into microservices would mean every feature calls 3-4 other services over the network just to render a dashboard. Unnecessary latency and complexity until a measurable bottleneck demands extraction.

### Why FastAPI

1. **Pydantic validation built-in** — workout logs have complex, category-varying structures. Pydantic defines strict schemas per category so a CrossFit WOD log looks different from a bodybuilding session, and invalid data gets rejected before hitting business logic.
2. **Async support** — the AI coach calls an LLM API and waits for a response. Async ensures that wait doesn't block other users from logging workouts or checking the leaderboard.
3. **Auto-generated API docs** — since the frontend comes last (feature 006), Swagger UI provides a testing interface for every endpoint during the entire backend phase.

### Project Structure

```
backend/
├── app/
│   ├── main.py              # Creates FastAPI app, registers routes
│   ├── config.py             # Reads env vars (DB URL, JWT secret, API keys)
│   ├── database.py           # PostgreSQL connection pool + session factory
│   │
│   ├── models/               # SQLAlchemy ORM — one file per database table
│   │   ├── user.py           # users table
│   │   ├── workout.py        # workouts table
│   │   ├── exercise_log.py   # exercise_logs table
│   │   ├── exercise.py       # exercises table (library)
│   │   ├── score.py          # scores table
│   │   ├── post.py           # posts table
│   │   ├── follow.py         # follows table
│   │   ├── like.py           # likes table
│   │   ├── comment.py        # comments table
│   │   ├── coach_conversation.py
│   │   ├── coach_message.py
│   │   ├── knowledge_document.py  # RAG knowledge base entries with pgvector
│   │   └── refresh_token.py  # Stored refresh tokens for JWT rotation
│   │
│   ├── schemas/              # Pydantic request/response schemas
│   │   ├── user.py
│   │   ├── workout.py
│   │   ├── score.py
│   │   ├── post.py
│   │   ├── coach.py
│   │   └── follow.py
│   │
│   ├── routes/               # Thin handlers — validate input, call service, return response
│   │   ├── auth.py
│   │   ├── workouts.py
│   │   ├── exercises.py
│   │   ├── leaderboard.py
│   │   ├── coach.py
│   │   ├── feed.py
│   │   └── follow.py
│   │
│   ├── services/             # All business logic
│   │   ├── auth.py
│   │   ├── workout.py
│   │   ├── scoring.py
│   │   ├── leaderboard.py
│   │   ├── coach.py
│   │   ├── feed.py
│   │   └── follow.py
│   │
│   ├── middleware/
│   │   ├── auth.py           # JWT verification on protected routes
│   │   └── error_handler.py  # Global exception handler — logs + returns consistent errors
│   │
│   └── ai/
│       ├── rag/
│       │   ├── embeddings.py # Convert knowledge base docs to vectors
│       │   ├── retriever.py  # Find relevant docs for user's question
│       │   └── chain.py      # Combine retrieved context + user history → LLM prompt
│       └── seed/             # JSON seed data loaded into knowledge_documents table on first run
│           ├── exercises.json
│           ├── nutrition.json
│           └── training_science.json
│
├── tests/
│   ├── unit/                 # Scoring algorithm, coach prompt building
│   └── integration/          # Full API endpoint tests with test database
│
├── alembic/                  # Database migration scripts
└── requirements.txt

frontend/
├── src/
│   ├── components/           # Reusable UI components
│   ├── pages/                # Dashboard, Leaderboard, Profile, CoachChat
│   ├── services/             # API client layer — all fetch() calls live here
│   ├── context/              # AuthContext (JWT), UserContext (profile)
│   └── App.jsx
└── tests/
```

### Why this separation

- **models/ vs schemas/ vs services/**: A workout log needs three different data shapes — the schema validates incoming JSON, the model maps to the database row (with auto-generated id, timestamps), and the service orchestrates the logic (save workout, then trigger score recalculation). They change for different reasons, so they live in different places.
- **Routes are thin**: A route handler is ~2 lines: validate via schema (automatic), call service, return result. Business logic never lives in routes so it can be reused from background jobs or other services.
- **Frontend services/**: All API calls go through `services/api.js`. Components never call `fetch()` directly. When an endpoint URL changes, it changes in one place.

### Knowledge base approach

Knowledge base documents are stored in the `knowledge_documents` PostgreSQL table with pgvector embeddings — not as static files. The `ai/seed/` directory contains JSON files with the initial knowledge base content (exercises, nutrition, training science). On first application startup, a seed script reads these JSON files, generates embeddings, and inserts them into the `knowledge_documents` table. After seeding, all reads and searches go through the database via the RAG retriever. New documents can be added via an admin endpoint without redeploying.

## Data Model

### users

| Column | Type | Purpose |
|--------|------|---------|
| id | UUID | Primary key |
| email | VARCHAR (unique) | Login identifier |
| hashed_password | VARCHAR | bcrypt hash, never plain text |
| username | VARCHAR (unique) | Display name on leaderboard/feed |
| category | ENUM | crossfit, bodybuilding, running, endurance, hybrid |
| bio | TEXT | Profile text |
| avatar_url | VARCHAR | Profile picture |
| height_cm | FLOAT | Body stats for AI coach calculations |
| weight_kg | FLOAT | Body stats for AI coach calculations |
| body_fat_pct | FLOAT (nullable) | Used by coach for nutrition advice |
| dietary_preference | ENUM | none, vegan, vegetarian, keto, etc. |
| injuries | TEXT[] | AI coach avoids exercises that aggravate these |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

Category is on the user, not on workouts. A bodybuilder and a runner doing the same 5K run are scored differently. The category determines which scoring formula applies to all their activity. Category switches reset the user's score — old score rows are deleted and fresh ones created with zeroed values, because metrics aren't comparable across categories.

### exercises

| Column | Type | Purpose |
|--------|------|---------|
| id | UUID | Primary key |
| name | VARCHAR | "Barbell Back Squat", "5K Run" |
| category | ENUM[] | Which athlete categories this applies to |
| muscle_groups | TEXT[] | ["quads", "glutes", "core"] |
| exercise_type | ENUM | strength, cardio, olympic, bodyweight, flexibility |
| difficulty | INT (1-5) | Used in scoring to weight harder exercises higher |
| instructions | TEXT | Proper form description (also used by RAG) |

Difficulty matters for scoring fairness: 10 sets of bicep curls and 10 sets of heavy deadlifts shouldn't score the same. The difficulty multiplier makes compound/hard movements worth more.

Exercise alternatives are stored in a junction table rather than a UUID array, so PostgreSQL can enforce foreign key integrity:

### exercise_alternatives

| Column | Type | Constraint |
|--------|------|------------|
| exercise_id | UUID → exercises | |
| alternative_id | UUID → exercises | |
| | | UNIQUE(exercise_id, alternative_id) |

### workouts

| Column | Type | Purpose |
|--------|------|---------|
| id | UUID | Primary key |
| user_id | UUID → users | Owner |
| date | DATE | |
| workout_type | ENUM | strength, cardio, mixed, competition |
| duration_minutes | INT | |
| notes | TEXT | User's own notes |
| score | FLOAT | Calculated score for this session |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

### exercise_logs

| Column | Type | Purpose |
|--------|------|---------|
| id | UUID | Primary key |
| workout_id | UUID → workouts | Parent workout |
| exercise_id | UUID → exercises | Which exercise |
| sets | INT (nullable) | For strength |
| reps | INT (nullable) | For strength |
| weight_kg | FLOAT (nullable) | For strength |
| distance_km | FLOAT (nullable) | For cardio |
| pace | FLOAT (nullable) | For cardio (min/km) |
| duration_seconds | INT (nullable) | For timed exercises (planks, AMRAPs) |
| rpe | INT (1-10) | Rate of perceived exertion, AI coach uses this |

Two tables (workouts + exercise_logs) instead of one because a single workout contains multiple exercises. Separate table enables direct queries like "show me all my squat progress over 3 months" with simple SQL instead of parsing JSON arrays.

### scores

| Column | Type | Purpose |
|--------|------|---------|
| id | UUID | Primary key |
| user_id | UUID → users | Owner |
| category | ENUM | Matches user's category |
| period | ENUM | weekly, monthly, all_time |
| total_score | FLOAT | The number shown on the leaderboard |
| consistency_score | FLOAT | Workout frequency |
| volume_score | FLOAT | Total work done |
| intensity_score | FLOAT | How hard they pushed |
| streak_days | INT | Current consecutive days with a workout |
| calculated_at | TIMESTAMP | When last recomputed |
| | | UNIQUE(user_id, category, period) |

The unique constraint ensures exactly one score record per user per category per period. When scores recalculate, the existing row is updated (upsert), not duplicated.

### refresh_tokens

| Column | Type | Purpose |
|--------|------|---------|
| id | UUID | Primary key |
| user_id | UUID → users | Owner |
| token_hash | VARCHAR | SHA-256 hash of the refresh token (never stored plain) |
| expires_at | TIMESTAMP | When this token expires |
| revoked | BOOLEAN | Set to true on logout or rotation |
| created_at | TIMESTAMP | |

Refresh tokens are stored server-side so they can be revoked on logout or if a token is compromised. The actual token is sent to the client; only its hash is stored. On `POST /refresh`, the server hashes the incoming token, looks it up, verifies it's not revoked or expired, then issues a new access JWT and rotates the refresh token (old one revoked, new one issued).

### follows

| Column | Type | Constraint |
|--------|------|------------|
| follower_id | UUID → users | |
| following_id | UUID → users | |
| created_at | TIMESTAMP | |
| | | UNIQUE(follower_id, following_id) |

One-directional (like Twitter). Users follow top-ranked athletes without needing permission.

### posts

| Column | Type | Purpose |
|--------|------|---------|
| id | UUID | Primary key |
| user_id | UUID → users | Author |
| workout_id | UUID → workouts (nullable) | Attached workout, if any |
| content | TEXT | Post text |
| created_at | TIMESTAMP | |

workout_id is nullable — users can share workout posts (with score and exercises attached) or plain text posts ("Rest day").

### likes

| Column | Type | Constraint |
|--------|------|------------|
| user_id | UUID → users | |
| post_id | UUID → posts | |
| | | UNIQUE(user_id, post_id) |

### comments

| Column | Type | Purpose |
|--------|------|---------|
| id | UUID | Primary key |
| user_id | UUID → users | Author |
| post_id | UUID → posts | Parent post |
| content | TEXT | |
| created_at | TIMESTAMP | |

No threading (replies-to-replies) — adds significant UI and query complexity for minimal MVP value.

### coach_conversations

| Column | Type | Purpose |
|--------|------|---------|
| id | UUID | Primary key |
| user_id | UUID → users | Owner |
| title | VARCHAR | Auto-generated from first message, editable by user |
| created_at | TIMESTAMP | |

Title is auto-generated by taking the first ~50 characters of the user's first message (e.g., "My squat isn't improving, what should..."). This gives the conversation list meaningful labels without requiring users to name conversations manually.

### coach_messages

| Column | Type | Purpose |
|--------|------|---------|
| id | UUID | Primary key |
| conversation_id | UUID → coach_conversations | Parent conversation |
| role | ENUM | user, assistant |
| content | TEXT | Message text |
| created_at | TIMESTAMP | |

Conversations stored in DB so the adaptive AI coach can reference past interactions. If a user mentioned a shoulder injury 2 weeks ago, the coach still knows.

### knowledge_documents

| Column | Type | Purpose |
|--------|------|---------|
| id | UUID | Primary key |
| title | VARCHAR | Document title |
| category | ENUM | exercise, nutrition, training_science |
| content | TEXT | Document text |
| embedding | VECTOR(1536) | pgvector for semantic search |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

pgvector instead of a separate vector database (Pinecone, Weaviate) because the knowledge base is ~500 documents. PostgreSQL + pgvector handles this easily — one less service to manage.

## Scoring System

### Formula

```
total_score = (consistency_score × 0.35) + (volume_score × 0.35) + (intensity_score × 0.30)
```

### Components

**Consistency (35%)** — how often you show up, weighted by workout difficulty.

```
consistency_score = (days_trained / days_in_period) × 100 × avg_session_difficulty
```

`avg_session_difficulty` is the average difficulty of all exercises performed across all workouts in the period. Calculated as: `sum(exercise.difficulty for each exercise_log) / count(exercise_logs)`. This means training 5 days/week doing easy exercises scores lower than training 5 days/week doing hard compound movements. Range: exercises have difficulty 1-5, so this multiplier is 1.0-5.0.

**Volume (35%)** — how much total work, calculated differently per exercise type:

```
# For strength exercises in the period:
strength_volume = sum(sets × reps × weight_kg × exercise.difficulty)

# For cardio exercises in the period:
cardio_volume = sum(distance_km × pace_multiplier)

# pace_multiplier converts pace to reward faster speeds:
#   pace_multiplier = 10 / pace (where pace is min/km)
#   Example: 5:00 min/km pace → multiplier = 2.0
#   Example: 4:00 min/km pace → multiplier = 2.5
#   Example: 6:00 min/km pace → multiplier = 1.67

# For timed exercises (planks, AMRAPs):
timed_volume = sum(duration_seconds × exercise.difficulty)

# Total volume is normalized to a 0-100 scale per category
volume_score = normalize(strength_volume + cardio_volume + timed_volume)
```

**Intensity (30%)** — how hard you pushed relative to exercise difficulty.

```
intensity_score = avg(rpe × exercise.difficulty) across all exercise_logs in the period
```

Uses RPE (1-10) × exercise difficulty (1-5), giving a range of 1-50 per exercise. Averaged across all logged exercises. Slightly lower weight (30% vs 35%) because RPE is self-reported — prevents gaming by always reporting RPE 10.

### Category-specific volume normalization

Each category normalizes volume to a 0-100 scale using different baselines, so a "perfect week" scores similarly across categories:

| Category | Volume calculation | Normalization baseline (100 = exceptional week) |
|----------|-------------------|------------------------------------------------|
| Bodybuilding | strength_volume only | 50,000 (e.g., 5 sessions × heavy compounds) |
| CrossFit | strength_volume + timed_volume | 40,000 (mixed modality) |
| Running | cardio_volume only | 100 (e.g., 50km at 5:00/km pace) |
| Endurance | cardio_volume + timed_volume | 150 (longer distances, slower pace expected) |
| Hybrid | strength_volume + cardio_volume + timed_volume | 60,000 (rewards breadth across types) |

These baselines are configurable constants in the scoring service — they'll be tuned based on real user data after launch.

### Recalculation trigger

Scores recalculate when a user logs, edits, or deletes a workout:
1. User submits/updates/deletes workout → service saves the change
2. Service calls `scoring_service.recalculate(user_id)`
3. Scoring service pulls all workouts for current week/month, runs formula, upserts into `scores` table
4. Leaderboard reads pre-calculated scores with a simple sorted query

## AI Coach & RAG System

### Conversation flow

User sends a message → system builds context → RAG retrieves relevant knowledge → LLM generates personalized response.

**Context assembly:**
1. User's profile (category, weight, injuries, dietary preference)
2. Last 30 days of workout history (exercises, progress, RPE trends)
3. Recent conversation history — uses a token budget (roughly 2000 tokens) rather than a fixed message count, so short exchanges include more history and long messages include fewer

**RAG retrieval:**
User's question is embedded into a vector. pgvector finds the 3-5 most relevant knowledge base documents (exercise form guides, nutrition info, training science articles).

**Prompt construction:**
All context (profile + workout history + retrieved documents + conversation history) is assembled into an LLM prompt. The LLM responds with specific, data-backed advice referencing the user's actual numbers.

### Why RAG over raw LLM

RAG forces the coach to reference a curated knowledge base rather than generic LLM training data. This means:
- Advice follows vetted training methodology
- Knowledge base can be updated without retraining
- Answers are grounded in specific documents you control

### Knowledge base contents (seeded at launch)

Stored as JSON seed files in `ai/seed/`, loaded into `knowledge_documents` table with generated embeddings on first run:

- **Exercise library** (~200-300 entries): form cues, muscle groups, difficulty, alternatives
- **Nutrition** (~100-200 entries): macros per goal (bulk/cut/maintain), meal timing, supplements, category-specific nutrition
- **Training science** (~100-150 entries): periodization, deload protocols, overtraining symptoms, progressive overload, recovery

### Adaptive behavior

The coach adapts because the data it sees changes over time:
- Week 1: workout history shows beginner → coach gives foundational advice
- Month 3: history shows consistent training with plateaus → coach suggests periodization
- Month 6: history shows positive response to volume blocks → coach recommends similar approaches for other lifts

Conversation history also carries context — if the user said "I'm cutting for a competition in March" three weeks ago, that message is stored and pulled into the prompt.

### Rate limiting

The coach endpoint (`POST /coach/chat`) is rate-limited to **20 messages per user per hour**. This prevents runaway LLM API costs from a single user while being generous enough for real coaching conversations (most users send 5-10 messages per session). Rate limiting is enforced at the route level using the user's JWT-identified `user_id`, not IP-based.

## Social Feed

### Feed query

Chronological feed of posts from people you follow. No algorithmic ranking — athletes want to see what their training partners did today, not what an algorithm thinks is engaging. Can add ranking later if user base demands it.

### Interactions

- **Like**: one per user per post, toggle on/off. No reaction types.
- **Comment**: plain text, no threading. Threading adds complexity for minimal MVP value.
- **Follow**: one-directional. Follow top-ranked athletes without needing permission.

### Workout post display

When a user shares a workout, the post automatically attaches: exercises performed, score earned, current streak. The competitive data is attached automatically — every post shows measurable output.

## API Overview

| Group | Endpoints | Purpose |
|-------|-----------|---------|
| Auth | `POST /register`, `POST /login`, `POST /refresh`, `POST /logout` | JWT auth, token refresh, revoke refresh token |
| Profile | `GET/PUT /profile`, `GET /users/{id}` | View/edit profile |
| Workouts | `POST /workouts`, `GET /workouts/history`, `GET /workouts/{id}`, `PUT /workouts/{id}`, `DELETE /workouts/{id}` | Log, retrieve, edit, delete (edit/delete trigger score recalculation) |
| Exercises | `GET /exercises`, `GET /exercises?category=crossfit` | Browse library |
| Leaderboard | `GET /leaderboard?category=X&period=weekly` | Ranked users |
| Coach | `POST /coach/chat`, `GET /coach/conversations` | AI chat (rate limited: 20 msg/user/hour) |
| Feed | `GET /feed`, `POST /posts`, `DELETE /posts/{id}/like`, `POST /posts/{id}/like`, `POST /posts/{id}/comments`, `DELETE /comments/{id}` | Social (like toggle, comment delete) |
| Follow | `POST /follow/{user_id}`, `DELETE /follow/{user_id}`, `GET /followers`, `GET /following` | Follow system |

### Pagination

All list endpoints use offset-based pagination with consistent query parameters:

```
GET /feed?limit=20&offset=0
GET /workouts/history?limit=20&offset=0
GET /leaderboard?category=crossfit&period=weekly&limit=50&offset=0
GET /coach/conversations?limit=20&offset=0
GET /exercises?limit=50&offset=0
GET /followers?limit=20&offset=0
GET /following?limit=20&offset=0
```

Defaults: `limit=20` (max 100), `offset=0`. Responses include a `total` count for the frontend to render pagination controls:

```json
{
  "data": [...],
  "total": 142,
  "limit": 20,
  "offset": 0
}
```

### Response format

Consistent JSON envelope across all endpoints:

```json
{ "data": { ... }, "message": "Workout logged successfully" }
{ "error": "Invalid exercise ID", "detail": "Exercise 'xyz' not found" }
```

Frontend always knows where to find data (`response.data`) and errors (`response.error`).

## Observability & Error Handling

Per constitution Principle IX, the platform MUST NOT have silent failures:

**Logging**: Every service logs key operations using Python's `logging` module with structured JSON output:
- Workout logged/edited/deleted (user_id, workout_id)
- Score recalculated (user_id, old_score, new_score)
- Coach message processed (user_id, conversation_id, response_time_ms)
- Auth events (login success/failure, token refresh, logout)
- Knowledge base operations (document added/updated, embedding generated)

**Error handling**: A global exception handler middleware (`middleware/error_handler.py`) catches all unhandled exceptions, logs the full traceback, and returns a consistent error response to the client. No endpoint silently swallows exceptions. Specific error types (validation errors, not found, unauthorized) return appropriate HTTP status codes (422, 404, 401) with descriptive messages.

**Health check**: `GET /health` returns service status and database connectivity — used for deployment verification per Principle VIII.
