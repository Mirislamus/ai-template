# Текущее состояние проекта (STATE.md)

## Текущий статус
- **Фаза:** Проектирование стандартов шаблона завершено на 100%.
- **Готовность:** Сформирован полный комплект архитектурных решений (18 ADR), прикладной документации (`docs/`) и корневых управляющих файлов.
- **Блокеры:** Отсутствуют.

---

## Завершенные этапы

### 1. Архитектурный реестр решений (`docs/decisions/` — 18 ADR):
- [x] **ADR 0001:** Архитектура проекта, профили Astro (SSG) vs Next.js (SSR), методология FSD-Lite, гибридный рендеринг.
- [x] **ADR 0002:** Стандарты разметки W3C HTML5, семантика ориентиров, нативный `<dialog>`, `:focus-visible`, Skip Link.
- [x] **ADR 0003:** Стандарты SCSS Modules, семантическая шкала Z-Index (1–800), BEM-нотация, изоляция стилей.
- [x] **ADR 0004:** Строгий TypeScript / JavaScript, запрет `any` и `enum`, методы ES2023+, безопасные обертки над Web API.
- [x] **ADR 0005:** Стандарты React 19, плоские импорты, отказ от `React.FC` и `forwardRef`, запрет React Context.
- [x] **ADR 0006:** Управление состоянием (4 уровня стейта, Nano Stores `$`, Zustand, URL как SSOT, кросс-вкладочность).
- [x] **ADR 0007:** Оптимизация медиа (AVIF/WebP, нулевой CLS, самохостинг WOFF2, `lucide-react`, SVGO).
- [x] **ADR 0008:** Сетевой клиент `ky`, TanStack Query, формы, Honeypot-антиспам, 5 состояний UI, безопасность.
- [x] **ADR 0009:** Контроль качества (Prettier, ESLint Flat Config, Stylelint, единый конвейер `bun run check`).
- [x] **ADR 0010:** Политика библиотек (Latest Stable, lockfile в Git, черный/белый списки, протокол аудита).
- [x] **ADR 0011:** Git Workflow (Conventional Commits, атомарность, GitHub Flow, запрет force-push).
- [x] **ADR 0012:** Пакетный менеджер Bun, бюджет бандла, `build:analyze`, кроссплатформенность (`engines`).
- [x] **ADR 0013:** SEO-архитектура (каноникалы без слэша, автогенерация `sitemap.xml`, JSON-LD, `schema-dts`, Rich Results).
- [x] **ADR 0014:** Стратегия тестирования (Vitest, Playwright smoke-тесты, W3C валидация, Definition of Done).
- [x] **ADR 0015:** Аналитика без потери Lighthouse (отложенная загрузка счетчиков, фасад `trackEvent`, Cookie Consent).
- [x] **ADR 0016:** Архитектура интернационализации (TS-словари, `astro:i18n`, утилита `t()`, `Intl` локализация).
- [x] **ADR 0017:** Анимации и микровзаимодействия (GPU-only `transform`/`opacity`, `prefers-reduced-motion`, Observer).
- [x] **ADR 0018:** Деплой и кеширование (HTTP `Cache-Control` `immutable`/`no-cache`, Edge CDN, Nginx/SSH deploy).

### 2. Оперативная документация (`docs/`):
- [x] [docs/tech.md](docs/tech.md) — Технологический стек и согласованные библиотеки.
- [x] [docs/design.md](docs/design.md) — Инженерная дизайн-система: 8pt Grid, брейкпоинты, семантические токены, резиновые заголовки `clamp()`.
- [x] [docs/quality.md](docs/quality.md) — Чеклист стабильности верстки, нормативы Lighthouse (90+/100), кроссбраузерный Baseline.
- [x] [docs/seo.md](docs/seo.md) — Лимиты мета-тегов, семантика `h1`–`h3`, OpenGraph и чеклист страниц.
- [x] [docs/content.md](docs/content.md) — Экранная типографика («елочки», длинное тире, `&nbsp;`), инфостиль и микрокопирайтинг.

### 3. Корневые инструкции:
- [x] [README.md](README.md) — Документация шаблона, быстрый старт, команды и архитектура.

---

## Следующий шаг
- Инициализация кодовой базы проекта под выбранный стек (развертывание Astro / Next.js, базовых SCSS-токенов и конфигов линтеров).

