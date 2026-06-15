# ATCHÉ Visuals — One-Time Setup

Two doors, same Supabase project as the rest of ATCHÉ — **project `aayigsbmmdolvnicxacs`** (the one every other ATCHÉ tool points to). Run everything below in that project's SQL Editor, not any other project.

- **Studio (you):** `atche-visuals-studio.html` — log in with your existing ATCHÉ account, start a session per client, upload their shots, get a link/QR.
- **Client gallery:** `atche-visuals.html?s=CODE` — they open this on their own phone, pick photos/reel, choose a package, pay via Cash App, download.

Run this once in **Supabase → SQL Editor** (project `aayigsbmmdolvnicxacs`) before using it tonight:

```sql
-- Sessions table
create table if not exists visuals_sessions (
  id uuid primary key default gen_random_uuid(),
  code text unique not null,
  label text,
  user_id uuid references auth.users not null,
  status text default 'active',
  package_name text,
  package_price numeric default 0,
  paid boolean default false,
  paid_at timestamptz,
  payment_requested boolean default false,
  payment_requested_at timestamptz,
  created_at timestamptz default now()
);

alter table visuals_sessions enable row level security;

-- If the table already existed without these columns, add them:
alter table visuals_sessions add column if not exists payment_requested boolean default false;
alter table visuals_sessions add column if not exists payment_requested_at timestamptz;

-- Optional contact (IG handle or phone) captured when you start a session,
-- so you can follow up with someone who didn't unlock/download that night.
alter table visuals_sessions add column if not exists contact text;

-- You (staff) can manage your own sessions, including flipping `paid`
drop policy if exists "owner_all" on visuals_sessions;
create policy "owner_all" on visuals_sessions
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- Clients (no login) can open a session by code
drop policy if exists "anon_read" on visuals_sessions;
create policy "anon_read" on visuals_sessions
  for select using (true);

-- IMPORTANT: clients can NOT mark themselves paid directly anymore.
-- Drop the old honor-system policy if it exists from a previous setup.
drop policy if exists "anon_mark_paid" on visuals_sessions;

-- Instead, clients call this function to flag "I sent payment" — it only
-- sets payment_requested, never `paid`. Only staff (via owner_all) can set
-- `paid = true`, which is what actually unlocks the download.
create or replace function request_payment_confirmation(p_code text, p_package_name text, p_package_price numeric)
returns void
language plpgsql
security definer
set search_path = public
as $$
begin
  update visuals_sessions
  set payment_requested = true,
      payment_requested_at = now(),
      package_name = p_package_name,
      package_price = p_package_price
  where code = p_code and status = 'active';
end;
$$;

grant execute on function request_payment_confirmation(text, text, numeric) to anon, authenticated;
```

Then **Storage** — create the bucket via SQL (faster than the UI, and works even if a bucket got created in the wrong project before):

```sql
-- Create the visuals bucket (public, so previews/downloads load without auth)
insert into storage.buckets (id, name, public)
values ('visuals', 'visuals', true)
on conflict (id) do update set public = true;

-- You can upload to the visuals bucket
drop policy if exists "auth_upload_visuals" on storage.objects;
create policy "auth_upload_visuals" on storage.objects
  for insert with check (bucket_id = 'visuals' and auth.uid() is not null);

-- Anyone can view files in the visuals bucket (clients need this)
drop policy if exists "public_read_visuals" on storage.objects;
create policy "public_read_visuals" on storage.objects
  for select using (bucket_id = 'visuals');

-- You can delete files in the visuals bucket (lets Studio remove a test/bad upload)
drop policy if exists "auth_delete_visuals" on storage.objects;
create policy "auth_delete_visuals" on storage.objects
  for delete using (bucket_id = 'visuals' and auth.uid() is not null);
```

If you'd rather use the UI for the bucket: Storage → New bucket → name it **`visuals`** → toggle **Public** on, then still run the two `create policy` statements above in SQL Editor.

Finally, the gallery's "Are you a creative?" card now lets a guest drop their Instagram handle right there and get added as a Circle lead — without filling out the full application. It writes into the same `circle_applications` table used by `circle-application.html` / `atche-studio.html`. Run this so the columns and permissions are in place:

