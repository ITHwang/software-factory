# Python Guidelines

## Quick Reference

| Aspect | Standard |
|--------|----------|
| Python | 3.14.x |
| Package Manager | uv |
| Linter/Formatter | ruff |
| Style Guide | Google + PEP 8 |
| Web Framework | FastAPI |
| CLI Framework | Typer |
| Testing | Pytest |
| Logging | Loguru |

## Development Environment

- Python Version: **3.14.x**
- Dependency Management & Package Manager: [uv](https://docs.astral.sh/uv/)
- Environment variables: Load from `.env` file with [python-dotenv](https://github.com/theskumar/python-dotenv)
- Code formatting/linting: [ruff](https://docs.astral.sh/ruff/). See Appendix.
- Web Programming:
  - [nginx](https://nginx.org/) (web server)
  - [FastAPI](https://fastapi.tiangolo.com/) (web framework)
  - [uvicorn](https://www.uvicorn.org/) (ASGI server)
  - [gunicorn](https://gunicorn.org/) (WSGI server)
  - [SQLAlchemy](https://www.sqlalchemy.org/) (ORM)
  - [Dependency Injector](https://python-dependency-injector.ets-labs.org/) (DI framework)
- CLI Programming:
  - [Typer](https://typer.tiangolo.com/) (CLI framework)
  - Use `asyncio.run()` in functions decorated with `@app.command()` for async CLI apps.
- Data Modeling & Validation:
  - [Pydantic](https://docs.pydantic.dev/latest/)
  - [dataclass](https://docs.python.org/3/library/dataclasses.html)
- Testing: [Pytest](https://docs.pytest.org/en/stable/)
- Logging: [Loguru](https://loguru.readthedocs.io/en/stable/). See Appendix.
- Exception handling: Define custom errors inheriting from `CustomError(Exception)`. See Appendix.

## Project Structure

```
├── src/
│   ├── api/              
│   │   ├── models/          # Persistence layer
│   │   ├── schemas/         # API layer (Pydantic)
│   │   ├── routers/
│   │   ├── services/
│   │   └── repositories/
│   ├── scripts/              # CLI, scripts
│   │   └── models/
│   ├── core/
│   │   ├── models/
│   │   ├── schemas/
│   │   ├── repositories/
│   │   └── errors/
│   ├── utils/
│   └── main.py
├── tests/
├── Makefile
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
├── README.md
├── .gitignore
├── .env.sample
└── uv.lock
```

## Coding Conventions

- By default, follow [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html) and [PEP 8](https://peps.python.org/pep-0008/)
- Principles:
  - **Naming**: Use self-descriptive and meaningful naming for variables, functions, and classes.
  - **Simplicity**: Avoid complexity and write simple and clear code.
  - **Security**: Avoid security vulnerabilities and minimize bugs.
  - **List Comprehension**: Use list comprehension when appropriate, instead of traditional loops.
  - **Type Hints**: Use type hints for code readability and type checking.
  - **Global Variables**: Avoid global variables to minimize side effects.
- Comments:
  - **Package Comments**: Create `README.md` in the package folder documenting shared concepts.
  - **Module Comments**: Add docstrings at the top of each module file (`*.py`).
  - **Function/Class Comments**: Use Google-style docstrings (summary + blank line + details).
      ```python
      def greet_user(name: str) -> str:
          """Generate a greeting message for the given user name.

          Keep the name in uppercase for consistency.

          Args:
              name (str): The user name

          Returns:
              str: A greeting message starting with uppercase
          """
          if not name:
              raise ValueError("Name is required.")

          return f"Hello, {name.capitalize()}!"
      ```
  - **Inline Comments**: Comment complex logic only; avoid excessive commenting.
- **`@override` Decorator**: Always use for methods overriding parent class or implementing abstract methods. Import from `typing`.
  - Example:
    ```python
    from typing import override


    class ChildRepository(BaseRepository):
        @override
        async def find_by_id(self, id: str) -> Model:
            return await super().find_by_id(id)
    ```
- **Testing**: TBD

## Appendix

### Loguru Usage

- Initialize logger in the main script:
  ```python
  # Main script example (e.g., __main__.py)

  import sys

  from loguru import logger

  logger.remove()
  logger.add(sys.stdout, level="INFO")
  ```
- Logging levels:

| Level | Purpose | Examples |
|-------|---------|----------|
| DEBUG | Detailed debugging info | Function entry/exit, variable values, branching |
| INFO | Normal service flow | State changes, successful operations, system health |
| WARNING | Requires attention, recoverable | Potential issues, degraded performance |
| ERROR | Serious problems | Failed requests, exceptions, unrecoverable states |

### Exception Handling

- Error Handling Flow:
  1. Lower-level layers raise custom errors (no logging).
  2. Errors propagate automatically (no try-catch needed).
  3. Global exception handlers catch errors and log tracebacks.
  4. Use structured error codes and FastAPI status codes.

- Error Architecture Principles:
  - **Logging separation**: Log only in global exception handlers, not in error classes.
  - **Hardcoded messages**: Each error class provides a user-friendly message.
  - **Detailed logs**: Pass debug details via `error_log` argument.
  - **Security**: Never expose internal logs in client responses.

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
    ):
        """Initialize a custom error.

        Args:
            error_msg (str): Hardcoded user-friendly message
            error_log (str | None): Diagnostic details for debugging (not sent to client)
            error_code (str | None): Structured error code (e.g., "DOCUMENT_NOT_FOUND")
            status_code (int | None): HTTP status code(only if web application, use fastapi.status)

        Example:
            class DocumentNotFoundError(CustomError):
                def __init__(self, error_log: str):
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


# Example custom error definition
class DocumentNotFoundError(CustomError):
    def __init__(self, error_log: str):
        error_msg = "Document not found"  # Hardcoded user-facing message
        super().__init__(
            status_code=status.HTTP_404_NOT_FOUND,
            error_code="DOCUMENT_NOT_FOUND",
            error_msg=error_msg,
            error_log=error_log,  # Diagnostic details
        )
```