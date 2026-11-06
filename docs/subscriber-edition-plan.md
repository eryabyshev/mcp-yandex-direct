# План: редакция «подписчик Telegram-канала» (Cloudflare + read-only MCP)

Документ описывает текущую архитектуру форка [mcp-yandex-direct](https://github.com/askads/mcp-yandex-direct), целевое устройство hosted-продукта и поэтапный план доработок. Код менять не требуется — это основа для последующих PR.

## 1. Текущая архитектура (как в репозитории)

### 1.1 Точка входа и транспорт

- **`src/index.ts`**: создаёт `McpServer` из `@modelcontextprotocol/sdk`, регистрирует все инструменты, подключается **только к `StdioServerTransport`** — то есть классический локальный режим «процесс ↔ MCP-клиент по stdio» (`npx`, Cursor, Claude Desktop и т.д.).
- **Удалённого HTTP/SSE-транспорта в репозитории нет** — «URL удалённого MCP» для end-user это **новый слой**, который нужно добавить (например, Cloudflare Workers + обёртка над MCP SDK или Agents SDK / McpAgent).

### 1.2 Конфигурация

- **`src/config.ts`**: переменные окружения (`YANDEX_DIRECT_TOKEN`, `LOGIN`, `CLIENTS`, язык, песочница, таймауты, ретраи). Отсутствие токена — **не ошибка**: сервер стартует и отдаёт инструкции для `start_login`.

### 1.3 Токен и OAuth

- **`src/auth.ts` (`TokenStore`)**: на каждый запрос к API токен берётся заново — **сначала env** (`YANDEX_DIRECT_TOKEN`), иначе файл **`src/credentials.ts`** (`~/.config/mcp-yandex-direct/credentials.json`, права 0600). Refresh — только при наличии **`YANDEX_DIRECT_OAUTH_CLIENT_SECRET`** (см. комментарии в коде и `docs/TOOLS.md`).
- **`src/oauth.ts`**: PKCE (S256), ссылка с `redirect_uri = https://oauth.yandex.ru/verification_code`, один слот `pending` на процесс, обмен кода на токен.
- **`src/tools/auth.ts`**: инструменты `auth_status`, `start_login`, `finish_login`, `logout` — завязаны на локальный файл и чат.

### 1.4 Клиент API Яндекс Директа

- **`src/client.ts` (`YandexDirectClient`)**:
  - **JSON v5** (`call`): POST на `https://api.direct.yandex.com/json/v5/`, сервис `campaigns` — на **v501** (ЕПК).
  - **Live v4** (`callV4`): баланс через `AccountManagement` (токен в теле JSON, не в заголовке Authorization).
  - **Reports** (`report`): POST на `.../json/v5/reports`, опрос 200/201/202/204/5xx, учёт **кода 56** и «темпа» между отчётами (`reportPaceMs`).
  - **Токен в заголовке** собирается **перед каждым fetch** — нет кеша токена на экземпляре клиента.
  - **Ретраи**: сетевые ошибки и 5xx — для **read-методов** (`isReadMethod`: `get`/`has`/`check`); 429 и коды 52/506 — с оговорками; код **53** — один цикл refresh + replay при возможности.
  - **getAll**: курсор `LimitedBy`, лимиты строк/байт, флаги `_truncated`.
  - **SSRF-защита**: `service` в `raw_request`/`call` не может увести запрос на чужой origin.

### 1.5 Контекст вызова и телеметрия

- **`src/operation.ts`**: `AsyncLocalStorage` для отмены вызова и прогресса (долгие ожидания отчётов).
- **`src/telemetry.ts`**: отправка на `https://usage.gistrec.cloud/v1/events`; файл `instance-id` в конфиг-каталоге; обёртка `instrumentToolCalls`. Отключение: **`ASKADS_TELEMETRY=0`** (см. `docs/DEVELOPMENT.md`).

### 1.6 Инструменты

- Регистрация — в `src/tools/*.ts`, агрегированное описание — в **`docs/TOOLS.md`**. Ниже — сводная таблица для планирования read-only редакции.

### 1.7 Тесты и CI

- **Тесты**: `npm test` — `node --test` по `src/**/*.test.ts`, без сети (моки fetch).
- **CI**: `.github/workflows/ci.yml` — на push в `main` и для PR: `npm ci`, `typecheck`, `build`, `test` на Node **20 / 22 / 24**.

---

## 2. Таблица инструментов: чтение / запись и что оставить подписчикам

Условие продукта: подписчикам — **только чтение и отчёты**; `raw_request` — **только** если метод чтения (в коде уже есть `isReadMethod` и проверка `confirmWrite` для записи).

| Инструмент | Тип | Примечание для редакции подписчика |
| --- | --- | --- |
| `auth_status` | чтение (локально) | Заменить/дополнить веб-OAuth; как MCP-инструмент может не понадобиться |
| `start_login` | настройка доступа | Убрать или заменить потоком в Mini App / веб-странице |
| `finish_login` | настройка доступа | То же |
| `logout` | изменение состояния | В hosted: «отозвать доступ» в UI, не обязательно через MCP |
| `get_account_info` | чтение | **оставить** |
| `get_balance` | чтение | **оставить** |
| `get_quota` | чтение | **оставить** |
| `list_clients` | чтение | **оставить** |
| `clients_overview` | чтение | **оставить** (тяжёлый; может понадобиться лимитирование) |
| `get_regions` | чтение | **оставить** |
| `get_dictionaries` | чтение | **оставить** |
| `list_campaigns` | чтение | **оставить** |
| `list_ad_groups` | чтение | **оставить** |
| `list_ads` | чтение | **оставить** |
| `list_keywords` | чтение | **оставить** |
| `get_bid_modifiers` | чтение | **оставить** |
| `get_sitelinks` | чтение | **оставить** |
| `get_callouts` | чтение | **оставить** |
| `get_ad_images` | чтение | **оставить** |
| `get_ad_videos` | чтение | **оставить** |
| `get_creatives` | чтение | **оставить** |
| `get_statistics` | чтение (отчёты) | **оставить** |
| `raw_request` | чтение или запись | **оставить** только в режиме, эквивалентном текущему правилу: методы `get`/`has`/`check` без `confirmWrite`; **запись отключить** |
| `create_text_campaign` | запись | **исключить** |
| `update_campaign` | запись | **исключить** |
| `campaign_action` | запись | **исключить** |
| `update_negative_keywords` | запись | **исключить** |
| `update_excluded_sites` | запись | **исключить** |
| `set_campaign_strategy` | запись | **исключить** |
| `create_ad_group` | запись | **исключить** |
| `update_ad_group` | запись | **исключить** |
| `delete_ad_groups` | запись | **исключить** |
| `create_text_ad` | запись | **исключить** |
| `update_text_ad` | запись | **исключить** |
| `ad_action` | запись | **исключить** |
| `add_keywords` | запись | **исключить** |
| `set_keyword_bids` | запись | **исключить** |
| `keyword_action` | запись | **исключить** |
| `add_bid_modifier` | запись | **исключить** |
| `set_bid_modifiers` | запись | **исключить** |
| `delete_bid_modifiers` | запись | **исключить** |
| `create_sitelinks_set` | запись | **исключить** |
| `delete_sitelinks` | запись | **исключить** |
| `add_callouts` | запись | **исключить** |
| `delete_callouts` | запись | **исключить** |
| `upload_ad_image` | запись | **исключить** |

**Итого для подписчиков (минимум):** все `get_*` / `list_*` / `clients_overview` / `get_statistics` / ограниченный `raw_request`.

**Локальная «полная» редакция:** без изменений списка инструментов — отдельная сборка или флаг окружения (`SUBSCRIBER_MODE=1`), чтобы не ломать `npx`/stdio.

---

## 3. Зависимости от Node.js, мешающие Workers

| Область | API | Замечание для Workers (runtime совместимый с моделью Workers, без Node compatibility) |
| --- | --- | --- |
| Точка входа | `readFileSync`, `import.meta.url` | Чтение `package.json` для версии — заменить на константу сборки или `worker_version` из CI |
| Учётные данные | `fs`, `os.homedir`, `path` | Нужно **D1/KV/R2** + шифрование вместо `~/.config/...` |
| OAuth PKCE | `node:crypto` (`createHash`, `randomBytes`) | Использовать **Web Crypto** (`crypto.subtle`, `crypto.getRandomValues`) |
| `reportName` / хеши | `node:crypto` `createHash` | Web Crypto SHA-256 |
| Телеметрия | `fs`, `randomUUID` | Удалить или заменить no-op; id не хранить на диске в Workers |
| Операции | `AsyncLocalStorage` | В Workers нет **Node** `async_hooks`; нужен другой механизм скоупа (явная передача context, `AsyncContext` из поддерживаемого polyfill, или упрощение без ALS) |
| SSRF в `upload_ad_image` | `dns.lookup`, `net.isIP` | Перенести на **наличный** API в Workers или вынести загрузку изображений из read-only редакции |
| Тесты | `node:test`, `node:fs` в тестах | Остаются на Node в CI; runtime-пакет может собираться под `workerd` отдельно |

**Вывод:** ядро (`client.ts`, большинство `tools`) в основном **fetch + JSON** и переносимо; блокеры — **файловое хранилище**, **Node crypto/async_hooks**, **telemetry fs**.

---

## 4. Целевая архитектура на Cloudflare

### 4.1 Компоненты

1. **Telegram Mini App (статический фронт)**  
   Получает `initData`, показывает статус, кнопки «Подписаться», «Подключить Яндекс», «Скопировать URL MCP».

2. **Worker HTTP API** (или тот же Worker, другие маршруты)  
   - `POST /api/telegram/session` — валидация подписи `initData` (секрет бота).  
   - `GET /api/oauth/yandex/start` / `callback` — OAuth **приложения владельца** (`client_id`/`secret` в секретах Worker).  
   - Выдача и ротация **длинного случайного MCP-токена** (не детерминированного от Telegram ID), привязка `telegram_user_id ↔ oauth tokens (encrypted) ↔ mcp_token`.  
   - Админ: отзыв токена, просмотр использования (без хранения полного токена в открытом виде).

3. **MCP над HTTP** (McpAgent / streamable HTTP из MCP SDK — конкретная схема выбирается в первом PR интеграции)  
   - Middleware: **Authorization: Bearer \<mcp_token\>** (или query — хуже по безопасности).  
   - Проверка подписки: `getChatMember` **с кешом** (KV/DO), TTL **несколько часов**; при отсутствии подписки — все инструменты возвращают единообразное сообщение со ссылкой на канал (можно также отклонять handshake).  
   - Режим **read-only**: регистрируется урезанный набор инструментов + ужесточённый `raw_request`.

4. **Хранилища**  
   - **D1** (или KV для MVP): пользователи, хеш MCP-токена, зашифрованные OAuth-токены Яндекса, метаданные.  
   - **KV**: кеш `getChatMember`, rate limit counters.  
   - **Secrets** (Worker): `TELEGRAM_BOT_TOKEN`, ключ шифрования (или Cloudflare Secrets + обёртка KMS-стиля).  
   - **R2** — опционально для аудит-логов.

5. **Телеметрия Ask Ads**  
   В hosted-сборке: **не компилировать** `instrumentToolCalls` / `Telemetry` или жёстко `ASKADS_TELEMETRY=0` + исключить сетевой исходящий endpoint.

### 4.2 Потоки

- **Mini App → подтверждение пользователя → проверка подписки → выдача MCP URL** с персональным токеном.  
- **Подключение Яндекса**: redirect OAuth → сохранение **refresh_token + access_token** в зашифрованном виде, привязка к Telegram-пользователю.  
- **Вызов MCP**: клиент (Cursor и т.д.) → Worker MCP endpoint → проверка Bearer → (кеш) проверка подписки → **TokenStore для облака** поднимает расшифрованный токен и дергает `YandexDirectClient`.  
- **Локальная полная версия**: по-прежнему `node dist/index.js` + stdio, без Telegram.

---

## 5. Совместимость с Workers и выбор: Workers vs Containers

### 5.1 Workers (рекомендуемая стартовая точка)

- **Плюсы:** низкая стоимость, встроенный edge fetch, D1/KV, Secret management, хорошо стык с **Agents SDK / McpAgent**.  
- **Минусы:** лимиты CPU/времени на запрос; долгие отчёты (`get_statistics` с множественными опросами) могут упираться в **wall-clock и лимиты Worker** — понадобятся таймауты, ограничение `maxPolls`, возможно «обрезание» сценариев `clients_overview` для подписчиков.

### 5.2 Cloudflare Containers

- **Когда имеет смысл:** если понадобится полный Node без портирования (`AsyncLocalStorage`, `node:crypto`, долгие процессы) или если Яндекс/API-политики выявят проблемы с исходящими запросами с edge.  
- **Минусы:** операционная сложность, стоимость, всё равно нужен защищённый периметр и секреты.

**Рекомендация:** **начать с Workers + D1/KV**, вынести `YandexDirectClient` и read-only инструменты в общий пакет; Containers держать как **Plan B** при несовместимости лимитов или необходимости неизменного Node-стека.

---

## 6. Безопасность и эксплуатация

| Тема | Меры |
| --- | --- |
| MCP-токен | Криптографически стойкая случайность (≥32 байта), хранить **только хеш** (например SHA-256) + optional prefix для поддержки; выдавать пользователю один раз при создании |
| OAuth токены Яндекса | Шифрование **AES-256-GCM** с ключом из Secrets; отдельный nonce; ротация ключа — миграция батчами |
| Отзыв | Удаление записи в D1 + инвалидация KV-кеша подписки |
| Telegram | Проверка подписи `initData`; не доверять `user.id` из клиента без подписи |
| Подписка на канал | `getChatMember`; кеш с TTL; при смене статуса — задержка до TTL приемлема продуктово |
| Rate limiting | Лимит запросов на `mcp_token` + IP (с осторожностью из-за NAT); лимит дорогих инструментов (`get_statistics`, `clients_overview`) |
| Квоты Units / Reports | Честное описание в `instructions`: один сервис на много пользователей **делит лимиты аккаунта Яндекса** — нужны метрики и алерты |
| `raw_request` | Жёстко whitelist только read-методов; опционально whitelist сервисов |

---

## 7. Риски и открытые вопросы

1. **Доступность API Яндекс Директа с IP Cloudflare** — нужно **практически проверить** исходящие запросы с Worker; при блокировках/антиботе — Containers или прокси с фиксированными исходами (юридически и технически оценить).  
2. **Условия использования API Директа и OAuth** для публичного multi-tenant сервиса — требуется **юридическая** проверка (ограничения на передачу токенов третьей стороне, агентские сценарии).  
3. **Лимиты Workers** на длительные MCP-сессии и большие тела ответов — может понадобиться streaming и усечение, как в текущем `get_statistics`.  
4. **Единый бот и канал** — права админа бота в канале обязательны для `getChatMember`.  
5. **AsyncLocalStorage** — без него нужно аккуратно протащить `Operation`/отмену в HTTP-MCP; иначе упростить (без прогресса в hosted).  
6. **Два продукта в одном репо** — чёткое разделение пакетов (`packages/core`, `packages/cli`, `packages/cloud`) чтобы не сломать **stdio**-релиз на npm.

---

## 8. План по этапам (каждый этап — отдельный PR)

Объём оценивается **относительно** (S/M/L), без календарных сроков.

| № | Этап (PR) | Содержание | Объём |
| --- | --- | --- | --- |
| 1 | **Архитектурный каркас monorepo** | Пакет с ядром (`client`, `types`, read-only registry утилит), CLI-пакет текущего stdio без изменений поведения | M |
| 2 | **Убрать/изолировать Ask Ads telemetry** | Сборка `cloud` без вызовов `usage.gistrec.cloud`; документация | S |
| 3 | **Абстракция хранения токенов** | Интерфейс `CredentialStore` + реализация файловая (текущая) + заготовка «encrypted remote» | M |
| 4 | **Cloudflare Worker + D1 схема** | Таблицы пользователей, OAuth, mcp_tokens; Secrets; миграции D1 | M |
| 5 | **Telegram Mini App + API** | Валидация initData, getChatMember, выдача MCP URL и токена | L |
| 6 | **OAuth Яндекса (приложение владельца)** | Redirect flow, шифрование, refresh; интеграция с `TokenStore` | L |
| 7 | **HTTP MCP (McpAgent)** | Транспорт, Bearer, middleware проверки подписки + кеш | L |
| 8 | **Read-only режим** | Регистрация только разрешённых инструментов; `raw_request` только read; обновление `docs/` и capability-тестов по контракту репозитория | M |
| 9 | **Лимиты и наблюдаемость** | Rate limits, метрики Units, алерты; нагрузочные проверки отчётов | M |
| 10 | **Документация деплоя** | Wrangler, секреты, регистрация бота, переменные владельца OAuth | S |

---

## 9. Связанные файлы в репозитории

- Вход и stdio: `src/index.ts`  
- Клиент API: `src/client.ts`  
- OAuth и токены: `src/oauth.ts`, `src/auth.ts`, `src/credentials.ts`  
- Утилиты и классификация read/write: `src/tools/util.ts`, `src/tools/raw.ts`  
- Телеметрия: `src/telemetry.ts`  
- Документация инструментов: `docs/TOOLS.md`, `docs/DEVELOPMENT.md`  
- CI: `.github/workflows/ci.yml`
