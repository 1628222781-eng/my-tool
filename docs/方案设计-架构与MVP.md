# 鸿蒙设备协同 · 方案设计与第一周 MVP

> 本文件按老师给出的「报销助手」范式补全：① 十个关键问题与决策 ② 四层架构 ③ 推荐目录 tree ④ 数据库表设计 ⑤ 第一周 MVP。
> 与 [`鸿蒙设备协同-方案设计报告.md`](./鸿蒙设备协同-方案设计报告.md) 配套阅读。

---

## 一、十个关键问题（需求评审）

| # | 问题 | 选项 | 不同答案导致的架构差异 | 当前决策 |
|---|---|---|---|---|
| Q1 | 告警来源有哪几路？ | A. 仅 IoT 平台 API ／ B. 平台 API + 边端 MQTT ／ C. 再加群截图 OCR | C 要多一路 OCR 适配器和图片入库；A 最薄 | **B+C**：MVP 先平台 API，截图作为演示输入 |
| Q2 | 三要素（采样时间/在线状态/处理人）由谁补？ | A. 端侧上报就带 ／ B. 服务端聚合补 ／ C. 人工补录 | A 改端侧；B 在服务层加 enrich 服务；C 要人工工作台 | **B**：服务端 enrich，端侧不改 |
| Q3 | 幂等键口径谁拍板？ | A. 林越（技术）定 ／ B. 唐悦（工单）定 ／ C. 顾岚定 | C 才是业务口径，技术方案会随业务改 | **C**：顾岚拍板前，去重规则写在 feature flag 里 |
| Q4 | 设备台账以哪张表为准？ | A. IoT 平台 ／ B. 资产系统 ／ C. 人工台账 | A 做实时同步；B 做 T+1 对账；C 只是参考 | **A 为主，B 月度对账**（邵宁负责） |
| Q5 | 助手权限边界？ | A. 只读告警 ／ B. 可建草稿 ／ C. 可直接派单 | C 要双签+审计；B 是当前目标 | **B**：只生成待审核草稿，不派单（f50/f52） |
| Q6 | 草稿谁审？ | A. 唐悦单审 ／ B. 唐悦 + 许安双审 | B 对涉安全告警走双审，要分流路由 | **A 起步，B 作为安全类告警分流** |
| Q7 | 试点范围？ | A. 一个机房一类非关键设备 ／ B. 多机房 ／ C. 含关键设备 | A 风险最低；C 触发许安一票否决 | **A**：PUMP-017 所在机房，非关键设备 |
| Q8 | LLM 在哪跑？ | A. 公有云 API ／ B. 私有化部署 ／ C. 端侧小模型 | B/C 要走内网评估，A 最快 | **A** MVP 期，预留接口可切换 |
| Q9 | 告警与工单数据留存？ | A. 7 天 ／ B. 90 天 ／ C. 永久 | 决定冷存储和审计表容量 | **B 90 天热存**，审计日志另存 |
| Q10 | 怎么算成功？ | A. 误报率下降 ／ B. 出勤减少 ／ C. 工时下降 + 采用率 | 决定埋点和观察期长度 | **C**：整理工时 / 复核工时 / 采用率（f51） |

**一页纸需求清单**（决策收敛后）：
1. 入口：IoT 平台 Webhook + 演示用群截图输入。
2. 服务层必出：采样时间、在线状态、当前处理人。
3. 按 `(device_id, event_id)` 幂等去重，断网重连不重复开单。
4. 助手只产出**待审核草稿**，审核人唐悦（安全类加许安）。
5. 控制 / OTA / 联锁 / 停机**不进**模型权限，一例越权即停试点。
6. 状态机固定：收到 → 接单 → 检查 → 复机，助手不跳转。
7. 试点：单机房、非关键设备、观察期先只记录不改流程。
8. 指标：有效告警数、整理工时、复核工时、采用率（基线未采集 f12）。

---

## 二、四层架构

