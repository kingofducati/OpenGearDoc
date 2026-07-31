# OpenGear

> **AI Agent Framework** — Let the LLM be controlled by the GearFlow engine, not the LLM controlling the Agent.
> **Core Idea**: OpenGear adopts a **dual-engine architecture** — the **GearFlow engine drives workflow execution** ("how to execute") and **GearDecision drives authorization decisions** ("whether to execute"). The two engines collaborate as peers, together ensuring system behavior stays controllable. The LLM acts as a reference for the engines, taking on logical reasoning and intent translation when needed, but decision authority always belongs to the engines.
>
> **Latest Version**: **v0.8.2** (2026-07-30, dual-engine architecture; decision engine renamed to GearDecision; security architecture moved up to Chapter 12)
>
> **Current Milestone**: Phase 7v3 6-dimension code quality full wrap-up (R106-R190, 85 Rounds, 2425 tests with 0 failures)
> - Dimension 1 Authenticity (workGuide contract) ✅ / Dimension 2 @Slf4j (5 files) ✅ / Dimension 3 Constructor Injection (AgentRegistry) ✅
> - Dimension 4 @Query (9 Dao + 9 Repository) ✅ / Dimension 5 Spring Test (14 Tests + 19/19 deps) ✅ / Dimension 6 Javadoc (19/19 sub-modules with 0 files lacking Javadoc) ✅

---

## One-Sentence Summary

OpenGear uses the **GearFlow engine + GearDecision** as drivers, while the LLM steps back to **serve as a reference and translator** — taking on logical reasoning and intent translation when needed, but decision authority and execution authority always belong to the engines. Execution is jointly carried out by the workflow skeleton, the GearDecision decision engine, Supervisor validation, and precision-model-driven Capabilities.

---

## Dual-Engine Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                    OpenGear Dual-Engine Architecture                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────┐              ┌─────────────────────┐      │
│  │  GearFlow Engine     │   Peer-level  │  GearDecision Engine │      │
│  │  (How to execute)    │  ◀────────▶  │  (Whether to execute)│      │
│  │                     │  Collaboration│                     │      │
│  │  - Planner generates │              │  - 4-Tier Decisions:  │      │
│  │    JSON              │              │    BYPASS / NOTIFY    │      │
│  │  - Executor executes │              │    HUMAN_CONFIRM      │      │
│  │    nodes            │              │    BLOCK              │      │
│  │  - Supervisor        │              │  - Two-level Rules:   │      │
│  │    validates         │              │    user-level +         │      │
│  │                     │              │    capability-level    │      │
│  │                     │              │  - DecisionCapability: │      │
│  │                     │              │    rule distillation + │      │
│  │                     │              │    human interaction  │      │
│  └─────────────────────┘              └─────────────────────┘      │
│            ▲                                       ▲                │
│            └──────────── LLM as Translator ─────────────┘           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Chapter Navigation

The complete architecture documentation is in [`wiki`](OpenGearDoc/wikis). It is organized in five major parts: **"Problem → Core Engines → Supporting Systems → Security + Extensions → Reference + Implementation"**:

| Chapter Range | Topic |
|---------------|-------|
| **Ch 1-2** | Why OpenGear, Core Architecture (dual-engine + translation/control separation + UML) |
| **Ch 3-4** | Core Engines: **GearFlow Workflow Engine** + **GearDecision Decision Engine** (peer-level since v0.8.2) |
| **Ch 5-11** | Supporting Systems: Memory / Capability / Precision Model / Frontend / A2A Protocol / Production-Line Engine / Data Architecture |
| **Ch 12-16** | Security + Extensions: Security Architecture (with GearDecision application) / Observability / DR & DevOps / Microservice Architecture / Testing & TDD |
| **Ch 17-18** | Reference + Implementation: References (with 55 external resources) / Implementation Plans (naming rule `18.XX-YYYYMMDD-<name>.md`) |

