# Coding Standards — Library Membership

Стандарты для AI-агентов при code review. Каждое правило имеет уровень: 🔴 **CRITICAL** (блокирующее), 🟡 **WARNING** (должно быть исправлено), 🔵 **INFO** (рекомендация).

> При написании кода (а не ревью) — сначала `docs/agents/architecture.md`: там таблица «куда класть код» и анти-паттерны. Этот файл — подробная рубрика ревью.

---

## 1. Layered Architecture

### 🔴 CRITICAL: Соблюдение dependency flow

Два раздельных потока — modules используется **только для write-операций**:

- **READ:** `entrypoints → application → repositories`
- **WRITE:** `entrypoints → application → modules → repositories`

Модули не вызывают другие модули. Модули не зависят от `application` слоя.

```python
# BAD: READ-операция заходит в modules (modules — только для write)
async def get_book_by_id(book_id: int):
    return await BookService.get_by_id(book_id)

# GOOD: READ-операция идёт напрямую в repository
async def get_book_by_id(book_id: int):
    return await BookRepository.get_by(BookFiltersDBSchema(id=book_id))
```

### 🔴 CRITICAL: Порядок реализации

При добавлении новой сущности: **Table → Schemas → Repository → Module → Entrypoint → Tests**. Нельзя реализовывать следующий слой, пока не готов предыдущий.

---

## 2. Database Tables (`db_schema/`)

### 🔴 CRITICAL: Современный SQLAlchemy синтаксис

Только `Mapped[]` и `mapped_column()`. Запрещён устаревший `Column()`.

```python
# GOOD
id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)

# BAD
id = Column(BigInteger, primary_key=True, autoincrement=True)
```

### 🔴 CRITICAL: Обязательные поля каждой таблицы

`id` (BigInteger PK), `is_active` (Boolean, `server_default=text("true")`), `created_at`, `updated_at` (DateTime timezone=True). Все таблицы наследуются от `db_schema.base.Table`.

### 🔴 CRITICAL: Soft-delete

Удаление только через `is_active = False`. Все запросы фильтруют `is_active = True`.

### 🟡 WARNING: Типы колонок

`BigInteger` для всех ID. `DateTime(timezone=True)` для временных меток. Имена таблиц: `snake_case`.

### 🔵 INFO: Комментарии на русском

Таблицы и колонки документируются комментариями на русском языке.

---

## 3. Schemas (`schemas/`)

### 🔴 CRITICAL: Разделение DB и Application схем

- `schemas/db/` — схемы для репозиториев: `Entity`, `EntityCreateDBSchema`, `EntityUpdateDBSchema`, `EntityFiltersDBSchema`
- `schemas/application/` — схемы для application: `EntityCreateSchema`, `EntityUpdateSchema`, `EntityFiltersSchema`, `EntityExtended`

```python
# GOOD: DB схема — плоская, без Field() декораторов
class Book(BaseModel):
    id: int
    title: str
    is_active: bool

# GOOD: Application схема — с валидацией
class BookCreateSchema(NormalizeTextMixin, BaseModel):
    title: str = Field(min_length=1, pattern=BOOK_TITLE_PATTERN)
```

### 🔴 CRITICAL: Update схемы — все поля Optional

`EntityUpdateDBSchema` и `EntityUpdateSchema` — все поля имеют `| None` для поддержки PATCH.

### 🟡 WARNING: Expand Enums

Для expands создаются два `StrEnum`: `EntityExpandEnum` (application слой) и `EntityDBExpandEnum` (DB слой). Расширенная схема `EntityExtended(Entity)` добавляет expand-поля как `list[...]` с `default_factory=list`.

---

## 4. Repositories (`repositories/`)

### 🔴 CRITICAL: Стандартные методы

Каждый репозиторий реализует: `get_list`, `get_by`, `create`, `update`, `delete`. По необходимости добавляются `create_many`, `update_many`, `delete_many` и агрегационные методы (`get_count`).

**Запрещены** методы `get_by_<attr>` / `get_list_by_<attr>` (например, `get_by_card_number`, `get_list_by_reader_id`). Одиночный поиск идёт через `get_by(filters)`, списковый — через `get_list(filters)`; атрибуты фильтра задаются полями `XxxFiltersDBSchema`.

Для write-операций с проверкой существования и конкурентной записью допускаются: `get_by(filters, for_update=True)` (SELECT … FOR UPDATE) и `create_if_not_exists(data)` (INSERT … ON CONFLICT DO NOTHING, возвращает `Entity | None`).

