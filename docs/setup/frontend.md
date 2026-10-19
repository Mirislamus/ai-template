# Развертывание фронтенда (docs/setup/frontend.md)

Инициализация фронтенда (Astro или Next.js), эталонные конфиги инструментов качества, фреймворка и базовых стилей. Сначала выполняется [common.md](common.md): структура репозитория, `package.json`, Prettier и переменные окружения. ADR описывают правила и ссылаются сюда.

Конфиги ниже проверяются командой `bun run check` сразу после копирования.

---

## 1. Инициализация проекта через Bun

Перед стартом изучите [PRODUCT.md](../../PRODUCT.md) и выберите профиль согласно [ADR 0001](../decisions/frontend/0001-project-architecture.md). Файлы шаблона в корне (`.gitignore`, `.gitattributes`, `.editorconfig`, `.prettierrc`, `.env.example`) уже на месте.

Генераторы `create-astro` и `create-next-app` создают проект только в пустой папке, а в корне шаблона уже лежит документация. Поэтому проект без бэкенда создается во временной папке с именем проекта (оно попадает в поле `name` файла `package.json`, в примерах — `my-site`) и переносится в корень без `README.md` и `.gitignore`: у шаблона свои. В проекте с бэкендом генератор запускается прямо в пустом каталоге `apps/web` ([common.md](common.md) §1) с именем `.` вместо `my-site`: перенос не нужен, `README.md` и `.gitignore` генератора удаляются. Флаги ниже отвечают на все вопросы генераторов и отключают их собственный `AGENTS.md`: правила агентов задает корневой [AGENTS.md](../../AGENTS.md).

### Профиль А: Astro (лендинги, каталоги, контентные сайты — по умолчанию)
```bash
# 1. Создание проекта на актуальной версии Astro во временной папке
bun create astro@latest my-site --template minimal --no-install --no-git --no-ai --skip-houston

# 2. Перенос в корень (README.md и .gitignore у шаблона свои) и установка
rm my-site/README.md my-site/.gitignore
cp -r my-site/. .
rm -rf my-site
bun install

# 3. Интеграции Astro
bun add @astrojs/react @astrojs/sitemap

# 4. Базовые пакеты
bun add react react-dom clsx

# 5. Инструменты разработки и качества
bun add -d typescript@^6 @astrojs/check @types/react @types/react-dom sass prettier prettier-plugin-astro eslint @eslint/js typescript-eslint eslint-plugin-astro eslint-plugin-react-hooks eslint-plugin-perfectionist eslint-config-prettier stylelint stylelint-config-standard-scss stylelint-order vitest happy-dom @playwright/test html-validate rollup-plugin-visualizer schema-dts

# 6. Браузеры Playwright: chromium — профиль desktop, webkit — профиль mobile (§2.5)
bunx playwright install chromium webkit

# 7. Адаптер под хостинг (нужен эндпоинтам /api/*, ADR 0001 §7; выбирается на допросе по продукту)
bunx astro add node --yes        # VPS с Nginx и PM2
# либо адаптер платформы: bunx astro add cloudflare --yes (vercel, netlify)
```

- Сгенерированные `tsconfig.json` и `astro.config.mjs` заменяются эталонными (§2.1, §3.1). Файл `astro.config.mjs` удаляется: Astro ищет его раньше `astro.config.ts` и иначе не прочитает эталонный конфиг.
- TypeScript ставится последней версией 6.x: `typescript-eslint` и `@astrojs/check` не поддерживают TypeScript 7 (peerDependencies, [ADR 0010](../decisions/common/0010-third-party-libraries-policy.md) §1).

