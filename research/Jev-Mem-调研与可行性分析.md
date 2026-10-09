# Jev-Mem 落地实践调研与可行性分析

> 基于 ChatGPT 讨论《探讨Jev Agent》（2026-10-09）+ 一手资料调研
> 调研日期：2026-10-09

---

## 0. 一句话结论

你讨论里的"记忆专家 + Router + 主动遗忘 + 生命周期"这套想法，**不是空中楼阁，而是已经有一篇 2026-09 的正式论文（Jev-Mem, arXiv:2609.23986）把它系统化了**；更关键的是——**你本地已经装的 `dsh-mnemon` 插件，架构和 Jev-Mem 几乎 1:1 重合**（连 `temporal / semantic / causal / entity` 四种关系边都一模一样）。

所以"落地成一个插件"的真正问题不是"能不能从零搭一个记忆系统"，而是：

> **要不要在已有的 `dsh-mnemon` 骨架上，加一个"System-One 风格（Jev/Laya）的轻量决策层"，把它从"确定性策略"升级成"模型驱动的记忆控制器"。**

这是一个**增量插件**，不是从零造轮子。可行性高，但有几个真实的工程约束（见 §5）。

---

## 1. 你的讨论到底提出了什么

把 ChatGPT 那段对话拆成可验证的技术命题：

| # | 命题 | 成熟度 |
|---|------|--------|
| C1 | 记忆分层：Working → Episodic → Semantic → 长期/用户记忆 | ✅ 业界共识 |
| C2 | **Log ≠ Memory**：日志全存，进 Memory 的要经筛选 | ✅ 业界共识 |
| C3 | 用一个判断器（你叫 JEV）对每条候选记忆输出 `remember / importance / scope / type / ttl / retrieval_keys` | ✅ 正是 Jev-Mem 的写入路径 |
| C4 | **记忆有生命周期**：active → stale → archived → forgotten，而非简单 delete | ✅ Jev-Mem / 多数系统都主张保留证据 + 状态流转 |
| C5 | **主动遗忘 / Memory Maintenance**：检测冲突、过期、冗余，做 supersede / merge | ✅ Jev-Mem 的 periodic consolidation |
| C6 | RAG / grep / structured DB 各司其职，由一个 **Memory Router** 调度 | ✅ 这正是 Jev-Mem 的 query routing |
| C7 | **记忆竞争**：`relevance × importance × recency × confidence × specificity` 打分后只塞最有用的 | ✅ Jev-Mem 的 candidate scoring（公式 23–25 几乎就是这个） |
| C8 | **Memory Experts + Router**（Mixture-of-Memory-Experts），不同类型记忆用不同检索器 | ✅ Jev-Mem 的"四个关系图视图 + 按 view 分配预算" |
| C9 | 做成 **Codex/Claude Code 通用 Memory Middleware，暴露 MCP 接口** | ⚠️ 可行，但这是另一条技术路线（见 §4 形态B） |

**评价：C1–C8 全部被 Jev-Mem 论文验证过是成立且有效的**。你当时"说不清"的 MoE 类比，论文给了准确名字：**System-One / System-Two 架构**——不是参数级 MoE，而是"把高频、有界、非生成的记忆决策，从昂贵的自回归生成里拆出来，交给一个轻量决策器"。你的直觉是对的。

---

## 2. 关键外部发现：Jev 与 Jev-Mem 是真实存在的东西

这不是你凭空想的，生态里已经有对应物：

### 2.1 Jev（TypeSafe AI 的决策模型）
- **定位**：一个"只做决策、不生成文本"的模型。给它一个 `state` + 一组**类型化问题**，它返回带概率/置信度的答案，而不是逐 token 生成。
- **三种问题类型**：
  - `noul`：二元命题 → 返回 P(true)（例："这条信息是否表达紧迫性？" → 0.95）
  - `choice`：互斥多选 → 返回选项 + 各选项概率分布
  - `score`：有序打分（Calm/Frustrated/Angry）→ 返回分值 + 概率
