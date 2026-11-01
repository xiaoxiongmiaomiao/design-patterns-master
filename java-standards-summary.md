# 《java开发规范》章节对齐速查

**规范来源**：用户既有技能《java开发规范》（技能名 `java-action-rules`，目录 `~/.codebuddy/skills/qjf-chen-java-rules/SKILL.md`）。

**使用方式**：
1. 写 Java 代码前**必须先 `use_skill` 加载《java开发规范》原文**并通读。
2. 本文件按原文**十六章编号一一对应**整理，仅作落地速查，不替代原文。
3. **冲突时以原文为准**；原文与用户指令冲突时以用户指令为准。

---

## 一、行为准则（通用）

1. 开口先问候「你好，AI 指挥者」。
2. 行动前先思考；写代码前先读现有文件，明确当前实现与约束，避免盲改。
3. 输出简洁但推理充分：关键假设、关键结论必须说明。
4. 优先编辑现有代码，不重写整文件，只改必要范围。
5. 不重复阅读已确认的文件内容。
6. 变更后先测试/验证，再宣称完成，并给出实际跑过的验证方式。
7. 不使用奉承式开场白或多余结束语，直达结论。
8. 方案尽量简单直接，先给最小可行且稳妥的改法。
9. 用户指令始终优先于本规则。
10. 删除用户文件前必须征得同意（确认对话框）。

## 二、包引用与 JDK 兼容

1. 引用的类路径必须真实存在，不存在则修正为存在的路径。
2. JDK 17+ 禁止 `javax.*`，改用 `jakarta.*`。
3. JDK 17+ 禁止 `javax.validation.constraints.NotNull`，改用 `jakarta.validation.constraints.NotNull`。
4. 禁用长引用包名（如 `@org.springframework.validation.annotation.Validated`），必须 `import`；禁止 `import *`（不同包同名类才可例外）。
5. 禁止中文乱码，出现即修。

## 三、注解与依赖注入

1. 注入容器中的 Service/ServiceImpl/Mapper，必须用 `@RequiredArgsConstructor` + `private final XxxService xxxService`。
2. 禁止 `@Autowired`（除非用户特别要求）。
3. 遇到 `@Resource` 必须替换为 `@RequiredArgsConstructor` + `private final`。
4. Service 必须继承 `IService`；ServiceImpl 必须继承 `ServiceImpl`（MyBatis-Plus）。

## 四、分层规范

1. Controller 禁止 `@RequestMapping`；`@PostMapping`/`@GetMapping`/`@DeleteMapping`/`@PutMapping` 必须写全路径且前缀加 `/private`。
2. Controller 调用其他表查询接口时，必须走对应 Service 接口，禁止直接调 Mapper。
3. Controller 接收参数超过两个字段时**必须用 DTO 接收，DTO 必须继承 BaseDTO**，字段必须写注释（含义 + 取值范围）；禁止用 Map 或 HttpRequest 接收（除非有特定契约）。
4. 所有业务代码必须写在 ServiceImpl，禁止出现在 Mapper、Controller。
5. ServiceImpl 调用其他表查询接口时，必须走对应 Service 接口，禁止直接调 Mapper。
6. ServiceImpl 优先使用自身方法（`save`/`update`/`remove`/`removeBatchByIds`/`getById`/`list` 等）；自身没有时才用 `baseMapper`，并在 mapper.xml 实现 SQL。
7. 禁止在 ServiceImpl 内实现内部实体类或工具代码；工具代码必须抽到 Utils 类。
8. ServiceImpl 不允许创建实体包裹在内；新实体必须在对应 api 模块实现。
9. Service 继承 `IService`，ServiceImpl 继承 `ServiceImpl`；禁止注入 Mapper 相关类（除 `ServiceImpl<>` 外），必须注入 Service 接口。

## 五、DTO / VO / Entity 规范

