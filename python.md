# Python 设计模式落地指南

Python 的一等函数、鸭子类型、装饰器、生成器、上下文管理器已经覆盖了 GoF 中相当一部分模式。**先判断能否用原生特性一行解决，再考虑类层次。**

## 一、可原生替代的模式

| GoF 模式 | Python 原生替代 |
| --- | --- |
| 策略 | 一等函数 / 字典映射函数 |
| 命令 | 闭包 / `functools.partial` |
| 模板方法 | 高阶函数（传入回调步骤） |
| 迭代器 | 生成器（`yield`） |
| 访问者 | `functools.singledispatch` / `match` 语句 |
| 装饰器 | `@decorator` 语法（原生） |
| 外观 | 模块级函数集合 |
| 单例 | 模块级对象（模块天然单例） |
| 空对象 | `None` 判断 + 默认返回值，或专门的 `NullObject` 类 |
| 备忘录 | `dataclasses.replace` / `copy.deepcopy` |
| 享元 | `__slots__` + 不可变对象缓存（`functools.lru_cache`） |

## 二、仍需显式实现的模式

### 策略（优先函数，其次 Protocol）

```python
from typing import Protocol


class Notifier(Protocol):
    def send(self, user_id: str, content: str) -> None: ...


def send_sms(user_id: str, content: str) -> None:
    ...


NOTIFIERS: dict[str, Notifier] = {"sms": send_sms}


def notify(channel: str, user_id: str, content: str) -> None:
    NOTIFIERS[channel].send(user_id, content)
```

3 个以内分支直接用字典映射函数；需要多方法协同时才定义 `Protocol` 或 `ABC`。

### 工厂

```python
def create_parser(kind: str) -> Parser:
    match kind:
        case "json":
            return JsonParser()
        case "csv":
            return CsvParser()
        case _:
            raise ValueError(f"unsupported kind: {kind}")
```

### 建造者（参数多时优先 dataclass）

```python
from dataclasses import dataclass, field


@dataclass(slots=True)
class ReportQuery:
    start: str
    end: str
    status: str = "ALL"
    tags: list[str] = field(default_factory=list)
```

链式建造者仅在不变量校验复杂时才引入。

### 装饰器（原生语法 + `functools.wraps`）

```python
import functools
import time


def timed(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            print(f"{func.__name__} cost {time.perf_counter() - start:.3f}s")

    return wrapper
```

必须加 `functools.wraps`；装饰器层数控制在 3 层以内；需要参数时用两层闭包。

### 适配器 / 外观

```python
class PaymentAdapter:
    """将第三方 SDK 接口适配为本地 PaymentPort 协议。"""

    def __init__(self, client: ThirdPartyClient) -> None:
        self._client = client

    def pay(self, order_no: str, amount: int) -> PayResult:
        raw = self._client.create_trade(order_no, amount)
        return PayResult(ok=raw["success"], trade_id=raw["trade_id"])
```

### 观察者 / 事件

同步场景用回调列表；异步场景用 `asyncio.Queue`。跨进程一律消息队列，不要用内存事件假装解耦。

### 状态

轻量：`enum.Enum` + 转移表（转移合法性校验）。
重量：状态对象实现同一 `Protocol`，上下文持有当前状态对象。

```python
ALLOWED: dict[OrderStatus, set[OrderStatus]] = {
    OrderStatus.DRAFT: {OrderStatus.SUBMITTED},
    OrderStatus.SUBMITTED: {OrderStatus.APPROVED, OrderStatus.REJECTED},
}
```

### 责任链 / 中间件

```python
def build_chain(handlers: list[Handler]) -> Handler:
    def run(ctx: Context) -> bool:
        return all(handler(ctx) for handler in handlers)

    return run
```

## 三、并发模式

| 场景 | 方案 |
| --- | --- |
| IO 密集并发 | `asyncio` + `asyncio.gather` |
| 并发度限制 | `asyncio.Semaphore` |
| CPU 密集 | `multiprocessing` / `concurrent.futures.ProcessPoolExecutor` |
| 批任务编排 | `ThreadPoolExecutor.map` 或 `asyncio.TaskGroup` |
| 超时与取消 | `asyncio.timeout` / `asyncio.wait_for` |
| 共享状态 | 优先"单线程封闭 + 消息传递"，其次 `asyncio.Lock` |

要点：GIL 决定 CPU 密集不能靠线程；`asyncio` 中禁止阻塞调用（阻塞需 `to_thread`）；任务必须处理取消与异常，禁止裸 `create_task` 后不 `await`。

## 四、Python 编码规范要点

- 遵循 PEP 8；格式化与静态检查用 `ruff format` + `ruff check`；类型检查用 `mypy`/`pyright`。
- 公开函数必须有类型注解与 docstring（参数、返回值、异常）。
- 优先 `dataclass`（`@dataclass(slots=True)`）承载数据，`frozen=True` 表达不可变。
- 常量用模块级大写；固定取值集合用 `enum.Enum` / `StrEnum`。
- 异常：定义业务异常基类，禁止裸 `except:`；`except` 必须指定异常类型。
- 资源用 `with`；自定义资源用 `@contextmanager`。
- 禁止可变默认参数（`def f(x=[])`）；默认用 `None` 或 `field(default_factory=...)`。
- 禁止 `from xxx import *`；导入顺序：标准库 → 第三方 → 本地。
- 项目结构：`src/` 布局，按领域分包，测试放 `tests/` 用 `pytest`。
- 配置与密钥走环境变量/配置中心，禁止硬编码。
- 交付前执行：`ruff format --check .`、`ruff check .`、`pytest`。

## 五、Python 模式常见错误

- 用 Java 式抽象工厂/多层继承套 Python：改为函数 + `Protocol`。
- 装饰器忘记 `functools.wraps`：导致元信息丢失、文档测试失败。
- 单例用元类或 `__new__` 黑魔法：改模块级对象或依赖注入。
- 在 `asyncio` 中调用阻塞 IO：改 `asyncio.to_thread` 或换同步栈。
- 状态用裸字符串常量：改 `enum`。
- 生成器当列表反复遍历：生成器只能消费一次，需复用则显式转列表。
