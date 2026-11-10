---
name: clarify-first
description: Understand a problem together with the user before the agent solves it, so the user keeps the decisions and knows what is going on. Use when the user brings a problem or a non-trivial task without a clear goal, done check, scope or chosen approach ("tengo un problema con X", "necesito implementar Y", "ayúdame a construir Z", "I need to build X", "how should I approach Y"). Do NOT use for trivial edits, pure knowledge questions, or when the user says "just do it" / "hazlo".
license: Apache-2.0
metadata:
  author: alexeira
  version: 0.3.0
  repository: https://github.com/alexeira/skills
  category: workflow
  tags: [task-scoping, planning, clarification, decisions]
---

# clarify-first

The user wants AI to solve the problem, not to decide it for them, and wants to understand what is happening on the way. Your job is execution; the user's is judgment. This skill makes both explicit before any code is written.

Reply in the user's language.

## 1. Understand the problem

Before you ask anything, read what you can: the code, the error, the logs, the docs. Then restate the problem in one or two lines and declare your position:

- What you verified, with where you saw it.
- What you assume.
- What you don't know and how you would find out.

If the restatement is wrong, the user corrects it here, before anything else is built on it. If the problem turns out to be clear and small, say so and do it.

## 2. Fill in the brief

A task is ready when it has these parts. Ask only for the ones the user's message and the repository don't answer:

- **Goal**: what is true when it is done.
- **Why**: what the user needs it for. It decides the tradeoffs.
- **Constraints**: what must not change, deadlines, tools.
- **Out of scope**: only what the user defers. Never invent items to look thorough.
- **Check**: the command, test or screenshot that proves it is done. A logic change gets a test that fails before and passes after; a UI change gets before and after screenshots; config gets the command that shows the effect.

If the user wants everything at once and part of it is new to them, offer the smallest first cut that can be checked. They can keep the full scope.

## 3. Surface the decisions

List the decisions the task contains: approach, library, data shape, behavior at the edges, what ships. For each one, give the options with their tradeoffs and your recommendation with its reason, then stop. Never pick silently between reasonable options.

A fact the code or docs can settle is not a decision: settle it yourself and show the evidence.

## 4. Choose how to work

Ask once, with your recommendation:

- **Orchestrate**: the user drives and you do one narrow step at a time. For learning or keeping control.
- **Accelerate**: you implement end to end and the user reviews the diff. For a domain the user knows.
- **Research first**: a one-page brief (concepts, options, the first step to try) and no code. For something new to the user. Deliver it and stop; no "did that help?".

## Output

A plan of at most 15 lines that the user approves:

```
Problem: <one line>
Goal: <what is true when done>
Check: <the command or evidence>
Out of scope: <what the user deferred, or —>
Decisions: <each one and what the user chose>
Mode: <Orchestrate | Accelerate | Research first>

Steps:
1. <action> → check: <how>
```

While you execute, stop at any decision that was not in the plan and bring it to the user the same way as in step 3. At the end, report the check as evidence and, in a few lines, what you did and why, so the user can explain it without you.

## Rules

- Use the harness's question tool (`AskUserQuestion`, `questionnaire`) with 2 to 4 options and a recommended one; without one, ask numbered questions in text. After you ask, wait.
- At most three rounds of questions. After that, propose the plan with your best guesses and let the user correct it.
- Skip what the user already answered. A message that has the goal, the check and the approach goes straight to the plan.
- On "just do it", "hazlo" or "stop asking": state your assumptions in one paragraph and proceed.
- Don't use it for typos, one-line changes, or questions like "how does X work": answer those directly.
