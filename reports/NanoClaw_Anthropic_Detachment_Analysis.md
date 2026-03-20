# Анализ: отвязка NanoClaw от Anthropic и альтернативные решения

## 1. Можно ли отвязать NanoClaw от Anthropic?

### Текущие зависимости от Anthropic

| Компонент | Зависимость | Критичность |
|-----------|-----------|-------------|
| `container/agent-runner` | `@anthropic-ai/claude-agent-sdk` — функция `query()` | **Критическая** — ядро агента |
| `container/Dockerfile` | `@anthropic-ai/claude-code` — глобально установленный CLI | **Критическая** |
| `src/credential-proxy.ts` | Проксирует запросы к `api.anthropic.com` | Средняя — легко перенаправить |
| `src/container-runner.ts` | Переменные `ANTHROPIC_BASE_URL`, `ANTHROPIC_API_KEY` | Средняя — уже конфигурируемые |
| `.env` | `ANTHROPIC_API_KEY` / `CLAUDE_CODE_OAUTH_TOKEN` | Средняя |

### Вывод

**Полная отвязка невозможна без замены ядра.** NanoClaw архитектурно построен на `@anthropic-ai/claude-agent-sdk` (проприетарный SDK), который:
- Экспортирует функцию `query()` — единственный способ запуска агента
- Ожидает Anthropic API формат сообщений
- Управляет сессиями, tools, MCP-серверами в формате Claude

**Частичная отвязка возможна:**
- `ANTHROPIC_BASE_URL` уже поддерживает перенаправление на любой endpoint
- Ollama можно подключить как *инструмент* через `/add-ollama-tool` (Claude остаётся оркестратором)
- MiniMax M2.7 через Ollama = вспомогательная модель, НЕ основной движок

### Что нужно для полной отвязки

1. Заменить `@anthropic-ai/claude-agent-sdk` на альтернативный SDK (OpenHands SDK, LangChain, custom)
2. Переписать `container/agent-runner/src/index.ts` — заменить вызов `query()` на OpenAI-compatible API
3. Переписать Dockerfile — убрать `@anthropic-ai/claude-code`
4. Модифицировать credential-proxy для работы с OpenAI-compatible endpoints
5. Адаптировать IPC-протокол и MCP-серверы

**Оценка трудоёмкости:** Значительная переработка (~60-70% agent-runner, ~30% host-кода).

---

## 2. NemoClaw: доступность и лицензия

| Параметр | Значение |
|----------|----------|
| **Доступен бесплатно** | Да |
| **Лицензия** | Apache License 2.0 |
| **Исходный код** | https://github.com/NVIDIA/NemoClaw |
| **Статус** | Alpha (с марта 2026) |

Apache 2.0 разрешает коммерческое использование, модификацию и распространение без royalty.

---

## 3. Альтернативные решения

### 3.1 Сравнительная таблица

| Решение | Локальные модели | Песочница | CLI | Лицензия | Зрелость | Кастомизация |
|---------|:---:|:---:|:---:|---------|---------|------------|
| **NemoClaw** | ✅ (vLLM/Ollama/NIM) | ✅ (4 уровня) | ✅ | Apache 2.0 | Alpha | Средняя |
| **OpenHands** | ✅ (Ollama/vLLM) | ✅ (Docker/K8s) | ✅ | MIT | Зрелый (65K★) | Высокая |
| **OpenCode** | ✅ (Ollama) | ❌ | ✅ (TUI) | Open Source | Зрелый | Средняя |
| **Aider** | ✅ (Ollama) | ❌ | ✅ | Apache 2.0 | Зрелый (40K★) | Средняя |
| **Nanocoder** | ✅ (Ollama/OpenRouter) | ❌ | ✅ | Open Source | Молодой (1.5K★) | Высокая |
| **NanoClaw** | ⚠️ (только как tool) | ✅ (Docker) | ❌ (messaging) | MIT | Средний | Высокая |

