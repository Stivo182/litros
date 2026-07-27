# Группировка подсистем внутри `src` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Сгруппировать внутренние production-классы БД, GitHub и пакетов по подпапкам внутри `src/internal/Классы`, сохранив все runtime-контракты и пользовательские изменения рабочего дерева.

**Architecture:** Верхнеуровневые слои `src` остаются неизменными; меняется только физическое расположение двенадцати классов внутри рекурсивно загружаемой OneScript-библиотеки `internal`. Плоские тесты не перемещаются: архитектурный набор получает явный layout-контракт и обновлённые пути к production-файлам.

**Tech Stack:** OneScript 2.1, Autumn, OneUnit, Git, PowerShell.

---

## Ограничения рабочего дерева

До начала реализации уже изменены и не принадлежат задаче:

```text
packagedef
src/internal/Классы/КлиентGitHubAPI.os
src/repository/Классы/ХранилищеОчередиПакетов.os
litros.log
```

`КлиентGitHubAPI.os` должен быть перемещён, но его пользовательский diff из двух
пустых строк нельзя включать в commit. Task 2 использует index-only rename: clean
blob берётся из `HEAD`, а рабочий dirty blob остаётся неизменным и после commit
показывается как modification уже по новому пути.

Не обновлять исторические specs/plans со старыми путями: они описывают состояние
на момент принятия решений. Актуальная структура закрепляется новым design и
архитектурным тестом.

### Task 1: Сгруппировать классы БД

**Files:**
- Rename: `src/internal/Классы/КлассификаторОшибокSQLite.os` → `src/internal/Классы/БД/КлассификаторОшибокSQLite.os`
- Rename: `src/internal/Классы/КомандаОперацииБД.os` → `src/internal/Классы/БД/КомандаОперацииБД.os`
- Rename: `src/internal/Классы/КоординаторОперацийБД.os` → `src/internal/Классы/БД/КоординаторОперацийБД.os`
- Rename: `src/internal/Классы/НастройкиОтказоустойчивостиБД.os` → `src/internal/Классы/БД/НастройкиОтказоустойчивостиБД.os`
- Rename: `src/internal/Классы/ОбъектыВзаимодействияСБД.os` → `src/internal/Классы/БД/ОбъектыВзаимодействияСБД.os`
- Rename: `src/internal/Классы/ТранзакционныйДоступКБД.os` → `src/internal/Классы/БД/ТранзакционныйДоступКБД.os`
- Rename: `src/internal/Классы/УстойчивыйМенеджерСущностей.os` → `src/internal/Классы/БД/УстойчивыйМенеджерСущностей.os`
- Modify: `tests/АрхитектураОтказоустойчивостиБД_test.os`

- [ ] **Step 1: Расширить path helpers до пяти сегментов**

Через `apply_patch` добавить `Сегмент5` во все три helper и в массив сегментов:

```bsl
Функция ПрочитатьФайлПроекта(
	Сегмент1,
	Сегмент2 = Неопределено,
	Сегмент3 = Неопределено,
	Сегмент4 = Неопределено,
	Сегмент5 = Неопределено)

	Возврат ПрочитатьФайл(
		ПутьПроекта(Сегмент1, Сегмент2, Сегмент3, Сегмент4, Сегмент5));
КонецФункции

Функция ПолноеИмяФайлаПроекта(
	Сегмент1,
	Сегмент2 = Неопределено,
	Сегмент3 = Неопределено,
	Сегмент4 = Неопределено,
	Сегмент5 = Неопределено)

	Возврат Новый Файл(
		ПутьПроекта(Сегмент1, Сегмент2, Сегмент3, Сегмент4, Сегмент5)).ПолноеИмя;
КонецФункции

Функция ПутьПроекта(
	Сегмент1,
	Сегмент2 = Неопределено,
	Сегмент3 = Неопределено,
	Сегмент4 = Неопределено,
	Сегмент5 = Неопределено)

	Путь = ОбъединитьПути(ТекущийСценарий().Каталог, "..");
	Сегменты = Новый Массив();
	Сегменты.Добавить(Сегмент1);
	Сегменты.Добавить(Сегмент2);
	Сегменты.Добавить(Сегмент3);
	Сегменты.Добавить(Сегмент4);
	Сегменты.Добавить(Сегмент5);
	Для Каждого Сегмент Из Сегменты Цикл
		Если Сегмент <> Неопределено Тогда
			Путь = ОбъединитьПути(Путь, Сегмент);
		КонецЕсли;
	КонецЦикла;
	Возврат Путь;
КонецФункции
```

