# Rust Guidelines

## Quick Reference

| Aspect | Standard |
|--------|----------|
| **Indentation** | 4 Spaces (No Tabs) |
| **Line Width** | 100 Characters |
| **Comment Width** | 80 Characters |
| **Sorting** | "Version Sorting" (e.g., x8 before x16) |
| **Imports** | `self`/`super` first, then alphabetically |
| **Trailing Commas** | Required in multi-line lists |

## Overview & Philosophy

- **Tooling First**: Use `rustfmt` to automate formatting.
- **Consistency**: Adhere to default style for familiar code across projects.
- **Version Sorting**: When sorting (imports, match arms), treat numeric chunks as numbers (`x8` before `x16`).

## Formatting Conventions

### Indentation and Alignment

- Always prefer **block indent** over visual indent.
- Visual indent (lining up with opening parenthesis) causes rightward drift and larger diffs.

```rust
// PREFERRED: Block Indent
a_function_call(
    foo,
    bar,
);

// AVOID: Visual Indent
a_function_call(foo,
                bar);
```

### Trailing Commas

Add trailing comma in any comma-separated list followed by newline.

### Code Structure

- Separate items by zero or one blank line (never two or more).
- No trailing whitespace.
- Attributes: Place each on its own line, combine derives into single `#[derive(...)]`.

### rustfmt Configuration

```toml
# rustfmt.toml
max_width = 100
comment_width = 80
```

## Naming & Ordering

| Item Type | Case Style | Example |
|-----------|------------|---------|
| Types (Structs, Enums, Traits) | `UpperCamelCase` | `UserRepository` |
| Enum Variants | `UpperCamelCase` | `Option::Some` |
| Functions / Methods | `snake_case` | `calculate_total` |
| Variables / Arguments | `snake_case` | `user_id` |
| Modules / Macros | `snake_case` | `my_module` |
| Constants / Statics | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES` |

### Module Headers Order

1. `extern crate` statements (if necessary)
2. `use` statements (imports)
3. Module declarations (`mod foo;`)

### Import List Order

Inside nested `use std::{...}`:

1. `self`
2. `super`
3. Normal identifiers (sorted)
4. Groups and glob imports (`*`) last

### Module File Hierarchy (Rust 2018+)

Prefer flatter module structures. Use file-level modules:

- **Single file module**: `filename.rs`
- **Directory module**: `filename.rs` + `filename/` folder with submodules
- **Avoid `mod.rs`**: Causes "Editor Tab Problem" with multiple indistinguishable tabs

```text
src/
├── main.rs
├── auth.rs          # Module definition
└── auth/            # Submodules
    ├── login.rs
    └── logout.rs
```

## Documentation & Comments

- Prefer line comments (`//`) over block comments (`/* */`).
- Single space after `//` sigil.
- Comment lines limited to 80 characters (code can be 100).
- Doc comments: `///` for outer (functions, structs), `//!` for inner (top of lib.rs).
- Put doc comments before attributes.
- Doc comments should explain *why*, not only *what*.
- Keep comments near the code they describe.

## Control Flow & Expressions

- No extraneous parentheses in `if`/`while` conditions.
  - Correct: `if x > 5 { ... }`
  - Incorrect: `if (x > 5) { ... }`
- Match arms: Break after opening brace, block-indent arms, no leading pipe on first pattern.
- Avoid wildcard `_` unless truly intentional. Prefer explicit variant matching for better error messages:

```rust
// AVOID
_ => bail!("something failed"),

// PREFER
unexpected => bail!("unexpected variant: {:?}", unexpected),
```

## Cargo.toml Conventions

- Keys within sections should be version-sorted.
- `[package]` section at top: `name` and `version` first, `description` at end.
- Do not quote standard key names.
- Feature flags should be additive, never mutually exclusive.
- Each crate should expose one clear responsibility (SRP).

## Development Environment

| Aspect | Standard |
|--------|----------|
| Rust | Stable (Edition 2021) |
| Package Manager | Cargo |
| Linter | Clippy (`-D warnings` in CI) |
| Formatter | rustfmt |
| Async Runtime | Tokio |
| HTTP Client | Reqwest |
| HTTP Server | Axum |
| CLI Framework | Clap (derive) |
| Serialization | Serde + serde_yaml |
| Error Handling | anyhow + thiserror |
| Logging | tracing + tracing-subscriber |
| Testing | Built-in + rstest + tokio::test |
| Environment | dotenvy |

- Avoid `serde_json::Value` unless necessary.

## Error Handling

- Use `anyhow::Result` for application errors.
- Add context with `.context("descriptive message")?`.
- Use `anyhow::bail!()` for early error returns.
- Pattern match for explicit error handling when needed.
- **Library crates should NOT use `anyhow`** — use `thiserror` instead.
  - `anyhow` is for applications (hides error types).
  - `thiserror` is for libraries (exposes typed errors).

## Async Patterns

- Use `#[tokio::main]` for async main.
- Batch processing with `tokio::spawn` and `futures::future::join_all`.
- Use `async move` to capture variables in spawned tasks.
- Clone Client and tokens per task for ownership.
- Prefer structured concurrency (`JoinSet`) over loose `tokio::spawn` to avoid orphan tasks.
- For concurrent I/O in loops, use `futures::stream::buffer_unordered`:

```rust
futures::stream::iter(chunks)
    .map(process)
    .buffer_unordered(16)
    .collect::<Result<Vec<_>>>()
    .await?;
```

## Testing

- Unit tests in same file with `#[cfg(test)]` module.
- Use `#[tokio::test]` for async tests.
- Use `rstest` for parameterized tests.
- Tests must not rely on real network calls (use `wiremock-rs` or mock traits).
- Avoid shared state between tests.
