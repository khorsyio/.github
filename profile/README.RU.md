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

## Сравнение с аналогами

В Python нет прямого аналога. Ближайшие инструменты решают только часть задачи.

`python-cqrs`, `pymediator`, `python-mediator` — реализуют паттерн медиатора с типизированными командами и хендлерами, концептуально близко. Но это библиотеки без HTTP, WebSocket, DB и scheduler. Поверх них нужно самостоятельно собирать стек из FastAPI или Litestar, SQLAlchemy, APScheduler и python-socketio.

`FastAPI` + `Litestar` — зрелые HTTP-фреймворки с DI и типизацией. Litestar использует тот же `msgspec`. Но они решают задачу HTTP API, а не событийной цепочки. Внутреннюю шину с блоками нужно строить поверх них самостоятельно.

`bubus`, `messagebus` — production event bus библиотеки с async поддержкой. Технически зрелее по части шины, но без HTTP-слоя и DB.

Реальное отличие khorsyio — full-stack монолитность: in-process event bus, HTTP ASGI, WebSocket, DB, HTTP client, scheduler, DI и multiprocessing в одном пакете с единым паттерном. Альтернатива этому — стек из 5-6 отдельных библиотек со своими паттернами интеграции.

---

## TODO

Вещи, которых пока нет, но которые нужны для production-ready использования.

**OpenAPI / Swagger.** FastAPI и Litestar автоматически генерируют документацию из типов. В khorsyio этого нет. Для API-first разработки и командной работы — существенный пробел.

**Тестовые утилиты.** Нет test client для HTTP, нет mock bus для изолированного тестирования хендлеров. Тестирование приходится выстраивать вручную.

**Type-safe DI.** Текущий DI по имени строки параметра (`db`, `client`) — магичен и не верифицируется при старте. Опечатка или незнакомое имя тихо подставляет `app` вместо нужной зависимости. Нужна явная проверка графа зависимостей при `startup`.

**Retry и dead letter queue.** Шина не имеет встроенной политики повторных попыток при ошибке хендлера и механизма складирования необработанных событий.

**Observability.** Метрики и event log есть, но нет интеграции с OpenTelemetry, Prometheus или structured logging. В production-сценарии это нужно строить самостоятельно.

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
