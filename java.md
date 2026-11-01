# Java 设计模式落地指南（Spring Boot + MyBatis-Plus + JDK 17）

前置要求（强制）：**先 `use_skill` 加载《java开发规范》原文**（技能名 `java-action-rules`，文件 `~/.codebuddy/skills/qjf-chen-java-rules/SKILL.md`，共十六章），通读后再对照 `java-standards-summary.md`（与原文十六章编号一一对应的速查）自检。

优先级：**用户指令 > 《java开发规范》原文 > 本文骨架**。本文所有骨架已按该规范编写，凡与原文冲突一律以原文为准（原文未覆盖的细节才参考本文）。

## 一、先考虑的语言原生特性（模式之前）

| 需求 | 原生方案 |
| --- | --- |
| 多算法切换 | `enum` + 函数式接口，或 `Map<Enum, Handler>` |
| 不可变数据载体 | `record` |
| 受限的类层次 | `sealed interface` + `record`/`final class` |
| 可选值 | `Optional`（不要用作字段/参数） |
| 集合转换 | Stream API |
| 异步编排 | `CompletableFuture` / 虚拟线程 |
| 资源释放 | try-with-resources |

## 二、各模式 Java 惯用实现

### 策略 + 工厂 + 枚举注册表（最常用）

用于"按类型走不同算法"：支付方式、导出格式、校验规则、消息渠道。

```java
public enum PayChannelEnum {
    ALIPAY, WECHAT, UNION_PAY
}

public interface PayStrategy {
    PayChannelEnum channel();
    PayResult pay(PayCommand command);
}

@Service
public class AlipayStrategy implements PayStrategy {
    @Override
    public PayChannelEnum channel() {
        return PayChannelEnum.ALIPAY;
    }

    @Override
    public PayResult pay(PayCommand command) {
        // 业务实现
        return PayResult.success();
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class PayStrategyRegistry {

    private final List<PayStrategy> strategies;

    private Map<PayChannelEnum, PayStrategy> strategyMap;

    @PostConstruct
    public void init() {
        strategyMap = strategies.stream()
                .collect(Collectors.toMap(PayStrategy::channel, Function.identity()));
    }

    public PayStrategy get(PayChannelEnum channel) {
        PayStrategy strategy = strategyMap.get(channel);
        if (strategy == null) {
            throw new IllegalArgumentException("未支持的支付渠道：" + channel);
        }
        return strategy;
    }
}
```

要点：新增实现只需加类并声明为 Bean，注册表零改动（开闭原则）；禁止用 `switch` 分发。

### 模板方法

固定流程 + 可变步骤：导入导出、审核流程、报表生成。

```java
public abstract class AbstractImportTemplate<T> {

    public final ImportResult execute(ImportContext context) {
        List<T> rows = read(context);
        validate(rows, context);
        saveBatch(rows, context);
        return buildResult(context);
    }

    protected abstract List<T> read(ImportContext context);

    protected abstract void validate(List<T> rows, ImportContext context);

    protected abstract void saveBatch(List<T> rows, ImportContext context);

    protected ImportResult buildResult(ImportContext context) {
        return ImportResult.success(context.getTotal());
    }
}
```

要点：流程方法 `final`；可变步骤 `protected abstract`；批处理必须走批量接口，禁止 `for` 循环逐条。

### 责任链

审批流、风控规则、参数校验链。

```java
public interface RiskHandler {
    boolean handle(RiskContext context);
}
```

```java
@Service
@RequiredArgsConstructor
public class RiskChain {

    private final List<RiskHandler> handlers;

    public boolean execute(RiskContext context) {
        for (RiskHandler handler : handlers) {
            if (!handler.handle(context)) {
                return false;
            }
        }
        return true;
    }
}
```

要点：顺序用 `@Order` 或 `Ordered` 控制；处理器保持无状态；链过长改为配置化管道。

### 装饰器 / 代理（AOP）

横切能力优先用 Spring AOP，而非手写装饰器类。

```java
@Aspect
@Component
@RequiredArgsConstructor
public class AuditLogAspect {

    @Around("@annotation(auditLog)")
    public Object around(ProceedingJoinPoint joinPoint, AuditLog auditLog) throws Throwable {
        // 前置：记录操作人、入参
        try {
            Object result = joinPoint.proceed();
            // 后置：写审计日志（批量落库或异步）
            return result;
        } catch (Throwable throwable) {
            // 异常：记录失败原因
            throw throwable;
        }
    }
}
```

要点：日志一律走 AOP 注解，禁止在 ServiceImpl 里到处打印；"并发创建/更新/删除"用 Redis 原子锁（可在切面或 Service 内）。

