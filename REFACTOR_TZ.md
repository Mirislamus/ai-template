# ТЗ: рефакторинг шаблона ai-template

Временный рабочий документ. Удаляется после закрытия всех пунктов.

**Статус: блоки A–F выполнены (без коммита).** Проверки: поиск остатков старых значений, разбор всех JSON-блоков, проверка относительных ссылок (0 битых), окончания строк (везде LF). Принятые решения по блоку G: Zod 4 (`z.email()`, параметр `error`), TypeScript 6 без `baseUrl`, `defineConfig` из `eslint/config`, Nginx `http2 on;` (nginx 1.25.1+), FAQPage без ожидания rich results Google, `cross-env` заменен записью `VAR=value команда` в оболочке Bun, версии Bun и Node в документации не фиксируются. Файл можно удалить после просмотра изменений.

## Цель, границы и правила выполнения

**Цель:** устранить поломки, противоречия между документами, устаревшие API и пробелы, чтобы шаблон можно было скопировать в новый проект и развернуть без ручных исправлений.

**Вне scope:** смена стека, новые технологии и правила, не упомянутые в этом ТЗ, инициализация кода проекта. Всё, что замечено дополнительно, попадает в раздел «Замечено, но не входит без решения» и не делается без решения.

**Правила выполнения:**

1. Каждое правило живёт в одном документе-источнике, остальные документы ссылаются на него.
2. Никаких захардкоженных счётчиков («18 ADR», «9 веток»).
3. Внешние факты подтверждаются первоисточником со ссылкой. Непроверенное помечается `[Предположение]`.
4. Пункты, зависящие от открытых вопросов (Q1–Q7), не выполняются до ответа.
5. Коммиты — по одному на блок, только после разрешения пользователя.

---

## Блок A. P0 — поломки

| # | Задача | Файлы | Критерий приёмки |
|---|---|---|---|
| A1 | Удалить вывод ошибок PowerShell (`oh-my-posh`, `Export-Clixml`) из начала файлов | `AGENTS.md`, `USAGE.md`, `docs/tech.md`, `docs/setup.md`, `docs/content.md` | Поиск `Failed to write init script` и `Export-Clixml` → 0 совпадений; файлы начинаются с заголовка |
| A2 | Исправить JSON в секции `scripts`: экранировать кавычки. Добавить `test:e2e` и `validate:html`. Добавить вариант `scripts` для Next.js | `docs/setup.md` | Каждый JSON-блок парсится; все команды из README есть в `scripts` |
| A3 | Синхронизировать команды установки с конфигами: `typescript-eslint`, `@eslint/js`, `@playwright/test`, `html-validate`, `schema-dts`, `@astrojs/sitemap`, `dompurify`, `react-error-boundary` (Astro). `prettier-plugin-astro` — только в профиле Astro. `cross-env` — по итогам G8 | `docs/setup.md`, `.prettierrc` | Каждый импорт и бинарь в конфигах и скриптах есть в команде установки своего профиля |
| A4 | Создать `.gitignore` по ADR 0011 (+ `.env`, `stats.html`, `.impeccable/`). В `.gitattributes` — `* text=auto eol=lf` | корень | Файлы существуют; список совпадает с ADR 0011 |
| A5 | Переписать конфиг Nginx: общий сниппет заголовков безопасности в каждом `location` (`add_header` не наследуется), `error_page 404 /404.html` и `try_files` без фоллбэка на `/index.html`, 301 со слэша на адрес без слэша, Brotli (с пометкой о модуле), синтаксис `http2` по G6. В ADR 0013 добавить `build.format: 'file'` для согласования с `trailingSlash: 'never'` | ADR 0018, ADR 0013 | Несуществующий URL → 404; `/page/` → 301 на `/page`; заголовки есть на HTML и ассетах |
| A6 | Серверный обработчик форм для Astro-профиля (Q2): эндпоинт `src/pages/api/contact.ts` с `export const prerender = false`, остальной сайт остаётся статикой. Адаптер по хостингу: `@astrojs/node` (standalone) для VPS, адаптер платформы для Cloudflare / Vercel / Netlify. Эндпоинт использует ту же Zod-схему, что и клиент, проверяет honeypot и time-trap, отправляет данные в интеграцию с серверными секретами. В ADR 0018: Node-процесс под PM2, в Nginx `location /api/` проксируется на него. Выбор адаптера — на допросе по продукту (ветка хостинга, E7) | ADR 0001, 0008, 0018, setup.md, `.env.example` | Статическая сборка + рабочий `POST /api/contact` в обоих сценариях деплоя ADR 0018; секреты не попадают в клиентский бандл |

