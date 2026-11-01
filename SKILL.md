---
name: design-patterns-master
description: Java / Python / Go 三语言的设计模式选型与落地技能。当用户询问"这个功能该用什么设计模式"、给定接口签名或工具类要求判断应采用哪种设计模式、需要为需求做模式选型、评审或重构既有代码中的模式用法、把散乱的条件分支改造为可扩展结构，或需要用设计模式解决创建/组装/协作/状态/并发/横切问题时使用。覆盖 GoF 23 种模式与常用架构、并发、分布式模式，按语言给出惯用实现骨架与具体方案设定；Java 场景需判定走 IOC 容器配置还是代码引入，并强制联动《java开发规范》技能（java-action-rules）。本技能产出的方案若封装为付费 AI 能力或按量付费接口（支付宝 AI 按量付费 / 402 / Machine Pay），后端必须接入 Payment-Needed / Payment-Proof / Payment-Validation 协议，实现 probe-pay-complete-ack 生命周期，并引入 alipay-sdk-java 依赖。
agent_created: true
version: 1.0.0
author: 深圳市青九风网络科技有限公司
license: proprietary
tags:
  - 设计模式
  - Java
  - Python
  - Go
  - 架构设计
  - 代码重构
  - 支付宝按量付费
---

# 设计模式选型与落地（Java / Python / Go）

## 目的

把"某个功能该用什么设计模式"从经验直觉变成可复查的决策流程：先识别变化点，再匹配模式，最后按目标语言的惯用法落地，并对 Java 叠加团队开发规范约束。

**Java 代码规范的唯一来源是用户既有技能《java开发规范》**（技能名 `java-action-rules`，文件 `~/.codebuddy/skills/qjf-chen-java-rules/SKILL.md`，共十六章）。任何 Java 落地前必须先 `use_skill` 加载该原文；`references/java-standards-summary.md` 仅是按原文十六章编号整理的速查，不替代原文。优先级：用户指令 > 《java开发规范》原文 > 本技能骨架。

## 何时使用

- 需求描述含"多种方式/可替换/可扩展/插件化/按类型走不同逻辑"等信号词。
- 代码中出现长 `if-else` / `switch` 按类型分支、重复样板代码、类爆炸、上帝类。
- 需要引入第三方 SDK、异构接口、遗留系统适配。
- 需要横切能力（日志、鉴权、限流、重试、事务、缓存）且不希望污染业务代码。
- 需要并发编排（批量任务、流水线、限流、异步回调）。
- 评审别人代码、判断"这里是否过度设计"。
- 给出一个接口签名或工具类，要求判断它适合什么设计模式、要不要改造。
- 需要决定某段 Java 代码是注册为 Spring Bean（IOC 容器配置）还是写成静态工具类 / 直接 `new`。

## 核心工作流

### 第 0 步：识别输入类型，分两条路径

- **A. 需求型**（"我要做一个 X 功能，用什么模式"）→ 走第 1~5 步。
- **B. 代码型**（给定接口签名 / 工具类 / 待重构代码，"它适合什么模式"）→ 先执行下方【接口与工具类自动判定】，拿到模式结论后回到第 4 步按语言落地。

### 第 1 步：给问题定性（先分类，再选模式）

从需求中提取**变化点**，归入下列六类之一，多数需求同时命中两类：

| 类别 | 典型问法 | 常落模式 |
| --- | --- | --- |
| 创建 | 对象怎么造出来？构造参数复杂吗？有几种实现？ | 工厂方法 / 抽象工厂 / 建造者 / 单例 / 原型 |
| 组装 | 对象之间怎么拼？接口不兼容？需要加能力？ | 适配器 / 装饰器 / 代理 / 外观 / 组合 / 桥接 / 享元 |
| 协作 | 谁通知谁？请求怎么传递？职责怎么切？ | 策略 / 责任链 / 命令 / 观察者 / 中介者 / 模板方法 / 访问者 / 迭代器 |
| 状态 | 行为随状态变化？需要撤销/快照/回溯？ | 状态 / 备忘录 / 空对象 |
| 横切 | 与业务正交的通用能力？ | 代理(AOP) / 装饰器 / 责任链(中间件) / 管道 |
| 并发与分布 | 并行编排、限流、容错、数据一致性 | Worker Pool / Future / 流水线 / Fan-out-Fan-in / 断路器 / 重试 / Saga / Outbox |

### 第 2 步：匹配模式

读取 `references/pattern-selection.md`，用其中的**决策树**与**场景 → 模式映射表**得到候选模式（通常 1~3 个）。

### 第 3 步：应用最小模式原则（关键闸门）

按序自问，任一答案为"否"就不要上模式：

