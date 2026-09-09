# 熊翔负责模块 Implementation Plan

> **For agentic workers:** 按任务顺序实施，并使用复选框维护进度。每个任务先完成可重复测试，再进行下一项。

**Goal:** 在五周内完成稳定契约、可运行的确定性排程、受控 Agent 动作生成、证据化用户模型和集成评审。

**Architecture:** 使用 Java 模块化单体。Agent 读取 `ScheduleSnapshot` 并生成 `ActionSpec`，排程器产生候选方案，`ActionExecutor` 负责最终校验和事务写入。外部模型、日历和文件服务均通过端口隔离。

**Tech Stack:** Java 21、Spring Boot 3.5.x、Spring AI 1.1.x、Spring Data JPA、MySQL 8、Flyway、JUnit 5、Vue 3 和 TypeScript。

## Global Constraints

- 初版周期为五周，每周建议投入 15 至 18 小时。
- 时间槽粒度固定为 30 分钟，默认时区为 `Asia/Shanghai`。
- Agent 不直接写数据库，所有日程变更经过 `ActionExecutor`。
- 模型失败时必须可以使用 Mock 和固定输入继续演示。
- 已交付契约的字段不得改名或改变含义；新增字段必须可选并递增 `schemaVersion`。
- 初版不引入微服务、消息队列、Redis、向量数据库或后台任务平台。

---

## 当前进度

更新时间：2026-09-09

| 项目 | 状态 | 结果 |
| --- | --- | --- |
| 阅读技术文档并识别熊翔职责 | 已完成 | 已确认契约、Agent、排程、用户模型、架构评审和核心测试 |
| 明确正式交接对象 | 已完成 | 向黄健华交付动作和校验能力；接收全员集成材料 |
| 技术栈讨论 | 已完成 | 决定继续使用 Java，选择适合期末项目的最小技术组合 |
| 项目代码和构建文件 | 未开始 | 当前工作区尚无实现代码 |
| 领域契约 | 未开始 | 等待创建 Java 类型和 JSON 样例 |
| 排程 Agent 和用户模型 | 未开始 | 等待契约冻结后开发 |

## 文件规划

| 位置 | 责任 |
| --- | --- |
| `backend/src/main/java/com/studyplanner/contract` | 四个稳定契约及公共枚举 |
| `backend/src/main/java/com/studyplanner/planning` | 排程算法、约束校验和缺口报告 |
| `backend/src/main/java/com/studyplanner/agent` | 模型适配、工具注册和 `ActionSpec` 生成 |
| `backend/src/main/java/com/studyplanner/profile` | 用户事实、临时例外和耗时修正 |
| `backend/src/test/java/com/studyplanner` | 核心单元测试和模块集成测试 |
| `samples/contracts` | 供赵欢、黄健华和樊华钰联调的 JSON 样例 |

## Task 1 冻结跨模块契约

**预计工时：** 8 至 10 小时

**Files:**

- Create: `backend/src/main/java/com/studyplanner/contract/ImportProposal.java`
- Create: `backend/src/main/java/com/studyplanner/contract/ScheduleSnapshot.java`
- Create: `backend/src/main/java/com/studyplanner/contract/ActionSpec.java`
- Create: `backend/src/main/java/com/studyplanner/contract/ProfileFact.java`
- Create: `backend/src/main/java/com/studyplanner/contract/CourseSnapshot.java`
- Create: `backend/src/main/java/com/studyplanner/contract/TaskSnapshot.java`
- Create: `backend/src/main/java/com/studyplanner/contract/TimeBlock.java`
- Create: `backend/src/main/java/com/studyplanner/contract/ActionType.java`
- Create: `backend/src/main/java/com/studyplanner/contract/ActionStatus.java`
- Create: `backend/src/main/java/com/studyplanner/contract/ImportStatus.java`
- Create: `backend/src/test/java/com/studyplanner/contract/ContractSerializationTest.java`
- Create: `samples/contracts/*.json`

**Produces:** 其他成员可以独立编译使用的 Java 类型、字段说明和正常及异常 JSON 样例。

- [ ] 用 Java `record` 定义四个契约，保留文档规定的核心字段。
- [ ] 统一 `schemaVersion`、日期时间、时区、状态和错误码表示。
- [ ] 编写 Jackson 序列化往返测试，确认样例 JSON 不丢字段。
- [ ] 增加缺少必填字段、未知枚举和过期版本样例。
- [ ] 将 `ImportProposal` 样例交给赵欢，将 `ActionSpec` 与 `ScheduleSnapshot` 样例交给黄健华，将状态样例交给樊华钰。

