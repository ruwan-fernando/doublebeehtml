# doubleBee — Setup & Maintenance Notes

Quick reference for how this app is hosted and connected, so you don't have to
remember it later.

## Hosting

- **Platform:** Netlify (static site — just the one `index.html` file)
- **How to update:** re-upload/redeploy `index.html` to Netlify whenever
  you get a new version of the file.

## Database (Supabase)

- **Project URL:** `https://icjvmsdbbymmudxxtcba.supabase.co`
- **Public (anon) key:** `sb_publishable_TK6j4J7TJZ9YsVbpv_NUPQ_cBIU52SD`
  - This key is safe to be visible in the HTML — it only allows what your
    Row Level Security (RLS) policy permits (see below). It is *not* a secret.
- **Dashboard:** [supabase.com/dashboard](https://supabase.com/dashboard) →
  sign in → select this project.

### Table: `app_state`

One row per user account. The whole app's data is stored as one JSON blob
per row.

```sql
create table if not exists public.app_state (
  user_id uuid primary key references auth.users(id) on delete cascade,
  data jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

alter table public.app_state enable row level security;

create policy "Users manage their own state"
  on public.app_state
  for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```

If this table is ever missing (e.g. new Supabase project), just re-run the
SQL above in **Supabase → SQL Editor**.

### Auth setting to know about

**Authentication → Providers → Email → "Confirm email"**
- If ON (Supabase default): new sign-ups must click a confirmation link in
  their email before they can log in.
- If OFF: sign-up logs you in immediately. Simpler for a single-owner app.

## How syncing works

- Every change in the app saves to `localStorage` on the device instantly
  (works even offline).
- ~0.7 seconds later, it also pushes to the Supabase `app_state` table for
  your logged-in account.
- Signing in with the same email/password on another device pulls that same
  row down, so all devices stay in sync.
- A small dot + label (top bar on mobile, sidebar on desktop) shows the sync
  status: *Syncing… / Synced / Sync error — saved locally*.
- If sync fails (e.g. no internet), nothing is lost — your local copy is
  still there and it will sync next time you're online and save something.

## ⚠️ Sync is not a backup — back up regularly

Sync means every device shares **one** copy of the data. That also means:
- A mistake (accidental delete, bad import) overwrites that one copy
  everywhere — there's no undo and no version history.
- If your Supabase project is ever removed for long-term inactivity
  (free-tier projects pause after 7 days idle, but don't get deleted just
  from pausing), you'd want a copy elsewhere anyway.

**What to do:** Open the app → **Backup & Restore** tab → **Download backup
(.json)**.

- Do this every week or two.
- Always do it right before anything risky: a bulk import, erasing data, or
  major changes.
- Save the file somewhere outside the app — Google Drive, Dropbox, email to
  yourself, etc.

## Quick troubleshooting

| Problem | Likely cause / fix |
|---|---|
| "Sync error" shown after login | No internet, or Supabase project is paused — wait a few seconds and try "Sync now" on the Backup & Restore tab. |
| Data missing after signing in on a new device | Make sure you're using the **exact same email**. Different email = different account = empty data. |
| Sign-up says "check your email" but you never got it | Check spam, or turn off "Confirm email" in Supabase auth settings (see above) and sign up again. |
| Want to start completely fresh | Backup & Restore tab → Erase all data (irreversible — export a backup first). |

## Useful links

- Supabase dashboard: https://supabase.com/dashboard
- Netlify dashboard: https://app.netlify.com
