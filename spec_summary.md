# NutriSnap — Project Spec Summary

> Calorie & macro tracking app powered by AI photo analysis (Cal AI clone)

---

## 1. Product & Users

| Dimension | Decision |
|---|---|
| **Problem** | Users want an effortless way to track calories and macros — just snap a photo of their meal instead of manually searching food databases. |
| **Target user** | Health-conscious individuals who want to lose, maintain, or gain weight with minimal friction. |
| **Core value prop** | Photo → instant nutritional breakdown, personalized daily goals, zero manual data entry. |
| **Platforms** | **Android first**, iOS fast-follow. Built with Expo / React Native (single codebase). |
| **Language** | English only (v1). |
| **Monetization** | None in v1 — completely free. |

---

## 2. Core User Journeys

### First-time user flow
```
Onboarding (no auth required)
  → Collect: age, gender, height, weight, goal weight, activity level, goal (lose/maintain/gain)
  → Store answers in AsyncStorage locally
  → Auth wall (Clerk: Google + Apple sign-in)
  → On auth success: persist onboarding data to Neon Postgres
  → Land on Home screen
```

### Returning user flow
```
App open → Clerk checks session → Home screen (today's meals + daily progress)
```

### Meal logging flow (photo)
```
Tap "Scan/Add" tab
  → Camera opens (expo-camera) OR pick from gallery
  → Photo uploads to ImageKit
  → Trigger.dev job: fetch image from ImageKit URL → send to Gemini Flash → parse response
  → User sees loading screen with animation/tips (3–8 sec)
  → Result screen: individual food items with calories, protein, carbs, fat breakdown
  → Meal type auto-assigned by time of day (editable)
  → User can edit/correct any field per item
  → Save → persisted to DB, photo URL stored permanently
  → Return to Home screen (updated)
```

### Meal logging flow (manual text)
```
Tap "Scan/Add" tab → switch to text input
  → User types food description (e.g. "chicken breast 200g with rice")
  → Description sent to Gemini Flash via Trigger.dev for nutritional estimation
  → Same result/edit/save flow as photo
```

### Water tracking
```
Home screen: tap to add a glass of water → daily total increments
```

---

## 3. Navigation Structure

**Bottom tab bar (3 tabs):**

| Tab | Screen | Description |
|---|---|---|
| 🏠 Home | Today's Log | Today's meals (swipe to navigate previous days), daily calorie/macro progress ring, water tracker, streak counter |
| 📸 Scan/Add | Camera / Text Input | Take photo, pick from gallery, or type a food description |
| 👤 Profile | Profile & Settings | Edit all onboarding fields, change units (metric/imperial), account management (sign out, delete account) |

---

## 4. Data Model (Core Entities)

### `users`
| Field | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| clerk_id | string (unique) | From Clerk |
| email | string | From Clerk |
| name | string | From Clerk |
| age | integer | |
| gender | enum | male / female / other |
| height_cm | float | Stored in cm, converted on display |
| weight_kg | float | Stored in kg, converted on display |
| goal_weight_kg | float | |
| activity_level | enum | sedentary / lightly_active / active / very_active |
| goal | enum | lose / maintain / gain |
| daily_calorie_target | integer | Calculated via Mifflin-St Jeor + activity multiplier |
| daily_protein_target | float | Derived from calorie target |
| daily_carbs_target | float | Derived from calorie target |
| daily_fat_target | float | Derived from calorie target |
| preferred_units | enum | metric / imperial |
| created_at | timestamp | |
| updated_at | timestamp | |

### `meals`
| Field | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK → users) | |
| meal_type | enum | breakfast / lunch / dinner / snack (auto-assigned by time, editable) |
| image_url | string (nullable) | ImageKit URL (null for text-only entries) |
| input_type | enum | photo / text |
| input_text | string (nullable) | Original text description (for text entries) |
| total_calories | float | Sum of all items |
| total_protein | float | Sum |
| total_carbs | float | Sum |
| total_fat | float | Sum |
| logged_at | timestamp | When the user logged it |
| created_at | timestamp | |
| updated_at | timestamp | |

### `meal_items`
| Field | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| meal_id | UUID (FK → meals) | |
| name | string | e.g. "Grilled chicken breast" |
| calories | float | |
| protein | float | |
| carbs | float | |
| fat | float | |
| quantity | string (nullable) | e.g. "200g", "1 cup" (as returned by AI) |

