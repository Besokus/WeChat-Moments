# V1 Enhancement Notes（工程化增强说明）

> 本文档内容属于 **V1 enhancement notes**，用于说明可选增强方向，**不是 V1 mandatory execution path**，不改变当前 V1 正式主链路。

## 1. 文档目的
在不改变 V1 主设计的前提下，沉淀可答辩的工程化增强思路，提升系统设计说明的完整度与可信度。

## 2. 与 V1 主链路的关系
本说明仅补充“可选治理与演进手段”，当前正式链路仍是 `Friendship + Post + FeedInbox` 与 `fan-out on write`。

## 3. 高活跃用户 / 大V 演进说明
在当前题目约束下（单用户好友上限 5000），V1 统一采用 `fan-out on write` 是合理的：发布侧写扩散上限可预期，读取侧可稳定走 `FeedInbox -> BatchGet(Post)`，避免读时跨好友集合的重聚合与排序放大。

当未来出现高活跃用户/广播型用户（短时间高频发布）时，写扩散会被持续放大，典型表现为 outbox 积压上升、分区写入热点增强、同一作者相关链路尾延迟抬升。

后续可在不改变主模型（`Friendship + Post + FeedInbox`）的前提下演进为 hybrid：普通用户继续 push，大V转为 pull 路径。该能力仅作为演进方向，不属于当前 V1 mandatory execution path。

## 4. 首页首屏缓存优化说明
在 `10M+` 用户规模下，首页首屏时间线是高并发刷新场景中的典型读热点；当大量用户短时间重复进入首页时，即使正式路径稳定，`FeedInbox -> BatchGet(Post)` 仍会承受明显重复读取压力。针对该热点，可增加 viewer-level first-page 的短 TTL 缓存（例如按 `viewerId + firstPageCursorKey` 组织），用于吸收短时间内的重复请求。

该缓存仅是读优化层，不是唯一真相源，且不改变当前 friend timeline 正式主链路：命中缓存直接返回首屏结果，cache miss 或过期后仍回到 `FeedInbox -> BatchGet(Post)` 执行查询并回填缓存。

线上审计口径下，如果没有首屏缓存，V1 仍然成立，但首页高峰的 p95/p99 延迟会更容易被 `BatchGet(Post)` 跨分片回表拉高。因此缓存不是正确性的必要条件，但是真实高并发读峰值下的重要削峰手段。

建议验收指标：`timeline_first_page_cache_hit_rate >= 70%` 作为压测目标，且 cache 异常时 `timeline_read_latency_p99` 不应导致主链路不可用。

该增强属于可选项，不是 V1 mandatory execution path；它的价值在于体现你在既有架构不变前提下，对 `10M+` 场景读热点治理的工程意识。

## 5. friend list cache 可选优化说明
发布动态进入 fan-out 时，需要读取作者好友集合；在题目约束“每人最多 5000 好友”下，高频作者会反复触发同一批好友列表读取。可选地增加 friend list cache，用于降低这类重复读取对 `Friendship` 存储层的瞬时压力。

该缓存只用于读优化，不作为唯一真相源：`Friendship` 仍是好友关系真相源。命中缓存则直接使用缓存好友集合执行 fan-out；cache miss、过期或异常时必须回源 `Friendship` 查询，并可按短 TTL 回填缓存。

此增强不改变 V1 主链路，也不要求复杂缓存一致性协议；它在 fan-out 场景下有现实价值，因为能直接降低“最多 5000 好友”约束带来的重复关系读取开销。

## 6. fan-out 写扩散代价量化说明
在题目约束下，fan-out 写扩散是发布链路的直接结果而不是实现细节：单用户好友上限为 `5000`，因此单次发布在最坏情况下最多触发约 `5000` 条 `FeedInbox` 写入。随着用户规模到 `10M+`，并发发布会把这类写放大快速累积到分发链路。

可用简化容量表达式做预算：`fanout_write_qps ≈ publish_qps * avg_friends_per_publisher`。在极端情况下，上界接近 `publish_qps * 5000`，如果把全部 fan-out 压在同步请求里，发布接口的延迟与失败率会直接受写扩散波动影响，难以稳定。

线上容量评估必须继续落到 worker 能力：`worker_write_capacity = partition_count * worker_per_partition * write_qps_per_worker`。若 `worker_write_capacity < fanout_write_qps`，outbox backlog 必然增长，系统只能通过背压、降级和扩容缩短恢复窗口，不能宣称实时分发。

