# reddit-routine

Carrier repo for the Claude Code Routine that drafts Reddit comments for the
Reddit credibility pipeline. The routine clones this repo on every run so it can
load the writing skill below. There is no application code here.

## What's in here
- `.claude/skills/social-media-writer/` — the writing voice + per-sub promo rules
  the routine follows. This is the permanent home for the voice; fold proven
  "Reddit Lessons" into this file over time and commit them.
- `ROUTINE-PROMPT.md` — the exact prompt to paste into the routine config at
  claude.ai/code/routines (kept here for reference/version history).

## How it fits together
Chrome extension polls Reddit -> writes new posts to the "Reddit Watch" Notion DB
-> extension fires this routine's /fire API endpoint -> routine reads new rows,
drafts comments using the skill + active Reddit Lessons, writes drafts back ->
you review and post from the on-page panel.

See SETUP.md and claude-routine.md in the parent project folder for full wiring.
