Когда читать: нужно знать, где в проекте на python, node, go или rust лежат тесты и как их находит раннер; какую команду тестов с отчётом JUnit вписать в `check.conf`; как в языке сделать метку `hardware`; как выглядит пропуск или ослабление теста; как задать в тесте время и обойтись без сети.

# Тесты по стекам

Сначала — как заведено в проекте; этот файл — когда заведённого нет или оно непонятно.

Хорошая команда тестов для `check.conf`, на любом языке:

- запускает все тесты своего набора, а не часть по выбору;
- пишет отчёт JUnit XML в `reports/<набор>-<коммит>-<время>.xml` (SKILL.md, «Отчёт и вердикт»);
  папку создаёт сама: `mkdir -p reports && …`;
- работает без сети: установка зависимостей — отдельный шаг до неё, не её часть;
- завершается сама — режим наблюдения (watch) не годится — и при падении возвращает ненулевой код;
- ничего не спрашивает у человека.

Ниже `$R` — сокращение; в `check.conf` пиши вместо него выражение целиком:

```text
$(git describe --always --dirty 2>/dev/null || echo new)-$(date +%Y%m%dT%H%M%S)
```

Проверить, что новый тест выполнен, — найти его имя в отчёте.

## python

**Где и как.** `tests/` в корне; файлы `test_*.py`; функции `test_*` под pytest или классы
`unittest.TestCase`; общие фикстуры — `conftest.py`. Приёмочные — `tests/acceptance/`.

**Раннер — pytest.** Отчёт JUnit у unittest не выходит, а pytest его пишет и запускает в том
числе классы `unittest.TestCase`. pytest — в манифесте и lock-файле.

**Как находит раннер.** `test_*.py` и `*_test.py`, классы `Test*` без `__init__`, функции
`test_*`; поиск сужают `testpaths`, `norecursedirs` в настройках и `--ignore` в команде.
Убедиться — `python3 -m pytest --collect-only -q`.

**Команда.**

```text
unit: mkdir -p reports && python3 -m pytest -q tests/unit -o junit_logging=system-out --junitxml="reports/unit-$R.xml"
push: mkdir -p reports && python3 -m pytest -q tests/acceptance -o junit_logging=system-out --junitxml="reports/acceptance-$R.xml"
```

`junit_logging=system-out` кладёт в отчёт то, что тест напечатал, — там виден фактический
результат. Можно записать его в `[tool.pytest.ini_options]`, тогда в команде он не нужен.

**Метка `hardware`.** `conftest.py` в корне тестов:

```python
import os
import pytest


def pytest_configure(config):
    config.addinivalue_line(
        "markers", "hardware(need): нужно железо; без TEST_HARDWARE=1 тест пропускается")


def pytest_collection_modifyitems(config, items):
    if os.environ.get("TEST_HARDWARE") == "1":
        return
    for item in items:
        mark = item.get_closest_marker("hardware")
        if mark:
            need = mark.args[0] if mark.args else "особое железо"
            item.add_marker(pytest.mark.skip(reason=f"hardware: нужно {need}"))
```

Тест: `@pytest.mark.hardware("GPU NVIDIA с 16 ГБ памяти")` — и на функции, и на методе класса
`unittest.TestCase`. В отчёте: `<skipped message="hardware: нужно GPU NVIDIA с 16 ГБ памяти">`.

**Как выглядит пропуск и ослабление.** `@unittest.skip`, `skipIf`, `skipUnless`,
`expectedFailure`, `self.skipTest()`; `pytest.mark.skip`, `skipif`, `xfail`, `pytest.skip()`;
`-m "not …"`, `-k`, `--deselect` в команде; меньший `places` в `assertAlmostEqual`, больший `rel`
или `abs` в `pytest.approx`; `assertTrue(x)` вместо `assertEqual(x, …)`; `try/except` вокруг
проверяемого вызова; `# noqa`, `# type: ignore`, `--exit-zero` у линтера.

