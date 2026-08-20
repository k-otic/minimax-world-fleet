# MiniMax World Fleet Map

A community-built fleet map and registry for the **MiniMax** class — a 220 mm,
3D-printed, one-design radio-controlled yacht designed by **Brett McCormack** of
Castlerock Yachtworks in Dunedin, New Zealand.

**Live site:** https://minimax-world-fleet.vercel.app

> This is an unofficial community project, put together as a proposal for Brett and
> the class's admins. It is not affiliated with, or adopted by, Castlerock
> Yachtworks — that's a decision for Brett to make. If adopted, ownership and hosting
> would move to Castlerock directly.

## What it does

- **Fleet map** — every registered MiniMax plotted on a world map, clustered by
  region, filterable by build status (planning / building / sailing). Only country,
  region and locality are ever shown — never a precise address.
- **Self-service registration** — builders and owners add themselves in under a
  minute: no account creation, no password. Sign-in is a one-time email code.
- **Licence verification** — registrants can optionally add their STL order
  reference; class admins can confirm it against a real Castlerock licence from a
  dedicated review panel, and verified boats get a small mark on the map.
- **Class library** — the founding documents (the concept, the draft class rules) in
  one place, plus links to suppliers, the builders' Facebook group, and the
  electronics used.

## Tech stack

