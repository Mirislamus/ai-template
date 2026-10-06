# Развертывание фронтенда (docs/setup/frontend.md)

Инициализация фронтенда (Astro или Next.js), эталонные конфиги инструментов качества, фреймворка и базовых стилей. Сначала выполняется [common.md](common.md): структура репозитория, `package.json`, Prettier и переменные окружения. ADR описывают правила и ссылаются сюда.

Конфиги ниже проверяются командой `bun run check` сразу после копирования.

---

## 1. Инициализация проекта через Bun

Перед стартом изучите [PRODUCT.md](../../PRODUCT.md) и выберите профиль согласно [ADR 0001](../decisions/frontend/0001-project-architecture.md). Файлы шаблона в корне (`.gitignore`, `.gitattributes`, `.editorconfig`, `.prettierrc`, `.env.example`) уже на месте. В проекте с бэкендом команды выполняются в каталоге `apps/web` ([common.md](common.md) §1).

### Профиль А: Astro (лендинги, каталоги, контентные сайты — по умолчанию)
```bash
# 1. Создание проекта на актуальной версии Astro
bun create astro@latest . --template minimal --typescript strict --install --no-git

# 2. Интеграции Astro
bun add @astrojs/react @astrojs/sitemap

# 3. Одобренный белый стек (ADR 0010)
bun add react react-dom clsx ky @tanstack/react-query nanostores @nanostores/react @nanostores/persistent react-hook-form @hookform/resolvers zod imask lucide-react sonner react-error-boundary

# 4. Инструменты разработки и качества
bun add -d typescript @astrojs/check @types/react @types/react-dom sass prettier prettier-plugin-astro eslint @eslint/js typescript-eslint eslint-plugin-astro eslint-plugin-react-hooks eslint-plugin-perfectionist eslint-config-prettier stylelint stylelint-config-standard-scss stylelint-order vitest happy-dom @playwright/test html-validate rollup-plugin-visualizer schema-dts

# 5. Адаптер под хостинг (нужен эндпоинтам /api/*, ADR 0001 §7; выбирается на допросе по продукту)
bunx astro add node        # VPS с Nginx и PM2
# либо: bunx astro add cloudflare | vercel | netlify

# 6. По необходимости: вывод сырого HTML (ADR 0008 §6)
bun add dompurify
```

### Профиль Б: Next.js (веб-сервисы, личные кабинеты, панели управления)
```bash
# 1. Создание проекта на актуальной версии Next.js App Router
bun create next-app@latest . --typescript --eslint --app --src-dir --no-tailwind --import-alias "@/*" --use-bun

# 2. Одобренный белый стек
bun add clsx ky @tanstack/react-query zustand react-hook-form @hookform/resolvers zod imask lucide-react sonner react-error-boundary

# 3. Инструменты разработки и качества
bun add -d sass prettier stylelint stylelint-config-standard-scss stylelint-order eslint-plugin-perfectionist eslint-config-prettier @eslint/js typescript-eslint vitest happy-dom @playwright/test html-validate @next/bundle-analyzer schema-dts

# 4. Для sitemap используется app/sitemap.ts (ADR 0013), отдельный пакет не нужен
```

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
          newlinesBetween: 'always',
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
    'order/properties-order': [
      [
        // 1. Позиционирование
        'position', 'inset', 'top', 'right', 'bottom', 'left', 'z-index',
        // 2. Отображение и сетка
        'display', 'flex', 'flex-direction', 'flex-wrap', 'align-items', 'justify-content', 'gap',
        'grid', 'grid-template-columns', 'grid-template-rows',
        // 3. Блочная модель
        'width', 'min-width', 'max-width', 'height', 'min-height', 'max-height', 'aspect-ratio',
        'margin', 'padding', 'box-sizing', 'overflow',
        // 4. Типографика
        'font-family', 'font-size', 'font-weight', 'line-height', 'text-align', 'text-wrap', 'color',
        // 5. Визуальное оформление
        'background', 'background-color', 'border', 'border-radius', 'box-shadow', 'outline', 'opacity',
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

Профиль Next.js отличается командами `"dev": "next dev"`, `"build": "next build"`, `"start": "next start"`, `"typecheck": "tsc --noEmit"`, `"build:analyze": "ANALYZE=true next build"`; остальные команды совпадают. Переменные окружения в скриптах задаются записью `VAR=value команда`: `bun run` выполняет скрипты в кроссплатформенной оболочке Bun, отдельный `cross-env` не нужен.

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
  site: 'https://domain.ru', // заменить на боевой домен
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
import { defineCollection } from 'astro:content';
import { glob } from 'astro/loaders';
import { z } from 'astro/zod';

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
  --color-bg-page: #ffffff;
  --color-bg-surface: #f8fafc;
  --color-bg-muted: #f1f5f9;
  --color-text-primary: #0f172a;
  --color-text-secondary: #475569;
  --color-text-muted: #64748b;
  --color-text-inverse: #ffffff;
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
  -webkit-line-clamp: $lines;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

@mixin button-reset {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  border: none;
  background: none;
  font: inherit;
  color: inherit;
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
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }
}

@layer base {
  html {
    scrollbar-gutter: stable;
    background-color: var(--color-bg-page);
    color: var(--color-text-primary);
    font-family: var(--font-sans);
    font-size: var(--font-size-base);
  }

  h1,
  h2,
  h3,
  h4 {
    text-wrap: balance;
    line-height: var(--line-height-tight);
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
    text-wrap: pretty;
    line-height: var(--line-height-normal);
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
  padding: 0 !important;
  margin: -1px !important;
  overflow: hidden !important;
  clip-path: inset(50%) !important;
  white-space: nowrap !important;
  border: 0 !important;
}

.skip-link {
  position: absolute;
  top: -999px;
  left: var(--space-4);
  z-index: var(--z-tooltip);
  padding: var(--space-3) var(--space-5);
  background-color: var(--color-accent);
  color: var(--color-text-inverse);
  border-radius: var(--radius-sm);
  font-weight: 600;
  text-decoration: none;

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
