# Git Workflow — mini-chatsmith

Adapted from Anfin Gitflow. Solo project, no Jira — dùng phase/feature name thay task code.

---

## Branches

| Branch | Vai trò | Ai merge vào |
|---|---|---|
| `main` | Production-ready, release history | Merge từ feature sau khi verify |
| `develop` | Integration, chứa toàn bộ dev history | Merge từ feature sau khi code xong |
| `phase1/chat-sse-history` | Feature branch Phase 1 | Tác giả |
| `phase2/segmentation` | Feature branch Phase 2 | Tác giả |
| `phase3/notification` | Feature branch Phase 3 | Tác giả |
| `hotfix/<tên>` | Patch nhanh trên main | Merge vào cả main + develop |

```
main ─────────────────────────────────────────────►
  │                              ▲         ▲
  │                              │         │
  ├── phase1/chat-sse-history ───┤         │
  │                              │         │
  ├── phase2/segmentation ───────┘         │
  │                                        │
  └── phase3/notification ─────────────────┘
```

---

## Branch naming

```
phase1/chat-sse-history
phase2/segmentation
phase3/notification
hotfix/fix-sse-flush
```

Không dùng tên chung chung (`feature-1`, `dev-2`). Tên branch mô tả nội dung.

---

## Commit message format

```
<scope>: <message>

Ví dụ:
migration: create users and ai_models tables
conversation: add create and list API
provider: implement OpenAI streaming adapter
sse: handle client disconnect and cancel
config: add Cloud Run secret manager support
hotfix: fix nil pointer in generation cancel
docs: add Phase 1 plan
```

Scope = domain hoặc layer bị thay đổi chính. Không viết hoa đầu câu.

---

## Workflow cho mỗi Phase

### 1. Tạo feature branch từ main

```bash
git checkout main
git pull origin main
git checkout -b phase1/chat-sse-history
```

### 2. Code và commit thường xuyên

```bash
# Commit nhỏ, mỗi commit = 1 bước verify được
git add -A
git commit -m "conversation: add create and list API"
```

### 3. Merge vào develop để test tích hợp

```bash
git checkout develop
git pull origin develop
git merge phase1/chat-sse-history
git push origin develop
```

### 4. Verify xong → merge vào main

```bash
git checkout main
git pull origin main
git merge phase1/chat-sse-history
git push origin main
git branch -d phase1/chat-sse-history
```

### 5. Bắt đầu Phase tiếp theo từ main

```bash
git checkout main
git checkout -b phase2/segmentation
```

---

## Hotfix

```bash
git checkout main
git checkout -b hotfix/fix-sse-flush

# fix code...

git checkout main
git merge hotfix/fix-sse-flush
git checkout develop
git merge hotfix/fix-sse-flush
git branch -d hotfix/fix-sse-flush
```

---

## Squash khi cần

Nếu feature branch có quá nhiều commit nhỏ, squash trước khi merge:

```bash
git checkout phase1/chat-sse-history
git rebase -i main
# squash các commit liên quan
```

---

## Resolve conflicts

Nếu feature branch conflict với develop:

```bash
git checkout phase1/chat-sse-history
git merge develop
# resolve conflicts trong feature branch, không trong develop
```

---

## Cloud session (Claude Code)

Cloud session push về branch `claude/*` do hệ thống chỉ định.
Sau khi review, merge thủ công về `main` hoặc `develop` trên local/GitHub.

---

## Checklist trước khi merge vào main

```
[ ] go build ./... clean
[ ] go vet ./... clean
[ ] go test ./... pass
[ ] Đã test thủ công luồng chính
[ ] Không commit .env, API key, secret
[ ] Commit message đúng format
```
