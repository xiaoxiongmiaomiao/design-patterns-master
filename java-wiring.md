# Java 装配方式决策：IOC 容器配置 vs 代码引入

模式定完后，必须给出**装配结论**。Java 中同一模式有两种装配路线，选错会导致：无法注入依赖、无法 mock、静态状态并发问题、或一堆无意义的 `@Bean` 样板。

本文件为 Java 专属。落地时须先 `use_skill` 加载《java开发规范》原文（技能名 `java-action-rules`，`~/.codebuddy/skills/qjf-chen-java-rules/SKILL.md`），并满足 `java-standards-summary.md`（与原文十六章编号对齐的速查）；冲突以原文为准。装配方式本身受原文第三、四、十三章约束：构造器注入、分层与 api 模块归属、跨服务接口不得落在 `service.impl`。

## 一、决策表（自上而下，命中即停）

| 序 | 条件 | 装配方式 |
| --- | --- | --- |
| 1 | 需要注入其他 Bean（Service / Mapper 能力 / Redis / Feign / 配置属性） | **IOC 组件**：`@Service` / `@Component` |
| 2 | 生命周期需容器管理（连接池、客户端、线程池、需 `close()`） | **IOC 配置类**：`@Configuration` + `@Bean(initMethod/destroyMethod)` |
| 3 | 第三方的类无法加注解（SDK、库对象） | **IOC 配置类**：`@Bean` 方法里 `new` |
| 4 | 需要按配置/环境/租户切换实现 | **IOC 条件装配**：`@ConditionalOnProperty` / `@Profile` / `@ConditionalOnMissingBean` |
| 5 | 需要被 mock、替换、按类型多实现分发（策略/适配器/状态） | **IOC 组件 + 接口**：多实现 `@Service`，注册表注入 `List<T>` 或 `Map<枚举,T>` |
| 6 | 无状态算法实现，但要参与策略/责任链分发 | **IOC 组件**（便于统一收集）；确认无需替换时可代码引入 |
| 7 | 纯函数、零依赖、不需要替换（格式化、加密、校验表达式） | **代码引入**：`final class` + 私有构造器 + `static` |
| 8 | 分支键是枚举且算法无外部依赖 | **代码引入**：枚举持有策略函数/常量；或 IOC 注册（见第五节对比） |
| 9 | 值对象 / 参数对象 / DTO / 命令对象 | **代码引入**：`record` / 构造器 / `@Builder`，不进容器 |
| 10 | 需要 jar 外扩展（插件、多租户自定义实现） | **SPI**：`ServiceLoader` 或 Spring `AutoConfiguration.imports` + `@Conditional` |
| 11 | 每次调用需独立实例（有可变上下文） | **代码引入**：工厂方法 `new`（优先）；确需容器托管用 `@Scope("prototype")`（谨慎） |
| 12 | 循环依赖风险 / 装配关系复杂 | **IOC**：改构造器注入 + 拆分职责；禁止字段注入掩盖问题 |

## 二、IOC 方式骨架

### 2.1 组件扫描（默认首选）

```java
public interface AuditRule {
    RuleEnum type();
    boolean match(AuditContext context);
}

@Service
public class AmountAuditRule implements AuditRule {

    @Override
    public RuleEnum type() {
        return RuleEnum.AMOUNT;
    }

    @Override
    public boolean match(AuditContext context) {
        return context.getAmount().compareTo(BigDecimal.ZERO) > 0;
    }
}
```

注册表靠容器收集实现类：

```java
@Service
@RequiredArgsConstructor
public class AuditRuleRegistry {

    private final List<AuditRule> rules;

    private Map<RuleEnum, AuditRule> ruleMap;

    @PostConstruct
    public void init() {
        ruleMap = rules.stream()
                .collect(Collectors.toMap(AuditRule::type, Function.identity()));
    }

    public AuditRule get(RuleEnum type) {
        AuditRule rule = ruleMap.get(type);
        if (rule == null) {
            throw new IllegalArgumentException("未支持的规则类型：" + type);
        }
        return rule;
    }
}
```

新增实现 = 加一个类 + `@Service`，注册表零改动。**优先组件扫描，不要为每实现类写 `@Bean`。**

### 2.2 配置类 + `@Bean`（第三方对象 / 生命周期）

```java
@Configuration
public class SmsClientConfiguration {

    @Bean(destroyMethod = "close")
    public SmsClient smsClient(SmsProperties properties) {
        return new SmsClient(properties.getEndpoint(), properties.getAccessKey());
    }
}
```

### 2.3 条件装配（按配置/环境切换）