- **上下文** 32K，**定价** 输入 $0.042/1M tokens、**输出 $0**（因为几乎不产生 token），零数据保留（[Cloudflare 文档](https://developers.cloudflare.com/ai/models/typesafe/jev/)）。
- 这正好对应你讨论里 C3 的那个 JSON 判断——**Jev 天生就是干这个的**。

### 2.2 Jev-Mem（arXiv:2609.23986, 2026-09-21）
- 论文标题：*Jev-Mem: System-One-Controlled Agentic Memory for Efficient AI Agents*（UT Dallas，[arXiv](https://arxiv.org/abs/2609.23986)，[代码 MIT 开源](https://github.com/libingzheren/Jev-Mem)）。
- **核心架构**（和你讨论的 §8 闭环图高度一致）：
  - **System-One 控制面**：用 Jev 做记忆 typing、关系判断、query routing、检索预算分配、图遍历、候选打分、自适应停止——全是"有界、高频、非生成"的决策。
  - **共享记忆数据面**：canonical 观测节点 + **四种关系图视图（semantic / temporal / causal / entity）** + 向量索引 + 词法索引。
  - **System-Two 推理面**：只有最终答案合成才调用大 LLM。
- **写入路径**：`observation → type → candidates → relations → store`。默认 **保留每一条有效观测**（`admission_enabled=false`），不在入库时做不可逆丢弃——**选择性放在"建关系"和"检索"阶段**，这恰好印证了你 C4 的"别真删，做状态流转"。
- **读取路径**：`route → retrieve → assess → expand → reassess` 的闭环，带证据充分性判断和自适应停止（公式 16–26）。
- **效果（LoCoMo 基准，GPT-4o-mini 作答）**：
  - 总分 **0.777**，比最强基线 MAGMA(0.700) **相对提升 11%**；
  - 记忆构建 **158s**，比最快基线 **快 6.6×**；
  - 查询延迟 **0.93s**，**降低 36.7%**。
  - 对抗类问题（Adversarial）0.962，远超基线。

### 2.3 本地可跑的替代：Laya
- Jev 要 TypeSafe API key（云端、收费）。**[Laya](https://huggingface.co/convaiinnovations/laya)（convaiinnovations, Apache-2.0）是同类"System-One 决策模型"，可本地跑**（Transformers / PyTorch / Apple MLX），Jev-Mem 已支持用 Laya 替换 Jev 做本地决策，无需 API key。
- 对你（Windows + 本地优先 + 已在用 Ollama embedding）来说，**Laya 是更现实的 System-One 后端**。

**结论：你讨论的每一个核心机制，现在都有"论文 + 开源代码 + 可本地运行的模型"三件套支撑。这已经从"想法"变成"可复现的工程"。**

---

## 3. 最关键发现：你本地已经有半个 Jev-Mem 了

我在你的环境里（`C:\Users\Pop\.dsh\profiles\web`）发现已安装 **`dsh-mnemon` 0.5.24** 全家桶。它不是玩具，是一套**可组合、视图化的记忆插件框架**，而且和 Jev-Mem 的对应关系惊人：

| Jev-Mem / 你的讨论 | dsh-mnemon 已有实现 | 对应度 |
|---|---|---|
| 记忆分层（C1） | **三种记忆**：Runtime memory（USER.md/MEMORY.md 常驻）、Project Documents（按需搜）、Memory Spaces（长期证据） | ✅ 完全对应 |
| Log ≠ Memory（C2） | Runtime memory 是"压缩热记忆"，Documents/Spaces 是"按需检索"，日志另存 | ✅ |
| 四关系图（C8） | `EdgeType = 'temporal' \| 'semantic' \| 'causal' \| 'entity'`（**逐字一致**） | ✅✅ 惊人 |
| Memory Router（C6） | **Strategy**（Layered / General）+ `MemorySourceMode: 'eager' \| 'routed'` | ✅ |
| 记忆竞争打分（C7） | `RecallQualityConfig`：lowScore/highScore 阈值、candidateMultiplier、relevanceTier(high/medium/low/unknown) | ✅ 部分（目前是确定性策略，非模型打分） |
| 主动写入判断（C3） | **auto-capture 策略扩展**（capture slot）：引导 LLM 每轮最多记一条持久事实 | ⚠️ 目前靠大 LLM，非 Jev |
| 遗忘 / 生命周期（C4/C5） | `forget`、`archive` capability + `deletionMode: soft/hard` + Document 冷归档 | ✅ |
| 多 Provider（C9 精神） | 9 个 Provider：Mnemon Native（本地）、Mem0、Honcho、Supermemory、ByteRover… | ✅ |
| MCP 暴露（C9） | DSH 本身支持 MCP bundle（`dsh-mcp-client`） | ✅ |

**这意味着：**

1. 你讨论的"Memory Operating System"愿景，**dsh-mnemon 已经实现了 70–80% 的骨架**（分层、四关系图、路由、软删除/归档、多 Provider、可组合插件）。
2. 它目前缺的、也正是 Jev-Mem 论文的贡献点：**用一个轻量 System-One 决策模型（Jev/Laya）替换掉当前"确定性阈值 + 大 LLM 判断"的记忆控制**。
3. dsh-mnemon 明确声明 **"Sources and Strategies are ordinary DSH plugins built on public SDKs"**，且提供 `dsh-mnemon/extension-sdk`（`defineMemoryStrategy` / `defineMemorySource` / `defineMemoryViewExtension`）和 `provider-sdk`。**扩展点是为第三方插件设计的。**

---

## 4. "落地成插件"的三种形态（按投入从小到大）

### 形态 A：Jev/Laya 驱动的 Strategy 插件（推荐起点）★
- **是什么**：写一个 `dsh-mnemon-strategy-jev`（或 `...-system-one`），实现 `MemoryStrategyDefinition.compose()` + `createTurn()`。在 compose 阶段用 Jev/Laya 做 query routing（决定激活哪些关系视图、分配多少检索预算），在读取阶段用模型做候选打分和自适应停止。
- **复用**：dsh-mnemon 已有的 Sources（Runtime/Documents/Spaces）、四关系图、Provider、UI、预算控制全都不动。
- **新增**：一个 Jev/Laya 客户端（HTTP 或本地推理）+ 一个 Strategy 实现 + 配置面板（`defineMemoryStrategyConfiguration`）。
- **工作量**：**中**。核心是把 Jev-Mem 论文 §3.3 的路由/打分/停止逻辑，翻译成 dsh-mnemon 的 `MemoryViewSpec` / `MemoryEvidence` 契约。
- **价值**：直接验证你 C6/C7/C8 的"模型驱动 Router"是否比现有确定性策略召回更准、更省 token。

### 形态 B：Jev 驱动的 Capture/Maintenance 扩展（最小可行）★★
- **是什么**：仿照现成的 `dsh-mnemon-strategy-auto-capture`，写一个 capture slot 扩展，但**把"记什么/重要性/类型/TTL"的判断从大 LLM 换成 Jev 的 `noul`+`choice`+`score`**（正好对应你 C3 的 JSON）。再加一个定时/空闲触发的 **maintenance** 任务做 C5 的冲突检测与 supersede。
- **复用**：auto-capture 的整个插件骨架（我已读到它的完整源码，~96 行，结构清晰可照抄）。
- **工作量**：**小**。这是验证"Jev 当记忆判断器"最快的路径，1 个插件 + 1 个 Jev 客户端即可。
- **价值**：直接验证 C3/C4/C5——你讨论里最有原创性的"主动遗忘 + 生命周期"。

### 形态 C：独立 Memory Middleware + MCP（你讨论的 C9 原话）
- **是什么**：不依赖 dsh-mnemon，直接 fork Jev-Mem 的 Python 代码，包一个 MCP server，暴露给 Codex / Claude Code / DSH 通用调用。
- **复用**：Jev-Mem 开源代码（MIT）。
- **工作量**：**大**。要自己处理存储、并发、跨 Agent 协议、UI。而且会和你已有的 dsh-mnemon **功能重叠、数据割裂**。
- **评价**：除非目标是"做一个跨 Agent 的通用记忆中间件产品"，否则**不推荐**——你本地已经有更成熟的 dsh-mnemon，重复造轮子收益低。

**推荐路线：B（先验证判断器）→ A（再升级 Router）→ 视情况考虑 C。**

---

## 5. 可行性评估（诚实版）

### 5.1 技术可行性：高 ✅
- 论文 + 开源代码 + 可本地模型（Laya）三件套齐全。
- 目标宿主（dsh-mnemon）有**公开、稳定、为第三方设计的扩展 SDK**，且架构与论文同构。
- 有现成同类插件（auto-capture）可照抄结构。

### 5.2 真实约束与风险 ⚠️

| 风险 | 说明 | 缓解 |
|---|---|---|
| **R1 Jev 是云端收费 API** | TypeSafe Jev 需 API key、按 token 计费、数据出本地 | 用 **Laya 本地推理**替代（Jev-Mem 已支持）；或仅在"高价值判断"上调用 Jev |
| **R2 Laya 本地推理的工程成本** | Windows + Node 环境跑 PyTorch/ONNX 模型，需解决进程隔离、首次下载、显存/内存 | 用独立 Python 子进程 + HTTP/stdio 桥接（dsh-mnemon 的 `runProcess` / MCP stdio 都支持）；或 ONNX Runtime |
| **R3 延迟 vs 收益** | System-One 的卖点是"快"，但本地小模型 + 每条记忆多次判断，可能比直接让大 LLM 判断更慢 | 批量判断（Jev 支持一次问多题）；只在写入/maintenance 用，读取热路径用确定性策略兜底 |
| **R4 与 dsh-mnemon 版本耦合** | 它支持 DSH 0.2.0-rc.2 / 0.1.7-rc.2，API 在 rc 阶段可能变动 | 锁定 `dsh-mnemon/extension-sdk` 版本；用 `defineMemory*` 官方 helper 而非内部结构 |
| **R5 评测缺失** | "记忆质量"很难自动衡量，论文用 LoCoMo，你的真实场景没有 ground truth | 先做 A/B：同一批对话，确定性策略 vs Jev 策略，人工抽查召回准确率 + token 消耗 |
| **R6 过度工程** | 你讨论的"记忆竞争 5 因子打分""MoE 多专家"全上，复杂度爆炸 | **MVP 只做 C3（写入判断）+ C6（路由二选一）**，其余迭代 |

### 5.3 投入产出判断
- **形态 B（MVP）**：~2–4 个有效工作日，能跑通"Jev/Laya 判断一条记忆该不该记、什么类型、什么 TTL"，并接到 dsh-mnemon 的 capture slot。**值得做，风险低。**
- **形态 A**：在 B 验证有效后，~1–2 周升级到模型驱动 Router。**有价值，但要 R5 的评测支撑。**
- **形态 C**：除非要做产品，否则**暂缓**。

---

## 6. 建议的下一步（MVP 蓝图）

如果决定动手，第一个可交付物（形态 B）：

```
dsh-mnemon-strategy-jev/
├── package.json              # dsh.bundle.patch 声明
├── cordis.patch.yml          # 挂载为 strategy-extension(capture slot)
├── lib/
│   ├── index.js              # apply() + defineMemoryViewExtension (照抄 auto-capture)
│   ├── jev-client.js         # Jev HTTP 客户端 或 Laya 子进程桥
│   └── judge.js              # 把对话片段 → Jev questions(noul/choice/score) → C3 的 JSON
└── locale/{en,zh}.json
```

**核心判断逻辑（对应你 C3）**：
```js
// 给 Jev 的 state = 候选记忆片段
// questions:
//   remember:   { type:'noul',  instructions:'这条信息是否值得长期记住？' }
//   scope:      { type:'choice',criteria:{user,project,session,none} }
//   type:       { type:'choice',criteria:{preference,constraint,fact,decision,episodic} }
//   importance: { type:'score', criteria:['可忽略','一般','重要','关键'] }
// → 输出 { remember, importance, scope, type, ttl } 写入对应 Memory Space
```

**验证方式**：拿你最近几段真实对话，跑 auto-capture（大 LLM 判断）vs jev-strategy（Jev/Laya 判断），对比：记住了什么、漏了什么、误记了什么、token/延迟开销。

---

## 7. 总评

| 维度 | 评分 | 说明 |
|---|---|---|
| 想法原创性 | ★★★★☆ | C4/C5/C7（生命周期+主动遗忘+记忆竞争）是真正有价值的部分，且被论文验证 |
| 技术成熟度 | ★★★★★ | 论文+开源+本地模型三件套齐全 |
| 落地可行性 | ★★★★☆ | 本地已有同构骨架 dsh-mnemon，扩展点公开 |
| 投入产出比 | ★★★★☆（形态B）/ ★★★☆☆（形态A）/ ★★☆☆☆（形态C） | 从 capture 扩展切入最划算 |
| 主要风险 | R1/R2/R3 | Jev 云端收费 → 用 Laya 本地；延迟 → 批量+热路径兜底 |

**一句话**：你这不是"要不要落地"的问题，而是"在已有的 dsh-mnemon 上，加一个 Jev/Laya 决策层"的增量工程问题。**强烈建议从形态 B（capture 判断扩展）做 MVP**，用真实对话验证"小模型判断记忆"是否比"大 LLM 判断"更省更准，再决定是否升级到形态 A 的完整 Router。

---

## 附：参考资料
- Jev-Mem 论文：https://arxiv.org/abs/2609.23986
- Jev-Mem 代码（MIT）：https://github.com/libingzheren/Jev-Mem
- Jev 模型文档（TypeSafe/Cloudflare）：https://developers.cloudflare.com/ai/models/typesafe/jev/
- Laya 本地决策模型（Apache-2.0）：https://huggingface.co/convaiinnovations/laya
- Jev 介绍：https://www.infoq.com/news/2026/10/typesafe-ai-jev-released/
- dsh-mnemon（本地已装 0.5.24）：https://github.com/omdsh-dev/dsh-mnemon
- 本地插件骨架参考：`dsh-mnemon-strategy-auto-capture`（capture slot 扩展，~96 行）
- 本地契约：`dsh-mnemon/lib/types/src/core/contracts/view.d.ts`（Strategy/Source/View 契约）
