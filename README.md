# OpenGear

> **AI Agent 框架** —— 让 LLM 受 GearFlow 引擎控制，而非让 LLM 控制 Agent。
> **核心思想**：OpenGear 采用**双引擎架构**——**GearFlow 引擎驱动工作流执行**（怎么执行），**GearDecision 驱动鉴权判定**（该不该执行）。两个引擎平级协同，共同保证系统行为可控。LLM 作为引擎的参谋，在需要时承担逻辑推理和意图翻译，决策权始终归于引擎。
>
> **最新版本**：**v0.8.2**（2026-07-30，双引擎架构；决策引擎重命名为 GearDecision；安全架构前置第12章）
>
> **当前里程碑**：Phase 7v3 6 维度代码质量全面收官（R106-R190，85 Rounds，2425 测试 0 失败）
> - 维度 1 真实性（workGuide 契约）✅ / 维度 2 @Slf4j（5 文件）✅ / 维度 3 构造器注入（AgentRegistry）✅
> - 维度 4 @Query（9 Dao + 9 Repository）✅ / 维度 5 Spring Test（14 Tests + 19/19 dep）✅ / 维度 6 Javadoc（19/19 子模块 0 文件无 Javadoc）✅

---

## 一句话总结

OpenGear 以 **GearFlow 引擎 + GearDecision** 为驾驶者，LLM 退居**参谋与翻译官**——在需要时承担逻辑推理和意图转译，但决策权与执行权始终归于引擎。由工作流骨架 + GearDecision 决策引擎 + Supervisor 验证 + 精准模型驱动的 Capability 共同执行。

---

## 双引擎架构总览

```
┌──────────────────────────────────────────────────────────────────────┐
│                          OpenGear 双引擎架构                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────────────┐              ┌─────────────────────┐       │
│  │  GearFlow 引擎       │   平级协同    │  GearDecision 决策引擎│       │
│  │  （怎么执行）         │  ◀────────▶  │  （该不该执行）        │       │
│  │                     │              │                     │       │
│  │  - Planner 生成 JSON │              │  - 4 档决策：          │       │
│  │  - Executor 执行节点 │              │    BYPASS / NOTIFY    │       │
│  │  - Supervisor 验证   │              │    HUMAN_CONFIRM      │       │
│  │                     │              │    BLOCK              │       │
│  │                     │              │  - 两级规则：          │       │
│  │                     │              │    用户级 + 能力级      │       │
│  │                     │              │  - DecisionCapability：│       │
│  │                     │              │    决策蒸馏 + 人机交互  │       │
│  └─────────────────────┘              └─────────────────────┘       │
│           ▲                                       ▲                   │
│           └─────────── LLM 作为翻译官 ─────────────┘                  │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 章节导航

完整架构文档见 [`wiki `](https://github.com/kingofducati/OpenGearDoc/wiki)。按 **"问题 → 核心引擎 → 支撑系统 → 业务安全 + 扩展 → 参考 + 实施"** 五大部分组织：

| 章节范围 | 主题 |
|---------|------|
| **1-2 章** | 为什么需要 OpenGear、核心架构（双引擎 + 翻译与控制分离 + UML） |
| **3-4 章** | 核心引擎：**GearFlow 工作流引擎** + **GearDecision 决策引擎**（v0.8.2 后独立平级） |
| **5-11 章** | 支撑系统：记忆架构 / 能力系统 / 精准模型 / 前端 / A2A 协议 / 生产线引擎 / 数据架构 |
| **12-16 章** | 业务安全 + 扩展：安全架构（含 GearDecision 应用） / 可观测性 / 灾备与 DevOps / 微服务架构 / 测试与 TDD |
| **17-18 章** | 参考 + 实施：参考资料（含 55 条外部资料引用）/ 实施计划（命名规则 `18.XX-YYYYMMDD-<name>.md`） |

> ⚠️ 文档有两套并行仓库：
> - **GitHub Wiki 仓库**：[`OpenGearDoc.wiki`](https://github.com/kingofducati/OpenGearDoc/wiki) （作为本仓库 `wiki-temp/` 子模块）
> - **本仓库**：[`open-gear`](https://gitee.com/duxu2004/open-gear)（Java + Vue + Gradle + Docker + K8s）

## 启动基础设施（Postgres / Redis / RocketMQ / MinIO / Nacos）

```bash
cd docker
docker-compose up -d
```

## 编译

```bash
# 单模块编译
gradlew :opengear-agent-engine:compileJava

