# DISTANCE

**English** | [Русский](README.ru.md)

**Keep your distance:** every piece of code knows as little as possible about the others. Then the code is cheap to understand and safe to change.

| | Principle | Essence | Rules |
|---|---|---|---|
| [**D**](#d---deep) | Deep | The public API is simpler than the internals | [D1](#d1-the-public-api-is-simpler-than-the-internals) [D2](#d2-fewer-special-cases) [D3](#d3-a-narrow-interface-for-each-caller) |
| [**I**](#i---inward) | Inward | Dependencies point inward, toward business logic | [I1](#i1-business-logic-does-not-depend-on-infrastructure) [I2](#i2-no-cycles-between-packages) |
| [**S**](#s---single-owner) | Single owner | Knowledge, an invariant and state each have one owner | [S1](#s1-every-piece-of-knowledge-has-one-owner) [S2](#s2-only-the-owner-changes-an-invariant) [S3](#s3-less-mutable-state) |
| [**T**](#t---translate-at-the-boundary) | Translate at the boundary | Foreign data and errors are translated into our own once, at the system boundary | [T1](#t1-foreign-data-is-parsed-into-value-objects-at-the-boundary) [T2](#t2-foreign-errors-are-translated-into-ours-in-the-adapter) |
| [**A**](#a---always-handle-errors) | Always handle errors | An error is handled or passed up, but never lost | [A1](#a1-an-error-is-handled-or-passed-up) [A2](#a2-a-bug-is-not-caught) |
| [**N**](#n---needed-now) | Needed now | Only what is needed now | [N1](#n1-only-what-is-needed-now) [N2](#n2-an-interface-only-where-a-second-implementation-is-real) |
| [**C**](#c---contract) | Contract | The contract is written down, fulfilled and checked by a test | [C1](#c1-the-contract-is-written-in-the-public-api) [C2](#c2-every-implementation-fulfills-the-whole-contract) [C3](#c3-a-test-checks-the-contract-through-the-public-api) |
| [**E**](#e---evident) | Evident | The code reads without surprises | [E1](#e1-a-name-says-what-it-is-and-why) [E2](#e2-the-same-thing-is-done-the-same-way) [E3](#e3-a-comment-explains-only-why) |

Code is complex when a change has to be made in many places, when a lot must be kept in mind to make it, or when it is unclear what else it will break. Every rule lowers one of these three. If a rule makes the code more complex in a particular place, not simpler, it does not apply there.

> **Example:** a reference-data module that only reads and writes table rows gains nothing from an adapter and a separate layer (I1).

The rules are written for applications with business logic and input/output: services, backends, command-line tools. In a library, a script or a user interface the same ideas hold, but handlers, adapters and use cases may be absent.

The document includes only rules that are independently written down in the guidelines of several ecosystems and do not contradict published research. The grounds for each rule are collected in the section [What the rules rest on](#what-the-rules-rest-on).

---

## Glossary

| Term | Meaning | Example |
|---|---|---|
| **Interface** | A type that describes only methods, without code. Classes implement it | `Orders` with methods `get` and `save` |
| **Implementation** | A class that implements an interface | `PostgresOrders`, `InMemoryOrders` implement `Orders` |
| **Public API** | What a function, class or module offers to the calling code: public methods, their parameters and behavior. Any code has one, even without an interface | the public API of `HttpAcquirer` is one method, `charge` |
| **Internals** | The code behind the public API that the caller does not need to know | the SQL inside `PostgresOrders.save` |
| **Contract** | The promises of a public API: what it accepts, what it guarantees, which errors it raises | "charges the full amount or nothing" |
| **Caller** | Code that calls a function or class | `OrderPayment` is the caller of `PostgresOrders` |
| **Business logic** | The rules of the domain. Made of entities and use cases | discounts, order statuses, delivery times |
| **Entity** | A domain object with an identity and invariants. It checks its own rules | `Order`, `Cart` |
| **Use case** (application service) | The business logic of one user action: sets the order of steps and connects entities with adapters | `OrderPayment`, `Shopping` |
| **Infrastructure** | Everything the code talks to outside its own memory. Makes no business decisions | Postgres, Redis, external APIs, queues, files, mail, clock |
| **External API** | Someone else's service that our code calls | payment acquirer, courier service |
| **Adapter** | A class through which our code works with an external system. Only adapters know how that system works. An adapter translates our objects into its format, and its responses and errors into ours. Makes no business decisions | `PostgresOrders`, `RedisCarts`, `HttpAcquirer` |
| **Handler** | Code through which the outside world reaches us: an HTTP endpoint, a CLI command, a queue consumer. Parses the input, calls business logic, returns the response | `POST /orders/{id}/pay` |
| **System boundary** | Handlers and adapters together: every place where our code meets the outside world | |
| **Dependency injection** | A class receives its adapters through the constructor instead of creating them | `OrderPayment(orders, acquirer)` |
| **Entry point** (`main`) | The place where the program starts: the only one where configuration is read and adapters are created and passed to business logic | `main.py` |
| **Domain type** | A type owned by business logic: an entity, a value object, our exception. Does not depend on infrastructure | `Order`, `Receipt`, `PaymentDeclined` |
| **Value object** | An immutable data type that validates itself on creation | `PostalCode`, `Money` |
| **Invariant** | A condition an object always maintains | "only a paid order is shipped" |
| **Fake** | A simple working implementation for tests. Not to be confused with a mock, which only checks calls | `InMemoryOrders` |
| **Composition** | A class holds another object and calls it, instead of inheriting from it | `OrderLines` holds a list |

### How to read the examples

The principles do not depend on the language. The examples are written in Python only as a notation: classes, methods, `if`, `for`, `raise`, `return`. Features that other languages lack are not used.

| In the examples | Meaning |
|---|---|
| `class Orders(Interface)` | an interface: only methods, without code |
| `class CashOnDelivery(PaymentMethod)` | the class implements the interface |
| `Order \| None` | the value may be absent |
| `self._status` | a private field |
| `raise SomeError(...)` | raise an exception; in a language without exceptions - return an error |
| `# ...` above a method | the method's documentation |
| `...` | code omitted |

A short example shows only what relates to the rule: constructors, handlers and the rest of the code are omitted. How everything fits together is shown in the [full example](#full-example-paying-for-an-order).

The shop types (`Order`, `Cart`, `Money` with methods `rub`, `plus`, `minus_percent`, `is_at_least` and others) are assumed to be defined already. In a language without inheritance the part of C2 about subclasses is not needed.

Instead of specific libraries and frameworks, the examples use placeholder types. Every language has them under its own names:

| Placeholder type | What it does |
|---|---|
| `HttpRequest`, `HttpResponse` | an incoming request and response. `request.path_param(name)` is a path parameter, `request.json_string(name)` is a string field of the body or `None`. `HttpResponse(status, body)` |
| `HttpClient` | outgoing HTTP requests. `post(path, body, headers)` returns an `HttpResult` with fields `status` and `body` (parsed JSON or `None`). A network failure or timeout -> `NetworkError` |
| `Database` | a Postgres client. `fetch_one(sql, params)` returns a row as a "column -> value" dictionary or `None`, `execute(sql, params)` runs a statement |
| `RedisClient` | a Redis client. `hgetall(key)` reads a whole hash, `hset(key, field, value)` writes one field |
| `Router` | any HTTP server or framework |
| `Mock`, `expect_error` | a mock from any mocking library; a check that a function raises the expected exception |
| `copy_of(x)` | an independent copy of an object |
| `env(name)`, `today()` | an environment variable; the current date |

---

## D - Deep

### D1. The public API is simpler than the internals

A function, class or module takes the complexity on itself: the caller makes one simple call, and the internals do all the work. A new function or class is extracted only if it hides something and its public API is simpler than the code it replaces. Length alone is no reason to split code.

```python
# Bad: the caller knows the acquirer's protocol and the order of steps
intent_id = acquirer.create_intent(order.total())
acquirer.attach_card(intent_id, card_token)
acquirer.confirm(intent_id)
payment_id = acquirer.capture(intent_id)
```

```python
# Good: one call, the steps are hidden inside
class HttpAcquirer:
    # Charges the full amount or nothing.
    # A repeated call with the same idempotency_key does not charge again.
    # Bank refusal -> PaymentDeclined. Network or service failure -> PaymentUnavailable.
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt: ...

receipt = acquirer.charge(order.total(), card_token, idempotency_key=order.id())
```

**Example: fine slicing that hides nothing is the same violation**

```python
# Bad: to understand one rule you read three functions
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
# Good: the whole rule is in one place
def price_after_volume_discount(order: Order) -> Money:
    if order.total().is_at_least(BIG_ORDER_FROM):
        return order.total().minus_percent(BIG_ORDER_DISCOUNT_PERCENT)
    return order.total()
```

### D2. Fewer special cases

Design the contract so the caller has nothing to handle separately. If the goal of the call is already reached, that is success, not an error. Do not hide situations the caller must react to (bank refusal, out of stock).

```python
# Bad: removing twice (a double click in the browser) is an error everyone has to catch
class Cart:
    def remove(self, sku: str) -> None:
        if sku not in self._items:
            raise ProductNotInCart(sku)
        self._items.remove(sku)
```

```python
# Good: nothing to catch
class Cart:
    # After the call the cart has no sku, whether it was there or not.
    def remove(self, sku: str) -> None:
        if sku in self._items:
            self._items.remove(sku)
```

**Example: hide only what the caller does not need to react to**

```python
# Bad: the bank refused, but pay returned as on success - an unpaid order will go to packing
def pay(self, order: Order, card_token: str) -> None:
    try:
        receipt = self._acquirer.charge(order.total(), card_token, order.id())
    except PaymentDeclined:
        return
    order.mark_paid(receipt)
```

```python
# Good: the bank refusal reaches the caller
def pay(self, order: Order, card_token: str) -> None:
    receipt = self._acquirer.charge(order.total(), card_token, order.id())
    order.mark_paid(receipt)
```

### D3. A narrow interface for each caller

If an interface is already needed (N2) and different places use it, each of them depends only on the methods it uses. If there is only one caller, do not split the interface.

> **Example:** while a single use case uses `Orders`, `get` and `save` stay in one interface.

```python
# Bad: the report needs one method but depends on five
class OrderStore(Interface):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...
    def delete(self, order_id: str) -> None: ...
    def paid_between(self, start: date, end: date) -> list[Order]: ...
    def lock_for_update(self, order_id: str) -> None: ...

def revenue(store: OrderStore, start: date, end: date) -> Money: ...
```

```python
# Good
class PaidOrders(Interface):
    def paid_between(self, start: date, end: date) -> list[Order]: ...

def revenue(orders: PaidOrders, start: date, end: date) -> Money: ...
```

---

## I - Inward

### I1. Business logic does not depend on infrastructure

Infrastructure is everything the code talks to outside its own memory: storage, external services, queues, the file system, mail, the clock, the random number generator. Each such system has its own way of talking: a query language, a protocol, a data format, a library, error codes. Only the adapter knows that way.

The adapter is not there so you can switch to another external system: that happens rarely. It is there because the system's details change all the time (schema, queries, keys, library, format, error codes), and such changes must touch only the adapter.

The adapter's public API is expressed in business terms: what to do and with which object, not how the external system does it. Business logic calls the adapter and does not know which system is behind it or how to talk to it.

The adapter is passed through the constructor (dependency injection) and created at the entry point. Infrastructure with a single operation is passed as a function parameter.

> **Example:** `is_expired(coupon, now)` receives the current time as an argument instead of reading the clock itself.

The rule applies where there are business rules: the adapter protects them from the details of the external system. Code that only moves data between a request and storage and decides nothing needs no adapter and no separate layer.

> **Example:** an admin panel endpoint that shows the list of delivery cities reads the table right in the handler. When a rule appears ("a city with active deliveries cannot be deleted"), it is moved out.

**Examples of an adapter API in business terms:**
- `orders.save(order)` instead of an SQL query;
- `carts.get(customer_id)` instead of Redis commands;
- `shipping.ship(parcel)` instead of an HTTP call;
- `events.publish(order_paid)` instead of writing bytes to a queue;
- a `now()` function passed as a parameter instead of reading the system clock inside a rule.

```python
# Bad: the "no more than 20 items" rule is mixed with Redis keys and commands
def add_to_cart(redis: RedisClient, customer_id: str, sku: str, quantity: int) -> None:
    key = "cart:" + customer_id
    items = redis.hgetall(key)
    if len(items) >= 20 and sku not in items:
        raise CartFull(customer_id)
    redis.hset(key, sku, int(items.get(sku, 0)) + quantity)
```

```python
# Good: the rule lives in Cart, only the adapter knows Redis
class Shopping:
    def __init__(self, carts: RedisCarts) -> None:
        self._carts = carts

    def add_to_cart(self, customer_id: str, sku: str, quantity: int) -> None:
        cart = self._carts.get(customer_id)
        cart.add(sku, quantity)  # raises CartFull if there are already 20 items
        self._carts.save(cart)

class RedisCarts:  # adapter: Redis keys, commands and format are only here
    def get(self, customer_id: str) -> Cart: ...
    def save(self, cart: Cart) -> None: ...

# main.py
shopping = Shopping(RedisCarts(RedisClient(env("REDIS_URL"))))
```

### I2. No cycles between packages

Imports between packages form no cycles. A cycle is broken in one of two ways: a package describes its public API in its own terms and stops knowing about its neighbor, or the interface is moved into the package that uses it.

> **Example of the second way:** `orders` must notify the customer, and `notifications` needs the order data. The `OrderNotifier` interface is declared in `orders` and implemented in `notifications`, and `orders` no longer imports `notifications`.

```
# Bad
shop/
  orders/     # imports payments to charge money
  payments/   # imports orders to set the order status to "paid"
```

```
# Good
shop/
  orders/     # knows payments: calls charge(), changes its own status
  payments/   # knows nothing about orders: charge(amount, card, key) -> Receipt
  carts/
  http/       # handlers
  main.py     # entry point: creates adapters and wires the packages together
```

---

## S - Single owner

### S1. Every piece of knowledge has one owner

Knowledge is a decision that could have been different: a business rule, a format, an algorithm, a choice of provider. Code is split by knowledge, not by execution steps. Each piece of knowledge lives in exactly one place (a function, class or module); knowledge that changes together lives together. Similar code that changes for different reasons is two pieces of knowledge, and it is not merged.

```python
# Bad: "free from 5000" lives in two places
# orders/checkout.py
def shipping_cost(order_total: Money) -> Money:
    if order_total.is_at_least(Money.rub(5000)):
        return Money.rub(0)
    return Money.rub(300)

# http/cart_handler.py
free_shipping = cart.total().is_at_least(Money.rub(5000))
```

```python
# Good: the knowledge belongs to shipping/tariff.py
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

**Example: similar code is not yet one piece of knowledge**

```python
# Bad: two decisions are glued together because the numbers happen to match today
MAX_ITEMS = 20  # both items in a cart and rows on a report page
```

```python
# Good: the cart limit is changed by the business, the page size by the report design
MAX_CART_ITEMS = 20
REPORT_PAGE_SIZE = 20
```

### S2. Only the owner changes an invariant

State with an invariant is closed inside the owning object. From outside it is changed only through the owner's methods, and the owner checks the transition. The caller says what to do instead of deciding for the owner.

```python
# Bad: the status is written from outside, the transition rule is bypassed
order.status = OrderStatus.SHIPPED  # this way an unpaid order can be "shipped"
orders.save(order)
```

```python
# Good: Order itself checks the transition
class Order:
    def ship(self, tracking_number: str) -> None:
        if self._status != OrderStatus.PAID:
            raise OrderCannotBeShipped(self._id)
        self._status = OrderStatus.SHIPPED
        self._tracking_number = tracking_number

order.ship(tracking_number)
orders.save(order)
```

### S3. Less mutable state

The less state a program has that can be changed, the fewer situations it can end up in. There is no global mutable state: everything the result depends on comes in as a parameter or through the constructor. Data that has no reason to change is immutable.

```python
# Bad: any code changes the discount at any moment, and the call to price does not show it
current_discount_percent = 0

def price(order: Order) -> Money:
    return order.total().minus_percent(current_discount_percent)
```

```python
# Good: everything the result depends on is visible in the parameters
def price(order: Order, discount_percent: int) -> Money:
    return order.total().minus_percent(discount_percent)
```

---

## T - Translate at the boundary

### T1. Foreign data is parsed into value objects at the boundary

A foreign format (request JSON, a table row, an external API response, a value from Redis) is parsed once into value objects that guarantee correctness. Incoming requests are parsed by the handler, infrastructure responses by the adapter. Business logic does not re-validate data and does not read foreign formats.

```python
# Bad: raw JSON goes into business logic, and every function validates it again
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
# Good: one check, in the value object; only it goes inside
class PostalCode:  # created means valid; does not change after creation
    def __init__(self, value: str) -> None:
        if len(value) != 6 or not value.isdigit():
            raise InvalidAddress("a postal code must be 6 digits: " + value)
        self.value = value

class Delivery:
    def set_address(self, order_id: str, postal_code: PostalCode) -> None: ...
    def estimate_days(self, postal_code: PostalCode) -> int: ...

# handler: builds a PostalCode from the request, answers 422 on InvalidAddress
delivery.set_address(order_id, PostalCode(zip_from_request))
```

### T2. Foreign errors are translated into ours in the adapter

The adapter translates an error of a library or external API that the caller can react to (a refusal, an unavailable service) into its own exception, once. Business logic sees only that exception. The adapter does not touch the other errors (A2). The same holds in the opposite direction: business logic errors are translated into the response to the client (an HTTP status, an exit code, a message) by the handler, and only by it.

```python
# Bad: a foreign response and a foreign error escape from the adapter into business logic
class Fulfillment:
    def ship(self, order: Order) -> None:
        try:
            result = self._shipping.ship(order.parcel())  # returns an HttpResult
        except NetworkError:
            order.postpone_shipping()
            return
        if result.status != 201:
            order.postpone_shipping()
            return
        order.ship(result.body["tracking"])
```

```python
# Good: the adapter returns the tracking number or raises its own exception
class CourierApiShipping:
    # Hands the parcel to the courier, returns the tracking number. Failure or refusal -> ShippingUnavailable.
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

### A1. An error is handled or passed up

An error must not be lost. Code either does something meaningful with it or hands it to the caller. There are no empty handlers. Catch only the error you expect and know what to do with: catching "everything" also hides the errors the author did not think of.

```python
# Bad: the error is swallowed, the receipt is not sent, and there is no trace anywhere
try:
    mailer.send_receipt(order)
except Exception:
    pass
```

```python
# Good: we expect one known error and do something with it, the rest goes up
try:
    mailer.send_receipt(order)
except MailUnavailable:
    outbox.retry_later(order.id())
```

### A2. A bug is not caught

A bug (a broken contract, an impossible state) and a failure there is nothing to answer with (the database is down) are not turned into an ordinary result. They pass as is to the single top-level handler, which logs them and answers 500. This way a bug is visible at once and where it arose, not later in wrong data.

```python
# Bad: the bug is hidden, the customer sees a zero total
def total(order: Order) -> Money:
    try:
        return sum_lines(order.lines())
    except Exception:
        return Money.rub(0)
```

```python
# Good: if sum_lines has a bug, the request fails and the log has the cause
def total(order: Order) -> Money:
    return sum_lines(order.lines())
```

**Example: three kinds of errors in one call**

```python
def ship(self, order: Order) -> None:
    try:
        tracking = self._shipping.ship(order.parcel())
    except ShippingUnavailable:  # expected, and we know what to do (A1); the adapter made it ours (T2)
        order.postpone_shipping()
        return
    order.ship(tracking)  # a bug or a database that is down is not caught here and goes up (A2)
```

---

## N - Needed now

### N1. Only what is needed now

Do not add features, parameters, settings or extension points the current task does not require. "Might come in handy" is not a reason. Generalize the shape of the public API, not the set of features. Think ahead only about what is expensive to change later: the format of stored data and our HTTP API for external consumers.

> **Example:** `ship(parcel)` fits any shipment, while `ship_vip_order(order)` has to be repeated for every new case.

> **Example:** an amount is stored together with its currency from the start: adding a currency to millions of rows already written is expensive, while renaming a method is cheap.

```python
# Bad: flexibility "for the future"
class CourierApiShipping:
    def ship(self, parcel: Parcel, priority: int = 0, split_by_warehouse: bool = False,
             callback_url: str | None = None) -> str: ...
```

```python
# Good
class CourierApiShipping:
    def ship(self, parcel: Parcel) -> str: ...
```

### N2. An interface only where a second implementation is real

Introduce an interface only when there really is more than one implementation:
- a fake for tests;
- a second provider or a second database;
- several variants of behavior already now, for example payment methods;
- a replacement that has already been decided.

"Might come in handy" is not a reason. While there is one implementation, the caller depends on the class directly. When a second one appears, the interface is extracted from the existing class, and that is a mechanical change: the caller already uses only its public API. In practice the second implementation comes quickly: a use case is tested on a fake (C3), so its adapters get an interface with the first test. The incoming side needs no interface: a request from a user or an external client is taken by a handler, which calls business logic itself (T1).

```python
# Bad: an interface for the sake of a single implementation
class DiscountPolicy(Interface):
    def apply(self, order: Order, customer: Customer) -> Money: ...

class DefaultDiscountPolicy(DiscountPolicy):  # there is no other one and none is planned
    def apply(self, order: Order, customer: Customer) -> Money: ...
```

```python
# Good: a plain function, called directly
def price_after_loyalty_discount(order: Order, customer: Customer) -> Money: ...
```

```python
# Good: PostgresOrders got a fake for tests - the interface was extracted
class Orders(Interface):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class PostgresOrders(Orders): ...
class InMemoryOrders(Orders): ...
```

---

## C - Contract

### C1. The contract is written in the public API

Everything the caller must know that is not visible from the name and types is written in the method's documentation: behavior in special cases, units, call order, exceptions.

```python
# Bad: what happens when stock runs short and how to release a reservation is unknown
def reserve(self, sku: str, quantity: int) -> bool: ...
```

```python
# Good
# Reserves quantity units of sku for an order.
# If there is not enough free stock, returns False and changes nothing.
# The reservation is released by release() or turned into a write-off by ship().
def reserve(self, sku: str, quantity: int) -> bool: ...
```

### C2. Every implementation fulfills the whole contract

An implementation does not require more from the caller than the interface does, does not promise less, and raises no exceptions outside the contract. If an implementation cannot fulfill the contract, change the contract or split the interface.

```python
# Bad: the implementation refuses a part of the contract
class CashOnDelivery(PaymentMethod):
    def refund(self, receipt: Receipt) -> None:
        raise NotSupported("cash cannot be refunded automatically")
```

```python
# Good: the contract honestly describes both outcomes
class PaymentMethod(Interface):
    # Returns the money right away (REFUNDED) or creates a task for an operator (OPERATOR_TASK_CREATED).
    def refund(self, receipt: Receipt) -> RefundResult: ...

class CashOnDelivery(PaymentMethod):
    def refund(self, receipt: Receipt) -> RefundResult:
        self._tasks.return_cash(receipt)
        return RefundResult.OPERATOR_TASK_CREATED
```

A subclass is an implementation too. Inherit from a class only when the subclass fully fulfills the parent's contract and the parent is designed for inheritance. Otherwise, use someone else's behavior by holding the object in a field and calling it (composition).

> **Example:** our own exceptions inherit from a common error class: that is what it exists for.

```python
# Bad: a list subclass gets all of its methods, and they bypass the rule of add
class OrderLines(list):
    def add(self, line: OrderLine) -> None: ...  # merges lines with the same sku

lines.append(line)  # an inherited method: one item is now in two lines
```

```python
# Good: the list is hidden in a field, only add is available from outside
class OrderLines:
    def __init__(self) -> None:
        self._lines = []

    def add(self, line: OrderLine) -> None: ...  # merges lines with the same sku
```

### C3. A test checks the contract through the public API

A test calls the code the same way a caller does and checks the result. Our own code that runs fast and without external systems is not replaced in a test. Infrastructure is replaced with a fake (N2), not with mocks. The fake and the real adapter are checked by the same test of the interface's contract, otherwise the fake drifts away from the real one. A test does not check private fields or the order of internal calls.

> **Example:** `Order` and `Money` are real in a test, only the order storage is replaced.

```python
# Bad: the test is tied to the internals
def test_place_order() -> None:
    orders = Mock()
    checkout = Checkout(orders)
    checkout.place_order(cart)
    assert orders.save.was_called_once()
    assert checkout._last_order.status == "new"
```

```python
# Good: the test acts like a caller and looks at the result
def test_placed_order_is_saved_as_new() -> None:
    orders = InMemoryOrders()  # fake: keeps orders in memory
    order_id = Checkout(orders).place_order(cart)
    assert orders.get(order_id).status() == OrderStatus.NEW
```

**Example: one set of checks for the fake and the real adapter**

```python
def check_orders_contract(orders: Orders) -> None:
    assert orders.get("no-such-id") is None
    orders.save(new_order("o-1"))
    assert orders.get("o-1").status() == OrderStatus.NEW

def test_in_memory_orders_keep_the_contract() -> None:
    check_orders_contract(InMemoryOrders())

def test_postgres_orders_keep_the_contract() -> None:  # slow: needs a real database
    check_orders_contract(PostgresOrders(test_database()))
```

---

## E - Evident

### E1. A name says what it is and why

A name states the meaning in the domain, not the data type and not the internals. Significant numbers and strings get names.

```python
# Bad
def calc(o: Order, c: Customer) -> Money:
    t = o.total()
    if c.lvl > 2:
        t = t.minus_percent(5)
    return t
```

```python
# Good
GOLD_LEVEL = 3
GOLD_DISCOUNT_PERCENT = 5

def price_after_loyalty_discount(order: Order, customer: Customer) -> Money:
    if customer.loyalty_level() >= GOLD_LEVEL:
        return order.total().minus_percent(GOLD_DISCOUNT_PERCENT)
    return order.total()
```

### E2. The same thing is done the same way

One task is solved one way across the system: the same method names, one way to say "no value" and "error". New code follows the established way. If the established way violates another rule, new code still follows it, and the discrepancy is named explicitly: changing the architecture is decided by the team, not by the author of a single change. The exceptions are A1 and T1: new code passes errors up and validates foreign input, whatever the project's convention is.

```python
# Bad: three adapters - three ways to say "not found"
class PostgresCustomers:
    def fetch_customer(self, customer_id: str) -> Customer: ...  # raises KeyError

class PostgresShipments:
    def load(self, shipment_id: str) -> Shipment | bool: ...  # False if there is no shipment

class PostgresOrders:
    def get(self, order_id: str) -> Order | None: ...
```

```python
# Good
class PostgresCustomers:
    def get(self, customer_id: str) -> Customer | None: ...

class PostgresShipments:
    def get(self, shipment_id: str) -> Shipment | None: ...

class PostgresOrders:
    def get(self, order_id: str) -> Order | None: ...
```

**Example: the established way matters more than the best one**

```python
# In the project all adapters raise an exception on "not found"
class PostgresCustomers:
    def get(self, customer_id: str) -> Customer: ...  # raises CustomerNotFound

# Bad: by D2 None is more convenient, but now the project has two ways to say "not found"
class PostgresShipments:
    def get(self, shipment_id: str) -> Shipment | None: ...
```

```python
# Good: the established way; the disagreement is voiced to the team, not in the code
class PostgresShipments:
    def get(self, shipment_id: str) -> Shipment: ...  # raises ShipmentNotFound
```

### E3. A comment explains only "why"

Inside code, a comment explains a reason the code cannot show. Retelling the code in a comment is forbidden.

```python
# Bad
# reservation is 30 minutes
RESERVATION_TTL_SECONDS = 30 * 60
```

```python
# Good
# The bank holds the card authorization for 30 minutes: keeping the item longer is pointless.
RESERVATION_TTL_SECONDS = 30 * 60
```

---

## Full example: paying for an order

A small but complete use case: `POST /orders/{order_id}/pay` with the body `{"card_token": "..."}`. The order is stored in Postgres, and the money is charged by an external acquirer API.

```
shop/
  payments/           # [I2] knows nothing about orders
    receipt.py        # Receipt and payment errors (domain types)
    http_acquirer.py  # HttpAcquirer (adapter)
  orders/             # [I2] knows payments, there is no reverse dependency
    order.py          # Order (entity) and its invariant
    payment.py        # OrderPayment (use case) and the interfaces it needs
    postgres.py       # PostgresOrders (adapter)
  http/
    pay_handler.py    # handler
  main.py             # entry point
tests/
  test_order_payment.py
```

### Request path

```
Legend
  (Name)    outside world: client, database, external API
  [Name]    system boundary: handler or adapter
  {Name}    business logic: use case
  <Name>    domain type: entity or value object
  !Name     our exception
  -N->      step N: a call
  <---      result of the step
  .....     what went wrong -> what is raised -> response to the client (chosen by the handler)

Foreign errors (HTTP 402, network failure, malformed JSON) are written in words:
they never go past the adapter; !OurException is raised instead.

==============================================================================

(Client)                                                        outside world
   |  POST /orders/o-1/pay   {"card_token": "tok_123"}
   v
[PayOrderHandler]                                               handler
   |  JSON -> order_id = "o-1", card_token = "tok_123"
   |  no card_token ......................... -> 400
   v
{OrderPayment}.pay("o-1", "tok_123")                            use case
   |
   |-1-> [PostgresOrders].get("o-1")                            adapter
   |       --SELECT--> (Postgres)
   |<---- <Order o-1, 990 RUB, NEW>           table row -> Order
   |       no row -> None ................... !OrderNotFound -> 404
   |       already PAID ..................... the order as is -> 200
   |
   |-2-> <Order>.ensure_payable()                               entity
   |       status is not NEW ................ !OrderNotPayable -> 409
   |
   |-3-> [HttpAcquirer].charge(<990 RUB>, "tok_123", key="o-1") adapter
   |       --POST /v1/charges--> (Acquirer)
   |<---- <Receipt pay_77, 990 RUB>           response JSON -> Receipt
   |       HTTP 402 ......................... !PaymentDeclined -> 402
   |       network, timeout, 5xx, bad JSON .. !PaymentUnavailable -> 503
   |
   |-4-> <Order>.mark_paid(<Receipt>)                           entity
   |       NEW -> PAID
   |
   |-5-> [PostgresOrders].save(<Order>)                         adapter
   |       --UPDATE--> (Postgres)
   |       Postgres failure ................. error as is -> 500
   v
[PayOrderHandler]                                               handler
   |  <Order> -> JSON
   v
(Client)  200 {"order_id": "o-1", "status": "paid"}             outside world
```

### Who depends on whom

An arrow means "imports". Domain types import nothing except each other; the database and HTTP clients (`Database`, `HttpClient`) are imported only by adapters and `main.py`. In Python and Go an adapter matches the interface without importing it; in Java, C# or Rust it names the interface explicitly, and that import also points inward.

```
[PayOrderHandler] -------> {OrderPayment} -------> <Order>, <Receipt>
 http/pay_handler           orders/payment          orders/order, payments/receipt
                                  ^                        ^
                                  | implement the          |
                                  | Orders and             |
                                  | Acquirer interfaces    |
[PostgresOrders], [HttpAcquirer] -+------------------------+
 orders/postgres, payments/http_acquirer

main.py -------> everything: creates adapters and passes them to the use case
```

### Code

A tag like `# [S2]` marks the place where a rule is visible in the code. These are notes for the reader of the document: working code does not have them.

**payments/receipt.py** - payment domain types. The use case imports them, so there is nothing from infrastructure here.

```python
class Receipt:  # [S3] value object: does not change after creation
    def __init__(self, payment_id: str, amount: Money) -> None:
        self.payment_id = payment_id
        self.amount = amount

# The bank refused. Retrying with the same card will not help.
class PaymentDeclined(Exception): ...

# The acquirer is unavailable. Can be retried later.
class PaymentUnavailable(Exception): ...
```

**payments/http_acquirer.py** - adapter to the acquirer. Only it knows the URL, request fields and response codes.

```python
class HttpAcquirer:
    def __init__(self, http: HttpClient) -> None:
        self._http = http

    # [D1] one call hides the whole acquirer protocol
    # [C1] the contract is written above the method:
    # Charges the full amount or nothing.
    # A repeated call with the same idempotency_key does not charge again.
    # Bank refusal -> PaymentDeclined. Network or service failure -> PaymentUnavailable.
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        try:
            result = self._http.post(
                "/v1/charges",
                {"amount": str(amount.amount), "currency": amount.currency, "card": card_token},
                headers={"Idempotency-Key": idempotency_key},
            )
        except NetworkError as error:
            raise PaymentUnavailable(error)  # [T2] foreign error -> ours
        if result.status == 402:
            raise PaymentDeclined()
        if result.status != 200 or result.body is None or "id" not in result.body:
            raise PaymentUnavailable()
        return Receipt(result.body["id"], amount)  # [T1] foreign JSON -> our Receipt
```

**orders/order.py** - entity. The order itself makes sure only a new order can be paid.

```python
class OrderStatus(Enum):
    NEW = "new"
    PAID = "paid"
    CANCELLED = "cancelled"

class OrderNotFound(Exception): ...

# The order is already paid or cancelled.
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

    # [S2] the rule "only a new order can be paid" lives only here
    def ensure_payable(self) -> None:
        if self._status != OrderStatus.NEW:
            raise OrderNotPayable(self._id)

    def mark_paid(self, receipt: Receipt) -> None:
        self.ensure_payable()
        self._status = OrderStatus.PAID
        self._payment_id = receipt.payment_id
```

**orders/payment.py** - use case. Knows neither SQL nor HTTP. The interfaces exist because each has two implementations: the real one and a fake for tests.

```python
class Orders(Interface):  # [N2] the interface exists because there is a second implementation - the fake
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class Acquirer(Interface):  # [D3] only what the use case uses
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt: ...

class OrderPayment:
    def __init__(self, orders: Orders, acquirer: Acquirer) -> None:  # [I1] adapters come from outside
        self._orders = orders
        self._acquirer = acquirer

    # Pays for the order by card.
    # An already paid order is returned as is, without a second charge.
    # Errors: OrderNotFound, OrderNotPayable, PaymentDeclined, PaymentUnavailable.
    def pay(self, order_id: str, card_token: str) -> Order:
        order = self._orders.get(order_id)
        if order is None:
            raise OrderNotFound(order_id)
        if order.status() == OrderStatus.PAID:
            return order  # [D2] a retry of a request already fulfilled is success, not an error
        order.ensure_payable()
        # [E3] The key is the order id: a retry after a failed save will not charge twice.
        receipt = self._acquirer.charge(order.total(), card_token, idempotency_key=order.id())
        order.mark_paid(receipt)
        self._orders.save(order)
        return order
```

**orders/postgres.py** - adapter to Postgres. Only it knows the table schema and the SQL.

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
        # [T1] table row -> Order
        return Order(
            row["id"],
            Money(row["total_amount"], row["currency"]),
            OrderStatus(row["status"]),
            row["payment_id"],
        )

    def save(self, order: Order) -> None:
        # [A2] a database failure is not caught here: it will reach the top-level handler
        self._db.execute(
            "UPDATE orders SET status = $1, payment_id = $2 WHERE id = $3",
            [order.status().value, order.payment_id(), order.id()],
        )
```

**http/pay_handler.py** - handler. Parses the request, calls business logic, translates its errors into HTTP statuses. Decides nothing itself.

```python
# POST /orders/{order_id}/pay  {"card_token": "..."}
class PayOrderHandler:
    def __init__(self, payment: OrderPayment) -> None:
        self._payment = payment

    def handle(self, request: HttpRequest) -> HttpResponse:
        card_token = request.json_string("card_token")  # [T1] input is parsed at the boundary
        if card_token is None:
            return HttpResponse(400, {"error": "card_token is required"})
        try:
            order = self._payment.pay(request.path_param("order_id"), card_token)
        except OrderNotFound:  # [T2] our errors -> HTTP statuses, only here
            return HttpResponse(404, {"error": "order not found"})
        except OrderNotPayable:
            return HttpResponse(409, {"error": "order is already paid or cancelled"})
        except PaymentDeclined:
            return HttpResponse(402, {"error": "the bank declined the payment"})
        except PaymentUnavailable:
            return HttpResponse(503, {"error": "payment is temporarily unavailable, try again later"})
        return HttpResponse(200, {"order_id": order.id(), "status": order.status().value})
```

**main.py** - entry point. The only place where adapters are created and everything is wired together. `Router` is a placeholder: any HTTP server or framework plugs in here.

```python
# [I1] the only place where adapters are created
db = Database(env("DATABASE_URL"))
acquirer = HttpAcquirer(HttpClient(base_url=env("ACQUIRER_URL"), timeout_seconds=10))
payment = OrderPayment(PostgresOrders(db), acquirer)

router = Router()
router.post("/orders/{order_id}/pay", PayOrderHandler(payment).handle)
router.serve(port=8080)
```

**tests/test_order_payment.py** - business logic tests on fakes, without a database or network.

```python
class InMemoryOrders:  # [C3] fake: a working in-memory implementation, not a mock
    def __init__(self) -> None:
        self._orders = {}  # order id -> Order

    def get(self, order_id: str) -> Order | None:
        return copy_of(self._orders.get(order_id))

    def save(self, order: Order) -> None:
        # A copy, like in a real database: a change made without save() must not be visible.
        self._orders[order.id()] = copy_of(order)

class ApprovingAcquirer:
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        return Receipt("pay-1", amount)

class DecliningAcquirer:  # [C2] the fake stays within the contract: raises only PaymentDeclined
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        raise PaymentDeclined()

def orders_with_new_order() -> InMemoryOrders:
    orders = InMemoryOrders()
    orders.save(Order("o-1", Money.rub(990), OrderStatus.NEW, payment_id=None))
    return orders

# [C3] the tests call pay like a caller and look at the result
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

## What the rules rest on

A rule gets into the document on two grounds: the guidelines of several unrelated ecosystems state it independently, and published research does not contradict it. The opinion of a single author, even a well-known one, does not count as a ground.

There are no strictly proven rules in code design. Almost all studies show a correlation, not a cause, and some effects weaken once code size is taken into account. So the "Evidence" column says only whether measurements support the rule, not that the rule is proven.

| Rule | Where it is written | Evidence |
|---|---|---|
| D1 | C++ Core Guidelines C.3, C.9; Effective Java 15; Rust API Guidelines C-STRUCT-PRIVATE; PEP 8; .NET Design Guidelines | Files with an unstable interface contain bugs noticeably more often and change more often (Mo et al. 2019, 19 projects) |
| D2 | Effective Java 54; .NET Design Guidelines (an empty collection instead of null); Go Code Review Comments (In-Band Errors); Azure API Guidelines and AWS Well-Architected (idempotency) | 92% of catastrophic failures in five distributed systems are incorrect handling of an explicitly reported error (Yuan et al. 2014) |
| D3 | Go Proverbs; Google Go Style (Interfaces); C++ Core Guidelines T.41; Rust API Guidelines C-GENERIC; .NET Design Guidelines | Interfaces that clients use in parts change more often (Romano & Pinzger 2011, 10 systems) |
| I1 | Microsoft .NET Architecture Guides (dependency inversion, persistence ignorance); Azure Architecture Center; AWS Prescriptive Guidance. Objection: Google Go Style advises against wrapping clients in your own interfaces | There are no comparative studies, so the rule is conditional |
| I2 | C++ Core Guidelines SF.9; in Go cycles between packages are forbidden by the compiler | Components in cycles have twice as many failures (Zimmermann & Nagappan 2007, Windows Server 2003) |
| S1 | C++ Core Guidelines ES.3; Microsoft .NET Architecture Guides; OWASP ASVS 1.1.6; Linux kernel coding style. Caveat about similar code: Go Proverbs, Google Testing Blog | Copies changed inconsistently are harmful (Juergens et al. 2009); the other copies add no bugs (Rahman et al. 2010; Kapser & Godfrey 2008) |
| S2 | C++ Core Guidelines C.2, C.40; Effective Java 16; Rust API Guidelines C-VALIDATE; .NET Design Guidelines | There are few direct studies |
| S3 | C++ Core Guidelines I.2, Con.1; Effective Java 17; Google Python Style; NASA/JPL Power of Ten, rule 6; Twelve-Factor App | 97% of concurrency bugs, not counting deadlocks, are a violation of atomicity or of the order of access to shared state (Lu et al. 2008) |
| T1 | OWASP Input Validation Cheat Sheet; OWASP ASVS 5.1; SEI CERT MET00-J; The Rust Book, ch. 9; Effective Java 49; C++ Core Guidelines I.4 | Vulnerabilities from unvalidated input rank 1, 3, 5 and 7 in the CWE Top 25 for 2024 |
| T2 | Effective Java 73; The Rust Book, ch. 9; .NET Design Guidelines; Google Go Style; Google AIP-193; C++ Core Guidelines I.10 | There are no direct studies |
| A1 | Effective Java 77; Go Code Review Comments (Handle Errors); PEP 8 and PEP 20; .NET CA1031; SEI CERT ERR00-J | 92% of catastrophic failures in five distributed systems are incorrect handling of an explicitly reported error; in a quarter of the cases it was simply ignored (Yuan et al. 2014) |
| A2 | Effective Java 70; The Rust Book, ch. 9; C++ Core Guidelines E.17; Go Code Review Comments (Don't Panic); .NET Design Guidelines | Catching an overly general exception is linked to bugs in some projects (de Padua & Shang 2018) |
| N1 | Google Engineering Practices; Google Testing Blog; C++ Core Guidelines Per.7; Google Go Style (least mechanism) | Most users set only 6-17% of the configuration parameters (Xu et al. 2015) |
| N2 | Go Code Review Comments; Google Go Style; .NET Design Guidelines; C++ Core Guidelines T.1 | There are no direct studies |
| C1 | C++ Core Guidelines I.5, I.7; Effective Java 56, 74; Rust API Guidelines C-FAILURE; .NET Design Guidelines; Google Go Style; AWS Well-Architected | A mismatch between documentation and code can be a bug (Tan et al. 2007) |
| C2 | C++ Core Guidelines C.120; Google C++ Style; Effective Java 10, 18, 19; .NET Design Guidelines; Go and Rust have no inheritance | Files with broken inheritance have 35-503% more bugs (Mo et al. 2019); experiments on inheritance depth contradict each other (Daly et al. 1996; Prechelt et al. 2003) |
| C3 | Software Engineering at Google, ch. 12-13; Google Testing Blog. This is one ecosystem | 51% of changes to mocks are forced by a change in production code (Spadini et al. 2019) |
| E1 | C++ Core Guidelines ES.45, NL.19; .NET Design Guidelines; SEI CERT DCL06-C; Linux kernel coding style | With words instead of abbreviations bugs are found 14-19% faster (Hofmeister et al. 2017; Schankin et al. 2018) |
| E2 | PEP 8; PEP 20; Google Style Guides; C++ Core Guidelines NL.4; Software Engineering at Google, ch. 8 | Code that is unusual for its own project contains bugs more often (Ray et al. 2016) |
| E3 | C++ Core Guidelines NL.1, NL.2; Google Style Guides; Google Engineering Practices; Linux kernel coding style | There are few direct studies |

### What is not here and why

| Idea | Why it is not included |
|---|---|
| Law of Demeter | one guideline and one study, the effect almost disappears once code size is taken into account |
| Decisions apart from side effects | one guideline, no studies |
| Polymorphism instead of branching | in experiments the simple solution is sometimes better; branching on type does not affect the number of bugs (Prechelt et al. 2001; Hall et al. 2014) |
| Pin the behavior with a test before a change | not found in the guidelines, no studies |
| Test before code (TDD) | the order "test before or after" does not affect the result (Fucci et al. 2017) |
| A limit on function length | the guidelines disagree, the studies contradict each other |
| Packages by feature, not by layer | not found in the guidelines, no comparative studies |

### Sources

Guidelines: [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines), [Effective Java](https://www.informit.com/store/effective-java-9780134686097), [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments), [Go Proverbs](https://go-proverbs.github.io), [Google Go Style](https://google.github.io/styleguide/go/decisions), [Google C++ Style](https://google.github.io/styleguide/cppguide.html), [Google Python Style](https://google.github.io/styleguide/pyguide.html), [Google Engineering Practices](https://google.github.io/eng-practices/review/), [Software Engineering at Google](https://abseil.io/resources/swe-book), [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html), [The Rust Book, ch. 9](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html), [.NET Design Guidelines](https://learn.microsoft.com/dotnet/standard/design-guidelines/), [.NET Architecture Guides](https://learn.microsoft.com/dotnet/architecture/modern-web-apps-azure/architectural-principles), [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/guide/design-principles/design-for-evolution), [PEP 8](https://peps.python.org/pep-0008), [PEP 20](https://peps.python.org/pep-0020), [.NET CA1031](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/ca1031), [SEI CERT ERR00-J](https://cmu-sei.github.io/secure-coding-standards/sei-cert-oracle-coding-standard-for-java/rules/exceptional-behavior-err/err00-j), [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html), [CWE Top 25](https://cwe.mitre.org/top25/archive/2024/2024_top25_list), [Linux kernel coding style](https://kernel.org/doc/html/latest/process/coding-style.html), [Twelve-Factor App](https://12factor.net).

Research: [Mo et al. 2019](https://www.cs.drexel.edu/~yc349/papers/2019/tse2019.pdf), [Yuan et al. 2014](https://gwern.net/doc/cs/algorithm/2014-yuan.pdf), [Zimmermann & Nagappan 2007](https://thomas-zimmermann.com/publications/files/zimmermann-issre-2007.pdf), [Juergens et al. 2009](https://ar5iv.labs.arxiv.org/html/1701.05472), [Rahman et al. 2010](https://www.microsoft.com/en-us/research/publication/clones-what-is-that-smell/), [Kapser & Godfrey 2008](https://cormack.uwaterloo.ca/~migod/papers/2008/emse08-ClonePatterns.pdf), [Lu et al. 2008](https://www.cs.columbia.edu/~junfeng/10fa-e6998/papers/concurrency-bugs.pdf), [Xu et al. 2015](https://cseweb.ucsd.edu/~tixu/papers/fse15.pdf), [Tan et al. 2007](https://www.cs.purdue.edu/homes/lintan/publications/icomment_sosp07.pdf), [Spadini et al. 2019](https://d-nb.info/1174174781/34), [Hofmeister et al. 2017](https://www.se.cs.uni-saarland.de/publications/docs/HoSeHo17.pdf), [Schankin et al. 2018](https://brains-on-code.github.io/descriptive-compound-identifier-names.pdf), [Ray et al. 2016](https://arxiv.org/abs/1506.01159), [Prechelt et al. 2001](https://ps.ipd.kit.edu/180_778.php), [Prechelt et al. 2003](https://ps.ipd.kit.edu/180_837.php), [Hall et al. 2014](https://uhra.herts.ac.uk/handle/2299/14912), [Fucci et al. 2017](https://arxiv.org/abs/1611.05994), [de Padua & Shang 2018](https://ece.uwaterloo.ca/~wshang/pubs/MSR2018_GUI.pdf).
