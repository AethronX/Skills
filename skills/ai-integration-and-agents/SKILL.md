---
name: ai-integration-and-agents
description: Builds AI-powered features — chat interfaces, agents, RAG pipelines, and voice AI. Use when adding an LLM-backed feature, streaming AI responses, designing a retrieval pipeline, giving an agent tools, or building a voice assistant. For securing these features (prompt injection, output handling, data leakage) see the security-and-hardening skill — use both together.
---

# AI Integration and Agents

## Overview

Building AI-powered features is a different discipline from securing them. This skill covers *implementation*: how to structure prompts, stream responses, build retrieval pipelines, and scope agentic tool use so the feature actually works well. The `security-and-hardening` skill's "Securing AI / LLM Features" section covers *protection*: treating model output as untrusted, preventing prompt injection, keeping secrets out of context. Use both — a well-built feature that leaks data, or a secure feature that's too slow and vague to be useful, both fail.

## When to Use

- Adding a chat interface, AI assistant, or copilot feature to a product
- Building a RAG (retrieval-augmented generation) pipeline over documents/data
- Giving an LLM tools to call (agentic workflows)
- Streaming AI responses to a UI
- Building a voice AI feature (speech-to-text, text-to-speech, real-time voice agents)
- Choosing between models for cost/latency/quality tradeoffs
- Evaluating whether an AI feature's output quality is good enough to ship

## Streaming Responses

Users perceive a streamed response as faster than a blocked wait, even at equal total latency — the first token arriving matters more than the last:

```typescript
// Next.js Route Handler streaming an LLM response (SSE-style)
export async function POST(req: Request) {
  const { messages } = await req.json();

  const stream = await client.messages.stream({
    model: 'claude-sonnet-5',
    messages,
    max_tokens: 1024,
  });

  return new Response(
    new ReadableStream({
      async start(controller) {
        for await (const event of stream) {
          if (event.type === 'content_block_delta') {
            controller.enqueue(new TextEncoder().encode(event.delta.text ?? ''));
          }
        }
        controller.close();
      },
    }),
    { headers: { 'Content-Type': 'text/plain; charset=utf-8' } }
  );
}
```

- Always show a "thinking"/streaming indicator immediately — never a silent gap before the first token.
- Handle client-side abort (user navigates away or cancels) by aborting the upstream stream too, not just hiding the UI — an orphaned stream still costs tokens and money.
- Design the UI to render partial/incomplete markdown or JSON gracefully; a stream cut mid-token should not visibly break the layout.

## Prompt and Context Engineering

- **Separate the system prompt (behavior/role) from user content (task-specific input) explicitly** — don't concatenate freeform strings where a template with clear boundaries would do. This also matters for security: mixing untrusted content into the instruction channel is how prompt injection works (see `security-and-hardening`).
- **Version and test prompts like code.** A prompt change is a behavior change; review it, and run it against a fixed evaluation set before shipping (see Evaluation below).
- **Keep prompts as short as the task allows.** Every instruction competes for the model's attention; a 3,000-word system prompt with five contradictory edge-case rules performs worse than a focused one with clear priorities.
- **Few-shot examples beat abstract instructions** for format-sensitive tasks (exact output shape, tone matching) — show 2–3 examples of the desired output rather than describing it in the abstract.
- **State what "good" looks like, not just what to avoid.** "Answer in under 3 sentences, cite the source doc" beats "don't be too verbose."

## RAG (Retrieval-Augmented Generation)

```
Pipeline: Ingest → Chunk → Embed → Store → Retrieve → Rerank (optional) → Generate
```

- **Chunk by semantic boundary, not fixed character count** where possible (paragraphs, sections, headings) — a chunk that cuts a sentence in half loses meaning for both embedding and retrieval.
- **Chunk size is a tradeoff**: smaller chunks retrieve more precisely but lose surrounding context; larger chunks keep context but dilute the embedding's specificity. Start around 300–800 tokens and tune against real queries, not a guess.
- **Store metadata alongside embeddings** (source document, section, date, permissions/tenant) — you will need to filter by it, and you will need to cite it back to the user.
- **Retrieval quality is a separate problem from generation quality.** If answers are wrong, check retrieval first (are the right chunks even being returned?) before tuning the prompt — a perfect prompt over the wrong context still gives a wrong answer.
- **Always cite what was retrieved.** Show the user (or make available) which source chunks the answer drew from — this is both a trust signal and your main debugging tool when an answer is wrong.
- **Partition retrieval by tenant/permission** at the vector-store query level, not just in the prompt — this is the same trust-boundary rule as `security-and-hardening`'s LLM08 guidance and `database-design`'s multi-tenancy guidance, applied to embeddings.
- **Re-embed on content change.** A RAG pipeline over stale embeddings confidently answers with outdated facts; treat the embedding index as a cache that needs invalidation, not a one-time build.

## Agentic Tool Use