### Профиль Б: Next.js (веб-сервисы, личные кабинеты, панели управления)
```bash
# 1. Создание проекта на актуальной версии Next.js App Router во временной папке
bun create next-app@latest my-site --typescript --eslint --app --src-dir --no-tailwind --import-alias "@/*" --use-bun --no-react-compiler --no-cache-components --no-agents-md --no-agent-feedback --skip-install --disable-git

# 2. Перенос в корень (README.md и .gitignore у шаблона свои) и установка
rm my-site/README.md my-site/.gitignore
cp -r my-site/. .
rm -rf my-site
bun install

# 3. Базовые пакеты
bun add clsx

# 4. Инструменты разработки и качества
bun add -d sass prettier stylelint stylelint-config-standard-scss stylelint-order eslint-plugin-perfectionist eslint-config-prettier @eslint/js typescript-eslint vitest happy-dom @playwright/test html-validate @next/bundle-analyzer schema-dts

# 5. Браузеры Playwright: chromium — профиль desktop, webkit — профиль mobile (§2.5)
bunx playwright install chromium webkit

# 6. Для sitemap используется app/sitemap.ts (ADR 0013), отдельный пакет не нужен
```

- `--no-react-compiler` — React Compiler не включается ([ADR 0005](../decisions/frontend/0005-react-standards.md) §8).
- `--no-cache-components` — рендеринг и ревалидация описаны в [ADR 0001](../decisions/frontend/0001-project-architecture.md) §5 для модели без Cache Components (`revalidate`, `dynamic`).

### Пакеты по функциям продукта

Базовые команды выше ставят только то, на что опираются конфиги и скрипты §2. Остальные пакеты белого списка ([ADR 0010](../decisions/common/0010-third-party-libraries-policy.md) §3) ставятся, только когда функция есть в [PRODUCT.md](../../PRODUCT.md) или в карточке текущей задачи, в той же задаче, где появляется их первый импорт.

| Функция | Пакеты |
|---|---|
| Первый React-остров (Astro) или клиентский компонент (Next.js) | `react-error-boundary` ([ADR 0005](../decisions/frontend/0005-react-standards.md)) |
| UI-иконки | `lucide-react` ([ADR 0007](../decisions/frontend/0007-assets-and-media.md)) |
| Формы с валидацией | `react-hook-form @hookform/resolvers zod` |
| Маски ввода (телефон, суммы) | `imask` |
| Запросы к API из клиента | `ky @tanstack/react-query` |
| Общий стейт между островами (Astro) | `nanostores @nanostores/react`; хранение в `localStorage` — `@nanostores/persistent` |
| Глобальный клиентский стейт (Next.js) | `zustand` |
| Всплывающие уведомления | `sonner` |
| Карусель | `embla-carousel-react` |
| Полноэкранный просмотр с pinch-to-zoom | `photoswipe` |
| Вывод сырого HTML ([ADR 0008](../decisions/frontend/0008-forms-and-api.md) §6) | `dompurify` |

Если функция удалена из продукта, ее пакеты удаляются через `bun remove` в той же задаче.

В профиле Next.js остается сгенерированный `eslint.config.mjs` (`eslint-config-next`), в его конец добавляются блоки `rules` и `perfectionist` из §2.2. Из `.prettierrc` удаляются `plugins` и `overrides`.

---

## 2. Эталонные конфигурации инструментов качества

`packageManager`, `engines` и `.prettierrc` — [common.md](common.md) §2–3.

### 2.1. `tsconfig.json` (строгая типобезопасность по ADR 0004)