### 观察者（事件）

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderCreated(OrderCreatedEvent event) {
    // 主事务提交后执行：通知、积分、消息
}
```

要点：同步事件在主事务内；跨服务/最终一致用 Outbox 或消息队列，不要靠本地事件硬撑。

### 建造者

```java
ReportQuery query = ReportQuery.builder()
        .startTime(start)
        .endTime(end)
        .status(ReportStatusEnum.FINISHED)
        .build();
```

实体用 Lombok `@Builder`；校验放 `build()`；参数 ≤ 3 个直接用构造器。

### 状态机（状态模式 + 枚举）

```java
public enum OrderStatusEnum {
    DRAFT(Set.of(SUBMITTED)),
    SUBMITTED(Set.of(APPROVED, REJECTED)),
    APPROVED(Set.of()),
    REJECTED(Set.of(DRAFT));

    private final Set<OrderStatusEnum> allowedNext;

    OrderStatusEnum(Set<OrderStatusEnum> allowedNext) {
        this.allowedNext = allowedNext;
    }

    public boolean canTransferTo(OrderStatusEnum target) {
        return allowedNext.contains(target);
    }
}
```

要点：状态/类型/选项一律枚举（规范强制）；禁止字符串或数字常量散落。

### 适配器 / 外观

第三方 SDK 统一包在 `adapter` 或 `client` 包下，对外暴露本地接口；Facade 只做编排，不写业务规则。

### 单例

直接交给 Spring 容器（`@Service`/`@Component` 默认单例），禁止手写双重检查锁与静态单例。

## 三、模式落地的规范硬约束（摘抄，完整以技能为准）

| 模式环节 | 强制约束 |
| --- | --- |
| 策略/状态的分支键 | 必须用枚举类，禁止字符串/数字常量 |
| 实现类注入 | `@RequiredArgsConstructor` + `private final XxxService xxxService`，禁止 `@Autowired` / `@Resource` 字段注入 |
| 跨表数据访问 | 调用对应 Service 接口；ServiceImpl 内禁止注入他人 Mapper，自身缺失能力用 `baseMapper` |
| 批处理 | 模式内的保存/更新/删除/查询必须批量接口，禁止 `for`/`while` 逐条 |
| 出参 | 返回 VO，禁止直接返回 Entity；DTO/VO 定义在 api 模块 |
| 实体转换 | `BeanUtils.copyProperties()`，禁止逐字段 `set` |
| 并发写 | Redis 分布式锁（带过期与归属校验） |
| 日志 | AOP 注解，禁止业务代码零散打日志 |
| 时间字段 | `LocalDateTime`/`Date` 加 `@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")`；`LocalDate` 用 `yyyy-MM-dd` |
| 唯一性 | 有编码/名称字段时创建与更新必须做唯一校验（名称不重复 + 编码不重复） |
| Controller | `@PostMapping`/`@GetMapping` 等写全路径，含 `/private` 前缀与模块名与动作（`create`/`update`/`delete`/`get`）；参数超两个字段必须用继承 `BaseDTO` 的 DTO，禁止 Map/HttpRequest 接参 |
| 分页查询 | 不用 id 作初始条件，优先编码/名称/状态/类型/创建时间区间 |
| 事务 | ServiceImpl 非查询方法必须 `@Transactional`，按需写 `rollbackFor` / `noRollbackFor` |
| 基本类型 | `long`/`int`/`short`/`byte`/`boolean` 一律用包装类 |
| 跨服务与第三方 | 跨微服务接口与参数定义在 api 模块；第三方接口独立建包；微服务项目禁用 `@RequestBody`，参数用 `@Validated` + `@Valid` |
| 批处理 | 禁止 `for`/`while` 逐条；Excel 导入 >1000 条、导出 >100 条必须批处理并给出定时器/多线程两方案建议 |
| Mapper.xml | SQL 必须写注释（功能/参数/返回值）；复杂查询须提示索引失效点与优化方案 |
| 公共逻辑 | 超过两处复用的方法/算法抽到工具类或 `service.impl` 下的 IOC 类 |
| 交付前 | 语法与格式化检查；执行 `mvn -pl <module> compile` 真编译验证 |

## 四、Java 模式常见错误

- 策略分发仍用 `switch`/`if-else`：改为枚举注册表。
- 策略实现里直接注入 Mapper：违反分层，改走 Service 接口。
- 模板方法子类用 `for` 循环逐条入库：改批量接口。
- 单例手写 `getInstance()` + 静态字段：改 Spring Bean。
- 装饰器层层 `new`：改 AOP 或责任链。
- 状态用 `int`/`String` 常量：改枚举状态机。
- 事件监听里做长耗时同步操作：改异步 + 事务提交后事件。
