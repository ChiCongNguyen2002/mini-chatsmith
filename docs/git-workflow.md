# Git Workflow — mini-chatsmith

Adapted from Anfin Gitflow. Solo project, no Jira — dùng feature name mô tả nội dung.

---

## Branches

| Branch | Vai trò |
|---|---|
| `main` | Production-ready, release history |
| `develop` | Integration, chứa toàn bộ dev history |
| `feature/<tên>` | Một thay đổi nhỏ, gọn, tạo PR vào develop |
| `hotfix/<tên>` | Patch nhanh trên main, merge vào cả main + develop |

Mỗi feature branch = **1 PR nhỏ**, review được trong 15–30 phút.

---

## Branch naming

```
feature/project-structure
feature/migration-and-seed
feature/conversation-crud
feature/provider-openai
feature/provider-anthropic
feature/sse-streaming
feature/generation-flow
feature/cloud-run-deploy
hotfix/fix-sse-flush
```

Tên mô tả nội dung, không dùng số thứ tự (`feature-1`, `task-2`).

---

## Feature branch breakdown — Phase 1

| # | Branch | Nội dung | Depends on |
|---|---|---|---|
| 1 | `feature/project-structure` | go mod, cmd/app, configs, pkg/*, Dockerfile, docker-compose, Makefile, .gitignore | — |
| 2 | `feature/migration-and-seed` | 5 migration SQL, seed user + model | #1 |
| 3 | `feature/conversation-crud` | model, dto, repo, service, controller cho conversation + message | #2 |
| 4 | `feature/provider-openai` | Provider interface, OpenAI adapter, registry | #1 |
| 5 | `feature/provider-anthropic` | Claude adapter, mock provider cho test | #4 |
| 6 | `feature/sse-streaming` | SSE writer, generation service (TX1→stream→TX2), cancel/error | #3 + #5 |
| 7 | `feature/cloud-run-deploy` | Cloud Run + Cloud SQL + Secret Manager config | #6 |

```
main
  │
  └─► develop
        │
        ├── feature/project-structure ──── PR #1
        ├── feature/migration-and-seed ─── PR #2
        ├── feature/conversation-crud ──── PR #3
        ├── feature/provider-openai ────── PR #4
        ├── feature/provider-anthropic ─── PR #5
        ├── feature/sse-streaming ──────── PR #6
        └── feature/cloud-run-deploy ───── PR #7
                                            │
develop ◄───────────────────────────────────┘
  │
main ◄──── merge develop sau khi Phase 1 verify xong
```

---

## Commit message format

```
<scope>: <message>

Ví dụ:
project: init go module and directory layout
migration: create users and ai_models tables
conversation: add create and list repository
provider: implement OpenAI streaming adapter
sse: add writer with flush support
deploy: add Cloud Run dockerfile and config
docs: add Phase 1 plan
hotfix: fix nil pointer in generation cancel
```

Scope = domain hoặc layer bị thay đổi chính. Không viết hoa đầu câu.
Một branch có thể có nhiều commit nhỏ (mỗi commit verify được 1 bước).

---

## Workflow

### 1. Tạo feature branch từ develop

```bash
git checkout develop
git pull origin develop
git checkout -b feature/conversation-crud
```

### 2. Code và commit thường xuyên

```bash
git add -A
git commit -m "conversation: add create and list repository"
git commit -m "conversation: add service with ownership check"
git commit -m "conversation: add controller and routes"
```

### 3. Push và tạo PR vào develop

```bash
git push -u origin feature/conversation-crud
# Tạo PR: feature/conversation-crud → develop
```

### 4. Review → Squash merge vào develop

Trên GitHub: Squash and merge vào develop.
Branch tự xoá sau merge.

### 5. Khi Phase xong → merge develop vào main

```bash
git checkout main
git pull origin main
git merge develop
git push origin main
```

---

## Hotfix

```bash
git checkout main
git checkout -b hotfix/fix-sse-flush

# fix...

# Tạo PR vào main, merge
# Sau đó merge main vào develop để đồng bộ
git checkout develop
git merge main
git push origin develop
```

---

## Squash

Mỗi PR dùng **Squash and merge** trên GitHub.
→ develop history sạch: mỗi feature = 1 commit.
→ Trong branch vẫn commit nhỏ thoải mái.

---

## Resolve conflicts

```bash
git checkout feature/conversation-crud
git merge develop
# resolve trong feature branch, không trong develop
```

---

## Cloud session (Claude Code)

Cloud session push về branch `claude/*` do hệ thống chỉ định.
Code trên cloud session coi như một feature branch.
Sau khi review → tạo PR vào develop trên GitHub.

---

## Checklist trước khi merge PR

```
[ ] go build ./... clean
[ ] go vet ./... clean
[ ] go test ./... pass
[ ] Đã test thủ công luồng chính
[ ] Không commit .env, API key, secret
[ ] Commit message đúng format
[ ] PR description mô tả thay đổi
```
