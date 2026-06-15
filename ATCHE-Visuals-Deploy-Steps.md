# ATCHÉ Visuals — What's Left To Run

Everything below is the only thing standing between tonight's code and live. Two steps: SQL (one-time, only if you haven't already run the AI Edit + feedback blocks) and a git push.

## 1. SQL — only if you haven't run this yet

If you already ran the **"AI-Enhanced Edit add-on (June 2026)"** SQL block from `ATCHE-Visuals-Setup.md` (the one that adds `ai_edit_*` columns and replaces `request_payment_confirmation`), that part's done — nothing new needed for it.

New this round: the **"Pilot feedback survey (June 2026)"** SQL block in `ATCHE-Visuals-Setup.md` — adds the `visuals_feedback` table + RLS policies that the new feedback page writes to. Open **Supabase → SQL Editor** (project `aayigsbmmdolvnicxacs`) and run that block if you haven't already.

## 2. Push everything to GitHub → Vercel auto-deploys

From your ATCHÉ project folder in Terminal:

```
cd ~/Documents/Claude/Projects/ATCHÉ
git add .
git commit -m "AI Edit prompts + copy button, pilot feedback survey + Send Feedback Request"
git push
```

Live in ~60 seconds. This push covers:

- **Client gallery** (`atche-visuals.html`) — unchanged this round (AI-Enhanced Edit add-on already live).
- **Studio** (`atche-visuals-studio.html`) — the "AI Edit owed" banner now shows the full non-destructive/editorial generation prompt for the session's theme (Private Jet Glam / Penthouse Night / Red Carpet) with a **Copy Prompt** button so you can paste it straight into Higgsfield. Every unpaid session now also gets a **Send Feedback** pill that texts/emails (or copies, if no contact on file) a short pilot-feedback survey link with a free-juice incentive.
- **New file** (`atche-visuals-feedback.html`) — the survey page itself: 60-second pilot feedback (rating, did-they-get-their-content, reason if not, comments, optional contact for the free juice), styled to match the rest of ATCHÉ Visuals. Reads `?s=CODE` if present and writes to the new `visuals_feedback` table.

## After it's live — quick smoke test

- Open Studio, start a test session, upload a photo.
- On the client gallery, toggle the AI-Enhanced Edit add-on, confirm the total/QR updates, tap "I've sent payment."
- Back in Studio, tap **Open →** on that session — it should scroll straight to a flashing **Confirm Paid** banner, not the QR.
- Tap **Confirm Paid** — gold "AI Edit owed" banner should appear with the theme + base photo + the full generation prompt. Tap **Copy Prompt** — confirm it copies and the button briefly shows "✓ Copied".
- Click a thumbnail — full-size lightbox should open; scroll the grid if there are more than ~8 items.
- Close the session, then use **Delete** to remove it — confirm it disappears from the list.
- Start another test session, **don't** tap "I've sent payment" on the client side — confirm a "Mark Paid" pill (not pulsing) appears for it in the session list, and that tapping it prompts for package + price, then unlocks the download.
- Tap **Notify Client** on a session with nothing uploaded — confirm you get the "nothing uploaded yet" warning before it sends.
- On that same unpaid session, tap **Send Feedback** — confirm the prompt, then check it opens a text/email (or copies the message) with a link to `atche-visuals-feedback.html?s=CODE`.
- Open that feedback link directly — fill it out (try both "Yes" and "No" on the content question to see the reason picker appear), submit, and confirm the "Thank you" screen shows. Check the `visuals_feedback` table in Supabase for the new row.
