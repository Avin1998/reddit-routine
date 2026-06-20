# Claude Code Routine — "Draft Reddit comments"

This is the cloud-hosted half. It runs on Anthropic's servers on a schedule,
reads `new` rows from the Notion database, drafts a comment for each using your
Reddit voice/skill, writes the draft back, and stops. **It never posts to Reddit.**

## What the routine needs configured

1. **Notion connector** authorized in the routine's environment, with access to
   the "Reddit Watch" database.
2. The **`social-media-writer` skill enabled** in the routine's environment (it's
   a markdown-only skill, no scripts — just turn the plugin/skill on for this
   routine). The routine loads it natively; do NOT paste the skill text into the
   webhook payload. One source of truth, no drift.
3. **Model: Sonnet.** Skills are model-agnostic, so Sonnet runs the skill
   identically and keeps cost down. Set the routine model to Sonnet.

### Note on the skill's "study real comments first" rule
The skill's Hard Rule 0 says to study live comments via Claude in Chrome before
drafting. The cloud routine has **no browser**, so it can't do that step. Two
options:
- **Default (simplest):** rely on the skill's baked-in texture rules — they
  already encode the real-comment patterns (lead personal, hedge, 1-3 sentences,
  no clever closer, no em dashes). Good enough to draft well.
- **Upgrade (if drafts read generic):** have the extension fetch a few top
  comments per sub (`/r/<sub>/top.json`) and attach them to the webhook payload
  as calibration context. That's dynamic data worth passing — unlike the static
  skill text. Ask and I'll wire it into the extension.

## Trigger: fired automatically by the extension (no cron, no click)

Create the routine with a **webhook / API trigger** instead of a cron schedule.
The extension fires that webhook **itself**, automatically, at the end of any
poll cycle where it found new posts. So:

- It is **not** time-scheduled — nothing runs on a fixed clock.
- You do **not** click anything — the extension is the trigger.
- It runs **only when there's new data**, so empty cycles don't spend a run.

Mechanics: put the trigger URL in `extension/config.js` (`ROUTINE_WEBHOOK_URL`,
plus `ROUTINE_WEBHOOK_TOKEN` if required). After the extension pushes new rows to
Notion, it POSTs to that URL and the routine drafts against the fresh `new` rows.

Optional manual override: the popup also has a **"Draft now"** button that fires
the same webhook, and you can hit it with
`curl -X POST "<ROUTINE_WEBHOOK_URL>" -H "Authorization: Bearer <token>"` — but you
don't need either for normal operation.

## Routine prompt

```
You are drafting Reddit comments for Avinash (founder of Levelop) to build
credibility. Work only inside the "Reddit Watch" Notion database.

STEP 0 — Load the voice
Load and follow the `social-media-writer` skill. It is the source of truth for
Avinash's voice, the per-sub promo rules, and the AI-tell blacklist. Everything
below is a safety net, not a replacement for the skill.

STEP 0.5 — Load Avinash's feedback (this is how you improve over time)
- Query the "Reddit Lessons" database for rows where Active = true. Treat every
  one as a HARD rule, on top of the skill. If a lesson ever conflicts with the
  skill, the lesson wins (it's the most recent human correction).
- Query the "Reddit Watch" database for the most recent ~8 rows that have a
  non-empty Final (the comment Avinash actually posted after editing your draft).
  Study each Draft → Final pair: the edit shows exactly what to change. Match the
  Final style, not your old Draft style.

STEP 1 — Read work
Query the database for pages where Status = "new". If none, stop.

STEP 2 — For each new post
- Read Title, Subreddit, Author, Body, Permalink.
- Decide if it's worth commenting on (is there something genuinely useful Avinash
  can add from a backend-engineer / founder perspective?). If not, set
  Status = "skipped" and write a one-line reason in Notes. Do not draft.
- If worth it, draft ONE comment in Avinash's voice using the Reddit writing
  skill: humble, specific, helpful, no AI tells, no corporate tone.

STEP 3 — Respect per-subreddit promo rules (HARD constraints)
- r/leetcode: NEVER mention Levelop or any product/site. Self-promo = permanent
  ban. Helpful comment only.
- r/LeetcodeDesi: NEVER mention Levelop, even indirectly. ANY self-promo = perma
  ban without appeal. No AI-sounding text. Helpful comment only.
- r/cscareerquestions: No product mention in normal comments. Only the monthly
  stickied promo thread allows promo — never assume a normal post is that thread.
- r/developersIndia: No selling. No product mention outside designated threads.
- r/SideProject: Product mention is allowed and welcome where relevant.
- Everywhere: Levelop lives in Avinash's profile/bio, not in comments. Helpful
  comment → profile click → discovery. When in doubt, NO mention.

STEP 4 — Write back
- Put the drafted comment in the Draft property.
- Put a short note in Notes: which sub-rule applied, and why this comment fits.
- Set Status = "drafted".

SAFETY-NET CHECK before writing each draft (from the skill — run every time):
- No "Levelop" and no feature name (Orion, Aurora, sprints) unless it's the cscq
  monthly promo thread. If present, rewrite without it.
- Comment is 1-3 sentences, not a paragraph.
- Last sentence is NOT quotable / a lesson. End plain.
- No em dashes. No "it's not X, it's Y". No rule-of-three.
- Leads with experience/feeling, not the point. Hedged, not declarative.

Never call any Reddit posting tool. Never change a row whose Status is not "new".

END OF RUN — output a one-line heartbeat so you have visibility (the routine's
delivery channel, e.g. email/Slack, shows this):
"[reddit-routine] <timestamp> — read <N> new, drafted <N>, skipped <N>,
 active lessons <N>, errors <N or none>"
```

> Run the **routine-healthcheck.md** prompt once after setup to confirm the
> routine can see the data, load the skill, and write in the right format.

## Your review loop

Open the Notion database, filter `Status = drafted`, read each `Draft`, edit if
needed, post it to Reddit yourself, then set `Status = posted`. That manual post
step is the safety valve that keeps the account clean.
