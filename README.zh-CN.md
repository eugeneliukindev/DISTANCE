# DISTANCE

[English](README.md) | [Русский](README.ru.md) | **中文**

**保持距离 (Keep your distance):** 每一段代码对其他代码知道得越少越好. 这样代码容易理解, 也能安全地修改.

| | 原则 | 要点 | 规则 |
|---|---|---|---|
| [**D**](#d---deep) | Deep (深模块) | 公共 API 比内部细节更简单 | [D1](#d1-公共-api-比内部细节更简单) [D2](#d2-更少的特殊情况) [D3](#d3-只和最近的邻居交谈) [D4](#d4-给每个调用方一个窄接口) |
| [**I**](#i---inward) | Inward (依赖向内) | 业务逻辑不了解基础设施 | [I1](#i1-业务逻辑不依赖基础设施) [I2](#i2-只有当第二个实现真实存在时才引入接口) [I3](#i3-决策与副作用分离) [I4](#i4-包之间没有循环依赖) |
| [**S**](#s---single-owner) | Single owner (唯一所有者) | 每一项知识和每一个不变量都只有一个所有者 | [S1](#s1-每一项知识只有一个所有者) [S2](#s2-只有所有者才能修改不变量) |
| [**T**](#t---translate-at-the-boundary) | Translate at the boundary (在边界处转换) | 外部的数据和错误只在系统边界处转换一次, 变成我们自己的 | [T1](#t1-外部数据在边界处解析为值对象) [T2](#t2-外部错误在适配器中转换为我们自己的错误) |
| [**A**](#a---add-a-kind-not-a-branch) | Add a kind, not a branch (新增种类而不是分支) | 新的行为变体是一个新的实现, 而不是一个新的条件 | [A1](#a1-新的行为变体是一个新的实现) [A2](#a2-每个实现都完整履行契约) [A3](#a3-组合优于继承) |
| [**N**](#n---needed-now) | Needed now (只做现在需要的) | 只做现在需要的东西 | [N1](#n1-只做现在需要的东西) |
| [**C**](#c---contract) | Contract (契约) | 契约被写下来, 并由测试检查 | [C1](#c1-契约写在公共-api-中) [C2](#c2-测试通过公共-api-检查契约) [C3](#c3-没有测试的代码-先提取决策-再用测试固定它) |
| [**E**](#e---evident) | Evident (一目了然) | 读代码时没有意外 | [E1](#e1-名称说明它是什么以及为什么) [E2](#e2-同样的事情用同样的方式做) [E3](#e3-注释只解释-为什么) |

如果某条规则在某个具体位置让代码变得更复杂而不是更简单, 那么在那里就不应用它.

---

## 术语表

| 术语 | 含义 | 示例 |
|---|---|---|
| **接口** (interface) | 只描述方法, 不包含代码的类型. 由类来实现它 | 带有 `get` 和 `save` 方法的 `Orders` |
| **实现** (implementation) | 实现某个接口的类 | `PostgresOrders`, `InMemoryOrders` 实现 `Orders` |
| **公共 API** (public API) | 函数, 类或模块提供给调用方代码的东西: 公共方法, 它们的参数和行为. 任何代码都有, 即使没有接口 | `HttpAcquirer` 的公共 API 是一个方法 `charge` |
| **内部细节** (internals) | 公共 API 背后, 调用方不需要知道的代码 | `PostgresOrders.save` 里面的 SQL |
| **契约** (contract) | 公共 API 的承诺: 接受什么, 保证什么, 抛出哪些错误 | "要么全额扣款, 要么不扣" |
| **调用方** (caller) | 调用函数或类的代码 | `OrderPayment` 是 `PostgresOrders` 的调用方 |
| **业务逻辑** (business logic) | 领域的规则. 由实体和用例组成 | 折扣, 订单状态, 配送时间 |
| **实体** (entity) | 具有标识和不变量的领域对象. 自己检查自己的规则 | `Order`, `Cart` |
| **用例** (use case, 应用服务) | 一个用户操作的业务逻辑: 规定步骤的顺序, 把实体和适配器连接起来 | `OrderPayment`, `Shopping` |
| **基础设施** (infrastructure) | 代码在自身内存之外打交道的一切. 不做业务决策 | Postgres, Redis, 外部 API, 队列, 文件, 邮件, 时钟 |
| **外部 API** (external API) | 我们的代码调用的别人的服务 | 收单服务, 快递服务 |
| **适配器** (adapter) | 我们的代码通过它与外部系统打交道的类. 只有适配器知道该系统如何运作. 适配器把我们的对象转换成它的格式, 把它的响应和错误转换成我们的. 不做业务决策 | `PostgresOrders`, `RedisCarts`, `HttpAcquirer` |
| **处理器** (handler) | 外部世界通过它访问我们的代码: HTTP 端点, CLI 命令, 队列消费者. 解析输入, 调用业务逻辑, 返回响应 | `POST /orders/{id}/pay` |
| **系统边界** (system boundary) | 处理器和适配器的总和: 我们的代码与外部世界相遇的所有地方 | |
| **依赖注入** (dependency injection) | 类通过构造函数接收适配器, 而不是自己创建它们 | `OrderPayment(orders, acquirer)` |
| **入口点** (entry point, `main`) | 程序启动的地方, 在这里创建适配器并传给业务逻辑 | `main.py` |
| **值对象** (value object) | 不可变的数据类型, 在创建时验证自身 | `PostalCode`, `Address`, `Money` |
| **不变量** (invariant) | 对象始终保持的条件 | "只有已支付的订单才能发货" |
| **副作用** (side effect) | 返回结果之外的动作: 写数据库, 网络调用, 发邮件, 修改别人的对象 | `payments.refund(...)` |
| **纯函数** (pure function) | 没有副作用的函数: 结果只取决于参数 | `decide_return(order, lines)` |
| **伪实现** (fake) | 用于测试的简单可用的实现. 不要和 mock 混淆, mock 只检查调用 | `InMemoryOrders` |
| **组合** (composition) | 类持有另一个对象并调用它, 而不是继承它 | `OrderLines` 持有一个字典 |

### 如何阅读示例

这些原则与编程语言无关. 示例用 Python 编写, 不使用框架和语言特有的功能, 让任何语言的程序员都能看懂.

| 示例中 | Java 中 | Go 中 |
|---|---|---|
| `class Orders(Protocol)` | `interface Orders` | `type Orders interface` |
| `@dataclass(frozen=True)` | `record` | 没有 setter 的 `struct` |
| `raise SomeError` | `throw new SomeException()` | `return ..., ErrSome` |
| `None` | `null` / `Optional` | `nil` |
| 方法下面的 `"""..."""` | Javadoc | 函数上方的注释 |
| `raise Ours(error)` | `throw new Ours(error)` | `fmt.Errorf("...: %w", err)` |
| `Decimal` | `BigDecimal` | 没有舍入误差的十进制类型 |

商店的类型 (带有 `plus`, `times`, `is_at_least` 方法的 `Money`, `Order`, `Cart` 等) 被认为已经定义好了.

示例中不使用具体的库和框架, 而是使用占位类型. 每种语言都有对应的东西, 只是名字不同:

| 占位类型 | 能做什么 |
|---|---|
| `HttpRequest`, `HttpResponse` | 传入的请求和响应. `request.path_param(name)` 是路径参数, `request.json_string(name)` 是请求体中的字符串字段或 `None`, `request.json_body()` 是整个请求体组成的字典. `HttpResponse(status, body)` |
| `HttpClient` | 发出的 HTTP 请求. `post(path, body, headers)` 返回带有 `status` 和 `body` 字段 (解析后的 JSON 或 `None`) 的 `HttpResult`. 网络故障或超时 -> `NetworkError` |
| `Database` | Postgres 客户端. `fetch_one(sql, params)` 返回一行 "列 -> 值" 字典或 `None`, `execute(sql, params)` 执行语句 |
| `RedisClient` | Redis 客户端. 方法以 Redis 命令命名: `hgetall`, `hset`, `hincrby`, `expire`, `delete`. `transaction()` 是一组在 `exec()` 时原子执行的命令 (MULTI/EXEC) |
| `Router` | 任意 HTTP 服务器或框架 |
| `Mock`, `expect_error` | 任意 mock 库中的 mock; 检查函数是否抛出预期的异常 |
| `env(name)` | 环境变量 |

简短的示例省略了处理器所用的服务; [完整示例](#完整示例-订单支付) 展示了它们是如何传入的.

---

## D - Deep

### D1. 公共 API 比内部细节更简单

函数, 类或模块把复杂性留给自己: 调用方只做一次简单的调用, 所有工作由内部细节完成. 只有当新的函数或类隐藏了某些东西, 并且它的公共 API 比它所替代的代码更简单时, 才把它提取出来. 仅仅因为代码长, 不是拆分的理由.

```python
# 不好: 调用方知道收单方的协议和步骤顺序
intent_id = acquirer.create_intent(order.total())
acquirer.attach_card(intent_id, card_token)
acquirer.confirm(intent_id)
payment_id = acquirer.capture(intent_id)
```

```python
# 好
class HttpAcquirer:
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        """要么全额扣款, 要么什么都不扣.

        使用相同 idempotency_key 的重复调用返回同一张收据, 不会再次扣款.
        银行拒绝 -> PaymentDeclined. 网络或服务故障 -> PaymentUnavailable.
        """
        ...

receipt = acquirer.charge(order.total(), card_token, idempotency_key=order.id())
```

### D2. 更少的特殊情况

设计契约时, 要让调用方没有需要单独处理的情况. 如果调用的目标已经达成, 那就是成功, 而不是错误. 不要隐藏调用方必须做出反应的情况 (银行拒绝, 缺货).

```python
# 不好: 重复删除 (浏览器里的双击) 是一个每个人都要捕获的错误
class Cart:
    def remove(self, sku: str) -> None:
        if sku not in self._items:
            raise ProductNotInCart(sku)
        del self._items[sku]

try:
    cart.remove(sku)
except ProductNotInCart:
    pass
```

```python
# 好
class Cart:
    def remove(self, sku: str) -> None:
        """调用之后购物车里没有 sku, 不管它之前在不在."""
        if sku in self._items:
            del self._items[sku]

cart.remove(sku)
```

### D3. 只和最近的邻居交谈

代码只调用自身, 自己的参数, 自己的字段以及自己创建的对象上的方法. 它不会沿着调用链穿过别人的内部细节. 例外是值对象和链式 API (例如查询构建器): 它们的结构就是它们的公共 API.

```python
# 不好
def loyalty_discount(order: Order) -> Decimal:
    return order.customer().profile().loyalty().level().discount_rate()
```

```python
# 好
def loyalty_discount(order: Order, loyalty: Loyalty) -> Decimal:
    return loyalty.discount_rate(order.customer_id())
```

### D4. 给每个调用方一个窄接口

如果已经需要一个接口 (I2), 并且不同的地方使用它, 那么每个地方只依赖它用到的方法. 如果只有一个调用方, 就不要拆分接口.

```python
# 不好: 报表只需要一个方法, 却依赖了五个
class OrderStore(Protocol):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...
    def delete(self, order_id: str) -> None: ...
    def paid_between(self, start: date, end: date) -> list[Order]: ...
    def lock_for_update(self, order_id: str) -> None: ...

def revenue(store: OrderStore, start: date, end: date) -> Money: ...
```

```python
# 好
class PaidOrders(Protocol):
    def paid_between(self, start: date, end: date) -> list[Order]: ...

def revenue(orders: PaidOrders, start: date, end: date) -> Money:
    total = Money.zero("RUB")
    for order in orders.paid_between(start, end):
        total = total.plus(order.total())
    return total
```

---

## I - Inward

### I1. 业务逻辑不依赖基础设施

基础设施是代码在自身内存之外打交道的一切: 存储, 外部服务, 队列, 文件系统, 邮件, 时钟, 随机数生成器. 每个这样的系统都有自己的交流方式: 查询语言, 协议, 数据格式, 库, 错误码. 只有适配器知道这种方式.

适配器的存在不是为了切换到另一个外部系统: 那种事很少发生. 它存在是因为系统的细节一直在变 (表结构, 查询, 键, 库, 格式, 错误码), 而这些变化只应该影响适配器.

适配器的公共 API 用业务术语表达: 做什么, 对哪个对象做, 而不是外部系统如何做. 业务逻辑调用适配器, 不知道它背后是哪个系统, 也不知道如何与它交流.

适配器通过构造函数传入 (依赖注入), 并在入口点创建. 只有一个操作的基础设施作为函数参数传入.

例如: 用 `orders.save(order)` 代替 SQL 查询, 用 `carts.get(customer_id)` 代替 Redis 命令, 用 `shipping.ship(parcel)` 代替 HTTP 调用, 用 `events.publish(order_paid)` 代替向队列写字节, 把 `now()` 函数作为参数传入, 而不是在规则内部读取系统时钟.

```python
# 不好: "最多 20 件商品" 的规则和 Redis 的键与命令混在一起
MAX_ITEMS = 20

def add_to_cart(redis: RedisClient, customer_id: str, sku: str, quantity: int) -> None:
    key = "cart:" + customer_id
    if redis.hlen(key) >= MAX_ITEMS and not redis.hexists(key, sku):
        raise CartFull(customer_id)
    redis.hincrby(key, sku, quantity)
    redis.expire(key, 7 * 24 * 3600)
```

```python
# 好: 只有适配器知道 Redis, 规则在 Cart 里
CART_TTL_SECONDS = 7 * 24 * 3600

class RedisCarts:
    """适配器: 购物车在 Redis 中存为哈希 "sku -> 数量"."""

    def __init__(self, redis: RedisClient) -> None:
        self._redis = redis

    def get(self, customer_id: str) -> Cart:
        """顾客的购物车. 如果还不存在, 就是一个空购物车."""
        items = {}
        for sku, quantity in self._redis.hgetall("cart:" + customer_id).items():
            items[sku] = int(quantity)
        return Cart(customer_id, items)

    def save(self, cart: Cart) -> None:
        key = "cart:" + cart.customer_id()
        tx = self._redis.transaction()  # MULTI/EXEC: 旧购物车和新购物车不会混在一起
        tx.delete(key)
        if cart.items():
            tx.hset(key, cart.items())
            tx.expire(key, CART_TTL_SECONDS)
        tx.exec()

class Shopping:
    def __init__(self, carts: RedisCarts) -> None:
        self._carts = carts

    def add_to_cart(self, customer_id: str, sku: str, quantity: int) -> None:
        cart = self._carts.get(customer_id)
        cart.add(sku, quantity)  # 如果已经有 20 件商品, 抛出 CartFull
        self._carts.save(cart)

# main.py
shopping = Shopping(RedisCarts(RedisClient(env("REDIS_URL"))))
```

### I2. 只有当第二个实现真实存在时才引入接口

只有当确实有不止一个实现时才引入接口:
- 用于测试的伪实现;
- 第二个供应商或第二个数据库;
- 现在就已经有多种行为变体, 例如多种支付方式 (A1);
- 已经决定要做的替换.

"可能会用得上" 不是理由. 只有一个实现时, 调用方直接依赖这个类. 当出现第二个实现时, 从现有的类中提取接口, 这是一个机械性的改动: 调用方本来就只使用它的公共 API. 我们的 HTTP API 和用户不需要接口: 它们的输入由处理器接收 (T1).

```python
# 不好: 为只有一个实现的业务逻辑定义接口
class DiscountPolicy(Protocol):
    def apply(self, order: Order, customer: Customer) -> Money: ...

class DefaultDiscountPolicy:
    def apply(self, order: Order, customer: Customer) -> Money: ...

class Checkout:
    def __init__(self, orders: PostgresOrders, discounts: DiscountPolicy) -> None:
        self._orders = orders
        self._discounts = discounts

    def quote(self, order: Order, customer: Customer) -> Money:
        return self._discounts.apply(order, customer).plus(shipping_cost(order.total()))
```

```python
# 好
class Checkout:
    def __init__(self, orders: PostgresOrders) -> None:
        self._orders = orders

    def quote(self, order: Order, customer: Customer) -> Money:
        return price_after_loyalty_discount(order, customer).plus(shipping_cost(order.total()))
```

```python
# 好: 出现了第二个实现 (用于测试的伪实现 InMemoryOrders) - 于是提取了接口
class Orders(Protocol):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class Checkout:
    def __init__(self, orders: Orders) -> None:
        self._orders = orders
```

### I3. 决策与副作用分离

纯函数计算业务决策. 薄薄的一层代码根据它的结果执行副作用 (支付, 邮件, 写数据库). 对象在自己的方法里修改自己的状态, 在这里不算副作用.

```python
# 不好: 决策和副作用混在一起
def process_return(order: Order, lines: list[OrderLine], payments: Payments, notifier: Notifier) -> None:
    refund = Money.zero(order.currency())
    for line in lines:
        if line.is_final_sale():
            notifier.item_not_returnable(order.customer_id(), line)
        else:
            refund = refund.plus(line.price())
    payments.refund(order.payment_id(), refund)
```

```python
# 好
@dataclass(frozen=True)
class ReturnDecision:
    refund: Money
    rejected: list[OrderLine]

def decide_return(order: Order, lines: list[OrderLine]) -> ReturnDecision:
    refund = Money.zero(order.currency())
    rejected = []
    for line in lines:
        if line.is_final_sale():
            rejected.append(line)
        else:
            refund = refund.plus(line.price())
    return ReturnDecision(refund, rejected)

class Returns:
    def __init__(self, payments: Payments, notifier: Notifier) -> None:
        self._payments = payments
        self._notifier = notifier

    def process(self, order: Order, lines: list[OrderLine]) -> None:
        decision = decide_return(order, lines)
        self._payments.refund(order.payment_id(), decision.refund)
        self._notifier.items_not_returnable(order.customer_id(), decision.rejected)
```

### I4. 包之间没有循环依赖

包之间的导入不形成循环. 打破循环有两种方式: 一个包用自己的术语描述它的公共 API, 不再了解它的邻居; 或者把接口移到使用它的包里.

```
# 不好
shop/
  orders/     # 导入 payments 来扣款
  payments/   # 导入 orders 来把订单状态设为 "已支付"
```

```
# 好
shop/
  orders/     # 了解 payments: 调用 charge(), 自己修改自己的状态
  payments/   # 对订单一无所知: charge(amount, card, key) -> Receipt
  carts/
  http/       # 处理器
  main.py     # 入口点: 创建适配器并把各个包连接起来
```

---

## S - Single owner

### S1. 每一项知识只有一个所有者

知识是一个本可以有不同选择的决定: 业务规则, 格式, 算法, 供应商的选择. 代码按知识拆分, 而不是按执行步骤拆分. 每一项知识只存在于一个地方 (函数, 类或模块); 一起变化的知识放在一起. 因为不同原因而变化的相似代码是两项知识, 不要合并它们.

```python
# 不好: "满 5000 免运费" 存在于两个地方
# orders/checkout.py
def shipping_cost(order: Order) -> Money:
    if order.total().amount >= Decimal("5000"):
        return Money.zero("RUB")
    return Money(Decimal("300"), "RUB")

# http/cart_handler.py
body = {"total": str(cart.total().amount), "free_shipping": cart.total().amount >= Decimal("5000")}
```

```python
# 好: 这项知识属于 shipping/tariff.py
FREE_SHIPPING_FROM = Money(Decimal("5000"), "RUB")
BASE_COST = Money(Decimal("300"), "RUB")

def ships_free(order_total: Money) -> bool:
    return order_total.is_at_least(FREE_SHIPPING_FROM)

def shipping_cost(order_total: Money) -> Money:
    if ships_free(order_total):
        return Money.zero(order_total.currency)
    return BASE_COST

# http/cart_handler.py
body = {"total": str(cart.total().amount), "free_shipping": ships_free(cart.total())}
```

### S2. 只有所有者才能修改不变量

带有不变量的状态被封闭在所有者对象内部. 外部只能通过所有者的方法修改它, 由所有者检查状态转换. 调用方说要做什么, 而不是替所有者做决定.

```python
# 不好: 管理后台直接写状态, 绕过了状态转换规则
def handle_update_order(request: HttpRequest) -> HttpResponse:
    order = orders.get(request.path_param("order_id"))
    order.status = request.json_string("status")  # 未支付的订单也能被 "发货"
    orders.save(order)
    return HttpResponse(204)
```

```python
# 好: 一个单独的动作, 由 Order 检查状态转换
class Order:
    def ship(self, tracking_number: str) -> None:
        if self._status != OrderStatus.PAID:
            raise OrderCannotBeShipped(self._id, self._status)
        self._status = OrderStatus.SHIPPED
        self._tracking_number = tracking_number

    def status(self) -> OrderStatus:
        return self._status

def handle_ship_order(request: HttpRequest) -> HttpResponse:
    tracking_number = request.json_string("tracking_number")
    if tracking_number is None:
        return HttpResponse(400, {"error": "需要 tracking_number"})
    try:
        fulfillment.ship(request.path_param("order_id"), tracking_number)
    except OrderCannotBeShipped:
        return HttpResponse(409, {"error": "只有已支付的订单才能发货"})
    return HttpResponse(204)
```

---

## T - Translate at the boundary

### T1. 外部数据在边界处解析为值对象

外部格式 (请求 JSON, 表中的一行, 外部 API 的响应, Redis 中的值) 只解析一次, 变成保证正确性的值对象. 传入的请求由处理器解析, 基础设施的响应由适配器解析. 业务逻辑不重复验证数据, 也不读取外部格式.

```python
# 不好: 原始 JSON 进入业务逻辑, 每个函数都要重新验证一遍
def handle_set_address(request: HttpRequest) -> HttpResponse:
    delivery.set_address(request.path_param("order_id"), request.json_body())
    return HttpResponse(204)

class Delivery:
    def set_address(self, order_id: str, payload: dict) -> None:
        if len(str(payload.get("zip", ""))) != 6:
            raise ValueError("邮编错误")
        ...

    def estimate_days(self, payload: dict) -> int:
        if len(str(payload.get("zip", ""))) != 6:
            raise ValueError("邮编错误")
        ...
```

```python
# 好: 处理器构造 Address, 只有它进入内部
class PostalCode:
    def __init__(self, value: str) -> None:
        if len(value) != 6 or not value.isdigit():
            raise InvalidAddress("邮编必须是 6 位数字: " + value)
        self.value = value

class Address:
    def __init__(self, postal_code: PostalCode, street: str) -> None:
        if street.strip() == "":
            raise InvalidAddress("街道为空")
        self.postal_code = postal_code
        self.street = street

def handle_set_address(request: HttpRequest) -> HttpResponse:
    zip_code = request.json_string("zip")
    street = request.json_string("street")
    if zip_code is None or street is None:
        return HttpResponse(400, {"error": "需要 zip 和 street"})
    try:
        address = Address(PostalCode(zip_code), street)
    except InvalidAddress as error:
        return HttpResponse(422, {"error": str(error)})
    delivery.set_address(request.path_param("order_id"), address)
    return HttpResponse(204)

class Delivery:
    def set_address(self, order_id: str, address: Address) -> None: ...
    def estimate_days(self, address: Address) -> int: ...
```

### T2. 外部错误在适配器中转换为我们自己的错误

适配器把库或外部 API 的错误一次性转换为它自己的异常. 业务逻辑只看到这个异常.

```python
# 不好: 适配器让外部错误和外部响应泄露出去
class CourierApiShipping:
    def ship(self, parcel: Parcel) -> HttpResult:
        return self._http.post("/v2/shipments", parcel_to_json(parcel), headers={})

class Fulfillment:
    def ship(self, order: Order) -> None:
        try:
            result = self._shipping.ship(order.parcel())
        except NetworkError:
            order.postpone_shipping()
            return
        if result.status != 201:
            order.postpone_shipping()
            return
        order.ship(result.body["tracking"])
```

```python
# 好
class CourierApiShipping:
    """快递服务 API 的适配器."""

    def __init__(self, http: HttpClient) -> None:
        self._http = http

    def ship(self, parcel: Parcel) -> str:
        """把包裹交给快递, 返回运单号. 故障或拒绝 -> ShippingUnavailable."""
        try:
            result = self._http.post("/v2/shipments", parcel_to_json(parcel), headers={})
        except NetworkError as error:
            raise ShippingUnavailable(parcel.order_id, error)
        if result.status != 201 or result.body is None or "tracking" not in result.body:
            raise ShippingUnavailable(parcel.order_id)
        return result.body["tracking"]

class Fulfillment:
    def ship(self, order: Order) -> None:
        try:
            order.ship(self._shipping.ship(order.parcel()))
        except ShippingUnavailable:
            order.postpone_shipping()
```

---

## A - Add a kind, not a branch

### A1. 新的行为变体是一个新的实现

行为变体 (一种支付方式, 一种配送服务) 是一个公共接口的独立实现. 在入口点或处理器中选择合适的实现. 业务逻辑中没有针对变体的 `if`/`elif` 或标志. 例外是封闭, 稳定且有很多操作的集合 (订单状态): 把它描述成枚举 (enum), 并在一个所有者处完整地处理.

```python
# 不好: 每种新的支付方式都是业务逻辑里的一个新分支
class Checkout:
    def pay(self, order: Order, method: str, card_token: str | None) -> Receipt:
        if method == "card":
            return self._acquirer.charge(order.total(), card_token, order.id())
        elif method == "cash_on_delivery":
            return Receipt.pending(order.id())
        raise ValueError(method)
```

```python
# 好
class PaymentMethod(Protocol):
    def charge(self, order: Order) -> Receipt: ...

class CardPayment:
    def __init__(self, acquirer: HttpAcquirer, card_token: str) -> None:
        self._acquirer = acquirer
        self._card_token = card_token

    def charge(self, order: Order) -> Receipt:
        return self._acquirer.charge(order.total(), self._card_token, order.id())

class CashOnDelivery:
    def charge(self, order: Order) -> Receipt:
        return Receipt.pending(order.id())

class Checkout:
    def pay(self, order: Order, method: PaymentMethod) -> Receipt:
        return method.charge(order)

# 处理器: 根据输入选择实现, 是解析请求的一部分
def payment_method_from(request: HttpRequest, acquirer: HttpAcquirer) -> PaymentMethod | None:
    method = request.json_string("method")
    card_token = request.json_string("card_token")
    if method == "card" and card_token is not None:
        return CardPayment(acquirer, card_token)
    if method == "cash_on_delivery":
        return CashOnDelivery()
    return None  # 处理器会响应 400
```

### A2. 每个实现都完整履行契约

实现对调用方的要求不比接口多, 承诺不比接口少, 也不抛出契约之外的异常. 如果某个实现无法履行契约, 就修改契约或拆分接口.

```python
# 不好
class CashOnDelivery:
    def refund(self, receipt: Receipt) -> None:
        raise NotImplementedError("现金无法自动退款")
```

```python
# 好: 契约如实描述了两种结果
class RefundResult(Enum):
    REFUNDED = "refunded"
    OPERATOR_TASK_CREATED = "operator_task_created"

class PaymentMethod(Protocol):
    def refund(self, receipt: Receipt) -> RefundResult:
        """立即退款, 或者为操作员创建一个任务."""
        ...

class CashOnDelivery:
    def __init__(self, tasks: OperatorTasks) -> None:
        self._tasks = tasks

    def refund(self, receipt: Receipt) -> RefundResult:
        self._tasks.return_cash(receipt)
        return RefundResult.OPERATOR_TASK_CREATED
```

### A3. 组合优于继承

复用别人的行为时, 把对象放在字段里并调用它. 只有当子类完整履行父类的契约, 并且父类本身就是为继承而设计的时候, 才继承这个类.

```python
# 不好: 列表的子类得到了 append, extend, insert, 它们都绕过了 add
class OrderLines(list):
    def add(self, line: OrderLine) -> None:
        for existing in self:
            if existing.sku == line.sku:
                existing.quantity += line.quantity
                return
        self.append(line)

lines.extend([line_a, line_a])  # 同一件商品现在占了两行
```

```python
# 好
class OrderLines:
    def __init__(self) -> None:
        self._quantities: dict[str, int] = {}

    def add(self, sku: str, quantity: int) -> None:
        self._quantities[sku] = self._quantities.get(sku, 0) + quantity

    def quantity_of(self, sku: str) -> int:
        return self._quantities.get(sku, 0)
```

---

## N - Needed now

### N1. 只做现在需要的东西

不要添加当前任务不需要的功能, 参数, 设置或扩展点. "可能会用得上" 不是理由. 要泛化的是公共 API 的形状 (接受一个包裹, 而不是 "一个 VIP 客户的订单"), 而不是功能的集合. 只有以后修改代价很高的东西才需要提前考虑: 存储数据的格式, 以及面向外部使用者的 HTTP API.

```python
# 不好: "为了将来" 的灵活性
SHIPPING_PLUGINS: dict[str, type] = {}

class CourierApiShipping:
    def ship(self, parcel: Parcel, priority: int = 0, split_by_warehouse: bool = False,
             insurance: Money | None = None, callback_url: str | None = None) -> str: ...
```

```python
# 好
class CourierApiShipping:
    def ship(self, parcel: Parcel) -> str: ...
```

---

## C - Contract

### C1. 契约写在公共 API 中

调用方必须知道, 但从名称和类型中看不出来的一切, 都写在方法的文档里: 特殊情况下的行为, 单位, 调用顺序, 异常.

```python
# 不好: 库存不足时会发生什么, 以及如何释放预留, 都不清楚
def reserve(self, sku: str, quantity: int) -> bool:
    if self._free.get(sku, 0) < quantity:
        return False
    self._free[sku] -= quantity
    return True
```

```python
# 好
def reserve(self, sku: str, quantity: int) -> bool:
    """为订单预留 quantity 件 sku.

    如果可用库存不足, 返回 False 且不做任何修改.
    预留由 release() 释放, 或由 ship() 转为出库.
    """
    if self._free.get(sku, 0) < quantity:
        return False
    self._free[sku] -= quantity
    return True
```

### C2. 测试通过公共 API 检查契约

测试像调用方一样调用代码, 并检查结果. 测试中的基础设施用伪实现 (I2) 替换, 而不是用 mock. 测试不检查私有字段, 也不检查内部调用的顺序.

```python
# 不好: 测试和内部细节绑在一起
def test_place_order() -> None:
    orders = Mock()
    checkout = Checkout(orders)
    checkout.place_order(make_cart(), make_customer())
    assert orders.save.was_called_once()
    assert checkout._last_order.status == "new"
```

```python
# 好
class InMemoryOrders:
    def __init__(self) -> None:
        self._orders: dict[str, Order] = {}

    def get(self, order_id: str) -> Order | None:
        return self._orders.get(order_id)

    def save(self, order: Order) -> None:
        self._orders[order.id()] = order

def test_placed_order_is_saved_as_new() -> None:
    orders = InMemoryOrders()
    order_id = Checkout(orders).place_order(make_cart(), make_customer())
    assert orders.get(order_id).status() == OrderStatus.NEW
```

### C3. 没有测试的代码: 先提取决策, 再用测试固定它

在修改没有测试的代码之前, 先把业务决策移到一个可以单独调用的函数里, 并用测试固定当前的行为, 即使它很奇怪. 然后才修改行为. 其他原则在之后的步骤中应用.

```python
# 不好: 我们直接在这里修改条件, 却不知道会破坏什么
def delivery_date(order_id: str) -> datetime:
    row = db.fetch_one("SELECT city FROM orders WHERE id = $1", [order_id])
    days = 2 if row["city"] == "Moscow" else 5
    if datetime.now().weekday() >= 4:
        days += 2
    return datetime.now() + timedelta(days=days)
```

```python
# 好
def delivery_date(order_id: str) -> datetime:
    row = db.fetch_one("SELECT city FROM orders WHERE id = $1", [order_id])
    return delivery_date_for(row["city"], datetime.now())

def delivery_date_for(city: str, ordered_at: datetime) -> datetime:
    days = 2 if city == "Moscow" else 5
    if ordered_at.weekday() >= 4:
        days += 2
    return ordered_at + timedelta(days=days)

def test_friday_order_to_moscow_arrives_on_tuesday() -> None:
    friday = datetime(2026, 11, 6, 12, 0)
    assert delivery_date_for("Moscow", friday) == datetime(2026, 11, 10, 12, 0)
```

---

## E - Evident

### E1. 名称说明它是什么以及为什么

名称表达领域中的含义, 而不是数据类型, 也不是内部细节. 有意义的数字和字符串要有名字.

```python
# 不好
def calc(o: Order, c: Customer) -> Money:
    t = o.total()
    if c.lvl > 2:
        t = t.times(Decimal("0.95"))
    return t
```

```python
# 好
GOLD_LEVEL = 3
GOLD_DISCOUNT = Decimal("0.05")

def price_after_loyalty_discount(order: Order, customer: Customer) -> Money:
    total = order.total()
    if customer.loyalty_level() >= GOLD_LEVEL:
        return total.times(1 - GOLD_DISCOUNT)
    return total
```

### E2. 同样的事情用同样的方式做

同一个问题在整个系统中用同一种方式解决: 相同的方法名, 用同一种方式表示 "没有值" 和 "错误". 新代码遵循已有的方式. 如果已有的方式违反了 D, I, S 或 T, 新的地方要按正确的方式写, 并明确指出这个不一致.

```python
# 不好: 三个适配器, 三种表示 "未找到" 的方式
class PostgresCustomers:
    def fetch_customer(self, customer_id: str) -> Customer: ...  # 抛出 KeyError

class PostgresShipments:
    def load(self, shipment_id: str) -> Shipment | bool: ...  # 没有该发货单时返回 False

class PostgresOrders:
    def get(self, order_id: str) -> Order | None: ...
```

```python
# 好
class PostgresCustomers:
    def get(self, customer_id: str) -> Customer | None: ...

class PostgresShipments:
    def get(self, shipment_id: str) -> Shipment | None: ...

class PostgresOrders:
    def get(self, order_id: str) -> Order | None: ...
```

### E3. 注释只解释 "为什么"

在代码内部, 注释解释代码本身无法表达的原因. 禁止在注释里复述代码.

```python
# 不好
# 预留 30 分钟
RESERVATION_TTL_SECONDS = 30 * 60
```

```python
# 好
# 银行对银行卡的预授权保留 30 分钟: 把商品保留得更久没有意义.
RESERVATION_TTL_SECONDS = 30 * 60
```

---

## 完整示例: 订单支付

一个小而完整的用例: `POST /orders/{order_id}/pay`, 请求体为 `{"card_token": "..."}`. 订单存放在 Postgres 中, 由外部收单 API 扣款.

```
shop/
  payments/
    receipt.py        # Receipt 和支付错误 (领域类型)
    http_acquirer.py  # HttpAcquirer (适配器)
  orders/
    order.py          # Order (实体) 及其不变量
    payment.py        # OrderPayment (用例) 及它需要的接口
    postgres.py       # PostgresOrders (适配器)
  http/
    pay_handler.py    # 处理器
  main.py             # 入口点
tests/
  test_order_payment.py
```

### 请求路径

```
图例
  (名称)    外部世界: 客户端, 数据库, 外部 API
  [名称]    系统边界: 处理器或适配器
  {名称}    业务逻辑: 用例
  <名称>    领域类型: 实体或值对象
  !名称     我们的异常
  -N->      第 N 步: 一次调用
  <---      该步的结果
  .....     出了什么问题 -> 抛出什么 -> 给客户端的响应 (由处理器决定)

外部错误 (HTTP 402, 网络故障, 格式错误的 JSON) 用文字写出:
它们不会越过适配器; 取而代之抛出的是我们的 !异常.

==============================================================================

(客户端)                                                        外部世界
   |  POST /orders/o-1/pay   {"card_token": "tok_123"}
   v
[PayOrderHandler]                                               处理器
   |  JSON -> order_id = "o-1", card_token = "tok_123"
   |  没有 card_token ...................... -> 400
   v
{OrderPayment}.pay("o-1", "tok_123")                            用例
   |
   |-1-> [PostgresOrders].get("o-1")                            适配器
   |       --SELECT--> (Postgres)
   |<---- <Order o-1, 990 RUB, NEW>           表中的行 -> Order
   |       没有这一行 -> None .............. !OrderNotFound -> 404
   |
   |-2-> <Order>.ensure_payable()                               实体
   |       状态不是 NEW .................... !OrderNotPayable -> 409
   |
   |-3-> [HttpAcquirer].charge(<990 RUB>, "tok_123", key="o-1") 适配器
   |       --POST /v1/charges--> (收单方)
   |<---- <Receipt pay_77, 990 RUB>           响应 JSON -> Receipt
   |       HTTP 402 ........................ !PaymentDeclined -> 402
   |       网络, 超时, 5xx, 错误的 JSON .... !PaymentUnavailable -> 503
   |
   |-4-> <Order>.mark_paid(<Receipt>)                           实体
   |       NEW -> PAID
   |
   |-5-> [PostgresOrders].save(<Order>)                         适配器
   |       --UPDATE--> (Postgres)
   |       Postgres 故障 ................... 错误原样抛出 -> 500
   v
[PayOrderHandler]                                               处理器
   |  <Order> -> JSON
   v
(客户端)  200 {"order_id": "o-1", "status": "paid"}             外部世界
```

### 谁依赖谁

箭头表示 "导入". 领域类型除了彼此之外不导入任何东西; 数据库和 HTTP 客户端 (`Database`, `HttpClient`) 只被适配器和 `main.py` 导入.

```
[PayOrderHandler] -------> {OrderPayment} -------> <Order>, <Receipt>
 http/pay_handler           orders/payment          orders/order, payments/receipt
                                  ^                        ^
                                  | 符合                   |
                                  | Orders 和              |
                                  | Acquirer 接口          |
                                  | (不导入)               |
[PostgresOrders], [HttpAcquirer] -+------------------------+
 orders/postgres, payments/http_acquirer

main.py -------> 全部: 创建适配器并把它们传给用例
```

### 代码

**payments/receipt.py** - 支付的领域类型. 用例会导入它们, 所以这里没有任何基础设施相关的东西.

```python
@dataclass(frozen=True)
class Receipt:
    payment_id: str
    amount: Money

class PaymentDeclined(Exception):
    """银行拒绝了. 用同一张卡重试也没用."""

class PaymentUnavailable(Exception):
    """收单服务不可用. 可以稍后重试."""
```

**payments/http_acquirer.py** - 收单服务的适配器. 只有它知道 URL, 请求字段和响应码.

```python
class HttpAcquirer:
    def __init__(self, http: HttpClient) -> None:
        self._http = http

    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        """要么全额扣款, 要么什么都不扣.

        使用相同 idempotency_key 的重复调用不会再次扣款.
        银行拒绝 -> PaymentDeclined. 网络或服务故障 -> PaymentUnavailable.
        """
        try:
            result = self._http.post(
                "/v1/charges",
                {"amount": str(amount.amount), "currency": amount.currency, "card": card_token},
                headers={"Idempotency-Key": idempotency_key},
            )
        except NetworkError as error:
            raise PaymentUnavailable(error)
        if result.status == 402:
            raise PaymentDeclined()
        if result.status != 200 or result.body is None or "id" not in result.body:
            raise PaymentUnavailable()
        return Receipt(result.body["id"], amount)
```

**orders/order.py** - 实体. 订单自己保证只有新订单才能被支付.

```python
class OrderStatus(Enum):
    NEW = "new"
    PAID = "paid"
    CANCELLED = "cancelled"

class OrderNotFound(Exception): ...

class OrderNotPayable(Exception):
    """订单已经支付或已取消."""

class Order:
    def __init__(self, order_id: str, total: Money, status: OrderStatus, payment_id: str | None) -> None:
        self._id = order_id
        self._total = total
        self._status = status
        self._payment_id = payment_id

    def id(self) -> str:
        return self._id

    def total(self) -> Money:
        return self._total

    def status(self) -> OrderStatus:
        return self._status

    def payment_id(self) -> str | None:
        return self._payment_id

    def ensure_payable(self) -> None:
        if self._status != OrderStatus.NEW:
            raise OrderNotPayable(self._id)

    def mark_paid(self, receipt: Receipt) -> None:
        self.ensure_payable()
        self._status = OrderStatus.PAID
        self._payment_id = receipt.payment_id
```

**orders/payment.py** - 用例. 既不知道 SQL, 也不知道 HTTP. 之所以有接口, 是因为每个接口都有两个实现: 真实的实现和用于测试的伪实现.

```python
class Orders(Protocol):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class Acquirer(Protocol):
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt: ...

class OrderPayment:
    def __init__(self, orders: Orders, acquirer: Acquirer) -> None:
        self._orders = orders
        self._acquirer = acquirer

    def pay(self, order_id: str, card_token: str) -> Order:
        """用银行卡支付订单.

        错误: OrderNotFound, OrderNotPayable, PaymentDeclined, PaymentUnavailable.
        """
        order = self._orders.get(order_id)
        if order is None:
            raise OrderNotFound(order_id)
        order.ensure_payable()
        # 键就是订单 id: 保存失败后重试不会扣款两次.
        receipt = self._acquirer.charge(order.total(), card_token, idempotency_key=order.id())
        order.mark_paid(receipt)
        self._orders.save(order)
        return order
```

**orders/postgres.py** - Postgres 的适配器. 只有它知道表结构和 SQL.

```python
class PostgresOrders:
    def __init__(self, db: Database) -> None:
        self._db = db

    def get(self, order_id: str) -> Order | None:
        row = self._db.fetch_one(
            "SELECT id, total_amount, currency, status, payment_id FROM orders WHERE id = $1",
            [order_id],
        )
        if row is None:
            return None
        return Order(
            row["id"],
            Money(row["total_amount"], row["currency"]),
            OrderStatus(row["status"]),
            row["payment_id"],
        )

    def save(self, order: Order) -> None:
        self._db.execute(
            "UPDATE orders SET status = $1, payment_id = $2 WHERE id = $3",
            [order.status().value, order.payment_id(), order.id()],
        )
```

**http/pay_handler.py** - 处理器. 解析请求, 调用业务逻辑, 把它的错误转换为 HTTP 状态码. 自己什么都不决定.

```python
class PayOrderHandler:
    """POST /orders/{order_id}/pay  {"card_token": "..."}"""

    def __init__(self, payment: OrderPayment) -> None:
        self._payment = payment

    def handle(self, request: HttpRequest) -> HttpResponse:
        card_token = request.json_string("card_token")
        if card_token is None:
            return HttpResponse(400, {"error": "需要 card_token"})
        try:
            order = self._payment.pay(request.path_param("order_id"), card_token)
        except OrderNotFound:
            return HttpResponse(404, {"error": "订单不存在"})
        except OrderNotPayable:
            return HttpResponse(409, {"error": "订单已经支付或已取消"})
        except PaymentDeclined:
            return HttpResponse(402, {"error": "银行拒绝了这笔支付"})
        except PaymentUnavailable:
            return HttpResponse(503, {"error": "支付暂时不可用, 请稍后重试"})
        return HttpResponse(200, {"order_id": order.id(), "status": order.status().value})
```

**main.py** - 入口点. 唯一创建适配器并把一切连接起来的地方. `Router` 是占位类型: 这里可以接入任意 HTTP 服务器或框架.

```python
db = Database(env("DATABASE_URL"))
acquirer = HttpAcquirer(HttpClient(base_url=env("ACQUIRER_URL"), timeout_seconds=10))
payment = OrderPayment(PostgresOrders(db), acquirer)

router = Router()
router.post("/orders/{order_id}/pay", PayOrderHandler(payment).handle)
router.serve(port=8080)
```

**tests/test_order_payment.py** - 基于伪实现的业务逻辑测试, 不需要数据库和网络.

```python
class InMemoryOrders:
    def __init__(self) -> None:
        self._orders: dict[str, Order] = {}

    def get(self, order_id: str) -> Order | None:
        return self._orders.get(order_id)

    def save(self, order: Order) -> None:
        self._orders[order.id()] = order

class ApprovingAcquirer:
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        return Receipt("pay-1", amount)

class DecliningAcquirer:
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        raise PaymentDeclined()

def orders_with_new_order() -> InMemoryOrders:
    orders = InMemoryOrders()
    orders.save(Order("o-1", Money(Decimal("990"), "RUB"), OrderStatus.NEW, payment_id=None))
    return orders

def test_paid_order_becomes_paid() -> None:
    orders = orders_with_new_order()
    OrderPayment(orders, ApprovingAcquirer()).pay("o-1", "card-token")
    assert orders.get("o-1").status() == OrderStatus.PAID

def test_declined_payment_keeps_order_new() -> None:
    orders = orders_with_new_order()
    payment = OrderPayment(orders, DecliningAcquirer())
    expect_error(PaymentDeclined, lambda: payment.pay("o-1", "card-token"))
    assert orders.get("o-1").status() == OrderStatus.NEW

def test_order_cannot_be_paid_twice() -> None:
    orders = orders_with_new_order()
    payment = OrderPayment(orders, ApprovingAcquirer())
    payment.pay("o-1", "card-token")
    expect_error(OrderNotPayable, lambda: payment.pay("o-1", "card-token"))
```

**DISTANCE 体现在哪里**

| 规则 | 在哪里看到 |
|---|---|
| D1 | `HttpAcquirer.charge` 和 `OrderPayment.pay` - 各用一次调用代替多个步骤 |
| I1 | `OrderPayment` 既不知道 SQL 也不知道 HTTP; 适配器只在 `main.py` 中创建 |
| I2 | `Orders` 和 `Acquirer` 接口之所以存在, 是因为它们都有伪实现 |
| I4 | `orders` 了解 `payments`, 而 `payments` 对订单一无所知 |
| S2 | "只有新订单才能被支付" 这条规则在 `Order` 里 |
| T1 | 处理器验证请求体, 适配器解析表中的行和收单服务的响应 |
| T2 | `NetworkError` 和收单服务的响应码在适配器中变成 `PaymentDeclined` 和 `PaymentUnavailable`; 业务逻辑的错误在处理器中变成 HTTP 状态码 |
| C1 | `charge` 和 `pay` 的契约写在方法文档中 |
| C2 | 测试通过 `pay` 检查结果, 不使用 mock |
| E3 | 业务逻辑中唯一的注释解释了为什么键是订单 id |