- [ ] **Step 2: Написать RED layout-тест БД**

Добавить тест и общий helper размещения:

```bsl
&Тест
Процедура ТестДолжен_РазместитьВнутренниеКлассыБДВПодсистеме() Экспорт
	ИменаКлассов = Новый Массив();
	ИменаКлассов.Добавить("КлассификаторОшибокSQLite");
	ИменаКлассов.Добавить("КомандаОперацииБД");
	ИменаКлассов.Добавить("КоординаторОперацийБД");
	ИменаКлассов.Добавить("НастройкиОтказоустойчивостиБД");
	ИменаКлассов.Добавить("ОбъектыВзаимодействияСБД");
	ИменаКлассов.Добавить("ТранзакционныйДоступКБД");
	ИменаКлассов.Добавить("УстойчивыйМенеджерСущностей");

	ПроверитьРазмещениеКлассов("БД", ИменаКлассов);
КонецПроцедуры

Процедура ПроверитьРазмещениеКлассов(Подсистема, ИменаКлассов)
	Для Каждого ИмяКласса Из ИменаКлассов Цикл
		НовыйПуть = ПутьПроекта(
			"src", "internal", "Классы", Подсистема, ИмяКласса + ".os");
		СтарыйПуть = ПутьПроекта(
			"src", "internal", "Классы", ИмяКласса + ".os");
		Ожидаем.Что(Новый Файл(НовыйПуть).Существует(),
			СтрШаблон("Класс %1 размещён в подсистеме %2", ИмяКласса, Подсистема))
			.ЭтоИстина();
		Ожидаем.Что(Новый Файл(СтарыйПуть).Существует(),
			СтрШаблон("Класс %1 отсутствует в корне internal/Классы", ИмяКласса))
			.ЭтоЛожь();
	КонецЦикла;
КонецПроцедуры
```

В существующем тесте core-типов изменить путь `ОсновныеТипы` на:

```bsl
ПутьКласса = ПутьПроекта(
	"src", "internal", "Классы", "БД", ИмяТипа + ".os");
```

Проверки удалённых исторических типов оставить на старом корневом пути: эти
файлы не должны появляться ни до, ни после группировки.

- [ ] **Step 3: Обновить все прямые DB-пути в architecture-test**

Использовать пять сегментов:

```bsl
Координатор = ПрочитатьФайлПроекта(
	"src", "internal", "Классы", "БД", "КоординаторОперацийБД.os");
ОбъектыБД = ПрочитатьФайлПроекта(
	"src", "internal", "Классы", "БД", "ОбъектыВзаимодействияСБД.os");
ТекстФасада = ПрочитатьФайлПроекта(
	"src", "internal", "Классы", "БД", "УстойчивыйМенеджерСущностей.os");
Фасад = ПрочитатьФайлПроекта(
	"src", "internal", "Классы", "БД", "УстойчивыйМенеджерСущностей.os");
Доступ = ПрочитатьФайлПроекта(
	"src", "internal", "Классы", "БД", "ТранзакционныйДоступКБД.os");
```

- [ ] **Step 4: Проверить RED до перемещения**

Run:

```powershell
& 'C:\Users\d.ivanov\AppData\Local\ovm\stable\bin\oscript.exe' `
  'oscript_modules\oneunit\src\cli\main.os' execute `
  -f 'tests\АрхитектураОтказоустойчивостиБД_test.os' --mode summary
```

Expected: layout/path assertions падают, потому что семь файлов ещё находятся
непосредственно в `src/internal/Классы`. Набор должен загрузиться без syntax error.

- [ ] **Step 5: Выполнить только механические rename БД**

```powershell
New-Item -ItemType Directory -Force -Path 'src\internal\Классы\БД' | Out-Null
$names = @(
  'КлассификаторОшибокSQLite',
  'КомандаОперацииБД',
  'КоординаторОперацийБД',
  'НастройкиОтказоустойчивостиБД',
  'ОбъектыВзаимодействияСБД',
  'ТранзакционныйДоступКБД',
  'УстойчивыйМенеджерСущностей'
)
foreach ($name in $names) {
  git mv -- "src/internal/Классы/$name.os" "src/internal/Классы/БД/$name.os"
}
```

Не редактировать содержимое перемещённых production-файлов.

- [ ] **Step 6: Проверить GREEN и рекурсивное discovery**

Run sequentially:

```powershell
$oscript = 'C:\Users\d.ivanov\AppData\Local\ovm\stable\bin\oscript.exe'
$oneunit = 'oscript_modules\oneunit\src\cli\main.os'
$suites = @(
  'tests\АрхитектураОтказоустойчивостиБД_test.os',
  'tests\КлассификаторОшибокSQLite_test.os',
  'tests\КомандаОперацииБД_test.os',
  'tests\НастройкиОтказоустойчивостиБД_test.os',
  'tests\ОбъектыВзаимодействияСБД_test.os',
  'tests\ТранзакционныйДоступКБД_test.os',
  'tests\УстойчивыйМенеджерСущностей_test.os',
  'tests\УстойчивыйМенеджерСущностейSQLite_test.os',
  'tests\УстойчивыйМенеджерСущностейОтказоустойчивость_test.os'
)
foreach ($suite in $suites) {
  & $oscript $oneunit execute -f $suite --mode summary
  if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
}
```

Expected: все девять наборов зелёные. Это одновременно доказывает discovery
классов через `#Использовать "../src/internal"` и Autumn wiring.

- [ ] **Step 7: Проверить отсутствие содержательных изменений и commit**

```powershell
git diff --find-renames --stat
git diff --cached --check
git -c core.quotepath=false diff --cached --summary
git add -- 'tests/АрхитектураОтказоустойчивостиБД_test.os'
git diff --cached --check
git commit -m 'refactor: сгруппировать классы БД в internal'
```

Перед commit проверить, что семь production-файлов определены как rename и их
содержимое не изменилось.

### Task 2: Сгруппировать классы GitHub без поглощения пользовательского diff

**Files:**
- Rename: `src/internal/Классы/КлиентGitHubAPI.os` → `src/internal/Классы/GitHub/КлиентGitHubAPI.os`
- Rename: `src/internal/Классы/КомпонентыGitHubAPI.os` → `src/internal/Классы/GitHub/КомпонентыGitHubAPI.os`
- Rename: `src/internal/Классы/НастройкиОтказоустойчивостиGitHub.os` → `src/internal/Классы/GitHub/НастройкиОтказоустойчивостиGitHub.os`
- Modify: `tests/АрхитектураОтказоустойчивостиБД_test.os`

- [ ] **Step 1: Написать RED layout-тест GitHub**

```bsl
&Тест
Процедура ТестДолжен_РазместитьВнутренниеКлассыGitHubВПодсистеме() Экспорт
	ИменаКлассов = Новый Массив();
	ИменаКлассов.Добавить("КлиентGitHubAPI");
	ИменаКлассов.Добавить("КомпонентыGitHubAPI");
	ИменаКлассов.Добавить("НастройкиОтказоустойчивостиGitHub");

	ПроверитьРазмещениеКлассов("GitHub", ИменаКлассов);
КонецПроцедуры
```

Run architecture suite. Expected: новый тест RED на трёх старых путях, остальные
architecture tests зелёные.

- [ ] **Step 2: Выполнить rename и заменить staged client blob на clean HEAD blob**

Выполнить одной PowerShell-командой, чтобы dirty blob оставался в памяти и был
проверен до и после операции:

```powershell
$old = 'src/internal/Классы/КлиентGitHubAPI.os'
$new = 'src/internal/Классы/GitHub/КлиентGitHubAPI.os'
$dirtyBlob = git hash-object -- $old
$cleanBlob = git rev-parse "HEAD:$old"
if ($dirtyBlob -eq $cleanBlob) { throw 'Ожидался пользовательский diff КлиентGitHubAPI.os' }

New-Item -ItemType Directory -Force -Path 'src\internal\Классы\GitHub' | Out-Null
git mv -- $old $new
git mv -- 'src/internal/Классы/КомпонентыGitHubAPI.os' `
  'src/internal/Классы/GitHub/КомпонентыGitHubAPI.os'
git mv -- 'src/internal/Классы/НастройкиОтказоустойчивостиGitHub.os' `
  'src/internal/Классы/GitHub/НастройкиОтказоустойчивостиGitHub.os'

git update-index --add --cacheinfo 100644 $cleanBlob $new
if ((git hash-object -- $new) -ne $dirtyBlob) {
  throw 'Рабочий blob КлиентGitHubAPI.os изменился при rename'
}
```

Не выполнять `git add` для нового `КлиентGitHubAPI.os`: его clean rename уже
подготовлен напрямую в index, а рабочая пользовательская правка должна остаться
unstaged.

- [ ] **Step 3: Проверить разделение staged rename и unstaged user diff**

