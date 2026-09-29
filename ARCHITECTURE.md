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

## Реестр архитектурных решений

Каталог `docs/decisions/` — единственный реестр ADR:

- `0001-project-architecture.md` — профили Astro и Next.js, FSD-Lite, гидратация, рендеринг, контент.
- `0002-html-standards.md` — семантика HTML5, доступность, нативный `<dialog>`, Skip Link.
- `0003-scss-standards.md` — SCSS Modules, токены, z-index, миксины, брейкпоинты.
- `0004-ts-js-standards.md` — строгий TypeScript и JavaScript, именование, импорты.
- `0005-react-standards.md` — React 19, хуки, мемоизация, провайдеры библиотек.
- `0006-state-management.md` — уровни состояния, Nano Stores, Zustand, URL.
- `0007-assets-and-media.md` — изображения, шрифты, иконки, фавиконки.
- `0008-forms-and-api.md` — `ky`, TanStack Query, формы, антиспам, безопасность.
- `0009-tooling-and-linting.md` — Prettier, ESLint, Stylelint.
- `0010-third-party-libraries-policy.md` — политика версий, черный и белый списки.
- `0011-git-workflow-and-commits.md` — Conventional Commits, ветки, безопасность Git.
- `0012-build-and-package-manager.md` — Bun, бюджет бандла, окружение.
- `0013-seo-standards.md` — канонические URL, sitemap, JSON-LD, валидация.
- `0014-testing-and-qa.md` — пирамида тестов, Definition of Done.
- `0015-analytics-and-tracking.md` — отложенная аналитика, фасад `trackEvent`, уведомление о cookie.
- `0016-i18n-standards.md` — словари, `astro:i18n`, `Intl`.
- `0017-motion-and-animations.md` — GPU-анимации, `prefers-reduced-motion`, Scroll Reveal.
- `0018-deployment-and-caching.md` — кеширование, деплой, Nginx, CI/CD.
