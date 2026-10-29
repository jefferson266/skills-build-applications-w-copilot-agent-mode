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

- Presentation tier decision: Keep route views focused on rendering and user interaction. Load and submit data through the shared API client, and provide accessible loading, error, and empty states without calling `fetch` directly from components.
- Logic tier decision: Keep HTTP parsing and response status handling in Express routes, with validation, calculations, and database queries in the service layer. Return deliberate 4xx responses for invalid or unknown input, and do not add authentication in this exercise.
- Data tier decision: Define required fields and relationships in the four Mongoose models, using ObjectId references for users, teams, and activities. Seed representative users, teams, activities, and workouts with idempotent upserts that preserve unrelated records.
- Activity tracking approach: Validate activity type, positive minutes, and well-formed IDs for existing users and teams before writing. Store the user and team references, submitted activity details, server-computed points, and an occurrence timestamp; translate invalid IDs into a deliberate 4xx rather than exposing cast errors.
- Team management approach: Require a non-empty name and mascot, accept an optional list of existing user members, and return created and listed teams with their submitted names, mascots, and members.
- Leaderboard approach: Sum activity points by team in the logic tier, sort totals from highest to lowest, and return each team's name, total points, and 1-based rank.
- Workout suggestions approach: Find the user by email, use the user's fitness goal and recent activities to choose relevant seeded workouts, and include a reason for the recommendation. Return a deliberate 4xx when no user matches.

## Test plan

Run the contract checks in data, API, then presentation order; run both builds after the contracts pass. A failing contract identifies a behavior that must be fixed in its owning tier before proceeding.

- Data tier: `npm run test:data --prefix octofit-tracker/backend` checks required model fields, references, and repeatable seeding. Failures point to schemas or seed behavior.
- Logic tier: `npm run test:api --prefix octofit-tracker/backend` checks activity and team validation/creation, leaderboard aggregation, and activity-based workout suggestions. Failures point to route/service behavior or its data queries.
- Presentation tier: `npm test --prefix octofit-tracker/frontend` checks API-backed rendering, route-specific requests, and accessible loading, empty, and error states. Failures point to component behavior or shared-client integration.
- Backend build: `npm run build --prefix octofit-tracker/backend` runs the TypeScript check; failures indicate backend type or module errors.
- Frontend build: `npm run build --prefix octofit-tracker/frontend` runs the TypeScript check and Vite production build; failures indicate frontend type or bundling errors.
- Supplemental exercise guard: `node --test tests/exercise-validation.test.mjs` checks repository exercise metadata and validator behavior, not the three application tiers.

Baseline on 2026-10-29: data contracts had 4 failures and 1 pass; API contracts had 4 failures; frontend contracts had 15 failures. Both builds passed. The supplemental exercise guard had 36 passes and 4 failures involving the README badge contract, toolkit checkout pinning, and block-scalar validation cases; track these separately from application contract work.
