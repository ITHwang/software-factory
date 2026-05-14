# Choosing Typed Data Structures

> Last updated: 2026-05-11

## TL;DR

How to pick the right typed container in Python: `dict[K, V]`, `TypedDict`, `dataclass`, `BaseModel`, `tuple`, `namedtuple`, `Literal`, `Enum`, `Protocol`.

**Use this when:**
- modelling structured data and unsure which container to reach for
- choosing between `TypedDict` and `dataclass` for an internal record type
- deciding when validation (`BaseModel`) is worth the runtime cost

**Don't use this for:**
- SQL-row models → `./sqlalchemy.md` (uses `SQLModel`)
- LangGraph agent state schemas → `./langgraph.md#building-a-graph`
- LangChain tool input schemas → `./langchain.md#tool-io`

Python offers `dict[K, V]`, `TypedDict`, `dataclass`, `BaseModel`, `tuple`, `namedtuple`, `Literal`, `Enum`, and `Protocol` for representing structured data.


## Table of Contents

| Phase               | Section                                                                                                                                                                                                                                                                                                                |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Decide           | [Quick Reference](#quick-reference), [The Two Questions](#the-two-questions)                                                                                                                                                                                                                                           |
| 2. Pick a structure | [dict](#dictk-v--homogeneous-mappings), [TypedDict](#typeddict--heterogeneous-schema-typed-dicts), [dataclass](#dataclass--lightweight-typed-objects), [BaseModel](#basemodel--validated-boundary-objects), [tuple](#tuple--temporary-positional-data), [namedtuple](#namedtuple--mostly-superseded) |
| 3. Constrain values | [Literal vs Enum](#literal-vs-enum), [Protocol](#protocol--structural-typing)                                                                                                                                                                                                                                    |
| 4. Avoid            | [Pitfalls](#pitfalls)                                                                                                                                                                                                                                                                                                  |


## Quick Reference


| Situation                                     | Use                                  | Why                                    |
| --------------------------------------------- | ------------------------------------ | -------------------------------------- |
| All values share one type                     | `dict[K, V]`                         | Homogeneous mapping; no schema         |
| Fixed-key heterogeneous JSON-shaped data      | `TypedDict`                          | Static typing on a real `dict` runtime |
| Internal trusted object                       | `dataclass`                          | Real object, no validation cost        |
| Trust boundary (HTTP body, env, external API) | `BaseModel`                          | Runtime validation, parsing, coercion  |
| Immutable value object                        | `dataclass(frozen=True, slots=True)` | Hashable, lightweight, no `__dict__`   |
| Short positional return                       | `tuple`                              | Cheap, unpackable                      |
| Constrained string/int values, no behavior    | `Literal[...]`                       | Type narrowing only                    |
| Shared categorical values with behavior       | `Enum` / `StrEnum`                   | Iterable, methods, stable identity     |
| Describe a shape you don't own                | `Protocol`                           | Structural typing for ports            |


## The Two Questions

Most ambiguity collapses if you answer these in order:

1. **Does the runtime need to remain a `dict`?** (Dumped to JSON, sent to Redis, passed to a library that expects a dict, used as graph state.)
  - Yes → `dict[K, V]` (homogeneous) or `TypedDict` (heterogeneous fixed schema).
  - No → an object: `dataclass` or `BaseModel`.
2. **Does the data cross a trust boundary that warrants runtime validation/coercion?** (Untrusted input, parsed external JSON, env config.)
  - Yes → `BaseModel`.
  - No → `dataclass`.

Both axes matter independently. A boundary `BaseModel` typically parses incoming JSON, then constructs a `dataclass` for the rest of the system to work with.

## `dict[K, V]` — Homogeneous Mappings

When every value shares the same type, a parameterized `dict` is the right tool. Use it for lookup tables, caches, counters, indexes, and configuration maps.

```python
scores: dict[str, int] = {"math": 95, "english": 88}
ratings: dict[str, list[str]] = {}
```

Reach for something else the moment values stop being uniform — that's the line into `TypedDict` or an object.

## `TypedDict` — Heterogeneous Schema-Typed Dicts

Use `TypedDict` when the runtime *must* stay a `dict` — JSON payloads, Redis values, Kafka messages, LangGraph state, anything serialized at the boundary or passed to a library expecting a mapping. It gives static type checkers a fixed schema while leaving the object as a normal `dict` at runtime.

```python
from typing import TypedDict, NotRequired


class UserPayload(TypedDict):
    name: str
    age: int
    is_admin: NotRequired[bool]
```

- `NotRequired[T]` / `Required[T]` mark individual keys as optional or mandatory.
- `total=False` flips the default so every key is optional unless wrapped in `Required[T]`.
- `TypedDict` performs **no runtime validation** — it only informs the type checker. If validation is needed at this boundary, that's a `BaseModel`.

`./langgraph-graph-architecture.md` is the canonical example: graph state is a `TypedDict` so LangGraph can merge updates by key.

## `dataclass` — Lightweight Typed Objects

`dataclass` is the default for **trusted internal objects** — domain models, service-layer DTOs, value objects, command/query objects, persistence records. It's a real object with attributes, methods, equality, and (optionally) immutability, with no runtime validation overhead.

```python
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True, kw_only=True)
class Money:
    amount: int
    currency: str = "USD"
    tags: list[str] = field(default_factory=list)

    def add(self, other: "Money") -> "Money":
        return Money(amount=self.amount + other.amount, currency=self.currency)
```

- `frozen=True` — instances are immutable and hashable; safe to use as dict keys or set members.
- `slots=True` — no `__dict__` per instance, lower memory, faster attribute access.
- `kw_only=True` — forces keyword arguments at construction, eliminating positional-argument confusion as fields grow.
- `field(default_factory=...)` — required for any mutable default (`list`, `dict`, `set`); a bare `= []` would share one list across all instances.

`dataclass` is not a DDD-only construct. It's a lightweight typed object and belongs anywhere in the application where a `dict` would lose meaning.

## `BaseModel` — Validated Boundary Objects

Use Pydantic `BaseModel` at trust boundaries: FastAPI request/response bodies, environment configuration, parsed external API responses, anything where runtime validation, coercion, and structured error reporting are worth the cost.

A common architecture: a `BaseModel` parses external JSON, then hands a validated `dataclass` to the rest of the system.

```python
from dataclasses import dataclass
from pydantic import BaseModel


class CreateUserRequest(BaseModel):
    name: str
    age: int


@dataclass(frozen=True, slots=True, kw_only=True)
class User:
    name: str
    age: int


def to_domain(req: CreateUserRequest) -> User:
    return User(name=req.name, age=req.age)
```

This keeps validation, parsing, and business logic in separate layers.

**Don't default to `BaseModel` everywhere.** It revalidates fields on every construction, which is unnecessary cost on hot paths or for objects built internally from already-trusted data. If the data has already crossed the boundary, `dataclass` is faster and cleaner.

## `tuple` — Temporary Positional Data

Tuples are appropriate when the structure is short-lived, the positional meaning is obvious, and no behavior is needed.

```python
size: tuple[int, int] = (width, height)
min_value, max_value = bounds
head, tail = os.path.split(path)
```

**Anti-rule:** the moment a tuple grows a third semantic field, or starts crossing module boundaries, or anyone has to ask "wait, which one was first?" — promote it to a `dataclass`. Position is a fragile contract.

## `namedtuple` — Mostly Superseded

`dataclass(frozen=True, slots=True)` covers nearly every case `namedtuple` used to: named fields, immutability, equality, hashing — plus methods, inheritance, and better type-checker support.

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class Point:
    x: int
    y: int
```

The legitimate remaining use cases for `namedtuple` (and its typed cousin `typing.NamedTuple`) are tuple-interop scenarios — destructuring into `*args`, comparing to legacy tuple APIs, or matching the shape of objects like SQLAlchemy `Row`. Outside those, prefer a frozen dataclass.

## `Literal` vs `Enum`

Both constrain a value to a fixed set. The decision is **behavior**, not scope.

- `**Literal`** — when you want type narrowing on an argument or return value and nothing else. No runtime object, no methods, no iteration.
  ```python
  from typing import Literal


  def render(mode: Literal["draft", "final"]) -> str: ...
  ```
  Best for: function arguments that take a small fixed set of strings, type-narrowing return values (`def kind(self) -> Literal["pdf", "html"]`), tagged-union discriminators inside a `TypedDict` or `BaseModel`.
- `**Enum**` (or `StrEnum`) — when you need behavior: `.value` access, iteration over members, methods on members, `isinstance` checks, or stable identity referenced from multiple modules.
  ```python
  from enum import StrEnum


  class DocumentType(StrEnum):
      PDF = "pdf"
      HTML = "html"

      def is_binary(self) -> bool:
          return self is DocumentType.PDF
  ```
  `StrEnum` (Python 3.11+) is the right choice when the value round-trips through JSON or a database — the enum member *is* its string value, so serialization is lossless. See `[./preprocessor-pipeline.md](./preprocessor-pipeline.md)` for an `enum + to_class()` dispatch pattern that uses this exact property.

If you find yourself adding methods to a `Literal` (you can't), or grepping a project for hardcoded string literals to keep them in sync, you wanted an `Enum`.

## `Protocol` — Structural Typing

Use `Protocol` when you need to describe a shape without owning it — capability contracts (ports), duck-typed interfaces, or third-party objects you don't control.

```python
from typing import Protocol


class UserRepository(Protocol):
    async def find(self, user_id: str) -> "User | None": ...
    async def save(self, user: "User") -> None: ...
```

Any class with matching method signatures satisfies it; no inheritance required. This is the right tool for ports in a Ports-and-Adapters layout.

For depth — Protocol vs ABC vs mixin, `@runtime_checkable` caveats, default method bodies — see `[./abstractions.md](./abstractions.md)`.

## Pitfalls

- **Mutable defaults in dataclasses.** `tags: list[str] = []` shares one list across every instance. Always use `field(default_factory=list)`.
- **Plain `dataclass` on SQLAlchemy-mapped classes.** The ORM owns instance construction and attribute access; layering a `@dataclass` on top conflicts with mapped attributes. Let the ORM base class handle it (e.g., SQLModel, `DeclarativeBase`).
- `**BaseModel` on hot paths or already-trusted internal data.** Per-construction revalidation isn't free. Validate once at the boundary, hand off `dataclass` instances after that.
- **A tuple that grew a third semantic field.** Position-only contracts break silently. Promote to `dataclass` the first time you reach for `result[2]`.
- `**dict[str, Any]` for known-shape JSON.** That's a `TypedDict`. `Any` discards every guarantee the type checker could give you.
- **Reaching for `Literal` when you wanted behavior.** If you ever want `.value`, iteration, or a method on a member, you wanted `Enum` from the start.

## See Also

- `[./python-guidelines.md](./python-guidelines.md)` — overall typing conventions and Data Modeling & Validation defaults.
- `[./abstractions.md](./abstractions.md)` — Protocol, ABC, mixin, and composition in depth.
- `[./preprocessor-pipeline.md](./preprocessor-pipeline.md)` — `StrEnum` + `to_class()` dispatch as a worked example.
- `[./langgraph-graph-architecture.md](./langgraph-graph-architecture.md)` — `TypedDict` as graph state schema.
