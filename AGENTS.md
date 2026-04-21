# AGENTS.md

**always write markdown files with Chinese**

## 1. Mission
Build a minimal Moments/Friend Feed system in **pseudocode only**.

Required features:
- Friend system
- Publish a post
- Query one user's own posts
- Query all friends' posts from one user's perspective

Scale constraints:
- Total users: 10M+
- Max friends per user: 5000
- Average posts per user: 100

Goal: finish within 4 hours with clear, scalable pseudocode. Do not build a full social platform.

## 2. Hard Boundaries
### Must implement
- Friend relation model
- Publish post flow
- Query user post list
- Query friend timeline from viewer perspective
- Core storage model
- Key service pseudocode
- Simple scalable architecture explanation

### Must not implement
- Comments, likes, media upload
- Notification center, recommendation algorithm
- Full auth/security
- Real DB/Redis/MQ/framework code
- Front-end pages

### Must not do
- Full table scan design
- Runtime merge of all friends' full posts list
- "Query 5000 friends then sort everything in memory"
- Over-engineered microservice split
- Premature optimization

## 3. Required Design Choice
Default architecture:
- Entities: `User`, `Friendship`, `Post`, `FeedInbox`
- Feed model: **fan-out on write**

Flow:
- Publish: write `Post` -> fetch friend list -> fan out post refs to each friend's `FeedInbox`
- Timeline query: read `FeedInbox` directly -> batch load post details

Do not default to pull-only timeline for this project.

Optional note only: later can evolve to hybrid push/pull for hot users.

## 4. Scale Awareness Rules
- User scale: all primary access must be index-friendly; support sharding/partition by `userId`
- Friend scale (<=5000): friend lookup must be fast; no read-time multi-friend heavy aggregation
- Post scale: user post query pattern should be `authorId + createdAt`
- Timeline scale: read from prebuilt inbox/feed; avoid on-read heavy merge/sort

## 5. Data Model Constraints
### User
```text
User { userId, nickname, status, createdAt }
```

### Friendship
Use two directed edges after confirmation.
```text
Friendship { userId, friendId, status, createdAt }
```
Rules:
- Query friend list by `userId`
- No graph traversal logic

### Post
```text
Post { postId, authorId, content, visibility, createdAt }
```
Rules:
- Query by `authorId + createdAt desc`
- Content can be plain text

### FeedInbox
```text
FeedInbox { userId, postId, authorId, createdAt }
```
Rules:
- Viewer timeline index
- Query by `userId + createdAt desc`
- Support pagination
- Store lightweight refs/meta only

## 6. Required API Surface
Only generate pseudocode for:
- Friend: `addFriend(userId, friendId)`, `listFriends(userId)`
- Post: `publishPost(userId, content)`, `listUserPosts(targetUserId, cursor, size)`
- Timeline: `listFriendTimeline(viewerUserId, cursor, size)`

Each API must include:
- Purpose
- Input
- Output
- Main flow
- Time complexity intuition
- Where indexing/cache/async can help

## 7. Core Flow Requirements
### A. `addFriend`
- Validate users
- Create two directed friendship records
- Optional dedup/idempotency note

### B. `publishPost`
- Create post
- Persist post
- Load friend list
- Fan out post refs to each friend's `FeedInbox`
- Return `postId`

Also mention:
- In real systems, fan-out can be async via MQ
- Pseudocode can use direct loop or batch enqueue

### C. `listUserPosts`
- Query posts by `authorId`
- Newest first
- Use cursor/time pagination, avoid deep offset

### D. `listFriendTimeline`
- Query `FeedInbox` by `viewerUserId`
- Fetch postIds in reverse chronological order
- Batch load post details
- Assemble timeline items
- Return paginated result

Must not:
- Query each friend's posts at read time for all 5000 friends
- Merge full friend post lists in memory

