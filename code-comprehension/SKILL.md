---
name: code-comprehension
description: >
  Ensures engineers understand AI-generated code before it ships. Use this skill
  whenever you finish generating, modifying, or substantially rewriting code for
  a user. Also trigger when the user says things like "write this for me",
  "implement this feature", "generate a migration", "add tests for X", or any
  request where Claude produces code the user will commit. Do NOT trigger for
  pure explanation requests ("how does X work?"), code review of the user's own
  code, or tiny one-line changes where comprehension is obvious.
---

# Code Comprehension Skill

## Why this skill exists

When AI agents write code for engineers, there's a real risk that the engineer
merges code they don't fully understand. This creates problems downstream:
debugging is harder, the codebase accumulates "nobody understands this" zones,
and the engineer's own skills atrophy. This skill exists to close that gap — not
by slowing people down with busywork, but by building understanding into the
workflow naturally.

The goal is that after every significant code generation, the engineer walks away
understanding *why* the code is shaped the way it is, not just *what* it does.

## When to activate

After you generate or substantially modify code (more than ~10 lines of
meaningful logic), run through the steps below before considering the task
complete. Skip this for trivial changes (renaming a variable, fixing a typo,
adding a simple accessor).

## The workflow

### Step 1: Generate the code as requested

Do the work the user asked for. Write good code, follow existing patterns in
the codebase, etc. Nothing changes here.

### Step 2: Explain your decisions (not your code)

After presenting the code, add a **Design Decisions** section. This is NOT a
line-by-line walkthrough — engineers can read code. Instead, explain:

- **Why this approach over alternatives.** For example: "I used a service object
  rather than a concern here because the logic touches three models and doesn't
  naturally belong to any of them."
- **Trade-offs you considered.** For example: "This adds a database index which
  speeds up the query but will slow down inserts slightly. For your table size
  this is worth it."
- **Anything non-obvious.** Edge cases handled, Rails conventions leveraged,
  performance implications, security considerations.

Keep it concise — 3-5 bullet points for a typical feature. The test is: would
a senior engineer reviewing this PR want to know these things?

Format example:

```
## Design Decisions

- **Service object over concern**: The discount calculation touches Order,
  LineItem, and Coupon — a concern on any one model would create awkward
  cross-model dependencies.
- **Chose `find_each` over `all.each`**: The coupons table could grow large;
  `find_each` batches the query to avoid loading everything into memory.
- **No caching yet**: The calculation is fast enough for now (~2ms). Added a
  TODO comment for when this needs caching at scale.
```

### Step 3: Comprehension check

After the design decisions, pose 2-3 short questions that test whether the
engineer understands the code well enough to own it. These should be practical
questions — the kind of thing that would come up in a real debugging session or
code review.

Good comprehension questions:
- Target the *interesting* parts — not syntax, but design and behaviour
- Are answerable in 1-2 sentences
- Would reveal whether someone actually understands vs just skimmed
- Relate to likely future scenarios (debugging, extending, reviewing)

Examples of GOOD questions:
- "If a coupon is expired but was applied before expiry, what happens here?"
- "What would you change if this needed to support bulk operations?"
- "Why might we get an N+1 query if we removed the `includes` on line 14?"

Examples of BAD questions (don't ask these):
- "What does `each` do?" (too basic — insulting)
- "What is the return type of this method?" (trivially readable from the code)
- "Can you explain the code back to me?" (vague, feels like a test)

Format example:

```
## Quick check

Before committing, make sure you're comfortable with these:

1. What happens if `apply_discount` is called twice on the same order?
2. If you needed to add a new discount type (e.g. "buy 2 get 1 free"), which
   class would you modify and why?
3. Why does the migration add an index on `[coupon_id, expires_at]` rather than
   just `coupon_id`?
```

### Step 4: Respond to answers

When the engineer answers the comprehension questions:

- **If they've got it**: Confirm briefly and move on. Don't over-praise or
  belabour the point. A simple "Exactly right — you're good to ship this" is
  fine.
- **If they're partially right**: Gently fill in the gap. "You're on the right
  track — the one thing to add is that the composite index also covers the
  query in `Coupon.active`, which filters on both columns."
- **If they're stuck**: That's useful signal, not failure. Explain the concept
  clearly, and consider whether the code should be simplified. If the engineer
  can't explain it, maybe the code is too clever.
- **If they say "skip" or "I know this"**: Respect that. Don't force it. The
  questions are there as a safety net, not a gate. You can say "No worries,
  just flagging in case — the main thing to know is [one-sentence summary]."

### Step 5: Decision log entry (for substantial work)

For significant features or architectural decisions (not every small task),
offer to generate a lightweight decision log entry. Only do this for work
where the *why* matters weeks or months later.

Format:

```markdown
<!-- decision-log: YYYY-MM-DD -->
## [Short title]

**Context**: [1-2 sentences on the problem]
**Decision**: [What we chose]
**Alternatives considered**: [What we didn't choose and why]
**Consequences**: [What follows from this — good and bad]
```

Offer this with: "This was a meaningful architectural choice — want me to
generate a decision log entry you can add to your docs?" Don't force it.

## Tone guidance

- Talk to the engineer like a knowledgeable colleague, not a teacher or
  examiner. The vibe should be "here's what I'd want to know if I were
  reviewing this PR" — not "quiz time."
- Assume competence. These are professional engineers. Don't explain language
  basics or framework fundamentals unless they ask.
- Be genuinely helpful, not performatively thorough. If the code is simple
  and the decisions are obvious, keep the design decisions short and ask
  fewer questions.
- Scale the depth to the complexity. A 10-line bugfix gets a sentence of
  context and maybe one question. A new service object with a migration gets
  the full treatment.

## Calibrating to complexity

| Change size | Design Decisions | Comprehension Qs | Decision Log |
|-------------|-----------------|-------------------|--------------|
| Trivial (<10 lines, obvious) | Skip entirely | Skip entirely | No |
| Small (10-50 lines, routine) | 1-2 bullets | 1 question | No |
| Medium (feature, new class) | 3-5 bullets | 2-3 questions | Offer |
| Large (architecture, migration) | Full section | 2-3 questions | Offer |
