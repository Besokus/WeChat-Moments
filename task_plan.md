# Task Plan

## Goal
在 10M+ 用户规模约束下，完成“微信朋友圈最小闭环”的系统设计说明与伪代码方案规划（当前阶段仅输出架构分析，不输出大段伪代码）。

## Phases
| Phase | Status | Description |
|---|---|---|
| 1 | in_progress | 输出需求理解、核心架构、规模适配原因、核心数据流、项目边界 |
| 2 | pending | 用户确认后再进入数据模型与接口细化 |
| 3 | pending | 用户确认后再进入服务伪代码 |

## Constraints
- 仅最小闭环：好友、发动态、查个人动态、查好友时间线
- 仅伪代码/设计说明，不写真实框架或中间件代码
- 时间线必须基于 Friendship + Post + FeedInbox
- 默认采用 fan-out on write
- 禁止“查询所有好友动态后内存排序”
- 禁止全表扫描

## Errors Encountered
| Error | Attempt | Resolution |
|---|---|---|
| 无 | 0 | - |

## Status Update 2026-04-21
- Phase 1: completed (architecture analysis delivered).
- Phase 2: in_progress (core data model pseudocode written to project).
- Phase 3: pending (service pseudocode remains).

## Errors Encountered (Update 2026-04-21)
- apply_patch failed in this environment with windows sandbox firewall argument error.
- workaround used: short PowerShell Set-Content/Add-Content writes, then explicit read-back validation.

## Status Update 
2026-04-21 22:25:24
- Phase 3: completed (core service pseudocode written for required 5 functions).
- Covered functions: addFriend, listFriends, publishPost, listUserPosts, listFriendTimeline.
- No controller/framework/SQL code added; pseudocode only.

## Status Update 
2026-04-21 22:38:45
- Phase 4: completed (final deliverable spec documented).
- Output now includes explicit Definition of Success and per-module acceptance criteria under 10M+/5000/100 constraints.

## Status Update 
2026-04-21 23:08:07
- Phase: architecture convergence completed in working tree.
- Formal timeline path is now FeedInbox read path; friendIds-based recent dynamic aggregation was removed from repository/service formal path.

## Status Update 
2026-04-21 23:21:09
- Consistency patch completed: interface layer now aligned with service layer contracts.

## Status Update 
2026-04-21 23:32:57
- Technical documentation baseline completed: architecture rationale and maintenance rules are now in place.

## Status Update 
2026-04-21 23:49:19
- New Optimization Phase: completed (P0/P1/P2 reliability pseudocode implemented).
- Focus delivered: idempotency, atomic boundary via outbox pattern, async chunk fan-out, reconciliation, shard routing contract, cursor consistency rule.

## Status Update 
2026-04-21 23:54:14
- Reliability refinement completed: shard-safe batch contracts aligned.
- Added repository/service contract updates for routed batch read/write to reduce cross-shard risk under high load.

## Status Update 
2026-04-22 00:16:50
- Phase 4: completed (timeline read-path optimization + commit prep).
- Optimized file: domain/timeline_service.pseudo
- Focus: clamp constant, dedupe post ids before batch load, missing-post graceful skip.

## Status Update 
2026-04-22 00:26:13
- Optimization phase updated: implemented enforceable throughput governance for fan-out path.
- Added explicit constraints for partitioned outbox consumption, backlog backpressure, timeline read amplification control, and feed retention lifecycle.

## Status Update 
2026-04-22 00:43:07
- Phase: documentation maintenance completed for this round.
- Updated docs/technical-architecture.md with architecture improvements and boundary optimization notes.

## Status Update 
2026-04-22 00:46:43
- Final review gap-fix phase completed in working tree.
- Fixed idempotent publish return, item-bounded fanout chunking, persisted friendship repair, timeline overfetch, post_id shard routing, retention boundary, and fanout capacity formula.

## Status Update 
2026-04-22 12:31:02
- Added final V1 architecture/design overview document for submission and interview explanation.
- Scope unchanged: docs-only consolidation, no new architecture modules introduced.

## Status Update 
2026-04-22 12:43:02
- Context management checkpoint completed via planning-with-files + context-compression style update.
- Project state: V1 docs consolidation done (v1-overview + v1-enhancement-notes), core architecture unchanged.
- Next focus: execute only user-approved doc refinements or commit/publish actions without expanding scope.