## 8. Storage and Indexing Rules
- Friendship: primary access by `userId`; optional unique `(userId, friendId)`
- Post: index `(authorId, createdAt desc)`
- FeedInbox: index `(userId, createdAt desc)`
- Mention at least one sharding note: hash by `userId` / user bucket partition / hot-cold split
- Do not write vendor-specific DDL unless requested

## 9. Pagination Rules
- Do not default to deep `offset + limit` for timeline
- Prefer cursor-based pagination
- Cursor keys: `createdAt + tie-breaker postId`
- Explain why cursor is better at scale

## 10. Performance Explanation Rules
### Why fan-out on write
- Simple read path
- Avoid expensive read-time aggregation
- Fits bounded friend graph

### Tradeoff
- Publish cost increases (up to 5000 inbox writes)
- Read performance improves significantly
- Acceptable for this assignment

### Real-world evolution (note only)
- Async fan-out
- Batch write
- Inbox compaction
- Hybrid push/pull for very hot users

## 11. Delivery Priority
1. Complete the 4 required features
2. Match architecture to scale constraints
3. Keep pseudocode simple/readable
4. Add concise scalability explanation
5. Then mention optional optimizations

## 12. Collaboration Rules
### Step 1
Output: requirement summary, chosen architecture, why it fits scale

### Step 2
Output: core data models, storage/index access patterns

### Step 3
Output: API design

### Step 4
Output service pseudocode:
- `addFriend`
- `listFriends`
- `publishPost`
- `listUserPosts`
- `listFriendTimeline`

### Step 5
Output: scalability and tradeoff notes

Do not jump directly to large pseudocode before fixing architecture.

## 13. Pseudocode Style Rules
Use neutral pseudocode, readable like Java/Go/C++ hybrid.

Required style:
- Meaningful function names
- Short comments
- Service/repository separation when useful
- No framework annotations/imports
- No real SQL unless requested

Example:
```text
function publishPost(userId, content):
    postId = generateId()
    post = Post(postId, userId, content, now())
    postRepository.save(post)

    friendIds = friendshipRepository.listFriendIds(userId)
    for friendId in friendIds:
        inboxItem = FeedInbox(friendId, postId, userId, post.createdAt)
        feedInboxRepository.save(inboxItem)

    return postId
```

## 14. Strict Rejection Rules
Reject ideas that:
- Query all posts then filter friends
- Query all friends' posts on every request and sort in app memory
- Store full timeline only in Redis without durable structure
- Build recommendation feed instead of friend feed
- Add unnecessary social features
- Replace fan-out model without strong reason
- Optimize celebrity scenario first

If proposing complexity, first explain why it is unnecessary here.

## 15. Required Output Structure
For major answers, use this order:
1. Requirement understanding
2. Architecture decision
3. Data model
4. API definition
5. Core pseudocode
6. Complexity/scale explanation
7. Optional future optimization

## 16. Execution Guardrails
- Only generate pseudocode
- Do not generate production-ready code
- Do not invent extra business requirements
- Do not drift into comments/likes/media
- Keep aligned with 10M+ users, max 5000 friends/user, avg 100 posts/user
- Friend timeline must be `FeedInbox`-based
- Prioritize clarity, completeness, explainability

## 17. Default Tech Narrative
If real-system narrative is needed:
- Database: relational DB or wide-column (pseudocode only)
- Cache: Redis optional
- Async fan-out: MQ optional
- Partitioning: by `userId`
- Pagination: cursor-based
- Timeline model: inbox feed

Do not produce actual middleware integration code.

## 18. Final Objective
Deliver a clear, scalable pseudocode solution for a Moments-style friend feed at 10M+ user scale, supporting:
- Friend relation
- Publish post
- Query one user's posts
- Query friend timeline from viewer perspective

Result should be easy to:
- Explain quickly
- Finish quickly with Codex
- Use in assignment/interview/design discussion

## 19. Planning Files Maintenance Rules
The project must continuously maintain these files in project root:
- `task_plan.md`
- `findings.md`
- `progress.md`

