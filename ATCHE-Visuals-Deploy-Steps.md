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
git commit -m "UX overhaul: flow reorder, custom modals, overflow menu, sign-out, skeleton states"
git push
```

Live in ~60 seconds. This push covers everything from the full UX/UI audit:

**Client gallery** (`atche-visuals.html`):
- Gallery-first flow — photos before packages, so clients see value before pricing
- Code entry moved above the package menu on the landing page
- Dynamic download button: "Download 3 Photos + 1 Reel" etc. based on selection
- Inline error messages replace native browser alerts
- Focus-visible keyboard styles (gold outline)
- `--text3` bumped from `#6A6258` → `#8A7E72` for WCAG AA contrast

**Studio** (`atche-visuals-studio.html`):
- All `alert()` / `confirm()` / `prompt()` calls replaced with on-brand modals and toasts
- Login button now says "Sign In" (was "ENTER")
- Sign-out link in the header next to your email address
- "Close Session" moved to its own section at the bottom of the active-session card (with a confirm modal)
- Secondary actions (Send Feedback, Reopen, Delete) collapsed into a `···` overflow menu per session row — primary actions (Confirm Paid, Mark Paid, Open →) stay inline
- Revenue stats show pulsing skeleton bars while loading, then snap in
- Drop zone copy simplified to "Tap to upload"
- Label + contact inputs stack vertically on mobile
- `--text3` bumped + focus-visible styles + aria-labels on icon-only buttons

**New file** (`atche-visuals-feedback.html`) — pilot survey (run the `visuals_feedback` SQL block from `ATCHE-Visuals-Setup.md` if you haven't already).

## After it's live — quick smoke test

- Open Studio — confirm login button says "Sign In" and revenue boxes show skeleton shimmer before data loads.
- Sign in, confirm your email shows in the header with a "Sign out" link.
- Create a test session — confirm the `···` menu appears on the row with Send Feedback inside it.
- Open the session — confirm "Close Session" is at the bottom of the card (not inline with Notify Client).
- Upload a photo and tap "Notify Client" on an empty session first — confirm the on-brand modal appears asking if you want to send anyway.
- Tap **Confirm Paid** on a session with a pending payment — confirm the gold modal appears (not a browser dialog).
- Use `···` → Delete on a closed session — confirm the red danger modal appears.
- On the client gallery, enter a test code — confirm the gallery loads first, packages second.
- Tap download after payment — confirm the button says "Download X Photos" not just "Download."
- On that same unpaid session, use `···` → Send Feedback — confirm the modal, then check the text/email/clipboard opens with the survey link.
