# MUSSA — Website

Next.js 14 (App Router) + TypeScript + Tailwind CSS + Supabase.

## What's built

- **Home** — hero slideshow, features, upcoming events, About section
- **Login** (`/login`) — email/password, Google OAuth, phone/SMS OTP
- **Register** (`/register`) — email/password sign-up
- **Membership** (`/membership`) — registration form, gated behind login
- **Events** (`/events`, `/events/[slug]`) — listing + detail pages, free
  RSVP or M-Pesa-paid tickets, admin-editable via Supabase (no redeploy
  needed to add/change an event)
- **Gallery** (`/gallery`) — photos served from Supabase Storage, with
  real downloads
- **Pillars** (`/pillars`, `/pillars/[slug]`) — Innovation, IT, Environment,
  Mental Health, each showing its own upcoming events
- **Questions** (`/questions`) — anonymous or named question submission,
  linked from the Mental Health pillar page
- **Contact** (`/contact`) — pillar leader contacts, editable from admin
- **Admin dashboard** (`/admin`) — full internal CMS, see below

---

## Part 1 — Run it locally in VS Code

```bash
# in VS Code's integrated terminal, inside the unzipped project folder
cd mussa
npm install
cp .env.local.example .env.local
```

Now open `.env.local` in VS Code and fill in the Supabase values (see
Part 2 below to get them). Then:

```bash
npm run dev
```

Open http://localhost:3000. Email/password login and the rest of the site
will work as soon as Part 2 is done; Google login needs Part 3; M-Pesa
payments need Part 4.

---

## Part 2 — Set up Supabase (do this first)

1. Go to https://supabase.com, sign up free, and create a new project.
2. In the dashboard: **Project Settings > API** → copy the **Project URL**
   and **anon public key** into `.env.local` as `NEXT_PUBLIC_SUPABASE_URL`
   and `NEXT_PUBLIC_SUPABASE_ANON_KEY`.
3. Same page → copy the **service_role key** into `.env.local` as
   `SUPABASE_SERVICE_ROLE_KEY`. **Keep this secret** — never commit it,
   never put it in a `NEXT_PUBLIC_` variable.
4. Go to **SQL Editor > New query**, paste the entire contents of
   `supabase/schema.sql`, and click **Run**. This creates every table
   (memberships, events, event_registrations, questions) with the right
   security rules, seeds 3 sample events, and sets up the gallery photo
   storage bucket.
5. Go to **Storage > gallery** and upload a few photos to see the Gallery
   page populate.
6. To edit/add real events later: **Table Editor > events** — no redeploy
   needed, changes show up on the live site within about a minute. Or, once
   you're an admin (see below), use `/admin/events` instead.

### Becoming an admin

There's a full admin dashboard at `/admin` (Dashboard, Members, Attendance,
Finance, Upcoming Sessions, Posters & Gallery, Ticket Management, Anonymous
Questions, Leadership, Notifications, System Settings) — but no one is an
admin by default, on purpose. To make your own account the first admin:

1. Register a normal account on `/register` (or Google/phone login)
2. In Supabase's **SQL Editor**, run:
   ```sql
   insert into public.admins (user_id)
   select id from auth.users where email = 'you@example.com';
   ```
3. Log in and visit `/admin` — you now have full access, and can add other
   admins yourself from **System Settings** (they just need to have
   registered an account first).

**What each admin section actually does:**
- **Members / Attendance / Finance / Ticket Management** — real data,
  read/write to Supabase, fully working
