# Workers AI Model Deprecation Migration — cloudless.gr

Remediation for Cloudflare's "Workers AI Model Deprecation November 2026" notice
(models removed **2026-11-20**).

Source of truth for code changes: [`Themis128/cloudless.gr`](https://github.com/Themis128/cloudless.gr)
PR [#2371](https://github.com/Themis128/cloudless.gr/pull/2371).

## Live usage audit (pre-migration)

Scanned all Workers on account `fb7dc7b69b662480cd5961a4d1913c78` for
`@cf/…` model references in deployed bundles + repo source.

| Worker | Env | Deprecated model | Status |
|---|---|---|---|
| `cloudless-web-production` | production | `@cf/deepseek-ai/deepseek-r1-distill-qwen-32b` | ✅ migrated (PR #2371) |
| `cloudless-gr-staging` | staging | `@cf/deepseek-ai/deepseek-r1-distill-qwen-32b` | ✅ migrated (same code path) |
| `cloudless-monorepo-production-mainnextappscript` | production | `@cf/meta/llama-3.2-3b-instruct` (NLP fallback default) | ✅ migrated (PR #2371) |
| `interview-coach` | prod | `@cf/myshell-ai/melotts` (`/tts`) | ✅ migrated + deployed 2026-11-06 — source recovered to [Themis128/interview-coach](https://github.com/Themis128/interview-coach), `/tts` now uses `@cf/deepgram/aura-2-en` |

Verified NOT deprecated (no action):
`@cf/meta/llama-3.1-8b-instruct-fast`, `@cf/meta/llama-3.3-70b-instruct-fp8-fast`,
`@cf/baai/bge-small-en-v1.5`, `@cf/deepgram/nova-3`, `@cf/openai/whisper-large-v3-turbo`.

## Model replacement map applied

| Deprecated | Replacement | Rationale |
|---|---|---|
| `@cf/deepseek-ai/deepseek-r1-distill-qwen-32b` | `@cf/deepseek-ai/deepseek-v4.1-flash-1017` | Cloudflare frontier recommendation; chat model |
| `@cf/meta/llama-3.2-3b-instruct` | `@cf/qwen/qwen3.8-27b` | Cloudflare "smaller model" recommendation; still env-overridable via `WORKERS_AI_NLP_MODEL` |
| `@cf/myshell-ai/melotts` | `@cf/deepgram/aura-2-en` | Cloudflare TTS recommendation — **requires request/response adaptation** |

## Code changes (cloudless.gr PR #2371)

### `src/agents/coding.ts`
- `AI.run()` model string swapped.
- Input migrated from legacy `{ prompt }` to chat `{ messages: [system, user] }`
  (v4.1-flash is a chat-completions model).
- `extractText()` extended to read `result.choices[0].message.content`
  (previously only `response`/`text`/`result` string fields).

### `src/lib/nlp/analyze-lead.ts`
- `NLP_MODEL` default swapped; `WORKERS_AI_NLP_MODEL` env override preserved.
- 3s timeout + Zod JSON parse + local-heuristic fallback unchanged.

### Docs
- `.claude/skills/contact-nlp/SKILL.md`, `docs/ai/contact-nlp.md` updated.

## Still TODO

1. Post-deploy smoke: coding-agent run + contact-form NLP on production.
2. Long-term: dedicated Google Cloud OAuth client_id for rclone `gdrive:` remote
   (shared client_id retires during 2026) — unrelated but flagged in same window.

## Verification commands

```bash
# confirm no deprecated IDs remain in app source
grep -rn "deepseek-r1-distill-qwen-32b\|llama-3.2-3b-instruct\|melotts" src/

# live worker model probe (needs CF API token)
curl -s "https://api.cloudflare.com/client/v4/accounts/$CF_ACCOUNT_ID/ai/run/@cf/qwen/qwen3.8-27b" \
  -H "Authorization: Bearer $CF_API_TOKEN" -d '{"messages":[{"role":"user","content":"ping"}]}'
```
