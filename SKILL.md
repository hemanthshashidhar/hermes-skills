---
name: token-context-optimizer
description: Optimize Hermes Agent and Codex workflows for lower token usage, lower input cost, better prompt-cache reuse, and longer useful context. Use whenever a task is becoming token-heavy, a session is long, context is filling, tool output is large, many files/tools are being inspected, repeated commands are being issued, Codex is being delegated work, model switching is being considered, or the user asks to make an agent workflow cheaper/faster/more context-efficient. Prefer concrete inspection and workflow changes over generic prompting advice. Do not optimize by sacrificing correctness, required context, security, or verification.
version: 1.0.0
author: Hemanth
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [tokens, context, prompt-cache, cost, performance, codex, optimization, compaction, delegation]
    category: productivity
---

# Token & Context Optimizer

Optimize the **whole agent loop**, not merely the wording of one prompt.

The goal is to reduce unnecessary input/output tokens and context growth while preserving task correctness, cache reuse, useful history, and verification quality.

## Core rule

**Never save tokens by throwing away information the task actually needs. Save tokens by eliminating repetition, irrelevant data, premature loading, oversized tool output, unnecessary model calls, and avoidable context churn.**

## When this skill is active

Use this skill when any of these are true:

- The session is long or context usage is rising.
- Hermes `/usage` or `/context` shows a large context footprint.
- The agent is repeatedly reading large files, logs, command output, or documents.
- Many tool calls perform small mechanical operations that could be batched.
- Multiple independent research or coding tasks can be isolated.
- The agent is considering `/compress`, `/new`, `/model`, or other context-affecting actions.
- A Codex delegation may inherit more context than it needs.
- The workflow is expensive, slow, or repeatedly hits context limits.
- The user explicitly asks to reduce tokens, cost, latency, context usage, or prompt size.

Do **not** activate merely because the user mentions tokens. If the requested task is already small and efficient, do not add optimization ceremony.

## Optimization hierarchy

Apply these in order. Stop when the workflow is already efficient enough.

1. **Remove unnecessary work.**
2. **Reduce unnecessary context entering the model.**
3. **Keep reusable prefixes stable for prompt caching.**
4. **Batch mechanical tool work.**
5. **Isolate independent work in subagents.**
6. **Compress or reset context when appropriate.**
7. **Reduce output verbosity only when it does not reduce usefulness.**
8. **Change models only when the tradeoff is clearly favorable.**

Do not start by rewriting prompts. First inspect the workflow.

---

## 1. Inspect before optimizing

For an active Hermes session, prefer local diagnostics before guessing:

- `/usage` for token/session usage.
- `/context` for the context-window breakdown.
- `/context all` when available and useful, especially to inspect skill/toolset costs.
- `/status` for session and token totals without an LLM call.

When working in a repository, inspect the project structure before reading large files.

Identify:

- large context files such as `AGENTS.md`, `CLAUDE.md`, `.hermes.md`, `.cursorrules`, and rule files;
- generated files and build artifacts that should not be loaded;
- logs, datasets, lockfiles, caches, and vendored dependencies;
- repeated tool calls producing overlapping results;
- files that can be searched for exact symbols before being opened in full.

### Diagnose the dominant source

Classify the waste before changing anything:

- **System/tool overhead:** tool schemas, skill index, MCP definitions, context files.
- **History overhead:** old turns, old tool results, repeated explanations.
- **Tool-result overhead:** giant file reads, logs, search results, command output.
- **Workflow overhead:** too many small tool calls or repeated failed attempts.
- **Model overhead:** expensive model used for trivial work or unnecessary auxiliary calls.
- **Cache churn:** model/provider changes or rewritten prefixes breaking cache reuse.

Optimize the largest avoidable source first.

---

## 2. Protect prompt-cache reuse

Prompt caching rewards a stable prefix. Treat the reusable prefix as infrastructure.

### Do

- Keep stable instructions, tool definitions, project guidance, and reusable reference material unchanged when possible.
- Append new conversation turns rather than rewriting earlier context.
- Keep tool definitions and ordering stable.
- Avoid unnecessary model/provider switches during a long session.
- When changing models for a substantially different task, consider starting a fresh session instead of repeatedly bouncing between models.
- Keep dynamic, frequently changing information out of stable reusable material when the runtime allows it.

### Do not

