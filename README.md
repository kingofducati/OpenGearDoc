# OpenGear

> **核心思想**：让 LLM 受 GearFlow 引擎控制，而非让 LLM 控制 Agent。
>
> 传统 AI Agent 的根本缺陷是让 LLM 同时做翻译和决策——但 LLM 是概率性模型，无法精确控制执行。OpenGear 通过**四层职责分离**解决这个矛盾。

## 一句话总结

OpenGear 把 LLM 当**翻译官**（意图 → GearFlow 工作流 JSON），把**决策权统一上移到 GearFlow 引擎**，由工作流骨架 + 决策引擎 + Supervisor 验证 + 精准模型驱动的 Capability 共同执行。

## 四层架构

| 层级 | 角色 | 承担者 | 确定性 | 职责 |
|------|------|--------|--------|------|
| 1 | **LLM（翻译官）** | 精准模型 1.5B-3B | 概率性 | 只把意图翻译为工作流 JSON |
| 2 | **工作流（骨架）** | JSON 配置文件 | **确定性** | 定义节点类型、连接关系、workGuide |
| 3 | **Capability（手）** | 插件 + 精准模型 | 概率性 | 声明能力 + 声明鉴权规则，**不持决策权** |
| 4 | **GearFlow 引擎** | 框架核心 | **确定性** | 调度 Planner / Executor / Supervisor / 决策引擎 |

**核心公式**：`GearFlow 引擎 = PlannerCapability + ExecutorCapability + 决策引擎 + Supervisor + Session DAG`

## 核心设计原则

1. **决策权归引擎**——Capability 仅声明规则（`decisionRule()`），引擎解释并执行
2. **LLM 只翻译，不决策**——翻译结果（工作流 JSON）一旦生成，LLM 不再参与执行控制
3. **工作流是确定性骨架**——节点类型、连接关系、workGuide 全部结构化，零决策语义
4. **三层工作流**（Task / Session / Production）——精度从单轮对话到跨 Session 复用
5. **Supervisor 验证每个节点**——`supervisor-verify-1.5b` + `supervisor-llm-1.5b` 双保险
6. **精准模型驱动 Capability**——每个 Capability 绑定 1.5B-3B 精准模型，本地运行 ~70ms 延迟
7. **学习库自升级**——每次成功翻译的 `(userInput → workflow)` 存入学习库

## 技术栈

| 层面 | 技术 |
|------|------|
| 语言 | Java 21 |
| 构建 | Gradle |
| 后端 | Spring Boot 3 + WebFlux（反应式） |
| 前端 | Vue 3 + Vite |
| 数据库 | PostgreSQL + pgvector（向量检索）、SQLite + CAS（Session DAG） |
| 消息 | MQTT（EMQX，A2A 协议实现） |
| 模型 | Ollama 本地推理 + 云端 Fallback（planner-1.5b、supervisor-verify-1.5b 等） |

## 核心引擎

### GearFlow 工作流引擎
- JSON 工作流模型：`agentTask`/`humanTask`/`decisionTask` 等 7 种节点
- 三层工作流：Task（单轮）→ Session（多轮）→ Production（跨 Session）
- 拓扑排序（Kahn 算法）+ 按序执行 + Supervisor 验证
- 决策引擎：BYPASS / NOTIFY / HUMAN_CONFIRM / BLOCK 四档鉴权

### 记忆架构
- 4 层记忆：Session → Working → Project → User
- 两阶段 Consolidation：实时压缩 + 离线蒸馏
- FTS5 全文检索 + pgvector 向量检索

### 能力系统（Capability）
- 13+ 个内置 Capability：Prompt、Memory、Filesystem、Shell、Browser、Python、CoreLLM、Human 等
- 统一 JSON 规范（`NaturalLanguageCapability` 接口）
- 每个 Capability 声明 `permissionLevel` + `decisionRule()`
- 扩展点：第三方插件通过 registry 动态加载

### 精准模型
- 两层配置体系：`PreciseModel`（具体模型绑定）+ `CapabilityLLMModel`（能力级配置）
- 4 级降级链：本地精准模型 → 本地通用模型 → 云端精准模型 → 云端通用模型
- 每个 Capability 绑定 1.5B-3B 精准模型，本地运行

## 生产线引擎（多 Agent 协作）

- Production Flow 工作流：预定义骨架 + 角色分工 + workGuide
- A2A 协议（Agent-to-Agent）：消息格式、路由、MQTT 集成
- PMBOK 知识领域驱动的项目内存管理
- 生产线蒸馏：从成功执行中自动提炼 Production Flow 模板

## 微服务架构

13 组件 → 6 微服务：Gateway、Auth、Agent、Memory、Production、Model

## 实施状态

| 组件 | 状态 |
|------|------|
| GearFlow 引擎 | ✅ 核心实现 |
| planning Capability | ✅ 核心实现 |
| Capability 接口 | ✅ 23 个实现 |
| Session DAG | ✅ 核心实现 |
| Working Memory | ✅ 核心实现 |
| 决策引擎 | ✅ 基础实现 |
| Project / User Memory | 📝 部分实现 |
| 生产线（多 Agent） | 📝 骨架就绪 |
| 微服务拆分 | ⚠️ 未开始 |

### 实施阶段

- **Phase 1**（当前~2026-08）：单 Agent 核心稳定
- **Phase 2**（2026-08~2026-10）：记忆系统完整 + Capability 扩展
- **Phase 3**（2026-10~2027-02）：多 Agent 协作 + 微服务拆分 + 生产就绪

## 架构文档

完整架构文档见 [Wiki](https://github.com/kingofducati/OpenGearDoc/wiki)，涵盖 15 个章节：

| 章节 | 内容 |
|------|------|
| 01 概述 | 为什么需要 OpenGear |
| 02 核心架构 | 哲学、任务状态、数据流、UML 设计、缓存架构 |
| 03 工作流引擎 | JSON 模型、动态生成、三层工作流、决策引擎 |
| 04 记忆架构 | 4 层记忆、Consolidation、检索、存储 |
| 05 能力系统 | 13 个 Capability + 统一 JSON 规范 |
| 06 精准模型 | 设计 API、4 级降级链、训练标准 |
| 07 前端界面 | ChatUI、鉴权确认弹窗、工作流面板 |
| 08 A2A 协议 | 消息格式、路由、MQTT |
| 09 生产线引擎 | 多 Agent 协作、PMBOK、蒸馏 |
| 10 数据架构 | 53 张 PG 表、缓存与通信 |
| 11 可观测性 | 监控、日志、追踪、告警 |
| 12 灾备与 DevOps | 备份恢复、CI/CD |
| 13 微服务架构 | 6 服务拆分、ACP 通信 |
| 14 测试与 TDD | 测试策略、引擎决策单测 |
| 15 安全架构 | 认证、授权、加密、审计 |

## 阅读路线图

| 角色 | 阅读路径 | 预计 |
|------|---------|------|
| 决策者/架构师 | 第1-2章 → 第3章 → 第5-8章 → 第13章 | 2h |
| 后端工程师 | 第3章 → 第5章 → 第6章 → 第13-15章 | 4h |
| 新加入工程师 | 按章节顺序，每章先读"本章导读" | 6h |

## License

Apache 2.0