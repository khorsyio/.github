# khorsyio

Async Python фреймворк для создания событийно-ориентированных приложений блочной архитектуры.

---

## Что это

khorsyio — это минималистичный async Python фреймворк, где вся бизнес-логика строится из изолированных обработчиков (Handler), связанных через внутреннюю шину событий. Каждый блок получает строго типизированный вход и возвращает строго типизированный выход. Блоки не знают друг о друге — только о своих данных.

Фреймворк спроектирован так, чтобы разработка с нуля, в том числе с помощью LLM-инструментов, была максимально предсказуемой и структурированной.

---

## Почему это работает лучше с LLM

Блочная архитектура khorsyio естественно совпадает с тем, как языковые модели лучше всего генерируют код.

**Изолированные блоки.** Каждый Handler — самодостаточная единица с явно описанными зависимостями, входом и выходом. LLM генерирует один блок за раз без необходимости удерживать в контексте всю систему.

**Структуры как контракт.** `msgspec.Struct` — это одновременно документация, валидация и описание API блока. Сначала описываешь структуры, потом просишь LLM реализовать логику. Контракт задан до того, как написана первая строка логики.

**Граф событий как архитектура.** Поток данных описывается строками — `subscribes_to` и `publishes`. Это можно объяснить LLM в одном абзаце, и она воспроизведет правильную цепочку без ошибок в маршрутизации.

**Минимальный boilerplate.** Сериализация, контекст, трассировка, DI — всё автоматически. LLM пишет только бизнес-логику внутри `process`.

---

## Ключевые возможности

**Событийная шина.** Асинхронная доставка событий с таймаутами, планировщиком, метриками и журналом. Поддерживает publish/subscribe, request/response и fan-out.

**HTTP + WebSocket из коробки.** Router с path-параметрами, middleware, CORS. SocketTransport на базе socket.io для двунаправленных соединений.

**Автоматический DI.** При монтировании Domain зависимости инжектируются по имени параметра конструктора: `db`, `client`, `bus`, `transport`.

**Database.** Обертка над SQLAlchemy AsyncEngine с простыми методами `fetch`, `fetchrow`, `execute` и поддержкой сессий. Утилиты фильтрации, сортировки и пагинации.

**Многоядерность.** CPU-bound обработчики запускаются в отдельных процессах через `execution_mode = "process"` без изменения остального кода.

**Декомпозиция и трассировка.** `trace_id` автоматически прокидывается через всю цепочку. Метрики и event log доступны из коробки.

---

## Быстрый старт

```python
from khorsyio import App, Response, CorsConfig

app = App(cors=CorsConfig())
app.router.get("/health", lambda req, send: Response.ok(send, status="ok"))

if __name__ == "__main__":
    app.run()
```

Обработчик события:

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

## Репозитории

- [khorsyio](https://github.com/khorsyio/khorsyio) — фреймворк
- [khorsyio-docs](https://github.com/khorsyio/khorsyio-docs) — документация
- [khorsyio-demos](https://github.com/khorsyio/khorsyio-demos) — примеры и шаблоны

---

## Установка

```bash
pip install khorsyio
```

---

## Лицензия

MIT