- **Upcoming Sessions** — full create/edit/delete for events (this replaces
  needing to use Supabase's Table Editor manually)
- **Posters & Gallery** — real photo upload/delete straight to Supabase
  Storage
- **Anonymous Questions** — view submissions, post answers (which then
  appear publicly on `/questions`)
- **Leadership** — edits the same data shown on the public Contact page
- **Notifications** — an **in-app announcements list only**. It does not
  send push notifications, SMS, or email — that needs a separate provider
  (e.g. Africa's Talking for SMS, Resend for email) which isn't wired up
- **System Settings** — grant/revoke admin access by email

---

## Part 3 — Enable Google login (optional but in your notes)

1. In Supabase: **Authentication > Providers > Google** → toggle it on.
2. Follow Supabase's linked guide to create OAuth credentials in
   [Google Cloud Console](https://console.cloud.google.com/apis/credentials).
3. In Google Cloud Console, add these as **Authorized redirect URIs**:
   - `http://localhost:3000/auth/callback` (for local testing)
   - `https://<your-supabase-project-ref>.supabase.co/auth/v1/callback`
     (Supabase's own callback — required regardless of your domain)
4. Paste the Google Client ID and Secret into the Supabase provider screen
   and save.

Phone/SMS login is coded and ready the same way, but additionally needs a
paid SMS provider (Twilio, MessageBird, or Vonage) connected under
**Authentication > Providers > Phone** — real text messages won't send
until that's connected.

---

## Part 3.5 — Email confirmations (Resend) and WhatsApp follow-ups (Twilio)

Members now get an automatic email + WhatsApp message when they register
for an event (free or paid), a welcome email when they join as a member,
and a reminder the day before any event they're registered for.

**Email (Resend):**
1. Sign up free at https://resend.com
2. Get your API key from the dashboard, add to `.env.local`:
   ```
   RESEND_API_KEY=...
   EMAIL_FROM=MUSSA <onboarding@resend.dev>
   ```
3. `onboarding@resend.dev` works immediately for testing. For a real
   `@mussa.mmu.ac.ke`-style sender, verify your own domain under
   **Domains** in Resend first, then update `EMAIL_FROM`.

**WhatsApp (Twilio):**
1. Sign up free at https://twilio.com
2. Go to **Console > Messaging > Try it out > Send a WhatsApp message** to
   activate Twilio's free WhatsApp Sandbox
3. Copy your **Account SID** and **Auth Token** from the Twilio Console
   dashboard into `.env.local`:
   ```
   TWILIO_ACCOUNT_SID=...
   TWILIO_AUTH_TOKEN=...
   TWILIO_WHATSAPP_FROM=whatsapp:+14155238886
   ```
4. **Sandbox limitation**: each recipient must first send the sandbox's
   join code (shown on that Twilio page, e.g. "join happy-tiger") to the
   Twilio number via WhatsApp once, before your app can message them. This
   is fine for testing with your own phone, but not for real members yet.
5. **To message anyone without that join step** (what you actually want
   once live): apply for a **WhatsApp Business sender** through Twilio.
   This requires Meta Business verification — a real business/organization
   profile, reviewed by Meta — and takes their own review time, same as
   M-Pesa's Go Live process. Twilio's dashboard walks you through starting
   that application.

**Daily reminders**: once deployed to Vercel (Part 5), add a `CRON_SECRET`
value (any random string) to both `.env.local` and Vercel's environment
variables. `vercel.json` schedules `/api/cron/event-reminders` to run
once daily; Vercel automatically sends `CRON_SECRET` as a Bearer token, so
no one else can trigger it. This won't run locally on its own — you can
test it manually by visiting `http://localhost:3000/api/cron/event-reminders`
with an `Authorization: Bearer <your CRON_SECRET>` header (e.g. via a
tool like Postman or `curl`).

---

## Part 4 — Enable M-Pesa payments (for the paid Freshers' Mixer ticket)

This is genuinely the part I can't fully finish for you — it needs your
own Safaricom developer account, and going live requires Safaricom's own
business verification process outside anyone's control but yours.

1. Register at https://developer.safaricom.co.ke
2. Create a new app with the **"Lipa Na M-Pesa Sandbox"** product — this
   gives you a **Consumer Key** and **Consumer Secret** for testing with
   fake money first.
3. Fill `.env.local`:
   ```
   MPESA_ENV=sandbox
   MPESA_CONSUMER_KEY=...
   MPESA_CONSUMER_SECRET=...
   MPESA_SHORTCODE=174379          # Safaricom's public sandbox shortcode
   MPESA_PASSKEY=...               # published on the Daraja docs site
   MPESA_CALLBACK_URL=...          # see note below
   ```
4. **`MPESA_CALLBACK_URL` must be a public HTTPS URL** — Safaricom's
   servers call it directly, so `localhost` will never work. Two options:
   - **Quickest for local testing:** install [ngrok](https://ngrok.com),
     run `ngrok http 3000`, and use the `https://xxxx.ngrok.app/api/mpesa/callback`
     URL it gives you.
   - **For real use:** deploy first (Part 5), then use
     `https://your-site.vercel.app/api/mpesa/callback`.
5. Test with Safaricom's published sandbox test phone number and PIN
   (in their docs) before touching real money.
6. When ready for real payments: apply to **"Go Live"** in the Daraja
   portal (Safaricom reviews and approves this — it's not instant), get
   your production Shortcode/Passkey/keys, and switch `MPESA_ENV=production`
   with the production values, both in `.env.local` and in Vercel's
   environment variables.

---

## Part 5 — Deploy it live (Vercel)

Vercel is the natural host for Next.js (made by the same team) and has a
generous free tier.

**From VS Code:**
```bash
git init
git add .
git commit -m "Initial MUSSA site"
```

**Push to GitHub:**
1. Create a new empty repository at https://github.com/new (don't
   initialize it with a README — you already have one)
2. ```bash
   git remote add origin https://github.com/<your-username>/<repo-name>.git
   git branch -M main
   git push -u origin main
   ```

**Deploy on Vercel:**
1. Go to https://vercel.com, sign up with your GitHub account
2. Click **Add New > Project**, select your repo, click **Import**
3. Before clicking Deploy, expand **Environment Variables** and add every
   variable from your `.env.local` (Supabase keys, M-Pesa keys once you
   have them) — Vercel needs its own copy, it doesn't read your local file
4. Click **Deploy**. You'll get a live URL like `https://mussa.vercel.app`
   within a couple of minutes.

**After your first deploy**, go back and update these to your real domain:
- `MPESA_CALLBACK_URL` in Vercel's environment variables
- The Google OAuth redirect URI in Google Cloud Console (add the Vercel
  URL alongside localhost)
- Optionally connect a custom domain under the Vercel project's
  **Settings > Domains**

From then on, every `git push` to `main` auto-deploys the update.

---

## Quality checks to run locally

```bash
npm run lint
npx tsc --noEmit
npm run build
```

Manually verify at desktop, tablet, and mobile widths: navigation
(including the mobile menu), the login/register/membership flow, an event
registration (free and paid), and the gallery once photos are uploaded.

---

## Known placeholders

- **Hero slideshow photos** (`public/hero/slide-1/2/3.jpg`) and **event
  posters** (`public/events/poster-1/2/3.jpg`) are flat brand-colored
  illustrations — no real photography was available. Replace the files
  (same names) with real photos/posters whenever you have them, or just
  upload real ones and re-point events at them from `/admin/events`.
- **Contact details**: pillar leader names in the `pillar_leaders` table
  are all `TBD` — edit them from `/admin/leadership`.
- **Logo**: replace `public/logo.svg` with your real logo file.
- **Notifications** is an in-app list only — no real push/SMS/email
  delivery is wired up (see the admin section above).

## Project structure

```
middleware.ts          Refreshes Supabase session; gates /membership and /admin
supabase/schema.sql     Run once in Supabase's SQL editor — all tables + RLS
src/
  lib/
    supabase/
      client.ts           Supabase client — Client Components
      server.ts             Supabase client — Server Components/Actions
      serviceRole.ts        Bypasses RLS — M-Pesa callback + admin user lookup
      middleware.ts          Session refresh + route gating logic
    events.ts             Fetch events from Supabase; format for EventCard
    mpesa.ts              Daraja OAuth + STK Push helper
    admin.ts              Server-side admin access check
    email.ts              Resend email helper
    whatsapp.ts           Twilio WhatsApp message helper
    notifications.ts      Combined email+WhatsApp registration confirmations
  app/
    page.tsx               Home
    login/, register/      Auth pages + their server actions
    membership/             Gated membership form + its server action
    events/                 Listing, [slug] detail, free-RSVP action
    gallery/                 Photo grid from Supabase Storage
    pillars/                 Listing + [slug] detail (links events by pillar)
    questions/               Anonymous Q&A form + answered questions
    contact/                 Pillar leader contacts (from the database)
    auth/callback/           Completes Google OAuth redirect
    api/
      mpesa/stkpush/          Starts an M-Pesa payment
      mpesa/callback/          Safaricom posts the payment result here
      registrations/status/    Client polls this to detect payment success
    admin/                   Gated by middleware.ts + layout.tsx admin check
      layout.tsx               Sidebar + topbar shell
      page.tsx                 Dashboard — stats, charts, upcoming events
      members/                 Searchable member list
      events/                  Create/edit/delete events (+ [id]/edit/, new/)
      tickets/                 All registrations; manual "Mark Paid"
      gallery/                 Upload/delete gallery photos
      questions/               View + answer anonymous questions
      leadership/              Edit pillar leader contacts
      attendance/              Per-event check-in
      finance/                 Record income/expense, running totals
      notifications/           In-app announcements CRUD
      settings/                Grant/revoke admin access by email
  components/             Header, Hero, Footer, EventCard, forms, etc.
    admin/                  Sidebar, topbar, stat cards, charts, shared
                             DeleteButton and form components for the
                             admin section
  data/                   Static content: nav links, features, pillars, etc.
```
#   M U S S A  
 #   M U S S A  
 