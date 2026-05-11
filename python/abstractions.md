# Abstractions in Python: Protocol, ABC, Mixin, and Composition

> Last updated: 2026-05-11

## TL;DR

How to choose between `Protocol`, `ABC`, mixin, and composition in Python — a design reference for DDD / Ports-and-Adapters codebases where the application/domain layers depend on capability contracts rather than infrastructure hierarchies.

**Use this when:**
- choosing between a capability contract (Protocol) and a lifecycle base class (ABC)
- deciding whether shared infrastructure behavior should be a mixin or a composed collaborator
- designing a port or adapter and unsure which abstraction encodes the right intent

**Don't use this for:**
- Python language tutorial — this doc assumes you know each mechanism syntactically
- DI container shape → `./dependency-injector.md`
- specific data-shape choices (`TypedDict` vs `BaseModel`) → `./typed-data-structures.md`

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Concepts | [Quick Reference](#quick-reference), [Core Concepts](#core-concepts) |
| 2. Decide | [Decision Rules](#decision-rules), [Recommended Pattern](#recommended-pattern-protocol--composition-or-mixin--concrete) |
| 3. Avoid | [Pitfalls](#pitfalls) |

Python offers four overlapping mechanisms for expressing abstraction and reuse: `Protocol`, `ABC`, mixins, and composition. They look interchangeable at first but encode distinct architectural intentions. This doc explains how to choose between them in DDD / Ports-and-Adapters codebases, where the application and domain layers must depend on capability contracts rather than infrastructure hierarchies.

This is a design reference, not a Python language tutorial. It assumes you know how each mechanism works syntactically.

## Quick Reference


| Tool        | Question it answers                                   | Primary DDD role                             |
| ----------- | ----------------------------------------------------- | -------------------------------------------- |
| Protocol    | "What can this object do?"                            | Capability contracts / ports                 |
| ABC         | "What identity, state, or lifecycle does this share?" | Shared lifecycle/state, runtime identity     |
| Mixin       | "What reusable behavior should this inherit?"         | Small reusable behavior — use sparingly      |
| Composition | "What collaborating object should this use?"          | Default for stateful infrastructure behavior |


## Core Concepts

### Protocol (structural typing)

A `Protocol` declares a capability shape. Any class with matching method signatures satisfies it — no inheritance required.

```python
from typing import Protocol


class UserRepository(Protocol):
    def get_by_id(self, id: str) -> User: ...
    def save(self, user: User) -> None: ...


class PostgresUserRepository:
    def get_by_id(self, id: str) -> User: ...
    def save(self, user: User) -> None: ...
```

This is **structural typing**: a class satisfies the abstraction because it behaves correctly, not because it declares membership.

In DDD terms, ports are capability contracts. The application layer needs *something capable of saving a user*, not something belonging to a `UserRepository` inheritance hierarchy. Protocol matches that intent precisely.

#### Do not inherit from Protocols

Concrete implementations satisfy a Protocol structurally — they do not inherit from it. Python permits `class SqlUserRepository(UserRepository):`, but this codebase forbids the pattern.

```python
# Forbidden
class SqlUserRepository(UserRepository): ...


# Required
class SqlUserRepository: ...
```

Why:

- **Preserves loose coupling.** The implementation is never nominally bound to the port; the relationship lives only in type-checker analysis at the call site.
- **Keeps Protocol semantics intact.** Protocols are structural by design; explicit inheritance turns them into "ABC-lite" and conflates the two mental models.
- **Respects layering.** A persistence-layer adapter inheriting from an application-layer Protocol creates a reverse-direction import that violates layer boundaries.

If you genuinely need explicit inheritance — runtime polymorphism, shared state, lifecycle hooks — switch to an `ABC` (see [Protocol vs ABC](#protocol-vs-abc)). Do not inherit from a Protocol just to get class-definition-time validation; the type checker already enforces structural conformance at call sites.

#### What static checkers actually catch

Protocol is not informal duck typing. `mypy` and `pyright` validate the **full method signature** structurally — parameter types, parameter count, optional and default arguments, return types, async-vs-sync, generic compatibility. Per `[setup-toolchains.md](./setup-toolchains.md)`, mypy is the pre-commit/CI gate; pyright runs in editors. Same checking capability, different gating role.

```python
from typing import Protocol


class UserRepository(Protocol):
    async def get_by_id(self, user_id: str) -> dict | None: ...


class SqlUserRepository:  # OK — signatures match
    async def get_by_id(self, user_id: str) -> dict | None: ...


class BadRepository:  # mypy: rejected
    async def get_by_id(self, user_id: int) -> dict | None:
        ...
        # Argument 1 has incompatible type "int"; expected "str"
```

This is the strong safety net. The runtime check covered below is much weaker — and, for that reason, rarely necessary on top of it.

#### Trade-off: discoverability

Structural typing has a real cost: the implementation does not declare which port it satisfies. With an ABC, `class StripePaymentGateway(PaymentGateway):` advertises the relationship at the class definition. With Protocol you have to grep for callers or rely on type-checker output to find the link.

Mitigate with naming and folder structure. Ports live in the application layer; concrete adapters live in infrastructure:

```text
application/
    ports/
        payment_gateway.py
infrastructure/
    stripe_payment_gateway.py
    fake_payment_gateway.py
```

Explicit DI wiring helps too — the port type appears in the adapter's constructor signature (or the container config), making the relationship visible at the boundary that matters most.

#### Do you need `@runtime_checkable`?

In this codebase, usually no. Static checking already validates the full signature; `@runtime_checkable` is a much weaker check that adds little on top.

`@runtime_checkable` lets you call `isinstance(x, SomeProtocol)`, but it only verifies **method-name presence** — not arity, parameter types, or return types:

#### Protocols can have default method bodies, but don't use that for ports

PEP 544 allows methods on a Protocol to have default implementations. That can be useful in some library contexts, but in DDD don't put implementation in your ports — once consumers rely on a default body, they're coupled to the port's implementation, not just its contract. Keep ports interface-only and put shared behavior in a mixin or a base class.

### ABC (nominal typing)

An `abc.ABC` declares a class hierarchy. Subclasses must explicitly inherit, and abstract methods are enforced at instantiation time.

```python
from abc import ABC, abstractmethod


class Repository(ABC):
    @abstractmethod
    def save(self, entity) -> None: ...
```

This is **nominal typing**: a class belongs to the abstraction because it declares so by inheritance.

ABCs fit when a base must contribute more than a method signature — shared state, lifecycle methods, or runtime identity. They are not "framework-only"; in-app ABCs are legitimate when a family of types genuinely shares state or initialization. A `DomainEvent` base is the canonical example:

```python
from abc import ABC
from datetime import datetime, timezone
from uuid import UUID, uuid4


class DomainEvent(ABC):
    def __init__(self) -> None:
        self.event_id: UUID = uuid4()
        self.occurred_at: datetime = datetime.now(timezone.utc)


class OrderPlaced(DomainEvent):
    def __init__(self, order_id: UUID) -> None:
        super().__init__()
        self.order_id = order_id
```

Every domain event needs an ID and timestamp, and downstream code (event log, dispatcher) does `isinstance(e, DomainEvent)` checks. ABC fits because shared state and runtime identity both matter.

#### `ABCMeta.register()` exists but offers little for DDD ports

You can register an external class as a virtual subclass without modifying it:

```python
Repository.register(SomeThirdPartyClass)
```

`isinstance` then returns `True`, but Python still does not validate that the class actually satisfies the abstract methods. In this codebase, prefer explicit subclassing for in-house code and an adapter wrapper at the boundary for third-party code — `register()` provides nominal tagging without contract validation, which is the worst of both worlds for port safety.

### Mixin

A mixin is a small class meant to be inherited for its implementation, not for its identity.

```python
import json


class JsonMixin:
    def to_json(self) -> str:
        return json.dumps(self.__dict__)


class User(JsonMixin): ...
```

Mixins answer *"what reusable behavior should this object inherit?"* — not *"what is this object?"*. They encode behavior reuse without becoming part of the type's primary identity.

#### Hidden dependencies are the main hazard

A mixin that references state it does not own creates a hidden contract on the host class. The classic offender:

```python
class TransactionMixin:
    def commit(self) -> None:
        self.session.commit()  # where does `self.session` come from?
```

`TransactionMixin` only works if every host class happens to define `self.session`. There is no compile-time check, no signal in the mixin's own definition, and the failure mode is an `AttributeError` deep in the call stack. This is a teaching example, not a recommendation — if you find yourself writing this, you almost certainly want **composition** instead.

Other mixin hazards:

- **MRO complexity** when multiple mixins call `super()` and the linearization does not match what each mixin assumed.
- **State coupling** when two mixins both want to set or read the same attribute name.
- **Diamond inheritance** when mixins share base classes.

Keep mixins small, stateless, and infrastructure-only. Do not mix them into domain models — they fragment behavior across inheritance chains and obscure ubiquitous language.

### Composition

Composition holds a collaborator as an attribute rather than inheriting from it.

```python
class SqlRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def save(self, entity: Entity) -> None:
        self._session.add(entity)

    def commit(self) -> None:
        self._session.commit()
```

The collaborator (`session`) is explicit — it appears in the constructor signature, it is typed, and tests can pass a fake without touching the inheritance graph. This is the default shape for stateful infrastructure behavior in DDD: clearer dependencies, easier testing, no MRO surprises.

Composition is what most "I want shared behavior" cases actually want. Reach for a mixin only when composition will not fit (see Decision Rules).

See also: [PEP 544 — Protocols: Structural subtyping](https://peps.python.org/pep-0544/), [`typing.Protocol`](https://docs.python.org/3/library/typing.html#typing.Protocol), [`abc` — Abstract Base Classes](https://docs.python.org/3/library/abc.html).

## Decision Rules

### Protocol vs ABC

Default to Protocol for ports. Reach for ABC when:

- you need shared state or initialization across all subclasses (`DomainEvent`'s `event_id`),
- you need lifecycle hooks (`__enter__` / `__exit__`, setup/teardown),
- runtime identity matters and `isinstance` is part of the design (event dispatch, sentinel detection).

If the only thing the base contributes is a method signature, it should be a Protocol.

### Mixin vs Composition

This is the contested case. The rule:

> **Default to composition.** Reach for a mixin only when one of these is true:
>
> 1. You need to override dunder methods (`__init__`, `__repr__`, `__eq__`) on the host class.
> 2. A framework requires inheritance (SQLAlchemy declarative bases, Django models, Pydantic v1 base classes).
> 3. The behavior is genuinely stateless and dependency-free (e.g. a pure formatting helper).

If none apply, hold the collaborator as an attribute instead. The `TransactionMixin` example above fails all three conditions — it has a state dependency, no dunder override, no framework requirement — so it should have been composition from the start.

### Inheritance vs capability

If the only thing the consumer cares about is *"can this object do X?"*, the abstraction is a Protocol. If the consumer cares *"is this object a Y?"* (because it will dispatch on type, share lifecycle, or live in a registry), it is an ABC.

## Recommended Pattern: Protocol → Composition (or Mixin) → Concrete

A clean Ports-and-Adapters layout:

```python
# Port — application/domain layer
from typing import Protocol


class UserRepository(Protocol):
    def get_by_id(self, id: str) -> User: ...
    def save(self, user: User) -> None: ...


# Concrete adapter — infrastructure layer
from sqlalchemy.orm import Session


class SqlUserRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def get_by_id(self, id: str) -> User:
        return self._session.get(User, id)

    def save(self, user: User) -> None:
        self._session.add(user)
```

`SqlUserRepository` satisfies `UserRepository` structurally — no `class SqlUserRepository(UserRepository)` declaration needed. Tests can substitute a `FakeUserRepository` with the same shape and zero ceremony.

If the adapter genuinely needs reusable behavior (retry, instrumentation), prefer **composition** — inject a retrier or tracer in the constructor. Use a mixin only when the rules above genuinely apply.

See also: [`./langgraph.md`](./langgraph.md), [`./dependency-injector.md`](./dependency-injector.md) — both use DDD ports in this codebase.

## Pitfalls

- `**isinstance(x, SomeProtocol)`** requires `@runtime_checkable` and only checks method-name presence. Do not rely on it for contract enforcement — it will pass classes with wrong arities and wrong types.
- **Mypy structural-typing errors** from Protocol mismatches can be confusing; the message names the offending method but the actual divergence is often a parameter name or default-argument difference. When debugging, compare full signatures, not just method existence.
- **Heavy mixin chains in domain models** fragment business behavior across inheritance — keep mixins infrastructure-only.
- **ABCs do not render cleanly in `help()`** — abstract methods show as regular methods in interactive help. Document the contract explicitly in docstrings or a README.
- **Forgetting `super().__init__()`** in an ABC subclass that defines its own `__init__` silently skips base-class state setup. Common with `DomainEvent`-style ABCs that initialize IDs/timestamps in the base.

See also: [`typing.runtime_checkable`](https://docs.python.org/3/library/typing.html#typing.runtime_checkable).
