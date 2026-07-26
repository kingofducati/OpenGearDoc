# OpenGear

> **Core Idea**: Let the LLM be controlled by the GearFlow engine, rather than letting the LLM control the Agent.
>
> The fundamental flaw of traditional AI Agents is forcing the LLM to act as both translator and decision-maker—but the LLM is a probabilistic model and cannot precisely control execution. OpenGear solves this contradiction through **four-layer responsibility separation**.

📖 **Complete Architecture Document → [Wiki ‌Home](https://github.com/kingofducati/OpenGearDoc/wiki)**

## One-Sentence Summary

OpenGear treats the LLM as a **translator** (intent → GearFlow workflow JSON), and **unifies decision-making authority in the GearFlow engine**. Execution is carried out by the workflow skeleton, the decision engine, Supervisor validation, and precision-model-driven Capabilities working together.

## Four-Layer Architecture

| Layer | Role | Implementer | Determinism | Responsibility |
|------|------|-------------|-------------|----------------|
| 1 | **LLM (Translator)** | Precision models 1.5B-3B | Probabilistic | Translate intent into workflow JSON only |
| 2 | **Workflow (Skeleton)** | JSON config files | **Deterministic** | Define node types, connections, workGuide |
| 3 | **Capability (Hand)** | Plugins + precision models | Probabilistic | Declare capabilities + authorization rules, **no decision authority** |
| 4 | **GearFlow Engine** | Framework core | **Deterministic** | Schedule Planner / Executor / Supervisor / Decision Engine |

**Core formula**: `GearFlow Engine = PlannerCapability + ExecutorCapability + Decision Engine + Supervisor + Session DAG`

## Core Design Principles

1. **Decision authority belongs to the engine**—Capabilities only declare rules (`decisionRule()`); the engine interprets and enforces them
2. **LLM only translates, never decides**—Once the translation (workflow JSON) is produced, the LLM no longer participates in execution control
3. **Workflows are deterministic skeletons**—Node types, connections, and workGuide are fully structured with zero decision semantics
4. **Three-tier workflows** (Task / Session / Production)—Precision ranges from a single conversation to cross-session reuse
5. **Supervisor validates every node**—Double safety with `supervisor-verify-1.5b` + `supervisor-llm-1.5b`
6. **Precision models drive Capabilities**—Each Capability binds a 1.5B-3B precision model running locally at ~70ms latency
7. **Self-upgrading learning library**—Every successful translation `(userInput → workflow)` is stored in the learning library

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Java 21 |
| Build | Gradle |
| Backend | Spring Boot 3 + WebFlux (reactive) |
| Frontend | Vue 3 + Vite |
| Database | PostgreSQL + pgvector (vector retrieval), SQLite + CAS (Session DAG) |
| Messaging | MQTT (EMQX, A2A protocol implementation) |
| Models | Ollama local inference + cloud fallback (planner-1.5b, supervisor-verify-1.5b, etc.) |

## Core Engines

### GearFlow Workflow Engine
- JSON workflow model: `agentTask` / `humanTask` / `decisionTask` and 7 node types in total
- Three-tier workflows: Task (single round) → Session (multi-round) → Production (cross-session)
- Topological sort (Kahn algorithm) + sequential execution + Supervisor validation
- Decision engine: four-level authorization—BYPASS / NOTIFY / HUMAN_CONFIRM / BLOCK

### Memory Architecture
- 4-tier memory: Session → Working → Project → User
- Two-phase consolidation: real-time compression + offline distillation
- FTS5 full-text retrieval + pgvector vector retrieval

### Capability System
- 13+ built-in Capabilities: Prompt, Memory, Filesystem, Shell, Browser, Python, CoreLLM, Human, etc.
- Unified JSON specification (`NaturalLanguageCapability` interface)
- Each Capability declares `permissionLevel` + `decisionRule()`
- Extension point: third-party plugins dynamically loaded via registry

### Precision Models
- Two-tier configuration: `PreciseModel` (specific model binding) + `CapabilityLLMModel` (capability-level config)
- 4-level fallback chain: local precision model → local general model → cloud precision model → cloud general model
- Each Capability binds a 1.5B-3B precision model running locally

## Production Line Engine (Multi-Agent Collaboration)

- Production Flow workflows: predefined skeleton + role division + workGuide
- A2A Protocol (Agent-to-Agent): message format, routing, MQTT integration
- PMBOK knowledge-area-driven project memory management
- Production line distillation: automatically distill Production Flow templates from successful executions

## Microservices Architecture

13 components → 6 microservices: Gateway, Auth, Agent, Memory, Production, Model

## Implementation Status

| Component | Status |
|-----------|--------|
| GearFlow Engine | ✅ Core implemented |
| planning Capability | ✅ Core implemented |
| Capability interface | ✅ 23 implementations |
| Session DAG | ✅ Core implemented |
| Working Memory | ✅ Core implemented |
| Decision Engine | ✅ Basic implementation |
| Project / User Memory | 📝 Partially implemented |
| Production Line (multi-agent) | 📝 Skeleton ready |
| Microservices split | ⚠️ Not started |

### Implementation Phases

- **Phase 1** (current ~2026-08): Single Agent core stabilized
- **Phase 2** (2026-08 ~ 2026-10): Memory system complete + Capability expansion
- **Phase 3** (2026-10 ~ 2027-02): Multi-agent collaboration + microservices split + production-ready

## Architecture Documentation

Complete architecture documentation is available in the [Wiki](https://github.com/kingofducati/OpenGearDoc/wiki), covering 15 chapters:

| Chapter | Content |
|---------|---------|
| 01 Overview | Why OpenGear is needed |
| 02 Core Architecture | Philosophy, task state, data flow, UML design, cache architecture |
| 03 Workflow Engine | JSON model, dynamic generation, three-tier workflow, decision engine |
| 04 Memory Architecture | 4-tier memory, consolidation, retrieval, storage |
| 05 Capability System | 13 Capabilities + unified JSON specification |
| 06 Precision Models | Design API, 4-level fallback chain, training standard |
| 07 Frontend | ChatUI, authorization confirmation dialog, workflow panel |
| 08 A2A Protocol | Message format, routing, MQTT |
| 09 Production Line Engine | Multi-agent collaboration, PMBOK, distillation |
| 10 Data Architecture | 53 PostgreSQL tables, cache and messaging |
| 11 Observability | Monitoring, logging, tracing, alerting |
| 12 Disaster Recovery & DevOps | Backup and recovery, CI/CD |
| 13 Microservices Architecture | 6-service split, ACP communication |
| 14 Testing & TDD | Testing strategy, engine decision unit tests |
| 15 Security Architecture | Authentication, authorization, encryption, audit |

## Reading Roadmap

| Role | Reading Path | Estimated Time |
|------|--------------|----------------|
| Decision-maker / Architect | Ch1-2 → Ch3 → Ch5-8 → Ch13 | 2h |
| Backend Engineer | Ch3 → Ch5 → Ch6 → Ch13-15 | 4h |
| New Engineer | Read chapters in order, start with each chapter's "Chapter Guide" | 6h |

## License

Apache 2.0