---

## Блок B. Единые стандарты (устранение противоречий)

| # | Задача | Решение | Источник истины |
|---|---|---|---|
| B1 | Путь и состав глобальных стилей | `src/shared/styles/`: `_tokens.scss`, `_tools.scss`, `global.scss` (структура ADR 0003, путь из ARCHITECTURE). Убрать `src/styles/`, `tokens/`, `base.scss` и отдельные `_spacing/_z-index/_typography` | ADR 0003 |
| B2 | Шкала z-index | Одна шкала 1–800 в CSS-переменных `--z-*`. Удалить §13 ADR 0003 (1–110) и SCSS-переменные `$z-*`. Skip Link — через токен | ADR 0003 |
| B3 | Цветовые токены | Семантические имена из design.md (`--color-bg-page`, `--color-text-primary`, `--color-accent`…). Заменить `--color-primary`, `--color-page`, `--color-text` и фоллбэки `#3b82f6`/`#000`/`#fff` во всех ADR | design.md |
| B4 | Типографические токены | Имена `--font-size-*` и `clamp()` (vw + px) из design.md. Токены интерлиньяжа — `--line-height-*`. Удалить набор `--font-h1…` из ADR 0003 | design.md |
| B5 | Брейкпоинты и контейнер | По Q3: `sm 576`, `md 768`, `lg 1024`, `xl 1280`, `2xl 1440`; нижняя граница поддержки 360px (не брейкпоинт). Контейнер `--container-max-width: 1280px`, поля 16px до `md` и 32px от `md`. Заменить 992/1200 в design.md и «1200 или 1280»; ADR 0003 ссылается на design.md. Брейкпоинты — Sass-переменные для медиазапросов (CSS-переменные в `@media` не работают) | design.md |
| B6 | Синтаксис медиазапросов | Range syntax: `(width >= X)`, мобильный — `(width < 768px)`. Исправить `client:media="(max-width: 768px)"` и bottom sheet (сейчас пересекаются на 768px) | ADR 0003 |
| B7 | Hover | Один миксин `hover` с `(hover: hover) and (pointer: fine)`; design.md и quality.md ссылаются на него | ADR 0003 |
| B8 | Токены анимации | `--motion-fast/base/slow` + `--ease-out`. Удалить `--transition-*`; значения в design.md и ADR 0017 — через токены | ADR 0003 |
| B9 | Порядок CSS-свойств | Один список, совпадающий с конфигом stylelint; ADR 0003 §9 ссылается на него | ADR 0009 |
| B10 | Trailing slash | Везде без слэша; исправить канонический URL и правило в ADR 0002 §14. Отдельно описать корень `https://domain.ru/` | ADR 0013 |
| B11 | БЭМ | Плоские классы SCSS Modules. Убрать «BEM» из README и STATE. В HTML-примерах ADR 0002 и в 404.md указать, что классы условные | ADR 0003 |
| B12 | FSD-Lite | Одно дерево `src/` в ARCHITECTURE.md (`shared/{ui,styles,config,api,lib,stores,i18n,types}`, `features`, `widgets`, `layouts`, `content`, `pages`). Направление: `pages → layouts → widgets → features → shared`. Заменить `shared/utils` → `shared/lib`, `entities` → `features`/`shared`, `widgets/features` → `widgets/benefits`. Остальные документы ссылаются | ARCHITECTURE.md |
| B13 | Prettier | Единственный источник — корневой `.prettierrc`; setup.md и ADR 0009 ссылаются на него | `.prettierrc` |
| B14 | Шрифты | `public/fonts/` (URL `/fonts/...`); убрать `src/assets/fonts/` | ADR 0007 |
| B15 | Кеш медиа | Одно значение TTL: таблица в ADR 0018 совпадает с конфигом Nginx | ADR 0018 |
| B16 | `.env.example` | Добавить `PUBLIC_SITE_URL` и серверные секреты интеграции форм без префикса `PUBLIC_` (пример: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`; конкретный набор — из допроса по продукту); убрать `NODE_ENV`; пометка о `NEXT_PUBLIC_` для Next.js. Дубль в setup.md §5 заменить ссылкой | `.env.example` |
| B17 | Терминология состояний | Разделить: состояния элемента (design.md, 6 шт.), состояния формы (ADR 0008, 5 шт.), состояния данных (loading/empty/error/success). AGENTS.md ссылается | design.md, ADR 0008 |
| B18 | Имя конфига Astro | `astro.config.ts` везде; в ADR 0004 §5 убрать `astro.config.mjs` и `prettier.config.mjs` | ADR 0016 |

**Критерий блока:** поиск по `--color-primary`, `src/styles/`, `entities`, `shared/utils`, `--transition-`, `max-width: 768px`, `$z-` → 0 совпадений.

---

## Блок C. Код в ADR противоречит правилам или устарел

| # | Файл | Задача |
|---|---|---|
| C1 | ADR 0005 §9, ADR 0008 | Разрешить провайдеры библиотек (`QueryClientProvider`, `FormProvider`); запрет касается собственного `createContext`. Паттерн для Astro: общий singleton `queryClient` + провайдер в корне каждого острова |
| C2 | ADR 0005 §11 | Заменить пример `useEffect` + `fetch` на хук TanStack Query с `signal` и Zod. `AbortController` оставить для эффектов вне Query |
| C3 | ADR 0005, 0008, 0016, 0017 | Убрать `as`: `instanceof HTTPError` (по G9), расширение типа `CSSProperties` через `declare module 'react'`, типизированные ключи i18n, `instanceof HTMLElement` в observer. В ADR 0004 — явный список допустимых исключений (`getKeys`, `as const`) |
| C4 | ADR 0004 | `formatPrice` → `function`; `logger` → механизм из ADR 0014 §5; `NodeJS.Timeout` → `ReturnType<typeof setTimeout>`; группа импортов без `@/entities` |
| C5 | ADR 0016 | Исправить `TranslationSchema`: из-за `as const` сейчас требует литеральных русских строк в `en.ts`, нужен тип с расширением до `string`. Ключи `t()` — типизированные. Интерполяция без `reduce`. Согласовать `t()` и `dict={t.common}` |
| C6 | ADR 0017, 0003 | `!important` в reduced-motion и анимацию `grid-template-rows` внести в явные исключения ADR 0003. Запретить `data-reveal` на элементах первого экрана (LCP) |
| C7 | ADR 0015 | Фоллбэк `setTimeout` для `requestIdleCallback` (нет в Safari); CookieConsent — vanilla-скрипт в `.astro` без React-острова; хранение через безопасную обёртку ADR 0004; типизация `window.gtag`; убрать лишний дженерик и обещание «TBT = 0 мс». Модель согласия по Q1: баннер уведомляет, загрузка счётчиков от него не зависит |
| C8 | ADR 0008 | Один слой ретраев: ретраи в `ky`, в `QueryClient` — `retry: false` (сейчас до 9 запросов). Убрать инвалидацию `productKeys` в отправке заявки. Одно имя honeypot-поля в ADR 0002 и 0008. Серверная проверка honeypot и time-trap. Синтаксис Zod — по G1 |
| C9 | design.md | Disabled без `pointer-events: none` (иначе нет `cursor: not-allowed`): атрибут `disabled` / `aria-disabled`. Исправить «5 состояний» при перечне из 6. `word-break: break-word` → `overflow-wrap: anywhere`. Примеры отступов — токенами |
| C10 | ADR 0014 | Ошибки `pageerror` и `console` собирать в массив и проверять в конце теста. DoD запускает `test` и `test:e2e`. Скрипт `validate:html`. Критерий W3C — «0 ошибок» везде |
| C11 | `docs/pages/page-template.md` | События `cta_click` и `catalog_filter` добавить в union `AnalyticsEvent` (ADR 0015) или заменить существующими. Открытие модалки из Hero — через лёгкий `<script>` по ADR 0004 §10 без противоречия с «0 КБ JS» |
| C12 | `docs/seo.md` | Примеры title и description уложить в 50–60 и 140–160 символов (проверить подсчётом). `<img>` всегда с `alt=""`, `aria-hidden` — только для inline SVG. Один бренд в примерах |
| C13 | ADR 0002 | Перенумеровать разделы после §14; объединить два описания Skip Link; закрытие по фону — `event.target === dialog` (сейчас клавиатурный клик с координатами 0,0 закрывает диалог); фокус — токеном; `.visually-hidden` — через `clip-path` |
| C14 | ADR 0006 | Zustand `persist` не синхронизирует вкладки: ручной слушатель `storage` + `rehydrate()`. В уровень серверного состояния добавить TanStack Query. Для Next.js — Zustand только для клиентского состояния без SSR-данных |
| C15 | ADR 0007 | Исключение для PNG/JPEG: `og:image` и фоллбэк `<img>` в `<picture>`. `twitter:image` — одинаково в ADR 0002, 0007 и seo.md |
| C16 | ADR 0013 | Убрать `SearchAction`; уточнить ограничения `FAQPage` для Google (по G5) |
| C17 | ADR 0003 | Исправить утверждение про `@layer`: CSS Modules не попадают в слой автоматически, неслойные стили сильнее слоёв. `.container` использует необъявленные `--container-max` и `--container-gutter`. Объявить или убрать `--shadow-hover`. Примеры с сырыми px → токены. `overflow-x: clip` на `html` убрать (противоречит quality.md) |
| C18 | ADR 0012 | Только `bun.lock` (без `bun.lockb`). Версия в `packageManager` и `engines` фиксируется при инициализации по G7. `cross-env` — по G8 |
| C19 | ADR 0010, tech.md | В белом списке — категория dev-инструментов + `dompurify`, `schema-dts`, `@astrojs/sitemap`, адаптеры Astro по Q2 (`@astrojs/node`, `@astrojs/cloudflare`, `@astrojs/vercel`, `@astrojs/netlify`) и `eslint-plugin-perfectionist` (Q7). Tailwind — в таблицу чёрного списка. В tech.md: альтернатива Axios — `ky`, Tailwind запрещён безусловно, убрать «тяжёлый рантайм» у Tailwind, проверить TheSVG.org (G11) |
| C20 | setup.md (ESLint, Stylelint, tsconfig) | Закрепить правила ADR 0004/0005 в линтере: `consistent-type-imports`, `no-non-null-assertion`, запрет `enum` (`no-restricted-syntax`), `no-floating-promises` (type-checked). Порядок импортов (Q7): `eslint-plugin-perfectionist`, правило `sort-imports` с группами ADR 0004 §8 (внешние → `@/` → относительные, пустая строка между группами), исправление через `eslint --fix`. Stylelint: `unit-disallowed-list: ["rem", "em"]`. tsconfig Astro — `extends` из `astro/tsconfigs`, настройки JSX, без избыточных флагов; `baseUrl` — по G2. Flat config — по G3 |
| C21 | ADR 0001 | Next.js 16: `revalidateTag(tag, 'max')`, `updateTag` в Server Actions. Astro 6: `src/content.config.ts`, `loader: glob()`, без `type: 'content'`, `z` из `astro/zod`, даты через `z.coerce.date()`. Обновить дерево в setup.md |

---

## Блок D. Юридический контур

| # | Задача |
|---|---|
| D1 | content.md и ADR 0002: убрать формулу «Нажимая кнопку, вы соглашаетесь». Оставить отдельный непроставленный чекбокс согласия со ссылкой на документ согласия (152-ФЗ в редакции с 01.09.2025: согласие оформляется отдельно от иных документов). Схема Zod — обязательный `true` |
| D2 | Создать `docs/pages/personal-data-consent.md` по формату page-template: URL, состав, реквизиты, чеклист |
| D3 | privacy-policy.md: раздел cookie описывает модель уведомления (Q1): какие cookie ставятся при входе на сайт и как отключить их в браузере |
| D4 | ADR 0015: явно зафиксировать модель Q1 — счётчики загружаются отложенно (первое взаимодействие или idle) без ожидания согласия, баннер информационный с кнопкой «Понятно» |

Источник: [consultant.ru — ст. 9 152-ФЗ](https://www.consultant.ru/document/cons_doc_LAW_508287/1500545af6526db1dfa11aba44d4df42208a0b05/).

---

## Блок E. Структура, дубли, корневые файлы

| # | Задача |
|---|---|
| E1 | Убрать захардкоженное «18» во всех файлах |
| E2 | Реестр ADR — только в ARCHITECTURE.md; README, STATE и TEMPLATE_STRUCTURE ссылаются |
| E3 | Пороги Lighthouse и Core Web Vitals — только в quality.md; ADR 0014, USAGE и page-template ссылаются. Убрать «Lighthouse 100» из README, USAGE, tech.md и ADR 0001. В quality.md заменить TBT в списке Core Web Vitals на INP ≤ 200 мс (TBT оставить лабораторной метрикой) |
| E4 | Эталонные конфиги — только в setup.md; ADR 0009 описывает правила и ссылается |
| E5 | STATE.md и docs/tasks.md превратить в пустые заготовки проекта; история шаблона остаётся в git log |
| E6 | Разделы PRODUCT.md привести в соответствие с ветками PROMPT.md |
| E7 | PROMPT.md: добавить ветки «хостинг и деплой (CDN / VPS)», «CMS», «источник дизайна (Figma / без макета)», «домен» |
| E8 | AGENTS.md: в источники истины добавить skills.md и setup.md; пункт «Использованные скилы» — в список «Итоговый отчёт» |
| E9 | README: список docs дополнить setup.md и skills.md; дерево `src/` — ссылкой на ARCHITECTURE.md |
| E10 | TEMPLATE_STRUCTURE.md: добавить конфиги (`.gitignore`, `.gitattributes`, `.editorconfig`, `.prettierrc`) и `personal-data-consent.md`; убрать счётчики |

---

## Блок F. Скилы и дизайн-вкус

| # | Задача |
|---|---|
| F1 | skills.md, таблица совместимости. `$impeccable`: не запускать `init` и `document`, PRODUCT.md — только через PROMPT.md, единственный источник дизайна — `docs/design.md`, создавать `DESIGN.md` запрещено (Q5); `.impeccable/` в `.gitignore` (A4). `$code-review`: спецификация = `docs/pages/*.md` + PRODUCT.md + `docs/tasks.md`, стандарты = AGENTS.md + `docs/decisions/` + design.md. `$tdd` и `$diagnosing-bugs`: `CONTEXT.md` → ARCHITECTURE.md + PRODUCT.md, ADR → `docs/decisions/`. `$request-refactor-plan`: план в `docs/tasks.md` вместо GitHub issue. `$vercel-react-best-practices`: SWR → TanStack Query, правила только для Next.js не применяются в Astro. `$agent-browser`: глобальная установка вне проекта |
| F2 | skills.md: условия запуска каждого Core-скила вместо «последовательно»; «двухфакторное» → «по двум осям»; в список приоритетов добавить SWR и `DESIGN.md` |
| F3 | Pre-commit хуки не используются (Q4), `setup-pre-commit` в skills.md не добавляется. ADR 0011 §2: правило «`bun run check` проходит для каждого коммита» переформулировать — агент запускает `check` перед коммитом (AGENTS.md), автоматически правило проверяет Quality Gate в CI (ADR 0018 §4) |
| F4 | design.md: раздел «Анти-шаблоны» (8–10 проверяемых запретов): фиолетовые и синие градиенты по умолчанию, декоративный glassmorphism, три одинаковые карточки как универсальное решение, эмодзи вместо иконок, выдуманные цифры и отзывы, тени и скругления на всём, центрирование всего текста, `#3b82f6` как цвет бренда. Отдельный taste-skill не добавляется |

---

## Блок G. Проверка внешних фактов (нужен доступ в сеть)

Выполняется до блоков B–F. По каждому пункту — ссылка на первоисточник.

| # | Что проверить | Влияет на |
|---|---|---|
| G1 | Zod 4: `z.email()`, параметр `error` вместо `errorMap`, рекомендуемый импорт | C8, D1 |
| G2 | TypeScript 6: статус `baseUrl` | C20, ADR 0001 §3 |
| G3 | typescript-eslint: `tseslint.config` против `defineConfig` из `eslint/config`; flat config `eslint-plugin-react-hooks` | C20 |
| G4 | React Compiler: статус и способ подключения в Next.js 16 (только для примечания в ADR 0005 §8) | Q6 |
| G5 | Google: ограничения FAQ rich results, отмена sitelinks search box | C16 |
| G6 | Nginx: синтаксис `http2 on;` | A5 |
| G7 | Актуальные версии Bun и Node.js LTS | C18, README |
| G8 | Переменные окружения в скриптах `bun run` на Windows без `cross-env` | A3, C18 |
| G9 | Актуальный мажор `ky`: `prefixUrl`, экспорт класса `HTTPError`, сигнатуры хуков | C3, C8 |
| G10 | Astro 6: встроенные Fonts API и CSP (только зафиксировать как вариант) | «Замечено, но не входит» |
| G11 | Существование и лицензия TheSVG.org | C19 |
| G12 | `eslint-plugin-perfectionist`: flat config, совместимость с `eslint-plugin-astro` и парсером `.astro`, настройка групп | C20 |

Уже подтверждено: Next.js 16 `revalidateTag` ([nextjs.org](https://nextjs.org/docs/messages/revalidate-tag-single-arg)), удаление старого API коллекций в Astro 6 ([docs.astro.build](https://v6.docs.astro.build/en/reference/errors/legacy-content-config-error/)), отдельное согласие по 152-ФЗ.

---

## Порядок выполнения и приёмка

1. A1, A4 — сразу, от вопросов не зависят.
2. G — проверка фактов.
3. B → C → D → E → F — по ответам на вопросы.
4. A2, A3, A5, A6 — после B и C, так как конфиги зависят от решений.
5. Финальная проверка.

**Общие критерии приёмки:**

- поиск остатков из блоков A и B → 0 совпадений;
- все JSON-блоки в документах парсятся;
- проверка относительных ссылок → 0 битых;
- каждое правило имеет один источник, остальные места ссылаются на него;
- вопросы Q1–Q7 закрыты, непроверенные факты помечены;
- `REFACTOR_TZ.md` удалён после закрытия пунктов.

---

## Открытые вопросы

| # | Вопрос | Варианты | Рекомендация |
|---|---|---|---|
| Q1 | Когда загружать аналитику относительно согласия на cookie | 1) Только после «Принять», есть «Отклонить» (opt-in). 2) Сразу, баннер только уведомляет. 3) Режим выбирается на допросе по продукту | **Решено: 2** — счётчики грузятся без ожидания согласия, баннер уведомляет |
| Q2 | Где обрабатывать формы в статическом Astro | 1) Эндпоинт Astro `src/pages/api/contact.ts` с `prerender = false` + адаптер под хостинг. 2) Отдельная serverless-функция. 3) Выбор на допросе по продукту | **Решено: 1** — эндпоинт Astro + адаптер; конкретный адаптер выбирается на допросе по хостингу |
| Q3 | Брейкпоинты и контейнер | 1) 576/768/1024/1280/1440, контейнер 1280. 2) 576/768/992/1200/1440, контейнер 1200 | **Решено: 1** — 576/768/1024/1280/1440, контейнер 1280, нижняя граница 360px |
| Q4 | Pre-commit хуки | 1) Husky + lint-staged через `setup-pre-commit` с переопределениями. 2) Lefthook. 3) Без хуков, только CI | **Решено: 3** — хуков нет, автоматическая проверка только в CI |
| Q5 | `DESIGN.md` для `$impeccable` | 1) Запретить создание, источник — `docs/design.md`. 2) Корневой `DESIGN.md` как короткий указатель на `docs/design.md` | **Решено: 1** — `DESIGN.md` не создаётся, источник дизайна — `docs/design.md` |
| Q6 | React Compiler | 1) Не включать, ручные правила мемоизации ADR 0005 §8. 2) Включить и упростить §8. 3) Включить только в Next.js | **Решено: 1** — компилятор не включается ни в одном профиле; в ADR 0005 §8 добавить строку: в отдельном Next.js-проекте с тяжёлым интерактивом включается флагом по решению пользователя |
| Q7 | Контроль порядка импортов (ADR 0004 §8) | 1) `eslint-plugin-perfectionist`. 2) Без плагина, контроль на ревью. 3) `@ianvs/prettier-plugin-sort-imports` | **Решено: 1** — `perfectionist/sort-imports` в ESLint, автоисправление через `--fix` |

---

## Замечено, но не входит без решения

- CSP, HSTS и `Permissions-Policy` в заголовках безопасности.
- Встроенные Fonts API и CSP в Astro 6 (G10).
- Автоматический контроль бюджета бандла (сейчас это только предупреждение Vite).
- Шаблон CI-воркфлоу GitHub Actions в репозитории. После Q4 это единственная автоматическая проверка `check`; сейчас ADR 0018 §4 описывает пайплайн только текстом.
