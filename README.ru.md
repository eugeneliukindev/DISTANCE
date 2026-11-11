# DISTANCE

[English](README.md) | **Русский**

**Держи дистанцию:** каждый кусок кода знает о других как можно меньше. Тогда код дёшево понять и безопасно изменить.

| | Принцип | Суть | Правила |
|---|---|---|---|
| [**D**](#d---deep) | Deep | Публичный API проще внутреннего устройства | [D1](#d1-публичный-api-проще-внутреннего-устройства) [D2](#d2-меньше-особых-случаев) [D3](#d3-каждому-вызывающему---узкий-интерфейс) |
| [**I**](#i---inward) | Inward | Зависимости направлены внутрь, к бизнес-логике | [I1](#i1-бизнес-логика-не-зависит-от-инфраструктуры) [I2](#i2-пакеты-без-циклов) |
| [**S**](#s---single-owner) | Single owner | У знания, инварианта и состояния один владелец | [S1](#s1-у-каждого-знания-один-владелец) [S2](#s2-инвариант-меняет-только-владелец) [S3](#s3-меньше-изменяемого-состояния) |
| [**T**](#t---translate-at-the-boundary) | Translate at the boundary | Чужие данные и ошибки переводятся в свои один раз, на границе системы | [T1](#t1-чужие-данные-разбираются-в-объекты-значения-на-границе) [T2](#t2-чужие-ошибки-переводятся-в-свои-в-адаптере) |
| [**A**](#a---always-handle-errors) | Always handle errors | Ошибку обрабатывают или передают выше, но не теряют | [A1](#a1-ошибку-обрабатывают-или-передают-выше) [A2](#a2-баг-не-ловят) |
| [**N**](#n---needed-now) | Needed now | Только то, что нужно сейчас | [N1](#n1-только-то-что-нужно-сейчас) [N2](#n2-интерфейс---только-там-где-вторая-реализация-реальна) |
| [**C**](#c---contract) | Contract | Контракт записан, выполняется и проверен тестом | [C1](#c1-контракт-записан-в-публичном-api) [C2](#c2-каждая-реализация-выполняет-весь-контракт) [C3](#c3-тест-проверяет-контракт-через-публичный-api) |
| [**E**](#e---evident) | Evident | Код читается без сюрпризов | [E1](#e1-имя-говорит-что-это-и-зачем) [E2](#e2-одно-и-то-же-делается-одинаково) [E3](#e3-комментарий-объясняет-только-почему) |

Код сложен, когда правку приходится вносить во многих местах, когда для неё нужно многое держать в голове или когда непонятно, что ещё она сломает. Каждое правило уменьшает что-то из этих трёх. Если правило в конкретном месте делает код сложнее, а не проще, оно не применяется.

> **Пример:** справочнику, который только читает и пишет строки таблицы, адаптер и отдельный слой ничего не дают (I1).

Правила написаны для приложений с бизнес-логикой и вводом-выводом: сервисов, бэкендов, консольных утилит. В библиотеке, скрипте или пользовательском интерфейсе идеи те же, но обработчиков, адаптеров и сценариев там может не быть.

В документ входят только правила, которые независимо записаны в руководствах нескольких экосистем и не противоречат опубликованным исследованиям. Основания каждого правила собраны в разделе [На чём стоят правила](#на-чём-стоят-правила).

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
| **Точка входа** (entry point, `main`) | Место запуска: единственное, где читается конфигурация, создаются адаптеры и передаются в бизнес-логику | `main.py` |
| **Доменный тип** | Тип, которым владеет бизнес-логика: сущность, объект-значение, наше исключение. Не зависит от инфраструктуры | `Order`, `Receipt`, `PaymentDeclined` |
| **Объект-значение** (value object) | Неизменяемый тип данных, который проверяет себя при создании | `PostalCode`, `Money` |
| **Инвариант** (invariant) | Условие, которое объект поддерживает всегда | "отгружен только оплаченный заказ" |
| **Фейк** (fake) | Простая рабочая реализация для тестов. Не путать с моком, который только проверяет вызовы | `InMemoryOrders` |
| **Композиция** (composition) | Класс хранит другой объект и вызывает его, вместо наследования | `OrderLines` хранит список |

### Как читать примеры

Принципы не зависят от языка. Примеры записаны на Python только как на нотации: классы, методы, `if`, `for`, `raise`, `return`. Возможности, которых нет в других языках, не используются.

| В примерах | Что значит |
|---|---|
| `class Orders(Interface)` | интерфейс: только методы, без кода |
| `class CashOnDelivery(PaymentMethod)` | класс реализует интерфейс |
| `Order \| None` | значения может не быть |
| `self._status` | приватное поле |
| `raise SomeError(...)` | бросить исключение; в языке без исключений - вернуть ошибку |
| `# ...` над методом | документация метода |
| `...` | код опущен |

Короткий пример показывает только то, что относится к правилу: конструкторы, обработчики и остальной код опущены. Как всё собирается вместе, видно в [полном примере](#полный-пример-оплата-заказа). Пример стоит сразу после утверждения, которое он поясняет.

Типы магазина (`Order`, `Cart`, `Money` с методами `rub`, `plus`, `minus_percent`, `is_at_least` и другие) считаются уже определёнными. В языке без наследования часть C2 про наследников не нужна.

Вместо конкретных библиотек и фреймворков в примерах условные типы. Они есть в любом языке под своими именами:

| Условный тип | Что умеет |
|---|---|
| `HttpRequest`, `HttpResponse` | входящий запрос и ответ. `request.path_param(name)` - параметр пути, `request.json_string(name)` - строковое поле тела или `None`. `HttpResponse(status, body)` |
| `HttpClient` | исходящие HTTP-запросы. `post(path, body, headers)` возвращает `HttpResult` с полями `status` и `body` (разобранный JSON или `None`). Сбой сети или таймаут -> `NetworkError` |
| `Database` | клиент Postgres. `fetch_one(sql, params)` возвращает строку как словарь "колонка -> значение" или `None`, `execute(sql, params)` выполняет запрос |
| `RedisClient` | клиент Redis. `hgetall(key)` читает хеш целиком, `hset(key, field, value)` пишет одно поле |
| `Router` | любой HTTP-сервер или фреймворк |
| `Mock`, `expect_error` | мок из любой библиотеки моков; проверка, что функция бросает нужное исключение |
| `copy_of(x)` | независимая копия объекта |
| `env(name)`, `today()` | переменная окружения; текущая дата |

---

## D - Deep

### D1. Публичный API проще внутреннего устройства

Функция, класс или модуль забирает сложность себе: вызывающий код делает один простой вызов, а всю работу выполняет внутреннее устройство. Новую функцию или класс выделяют, только если они что-то скрывают и их публичный API проще кода, который они заменяют.

```python
# Плохо: вызывающий код знает протокол эквайринга и порядок шагов
intent_id = acquirer.create_intent(order.total())
acquirer.attach_card(intent_id, card_token)
acquirer.confirm(intent_id)
payment_id = acquirer.capture(intent_id)
```

```python
# Хорошо: один вызов, шаги спрятаны внутри
class HttpAcquirer:
    # Списывает сумму целиком или не списывает ничего.
    # Повторный вызов с тем же idempotency_key не списывает второй раз.
    # Отказ банка -> PaymentDeclined. Сбой связи или сервиса -> PaymentUnavailable.
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt: ...

receipt = acquirer.charge(order.total(), card_token, idempotency_key=order.id())
```

Длина кода сама по себе не повод делить.

```python
# Плохо: чтобы понять одно правило, читают три функции
def price(order: Order) -> Money:
    if is_big(order):
        return with_discount(order)
    return order.total()

def is_big(order: Order) -> bool:
    return order.total().is_at_least(Money.rub(10000))

def with_discount(order: Order) -> Money:
    return order.total().minus_percent(10)
```

```python
# Хорошо: правило целиком в одном месте
def price_after_volume_discount(order: Order) -> Money:
    if order.total().is_at_least(BIG_ORDER_FROM):
        return order.total().minus_percent(BIG_ORDER_DISCOUNT_PERCENT)
    return order.total()
```

### D2. Меньше особых случаев

Контракт строят так, чтобы вызывающему коду нечего было обрабатывать отдельно. Если цель вызова уже достигнута, это успех, а не ошибка.

```python
# Плохо: повторное удаление (двойной клик в браузере) - ошибка, которую ловит каждый
class Cart:
    def remove(self, sku: str) -> None:
        if sku not in self._items:
            raise ProductNotInCart(sku)
        self._items.remove(sku)
```

```python
# Хорошо: ловить нечего
class Cart:
    # После вызова товара sku в корзине нет, был он там или нет.
    def remove(self, sku: str) -> None:
        if sku in self._items:
            self._items.remove(sku)
```

Ситуации, на которые вызывающий код обязан реагировать (отказ банка, нехватка товара), не прячут.

```python
# Плохо: банк отказал, а pay вернулся как при успехе - неоплаченный заказ уйдёт на сборку
def pay(self, order: Order, card_token: str) -> None:
    try:
        receipt = self._acquirer.charge(order.total(), card_token, order.id())
    except PaymentDeclined:
        return
    order.mark_paid(receipt)
```

```python
# Хорошо: отказ банка доходит до вызывающего
def pay(self, order: Order, card_token: str) -> None:
    receipt = self._acquirer.charge(order.total(), card_token, order.id())
    order.mark_paid(receipt)
```

### D3. Каждому вызывающему - узкий интерфейс

Если интерфейс уже нужен (N2) и его используют разные места, каждое из них зависит только от методов, которыми пользуется.

```python
# Плохо: отчёту нужен один метод, а зависит он от пяти
class OrderStore(Interface):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...
    def delete(self, order_id: str) -> None: ...
    def paid_between(self, start: date, end: date) -> list[Order]: ...
    def lock_for_update(self, order_id: str) -> None: ...

def revenue(store: OrderStore, start: date, end: date) -> Money: ...
```

```python
# Хорошо
class PaidOrders(Interface):
    def paid_between(self, start: date, end: date) -> list[Order]: ...

def revenue(orders: PaidOrders, start: date, end: date) -> Money: ...
```

Если вызывающий код один, интерфейс не дробят.

> **Пример:** пока `Orders` использует один сценарий, `get` и `save` остаются в одном интерфейсе.

---

## I - Inward

### I1. Бизнес-логика не зависит от инфраструктуры

Инфраструктура - всё, с чем код общается за пределами своей памяти: хранилища, внешние сервисы, очереди, файловая система, почта, часы, генератор случайных чисел. У каждой такой системы свой способ общения: язык запросов, протокол, формат данных, библиотека, коды ошибок. Этот способ знает только адаптер.

Адаптер нужен не ради смены внешней системы: её меняют редко. Он нужен потому, что постоянно меняется её устройство (схема, запросы, ключи, библиотека, формат, коды ошибок), и такие изменения должны затрагивать только адаптер.

Публичный API адаптера выражен в терминах бизнеса: что сделать и с каким объектом, а не как это делает внешняя система. Бизнес-логика вызывает адаптер и не знает, какая система за ним и как с ней говорят.

**Примеры API адаптера в терминах бизнеса:**
- `orders.save(order)` вместо SQL-запроса;
- `carts.get(customer_id)` вместо команд Redis;
- `shipping.ship(parcel)` вместо HTTP-вызова;
- `events.publish(order_paid)` вместо записи байтов в очередь;
- функция `now()` в параметре вместо чтения системных часов внутри правила.

Адаптер передают через конструктор (внедрение зависимостей) и создают в точке входа.

```python
# Плохо: правило "не больше 20 позиций" перемешано с ключами и командами Redis
def add_to_cart(redis: RedisClient, customer_id: str, sku: str, quantity: int) -> None:
    key = "cart:" + customer_id
    items = redis.hgetall(key)
    if len(items) >= 20 and sku not in items:
        raise CartFull(customer_id)
    redis.hset(key, sku, int(items.get(sku, 0)) + quantity)
```

```python
# Хорошо: правило живёт в Cart, Redis знает только адаптер
class Shopping:
    def __init__(self, carts: RedisCarts) -> None:
        self._carts = carts

    def add_to_cart(self, customer_id: str, sku: str, quantity: int) -> None:
        cart = self._carts.get(customer_id)
        cart.add(sku, quantity)  # бросает CartFull, если позиций уже 20
        self._carts.save(cart)

class RedisCarts:  # адаптер: ключи, команды и формат Redis - только здесь
    def get(self, customer_id: str) -> Cart: ...
    def save(self, cart: Cart) -> None: ...

# main.py
shopping = Shopping(RedisCarts(RedisClient(env("REDIS_URL"))))
```

Инфраструктуру с одной операцией передают параметром функции.

> **Пример:** `is_expired(coupon, now)` получает текущее время аргументом, а не читает часы сама.

Правило действует там, где есть бизнес-правила: адаптер защищает их от устройства внешней системы. Коду, который только перекладывает данные между запросом и хранилищем и ничего не решает, адаптер и отдельный слой не нужны.

> **Пример:** эндпоинт админки, который показывает список городов доставки, читает таблицу прямо в обработчике. Когда появится правило ("нельзя удалить город, в который есть активные доставки"), его выносят.

### I2. Пакеты без циклов

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

> **Пример второго способа:** `orders` должен уведомить покупателя, а `notifications` нужны данные заказа. Интерфейс `OrderNotifier` объявляют в `orders`, реализуют в `notifications`, и `orders` больше не импортирует `notifications`.

---

## S - Single owner

### S1. У каждого знания один владелец

Знание - это решение, которое могло быть другим: бизнес-правило, формат, алгоритм, выбор поставщика. Код делят по знаниям, а не по шагам выполнения. Каждое знание живёт ровно в одном месте (функции, классе или модуле), знания, которые меняются вместе, лежат вместе.

```python
# Плохо: "бесплатно от 5000" живёт в двух местах
# orders/checkout.py
def shipping_cost(order_total: Money) -> Money:
    if order_total.is_at_least(Money.rub(5000)):
        return Money.rub(0)
    return Money.rub(300)

# http/cart_handler.py
free_shipping = cart.total().is_at_least(Money.rub(5000))
```

```python
# Хорошо: знание принадлежит shipping/tariff.py
FREE_SHIPPING_FROM = Money.rub(5000)

def ships_free(order_total: Money) -> bool:
    return order_total.is_at_least(FREE_SHIPPING_FROM)

def shipping_cost(order_total: Money) -> Money:
    if ships_free(order_total):
        return Money.rub(0)
    return Money.rub(300)

# http/cart_handler.py
free_shipping = ships_free(cart.total())
```

Похожий код, который меняется по разным причинам, - это два знания, и их не объединяют.

```python
# Плохо: два решения склеены, потому что сегодня числа совпали
MAX_ITEMS = 20  # и позиций в корзине, и строк на странице отчёта
```

```python
# Хорошо: лимит корзины меняет бизнес, размер страницы - дизайн отчёта
MAX_CART_ITEMS = 20
REPORT_PAGE_SIZE = 20
```

### S2. Инвариант меняет только владелец

Состояние с инвариантом закрыто внутри объекта-владельца. Снаружи его меняют только методами владельца, и владелец проверяет переход. Вызывающий код говорит, что сделать, а не решает за владельца.

```python
# Плохо: статус пишут снаружи, правило перехода обходится
order.status = OrderStatus.SHIPPED  # так можно "отгрузить" неоплаченный заказ
orders.save(order)
```

```python
# Хорошо: переход проверяет сам Order
class Order:
    def ship(self, tracking_number: str) -> None:
        if self._status != OrderStatus.PAID:
            raise OrderCannotBeShipped(self._id)
        self._status = OrderStatus.SHIPPED
        self._tracking_number = tracking_number

order.ship(tracking_number)
orders.save(order)
```

### S3. Меньше изменяемого состояния

Чем меньше в программе состояния, которое можно изменить, тем меньше положений, в которых она может оказаться. Глобального изменяемого состояния нет: всё, от чего зависит результат, приходит параметром или через конструктор. Данные, которым незачем меняться, неизменяемы.

```python
# Плохо: скидку меняет любой код в любой момент, по вызову price этого не видно
current_discount_percent = 0

def price(order: Order) -> Money:
    return order.total().minus_percent(current_discount_percent)
```

```python
# Хорошо: всё, от чего зависит результат, видно в параметрах
def price(order: Order, discount_percent: int) -> Money:
    return order.total().minus_percent(discount_percent)
```

---

## T - Translate at the boundary

### T1. Чужие данные разбираются в объекты-значения на границе

Внешний формат (JSON запроса, строку таблицы, ответ внешнего API, значение из Redis) один раз разбирают в объекты-значения, которые гарантируют корректность. Входящие запросы разбирает обработчик, ответы инфраструктуры - адаптер. Бизнес-логика данные повторно не проверяет и внешний формат не читает.

```python
# Плохо: сырой JSON уходит в бизнес-логику, и каждая функция проверяет его заново
class Delivery:
    def set_address(self, order_id: str, payload: dict) -> None:
        if len(payload["zip"]) != 6:
            raise InvalidAddress(payload["zip"])
        ...

    def estimate_days(self, payload: dict) -> int:
        if len(payload["zip"]) != 6:
            raise InvalidAddress(payload["zip"])
        ...
```

```python
# Хорошо: проверка одна, в объекте-значении; внутрь уходит только он
class PostalCode:  # создан - значит корректен; после создания не меняется
    def __init__(self, value: str) -> None:
        if len(value) != 6 or not value.isdigit():
            raise InvalidAddress("индекс должен состоять из 6 цифр: " + value)
        self.value = value

class Delivery:
    def set_address(self, order_id: str, postal_code: PostalCode) -> None: ...
    def estimate_days(self, postal_code: PostalCode) -> int: ...

# обработчик: строит PostalCode из запроса, на InvalidAddress отвечает 422
delivery.set_address(order_id, PostalCode(zip_from_request))
```

### T2. Чужие ошибки переводятся в свои в адаптере

Ошибку библиотеки или внешнего API, на которую вызывающий код может отреагировать (отказ, недоступность сервиса), адаптер один раз переводит в собственное исключение. Бизнес-логика видит только его. Остальные ошибки адаптер не трогает (A2). В обратную сторону так же: ошибки бизнес-логики в ответ клиенту (HTTP-статус, код выхода, сообщение) переводит обработчик, и только он.

```python
# Плохо: чужой ответ и чужая ошибка выходят из адаптера в бизнес-логику
class Fulfillment:
    def ship(self, order: Order) -> None:
        try:
            result = self._shipping.ship(order.parcel())  # возвращает HttpResult
        except NetworkError:
            order.postpone_shipping()
            return
        if result.status != 201:
            order.postpone_shipping()
            return
        order.ship(result.body["tracking"])
```

```python
# Хорошо: адаптер возвращает трек-номер или бросает своё исключение
class CourierApiShipping:
    # Передаёт посылку курьеру, возвращает трек-номер. Сбой или отказ -> ShippingUnavailable.
    def ship(self, parcel: Parcel) -> str:
        try:
            result = self._http.post("/v2/shipments", parcel_to_json(parcel), headers={})
        except NetworkError as error:
            raise ShippingUnavailable(error)
        if result.status != 201:
            raise ShippingUnavailable(result.status)
        return result.body["tracking"]

class Fulfillment:
    def ship(self, order: Order) -> None:
        try:
            order.ship(self._shipping.ship(order.parcel()))
        except ShippingUnavailable:
            order.postpone_shipping()
```

---

## A - Always handle errors

### A1. Ошибку обрабатывают или передают выше

Ошибку нельзя потерять. Код либо делает с ней что-то осмысленное, либо отдаёт её вызывающему. Пустых обработчиков нет. Ловят только ту ошибку, которую ждут и с которой знают, что делать: перехват "всего подряд" прячет и ошибки, о которых автор не думал.

```python
# Плохо: ошибка проглочена, чек не отправлен, и следа нигде нет
try:
    mailer.send_receipt(order)
except Exception:
    pass
```

```python
# Хорошо: ждём одну известную ошибку и что-то с ней делаем, остальное уходит выше
try:
    mailer.send_receipt(order)
except MailUnavailable:
    outbox.retry_later(order.id())
```

### A2. Баг не ловят

Баг (нарушенный контракт, невозможное состояние) и сбой, на который нечем ответить (база недоступна), не превращают в обычный результат. Они как есть доходят до единого верхнего обработчика, который пишет их в лог и отвечает 500. Так баг виден сразу и там, где возник, а не позже в неверных данных.

```python
# Плохо: баг спрятан, покупатель видит нулевую сумму
def total(order: Order) -> Money:
    try:
        return sum_lines(order.lines())
    except Exception:
        return Money.rub(0)
```

```python
# Хорошо: если в sum_lines баг, запрос упадёт, а в логе будет причина
def total(order: Order) -> Money:
    return sum_lines(order.lines())
```

В одном вызове встречаются все три вида ошибок: ожидаемая (A1), переведённая адаптером (T2) и та, которую не ловят (A2).

```python
def ship(self, order: Order) -> None:
    try:
        tracking = self._shipping.ship(order.parcel())
    except ShippingUnavailable:  # её ждут и знают, что делать (A1); своей её сделал адаптер (T2)
        order.postpone_shipping()
        return
    order.ship(tracking)  # баг или упавшая база здесь не ловятся и уходят выше (A2)
```

---

## N - Needed now

### N1. Только то, что нужно сейчас

Не добавляют возможностей, параметров, настроек и точек расширения, которых не требует текущая задача. "Может пригодиться" - не причина.

```python
# Плохо: гибкость "на будущее"
class CourierApiShipping:
    def ship(self, parcel: Parcel, priority: int = 0, split_by_warehouse: bool = False,
             callback_url: str | None = None) -> str: ...
```

```python
# Хорошо
class CourierApiShipping:
    def ship(self, parcel: Parcel) -> str: ...
```

Обобщают форму публичного API, а не набор возможностей.

> **Пример:** `ship(parcel)` подходит любому отправлению, а `ship_vip_order(order)` придётся повторять для каждого нового случая.

Заранее продумывают только то, что дорого изменить потом: формат хранимых данных и наш HTTP API для внешних потребителей.

> **Пример:** сумму сразу хранят вместе с валютой: добавить валюту в миллионы уже записанных строк дорого, а переименовать метод дёшево.

### N2. Интерфейс - только там, где вторая реализация реальна

Интерфейс вводят, только когда реализаций реально больше одной:
- фейк для тестов;
- второй поставщик или вторая база;
- несколько вариантов поведения уже сейчас, например способов оплаты;
- замена, которую уже решили сделать.

"Может пригодиться" - не причина. Пока реализация одна, вызывающий код зависит от класса напрямую. Когда появляется вторая, интерфейс выделяют из готового класса, и это механическая правка: вызывающий код и так пользуется только его публичным API. На практике вторая реализация появляется быстро: сценарий тестируют на фейке (C3), поэтому его адаптеры получают интерфейс с первым тестом. Входящей стороне интерфейс не нужен: запрос пользователя или внешнего клиента принимает обработчик и сам вызывает бизнес-логику (T1).

```python
# Плохо: интерфейс ради единственной реализации
class DiscountPolicy(Interface):
    def apply(self, order: Order, customer: Customer) -> Money: ...

class DefaultDiscountPolicy(DiscountPolicy):  # другой нет и не планируется
    def apply(self, order: Order, customer: Customer) -> Money: ...
```

```python
# Хорошо: обычная функция, её вызывают напрямую
def price_after_loyalty_discount(order: Order, customer: Customer) -> Money: ...
```

```python
# Хорошо: у PostgresOrders появился фейк для тестов - интерфейс выделили
class Orders(Interface):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class PostgresOrders(Orders): ...
class InMemoryOrders(Orders): ...
```

---

## C - Contract

### C1. Контракт записан в публичном API

Всё, что вызывающий код должен знать и чего не видно из имени и типов, записано в документации метода: поведение в особых случаях, единицы, порядок вызовов, исключения.

```python
# Плохо: что будет при нехватке и как снять резерв - неизвестно
def reserve(self, sku: str, quantity: int) -> bool: ...
```

```python
# Хорошо
# Резервирует quantity единиц sku под заказ.
# Если свободного остатка не хватает, возвращает False и ничего не меняет.
# Резерв снимает release() или превращает в списание ship().
def reserve(self, sku: str, quantity: int) -> bool: ...
```

### C2. Каждая реализация выполняет весь контракт

Реализация не требует от вызывающего кода больше, чем интерфейс, не обещает меньше и не бросает исключений вне контракта. Если реализация не может выполнить контракт, меняют контракт или делят интерфейс.

```python
# Плохо: реализация отказывается от части контракта
class CashOnDelivery(PaymentMethod):
    def refund(self, receipt: Receipt) -> None:
        raise NotSupported("наличные автоматически не возвращаются")
```

```python
# Хорошо: контракт честно описывает оба исхода
class PaymentMethod(Interface):
    # Возвращает деньги сразу (REFUNDED) или ставит задачу оператору (OPERATOR_TASK_CREATED).
    def refund(self, receipt: Receipt) -> RefundResult: ...

class CashOnDelivery(PaymentMethod):
    def refund(self, receipt: Receipt) -> RefundResult:
        self._tasks.return_cash(receipt)
        return RefundResult.OPERATOR_TASK_CREATED
```

Наследник - тоже реализация. Наследовать класс можно, только когда наследник полностью выполняет контракт родителя, а родитель создан для наследования.

> **Пример:** свои исключения наследуют от общего класса ошибок: он для этого и существует.

Иначе чужое поведение используют, храня объект в поле и вызывая его (композиция).

```python
# Плохо: наследник списка получает все его методы, и они обходят правило add
class OrderLines(list):
    def add(self, line: OrderLine) -> None: ...  # склеивает строки с одинаковым sku

lines.append(line)  # унаследованный метод: одна позиция теперь в двух строках
```

```python
# Хорошо: список спрятан в поле, снаружи доступен только add
class OrderLines:
    def __init__(self) -> None:
        self._lines = []

    def add(self, line: OrderLine) -> None: ...  # склеивает строки с одинаковым sku
```

### C3. Тест проверяет контракт через публичный API

Тест вызывает код так же, как вызывающий код, и проверяет результат. Приватные поля и порядок внутренних вызовов тест не проверяет. Инфраструктуру заменяют фейком (N2), а не моками.

```python
# Плохо: тест привязан к внутреннему устройству
def test_place_order() -> None:
    orders = Mock()
    checkout = Checkout(orders)
    checkout.place_order(cart)
    assert orders.save.was_called_once()
    assert checkout._last_order.status == "new"
```

```python
# Хорошо: тест действует как вызывающий код и смотрит на результат
def test_placed_order_is_saved_as_new() -> None:
    orders = InMemoryOrders()  # фейк: хранит заказы в памяти
    order_id = Checkout(orders).place_order(cart)
    assert orders.get(order_id).status() == OrderStatus.NEW
```

Свой код, который работает быстро и без внешних систем, в тесте не подменяют.

> **Пример:** `Order` и `Money` в тесте настоящие, подменяется только хранилище заказов.

Фейк и настоящий адаптер проверяют одним и тем же тестом контракта интерфейса, иначе фейк расходится с настоящим.

```python
def check_orders_contract(orders: Orders) -> None:
    assert orders.get("no-such-id") is None
    orders.save(new_order("o-1"))
    assert orders.get("o-1").status() == OrderStatus.NEW

def test_in_memory_orders_keep_the_contract() -> None:
    check_orders_contract(InMemoryOrders())

def test_postgres_orders_keep_the_contract() -> None:  # медленный: нужна настоящая база
    check_orders_contract(PostgresOrders(test_database()))
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
        t = t.minus_percent(5)
    return t
```

```python
# Хорошо
GOLD_LEVEL = 3
GOLD_DISCOUNT_PERCENT = 5

def price_after_loyalty_discount(order: Order, customer: Customer) -> Money:
    if customer.loyalty_level() >= GOLD_LEVEL:
        return order.total().minus_percent(GOLD_DISCOUNT_PERCENT)
    return order.total()
```

### E2. Одно и то же делается одинаково

Одна задача решается в системе одним способом: одинаковые имена методов, один способ сказать "нет значения" и "ошибка". Новый код следует принятому способу.

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

Если принятый способ нарушает другое правило, новый код всё равно следует ему, а расхождение называют явно: менять архитектуру решает команда, а не автор одной правки. Исключения - A1 и T1: новый код передаёт ошибки выше и проверяет чужой ввод, как бы ни было принято в проекте.

```python
# В проекте все адаптеры на "не найдено" бросают исключение
class PostgresCustomers:
    def get(self, customer_id: str) -> Customer: ...  # бросает CustomerNotFound

# Плохо: по D2 удобнее None, но теперь в проекте два способа сказать "не найдено"
class PostgresShipments:
    def get(self, shipment_id: str) -> Shipment | None: ...
```

```python
# Хорошо: как принято; несогласие высказано команде, а не в коде
class PostgresShipments:
    def get(self, shipment_id: str) -> Shipment: ...  # бросает ShipmentNotFound
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
  payments/           # [I2] о заказах не знает
    receipt.py        # Receipt и ошибки оплаты (доменные типы)
    http_acquirer.py  # HttpAcquirer (адаптер)
  orders/             # [I2] знает payments, обратной зависимости нет
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
   |       уже PAID ......................... заказ как есть -> 200
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

Стрелка - "импортирует". Доменные типы не импортируют ничего, кроме друг друга; клиенты базы и HTTP (`Database`, `HttpClient`) импортируют только адаптеры и `main.py`. В Python и Go адаптер соответствует интерфейсу, не импортируя его; в Java, C# или Rust он называет интерфейс явно, и этот импорт тоже направлен внутрь.

```
[PayOrderHandler] -------> {OrderPayment} -------> <Order>, <Receipt>
 http/pay_handler           orders/payment          orders/order, payments/receipt
                                  ^                        ^
                                  | реализуют              |
                                  | интерфейсы Orders      |
                                  | и Acquirer             |
[PostgresOrders], [HttpAcquirer] -+------------------------+
 orders/postgres, payments/http_acquirer

main.py -------> всё: создаёт адаптеры и передаёт их в сценарий
```

### Код

Метка вида `# [S2]` показывает место, где в коде видно правило. Это пометки для читателя документа: в рабочем коде их нет.

**payments/receipt.py** - доменные типы оплаты. Их импортирует сценарий, поэтому здесь нет ничего из инфраструктуры.

```python
class Receipt:  # [S3] объект-значение: после создания не меняется
    def __init__(self, payment_id: str, amount: Money) -> None:
        self.payment_id = payment_id
        self.amount = amount

# Банк отказал. Повтор с той же картой не поможет.
class PaymentDeclined(Exception): ...

# Эквайринг недоступен. Можно повторить позже.
class PaymentUnavailable(Exception): ...
```

**payments/http_acquirer.py** - адаптер к эквайрингу. Только он знает URL, поля запроса и коды ответов.

```python
class HttpAcquirer:
    def __init__(self, http: HttpClient) -> None:
        self._http = http

    # [D1] один вызов прячет весь протокол эквайринга
    # [C1] контракт записан над методом:
    # Списывает сумму целиком или не списывает ничего.
    # Повторный вызов с тем же idempotency_key не списывает второй раз.
    # Отказ банка -> PaymentDeclined. Сбой связи или сервиса -> PaymentUnavailable.
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        try:
            result = self._http.post(
                "/v1/charges",
                {"amount": str(amount.amount), "currency": amount.currency, "card": card_token},
                headers={"Idempotency-Key": idempotency_key},
            )
        except NetworkError as error:
            raise PaymentUnavailable(error)  # [T2] чужая ошибка -> наша
        if result.status == 402:
            raise PaymentDeclined()
        if result.status != 200 or result.body is None or "id" not in result.body:
            raise PaymentUnavailable()
        return Receipt(result.body["id"], amount)  # [T1] чужой JSON -> наш Receipt
```

**orders/order.py** - сущность. Заказ сам следит, что оплатить можно только новый заказ.

```python
class OrderStatus(Enum):
    NEW = "new"
    PAID = "paid"
    CANCELLED = "cancelled"

class OrderNotFound(Exception): ...

# Заказ уже оплачен или отменён.
class OrderNotPayable(Exception): ...

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

    # [S2] правило "оплатить можно только новый заказ" живёт только здесь
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
class Orders(Interface):  # [N2] интерфейс есть, потому что есть вторая реализация - фейк
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class Acquirer(Interface):  # [D3] только то, чем пользуется сценарий
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt: ...

class OrderPayment:
    def __init__(self, orders: Orders, acquirer: Acquirer) -> None:  # [I1] адаптеры приходят снаружи
        self._orders = orders
        self._acquirer = acquirer

    # Оплачивает заказ картой.
    # Уже оплаченный заказ возвращается как есть, без повторного списания.
    # Ошибки: OrderNotFound, OrderNotPayable, PaymentDeclined, PaymentUnavailable.
    def pay(self, order_id: str, card_token: str) -> Order:
        order = self._orders.get(order_id)
        if order is None:
            raise OrderNotFound(order_id)
        if order.status() == OrderStatus.PAID:
            return order  # [D2] повтор уже выполненного запроса - успех, а не ошибка
        order.ensure_payable()
        # [E3] Ключ - id заказа: повтор после сбоя записи не спишет деньги дважды.
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
        # [T1] строка таблицы -> Order
        return Order(
            row["id"],
            Money(row["total_amount"], row["currency"]),
            OrderStatus(row["status"]),
            row["payment_id"],
        )

    def save(self, order: Order) -> None:
        # [A2] сбой базы здесь не ловят: он дойдёт до верхнего обработчика
        self._db.execute(
            "UPDATE orders SET status = $1, payment_id = $2 WHERE id = $3",
            [order.status().value, order.payment_id(), order.id()],
        )
```

**http/pay_handler.py** - обработчик. Разбирает запрос, вызывает бизнес-логику, переводит её ошибки в HTTP-статусы. Сам ничего не решает.

```python
# POST /orders/{order_id}/pay  {"card_token": "..."}
class PayOrderHandler:
    def __init__(self, payment: OrderPayment) -> None:
        self._payment = payment

    def handle(self, request: HttpRequest) -> HttpResponse:
        card_token = request.json_string("card_token")  # [T1] ввод разбирается на границе
        if card_token is None:
            return HttpResponse(400, {"error": "нужен card_token"})
        try:
            order = self._payment.pay(request.path_param("order_id"), card_token)
        except OrderNotFound:  # [T2] наши ошибки -> HTTP-статусы, только здесь
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
# [I1] единственное место, где создаются адаптеры
db = Database(env("DATABASE_URL"))
acquirer = HttpAcquirer(HttpClient(base_url=env("ACQUIRER_URL"), timeout_seconds=10))
payment = OrderPayment(PostgresOrders(db), acquirer)

router = Router()
router.post("/orders/{order_id}/pay", PayOrderHandler(payment).handle)
router.serve(port=8080)
```

**tests/test_order_payment.py** - тесты бизнес-логики на фейках, без базы и сети.

```python
class InMemoryOrders:  # [C3] фейк: рабочая реализация в памяти, а не мок
    def __init__(self) -> None:
        self._orders = {}  # id заказа -> Order

    def get(self, order_id: str) -> Order | None:
        return copy_of(self._orders.get(order_id))

    def save(self, order: Order) -> None:
        # Копия, как в настоящей базе: изменение без save() не должно быть видно.
        self._orders[order.id()] = copy_of(order)

class ApprovingAcquirer:
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        return Receipt("pay-1", amount)

class DecliningAcquirer:  # [C2] фейк не выходит за контракт: бросает только PaymentDeclined
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        raise PaymentDeclined()

def orders_with_new_order() -> InMemoryOrders:
    orders = InMemoryOrders()
    orders.save(Order("o-1", Money.rub(990), OrderStatus.NEW, payment_id=None))
    return orders

# [C3] тесты вызывают pay, как вызывающий код, и смотрят на результат
def test_paid_order_becomes_paid() -> None:
    orders = orders_with_new_order()
    OrderPayment(orders, ApprovingAcquirer()).pay("o-1", "card-token")
    assert orders.get("o-1").status() == OrderStatus.PAID

def test_declined_payment_keeps_order_new() -> None:
    orders = orders_with_new_order()
    payment = OrderPayment(orders, DecliningAcquirer())
    expect_error(PaymentDeclined, lambda: payment.pay("o-1", "card-token"))
    assert orders.get("o-1").status() == OrderStatus.NEW

def test_repeated_payment_does_not_charge_again() -> None:
    orders = orders_with_new_order()
    OrderPayment(orders, ApprovingAcquirer()).pay("o-1", "card-token")
    order = OrderPayment(orders, DecliningAcquirer()).pay("o-1", "card-token")
    assert order.status() == OrderStatus.PAID
```

---

## На чём стоят правила

Правило попадает в документ по двум признакам: его независимо формулируют руководства нескольких несвязанных экосистем, и опубликованные исследования ему не противоречат. Мнение одного автора, даже известного, основанием не считается.

Строго доказанных правил в проектировании кода нет. Почти все исследования показывают связь, а не причину, и часть эффектов слабеет, если учесть размер кода. Поэтому колонка "Данные" говорит только о том, подтверждается ли правило измерениями, а не о том, что оно доказано.

| Правило | Где записано | Данные |
|---|---|---|
| D1 | C++ Core Guidelines C.3, C.9; Effective Java 15; Rust API Guidelines C-STRUCT-PRIVATE; PEP 8; .NET Design Guidelines | Файлы с нестабильным интерфейсом заметно чаще содержат ошибки и чаще меняются (Mo et al. 2019, 19 проектов) |
| D2 | Effective Java 54; .NET Design Guidelines (пустая коллекция вместо null); Go Code Review Comments (In-Band Errors); Azure API Guidelines и AWS Well-Architected (идемпотентность) | 92% катастрофических сбоев в пяти распределённых системах - неверная обработка явно сообщённой ошибки (Yuan et al. 2014) |
| D3 | Go Proverbs; Google Go Style (Interfaces); C++ Core Guidelines T.41; Rust API Guidelines C-GENERIC; .NET Design Guidelines | Интерфейсы, которыми клиенты пользуются по частям, меняются чаще (Romano & Pinzger 2011, 10 систем) |
| I1 | Microsoft .NET Architecture Guides (dependency inversion, persistence ignorance); Azure Architecture Center; AWS Prescriptive Guidance. Возражение: Google Go Style не советует оборачивать клиентов в свои интерфейсы | Сравнительных исследований нет, поэтому правило условное |
| I2 | C++ Core Guidelines SF.9; в Go циклы между пакетами запрещены компилятором | У компонентов в циклах вдвое больше отказов (Zimmermann & Nagappan 2007, Windows Server 2003) |
| S1 | C++ Core Guidelines ES.3; Microsoft .NET Architecture Guides; OWASP ASVS 1.1.6; Linux kernel coding style. Оговорка про похожий код: Go Proverbs, Google Testing Blog | Вредны копии, изменённые несогласованно (Juergens et al. 2009); остальные копии ошибок не добавляют (Rahman et al. 2010; Kapser & Godfrey 2008) |
| S2 | C++ Core Guidelines C.2, C.40; Effective Java 16; Rust API Guidelines C-VALIDATE; .NET Design Guidelines | Прямых исследований мало |
| S3 | C++ Core Guidelines I.2, Con.1; Effective Java 17; Google Python Style; NASA/JPL Power of Ten, правило 6; Twelve-Factor App | 97% ошибок конкурентности, не считая взаимных блокировок, - нарушение атомарности или порядка доступа к общему состоянию (Lu et al. 2008) |
| T1 | OWASP Input Validation Cheat Sheet; OWASP ASVS 5.1; SEI CERT MET00-J; The Rust Book, гл. 9; Effective Java 49; C++ Core Guidelines I.4 | Уязвимости от непроверенного ввода занимают 1, 3, 5 и 7 места в CWE Top 25 за 2024 год |
| T2 | Effective Java 73; The Rust Book, гл. 9; .NET Design Guidelines; Google Go Style; Google AIP-193; C++ Core Guidelines I.10 | Прямых исследований нет |
| A1 | Effective Java 77; Go Code Review Comments (Handle Errors); PEP 8 и PEP 20; .NET CA1031; SEI CERT ERR00-J | 92% катастрофических сбоев в пяти распределённых системах - неверная обработка явно сообщённой ошибки; в четверти случаев её просто проигнорировали (Yuan et al. 2014) |
| A2 | Effective Java 70; The Rust Book, гл. 9; C++ Core Guidelines E.17; Go Code Review Comments (Don't Panic); .NET Design Guidelines | Перехват слишком общего исключения связан с ошибками в части проектов (de Padua & Shang 2018) |
| N1 | Google Engineering Practices; Google Testing Blog; C++ Core Guidelines Per.7; Google Go Style (least mechanism) | Большинство пользователей задаёт только 6-17% параметров настройки (Xu et al. 2015) |
| N2 | Go Code Review Comments; Google Go Style; .NET Design Guidelines; C++ Core Guidelines T.1 | Прямых исследований нет |
| C1 | C++ Core Guidelines I.5, I.7; Effective Java 56, 74; Rust API Guidelines C-FAILURE; .NET Design Guidelines; Google Go Style; AWS Well-Architected | Расхождение документации с кодом бывает ошибкой (Tan et al. 2007) |
| C2 | C++ Core Guidelines C.120; Google C++ Style; Effective Java 10, 18, 19; .NET Design Guidelines; в Go и Rust наследования нет | В файлах с нарушенным наследованием на 35-503% больше ошибок (Mo et al. 2019); эксперименты о глубине наследования противоречат друг другу (Daly et al. 1996; Prechelt et al. 2003) |
| C3 | Software Engineering at Google, гл. 12-13; Google Testing Blog. Это одна экосистема | 51% правок моков вынуждены изменением рабочего кода (Spadini et al. 2019) |
| E1 | C++ Core Guidelines ES.45, NL.19; .NET Design Guidelines; SEI CERT DCL06-C; Linux kernel coding style | Со словами вместо сокращений ошибки находят на 14-19% быстрее (Hofmeister et al. 2017; Schankin et al. 2018) |
| E2 | PEP 8; PEP 20; Google Style Guides; C++ Core Guidelines NL.4; Software Engineering at Google, гл. 8 | Необычный для своего проекта код чаще содержит ошибки (Ray et al. 2016) |
| E3 | C++ Core Guidelines NL.1, NL.2; Google Style Guides; Google Engineering Practices; Linux kernel coding style | Прямых исследований мало |

### Чего здесь нет и почему

| Идея | Почему не вошла |
|---|---|
| Закон Деметры | одно руководство и одно исследование, эффект почти исчезает с учётом размера кода |
| Решение отдельно от побочных эффектов | одно руководство, исследований нет |
| Полиморфизм вместо ветвления | в экспериментах простое решение иногда лучше; ветвление по типу на число ошибок не влияет (Prechelt et al. 2001; Hall et al. 2014) |
| Закрепить поведение тестом перед правкой | в руководствах не найдено, исследований нет |
| Тест раньше кода (TDD) | порядок "тест до или после" на результат не влияет (Fucci et al. 2017) |
| Лимит длины функции | руководства расходятся, исследования противоречат друг другу |
| Пакеты по фичам, а не по слоям | в руководствах не найдено, сравнительных исследований нет |

### Источники

Руководства: [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines), [Effective Java](https://www.informit.com/store/effective-java-9780134686097), [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments), [Go Proverbs](https://go-proverbs.github.io), [Google Go Style](https://google.github.io/styleguide/go/decisions), [Google C++ Style](https://google.github.io/styleguide/cppguide.html), [Google Python Style](https://google.github.io/styleguide/pyguide.html), [Google Engineering Practices](https://google.github.io/eng-practices/review/), [Software Engineering at Google](https://abseil.io/resources/swe-book), [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html), [The Rust Book, гл. 9](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html), [.NET Design Guidelines](https://learn.microsoft.com/dotnet/standard/design-guidelines/), [.NET Architecture Guides](https://learn.microsoft.com/dotnet/architecture/modern-web-apps-azure/architectural-principles), [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/guide/design-principles/design-for-evolution), [PEP 8](https://peps.python.org/pep-0008), [PEP 20](https://peps.python.org/pep-0020), [.NET CA1031](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/ca1031), [SEI CERT ERR00-J](https://cmu-sei.github.io/secure-coding-standards/sei-cert-oracle-coding-standard-for-java/rules/exceptional-behavior-err/err00-j), [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html), [CWE Top 25](https://cwe.mitre.org/top25/archive/2024/2024_top25_list), [Linux kernel coding style](https://kernel.org/doc/html/latest/process/coding-style.html), [Twelve-Factor App](https://12factor.net).

Исследования: [Mo et al. 2019](https://www.cs.drexel.edu/~yc349/papers/2019/tse2019.pdf), [Yuan et al. 2014](https://gwern.net/doc/cs/algorithm/2014-yuan.pdf), [Zimmermann & Nagappan 2007](https://thomas-zimmermann.com/publications/files/zimmermann-issre-2007.pdf), [Juergens et al. 2009](https://ar5iv.labs.arxiv.org/html/1701.05472), [Rahman et al. 2010](https://www.microsoft.com/en-us/research/publication/clones-what-is-that-smell/), [Kapser & Godfrey 2008](https://cormack.uwaterloo.ca/~migod/papers/2008/emse08-ClonePatterns.pdf), [Lu et al. 2008](https://www.cs.columbia.edu/~junfeng/10fa-e6998/papers/concurrency-bugs.pdf), [Xu et al. 2015](https://cseweb.ucsd.edu/~tixu/papers/fse15.pdf), [Tan et al. 2007](https://www.cs.purdue.edu/homes/lintan/publications/icomment_sosp07.pdf), [Spadini et al. 2019](https://d-nb.info/1174174781/34), [Hofmeister et al. 2017](https://www.se.cs.uni-saarland.de/publications/docs/HoSeHo17.pdf), [Schankin et al. 2018](https://brains-on-code.github.io/descriptive-compound-identifier-names.pdf), [Ray et al. 2016](https://arxiv.org/abs/1506.01159), [Prechelt et al. 2001](https://ps.ipd.kit.edu/180_778.php), [Prechelt et al. 2003](https://ps.ipd.kit.edu/180_837.php), [Hall et al. 2014](https://uhra.herts.ac.uk/handle/2299/14912), [Fucci et al. 2017](https://arxiv.org/abs/1611.05994), [de Padua & Shang 2018](https://ece.uwaterloo.ca/~wshang/pubs/MSR2018_GUI.pdf).