1. DTO、VO、Entity 字段必须写注释，注释写字段含义，含义写取值范围。
2. 返回前端禁止直接用 Entity，必须转 VO。
3. 两个实体间赋值、字段类型相近时，必须用 `BeanUtils.copyProperties()`。
4. `LocalDateTime`、`Date` 必须加 `@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")`；`LocalDate` 用 `yyyy-MM-dd`。
5. `long`/`int`/`short`/`byte`/`boolean` 等基本类型**必须使用包装类**，禁止基本类型（除非用户特别要求）。

## 六、枚举类规范

1. 状态、类型、选项等固定取值字段，必须显式定义并使用枚举类；禁止字符串或数字常量散落；判断、转换、传参、返回值均基于枚举。

## 七、事务规范

1. ServiceImpl 中操作方法（非查询）**必须写 `@Transactional`**；需回滚写 `rollbackFor`，不需回滚写 `noRollbackFor`，采用全局事务方式。

## 八、批处理规范

1. 禁止用 `for`/`while`/`do-while` 对数据库表批量处理；多条数据增删改必须批处理。
2. 批量操作、查询必须批处理，禁止循环。
3. 前端 Excel 导入超过 **1000 条**必须批处理，禁止循环。
4. 前端 Excel 导出超过 **100 条**必须批处理，禁止循环。

## 九、Mapper 与 Mapper.xml 规范

1. Mapper.xml 的 SQL 必须写注释：SQL 功能、参数、返回值。
2. Mapper.xml 复杂查询必须告知用户哪里会索引失效、如何解决、索引优化方案。
3. `IService`、`ServiceImpl`、`Controller`、`Mapper` 等类必须写类注释：类功能 + 类中方法含义。

## 十、分页查询规范

1. 分页（特殊列表）查询不允许用 Id 作查询条件（除非特意指定），优先用编码、名称、状态、类型、创建时间区间作为初始化条件。

## 十一、唯一性校验

1. 创建/更新操作若表存在编码、名称字段：第一名称不重复，第二编码不重复（如 `RecordNoUtils` 生成），除非用户特别指定；否则一律按此优先级。

## 十二、Excel 导入导出

1. 导入超过 1000 条必须批处理，禁止循环。
2. 导入条数超过一定行数时，必须给用户两种方案建议：
   - 定时器方案：定时器定期执行导入，完成一份再进行下一份；
   - 多线程方案：多线程多路复用，同时执行多份导入，全部完成再进行下一份。
3. 导出超过 100 条必须批处理，禁止循环。

## 十三、微服务与第三方接口

1. 微服务项目**禁止使用 `@RequestBody`**；必须用 `@Validated` + `@Valid` 校验参数（除非用户特别要求）。
2. 调用其他微服务的接口与参数必须在 **api 模块**实现，禁止在 `service.impl` 实现；`service.impl` 只允许业务逻辑。
3. 调用外部第三方接口，必须单独建立软件包与实体类包并实现对应接口，禁止在 `service.impl` 实现。

## 十四、日志与并发

1. 日志必须采用 AOP 切面 + 注解实现，禁止在 ServiceImpl 到处调用。
2. 并发创建、更新、删除操作，必须优先建立 Redis 原子性操作实现锁。
3. 超过两个地方调用的方法或算法，必须分离到 Java 工具类或可注入 IOC 容器的业务类，并在 `service.impl` 下实现。

## 十五、代码质量与验证

1. 骨架实现：只实现 Entity、Mapper、Mapper.xml、Service、ServiceImpl，其他方法禁止实现。
2. 补字段：必须同步补全 DTO、PageDto、VO、Entity、Mapper、Mapper.xml、Service、ServiceImpl、Controller 等，并补字段注释、类注释、方法注释。
3. 删除表/Entity 时必须严格检查哪些 ServiceImpl 实现了相应逻辑，一并删除。
4. 编码完成后必须检查语法正确性与格式化（如 `@Schema(description = "...", example = "...")` 需换行），遵守阿里巴巴 Java 开发规范。
5. 写完代码执行 `mvn -pl compile` 做真编译验证，报错及时修改。