### 🔴 CRITICAL: Возврат Pydantic-схем

Методы возвращают Pydantic-схемы (не dict, не Record).

```python
# GOOD
@classmethod
async def get_by_id(cls, entity_id: int) -> Book | None:
    row = await library_db.fetch_one(stmt)
    return Book.model_validate(row) if row else None

# BAD
async def get_by_id(self, entity_id: int) -> dict | None:
    ...
```

### 🟡 WARNING: SelectExtended для joins

Для сложных запросов с отношениями используется `SelectExtended` с `Relation`. Методы `with_*` добавляют `jsonb_build_object` для связанных данных.

### 🟡 WARNING: Аннотации mypy

При использовании `SelectExtended` добавляется `# type: ignore[return-value]`.

---

## 5. Modules (`modules/`)

### 🔴 CRITICAL: Только для write-операций

Modules реализуют доменные операции **только изменяющие данные** (create, update, delete). Read-операции не проходят через модули — application слой вызывает repository напрямую. У модулей нет публичных read-методов (`get_by`, `get_list`).

Внутри write-операции сервис **может** читать текущее состояние через repository (в своей транзакции), когда это нужно для корректного сохранения (дифф для истории, проверка идемпотентности). Это внутренняя деталь реализации, а не часть read-пути.

### 🔴 CRITICAL: Структура: `service.py` + `rules/`

Модуль делится на две части:

- `modules/<entity>/service.py` — класс `<Entity>Service` для сохранения агрегата. Содержит логику записи данных: транзакцию, запись истории, отправку событий. Методы: `create`, `update`, `delete` или единый `upsert` для идемпотентного create-or-update по natural key.
- `modules/<entity>/rules/` — чистые доменные правила (функции). Группируются по доменному концепту — **один файл на концепт, не на функцию**; при разрастании концепта файл разбивается на несколько. Правила чистые: без обращения к БД/IO, не вызывают другие модули, не зависят от `application` или `entrypoints`. Вызываются из `application`.

### 🔴 CRITICAL: Исключения в modules

Модули (включая `rules/`) используют **только кастомные исключения**. Запрещены `HTTPException`, `RequestValidationError`.

```python
# GOOD
class BookNotFoundError(Exception):
    pass

# BAD
from fastapi import HTTPException
raise HTTPException(status_code=404, detail="Book not found")
```

### 🟡 WARNING: Именование по слоям

Имя говорит, **что** функция вычисляет, директория — **где** она живёт (и, следовательно, может ли ходить в БД/IO). Чистоту никогда не выводят из имени — только из расположения.

| Слой | Глаголы | Возвращает | БД/IO |
|---|---|---|---|
| `rules/` | `is_*`, `has_*` | `bool` | нет |
| | `validate_*`, `assert_*` | доменное исключение | нет |
| | `resolve_*`, `derive_*`, `select_*`, `choose_*`, `compute_*` | чистое значение | нет |
| `application` | `create_*`, `update_*`, `delete_*`, `get_*`, `apply_*` (use case) | результат use case | да |
| | `enrich_<entity>` | расширенная сущность | да |
| | `resolve_*`, `derive_*` (семантическое вычисление) | значение | да |
| | `fetch_*`, `load_*` (сырой сбор данных) | данные | да |

### 🔴 CRITICAL: Чистые правила — `def`, application с БД — `async def`

Правила в `rules/` — чистые **синхронные** функции (`def`, без `await`). Функции в `application`, которым нужна БД/IO, — `async def`. Это главный механический маркер: `resolve_*` в `rules/` всегда `def`, `resolve_*` в `application` всегда `async def`.

### 🔴 CRITICAL: Запрет одинаковых имён в `application` и `rules/`

Одна и та же функция не существует под одним именем в `application` и `rules/` одновременно — иначе при импорте легко перепутать, какая нужна. Операция «собрать + решить» разбивается на функции с **разными** глаголами: `fetch_*` (application, сбор) → `select_*`/`compute_*` (rules, чистое решение).

```python
# GOOD — разные имена в разных слоях
# application — сбор (IO)
async def fetch_librarian_context(reader_id: int) -> LibrarianCtx:
    ...

# rules — чистое решение
def select_responsible_librarian(ctx: LibrarianCtx) -> int:
    ...
```

