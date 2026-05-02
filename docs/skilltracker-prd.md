# SkillTracker — Product Requirements Document (PRD)

| | |
|---|---|
| **Version** | 1.1.0 — Updated |
| **Date** | May 2026 |
| **Author** | Aminul Karim |
| **Status** | In Development |
| **Platform** | React Native CLI (Android + iOS) |
| **Backend** | Node.js + Express.js + PostgreSQL |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Product Vision & Goals](#2-product-vision--goals)
3. [User Personas](#3-user-personas)
4. [Screens & Feature Inventory](#4-screens--feature-inventory)
5. [Technical Architecture](#5-technical-architecture)
6. [Database Schema](#6-database-schema-postgresql)
7. [API Specification](#7-api-specification)
8. [UX Flows](#8-ux-flows)
9. [Offline-First Sync Architecture](#9-offline-first-sync-architecture)
10. [Development Roadmap](#10-development-roadmap)
11. [Security Requirements](#11-security-requirements)
12. [Non-Functional Requirements](#12-non-functional-requirements)
13. [Open Questions & Decisions](#13-open-questions--decisions)

---

## 1. Executive Summary

SkillTracker is a mobile-first skill growth tracker for developers, students, and self-learners. Users log daily learning sessions by skill, build daily streaks, visualise progress through charts and activity grids, and set goals with deadlines.

The app is **offline-first** — every action writes to local SQLite immediately, with automatic background sync to PostgreSQL whenever a connection is available.

| | |
|---|---|
| **Platform** | iOS + Android via React Native CLI (no Expo) |
| **Backend** | Node.js 20 LTS + Express.js 4 |
| **Database** | PostgreSQL 16 (cloud) + SQLite (local device) |
| **Auth** | JWT HS256 (7-day expiry) + bcrypt passwords |
| **Sync** | Offline-first, UUID v4 IDs, last-write-wins by `updated_at` |
| **Navigation** | React Navigation (stack + bottom tabs) |

---

## 2. Product Vision & Goals

### 2.1 Vision Statement

> "Build an addictive, minimal mobile app that makes skill growth visible — so users come back every day not because they have to, but because they want to."

### 2.2 Problem Statement

Self-learners lack a simple, motivating tool to track daily skill investment. Existing tools are too complex, too generic, or break without internet — a critical gap for mobile-first users in regions with intermittent connectivity.

### 2.3 Key Performance Indicators

| KPI | Target |
|---|---|
| D7 Retention | >= 40% of users return on Day 7 |
| Streak Adoption | > 60% maintain streak >= 3 days |
| Session Log Rate | >= 1 log per active user per day |
| Sync Reliability | 99% pending records synced within 60 s of reconnect |
| Crash-Free Sessions | > 99.5% on both platforms |
| Goal Creation Rate | > 50% of users create >= 1 goal in first week |

---

## 3. User Personas

### Persona A — Developer Learner

| | |
|---|---|
| **Name** | Aminul, 24, Dhaka |
| **Goal** | Land a senior React Native role within 12 months |
| **Skills** | React, DSA, Node.js, System Design |
| **Pain point** | Loses motivation after 2-3 days without visible progress |
| **Key features** | Streak system, activity heatmap, skill-level badges |

### Persona B — English Learner

| | |
|---|---|
| **Name** | Nadia, 21, Chittagong |
| **Goal** | Score 7.0 in IELTS within 6 months |
| **Skills** | IELTS Writing, Speaking, Vocabulary |
| **Pain point** | Inconsistent daily practice, no accountability |
| **Key features** | Daily reminder, streak calendar, goal progress bars |

---

## 4. Screens & Feature Inventory

The app has **10 screens** and **5 bottom-sheet modals**.

### 4.1 Onboarding (4 Slides — First Launch Only)

| Slide | Content | Data Captured |
|---|---|---|
| 1 — Welcome | App intro, tagline, animated rocket | — |
| 2 — Streaks | Explains streak system and grace day | — |
| 3 — Progress | Dashboard and activity grid preview | — |
| 4 — Setup | Name input, skill multi-select, daily goal picker | `user.name`, `selected_skills[]`, `daily_goal_hours` |

---

### 4.2 Login / Register Screen

| Element | Behaviour | API Call |
|---|---|---|
| Sign In tab | Email + password; validates demo@gmail.com / demo | `POST /auth/login` |
| Create Account tab | Name + email + password (>= 4 chars) | `POST /auth/register` |
| Forgot Password sheet | Email input → "Reset link sent" → auto-close 2.2 s | `POST /auth/forgot-password` |
| Google button | Bypass credentials, enter app | `POST /auth/social` (Phase 2) |
| GitHub button | Bypass credentials, enter app | `POST /auth/social` (Phase 2) |
| Demo hint | Shows demo@gmail.com / demo under Sign in button | — |

---

### 4.3 Home (Dashboard)

| Element | Data Source |
|---|---|
| Greeting + current date | `user.name`, device date |
| Streak banner (tap → Streak Calendar) | `streaks.current_streak` |
| This Week hrs | `SUM(logs.hours) WHERE date >= week_start` |
| Total hrs | `SUM(logs.hours) WHERE user_id = me` |
| Consistency % chip | `(days_logged / days_since_join) * trend_multiplier` |
| Today's skill cards | `logs WHERE date = today ORDER BY created_at DESC` |
| 119-day activity grid (tap → Streak Calendar) | `logs GROUP BY date` last 119 days, 5 intensity levels |
| FAB (+) | Navigate to Log Session |

---

### 4.4 Log Session Screen

| Element | Data | API Call |
|---|---|---|
| Skill chip selector | `GET /skills` | `GET /skills` |
| `+ New` chip | Inline input → creates skill on Add | `POST /skills` |
| Time pills | 30m / 1h / 2h / 3h / 4h | — |
| Custom time pill | Number input (0.5-24 hrs); pill label updates after Set | — |
| Notes textarea | Optional free text | — |
| Save session | SQLite write (pending) → toast → Home | `POST /logs` (via sync) |
| Recent entries | Last 5 logs from local DB | `GET /logs?limit=5` |
| View all link | Navigate to All Sessions screen | `GET /logs?page=1` |

---

### 4.5 Progress Screen

| Element | Data | API Call |
|---|---|---|
| Week / Month / All-time tabs | Controls date range | — |
| Hours logged bar chart | Hours grouped by date for selected range | `GET /logs/summary?range=week\|month\|all` |
| React goal ring | `SUM(hours WHERE skill='React') / goal.target * 100` | `GET /goals` |
| IELTS goal ring | Same for IELTS | `GET /goals` |
| Skill breakdown rows | `SUM(hours) GROUP BY skill_name` with progress bars | `GET /stats/by-skill` |

---

### 4.6 Goals Screen

| Element | Data | API Call |
|---|---|---|
| Goal cards | progress_pct, status label | `GET /goals` |
| Status logic | >= 80% On track; 40-79% Keep going; < 40% Behind | server-computed |
| `+ Add new goal` | Opens Add Goal bottom sheet | `POST /goals` |

**Add Goal Sheet fields:** title (required), skill, type (hours / days), target value (required), deadline (optional)

---

### 4.7 Streak Calendar Screen

| Element | Data | API Call |
|---|---|---|
| Hero stats: Current / Longest / Total days / Grace days | `streaks.*` | `GET /streak` |
| **Week tab** | 7 cells: day + date + hours + dot; summary (total hrs, days active, avg/day) | `GET /streak/weekly` |
| **Month tab** | Full month calendar grid, colour-coded cells | `GET /streak/calendar?year=&month=` |
| **3 Months tab** | Stacked Mar + Apr + May grids | `GET /streak/calendar` x3 |
| **Year tab** | 52-week heatmap, month labels | `GET /logs/activity?days=365` |
| Recent Activity | Last 5 log days with skills and hours | `GET /logs?limit=5` |

**Cell states:** `logged` (lime), `today` (solid lime), `grace` (amber), `missed` (grey), `empty`

---

### 4.8 Profile Screen

| Element | Data | API Call |
|---|---|---|
| Avatar + name + handle + join date | `user.*` | `GET /auth/me` |
| Achievement badges | streak >= 10 = Streak Master; percentile from consistency | `GET /stats/consistency` |
| Stat cards: Total hrs / Streak / Consistency % | aggregated | `GET /stats/overview` |
| Skill levels | Beginner < 5h, Intermediate 5-30h, Advanced > 30h | `GET /stats/by-skill` |
| Daily reminder toggle | `user_settings.reminder_enabled` | `PUT /settings/reminder` |
| Dark mode toggle | Local only — toggles `body.light-mode` CSS | `PUT /settings` |
| Weekly report row | Opens Weekly Report sheet | `POST /reports/weekly` |
| Export data row | Opens Export Data sheet | `GET /export?format=` |
| About row | Navigate to About screen | — |
| Feedback row | Opens Feedback sheet | `POST /feedback` |
| Sign out | Clear JWT from AsyncStorage → Login | — |

---

### 4.9 About Screen

Sections: Hero (logo, version v1.0.0), Feature tiles (4), App info, Links (Privacy Policy, Terms of Use, GitHub, Feedback), Footer.

---

### 4.10 Privacy Policy Screen

In-app. 8 sections covering data collection, usage, offline storage, deletion, third parties, children's privacy, user rights, contact.

---

### 4.11 Terms of Use Screen

In-app. 11 sections covering acceptance, account rules, acceptable use, IP, content, disclaimers, liability, changes, termination, governing law (Bangladesh), contact.

---

### 4.12 All Sessions Screen

Full log history grouped by date. Back navigates to Log Session.
API: `GET /logs?page=1&limit=50`

---

### 4.13 Bottom Sheet Modals

| Sheet | Trigger | Key Actions | API Call |
|---|---|---|---|
| Weekly Report | Profile → Weekly report | Send now, change schedule, choose skills | `POST /reports/weekly`, `PUT /settings/report` |
| Export Data | Profile → Export data | Download PDF / CSV / JSON | `GET /export?format=pdf\|csv\|json` |
| Feedback | Profile or About → Feedback | Stars + category tags + message | `POST /feedback` |
| Add Goal | Goals → + Add new goal | Title, skill, type, target, deadline | `POST /goals` |
| Forgot Password | Login → Forgot password? | Email input → send reset | `POST /auth/forgot-password` |

---

## 5. Technical Architecture

### 5.1 Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| Mobile framework | React Native CLI (no Expo) | Full native control, cross-platform |
| Language | JavaScript ES2022+ | Matches React knowledge |
| Networking | Built-in `fetch()` | Zero external dependency |
| Local DB | SQLite (react-native-sqlite-storage) | Structured offline data |
| Session storage | AsyncStorage | JWT + user profile cache |
| State | React Context + useReducer | Sufficient for MVP |
| Navigation | React Navigation | Stack + bottom tab screens |
| Backend | Node.js 20 LTS + Express.js 4 | Minimal, JS-native REST |
| Cloud DB | PostgreSQL 16 | ACID, UUID support |
| Auth | JWT + bcrypt | Stateless, industry standard |
| Config | dotenv | Secret management |

### 5.2 Mobile Project Structure

```
src/
├── screens/          OnboardingScreen, LoginScreen, HomeScreen, LogScreen,
│                     ProgressScreen, GoalsScreen, StreakScreen, ProfileScreen,
│                     AboutScreen, PrivacyScreen, TermsScreen, AllLogsScreen
├── components/       SkillCard, StreakBanner, ActivityGrid, GoalCard, LogEntry,
│                     StatCard, BarChart, RingChart, BottomSheet
├── navigation/       AppNavigator.js (stack + bottom tabs)
├── services/         api.js (fetch calls), sync.js (offline queue + NetInfo)
├── storage/          db.js (SQLite schema + CRUD), session.js (AsyncStorage JWT)
├── context/          AppContext.js (user, streak, skills, sync status)
└── utils/            dateHelpers.js, streakCalculator.js, consistencyScore.js
```

### 5.3 Backend Project Structure

```
src/
├── controllers/      auth, skill, log, goal, streak, sync,
│                     report, export, feedback, stats, settings
├── routes/           auth, skills, logs, goals, streak, sync,
│                     reports, export, feedback, stats, settings
├── middleware/       authMiddleware.js, errorHandler.js, rateLimiter.js
├── services/         streakService.js, consistencyService.js,
│                     conflictResolver.js, exportService.js
├── models/           pg Pool query wrappers per entity
├── utils/            uuid.js, timestampHelpers.js, responseFormatter.js
├── app.js            Express setup: CORS, JSON parser, routes, error handler
└── db.js             PostgreSQL Pool init + migration runner
```

---

## 6. Database Schema (PostgreSQL)

### `users`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK, DEFAULT gen_random_uuid() | Offline-safe, no auto-increment |
| `email` | VARCHAR(255) | UNIQUE NOT NULL | Login identifier |
| `password_hash` | VARCHAR(255) | NOT NULL | bcrypt, never plain text |
| `name` | VARCHAR(100) | NOT NULL | Display name |
| `avatar_url` | TEXT | NULLABLE | Profile photo |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() | Conflict resolution |
| `sync_status` | VARCHAR(20) | DEFAULT 'synced' | |

### `user_settings`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `user_id` | UUID | PK, REFERENCES users(id) | One row per user |
| `reminder_enabled` | BOOLEAN | DEFAULT true | |
| `reminder_time` | TIME | DEFAULT '21:00:00' | 9:00 PM |
| `dark_mode` | BOOLEAN | DEFAULT true | |
| `daily_goal_hours` | DECIMAL(3,1) | DEFAULT 2.0 | From onboarding |
| `report_enabled` | BOOLEAN | DEFAULT true | Weekly email |
| `report_day` | VARCHAR(10) | DEFAULT 'monday' | |
| `report_time` | TIME | DEFAULT '08:00:00' | |
| `report_skills` | TEXT[] | NULLABLE | NULL = all skills |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() | |

### `skills`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK, DEFAULT gen_random_uuid() | |
| `user_id` | UUID | REFERENCES users(id) ON DELETE CASCADE | |
| `name` | VARCHAR(100) | NOT NULL | e.g. "React", "IELTS" |
| `color` | VARCHAR(7) | DEFAULT '#888888' | Hex UI color |
| `is_deleted` | BOOLEAN | DEFAULT false | Soft delete |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `sync_status` | VARCHAR(20) | DEFAULT 'pending' | pending / synced |

### `logs`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK, DEFAULT gen_random_uuid() | UUID for offline creation |
| `user_id` | UUID | REFERENCES users(id) ON DELETE CASCADE | |
| `skill_id` | UUID | REFERENCES skills(id) NULLABLE | NULL if skill deleted |
| `skill_name` | VARCHAR(100) | NOT NULL | Denormalised — safe offline |
| `skill_color` | VARCHAR(7) | DEFAULT '#888888' | Denormalised — safe offline |
| `hours` | DECIMAL(4,1) | NOT NULL, CHECK (hours > 0 AND hours <= 24) | e.g. 2.5, 0.5 |
| `notes` | TEXT | NULLABLE | Optional |
| `date` | DATE | NOT NULL | Date of session |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() | Last-write-wins key |
| `sync_status` | VARCHAR(20) | DEFAULT 'pending' | pending / synced / conflict |

### `goals`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK, DEFAULT gen_random_uuid() | |
| `user_id` | UUID | REFERENCES users(id) ON DELETE CASCADE | |
| `skill_id` | UUID | REFERENCES skills(id) NULLABLE | |
| `title` | VARCHAR(200) | NOT NULL | e.g. "React mastery" |
| `skill_name` | VARCHAR(100) | NULLABLE | Denormalised |
| `target_type` | VARCHAR(10) | NOT NULL CHECK IN ('hours','days') | |
| `target_value` | INTEGER | NOT NULL CHECK > 0 | e.g. 50 or 30 |
| `deadline` | DATE | NULLABLE | |
| `is_completed` | BOOLEAN | DEFAULT false | |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `sync_status` | VARCHAR(20) | DEFAULT 'pending' | |

### `streaks`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `user_id` | UUID | PK, REFERENCES users(id) | One row per user |
| `current_streak` | INTEGER | DEFAULT 0 NOT NULL | Consecutive active days |
| `longest_streak` | INTEGER | DEFAULT 0 NOT NULL | All-time best |
| `last_log_date` | DATE | NULLABLE | Detects streak break |
| `grace_used` | BOOLEAN | DEFAULT false | Grace day used today |
| `grace_days_remaining` | INTEGER | DEFAULT 1 | Resets on new streak start |
| `total_days_logged` | INTEGER | DEFAULT 0 | Cumulative |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() | |

### `feedback`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK, DEFAULT gen_random_uuid() | |
| `user_id` | UUID | REFERENCES users(id) NULLABLE | Anonymous allowed |
| `rating` | SMALLINT | CHECK BETWEEN 1 AND 5 | Star rating |
| `category` | VARCHAR(50) | NULLABLE | Bug report / Feature idea / etc. |
| `message` | TEXT | NOT NULL | Required |
| `app_version` | VARCHAR(20) | NULLABLE | e.g. "1.0.0" |
| `platform` | VARCHAR(10) | NULLABLE | android / ios |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | |

### `export_jobs` *(Phase 2)*

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | REFERENCES users(id) | |
| `format` | VARCHAR(10) | CHECK IN ('pdf','csv','json') | |
| `status` | VARCHAR(20) | DEFAULT 'pending' | pending / processing / done / failed |
| `file_url` | TEXT | NULLABLE | Download URL when done |
| `requested_at` | TIMESTAMPTZ | DEFAULT NOW() | |
| `completed_at` | TIMESTAMPTZ | NULLABLE | |

---

## 7. API Specification

**Base URL:** `https://api.skilltracker.app/api/v1`

All protected routes require: `Authorization: Bearer <jwt_token>`

### 7.1 Authentication

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/auth/register` | Register — name, email, password | No |
| `POST` | `/auth/login` | Login — returns JWT + user | No |
| `POST` | `/auth/forgot-password` | Send password reset email | No |
| `POST` | `/auth/reset-password` | Reset with token + new password | No |
| `POST` | `/auth/social` | Google / GitHub OAuth (Phase 2) | No |
| `GET` | `/auth/me` | Get current user profile | Yes |
| `PUT` | `/auth/me` | Update name or avatar_url | Yes |
| `DELETE` | `/auth/me` | Delete account + all data | Yes |

### 7.2 Settings

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/settings` | Get all user settings | Yes |
| `PUT` | `/settings` | Update any settings field(s) | Yes |
| `PUT` | `/settings/reminder` | Toggle reminder, change time | Yes |
| `PUT` | `/settings/report` | Change report schedule / skills | Yes |

### 7.3 Skills

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/skills` | List active skills | Yes |
| `POST` | `/skills` | Create skill (name, color) | Yes |
| `PUT` | `/skills/:id` | Update name or color | Yes |
| `DELETE` | `/skills/:id` | Soft-delete (`is_deleted = true`) | Yes |

### 7.4 Logs

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/logs` | List logs — supports `?date=`, `?skill_id=`, `?page=`, `?limit=` | Yes |
| `GET` | `/logs/today` | Logs for today only | Yes |
| `GET` | `/logs/summary` | `?range=week\|month\|all` — hours grouped by date | Yes |
| `GET` | `/logs/activity` | `?days=119` — activity level per day (0-4) for heatmap | Yes |
| `POST` | `/logs` | Create log (skill_id, skill_name, skill_color, hours, notes, date) | Yes |
| `PUT` | `/logs/:id` | Update hours, notes, date | Yes |
| `DELETE` | `/logs/:id` | Delete log | Yes |

### 7.5 Goals

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/goals` | List goals with `progress_pct` and `status` computed | Yes |
| `POST` | `/goals` | Create goal (title, skill_name, target_type, target_value, deadline) | Yes |
| `PUT` | `/goals/:id` | Update goal | Yes |
| `PATCH` | `/goals/:id/complete` | Mark goal as completed | Yes |
| `DELETE` | `/goals/:id` | Delete goal | Yes |

### 7.6 Streak

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/streak` | Current, longest, total days, grace remaining | Yes |
| `GET` | `/streak/calendar` | `?year=&month=` — day-level status for calendar grid | Yes |
| `GET` | `/streak/weekly` | This week's 7 days with hours per day | Yes |
| `POST` | `/streak/recalculate` | Rebuild streak from log history | Yes |
| `POST` | `/streak/grace` | Use a grace day | Yes |

### 7.7 Stats

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/stats/overview` | Total hrs, this week, consistency %, best streak | Yes |
| `GET` | `/stats/by-skill` | Hours per skill + % of total + level badge | Yes |
| `GET` | `/stats/consistency` | Consistency score (0-100) + percentile rank | Yes |

### 7.8 Reports

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/reports/weekly` | Send this week's report to user email now | Yes |
| `GET` | `/reports/history` | List past sent reports | Yes |

### 7.9 Export

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/export?format=pdf` | PDF progress report | Yes |
| `GET` | `/export?format=csv` | CSV of all logs | Yes |
| `GET` | `/export?format=json` | Full JSON backup | Yes |

### 7.10 Feedback

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/feedback` | Submit (rating, category, message, app_version, platform) | Optional |

### 7.11 Sync

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/sync` | Batch upsert all pending records (logs + skills + goals + settings) | Yes |
| `GET` | `/sync/pull` | `?since=<timestamp>` — all server records updated after timestamp | Yes |

---

## 8. UX Flows

### 8.1 App Entry

```
App launch
  └─ JWT in AsyncStorage?
       ├─ Yes + valid → Home
       └─ No / expired
              ├─ First launch → Onboarding (4 slides) → Login
              └─ Returning   → Login
```

### 8.2 Log Session — Critical Path

```
FAB (+) tap → Log screen
  → Select skill chip  [or: + New → inline input → Add ✓ → new chip created]
  → Select time pill   [or: Custom → number input → Set ✓ → pill label updates]
  → Optional: type notes
  → "Save session →"
  → Write to SQLite (sync_status = pending)
  → Toast "🔥 Logged! Streak +1"
  → Navigate Home (800 ms)
  → Background: if online → sync engine flushes queue
```

### 8.3 Add Goal

```
Goals screen → "+ Add new goal" → sheet opens
  → Title (required) + Skill + Type toggle + Target (required) + Deadline
  → "Create goal →"
  → Validates: empty required fields turn red
  → New card appears at top of Goals list (0% progress)
  → Toast "Goal created!" → sheet closes
```

### 8.4 Streak Calendar Views

```
Streak Calendar
  ├─ Week    → 7 day cells (day name + date + hours) + summary stats
  ├─ Month   → Full calendar grid, current month
  ├─ 3 Months → Stacked Mar + Apr + May grids
  └─ Year    → 52-week heatmap + month labels
```

### 8.5 Sync Flow

```
App start     → GET /sync/pull?since=<last_pull>
              → merge server records (last-write-wins by updated_at)

User action   → write locally (sync_status = pending)
              → add to sync queue

NetInfo: online → SELECT * FROM logs/skills/goals WHERE sync_status = 'pending'
                → POST /sync (batch max 50)
                → on 200 → UPDATE sync_status = 'synced'
                → on 409 (conflict) → server updated_at wins
```

---

## 9. Offline-First Sync Architecture

### 9.1 Sync Status States

| Status | Meaning |
|---|---|
| `pending` | Created/updated locally, not yet sent to server |
| `syncing` | Request in-flight (prevents double-send) |
| `synced` | Server acknowledged |
| `conflict` | Server has newer `updated_at`; resolved last-write-wins |

### 9.2 Conflict Resolution

- Strategy: **Last Write Wins** on `updated_at` (UTC ISO 8601)
- `server.updated_at > local.updated_at` → server wins
- `local.updated_at > server.updated_at` → local wins; push to server
- All records use **UUID v4** — no integer collisions across offline devices
- **Never delete** local records after sync — only update `sync_status`

### 9.3 Required Fields on Every Syncable Record

```json
{
  "id": "uuid-v4",
  "user_id": "uuid-v4",
  "updated_at": "2026-05-02T14:30:00Z",
  "sync_status": "pending | syncing | synced | conflict"
}
```

**Tables requiring sync:** `users`, `skills`, `logs`, `goals`, `user_settings`

**Server-only (not synced):** `streaks` (computed server-side), `feedback`, `export_jobs`

---

## 10. Development Roadmap

| Phase | Timeline | Deliverables | Status |
|---|---|---|---|
| **Phase 1** | Week 1-2 | PostgreSQL schema + migrations; Express app; Auth + Settings endpoints; Skills + Logs CRUD | Planned |
| **Phase 2** | Week 3-4 | React Native CLI setup; Navigation stack; Onboarding + Login; Auth + Skills API integration | Planned |
| **Phase 3** | Week 5-6 | Home, Log, Progress screens with live API; SQLite local storage; Streak calculation | Planned |
| **Phase 4** | Week 7-8 | Goals screen + Add Goal sheet; Profile screen + Settings API; Dark mode; Offline sync engine | Planned |
| **Phase 5** | Week 9-10 | Streak Calendar (all 4 views); Export (PDF/CSV/JSON); Weekly report email; Feedback API | Planned |
| **Phase 6** | Week 11-12 | Push notifications (FCM/APNs); All Sessions screen; Polish + empty states; Android + iOS testing; Deploy | Planned |

---

## 11. Security Requirements

- Passwords: bcrypt hash, salt rounds 12 — never stored plain
- JWT: HS256, 7-day expiry, `SECRET_KEY` from `.env` only
- All routes except `/auth/*` and `POST /feedback` require valid JWT
- CORS: allow only mobile app bundle ID in production
- Rate limiting: `/auth/login` max 10 req/IP/15 min; `/auth/register` max 5 req/IP/hour
- SQL: parameterised statements only — no string concatenation
- All secrets in `.env` — never committed to VCS
- Token storage: AsyncStorage (MVP) → react-native-keychain (Phase 2)
- HTTPS/TLS 1.2+ for all API traffic
- Account deletion: purge all personal data within 30 days

---

## 12. Non-Functional Requirements

### Performance

| Metric | Target |
|---|---|
| Cold start time | < 2 s on mid-range Android (2 GB RAM) |
| SQLite write | < 50 ms — instant user feedback |
| API P95 response (GET) | < 300 ms under normal load |
| Sync batch size | <= 50 records per request |
| Activity grid render | < 100 ms for 119-day heatmap |

### Reliability

| Metric | Target |
|---|---|
| Offline capability | 100% read + write with no internet |
| Sync success rate | 99% synced within 60 s of reconnection |
| Data loss | Zero — SQLite is source of truth until synced |
| Crash-free sessions | > 99.5% Android and iOS |

### Compatibility

| | |
|---|---|
| Android | API 26+ (Android 8.0 Oreo and above) |
| iOS | iOS 14+ (Phase 2) |
| React Native | 0.73+ (New Architecture compatible) |
| Node.js | 20 LTS |
| PostgreSQL | 16+ |

---

## 13. Open Questions & Decisions

| ID | Topic | Question | Priority |
|---|---|---|---|
| OQ-1 | SQLite library | `react-native-sqlite-storage` vs `op-sqlite` vs `expo-sqlite` — CLI compatibility | High |
| OQ-2 | Grace day rule | Does grace day preserve streak count or reset to 1? | High |
| OQ-3 | Consistency score | Confirm formula: `(days_logged / days_since_join) * trend_multiplier` | High |
| OQ-4 | Push notifications | `react-native-push-notification` vs bare FCM/APNs | Medium |
| OQ-5 | PDF export | Server-side (puppeteer/pdfkit) or client-side (react-native-html-to-pdf)? | Medium |
| OQ-6 | Deployment | Railway vs self-hosted VPS | Medium |
| OQ-7 | Weekly report email | Resend vs SendGrid vs Nodemailer + SMTP | Medium |
| OQ-8 | Avatar upload | S3 vs Cloudinary for profile photo storage | Low |
| OQ-9 | Google/GitHub OAuth | Confirm provider SDK choices for Phase 2 | Low |
| OQ-10 | Streak recalculate | Auto-trigger on login? On sync? Manual only? | Medium |

---

*SkillTracker PRD v1.1.0 — Aminul Karim — May 2026*
*Confidential — development team only.*