Профиль Astro:
```json
{
  "extends": "astro/tsconfigs/strict",
  "include": [".astro/types.d.ts", "**/*"],
  "exclude": ["dist"],
  "compilerOptions": {
    "jsx": "react-jsx",
    "jsxImportSource": "react",
    "noUncheckedIndexedAccess": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

Профиль Next.js: сгенерированный `tsconfig.json` (`strict: true`) дополняется `"noUncheckedIndexedAccess": true`; алиас `@/*` указывает на `./src/*` и `baseUrl` не используется.

### 2.2. `eslint.config.mjs` (Flat Config, ADR 0004, 0005, 0009)
```javascript
import eslint from '@eslint/js';
import { defineConfig, globalIgnores } from 'eslint/config';
import astro from 'eslint-plugin-astro';
import prettier from 'eslint-config-prettier';
import perfectionist from 'eslint-plugin-perfectionist';
import reactHooks from 'eslint-plugin-react-hooks';
import tseslint from 'typescript-eslint';

export default defineConfig([
  globalIgnores(['dist', '.astro', '.next', 'coverage', 'playwright-report', 'test-results']),
  eslint.configs.recommended,
  tseslint.configs.recommended,
  astro.configs.recommended,
  {
    // Правила, которым нужна информация о типах (только .ts/.tsx)
    files: ['**/*.{ts,tsx}'],
    extends: [tseslint.configs.recommendedTypeChecked],
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
    },
    rules: {
      '@typescript-eslint/no-floating-promises': 'error',
    },
  },
  {
    plugins: {
      'react-hooks': reactHooks,
      perfectionist,
    },
    rules: {
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-non-null-assertion': 'error',
      '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports', fixStyle: 'separate-type-imports' }],
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_', varsIgnorePattern: '^_' }],
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'error',
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'no-restricted-syntax': [
        'error',
        { selector: 'TSEnumDeclaration', message: 'enum запрещен (ADR 0004 §4): используйте union-типы или объект as const.' },
        {
          selector: "CallExpression[callee.name=/^(createContext|useContext)$/]",
          message: 'Собственный React Context запрещен (ADR 0005 §9): используйте Nano Stores или Zustand.',
        },
      ],
      // Порядок импортов ADR 0004 §8: внешние → алиасы @/ → относительные, пустая строка между группами
      'perfectionist/sort-imports': [
        'error',
        {
          type: 'natural',
          newlinesBetween: 1,
          internalPattern: ['^@/.*'],
          groups: [
            ['type-builtin', 'value-builtin', 'type-external', 'value-external'],
            ['type-internal', 'value-internal'],
            ['type-parent', 'type-sibling', 'type-index', 'value-parent', 'value-sibling', 'value-index'],
            'unknown',
          ],
        },
      ],
    },
  },
  prettier,
]);
```

### 2.3. `.stylelintrc.mjs` (порядок свойств и ограничения по ADR 0003 и ADR 0009)
```javascript
export default {
  extends: ['stylelint-config-standard-scss'],
  plugins: ['stylelint-order'],
  rules: {
    'selector-max-id': 0,
    'selector-no-qualifying-type': true,
    'selector-class-pattern': ['^[a-z][a-zA-Z0-9-]*$', { message: 'Плоские имена классов SCSS Modules (ADR 0003 §3)' }],
    'max-nesting-depth': 3,
    'color-named': 'never',
    'unit-disallowed-list': ['rem', 'em'],
    'declaration-property-value-allowed-list': {
      'z-index': ['/^var\\(--z-/', 'auto'],
    },
    'declaration-property-value-disallowed-list': {
      '/^transition/': ['/\\ball\\b/'],
    },
    // Группы — ADR 0009 §3; свойства вне списка идут в конце блока
    'order/properties-order': [
      [
        // 1. Позиционирование
        'position', 'inset', 'inset-block', 'inset-inline', 'top', 'right', 'bottom', 'left', 'z-index',
        // 2. Отображение и сетка
        'display', 'flex', 'flex-flow', 'flex-direction', 'flex-wrap', 'flex-grow', 'flex-shrink', 'flex-basis',
        'grid', 'grid-template', 'grid-template-columns', 'grid-template-rows', 'grid-template-areas',
        'grid-auto-flow', 'grid-auto-columns', 'grid-auto-rows', 'grid-area', 'grid-column', 'grid-row',
        'gap', 'row-gap', 'column-gap',
        'place-content', 'place-items', 'place-self', 'align-content', 'align-items', 'align-self',
        'justify-content', 'justify-items', 'justify-self', 'order',
        // 3. Блочная модель
        'width', 'min-width', 'max-width', 'height', 'min-height', 'max-height', 'aspect-ratio',
        'margin', 'margin-block', 'margin-block-start', 'margin-block-end',
        'margin-inline', 'margin-inline-start', 'margin-inline-end',
        'margin-top', 'margin-right', 'margin-bottom', 'margin-left',
        'padding', 'padding-block', 'padding-block-start', 'padding-block-end',
        'padding-inline', 'padding-inline-start', 'padding-inline-end',
        'padding-top', 'padding-right', 'padding-bottom', 'padding-left',
        'box-sizing', 'overflow', 'overflow-x', 'overflow-y',
        // 4. Типографика
        'font', 'font-family', 'font-size', 'font-weight', 'font-style', 'line-height', 'letter-spacing',
        'text-align', 'text-transform', 'text-decoration', 'text-overflow', 'text-wrap',
        'white-space', 'overflow-wrap', 'color',
        // 5. Визуальное оформление
        'background', 'background-color', 'background-image', 'background-position', 'background-size',
        'background-repeat', 'border', 'border-width', 'border-style', 'border-color',
        'border-block', 'border-inline', 'border-top', 'border-right', 'border-bottom', 'border-left',
        'border-radius', 'box-shadow', 'outline', 'outline-offset', 'opacity',
        // 6. Анимации и интерактив
        'transform', 'transition', 'animation', 'cursor', 'pointer-events', 'user-select',
      ],
      { unspecified: 'bottom' },
    ],
  },
};
```

### 2.4. Секция `scripts` в `package.json`

Профиль Astro:
```json
{
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview",
    "lint:js": "eslint . --max-warnings 0",
    "lint:style": "stylelint \"src/**/*.{css,scss}\" --max-warnings 0",
    "lint": "bun run lint:js && bun run lint:style",
    "format": "prettier --write \"src/**/*.{ts,tsx,astro,scss,json,md}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,astro,scss,json,md}\"",
    "typecheck": "astro check",
    "check": "bun run format:check && bun run lint && bun run typecheck",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:e2e": "playwright test",
    "validate:html": "html-validate \"dist/**/*.html\"",
    "build:analyze": "ANALYZE=true astro build"
  }
}
```

Профиль Next.js отличается командами `"dev": "next dev"`, `"build": "next build"`, `"start": "next start"`, `"typecheck": "tsc --noEmit"`, `"build:analyze": "ANALYZE=true next build"`; остальные команды совпадают. Переменные окружения в скриптах задаются записью `VAR=value команда`: `bun run` выполняет скрипты в Bun Shell на Windows и в системной оболочке на Linux и macOS, обе понимают такую запись, отдельный `cross-env` не нужен ([ADR 0012](../decisions/common/0012-build-and-package-manager.md) §4).

### 2.5. Конфиги тестов (ADR 0014)

`vitest.config.ts` (Astro):
```ts
import { getViteConfig } from 'astro/config';

