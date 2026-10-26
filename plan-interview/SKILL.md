---
name: "plan-interview"
description: "Asks the user a list of questions in order to refine an idea and produce a final plan."
---

Interview the user relentlessly about their plan until you reach a shared understanding. Your goal is to uncover every material decision, assumption, constraint, dependency, and trade-off—not merely to gather enough information to get started.

## 1. Build and maintain a decision tree

Represent the plan as a **design tree**: each decision may create, constrain, or unblock other decisions.

Maintain an internal record of:
- Decisions that are unresolved.
- Decisions the user has settled.
- Dependencies between decisions.
- Facts that need to be investigated.
- Assumptions and uncertainties that could affect the plan.

A decision is eligible to be asked only when its prerequisites have been resolved. Never ask a downstream question that requires guessing the answer to an unresolved upstream question.

After every answer, update the tree and identify the next eligible decision.

## 2. Ask exactly one question per turn

**Ask one question, then stop and wait for the user's answer before asking another.** Never batch questions into rounds, even when multiple decisions are currently eligible.

Among eligible decisions, choose the one that most usefully advances the interview, prioritizing decisions that unblock other decisions or have significant downstream consequences.

Use the `AskUserQuestion` tool for each question when it is available. Ask one question per tool call. If the tool is unavailable, present the question in ordinary text using the same interaction rules.

For each question:
- Provide 2–4 clear, meaningful answer options.
- Put your recommended option first and mark it **(Recommended)**.
- Briefly explain why you recommend it.
- Include any necessary context to make the decision understandable.
- Rely on the question tool's built-in "Other" option for answers outside the listed choices, when available.
- Phrase the question so that "yes" clearly accepts your recommended answer, where practical.

Ask for a decision, not a rubber stamp. Respect the user's judgment even when they choose an option you do not recommend.

## 3. Investigate facts; ask the user for decisions

Finding facts is your responsibility. If information can be obtained from available files, tools, documentation, or the environment, investigate it rather than asking the user to supply it.

If an investigation is possible, perform it using the available tools. If the environment supports sub-agents, delegate independent investigations when useful. Do not claim to have inspected something you could not access.

An investigation that could change a decision is an unresolved dependency: do not ask a question that relies on its result until the relevant fact is known. Continue with another eligible decision if one exists.

Distinguish facts from assumptions and recommendations. Do not present an unverified assumption as a fact.

## 4. Interpret answers carefully

After each answer:
1. Record what the user actually decided.
2. Resolve any ambiguity that materially affects the plan before proceeding.
3. Update the decision tree and its dependencies.
4. Reconsider which questions are now eligible.
5. Ask the next eligible question, if one remains.

If the user's answer introduces a new decision, add it to the tree. If the user changes an earlier decision, revisit any downstream decisions that depend on it. Do not silently preserve decisions that the change has invalidated.

Accept answers in the user's own words; do not force them into the predefined options.

## 5. Be exhaustive without being wasteful

Explore every material branch of the decision tree. Do not skip a decision merely because its answer seems obvious or because you have a strong recommendation.

However, do not ask questions whose answers are already established, whose outcomes cannot affect the plan, or which duplicate a decision the user has already made.

When several decisions are eligible, prioritize the one with the greatest impact, uncertainty, or ability to unblock subsequent decisions. Do not ask about downstream details prematurely.

## 6. Do not act before approval

Do not implement, execute, publish, purchase, modify files, or otherwise act on the plan while the interview is in progress, unless the user explicitly authorizes a separate investigative action that does not commit the plan. The one file you may write is the plan document described in section 7.

When no material unresolved decisions remain, stop asking and move to section 7. Do not ask a separate "did I understand you?" question first: the approval step in section 8 is that confirmation.

## 7. Write the plan

Produce a single plan document, in the style of a plan-mode plan: concise enough to scan, detailed enough that someone could execute it without re-reading the interview. Write it from the agreed decisions, not from the order the questions were asked.

Use this structure, omitting sections that genuinely don't apply (for example, "Files to change" for a plan that involves no code):

```markdown
# <Plan title>

## Context
Why this is being done: the problem, the goal, and what "done" looks like. 2–4 sentences.

## Decisions
| Decision | Choice | Why | Rejected alternatives |
|---|---|---|---|
| <each settled decision> | <what the user chose> | <rationale, one line> | <options not taken> |

## Approach
The chosen solution in a short paragraph or a few bullets. Only the recommended approach, not a survey of alternatives.

## Implementation steps
1. <Concrete step> — <what it produces>
2. ...
Order steps by dependency. Note any step that can run in parallel with another.

## Files to change
- `path/to/file` — what changes and why
- Existing functions, utilities, or patterns to reuse (with paths), so nothing is rebuilt needlessly.

## Verification
How to check the result end to end: commands to run, tests to add or run, what to look at, and what success looks like.

## Assumptions and risks
- **Assumption:** <unverified thing the plan relies on> — <how it will be checked>
- **Risk:** <what could go wrong> — <mitigation>

## Out of scope
Things deliberately not being done, so they are not added later by accident.
```

Rules for the plan:
- Every decision settled in the interview appears in the Decisions table. If the user overrode your recommendation, record their choice, not yours.
- Mark facts you verified as facts and assumptions as assumptions; never upgrade an assumption to a fact.
- Reference real paths, names, and commands you actually found during investigation. Do not invent file paths.
- No filler: no restating the question, no generic best-practice advice that isn't specific to this plan.

Save the plan to a Markdown file so it survives the conversation: `.claude/plans/<short-kebab-name>.md` inside the project when you are working in one, otherwise in the current working directory. Then show the **full plan** in your message as rendered Markdown (not only a link to the file), followed immediately by the approval question in section 8.

## 8. Ask for approval

Ask for approval with a single `AskUserQuestion` call, mirroring plan mode's approval prompt:

- **Question:** "Approve this plan?"
- **Options**, in this order:
  1. **Approve and implement (Recommended)** — start executing the plan now, step by step.
  2. **Approve, don't implement yet** — keep the saved plan; stop here.
  3. **Keep refining** — there's a gap or something to change.
- In the first option's `preview`, put a condensed version of the plan (the Decisions table and the numbered Implementation steps), so the user can review it right in the question card.
- The built-in "Other" option covers free-form feedback.

If the tool is unavailable, ask the same question in plain text with the same three choices.

Handle the answer:
- **Approve and implement:** this is the authorization to execute the plan as written. Turn the implementation steps into a task list, work through them in order, and run the Verification steps at the end. Anything consequential that the plan did not cover (deleting data, publishing, pushing, spending money, actions outside the agreed scope) still needs its own explicit approval.
- **Approve, don't implement yet:** confirm where the plan is saved and stop.
- **Keep refining** or feedback via "Other": update the decision tree with the feedback, resume the interview from section 2 (one question at a time) for anything still unresolved, then rewrite the plan file and ask for approval again. Do not patch the plan silently without showing the updated version. If the user picks Keep refining without saying what to change, ask a single open question about what's missing or wrong before resuming; do not guess.

If, while implementing, you discover something that invalidates a decision in the plan, stop, explain what changed, and ask before deviating.
