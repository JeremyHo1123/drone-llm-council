---
name: drone-llm-council
description: "Engineering decision council for drone, PX4 and robotics work. Five advisors with different lenses (Safety & Failure Modes, Domain Principles, Verification, Field Operations, First Principles) analyse one technical decision in parallel, review each other anonymously, and the host chairs a verdict with what to verify first and a concrete first step. Use ONLY when the user explicitly asks for it: '/drone-llm-council', 'council this', 'run the council', 'convene the council'. Do not start it for ordinary 'A or B?', 'should I…?' or 'what would you do?' questions — a full session runs 10 subagents."
---

# Drone LLM Council

Put one engineering decision in front of five advisors who look at it from different angles. Let them review each other without knowing who wrote what. Then chair a verdict the user can act on.

It is built for drones, PX4 and other hardware that moves, and works for any technical decision with real consequences.

The method follows Andrej Karpathy's [LLM Council](https://github.com/karpathy/llm-council): independent answers, anonymous peer review, a chairman's synthesis. The idea of packaging it as a single Claude Code skill comes from [aiwithremy/claude-skills-llm-council](https://github.com/aiwithremy/claude-skills-llm-council) by Ole Lehmann. This skill is an independent rewrite for engineering work.

---

## When to use it, and when not to

**Good fit**

- Choosing between designs, fallback behaviours or test plans
- Reviewing a change before it goes onto a vehicle
- Surfacing assumptions nobody has checked yet

**Poor fit**

- A question with one right answer (a parameter default, an API signature): look it up in the source or the docs
- Writing code or documentation: just do the work

**Limits to state honestly**

- All five advisors are the same model with the same knowledge gaps, and the reviewers are that model grading itself. A council reduces framing bias and missed angles; it does not add knowledge the model lacks.
- It does not replace reading the source, the official documentation, simulation, props-off tests or flight logs.
- For a change that will fly or move hardware, the verdict must recommend an additional review of the brief by a model from a different vendor, or by a person.

---

## The five advisors

Each advisor is a way of thinking, not a job title. If the project has safety rules (usually in `CLAUDE.md`: actions that need the user's permission, folders that must not be modified), every advisor works within them.

1. **Safety & Failure Modes.** Assume it will fail. Find the worst plausible outcome, the moment it happens (on the ground, at takeoff, in the air, during a mode or state transition, at landing), what catches it, and how bad it is when nothing does. Look hardest at transitions, latency and dropped messages, bad sensor values (NaN, spikes, drift), operator mistakes, and two programs commanding the vehicle at the same time.
2. **Domain Principles.** Check the plan against how the system really behaves: firmware, libraries, protocols, physics. Point every key claim at something that proves it (a source file, a parameter, a documentation page). Watch versions: the firmware on the vehicle can be older or newer than the documentation being read.
3. **Verification.** Work out how to prove it before anyone trusts it: unit tests, log replay, simulation, props-off, tethered, then field. Define pass and fail criteria and the log fields to look at. Name the evidence that is missing, and the ways a test could pass while the real system fails.
4. **Field Operations.** Stand where the operator stands. What do they see, what do their hands do, what happens under time pressure, and how do they recover when it goes wrong? Flag procedures with too many steps, steps that are easy to do in the wrong order, and messages that only make sense if you have read the code.
5. **First Principles.** Ask what problem is really being solved. Is there a simpler move: one parameter, a procedure change, or not doing it at all? Is the question itself the wrong one?

**Built-in tensions.** Safety adds guards while First Principles removes complexity. Domain Principles says what is correct while Field Operations says what is doable. Verification asks everyone for evidence.

---

## Procedure

### 1. Build the brief (you, the host)

**Gather just enough context, then stop:**

- The project's `CLAUDE.md` (hardware facts, safety rules, constraints)
- Files the user mentioned, plus code, docs, tests or logs directly tied to the decision
- Earlier council transcripts in `discussions/` (`*-council-*.md`), so the same ground is not covered twice
- If the folder is a documentation knowledge base (it has `docs-map/` and `raw/documents/`, as in [Drone Knowledge Wiki](https://github.com/JeremyHo1123/drone-knowledge-wiki)), look up the relevant official pages the way its `CLAUDE.md` describes, and list their paths in the brief so advisors can open them

**Write one brief that all five advisors receive:**

1. **Decision** — one sentence
2. **Options** — if the user already has some
3. **Verified facts** — each with its source (file and line, parameter read-back, log name, test result)
4. **Unverified assumptions**
5. **Constraints and safety rules**
6. **Stakes** — what happens if the decision is wrong

Keep your own opinion out of the brief, and do not steer toward an answer. If the brief is long, save it to a file (the session scratchpad if there is one) and give the advisors the path; otherwise paste it inline.

If the request is too vague to brief, ask one clarifying question, then proceed.

### 2. Run the advisors (5 subagents, launched in one message)

Launch all five in the same message so no answer can influence another. Use `subagent_type: Plan`, which has no file-editing tools. If `Plan` is not available, use `general-purpose` and rely on the read-only rule in the prompt.

Advisor prompt:

```
You are the {ADVISOR NAME} advisor on an engineering decision council.

Your lens: {that advisor's description from the list above}

The decision and its brief:
---
{the full brief, or: read the brief at <path> first}
---

Rules:
- Think only through your lens. Be concrete and specific; other advisors cover the other angles.
- Read-only. You may use Read, Grep and Glob to check files the brief mentions. Do not edit files, do not connect to other machines, and do not run anything that changes state. Spend a few minutes at most on checking.
- Tag every key claim [verified: source] or [unverified].
- Answer in the user's language. Keep parameter names, file names and commands exactly as written.
- 250–500 words, no preamble. End with one line: "If I am wrong, it is most likely because: …"
```

### 3. Blind peer review (5 subagents, launched in one message)

Shuffle the five answers and label them Response A to E in random order, never in advisor order. Keep the mapping to yourself. Launch five reviewers at once (same `subagent_type`); each one sees all five responses.

Reviewer prompt:

```
You are reviewing five independent answers from an engineering decision council.

The decision and its brief:
---
{the full brief, or: read the brief at <path> first}
---

Response A:
{text}

Response B:
{text}

Response C:
{text}

Response D:
{text}

Response E:
{text}

Answer the following. Cite responses by letter and be specific.
1. Which response is the most convincing, and why?
2. Which response has the biggest blind spot, and what is missing from it?
3. What did all five responses miss that should be considered?
4. Which [unverified] or untagged claims would change the conclusion if they turned out to be wrong? For each, say how to check it (file, parameter, test).

Rules: read-only — do not edit files or connect to other machines. Answer in the user's language, 250 words or fewer.
```

**Quick mode.** If the user asks for a quick council, skip this step. The session then uses 5 subagents in total.

### 4. Chair the verdict (you, not another subagent)

You now hold the brief, the five answers (map the letters back to advisor names) and the five reviews.

- You may side with a minority when its reasoning is the strongest. Say why.
- Never recommend skipping a step the project's safety rules require, such as a props-off test or asking before touching hardware.
- For flagged claims you can check yourself in a few minutes, check them before writing the verdict and mark them "checked by the host". In quick mode there are no reviews, so do this for the claims you judge most decisive.

### 5. Present the verdict in the conversation

Write it in the user's language, using this structure. Do not generate HTML or other files.

```
## Council verdict: {topic}

### Where the advisors agree
{conclusions several advisors reached independently — the most reliable part}

### Where they disagree
{real disagreements: state both sides and why reasonable people land differently}

### Blind spots found in review
{what individual advisors missed and the reviewers caught}

### Verify before deciding
{unverified claims that would change the conclusion; for each, how to check it (file / parameter / test)}

### Recommendation
{a clear recommendation, not "it depends", with the reasons}

### First step
{one concrete, small, reversible action}
```

- If the first step touches hardware, on-vehicle code or parameters, write it as "ask the user for permission to …", following the project's safety rules.
- If the change will fly, add to the recommendation: have a model from a different vendor, or a person, review the brief before flight.

### 6. Save a transcript (when the user asks, or when the decision will be revisited)

Write `discussions/YYYY-MM-DD-council-<kebab-case-topic>.md` in the project. If there is no `discussions/` folder, ask the user where to put it.

```yaml
---
title: "Council: <topic>"
date: YYYY-MM-DD
tags:
  - discussion
  - council
---
```

Contents, in order: the brief, the five answers (labelled by advisor), the anonymisation mapping, the five reviews, the verdict.

---

## Notes

- Start only on an explicit request. A full council runs 10 subagents; quick mode runs 5. The chair is the main session, not an extra subagent.
- Subagents run on the main conversation's model unless Claude Code is configured otherwise (for example with `CLAUDE_CODE_SUBAGENT_MODEL`), so a full council costs about ten extra long-context model runs.
- Launch the advisors together and the reviewers together. Staggered launches let early answers leak into later ones.
- Reviews are always anonymous and shuffled, so reviewers judge arguments rather than advisor names.
- For fact questions, skip the council: reading the source, the docs or the logs is faster and more accurate.
