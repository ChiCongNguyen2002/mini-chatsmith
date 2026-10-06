# Phase 1 — Chat, SSE và Lịch sử

## Goals

Xây một Chat Engine thật, gọi được OpenAI và Claude, stream câu trả lời qua SSE, lưu lịch sử vào PostgreSQL, deploy lên Cloud Run.

**Sau 3 ngày, user làm được:**

1. Chọn model (Free hoặc PRO).
2. Tạo cuộc chat mới.
3. Gửi câu hỏi → nhận câu trả lời xuất hiện dần (SSE).
4. Refresh trang → lịch sử vẫn còn.
5. Restart server → lịch sử vẫn còn.
6. Mở conversation cũ → hỏi tiếp với đầy đủ ngữ cảnh.
7. User Free không chọn được model PRO.
8. AI lỗi giữa chừng → câu hỏi không mất, câu trả lời ghi trạng thái incomplete.

---

## Tại sao làm

- Học cách thiết kế một hệ thống chat hoàn chỉnh: không chỉ gọi API mà còn quản lý trạng thái, lưu trữ, streaming.
- Hiểu SSE từ gốc: server gửi delta → client render dần → server commit DB → gửi done.
- Hiểu vì sao DB là source of truth, không phải provider.
- Nền tảng cho Phase 2 (Segmentation đọc dữ liệu chat) và Phase 3 (Notification gửi cho nhóm user).
- Học GCP thực tế: Cloud Run + Cloud SQL + Secret Manager.

---

## Làm như nào

### Database (5 bảng)

```
users (id, name, plan, created_at)
ai_models (id, display_name, provider, provider_model, required_plan, enabled, max_output_tokens, created_at)
conversations (id, user_id, title, created_at, updated_at)
chat_messages (id, conversation_id, generation_id, role, content, status, created_at)
generations (id, conversation_id, user_message_id, assistant_message_id, model_id, provider, provider_model, provider_request_id, input_tokens, output_tokens, finish_reason, status, error_code, started_at, completed_at)
```

### Luồng streaming

```
User gửi câu hỏi
  ↓
TX1: lưu user message + tạo assistant message (streaming) + tạo generation (running)
  ↓
Đọc history (20 message gần nhất, chỉ completed) → chuyển sang format provider
  ↓
Gọi OpenAI hoặc Claude streaming
  ↓
Nhận delta → gửi SSE cho client + nối vào buffer (không ghi DB)
  ↓
Provider xong:
  TX2: lưu assistant content + usage + completed → gửi SSE done
Provider lỗi:
  TX2: lưu phần đã nhận + incomplete/failed → gửi SSE error
Client disconnect:
  Cancel context → dừng provider upstream → lưu phần đã nhận + incomplete/cancelled
```

### Provider interface

```go
type Provider interface {
    Stream(ctx context.Context, req StreamRequest) (<-chan ProviderEvent, error)
}

type ProviderEvent struct {
    Type    EventType // delta | usage | done | error
    Delta   string
    Usage   *Usage
    Finish  string
    Error   error
}
```

OpenAI adapter: Responses API, stream=true.
Claude adapter: Messages API, NewStreaming.
Mock adapter: chỉ dùng trong test.

### API

```
GET  /healthz
GET  /readyz

GET  /v1/models

POST /v1/conversations
GET  /v1/conversations
GET  /v1/conversations/:id/messages

POST /v1/conversations/:id/generations
GET  /v1/generations/:id
POST /v1/generations/:id/cancel
```

### SSE events gửi cho client

```
event: accepted     data: {"generation_id": "..."}
event: delta        data: {"text": "Xin "}
event: delta        data: {"text": "chào "}
event: delta        data: {"text": "Công!"}
event: usage        data: {"input_tokens": 42, "output_tokens": 15}
event: done         data: {}
event: error        data: {"code": "provider_error", "message": "..."}
```

### Giới hạn chi phí

- Max input history: 20 message gần nhất.
- Max output tokens: theo config từng model.
- Generation timeout: 60 giây.
- Mỗi user: 1 generation đồng thời.
- Không tự retry sau khi đã nhận delta.

### GCP

```
Artifact Registry → Cloud Run → Cloud SQL PostgreSQL
                         ↑
                   Secret Manager (openai-api-key, anthropic-api-key)
```

