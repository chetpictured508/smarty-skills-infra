# The Story of Smarty Skills-Infra

## The Spark

It started with an article. [Context Infrastructure](https://yage.ai/context-infrastructure.html) by grapeot made a provocative claim: LLMs produce "correct nonsense" by default — technically accurate but soullessly generic output — because they're trained to optimize for consensus. The unlock isn't a smarter model. It's injecting *your* accumulated judgment as structured context.

The author had built a personal system: a three-layer memory hierarchy that silently observed his interactions, reflected on patterns, and distilled stable preferences into compact "axioms" loaded into every future session. L1 Observer → L2 Reflector → L3 Axiom. Over a year, he'd accumulated 44 axioms that turned his AI from a generic assistant into something that thought like *him*.

The question was: could we turn this into an OpenClaw skill that gives every user this superpower — without them having to build the infrastructure themselves?

## Phase 1: The Product Argument

The first instinct was "make a skill that does the three-layer thing." But we pushed back hard on that.

A skill is a task-execution template. What the article describes is a *personal infrastructure* — a living system. These are different things. A skill is a tool. This is a factory that builds tools.

We went through several rounds of scoping:

- **Option A**: A "Context Distiller" that processes raw inputs into axioms (the engine)
- **Option B**: An "Axiom Loader" that manages and loads preferences (the runtime)
- **Option C**: Both combined — the full pipeline

We picked C but with a crucial design constraint from the user: **it had to be frictionless.** Not invisible-sneaky, but invisible-effortless. Install it, forget about it, and your AI just gets better over time.

Then came the platform constraint that changed everything: **OpenClaw doesn't support hooks.** No `PreToolUse`, no `PostToolUse`, no event listeners. The entire mechanism had to work through prompting alone — SKILL.md instructions and file writes to the memory directory.

No hooks meant no passive observation. So the three-layer hierarchy had to be woven into *how the skill instructs the agent to behave during normal work.* The observation, reflection, and distillation all happen through prompt engineering, not infrastructure.

## Phase 2: Architecture Through Autoresearch

For the technical design, we borrowed [Karpathy's autoresearch method](https://github.com/karpathy/autoresearch) — an iterative optimization loop where each round modifies one variable, evaluates against fixed criteria, and keeps or discards the change.

We ran **15 rounds** of design iteration, scoring each against five weighted criteria:

| Criterion | Weight |
|---|---|
| Context cost (tokens consumed) | 30% |
| Signal quality (real preferences vs noise) | 25% |
| Invisibility (zero friction) | 20% |
| Convergence speed (sessions to first useful axiom) | 15% |
| Model-agnosticism (works on any LLM) | 10% |

### The Breakthroughs

**Round 4 — Selective observation triggers.** The baseline observed after every task. Score: 5.75. Switching to observing *only* when the user corrects, rejects, or states a preference jumped signal quality from 4 to 9. This was the single biggest improvement. Most tasks produce zero observations — and that's correct. Silence is signal too.

**Round 5 — Session-start reflection trigger.** Without hooks, the only reliable trigger point is when SKILL.md loads fresh — session start. We set a 15-observation threshold: when enough signals accumulate, reflection runs automatically before the user's first task.

**Round 6 — First-person axiom voice.** We tested four axiom formats: prose, structured rules, weighted codes, and natural voice. Natural voice won decisively — "I prefer explicit error handling with early returns" works across every LLM because it reads like a system prompt. No parsing needed.

**Round 9 — Load everything, cap at 25.** We considered domain-based loading, strength-based filtering, tiered strategies. The winner was the simplest: load all axioms every session, but cap the file at 25 entries. L3's job includes pruning. Premature optimization for a scaling problem that doesn't exist yet.

**Round 10 — Bootstrap mode.** Cold start is real. A new user with zero observations gets zero value for potentially weeks. The fix: during the first few sessions, cast a wider net — observe not just corrections but also what the user accepts, consistent choices, and tool selections. Then narrow to selective triggers once enough mass exists.

After 15 rounds: **score 8.6/10, ~850 tokens.**

## Phase 3: Building It

Implementation was straightforward — the design was specific enough that the skill almost wrote itself. Two files:

- `SKILL.md` — the main skill with always-on observation instructions, session-start reflection logic, and a worked example
- `references/profile-format.md` — the axiom format spec, loaded only during reflection

The skill creates two runtime files in the user's memory directory:
- `observations.log` — append-only raw signals
- `context-profile.md` — the distilled profile with axioms, patterns, dormant entries, and contradictions

## Phase 4: Code Review Caught Real Bugs

The paranoid review found three must-fix issues:

1. **Bootstrap mode was expensive.** Checking observation count on every task required a file read. Fix: check at session start only, using a flag in the profile.
2. **Selective line deletion during reflection was fragile.** "Delete promoted entries but keep others" is the most error-prone file operation across LLMs. Fix: full rewrite of the log with only un-promoted entries.
3. **No session dedup.** A user correcting the same thing 3 times would generate 3 observations, inflating the count. Fix: one observation per preference per session.

## Phase 5: 20 More Rounds of Optimization

The code itself went through a second autoresearch pass — **20 more rounds** focused on prompt engineering quality. The eval criteria shifted:

| Criterion | Weight |
|---|---|
| LLM compliance rate | 30% |
| Token efficiency | 25% |
| Cross-LLM robustness | 20% |
| Edge case coverage | 15% |
| Progressive disclosure | 10% |

### What Moved the Needle

**Positive gating (Round 4/R2).** Replacing "Do NOT observe routine completions" with "ONLY observe when one of these triggers fires" was the single biggest compliance win. LLMs reliably ignore negative instructions. Positive constraints stick.

**Execution-order sections (Round 2/R2).** The original SKILL.md was organized by abstraction layer (L1, L2, L3). But the agent's actual execution order is: load profile → check reflection → do tasks → observe. Reordering sections to match execution order improved compliance across all models.

**Role-assignment opener.** "You maintain a lightweight memory of this user's preferences" — priming the agent's identity at the top of the skill sets the tone for everything that follows.

**Example-driven reflection.** The original had an 8-step reflection algorithm. We replaced it with 4 high-level steps plus a concrete worked example. LLMs learn procedures better from examples than from numbered instructions.

**Retraction trigger (Round 28 stress test).** Adversarial scenario testing revealed a gap: what if the user says "forget that preference"? Without an explicit retraction trigger, the system had no mechanism for immediate removal. We added a third trigger type — `retraction` — with immediate axiom removal during reflection, no threshold needed.

### The Final Numbers

| Metric | Start | After 15 rounds | After 35 rounds |
|---|---|---|---|
| Score | 5.45 | 8.6 | **9.2** |
| SKILL.md tokens | ~1,000 | ~850 | **~570** |
| profile-format.md tokens | ~430 | ~430 | **~215** |
| Total | ~1,430 | ~1,280 | **~785 (-45%)** |

45% fewer tokens. 69% higher score. Same behavior.

## Phase 6: QA and Ship

QA caught six more issues: a leaked `.claude/settings.local.json`, a missing LICENSE file, name mismatches between the display name (Smarty Skills-Infra) and internal paths (`context-infra`), and a stale line count in the README.

We kept `context-infra` as the internal directory name — it's descriptive and changing all paths would add migration complexity. "Smarty Skills-Infra" lives in the frontmatter, headings, and README as the brand name.

The repo shipped to GitHub with four commits that tell the real story:
1. Add the skill
2. Remove personal config that doesn't belong
3. Rename to Smarty Skills-Infra
4. Fix QA bugs

## What We Learned

**The consensus ceiling is real.** Without personalized context, every user gets the same median output. The three-layer hierarchy is a concrete, buildable mechanism to break through it.

**Hooks aren't necessary.** The entire system works through prompting and file writes. This was initially seen as a constraint, but it turned out to be a feature — it makes the skill model-agnostic and platform-portable.

**Autoresearch works for design, not just ML.** Running 35 rounds of modify-evaluate-keep/discard on prompt engineering produced measurably better output than trying to write the perfect skill in one shot. The method forced single-variable changes, which made it clear what actually moved the needle.

**Most tasks should produce zero observations.** This was counterintuitive. The instinct is to capture everything. But the selective trigger design — only on corrections, stated preferences, and retractions — produces dramatically cleaner data. The L2 reflector works with concentrated signal instead of sifting through noise.

**The hardest problem is the cold start.** A skill that does nothing for the first 10 sessions feels broken. Bootstrap mode — casting a wider observation net early, then narrowing — is the pragmatic solution. It trades a little noise for much faster time-to-value.

## What's Next

The skill is live at [github.com/ckpxgfnksd-max/smarty-skills-infra](https://github.com/ckpxgfnksd-max/smarty-skills-infra). Install it, use your AI normally, and watch it gradually learn how you think.

The axioms it builds are yours — your taste, your judgment, your biases. That's the point. As the original article put it:

> Silicon-based brains pursuing pure objectivity can only achieve intelligent mediocrity. Only human judgment accumulated over decades, filled with bias and taste, can reshape it.
