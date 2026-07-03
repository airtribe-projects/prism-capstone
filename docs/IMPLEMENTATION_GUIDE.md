# Prism Implementation Guide

This guide gives you a practical path through the capstone. It does not add new requirements. The source of truth for scope is `PRISM_PROBLEM_STATEMENT.md`.

## Must Have Checklist

Before working on Good To Have or Stretch items, make sure you can check off every item below:

- [ ] Load the provided pricing, seed keys, and gateway configuration.
- [ ] Serve OpenAI-compatible completions through 2 providers (the mocks count).
- [ ] Return the `x-prism-*` header contract on every data-plane response.
- [ ] Stream token by token, terminating with `data: [DONE]`, with usage still recorded.
- [ ] Reject invalid keys, disallowed models, rate-limit breaches, and exhausted budgets with distinct errors.
- [ ] Resolve `fast`/`smart` aliases; retry with backoff; fail over when a provider is down.
- [ ] Serve semantically similar repeats from a per-key cache.
- [ ] Meter cost per request from provider usage and the price table.
- [ ] Store a request log entry for every request, including rejections.
- [ ] Expose a usage summary API that reconciles with the load test's client-side totals.
- [ ] Demo usage, recent requests, and cache hit rate in a simple ops console.
- [ ] Pass `scripts/smoke_test.py`; run `scripts/load_test.py` with zero over-admission.

## Suggested Build Order

### Step 1: Run the Mock Providers

```bash
python3 scripts/mock_provider.py --port 9001 --name alpha
python3 scripts/mock_provider.py --port 9002 --name beta
```

Send them a few curl requests (streaming and non-streaming) before writing any gateway code, so you know exactly what an upstream looks like.

### Step 2: Core Pass-Through

- Load `data/model_pricing.json`, `data/seed_keys.json`, and your provider/alias config.
- Implement `POST /v1/chat/completions` against a single provider, non-streaming.
- Add the header contract now — retrofitting it later touches every code path.

### Step 3: Keys, Limits, Budgets

- Authenticate virtual keys; enforce the model allowlist.
- Add a race-safe requests-per-minute limiter (token bucket or atomic sliding window).
- Meter cost per request; maintain per-key monthly spend; reject when the budget is gone.
- Log every request, including rejections.

### Step 4: Streaming

- Forward upstream SSE chunks as they arrive; do not assemble the full response first.
- Capture `usage` from the final upstream chunk so streamed requests are metered too.
- Decide your mid-stream failure behavior and document it.

### Step 5: Routing, Retries, Failover

- Resolve aliases to primary + ordered fallbacks from your config.
- Retry transient errors with exponential backoff; on a down or rate-limited provider, move to the next in the chain.
- Set `x-prism-fallback: true` and log fallback events.
- Verify with the mock's failure injection before moving on.

### Step 6: Semantic Cache

- Start with normalized exact-match to get the plumbing (per-key scope, hit headers, stats) right.
- Swap the matcher for similarity: an embeddings API, a local embedding model, or a documented substitute.
- Tune the threshold until `req_cache_a1`/`req_cache_a2` hit and `req_cache_a3` (different intent) does not.
- Only cache successful responses. Respect per-key cache settings from the seed data.

### Step 7: Usage API and Ops Console

- Usage summary by key; recent request logs; cache stats.
- A simple page showing those three things is a complete console. You may vibe-code it.

### Step 8: Verify

- Smoke test green.
- Load test: zero over-admission, totals reconciled against your usage API.
- Automated tests for cost math, the limiter, and the cache decision.

## Recommended Demo Scenarios

### Scenario 1: Normal Traffic and Cache

Use `prism-sk-search-1a2b3c` with the `fast` alias.

Expected demo:

- Non-streaming request shows `x-prism-provider`, `x-prism-cache: miss`, and a cost.
- A streaming request visibly arrives token by token.
- The same prompt again: `x-prism-cache: hit`, cost `0`.
- The paraphrase (`req_cache_a2`): also a hit — that is the semantic part.

### Scenario 2: Limits and Budgets

Use `prism-sk-free-7g8h9i` (10 requests/minute) for rate limits and `prism-sk-budget-demo-0j1k2l` (tiny budget, cache disabled) for budgets.

Expected demo:

- A burst past 10 requests/minute with the free-tier key returns clean `rate_limit_exceeded` errors.
- With the budget-demo key, the first request consumes the tiny budget and the second returns `budget_exceeded` — no setup needed.
- Both rejection types appear in the request log and console.

### Scenario 3: Provider Failure

Expected demo:

- `POST /admin/config {"mode": "down"}` on alpha; the next `fast` request is served by beta with `x-prism-fallback: true`.
- Make alpha slow instead (`latency_ms: 3000`); show your timeout keeps the client from hanging.
- Restore alpha; traffic returns to the primary.

## FAQ

### Do I need a real AI provider?

No. The two mock providers are a complete setup for development, demos, and load testing. Adding a real provider (or Ollama) is optional.

### Do I need real embeddings for the semantic cache?

No. An embeddings API or local model is ideal, but a clearly documented local substitute (for example normalized bag-of-words cosine) is acceptable if the provided paraphrase pairs hit. Exact-match-only is not enough.

### What counts as "semantic"?

The paraphrase pair `req_cache_a1`/`req_cache_a2` should hit; the near-miss `req_cache_a3` (password reset vs 2FA reset) should not. If your matcher achieves that, it qualifies.

### Can rate limiting be in memory?

Yes, for a single gateway instance — as long as it is race-safe under concurrent requests. Distributed rate limiting across instances is Stretch.

### How do budgets reset?

A simple calendar-month window with reset-on-read is fine. Document your choice.

### What happens if a provider dies mid-stream?

Decide and document. Acceptable Must Have behavior: terminate the client stream with an error event. Silently restarting on another provider and splicing outputs is not acceptable.

### Can I mock providers in automated tests?

Yes — that is why the provider integration must sit behind an adapter. Point it at the mock providers or an in-process fake.

### How do I measure the gateway's added latency?

Coarse is fine: timestamp before/after the upstream call and subtract from total request time. Report the number in your verification report.

### Can I change API route names?

Admin routes, yes — document them. The data plane must stay `POST /v1/chat/completions`, OpenAI-compatible, with the `x-prism-*` headers, because the scripts depend on it.

### Does the console need authentication or polish?

A simple demo token is fine. Polish is not graded; the console exists to make usage, logs, and cache stats visible in your demo.

### What should be in the final README?

Include:

- setup instructions,
- environment variables,
- how to seed keys and configure providers,
- how to run the app and the mock providers,
- API overview,
- how to run the smoke and load tests,
- architecture/design decisions (routing, cache scoping, budget accounting),
- known limitations.
