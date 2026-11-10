# DISTANCE

[English](README.md) | **Русский** | [中文](README.zh-CN.md)

**Держи дистанцию:** каждый кусок кода знает о других как можно меньше. Тогда код дёшево понять и безопасно изменить.

| | Принцип | Суть | Правила |
|---|---|---|---|
| [**D**](#d---deep) | Deep | Публичный API проще внутреннего устройства | [D1](#d1-публичный-api-проще-внутреннего-устройства) [D2](#d2-меньше-особых-случаев) [D3](#d3-обращайся-к-ближайшему) [D4](#d4-каждому-вызывающему---узкий-интерфейс) |
| [**I**](#i---inward) | Inward | Бизнес-логика не знает инфраструктуры | [I1](#i1-бизнес-логика-не-зависит-от-инфраструктуры) [I2](#i2-интерфейс---только-там-где-вторая-реализация-реальна) [I3](#i3-решение-отдельно-от-побочных-эффектов) [I4](#i4-пакеты-без-циклов) |
| [**S**](#s---single-owner) | Single owner | У каждого знания и каждого инварианта один владелец | [S1](#s1-у-каждого-знания-один-владелец) [S2](#s2-инвариант-меняет-только-владелец) |
| [**T**](#t---translate-at-the-boundary) | Translate at the boundary | Чужие данные и ошибки переводятся в свои один раз, на границе системы | [T1](#t1-чужие-данные-разбираются-в-объекты-значения-на-границе) [T2](#t2-чужие-ошибки-переводятся-в-свои-в-адаптере) |
| [**A**](#a---add-a-kind-not-a-branch) | Add a kind, not a branch | Новый вариант поведения - новая реализация, а не новое условие | [A1](#a1-новый-вариант-поведения---новая-реализация) [A2](#a2-каждая-реализация-выполняет-весь-контракт) [A3](#a3-композиция-вместо-наследования) |
| [**N**](#n---needed-now) | Needed now | Только то, что нужно сейчас | [N1](#n1-только-то-что-нужно-сейчас) |
| [**C**](#c---contract) | Contract | Контракт записан и проверен тестом | [C1](#c1-контракт-записан-в-публичном-api) [C2](#c2-тест-проверяет-контракт-через-публичный-api) [C3](#c3-код-без-тестов-сначала-вынести-решение-и-закрепить-тестом) |
| [**E**](#e---evident) | Evident | Код читается без сюрпризов | [E1](#e1-имя-говорит-что-это-и-зачем) [E2](#e2-одно-и-то-же-делается-одинаково) [E3](#e3-комментарий-объясняет-только-почему) |

Если правило в конкретном месте делает код сложнее, а не проще, оно не применяется.

---

## Глоссарий

| Термин | Что значит | Пример |
|---|---|---|
| **Интерфейс** (interface) | Тип, который описывает только методы, без кода. Его реализуют классы | `Orders` с методами `get` и `save` |
| **Реализация** (implementation) | Класс, который реализует интерфейс | `PostgresOrders`, `InMemoryOrders` реализуют `Orders` |
| **Публичный API** (public API) | То, что функция, класс или модуль дают вызывающему коду: публичные методы, их параметры и поведение. Есть у любого кода, даже без интерфейса | у `HttpAcquirer` публичный API - один метод `charge` |
| **Внутреннее устройство** | Код за публичным API, который вызывающему знать не нужно | SQL внутри `PostgresOrders.save` |
| **Контракт** (contract) | Обещания публичного API: что принимает, что гарантирует, какие ошибки бросает | "списывает целиком или ничего" |
| **Вызывающий код** (caller) | Код, который вызывает функцию или класс | `OrderPayment` - вызывающий код для `PostgresOrders` |
| **Бизнес-логика** (business logic) | Правила предметной области. Состоит из сущностей и сценариев | скидки, статусы заказа, сроки доставки |
| **Сущность** (entity) | Объект предметной области с идентичностью и инвариантами. Сам проверяет свои правила | `Order`, `Cart` |
| **Сценарий** (use case, сервис приложения) | Бизнес-логика одного действия пользователя: задаёт порядок шагов и связывает сущности с адаптерами | `OrderPayment`, `Shopping` |
| **Инфраструктура** (infrastructure) | Всё, с чем код общается за пределами своей памяти. Бизнес-решений не принимает | Postgres, Redis, внешние API, очереди, файлы, почта, часы |
| **Внешний API** | Чужой сервис, который вызывает наш код | эквайринг, курьерская служба |
| **Адаптер** (adapter) | Класс, через который наш код работает с внешней системой. Только адаптеры знают, как она устроена. Адаптер переводит наши объекты в её формат, а её ответы и ошибки - в наши. Бизнес-решений не принимает | `PostgresOrders`, `RedisCarts`, `HttpAcquirer` |
| **Обработчик** (handler) | Код, через который внешний мир обращается к нам: HTTP-эндпоинт, команда CLI, консьюмер очереди. Разбирает ввод, вызывает бизнес-логику, отдаёт ответ | `POST /orders/{id}/pay` |
| **Граница системы** | Обработчики и адаптеры вместе: всё, где наш код встречается с внешним миром | |
| **Внедрение зависимостей** (dependency injection) | Класс получает адаптеры через конструктор, а не создаёт их сам | `OrderPayment(orders, acquirer)` |
| **Точка входа** (entry point, `main`) | Место запуска, где создаются адаптеры и передаются в бизнес-логику | `main.py` |
| **Объект-значение** (value object) | Неизменяемый тип данных, который проверяет себя при создании | `PostalCode`, `Address`, `Money` |
| **Инвариант** (invariant) | Условие, которое объект поддерживает всегда | "отгружен только оплаченный заказ" |
| **Побочный эффект** (side effect) | Действие помимо возврата результата: запись в базу, сетевой вызов, письмо, изменение чужого объекта | `payments.refund(...)` |
| **Чистая функция** (pure function) | Функция без побочных эффектов: результат зависит только от аргументов | `decide_return(order, lines)` |
| **Фейк** (fake) | Простая рабочая реализация для тестов. Не путать с моком, который только проверяет вызовы | `InMemoryOrders` |
| **Композиция** (composition) | Класс хранит другой объект и вызывает его, вместо наследования | `OrderLines` хранит словарь |

### Как читать примеры

Принципы не зависят от языка. Примеры написаны на Python без фреймворков и особенностей языка, чтобы их читал программист на любом языке.

| В примерах | В Java | В Go |
|---|---|---|
| `class Orders(Protocol)` | `interface Orders` | `type Orders interface` |
| `@dataclass(frozen=True)` | `record` | `struct` без сеттеров |
| `raise SomeError` | `throw new SomeException()` | `return ..., ErrSome` |
| `None` | `null` / `Optional` | `nil` |
| `"""..."""` под методом | Javadoc | комментарий над функцией |
| `raise Ours(error)` | `throw new Ours(error)` | `fmt.Errorf("...: %w", err)` |
| `Decimal` | `BigDecimal` | десятичный тип без ошибок округления |

Типы магазина (`Money` с методами `plus`, `times`, `is_at_least`, `Order`, `Cart` и другие) считаются уже определёнными.

Вместо конкретных библиотек и фреймворков в примерах условные типы. Они есть в любом языке под своими именами:

| Условный тип | Что умеет |
|---|---|
| `HttpRequest`, `HttpResponse` | входящий запрос и ответ. `request.path_param(name)` - параметр пути, `request.json_string(name)` - строковое поле тела или `None`, `request.json_body()` - всё тело как словарь. `HttpResponse(status, body)` |
| `HttpClient` | исходящие HTTP-запросы. `post(path, body, headers)` возвращает `HttpResult` с полями `status` и `body` (разобранный JSON или `None`). Сбой сети или таймаут -> `NetworkError` |
| `Database` | клиент Postgres. `fetch_one(sql, params)` возвращает строку как словарь "колонка -> значение" или `None`, `execute(sql, params)` выполняет запрос |
| `RedisClient` | клиент Redis. Методы называются как команды Redis: `hgetall`, `hset`, `hincrby`, `expire`, `delete`. `transaction()` - группа команд, которая выполняется атомарно по `exec()` (MULTI/EXEC) |
| `Router` | любой HTTP-сервер или фреймворк |
| `Mock`, `expect_error` | мок из любой библиотеки моков; проверка, что функция бросает нужное исключение |
| `env(name)` | переменная окружения |

В коротких примерах сервисы обработчика опущены, в [полном примере](#полный-пример-оплата-заказа) видно, как они передаются.

---

## D - Deep

### D1. Публичный API проще внутреннего устройства

Функция, класс или модуль забирает сложность себе: вызывающий код делает один простой вызов, а всю работу выполняет внутреннее устройство. Новую функцию или класс выделяют, только если они что-то скрывают и их публичный API проще кода, который они заменяют. Длина кода сама по себе не повод делить.

```python
# Плохо: вызывающий код знает протокол эквайринга и порядок шагов
intent_id = acquirer.create_intent(order.total())
acquirer.attach_card(intent_id, card_token)
acquirer.confirm(intent_id)
payment_id = acquirer.capture(intent_id)
```

```python
# Хорошо
class HttpAcquirer:
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        """Списывает сумму целиком или не списывает ничего.

        Повторный вызов с тем же idempotency_key возвращает тот же чек и не списывает второй раз.
        Отказ банка -> PaymentDeclined. Сбой связи или сервиса -> PaymentUnavailable.
        """
        ...

receipt = acquirer.charge(order.total(), card_token, idempotency_key=order.id())
```

### D2. Меньше особых случаев

Контракт строят так, чтобы вызывающему коду нечего было обрабатывать отдельно. Если цель вызова уже достигнута, это успех, а не ошибка. Ситуации, на которые вызывающий код обязан реагировать (отказ банка, нехватка товара), не прячут.

```python
# Плохо: повторное удаление (двойной клик в браузере) - ошибка, которую ловит каждый
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
# Хорошо
class Cart:
    def remove(self, sku: str) -> None:
        """После вызова товара sku в корзине нет, был он там или нет."""
        if sku in self._items:
            del self._items[sku]

cart.remove(sku)
```

### D3. Обращайся к ближайшему

Код вызывает методы только у себя, своих параметров, своих полей и созданных им объектов. Он не ходит по цепочке через чужое внутреннее устройство. Исключения - объекты-значения и цепочные API вроде построителей запросов: у них структура и есть публичный API.

```python
# Плохо
def loyalty_discount(order: Order) -> Decimal:
    return order.customer().profile().loyalty().level().discount_rate()
```

```python
# Хорошо
def loyalty_discount(order: Order, loyalty: Loyalty) -> Decimal:
    return loyalty.discount_rate(order.customer_id())
```

### D4. Каждому вызывающему - узкий интерфейс

Если интерфейс уже нужен (I2) и его используют разные места, каждое из них зависит только от методов, которыми пользуется. Если вызывающий код один, интерфейс не дробят.

```python
# Плохо: отчёту нужен один метод, а зависит он от пяти
class OrderStore(Protocol):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...
    def delete(self, order_id: str) -> None: ...
    def paid_between(self, start: date, end: date) -> list[Order]: ...
    def lock_for_update(self, order_id: str) -> None: ...

def revenue(store: OrderStore, start: date, end: date) -> Money: ...
```

```python
# Хорошо
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

### I1. Бизнес-логика не зависит от инфраструктуры

Инфраструктура - всё, с чем код общается за пределами своей памяти: хранилища, внешние сервисы, очереди, файловая система, почта, часы, генератор случайных чисел. У каждой такой системы свой способ общения: язык запросов, протокол, формат данных, библиотека, коды ошибок. Этот способ знает только адаптер.

Адаптер нужен не ради смены внешней системы: её меняют редко. Он нужен потому, что постоянно меняется её устройство (схема, запросы, ключи, библиотека, формат, коды ошибок), и такие изменения должны затрагивать только адаптер.

Публичный API адаптера выражен в терминах бизнеса: что сделать и с каким объектом, а не как это делает внешняя система. Бизнес-логика вызывает адаптер и не знает, какая система за ним и как с ней говорят.

Адаптер передают через конструктор (внедрение зависимостей) и создают в точке входа. Инфраструктуру с одной операцией передают параметром функции.

Например: `orders.save(order)` вместо SQL-запроса, `carts.get(customer_id)` вместо команд Redis, `shipping.ship(parcel)` вместо HTTP-вызова, `events.publish(order_paid)` вместо записи байтов в очередь, функция `now()` в параметре вместо чтения системных часов внутри правила.

```python
# Плохо: правило "не больше 20 позиций" перемешано с ключами и командами Redis
MAX_ITEMS = 20

def add_to_cart(redis: RedisClient, customer_id: str, sku: str, quantity: int) -> None:
    key = "cart:" + customer_id
    if redis.hlen(key) >= MAX_ITEMS and not redis.hexists(key, sku):
        raise CartFull(customer_id)
    redis.hincrby(key, sku, quantity)
    redis.expire(key, 7 * 24 * 3600)
```

```python
# Хорошо: Redis знает только адаптер, правило живёт в Cart
CART_TTL_SECONDS = 7 * 24 * 3600

class RedisCarts:
    """Адаптер: корзина хранится в Redis как хеш "sku -> количество"."""

    def __init__(self, redis: RedisClient) -> None:
        self._redis = redis

    def get(self, customer_id: str) -> Cart:
        """Корзина покупателя. Если её ещё нет - пустая корзина."""
        items = {}
        for sku, quantity in self._redis.hgetall("cart:" + customer_id).items():
            items[sku] = int(quantity)
        return Cart(customer_id, items)

    def save(self, cart: Cart) -> None:
        key = "cart:" + cart.customer_id()
        tx = self._redis.transaction()  # MULTI/EXEC: старая корзина и новая не смешаются
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
        cart.add(sku, quantity)  # бросает CartFull, если позиций уже 20
        self._carts.save(cart)

# main.py
shopping = Shopping(RedisCarts(RedisClient(env("REDIS_URL"))))
```

### I2. Интерфейс - только там, где вторая реализация реальна

Интерфейс вводят, только когда реализаций реально больше одной:
- фейк для тестов;
- второй поставщик или вторая база;
- несколько вариантов поведения уже сейчас, например способов оплаты (A1);
- замена, которую уже решили сделать.

"Может пригодиться" - не причина. Пока реализация одна, вызывающий код зависит от класса напрямую. Когда появляется вторая, интерфейс выделяют из готового класса, и это механическая правка: вызывающий код и так пользуется только его публичным API. Наш HTTP API и пользователь интерфейса не требуют: их ввод принимает обработчик (T1).

```python
# Плохо: интерфейс для бизнес-логики с единственной реализацией
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
# Хорошо
class Checkout:
    def __init__(self, orders: PostgresOrders) -> None:
        self._orders = orders

    def quote(self, order: Order, customer: Customer) -> Money:
        return price_after_loyalty_discount(order, customer).plus(shipping_cost(order.total()))
```

```python
# Хорошо: появилась вторая реализация (фейк InMemoryOrders для тестов) - выделили интерфейс
class Orders(Protocol):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class Checkout:
    def __init__(self, orders: Orders) -> None:
        self._orders = orders
```

### I3. Решение отдельно от побочных эффектов

Бизнес-решение вычисляет чистая функция. Побочные эффекты (оплата, письма, запись в базу) выполняет тонкий код по её результату. Изменение объектом собственного состояния в его методе побочным эффектом здесь не считается.

```python
# Плохо: решение и побочные эффекты перемешаны
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
# Хорошо
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

### I4. Пакеты без циклов

Импорты между пакетами не образуют циклов. Цикл разрывают одним из двух способов: пакет описывает свой публичный API в собственных терминах и перестаёт знать о соседе, либо интерфейс переносят в пакет, который им пользуется.

```
# Плохо
shop/
  orders/     # импортирует payments, чтобы списать деньги
  payments/   # импортирует orders, чтобы поставить заказу статус "оплачен"
```

```
# Хорошо
shop/
  orders/     # знает payments: вызывает charge(), сам меняет свой статус
  payments/   # не знает о заказах: charge(amount, card, key) -> Receipt
  carts/
  http/       # обработчики
  main.py     # точка входа: создаёт адаптеры и связывает пакеты
```

---

## S - Single owner

### S1. У каждого знания один владелец

Знание - это решение, которое могло быть другим: бизнес-правило, формат, алгоритм, выбор поставщика. Код делят по знаниям, а не по шагам выполнения. Каждое знание живёт ровно в одном месте (функции, классе или модуле), знания, которые меняются вместе, лежат вместе. Похожий код, который меняется по разным причинам, - это два знания, и их не объединяют.

```python
# Плохо: "бесплатно от 5000" живёт в двух местах
# orders/checkout.py
def shipping_cost(order: Order) -> Money:
    if order.total().amount >= Decimal("5000"):
        return Money.zero("RUB")
    return Money(Decimal("300"), "RUB")

# http/cart_handler.py
body = {"total": str(cart.total().amount), "free_shipping": cart.total().amount >= Decimal("5000")}
```

```python
# Хорошо: знание принадлежит shipping/tariff.py
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

### S2. Инвариант меняет только владелец

Состояние с инвариантом закрыто внутри объекта-владельца. Снаружи его меняют только методами владельца, и владелец проверяет переход. Вызывающий код говорит, что сделать, а не решает за владельца.

```python
# Плохо: админка пишет статус напрямую, правило перехода обходится
def handle_update_order(request: HttpRequest) -> HttpResponse:
    order = orders.get(request.path_param("order_id"))
    order.status = request.json_string("status")  # можно "отгрузить" неоплаченный заказ
    orders.save(order)
    return HttpResponse(204)
```

```python
# Хорошо: отдельное действие, переход проверяет Order
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
        return HttpResponse(400, {"error": "нужен tracking_number"})
    try:
        fulfillment.ship(request.path_param("order_id"), tracking_number)
    except OrderCannotBeShipped:
        return HttpResponse(409, {"error": "отгрузить можно только оплаченный заказ"})
    return HttpResponse(204)
```

---

## T - Translate at the boundary

### T1. Чужие данные разбираются в объекты-значения на границе

Внешний формат (JSON запроса, строку таблицы, ответ внешнего API, значение из Redis) один раз разбирают в объекты-значения, которые гарантируют корректность. Входящие запросы разбирает обработчик, ответы инфраструктуры - адаптер. Бизнес-логика данные повторно не проверяет и внешний формат не читает.

```python
# Плохо: сырой JSON уходит в бизнес-логику, и каждая функция проверяет его заново
def handle_set_address(request: HttpRequest) -> HttpResponse:
    delivery.set_address(request.path_param("order_id"), request.json_body())
    return HttpResponse(204)

class Delivery:
    def set_address(self, order_id: str, payload: dict) -> None:
        if len(str(payload.get("zip", ""))) != 6:
            raise ValueError("bad zip")
        ...

    def estimate_days(self, payload: dict) -> int:
        if len(str(payload.get("zip", ""))) != 6:
            raise ValueError("bad zip")
        ...
```

```python
# Хорошо: обработчик собирает Address, внутрь уходит только он
class PostalCode:
    def __init__(self, value: str) -> None:
        if len(value) != 6 or not value.isdigit():
            raise InvalidAddress("индекс должен состоять из 6 цифр: " + value)
        self.value = value

class Address:
    def __init__(self, postal_code: PostalCode, street: str) -> None:
        if street.strip() == "":
            raise InvalidAddress("пустая улица")
        self.postal_code = postal_code
        self.street = street

def handle_set_address(request: HttpRequest) -> HttpResponse:
    zip_code = request.json_string("zip")
    street = request.json_string("street")
    if zip_code is None or street is None:
        return HttpResponse(400, {"error": "нужны zip и street"})
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

### T2. Чужие ошибки переводятся в свои в адаптере

Ошибку библиотеки или внешнего API адаптер один раз переводит в собственное исключение. Бизнес-логика видит только его.

```python
# Плохо: адаптер пропускает наружу чужую ошибку и чужой ответ
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
# Хорошо
class CourierApiShipping:
    """Адаптер к API курьерской службы."""

    def __init__(self, http: HttpClient) -> None:
        self._http = http

    def ship(self, parcel: Parcel) -> str:
        """Передаёт посылку курьеру, возвращает трек-номер. Сбой или отказ -> ShippingUnavailable."""
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

### A1. Новый вариант поведения - новая реализация

Вариант поведения (способ оплаты, служба доставки) - это отдельная реализация общего интерфейса. Нужную реализацию выбирают в точке входа или в обработчике. В бизнес-логике нет `if`/`elif` и флагов по варианту. Исключение - закрытый стабильный набор, над которым много операций (статусы заказа): его описывают перечислением (enum) и полностью разбирают в одном месте-владельце.

```python
# Плохо: каждый новый способ оплаты - новая ветка в бизнес-логике
class Checkout:
    def pay(self, order: Order, method: str, card_token: str | None) -> Receipt:
        if method == "card":
            return self._acquirer.charge(order.total(), card_token, order.id())
        elif method == "cash_on_delivery":
            return Receipt.pending(order.id())
        raise ValueError(method)
```

```python
# Хорошо
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

# обработчик: выбор реализации по вводу - это разбор запроса
def payment_method_from(request: HttpRequest, acquirer: HttpAcquirer) -> PaymentMethod | None:
    method = request.json_string("method")
    card_token = request.json_string("card_token")
    if method == "card" and card_token is not None:
        return CardPayment(acquirer, card_token)
    if method == "cash_on_delivery":
        return CashOnDelivery()
    return None  # обработчик ответит 400
```

### A2. Каждая реализация выполняет весь контракт

Реализация не требует от вызывающего кода больше, чем интерфейс, не обещает меньше и не бросает исключений вне контракта. Если реализация не может выполнить контракт, меняют контракт или делят интерфейс.

```python
# Плохо
class CashOnDelivery:
    def refund(self, receipt: Receipt) -> None:
        raise NotImplementedError("наличные автоматически не возвращаются")
```

```python
# Хорошо: контракт честно описывает оба исхода
class RefundResult(Enum):
    REFUNDED = "refunded"
    OPERATOR_TASK_CREATED = "operator_task_created"

class PaymentMethod(Protocol):
    def refund(self, receipt: Receipt) -> RefundResult:
        """Возвращает деньги сразу или ставит задачу оператору."""
        ...

class CashOnDelivery:
    def __init__(self, tasks: OperatorTasks) -> None:
        self._tasks = tasks

    def refund(self, receipt: Receipt) -> RefundResult:
        self._tasks.return_cash(receipt)
        return RefundResult.OPERATOR_TASK_CREATED
```

### A3. Композиция вместо наследования

Чужое поведение используют, храня объект в поле и вызывая его. Наследовать класс можно, только когда наследник полностью выполняет контракт родителя, а родитель создан для наследования.

```python
# Плохо: наследник списка получает append, extend, insert в обход add
class OrderLines(list):
    def add(self, line: OrderLine) -> None:
        for existing in self:
            if existing.sku == line.sku:
                existing.quantity += line.quantity
                return
        self.append(line)

lines.extend([line_a, line_a])  # одна позиция теперь в двух строках
```

```python
# Хорошо
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

### N1. Только то, что нужно сейчас

Не добавляют возможностей, параметров, настроек и точек расширения, которых не требует текущая задача. "Может пригодиться" - не причина. Обобщают форму публичного API (принять посылку, а не "заказ VIP-клиента"), а не набор возможностей. Заранее продумывают только то, что дорого изменить потом: формат хранимых данных и наш HTTP API для внешних потребителей.

```python
# Плохо: гибкость "на будущее"
SHIPPING_PLUGINS: dict[str, type] = {}

class CourierApiShipping:
    def ship(self, parcel: Parcel, priority: int = 0, split_by_warehouse: bool = False,
             insurance: Money | None = None, callback_url: str | None = None) -> str: ...
```

```python
# Хорошо
class CourierApiShipping:
    def ship(self, parcel: Parcel) -> str: ...
```

---

## C - Contract

### C1. Контракт записан в публичном API

Всё, что вызывающий код должен знать и чего не видно из имени и типов, записано в документации метода: поведение в особых случаях, единицы, порядок вызовов, исключения.

```python
# Плохо: что будет при нехватке и как снять резерв - неизвестно
def reserve(self, sku: str, quantity: int) -> bool:
    if self._free.get(sku, 0) < quantity:
        return False
    self._free[sku] -= quantity
    return True
```

```python
# Хорошо
def reserve(self, sku: str, quantity: int) -> bool:
    """Резервирует quantity единиц sku под заказ.

    Если свободного остатка не хватает, возвращает False и ничего не меняет.
    Резерв снимает release() или превращает в списание ship().
    """
    if self._free.get(sku, 0) < quantity:
        return False
    self._free[sku] -= quantity
    return True
```

### C2. Тест проверяет контракт через публичный API

Тест вызывает код так же, как вызывающий код, и проверяет результат. Инфраструктуру в тесте заменяют фейком (I2), а не моками. Приватные поля и порядок внутренних вызовов тест не проверяет.

```python
# Плохо: тест привязан к внутреннему устройству
def test_place_order() -> None:
    orders = Mock()
    checkout = Checkout(orders)
    checkout.place_order(make_cart(), make_customer())
    assert orders.save.was_called_once()
    assert checkout._last_order.status == "new"
```

```python
# Хорошо
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

### C3. Код без тестов: сначала вынести решение и закрепить тестом

Прежде чем менять код без тестов, бизнес-решение выносят в функцию, которую можно вызвать отдельно, и тестом фиксируют текущее поведение, даже странное. Только потом меняют поведение. Остальные принципы применяют следующими шагами.

```python
# Плохо: правим условие прямо здесь и не знаем, что ломаем
def delivery_date(order_id: str) -> datetime:
    row = db.fetch_one("SELECT city FROM orders WHERE id = $1", [order_id])
    days = 2 if row["city"] == "Москва" else 5
    if datetime.now().weekday() >= 4:
        days += 2
    return datetime.now() + timedelta(days=days)
```

```python
# Хорошо
def delivery_date(order_id: str) -> datetime:
    row = db.fetch_one("SELECT city FROM orders WHERE id = $1", [order_id])
    return delivery_date_for(row["city"], datetime.now())

def delivery_date_for(city: str, ordered_at: datetime) -> datetime:
    days = 2 if city == "Москва" else 5
    if ordered_at.weekday() >= 4:
        days += 2
    return ordered_at + timedelta(days=days)

def test_friday_order_to_moscow_arrives_on_tuesday() -> None:
    friday = datetime(2026, 11, 6, 12, 0)
    assert delivery_date_for("Москва", friday) == datetime(2026, 11, 10, 12, 0)
```

---

## E - Evident

### E1. Имя говорит, что это и зачем

Имя называет смысл в предметной области, а не тип данных и не внутреннее устройство. Значимые числа и строки получают имена.

```python
# Плохо
def calc(o: Order, c: Customer) -> Money:
    t = o.total()
    if c.lvl > 2:
        t = t.times(Decimal("0.95"))
    return t
```

```python
# Хорошо
GOLD_LEVEL = 3
GOLD_DISCOUNT = Decimal("0.05")

def price_after_loyalty_discount(order: Order, customer: Customer) -> Money:
    total = order.total()
    if customer.loyalty_level() >= GOLD_LEVEL:
        return total.times(1 - GOLD_DISCOUNT)
    return total
```

### E2. Одно и то же делается одинаково

Одна задача решается в системе одним способом: одинаковые имена методов, один способ сказать "нет значения" и "ошибка". Новый код следует принятому способу. Если принятый способ нарушает D, I, S или T, новое место делают правильно, а расхождение называют явно.

```python
# Плохо: три адаптера - три способа сказать "не найдено"
class PostgresCustomers:
    def fetch_customer(self, customer_id: str) -> Customer: ...  # бросает KeyError

class PostgresShipments:
    def load(self, shipment_id: str) -> Shipment | bool: ...  # False, если отправки нет

class PostgresOrders:
    def get(self, order_id: str) -> Order | None: ...
```

```python
# Хорошо
class PostgresCustomers:
    def get(self, customer_id: str) -> Customer | None: ...

class PostgresShipments:
    def get(self, shipment_id: str) -> Shipment | None: ...

class PostgresOrders:
    def get(self, order_id: str) -> Order | None: ...
```

### E3. Комментарий объясняет только "почему"

Внутри кода комментарий объясняет причину, которую код показать не может. Пересказывать код в комментарии запрещено.

```python
# Плохо
# резерв 30 минут
RESERVATION_TTL_SECONDS = 30 * 60
```

```python
# Хорошо
# Банк держит холд по карте 30 минут: дольше товар держать бессмысленно.
RESERVATION_TTL_SECONDS = 30 * 60
```

---

## Полный пример: оплата заказа

Маленький, но целый сценарий: `POST /orders/{order_id}/pay` с телом `{"card_token": "..."}`. Заказ лежит в Postgres, деньги списывает внешний API эквайринга.

```
shop/
  payments/
    receipt.py        # Receipt и ошибки оплаты (доменные типы)
    http_acquirer.py  # HttpAcquirer (адаптер)
  orders/
    order.py          # Order (сущность) и его инвариант
    payment.py        # OrderPayment (сценарий) и интерфейсы, которые ему нужны
    postgres.py       # PostgresOrders (адаптер)
  http/
    pay_handler.py    # обработчик
  main.py             # точка входа
tests/
  test_order_payment.py
```

### Путь запроса

```
Обозначения
  (Имя)     внешний мир: клиент, база, внешний API
  [Имя]     граница системы: обработчик или адаптер
  {Имя}     бизнес-логика: сценарий
  <Имя>     доменный тип: сущность или объект-значение
  !Имя      наше исключение
  -N->      шаг N: вызов
  <---      результат шага
  .....     что пошло не так -> что поднимается -> ответ клиенту (его выбирает обработчик)

Чужие ошибки (HTTP 402, сбой сети, кривой JSON) написаны словами:
дальше адаптера они не уходят, вместо них поднимается !НашеИсключение.

==============================================================================

(Клиент)                                                        внешний мир
   |  POST /orders/o-1/pay   {"card_token": "tok_123"}
   v
[PayOrderHandler]                                               обработчик
   |  JSON -> order_id = "o-1", card_token = "tok_123"
   |  нет card_token ........................ -> 400
   v
{OrderPayment}.pay("o-1", "tok_123")                            сценарий
   |
   |-1-> [PostgresOrders].get("o-1")                            адаптер
   |       --SELECT--> (Postgres)
   |<---- <Order o-1, 990 RUB, NEW>           строка таблицы -> Order
   |       нет строки -> None ............... !OrderNotFound -> 404
   |
   |-2-> <Order>.ensure_payable()                               сущность
   |       статус не NEW .................... !OrderNotPayable -> 409
   |
   |-3-> [HttpAcquirer].charge(<990 RUB>, "tok_123", key="o-1") адаптер
   |       --POST /v1/charges--> (Эквайринг)
   |<---- <Receipt pay_77, 990 RUB>           JSON ответа -> Receipt
   |       HTTP 402 ......................... !PaymentDeclined -> 402
   |       сеть, таймаут, 5xx, кривой JSON .. !PaymentUnavailable -> 503
   |
   |-4-> <Order>.mark_paid(<Receipt>)                           сущность
   |       NEW -> PAID
   |
   |-5-> [PostgresOrders].save(<Order>)                         адаптер
   |       --UPDATE--> (Postgres)
   |       сбой Postgres .................... ошибка как есть -> 500
   v
[PayOrderHandler]                                               обработчик
   |  <Order> -> JSON
   v
(Клиент)  200 {"order_id": "o-1", "status": "paid"}             внешний мир
```

### Кто от кого зависит

Стрелка - "импортирует". Доменные типы не импортируют ничего, кроме друг друга; клиенты базы и HTTP (`Database`, `HttpClient`) импортируют только адаптеры и `main.py`.

```
[PayOrderHandler] -------> {OrderPayment} -------> <Order>, <Receipt>
 http/pay_handler           orders/payment          orders/order, payments/receipt
                                  ^                        ^
                                  | соответствуют          |
                                  | интерфейсам Orders     |
                                  | и Acquirer             |
                                  | (без импорта)          |
[PostgresOrders], [HttpAcquirer] -+------------------------+
 orders/postgres, payments/http_acquirer

main.py -------> всё: создаёт адаптеры и передаёт их в сценарий
```

### Код

**payments/receipt.py** - доменные типы оплаты. Их импортирует сценарий, поэтому здесь нет ничего из инфраструктуры.

```python
@dataclass(frozen=True)
class Receipt:
    payment_id: str
    amount: Money

class PaymentDeclined(Exception):
    """Банк отказал. Повтор с той же картой не поможет."""

class PaymentUnavailable(Exception):
    """Эквайринг недоступен. Можно повторить позже."""
```

**payments/http_acquirer.py** - адаптер к эквайрингу. Только он знает URL, поля запроса и коды ответов.

```python
class HttpAcquirer:
    def __init__(self, http: HttpClient) -> None:
        self._http = http

    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        """Списывает сумму целиком или не списывает ничего.

        Повторный вызов с тем же idempotency_key не списывает второй раз.
        Отказ банка -> PaymentDeclined. Сбой связи или сервиса -> PaymentUnavailable.
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

**orders/order.py** - сущность. Заказ сам следит, что оплатить можно только новый заказ.

```python
class OrderStatus(Enum):
    NEW = "new"
    PAID = "paid"
    CANCELLED = "cancelled"

class OrderNotFound(Exception): ...

class OrderNotPayable(Exception):
    """Заказ уже оплачен или отменён."""

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

**orders/payment.py** - сценарий. Не знает ни SQL, ни HTTP. Интерфейсы есть, потому что у каждого по две реализации: настоящая и фейк для тестов.

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
        """Оплачивает заказ картой.

        Ошибки: OrderNotFound, OrderNotPayable, PaymentDeclined, PaymentUnavailable.
        """
        order = self._orders.get(order_id)
        if order is None:
            raise OrderNotFound(order_id)
        order.ensure_payable()
        # Ключ - id заказа: повтор после сбоя записи не спишет деньги дважды.
        receipt = self._acquirer.charge(order.total(), card_token, idempotency_key=order.id())
        order.mark_paid(receipt)
        self._orders.save(order)
        return order
```

**orders/postgres.py** - адаптер к Postgres. Только он знает схему таблицы и SQL.

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

**http/pay_handler.py** - обработчик. Разбирает запрос, вызывает бизнес-логику, переводит её ошибки в HTTP-статусы. Сам ничего не решает.

```python
class PayOrderHandler:
    """POST /orders/{order_id}/pay  {"card_token": "..."}"""

    def __init__(self, payment: OrderPayment) -> None:
        self._payment = payment

    def handle(self, request: HttpRequest) -> HttpResponse:
        card_token = request.json_string("card_token")
        if card_token is None:
            return HttpResponse(400, {"error": "нужен card_token"})
        try:
            order = self._payment.pay(request.path_param("order_id"), card_token)
        except OrderNotFound:
            return HttpResponse(404, {"error": "заказ не найден"})
        except OrderNotPayable:
            return HttpResponse(409, {"error": "заказ уже оплачен или отменён"})
        except PaymentDeclined:
            return HttpResponse(402, {"error": "банк отклонил оплату"})
        except PaymentUnavailable:
            return HttpResponse(503, {"error": "оплата временно недоступна, повторите позже"})
        return HttpResponse(200, {"order_id": order.id(), "status": order.status().value})
```

**main.py** - точка входа. Единственное место, где создаются адаптеры и всё связывается. `Router` - условный: здесь подключается любой HTTP-сервер или фреймворк.

```python
db = Database(env("DATABASE_URL"))
acquirer = HttpAcquirer(HttpClient(base_url=env("ACQUIRER_URL"), timeout_seconds=10))
payment = OrderPayment(PostgresOrders(db), acquirer)

router = Router()
router.post("/orders/{order_id}/pay", PayOrderHandler(payment).handle)
router.serve(port=8080)
```

**tests/test_order_payment.py** - тесты бизнес-логики на фейках, без базы и сети.

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

**Где здесь DISTANCE**

| Правило | Где видно |
|---|---|
| D1 | `HttpAcquirer.charge` и `OrderPayment.pay` - по одному вызову вместо нескольких шагов |
| I1 | `OrderPayment` не знает ни SQL, ни HTTP; адаптеры создаются только в `main.py` |
| I2 | интерфейсы `Orders` и `Acquirer` есть, потому что у каждого есть фейк |
| I4 | `orders` знает `payments`, а `payments` о заказах не знает |
| S2 | правило "оплатить можно только новый заказ" живёт в `Order` |
| T1 | обработчик проверяет тело запроса, адаптеры разбирают строку таблицы и ответ эквайринга |
| T2 | `NetworkError` и коды ответа эквайринга превращаются в `PaymentDeclined` и `PaymentUnavailable` в адаптере; ошибки бизнес-логики превращаются в HTTP-статусы в обработчике |
| C1 | контракт `charge` и `pay` записан в документации методов |
| C2 | тесты проверяют результат через `pay`, без моков |
| E3 | единственный комментарий в бизнес-логике объясняет, почему ключ - id заказа |