**验收：** `mvnw.cmd test -Dtest=ContractSerializationTest` 通过，四个样例能序列化并反序列化为等价对象。

## Task 2 实现硬约束校验器

**预计工时：** 8 至 10 小时

**Files:**

- Create: `backend/src/main/java/com/studyplanner/planning/HardConstraintValidator.java`
- Create: `backend/src/main/java/com/studyplanner/planning/CandidateBlock.java`
- Create: `backend/src/main/java/com/studyplanner/planning/ConstraintViolation.java`
- Create: `backend/src/test/java/com/studyplanner/planning/HardConstraintValidatorTest.java`

**Consumes:** `ScheduleSnapshot` 和候选时间块。

**Produces:** 排程器和黄健华的 `ActionExecutor` 共用的 `validate(snapshot, candidate)` 接口。

- [ ] 先写截止时间、时间重叠、锁定块、禁止时段和每日上限的失败测试。
- [ ] 实现 `List<ConstraintViolation> validate(ScheduleSnapshot snapshot, CandidateBlock candidate)`。
- [ ] 为每个违反项提供稳定错误码、涉及对象和人类可读说明。
- [ ] 添加跨午夜、截止时间相等和空闲区间边界测试。

**验收：** 相同输入始终返回相同顺序的违反项；规则测试全部通过。

## Task 3 实现确定性排程最小版本

**预计工时：** 12 至 16 小时

**Files:**

- Create: `backend/src/main/java/com/studyplanner/planning/GreedyScheduler.java`
- Create: `backend/src/main/java/com/studyplanner/planning/ScheduleCandidate.java`
- Create: `backend/src/main/java/com/studyplanner/planning/PlanningGap.java`
- Create: `backend/src/test/java/com/studyplanner/planning/GreedySchedulerTest.java`

**Consumes:** `ScheduleSnapshot`、`HardConstraintValidator` 和 `Clock`。

**Produces:** 候选时间块、未排入任务及原因；不写数据库。

- [ ] 用固定输入编写失败测试，覆盖截止升序、优先级降序和 ID 升序。
- [ ] 展开课程和允许时段，减去固定占用、已开始块和锁定块。
- [ ] 按 30 分钟粒度从可行区间中选择最早时间槽。
- [ ] 对无法完整排入的任务返回 `PlanningGap`，禁止悄悄截短任务。
- [ ] 增加 100 个任务、14 天窗口、重复运行 30 次的确定性测试并记录耗时。

**验收：** 固定输入可生成可展示的日程；没有硬约束冲突；30 次结果完全一致。

## Task 4 实现 Agent 动作生成

**预计工时：** 10 至 14 小时

**Files:**

- Create: `backend/src/main/java/com/studyplanner/agent/ModelAdapter.java`
- Create: `backend/src/main/java/com/studyplanner/agent/MockModelAdapter.java`
- Create: `backend/src/main/java/com/studyplanner/agent/AgentService.java`
- Create: `backend/src/main/java/com/studyplanner/agent/AgentResult.java`
- Create: `backend/src/main/java/com/studyplanner/agent/AgentToolRegistry.java`
- Create: `backend/src/test/java/com/studyplanner/agent/AgentServiceTest.java`

**Consumes:** 赵欢交付的模型适配能力、`ScheduleSnapshot`、排程器和查询工具。

**Produces:** `NEEDS_CLARIFICATION`、`NEEDS_CONFIRMATION`、`APPLIED` 或 `FAILED` 结果以及结构化 `ActionSpec`。

- [ ] 使用 `MockModelAdapter` 编写创建、移动、取消和信息不足场景测试。
- [ ] 第一版只注册查询日程、生成排程、创建任务、移动任务和取消任务五类工具。
- [ ] 限制一次请求最多进行一次规划和一次工具选择，避免复杂循环。
- [ ] 对影响其他任务或多对象的动作返回预览和待确认状态。
- [ ] 将模型输出转换为 `ActionSpec`，禁止 Agent 调用 Repository。

**验收：** 断网并未配置模型密钥时，Mock 演示链路仍能完整运行；错误输出不会进入数据库。

## Task 5 实现证据化用户模型

**预计工时：** 10 至 12 小时

**Files:**

