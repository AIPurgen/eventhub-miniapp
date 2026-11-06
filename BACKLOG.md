# VKOR Mini App — единый backlog диагностики и исправлений

Дата обновления: 4 ноября 2026 года.

## Production-контракт

- Frontend: `https://aipurgen.github.io/eventhub-miniapp/`
- Production workflow: `EventHub_Webhook` (`gbrdeDekGrArY2e7`)
- Production API: `https://n8n.arima.ru/webhook/5dbde993-ac88-40f8-889a-417849aad45f`
- Social API: `https://n8n.arima.ru/webhook/eventhub-social-api-7a9d2f`

## Исправленные пункты

- [x] Устранён ранний конкурентный запуск Telegram bootstrap.
- [x] Добавлено ожидание Telegram `initData`/user context.
- [x] Повторная auth не прекращается после первой ранней попытки.
- [x] Профиль отрисовывается сразу после успешного webhook auth.
- [x] Профиль отрисовывается сразу после получения Telegram user context, независимо от задержки auth ответа.
- [x] Нормализованы admin-флаги `is_admin`, `isAdmin`, `role`, `user_role`.
- [x] Обрабатываются ответы webhook в оболочках `body`, `response`, `json`, `result`, `data`.
- [x] Обрабатываются массивы и массивы массивов `[[...]]` для событий и модераторов.
- [x] Не отбрасываются обычные массивы объектов, которые production n8n возвращает через `allIncomingItems`.
- [x] Создание события доступно только при `isAdmin`.
- [x] Ошибки/пустые ответы webhook не считаются успешными данными.
- [x] JSON-строки и двойная сериализация webhook-ответов разбираются рекурсивно.
- [x] Ошибки `success:false`, `ok:false`, `status:"error"` распознаются внутри n8n envelopes.
- [x] Служебные/error-объекты не превращаются в карточки событий или модераторов.
- [x] Action-ответы `join_event`/`leave_event` поддерживают boolean и строковый `"true"`.
- [x] Telegram auth использует только подписанный `tg_init_data`; `initDataUnsafe.user` не используется как самостоятельная авторизация.
- [x] GitHub Pages проверен на совпадение с `main` и deployment SHA.

## Подтверждено проверками

- [x] Live Pages возвращает HTTP 200 и актуальный commit.
- [x] `OPTIONS` для обоих webhook возвращает CORS 204.
- [x] `events/list` возвращает данные с HTTP 200.
- [x] `users/get_moderators` возвращает данные с HTTP 200.
- [x] `auth/validate` возвращает пользователя и `is_admin` при валидном контексте.
- [x] Inline JavaScript проходит `node --check` после извлечения из HTML.
- [x] Parser mock-проверки проходят для двойного JSON, n8n envelopes, вложенных collections, error responses и action boolean.
- [x] Mini App trace фиксирует фактический webhook body, HTTP status, response headers, parsed shape и ошибки чтения/парсинга для каждого API-вызова.

## Ограничения, требующие ручной проверки в Telegram

- [ ] Открыть Mini App именно из Telegram, а не в обычном браузере.
- [ ] Убедиться, что после открытия отображаются имя/username без входа в профиль.
- [ ] Для admin-пользователя проверить видимость `+`, панели организатора и списка модераторов.
- [ ] Для обычного пользователя проверить отсутствие `+` и админских интерфейсов.
- [ ] Проверить создание события только admin-пользователем.

Production n8n пока не изменялся: response-ноды событий и модераторов используют `allIncomingItems`, поэтому frontend поддерживает текущий формат `[[...]]`. Frontend также принимает JSON-строки и вложенные оболочки `body`/`response`/`json`/`result`/`data`/`items`/`rows`/`records`, но не считает произвольный status/error-объект строкой данных. Изменение backend без безопасного mutation-теста не выполняется автоматически.

## Текущий frontend response contract

Нормализатор поддерживает переходные форматы n8n, но рекомендуемый целевой ответ backend:

```json
{
  "ok": true,
  "data": {},
  "error": null
}
```

Для ошибок допустимы `ok:false` или `success:false` с полем `error`/`message`. Список endpoint-ов может возвращаться массивом строк или через `data`/`result`/`items`/`rows`.