### `water_logs`
| Field | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK → users) | |
| glasses | integer | Number of glasses logged |
| date | date | One record per user per day |

### `streaks`
| Field | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK → users) | |
| current_streak | integer | Consecutive days with at least one meal logged |
| longest_streak | integer | All-time best |
| last_logged_date | date | To check continuity |

---

## 5. Architecture & Tech Stack

```
┌─────────────────────────────────────────────────┐
│              MOBILE APP (Expo / RN)             │
│  Expo Router · expo-camera · Clerk RN SDK       │
│  AsyncStorage (temp onboarding)                 │
└────────────────────┬────────────────────────────┘
                     │ REST API
                     ▼
┌─────────────────────────────────────────────────┐
│           BACKEND (Express.js / Node)           │
│  Clerk middleware (JWT verify)                  │
│  Drizzle ORM · Neon serverless driver           │
│  Routes: auth, users, meals, water, streaks     │
└────────┬───────────────────┬────────────────────┘
         │                   │
         ▼                   ▼
┌─────────────┐   ┌──────────────────────┐
│ Neon Postgres│   │   Trigger.dev        │
│  (Database)  │   │  - AI analysis job   │
│              │   │  - Auto-retry (1x)   │
└──────────────┘   │  - Fetches from      │
                   │    ImageKit → Gemini  │
                   └──────────┬───────────┘
                              │
                   ┌──────────┴───────────┐
                   │                      │
                   ▼                      ▼
            ┌───────────┐         ┌──────────────┐
            │  ImageKit  │         │ Gemini Flash │
            │  (Storage) │         │  (Google AI) │
            └───────────┘         └──────────────┘

Cross-cutting: Sentry (error tracking on both client & server)
```

### Tech stack (locked in — non-negotiable)

| Layer | Technology | Purpose |
|---|---|---|
| Mobile framework | Expo + React Native | Cross-platform app |
| Language | TypeScript | End-to-end type safety |
| Navigation | Expo Router | File-based routing |
| Auth | Clerk (Google + Apple) | Authentication & session management |
| Database | Neon Postgres | Serverless Postgres |
| ORM | Drizzle ORM | Type-safe queries |
| Backend | Express.js (Node) | REST API server |
| AI | Google Gemini Flash | Food image/text analysis |
| Background jobs | Trigger.dev | AI analysis jobs, retries |
| Image storage | ImageKit | Upload, store, optimize, transform |
| Error tracking | Sentry | Client + server monitoring |
| Camera | expo-camera | Photo capture |
| Dev workflow | Expo Go (dev) + EAS Build (prod) | Standard Expo pipeline |

---

## 6. TDEE & Goal Calculation

**Formula: Mifflin-St Jeor** (calculated client-side, stored in DB)

```
Male:   BMR = (10 × weight_kg) + (6.25 × height_cm) − (5 × age) + 5
Female: BMR = (10 × weight_kg) + (6.25 × height_cm) − (5 × age) − 161

TDEE = BMR × activity_multiplier
  sedentary:       1.2
  lightly_active:  1.375
  active:          1.55
  very_active:     1.725

Calorie target:
  lose:     TDEE − 500
  maintain: TDEE
  gain:     TDEE + 500

Macro split (default):
  protein: 30% of calories → grams = (calories × 0.30) / 4
  carbs:   40% of calories → grams = (calories × 0.40) / 4
  fat:     30% of calories → grams = (calories × 0.30) / 9
```

---

## 7. AI Analysis: Gemini Flash Prompt Strategy

The Trigger.dev job will send a structured prompt to Gemini Flash:

- For **photos**: send the ImageKit URL with a system prompt requesting JSON output with an array of food items, each containing name, estimated quantity, calories, protein, carbs, fat.
- For **text input**: send the description text with the same JSON-output prompt.
- **Response format**: enforce structured JSON so we can parse reliably.
- **On failure**: auto-retry once, then return error to client.

---

## 8. Failure & Edge Cases

| Scenario | Behavior |
|---|---|
| AI analysis fails | Auto-retry 1x in background, then show "Try Again" button |
| Unrecognizable photo (not food) | AI returns low-confidence / "not food" → show message asking to retake |
| Empty state (no meals today) | Friendly illustration + "Log your first meal!" CTA |
| Network offline | Show cached today's data if available, disable scan |
| Onboarding abandoned before auth | Data stays in AsyncStorage, resumed on next app open |
| User deletes account | Cascade delete all user data from DB, sign out from Clerk |

