# VKOR EventHub Mini App — архитектура проекта

Документ описывает актуальную production-архитектуру на **3 ноября 2026 года**.
Его нужно обновлять при каждом изменении frontend, webhook-маршрутов или процесса публикации.

## 1. Репозитории и окружение

### Frontend

- GitHub repository: `https://github.com/AIPurgen/eventhub-miniapp`
- Ветка production: `main`
- GitHub Pages: `https://aipurgen.github.io/eventhub-miniapp/`
- Telegram BotFather Mini App URL: `https://aipurgen.github.io/eventhub-miniapp/`
- Telegram menu button URL: `https://aipurgen.github.io/eventhub-miniapp/`
- Фактический entrypoint Mini App — GitHub Pages; запуск не идёт через n8n HTML.
- Исходный файл: `C:\Downloads\123\LK_Client\github-pages\index.html`
- Основной документ: `C:\Downloads\123\LK_Client\github-pages\ARCHITECTURE.md`
- Frontend — один статический HTML-файл с CSS и JavaScript внутри.
- Сборщика, npm-проекта и frontend-фреймворка нет.

### Backend

Backend не хранится в GitHub frontend-репозитории. Он работает в n8n и PostgreSQL.

- n8n base URL: `https://n8n.arima.ru`
- Main page workflow (историческое имя EventHub_Main): ID `02jlGIzn2H9UxNi0`
- Production API workflow: `EventHub_Webhook`, ID `gbrdeDekGrArY2e7`
- Production API webhook: `https://n8n.arima.ru/webhook/5dbde993-ac88-40f8-889a-417849aad45f`
- Social API — отдельный production workflow; его runtime URL указан ниже. Точный workflow ID не фиксируется в этом документе, чтобы не смешивать его с диагностическими export-файлами.
- Social API webhook: `https://n8n.arima.ru/webhook/eventhub-social-api-7a9d2f`

`EventHub_Main` (`02jlGIzn2H9UxNi0`) — исторический n8n workflow страницы и не является текущим источником запуска Mini App. Не путать его HTML с production frontend из GitHub Pages.

Секреты n8n API и PostgreSQL находятся только в локальном файле:
`C:\Downloads\123\LK_Client\.secrets\production.env`.
Секреты нельзя добавлять в Git или этот документ.

## 2. Frontend и запуск приложения

Главная страница отдаётся GitHub Pages. Внутри HTML подключается официальный Telegram bridge:

```html
<script src="https://telegram.org/js/telegram-web-app.js"></script>
```

При загрузке страницы:

1. `window.Telegram?.WebApp` сохраняется в переменную `tg`.
2. Вызываются `tg.ready()` и `tg.expand()`.
3. `refreshTelegramContext()` повторно получает актуальные:
   - `tg.initData`;
   - `tg.initDataUnsafe.user`.
4. `init()` выполняет автоматическую авторизацию и загружает события.

Telegram bridge может отдать `initData` не в первый момент загрузки WebView. Поэтому bootstrap повторно вызывает `refreshTelegramContext()` после `DOMContentLoaded`, `load` и через короткий retry-интервал. `tg.initDataUnsafe.user` может использоваться для предварительного отображения профиля, но Telegram auth запускается только при наличии непустого подписанного `tg_init_data`. Один только `tg.initDataUnsafe.user.id` не является доказательством авторизации.

Основные frontend-функции находятся в `index.html`:

- `refreshTelegramContext()` — обновление Telegram-контекста.
- `api(entity, action, data)` — единый вызов production API webhook.
- `authUserFromResponse(response)` — нормализация ответа авторизации.
- `init()` — авторизация, получение роли и первичная загрузка событий.
- `loadModerators()` — загрузка списка модераторов для формы создания события.
- `saveNewEvent()` — создание события.
- `captureReferral()` — сохранение Telegram startapp/referral-параметров.

Ответы n8n могут приходить как объект, массив элементов, JSON-строка или объект с полем `result`/`data`.
Frontend использует общий слой нормализации ответа:

- `parseWebhookValue()` рекурсивно разбирает JSON-строки, включая двойную сериализацию, и считает строку из пробелов пустым ответом.
- `webhookError()` обнаруживает `success: false`, `ok: false`, `status: "error"` и такие же ошибки внутри массивов и n8n-оболочек.
- `authUserFromResponse()` раскрывает `body`/`response`/`json`/`result`/`data` и auth-поля, чтобы роль и `is_admin` не терялись при Telegram validate.
- `webhookCollection()` раскрывает `body`/`response`/`json`/`result`/`data`/`items`/`rows`/`records`, распрямляет массивы n8n и не превращает служебные/error-объекты в события или модераторов.
- `webhookValue()` извлекает значения из тех же оболочек для создания события и action-ответов.
- `isWebhookSuccess()` поддерживает boolean-результаты и значения `true`/`"true"` внутри n8n-массивов.

Production webhook может возвращать n8n-ответ как массив массивов (`[[...]]`); frontend распрямляет такую структуру перед отрисовкой. Неизвестный объект считается строкой только при наличии `id`, чтобы `{success: true}` или `{error: "..."}` не отображались как бизнес-данные.

## 3. Авторизация

### Запуск из Telegram Mini App

Форма логина скрыта первоначальным классом:

```html
<div id="authShell" class="auth-shell hidden">
```

При наличии подписанного `tg.initData` frontend считает Telegram-контекст пригодным для автоматической авторизации. Наличие только `Telegram.WebApp` или `tg.initDataUnsafe.user` позволяет отобразить предварительный профиль, но не заменяет signed auth context.

Основной auth-запрос:

```json
{
  "entity": "auth",
  "action": "validate",
  "data": {},
  "auth": {
    "tg_init_data": "<Telegram WebApp initData>",
    "user": {
      "id": 123,
      "username": "...",
      "first_name": "...",
      "last_name": "...",
      "photo_url": "..."
    }
  }
}
```

Backend Telegram-ветка использует Telegram-пользователя для `eventhub.upsert_user(...)`.
Полученный объект пользователя содержит ID, имя, роль и административные признаки.

После успешной авторизации frontend:

- сохраняет пользователя в `user`;
- определяет `isAdmin` по `is_admin`, `role` и `user_role`;
- скрывает auth-форму;
- показывает административные элементы интерфейса;
- загружает события через `events/list`.

Если Telegram-контекст есть, но автоматическая авторизация не удалась, форма логина не должна подменять Telegram-авторизацию. Показывается сообщение о повторном открытии Mini App из Telegram.

### Прямая ссылка вне Telegram

Если `Telegram.WebApp` и `tg.initData` отсутствуют, после завершения первичной проверки показывается `authShell`.
Пользователь может использовать:

- `auth/login` — логин и пароль;
- `auth/register` — создание аккаунта.

Access token сохраняется в `sessionStorage` под ключом:
`eventhub_access_token`.

При наличии access token сначала выполняется `auth/validate_session`. Если сессия недействительна, frontend очищает токен и пытается использовать Telegram `auth/validate` только при наличии signed `tg_init_data`.

## 4. Единый API-контекст

Все основные вызовы идут через `api()` и отправляют актуальный auth-контекст:

- если есть access token:
  `{ access_token, user }`;
- иначе:
  `{ tg_init_data, user }`.

Перед каждым вызовом выполняется `refreshTelegramContext()`, чтобы не использовать устаревшие значения Telegram bridge.

Основные API-операции:

| Entity | Action | Назначение |
|---|---|---|
| `auth` | `validate` | Telegram Mini App авторизация |
| `auth` | `validate_session` | Проверка access token |
| `auth` | `login` | Вход по логину и паролю |
| `auth` | `register` | Регистрация аккаунта |
| `auth` | `logout` | Выход |
| `events` | `list` | Список событий |
| `events` | `create` | Создание события |
| `events` | `join` | Запись на событие |
| `events` | `leave` | Отмена записи |
| `events` | `delete` | Удаление события |
| `users` | `get_moderators` | Список модераторов |

## 5. n8n production workflow

Workflow `EventHub_Webhook` начинается с узла `Webhook` и передаёт запрос в switch `auth`.

Основные ветки:

- `auth` — отделяет auth-запросы от событий и пользователей;
- `validate` — выбирает `login`, `register`, `logout`, `validate_session`, Telegram `validate`;
- `Switch` — маршрутизирует операции с событиями;
- `Switch1` — маршрутизирует дополнительные операции, включая модераторов;
- PostgreSQL-ноды выполняют функции схемы `eventhub`;
- response-ноды возвращают результат webhook.

Ключевые PostgreSQL-функции:

- `eventhub.auth_login(...)`
- `eventhub.auth_register(...)`
- `eventhub.auth_validate(...)`
- `eventhub.auth_logout(...)`
- `eventhub.upsert_user(...)`
- `eventhub.get_events()`
- `eventhub.get_moderators()`
- `eventhub.create_event(...)`
- `eventhub.join_event(...)`
- `eventhub.leave_event(...)`

Auth response-ноды должны возвращать JSON, а не пустой ответ. Для auth-веток используется выражение вида:

```text
={{ JSON.stringify($json.result || $json) }}
```

SQL auth-ноды должны возвращать хотя бы одну строку даже при ошибке, чтобы frontend не получал HTTP 200 с пустым body.

## 6. Роли и создание события

Администратор определяется frontend по одному из признаков:

- `is_admin === true`;
- `role === 'admin'` или `role === 'administrator'`;
- `user_role === 'admin'` или `user_role === 'administrator'`.

При `isAdmin` становятся доступны:

- кнопка создания события;
- административная навигация;
- dashboard организатора;
- загрузка списка модераторов через `users/get_moderators`.

Создание события передаёт:

- `title`;
- `description`;
- `event_time`;
- `max_participants`;
- `moderator_id`;
- `price_rub`.

Auth берётся из общего `api()` и не должен формироваться отдельной устаревшей логикой.

## 7. Telegram referral-ссылки

Поддерживаются параметры:

- `tg.initDataUnsafe.start_param`;
- query `tgWebAppStartParam`;
- query `startapp`.

Формат приложения:

```text
event_<event_id>_ref_<telegram_user_id>
```

`captureReferral()` сохраняет данные в `localStorage`:

- `eventhub_referral_event`;
- `eventhub_referral_token`.

Referral-информация используется при записи на событие и social analytics.

## 8. Social API

Social-операции идут отдельным webhook:

`https://n8n.arima.ru/webhook/eventhub-social-api-7a9d2f`

Используемые действия включают:

- `view`;
- `share`;
- `join`;
- `stats`;
- `user_stats`.

Social API также получает `tg_init_data` и `user` в auth-контексте.

## 9. Deployment и backups

### Публикация GitHub Pages

Изменения frontend публикуются так:

```powershell
git -C C:\Downloads\123\LK_Client\github-pages add index.html ARCHITECTURE.md
git -C C:\Downloads\123\LK_Client\github-pages commit -m "описание изменения"
git -C C:\Downloads\123\LK_Client\github-pages push origin main
```

После push нужно проверить raw GitHub и GitHub Pages. CDN GitHub Pages может обновляться с задержкой.

Текущий production commit публикуется из ветки `main`; отдельного npm build нет. Для этой структуры достаточно отправить отслеживаемые `index.html`, `ARCHITECTURE.md` и `BACKLOG.md` в `origin/main`.

### Обновление HTML в n8n

Скрипт:

`C:\Downloads\123\LK_Client\tools\deploy_telegram_auth_fix.js`

Он:

1. читает локальный `github-pages/index.html`;
2. получает workflow через n8n API;
3. сохраняет backup;
4. заменяет содержимое узла `Respond to Webhook`;
5. обновляет workflow;
6. проверяет наличие Telegram auth fixes.

### Исправление пустых auth-ответов

Скрипт:

`C:\Downloads\123\LK_Client\tools\fix_auth_empty_responses.js`

Он обновляет auth SQL и response-ноды production workflow, предварительно создавая backup в:

`C:\Downloads\123\LK_Client\.backups\`

Перед любым изменением n8n workflow нужно создавать backup и после изменения выполнять реальный POST-тест webhook.

## 10. Диагностика

При проблемах проверять в таком порядке:

1. GitHub Pages отдаёт ли новый commit, а не старый CDN-кэш.
2. В HTML есть ли `authShell hidden`, `refreshTelegramContext`, `parseWebhookValue` и `webhookError`.
3. Передаётся ли непустой signed `tg_init_data` в Network POST-запросе `auth/validate`.
4. Возвращает ли `auth/validate` JSON пользователя с ролью, а не `success: false` внутри n8n envelope.
5. Возвращает ли `events/list` массив событий; error-object не должен превращаться в карточку.
6. Возвращает ли `users/get_moderators` массив модераторов; `{success: true}` не должен создавать пустую option.
7. Не получает ли webhook HTTP 200 с пустым или пробельным body.
8. Корректно ли обрабатываются JSON-string, двойной JSON и массивы `[{json: {...}}]`.
9. Не используется ли старый session token из `sessionStorage`.

### Trace фактического webhook-ответа

Frontend записывает диагностический trace для **каждого** вызова `api(entity, action)`, а не только для auth:

- `api_request` — entity/action и correlation `request_id` перед отправкой;
- `fetch_start` и `network_context` — начало запроса и сетевой контекст;
- `http_response_received` — HTTP status, `ok`, redirect URL и доступные response headers;
- `response_body_read_error` — ошибка чтения body, если HTTP-ответ получен, но body не прочитан;
- `webhook_response` — фактический body ответа после `response.text()`, его размер, parsed shape и ошибка нормализации;
- `fetch_watchdog_timeout`, `fetch_timeout`, `fetch_error`, `api_request_exception` — сбои до получения/обработки ответа. Независимый watchdog завершает зависший fetch через 10 секунд, чтобы auth не оставался в состоянии `checking`.

Диагностика хранится в `localStorage` под ключом `eventhub_auth_debug`, последние 100 событий доступны в `window.__eventhubAuthDebug`. В профиле кнопка `Скопировать диагностику webhook` копирует JSON trace. Все записи коррелируются по `request_id`, который также передается в POST body как `meta.request_id`.

Для защиты пользовательских данных response body ограничивается 12 000 символами и маскирует `access_token`, `token`, `password`, `tg_init_data` и `initData`. Поэтому trace показывает фактически полученный ответ и его структуру, но не сохраняет секреты.

### Сквозная диагностика входа n8n

В production workflow `EventHub_Webhook` сразу после `Webhook` установлен узел `Webhook Intake Trace`. Он не изменяет бизнес-логику, а добавляет в execution data безопасный `_diagnostics`-объект с:

- `request_id`;
- временем приема;
- `webhook_url` и execution mode;
- origin, content type и content length;
- ключами входного body;
- entity/action;
- наличием auth, signed Telegram init data и Telegram user ID;
- безопасными HTTP headers без credentials/cookies.

Цепочка production auth теперь проверяется фактическим execution как:

```text
Webhook → Webhook Intake Trace → auth → validate → Execute a SQL query → Respond Telegram Validate
```

Если клиентский trace содержит `request_dispatch`, но в n8n Executions API отсутствует тот же `request_id`, запрос не подтвержден на входе n8n. Если execution есть, `_diagnostics.intake` подтверждает прием webhook, а список node runs показывает точную последнюю выполненную ноду.

После повторного открытия Mini App нужно убедиться, что в n8n появился новый execution. Если после открытия по GitHub Pages новый execution отсутствует, запрос не вышел из WebView и нужно проверять production HTML/браузерную консоль, а не SQL-ветки n8n.

Для проверки текущих n8n-данных используются локальные диагностические exports в корне рабочего каталога:

- `_live_routes.json`;
- `_live_connections.json`;
- `_auth_live_extract.json`;
- `_workflow_after_auth_response_fix.json`.

Это диагностические файлы, а не источник runtime-истины. Источник runtime-истины — active workflow в n8n и опубликованный GitHub Pages HTML.

## 11. Правила поддержания актуальности

При каждом изменении обязательно обновлять этот документ, если изменились:

- URL или ID workflow;
- API entity/action;
- auth payload;
- роли или признаки администратора;
- источник GitHub Pages;
- deployment-скрипты;
- структура referral-ссылок;
- порядок авторизации и fallback-логика.

Не записывать в документацию API keys, пароли PostgreSQL, токены Telegram или пользовательские access tokens.