- Reformat or regenerate a large instruction block every turn without a reason.
- Frequently switch models/providers in a long session merely to save a few output tokens.
- Put volatile timestamps, transient status, or per-turn scratch data into reusable instruction material when avoidable.
- Assume that a session automatically guarantees a cache hit.

### Important tradeoff

Compaction can reduce the number of tokens in context while also changing the prefix and reducing immediate cache reuse. Evaluate **total cost and latency**, not cache-hit percentage alone.

For OpenAI GPT-5.6+ workflows, use the provider's current prompt-caching behavior as the source of truth rather than hard-coding old cache thresholds or discounts. For Hermes itself, follow its configured transport and compression implementation.

---

## 3. Read less, but read the right thing

Large tool results are one of the easiest ways to poison future context.

### File strategy

Prefer:

1. list/tree/search;
2. exact symbol or phrase search;
3. targeted range read;
4. full-file read only when necessary.

For logs:

- inspect the tail first for the current failure;
- search for `ERROR`, `Traceback`, failed test names, status codes, or relevant timestamps;
- read surrounding lines around the match;
- only read the complete log when chronology is genuinely required.

For structured data:

- filter, aggregate, or summarize before returning data to the model;
- use scripts for repetitive parsing;
- return statistics, selected rows, and anomalies rather than the whole dataset.

For code:

- inspect the relevant module and call graph before dumping unrelated files;
- search for definitions/usages first;
- avoid reading generated files, dependencies, lockfiles, and build output unless directly relevant.

### Never use truncation as a substitute for understanding

A blind `head -100` or `tail -100` is not automatically efficient. If the relevant information is elsewhere, it merely creates a cheap wrong answer.

---

## 4. Batch mechanical work

If several tool calls are deterministic and independent, batch them.

Prefer one script or `execute_code` operation for:

- checking many files;
- extracting repeated metadata;
- renaming or transforming many files;
- collecting git status/diff information;
- parsing multiple logs;
- computing statistics;
- making many small filesystem checks.

Keep intermediate tool results out of the model context when the execution mechanism supports it. Return only the compact result needed for reasoning.

Do not batch actions that require independent reasoning, risky side effects, or user approval merely to save tokens.

---

## 5. Delegate independent work strategically

Use isolated subagents when independent workstreams would otherwise pollute the main context.

Good candidates:

- researching several independent sources/topics;
- inspecting separate modules;
- comparing multiple implementation options;
- running independent tests or audits;
- reviewing a completed change with fresh context.

Give each child only the context it needs. Do not paste the entire parent conversation into every delegation request.

Ask children for **compact, decision-oriented summaries**:

- finding;
- evidence/location;
- recommendation;
- unresolved uncertainty.

Do not delegate a trivial one-command task just because delegation exists. The subagent itself has overhead.

When Hermes supports parallel delegation, prefer parallel independent tasks over serial context accumulation.

---

## 6. Use compaction deliberately

Compaction is a context-management operation, not a universal cost-saving button.

### Use `/compress` when

- context is becoming large;
- old tool output is no longer needed verbatim;
- the session is still logically one task;
- preserving the current working thread is more useful than starting fresh.

### Prefer a fresh session when

- the task has changed substantially;
- the old conversation is mostly irrelevant;
- the old context is large but contains little reusable state;
- model/provider switching would otherwise repeatedly invalidate cache reuse.

### Before compressing

Preserve durable facts that must survive:

- current objective;
- constraints;
- decisions already made;
- files changed;
- tests run and results;
- important errors and their causes;
- exact next step;
- user preferences that matter to the current task.

Never claim compaction is lossless. Verify important facts after compaction when correctness matters.

Do not enable or recommend micro-compaction solely because it sounds efficient. Continuous compression can add auxiliary model calls and may interfere with prompt-cache reuse.

---

## 7. Keep skills and persistent instructions lean

Skills are themselves context. Their purpose is to provide reusable procedures, not to become encyclopedias.

When authoring or improving a skill:

- make the frontmatter description specific enough for reliable routing;
- keep the main `SKILL.md` focused on the workflow;
- move large reference material to `references/`;
- load reference files only when needed;
- avoid duplicating the same instructions across multiple skills;
- do not place stable factual memory into a skill if it belongs in memory/context files instead;
- do not add examples merely to make the skill look sophisticated.

Prefer progressive disclosure:

`metadata → SKILL.md → targeted reference`

The optimizer itself must follow this rule. Never load a large reference just to quote optimization theory.

