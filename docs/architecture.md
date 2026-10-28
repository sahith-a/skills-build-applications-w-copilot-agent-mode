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

- Presentation tier decision: Keep React components responsible for rendering API data and user-visible states; use the shared API client for relative `/api` requests. Each capability view must expose loading, meaningful empty, and accessible error states.
- Logic tier decision: Keep HTTP routing and response envelopes in the Express router, with validation and capability behavior in `octofitService.ts`. Reject invalid input and missing references with deliberate 4xx responses rather than leaking Mongoose cast or validation errors as 500s.
- Data tier decision: Enforce required model fields and sensible numeric constraints in Mongoose, retain the default empty team-members array, and keep seed operations additive and idempotent without deleting unrelated records.
- Activity tracking approach: Validate the request shape and both object IDs, confirm the referenced user and team exist, then persist the submitted user, team, type, and minutes with server-computed points and occurrence time.
- Team management approach: Require a non-empty name, preserve the submitted name and mascot, and persist the provided member IDs; listed and created teams should expose their members consistently.
- Leaderboard approach: Compute each team's total from its activity points, order totals descending, and assign ranks starting at 1.
- Workout suggestions approach: Resolve the user by email, use their fitness goal and recent activity to prioritize a relevant workout, and include a brief reason. Return a deliberate 4xx when the email has no matching user.

## Test plan

Run the narrow contract suites first, then the builds. Contract failures identify behavior gaps in the tier named by the suite; build failures identify TypeScript or bundling errors. Baseline recorded on 2026-10-28:

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` - checks model validation and seed behavior. Baseline: 4 failures (required fields are not enforced for User, Team, Activity, or Workout); the seed/idempotency test passes.
- API command: `npm run test:api --prefix octofit-tracker/backend` - checks activity and team validation/CRUD, leaderboard totals/ranks, and user-specific workout suggestions. Baseline: 4 failures; activity/team creation return 501, leaderboard is empty, and workout suggestions do not provide the required rationale or user-specific behavior.
- Frontend command: `npm test --prefix octofit-tracker/frontend` - checks API-backed content, workout request parameters, loading, empty, and accessible error states. Baseline: 15 failures; all four views still render the implementation-pending placeholder.
- Backend build command: `npm run build --prefix octofit-tracker/backend` - checks TypeScript compilation. Baseline: passes.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` - checks TypeScript compilation and Vite production bundling. Baseline: passes.
