# Упрощение отказоустойчивого доступа к БД Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Упростить DB-pipeline до фасада, executor, координатора и ограниченного транзакционного доступа, вынеся persistence очереди из service-слоя без изменения поведения.

**Architecture:** `УстойчивыйМенеджерСущностей` формирует неизменяемые команды и передаёт их через существующий `ИсполнительОперацийБД` единственному callback координатора. `КоординаторОперацийБД` владеет raw-менеджером, fatal-состоянием и одной транзакционной попыткой; repositories получают только краткоживущий `ТранзакционныйДоступКБД`.

**Tech Stack:** OneScript, Autumn, entity, resilience, OneUnit, asserts, moskito, SQLite.

---

### Task 1: Переименовать и сузить транзакционный доступ

**Files:**
- Rename: `src/internal/Классы/ЕдиницаРаботыБД.os` → `src/internal/Классы/ТранзакционныйДоступКБД.os`
- Rename: `tests/ЕдиницаРаботыБД_test.os` → `tests/ТранзакционныйДоступКБД_test.os`
- Modify: `src/internal/Классы/ТранзакционнаяОперацияМенеджераСущностей.os`
- Modify: `src/repository/Классы/ХранилищеЗависимыхРепозиториев.os`
- Modify: `src/service/Классы/ПроцессорОчередиПакетов.os`
- Modify references in `tests/УстойчивыйМенеджерСущностей_test.os`, `tests/УстойчивыйМенеджерСущностейSQLite_test.os`, repository/queue tests and transaction fixtures

- [ ] **Step 1: Write the failing contract test**

  Rename the test suite and change its wished-for API to `ТранзакционныйДоступКБД`. Assert that it delegates exactly `Получить`, `ПолучитьОдно`, `Сохранить`, `Удалить`, rejects every call after `Деактивировать`, and has no exported `ВыполнитьЗапрос`.

- [ ] **Step 2: Verify RED**

  Run:

  ```powershell
  & 'C:\Users\d.ivanov\AppData\Local\ovm\stable\bin\oscript.exe' 'oscript_modules\oneunit\src\cli\main.os' execute -f 'tests\ТранзакционныйДоступКБД_test.os' --mode summary
  ```

  Expected: suite load fails because type `ТранзакционныйДоступКБД` does not exist.

- [ ] **Step 3: Implement the narrow scoped capability**

  Rename the class and keep only this contract:

  ```bsl
  Функция Получить(ТипСущности, ОпцииПоиска = Неопределено) Экспорт
  Функция ПолучитьОдно(ТипСущности, ОпцииПоиска = Неопределено) Экспорт
  Процедура Сохранить(Сущность) Экспорт
  Процедура Удалить(Сущность) Экспорт
  Процедура Деактивировать() Экспорт // служебный интерфейс
  ```

  Preserve the activity guard and clearing of the raw-manager reference. Remove `ВыполнитьЗапрос`. Update callback parameter names and docs from `ЕдиницаРаботы` to `ДоступКБД` without changing transaction behavior.

- [ ] **Step 4: Verify GREEN and absence of the old concept**

  Run the renamed suite, facade suite, dependent-repository suite and queue suite sequentially. Then run:

  ```powershell
  rg -n 'ЕдиницаРаботыБД|ЕдиницаРаботы' src tests
  ```

  Expected: tests are green; old type/parameter name has no matches in active source and tests.

- [ ] **Step 5: Review before staging**

  Perform spec and quality review on the unstaged diff. Do not stage `packagedef`, `litros.log`, the spec, or unrelated user changes. After approval, commit only Task 1 files:

  ```text
  refactor: переименовать транзакционный доступ к БД
  ```

### Task 2: Ввести координатор и упростить фасад

**Files:**
- Create: `src/internal/Классы/КоординаторОперацийБД.os`
- Modify: `src/internal/Классы/УстойчивыйМенеджерСущностей.os`
- Delete: `src/internal/Классы/КонтекстУстойчивогоМенеджераСущностей.os`
- Delete: `src/internal/Классы/ОперацияУстойчивогоМенеджераСущностей.os`
- Delete: `src/internal/Классы/ТранзакционнаяОперацияМенеджераСущностей.os`
- Modify: `tests/УстойчивыйМенеджерСущностей_test.os`
- Modify: `tests/УстойчивыйМенеджерСущностейSQLite_test.os`
- Modify: transaction fixtures only where wiring changes