export default getViteConfig({
  test: {
    environment: 'happy-dom',
    include: ['src/**/*.test.ts'],
  },
});
```

`playwright.config.ts`:
```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: 'tests/e2e',
  use: { baseURL: 'http://localhost:4321' },
  webServer: {
    command: 'bun run build && bun run preview --port 4321',
    url: 'http://localhost:4321',
    reuseExistingServer: true,
  },
  projects: [
    { name: 'desktop', use: { ...devices['Desktop Chrome'], viewport: { width: 1280, height: 720 } } },
    { name: 'mobile', use: { ...devices['iPhone 14'] } },
  ],
});
```

---

## 3. Конфиг фреймворка

### 3.1. `astro.config.ts` (Astro)
```ts
import node from '@astrojs/node'; // адаптер по хостингу (ADR 0001 §7)
import react from '@astrojs/react';
import sitemap from '@astrojs/sitemap';
import { defineConfig, envField } from 'astro/config';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  site: 'https://domain.uz', // заменить на боевой домен
  trailingSlash: 'never', // ADR 0013
  build: { format: 'file' }, // /page → page.html, без каталога и слэша
  adapter: node({ mode: 'standalone' }),
  integrations: [react(), sitemap()],
  // Раздел i18n включается только в мультиязычных проектах (ADR 0016 §3)
  env: {
    schema: {
      PUBLIC_SITE_URL: envField.string({ context: 'client', access: 'public', url: true }),
      PUBLIC_API_URL: envField.string({ context: 'client', access: 'public', default: '/api' }),
      PUBLIC_YM_ID: envField.string({ context: 'client', access: 'public', optional: true }),
      PUBLIC_GA_ID: envField.string({ context: 'client', access: 'public', optional: true }),
      // Секреты интеграции форм: набор определяется допросом по продукту (пример — Telegram)
      TELEGRAM_BOT_TOKEN: envField.string({ context: 'server', access: 'secret' }),
      TELEGRAM_CHAT_ID: envField.string({ context: 'server', access: 'secret' }),
    },
  },
  vite: {
    build: { chunkSizeWarningLimit: 250 }, // ADR 0012 §2
    plugins: process.env.ANALYZE ? [visualizer({ filename: 'stats.html', gzipSize: true })] : [],
  },
});
```

### 3.2. `src/content.config.ts` (Astro Content Collections)
```ts
import { glob } from 'astro/loaders';
import { z } from 'astro/zod';
import { defineCollection } from 'astro:content';

