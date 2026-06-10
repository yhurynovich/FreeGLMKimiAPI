# FreeGLMKimiAPI

Отдельный OpenAI/Anthropic-compatible прокси для web-аккаунтов **GLM/Z.ai (chat.z.ai, legacy chatglm.cn)** и **Kimi (kimi.com)** — по духу как локальные FreeQwenApi/FreeDeepseekAPI.

Главное:
- `/v1/chat/completions` — OpenAI-compatible chat, streaming и non-streaming.
- `/v1/messages` — минимальный Anthropic Messages shim для Claude Code.
- `/v1/models`, `/health`, `/sessions`.
- Prompt-based симуляция tool use для моделей без нативного function calling: прокси инжектит протокол, парсит `[function_calls]`, JSON, XML-ish и legacy `TOOL_CALL` ответы и возвращает OpenAI `tool_calls`.
- Per-agent сессии по `user`/заголовкам; Kimi и Z.ai хранят `chat_id + parent_id`, legacy ChatGLM хранит `conversation_id`.
- `MOCK_PROVIDER=1` для локальных тестов без токенов и проверки агентных клиентов.

## Быстрый старт

```bash
cd /Users/forgetme/projects/FreeGLMKimiAPI
npm install
cp .env.example .env
npm test
MOCK_PROVIDER=1 PORT=9766 npm start
```

Проверка:

```bash
curl http://127.0.0.1:9766/health
curl http://127.0.0.1:9766/v1/models
node scripts/smoke.js
```

## Конфиг аккаунтов и токенов

Есть три способа добавить аккаунты.

### 1. `auth.json` — основной способ

Создай `auth.json` рядом с проектом или укажи `AUTH_PATH=/path/to/auth.json`:

```json
{
  "accounts": [
    { "id": "glm1", "provider": "glm", "token": "zai_token..." },
    { "id": "glm2", "provider": "glm", "token": "another_zai_token..." },
    { "id": "glm-cn-legacy", "provider": "glm", "backend": "chatglm", "refresh_token": "chatglm_refresh_token..." },
    { "id": "kimi1", "provider": "kimi", "token": "Kimi JWT/access token..." },
    { "id": "kimi2", "provider": "kimi", "token": "another_Kimi_JWT..." }
  ]
}
```

### 2. Env-переменные — удобно для одного аккаунта

```bash
GLM_TOKEN=... KIMI_TOKEN=... npm start
# legacy chatglm.cn only:
GLM_BACKEND=chatglm GLM_REFRESH_TOKEN=... npm start
```

### 3. Admin API — добавить без рестарта

Если `API_KEYS` задан, все admin-запросы тоже требуют HTTP-заголовок авторизации с твоим API key.

```bash
# список аккаунтов без секретов
curl http://127.0.0.1:9766/admin/accounts

# добавить Kimi runtime-only, без записи в auth.json
curl -X POST http://127.0.0.1:9766/admin/accounts \
  -H 'Content-Type: application/json' \
  -d '{"id":"kimi3","provider":"kimi","token":"KIMI_TOKEN_HERE","persist":false}'

# добавить GLM и сохранить в auth.json
curl -X POST http://127.0.0.1:9766/admin/accounts \
  -H 'Content-Type: application/json' \
  -d '{"id":"glm3","provider":"glm","token":"ZAI_TOKEN_HERE","persist":true}'

# удалить аккаунт
curl -X DELETE 'http://127.0.0.1:9766/admin/accounts/kimi3?persist=false'

# перечитать auth.json
curl -X POST http://127.0.0.1:9766/admin/accounts/reload -d '{}'
```

Поведение пула аккаунтов:

- выбор провайдера идёт по модели: `glm*` → GLM, `kimi*` → Kimi;
- новые агенты распределяются round-robin;
- сессия агента sticky: один `user`/agent id старается ходить через тот же аккаунт;
- если аккаунт дал ошибку провайдера, он уходит в cooldown на `ACCOUNT_COOLDOWN_MS`, запрос пробуется на следующем аккаунте;
- `/health` и `/admin/accounts` показывают только `id/provider/hasToken/stat`, сами токены не отдают.

## Как получить токены

Токены — это секреты уровня пароля. Не публикуй их, не коммить `auth.json`, лучше держать сервис только локально или за `API_KEYS`.

