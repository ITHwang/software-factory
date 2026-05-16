# Python Logging

> Last updated: 2026-05-16

## TL;DR

Loguru for logging. Initialize once in the entrypoint with `logger.remove()` + `logger.add(sys.stdout, level="INFO")`. Levels: DEBUG / INFO / WARNING / ERROR.

**Use this when:**
- bootstrapping logging in a new script/service
- picking a level for a log statement

**Don't use this for:**
- error class design → [`./py-errors.md`](./py-errors.md)

## Initialization

Initialize logger in the main script:

```python
# Main script example (e.g., __main__.py)

import sys

from loguru import logger

logger.remove()
logger.add(sys.stdout, level="INFO")
```

## Levels

| Level | Purpose | Examples |
|-------|---------|----------|
| DEBUG | Detailed debugging info | Function entry/exit, variable values, branching |
| INFO | Normal service flow | State changes, successful operations, system health |
| WARNING | Requires attention, recoverable | Potential issues, degraded performance |
| ERROR | Serious problems | Failed requests, exceptions, unrecoverable states |

## See Also

- [`./py-errors.md`](./py-errors.md) — global exception handler logs through Loguru.