```
┌─────────────────────────────────────────────────────────────┐
│ 入口层 Ingestion                                             │
│  • 边端 MQTT (鸿蒙设备)    • IoT 平台 Webhook                │
│  • 群截图 OCR（演示输入）                                     │
└──────────────────────┬──────────────────────────────────────┘
                       │  归一化告警事件
┌──────────────────────▼──────────────────────────────────────┐
│ 服务层 Services                                              │
│  • enrich.py     补三要素（采样时间/在线状态/处理人）        │
│  • dedup.py      幂等去重 (device_id + event_id)             │
│  • draft.py      生成待审核工单草稿                          │
│  • ticket.py     状态机（收到→接单→检查→复机）              │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│ 智能体编排层 Agents                                          │
│  • advisor.py    生成巡检建议（LLM 调用，可换供应商）        │
│  • retriever.py  巡检手册 / 设备台账向量检索                  │
│  • guard.py      安全红线校验（控制/OTA/联锁/停机 → 拦截）   │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│ 数据模型层 Data                                              │
│  devices · alert_events · idempotency_keys                  │
│  ticket_drafts · audit_logs · 巡检手册向量表                 │
└─────────────────────────────────────────────────────────────┘
```

**技术栈约束**：
- 后端：FastAPI（异步）+ Python 3.11+
- LLM：OpenAI 兼容 HTTP API，工厂模式可切换供应商
- 向量库：MVP 用 `sqlite-vec`，试点期迁 `pgvector`
- DB：MVP SQLite，试点期 PostgreSQL
- 前端：MVP 单文件 HTML，后续 React

---

## 三、推荐目录 tree

```
src/my_tool/
├── api/                    # 入口层 HTTP
│   ├── alerts.py            # POST /ingest  接收告警
│   └── tickets.py           # GET  /drafts  审核工作台
├── ingestion/              # 多源适配
│   ├── edge_mqtt.py
│   ├── platform_webhook.py
│   └── screenshot_ocr.py
├── services/               # 服务层（纯业务，不依赖 LLM）
│   ├── enrich.py
│   ├── dedup.py
│   ├── draft.py
│   └── state_machine.py
├── agents/                 # 编排层（依赖 LLM / 向量库）
│   ├── advisor.py
│   ├── retriever.py
│   └── guard.py             # 安全红线，硬编码
├── db/
│   ├── models.py           # SQLAlchemy ORM
│   └── migrations/
├── config/
│   └── settings.py
└── main.py                 # FastAPI app 工厂
```

---

## 四、数据库表设计

```sql
-- 设备台账
devices(
  device_id        TEXT PRIMARY KEY,     -- e.g. PUMP-017
  name             TEXT NOT NULL,
  site             TEXT NOT NULL,        -- 机房
  asset_owner      TEXT,
  status           TEXT,                 -- online / offline / stale
  last_seen_at     TIMESTAMP
)

-- 告警事件流
alert_events(
  event_id         TEXT PRIMARY KEY,     -- 端侧生成
  device_id        TEXT REFERENCES devices(device_id),
  raw_payload      JSONB,
  sample_time      TIMESTAMP,            -- 三要素之一
  online_status    TEXT,                 -- online / offline / stale
  handler          TEXT,                 -- 当前处理人
  received_at      TIMESTAMP,
  is_duplicate_of  TEXT REFERENCES alert_events(event_id)
)

-- 幂等键（唯一约束防重）
idempotency_keys(
  device_id        TEXT,
  event_id         TEXT,
  ticket_ref       TEXT,
  created_at       TIMESTAMP,
  PRIMARY KEY (device_id, event_id)
)

-- 待审核工单草稿
ticket_drafts(
  id               TEXT PRIMARY KEY,     -- WO-001
  event_id         TEXT REFERENCES alert_events(event_id),
  suggestion       TEXT,
  status           TEXT,                 -- pending / approved / rejected
  reviewer         TEXT,                 -- 唐悦 / 许安
  decided_at       TIMESTAMP
)

-- 审计日志
audit_logs(
  id               SERIAL PRIMARY KEY,
  actor            TEXT,
  action           TEXT,
  before_state     JSONB,
  after_state      JSONB,
  created_at       TIMESTAMP
)
```

---

## 五、第一周 MVP（能跑通的最小闭环）

**目标**：演示告警 → 补三要素 → 幂等去重 → 生成待审核草稿 → 人工通过/驳回。

**范围**：
- ✅ 单文件前端 [`examples/demo.html`](../examples/demo.html)（已交付）
- ✅ 5 条假数据（含 1 条断网重连重复告警）
- ✅ 安全红线提示（底部红条）
- ❌ 不接真 LLM，巡检建议写死
- ❌ 不接真 IoT 平台，告警手动注入
- ❌ 不部署后端，纯前端 mock

**验收**：当着老师点一遍，能讲清三个 P0 闭环（补三要素 / 幂等去重 / 只出草稿）。

**第二周起**：FastAPI 后端 + SQLite + LLM 接 OpenAI 兼容接口 + 巡检手册向量化。