const faq = defineCollection({
  loader: glob({ pattern: '**/*.md', base: './src/content/faq' }),
  schema: z.object({
    question: z.string(),
    order: z.number().int(),
  }),
});

export const collections = { faq };
```

---

## 4. Структура папок и базовые стили

Дерево `src/` описано в [ARCHITECTURE.md](../../ARCHITECTURE.md). Глобальные стили создаются в `src/shared/styles/` тремя файлами ([ADR 0003](../decisions/frontend/0003-scss-standards.md)); имена и смысл токенов — [design.md](../design.md).

### `src/shared/styles/_tokens.scss`
```scss
@use './tools' as *;

:root {
  /* Отступы (8pt Grid) */
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-5: 20px;
  --space-6: 24px;
  --space-8: 32px;
  --space-10: 40px;
  --space-12: 48px;
  --space-16: 64px;
  --space-20: 80px;
  --space-24: 96px;
  --space-32: 128px;

  /* Цвета (семантические роли; значения заменяются под бренд проекта) */
  --color-bg-page: #fff;
  --color-bg-surface: #f8fafc;
  --color-bg-muted: #f1f5f9;
  --color-text-primary: #0f172a;
  --color-text-secondary: #475569;
  --color-text-muted: #64748b;
  --color-text-inverse: #fff;
  --color-border-subtle: #e2e8f0;
  --color-border-strong: #94a3b8;
  --color-accent: #0f766e;
  --color-accent-hover: #115e59;
  --color-status-error: #b91c1c;
  --color-status-success: #15803d;

  /* Типографика */
  --font-sans: 'Inter', system-ui, -apple-system, sans-serif;
  --font-size-h1: clamp(32px, 4vw + 16px, 56px);
  --font-size-h2: clamp(24px, 2.5vw + 14px, 40px);
  --font-size-h3: clamp(20px, 1.5vw + 14px, 28px);
  --font-size-base: 16px;
  --font-size-sm: 14px;
  --font-size-xs: 12px;
  --line-height-tight: 1.15;
  --line-height-snug: 1.3;
  --line-height-normal: 1.55;

  /* Радиусы и тени */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 16px;
  --radius-xl: 24px;
  --radius-full: 9999px;
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 5%);
  --shadow-md: 0 4px 12px 0 rgb(0 0 0 / 8%);
  --shadow-lg: 0 12px 32px 0 rgb(0 0 0 / 8%);

  /* Слои (единая шкала, ADR 0003 §13) */
  --z-base: 1;
  --z-dropdown: 100;
  --z-sticky: 200;
  --z-drawer: 300;
  --z-modal-backdrop: 400;
  --z-modal: 500;
  --z-popover: 600;
  --z-toast: 700;
  --z-tooltip: 800;

  /* Анимации */
  --motion-fast: 150ms;
  --motion-base: 250ms;
  --motion-slow: 350ms;
  --ease-out: cubic-bezier(0.22, 1, 0.36, 1);

  /* Контейнер */
  --container-max-width: 1280px;
  --container-gutter: var(--space-4);
}

