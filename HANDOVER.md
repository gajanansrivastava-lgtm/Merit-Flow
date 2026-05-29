# MeritFlow Project Handover

## Project Summary

MeritFlow is an internal performance review and appraisal MVP for Openhouse. It supports a strict three-level review workflow:

1. Employee self review
2. Manager evaluation
3. Founder/admin validation and finalization

The system uses the employee master CSV as the source of truth. No dummy employee data should be used.

## Current Stack

- Frontend: React + Vite
- Backend: Node.js + Express
- Database: Neon PostgreSQL
- Auth: Google OAuth + JWT
- Styling: plain CSS
- Icons: `lucide-react`
- Deployment target: Vercel

## Important Security Rules

- Only users already present in the `users` table can sign in with Google.
- Google OAuth does not auto-create users.
- Employee access is restricted to their own review.
- Employee self-review responses do not include manager/founder ratings.
- Managers can only access direct reports where `users.manager_id = logged-in manager id`.
- Admin/founder users can view all non-admin review subjects.
- Workflow transitions are enforced in the backend.
- Finalized reviews cannot be edited.

## Folder Structure

```text
api/
  index.js                    # Vercel serverless entrypoint
backend/
  scripts/
    db-summary.js             # Read-only DB summary
    generate-seed-sql.js      # Rebuild db/seed.sql from CSV
    run-sql.js                # Run SQL files against Neon
    set-password.js           # Legacy password utility
    set-teams.js              # Update manager_id from CSV Reporting To
  src/
    config/
    middleware/
    routes/
    services/
    utils/
db/
  schema.sql                  # Main schema
  seed.sql                    # Generated seed from CSV
  google-oauth.sql            # OAuth columns migration
  remove-test-users.sql       # Removes fake TEST user
  reset.sql                   # Dev reset only
frontend/
  src/
    components/
    contexts/
    pages/
    services/
```

## Environment Variables

Local `.env` should contain:

```env
DATABASE_URL=postgresql://...
JWT_SECRET=long-random-secret
JWT_EXPIRES_IN=8h
GOOGLE_CLIENT_ID=google-client-id
GOOGLE_CLIENT_SECRET=google-client-secret
GOOGLE_CALLBACK_URL=http://localhost:4000/api/auth/google/callback
CLIENT_ORIGIN=http://localhost:5173
PORT=4000
NODE_ENV=development
```

Vercel environment variables:

```env
DATABASE_URL=neon-pooled-connection-string
JWT_SECRET=long-random-secret
JWT_EXPIRES_IN=8h
GOOGLE_CLIENT_ID=google-client-id
GOOGLE_CLIENT_SECRET=google-client-secret
GOOGLE_CALLBACK_URL=https://your-vercel-app.vercel.app/api/auth/google/callback
CLIENT_ORIGIN=https://your-vercel-app.vercel.app
NODE_ENV=production
```

## Google OAuth Setup

In Google Cloud Console:

1. Create a Web OAuth Client ID.
2. Configure the OAuth consent screen.
3. Add authorized redirect URIs:

```text
http://localhost:4000/api/auth/google/callback
https://your-vercel-app.vercel.app/api/auth/google/callback
```

Users must sign in with an email that exists in the Neon `users` table.

## Database Setup

From the project root:

```powershell
npm run db:schema
npm run db:seed
npm run db:set-teams
npm run db:oauth
```

For a completely fresh development database only:

```powershell
npm run db:reset
npm run db:setup
npm run db:set-teams
npm run db:oauth
```

Warning: `db:reset` drops MeritFlow tables and types.

## Employee Data Source

The seed data is generated from:

```text
C:\Users\gajan\OneDrive\Desktop\OH Downs\Emp Master Sheet - Sheet1 (1).csv
```

Seed assumptions:

- `Co-Founder` and `Admin` job titles become `admin`.
- Any non-admin person named in `Reporting To` becomes `manager`.
- Everyone else becomes `employee`.
- `Reporting To` is resolved to `manager_id`.
- The fake `TEST` / `support@openhouse.in` row is excluded.

