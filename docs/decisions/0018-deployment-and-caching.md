# ADR 0018: Деплой, инфраструктура хостинга и стратегия кеширования

## Контекст и цели

Способ доставки и кеширования приложения определяет доступность сайта, скорость повторных визитов и стабильность релизов без даунтайма.
Цели архитектурного стандарта:
1. **Золотой стандарт кеширования (Cache-Control):** мгновенная отдача хэшированных ассетов из кеша (`immutable`) и гарантия свежести HTML (`no-cache`).
2. **Гибкость инфраструктуры:** бессерверные Edge CDN (Cloudflare Pages, Vercel, Netlify) либо собственный Linux-сервер с Nginx (деплой по SSH, PM2 для Node-процесса).
3. **Автоматизированный CI/CD:** Quality Gate (проверки, тесты, сборка) перед любой выкаткой.
4. **Сжатие трафика:** отдача статики через Brotli и Gzip на уровне веб-сервера.

---

## Принятые стандарты

### 1. HTTP-стратегия кеширования (Cache-Control)

На уровне веб-сервера (Nginx / CDN) действует трехуровневая политика:

1. **HTML-документы (чистые роуты без расширения):**
   - `Cache-Control: public, no-cache, must-revalidate`. Браузер кеширует документ, но перед показом отправляет условный запрос (`304 Not Modified`), поэтому после релиза пользователь сразу видит новую версию без ручной очистки кеша.
2. **Хэшированные статические ассеты (`_astro/*`, `_next/static/*`):**
   - `Cache-Control: public, max-age=31536000, immutable` (год). Сборщик добавляет контентный хэш в имя файла (`app-a1b2c3.js`), поэтому браузер не запрашивает файл повторно.
3. **Публичные файлы без хэша (`/fonts/*`, изображения, `/favicon.*`, `/robots.txt`, `/og/*`):**
   - `Cache-Control: public, max-age=86400, stale-while-revalidate=604800` (кеш 24 часа с фоновым обновлением). Это единственное значение для всех файлов без хэша, оно же используется в конфиге Nginx ниже.
4. **API (`/api/*`):** `Cache-Control: no-store`.

---

### 2. Сценарии хостинга и деплоя

#### Сценарий А: Edge CDN (рекомендуется для статического Astro)
- **Платформы:** Cloudflare Pages, Vercel, Netlify.
- **Команда сборки:** `bun run build` (артефакты в `dist/`).
- **Серверные эндпоинты** (`/api/*`, [ADR 0001](0001-project-architecture.md) §7) работают через адаптер платформы (`@astrojs/cloudflare`, `@astrojs/vercel`, `@astrojs/netlify`).
- **Плюсы:** глобальный CDN, отдача страниц за 10–20 мс. Редиректы (в том числе со слэша на адрес без слэша) настраиваются средствами платформы.

#### Сценарий Б: Собственный сервер Linux / Nginx (деплой по SSH)
- **Статика (Astro):**
  - Сборка на CI или локально (`bun run build`), синхронизация `dist/` на сервер по SSH:
    ```bash
    rsync -avz --delete dist/ user@server:/var/www/site/
    ```
  - Nginx сразу отдает новые файлы, простоя нет.
- **Node-процесс** (эндпоинты `/api/*` через `@astrojs/node`, Next.js SSR):
  - Процесс запускается под управлением **PM2**; Nginx проксирует на него запросы (см. конфиг):
    ```bash
    pm2 reload ecosystem.config.js --update-env
    ```
- Деплой без контейнеров является нормальным сценарием; Docker не обязателен.

---

### 3. Эталонная конфигурация Nginx

Требуется **nginx 1.25.1 или новее** (директива `http2 on;`; параметр `http2` в `listen` устарел). Заголовки безопасности вынесены в сниппет, потому что `add_header` внутри `location` отменяет заголовки уровня `server`; сниппет подключается в каждом блоке, где есть свой `add_header`.

`/etc/nginx/snippets/security-headers.conf`:
```nginx
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Виртуальный хост:
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name domain.ru;
    return 301 https://domain.ru$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name domain.ru;

    root /var/www/site;
    index index.html;

    # Сжатие
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;
    gzip_min_length 256;
    # brotli on;                 # если установлен модуль ngx_brotli
    # brotli_types text/css application/javascript image/svg+xml;

    # 301 со слэша на адрес без слэша (ADR 0013)
    rewrite ^/(.+)/$ /$1 permanent;

    error_page 404 /404.html;

    # 1. Хэшированные ассеты сборщика: вечный кеш
    location ^~ /_astro/ {
        include snippets/security-headers.conf;
        add_header Cache-Control "public, max-age=31536000, immutable";
        access_log off;
    }

    # 2. Публичные файлы без хэша
    location ~* \.(?:woff2|png|jpg|jpeg|webp|avif|ico|svg|txt|xml)$ {
        include snippets/security-headers.conf;
        add_header Cache-Control "public, max-age=86400, stale-while-revalidate=604800";
        access_log off;
    }

    # 3. Серверные эндпоинты Astro (сценарий Б, Node-процесс)
    location /api/ {
        include snippets/security-headers.conf;
        add_header Cache-Control "no-store";
        proxy_pass http://127.0.0.1:4321;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 4. HTML: проверка актуальности, несуществующий адрес отдает 404
    location / {
        include snippets/security-headers.conf;
        add_header Cache-Control "public, no-cache, must-revalidate";
        try_files $uri $uri.html $uri/index.html =404;
    }
}
```

- Сборка `build.format: 'file'` создает `services.html` для адреса `/services`, поэтому `try_files` подбирает `$uri.html`.
- Строка `error_page 404 /404.html;` отдает страницу [404](../pages/404.md) со статусом 404.
- Заголовки `Content-Security-Policy` и `Strict-Transport-Security` в шаблон не входят: они настраиваются под конкретный проект.

---

### 4. Автоматизация CI/CD (GitHub Actions)

Пайплайн выкатки состоит из двух этапов:

1. **Этап 1: Quality Gate (на каждый Pull Request и push в ветку):**
   - `bun install --frozen-lockfile` — детерминированная установка.
   - `bun run check` — форматирование, линтеры и типы.
   - `bun run test` — модульные тесты Vitest.
   - `bun run build` — сборка в рамках бюджета чанков.
   - `bun run validate:html` и `bun run test:e2e` — проверка разметки и smoke-сценарии.
   - При падении любого шага слияние в `main` запрещено. Поскольку локальных хуков в шаблоне нет ([ADR 0011](0011-git-workflow-and-commits.md)), этот этап — единственная автоматическая проверка `check`.
2. **Этап 2: Production Deploy (строго при слиянии в `main`):**
   - Публикация на CDN либо выкатка `rsync` по SSH на боевой Nginx-сервер.
