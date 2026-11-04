# TEAM GHAITH — Central ID & License Management System

Arabic-first responsive web application built with React + TypeScript + Vite and Supabase. The original Card ID BUC editor is copied to `public/card-studio.html` with a minimal parent-message bridge after the original save action; the byte-for-byte untouched backup is at `docs/original/Card-ID-BUC-original.html`. The portal hosts the integrated copy at `/card-studio` and forwards the save payload into the signed-in member's `member_cards` table.

## Included

- Landing page, member email-OTP login, license activation and email verification.
- Member dashboard, original card editor iframe, cloud metadata sync, and device/session revocation.
- Admin dashboard, member and license listings, permanent/temporary license creation, limited activations, license status changes, activity log view.
- Database-backed RLS policies and Edge Functions for activation, license creation and the Super Admin emergency command.
- `MØX//OMEGA` emergency lock: server-side hash check + database Super Admin authorization; blocks member RLS access and revokes registered member sessions while preserving records. Restore leaves old session IDs revoked; members must sign in again.
- Netlify SPA routing configuration and deployment instructions.

## Local development

Requirements: Node.js 20+, npm, Supabase project and Supabase CLI for backend deployment.

```bash
cp .env.example .env
# Fill VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY
npm install
npm run dev
```

## Connected Supabase project (provisioned 2026-11-04)

The live Supabase project has already been created and provisioned by the connected Supabase account:

- Project ref: `yebbcfgcdfavnotuaviv`
- API URL: `https://yebbcfgcdfavnotuaviv.supabase.co`
- Region: `eu-central-1`
- Status at last verification: `ACTIVE_HEALTHY`
- Database: PostgreSQL 17
- Public tables: 13; all 13 have RLS enabled
- Applied migrations: initial schema, activation finalization RPC, OMEGA/security hardening, public emergency-status restriction, and one-time Super Admin bootstrap
- Deployed Edge Functions: `request-activation-otp`, `finalize-activation`, `create-license`, `omega-emergency`, `bootstrap-first-super-admin`

The Vercel project `team-ghaith-central-system` has also been created and its public Supabase URL and publishable key have been added to Production, Preview, and Development environment variables. The Vercel project is not yet deployed because the source repository has not been connected and local dependency installation timed out.

For a fresh environment, run all migration files in ascending order. The fifth migration seeds a one-time bootstrap email hash and a single-use initial administrator license. The bootstrap function grants Super Admin only to the matching verified email and deletes the bootstrap hash after successful use. The initial license code is provided privately in the ChatGPT conversation, not stored in this repository.

## Auth email and first administrator

The activation and login flows use Supabase email OTP. Before production use, configure Authentication → Email templates so the sign-in template includes `{{ .Token }}`, configure a verified SMTP sender (for example Resend), and set OTP expiry / rate limits. The Supabase default SMTP setup has not yet been proven to deliver production mail.

The first Super Admin must activate the one-time initial license using the same verified email address as the connected Vercel account. On the first authenticated refresh after activation, the app calls `bootstrap-first-super-admin`; the server-side function checks the one-time hashed email allowlist and grants the role only if no Super Admin exists. Later accounts cannot reuse this bootstrap path.

The production app requires these public build variables (already configured on the Vercel project):

```text
VITE_SUPABASE_URL=https://yebbcfgcdfavnotuaviv.supabase.co
VITE_SUPABASE_ANON_KEY=YOUR_SUPABASE_PUBLISHABLE_KEY
```

Never place a Supabase secret / service-role key in a `VITE_` variable or frontend code. The deployed Edge Functions use Supabase's server-side environment variables.

## Security notes and remaining production setup

- The Supabase SQL is intended for a fresh project. Do not blindly run it on a populated project with conflicting tables or policies.
- RLS is the security boundary. The browser anon key is public by design; the service-role key is server-only.
- `MØX//OMEGA` is accepted only by the `omega-emergency` Edge Function. The database function independently checks the caller's `super_admin` role. The command is hashed and is not written to database logs.
- During lock, member-facing table/storage policies deny access. Super Admin access remains available for restoration. Registered member sessions/devices are revoked; restoring services does not reactivate old session IDs.
- The original card editor's own localStorage save remains for backwards compatibility. When embedded in the portal, clicking its save button also sends the card payload to the authenticated cloud table. The database trigger overwrites the editable ID-number slot with the server-issued member number, so a member cannot change that protected field in a forged payload. Existing old browser-only cards are not automatically migrated; they must be opened and saved while signed in.
- Card image data currently travels with the editor's card JSON payload. For large production uploads, move image blobs into the private `member-assets` Storage bucket and keep only object paths in `card_data`.
- This repository does not contain live Supabase/Netlify/Resend credentials. No public Vercel deployment, live SMTP delivery, production RLS penetration test, or real account end-to-end test is claimed until the repository is connected, the site is deployed, SMTP is configured, and a real member activation/login is exercised.

## Useful checks

```bash
npm run typecheck
npm run build
```

Before production launch, test at minimum: valid/invalid/expired/revoked license, OTP delivery and expiry, activation limit concurrency, member A cannot read member B's profile/cards/assets, admin vs viewer actions, device revocation, emergency lock while a member is already logged in, restoration requiring a fresh login, and mobile card export/print.

## Project structure

- `src/` — React app and portal UI.
- `public/card-studio.html` — byte-preserved original editor plus a minimal postMessage save bridge.
- `supabase/migrations/` — schema, RLS, session and emergency functions.
- `supabase/functions/` — activation, license creation, emergency endpoint.
- `docs/` — original audit and supplied requirements.