```python
# BAD — одинаковое имя в обоих слоях
# rules
def resolve_responsible_librarian(ctx: LibrarianCtx) -> int:
    ...

# application
async def resolve_responsible_librarian(reader_id: int) -> int:
    ...
```

### 🟡 WARNING: Извлекать правило только при нетривиальной логике

Извлечение чистой функции в `rules/` оправдано, когда решение нетривиально и нуждается в отдельном тестировании. Тривиальную логику (одна проверка) держат inline в `application`; не требуется обязательное наличие обеих функций.

---

## 6. Application Layer (`application/`)

### 🔴 CRITICAL: Координация + валидация с БД, не доменные правила

Application слой координирует вызовы, вызывает modules/repositories, выполняет мэппинг и валидацию с использованием БД (проверка существования переданных ИД перед сохранением и т.п.). Доменные правила валидации живут в `modules/<entity>/rules/` и вызываются из `application`.

### 🔴 CRITICAL: Валидация через ApplicationValidationError

Валидация бизнес-логики в `application` сигнализирует об ошибке через общий `ApplicationValidationError` из `application/exceptions.py` (один класс на весь слой, не на use case). Не HTTP-концепция: `application` не зависит от FastAPI.

```python
from library_membership.application.exceptions import (
    ApplicationValidationError,
    ValidationErrorDetail,
)

raise ApplicationValidationError(
    [
        ValidationErrorDetail(
            field="category_id",
            message="Category not found.",
        )
    ]
)
```

### 🔴 CRITICAL: Перевод в HTTP-исход

`ApplicationValidationError` переводится в HTTP-исход на уровне entrypoints, а не в `application`:
- **API** — глобальный обработчик `ApplicationValidationError` в `app.py` возвращает 422 `ValidationErrorResponse` (аналог обработчика `RequestValidationError`).
- **Consumer** — сам решает nack/retry/reject.

### 🔴 CRITICAL: Перевод доменных исключений

`application` перехватывает кастомные исключения из `modules` и переводит их в `ApplicationValidationError` или семантический результат (например, `None`), не пробрасывая доменные исключения в `entrypoints`.

### 🔴 CRITICAL: Enrichment паттерн

Для расширения сущностей создаётся `enrich_<entity>()` в `<entity>/<entity>_enrichment.py`. Разделение expands:
- **Repository expands** — передаются в методы репозитория (one-to-one)
- **Enrichment expands** — обрабатываются в enrichment-функции (many-to-many, many-to-one)

### 🟡 WARNING: Именование файлов

Файлы из одного use case: `create_<entity>.py`, `get_<entity>_by_id.py`, `update_<entity>.py`, `delete_<entity>.py`. Полная таблица именования функций по слоям — в §5 «Именование по слоям».

---

## 7. Entrypoints & DI (`entrypoints/`)

### 🔴 CRITICAL: Entrypoints — только routing и валидация

Entrypoints не содержат бизнес-логики. Определяют endpoint'ы, валидируют входные данные, вызывают application слой, оборачивают ответы.

### 🔴 CRITICAL: Response model

Всегда указан `response_model=ApiResponse[Schema]`, `ApiListResponse[Schema]` или `ApiResponsePaginated[Schema]`.

```python
# GOOD
@router.get("/{book_id}", response_model=ApiResponse[BookExtended])
async def get_book(book_id: int) -> ApiResponse[BookExtended]:
    ...

# BAD — нет response_model
@router.get("/{book_id}")
async def get_book(book_id: int):
    ...
```

### 🔴 CRITICAL: DI через Depends

`Depends()` используется **только** на уровне entrypoints и только для фреймворковых зависимостей (авторизация, `Message`, заголовки). БД (`library_db`) и внешние клиенты (`sso_client` и т.д.) — синглтоны из `infrastructure/`, импортируемые напрямую в `application`/`modules`/`repositories`, а не передаваемые как параметры.

Провалидированный payload (событие, query-параметры) передаётся в `application` как обычный параметр функции.

```python
# GOOD — dependency injection на уровне роутера
router = APIRouter(dependencies=[is_authorized(AccessTypeEnum.internal)])

# BAD — создание зависимостей внутри endpoint'а
@router.get("/books")
async def get_books():
    db = get_db()  # антипаттерн
```

### 🟡 WARNING: Именование

Router в каждом файле называется `router`. Имена endpoint'ов — глаголы. Файлы роутеров — множественное число (`books.py`, `branches.py`).

---

## 8. Error Handling

### 🔴 CRITICAL: Разделение зон ответственности

