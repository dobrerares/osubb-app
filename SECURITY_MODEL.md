# OSUBB App — Threat Model

Scope: this repository. No separate infrastructure repository: hosting, CI/CD and backend configuration live in this repo (`supabase/config.toml`, `.github/workflows/`, `app/src/pwa/_headers.template`). Sources: README, CLAUDE.md, ADR-0001/0003/0010, `docs/architecture-data-flow.md`, `docs/backend/conventions.md`, and the code. Facts marked **(user)** were given by the repository owner in the 2026-11-20 threat-model session; everything else is read from the code and docs.

## System overview and trust boundaries

Internal app for the OSUBB student organization (UBB Cluj): Task Tracker with gamified points, Calendar/RSVP, announcements, deals, notifications, Web Push, member/role management, role evaluations (Portofoliu Interne, Drept de Vot).

- **Browser (untrusted)** → Supabase PostgREST (`public` schema only) with publishable key + Member JWT. Authorization is RLS + `security definer` commands in `private` that read the actor's live level.
- **Supabase Auth** (magic link / 6-digit OTP, `enable_signup = false`, `jwt_expiry = 3600`) → `public.custom_access_token_hook` stamps org claims only for an `activ` profile.
- **Edge Functions** (Deno, `supabase/functions/`) hold the secret key (`sb_secret_…`), which bypasses RLS.
- **pg_cron → pg_net → Edge Functions** (`send-push`, `send-digest`) authenticated by the secret key on `apikey`.
- **Third parties**: Resend (email + signed webhook), Web Push services (VAPID), Cloudflare Pages (Direct Upload from GitHub Actions only).
- **CI/CD**: GitHub Actions deploys migrations, functions and web to staging on merge; production via reviewer-gated Release workflow.

## Components

| Component                | Code                                                                                                                                                   |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Web client**           | `app/src/` (React PWA, service worker `app/src/pwa/`, Formatted Text reader/editor `app/src/components/formatted-text/`)                               |
| **Database & RLS**       | `supabase/migrations/` (tables, policies, `public` RPC wrappers, `private` commands, points engine, evaluations, member deletion), `supabase/seed.sql` |
| **Auth**                 | `supabase/config.toml` `[auth]`, `custom_access_token_hook`, `supabase/templates/`, login screens `app/src/screens/login`, `app/src/lib/auth*.ts`      |
| **Invitation functions** | `supabase/functions/invite-member`, `reinvite-member`, `send-invitations`, `csv-import`, `request-invitation`, `_shared/`                              |
| **Notification workers** | `supabase/functions/send-push`, `send-digest`, `resend-webhook`                                                                                        |
| **CI/CD & hosting**      | `.github/workflows/`, `.github/actions/`, `scripts/`, `app/src/pwa/_headers.template`                                                                  |

## Entry points and who can reach them

- PostgREST `public` tables/RPCs: any holder of the publishable key; data requires a JWT with membership claims (`auth_is_member()` / `auth_level()`).
- Unauthenticated Edge Functions (`verify_jwt = false`): `request-invitation` (origin allow-list + DB throttle), `resend-webhook` (Svix HMAC), `send-push` and `send-digest` (secret key, constant-time compare).
- JWT-gated Edge Functions (`invite-member`, `reinvite-member`, `send-invitations`, `csv-import`): function checks level ≥ 6 from the database.
- Auth endpoints: magic-link/OTP requests, redirect allow-list.
- Service worker / push payloads; Formatted Text (Markdown) rendered from Member-authored content.
- CI: pull requests (previews deploy from same-repo branches only).

## Assets

**(user)** All assets matter equally; no single asset is prioritized:

- Member personal data (emails, university, profile, push tokens).
- Points, Task evaluations, role evaluations and Drept de Vot (voting right) integrity.
- Leadership powers: role/status changes, invitations, member deletion, org settings.
- Secrets: Supabase secret key, Resend API key and webhook secret, VAPID private key, Cloudflare and Supabase access tokens in GitHub Environments.

## Attackers that matter

- **Unauthenticated internet attacker** — reaches the unauthenticated Edge Functions and Auth endpoints. Public sign-up is disabled, so accounts exist only by invitation (from code/config).
- **Invited, non-BC Member** (recruits up to BCE; a few hundred students) — trying to escalate to BC powers, read other Members' private data, or forge points/evaluations.
- **Deactivated Member or signed-in identity without a profile** — must see nothing.
- **Malicious content author** — Member-authored Formatted Text, links, names reaching other Members' browsers or notifications.
- **(user)** **Attacker holding a compromised BC or Moderator account** (stolen session, hijacked mailbox or magic link, phished OTP). BC and the Moderator are trusted people, but their accounts may be compromised, so an attacker acting with level-6/Moderator authority is in scope: what that session can reach beyond the intended powers (secrets, the secret key, other Members' sessions, bulk export, irreversible destruction, persistence after the account is recovered), and every path that helps take over such an account (XSS in content BC reads, open redirects, token leakage, weak session revocation).

## Existing controls

Deny-by-default RLS on every table; membership predicate on every policy; `security definer` with `search_path = ''`; execute grants revoked and re-granted per role; commands read live level and lock rows; CORS origin allow-list; DB throttle on `request-invitation`; Svix HMAC on `resend-webhook`; constant-time secret-key checks; CSP (report-only per #770) and hosting headers; pinned Deno deps; gitleaks + CI gates; reviewer-gated production Release.

## Out of scope / accepted risk

- **(user)** Nothing in the repository is out of scope (staging, previews, mockup/, docs/ included).
- **(user)** Superseded 2026-11-20: BC/Moderator were first called trusted insiders; the owner then ruled that their accounts may be compromised, so issues requiring BC/Moderator privileges are **not** accepted risk.
- **(user)** Accepted risk: a compromised BC/Moderator account redirecting another Member's Email Digest by editing that Member's profile email (ruled not relevant 2026-11-20).
- Documented design decision (ADR-0003, not restated by the user): stale JWT claims for up to one hour after deactivation/demotion affect reads only.

## Finding tags

- `Component` — one of the component names above.
- `CWE`, `OWASP`, `Data` — see tag kinds.

## Open questions

- Is CSP still report-only in production, and is that accepted?
- Which production secrets are set (VAPID, Vault rows) — affects reachability of push findings.
