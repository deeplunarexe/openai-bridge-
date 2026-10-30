# deepseek-chat2api

Локальный OpenAI-совместимый мост: HTTP-сервер (Node.js) + расширение Chrome.
Любой клиент с OpenAI API (Astra, CLI и т.д.) → MCP-класса тулы → ответы из
веб-чатов Qwen / Kimi / DeepSeek / ChatGPT / Duck.ai. Без API-ключей, всё
локально, генерация идёт в вашем браузере на вашем аккаунте.

Версия: **1.7.0** (эксперимент: markdown-формат кода).

## Запуск

1. `node server.js` — сервер по умолчанию `http://127.0.0.1:8000`
   (env: `BRIDGE_PORT`, `BRIDGE_HOST`, `BRIDGE_IDLE_TIMEOUT`).
2. Загрузить расширение: `chrome://extensions` → «Загрузить распакованное» →
   папка `extension/`.
3. Открыть вкладку нужного чата (Qwen/Kimi/DeepSeek/ChatGPT/duck.ai).
4. Клиенту указать API Base `http://127.0.0.1:8000/v1` (ключ любой).

После правок server.js — перезапуск сервера. После правок
content/injected/manifest/markdown — перезагрузка расширения + F5 вкладки.

## Эндпоинты

- `POST /v1/chat/completions` — OpenAI-совместимая генерация, `stream` и
  не-стрим, `tools`/`tool_choice` (тул-юз), `reasoning_content` для мыслей.
- `GET /v1/models` — список моделей.
- `GET /` — дашборд-тест.
- `WS /ws` — канал расширения.

## Модели

| Провайдер | Модели | Сайт |
|---|---|---|
| qwen | qwen-max, qwen-plus, qwen-turbo, qwen3-max | chat.qwen.ai |
| kimi | kimi-k3, kimi-k2.6, kimi-k2.5 | www.kimi.ai (международный) |
| deepseek | deepseek-chat, deepseek-reasoner | chat.deepseek.com |
| chatgpt | gpt-4o, gpt-4o-mini | chatgpt.com |
| duckai | duck, duck-gpt-4o-mini, duck-claude-haiku, duck-llama | duck.ai |
| claude | claude-sonnet-4, claude-opus-4 | виртуальный (роутинг) |

Модель выбирается по `modelPatterns`/`modelPriority` в `providers/*.js`
(напр. `duck-gpt-4o-mini` идёт в duckai, не в chatgpt). Провайдер = вкладка
с его сайтом; каждый провайдер изолирован в своём файле — падение одного не
роняет остальных.

## Как работает

```
клиент ─ HTTP ─ server.js ─ WebSocket ─ background.js ─ content.js ─ injected.js ─ сайт
```

- **injected.js** (MAIN world): перехват fetch/XHR ответа сайта, STREAM_CHUNK.
- **content.js** (ISOLATED): оркестрация — отправка промпта через собственную
  логику сайта (React onClick / form.requestSubmit / click / Enter), сбор
  ответа: сетевой перехват → DOM-фоллбэк (markdown из код-блоков, чистка
  плашек/дисклеймеров/UI-мусора).
- **background.js**: маршрутизация промпта по модели в нужную вкладку.
- **server.js**: протокол OpenAI, сборка промпта, парсинг тулов, SSE клиенту.

Мост content↔injected — через document-CustomEvent (`dsb-*`), НЕ
window.postMessage: сайты слушают window-"message" и падали от наших
объектов (v1.6.8+).

## Тул-юз

DeepSeek и др. не поддерживают tools нативно, поэтому:

1. server.js вшивает в промпт список тулов + формат
   `<|CHAT2API|tool_calls><|CHAT2API|invoke name="...">…`.
2. SSE-стрим буферизует маркеры и не пускает XML в контент.
3. Готовый блок парсится → клиенту идут `tool_calls` дельты (OpenAI-вид),
   стрим закрывается `finish_reason: "tool_calls"`.
4. Клиент исполняет тул, шлёт новый запрос с `role:"tool"`;
   результат вшивается как `<|CHAT2API|tool_result id=...><![CDATA[...]]>`
   и уходит в ТОТ ЖЕ чат сайта (контекст — история вкладки).
5. Модель отвечает финальным текстом — обычный стрим.

XML внутри ```-фенса тоже парсится (parser.js ищет управляемый блок по
всему тексту); примеры `<tool_calls>` в фенсах кодом — игнорируются.

## Формат ответа (v1.7.0, эксперимент)

- server.js шлёт всем хода короткую `FORMAT_RULES`: Markdown, весь код
  только в ```-фенсах с языком, tool_calls вне фенсов.
- markdown.js собирает DOM обратно в фенсы, включая чистку колонки
  номеров строк (последовательность 1,2,3… — убирается; настоящие цифры
  в коде — остаются).

## Экономия промпта

Статические system-блоки >400 символов шлются в сайт не каждый раз:
первый раз при появлении/изменении, каждый 15-й запрос
(`BRIDGE_SYSTEM_RESEND_EVERY`), всегда при `new_chat`. Короткая
формат-инструкция и компактный список имён тулов — каждый запрос.

## Файлы

- `server.js` — HTTP/SSE/WS, промпт, тулы
- `parser.js` — парсер tool-calls (managed XML, `call`...\``, экранирование)
- `providers.js` + `providers/*.js` — реестр провайдеров
- `injected.js` / `content.js` / `background.js` / `markdown.js` — расширение
- `cli.js` — минимальный CLI-клиент
- `extension/` — копия для загрузки в Chrome (после правок синкать!)
- `tests/` — `node tests\test_providers.js`, `test_server.js`,
  `test_parser.js`, `injected_test.js`, `test_markdown.js`,
  `test_kimi_unclosed.js`

## Диагностика

Логи сервера: `[Промпт]`, `[Тул-колл]`, `⚡ [SSE Чанк/…]`.
F12 на вкладке: `[DeepSeek-Bridge]`, в html — `data-dsb-ver` (версия).
Частые проблемы: расширение не подключено (открыть вкладку чата),
пустой ответ (F5 вкладки после апдейта), старая версия injected
(перезагрузить расширение + F5).



##Примечание
код написан полностью с помощью ИИ, автор не js разработчик. код завален комментарием и скорее всего смотрится как спаггети. 


##license 
MIT