```sql
-- Make sure circle_applications has the columns this and the Studio form use
alter table circle_applications add column if not exists track text;
alter table circle_applications add column if not exists interests jsonb;
alter table circle_applications add column if not exists referral text;

-- Anyone (no login) can submit a lead — same as the main application form
drop policy if exists "anon_insert_applications" on circle_applications;
create policy "anon_insert_applications" on circle_applications
  for insert with check (true);
```

## Following up with a guest after the fact

If someone doesn't unlock/download their gallery that night, the **contact** field you optionally fill in when creating their session (IG handle or phone) is the only way back to them — there's no automatic notification. Check the session list in Studio for anyone still "Awaiting" with a contact on file and follow up directly (a DM with their gallery link works well: `atche-visuals.html?s=CODE`).

Separately, anyone who taps **"Notify Me"** on the "Are you a creative?" card shows up as a new lead in `model-tracker.html` (Applicants Inbox, track = Circle) tagged with `ATCHÉ Visuals` and the session code — that's your sign to reach out about The Circle.

## How a night runs

1. Open **Studio** on your phone, sign in.
2. New client/performer → type a label (optional) → **Create Session**. You get a 5-character code + QR.
3. Upload their photos/reel right there — they appear in the gallery instantly.
4. Show them the QR (or text the link). They open it on their phone, pick what they want, choose a package, and pay via Cash App.
5. They tap **"I've sent payment via Cash App"** — this just flags the session as "awaiting confirmation," it does NOT unlock anything yet.
6. Back in Studio, that session shows a **Confirm Paid** button once you (or whoever's working) see the Cash App notification land. Tap it.
7. The moment you confirm, the client's gallery auto-unlocks the download — no refresh needed on their end.
8. Studio dashboard tracks tonight's paid total and pending sessions in real time.

## Watermarked previews (no free screenshots)

Every upload is automatically processed into two copies in the `visuals` bucket:

- `<code>/full/<filename>` — the clean, full-resolution original. Only fetched on download, after payment is confirmed.
- `<code>/preview/<filename>.jpg` — a downsized copy with an ATCHÉ Visuals watermark (diagonal repeating mark + corner brand mark), generated instantly in the browser when you upload. For videos, this is a watermarked poster frame, not the clip itself.

The client gallery only ever shows the `preview/` copy. So a screenshot of the gallery only captures a watermarked, lower-res image — not the deliverable. This is honest-friction, not a hard lock: the bucket is still public, so it's not airtight against someone determined to dig through dev tools. But it closes the "just screenshot it for free" gap, which is the realistic risk at these price points.

## AI-Enhanced Edit add-on (June 2026)

Clients can add a one-off "AI-Enhanced Edit" to their order — same idea as the example renders (private jet tarmac, designer luggage, champagne, etc.), but generated by you after the fact and dropped into their gallery as a bonus deliverable. It's an add-on chosen at checkout (bundled into the same Cash App payment as their package), not a separate purchase flow.

Run this once in **Supabase → SQL Editor** (project `aayigsbmmdolvnicxacs`):

```sql
-- New columns to track the AI-enhanced edit add-on per session
alter table visuals_sessions add column if not exists ai_edit_requested boolean default false;
alter table visuals_sessions add column if not exists ai_edit_requested_at timestamptz;
alter table visuals_sessions add column if not exists ai_edit_theme text;
alter table visuals_sessions add column if not exists ai_edit_photo text;
alter table visuals_sessions add column if not exists ai_edit_price numeric default 0;
alter table visuals_sessions add column if not exists ai_edit_delivered boolean default false;
alter table visuals_sessions add column if not exists ai_edit_delivered_at timestamptz;

-- Replace request_payment_confirmation so it can carry the optional AI edit
-- selection alongside the normal package. A plain 3-arg call (no AI edit)
-- still works -- the new params default to null/0.
drop function if exists request_payment_confirmation(text, text, numeric);
create or replace function request_payment_confirmation(
  p_code text,
  p_package_name text,
  p_package_price numeric,
  p_ai_edit_theme text default null,
  p_ai_edit_photo text default null,
  p_ai_edit_price numeric default null
)
returns void
language plpgsql
security definer
set search_path = public
as $$
begin
  update visuals_sessions
  set payment_requested = true,
      payment_requested_at = now(),
      package_name = p_package_name,
      package_price = p_package_price,
      ai_edit_requested = case when p_ai_edit_theme is not null then true else ai_edit_requested end,
      ai_edit_requested_at = case when p_ai_edit_theme is not null then now() else ai_edit_requested_at end,
      ai_edit_theme = coalesce(p_ai_edit_theme, ai_edit_theme),
      ai_edit_photo = coalesce(p_ai_edit_photo, ai_edit_photo),
      ai_edit_price = coalesce(p_ai_edit_price, ai_edit_price, 0)
  where code = p_code and status = 'active';
end;
$$;

grant execute on function request_payment_confirmation(text, text, numeric, text, text, numeric) to anon, authenticated;
```

No new storage policy is needed -- the AI-enhanced result is uploaded to `<code>/ai/<filename>` in the same `visuals` bucket, which the existing `auth_upload_visuals` / `public_read_visuals` policies already cover (they're scoped to the whole bucket, not specific subfolders).

### How it works
1. **Client side** (`atche-visuals.html`): at Step 1 (packages), an "AI-Enhanced Edit" add-on card lets them pick a theme (Private Jet Glam, Penthouse Night, Red Carpet) and one of their uploaded photos as the basis. Picking it adds its price to the total. When they tap "I've sent payment," the theme/photo/price ride along with the package info in one `request_payment_confirmation` call.
2. **Studio side** (`atche-visuals-studio.html`): once you tap **Confirm Paid**, if the session has `ai_edit_requested`, the active session card shows an "AI Edit owed" banner with the theme + which photo to use as the base. Generate the edit yourself (Higgsfield or similar), then use the **Upload AI Edit** control on that banner -- it drops the file into `<code>/ai/` and marks `ai_edit_delivered = true`.
3. **Client side again**: once delivered, the gallery shows a new "Your AI-Enhanced Edit" section with the result, downloadable like everything else.

## Note on the "paid" check

This is no longer honor-system. The client tapping "I've sent payment" does not unlock anything by itself — it only flags the session in Studio as **Confirm $X**. The download stays locked until staff taps **Confirm Paid** in Studio after seeing the Cash App notification land. The client's gallery polls automatically and unlocks within a few seconds of confirmation, with no refresh needed.

This means: a client cannot self-unlock by editing the page, replaying requests, etc. — the only path to `paid = true` is a row-level-security policy that requires the logged-in staff account's `auth.uid()`.

## Pilot feedback survey (June 2026)

A short, optional survey at `atche-visuals-feedback.html?s=CODE` — framed as pilot-launch feedback, with a "free juice next time" incentive for leaving contact info. Studio's **Send Feedback Request** action (on any session) texts/emails this link, mainly aimed at people who didn't end up purchasing, but works for anyone.

Run this once in **Supabase → SQL Editor** (project `aayigsbmmdolvnicxacs`):

```sql
-- Pilot feedback responses
create table if not exists visuals_feedback (
  id uuid primary key default gen_random_uuid(),
  session_code text,
  rating text,
  got_content boolean,
  reason text,
  comments text,
  contact text,
  created_at timestamptz default now()
);

alter table visuals_feedback enable row level security;

-- Anyone (no login) can submit feedback -- this is a public survey form
drop policy if exists "anon_insert_feedback" on visuals_feedback;
create policy "anon_insert_feedback" on visuals_feedback
  for insert with check (true);

-- Only logged-in staff can read responses
drop policy if exists "auth_read_feedback" on visuals_feedback;
create policy "auth_read_feedback" on visuals_feedback
  for select using (auth.uid() is not null);
```

`rating` is one of `loved` / `good` / `okay` / `meh`. `reason` (only set when `got_content` is false) is one of `price` / `cashapp` / `preview` / `time` / `other`. `session_code` and `contact` are both optional -- a client can submit anonymously with no code at all.