@include up(md) {
  :root {
    --container-gutter: var(--space-8);
  }
}

[data-theme='dark'] {
  --color-bg-page: #0b0f19;
  --color-bg-surface: #111827;
  --color-bg-muted: #1f2937;
  --color-text-primary: #f8fafc;
  --color-text-secondary: #cbd5e1;
  --color-text-muted: #94a3b8;
  --color-border-subtle: #1e293b;
  --color-border-strong: #475569;
}
```

### `src/shared/styles/_tools.scss`
```scss
@use 'sass:map';

// Брейкпоинты (единая шкала, ADR 0003 §1). Ключ '2xl' передается в кавычках: up('2xl').
$breakpoints: (
  'sm': 576px,
  'md': 768px,
  'lg': 1024px,
  'xl': 1280px,
  '2xl': 1440px,
);

@mixin up($name) {
  @media (width >= #{map.get($breakpoints, $name)}) {
    @content;
  }
}

@mixin down($name) {
  @media (width < #{map.get($breakpoints, $name)}) {
    @content;
  }
}

@mixin hover {
  @media (hover: hover) and (pointer: fine) {
    @content;
  }
}

@mixin reduced-motion {
  @media (prefers-reduced-motion: reduce) {
    @content;
  }
}

@mixin focus-ring {
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
}

@mixin line-clamp($lines: 1) {
  display: -webkit-box;
  overflow: hidden;
  -webkit-line-clamp: $lines;
  -webkit-box-orient: vertical;
}

@mixin button-reset {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  font: inherit;
  color: inherit;
  background: none;
  border: none;
  cursor: pointer;
}
```
Миксин `custom-scrollbar` добавляется по необходимости ([ADR 0003](../decisions/frontend/0003-scss-standards.md) §12).

### `src/shared/styles/global.scss`
```scss
@use './tokens';
@layer reset, base;

@layer reset {
  *,
  *::before,
  *::after {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }
}

@layer base {
  html {
    font-family: var(--font-sans);
    font-size: var(--font-size-base);
    color: var(--color-text-primary);
    background-color: var(--color-bg-page);
    scrollbar-gutter: stable;
  }

  h1,
  h2,
  h3,
  h4 {
    line-height: var(--line-height-tight);
    text-wrap: balance;
  }

  h1 {
    font-size: var(--font-size-h1);
  }

  h2 {
    font-size: var(--font-size-h2);
  }

  h3 {
    font-size: var(--font-size-h3);
  }

  p {
    max-width: 680px;
    line-height: var(--line-height-normal);
    text-wrap: pretty;
    overflow-wrap: anywhere;
  }

  :focus-visible {
    outline: 2px solid var(--color-accent);
    outline-offset: 2px;
  }
}

.container {
  width: 100%;
  max-width: var(--container-max-width);
  margin-inline: auto;
  padding-inline: var(--container-gutter);
}

.visually-hidden {
  position: absolute !important;
  width: 1px !important;
  height: 1px !important;
  margin: -1px !important;
  padding: 0 !important;
  overflow: hidden !important;
  white-space: nowrap !important;
  border: 0 !important;
  clip-path: inset(50%) !important;
}

.skip-link {
  position: absolute;
  top: -999px;
  left: var(--space-4);
  z-index: var(--z-tooltip);
  padding: var(--space-3) var(--space-5);
  font-weight: 600;
  text-decoration: none;
  color: var(--color-text-inverse);
  background-color: var(--color-accent);
  border-radius: var(--radius-sm);

  &:focus {
    top: var(--space-4);
  }
}

@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

Шрифты подключаются через `@font-face` из `public/fonts/` ([ADR 0003](../decisions/frontend/0003-scss-standards.md) §10).