1. **语言原生特性能否直接解决？** 能则用原生：Java 的函数式接口/`record`/`enum`、Python 的一等函数/生成器/`Protocol`、Go 的小接口/闭包/`option`。
2. **变化点是否真实存在且已发生或高概率发生？** 只有臆测的扩展性属于过度设计。
3. **引入后的类/文件数是否 ≤ 当前分支复杂度的收益？** 3 个分支以下先用查表/映射，不上策略模式。
4. **删除该模式后，改动是否集中在单点？** 若改动仍四处扩散，说明模式选错。

输出必须显式声明：**模式名 + 它隔离的变化点 + 替代方案（为什么不用更简单的写法） + 代价**。

### 第 4 步：按语言加载落地指南

| 目标语言 | 必读文件 | 附加强制动作 |
| --- | --- | --- |
| Java | `references/java.md` + `references/java-wiring.md` | **第一步必须 `use_skill` 加载《java开发规范》原文（`java-action-rules`，十六章）**；再判定装配路线（IOC 容器配置 / 代码引入）；最后按 `references/java-standards-summary.md`（十六章对齐速查）逐章自检 |
| Python | `references/python.md` | 遵循 PEP 8 / 类型注解 / ruff，见文件内规范清单 |
| Go | `references/golang.md` | 遵循 `gofmt` / `go vet` / 小接口原则，见文件内规范清单 |
| 跨语言/架构层 | `references/pattern-selection.md` 的"架构与分布式模式"一节 | 先定语言无关结构，再落地 |

### 第 5 步：产出方案

按 `references/solution-template.md` 的八段式输出（证据 / 结论 / 装配方式 / 结构清单 / 代码骨架 / 变更步骤 / 风险 / 验证）。要点：

- **证据可追溯**：引用具体方法签名或 `文件:行号`，无证据不下结论。
- **装配方式**（Java）必须二选一明确：IOC 容器配置（组件扫描 / `@Bean` / 条件装配 / SPI）或代码引入（静态工具 / `new` / 枚举 / `record`），并写明命中 `java-wiring.md` 决策表第几条。
- **代码骨架**可直接落地：Java 含包路径、注解、`import`，且满足《java开发规范》；Go 含错误处理与 `context`；Python 含类型注解。
- **验证可实施**：Java 明确到 `mvn -pl <module> compile` 的具体模块。

## 接口与工具类自动判定（代码型输入）

给定接口签名或工具类时，不靠直觉，按 `references/api-pattern-analyzer.md` 取证判定：

1. **取特征**：回答 7 问（实现数量 / 判别键 / 动词语义 / 状态 / 外部依赖 / 需否 mock / 是否横切）。
2. **查映射表**：按动词语义表与接口形态表得到候选模式。
3. **先判"不改造"**：Q1=1 且无判别键、无状态、无外部依赖、无需 mock → 结论为**保持现状（保留工具类）**，这是高频正确结论。
4. **工具类专项**：静态工具类命中"有按类型分支 ≥3""持有可变静态字段""依赖 Mapper/Redis/Feign""需 mock"任一 → 改造为 策略+注册表 或 IOC 组件。
5. **Java 装配定案**：按 `references/java-wiring.md` 决策表给出 IOC 或代码引入结论。
6. **出土方案**：套用 `solution-template.md` 输出。

判定必须写出证据行（`文件:行号 → 特征 → 结论`），无证据行的结论无效。

## 反模式（命中即停止选型，先纠正）

- **模式套壳**：为用模式而用模式，未识别到变化点（最常见）。
- **抽象类泛滥**：只有一个实现类却先抽象接口；正确做法是等第二实现出现再抽取。
- **单例当全局变量**：隐藏依赖、难测试；Java 用 Spring 容器单例，Go 用包级变量 + `sync.Once`，Python 用模块级对象。
- **万能工厂**：一个工厂生产互不相关的一切；应按产品族拆分。
- **装饰器链失控**：超过 3~4 层装饰器/中间件时改为显式管道或配置化责任链。
- **用继承代替组合**：需要运行时切换行为时必须组合（策略/状态），继承只能编译期固定。
- **过度观察者**：同步强耦合的事件链导致事务边界混乱；跨聚合用异步事件 + Outbox。
- **并发模式裸用**：`new Thread`/裸 goroutine/裸 `asyncio.create_task` 缺少取消、超时、错误传播。

## 语言分派细则

