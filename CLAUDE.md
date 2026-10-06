# mini-chatsmith — Project Context

Learning project: Chat Engine → User Segmentation → Notification, built in 7 days on Go + PostgreSQL + GCP.
Detailed Phase 1 plan: `docs/phase1-plan.md`.

## Timeline (whole project = 1 week)
| Days | Phase | Visible result |
|---|---|---|
| 1–3 | Phase 1 — Chat, SSE, history | Chat with OpenAI/Claude on Cloud Run, answer streams, history survives restart |
| 4–5 | Phase 2 — Segmentation | Parquet import into versioned snapshots, segment query via REST + gRPC |
| 6–7 | Phase 3 — Notification | Campaign → audience via gRPC → lazy fan-out inbox → Fake FCM |

## Locked decisions
- D1: Chat history = `conversations` + `chat_messages`. No `chat_history` table (would duplicate data).
- D2: PostgreSQL is the source of truth for history. Never rely on provider-side conversation state.
- D3: No DB write per token. Write once at the end (completed) or on cancel/error (incomplete).
- D4: SSE `done` is sent only after the final transaction commits.
- D5: Real providers (OpenAI + Anthropic). Mock provider lives only in tests.
- D6: Provider model IDs come from config/DB (`provider_model`), never hard-coded.
- D7: No automatic provider fallback after the first delta was streamed.
- D8: API keys only in `.env` (gitignored) locally and Secret Manager on GCP.

## Stack
Go 1.25 · Gin · PostgreSQL 16 · GORM · golang-migrate (plain SQL) · Zap JSON · UUID ·
openai-go · anthropic-sdk-go · Docker Compose · Cloud Run · Cloud SQL · Secret Manager · Artifact Registry.

## Layout
`cmd/app` · `configs` · `internal/<domain>/{model,dto,repository,service,module}.go` ·
`internal/provider/{openai,anthropic,mock}` · `internal/{controller,router,middleware,server}` ·
`pkg/{database,logger,response,sse}` · `migrations/`.

Layer rules: controller = HTTP only; service = business rules; repository = SQL only; provider = AI vendor adapter.
