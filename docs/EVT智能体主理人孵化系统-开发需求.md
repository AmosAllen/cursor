# EVT 智能体主理人孵化系统 — 开发规格（压缩版）

> V1.0-dev | 2026-07-16

---

## 0. 系统定义

| 项 | 值 |
|----|-----|
| 顾问端 | 个人微信（唯一入口，无小程序） |
| 配置端 | 运营后台 |
| 外部依赖 | 第三方微信网关（收发消息、定时推送、回调、人工接管） |
| 完成判定 | **仅顾问文字确认**，不验截图、不验真实发布 |
| 运营模式 | 任务自动生成+自动推送，**不经人工审核** |

**主循环：**

```
加好友 → D1-D3建档 → D4+每日任务循环 → 月末复盘改阶段 → 下月任务
```

---

## 1. 领域模型

### 1.1 表/实体

| 实体 | 关键字段 | 说明 |
|------|----------|------|
| `advisor` | `wx_id`, `onboarding_day`, `growth_stage`, `status` | 顾问主表 |
| `profile` | `experience*`, `expertise*`, `target_customer*`, `direction*`, `style`, `source`, `version` | *必填；`source`=ai/manual |
| `goal` | `type`, `value`, `period`, `priority` | 短/长期目标 |
| `channel_ability` | `channel`, `level`, `excluded` | `level`∈L0-L3；`excluded`=不纳入计划 |
| `learning_record` | `plan_deadline`, `status` | 语雀学习 |
| `task_template` | `type`, `stage`, `channel`, `steps`, `frequency`, `enabled` | 后台配置 |
| `growth_rule` | `stage`, `task_type`, `cron`, `send_window`, `enabled` | 每(stage+type+cron)唯一 |
| `task_instance` | `advisor_id`, `template_id`, `status`, `gen_reason`, `confirm_text` | 系统生成 |
| `pause_state` | `scope`, `task_type?`, `wake_at`, `reason` | `scope`∈type/global |
| `monthly_review` | `month`, `metrics_json`, `stage_before`, `stage_after` | 每月1条 |
| `knowledge` | `content`, `status`, `version` | 仅`published`可RAG |
| `material` | `tags`, `expire_at`, `enabled` | 沿用电商后台 |
| `handoff` | `trigger`, `context`, `assignee` | 人工承接 |

### 1.2 枚举

```ts
// 顾问状态
AdvisorStatus = onboarding_d1 | onboarding_d2 | onboarding_d3 | active | global_paused

// 成长阶段（任务生成用）
GrowthStage = basic | stable | advanced

// 任务状态
TaskStatus = pending | pushed | tracking | completed | partial | incomplete | paused | cancelled

// 渠道
Channel = wechat_moments | wechat_group | xiaohongshu | douyin | weibo

// 渠道等级
ChannelLevel = L0 | L1 | L2 | L3
```

### 1.3 状态机

**顾问：**
```
onboarding_d1 → d2 → d3 → active ⇄ global_paused
```
- 进入 `active` 条件：D1-D3 必填项齐全 + 初始 `growth_stage` 已计算

**任务：**
```
pending → pushed → tracking → completed | partial | incomplete | paused
```
- `incomplete`：次日继续 `tracking`，同日不重复催
- `paused`：到 `wake_at` 发唤醒询问，不自动续推

---

## 2. 事件驱动（按此拆服务/定时任务）

### 2.1 入站事件

| 事件 | 触发 | 处理 |
|------|------|------|
| `friend.added` | 微信加好友 | 创建 advisor，`status=onboarding_d1`，发欢迎+能力介绍 |
| `message.received` | 顾问发消息 | 按 `advisor.status` 路由到对应 handler（见 §3） |
| `wechat.callback` | 第三方回调 | 更新消息发送状态；失败入重试队列 |

### 2.2 定时任务

