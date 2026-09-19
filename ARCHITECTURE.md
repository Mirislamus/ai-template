# Архитектура системы

## Обзор и подход: Lite FSD

В проекте используется адаптированная и практичная версия Feature-Sliced Design (Lite FSD). Она исключает избыточные слои (`processes`, `entities`), оставляя строгую и понятную иерархию для веб-сайтов и приложений.

## Структура слоев (сверху вниз)

```text
src/
├── pages/      # Страницы сайта (в Astro) или app/ (в Next.js App Router)
├── layouts/    # Каркасы страниц и общие обертки (BaseLayout)
├── widgets/    # Крупные самостоятельные блоки и секции (Header, Footer, Hero, FAQ)
├── features/   # Интерактивные действия пользователя и формы (lead-form, theme-toggle, catalog-filter)
└── shared/     # Переиспользуемый базис без привязки к конкретному контексту
    ├── ui/     # Атомарный UI-кит (кнопки, инпуты, карточки, модалки)
    ├── styles/ # Глобальные токены, переменные, миксины
    ├── config/ # Валидация окружения (env.ts)
    ├── api/    # Сетевой клиент ky, фабрики ключей TanStack Query
    ├── lib/    # Утилиты, хелперы аналитики, кастомные хуки
    └── types/  # Общие типы данных и DTO
```

### Зоны ответственности слоев

1. **pages / app:** собирает страницу из готовых секций (`widgets`), настраивает SEO и мета-теги.
2. **layouts:** каркас документа, подключение шапки, подвала и общих провайдеров.
3. **widgets:** визуально цельные блоки страницы (секции лендинга, шапка, подвал). Виджет может объединять несколько `features` и элементы `shared`.
4. **features:** законченные пользовательские сценарии с бизнес-логикой (форма заявки, фильтрация, переключатель языка).
5. **shared:** фундамент проекта, не содержащий бизнес-специфики конкретного блока.

## Главное правило импортов

**Импорты разрешены только строго сверху вниз:**
`pages` → `layouts` → `widgets` → `features` → `shared`.

- Нижний слой ничего не знает о верхнем.
- **Запрещены горизонтальные импорты:** модуль из `features` не может импортировать другой модуль из `features`. Модуль из `widgets` не импортирует соседний `widgets`. Если логика нужна обоим — она выносится в `shared`.

## Движение данных и формы

- **Формы (`features`):** содержат валидацию Zod, маску `imask`, вызов API через `ky` и отработку 5 состояний интерфейса.
- **Состояние UI:** хранится максимально локально внутри конкретного компонента.
- **Глобальное состояние:** выносится в `shared/stores/` через Nano Stores (в Astro) или Zustand (в Next.js).
- **URL как SSOT:** параметры фильтров, пагинации и поиска синхронизируются со строкой запроса (`URLSearchParams`).

## Архитектурные решения (Реестр из 18 ADR)

Все инженерные решения формализованы в каталоге `docs/decisions/`:
- `0001-project-architecture.md` — Профили Astro (SSG) vs Next.js (SSR), FSD-Lite и гибридный рендеринг.
- `0002-html-standards.md` — Семантика W3C HTML5, доступность a11y, нативный `<dialog>`, `:focus-visible`, Skip Link.
- `0003-scss-standards.md` — SCSS Modules, семантическая шкала Z-Index (1–800), BEM-нотация.
- `0004-ts-js-standards.md` — Строгий TypeScript / JavaScript, запрет `any`/`enum`, методы ES2023+.
- `0005-react-standards.md` — React 19, плоские импорты, `ref` без `forwardRef`, отказ от React Context.
- `0006-state-management.md` — 4 уровня стейта, Nano Stores (`$`), Zustand, URL как единственный источник истины.
- `0007-assets-and-media.md` — Оптимизация AVIF/WebP, нулевой CLS, самохостинг WOFF2, `lucide-react`.
- `0008-forms-and-api.md` — Сетевой клиент `ky`, TanStack Query, формы, Honeypot-антиспам, безопасность.
- `0009-tooling-and-linting.md` — Prettier, ESLint Flat Config, Stylelint, единый конвейер `bun run check`.
- `0010-third-party-libraries-policy.md` — Актуальность пакетов (Latest Stable), черный/белый списки, аудит.
- `0011-git-workflow-and-commits.md` — Conventional Commits, атомарность, GitHub Flow, безопасность.
- `0012-build-and-package-manager.md` — Пакетный менеджер Bun, бюджет бандла, `build:analyze`, кроссплатформенность.
- `0013-seo-standards.md` — Каноникалы, автогенерация `sitemap.xml`, JSON-LD, `schema-dts`, Google Rich Results.
- `0014-testing-and-qa.md` — Пирамида тестирования, Vitest, Playwright smoke-тесты, W3C валидация, Definition of Done.
- `0015-analytics-and-tracking.md` — Отложенная аналитика без просадки Lighthouse, фасад `trackEvent`, Cookie Consent.
- `0016-i18n-standards.md` — Архитектура интернационализации, TS-словари, `astro:i18n`, утилита `t()`.
- `0017-motion-and-animations.md` — GPU-анимации (`transform`/`opacity`), `prefers-reduced-motion`, `IntersectionObserver`.
- `0018-deployment-and-caching.md` — HTTP `Cache-Control` (`immutable`/`no-cache`), Edge CDN, Nginx/SSH deploy.