> ⚠️ Documentation lives in two parallel repositories:
> - **GitHub Wiki Repo**: [`OpenGearDoc.wiki`](https://github.com/kingofducati/OpenGearDoc/wiki) (used as the `wiki-temp/` submodule of this repo)
> - **This Repo**: [`open-gear`](https://gitee.com/duxu2004/open-gear) (Java + Vue + Gradle + Docker + K8s)

## Bootstrapping Infrastructure (Postgres / Redis / RocketMQ / MinIO / Nacos)

```bash
cd docker
docker-compose up -d
```

## Compilation

```bash
# Compile a single module
gradlew :opengear-agent-engine:compileJava

# Compile all modules
gradlew compileJava
```

---

## Run Tests

```bash
# All-module tests
gradlew test

# Single test suite
gradlew :opengear-agent-engine:test --tests "io.opengear.agentengine.risk.RiskRegistryTest"

# v2 decision engine-related tests
gradlew :opengear-agent-engine:test --tests "io.opengear.agentengine.engine.*" --tests "io.opengear.agentengine.auth.*" --tests "io.opengear.agentengine.web.Decision*"

# GearDecision-related tests
gradlew :opengear-decision-service:test
```

---

## Implementation Principles (**Most Important**)

- **JDK**: Use JDK 17
- **Strict 1:1 Implementation**: Design docs translate into Java code, **prohibiting "load-reduction refactors", "simplifications", placeholder implementations and other self-deceptive / lazy logic**
- Every branch, condition, and path in design docs MUST appear in the Java implementation
- Code MUST strictly follow Javadoc conventions
- Use Lombok `@Data`/`@Getter`/`@Setter`
- Field injection is replaced by Constructor injection
- `@Query` JPQL/SQL annotations + Specification
- Use Spring Test (`@SpringBootTest`/`@MockBean`/`TestRestTemplate`)
- Batch rhythm: implement in batches

### Split Strategy for Large Java Files

For Java files exceeding 500 lines, split them using these patterns:

- **Composition Pattern**: A large method is composed of multiple independent sections → `orchestrator` + N `*Filter` small classes
  ```java
  // The large method is split into independent small classes
  class RequestPipeline {
    private List<RequestFilter> filters;
    public Response execute(Request req) {
      for (RequestFilter f : filters) f.apply(req);
    }
  }
  ```
- **Inheritance Pattern**: Different files provide independent features → abstract base class + multiple subclasses
  ```java
  abstract class BaseCapability { /* shared code */ }
  class BashCapability extends BaseCapability { /* specific implementation */ }
  class FileCapability extends BaseCapability { /* specific implementation */ }
  ```
- **Proxy Pattern**: Heavy delegation to helper classes → Spring `@Autowired` injects the helper, method body delegates with one line
  ```java
  class ChatController {
    @Autowired AuthorizationRequestBroadcaster broadcaster;
    public ResponseEntity<?> respond(...) { return broadcaster.broadcast(...); }
  }
  ```

Splitting principles:

- ✅ Strict 1:1 port-first
- ✅ After splitting, logic must remain 100% equivalent to design docs
- ❌ "Load-reduction refactoring" / "simplification" is forbidden (e.g., simplifying conditions, merging logic)

### Code Comment Rules (**Mandatory production-grade Javadoc**)

See details in [`wiki-temp/03-工作流引擎/03.04-comment-style.md`](wiki-temp/03-工作流引擎/03.05-四层Executor) (02-Core Architecture §Javadoc Rules). Key points:

1. **Class-level Javadoc**: Java implementation source reference + What/How/Why + design trade-offs + lines-to-implementation mapping
2. **Method-level Javadoc**: Function description + `@param`/`@return`/`@throws` + side effects
3. **Field-level Javadoc**: Field purpose + unit/format + constraints + field name + business rules
4. **Internal Implementation**: Key algorithm steps + decision points + performance considerations + deltas vs. design docs
5. **Comment Language**: Javadoc body in Chinese, inline `//` in Chinese, technical terms retained in English

### TDD Workflow (Established)

- 🔴 **RED**: Write the test first (JUnit 5)
  - Each test covers exactly one behavior
  - Use human-language descriptions for test names (not `testFoo`)
  - TDD failure messages must provide diagnostic value
- 🟢 **GREEN**: Write the implementation that just barely passes the test
  - Minimal implementation, no premature optimization
  - Compiling/passing tests is the only standard
- ✅ **VERIFY**: `gradlew test` full run
  - Run all-module tests, confirm no regressions
  - Performance baseline (e.g., decision P99 < 50ms)
- 📝 **COMMIT**: Single commit message describing the actual file list
  - Format: `[Round N] <description>`
  - Include the touched files list + touched line count

### Batch Rhythm

After each batch completes, immediately execute in this order:

1. `gradlew test` all-module tests pass (`opengear-agent-engine:test`)
2. Update this README.md (progress table numbers + ✅ list + stats time)
3. Update `memory/YYYY-MM-DD-rN.md` (detailed meeting notes) — removed by v0.8.2 repo hygiene cleanup
4. `git add <exact file list>` → `git commit -m "Round N: <description>"` → `git push --force-with-lease origin <branch>`

---

## Contributors

- 📌 **Architecture Decisions + Documentation Content**: Project Owner
- 🔧 **AI-Assisted Construction**: MiniMax-M3 (MiniMax, 2026)
  - 18.46/18.40 Decision-system design
  - Documentation chapter reordering / numbering normalization / dual-engine architecture write-up
  - Security architecture references to risk-control / payment / anti-fraud system design
- 🤝 **Contribution Method**: See [`wiki-temp/17-参考资料/`](wiki-temp/17-参考资料/) for the git management guidelines

---

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