| Cron | 任务 | 逻辑 |
|------|------|------|
| 每日 09:00（可配） | `job.daily_task_gen` | 见 §4.1 |
| 按任务 `track_at` | `job.task_track` | 发四选一追问：已完成/未完成/部分/暂停 |
| 学习 `plan_deadline` | `job.learning_remind` | 询问是否学完；未完成可改期，同日不反复催 |
| 暂停 `wake_at` | `job.pause_wake` | 仅问是否恢复，不推新任务 |
| 每月最后一天 | `job.monthly_review` | 见 §4.3 |
| D2/D3 日切 | `job.onboarding_advance` | 推进 onboarding 日程 |

### 2.3 出站动作

| 动作 | 调用 |
|------|------|
| 发微信消息 | `wechatGateway.send(advisor_id, payload)` |
| 人工接管 | `wechatGateway.handoff(advisor_id, assignee)` |
| 通知承接人 | `notify(assignee, context_summary)` |

**网关异常：** 入 `message_queue` → 重试 N 次 → 写 `exception_log`，后台可见。

---

## 3. 消息路由（`message.received`）

```text
IF advisor.status IN (onboarding_d1, d2, d3)
  → onboardingHandler()      // 画像/目标/渠道采集

ELSE IF intent = knowledge_qa
  → ragAnswer()              // 仅 published 知识；无命中则兜底，不编造

ELSE IF intent = content_request OR active_task.needs_content
  → contentGen()             // 画像+素材+知识组装

ELSE IF intent = task_confirm
  → taskConfirm()            // 见 §4.2

ELSE IF intent = pause
  → pauseHandler()           // 记 scope + wake_at（默认+7d）

ELSE IF intent = resume
  → resumeHandler()

ELSE IF intent = request_human
  → handoffHandler()

ELSE
  → generalChat()              // 日常陪伴，不阻塞定时推送（推送延后）
```

---

## 4. 核心业务逻辑

### 4.1 任务生成 `job.daily_task_gen`

```text
IF advisor.status = global_paused → RETURN
IF advisor.status != active → RETURN

rules = growth_rule WHERE stage=advisor.growth_stage AND enabled=true
IF rules.empty → log("缺少任务生成规则"); RETURN

FOR each rule:
  IF task_type IN pause_state → SKIP
  IF channel.excluded OR channel.paused → SKIP
  IF 同(stage+type+cron)存在多条enabled → 后台禁止发布（校验在配置时）

  difficulty = f(advisor近7天完成率)   // 只跟自己比
  task = assemble(template, profile, goal, channel, materials)
  INSERT task_instance(status=pending)
  IF now IN send_window → push(task)
```

**推送约束：**
- 窗口默认 09:00-20:00
- 顾问对话进行中 → 延后 push
- 无人工审核

### 4.2 任务确认 `taskConfirm`

```text
IF 明确完成词（已发/完成了/建好了/做完了）→ status=completed, 存 confirm_text
ELSE IF 模糊词（差不多/正在做/应该可以）→ 追问一次，不改状态
ELSE → 引导选择四选一按钮/话术

ON completed → 鼓励话术 + 更新当月统计（不改 growth_stage）
```

### 4.3 月度复盘 `job.monthly_review`

```text
IF 当月已有 monthly_review → 复用，不重复问
metrics = 聚合当月任务/互动数据（口径见下表）
IF global_paused → 轻量复盘（原因+下月计划），不推新任务
ELSE → 多轮问答（公司认知仅首次、行业认知、执行障碍、体验建议）

new_stage = judge(metrics, 顾问反馈)   // 唯一改阶段入口
stage_after 次月1日00:00生效，不回溯

IF 含个人建议 → handoff(assignee, context)
```

| 指标字段 | 计算 |
|----------|------|
| `completion_rate` | completed / (到期且未取消) |
| `effective_done` | 有 confirm_text 的 completed 数 |
| `partial_count` | partial 且月末仍未完成 |
| `incomplete_count` | 明确选未完成 |
| `pause_count` | 进入过 paused 的任务（去重） |
| `active_days` | 有回复的自然日数 |

