# Заметки по CloakBrowser для FreeGLMKimiAPI

Изученные источники:

- `https://github.com/CloakHQ/CloakBrowser` на коммите `b06499b`
- `https://github.com/CloakHQ/CloakBrowser-Manager` на коммите `a85b213`

## Полезные идеи

1. **Когда упираемся в anti-bot, нужен настоящий пропатченный браузер.** CloakBrowser — это не просто JS stealth-инъекции: он поставляет пропатченный Chromium и даёт Playwright-совместимый `launchPersistentContext()`.
2. **Постоянные не-incognito профили важны.** CloakBrowser и Manager предпочитают постоянные директории user data: так сохраняются cookies/localStorage и меньше риск детекта, характерного для incognito-сессий.
3. **«Человечные» действия в интерфейсе — часть fallback-сценария.** Их слой `humanize` печатает, кликает и скроллит с реалистичными задержками и через доверенные input-пути. Для Z.ai это особенно важно, когда прямой `fetch()` из браузерного контекста попадает в captcha/WAF и приходится отправлять запрос через видимый UI.
4. **Согласованность locale/timezone/proxy важна.** CloakBrowser прокидывает timezone/locale через флаги браузера и может согласовывать их с IP прокси (`geoip`). В этом проекте для Z.ai это вынесено в env-переменные, чтобы не хардкодить один fingerprint.
5. **Нужно чистить старые lock-файлы профиля.** Manager удаляет `SingletonLock`, `SingletonCookie` и `SingletonSocket` перед запуском, чтобы упавшие браузерные сессии не ломали следующий старт профиля.

## Что применено в этом проекте

- Добавлен опциональный режим `ZAI_BROWSER_ENGINE=cloak` для browser fallback.
- Добавлены зависимости `cloakbrowser` и `playwright-core`.
- Движком по умолчанию оставлен `puppeteer`, чтобы не заставлять каждого пользователя скачивать бинарник CloakBrowser примерно на 200 МБ.
- Для Cloak-режима используется отдельный профиль по умолчанию (`~/.free-glm-kimi-api/zai-cloak-profile`), чтобы он не падал на профиле, ранее созданном обычным Chrome/Puppeteer.
- Cloak-режим использует `launchPersistentContext()` с:
  - `userDataDir` = `ZAI_BROWSER_PROFILE_DIR` / профиль Z.ai по умолчанию;
  - включённым по умолчанию `humanize`;
  - поддержкой env-переменных для locale/timezone/proxy/geoip;
  - постоянным не-incognito профилем.
- Добавлена очистка старых lock-файлов для обоих движков.
- UI fallback для ввода промпта изменён с прямого присваивания value на click/select/type со случайными задержками, где это поддерживается.
- Расширена классификация captcha/WAF для маркеров Aliyun/WAF.

## Переменные запуска

```bash
ZAI_BROWSER_ENGINE=cloak \
ZAI_BROWSER_FALLBACK=1 \
ZAI_BROWSER_HEADLESS=0 \
ZAI_BROWSER_HUMANIZE=1 \
MODEL=GLM-5.1 npm run smoke:zai
```

Дополнительно:

- `ZAI_BROWSER_PROFILE_DIR=...`
- `ZAI_BROWSER_PROXY=http://user:pass@host:port` или `socks5://...`
- `ZAI_BROWSER_GEOIP=1`
- `ZAI_BROWSER_LOCALE=ru-RU`
- `ZAI_BROWSER_TIMEZONE=Europe/Samara`
- `ZAI_BROWSER_ARGS="--flag=value ..."`

## Важное ограничение

CloakBrowser может уменьшить automation fingerprints и предотвратить часть проверок, но он не решает уже сработавшую captcha на аккаунте/IP магическим образом. Если Z.ai требует ручную проверку, открой тот же постоянный профиль в видимом режиме и один раз пройди проверку руками.
