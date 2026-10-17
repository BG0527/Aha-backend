# AGENTS.md — Aha Backend

> **归属**：本文件只管辖 `Aha/backend` 仓。前端域见 `Aha/frontend/AGENTS.md`。
> **设计事实源**：`D:\Develop\projects\nixiang\study\dev\`（10 份 L4 契约 + OpenAPI v1 + DDL 草案 + ADR）。
> **本仓不复制设计文档**。本文件只写「本仓怎么干活」；一切「设计成什么样」以 `dev/` 为准，冲突时以 `dev/` 为准并上报。

---

## 0. 仓库身份

| 项 | 值 |
|---|---|
| 仓路径 | `D:\Develop\projects\nixiang\study\Aha\backend` |
| 部署单元 | 单个 Spring Boot 可部署单元（模块化单体，不是微服务） |
| 语言基线 | **Java 21（LTS）** —— 不得自行降级，也不得自行升级 |
| 构建工具 | **Maven**（`apache-maven-3.9.12`，JDK 21） |
| 包根 | `com.nixiang.tutor` |
| 上层仓库 | **无**。顶层 `study/` 不是 git 仓；`competitor/` 下的嵌套仓与本仓无关，**不得修改** |

---

## 1. 技术基线（人类已冻结，不得重议）

| 决策 | 取值 | 依据 |
|---|---|---|
| 语言 | Java 21 | `dev/CONTEXT.md` D3 |
| 框架 | Spring Boot 3.x，**模块化单体** | ADR-T22 |
| 分层 | Package-by-Feature + `api / application / domain / port / adapter` | ADR-T22、`dev/60-计划与审计/M0工程骨架与实验计划.md` §2 |
| 数据库 | PostgreSQL + Flyway | ADR-T03 |
| 持久化分工 | JPA 写聚合 + MyBatis 查询 | P-064、ADR-T14 |
| 认证 | Spring Security + HttpOnly Cookie 服务端会话 | P-059、`契约-身份认证与对象级授权.md` |
| 接口 | OpenAPI spec-first 生成接口 | P-070 |
| 迁移 | Flyway，`up` 必须配 `down`，无 `down` 禁止合入 | `契约-部署与容量.md` §4、不变量 I-D4 |
| 可观测性 | 轻量 OpenTelemetry | ADR-T16 |

**首版不拆 Maven 多模块**，用 package-by-feature + ArchUnit 约束边界（理由见 `M0工程骨架与实验计划.md` §2.2）。

---

## 2. 模块边界（16 个包，代码结构的唯一坐标）

```
com.nixiang.tutor
├─ bootstrap    identity   source     evidence   learning
├─ teaching     task       playback   outcome    gateway
├─ worker       evaluation governance memory     tool
└─ context     （可与 memory 合并，拆分需 ADR）
```

规则（完整矩阵见 `dev/10-架构/架构总览-可视化AI私人教师.md` §5.5.2）：

```text
Controller/WebSocket Adapter
  → Application Command/Query
    → Domain Aggregate + Policy
      → Outbound Port
        → JPA/MyBatis/Blob/Provider Adapter
```

- `domain` **不依赖** Spring、AgentScope、JPA Entity、Provider SDK。
- `application` 负责事务边界、幂等、权限、发布协调；**不等待外部模型**。
- `task` 是**唯一**长任务调度事实源；`teaching`/`evidence` 不得私开隐藏线程。
- `playback` 只消费已发布的 Teaching/Media 版本，不读 AgentScope 内部 Session。
- `gateway` 只返回结构化 Provider Result；Provider **不得直接写业务事实**。
- 跨模块只经公开 Application Command / Domain Event / Port；**不得直接访问对方 Repository**。

### 2.1 ArchUnit 必须捕获的禁止项（缺一即 G7 不通过）

```java
// 包级无环
slices().matching("com.nixiang.tutor.(*)..").should().beFreeOfCycles();

// 1. gateway 不得依赖 teaching（Provider 不能写教学事实）
noClasses().that().resideInAPackage("..gateway..")
    .should().dependOnClassesThat().resideInAPackage("..teaching..");

// 2. playback 不得访问 task 的 Repository
noClasses().that().resideInAPackage("..playback..")
    .should().accessClassesThat().resideInAPackage("..task.repository..");

// 3. teaching 不得直接依赖 Controller
noClasses().that().resideInAPackage("..teaching..")
    .should().dependOnClassesThat().resideInAPackage("..api..");

// 4. memory 不得依赖 teaching
noClasses().that().resideInAPackage("..memory..")
    .should().dependOnClassesThat().resideInAPackage("..teaching..");