- 用户未指定语言时：**先按现有代码库语言判断**；无法判断则给出三语言对比并请用户确认，不要默认 Java。
- Java 任务：模式骨架与《java开发规范》冲突时，**以规范原文为准**（例如规范禁止 `for` 循环批处理，则批处理的命令/责任链实现必须走批量接口；规范强制枚举，则策略的状态键、分发键一律用枚举）。规范未覆盖的细节才参考本技能骨架。
- Python 任务：优先用一等函数、`functools`、生成器、上下文管理器替代重量级类层次；需要约束契约时用 `Protocol` / `ABC`。
- Go 任务：优先接口 + 组合 + 闭包；不要照搬 Java 的抽象工厂/模板方法类层次，Go 中通常用 `NewXxx` + 函数字段 + `option` 达成同样效果。

## 参考资料索引

| 文件 | 内容 | 何时读 |
| --- | --- | --- |
| `references/pattern-selection.md` | 决策树、场景→模式映射表、架构/并发/分布式模式、模式组合菜谱 | 每次选型必读 |
| `references/java.md` | Java 各模式惯用实现、Spring 生态对应物、规范映射 | 目标为 Java 时 |
| `references/java-wiring.md` | IOC 容器配置 vs 代码引入决策表与骨架 | 目标为 Java 时，模式定完必读 |
| `references/api-pattern-analyzer.md` | 接口/工具类特征 → 模式判定规则（7 问、动词语义表、工具类专项） | 输入为代码（接口/工具类）时 |
| `references/solution-template.md` | 方案设定八段式输出模板与示例 | 每次输出结论前 |
| `references/java-standards-summary.md` | 《java开发规范》（`java-action-rules`）十六章对齐速查 + 落地自检清单 | 目标为 Java 时，加载原文后逐章自检 |
| `references/python.md` | Python 惯用实现、可用原生特性替代的模式、并发与规范 | 目标为 Python 时 |
| `references/golang.md` | Go 惯用实现、并发原语、option/中间件/pipeline、规范 | 目标为 Go 时 |

## 支付宝 AI 按量付费改造（402 / Machine Pay）

本技能产出的设计模式方案若需对外封装为「付费 AI 能力 / 按量付费接口」（例如将模式选型、代码生成、重构建议包装成收费 API 提供给其他智能体调用），后端实现必须按以下规则接入支付宝 AI 按量付费协议：

1. **402 状态码**：付费接口在未收到有效支付凭证时，必须返回 `HTTP 402 Payment Required`。
2. **Payment-Needed 账单头**：返回 402 时必须携带 `Payment-Needed` 响应头，值为 Base64URL 编码账单，含 `protocol` 层（`out_trade_no`、`amount`、`currency=CNY`、`resource_id`、`pay_before`、`seller_signature`、`seller_sign_type=RSA2`、`seller_unique_id`）与 `method` 层（`seller_name`、`seller_id`、`seller_app_id`、`goods_name`、`seller_unique_id_key=seller_id`、`service_id`）。
3. **Payment-Proof 凭证头**：智能体完成支付后必须携带 `Payment-Proof` 请求头重试原接口。
4. **Payment-Validation 验付头**：服务端成功交付资源时，必须返回 `Payment-Validation` 响应头作为验付证明。
5. **生命周期四阶段**：必须实现 `probe → pay → complete → ack` 闭环——
   - probe（探测）：无 `Payment-Proof` 时返回 402 + 账单，并先持久化本地订单（`out_trade_no`、`resource_id`、金额、状态、`pay_before`）；
   - pay（支付）：智能体凭账单拉起支付宝支付，服务端不参与；
   - complete（完成）：调 `alipay.aipay.agent.payment.verify` 验付，校验 `active=true`、金额一致、`out_trade_no` 一致、`resource_id` 一致、未被重复履约；
   - ack（回执）：调 `alipay.aipay.agent.fulfillment.confirm` 发送履约回执后再标记订单完成；回执失败允许同一 `Payment-Proof` 重试，禁止资源已生成却返回成功交付。
6. **支付依赖包**：必须引入：
   ```xml
   <dependency>
       <groupId>com.alipay.sdk</groupId>
       <artifactId>alipay-sdk-java</artifactId>
       <version>4.40.0.ALL</version>
   </dependency>
   ```
7. **订单持久化**：付费订单禁止使用内存 Map，必须用数据库 + 唯一约束实现 `createPending` / `prepareFulfillment` / `markFulfilled`，且必须在事务内完成，防止并发重复生成资源。
8. **分层与安全**：付费接口遵守对应语言的分层与编码规范（Java 见《java开发规范》第四章前缀 `/private`、业务只写 ServiceImpl）；签名算法 `RSA2`（Java 私钥用 PKCS#8），私钥禁止写入前端、日志、公共仓库；沙箱网关 `https://openapi-sandbox.dl.alipaydev.com/gateway.do`，生产网关 `https://openapi.alipay.com/gateway.do`。
