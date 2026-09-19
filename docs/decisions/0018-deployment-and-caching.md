# ADR 0018: Деплой, инфраструктура хостинга и стратегия кеширования

## Контекст и цели

Способ доставки и кеширования приложения определяет доступность сайта, скорость повторных визитов и стабильность релизов без даунтайма (Zero Downtime).
Цели архитектурного стандарта:
1. **Золотой стандарт кеширования (Cache-Control):** мгновенная отдача статических ассетов из кеша (`immutable`) и 100% гарантия свежести HTML (`no-cache`).
2. **Гибкость инфраструктуры хостинга:** поддержка как бессерверных Edge CDN (Cloudflare Pages, Vercel), так и собственных Linux VPS с Nginx / PM2 через безопасный SSH-деплой.
3. **Автоматизированный CI/CD пайплайн:** жесткий Quality Gate (линты, тесты, сборка) перед любой выкаткой в продакшен.
4. **Сжатие трафика:** обязательная отдача статики через Brotli и Gzip на уровне веб-сервера.

---

## Принятые стандарты

### 1. HTTP-стратегия кеширования (Cache-Control)

На уровне веб-сервера (Nginx / CDN) настраивается трехуровневая политика кеширования:

1. **HTML-документы (`*.html` и чистые роуты):**
   - Заголовок: `Cache-Control: public, no-cache, must-revalidate` (или `max-age=0`).
   - Браузер кеширует документ, но перед показом отправляет условный запрос к серверу (`304 Not Modified`). При релизе новой версии пользователь моментально видит обновленный HTML без ручной очистки кеша (Ctrl+F5).
2. **Хэшированные статические ассеты (`_astro/*`, `_next/static/*`, JS, CSS):**
   - Заголовок: `Cache-Control: public, max-age=31536000, immutable` (вечный кеш на 1 год).
   - Так как сборщики (Vite / Turbopack) вшивают контентный хэш в имя файла (`app-a1b2c3.js`), браузер никогда не запрашивает один и тот же файл повторно.
3. **Публичные медиа без хэша (`/favicon.ico`, `/robots.txt`, `/og-image.jpg`):**
   - Заголовок: `Cache-Control: public, max-age=86400, stale-while-revalidate=604800` (кеш на 24 часа с фоновым обновлением).

---

### 2. Сценарии хостинга и деплоя

Проект поддерживает два основных сценария развертывания:

#### Сценарий А: Статический Edge CDN (Рекомендуется для Astro SSG)
- **Платформы:** Cloudflare Pages, Vercel, Netlify.
- **Команда сборки:** `bun run build` (артефакты в `dist/`).
- **Плюсы:** бесплатный глобальный CDN, отдача страниц за 10–20 мс, абсолютная защита от падений под нагрузкой.

#### Сценарий Б: Собственный сервер Linux / Nginx (SSH Deploy)
- **Для статики (Astro SSG):**
  - Сборка на CI или локально (`bun run build`).
  - Безопасная синхронизация папки `dist/` на сервер через `rsync` по SSH:
    ```bash
    rsync -avz --delete dist/ user@server:/var/www/site/
    ```
  - Отсутствие простоя (Zero Downtime) — Nginx мгновенно подхватывает новые файлы.
- **Для Node.js сервера (Next.js SSR / Astro Node Adapter):**
  - Запуск процесса под управлением менеджера **PM2**:
    ```bash
    pm2 reload ecosystem.config.js --update-env
    ```

---

### 3. Эталонная конфигурация Nginx (`nginx.conf`)

Для собственных VPS-серверов используется стандартизированный блок виртуального хоста:

```nginx
server {
    listen 443 ssl http2;
    server_name domain.ru;

    root /var/www/site;
    index index.html;

    # Сжатие Brotli и Gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;
    gzip_min_length 256;

    # Заголовки безопасности
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # 1. Вечный кеш для хэшированных ассетов сборщика
    location ~* ^/(?:_astro|_next/static)/ {
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable";
        access_log off;
    }

    # 2. Кеш для картинок, шрифтов и медиа
    location ~* \.(?:woff2|png|jpg|jpeg|webp|avif|ico|svg)$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, stale-while-revalidate=604800";
        access_log off;
    }

    # 3. HTML-страницы: проверка актуальности (no-cache)
    location / {
        try_files $uri $uri/ $uri.html /index.html =404;
        add_header Cache-Control "public, no-cache, must-revalidate";
    }
}
```

---

### 4. Автоматизация CI/CD (GitHub Actions)

Пайплайн выкатки состоит из строгого двухэтапного конвейера:

1. **Этап 1: Quality Gate (на каждый Pull Request и push в ветку):**
   - `bun install --frozen-lockfile` — чистая детерминированная установка.
   - `bun run check` — проверка форматирования, линтеров и типов.
   - `bun run test` — выполнение модульных тестов Vitest.
   - `bun run build` — верификация успешности сборки в рамках бюджета чанков.
   - *Блокировка:* при падении любого шага мердж в `main` запрещен.
2. **Этап 2: Production Deploy (строго при мердже в `main`):**
   - Автоматическая публикация на CDN либо выкатка через SSH `rsync` на боевой Nginx-сервер.