### Kimi token

Kimi сейчас ходит через gRPC-Web endpoint `/apiv2/kimi.gateway.chat.v1.ChatService/Chat`. Проекту нужен **Bearer/access JWT**, без слова `Bearer`.

**Вариант A — Network, самый надёжный:**

1. Открой `https://www.kimi.com/` и залогинься.
2. Открой DevTools: `F12` / `Cmd+Option+I` → вкладка **Network**.
3. В фильтре Network введи `ChatService/Chat` или `kimi.gateway.chat`.
4. Отправь короткое сообщение в Kimi.
5. Открой запрос к `/apiv2/kimi.gateway.chat.v1.ChatService/Chat`.
6. В **Request Headers** найди `Authorization: Bearer eyJ...`.
7. Скопируй только часть после `Bearer ` — длинный JWT вида `eyJ...`.`...`.`...`.

**Вариант B — Application/Console:**

1. DevTools → **Application** → Local Storage → `https://www.kimi.com`.
2. Ищи ключи вроде `access_token`, `token`, `auth`, `jwt`.
3. Быстрый поиск в Console:

```js
Object.entries(localStorage)
  .filter(([k, v]) => /token|access|auth|jwt/i.test(`${k} ${v}`))
```

**Проверка токена:**

- JWT обычно начинается с `eyJ` и состоит из 3 частей через точки.
- В payload часто есть `app_id: "kimi"`, `typ: "access"`, `sub`, `exp`.
- Если получил 401/`auth.token.invalid`, открой Kimi в браузере, обнови страницу, отправь сообщение и скопируй свежий Bearer.

Формат для проекта:

```json
{ "id": "kimi1", "provider": "kimi", "token": "PASTE_KIMI_BEARER_TOKEN_HERE" }
```

Через env:

```bash
KIMI_TOKEN="PASTE_KIMI_BEARER_TOKEN_HERE" npm start
```

### GLM / Z.ai token — основной вариант

Для актуального GLM web используется `https://chat.z.ai`. Это **Z.ai adapter**, не legacy `chatglm.cn`. Для GLM-5 фронт может использовать completion v2 endpoint `/api/v2/chat/completions`, плюс подпись `X-Signature` и `X-FE-Version`.

**Вариант A — Application, самый быстрый:**

1. Открой `https://chat.z.ai/` и залогинься.
2. DevTools → **Application**.
3. Проверь оба места:
   - Local Storage → `https://chat.z.ai` → ключ `token`;
   - Cookies → `chat.z.ai` → cookie `token`.
4. Скопируй только значение токена, без `Bearer` и без `token=`.

**Вариант B — Network, если в Application непонятно:**

1. DevTools → **Network**.
2. Отправь короткое сообщение в Z.ai/GLM.
3. Ищи запросы:
   - `/api/v1/chats/new` — создание чата;
   - `/api/v2/chat/completions` — GLM-5 completion;
   - иногда `/api/chat/completions` для старой completion version.
4. В Request Headers скопируй:
   - `Authorization: Bearer eyJ...` → нужен только JWT после `Bearer `;
   - или cookie `token=eyJ...` → нужен только value после `token=`.
5. Если Network показывает `FRONTEND_CAPTCHA_REQUIRED`, сначала в обычном браузере обнови `chat.z.ai`, отправь сообщение и пройди проверку/captcha. После успешной отправки скопируй свежий token и повтори тест в прокси.

**Проверка токена:**

- JWT Z.ai часто содержит `id` и `email` в payload.
- Если `/health` видит аккаунт, но chat возвращает `Z.ai error: 请刷新页面以更新应用后重试。` / `FRONTEND_CAPTCHA_REQUIRED`, сам token живой, но web API требует браузерную captcha-проверку/обновление фронта. Решение: открыть `chat.z.ai` тем же аккаунтом, refresh, отправить сообщение вручную, затем скопировать новый token.
- Если внутри SSE видно `captcha_error_type: "missing_param"`, текущий фронт Z.ai требует `captcha_verify_param` из браузерной Aliyun-captcha проверки. Прокси умеет передать его через поле аккаунта `captcha_verify_param` или env `ZAI_CAPTCHA_VERIFY_PARAM` (обычно это base64-строка из request body, её не надо JSON.parse). Для устойчивости также можно передать полный browser Cookie через поле аккаунта `cookie` / env `ZAI_COOKIE`, потому что Z.ai проверяет cdn/acw/ssxmod anti-bot cookies, а не только JWT token.
- Если ошибка 401 — токен протух/не тот аккаунт.
- Если ошибка 426 или “New version detected” — обнови `ZAI_FE_VERSION` через env или возьми актуальное значение из HTML `chat.z.ai` (`prod-fe-...`).

