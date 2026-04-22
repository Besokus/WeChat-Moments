# 微信朋友圈最小闭环 V2（V1 迭代）

## 1. V2 的定位
V2 不是重写 V1，而是在 V1 已经成立的前提下，把“高活跃用户治理”“读写分层”“容量自适应”做得更像真实线上系统。

V1 解决的是：
- 最小闭环能否成立
- 大规模约束下主链路是否正确
- 高压下能否有尊严地退化

V2 解决的是：
- V1 在更极端流量下如何进一步分层治理
- 普通用户和高活跃用户如何走不同路径
- 读热点和写热点如何继续分离
- 生产边界如何更硬

## 2. V2 保持不变的东西
V2 继续保留 V1 的主架构：
- `Friendship`
- `Post`
- `FeedInbox`

V2 继续保留 V1 的核心路径：
- 写主链路仍然以 `Post + Outbox + Idempotency` 为主
- 读主链路仍然以 `FeedInbox -> BatchGet(Post)` 为主
- 所有列表查询仍然使用 `cursor + limit`
- 仍然禁止读时聚合 5000 好友动态再排序

也就是说，V2 不是把 V1 推倒重来，而是在 V1 的骨架上增加更细的治理层。

## 3. V2 的核心升级点
### 3.1 从统一 push 走向 push/pull hybrid
V1 在题目约束下统一采用 `fan-out on write` 是合理的，因为 5000 好友上限让写扩散是可预算的。

V2 进一步把用户分层：
- 普通用户：继续走 push
- 高活跃用户 / 广播型用户 / 大V：逐步转为 pull 或 hybrid

这个变化的价值在于：
- 减少极端作者的写扩散压力
- 减少 outbox backlog 的集中增长
- 避免少数热点作者拖垮整个写链路

### 3.2 First-page cache 从可选优化变成高频刚需
在 V1 里，首屏缓存只是优化层。
在 V2 里，首屏缓存可以作为首页体验的第一层保护：
- viewer-level first-page cache
- 短 TTL
- cache miss 立即回退正式路径

它不会改变正确性，但会显著降低首页高峰对 `BatchGet(Post)` 的直接压力。

### 3.3 friend list cache 进一步服务 fan-out
V2 继续允许 friend list cache，但它不再只是“可选优化”这么简单，而是 fan-out 读放大的重要缓冲层。

它的作用仍然只有一个：
- 降低重复读取 `Friendship` 的成本
- 不成为唯一真相源
- miss / 失效后回源 `Friendship`

### 3.4 admission control 更细粒度化
V1 的 admission control 已经解决“无限接单”问题。
V2 可以进一步做分层治理：
- 按发布分区观察 backlog
- 按作者热度区分阈值
- 按用户等级选择不同流量策略

目标不是把规则做复杂，而是让系统更适合真实流量分布，而不是只适合平均值。

### 3.5 失败恢复从“能恢复”走向“可控恢复”
V2 在 V1 的 `reconcile + dead-letter + repair` 基础上，进一步强调：
- 恢复优先级
- 恢复窗口
- 热点分区优先排空
- 低优先级保留可见延迟

也就是说，V2 不是只关心“最终能补齐”，还关心“先补谁、多久补完”。

## 4. V2 的结构图
V2 仍然以 V1 为基础，只是多了几层治理：

```text
Publish Request
  -> Admission Control
  -> Post + Outbox + Idempotency
  -> Async Fan-out Worker
  -> FeedInbox
  -> Timeline Read
  -> Cache / BatchGet / Fallback
```

对于高活跃用户：

```text
Publish Request
  -> Admission Control
  -> Post + Outbox
  -> Pull-oriented / Hybrid timeline path
```

这里的重点不是路径更长，而是路径更分层。

## 5. V2 的工程价值
V2 的工程价值在于把系统从“能讲清楚”推进到“更像线上会这么做”：
- 普通流量保留简单路径
- 热点流量走更适合自己的路径
- 缓存和 admission control 形成真正的护栏
- 容量、恢复、失败边界更可量化

如果说 V1 的重点是“架构正确”，那么 V2 的重点是“治理更细”。

## 6. V2 仍然不做什么
V2 依然不做：
- 评论
- 点赞
- 媒体上传
- 推荐流
- 完整生产级运维平台
- 真实数据库/中间件实现

V2 仍然是伪代码和设计说明，不是生产实现。

## 7. 为什么这个 V2 适合做“V1 的迭代”
因为它没有改掉 V1 的核心判断：
- 时间线仍然基于 `FeedInbox`
- 发布仍然以 `Post + Outbox + Idempotency` 为主
- 高压下仍然承认降级和拒绝
- 所有列表仍然坚持 cursor 分页

V2 只是把 V1 中“可选增强”“工程治理”“未来演进”的部分明确成下一版的自然延伸。

## 8. 一句话总结
V1 证明这套朋友圈最小闭环“能成立”；V2 证明这套方案“能继续往真实线上靠近”。