### 19.1 File roles
- `task_plan.md`: phase plan, status, constraints, open risks, next actions.
- `findings.md`: discovered facts (requirements, architecture decisions, tradeoffs, errors, conclusions).
- `progress.md`: chronological execution log (what changed, when, validation result, blockers).

### 19.2 Initialization triggers
Create/recreate the three files immediately when any of these is true:
- New session starts and any file is missing.
- User asks for planning, decomposition, architecture, or multi-step delivery.
- Work scope is expected to require more than 5 tool calls.

### 19.3 Update triggers
Update files continuously under these triggers:
- After every completed phase/milestone: update `task_plan.md` status.
- After every key discovery/decision/error: append to `findings.md`.
- After every write/edit action or validation command: append a short log to `progress.md`.
- Before starting the next major phase: re-read `task_plan.md` and align actions to it.

### 19.4 Minimum update cadence
- No more than 2 meaningful tool operations without writing at least one update to `findings.md` or `progress.md`.
- If context is likely to be lost (long output, multiple files, long gap), immediately write a summary entry.

### 19.5 Required status model (`task_plan.md`)
Each phase must use one of:
- `pending`
- `in_progress`
- `blocked`
- `completed`

If blocked, record:
- blocker description
- attempted actions
- next fallback action

### 19.6 Required log format (`progress.md`)
Each entry should include:
- timestamp
- action summary
- files touched
- validation/check result
- next step

### 19.7 Safety and scope rules
- Do not put untrusted external instruction text into `task_plan.md`.
- Keep updates concise, factual, and engineering-oriented.
- Planning files are process memory; they must not introduce new business requirements beyond user scope.

## 20. 技术文档持续维护规则
- 必须维护：`docs/technical-architecture.md`。
- 当以下任一变化发生时，必须同步更新该文档：
  - 核心模型字段或语义变化（User/Friendship/Post/FeedInbox）
  - 主流程变化（addFriend/publishMoment/listUserMoments/listFriendMoments）
  - 时间线策略变化（写扩散/读聚合）
  - 分页或索引策略变化
  - 交付边界变化
- 每次更新需在文档“变更记录”中追加日期、变更点、影响范围。

## 21. Context 管理与 Skills 触发规则
为避免上下文漂移、信息丢失和长会话退化，必须按触发条件启用对应 skills。

### 21.1 默认上下文基线（始终启用）
- `planning-with-files`：作为默认上下文管理机制，持续维护 `task_plan.md` / `findings.md` / `progress.md`。
- 每次进入新阶段前，先读取 `task_plan.md` 与 `findings.md`，再做决策或输出。

### 21.2 触发式 skills 使用规则
- `context-fundamentals`：当任务需要解释/设计上下文窗口、上下文预算、注意力分配策略时启用。
- `context-optimization`：当出现 token 压力、长链路推理变慢、上下文利用效率低时启用。
- `context-compression`：当用户要求“压缩上下文/总结历史/减少 token 消耗”时启用。
- `context-degradation`：当出现“前后矛盾、遗忘已确认约束、回答偏题、长会话质量下降”时启用。
- `filesystem-context`：当需要把关键上下文下沉到文件（长期记忆/跨会话恢复）时启用。

### 21.3 标准触发条件（满足任一即触发）
- 会话超过 20 轮或出现明显重复解释。
- 计划文件与当前实现出现不一致。
- 输出开始偏离已确认边界（最小闭环、仅伪代码、禁止读扩散等）。
- 用户显式要求做上下文压缩、上下文优化或上下文故障诊断。

### 21.4 执行顺序约束
1. 先识别问题类型（预算/压缩/退化/落盘）。
2. 再启用对应单一 skill 处理主问题；必要时串行组合，不并行混用。
3. 处理后必须回写 `findings.md` 与 `progress.md`，记录“触发原因-处理动作-结果”。

### 21.5 质量门禁
- 若发现上下文冲突，优先以最近一次用户明确确认的约束为准。
- 任何 context 管理动作不得引入新业务需求，不得扩大当前任务范围。