# 全模块编译
gradlew compileJava
```

---

## 跑测试

```bash
# 全模块测试
gradlew test

# 单个测试套件
gradlew :opengear-agent-engine:test --tests "io.opengear.agentengine.risk.RiskRegistryTest"

# v2 决策引擎相关测试
gradlew :opengear-agent-engine:test --tests "io.opengear.agentengine.engine.*" --tests "io.opengear.agentengine.auth.*" --tests "io.opengear.agentengine.web.Decision*"

# GearDecision 相关
gradlew :opengear-decision-service:test
```

---

## 实现原则（**最重要**）

- **JDK**：使用 JDK 17
- **严格 1:1 实现**：设计文档转化为 Java 代码，**禁用"减负式重构"、"简化"、占位实现等自我欺骗、偷懒的逻辑**
- 设计文档中的每一个分支、每一个条件、每一条路径都必须出现在 Java 实现中
- 代码需要严格遵守代码注释规则对代码进行 Javadoc
- 使用 Lombok `@Data`/`@Getter`/`@Setter`
- field injection 为构造器注入
- `@Query` JPQL/SQL 注解 + Specification
- 使用 Spring Test（`@SpringBootTest`/`@MockBean`/`TestRestTemplate`）
- 按 Batch 节奏，分批实施

### 大文件代码分批策略

超 500 行以上的 Java 大文件，按以下模式拆分：

- **组合模式**：大方法由多个独立 section 组成 → `orchestrator` + N 个 `*Filter` 小类
- **继承模式**：不同文件提供独立功能 → 抽象基类 + 多个子类
- **代理模式**：大量委托给 helper 类 → Spring `@Autowired` 注入 helper，方法体一行委托

拆分原则：

- ✅ 严格 1:1 port 优先
- ✅ 拆分后逻辑与设计文档 100% 等价
- ❌ 不允许"减负式重构"、"简化"（如简化条件、合并逻辑）

### 代码注释规则（**强制 production-grade Javadoc**）

详情见 [`wiki-temp/03-工作流引擎/03.04-comment-style.md`](wiki-temp/03-工作流引擎/03.05-四层Executor) （02-核心架构 §Javadoc 规则），要点：

1. **类级 Javadoc**：Java 实现文档来源标注 + What/How/Why + 设计取舍 + 行数→Java 实现映射
2. **方法级 Javadoc**：功能描述 + `@param`/`@return`/`@throws` + 副作用
3. **字段级 Javadoc**：字段用途 + 单位/格式 + 约束 + 字段名 + 业务规则
4. **内部实现**：关键算法步骤 + 决策点 + 性能考量 + 与文档设计差异点
5. **注释语言**：Javadoc 主体中文，行内 `//` 中文，专业术语保留英文

### TDD 工作流（已固化）

- 🔴 **RED**：先写测试（JUnit 5）
  - 一个测试只测一个行为
  - 测试名用人类语言描述场景（不是 `testFoo`）
  - TDD 失败信息要有诊断价值
- 🟢 **GREEN**：编写刚好通过测试的实现
  - 最小实现，不提前考虑优化
  - 编译/测试通过是唯一标准
- ✅ **VERIFY**：`gradlew test` 全量
  - 跑全模块测试，确认无回归
  - 性能基线（如决策 P99 < 50ms）
- 📝 **COMMIT**：单条提交信息描述实际文件清单
  - 格式：`[Round N] <描述>`
  - 含触动文件清单 + 触动行数

### Batch 节奏

每批完成必须立即执行以下顺序：

1. `gradlew test` 全量测试通过（`opengear-agent-engine:test`）
2. 更新本 README.md（进度表数字 + ✅ 列表 + 统计时间）
3. 更新 `memory/YYYY-MM-DD-rN.md`（详细会议记录）— 已被 v0.8.2 仓库卫生清理删除
4. `git add <准确文件清单>` → `git commit -m "Round N: <描述>"` → `git push --force-with-lease origin <branch>`


---

## 贡献者

- 📌 **架构决策 + 文档内容**：项目所有者
- 🔧 **AI 辅助施工**：MiniMax-M3 (MiniMax, 2026)
  - 18.46/18.40 决策体系方案
  - 文档章节重排 / 编号归一 / 双引擎架构落写
  - 安全架构借鉴风控/支付/反欺诈系统设计
- 🤝 **贡献方式**：见 [`wiki-temp/17-参考资料/`](wiki-temp/17-参考资料/) 中的 git 管理规范

---

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
