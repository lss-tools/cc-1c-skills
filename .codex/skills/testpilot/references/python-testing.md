# Python API и pytest

Агент может исследовать форму через MCP, а затем написать Python-тест и повторно
запускать его через pytest. Python API обращается к клиенту тестирования 1С напрямую:
MCP-сервер запускать не нужно. Оба интерфейса используют общую реализацию операций.

Полноценный пример для демобазы УТ: [повторный заказ клиента](https://github.com/ROCTUP/1c-testpilot/blob/main/examples/pytest/ut/README.md)
с копированием документа, групповым заполнением, проверкой сумм и повторным открытием.

Пример для демобазы ЗУП: [приём нового сотрудника](https://github.com/ROCTUP/1c-testpilot/blob/main/examples/pytest/zup/README.md)
с созданием физлица и сотрудника, назначением оклада, проведением приёма
и проверкой сохранённых кадровых данных.

## Установка и запуск

Из каталога проекта:

```bash
python -m pip install ".[test]"
```

Задайте `TC1C_PROFILES_FILE` с путём к YAML-файлу профилей. Профили и переменные
`password_env` используются так же, как при работе через MCP. Затем:

```bash
python -m pytest tests/test_document.py --tc-profile ut_admin --tc-artifacts test-results -v
```

Плагин pytest регистрируется при установке пакета. При работе из исходников без
установки добавьте `app` в `PYTHONPATH` и передайте `-p testpilot_pytest`.
Настройки `TC1C_*` должны быть заданы до импорта `testpilot`; pytest сам не читает `.env`.

## Первый тест

`testpilot` — fixture, которая для каждого теста создаёт отдельное подключение.
Профиль с `base` запускает клиент; профиль с `host`/`port` подключает уже запущенный.
Следующий пример рассчитан на конфигурацию с документом `ЗаказКлиента` и полем
`Комментарий`. Имена нужно предварительно проверить в вашей базе через MCP.

```python
import pytest


@pytest.fixture
def document(testpilot):
    testpilot.execute_command(command="e1cib/data/Документ.ЗаказКлиента")
    form = testpilot.find_object(cls="ManagedForm", timeout=30)
    context = form.get_context()
    assert context["form"]["form_name"].startswith("Документ.ЗаказКлиента.")
    return form


def test_comment(testpilot, document):
    comment = document.find_object(name="Комментарий")
    comment.input_text(text="Проверка заполнения")
    assert comment.get_data_presentation() == "Проверка заполнения"
```

Проверка имени формы отличает целевую карточку от неожиданного диалога при открытии.
В примере документ не записывается. Подготовку обязательных данных, запись и очистку
созданных документов определяет конкретный тест.

## Интерфейс Python

```python
from testpilot import Client, ActionError

with Client(profile="ut_admin") as client:
    window = client.get_active_window()
    field = client.find_object(name="Комментарий")
    field.input_text(text="Проверка")
    assert field.get_text() == "Проверка"
```

Также можно создать `Client()` и явно вызвать `connect(...)` или `launch_client(...)`.
Вызовы синхронные. Действия одного клиента выполняются последовательно; для
параллельных тестов нужны отдельные клиенты и подходящие тестовые данные.
Два теста не должны одновременно управлять одним и тем же внешним клиентом 1С.

- `client.call("действие", **параметры)` предоставляет все действия из
  [справочника инструментов](tools.md). Префикс группы `tc_field` не нужен.
- `client.input_text(...)` и другие методы с именем действия эквивалентны `call`.
- `client.find_objects(...)` возвращает список `Element`;
  `client.find_object(...)` требует **ровно одно** совпадение. Если нужно штатное
  поведение поиска первого объекта, используйте `client.call("find_object", ...)`.
- `element.call("действие", ...)` и `element.input_text(...)` сами передают адрес
  элемента. `element.find_object(...)` ищет внутри него.
- `element.get_text()` и `element.get_data_presentation()` возвращают строку.
  Недоступное значение вызывает `ActionError`, прочитанное пустое значение — `""`.
- Остальные действия возвращают исходные словари и списки Python. Полные адреса
  находятся в `key`/`handle`, независимо от настроек JSON/TOON и коротких ссылок MCP.
- `get_screenshot()` возвращает `Screenshot` с полями `png: bytes` и `metadata: dict`.

Сохраните в сценарии имена элементов и условия поиска. Получайте `Element` заново
при каждом прогоне; после переподключения прежние объекты недействительны.

## Несколько полей, таблицы и снимки

```python
# Найдите элементы по именам из своей формы.
client.set_fields({comment: "Тест", enabled_checkbox: True})
result = client.read_fields([comment], properties=["presentation", "readonly"])
assert result["results"][0]["values"]["presentation"] == "Тест"

table.set_row_values(cells=[{"column": "ИмяКолонки", "text": "Значение"}])
table.add_rows(rows=[{"cells": [{"column": "ИмяКолонки", "text": "Ещё значение"}]}])
rows = table.read_rows()
assert rows["row_count"] == 2

baseline = form.create_snapshot(include_tables=True)
# Действие, результат которого проверяем.
changes = client.compare_snapshot(snapshot_id=baseline["snapshot_id"])
```

Пример с таблицей предполагает одну существующую строку и возможность ввести эти
значения. Строки читаются с текущими отборами и свёрнутыми узлами. Семантика действий,
включая завершение ввода и подготовку таблицы, совпадает с MCP.

Для полного ответа чтения используйте `element.call("get_data_presentation")`.
Для передачи готовых массивов `{key, handle, ...}` используйте
`client.call("set_fields", entries=...)` или `client.call("read_fields", targets=...)`.

## Ошибки и отчёты

Ожидание результата можно явно оформить через `wait_until`:

```python
def read_startup_window():
    result = client.call("get_active_window", check=False)
    if result.get("error") or result.get("exception"):
        raise ActionError("get_active_window", result)
    return result

window = client.wait_until(
    "Ожидание готовности клиента",
    read_startup_window,
    condition=lambda result: bool(result.get("ok") and result.get("key")),
    timeout=120,
    interval=1,
)
```

Первый callback читает состояние, `condition` проверяет результат. При выполнении
условия возвращается прочитанный результат. Проверки повторяются только при ложном
условии; исключения сразу прерывают ожидание. По таймауту возникает `ActionError`
с кодом `wait_timeout` и последним ответом в `result["last_result"]`.
Таймаут ограничивает цикл опроса; уже выполняющийся вызов ограничивается собственным
таймаутом операции. Вложенные ожидания не поддерживаются.

В HTML-журнале это один этап с длительностью, количеством проверок и результатом.
Ответы проверок находятся внутри свёрнутого блока подробностей. В фильтр ошибок
попадает неуспешный этап ожидания. Проверки внутри ожидания не создают скриншоты
журнала, в том числе в режиме `all`. В JSONL сохраняются отдельные вызовы с `wait_id`
и события `wait_start`/`wait_finish`. Вне `wait_until` оформление вызовов прежнее:
сам по себе `check=False` не меняет их статус в журнале. Без журналирования
ожидание выполняется так же.

После успешного отключения или остановки клиента журнал также не пытается снять
экран уже освобождённого подключения.

Отказ действия вызывает `ActionError`. В `error.action`, `error.code` и
`error.result` доступны название действия, код и полный результат, включая
успешно выполненную часть групповой операции. Несовпадение ожидаемого значения
проверяется обычным `assert`; отсутствие элемента — отдельная ошибка поиска.

```python
with pytest.raises(ActionError) as error:
    form.find_object(name="НесуществующееПоле")
assert error.value.code == "object_not_found"

# Когда отказ — ожидаемый результат, его можно проверить без исключения.
result = table.call("delete_rows", scope="selected", check=False)
assert result["code"] == "no_selected_rows"
```

После теста fixture останавливает запущенный ею клиент либо отключается от внешнего.
Созданные в базе данные автоматически не удаляются и не откатываются.

В `--tc-artifacts` создаётся отдельный каталог каждого теста: `result.json`, журнал
вызовов с HTML-отчётом при включённом `TC1C_LOGGING`, а при падении — попытка прочитать
контекст формы в `context.json`. Ошибка завершения клиента отражается в отчёте pytest.
Путь к каталогу также включается в JUnit XML, если передан `--junitxml report.xml`.

Скриншоты журнала по умолчанию выключены. Для их включения используйте
`--tc-screenshots actions` или `--tc-screenshots all`; настройки
`TC1C_SCREENSHOTS`, `TC1C_LOGGING` и `TC1C_LOG_SCREENSHOTS` должны разрешать захват.
Это не запрещает самому тесту явно вызвать `get_screenshot`, если функция разрешена.

## Останов клиента

После теста запущенный клиент завершается штатно с подтверждением известных вопросов
выхода; если за 15 секунд он не завершился, процесс останавливается принудительно.
Это также действует для `Client.close()`. Для явной остановки —
`client.stop_client(graceful_timeout=30)`; допустимо 0–120 секунд, 0 — сразу
принудительно. Ответ содержит `shutdown`: `graceful`, `forced` или `already_exited`;
при `forced` поле `shutdown_reason` объясняет причину. Несохранённые изменения могут
быть потеряны.

## Форматы отчётов

`--tc-reports` выбирает `html`, `allure`, `html,allure` или `none` (только JSONL);
по умолчанию берётся `TC1C_LOG_REPORTS`, без неё — `html`. Журнал и скриншоты собираются
один раз, HTML и Allure используют одни события. При Allure результаты пишутся в
`<tc-artifacts>/allure-results` (переопределяется `--alluredir`); сборка и просмотр —
`allure generate <каталог>/allure-results -o allure-report && allure open allure-report`.
Каждый тест имеет собственный результат; действия Testpilot отображаются шагами
с параметрами, ответами и скриншотами, `wait_until` группирует проверки ожидания.
Смысловые этапы можно объединять стандартным `with allure.step("…")`.

## Код и запросы в текущем сеансе 1С

Запусти клиент с обработкой Testpilot — в профиле (`code_epf:`), аргументом
`launch_client(..., code_epf=...)` или через `TC1C_CODE_EPF`. Приоритет: аргумент,
профиль, окружение; `code_epf=""` запускает клиент без обработки. Настройки публикации
MCP (`TC1C_CODE_EXECUTION` и остальные) для Python-тестов не требуются:

```python
with Client() as client:
    client.launch_client(base="D:/Bases/Demo", code_epf="D:/Tools/Testpilot.epf")
    assert client.execute_query(query="ВЫБРАТЬ 1 КАК Число")["rows"] == [{"Число": 1}]
```

Пример теста со стандартной фикстурой и профилем с `code_epf`:

```python
def test_document_was_saved(testpilot):
    # Тест перед этим создаёт документ через обычные действия над формой.
    rows = testpilot.execute_query(
        query="ВЫБРАТЬ Номер, Проведен ИЗ Документ.ЗаказКлиента ГДЕ Номер = &Номер",
        parameters={"Номер": "ТД00-000001"},
        limit=2,
    )
    assert rows["returned_rows"] == 1
    assert rows["rows"][0]["Проведен"] is True

    value = testpilot.execute_code(
        context="client",
        code='Результат = Параметры["текст"] + "!";',
        parameters={"текст": "Проверка"},
    )
    assert value["result"] == "Проверка!"
```

Подключение к работающему клиенту использует уже открытую обработку; если её нет,
выполнение сообщает `helper_not_ready`. Запросы учитывают текущие права пользователя;
даты, ссылки и перечисления возвращаются в типизированном JSON-представлении
([epf.md](epf.md), «Типы значений»). Ошибки — через `ActionError`.

Запись XML-сценариев и `run_scenario` доступны через тот же Python API. Для новых
pytest-тестов действия и проверки можно писать непосредственно на Python.