```powershell
git -c core.quotepath=false status --short
git -c core.quotepath=false diff --cached --find-renames -- `
  'src/internal/Классы/КлиентGitHubAPI.os' `
  'src/internal/Классы/GitHub/КлиентGitHubAPI.os'
git -c core.quotepath=false diff -- `
  'src/internal/Классы/GitHub/КлиентGitHubAPI.os'
```

Expected:

- cached diff — чистый rename без двух добавленных пустых строк;
- worktree diff нового пути — те же две пользовательские строки;
- status различает staged rename и unstaged modification;
- `git diff --cached --check` не сообщает whitespace из пользовательского diff.

- [ ] **Step 4: Проверить GitHub discovery и GREEN**

Run sequentially:

```powershell
$oscript = 'C:\Users\d.ivanov\AppData\Local\ovm\stable\bin\oscript.exe'
$oneunit = 'oscript_modules\oneunit\src\cli\main.os'
& $oscript $oneunit execute -f 'tests\АрхитектураОтказоустойчивостиБД_test.os' --mode summary
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
& $oscript $oneunit execute -f 'tests\КлиентGitHubAPI_test.os' --mode summary
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
& $oscript $oneunit execute -f 'tests\НастройкиПодключенияGitHub_test.os' --mode summary
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
```

Expected: layout, direct class loading and Autumn component discovery green.

- [ ] **Step 5: Stage только architecture-test и commit**

```powershell
git add -- 'tests/АрхитектураОтказоустойчивостиБД_test.os'
git diff --cached --check
git -c core.quotepath=false diff --cached --summary
git commit -m 'refactor: сгруппировать классы GitHub в internal'
```

После commit обязательно проверить:

```powershell
git -c core.quotepath=false status --short
git -c core.quotepath=false diff -- 'src/internal/Классы/GitHub/КлиентGitHubAPI.os'
```

Expected: пользовательская правка осталась unstaged по новому пути; старый путь
отсутствует; commit содержит три rename и architecture-test.

### Task 3: Сгруппировать внутренние классы пакетов

**Files:**
- Rename: `src/internal/Классы/ВалидаторИмениПакета.os` → `src/internal/Классы/Пакеты/ВалидаторИмениПакета.os`
- Rename: `src/internal/Классы/НакопленныеИспользованияПакета.os` → `src/internal/Классы/Пакеты/НакопленныеИспользованияПакета.os`
- Modify: `tests/АрхитектураОтказоустойчивостиБД_test.os`

- [ ] **Step 1: Написать RED layout-тест пакетов**

```bsl
&Тест
Процедура ТестДолжен_РазместитьВнутренниеКлассыПакетовВПодсистеме() Экспорт
	ИменаКлассов = Новый Массив();
	ИменаКлассов.Добавить("ВалидаторИмениПакета");
	ИменаКлассов.Добавить("НакопленныеИспользованияПакета");

	ПроверитьРазмещениеКлассов("Пакеты", ИменаКлассов);
КонецПроцедуры
```

Run architecture suite. Expected: RED, потому что оба файла пока находятся в
корне `internal/Классы`.

- [ ] **Step 2: Выполнить механические rename**

```powershell
New-Item -ItemType Directory -Force -Path 'src\internal\Классы\Пакеты' | Out-Null
git mv -- 'src/internal/Классы/ВалидаторИмениПакета.os' `
  'src/internal/Классы/Пакеты/ВалидаторИмениПакета.os'
git mv -- 'src/internal/Классы/НакопленныеИспользованияПакета.os' `
  'src/internal/Классы/Пакеты/НакопленныеИспользованияПакета.os'