### 3.2 Лучшие кандидаты для закрытого контура

#### Вариант A: NemoClaw + Ollama/vLLM (рекомендуется)
- **Плюсы**: корпоративная безопасность, NVIDIA backing, 4 уровня изоляции
- **Минусы**: alpha-статус, зависимость от OpenShell и OpenClaw

#### Вариант B: OpenHands + Ollama/vLLM
- **Плюсы**: зрелый (65K★), модель-агностик, Docker sandbox, REST API
- **Минусы**: тяжелее NemoClaw, нет kernel-level isolation

#### Вариант C: OpenCode + Docker-обвязка (гибрид)
- **Плюсы**: лёгкий CLI, нативная поддержка Ollama, TUI
- **Минусы**: нет встроенной песочницы — нужно добавлять

---

## 4. Рекомендуемый гибрид: LightClaw

### Концепция

Лёгкий AI-агент для изолированного контура, объединяющий:
- **OpenCode** как CLI-интерфейс (TUI, OpenAI-compatible)
- **Ollama** как локальный inference (MiniMax M2.7, Qwen3, DeepSeek)
- **Docker-песочница** по образцу NanoClaw (контейнерная изоляция)
- **Сетевые политики** по образцу NemoClaw (strict-by-default)

### Архитектура

```
┌─────────────────────────────────┐
│         LightClaw CLI           │
│  (OpenCode / custom TUI)        │
└──────────┬──────────────────────┘
           │
┌──────────▼──────────────────────┐
│      Security Wrapper            │
│  - Docker sandbox isolation      │
│  - Network policy (iptables)     │
│  - Filesystem restrictions       │
│  - Audit logging                 │
└──────────┬──────────────────────┘
           │
┌──────────▼──────────────────────┐
│      Agent Container             │
│  ┌────────────────────────────┐ │
│  │ OpenCode agent process     │ │
│  │ - OpenAI-compatible API    │ │
│  │ - File/Bash/Web tools      │ │
│  └────────────────────────────┘ │
└──────────┬──────────────────────┘
           │ http://host:11434/v1
┌──────────▼──────────────────────┐
│         Ollama / vLLM            │
│  - MiniMax M2.7                  │
│  - Qwen3-Coder:30b              │
│  - DeepSeek-Coder-V2            │
└──────────────────────────────────┘
```

---

## 5. План разработки LightClaw

### Фаза 1: Прототип (1-2 спринта)

1. **Настройка Ollama + модели**
   - Установить Ollama
   - Скачать MiniMax M2.7 и Qwen3-Coder
   - Проверить inference через OpenAI-compatible API

2. **Интеграция OpenCode**
   - Установить OpenCode CLI
   - Настроить на Ollama endpoint (`http://localhost:11434/v1`)
   - Проверить агентные задачи

3. **Docker-песочница**
   - Создать Dockerfile для агентного контейнера
   - Настроить volume mounts (по образцу NanoClaw)
   - Ограничить файловую систему

### Фаза 2: Безопасность (1-2 спринта)

4. **Сетевые политики**
   - Реализовать strict-by-default сетевую политику (iptables)
   - Whitelist для Ollama endpoint
   - Логирование заблокированных запросов

5. **Аудит и логирование**
   - Логирование всех команд агента
   - Логирование всех inference-запросов
   - Формат для интеграции с SIEM

6. **Credential management**
   - Proxy для API-ключей (по образцу NanoClaw credential-proxy)
   - Изоляция секретов от контейнера

### Фаза 3: Интеграция (1 спринт)

7. **CLI-обвязка**
   - Единый CLI для управления (launch, connect, status, logs)
   - Конфигурация моделей и политик
   - TUI для оператора

8. **Документация и тесты**
   - README, архитектурная документация
   - Юнит-тесты, интеграционные тесты
   - Инструкция по развёртыванию в air-gapped контуре