| Слой | Разрешённые исключения |
|---|---|
| `entrypoints` | `HTTPException`, `RequestValidationError`, `HTTPCustomException` |
| `application` | `ApplicationValidationError` (для бизнес-валидации); перехватывает доменные кастомные исключения и переводит их в `ApplicationValidationError` / семантический результат |
| `modules` | **Только кастомные** исключения |

```python
# BAD: module выбрасывает HTTPException
class BookService:
    async def create(self, data):
        if not category_exists:
            raise HTTPException(404)  # НЕТ!

# GOOD: module выбрасывает кастомное исключение
class BookService:
    async def create(self, data):
        if not category_exists:
            raise CategoryNotFoundError(category_id=data.category_id)

# GOOD: application перехватывает доменное исключение и переводит в ApplicationValidationError
async def create_book(data: BookCreateSchema):
    try:
        return await BookService.create(data)
    except CategoryNotFoundError as exc:
        raise ApplicationValidationError(
            [
                ValidationErrorDetail(
                    field="category_id",
                    message="Category not found.",
                )
            ]
        )
```

### 🟡 WARNING: Обработчики исключений

Глобальные exception handlers регистрируются в `app.py` для `RequestValidationError`, `ApplicationValidationError`, `HTTPCustomException`, `Exception`.

---

## 9. Testing

### 🔴 CRITICAL: Public API only

Тесты проверяют поведение через публичные интерфейсы, не через внутренние детали реализации.

```python
# GOOD: тестирует observable behavior
async def test_get_branch_returns_200(api_app_client, database_connection):
    load_data(database_connection, BranchTable, [branch_stub({"id": 1})])
    response = await api_app_client.get("/internal/v1/branches/1")
    assert response.status_code == 200

# BAD: залезает в базу напрямую
async def test_create_branch(library_database):
    row = await library_database.fetch_one("SELECT * FROM branch WHERE id=1")
    assert row["name"] == "Test"
```

### 🔴 CRITICAL: Фикстуры

- Unit тесты: `library_database` + `database_connection`
- Integration тесты: `api_app_client` + `database_connection`
- Данные загружаются через `load_data(connection, table, data)`

### 🟡 WARNING: Stub'ы

Stub-функции из `tests.stubs` создают тестовые данные: `branch_stub({"name": "Test"})`. Не добавляют системные поля (`is_active`, `created_at`, `updated_at`).

### 🟡 WARNING: Сравнение JSON

Использовать `assert_list_equals` / `assert_dict_equals` из `tests.utils.asserts`. Сравниваются только ключи из expected.

---

## 10. Code Quality

### 🔴 CRITICAL: Переменные не конфликтуют с builtins

`id` → `entity_id`, `type` → `entity_type`, `filter` → `filters`.

### 🔴 CRITICAL: Импорты

- Только top-level импорты. Запрещены локальные импорты внутри функций.
- Не добавлять импорты в `__init__.py` без явного указания в правилах.

### 🔴 CRITICAL: Async/await консистентность

Методы, работающие с БД через async, объявлены как `async def`. Вызовы репозиториев — с `await`.

### 🟡 WARNING: Форматирование и линтинг

После изменений запускать:
```bash
uv run black . && uv run isort --profile black .
uv run pytest
uv run flake8 src && uv run mypy src
```

### 🔴 CRITICAL: Transactional Operations

Все транзакции идут через `library_transaction_manager` (синглтон из `infrastructure/db.py`), а не через `library_db.transaction()` напрямую.

- **Один агрегат** — сервис `modules/<entity>/service.py` открывает транзакцию внутри своего write-метода.
- **Несколько агрегатов** — `application` открывает транзакцию сам (`library_transaction_manager.transaction()`); вложенные `transaction()` сервисов становятся savepoint'ами, commit-хуки выполняются на корневой транзакции.

Побочные эффекты записи (отправка событий) регистрируются как post-commit callback через `library_transaction_manager.add_post_commit_callback(...)` внутри сервиса, а не вызываются вручную из каждого use case — иначе легко забыть при добавлении нового use case. Уведомления отправляются только по этим событиям.

```python
# GOOD — одиночная write-операция в service
async with library_transaction_manager.transaction():
    reader = await ReaderRepository.create(data)

# GOOD — атомарная запись нескольких агрегатов в application
async with library_transaction_manager.transaction():
    await ReaderService.create(reader_data)
    await SubscriptionService.create(subscription_data)
```