---

## 8. Optimize Codex tasks

When Hermes delegates to Codex:

### Give Codex a clean task boundary

Include:

- exact objective;
- relevant files/directories;
- constraints;
- expected verification;
- important existing decisions.

Avoid including:

- the entire conversation;
- irrelevant project history;
- huge logs when a focused error excerpt is enough;
- repeated instructions already available through repository guidance.

### Let Codex inspect when inspection is cheap

Do not pre-dump an entire repository into the prompt. Point Codex at the relevant project and tell it what to investigate.

### Batch related changes

If several changes are tightly coupled, one Codex task is often better than multiple separate tasks that each rediscover the repository.

If tasks are genuinely independent, separate them so each receives a clean context.

### Verify efficiently

Prefer targeted tests first, then broader tests when justified. Do not run a giant test suite after every microscopic edit unless project requirements demand it.

Never remove verification solely to reduce token usage.

---

## 9. Model selection and output control

Use the cheapest capable reasoning path, not the cheapest model by default.

A practical hierarchy:

- deterministic shell/script operation → use tools/scripts;
- simple transformation → fast/cheap capable model;
- straightforward coding → coding-capable model;
- architecture, security, ambiguous debugging → stronger reasoning model;
- independent research → parallel isolated agents when useful.

Avoid frequent model switching inside a long cached session unless the quality/cost benefit is substantial.

For generated output, request the amount of detail needed for the decision. Do not ask for giant explanations when a concise result is sufficient.

Do not force terse output when the task requires reasoning, evidence, code, or verification.

---

## 10. Anti-patterns

Reject these optimization patterns:

- **Prompt shrinking by deletion:** removing constraints until the agent becomes cheaper and wrong.
- **Blind compression:** compressing context without preserving active state.
- **Repeated summarization:** summarizing the same information over and over.
- **Context dumping:** reading whole repositories/files because it feels safe.
- **Tool-call fragmentation:** using ten calls where one script can do the mechanical work.
- **Delegation spam:** spawning agents for tiny tasks.
- **Cache superstition:** assuming every unchanged-looking turn is cached or every cache miss is a bug.
- **Model ping-pong:** switching models repeatedly in a long session.
- **Reference hoarding:** loading every skill reference “just in case.”
- **Premature optimization:** spending more tokens analyzing token savings than the workflow would ever consume.

---

## 11. Standard optimization procedure

When asked to optimize a workflow, follow this sequence:

### A. Measure

Inspect `/usage` and `/context` when available. Identify the largest context/cost contributors.

### B. Classify

Label each contributor as system/tool overhead, history, tool output, workflow calls, model choice, or cache churn.

### C. Remove

Eliminate irrelevant reads, duplicate calls, repeated explanations, unnecessary searches, and redundant model calls.

### D. Restructure

Use targeted reads, scripts, batching, parallel delegation, and clean task boundaries.

### E. Protect cache

Keep stable prefixes and tool definitions stable. Avoid unnecessary model/provider changes.

### F. Compact or reset

Choose `/compress` or a new session only after deciding what state must survive.

### G. Verify

Confirm the optimized workflow still produces the same required result and verification coverage.

### H. Report

When the user asks for an optimization result, report:

- **Before:** main source of waste.
- **Change:** what was altered.
- **Expected effect:** fewer tokens/calls, better cache reuse, lower latency, or longer context.
- **Tradeoff:** what may become worse.
- **Verification:** how to measure whether it actually improved.

Do not invent percentage savings. If there is no measurement, say the improvement is expected rather than measured.

---

## 12. Quick decision tree

```text
Is the workflow expensive or context-heavy?
        |
       yes
        |
   inspect usage/context
        |
        v
What dominates?
  |       |        |        |        |
 history  tools   workflow  model    cache
  |       |        |        |        |
compress  narrow  batch    right     stabilize
/reset    results calls    model     prefix
  |       |        |        |        |
  +-------+--------+--------+--------+
                  |
               verify
```

## Success criteria

A successful optimization:

- uses less irrelevant context;
- avoids unnecessary tool/model calls;
- preserves required information and verification;
- improves or preserves cache reuse where applicable;
- leaves a workflow that is simpler, not merely more clever;
- can be measured with Hermes `/usage`, `/context`, provider usage data, latency, or task-success evaluation.

When there is a conflict, **correctness and required task context beat token savings**.