Cloud Run: asia-southeast1, timeout 120s, concurrency 20, min 0, max 2.
DB pool: max open 5, max idle 2, lifetime 30 phút.

---

## Workflow (3 ngày)

### Ngày 1 — Storage

| Bước | Verify |
|---|---|
| Source base: cmd/app, configs, internal, pkg, migrations | `go build ./...` pass |
| Docker Compose: app + PostgreSQL | `docker compose up` chạy |
| 5 migration files | `migrate up` tạo đúng 5 bảng |
| Seed: 2 user (free, pro), 2 model | `SELECT * FROM users` trả 2 row |
| Config, logger (Zap JSON), database (GORM) | App log JSON, connect DB |
| `/healthz`, `/readyz` | `curl` trả 200 |
| Conversation CRUD: create, list, get messages | `curl POST` tạo conversation, `curl GET` trả list |
| Message repository: save, list by conversation | Insert + select đúng |
| Conversation ownership check | User A không đọc được conversation user B |

**Ngày 1 done khi:** tạo conversation, lưu message thủ công, restart app, đọc lại đúng.

### Ngày 2 — Provider thật

| Bước | Verify |
|---|---|
| Provider interface + ProviderEvent | `go build` |
| OpenAI adapter (Responses API, stream) | Gọi 1 request ngắn, nhận delta |
| Claude adapter (Messages API, stream) | Gọi 1 request ngắn, nhận delta |
| Provider registry: chọn adapter theo model | Registry trả đúng adapter |
| Mock adapter cho test | Unit test pass không cần API key |
| Normalize delta, usage, error | Cả 3 provider trả cùng ProviderEvent format |
| Lưu provider_request_id, usage vào generation | SELECT generation có đủ field |

**Ngày 2 done khi:** gọi OpenAI nhận delta, gọi Claude nhận delta, usage lưu đúng.

### Ngày 3 — SSE và GCP

| Bước | Verify |
|---|---|
| SSE writer: flush từng event | `curl` thấy delta xuất hiện dần |
| Generation service: TX1 → stream → TX2 | DB có user message + assistant message + generation |
| Success flow: completed | `curl GET messages` trả đúng nội dung |
| Error flow: incomplete + failed | Simulate lỗi → message incomplete, generation failed |
| Cancel flow: client disconnect | Đóng curl → generation cancelled, lưu phần đã nhận |
| Free/PRO check | User free chọn model PRO → 403 |
| 1 generation đồng thời | Gửi 2 request cùng lúc → request 2 bị reject |
| Test với mock provider | `go test ./...` pass |
| Dockerfile + build image | `docker build` thành công |
| Deploy Cloud Run + Cloud SQL + Secret Manager | `curl` Cloud Run URL nhận SSE |
| Đo TTFT | Log time_to_first_token cho cả OpenAI và Claude |

**Ngày 3 done khi:** gọi Cloud Run, thấy SSE delta, refresh đọc lại history, restart không mất data.

---

## Verify checklist (toàn Phase 1)

```
[ ] go build ./... clean
[ ] go vet ./... clean
[ ] go test ./... pass (mock provider)
[ ] Tạo conversation → 201
[ ] List conversations → đúng user, sắp theo updated_at
[ ] Gửi message → SSE delta xuất hiện dần
[ ] SSE done chỉ sau khi DB commit
[ ] Refresh → GET messages trả đúng history
[ ] Restart server → history vẫn còn
[ ] User free + model PRO → 403
[ ] User pro + model PRO → OK
[ ] Provider lỗi → message incomplete, generation failed, SSE error
[ ] Client disconnect → generation cancelled, lưu phần đã nhận
[ ] 2 generation cùng lúc → reject thứ 2
[ ] Generation có provider_request_id, input_tokens, output_tokens
[ ] Log JSON có generation_id, provider, duration, tokens, status
[ ] Không log API key hoặc nội dung chat
[ ] Cloud Run trả SSE đúng
[ ] Cloud SQL giữ data sau redeploy
```

---

## Done

Phase 1 hoàn thành khi luồng này chạy đúng trên Cloud Run:

```
User chọn model → tạo conversation → gửi câu hỏi
→ SSE delta xuất hiện dần
→ done sau khi DB commit
→ refresh đọc lại history
→ restart server history vẫn còn
→ mở conversation cũ hỏi tiếp với ngữ cảnh
→ lỗi không mất câu hỏi
→ free không dùng model PRO
```