- **Scope each tool narrowly.** A tool named `run_sql` that accepts arbitrary SQL is a different risk profile than `get_order_by_id` — prefer many narrow, purpose-built tools over few powerful generic ones.
- **Validate every tool argument the model produces** as untrusted input, the same as any external user input (see `api-and-interface-design`'s boundary-validation guidance and `security-and-hardening`'s LLM06).
- **Require confirmation for destructive or irreversible actions** (sending an email, charging a card, deleting data) — an agent's plan should pause for human approval at that step, not execute autonomously.
- **Bound the loop.** Set a max number of tool calls / iterations per task so a confused agent can't spiral into an expensive or stuck loop; log every step for debugging.
- **Make failures visible to the agent, not just to logs.** If a tool call fails, the agent needs the error back in its context to retry or report — a swallowed failure produces a confidently wrong final answer.

## Model Selection and Cost/Latency Management

```
Task complexity?
├── Simple classification, extraction, formatting → smallest/cheapest capable model
├── Multi-step reasoning, nuanced writing, code    → mid/flagship model
└── Escalate only on failure                       → try cheap first, retry with a
                                                       stronger model on low-confidence
                                                       or failed output, not by default
```

- **Tier models to the task**, not uniformly to the flagship — a classification or extraction step rarely needs the most expensive model available.
- **Cache aggressively** where inputs repeat (system prompts, retrieved context, common queries) — prompt caching materially cuts cost and latency for anything with a stable prefix.
- **Set token/rate/cost budgets per feature** and alert on them (ties to `observability-and-instrumentation`'s cost-relevant metrics) — an unbounded agent loop or a viral feature can produce a surprising bill fast.

## Voice AI

- **Budget latency end-to-end**, not per component: speech-to-text → LLM → text-to-speech each add delay; a real-time conversational feel generally needs the full round trip under roughly 500ms–1s, which usually means streaming every stage rather than waiting for each to fully complete before starting the next.
- **Handle interruption/turn-taking explicitly.** A voice agent that keeps talking over the user, or can't tell when the user has finished speaking, feels broken regardless of how good the underlying model is — use voice-activity detection and let the user interrupt playback.
- **Design for transcription errors.** STT will occasionally mis-hear; don't let a single garbled transcript derail an entire multi-turn flow — confirm ambiguous or high-stakes inputs (amounts, names, irreversible commands) back to the user before acting.
- **Have a fallback to text/human handoff** for when voice recognition or the model is clearly failing a user — don't trap them in a loop with no escape.

## Evaluation

Ship an AI feature with a way to tell if it's actually working, before and after every prompt or model change:

- **Build a golden test set**: real (or realistic) inputs with known-good expected outputs or graded criteria, run automatically on every change to the prompt, model, or retrieval pipeline.
- **Sample production output for human review** on an ongoing basis — automated evals catch regressions on known cases; human sampling catches the failure modes you didn't think to test.
- **Track a small number of concrete quality metrics** (task success rate, citation accuracy for RAG, user thumbs-up/down, escalation-to-human rate) rather than relying on vibes.
- **Treat a prompt/model change like any other production change** — it needs the same before/after measurement discipline as `performance-optimization`'s keep-or-revert rule: if the eval score doesn't move (or moves inside noise), don't ship the change.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "We'll just use the biggest model for everything" | Cost and latency scale with it; most sub-tasks (classification, extraction) don't need flagship reasoning. |
| "The prompt works in my testing" | One manual test is an anecdote. A golden eval set catches regressions the next prompt tweak introduces. |
| "RAG isn't working, let's improve the prompt" | Usually a retrieval problem in disguise. Check what was actually retrieved before touching the prompt. |
| "The agent can just figure out the right tool arguments" | Models produce malformed or out-of-range arguments regularly. Validate them like any other untrusted input. |
| "Voice just needs a good STT and TTS model" | Turn-taking, interruption handling, and end-to-end latency budget matter as much as transcription/synthesis quality. |
| "We don't need citations, users trust the answer" | Citations are also your primary debugging tool — without them, a wrong answer gives no trace of why. |

## Red Flags

- No streaming — users staring at a blank state until the full response is ready
- System prompt and untrusted user/retrieved content concatenated with no clear boundary
- RAG pipeline with no visibility into which chunks were retrieved for a given answer
- Vector store queried without a tenant/permission filter
- Agent tools that accept broad, unvalidated arguments (raw SQL, arbitrary shell commands)
- Destructive tool actions executed without a confirmation step
- No bound on agent loop iterations
- No evaluation set — prompt/model changes shipped on vibes
- Voice feature with no fallback when recognition or intent confidence is low

## Verification

- [ ] Responses stream to the UI; client-side abort cancels the upstream request
- [ ] System instructions and untrusted content (user input, retrieved documents, tool output) are structurally separated, not concatenated as one string
- [ ] RAG answers can be traced back to the specific retrieved chunks
- [ ] Retrieval is filtered by tenant/permission at the query level
- [ ] Every agent tool validates its arguments before acting
- [ ] Destructive/irreversible tool actions require explicit confirmation
- [ ] Agent loops have a maximum iteration/cost bound
- [ ] A golden evaluation set exists and runs on prompt/model/retrieval changes
- [ ] Cost and latency are tracked per AI feature, with an alert threshold
- [ ] (Voice) end-to-end latency is measured, and interruption/turn-taking is handled
