# DISTANCE

**English** | [Русский](README.ru.md) | [中文](README.zh-CN.md)

**Keep your distance:** every piece of code knows as little as possible about the others. Then the code is cheap to understand and safe to change.

| | Principle | Essence | Rules |
|---|---|---|---|
| [**D**](#d---deep) | Deep | The public API is simpler than the internals | [D1](#d1-the-public-api-is-simpler-than-the-internals) [D2](#d2-fewer-special-cases) [D3](#d3-talk-to-your-nearest-neighbor) [D4](#d4-a-narrow-interface-for-each-caller) |
| [**I**](#i---inward) | Inward | Business logic does not know the infrastructure | [I1](#i1-business-logic-does-not-depend-on-infrastructure) [I2](#i2-an-interface-only-where-a-second-implementation-is-real) [I3](#i3-decisions-apart-from-side-effects) [I4](#i4-no-cycles-between-packages) |
| [**S**](#s---single-owner) | Single owner | Every piece of knowledge and every invariant has one owner | [S1](#s1-every-piece-of-knowledge-has-one-owner) [S2](#s2-only-the-owner-changes-an-invariant) |
| [**T**](#t---translate-at-the-boundary) | Translate at the boundary | Foreign data and errors are translated into our own once, at the system boundary | [T1](#t1-foreign-data-is-parsed-into-value-objects-at-the-boundary) [T2](#t2-foreign-errors-are-translated-into-ours-in-the-adapter) |
| [**A**](#a---add-a-kind-not-a-branch) | Add a kind, not a branch | A new variant of behavior is a new implementation, not a new condition | [A1](#a1-a-new-variant-of-behavior-is-a-new-implementation) [A2](#a2-every-implementation-fulfills-the-whole-contract) [A3](#a3-composition-over-inheritance) |
| [**N**](#n---needed-now) | Needed now | Only what is needed now | [N1](#n1-only-what-is-needed-now) |
| [**C**](#c---contract) | Contract | The contract is written down and checked by a test | [C1](#c1-the-contract-is-written-in-the-public-api) [C2](#c2-a-test-checks-the-contract-through-the-public-api) [C3](#c3-code-without-tests-first-extract-the-decision-and-pin-it-with-a-test) |
| [**E**](#e---evident) | Evident | The code reads without surprises | [E1](#e1-a-name-says-what-it-is-and-why) [E2](#e2-the-same-thing-is-done-the-same-way) [E3](#e3-a-comment-explains-only-why) |

If a rule makes the code more complex in a particular place, not simpler, it does not apply there.

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
| **Entry point** (`main`) | The place where the program starts, creates adapters and passes them to business logic | `main.py` |
| **Value object** | An immutable data type that validates itself on creation | `PostalCode`, `Address`, `Money` |
| **Invariant** | A condition an object always maintains | "only a paid order is shipped" |
| **Side effect** | An action beyond returning a result: writing to a database, a network call, an email, changing someone else's object | `payments.refund(...)` |
| **Pure function** | A function without side effects: the result depends only on the arguments | `decide_return(order, lines)` |
| **Fake** | A simple working implementation for tests. Not to be confused with a mock, which only checks calls | `InMemoryOrders` |
| **Composition** | A class holds another object and calls it, instead of inheriting from it | `OrderLines` holds a dictionary |

### How to read the examples

The principles do not depend on the language. The examples are written in Python without frameworks or language-specific features, so that a programmer in any language can read them.

| In the examples | In Java | In Go |
|---|---|---|
| `class Orders(Protocol)` | `interface Orders` | `type Orders interface` |
| `@dataclass(frozen=True)` | `record` | `struct` without setters |
| `raise SomeError` | `throw new SomeException()` | `return ..., ErrSome` |
| `None` | `null` / `Optional` | `nil` |
| `"""..."""` under a method | Javadoc | comment above the function |
| `raise Ours(error)` | `throw new Ours(error)` | `fmt.Errorf("...: %w", err)` |
| `Decimal` | `BigDecimal` | a decimal type without rounding errors |

The shop types (`Money` with methods `plus`, `times`, `is_at_least`, `Order`, `Cart` and others) are assumed to be defined already.

Instead of specific libraries and frameworks, the examples use placeholder types. Every language has them under its own names:

| Placeholder type | What it does |
|---|---|
| `HttpRequest`, `HttpResponse` | an incoming request and response. `request.path_param(name)` is a path parameter, `request.json_string(name)` is a string field of the body or `None`, `request.json_body()` is the whole body as a dictionary. `HttpResponse(status, body)` |
| `HttpClient` | outgoing HTTP requests. `post(path, body, headers)` returns an `HttpResult` with fields `status` and `body` (parsed JSON or `None`). A network failure or timeout -> `NetworkError` |
| `Database` | a Postgres client. `fetch_one(sql, params)` returns a row as a "column -> value" dictionary or `None`, `execute(sql, params)` runs a statement |
| `RedisClient` | a Redis client. Methods are named after Redis commands: `hgetall`, `hset`, `hincrby`, `expire`, `delete`. `transaction()` is a group of commands executed atomically on `exec()` (MULTI/EXEC) |
| `Router` | any HTTP server or framework |
| `Mock`, `expect_error` | a mock from any mocking library; a check that a function raises the expected exception |
| `env(name)` | an environment variable |

The short examples omit the handler's services; the [full example](#full-example-paying-for-an-order) shows how they are passed in.

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
# Good
class HttpAcquirer:
    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        """Charges the full amount or nothing.

        A repeated call with the same idempotency_key returns the same receipt and does not charge again.
        Bank refusal -> PaymentDeclined. Network or service failure -> PaymentUnavailable.
        """
        ...

receipt = acquirer.charge(order.total(), card_token, idempotency_key=order.id())
```

### D2. Fewer special cases

Design the contract so the caller has nothing to handle separately. If the goal of the call is already reached, that is success, not an error. Do not hide situations the caller must react to (bank refusal, out of stock).

```python
# Bad: removing twice (a double click in the browser) is an error everyone has to catch
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
# Good
class Cart:
    def remove(self, sku: str) -> None:
        """After the call the cart has no sku, whether it was there or not."""
        if sku in self._items:
            del self._items[sku]

cart.remove(sku)
```

### D3. Talk to your nearest neighbor

Code calls methods only on itself, its parameters, its fields and the objects it created. It does not walk a chain through someone else's internals. Exceptions are value objects and chained APIs such as query builders: their structure is their public API.

```python
# Bad
def loyalty_discount(order: Order) -> Decimal:
    return order.customer().profile().loyalty().level().discount_rate()
```

```python
# Good
def loyalty_discount(order: Order, loyalty: Loyalty) -> Decimal:
    return loyalty.discount_rate(order.customer_id())
```

### D4. A narrow interface for each caller

If an interface is already needed (I2) and different places use it, each of them depends only on the methods it uses. If there is only one caller, do not split the interface.

```python
# Bad: the report needs one method but depends on five
class OrderStore(Protocol):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...
    def delete(self, order_id: str) -> None: ...
    def paid_between(self, start: date, end: date) -> list[Order]: ...
    def lock_for_update(self, order_id: str) -> None: ...

def revenue(store: OrderStore, start: date, end: date) -> Money: ...
```

```python
# Good
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

### I1. Business logic does not depend on infrastructure

Infrastructure is everything the code talks to outside its own memory: storage, external services, queues, the file system, mail, the clock, the random number generator. Each such system has its own way of talking: a query language, a protocol, a data format, a library, error codes. Only the adapter knows that way.

The adapter is not there so you can switch to another external system: that happens rarely. It is there because the system's details change all the time (schema, queries, keys, library, format, error codes), and such changes must touch only the adapter.

The adapter's public API is expressed in business terms: what to do and with which object, not how the external system does it. Business logic calls the adapter and does not know which system is behind it or how to talk to it.

The adapter is passed through the constructor (dependency injection) and created at the entry point. Infrastructure with a single operation is passed as a function parameter.

For example: `orders.save(order)` instead of an SQL query, `carts.get(customer_id)` instead of Redis commands, `shipping.ship(parcel)` instead of an HTTP call, `events.publish(order_paid)` instead of writing bytes to a queue, a `now()` function passed as a parameter instead of reading the system clock inside a rule.

```python
# Bad: the "no more than 20 items" rule is mixed with Redis keys and commands
MAX_ITEMS = 20

def add_to_cart(redis: RedisClient, customer_id: str, sku: str, quantity: int) -> None:
    key = "cart:" + customer_id
    if redis.hlen(key) >= MAX_ITEMS and not redis.hexists(key, sku):
        raise CartFull(customer_id)
    redis.hincrby(key, sku, quantity)
    redis.expire(key, 7 * 24 * 3600)
```

```python
# Good: only the adapter knows Redis, the rule lives in Cart
CART_TTL_SECONDS = 7 * 24 * 3600

class RedisCarts:
    """Adapter: a cart is stored in Redis as a hash "sku -> quantity"."""

    def __init__(self, redis: RedisClient) -> None:
        self._redis = redis

    def get(self, customer_id: str) -> Cart:
        """The customer's cart. If it does not exist yet - an empty cart."""
        items = {}
        for sku, quantity in self._redis.hgetall("cart:" + customer_id).items():
            items[sku] = int(quantity)
        return Cart(customer_id, items)

    def save(self, cart: Cart) -> None:
        key = "cart:" + cart.customer_id()
        tx = self._redis.transaction()  # MULTI/EXEC: the old cart and the new one will not mix
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
        cart.add(sku, quantity)  # raises CartFull if there are already 20 items
        self._carts.save(cart)

# main.py
shopping = Shopping(RedisCarts(RedisClient(env("REDIS_URL"))))
```

### I2. An interface only where a second implementation is real

Introduce an interface only when there really is more than one implementation:
- a fake for tests;
- a second provider or a second database;
- several variants of behavior already now, for example payment methods (A1);
- a replacement that has already been decided.

"Might come in handy" is not a reason. While there is one implementation, the caller depends on the class directly. When a second one appears, the interface is extracted from the existing class, and that is a mechanical change: the caller already uses only its public API. Our HTTP API and the user do not require an interface: their input is taken by a handler (T1).

```python
# Bad: an interface for business logic with a single implementation
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
# Good
class Checkout:
    def __init__(self, orders: PostgresOrders) -> None:
        self._orders = orders

    def quote(self, order: Order, customer: Customer) -> Money:
        return price_after_loyalty_discount(order, customer).plus(shipping_cost(order.total()))
```

```python
# Good: a second implementation appeared (the InMemoryOrders fake for tests) - the interface was extracted
class Orders(Protocol):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class Checkout:
    def __init__(self, orders: Orders) -> None:
        self._orders = orders
```

### I3. Decisions apart from side effects

A pure function computes the business decision. Thin code performs the side effects (payment, emails, database writes) based on its result. An object changing its own state in its own method does not count as a side effect here.

```python
# Bad: the decision and the side effects are mixed
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
# Good
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

### I4. No cycles between packages

Imports between packages form no cycles. A cycle is broken in one of two ways: a package describes its public API in its own terms and stops knowing about its neighbor, or the interface is moved into the package that uses it.

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
def shipping_cost(order: Order) -> Money:
    if order.total().amount >= Decimal("5000"):
        return Money.zero("RUB")
    return Money(Decimal("300"), "RUB")

# http/cart_handler.py
body = {"total": str(cart.total().amount), "free_shipping": cart.total().amount >= Decimal("5000")}
```

```python
# Good: the knowledge belongs to shipping/tariff.py
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

### S2. Only the owner changes an invariant

State with an invariant is closed inside the owning object. From outside it is changed only through the owner's methods, and the owner checks the transition. The caller says what to do instead of deciding for the owner.

```python
# Bad: the admin panel writes the status directly, bypassing the transition rule
def handle_update_order(request: HttpRequest) -> HttpResponse:
    order = orders.get(request.path_param("order_id"))
    order.status = request.json_string("status")  # an unpaid order can be "shipped"
    orders.save(order)
    return HttpResponse(204)
```

```python
# Good: a separate action, Order checks the transition
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
        return HttpResponse(400, {"error": "tracking_number is required"})
    try:
        fulfillment.ship(request.path_param("order_id"), tracking_number)
    except OrderCannotBeShipped:
        return HttpResponse(409, {"error": "only a paid order can be shipped"})
    return HttpResponse(204)
```

---

## T - Translate at the boundary

### T1. Foreign data is parsed into value objects at the boundary

A foreign format (request JSON, a table row, an external API response, a value from Redis) is parsed once into value objects that guarantee correctness. Incoming requests are parsed by the handler, infrastructure responses by the adapter. Business logic does not re-validate data and does not read foreign formats.

```python
# Bad: raw JSON goes into business logic, and every function validates it again
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
# Good: the handler builds an Address, and only it goes inside
class PostalCode:
    def __init__(self, value: str) -> None:
        if len(value) != 6 or not value.isdigit():
            raise InvalidAddress("a postal code must be 6 digits: " + value)
        self.value = value

class Address:
    def __init__(self, postal_code: PostalCode, street: str) -> None:
        if street.strip() == "":
            raise InvalidAddress("empty street")
        self.postal_code = postal_code
        self.street = street

def handle_set_address(request: HttpRequest) -> HttpResponse:
    zip_code = request.json_string("zip")
    street = request.json_string("street")
    if zip_code is None or street is None:
        return HttpResponse(400, {"error": "zip and street are required"})
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

### T2. Foreign errors are translated into ours in the adapter

The adapter translates an error of a library or external API into its own exception, once. Business logic sees only that exception.

```python
# Bad: the adapter lets a foreign error and a foreign response escape
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
# Good
class CourierApiShipping:
    """Adapter to the courier service API."""

    def __init__(self, http: HttpClient) -> None:
        self._http = http

    def ship(self, parcel: Parcel) -> str:
        """Hands the parcel to the courier, returns the tracking number. Failure or refusal -> ShippingUnavailable."""
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

### A1. A new variant of behavior is a new implementation

A variant of behavior (a payment method, a delivery service) is a separate implementation of a common interface. The right implementation is chosen at the entry point or in the handler. Business logic has no `if`/`elif` or flags on the variant. The exception is a closed, stable set with many operations on it (order statuses): describe it as an enum and handle it completely in one owning place.

```python
# Bad: every new payment method is a new branch in business logic
class Checkout:
    def pay(self, order: Order, method: str, card_token: str | None) -> Receipt:
        if method == "card":
            return self._acquirer.charge(order.total(), card_token, order.id())
        elif method == "cash_on_delivery":
            return Receipt.pending(order.id())
        raise ValueError(method)
```

```python
# Good
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

# handler: choosing an implementation from the input is part of parsing the request
def payment_method_from(request: HttpRequest, acquirer: HttpAcquirer) -> PaymentMethod | None:
    method = request.json_string("method")
    card_token = request.json_string("card_token")
    if method == "card" and card_token is not None:
        return CardPayment(acquirer, card_token)
    if method == "cash_on_delivery":
        return CashOnDelivery()
    return None  # the handler will respond with 400
```

### A2. Every implementation fulfills the whole contract

An implementation does not require more from the caller than the interface does, does not promise less, and raises no exceptions outside the contract. If an implementation cannot fulfill the contract, change the contract or split the interface.

```python
# Bad
class CashOnDelivery:
    def refund(self, receipt: Receipt) -> None:
        raise NotImplementedError("cash cannot be refunded automatically")
```

```python
# Good: the contract honestly describes both outcomes
class RefundResult(Enum):
    REFUNDED = "refunded"
    OPERATOR_TASK_CREATED = "operator_task_created"

class PaymentMethod(Protocol):
    def refund(self, receipt: Receipt) -> RefundResult:
        """Returns the money right away or creates a task for an operator."""
        ...

class CashOnDelivery:
    def __init__(self, tasks: OperatorTasks) -> None:
        self._tasks = tasks

    def refund(self, receipt: Receipt) -> RefundResult:
        self._tasks.return_cash(receipt)
        return RefundResult.OPERATOR_TASK_CREATED
```

### A3. Composition over inheritance

Reuse someone else's behavior by holding the object in a field and calling it. Inherit from a class only when the subclass fully fulfills the parent's contract and the parent is designed for inheritance.

```python
# Bad: a list subclass gets append, extend, insert that bypass add
class OrderLines(list):
    def add(self, line: OrderLine) -> None:
        for existing in self:
            if existing.sku == line.sku:
                existing.quantity += line.quantity
                return
        self.append(line)

lines.extend([line_a, line_a])  # one item is now in two lines
```

```python
# Good
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

### N1. Only what is needed now

Do not add features, parameters, settings or extension points the current task does not require. "Might come in handy" is not a reason. Generalize the shape of the public API (accept a parcel, not "an order of a VIP customer"), not the set of features. Think ahead only about what is expensive to change later: the format of stored data and our HTTP API for external consumers.

```python
# Bad: flexibility "for the future"
SHIPPING_PLUGINS: dict[str, type] = {}

class CourierApiShipping:
    def ship(self, parcel: Parcel, priority: int = 0, split_by_warehouse: bool = False,
             insurance: Money | None = None, callback_url: str | None = None) -> str: ...
```

```python
# Good
class CourierApiShipping:
    def ship(self, parcel: Parcel) -> str: ...
```

---

## C - Contract

### C1. The contract is written in the public API

Everything the caller must know that is not visible from the name and types is written in the method's documentation: behavior in special cases, units, call order, exceptions.

```python
# Bad: what happens when stock runs short and how to release a reservation is unknown
def reserve(self, sku: str, quantity: int) -> bool:
    if self._free.get(sku, 0) < quantity:
        return False
    self._free[sku] -= quantity
    return True
```

```python
# Good
def reserve(self, sku: str, quantity: int) -> bool:
    """Reserves quantity units of sku for an order.

    If there is not enough free stock, returns False and changes nothing.
    The reservation is released by release() or turned into a write-off by ship().
    """
    if self._free.get(sku, 0) < quantity:
        return False
    self._free[sku] -= quantity
    return True
```

### C2. A test checks the contract through the public API

A test calls the code the same way a caller does and checks the result. Infrastructure in a test is replaced with a fake (I2), not with mocks. A test does not check private fields or the order of internal calls.

```python
# Bad: the test is tied to the internals
def test_place_order() -> None:
    orders = Mock()
    checkout = Checkout(orders)
    checkout.place_order(make_cart(), make_customer())
    assert orders.save.was_called_once()
    assert checkout._last_order.status == "new"
```

```python
# Good
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

### C3. Code without tests: first extract the decision and pin it with a test

Before changing code without tests, move the business decision into a function that can be called on its own, and pin the current behavior with a test, even if it is odd. Only then change the behavior. The other principles are applied in later steps.

```python
# Bad: we change the condition right here and do not know what we break
def delivery_date(order_id: str) -> datetime:
    row = db.fetch_one("SELECT city FROM orders WHERE id = $1", [order_id])
    days = 2 if row["city"] == "Moscow" else 5
    if datetime.now().weekday() >= 4:
        days += 2
    return datetime.now() + timedelta(days=days)
```

```python
# Good
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

### E1. A name says what it is and why

A name states the meaning in the domain, not the data type and not the internals. Significant numbers and strings get names.

```python
# Bad
def calc(o: Order, c: Customer) -> Money:
    t = o.total()
    if c.lvl > 2:
        t = t.times(Decimal("0.95"))
    return t
```

```python
# Good
GOLD_LEVEL = 3
GOLD_DISCOUNT = Decimal("0.05")

def price_after_loyalty_discount(order: Order, customer: Customer) -> Money:
    total = order.total()
    if customer.loyalty_level() >= GOLD_LEVEL:
        return total.times(1 - GOLD_DISCOUNT)
    return total
```

### E2. The same thing is done the same way

One task is solved one way across the system: the same method names, one way to say "no value" and "error". New code follows the established way. If the established way violates D, I, S or T, the new place is done right, and the discrepancy is named explicitly.

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
  payments/
    receipt.py        # Receipt and payment errors (domain types)
    http_acquirer.py  # HttpAcquirer (adapter)
  orders/
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
they never go past the adapter; our !Exception is raised instead.

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

An arrow means "imports". Domain types import nothing except each other; the database and HTTP clients (`Database`, `HttpClient`) are imported only by adapters and `main.py`.

```
[PayOrderHandler] -------> {OrderPayment} -------> <Order>, <Receipt>
 http/pay_handler           orders/payment          orders/order, payments/receipt
                                  ^                        ^
                                  | match the              |
                                  | Orders and             |
                                  | Acquirer interfaces    |
                                  | (no import)            |
[PostgresOrders], [HttpAcquirer] -+------------------------+
 orders/postgres, payments/http_acquirer

main.py -------> everything: creates adapters and passes them to the use case
```

### Code

**payments/receipt.py** - payment domain types. The use case imports them, so there is nothing from infrastructure here.

```python
@dataclass(frozen=True)
class Receipt:
    payment_id: str
    amount: Money

class PaymentDeclined(Exception):
    """The bank refused. Retrying with the same card will not help."""

class PaymentUnavailable(Exception):
    """The acquirer is unavailable. Can be retried later."""
```

**payments/http_acquirer.py** - adapter to the acquirer. Only it knows the URL, request fields and response codes.

```python
class HttpAcquirer:
    def __init__(self, http: HttpClient) -> None:
        self._http = http

    def charge(self, amount: Money, card_token: str, idempotency_key: str) -> Receipt:
        """Charges the full amount or nothing.

        A repeated call with the same idempotency_key does not charge again.
        Bank refusal -> PaymentDeclined. Network or service failure -> PaymentUnavailable.
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

**orders/order.py** - entity. The order itself makes sure only a new order can be paid.

```python
class OrderStatus(Enum):
    NEW = "new"
    PAID = "paid"
    CANCELLED = "cancelled"

class OrderNotFound(Exception): ...

class OrderNotPayable(Exception):
    """The order is already paid or cancelled."""

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

**orders/payment.py** - use case. Knows neither SQL nor HTTP. The interfaces exist because each has two implementations: the real one and a fake for tests.

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
        """Pays for the order by card.

        Errors: OrderNotFound, OrderNotPayable, PaymentDeclined, PaymentUnavailable.
        """
        order = self._orders.get(order_id)
        if order is None:
            raise OrderNotFound(order_id)
        order.ensure_payable()
        # The key is the order id: a retry after a failed save will not charge twice.
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

**http/pay_handler.py** - handler. Parses the request, calls business logic, translates its errors into HTTP statuses. Decides nothing itself.

```python
class PayOrderHandler:
    """POST /orders/{order_id}/pay  {"card_token": "..."}"""

    def __init__(self, payment: OrderPayment) -> None:
        self._payment = payment

    def handle(self, request: HttpRequest) -> HttpResponse:
        card_token = request.json_string("card_token")
        if card_token is None:
            return HttpResponse(400, {"error": "card_token is required"})
        try:
            order = self._payment.pay(request.path_param("order_id"), card_token)
        except OrderNotFound:
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
db = Database(env("DATABASE_URL"))
acquirer = HttpAcquirer(HttpClient(base_url=env("ACQUIRER_URL"), timeout_seconds=10))
payment = OrderPayment(PostgresOrders(db), acquirer)

router = Router()
router.post("/orders/{order_id}/pay", PayOrderHandler(payment).handle)
router.serve(port=8080)
```

**tests/test_order_payment.py** - business logic tests on fakes, without a database or network.

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

**Where DISTANCE shows up**

| Rule | Where you see it |
|---|---|
| D1 | `HttpAcquirer.charge` and `OrderPayment.pay` - one call each instead of several steps |
| I1 | `OrderPayment` knows neither SQL nor HTTP; adapters are created only in `main.py` |
| I2 | the `Orders` and `Acquirer` interfaces exist because each has a fake |
| I4 | `orders` knows `payments`, and `payments` knows nothing about orders |
| S2 | the rule "only a new order can be paid" lives in `Order` |
| T1 | the handler validates the request body, adapters parse the table row and the acquirer response |
| T2 | `NetworkError` and acquirer response codes become `PaymentDeclined` and `PaymentUnavailable` in the adapter; business logic errors become HTTP statuses in the handler |
| C1 | the contract of `charge` and `pay` is written in the method documentation |
| C2 | the tests check the result through `pay`, without mocks |
| E3 | the only comment in business logic explains why the key is the order id |
