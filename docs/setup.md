Failed to write init script: open C:\Users\Windows 10\AppData\Local\Packages\ohmyposh.cli_96v55e8n804z4\LocalCache\Local\oh-my-posh\init.814522496948324317.ps1: Access is denied.
Export-Clixml: Access to the path 'C:\Users\Windows
10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_light_color.xml' is denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_color.xml' is
denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\devblackops_icon.xml' is
denied.
Export-Clixml: Access to the path 'C:\Users\Windows 10\AppData\Roaming\powershell\Community\Terminal-Icons\prefs.xml' is denied.
# Руководство по развертыванию проекта (docs/setup.md)

Инженерная пошаговая инструкция для разработчиков и AI-агентов по инициализации, настройке инструментов качества и генерации базовой структуры при старте нового проекта на базе шаблона.

---

## 1. Инициализация проекта через Bun

Перед стартом изучите [PRODUCT.md](../PRODUCT.md) и выберите профиль согласно [ADR 0001](decisions/0001-project-architecture.md):

### Профиль А: Astro (Лендинги, каталоги, контентные сайты — по умолчанию)
```bash
# 1. Создание чистого проекта на актуальной версии Astro
bun create astro@latest . --template minimal --typescript strict --install --no-git

# 2. Установка одобренного белого стека (ADR 0010)
bun add @astrojs/react react react-dom clsx ky @tanstack/react-query nanostores @nanostores/react @nanostores/persistent react-hook-form @hookform/resolvers zod imask lucide-react sonner

# 3. Установка инструментов разработки и линтинга
bun add -d sass prettier prettier-plugin-astro eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin eslint-plugin-astro eslint-plugin-react-hooks eslint-config-prettier stylelint stylelint-config-standard-scss stylelint-order vitest happy-dom rollup-plugin-visualizer cross-env
```

### Профиль Б: Next.js (Веб-сервисы, личные кабинеты, панели управления)
```bash
# 1. Создание проекта на актуальной версии Next.js App Router
bun create next-app@latest . --typescript --eslint --app --src-dir --use-bun

# 2. Установка одобренного белого стека
bun add clsx ky @tanstack/react-query zustand react-hook-form @hookform/resolvers zod imask lucide-react sonner react-error-boundary

# 3. Установка инструментов разработки
bun add -d sass prettier eslint-config-prettier stylelint stylelint-config-standard-scss stylelint-order vitest happy-dom @next/bundle-analyzer cross-env
```

---

## 2. Эталонные конфигурации инструментов качества

После инициализации создаются или заменяются конфигурационные файлы строго по принятым стандартам:

### 2.1. `tsconfig.json` (Строгая типобезопасность по ADR 0004)
```json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

### 2.2. `.prettierrc` (Единый формат форматирования по ADR 0009)
```json
{
  "printWidth": 120,
  "tabWidth": 2,
  "useTabs": false,
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "bracketSpacing": true,
  "arrowParens": "avoid",
  "endOfLine": "lf",
  "plugins": ["prettier-plugin-astro"]
}
```

### 2.3. `eslint.config.mjs` (Flat Config по ADR 0004 и ADR 0009)
```javascript
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import astroPlugin from 'eslint-plugin-astro';
import reactHooks from 'eslint-plugin-react-hooks';
import prettier from 'eslint-config-prettier';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommended,
  ...astroPlugin.configs.recommended,
  {
    plugins: {
      'react-hooks': reactHooks,
    },
    rules: {
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_', varsIgnorePattern: '^_' }],
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'error',
      'no-console': ['warn', { allow: ['warn', 'error'] }],
    },
  },
  prettier
);
```

### 2.4. `.stylelintrc.mjs` (Порядок свойств и ограничения по ADR 0003 и ADR 0009)
```javascript
export default {
  extends: ['stylelint-config-standard-scss'],
  plugins: ['stylelint-order'],
  rules: {
    'selector-max-id': 0,
    'selector-no-qualifying-type': true,
    'max-nesting-depth': 3,
    'color-named': 'never',
    'order/properties-order': [
      // 1. Позиционирование
      ['position', 'top', 'right', 'bottom', 'left', 'z-index'],
      // 2. Отображение и сетки
      ['display', 'flex', 'grid', 'flex-direction', 'align-items', 'justify-content', 'gap'],
      // 3. Блочная модель
      ['width', 'min-width', 'max-width', 'height', 'min-height', 'max-height', 'margin', 'padding', 'box-sizing'],
      // 4. Типографика
      ['font-family', 'font-size', 'line-height', 'font-weight', 'text-align', 'color'],
      // 5. Оформление
      ['background', 'background-color', 'border', 'border-radius', 'box-shadow', 'outline'],
      // 6. Анимации и трансформации
      ['opacity', 'transform', 'transition', 'animation', 'cursor', 'user-select']
    ].flat()
  }
};
```

### 2.5. Секция `scripts` в `package.json`
```json
{
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview",
    "lint:js": "eslint . --max-warnings 0",
    "lint:style": "stylelint "src/**/*.{css,scss}" --max-warnings 0",
    "lint": "bun run lint:js && bun run lint:style",
    "format": "prettier --write "src/**/*.{ts,tsx,astro,scss,json,md}"",
    "format:check": "prettier --check "src/**/*.{ts,tsx,astro,scss,json,md}"",
    "typecheck": "astro check",
    "check": "bun run format:check && bun run lint && bun run typecheck",
    "test": "vitest run",
    "test:watch": "vitest",
    "build:analyze": "cross-env ANALYZE=true astro build"
  }
}
```

---

## 3. Эталонная структура папок FSD-Lite

В папке `src/` разворачивается трехуровневая архитектура согласно [ADR 0001](decisions/0001-project-architecture.md):

```text
src/
├── shared/
│   ├── ui/               # Базовые UI-компоненты (Button, Input, Modal, Container)
│   ├── styles/           # Токены, переменные, глобальные стили
│   ├── config/           # env.ts (валидация Zod)
│   ├── api/              # client.ts (инстанс ky)
│   ├── lib/              # Утилиты аналитики, форматирования
│   └── types/            # Общие DTO и модели
├── features/             # Бизнес-модули (lead-form, theme-toggle, catalog-filter)
├── widgets/              # Секции страниц (header, footer, hero, faq)
├── layouts/              # Макеты страниц (BaseLayout.astro)
├── content/              # Astro Content Collections (config.ts)
└── pages/                # Роутинг (index.astro, 404.astro)
```

---

## 4. Эталонные SCSS-токены (`src/shared/styles/`)

### `src/shared/styles/_spacing.scss`
```scss
:root {
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
}
```

### `src/shared/styles/_z-index.scss`
```scss
:root {
  --z-base: 1;
  --z-dropdown: 100;
  --z-sticky: 200;
  --z-drawer: 300;
  --z-modal-backdrop: 400;
  --z-modal: 500;
  --z-popover: 600;
  --z-toast: 700;
  --z-tooltip: 800;
}
```

### `src/shared/styles/_typography.scss`
```scss
:root {
  --font-size-h1: clamp(32px, 4vw + 16px, 56px);
  --font-size-h2: clamp(24px, 2.5vw + 14px, 40px);
  --font-size-h3: clamp(20px, 1.5vw + 14px, 28px);

  --font-size-base: 16px;
  --font-size-sm: 14px;
  --font-size-xs: 12px;
}

h1, h2, h3, h4 {
  text-wrap: balance;
  line-height: 1.15;
}

p {
  text-wrap: pretty;
  line-height: 1.55;
  max-width: 680px;
}
```

---

## 5. Эталонный `.env.example`

В корне проекта создается файл-шаблон:

```bash
# Базовый публичный URL бэкенда/API
PUBLIC_API_URL="https://api.domain.ru"

# Идентификатор счетчика Яндекс.Метрики
PUBLIC_YM_ID="12345678"

# Режим окружения
NODE_ENV="development"
```



