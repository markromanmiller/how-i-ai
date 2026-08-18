Generate SYSARCH.md (technical architecture doc) and INTENT.md (decisions not visible in the code) for the current project via code-read + interview.

## What these files are

**SYSARCH.md** is a technical reference document: what the project does, how it is structured, what each component is responsible for, and how the pieces fit together. A reader who has never seen the code should be able to understand the system from SYSARCH. Claude can write this entirely from reading the codebase.

**INTENT.md** captures what is not visible in the code: design decisions where the code shows the outcome but not the reasoning, alternatives that were considered and rejected, constraints that shaped a choice but aren't recorded anywhere in the implementation. The test for an INTENT entry: if a future developer could reconstruct the reasoning by reading the code, it belongs in SYSARCH, not INTENT. If they couldn't — if it lives only in the developer's head — it belongs in INTENT.

## Process

### Phase 1 — Write SYSARCH

Read the project files thoroughly. Write `SYSARCH.md` in the current directory. Structure it to match the project: an overview paragraph, then sections for the major components, data model, external dependencies, file layout, and anything operationally important. Use tables and code blocks where they add clarity. Do not invent or speculate — only document what the code actually does.

### Phase 2 — Develop INTENT via interview

Do not write INTENT.md from the code alone. The information it needs is not there.

Instead, ask the user questions that surface what's outside the code. Ask in small batches — 2 or 3 at a time — not as a survey. Each question should target a specific decision visible in the code where the reasoning is not: a threshold value, an architectural shape, a scope boundary, a feature that's absent. Ask "why is X the way it is?" or "what did you consider instead of X?"

After the user answers, synthesize the answers and ask the next batch if more decisions remain unexplored. Once you have enough, write `INTENT.md`.

## Format for INTENT.md

- Open with a dateline: `*Written YYYY-MM-DD, based on an interview.*`
- Each section covers one decision or cluster of related decisions
- Section headings are noun phrases naming the thing being explained — not generic labels like "Design constraints." Examples: "Why low confidence detections are kept", "The 10-second-per-minute duty cycle", "Why a monolith"
- Prose, not bullet points
- Each section should explicitly name what was considered and why it was rejected or deferred, if applicable
- An empty screen is a valid outcome: if a decision has no interesting reasoning behind it, don't manufacture a section for it

## Format for SYSARCH.md

- Open with a one-paragraph overview: what the project is and what it does
- Sections should reflect the actual structure of the project — don't force a template
- Common sections: How it works, Data model, Files / components, Configuration, External services, Operational commands
- End with a "Known issues / gaps" section only if there are meaningful ones
