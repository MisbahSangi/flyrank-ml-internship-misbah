# Weekly Report Automation Agent

An AI-assisted pipeline that turns raw weekly activity notes into a properly structured internship report, with a built-in honesty check before anything gets finalized.

## What it does and for whom

Built for my COMSATS University internship reporting requirement (weekly reports covering Activities Performed, Tasks Accomplished, Challenges Faced, Skills Acquired, Goals for Next Week). Anyone with a recurring structured-report requirement — internship logs, standups, status updates — can adapt this same 4-step pattern.

## Architecture

```
Raw notes (me, no AI)
      ↓
Claude Project — Structure
  (formats into the 5 required sections)
      ↓
Claude Project — Critique
  (flags invented details, vague lines, thin sections)
      ↓
Claude Project — Revise
  (applies the critique, outputs final report only)
      ↓
Final report (Word doc, ready for submission)
```

Extended with **Gmail MCP** (connected via FL-05): the same Claude Project can read source material and reference context directly from Gmail rather than requiring everything to be manually copy-pasted. This is what pushes the system from a fixed workflow toward genuinely agentic tool use, since the model decides what to pull from Gmail based on context, not a hardcoded step.

## Setup (a stranger could follow this)

1. Create a Claude Project (any Claude account).
2. Paste this into the Project's custom instructions:

```
I am [your name], [role/program details].

My weekly report template has these exact sections:
1. Activities Performed
2. Tasks Accomplished
3. Challenges Faced
4. Skills Acquired
5. Goals for Next Week

Rules you must follow every time:
- Never invent details not present in my raw notes
- Never use generic filler lines
- Every bullet must be specific: name the tool, error, outcome, number
- If my raw notes are thin for a section, say so explicitly
- Each section: 3-5 bullets, each bullet 1-2 sentences max
```

3. (Optional) Connect the Gmail MCP connector in Claude's settings if you want the agent to pull context from email directly.
4. Each week, run the 3 prompts below in a new chat inside the Project.

## Usage

**Step 1 — write raw notes** (you, not AI): bullet points, messy is fine.

**Step 2 — structure:**
```
Structure these notes into the weekly report format: [sections].
My raw notes: [paste]
```

**Step 3 — critique:**
```
Review this draft. Flag: any invented details not in my raw notes,
any vague or generic lines that need specifics, any section that
is too long or too short.
```

**Step 4 — finalize:**
```
Apply the critique. Keep all details accurate to my raw notes.
Output the final report only, no commentary.
```

## v2 Eval Results (from FL-04, 5 real runs)

| Metric | Result |
|---|---|
| Setup cost (one-time) | ~20 minutes |
| Per-week time (Steps 1-4 + manual edits) | 45-75 minutes |
| Manual baseline (writing from scratch) | 4-6 hours |
| Time saved per week (conservative) | 3-5 hours |
| Time saved across 4 documented weeks | 12-20 hours |

**Real critique catches, not hypothetical:** Run 1 caught an invented framing ("baseline" not in raw notes), a vague line (unspecified F1 numbers), and a thin section (Goals had only 2 bullets, correctly disclosed rather than padded) — full detail in the FL-04 walkthrough document.

## Limitations (guardrail explained on camera)

- **Cannot verify facts, only consistency with input.** If raw notes contain a wrong number, the final report repeats it faithfully. Every claimed number must be manually checked against the actual source (notebook output, commit, etc.) before submission — this is a hard guardrail, not optional.
- **Cannot infer difficulty.** If raw notes don't explicitly name what was hard and why, "Challenges Faced" goes generic — the model has no way to infer struggle that wasn't written down.
- **Section length depends entirely on input richness.** Thin notes produce thin (but honestly-disclosed) sections, not padded ones.
- **Gmail MCP extension is read-context only in current use** — it does not currently auto-send the final report; that step stays manual by design, since publishing/submitting should always be a deliberate human action.