---

## 9. Design Direction

| Property | Choice |
|---|---|
| Theme | **Dark mode only** |
| Palette | Green/health-focused (greens, whites, dark backgrounds — similar to Cal AI) |
| Typography | Modern sans-serif (Inter or similar) |
| Feel | Clean, premium, health-focused |

---

## 10. Explicit v1 Scope

### ✅ IN scope
- Onboarding flow (6 fields + goal)
- Clerk auth (Google + Apple)
- Photo-based meal logging (expo-camera + gallery)
- Manual text-based meal logging
- AI analysis via Gemini Flash (Trigger.dev)
- Editable AI results before saving
- Per-meal food item breakdown
- Auto-assigned meal type (editable)
- Daily calorie & macro progress tracking with goals
- Day-by-day navigation on home screen
- Meal photo stored permanently (ImageKit)
- Edit and delete past meals
- Simple water tracking (tap to add glass)
- Daily streak counter
- Profile editing (all fields + account management)
- Dark mode, green palette
- Sentry error tracking
- Android-first, iOS fast-follow

### ❌ OUT of scope (v1)
- Barcode scanning
- Push notifications
- Monetization / paywall
- Multi-language / i18n
- Light mode / theme toggle
- Charts / stats / analytics screen
- Social features
- Meal planning / recipes
- Exercise tracking
- Food database lookup (USDA, etc.)
- Web app / dashboard
- Offline-first with sync

---

## 11. ASSUMPTIONS (where you didn't give a firm answer)

> These are defaults I've chosen. Flag any you disagree with.

| # | Assumption | Default chosen |
|---|---|---|
| A1 | **Gender options for "other"**: We won't use gender in the Mifflin-St Jeor formula for "other" | Use the average of male/female BMR formulas |
| A2 | **Macro split ratio** | 30% protein / 40% carbs / 30% fat (standard balanced split) |
| A3 | **Water glass size** | Fixed at 250ml (1 standard glass), not configurable in v1 |
| A4 | **Streak reset** | Streak resets to 0 if the user doesn't log any meal for a full calendar day |
| A5 | **Image compression before upload** | Compress/resize photos client-side before uploading to ImageKit (save bandwidth) |
| A6 | **Meal type auto-assign thresholds** | Breakfast: before 11am, Lunch: 11am–3pm, Dinner: 3pm–9pm, Snack: 9pm–midnight & midnight–6am |
| A7 | **Backend API style** | RESTful JSON API (not GraphQL or tRPC) |
| A8 | **Expo SDK version** | Latest stable (SDK 53 as of writing) — will confirm against docs |
| A9 | **Containerization** | Dockerfile for the Express backend, deployment target TBD |
| A10 | **Daily targets recalculate** | When user updates weight or any profile field, daily targets recalculate immediately |

---

## 12. OPEN RISKS / UNKNOWNS

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | **Gemini Flash accuracy on food photos** | Users may get frustrated if estimates are consistently off | Editable results + consider A/B testing with GPT-4o-mini later |
| R2 | **Gemini Flash rate limits / quotas** | Could throttle users during peak usage | Monitor usage, implement client-side rate limiting, have a queue in Trigger.dev |
| R3 | **ImageKit free tier limits** | May hit bandwidth/storage caps as users grow | Monitor usage, plan for paid tier or alternative |
| R4 | **Expo Go limitations with expo-camera** | Some camera features may not work in Expo Go, requiring dev builds | Test early, switch to dev builds if needed |
| R5 | **Clerk free tier limits** | 10,000 MAU on free tier — fine for launch but watch growth | Monitor, budget for Pro plan |
| R6 | **Backend deployment not decided** | Could delay launch if we hit platform-specific issues late | Containerize early, test on Railway or Render during dev |
| R7 | **Neon free tier cold starts** | First query after inactivity can be slow (500ms+) | Use Neon's serverless driver with connection pooling |
| R8 | **Food photo + text both through Gemini** | Text-based queries may be less accurate without visual context | Fine-tune the prompt, consider a food database fallback for v2 |
| R9 | **No offline support in v1** | Users without connectivity can't log meals | Accept for v1, plan offline queue for v2 |
| R10 | **Account deletion compliance** | Google Play requires functional account deletion | Implement cascade delete + Clerk account removal in v1 |

---

*Interview complete. All 7 batches resolved. Spec is locked.*
