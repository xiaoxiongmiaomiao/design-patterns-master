# 方案设定输出模板（强制格式）

任何"接口/工具类用什么模式"的结论，必须按本模板输出。缺项视为方案不完整。**结论可简，但八段标题不可省**；无内容时写"无"并说明原因。

## 输出模板

```markdown
## 1. 输入与证据
- 输入：<接口签名 / 工具类方法列表 / 文件:行号>
- 特征：Q1 实现数=<n>；Q2 判别键=<有/无，字段名>；Q3 动词=<parse/send/...>；
        Q4 状态=<有/无>；Q5 外部依赖=<有/无>；Q6 需 mock=<是/否>；Q7 横切=<是/否>
- 证据行：<文件:行号> → <特征> → <结论>

## 2. 结论
- 模式：<模式名，可组合，如 策略 + 工厂/注册表>
- 隔离的变化点：<一句话>
- 否定的备选：<更简单的方案 + 为什么不用>
- 代价：<新增类数量 / 理解成本 / 性能影响>

## 3. 装配方式（Java 专属，其他语言写"不适用"）
- 路线：<IOC 容器配置（组件扫描 / @Bean / 条件装配 / SPI）| 代码引入（静态工具 / new / 枚举 / record）>
- 依据：命中 java-wiring.md 决策表第 <n> 条
- 注入：@RequiredArgsConstructor + private final（如为 IOC）
- 顺序/条件：@Order(n) / @ConditionalOnProperty(name=..., havingValue=..., matchIfMissing=...)
- 生命周期：destroyMethod / 线程池关闭 / 锁释放

## 4. 结构清单
| 角色 | 类型名 | 包/路径 | 装配 | 说明 |
| --- | --- | --- | --- | --- |

## 5. 代码骨架
<目标语言可编译的最小骨架，含必要 import 与注释；Java 需满足 java-standards-summary.md>

## 6. 变更步骤（增量可回滚）
1. 新增 <X>（不影响既有调用）
2. 切换调用方 <A> 到新入口
3. 删除旧分支代码
4. 回滚方式：<说明>

## 7. 风险与约束
- <并发/事务/幂等/兼容性/枚举取值/规范冲突点>

## 8. 验证清单
- [ ] 单元测试覆盖每个实现/分支
- [ ] 新增实现是否零改动既有类（开闭原则）
- [ ] Java：mvn -pl <module> compile 通过
- [ ] 规范自检（逐项对照 java-standards-summary.md 十六章）：枚举替代常量、`@RequiredArgsConstructor` 构造器注入、DTO 继承 BaseDTO、基本类型用包装类、`@Transactional`、批量操作（Excel 导入>1000/导出>100）、VO 出参、`@JsonFormat`、AOP 日志、Redis 锁、Controller 全路径含 `/private`
```

## 填表规则

1. **第 1 段必须有可追溯证据**，引用具体方法签名或 `文件:行号`；无法取证则先读代码再下结论。
2. **第 2 段必须给出被否定的备选**，否则无法判断是否过度设计。
3. **第 3 段仅 Java 需要**，且必须二选一给出明确路线，不允许"都可以"。
4. **第 5 段骨架要能直接落地**：Java 含包结构、注解、`import`；Go 含包名与错误处理；Python 含类型注解。
5. **第 6 步骤必须可回滚**：禁止"一次性大改"，优先"新增 → 切流 → 删除旧代码"。
6. **第 8 段必须可验证**：Java 明确到 `mvn -pl <module> compile` 的具体模块名。

## 示例（Java 支付渠道工具类 → 策略）

```markdown
## 1. 输入与证据
- 输入：PayUtils.pay(String payType, PayRequest req)，内部 switch 三个分支
- 特征：Q1=3 实现；Q2 判别键=payType；Q3 动词=pay；Q4 无状态；Q5 依赖订单 Mapper；
        Q6 需 mock（单测难写）；Q7 否
- 证据：PayUtils.java:23 switch(payType) → Q2 命中 → 策略
- 证据：PayUtils.java:41 调用 OrderMapper → Q5 命中 → 静态工具类不成立，必须 IOC

## 2. 结论
- 模式：策略 + 工厂/注册表（枚举 key）
- 隔离的变化点：支付渠道算法与渠道增减
- 否定的备选：枚举持有行为——因实现需注入 OrderService，枚举无法被容器注入依赖
- 代价：新增 1 接口 + 3 实现类 + 1 注册表

## 3. 装配方式
- 路线：IOC 容器配置（组件扫描 + List 注入注册表）
- 依据：命中 java-wiring.md 第 1、5 条
- 注入：@RequiredArgsConstructor + private final
- 顺序/条件：无需（按枚举直接定位，非链式）
- 生命周期：无

## 4. 结构清单
| 角色 | 类型名 | 包 | 装配 | 说明 |
| --- | --- | --- | --- | --- |
| 策略接口 | PayStrategy | service.strategy | — | 声明 channel()/pay() |
| 实现 | AlipayStrategy 等 | service.strategy.impl | @Service | 每个渠道一个类 |
| 枚举键 | PayChannelEnum | api.enums | — | 渠道取值集合 |
| 注册表 | PayStrategyRegistry | service.strategy | @Service | 注入 List<PayStrategy> |
```

（后续第 5~8 段按实际代码与模块补全。）