- [ ] **Step 1: Write RED tests for the simplified composition**

  Change facade construction to explicit constructor injection instead of private-field reflection:

  ```bsl
  Координатор = Новый КоординаторОперацийБД(ТабакеркаМенеджера);
  Фасад = Новый УстойчивыйМенеджерСущностей(Исполнитель, Координатор);
  ```

  Keep black-box assertions for one executor call, lazy manager resolution, whole-transaction retry, fresh scoped access, BEGIN/callback/COMMIT errors, rollback fatal state, close diagnostics and deterministic queued race. Add an assertion that both simple and transaction calls pass the same `Координатор.Исполнить` action target and an immutable command.

- [ ] **Step 2: Verify RED**

  Run the facade suite. Expected: `КоординаторОперацийБД` is missing and the facade constructor does not accept the new dependencies.

- [ ] **Step 3: Implement the coordinator**

  `КоординаторОперацийБД` is the only owner of the named raw-manager Tabakerka, cached manager, fatal diagnostic and one state lock. Its service API is:

  ```bsl
  Процедура ПроверитьРаботоспособность() Экспорт
  Функция Исполнить(Команда) Экспорт
  ```

  `Исполнить` rechecks fatal state after the executor slot, lazily resolves the manager, dispatches only the five allowed simple operations, or executes one whole transaction attempt. Request data is a `ФиксированнаяСтруктура` with `Режим, ИмяОперации, Операция, Параметры`; the singleton must not keep it in fields.

  Preserve exactly:

  - BEGIN error without rollback;
  - callback/COMMIT error → rollback → rethrow original `ИнформацияОбОшибке`;
  - rollback failure → deactivate access, best-effort close, `litros_db_rollback_failed:`, store fatal state before slot release;
  - parameter expansion for Undefined/scalar/Array/FixedArray;
  - a new `ТранзакционныйДоступКБД` for every retry attempt.

- [ ] **Step 4: Simplify the facade and remove adapters**

  The facade constructor receives/injects only `ИсполнительОперацийБД` and `КоординаторОперацийБД`. It retains exactly six exports. Every public method performs fail-fast through the coordinator, creates a fixed command and calls executor once with `Новый Действие(Координатор, "Исполнить")`. Preserve the user's existing formatting intent in this already-dirty file.

- [ ] **Step 5: Verify GREEN**

  Run facade 14 tests, real SQLite 4 tests, executor tests and classifier tests sequentially. Repeat the queued-race test at least three times. Verify deleted names have no source/test references.

- [ ] **Step 6: Review before staging**

  Review the unstaged diff, including concurrency and exception identity. After both reviews approve, commit only Task 2 files:

  ```text
  refactor: объединить выполнение операций БД в координаторе
  ```

### Task 3: Вынести persistence очереди в repository

**Files:**
- Create: `src/repository/Классы/ХранилищеОчередиПакетов.os`
- Create: `tests/ХранилищеОчередиПакетов_test.os`
- Reuse/modify: `tests/fixtures/Классы/СценарийОчередиПакетов.os`

- [ ] **Step 1: Write repository RED tests**

  Specify the public contract:

  ```text
  Добавить(ИмяПакета) -> Булево
  ПолучитьПачку(РазмерВыборки) -> Массив строк
  Удалить(ИмяПакета)
  ```

  Cover normalized atomic `ПолучитьОдно → Сохранить`, duplicate no-op, retry of the whole add transaction, ambiguous save (`2` checks/`1` save), ordering and limit of a batch, and idempotent delete by name. The repository callback receives `ТранзакционныйДоступКБД`; raw manager/executor must not appear.

- [ ] **Step 2: Verify RED**

  Run the new suite. Expected: type `ХранилищеОчередиПакетов` is not registered.

- [ ] **Step 3: Implement the repository**

  Inject only `УстойчивыйМенеджерСущностей`. Keep `ДобавитьВТранзакции(ДоступКБД, ИмяПакета)` in `СлужебныйПрограммныйИнтерфейс`. `ПолучитьПачку` hides `ОпцииПоиска` and `СущностьОчередьПакета` and returns names. `Удалить` finds by normalized name and deletes when present.

- [ ] **Step 4: Verify GREEN and integrations**

  Run the new suite, facade suite and SQLite integration. Ensure transaction retry tests use the real executor path rather than only a mocked callback.

- [ ] **Step 5: Review before staging**

  After spec/quality approval, commit only repository, its tests and necessary fixture changes:

  ```text
  refactor: выделить хранилище очереди пакетов
  ```

