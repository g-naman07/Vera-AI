# vera-bot — magicpin AI Challenge

**Team:** Naman Gupta
**Model:** Groq `qwen/qwen3.6-27b` (temperature: 0)
**Contact:** [namangupta9075@gmail.com](mailto:namangupta9075@gmail.com)

A live HTTP bot implementing the complete 5-endpoint contract defined in `challenge-testing-brief.md`. The **publicly deployed URL** is the primary interface evaluated by the judge. The repository also includes `bot.py` and `submission.jsonl` as required by `challenge-brief.md` §7.

## Approach

### 1. Context Store

* Maintains merchant context using an in-memory store.
* Supports upsert operations with versioning and `409` conflict semantics.
* Tracks suppression keys independently for each merchant.

### 2. Tick Processing

* Ranks available triggers according to their dataset-defined `urgency`.
* Selects at most one action per merchant during each tick.
* Skips actions whose suppression keys are already active.
* Generates responses using a grounded fact sheet followed by the LLM.
* Validates generated messages for prohibited content and CTA requirements.
* Falls back to deterministic templates if LLM generation or validation fails.

### 3. Reply Handling

* Detects potential auto-replies by identifying identical messages appearing across three or more conversations.
* Ends conversations when the incoming message indicates hostility or an opt-out request.
* Switches to action mode when the merchant makes an explicit commitment.
* Otherwise, generates an on-mission follow-up response.

## Design Tradeoffs

### LLM-Primary with Template Fallback

The LLM is the default generation path when `GROQ_API_KEY` is configured. Deterministic templates are used as a fallback to maintain specification compliance and reliability when the LLM is unavailable or produces an invalid response.

### In-Memory Context Store

The implementation uses an in-memory store instead of Redis. This keeps the system simple and fast for the 60-minute evaluation window and avoids additional infrastructure requirements.

### Groq Free Tier

LLM requests are throttled to approximately 30 requests per minute in `llm.ts`. Tick processing is designed to remain within the judge's 30-second request budget.

## Improvements That Would Have Helped

The following additional inputs could further improve response quality:

* Full merchant conversation history instead of only the inbound message provided to `/v1/reply`.
* Post-tick performance metrics and deltas directly available in merchant context.
* An explicit merchant-level language preference field.

## Deployment — Render

### Primary Reliability Consideration

**Cold starts are the main reliability risk on Render's free tier.**

The judge may call `/v1/healthz` at unpredictable times, and three failed health checks can result in an offline penalty.

### Option A — Always-On Deployment

For maximum reliability, change the Render service plan from:

`free` → `starter`

The Starter plan keeps the service running continuously and avoids free-tier spin-downs.

### Option B — Free Tier with Keep-Alive

If using the free tier:

1. Deploy the service to Render.
2. Open **GitHub → Settings → Secrets and variables → Actions**.
3. Add the secret:

`BOT_PUBLIC_URL=https://your-bot.onrender.com`

4. Enable the GitHub Actions workflow in `.github/workflows/keepalive.yml`.
5. The workflow sends a request to `/v1/healthz` every 10 minutes to keep the service active.

Alternatively, a scheduled service such as cron-job.org can send a `GET` request to `/v1/healthz` every 10 minutes.

## Deployment Steps

1. Push the repository to GitHub.
2. Open Render and select **New Blueprint**.
3. Connect the GitHub repository.
4. Configure the following environment variables:

   * `GROQ_API_KEY`
   * `TEAM_NAME`
   * `TEAM_MEMBER`
   * `CONTACT_EMAIL`
5. Deploy the service.
6. Verify the health endpoint:

```bash
curl https://YOUR-URL/v1/healthz
```

7. Configure either the keep-alive workflow or an always-on Render plan.
8. Submit the public bot URL through the challenge portal.

## Reliability Features

* **28-second processing limit:** Requests are designed to complete within the judge's 30-second timeout.
* **LLM warm-up:** The application initializes the LLM path during startup to reduce latency on the first request.
* **Template fast path:** If the available processing time becomes too short, the system skips LLM generation and uses a deterministic template instead of risking a timeout.

## Local Development

```bash
npm install

cp .env.example .env
# Add GROQ_API_KEY to .env

npm run dev
# Starts the server on port 8787

npm run load-dataset
# Loads the base dataset into the running bot

npm run judge-replay
# Runs the deterministic local judge

npm run generate-submission
# Generates submission.jsonl with 30 canonical test cases
```

## Submission Checklist

* [ ] Bot deployed at a public URL.
* [ ] `/v1/healthz` returns `status: ok`.
* [ ] `/v1/metadata` contains the team name, model, and contact email.
* [ ] `npm run judge-replay` completes with all tests passing.
* [ ] `submission.jsonl` generated successfully with 30 lines.
* [ ] Public URL submitted through the challenge portal.
* [ ] `bot.py`, `submission.jsonl`, and README attached if requested.

## Repository Structure

| File                              | Purpose                                                       |
| --------------------------------- | ------------------------------------------------------------- |
| `src/server.ts`                   | HTTP API and primary implementation used for judging          |
| `src/composer.ts`                 | Message composition using LLM, validation, and fallback logic |
| `bot.py`                          | Offline `compose()` implementation for batch evaluation       |
| `submission.jsonl`                | 30 canonical test-pair outputs                                |
| `judge_simulator.py`              | Local judge/replay harness                                    |
| `render.yaml`                     | Render deployment configuration                               |
| `.github/workflows/keepalive.yml` | Periodic health-check workflow                                |

## Summary

`vera-bot` combines deterministic rule-based handling with LLM-based message generation. The system prioritizes grounded responses, suppression handling, reliable fallback behavior, and strict response-time limits while maintaining compatibility with the challenge's 5-endpoint HTTP contract.
