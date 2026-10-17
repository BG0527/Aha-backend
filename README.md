# Aha — 可视化 AI 私人教师（后端）

资料驱动的个性化学习系统的后端仓。单 Spring Boot 部署单元，
以**证据门禁**决定哪些内容可以作为确定知识发布。

> **状态：仅仓库基础设施，无业务实现。**
> 本仓当前只有 `AGENTS.md`、`.gitignore`、`.gitattributes` 三个文件。
> 尚无 `pom.xml`、`src/`、任何服务、迁移或测试。后端切片 1 才建立脚手架。

---

## 技术基线

| 项 | 取值 |
|---|---|
| 语言 | **Java 21（LTS）**，不降级也不升级 |
| 框架 | Spring Boot 3.x，**模块化单体**（不是微服务） |
| 构建 | Maven |
| 分层 | Package-by-Feature + `api / application / domain / port / adapter` |
| 数据库 | PostgreSQL + Flyway |
| 持久化 | JPA 写聚合 + MyBatis 查询 |
| 认证 | Spring Security + HttpOnly Cookie 服务端会话 |
| 接口 | OpenAPI spec-first 生成 |
| 包根 | `com.nixiang.tutor` |

首版**不拆 Maven 多模块**，用 package-by-feature + ArchUnit 约束边界。

### 版本锚（本机实测）

| 工具 | 实测版本 |
|---|---|
| JDK | `openjdk 21.0.1` |
| Maven | `3.9.12` |

---

## 命令（尚未可用）

以下命令是**约定口径**，`pom.xml` 尚未落地，**从未执行过**：

```bash
mvn -B verify
mvn -B test
```

---

## 核心不变量（违反即拒绝合入）

1. **受控自主** —— 模型只提候选，任务状态、证据门禁与发布权由确定性流程掌握。
2. **证据不足必须局部暂停** —— 冲突或不可回放时暂停受影响闭包，不得默认通过。
3. **额度耗尽只排队或暂停，绝不自动付费。**
4. **未实测不得对外宣称任何性能或可用性承诺** —— 无测量证据一律标 `[待测]`。

完整 12 条见 `AGENTS.md`。

---

## 分支模型

```
main      生产，仅 human 审批后合入
release   发布候选，仅 human 审批后合入
dev       集成分支，PR 目标
feat/*    从 dev 切出
```

`main` / `release` / `dev` 均为受保护分支，禁止直推。

---

## 先读什么

- **`AGENTS.md`** —— 本仓唯一作业说明：16 个模块包的依赖矩阵、ArchUnit 禁止项、
  安全红线、门禁 G0–G9、提交纪律。

设计文档**不在本仓**。契约与 ADR 记录在项目外部的 `dev/` 目录（本仓只引用，不复制）。

---

## 许可证

尚未选定。