To regenerate seed SQL after CSV changes:

```powershell
npm run seed:sql
```

Then apply:

```powershell
npm run db:seed
npm run db:set-teams
```

## Current Known DB State After Cleanup

Last verified state:

- Employees: 39
- Managers: 8
- Admins: 3

Known teams:

- Manish Pal: 17
- Ashish Bibyan: 10
- Saurabh Makhariya: 6
- Rahool Sureka: 4
- Abhishek Singh Rathore: 3
- Ankit Khemka: 3
- Rajnish Prjapat: 2
- Akshit Chaudhary: 1
- Prashant Kumar: 1

The fake `TEST` direct report under Gajanan Srivastava was removed from Neon and excluded from regenerated seed SQL.

## Local Run

Install dependencies:

```powershell
npm install
```

Start frontend and backend:

```powershell
npm run dev
```

Open:

```text
http://localhost:5173
```

Health check:

```text
http://localhost:4000/api/health
```

## Build

```powershell
npm run build
```

The built frontend outputs to:

```text
dist/
```

## Vercel Deployment

Project settings:

```text
Framework Preset: Vite
Root Directory: .
Build Command: npm run build
Output Directory: dist
Install Command: npm install
```

`vercel.json` routes:

- `/api/*` to `api/index.js`
- all other routes to the React SPA

After deploy, test:

```text
https://your-vercel-app.vercel.app
https://your-vercel-app.vercel.app/api/health
```

## Core API Routes

Auth:

- `GET /api/auth/google`
- `GET /api/auth/google/callback`
- `GET /api/auth/me`

Employee:

- `GET /api/employee/my-review`
- `POST /api/employee/self-review`

Manager:

- `GET /api/manager/team`
- `GET /api/manager/team/:employeeId/review`
- `POST /api/manager/team/:employeeId/rate`

Admin:

- `GET /api/admin/employees`
- `GET /api/admin/dashboard`
- `POST /api/admin/reviews/:reviewId/override`
- `POST /api/admin/reviews/:reviewId/finalize`

## Review Workflow

```text
NOT_STARTED
SELF_SUBMITTED
MANAGER_REVIEWED
FOUNDER_REVIEWED
FINALIZED
```

Allowed transitions:

- Employee: `NOT_STARTED -> SELF_SUBMITTED`
- Manager: `SELF_SUBMITTED -> MANAGER_REVIEWED`
- Admin/founder: `MANAGER_REVIEWED -> FOUNDER_REVIEWED`
- Admin/founder: `FOUNDER_REVIEWED -> FINALIZED`

## Rating Formula

```text
final_rating = (0.5 * KRA) + (0.3 * Behavioral) + (0.2 * Overall)
```

Founder override ratings take precedence over manager ratings where provided.

Ratings are rounded to one decimal place. The backend calculates the value, and the database trigger recalculates it as a guardrail.

## Useful Scripts

```powershell
npm run dev              # Start backend and frontend
npm run build            # Build frontend
npm run db:schema        # Apply schema
npm run db:seed          # Apply generated seed
npm run db:set-teams     # Rebuild manager_id links from CSV
npm run db:oauth         # Add OAuth columns
npm run db:reset         # Drop MeritFlow DB objects, dev only
npm run seed:sql         # Regenerate db/seed.sql from CSV
node backend/scripts/db-summary.js
```

## Notes For Next Engineer

- Email/password login has been replaced by Google OAuth, but some password utility code remains for emergency/manual use.
- The app stores JWTs in `localStorage`.
- `DATABASE_URL` should use Neon’s pooled/serverless connection string on Vercel.
- The SSL warning from `pg` about `sslmode=require` is currently non-blocking.
- If teams disappear, run `npm run db:set-teams`.
- If a new employee appears in the CSV, regenerate seed SQL and apply seed/team scripts.