### Task 4: Оставить в процессоре только orchestration

**Files:**
- Modify: `src/service/Классы/ПроцессорОчередиПакетов.os`
- Modify: `tests/ПроцессорОчередиПакетов_test.os`
- Modify: `tests/fixtures/Классы/СценарийОчередиПакетов.os`
- Delete if unused: `tests/fixtures/Классы/РегистраторВызововИсполнителяБД.os`

- [ ] **Step 1: Write RED orchestration tests**

  Construct the processor with a mocked `ХранилищеОчередиПакетов` and updater. Assert:

  - public add normalizes the package name, delegates once and logs only when added;
  - batch processing receives names, calls updater outside any DB callback and deletes only successful names;
  - updater exception is logged and does not delete;
  - processor tests never create raw manager, executor, facade, transaction access, `ОпцииПоиска` or persistence queue entities.

- [ ] **Step 2: Verify RED**

  Run processor tests. Expected: constructor/dependency mismatch while it still injects the facade and contains DB methods.

- [ ] **Step 3: Implement pure orchestration**

  Inject `ХранилищеОчередиПакетов` instead of the facade. Remove `ДобавитьВОчередьВБД`, `ПолучитьПачкуОчереди`, `УдалитьИзОчереди` and all entity/persistence imports. Keep `РаботаСДаннымиБД.НормализироватьИмяПакета` only as the existing pure normalization utility so log text and public behavior remain unchanged.

- [ ] **Step 4: Verify GREEN**

  Run processor and queue-repository tests. Confirm updater invocation happens between repository `ПолучитьПачку` and `Удалить`, not inside a transaction callback.

- [ ] **Step 5: Review before staging**

  After both reviews approve, commit only Task 4 files:

  ```text
  refactor: отделить процессор очереди от БД
  ```

### Task 5: Закрепить архитектурные границы и выполнить полную проверку

**Files:**
- Modify: `tests/АрхитектураОтказоустойчивостиБД_test.os`
- Modify: tests that still inspect private fields or duplicate global architecture scans
- Add: `docs/superpowers/specs/2026-07-24-simplify-db-resilience-design.md`
- Add: `docs/superpowers/plans/2026-07-24-simplify-db-resilience.md`

- [ ] **Step 1: Write RED architecture assertions**

  Require:

  - removed class files/types/references do not exist;
  - raw `ВнутреннийМенеджерСущностей` injection exists only in coordinator;
  - facade has exactly six exports and depends only on executor/coordinator;
  - the only executor action target for DB facade is `КоординаторОперацийБД.Исполнить`;
  - transactional access exposes CRUD + `Деактивировать`, no SQL;
  - `ВыполнитьВТранзакции` production consumers are only the two repositories;
  - processor has no facade/executor/entity/scoped-access/persistence-entity references;
  - consumer tests do not construct raw DB infrastructure or set private facade fields.

- [ ] **Step 2: Verify RED then remove brittle duplicates**

  Run the architecture suite before adjusting production references; it must fail on at least the old adapters or processor DB dependency. Remove per-file text scans that duplicate the global guard, but keep behavioral tests.

- [ ] **Step 3: Run the full serialized verification matrix**

  Run, never in parallel because of `litros.log`:

  ```powershell
  & 'C:\Users\d.ivanov\AppData\Local\ovm\stable\bin\oscript.exe' 'oscript_modules\oneunit\src\cli\main.os' execute --mode summary
  git diff --check
  git status --short
  ```

  Expected: all suites green, no staged files before review, and only the known user changes (`packagedef`, pre-existing facade formatting if still applicable), docs and generated `litros.log` outside reviewed task diffs.

- [ ] **Step 4: Check changed OneScript files and structure metrics**

  Obtain Unicode paths with:

  ```powershell
  git -c core.quotepath=false diff --name-only --diff-filter=ACMRT 'a377dec..HEAD' -- '*.os'
  ```

  Run stable `oscript -check` for each file, recording any pre-existing global-module-context exceptions honestly. Verify core DB classes are exactly facade, executor, coordinator and transactional access; service DB execution dependencies are zero.

- [ ] **Step 5: Final reviews and staging**

  Run full-diff spec review and code-quality review before staging. After approval, stage only architecture/tests/docs belonging to this refactor; never stage unrelated `packagedef` or generated files. Commit:

  ```text
  test: закрепить упрощённые границы доступа к БД
  ```
