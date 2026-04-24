---
name: clarify-first
description: Interrogate the user with concrete multiple-choice questions before executing non-trivial tasks. Use when the user says "tengo que hacer X", "necesito implementar Y", "ayúdame a construir Z", "I need to build X", "how should I approach Y", or describes a task without clear scope, success criteria, or chosen approach. Forces clarifying questions about scope, success criteria, user's prior knowledge, and tradeoffs BEFORE any code is written. Prevents over-delegation (handing a half-understood task to the LLM) and analysis paralysis. User may invoke and respond in Spanish or English — match their language. Do NOT use for trivial edits, typo fixes, single-line changes, or when the user explicitly says "just do it" / "hazlo".
---

# clarify-first

A cognitive brake for the agent. Before writing any code, expose what the user knows, what you assume, what's ambiguous, and what "done" actually means. Orchestrate, don't delegate.

## Core Principle

> Research enough to decide. Implement enough to ship. Learn seriously from what you investigated and shipped.

The user wants AI as an accelerator, not as an excuse to skip understanding. This skill forces the conversation that an unclarified task would otherwise skip.

## Language

The user may invoke this skill and answer questions in **Spanish, English, or mixed**. Detect their language from the initial message and **respond in the same language**. Keep your internal reasoning and the question templates in English for consistency, but user-facing text must match the user's language.

Example invocations to expect:
- `necesito hacer un timeline con animación en un path de SVG`
- `I need to add rate limiting to this endpoint`
- `ayúdame a migrar esto de webpack a vite`

## Workflow

Run 3–4 rounds of focused questions, **maximum**. Each round uses `AskUserQuestion` with 2–4 options and a recommended default. Skip rounds whose answers are already obvious from the user's initial message.

### Round 1 — Task shape

Goal: classify the task so you don't waste the user's time.

Ask up to two questions:

1. **Is this new work or iteration on existing code?**
2. **Has the user already chosen the approach, or are they asking you to help decide?**

If the task is trivial (typo, rename, one-liner, config tweak) or the user already has a clear approach AND clear success criteria — **exit the skill** and execute directly. Say so plainly: *"This looks clear enough, I'll just do it."*

### Round 2 — Knowledge exposure

Goal: surface what both parties know and don't know. This is the most important round.

- Ask the user: **"Have you worked with this before?"** Options: `Yes, comfortable` / `Some exposure` / `First time` / `Not sure`.
- Then **declare your own position explicitly**:
  - What you're confident about.
  - What you're assuming (list the assumptions).
  - What you don't know or would need to check.
- If there are 2+ reasonable approaches, present them with tradeoffs. Do **not** silently pick one.

Never hide confusion. If something is ambiguous, name it.

### Round 3 — Success criteria

Goal: turn a vague task into something verifiable.

Ask the user to pick the **minimum deliverable**:

- What does "done" look like? Which test passes, which output appears, which screen renders?
- What's explicitly **out of scope** for this cycle?
- What can be deferred to a follow-up?

Bias toward the smallest scope that still ships value. If the user asks for four things, propose shipping one first and confirm.

### Round 4 — Collaboration mode

Goal: make the user choose how to work, not default to "agent does everything."

Present three modes with `AskUserQuestion`:

- **Orchestrate** — user drives, agent executes specific narrow steps on request. Best when the user wants to learn or retain control.
- **Accelerate** — agent implements end-to-end, user reviews the diff and asks questions. Best when the user knows the domain and wants velocity.
- **Research first** — pause implementation, produce a 5–10 min reading brief (key concepts, library choices, caveats) before touching code. Best when the user admitted "first time" in Round 2.

Default recommendation depends on Round 2's answer:
- `First time` or `Not sure` → **Research first**.
- `Some exposure` → **Orchestrate**.
- `Yes, comfortable` → **Accelerate**.

### Output — Execution plan

After the rounds, produce a short plan the user can approve:

```
Task: [one-line restatement]
Mode: [Orchestrate | Accelerate | Research first]
Success = [verifiable criterion]
Out of scope: [what's deferred]

Steps:
1. [action] → verify: [check]
2. [action] → verify: [check]
3. [action] → verify: [check]
```

Keep it under 15 lines. If it needs more, the scope is too big — loop back to Round 3.

## Non-negotiable Rules

- **Use `AskUserQuestion`** with 2–4 options and a recommended default. No chains of open-ended questions.
- **Max 3–4 rounds.** If you're still asking after round 4, you're interrogating, not clarifying. Propose a plan with your best guesses and let the user correct it.
- **Skip rounds aggressively** when the user's message already contains the answer. A user who says *"I want to add JWT auth to my Express API, I've done it twice before, I just want help wiring it up fast"* has already answered Rounds 1, 2, and 4 — jump straight to Round 3.
- **Exit clean on bypass signals.** If the user says `"just do it"`, `"hazlo"`, `"just implement"`, `"confío en ti"`, `"stop asking"`, or equivalent — stop the skill, summarize your current assumptions in one paragraph, and proceed.
- **Never silently pick** between approaches when more than one is reasonable. Name them, show tradeoffs, ask.
- **Declare unknowns.** If you don't know something, say so before asking the user. Don't bluff.

## When NOT to Trigger

This skill should stay quiet for:

- Trivial edits, typo fixes, single-line changes, reformatting.
- Pure knowledge questions (`cómo funciona useEffect`, `what is a monad`). The user wants an explanation, not a task plan.
- Tasks where the user already provided scope, criteria, and approach in the initial message.
- Explicit bypass (`just do it`, `hazlo`, etc.) even if the task is complex — respect the user's call.

## Example

**User:** *"Tengo que agregar auth al proyecto"*

**Round 1 (task shape):**
- Q1: *"¿Es auth desde cero o integrar con algo existente?"* → `Desde cero` / `Integrar con proveedor (Auth0, Clerk, etc.)` / `Extender lo que ya hay`
- Q2: *"¿Ya tienes elegido el enfoque?"* → `Sí, solo necesito ayuda implementando` / `No, ayúdame a decidir`

**Round 2 (knowledge):**
- Q3: *"¿Has implementado auth antes en este stack?"* → `Sí, cómodo` / `Algo de experiencia` / `Primera vez`
- Agent declares: *"Asumo que es una app web con sesiones server-side. No sé si ya tienes users en DB ni qué proveedor de email usas. Si es Next.js App Router, recomendaría Auth.js; si es Express puro, una combinación de Lucia + argon2."*

**Round 3 (scope):**
- Q4: *"¿Qué flujos mínimos entran en este ciclo?"* → `Solo login/logout` / `+ signup` / `+ password reset` / `Todo lo anterior`

**Round 4 (mode):**
- Q5: *"¿Cómo prefieres que trabajemos?"* → `Orquestar` / `Acelerar` / `Investigar primero` (default basado en Q3).

**Final plan** (5 lines): tarea, modo, criterio de éxito, fuera de alcance, 3 pasos verificables.

## Troubleshooting

### Skill triggers on knowledge questions

**Symptom:** user asked *"cómo funciona X"* and the skill started interrogating.
**Fix:** knowledge questions are not tasks. Answer the question; don't plan.

### Skill over-interrogates a clear task

**Symptom:** user gave full context, agent still runs all 4 rounds.
**Fix:** re-read the initial message before asking. If Rounds 1–3 are already answered, jump to Round 4 or straight to the plan.

### User gets impatient mid-interrogation

**Symptom:** user says *"just do it"* or goes silent.
**Fix:** stop immediately. State your current assumptions in one paragraph and proceed to execute. The skill served its purpose — it exposed the assumptions even if the user skipped the remaining rounds.
