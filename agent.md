# 学习日程 Agent 项目约定

本文档规定项目的技术栈、编码规范和架构边界。目标是在五周内完成可运行、可测试、可答辩演示的初版。实现应优先选择团队容易理解和调试的方案，不为初版增加微服务、消息队列、向量数据库或独立工作流平台。

## 技术栈

### 后端

- Java 21。
- Spring Boot 3.5.x，使用 Spring MVC 构建 REST API。
- Maven Wrapper 管理构建，所有成员使用 `./mvnw` 或 `mvnw.cmd`。
- Spring Validation 校验请求参数。
- Spring Data JPA 访问数据库。
- MySQL 8 保存正式数据，Flyway 管理数据库迁移。
- H2 的 MySQL 兼容模式只用于快速单元测试；演示前必须在 MySQL 上完成一次集成测试。
- Spring AI 1.1.x 封装模型调用、结构化输出和工具调用。只接入赵欢已验证的一家模型服务。
- `springdoc-openapi` 生成 Swagger 页面，作为前后端联调接口说明。

### 前端

- Vue 3、TypeScript、Vite。
- Vue Router 管理页面路由。
- Element Plus 提供表单、表格、弹窗和反馈组件。
- FullCalendar 展示日历和时间块；初版只实现项目文档要求的交互。
- Axios 调用后端 API。
- 状态较少时使用组合式函数；只有跨页面状态明显增多时才引入 Pinia。

### 测试和开发工具

- JUnit 5、Mockito、Spring Boot Test。
- 核心排程测试注入 JDK 的 `java.time.Clock`，并使用 `Clock.fixed(...)` 和固定输入，禁止依赖真实模型。
- 前端使用 `vue-tsc` 做类型检查；关键工具函数可使用 Vitest。
- Git 分支名使用 `feature/<模块>-<说明>`，提交信息使用 `feat:`、`fix:`、`test:`、`docs:`、`refactor:`。

## 项目架构

项目采用模块化单体。所有后端模块在同一个 Spring Boot 应用中运行，通过 Java 接口和稳定契约协作。数据库写入统一经过应用服务和事务入口。

```text
frontend
  -> REST API
      -> importing   输入接收 识别 校对 ImportProposal
      -> agent       读取上下文 选择工具 生成 ActionSpec
      -> planning    排程 硬约束校验 缺口报告
      -> profile     偏好 临时例外 耗时修正 ProfileFact
      -> business    课程 任务 时间块 ScheduleSnapshot
      -> persistence Repository 事务 Flyway
          -> MySQL
```

推荐目录结构：

```text
backend/
  pom.xml
  src/main/java/com/studyplanner/
    common/          时间 错误码 统一响应
    contract/        跨模块 DTO 枚举和接口
    importing/       多模态导入流程
    agent/           模型适配 工具注册 动作生成
    planning/        排程器和硬约束校验
    profile/         用户事实和耗时统计
    business/        账户 课程 任务 时间块
    persistence/     JPA 实体 Repository 和端口实现
  src/main/resources/
    application.yml
    db/migration/
  src/test/java/com/studyplanner/
frontend/
  src/
    api/
    components/
    composables/
    pages/
    router/
    types/
samples/
  contracts/         联调用 JSON 样例
```

## 稳定契约

熊翔负责维护以下四个跨模块契约。契约一旦交给其他成员使用，已有字段不改名、不改变含义；新增字段必须可选，并递增 `schemaVersion`。

- `ImportProposal`：`kind`、`title`、`courseCandidate`、`deadline`、`durationMinutes`、`sourceEvidence`、`ambiguities`。
- `ScheduleSnapshot`：`now`、`timezone`、`courses`、`tasks`、`lockedBlocks`、`activeFacts`、`planningRevision`。
- `ActionSpec`：`operationKey`、`baseRevision`、`type`、`targets`、`changes`、`authorizationScope`、`reasonFacts`。
- `ProfileFact`：`key`、`value`、`scope`、`strength`、`source`、`state`、`evidenceIds`、`version`。

跨模块端口至少包括 `ModelAdapter`、`CalendarPort`、`NotificationPort` 和 `FileStore`。时间统一注入 JDK 的 `java.time.Clock`。初版必须提供 `MockModelAdapter` 和 `InternalCalendarAdapter`，保证没有外部服务时仍可演示。

## 编码规范

### Java 代码

- 包名全小写，类名使用 PascalCase，方法和变量使用 camelCase，常量使用 UPPER_SNAKE_CASE。
- 每个类只承担一个主要职责。超过约 300 行时优先拆分，但不为了数字进行无意义拆分。
- DTO 优先使用 Java `record`；JPA 实体使用普通类。控制器不得直接返回 JPA 实体。
- 控制器只负责参数校验、调用应用服务和转换响应；业务规则放在领域服务或应用服务中。
- Agent 只能生成 `ActionSpec`，不得直接访问 Repository、拼接 SQL 或宣称写入成功。
- 正式日程写入统一进入 `ActionExecutor`。短事务内校验 `baseRevision`、硬约束和权限，然后提交并递增版本。
- 排程器只生成候选结果；`HardConstraintValidator` 是排程与执行共用的唯一硬约束实现。
- 不捕获后忽略异常。可预期错误转换为稳定错误码，未知异常记录关联 ID 并返回通用错误。

### 时间和状态

- 日期使用 `LocalDate`，带时刻的数据使用 `Instant`；用户时区使用 IANA `ZoneId` 字符串，例如 `Asia/Shanghai`。
- 核心逻辑不得直接调用 `Instant.now()`；必须注入 `Clock`。
- 初版时间槽粒度固定为 30 分钟。
- 状态使用枚举，禁止在业务代码中散落字符串。导入状态至少包括 `UPLOADED`、`PROCESSING`、`REVIEW`、`COMMITTED`、`FAILED`、`EXPIRED`。
- `operationKey` 和导入 `requestKey` 必须支持幂等；重复请求返回原结果，不重复写入。

### 数据库

- 每次表结构变化都增加新的 Flyway 迁移文件，已提交的迁移文件不得修改。
- 表名和字段名使用 snake_case，主键统一使用 `id`，业务表必须明确用户归属。
- 核心业务字段使用明确列；JSON 只保存可扩展的候选信息、证据和快照。
- 外部模型调用必须在数据库事务之外完成。
- 机密只保存在环境变量中。日志不得记录 API Key、原始录音或完整个人日程。

### API 和前端

- API 路径使用 `/api/v1/...`，返回统一的成功数据或错误结构。
- 前端类型名称与后端契约一致；联调以 `samples/contracts` 中的 JSON 为准。
- 页面必须区分处理中、待补充、待确认、已执行和失败，不能把处理中显示为成功。
- 批量移动、取消或影响其他任务的动作必须显示预览并由用户确认。

## 测试和完成标准

- 排程规则至少覆盖截止、重叠、锁定时间块、每日上限、确定性排序和耗时修正。
- Agent 测试使用 Mock 模型，验证生成的 `ActionSpec` 以及失败和澄清路径。
- 每个模块必须包含正常样例、异常样例、可重复运行测试、配置说明和一份可复现输入。
- 模块只有在真实数据库变更可验证、失败可恢复且同伴完成评审后才视为完成。

## 熊翔的架构责任

熊翔负责领域契约、Agent 工具、排程算法、用户模型、架构评审和核心测试。他向黄健华交付 `ActionSpec`、硬约束校验器、撤销规则和联调样例；全体成员向熊翔提交源码、测试、说明和可复现样例，由熊翔完成最终集成评审。