## 十六、支付宝 AI 按量付费改造（402 / Machine Pay）

需求涉及「支付宝 AI 付 / 按量付费 / 402 收款 / 智能体付费调用 API」时适用：

1. **402 状态码**：未收到有效支付凭证时必须返回 `HTTP 402 Payment Required`。
2. **Payment-Needed 账单头**：402 响应携带，值为 Base64URL 编码账单，含
   - `protocol` 层：`out_trade_no`、`amount`、`currency=CNY`、`resource_id`、`pay_before`、`seller_signature`、`seller_sign_type=RSA2`、`seller_unique_id`；
   - `method` 层：`seller_name`、`seller_id`、`seller_app_id`、`goods_name`、`seller_unique_id_key=seller_id`、`service_id`。
3. **Payment-Proof 凭证头**：智能体完成支付后携带该请求头重试原接口。
4. **Payment-Validation 验付头**：服务端成功交付资源时必须返回该响应头。
5. **生命周期四阶段** `probe → pay → complete → ack`：
   - probe：无 `Payment-Proof` 时返回 402 + 账单，并先持久化本地订单（`out_trade_no`、`resource_id`、金额、状态、`pay_before`）；
   - pay：智能体凭账单拉起支付，服务端不参与；
   - complete：调 `alipay.aipay.agent.payment.verify` 验付，校验 `active=true`、金额一致、`out_trade_no` 一致、`resource_id` 一致、未被重复履约；
   - ack：调 `alipay.aipay.agent.fulfillment.confirm` 发履约回执后再标记订单完成；回执失败允许同一 `Payment-Proof` 重试，禁止资源已生成却返回成功交付。
6. **依赖包**：`com.alipay.sdk:alipay-sdk-java:4.40.0.ALL`。
7. **订单持久化**：禁止内存 Map，必须 MyBatis-Plus + 数据库唯一约束实现 `createPending`/`prepareFulfillment`/`markFulfilled`，且必须在事务内完成（遵守第七、八章）。
8. **分层与安全**：付费接口遵守第四章（`/private` 前缀、业务只写 ServiceImpl）；签名 `RSA2`（Java 私钥 PKCS#8），私钥禁止写入前端、日志、公共仓库；沙箱网关 `https://openapi-sandbox.dl.alipaydev.com/gateway.do`，生产网关 `https://openapi.alipay.com/gateway.do`。

---

## 落地前自检（按章节快速打勾）

- [ ] 一：未盲改；删除文件已征得同意
- [ ] 二：无 `javax.*`；无长引用包名；无中文乱码
- [ ] 三：`@RequiredArgsConstructor` + `private final`；无 `@Autowired`/`@Resource`
- [ ] 四：Controller 全路径含 `/private` 与模块名、动作；DTO 继承 BaseDTO 且字段有注释；业务只在 ServiceImpl；跨表走 Service 接口
- [ ] 五：字段注释含取值范围；出参用 VO；`BeanUtils.copyProperties()`；时间字段 `@JsonFormat`；**基本类型用包装类**
- [ ] 六：固定取值全部枚举
- [ ] 七：非查询方法加 `@Transactional`（含 `rollbackFor`/`noRollbackFor`）
- [ ] 八：无循环批操作；Excel 导入 >1000 / 导出 >100 走批处理
- [ ] 九：Mapper.xml SQL 有注释；复杂查询已提示索引风险与优化
- [ ] 十：分页初始条件不含 Id
- [ ] 十一：名称不重复 + 编码不重复校验
- [ ] 十二：Excel 大批量已给出定时器/多线程两方案建议
- [ ] 十三：无 `@RequestBody`；跨服务接口在 api 模块；第三方接口独立建包
- [ ] 十四：日志走 AOP；并发写有 Redis 锁；复用算法已抽工具类/IOC 类
- [ ] 十五：语法与格式化检查通过；`mvn -pl compile` 通过
- [ ] 十六：涉及 AI 付费时 402 + 四阶段闭环已实现（如适用）
