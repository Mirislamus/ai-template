# Архитектура системы

## Обзор и подход: FSD-Lite

В проекте используется адаптированная версия Feature-Sliced Design (FSD-Lite). Она исключает избыточные слои (`processes`, `entities`), оставляя строгую иерархию для веб-сайтов и приложений. Этот файл — единственный источник структуры `src/`; остальные документы ссылаются на него.

## Структура слоев

```text
src/
├── pages/            # Страницы (Astro) или app/ (Next.js App Router)
├── layouts/          # Каркасы страниц и общие обертки (BaseLayout)
├── widgets/          # Крупные блоки и секции (header, footer, hero, faq, benefits)
├── features/         # Пользовательские сценарии и формы (lead-form, theme-toggle, catalog-filter)
├── content/          # Контент Astro Content Collections (схемы — в src/content.config.ts)
└── shared/           # Переиспользуемый базис без привязки к конкретному блоку
    ├── ui/           # Атомарный UI-кит (кнопки, инпуты, карточки, модалки)
    ├── styles/       # Токены, миксины, глобальные стили (_tokens.scss, _tools.scss, global.scss)
    ├── config/       # Валидация окружения (env.ts)
    ├── api/          # Клиент ky, QueryClient, фабрики ключей общего назначения
    ├── lib/          # Утилиты, аналитика, безопасные обертки над Web API
    ├── stores/       # Nano Stores (Astro) или Zustand (Next.js)
    ├── i18n/         # Словари и утилита t()
    └── types/        # Общие типы и DTO
```

Внутри слоя `features` и `widgets` каждый модуль имеет свои подпапки по необходимости (`api/`, `model/`, `ui/`) и явный фасад `index.ts` (ADR 0004 §5).

### Зоны ответственности слоев

1. **pages / app:** собирает страницу из готовых секций (`widgets`), задает SEO и мета-теги.
2. **layouts:** каркас документа, подключение шапки, подвала и общих скриптов.
3. **widgets:** визуально цельные блоки страницы. Виджет может объединять несколько `features` и элементы `shared`.
4. **features:** законченные пользовательские сценарии с бизнес-логикой (форма заявки, фильтрация, переключатель языка), включая свои TanStack Query хуки в `api/`.
5. **shared:** фундамент проекта без бизнес-специфики конкретного блока.

## Главное правило импортов

Импорты разрешены только сверху вниз:
`pages` → `layouts` → `widgets` → `features` → `shared`.

- Нижний слой ничего не знает о верхнем.
- Запрещены горизонтальные импорты: модуль из `features` не импортирует другой модуль из `features`, модуль из `widgets` не импортирует соседний `widgets`. Если логика нужна обоим, она выносится в `shared`.

## Движение данных и формы

- **Формы (`features`):** валидация Zod, маска `imask`, вызов API через `ky` и TanStack Query, пять состояний формы (ADR 0008).
- **Состояние UI:** хранится максимально локально внутри компонента.
- **Глобальное состояние:** `shared/stores/` через Nano Stores (Astro) или Zustand (Next.js).
- **URL как SSOT:** фильтры, пагинация и поиск синхронизируются со строкой запроса (`URLSearchParams`).

## Бэкенд (монорепо)

Отдельный бэкенд подключается только по решению в PRODUCT.md ([ADR 0019](docs/decisions/backend/0019-backend-architecture.md) §1). Тогда репозиторий становится монорепо: дерево фронтенда выше располагается в `apps/web/src/`, бэкенд — в `apps/api/`.

```text
apps/
├── web/                  # Фронтенд: структура src/ — выше (FSD-Lite)
└── api/
    ├── drizzle/          # SQL-миграции drizzle-kit (коммитятся)
    └── src/
        ├── index.ts      # Точка входа HTTP-сервера, корректная остановка
        ├── app.ts        # Сборка Elysia (префикс /api, обработчик ошибок, модули), export type App
        ├── worker.ts     # Точка входа обработчика очереди задач
        ├── config/       # env.ts: Zod-валидация окружения
        ├── db/           # client.ts, migrate.ts, seed.ts, schema/ (таблицы по сущностям)
        ├── modules/      # Фичи: <feature>/index.ts (маршруты), service.ts, model.ts, *.test.ts
        ├── jobs/         # Обработчики фоновых задач
        └── shared/       # auth, errors, logger, request-id, rate-limit, storage, alert
packages/
└── shared/               # Общие Zod-схемы, типы и константы фронтенда и бэкенда
```

- `apps/web` импортирует из `apps/api` только тип `App` для клиента Eden ([ADR 0020](docs/decisions/backend/0020-api-contract-and-errors.md) §1).
- Внутри модуля: маршруты → сервис → БД. Маршруты не обращаются к БД, сервисы не зависят от контекста Elysia ([ADR 0019](docs/decisions/backend/0019-backend-architecture.md) §3).

## Реестр архитектурных решений

Каталог `docs/decisions/` — единственный реестр ADR. Нумерация сквозная, ADR разложены по областям.

**Фронтенд (`docs/decisions/frontend/`):**
- `0001-project-architecture.md` — профили Astro и Next.js, FSD-Lite, гидратация, рендеринг, контент.
- `0002-html-standards.md` — семантика HTML5, доступность, нативный `<dialog>`, Skip Link.
- `0003-scss-standards.md` — SCSS Modules, токены, z-index, миксины, брейкпоинты.
- `0005-react-standards.md` — React 19, хуки, мемоизация, провайдеры библиотек.
- `0006-state-management.md` — уровни состояния, Nano Stores, Zustand, URL.
- `0007-assets-and-media.md` — изображения, шрифты, иконки, фавиконки.
- `0008-forms-and-api.md` — `ky`, TanStack Query, формы, антиспам, безопасность.
- `0013-seo-standards.md` — канонические URL, sitemap, JSON-LD, валидация.
- `0015-analytics-and-tracking.md` — отложенная аналитика, фасад `trackEvent`, уведомление о cookie.
- `0016-i18n-standards.md` — словари, `astro:i18n`, `Intl`.
- `0017-motion-and-animations.md` — GPU-анимации, `prefers-reduced-motion`, Scroll Reveal.

**Бэкенд (`docs/decisions/backend/`):**
- `0019-backend-architecture.md` — когда нужен бэкенд, монорепо, модули по фичам, очередь задач.
- `0020-api-contract-and-errors.md` — Eden Treaty, адреса и формат данных, пагинация, ошибки RFC 9457.
- `0021-database-and-migrations.md` — PostgreSQL, Drizzle, соглашения схемы, миграции, бэкапы.
- `0022-auth-and-access.md` — Better Auth, сессии, способы входа, роли, проверка владельца, 2FA.
- `0023-security-logging-and-config.md` — валидация, лимиты запросов, секреты, логи, алерты, файлы.

**Общие (`docs/decisions/common/`):**
- `0004-ts-js-standards.md` — строгий TypeScript и JavaScript, именование, импорты.
- `0009-tooling-and-linting.md` — Prettier, ESLint, Stylelint.
- `0010-third-party-libraries-policy.md` — политика версий, черный и белый списки.
- `0011-git-workflow-and-commits.md` — Conventional Commits, ветки, безопасность Git.
- `0012-build-and-package-manager.md` — Bun, бюджет бандла, окружение.
- `0014-testing-and-qa.md` — пирамида тестов, тесты бэкенда, Definition of Done.
- `0018-deployment-and-caching.md` — кеширование, деплой, Nginx, Docker Compose, CI/CD.

Новый ADR получает следующий свободный номер и кладется в папку своей области; общая для фронтенда и бэкенда тема — в `common/`.