### 4.4 渠道定级（D2-D3）

| level | 条件 | 生成任务倾向 |
|-------|------|--------------|
| L0 | 无账号/群 + 有意愿 | 搭建类 |
| L1 | 有渠道，近7天无稳定执行 | 低难度启动 |
| L2 | 近7天有执行但不稳 | 频率+内容 |
| L3 | 稳定习惯 | 优化/转化/IP |

`excluded=true` → 该 channel 永不生成任务。

### 4.5 Onboarding 日程

| 日 | `advisor.status` | 必完成 |
|----|------------------|--------|
| D1 | onboarding_d1 | 画像4必填 + 语雀链接已发 + 学习计划 |
| D2 | onboarding_d2 | 目标含方向+周期 + 各渠道调研 |
| D3 | onboarding_d3 | 补齐缺失 → 算 channel level + `growth_stage` → `active` |

规则：当日缺项**不造假**；次日**先补关键项**再跑当日流程。

---

## 5. 后台模块 → 开发任务

| 模块 | 做什么 | 智能体读什么 |
|------|--------|--------------|
| 顾问管理 | CRUD + 画像版本 + 人工改画像 | 下次生成用最新 profile（manual优先） |
| 知识库 | 上传/发布/版本/测试检索 | 仅 `published` |
| 任务模板 | 类型/阶段/渠道/步骤/互斥/启停 | 生成时匹配 |
| 成长规则 | 阶段阈值/时段/频控/唤醒默认7d | `job.daily_task_gen` |
| 任务记录 | 查生成依据/状态/确认原文 | 只读 |
| 素材 | 标签/有效期/启停 | `enabled && !expired` |
| 复盘 | 看摘要/建议/接管入口 | 月末写入 |
| 数据分析 | 指标看板+下钻 | 只读 |
| 系统配置 | 网关参数/重试/承接人/话术 | 全局配置 |

---

## 6. 硬规则（实现时必须遵守）

1. `growth_stage` **只在** `job.monthly_review` 修改
2. 日常任务结束只写统计，**不**改阶段
3. 任务完成 = 文字确认；模糊必追问
4. RAG/内容生成：**禁止**编造产品事实；素材/知识过期不可用
5. 暂停类型：不 gen、不 push、不 track
6. 全局暂停：仅唤醒询问 + 主动问答
7. 画像冲突：取顾问最新明确表述，保留 version 历史
8. 后台改画像 > AI 推断
9. 每自然月 `monthly_review` 最多 1 条正式记录
10. 网关失败：队列重试 → 超限记异常

---

## 7. 人工边界

| 系统做 | 人做 |
|--------|------|
| 对话/生成/推送/追踪/复盘/定阶段 | 维护知识、模板、规则、素材、语雀链接 |
| 触发 handoff + 摘要 | 承接人微信接管 |
| 汇总指标 | 看板分析、调规则 |

---

## 8. 验收（测试用例级）

| # | 用例 | 预期 |
|---|------|------|
| 1 | 新好友 | D1→D2→D3 顺序执行，缺项可次日补 |
| 2 | D4 定时 | 按规则生成并推送，后台可查 `gen_reason` |
| 3 | 回复「完成了」 | `task.status=completed`，存原文 |
| 4 | 回复「差不多」 | 状态不变，追问一次 |
| 5 | 暂停朋友圈任务 | 该类型不再 push/track，7d 后唤醒询问 |
| 6 | 月末 | 触发复盘，阶段变更次月生效 |
| 7 | 说「转人工」 | 停 AI 回复，承接人收到摘要 |
| 8 | 知识未发布 | RAG 检索不到 |
| 9 | 网关失败 | 重试后入异常列表 |

---

## 9. 待对接

- 微信好友 ↔ `advisor_id` 绑定方式（本期假定第三方已绑）
- 语雀链接配置位（后台字段 vs 常量）
- 微信网关 API 契约
- 电商素材 API
- 任务/内容生成 Prompt（另文档）
