# khorsyio

Async Python framework for building event-driven applications with block-based architecture.

---

## What it is

khorsyio is a minimalist async Python framework where all business logic is built from isolated handlers connected through an internal event bus. Each block receives a strictly typed input and returns a strictly typed output. Blocks have no knowledge of each other — only of their own data.

The framework is designed so that building from scratch, including with LLM-assisted development, is as predictable and structured as possible.

---

## Why it works better with LLM

The block-based architecture of khorsyio naturally aligns with how language models generate code most effectively.

**Isolated blocks.** Each Handler is a self-contained unit with explicitly defined dependencies, input, and output. An LLM generates one block at a time without needing to hold the entire system in context.

**Structs as contracts.** `msgspec.Struct` is simultaneously documentation, validation, and an API description for the block. You define the structs first, then ask the LLM to implement the logic. The contract is established before a single line of logic is written.

**Event graph as architecture.** Data flow is described with strings — `subscribes_to` and `publishes`. You can explain this to an LLM in one paragraph and it will reproduce the correct chain without routing errors.

**Minimal boilerplate.** Serialization, context, tracing, DI — all automatic. The LLM only writes business logic inside `process`.

---

## Key features

**Event bus.** Async event delivery with timeouts, scheduler, metrics, and an event log. Supports publish/subscribe, request/response, and fan-out.

**HTTP + WebSocket out of the box.** Router with path parameters, middleware, CORS. SocketTransport built on socket.io for bidirectional connections.

**Automatic DI.** When mounting a Domain, dependencies are injected by constructor parameter name: `db`, `client`, `bus`, `transport`.

**Database.** A wrapper over SQLAlchemy AsyncEngine with simple methods `fetch`, `fetchrow`, `execute`, and session support. Utilities for filtering, sorting, and pagination.

**Multi-core.** CPU-bound handlers run in separate processes via `execution_mode = "process"` without changing any other code.

**Decomposition and tracing.** `trace_id` is automatically propagated through the entire chain. Metrics and event log are available out of the box.

---

## Quick start

```python
from khorsyio import App, Response, CorsConfig

app = App(cors=CorsConfig())
app.router.get("/health", lambda req, send: Response.ok(send, status="ok"))

if __name__ == "__main__":
    app.run()
```

Event handler:

```python
import msgspec
from khorsyio import Handler, Context

class UserIn(msgspec.Struct):
    name: str = ""

class UserOut(msgspec.Struct):
    id: int = 0
    name: str = ""

class CreateUser(Handler):
    subscribes_to = "user.create"
    publishes = "user.created"
    input_type = UserIn
    output_type = UserOut

    def __init__(self, db):
        self._db = db

    async def process(self, data: UserIn, ctx: Context) -> UserOut:
        row = await self._db.fetchrow(
            "insert into users (name) values ($1) returning id, name", data.name)
        return UserOut(**row)
```

---

## Repositories

- [khorsyio](https://github.com/khorsyio/khorsyio) — framework
- [khorsyio-docs](https://github.com/khorsyio/khorsyio-docs) — documentation
- [khorsyio-demos](https://github.com/khorsyio/khorsyio-demos) — examples and templates

---

## Installation

```bash
pip install khorsyio
```

---

## License

MIT
