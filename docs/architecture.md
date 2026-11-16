# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

Record your plan here before editing code.

- **Presentation tier decision**: React components render activity, teams, leaderboard, and workouts pages. Each page fetches from the API client (`src/api/client.ts`) and displays four states: loading (with accessible text), empty (meaningful message), success (data list), error (alert). Use React Router for navigation and Bootstrap for styling.

- **Logic tier decision**: Express routes under `/api/` invoke explicit service functions. Services validate input, check foreign keys, compute derived fields (e.g., activity points), and return data or deliberate 4xx errors (never 500 from validation). Routes return JSON with consistent shape (e.g., `{ activity }`, `{ leaderboard: [...] }`).

- **Data tier decision**: Mongoose schemas define required fields with validation. User, Team, and Activity schemas enforce constraints. Seed script uses upserts to ensure stable demo records without clearing user data. Demo user fixture reconciles team membership when moved.

- **Activity tracking approach**: POST `/api/activities` accepts `userId`, `teamId`, `type`, `minutes`. Service validates both IDs exist (return 400 if not), computes `points` from type/minutes, stores with `occurredAt` timestamp. GET returns all activities.

- **Team management approach**: POST `/api/teams` accepts `name`, `mascot`, optional `members` array. Service validates `name` is present (return 400 if missing), stores name, mascot, and members. GET returns teams with members populated.

- **Leaderboard approach**: GET `/api/leaderboard` aggregates all activities by team, sums their points, sorts descending by total points, assigns 1-based rank. Response: `{ leaderboard: [{ teamName, totalPoints, rank }, ...] }`.

- **Workout suggestions approach**: GET `/api/workouts/suggestions/:userEmail` finds the user, retrieves their activities, filters seeded workouts by matching their fitness goal or recent activity type. Response includes title, focus, minutes, difficulty, and a `reason` string explaining why it matches (e.g., "matches your endurance goal"). Returns 404 if user not found.

## Test plan

Record the commands you run and what each failure means.

- **Data-tier command**: `npm run test:data --prefix octofit-tracker/backend`
  - **Validates**: Mongoose schemas enforce required fields (name, email, fitnessGoal on User; name, mascot on Team; type, user, team, minutes on Activity; focus, minutes, difficulty on Workout)
  - **Current state**: 4 failures (missing validators), 1 pass (seed idempotence works)
  - **Success criteria**: All 5 tests pass; schemas define field requirements

- **API command**: `npm run test:api --prefix octofit-tracker/backend`
  - **Validates**: HTTP endpoints return correct status codes (400 for invalid input, 201 for creation, 200 for read/list), response shapes match contract (activities array, teams with members, leaderboard with rank, suggestions with reason)
  - **Current state**: 4 failures (endpoints return 501 "not implemented"); tests expect 400/200 and proper response structures
  - **Success criteria**: All 4 tests pass; all 4 API capabilities work per contract

- **Frontend command**: `npm test --prefix octofit-tracker/frontend`
  - **Validates**: React components render activity, teams, leaderboard, and workouts. Each page handles loading state (accessible text visible), empty state (meaningful message), success (data rendered), and error (alert shown). Integration test verifies all pages work with seeded data.
  - **Current state**: 15 failures (API returns 501 on all endpoints); components can't hydrate with data
  - **Success criteria**: All 15 tests pass; pages render data and all state transitions are tested

- **Backend build command**: `npm run build --prefix octofit-tracker/backend`
  - **Validates**: TypeScript compiles without errors
  - **Success criteria**: Build exits 0; dist/ contains compiled routes, services, models

- **Frontend build command**: `npm run build --prefix octofit-tracker/frontend`
  - **Validates**: Vite builds and tree-shakes successfully
  - **Success criteria**: Build exits 0; dist/ contains optimized bundle