```java
@Configuration
public class StorageConfiguration {

    @Bean
    @ConditionalOnProperty(name = "storage.type", havingValue = "oss")
    public StorageService ossStorageService(OssProperties properties) {
        return new OssStorageService(properties);
    }

    @Bean
    @ConditionalOnProperty(name = "storage.type", havingValue = "local", matchIfMissing = true)
    public StorageService localStorageService(LocalStorageProperties properties) {
        return new LocalStorageService(properties);
    }
}
```

`@ConditionalOnMissingBean` 用于提供默认实现并允许覆写。

### 2.4 顺序控制（责任链/中间件）

实现类加 `@Order(n)` 或实现 `Ordered`；注入 `List<Handler>` 时按序排列。顺序是业务契约时必须显式写 `@Order`，不能依赖扫描顺序。

### 2.5 SPI 插件（jar 外扩展）

```java
public final class RuleLoader {

    private RuleLoader() {
    }

    public static List<Rule> load() {
        List<Rule> rules = new ArrayList<>();
        ServiceLoader.load(Rule.class).forEach(rules::add);
        return rules;
    }
}
```

配合 `META-INF/services/<接口全限定名>` 声明实现类；若需 Spring 托管，在自动配置类中读取 SPI 结果并注册为 Bean。

## 三、代码引入方式骨架

### 3.1 静态工具类（纯函数）

```java
public final class AmountUtils {

    private AmountUtils() {
        throw new UnsupportedOperationException("工具类禁止实例化");
    }

    public static BigDecimal safeAdd(BigDecimal left, BigDecimal right) {
        return Objects.isNull(left) ? right : left.add(Objects.isNull(right) ? BigDecimal.ZERO : right);
    }
}
```

判据：无状态 + 无外部依赖 + 无需替换。三条不全满足就不得用静态。

### 3.2 枚举持有行为（无依赖的轻量策略）

```java
public enum DiscountTypeEnum {

    NONE {
        @Override
        public BigDecimal discount(BigDecimal amount) {
            return amount;
        }
    },
    HALF {
        @Override
        public BigDecimal discount(BigDecimal amount) {
            return amount.multiply(new BigDecimal("0.5"));
        }
    };

    public abstract BigDecimal discount(BigDecimal amount);
}
```

适用：算法无外部依赖、分支稳定、不需要单独测试替身。一旦实现需要注入其他 Bean，立即改为 IOC 策略组件。

### 3.3 显式 `new`（值对象 / 命令 / 参数对象）

```java
PayCommand command = PayCommand.builder()
        .orderNo(orderNo)
        .amount(amount)
        .channel(PayChannelEnum.ALIPAY)
        .build();
```

值对象不进容器；构造函数或 `@Builder` 创建。

## 四、两条路线的对比与迁移信号

| 维度 | IOC 容器配置 | 代码引入 |
| --- | --- | --- |
| 依赖注入 | 支持 | 不支持（只能传参或静态持有） |
| 可测试性 | 可 `@MockBean`/替换 | 静态难以 mock |
| 生命周期 | 容器管理（init/destroy） | 手动管理 |
| 复杂度 | 需注解与装配关系 | 零装配成本 |
| 典型风险 | 循环依赖、扫描不到、Bean 过多 | 静态可变状态、隐式依赖、无法替换 |

迁移信号（出现任一即改路线）：

- 代码引入 → IOC：开始注入依赖、需要 mock、出现静态可变字段、需要按环境切换。
- IOC → 代码引入：实现变成纯函数且无人替换、Bean 只有常量查表能力（改为枚举或常量类）。

## 五、装配结论的必答项

在方案里必须显式写出：

1. **装配路线**：IOC（组件扫描 / `@Bean` / 条件装配 / SPI）还是代码引入（静态工具 / `new` / 枚举 / `record`）。
2. **理由**：命中决策表第几条。
3. **注入方式**：`@RequiredArgsConstructor` + `private final`（禁止 `@Autowired`/`@Resource`）。
4. **新增实现的改动面**：是否需要改注册表/配置类（理想为"零改动"）。
5. **顺序/条件约束**：`@Order`、`@ConditionalOnProperty` 等具体值与含义。
6. **生命周期**：是否需要 `destroyMethod`、线程池关闭、Redis 锁释放。

## 六、装配常见错误

- 为每个策略实现手写 `@Bean`：改组件扫描 + `List` 注入。
- 用 `@Autowired` 字段注入掩盖循环依赖：改构造器注入并拆分职责。
- 工具类里 `static` 持有 `ApplicationContext` 或 Mapper 实例：改 IOC 组件。
- `@Configuration` 里 `new` 的对象又注入 Mapper：违反分层，改 Service 组件。
- 原型对象注入单例导致状态串用：改工厂方法显式 `new`。
- 条件装配无默认值：`matchIfMissing = true` 兜底，避免启动失败。
- 枚举策略实现里注入 Bean（枚举无法被 Spring 管理依赖）：改 IOC 策略组件。