// 5. tool 不得依赖 governance（工具登记不判授权，避免循环）
noClasses().that().resideInAPackage("..tool..")
    .should().dependOnClassesThat().resideInAPackage("..governance..");

// 6. domain 不得依赖框架
noClasses().that().resideInAPackage("..domain..")
    .should().dependOnClassesThat()
    .resideInAnyPackage("org.springframework..", "jakarta.persistence..", "io.agentscope..");
```

---

## 3. 不可违背的 12 条不变量

来源：`dev/HANDOFF-20261017.md` §5（逐字保留，本仓内**不得弱化**）。

1. **受控自主**：确定性流程掌握 Task / Evidence Gate / 权限 / 预算 / 发布权；Tutor Harness 只提出候选，不得直接发布教学事实。
2. **六类事实账本分离**：Source/Evidence、Teaching/Media、Task/Playback、Outcome/Mastery、Memory、Eval/Release 互不混淆。
3. **Agent 不生成任意 JavaScript**：教学画面由声明式 `TeachingAction` 渲染（版本化、可重放、可安全校验）。
4. **证据不足必须局部暂停，不得默认通过**：受影响依赖闭包暂停并说明缺口。
5. **额度耗尽只排队/暂停，绝不自动付费**（P-016）。
6. **首轮资料边界**：只接受公开或用户明确授权第三方处理的资料；公开 ≠ 可任意再分发/无限留存。
7. **播放完成 ≠ 掌握**：掌握需可观察证据，且只作建议、不强制解锁。
8. **Release 变更不能仅靠自动分晋级**：须 replay / shadow / canary + 显式准入（ADR-F06）。
9. **Memory 只存策略批准的偏好与经历**：不得吸收未核验知识。
10. **未实测不得对外宣称 SLO**：无测量证据一律标 `[待测]`。
11. **对象级授权**：读取路径必须有可测的跨作用域拒绝，不得只靠"记得加 where"。
12. **合规责任主体存在**：发布 Gate 含 `content_marking` 轴。

---

## 4. 安全基线（硬红线）

- **密钥永不入库**。任何 `.env*`、`*.pem/*.key/*.jks`、连接串口令、Provider token 走本地未入库文件或环境变量；提交前必须自查。
- **日志脱敏**：不得打印完整凭证、Cookie、用户原文资料内容、Provider 请求体。
- **对象级授权必须可测**：每条读取路径要有跨作用域拒绝用例；只写 `@PreAuthorize` 不算完成。
- **SQL 只允许参数化**：MyBatis `#{}`，禁止 `${}` 拼接用户输入。
- **CSP / CSRF / Cookie 属性**按 `dev/20-契约/契约-安全基线.md` §2 取值，不得自定。
- 依赖漏洞与 SBOM 门禁见 `契约-安全基线.md` §5。

---

## 5. 反面模式清单（踩过，务必避开）

来源：`dev/HANDOFF-20261017.md` §6。

| 反面模式 | 为什么危险 |
|---|---|
| 用「口径差异」解释**可确定性判定**的差异 | 凡可机器判定的差异都不是口径差异，是在掩盖规则不够强 |
| 把修复写进勘误表却未施加到正文 | 「已记录 ≠ 已修复」 |
| 加了 `CHECK` 却没写违反时的**错误码** | 实现者无法区分拒绝原因 |
| 引用指向**空行**或**非空但错误的行** | 必须带内容锚，不能只写行号 |
| 用「应/宜/建议」替代「必须 + 拒绝行为」 | L1 冒充 L3 |
| 把**设计意图写成已实现** | `dev/` 零代码；本仓代码才叫已实现 |
| 用**固定行号**做硬断言 | 设计文档行数持续增长，行号必然漂移 |
| 未实测就写 SLO / 性能数字 | 只能标 `[待测]` |

---

## 6. 开发流程（强制）

### 6.1 每个切片的固定动作

```
grill-requirements（需求六维：目标/范围/验收/失败模式/优先级/非目标）
  → doublecheck_spec（落盘 spec）
    → red-green-tdd（先写会失败的测试并真实跑红）
      → 实现到绿（真实跑绿，不只编译通过）
        → ponytail 收尾（YAGNI / stdlib 优先 / 删掉没人要求的抽象）
          → delivery-review（对抗性自审：假设交付会失败，找最强反对意见）
            → 上报，等待门禁
```

- **先红后绿**：红必须是**真断言失败**，不是编译失败或异常崩溃。
- **ponytail 是收尾必做项**，结论写进交付说明；发现未被要求的抽象要当场删。
- 交付判据是「**可被验证的行为成立**」，不是「代码写完 / 能启动 / 编译通过」。

### 6.2 用户硬性约束（三句话）

1. **不要重复造轮子**：能用 Spring 内置 / JDK 标准库 / 成熟库 / 已有模块就用。不自写连接池、重试、序列化、迁移引擎、ID 生成、并发原语。自研前必须举证「找不到能用的」。
2. **不要以实现为目的**：以「可被验证的行为成立」为完成。
3. **实在做不了就停下来上报**（captain → 人类 grill）。**不得造假实现、不得静默降级、不得把未验证写成已验证。**

### 6.3 命令（可复跑）

```bash
# 编译 + 单测 + 架构测试
mvn -B verify

# 只跑架构测试
mvn -B test -Dtest='*ArchitectureTest,*ArchTest'

# 变异测试（PIT，JUnit5 + JVM 21）
mvn -B org.pitest:pitest-maven:mutationCoverage
```

> 首次执行上述命令前，`pom.xml` 尚未落地（见 §8）。命令写在这里作为**约定的验收口径**，不代表已跑过。

---

## 7. 门禁 G0–G9

| 门 | 名称 | 守门人 | 本仓要交什么 |
|---|---|---|---|
| **G0** | 需求门 | product-scope | 切片范围与可机器判定的验收标准 |
| **G1** | 架构安全门 | arch-contract | 分层落点、依赖方向、威胁模型 |
| **G2** | 契约门 | arch-contract | OpenAPI / WS Schema 与实现一致 |
| **G3** | 测试就绪门 | test-lead | 测试方案、需求级变异体、对抗性自审清单 |
| **G4** | 代码 Review 门 | code-reviewer | 可评审的 diff + 复跑命令 + 真实输出 |
| **G5** | 集成门 | integration-qa | 真实集成/E2E 证据，不许 mock 冒充集成 |
| **G6** | 专项门 | integration-qa | 性能/安全/兼容性专项 |
| **G7** | 环境 / CI 门 | platform-release | 编译、单测、架构测试、契约漂移、敏感信息扫描、依赖漏洞、迁移 `has_down` |
| **G8** | 发布门 | platform-release | 证据包 + CHANGELOG + 回滚预案 + PR 材料 |
| **G9** | 生产验证门 | platform-release | 部署后真实验证；未实测不得宣称 SLO |

**G7 六项 CI 门禁**（`dev/20-契约/契约-部署与容量.md` §5）——全部在 CI 阻断，缺一不得合入 `main`：
编译与单测 · 架构测试 · 契约漂移 · 敏感信息扫描 · 依赖漏洞/SBOM · 迁移 `has_down` 校验。

---

## 8. 分支模型（四级）

```
main      ← 只接受来自 release 的合入；受保护，仅人类审批后合并
release   ← 只接受来自 dev 的合入；仅人类审批后合并
dev       ← 集成分支；feat/* 的 PR 目标
feat/*    ← 从 dev 切出，如 feat/task-lease、feat/evidence-gate
```

硬规则：

- `feat/*` **必须**从 `dev` 切出，PR 回 `dev`。
- `dev → release → main` **仅人类审批后**执行；Agent 不得自行合并。
- 禁止直推 `main`、`release`、`dev`。
- 分支命名：`feat/<小写短横线主题>`；修复用 `fix/<主题>`（同样从 `dev` 切）。

---

## 9. 提交与 PR 纪律（人类硬性要求）

- **禁止频繁 commit。** 粒度为「**功能点实现切片**」——一个切片完整（含测试、文档、复跑证据）后才提交一次。
- **commit 前必须先向人类报审**，人工通过后才执行 `git commit`。**不得先斩后奏。**
- **PR 前必须先向人类报审** PR 材料（标题/范围/证据/风险/回滚），人工通过后才开 PR。
- 提交信息：`<type>(<scope>): <中文摘要>`，type ∈ `feat|fix|refactor|test|docs|build|ci|chore`，scope 用模块包名（如 `task`、`evidence`）。

---

## 10. 目录约定（当前状态）

```
Aha/backend/
├─ AGENTS.md          ← 本文件
├─ .gitignore
├─ .gitattributes
└─ （源码目录尚未落地：pom.xml / src/main/java/... 由后端切片 1 建立）
```

> 本仓当前只有仓库基础设施。**没有任何业务实现**；任何「已完成」的表述都必须先有代码与测试证据。

---

## 11. 引用纪律

- 引用设计结论必须带 `文件路径` + `内容锚`；**只写行号不算引用**（行号必然漂移）。
- 无法判定的写「信息不足」，不要猜。
- 发现 `dev/` 与本仓实现冲突：**停下来上报**，不得自行改契约或自行放宽约束。
- **不往本仓写设计文档**。`dev/` 是唯一事实源，本仓只写代码与其说明；本文件是例外（它是「怎么干活」而不是「设计成什么样」）。
