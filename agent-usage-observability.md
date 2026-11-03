# Локальное логирование Codex и OpenCode

## Цель

Собирать на одном компьютере статистику использования Codex, Codex VS Code extension и OpenCode:

- инструментов;
- MCP-серверов и MCP tools;
- skills;
- hooks;
- subagents;
- статусов, ошибок и latency.

Данные не отправляются во внешнее облако.

## Архитектура

```text
Codex CLI + VS Code extension
              │
OpenCode plugin
              │
              ▼
OpenTelemetry Collector
              │
              ▼
ClickHouse ───────── Grafana
```

## Что потребуется

1. Docker Desktop или Docker Engine.
2. OpenTelemetry Collector.
3. ClickHouse для хранения событий.
4. Grafana для дашбордов.
5. OTel-конфигурация Codex.
6. OpenCode plugin для отправки событий.

Codex VS Code extension обычно использует тот же Codex host и конфигурацию, поэтому отдельный collector для extension не нужен.

## Формат события

```json
{
  "timestamp": "2026-11-04T12:00:00Z",
  "client": "codex",
  "source": "vscode",
  "event_type": "tool_call",
  "tool_name": "mcp__github__search_issues",
  "mcp_server": "github",
  "skill_name": "",
  "session_hash": "…",
  "project_hash": "…",
  "agent": "default",
  "model": "…",
  "status": "success",
  "duration_ms": 842,
  "error_type": ""
}
```

Не сохранять без необходимости:

- полный prompt;
- содержимое файлов;
- аргументы и результаты MCP;
- API-ключи и токены.

## Codex

В `~/.codex/config.toml` указать локальный OTLP endpoint:

```toml
[otel]
exporter = { otlp-http = {
  endpoint = "http://127.0.0.1:4318/v1/logs"
}}
log_user_prompt = false
```

Интересующие события Codex:

```text
tool.call
mcp.call
hooks.run
skill.injected
multi_agent.spawn
```

После настройки проверить отдельно:

- Codex CLI;
- Codex VS Code extension;
- обычные tools;
- MCP-вызовы;
- skills;
- hooks;
- subagents.

## OpenCode

Для OpenCode нужен plugin с hooks:

```ts
ctx.tool.hook("execute.before", ...)
ctx.tool.hook("execute.after", ...)
```

Он должен отправлять в Collector:

- имя инструмента;
- MCP server/tool;
- статус;
- длительность;
- хеш session ID;
- хеш project ID.

Для skills нужно дополнительно отслеживать prompt hook. Полезная цепочка:

```text
skill доступен
→ skill выбран/injected
→ вызван связанный tool
→ получен успешный результат
```

Сам факт загрузки skill ещё не доказывает, что он был полезен.

## ClickHouse

Пример таблицы событий:

```sql
CREATE TABLE agent_events
(
    timestamp DateTime64(3),
    client LowCardinality(String),
    source LowCardinality(String),
    event_type LowCardinality(String),
    tool_name String,
    mcp_server String,
    skill_name String,
    session_hash String,
    project_hash String,
    agent String,
    model String,
    status LowCardinality(String),
    duration_ms UInt64,
    error_type String
)
ENGINE = MergeTree
ORDER BY (timestamp, client, event_type);
```

## Дашборд Grafana

### Общая статистика

- всего вызовов за день/неделю/месяц;
- активные сессии;
- активные инструменты;
- сравнение Codex и OpenCode;
- количество MCP-вызовов.

### Инструменты

- top tools;
- вызовы по дням;
- success/error rate;
- средняя и p95 latency;
- последний вызов;
- инструменты без использования за 30 дней.

### Skills

- сколько раз skill был injected;
- в скольких сессиях он использовался;
- какие tools вызывались после него;
- skills с нулевым использованием.

### MCP

- вызовы по серверу;
- вызовы по MCP tool;
- ошибки;
- latency;
- подключённые, но неиспользуемые серверы.

Главная таблица:

```text
Инструмент | Доступен в сессиях | Использован в сессиях | Adoption | Calls | Errors | Last used
```

Формула:

```text
adoption = sessions_with_tool_call / sessions_where_tool_was_available
```

## Этапы внедрения

### 1. Инфраструктура

Запустить через Docker Compose:

```text
OpenTelemetry Collector
ClickHouse
Grafana
```

### 2. Codex

- включить OTel;
- проверить `tool.call`, `mcp.call`, `skill.injected`;
- убедиться, что данные приходят из CLI и VS Code extension.

### 3. OpenCode

- написать usage-tracking plugin;
- отправлять события в OTLP endpoint;
- добавить tracking skills и MCP.

### 4. Нормализация

Привести разные имена к единому виду:

```text
codex:    mcp__github__search
opencode: github_search
```

Результат:

```text
server = github
tool = search
```

### 5. Дашборд

- подключить ClickHouse к Grafana;
- создать SQL-запросы;
- добавить фильтры по client, project, agent, model и периоду;
- настроить еженедельный отчёт.

## Критерии полезности инструмента

- 0 вызовов за 30 дней — кандидат на удаление;
- 1–2 вызова за 30 дней — оставить только при уникальной ценности;
- низкий success rate — чинить или отключать;
- доступен, но не используется — улучшить описание или discoverability;
- часто вызывается, но не помогает — анализировать ошибки и повторные вызовы.

## Рекомендуемый первый вариант

Начать только с событий вызовов и статусов, без prompt и содержимого данных. Затем добавить latency, skills и adoption-метрики.