**Время, окружение.** Дата или часы — параметром. Иначе `unittest.mock.patch` на имени там, где
его использует код (`patch("app.report.now")`), а не в модуле `datetime`. Случайность —
`random.Random(42)`, переданный в код. Файлы — `tmp_path` или `tempfile.TemporaryDirectory()`.
Переменные — `monkeypatch.setenv` или `patch.dict(os.environ, {...})`.

**Сеть.** Подменить клиента, которого вызывает код (`patch("app.api.fetch_rates")`), и вернуть
из него заготовленный ответ из `tests/fixtures/`.

## node

**Где и как.** Рядом с кодом `*.test.ts` или `*.spec.ts`, либо в `__tests__/`, `test/`,
`tests/` — по проекту; для JS — те же имена с `.js`. Приёмочные Playwright — `tests/acceptance/`.

**Как находит раннер.**
- vitest — `*.test.*`, `*.spec.*`; задаётся `include`, исключается `exclude`.
- jest — `__tests__/**` и `*.test.*`, `*.spec.*`; задаётся `testMatch`, исключается
  `testPathIgnorePatterns`.
- `node --test` — свои шаблоны имён; без аргументов ищет файлы в текущем каталоге и ниже.
- Playwright — `testDir` и `testMatch` в `playwright.config.ts`.

Исключение в настройках раннера — тоже способ спрятать тест: правка `exclude` или
`testPathIgnorePatterns` объясняется так же, как правка самого теста.

**Команда.**

```text
unit: mkdir -p reports && npx vitest run --reporter=default --reporter=junit --outputFile.junit="reports/unit-$R.xml"
unit: mkdir -p reports && JEST_JUNIT_OUTPUT_FILE="reports/unit-$R.xml" npx jest --ci --reporters=default --reporters=jest-junit
unit: mkdir -p reports && node --test --test-reporter=spec --test-reporter-destination=stdout --test-reporter=junit --test-reporter-destination="reports/unit-$R.xml"
push: mkdir -p reports && PLAYWRIGHT_JUNIT_OUTPUT_FILE="reports/acceptance-$R.xml" npx playwright test --reporter=line,junit
```

Одна строка `unit` — по раннеру проекта. Для jest нужен пакет `jest-junit` в `devDependencies`.
`npm ci` — отдельный шаг, не часть команды; для vitest — `vitest run`, не `vitest`.

**Метка `hardware`.** Где у пропуска есть причина, она начинается с `hardware:`:

```ts
// Playwright — внутри теста
test.skip(process.env.TEST_HARDWARE !== "1", "hardware: нужно устройство считывания карт");
// node --test
test("…", { skip: process.env.TEST_HARDWARE !== "1" && "hardware: нужно GPU" }, () => { … });
```

У vitest и jest причины пропуска нет — метка идёт в название через общий помощник:

```ts
// tests/hardware.ts
import { test } from "vitest";
export const hardwareTest = (need: string, name: string, fn: () => unknown) =>
  test.skipIf(process.env.TEST_HARDWARE !== "1")(`${name} [hardware: ${need}]`, fn);
```

Для jest — то же с `(process.env.TEST_HARDWARE === "1" ? test : test.skip)`.

**Как выглядит пропуск и ослабление.** `it.skip`, `test.skip`, `describe.skip`, `xit`,
`xdescribe`, `it.todo`; `test.fails` в vitest; `.only` — выключает соседние тесты файла;
`toBeTruthy()`, `toBeDefined()` вместо `toEqual(…)`; запуск с `-u` — снимки переписываются без
просмотра разницы; `retries` в Playwright, чтобы пройти нестабильный тест; `eslint-disable`.

**Время, окружение.** `vi.useFakeTimers()` и `vi.setSystemTime(…)`; `jest.useFakeTimers()` и
`jest.setSystemTime(…)`; после теста — вернуть настоящие таймеры. Часовой пояс — переменная `TZ`
в скрипте `test`, чтобы он был одинаков на всех машинах.

**Сеть.** Подменить `fetch` или клиента, которого вызывает код: `vi.spyOn(globalThis, "fetch")`,
`jest.spyOn(api, "getRates")`; в Playwright — `page.route(…)` для внешних адресов.