- Create: `backend/src/main/java/com/studyplanner/profile/ProfileService.java`
- Create: `backend/src/main/java/com/studyplanner/profile/DurationEstimator.java`
- Create: `backend/src/main/java/com/studyplanner/profile/ProfileFactPort.java`
- Create: `backend/src/test/java/com/studyplanner/profile/DurationEstimatorTest.java`
- Create: `backend/src/test/java/com/studyplanner/profile/ProfileServiceTest.java`

**Consumes:** `ProfileFact`、任务完成反馈和 `Clock`。

**Produces:** 有来源、范围、状态、证据和版本的有效事实，以及排程可读取的耗时倍率。

- [ ] 实现明确偏好、指定日期临时例外和事实删除。
- [ ] 三个有效样本之前使用原始估计；达到三个样本后使用平滑倍率。
- [ ] 将倍率限制在 0.5 至 2.0，并保存样本数和来源。
- [ ] 用户手动修改剩余量后，本轮不再二次应用倍率。
- [ ] 删除事实后重算统计，并使引用旧 `profileVersion` 的动作草案失效。

**验收：** 用户可以查看、修改、否认和删除事实；任何偏好都能说明来源和样本数。

## Task 6 完成交接与集成评审

**预计工时：** 12 至 16 小时

**Files:**

- Create: `backend/src/test/java/com/studyplanner/integration/SchedulingActionFlowTest.java`
- Create: `samples/demo/fixed-schedule-demo.json`
- Modify: `planning.md`

**Consumes:** 黄健华的事务与持久化、赵欢的导入和模型适配、樊华钰的前端演示链路。

**Produces:** 可重复答辩演示、集成测试记录和模块评审结论。

- [ ] 向黄健华交付 `ActionSpec`、硬约束校验器、撤销条件、版本冲突样例和错误码。
- [ ] 与黄健华验证过期 `baseRevision` 返回 409，重复 `operationKey` 不重复写入。
- [ ] 与赵欢验证 `ImportProposal` 到任务候选和 Agent 上下文的转换。
- [ ] 与樊华钰验证待补充、待确认、已执行和失败状态映射。
- [ ] 接收全员源码、测试、说明和可复现样例，完成架构边界检查。
- [ ] 运行完整演示：固定输入、排程、动作预览、确认写入、耗时反馈、撤销。

**验收：** 断网情况下可使用固定样例完成演示；使用 MySQL 时事务、版本冲突和撤销测试通过。

## 五周安排

| 周次 | 熊翔主要任务 | 周末退出条件 |
| --- | --- | --- |
| 第 1 周 | 完成 Task 1，启动 Task 2 和固定输入排程样例 | 契约与 JSON 样例冻结，其他成员可以并行开发 |
| 第 2 周 | 完成 Task 2 和 Task 3 | 手动录入的数据可生成无硬冲突日程 |
| 第 3 周 | 完成 Task 4 | 两种以上指令可生成并预览 `ActionSpec` |
| 第 4 周 | 完成 Task 5，开始端到端联调 | 用户事实和耗时修正生效，失败路径可恢复 |
| 第 5 周 | 完成 Task 6，修复、文档和答辩排练 | 核心测试通过，项目可从固定样例复现启动 |

## 交接对象

| 方向 | 对象 | 内容 | 完成标准 |
| --- | --- | --- | --- |
| 熊翔交出 | 黄健华 | `ActionSpec`、硬约束校验器、撤销规则、版本冲突样例 | 接入事务并共同通过冲突和回滚测试 |
| 熊翔对齐 | 赵欢 | `ImportProposal`、`ModelAdapter` 和错误响应 | 样例输入可以转为 Agent 可读对象 |
| 熊翔对齐 | 樊华钰 | 状态枚举、动作预览、回执和用户事实 | 前端状态与后端返回一一对应 |
| 熊翔接收 | 全体成员 | 源码、测试、说明、配置和可复现样例 | 完成集成评审并记录问题 |

## 下一步计划

1. 创建 `backend` Maven 工程并固定 Java 21、Spring Boot 3.5.x 和包名 `com.studyplanner`。
2. 先完成 Task 1 的四个契约、枚举、序列化测试和 JSON 样例。
3. 在契约冻结当天把样例分别发给赵欢、黄健华和樊华钰，随后开始硬约束校验器。
4. 每完成一个任务立即更新本文件的复选框和“当前进度”表。