| Layer | Service |
|---|---|
| Hosting | [Vercel](https://vercel.com) — auto-deploys on every push to `main` |
| Source control | GitHub (this repo) |
| Database & auth | [Supabase](https://supabase.com) (Postgres + Row-Level Security + Auth) |
| Transactional email | [Brevo](https://www.brevo.com), as custom SMTP configured inside Supabase Auth — used specifically to get past Supabase's built-in sender limit (~2 emails/hour) |

There's no build step — `index.html` and `review.html` are plain static HTML/CSS/JS
(Leaflet for the map, the Supabase JS client for data). To preview locally, just
serve the folder statically, e.g. `python -m http.server` and open
`http://localhost:8000`.

The Supabase URL and anon key are hardcoded in both HTML files in plain sight —
that's intentional, not a leaked secret. Supabase's anon key is meant to be public;
the actual protection is Row-Level Security on the `boats` table and the
`is_class_admin()` / `set_verification()` RPCs, which check the caller's
authenticated identity server-side. Nothing sensitive is reachable with the anon key
alone (see Architecture notes below).

## Pages

- **`index.html`** — the public site: map, registration, login, class library, FAQ.
- **`review.html`** — admin-only panel. Lists pending registrations with an STL
  order reference and lets a class admin mark them verified/withdrawn. Reachable
  from `index.html`'s header once signed in with an admin email (see below).

## For class admins

1. Ask whoever maintains the Supabase project to add your email to the
   `class_admins` table (Table Editor → `class_admins`).
2. Sign in from the main site (top-right "Log In"). A **Review** link appears in the
   header automatically once you're recognised as an admin — that's the only way in,
   there's no separate admin URL to remember.
3. On the review panel, tick off registrations whose order reference matches a real
   licence, with an optional note.

## Architecture notes (for whoever maintains this next)

- **Sign-in uses one-time email codes, not clickable magic links.** Brevo rewrites
  every link in a transactional email for its own click-tracking, and that rewrite
  gets *followed* — by Brevo's own tracking pass, or by a mail client prefetching it
  — before the real recipient ever clicks it. Supabase's one-time link gets consumed
  by that first, silent visit, so the human arrives to an `otp_expired` error. Brevo
  has no per-sender setting to turn this off for transactional SMTP. If you ever
  touch the sign-in code, keep the OTP-code flow (`signInWithOtp` +
  `verifyOtp({ type: 'email' })`) rather than reintroducing `emailRedirectTo`.
- **The OTP code is 8 digits on this Supabase project**, not the more common 6 —
  `GOTRUE_MAILER_OTP_LENGTH` is configurable per-project (6–10 digits). Don't
  hardcode a digit count anywhere a code is typed or validated.
- Supabase's **"Confirm signup"** and **"Magic Link"** email templates (Authentication
  → Email Templates) must print `{{ .Token }}` and must **not** contain
  `{{ .ConfirmationURL }}` — leaving the link in alongside the code reintroduces the
  same failure.
- `public_fleet` and `fleet_stats` are public, read-only Postgres views used by the
  map and the stat counters — coordinates are rounded before they ever leave the
  database. The underlying `boats` table itself is RLS-protected and only reachable
  through an authenticated session (for your own row) or those views (for public
  data).
- The admin **Review** link on `index.html` is a discoverability convenience only —
  it calls the same `is_class_admin()` RPC that `review.html` itself enforces, so
  hiding or showing that link can never grant or block real access on its own.
- **The HTML `hidden` attribute only works via the browser's own lowest-priority
  stylesheet rule** (`[hidden]{display:none}`) — any author CSS class that sets
  `display` on the same element (`.btn-outline`, `.row`, etc.) silently overrides it,
  regardless of source order or specificity, because author rules always beat the
  user-agent stylesheet. Both HTML files now carry an explicit
  `[hidden]{display:none !important;}` rule near the top of `<style>` to guard
  against this — keep it if you add more conditionally-shown elements, and don't
  trust `element.hidden` reading `true` in the DOM as proof something is actually
  invisible on screen; check `getComputedStyle(el).display` too.
- **Data protection is via Row-Level Security, not column encryption.** Neither the
  EU/UK GDPR nor New Zealand's Privacy Act 2020 (Castlerock Yachtworks is
  NZ-based, many registrants are in the EU — both regimes apply) mandate encrypting
  stored personal data; both ask for security "reasonable"/"appropriate" to the
  risk. Verified live: `boats.email` is unreadable to anonymous requests
  (`permission denied for table boats`), and the public `public_fleet` view doesn't
  even have an `email` column (`column public_fleet.email does not exist`) — so it
  can never leak through the map or the stat counters. A user can delete their own
  `boats` row (right to erasure) via **Log In → Delete my entry** on `index.html`;
  this relies on an existing RLS policy permitting `delete` where
  `auth.uid() = user_id` — if that policy is ever removed, this feature needs it
  back to keep working.

## Changelog

Dated by when each change went live.

**2026-08-20 — Privacy: self-service deletion, and a fixed "hidden" bug.** Added a
short privacy note to the registration form, two FAQ entries covering what's
collected/why and which privacy laws apply (GDPR and NZ's Privacy Act 2020), and a
**Delete my entry** option for signed-in users — verified live, including that the
row is genuinely gone from the database afterwards, not just hidden client-side.
Also fixed a real bug caught while testing the admin link below: it was visible to
everyone, logged in or not, because a CSS class overrode the `hidden` attribute
(see Architecture notes).

**2026-08-20 — Admin review link.** The review panel had no link to it anywhere on
the public site. `index.html` now shows a "Review" link in the header for signed-in
class admins, using the same admin check the panel already enforces server-side.

**2026-08-20 — One-time sign-in codes, replacing magic links.** Fixed a bug where
every sign-in email (registration, login, and the admin panel) failed with
`otp_expired` — traced to Brevo's forced click-tracking consuming Supabase's
one-time links before recipients could click them. Switched the whole site to
email-a-code sign-in instead. Also fixed a related bug: the code input was capped at
6 characters, but this project issues 8-digit codes.

**2026-08-20 — Fixed the admin panel and the class library.** `review.html` carried
a leftover Content-Security-Policy tag from the tool that generated it, which didn't
allow requests to Supabase — every action in the admin panel failed with "Failed to
fetch". The two class-library PDFs pointed at a `docs/` folder that doesn't exist in
this repo, producing 404s. Both fixed.

**2026-08-20 — Full rewrite (v0.9.8-alpha).** Rebuilt `index.html`'s structure and
added an on-page self-check that explains, in plain language, exactly why the map or
database might not be loading (a stray security-policy tag, the wrong GitHub URL, a
host injecting its own policy) rather than failing silently.

**2026-08-19 — Admin review panel added.** First version of `review.html`, so class
admins can verify STL licences against submitted order references.

**2026-08-19 — Class library added.** The concept document and the draft class
rules, published as PDFs.

**2026-08-19 — Initial build.** First version of the fleet map published: `index.html`
with the map, registration form and login.