## go

**Где и как.** `*_test.go` рядом с кодом. `package foo_test` — тест через публичный API, для
проверки поведения; `package foo` — когда нужен доступ к внутреннему. Данные — `testdata/`:
go не считает этот каталог пакетом.

**Как находит раннер.** `go test ./...` — все пакеты модуля, функции `TestXxx(t *testing.T)`.
Файл с ограничением сборки (`//go:build integration`) без нужного тега не собирается, и его тесты
молча не выполняются. Убедиться — `go test -list . ./<пакет>`.

**Команда.** `go test` отчёт JUnit не пишет; его пишет `gotestsum`, закреплённый в `go.mod` как
инструмент (`go get -tool gotest.tools/gotestsum`, Go ≥ 1.24):

```text
unit: mkdir -p reports && go tool gotestsum --junitfile "reports/unit-$R.xml" -- -count=1 ./...
```

Без сети работает, когда модули скачаны заранее (`go mod download` — отдельным шагом) или лежат в
`vendor/`; `GOPROXY=off` показывает, что сеть команде не нужна.

**Метка `hardware`.** Общий помощник в пакете тестов:

```go
func requireHardware(t *testing.T, need string) {
	t.Helper()
	if os.Getenv("TEST_HARDWARE") != "1" {
		t.Skip("hardware: нужно " + need)
	}
}
```

Первая строка теста — `requireHardware(t, "GPU NVIDIA")`. Не через `//go:build`: файл без тега не
собирается, и в отчёте теста нет.

**Как выглядит пропуск и ослабление.** `t.Skip`, `t.Skipf`, `t.SkipNow`, ранний `return`,
условие на `testing.Short()`, `//go:build` на файле теста; `t.Log(err)` вместо `t.Fatal(err)`;
`//nolint`.

**Время, окружение.** Часы — параметром (`now func() time.Time`); `t.TempDir()`, `t.Setenv()`.
Таблица случаев — срез структур и `t.Run(tc.name, …)` на каждый случай.

**Сеть.** `net/http/httptest.NewServer` — локальный сервер внутри теста вместо внешнего адреса.

## rust

**Где и как.** Модульные тесты — `#[cfg(test)] mod tests { use super::*; … }` в том же файле, что
и код. Через публичный API — `tests/*.rs`: каждый файл — отдельный крейт. Общий помощник —
`tests/common/mod.rs`; файл `tests/common.rs` cargo примет за ещё один набор тестов.

**Команда.** `cargo test` отчёт JUnit не пишет; его пишет `cargo-nextest`. В
`.config/nextest.toml`:

```toml
[profile.ci.junit]
path = "junit.xml"
```

```text
unit: mkdir -p reports && cargo nextest run --locked --profile ci; s=$?; cp target/nextest/ci/junit.xml "reports/unit-$R.xml"; exit $s
```

Без сети — `--offline`, зависимости скачиваются заранее (`cargo fetch`). Примеры в комментариях
`///` nextest не запускает: если проект на них опирается — отдельная строка `cargo test --doc`.

**Метка `hardware`.** `#[ignore = "hardware: нужно GPU NVIDIA"]`; на машине с железом —
`cargo nextest run --run-ignored all`. Добавив такой тест, открой отчёт: тест должен быть в нём
пропущенным. Нет его в отчёте — скажи человеку, не меняй способ метки сам.

**Как выглядит пропуск и ослабление.** `#[ignore]`; `#[should_panic]` без `expected = "…"` —
зелёный при любой панике; `#[cfg(…)]` на тесте; `assert!(result.is_ok())` вместо сравнения
значения; `#[allow(…)]` на предупреждение линтера.

**Время, окружение.** Часы — параметром: трейт или замыкание. Временные каталоги — крейт
`tempfile` (dev-зависимость, в lock-файле) или `std::env::temp_dir()` с уникальным именем.

**Сеть.** Клиент — за трейтом; в тесте — своя реализация трейта с заготовленным ответом.