恢复时间用 `backlog_recovery_time = backlog_size / max(1, worker_write_capacity - fanout_write_qps)` 估算。V1 默认目标为 `max_fanout_lag_target = 300s`、`max_backlog_recovery_time = 1800s`，这些参数不是拍脑袋常量，后续必须通过压测校准。

因此当前 V1 采用 `Outbox + async fan-out + chunk + backpressure` 是合理的工程取舍：同步请求只负责原子主写，扩散写在异步分区消费中完成，并通过分批与背压控制链路抖动。这个小节不是新设计，而是对现有 tradeoff 的量化解释。

## 7. 可观测性与核心指标说明
虽然当前 V1 是 pseudocode-only / docs-only 交付，但真实落地时仍需为核心链路建立最小可观测性，以便快速定位“发得慢、分发滞后、首页变慢”等问题。该项属于工程治理增强，不改变四个核心业务能力，也不引入新的业务模块。

指标设计应直接映射现有主流程：发布动态（publish）、写扩散分发（fan-out）、收件箱写入（FeedInbox write）和时间线读取（timeline read）。当写扩散链路出现积压或消费滞后时，应能通过同一组指标看到从发布入口到分发末端的退化路径；当首页读取性能下降时，应能通过延迟分位数快速识别读取侧瓶颈。

- `publish qps`：反映发布入口流量强度，用于判断是否进入高写压区间。
- `fan-out lag`：反映从发布到分发完成的滞后程度，用于识别消费积压与处理延迟。
- `inbox write failure rate`：反映写入 FeedInbox 的失败比例，用于发现写扩散链路不稳定。
- `timeline read latency (p95 / p99)`：反映时间线读取尾延迟，用于识别首页读取退化与用户体验风险。

这个小节能明显提升项目工程感，因为它把“功能能跑”提升为“链路可观测、问题可定位”的可运维设计表达。

## 8. 高负载下的降级策略说明
在题目约束（10M+ 用户、单用户最多 5000 好友、人均 100 动态）下，高并发高峰时应采用“发布成功优先”的降级策略：当 fan-out backlog 超过阈值，主链路优先保障 `Post + Outbox` 写入成功，确保用户“发得出去”；“所有好友立即可见”降为次级目标。这个策略是容量保护下的优先级管理，不是功能削减。

在降级窗口内，好友时间线允许短暂延迟可见：新动态可先处于“已发布、待分发”状态，由异步 worker 持续消费 backlog 并补齐 `FeedInbox`。这样可以避免写扩散在峰值时反向拖垮发布入口和整体可用性，属于面向大规模并发场景的工程权衡。

硬降级并不等于停消费：当 backlog 超过硬阈值时，仍应保留最小消费配额持续排空，避免分区冻结导致最终一致性只停留在口号层。

降级只改变可见性时延，不改变一致性目标：当压力回落后，系统继续按重试与补偿流程完成分发，最终一致性仍保留。该取舍直接服务于本题的规模约束，目标是在极端负载下保持核心功能稳定并可恢复。

## 9. V1 与未来版本边界说明
V1 仅保证最小闭环与近期时间线可用性，增强项属于未来演进方向，不作为当前必须实现内容。

## 10. 线上高并发验收标准
以下标准用于判断增强说明是否真正支撑高并发审计，而不是只停留在文案层：

- fan-out：能根据 `publish_qps * avg_friend_count` 计算写扩散压力，并能判断 worker 是否欠配。
- Outbox：高 backlog 下仍保留最小消费配额，避免最终一致性冻结。
- 首页读取：能定位 `BatchGet(Post)` 是读峰值瓶颈，first-page cache 失败必须 fallback 到正式链路。
- retention：按 route partition 滚动清理，限制用户数和删除 chunk，禁止全局扫描。
- 分片：批读/批写必须按 route 分组，禁止广播所有分片。
- 边界：retention window 外历史时间线不在 V1 保证范围。

## 11. 总结
V1 当前方案已满足题目交付要求，本增强说明用于表达“可持续扩展能力”而非改变当前架构结论。



## 12. 发布入口准入增强说明（可选）
当前 V1 已有分区消费背压与重试补偿，但这些机制属于“消费端治理”。在高压场景下，为防止入口无限接单把风险转移为 backlog 延迟爆炸，可增加 publish admission control：按分区 pending 定义 soft/hard 阈值，分别返回 `accepted_but_delayed` 与 `rejected_retry_later`。

该增强不改变 V1 主架构与主数据模型，只补充“入口受控退化”语义，使系统在高负载下具备有尊严的拒绝能力，而不是单纯依赖异步兜底。