```

Не менять содержимое production-файлов.

- [ ] **Step 3: Проверить consumers и GREEN**

Run sequentially:

```powershell
$oscript = 'C:\Users\d.ivanov\AppData\Local\ovm\stable\bin\oscript.exe'
$oneunit = 'oscript_modules\oneunit\src\cli\main.os'
& $oscript $oneunit execute -f 'tests\АрхитектураОтказоустойчивостиБД_test.os' --mode summary
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
& $oscript $oneunit execute -f 'tests\СканерИспользованийПакета_test.os' --mode summary
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
& $oscript $oneunit execute -f 'tests\ПроцессорОчередиПакетов_test.os' --mode summary
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
```

Expected: scanner, validator consumers and queue service continue discovering
types by name through unchanged libraries.

- [ ] **Step 4: Commit**

```powershell
git add -- 'tests/АрхитектураОтказоустойчивостиБД_test.os'
git diff --cached --check
git -c core.quotepath=false diff --cached --summary
git commit -m 'refactor: сгруппировать внутренние классы пакетов'
```

Expected: commit содержит два pure rename и изменение layout-теста.

### Task 4: Выполнить полную структурную и runtime-проверку

**Files:**
- Verify: `src/internal/Классы/БД/*.os`
- Verify: `src/internal/Классы/GitHub/*.os`
- Verify: `src/internal/Классы/Пакеты/*.os`
- Verify: `src/internal/Классы/СтартерПриложения.os`
- Verify: `tests/АрхитектураОтказоустойчивостиБД_test.os`

- [ ] **Step 1: Проверить итоговое дерево и отсутствие старых файлов**

```powershell
$rootClasses = Get-ChildItem -LiteralPath 'src\internal\Классы' -File
$expectedRootClasses = @('СтартерПриложения.os')
$actualRootClasses = @($rootClasses.Name | Sort-Object)
if (Compare-Object $expectedRootClasses $actualRootClasses) {
  throw 'В корне internal/Классы должен остаться только СтартерПриложения.os'
}
if ((Get-ChildItem 'src\internal\Классы\БД' -File).Count -ne 7) { throw 'Ожидалось 7 DB-классов' }
if ((Get-ChildItem 'src\internal\Классы\GitHub' -File).Count -ne 3) { throw 'Ожидалось 3 GitHub-класса' }
if ((Get-ChildItem 'src\internal\Классы\Пакеты' -File).Count -ne 2) { throw 'Ожидалось 2 класса пакетов' }
```

- [ ] **Step 2: Проверить syntax всех двенадцати перемещённых файлов**

```powershell
$oscript = 'C:\Users\d.ivanov\AppData\Local\ovm\stable\bin\oscript.exe'
$files = @(
  Get-ChildItem 'src\internal\Классы\БД' -File -Filter '*.os'
  Get-ChildItem 'src\internal\Классы\GitHub' -File -Filter '*.os'
  Get-ChildItem 'src\internal\Классы\Пакеты' -File -Filter '*.os'
)
foreach ($file in $files) {
  & $oscript -check $file.FullName
  if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
}
```

Expected: 12/12 `No errors`, exit `0`.

- [ ] **Step 3: Выполнить полный сериализованный OneUnit**

```powershell
& 'C:\Users\d.ivanov\AppData\Local\ovm\stable\bin\oscript.exe' `
  'oscript_modules\oneunit\src\cli\main.os' execute --mode summary
```

Expected: `16/16` suites и `145/145` tests successful, exit `0`.

- [ ] **Step 4: Проверить CLI-загрузку production-библиотек**

```powershell
& 'C:\Users\d.ivanov\AppData\Local\ovm\stable\bin\oscript.exe' `
  'src\main.os' --help
```

Expected: help приложения выводится, exit `0`; классы public/shared/internal
загружаются из нового дерева.

- [ ] **Step 5: Проверить commit range и сохранённый user diff**

```powershell
$implementationBase = git rev-list -1 `
  --grep='^docs: спланировать группировку подсистем в src$' HEAD
if (-not $implementationBase) { throw 'Не найден commit плана реализации' }
git diff --check "$implementationBase..HEAD"
git -c core.quotepath=false diff --find-renames --summary "$implementationBase..HEAD"
git diff --cached --name-only
git -c core.quotepath=false status --short
git -c core.quotepath=false diff -- 'src/internal/Классы/GitHub/КлиентGitHubAPI.os'
```

Expected:

- range содержит ровно 12 rename и изменения architecture-test;
- индекс пуст;
- user diff `КлиентGitHubAPI.os` остаётся по новому пути и не входит в commits;
- `packagedef`, production `ХранилищеОчередиПакетов.os` и `litros.log` сохранены;
- global worktree `git diff --check` может сообщать только заранее известный
  whitespace пользовательского `КлиентGitHubAPI.os`; не исправлять его.

- [ ] **Step 6: Провести final spec и quality review**

Проверить полный диапазон от commit с сообщением
`docs: спланировать группировку подсистем в src` до `HEAD` против
`docs/superpowers/specs/2026-07-27-organize-src-subsystems-design.md`:

- все верхнеуровневые каталоги `src` неизменны;
- перемещены только 12 согласованных production-классов;
- тестовые файлы и fixtures физически не перемещены;
- типы, exports, DI и поведение не изменены;
- layout guard запрещает возврат файлов в плоский корень;
- пользовательские dirty-изменения сохранены вне commit.

После approval новых изменений не коммитить: Task 4 является verification-only.
