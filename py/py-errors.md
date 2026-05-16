# Python Errors

> Last updated: 2026-05-16

## TL;DR

Domain errors are `CustomError` subclasses with hardcoded user-facing `error_msg` and diagnostic `error_log`. Services and adapters raise; global exception handlers log + map to HTTP. Never log inside error classes, never leak `error_log` to clients.

**Use this when:**
- defining a new domain error
- wiring or reviewing a global exception handler
- reviewing a PR for error-discipline violations

**Don't use this for:**
- logger init/config → [`./py-logging.md`](./py-logging.md)
- FastAPI status code lookup → FastAPI docs
- concrete auth error example → [`./py-auth.md`](./py-auth.md)

## Error Architecture Principles

- **Logging separation**: Log only in global exception handlers, not in error classes.
- **Hardcoded messages**: Each error class provides a user-friendly message.
- **Detailed logs**: Pass debug details via `error_log` argument.
- **Security**: Never expose internal logs in client responses.

## Error Handling Flow

1. Lower-level layers raise custom errors (no logging).
2. Errors propagate automatically (no try-catch needed).
3. Global exception handlers catch errors and log tracebacks.
4. Use structured error codes and FastAPI status codes.

## CustomError Base Class

**Purpose**: Base class for domain errors; centralizes the separation between user-facing `error_msg` and diagnostic `error_log`.

**Responsibilities**:
- Owns the HTTP `status_code`, structured `error_code`, hardcoded user-facing `error_msg`, and diagnostic `error_log` carried alongside the exception.
- Enforces a keyword-only constructor signature so subclasses cannot accidentally swap user-facing and diagnostic fields by position.
- Must not perform logging — that responsibility belongs to the global exception handler at the framework boundary.
- Must not leak `error_log`, stack traces, or other diagnostic content into client responses; only `error_msg` and `error_code` are safe to surface.
- Must not be raised directly — subclass it (e.g. `DocumentNotFoundError`) so each failure mode owns its hardcoded message and status code.

```python
from fastapi import status


class CustomError(Exception):
    """Base class for custom errors

    Related convention: error-suffix-on-exception-name (N818)
    See: https://docs.astral.sh/ruff/rules/error-suffix-on-exception-name/
    """

    def __init__(
        self,
        *,
        error_msg: str,
        error_log: str | None = None,
        error_code: str | None = None,
        status_code: int | None = None,
    ) -> None:
        """Initialize a custom error.

        Args:
            error_msg (str): Hardcoded user-friendly message
            error_log (str | None): Diagnostic details for debugging (not sent to client)
            error_code (str | None): Structured error code (e.g., "DOCUMENT_NOT_FOUND")
            status_code (int | None): HTTP status code(only if web application, use fastapi.status)

        Example:
            class DocumentNotFoundError(CustomError):
                def __init__(self, error_log: str) -> None:
                    error_msg = "Document not found"  # Hardcoded
                    super().__init__(
                        status_code=status.HTTP_404_NOT_FOUND,
                        error_code="DOCUMENT_NOT_FOUND",
                        error_msg=error_msg,
                        error_log=error_log,  # Diagnostic info
                    )
        """
        self.status_code = status_code
        self.error_code = error_code
        self.error_msg = error_msg
        super().__init__(error_log if error_log else "No Error Log")
```

## Defining A Custom Error

```python
class DocumentNotFoundError(CustomError):
    def __init__(self, error_log: str) -> None:
        error_msg = "Document not found"  # Hardcoded user-facing message
        super().__init__(
            status_code=status.HTTP_404_NOT_FOUND,
            error_code="DOCUMENT_NOT_FOUND",
            error_msg=error_msg,
            error_log=error_log,  # Diagnostic details
        )
```

## See Also

- [`./py-backend-architecture.md`](./py-backend-architecture.md) — Services And FastAPI: services raise, routes/global-handler translate.
- [`./py-logging.md`](./py-logging.md) — where the global exception handler logs through.
- [`./py-auth.md`](./py-auth.md) — `AuthenticationError` as a concrete subclass.
