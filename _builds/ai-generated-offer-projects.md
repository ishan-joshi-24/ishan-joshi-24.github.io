---
layout: build
title: "Generate + Judge: A Two-Call LLM Pipeline for User-Facing Content"
description: "An LLM generates user-facing content. A second LLM call audits it. The audit step is the part that makes this safe to put in front of people without eroding trust over time."
---

Imagine a feature where an LLM generates a multi-step plan for a user. Four items, sequenced, with deliverables and effort estimates. The user can edit, but the hope is they edit maybe 30% of it instead of writing 100% from scratch.

The interesting part of building this isn't the generation. It's the audit.

## The setup

Inputs:

- A role / context blob
- The target user's existing skills and background
- A duration in weeks
- A bias toggle (learning-focused vs delivery-focused)

Output:

```json
{
  "items": [
    {
      "title": "...",
      "summary": "...",
      "deliverables": ["...", "..."],
      "skills_developed": ["...", "..."],
      "estimated_weeks": 3
    }
  ]
}
```

Four items, sequenced, ramp-up first, capstone last.

## Generation prompt skeleton

```
You are designing a {duration}-week plan.

User context:
- Skills: {skills}
- Background: {summary}

Role context:
- Description: {description}
- Bias: {learning_focused | delivery_focused}

Generate exactly 4 items.

Requirements:
- Item 1 is week-1 appropriate: ramp-up, low ambiguity, clear deliverable.
- Item 2 builds on item 1.
- Item 3 is the largest in scope.
- Item 4 is a capstone with a tangible artifact.
- Total duration should approximately match {duration} weeks.
- Skills developed should match existing skills plus 1-2 stretch skills.
- Do not invent specific tooling that wasn't mentioned in the role description.

Return JSON conforming to the schema.
```

"Do not invent specific tooling" is critical. Without it the model writes "Migrate the Kafka consumer to use Avro" whether or not the role description mentions Kafka or Avro.

"Approximately match duration" is the most-violated rule. Arithmetic across four estimates up to a target is hard for the model. We re-check in code.

## The audit prompt is the interesting part

Could ship the generator output and trust users to edit. We don't.

Generic output erodes trust. If the user sees a vague plan, they think "this AI is useless" and write their own, defeating the point. We'd rather refuse to ship than ship generic.

So we run a *second* LLM call that audits the first:

```
You are a senior reviewer evaluating an AI-generated plan.

Plan: {plan_json}
Context: {context}

Rate the plan on:
1. Specificity (concrete deliverables vs vague)
2. Skill match (matches user's skills, appropriate stretch)
3. Sequencing (difficulty curve)
4. Realism (duration and scope achievable)
5. Relevance (relates to role description)

Each 1-5.

Return JSON: {
  "scores": { "specificity": int, "skill_match": int, ... },
  "issues": ["...", "..."],
  "verdict": "ship" | "revise" | "regenerate"
}

Use "ship" only if all scores >= 3 and there are no critical issues.
Use "regenerate" if the plan has fundamental problems.
Use "revise" otherwise.
```

In code:

```ts
const plan = await generate(context);
const audit = await audit(plan, context);

if (audit.verdict === 'ship') return plan;
if (audit.verdict === 'regenerate') {
  return generate(context, { temperature: 0.3 });
}
return { ...plan, needs_review: true };
```

### Why a separate audit prompt instead of one big prompt

You can ask GPT to "generate and critique" in one prompt. Models will do it. Quality is much higher when you separate the two calls.

- The generator optimizes for plausible output.
- The judge optimizes for finding problems.

When you combine them, the model defends its own output. When you separate them, the judge has no skin in the generator's output. It's just rating a JSON blob.

This is the LLM equivalent of "don't merge your own PRs."

### Why three verdicts and not two

`ship` vs `regenerate` is the obvious binary. The middle case, `revise`, is where most outputs land. Mostly fine, not perfect. If we treated `revise` as `regenerate` we'd burn quota on retries unlikely to improve. If we treated it as `ship` we'd erode trust over time.

`revise` returns the output flagged. The UI shows a "review needed" badge. The user is primed to edit.

Verdict distribution in production:

- `ship` 55%
- `revise` 38%
- `regenerate` 7% (second attempt ships ~80% of the time)

So about 1.4% of requests end up needing a manual rewrite.

## What the model gets right

- Sequencing. GPT is good at "easier first, harder later".
- Skill match. Given a profile and a role, the stretch skills are reasonable.
- Variety. Four items from one prompt are meaningfully different.

## What it gets wrong

- Specific context. It doesn't know your codebase. Generated deliverables are generic by necessity.
- Time estimates. Arithmetic is hard.
- Cross-item coherence. Items are individually fine but don't always compose into a coherent arc. The sequencing rule plus the audit's "Sequencing" score catch the worst cases.

The generator-plus-audit means the worst outputs never reach the user. The mediocre outputs reach them flagged. The good ones go through.

## Generalizing

Generator-plus-judge is the part that generalizes. Anywhere you're generating user-facing content with an LLM and you can articulate what "good" looks like, a second LLM call as a judge is cheaper than building heuristic quality checks and dramatically more nuanced.

Requirement: you can write down the criteria. Three to five, each 1-5, with a forced verdict.
