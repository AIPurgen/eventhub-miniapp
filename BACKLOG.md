# VKOR Mini App — единый backlog диагностики и исправлений

Дата аудита: 3 ноября 2026 года.

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
- [x] Создание события доступно только при `isAdmin`.
- [x] Ошибки/пустые ответы webhook не считаются успешными данными.
- [x] GitHub Pages проверен на совпадение с `main` и deployment SHA.

## Подтверждено проверками

- [x] Live Pages возвращает HTTP 200 и актуальный commit.
- [x] `OPTIONS` для обоих webhook возвращает CORS 204.
- [x] `events/list` возвращает данные с HTTP 200.
- [x] `users/get_moderators` возвращает данные с HTTP 200.
- [x] `auth/validate` возвращает пользователя и `is_admin` при валидном контексте.
- [x] Inline JavaScript проходит `node --check` после извлечения из HTML.

## Ограничения, требующие ручной проверки в Telegram

- [ ] Открыть Mini App именно из Telegram, а не в обычном браузере.
- [ ] Убедиться, что после открытия отображаются имя/username без входа в профиль.
- [ ] Для admin-пользователя проверить видимость `+`, панели организатора и списка модераторов.
- [ ] Для обычного пользователя проверить отсутствие `+` и админских интерфейсов.
- [ ] Проверить создание события только admin-пользователем.

Production n8n пока не изменялся: response-ноды событий и модераторов используют `allIncomingItems`, поэтому frontend поддерживает текущий формат `[[...]]`. Изменение backend без безопасного mutation-теста не выполняется автоматически.