## Status Update
2026-04-22 13:08:00
- Phase: enhancement 落盘复核与最小伪代码执行 completed。
- 已核验：`docs/v1-enhancement-notes.md`、`docs/v1-overview.md`、`docs/technical-architecture.md`、`docs/final-delivery-spec.md` 与关键伪代码一致。
- 当前范围控制：仅增强说明与最小伪代码约束补强，无主架构重构、无题外功能扩展。
## Status Update
2026-04-22 13:20:00
- Phase: V1 enhancement 最小补强 completed。
- 本轮新增：增强文档小节扩写 + fanout worker 边界注释（仅说明，不改主链路行为）。
- 下一步：按你的指令执行提交或继续文档整理。
## Status Update
2026-04-22 14:02:00
- Phase: friend list cache 增强说明与最小伪代码优化 completed。
- 已完成：第5节扩写（2~3段工程化说明）+ fan-out 读取好友列表的可选缓存回源逻辑。
- 约束保持：Friendship 仍是真相源；V1 主链路不变；无复杂缓存协议扩展。
## Status Update
2026-04-22 14:18:00
- Phase: 可观测性增强说明 completed。
- 已完成：第7节扩写，明确核心指标、用途与边界。
- 约束保持：仅文档增强，不引入完整监控平台设计。

## Status Update 
2026-04-22 19:10:39
- Completed docs enhancement for fan-out write amplification quantification.
- Scope remained docs-only; V1 publish main flow unchanged.
## Status Update
2026-04-22 14:30:00
- Phase: 高负载降级策略增强 completed。
- 已完成：文档第8节扩写 + worker/flow/service 最小可执行降级分支。
- 约束保持：不改V1主功能范围，不改一致性目标，仅做工程治理增强。

## Status Update
2026-04-22 19:35:35
- Phase: 高并发关键风险最小化修复 completed。
- 已完成：硬降级从“停消费”调整为“最小配额持续消费”；幂等键作用域补充 `actor_id`；发布原子边界前提与容量恢复目标写入文档与伪代码契约。
- 约束保持：主架构 `Friendship + Post + FeedInbox` 不变，未引入题外功能、未做大范围重构。

## Status Update
2026-04-22 20:00:00
- Phase: 发布入口流量治理（P0） in_progress。
- 当前判断：风险真实存在。现状缺口为“仅靠异步消费背压，入口缺少受控拒绝语义”。
- 下一步：仅做最小补丁（soft/hard 阈值 -> 行为 -> 对外返回语义），不重构主架构。

## Status Update
2026-04-22 20:10:00
- Phase: 生产就绪补强文档 completed。
- 已完成：新增容量规划、失败矩阵、生产就绪检查清单三份文档。
- 目标：把真实线上高并发缺口转化为可压测、可恢复、可验收的上线前门槛。
- 范围：仅文档补强，不改变 `Friendship + Post + FeedInbox` 主架构。

## Status Update
2026-04-22 20:08:00
- Phase: 发布入口流量治理（P0） completed。
- 已完成：publish 入口 admission control（soft/hard 阈值、分级行为、返回语义）落盘到 service/flow/docs。
- 约束保持：不改主架构，不新增题外功能，不把问题推给无限异步堆积。

## Status Update
2026-04-22 19:38:05
- Phase: 线上高并发审计级补强计划 completed。
- 目标：针对 fan-out 容量不可证明、首页读回表瓶颈、FeedInbox retention 执行成本、分区策略偏契约级等问题制定最小改进计划。
- 范围：仅围绕 `Friendship + Post + FeedInbox`、Outbox fan-out、FeedInbox timeline 主链路做计划，不引入评论/点赞/推荐等题外功能。
- 验收：容量公式、首页读峰值、retention 分区清理、分片禁止广播查询均已落盘并通过关键词检查。

## Status Update
2026-04-22 20:25:00
- Phase: README 总入口文档 completed.
- 已完成：新增 README.md，总览项目目标、约束、架构、流程、工程化增强与文档索引。
- 约束保持：仅做总入口整理，不改主架构、不改核心伪代码语义。

## Status Update
2026-04-22 20:40:00
- Phase: README 总入口文档细化增强 completed.
- 已完成：README.md 重写为更完整的项目总览，强化并发设计、一致性、降级、恢复、边界与面试讲法。
- 约束保持：仅改文档表达，不改主架构和核心伪代码语义。

## Status Update
2026-04-22 21:00:00
- Phase: V2 迭代说明完成.
- 已完成：新增 `docs/v2-iteration.md`，并在 README 中加入 V2 入口说明。
- 约束保持：V1 主架构不变，V2 仅作为下一阶段演进展示。