Формат для проекта:

```json
{ "id": "zai1", "provider": "glm", "token": "PASTE_ZAI_TOKEN_HERE" }
```

Если копируешь полный successful browser request и Z.ai требует captcha:

```json
{
  "id": "zai1",
  "provider": "glm",
  "token": "PASTE_ZAI_TOKEN_HERE",
  "cookie": "cdn_sec_tc=...; acw_tc=...; ssxmod_itna=...; token=...",
  "captcha_verify_param": "PASTE_BASE64_CAPTCHA_VERIFY_PARAM_HERE"
}
```

Через env:

```bash
GLM_TOKEN="PASTE_ZAI_TOKEN_HERE" npm start
# если фронт обновился:
ZAI_FE_VERSION="prod-fe-1.1.46" GLM_TOKEN="PASTE_ZAI_TOKEN_HERE" npm start
```

Ручной curl больше не обязателен. Для устойчивого режима используй browser-profile: скрипт один раз открывает видимый Chrome, ждёт логин, вытаскивает Z.ai token из localStorage/network и сохраняет `auth.json` с `browser_fallback: true`.

```bash
npm run auth:browser -- ./auth.json
ZAI_BROWSER_FALLBACK=1 MODEL=GLM-5.1 npm run smoke:zai
ZAI_BROWSER_FALLBACK=1 MODEL=GLM-5.1 npm start
```

По умолчанию профиль хранится тут:

```text
~/.free-glm-kimi-api/zai-browser-profile
```

После первого логина он переиспользуется, поэтому не нужно каждый раз копировать fresh curl/captcha. Если Z.ai снова просит проверку, запусти `npm run auth:browser` — он откроет тот же профиль, достаточно пройти проверку в видимом окне.

Важно: Z.ai может выдать временный guest-token ещё до настоящего логина. `auth:browser` по умолчанию игнорирует такие `guest-...@guest.com` токены и держит окно открытым, пока ты не залогинишься реальным аккаунтом. Если браузер всё равно сам закрылся, проверь вывод: `email` в сохранённом JSON не должен быть `guest-...@guest.com`.

Для более антидетектного browser fallback есть экспериментальный режим CloakBrowser, подсмотренный у CloakHQ: source-level stealth Chromium, persistent non-incognito profile, humanized input, timezone/locale через browser flags, очистка stale profile locks.

```bash
ZAI_BROWSER_ENGINE=cloak \
ZAI_BROWSER_FALLBACK=1 \
ZAI_BROWSER_HEADLESS=0 \
ZAI_BROWSER_HUMANIZE=1 \
MODEL=GLM-5.1 npm run smoke:zai
```

Полезные env-переменные:

- `ZAI_BROWSER_ENGINE=puppeteer|cloak` — обычный `puppeteer-extra` или CloakBrowser.
- `ZAI_BROWSER_PROFILE_DIR=...` — отдельный Chrome/Cloak профиль. Если не задано, `puppeteer` использует `~/.free-glm-kimi-api/zai-browser-profile`, а `cloak` — отдельный `~/.free-glm-kimi-api/zai-cloak-profile`, чтобы не падать на несовместимом Chrome profile.
- `ZAI_BROWSER_PROXY=http://user:pass@host:port` или `socks5://...` — если нужен residential proxy.
- `ZAI_BROWSER_GEOIP=1` — в CloakBrowser подгонять timezone/locale под proxy IP.
- `ZAI_BROWSER_LOCALE=ru-RU`, `ZAI_BROWSER_TIMEZONE=Europe/Samara` — явная связка локали/таймзоны.

Fallback включается либо env-переменной:

```bash
ZAI_BROWSER_FALLBACK=1 npm start
```

либо полем аккаунта:

```json
{ "id": "zai1", "provider": "glm", "token": "PASTE_ZAI_TOKEN_HERE", "browser_fallback": true }
```

