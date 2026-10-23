# Architecture guide (for agents)

How to place code in `library-membership`. This is the author-facing companion to `CODING_STANDARDS.md` (the review rubric). Read this **before** writing to `src/`.

Layers, top to bottom:

```
entrypoints  →  application  →  modules  →  repositories
                (READ path skips modules: entrypoints → application → repositories)
```

`modules` is write-only. Modules never import `application` or `entrypoints`. `application` never imports `entrypoints`.

## Where does this go?

| You are writing… | Put it in | Naming | Forbidden |
|---|---|---|---|
| Input parsing / DTO mapping (Kafka/RMQ/HTTP body → typed value) | `schemas/consumer/*` (pydantic model) | `XxxEvent` / `XxxMessage` | never `modules/**/rules/` |
| Pure decision over stored state (no IO) | `modules/<entity>/rules/<concept>.py` | `is_*`, `has_*`, `validate_*`, `assert_*`, `resolve_*`, `derive_*`, `select_*`, `choose_*`, `compute_*`, `decide_*` — sync `def` | no `async`, no IO, no `parse_*`, no `map_*`, no dict projection |
| Persistence + history/journaling | `modules/<entity>/service.py` | `create`, `update`, `delete`, `upsert` | no `apply`, no `get_*`, no orchestration/branching |
| Orchestration, DB-backed validation, use-case flow | `application/<verb>_<entity>.py` | `create_*`, `update_*`, `delete_*`, `get_*`, `apply_*` — `async def` | no domain exceptions leaking out |
| Reply/result code enum | `modules/<entity>/result_codes.py` | `XxxResultCode` | not bundled with human text |
| Human-readable result text / reply formatting | `entrypoints/consumer/*` (or api) | — | not in `modules/` |
| DB tables / ORM mappings | `db_schema/*` | `XxxTable` | no business logic |
| Repository queries | `repositories/*` | `get_list`, `get_by`, `create`, `update`, `delete` | no `get_by_<attr>`; return pydantic schemas |
| Neutral read-model produced by application enrichment | `schemas/application/*` | `XxxExtended(<DbSchema>)` | never in `schemas/db`, never in `schemas/integration_api` |

## Rules with rationale and when to break them

### `rules/` contains only pure, sync decision functions

- **Rationale:** keeps domain decisions testable in isolation and free of transport/persistence, so they can't silently depend on a DB or broker.
- **When to break it:** a decision genuinely needs data that isn't in its inputs — then it isn't a pure rule; it's `application` logic (DB fetch + a pure rule fed the fetched data). Never add IO to `rules/`; move the function up.

### `service.py` persists; it doesn't orchestrate

- **Rationale:** use-case branching (create vs close vs conflict vs no-op) is policy that changes with the use case; persistence (write + history) is stable. Mixing them makes the service impossible to reuse.
- **When to break it:** a single `upsert` may branch create-vs-update internally — that's persistence, allowed. But "which operation applies, given stored state" is `application` (+ `rules/`), not the service.

### Result-code enum vs human text live apart

- **Rationale:** the enum is domain vocabulary (`CONTEXT.md` "Код результата"); the Russian message is presentation for one transport. Bundling them (and a custom exception) in one `exceptions.py` couples domain to presentation.
- **When to break it:** a code with no transport text yet still gets a domain enum; text is added where the reply is built.

### Consumers use a typed pydantic body

- **Rationale:** faststream validates a `schemas/consumer` model, matching the API entrypoint pattern and keeping raw-dict parsing out of `modules`.
- **When to break it:** when you must reply a specific code on malformed input *and* the framework's auto-validation can't express it — then `model_validate` by hand in the consumer, still against a `schemas/consumer` model.

### Application enrichment models extend the DB schema

- **Rationale:** the READ path often enriches a stored entity with data from other tables or services (e.g., a reader plus their subscriptions). The application returns a model that *inherits* the DB schema (`schemas/db/*`) and adds the enriched fields — e.g. `ReaderExtended(Reader)` with `subscriptions: list[str]`. API entrypoints then map this neutral result to their own transport schemas (`schemas/integration_api/*`, internal).
- **When to break it:** a pure pass-through with no enrichment returns the DB schema unchanged, with no extended model.

## Anti-patterns that keep recurring

| Wrong | Why it's wrong | Right |
|---|---|---|
| `parse_message` / `parse_operation` in `rules/` | parsing is transport mapping, not a domain decision | `schemas/consumer/*` pydantic model |
| `subscription_state` / `diff_state` (dict projection) in `rules/` | journaling projection, not a decision; free-form `dict` hides the entity | typed helper next to the service |
| `apply` / `_apply_create` / `_apply_close` in `service.py` | use-case branching in the persistence layer | `application/apply_subscription_operations.py` + `decide_operation_outcome` in `rules/` |
| `exceptions.py` bundling enum + message text + error class | domain + presentation + error glued together | enum in `result_codes.py`, text in consumer |
| `message: Annotated[Any, Body()]` in a consumer | bypasses typed validation, reintroduces raw parsing | typed `schemas/consumer` model |

## Golden reference

Copy the shape of `modules/reader/`:

- `service.py` — `ReaderService` with `create_if_absent`/`upsert` (persistence + history only).
- `rules/rules_acceptance.py` — a pure `def is_rules_acceptance_event(...)` decision.
- `entrypoints/consumer/consumers/rules_signed.py` — typed pydantic body via faststream.

## Self-review checklist (run before "done")

1. Did anything land in `modules/**/rules/` that isn't a pure sync `def` (`is_/has_/validate_/assert_/resolve_/derive_/select_/choose_/compute_/decide_`)?
2. Does any `service.py` method exist that isn't `create`/`update`/`delete`/`upsert`?
3. Did I import `repositories` or `infrastructure` from a `rules/` file, or `application`/`entrypoints` from `modules/`?
4. Does a consumer use `Annotated[Any, Body()]` instead of a typed `schemas/consumer` model?
5. Is any human-readable message text or reply formatting sitting in `modules/`?