Старый вариант с successful browser curl тоже оставлен для диагностики:

```bash
pbpaste | node scripts/import_zai_curl.js /dev/stdin /tmp/freeglm-auth.json
AUTH_PATH=/tmp/freeglm-auth.json MODEL=GLM-5.1 npm run smoke:zai
```

Скрипты печатают только статус/наличие полей, сами секреты не выводят.

### Legacy GLM / chatglm.cn refresh token

Старый китайский backend `chatglm.cn` оставлен как fallback. Включается через `GLM_BACKEND=chatglm` или поле аккаунта `"backend":"chatglm"`. Это не то же самое, что `chat.z.ai`.

1. Открой `https://chatglm.cn/` и залогинься.
2. DevTools → **Network** → отправь сообщение.
3. Найди `/user-api/user/refresh` или `/backend-api/assistant/stream`.
4. Для `/user-api/user/refresh` скопируй refresh token и клади его как `refresh_token`.
5. Если копируешь `Authorization: Bearer ...`, не включай слово `Bearer`.

```json
{ "id": "glm-cn1", "provider": "glm", "backend": "chatglm", "refresh_token": "PASTE_CHATGLM_REFRESH_TOKEN_HERE" }
```

Быстрая проверка после добавления:

```bash
curl http://127.0.0.1:9766/health
node scripts/doctor.js
```

## Модели

- `glm-5`, `glm-5-thinking`, `glm-5-search`, `glm-5-deepresearch`
- `kimi-k2.5`, `kimi-k2.5-thinking`, `kimi-k2.5-search`

Выбор провайдера идёт по имени модели (`glm*` → GLM, `kimi*` → Kimi). `DEFAULT_PROVIDER` и `DEFAULT_MODEL` — fallback.

## OpenAI tools

Клиент отправляет обычные OpenAI tools:

```json
{
  "model": "kimi-k2.5",
  "messages": [{"role":"user","content":"создай файл hello.txt"}],
  "tools": [{"type":"function","function":{"name":"write_file","description":"write file","parameters":{"type":"object","properties":{"path":{"type":"string"},"content":{"type":"string"}},"required":["path","content"]}}}]
}
```

Если web-модель отвечает протокольным текстом, прокси возвращает:

```json
{"choices":[{"message":{"role":"assistant","content":null,"tool_calls":[...]},"finish_reason":"tool_calls"}]}
```

## Anthropic / Claude Code smoke

Прокси принимает `/v1/messages` и конвертирует Anthropic tools/tool_result в OpenAI-style loop.

Локальные protocol-level agent smoke-тесты без реальных токенов:

```bash
MOCK_PROVIDER=1 PORT=9766 npm start
npm run agent:all        # hermes + opencode + openclaw + claude
npm run agent:hermes
npm run agent:claude     # Anthropic Messages / Claude Code shape
npm run agent:opencode   # OpenAI-compatible AI SDK shape
npm run agent:openclaw   # OpenAI-compatible local agent shape
```

Для Claude Code CLI:

```bash
ANTHROPIC_BASE_URL=http://127.0.0.1:9766 \
ANTHROPIC_API_KEY=dummy \
ANTHROPIC_MODEL=kimi-k2.5 \
claude --bare -p 'Reply exactly: CLAUDE_SMOKE_OK' --model kimi-k2.5 --output-format json
```

Для OpenCode:

```bash
export OPENCODE_CONFIG_CONTENT='{"$schema":"https://opencode.ai/config.json","provider":{"free-glm-kimi":{"npm":"@ai-sdk/openai-compatible","name":"FreeGLMKimiAPI","options":{"baseURL":"http://127.0.0.1:9766/v1","apiKey":"dummy"},"models":{"kimi-k2.5":{"name":"kimi-k2.5"},"glm-5":{"name":"glm-5"}}}}}'
opencode run 'Reply exactly: OPENCODE_SMOKE_OK' --model free-glm-kimi/kimi-k2.5 --agent build
```

## Реальные ограничения

Web API GLM/Z.ai/Kimi меняются. Реализация сверена по Chat2API (`babsso25/Chat2API`, commit `9c65481`), включая отдельные adapters `zai` и legacy `glm` и покрыта локальными мок-тестами. Для настоящего E2E нужны живые токены и возможная донастройка headers после очередного изменения фронта.
