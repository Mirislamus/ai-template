# ADR 0003: Стандарты стилизации (SCSS Modules и дизайн-токены)

## Контекст и цели

Стилизация проекта обязана быть модульной, предсказуемой, производительной и удобной для поддержки.
Цели стандарта:
1. **Изоляция:** исключить глобальные конфликты классов и перебивания селекторов (CSS Modules).
2. **Предсказуемость единиц:** строгий отказ от `rem` и `em` в пользу точного `px`-позиционирования и современных адаптивных единиц (`vw`, `vh`, `dvh`, `%`).
3. **Mobile-First адаптивность:** базовые стили пишутся для мобильных устройств, а расширение идет вверх через современный Range Syntax (`@media (width >= ...)`).
4. **Устранение тач-артефактов:** предотвращение залипания псевдокласса `:hover` на смартфонах.
5. **Централизация дизайн-токенов:** единый источник правды для цветов, отступов, радиусов и теней через CSS Custom Properties.

---

## Архитектура файлов стилей (`src/styles/`)

Глобальная база стилей состоит строго из трех файлов без лишних подпапок:

```text
src/styles/
├── _tokens.scss  # Все дизайн-токены: переменные :root (цвета, шрифты, радиусы, отступы, z-index)
├── _tools.scss   # Служебные миксины и функции (hover, reduced-motion, focus-ring, сбросы)
└── global.scss   # Сброс стилей (CSS Reset), стили html/body, типографика заголовков, сетка .container
```

Локальные стили компонентов хранятся строго рядом с компонентом в файле `[Component].module.scss`.

---

## Принятые стандарты и правила

### 1. Адаптивность: Mobile-First и Range Syntax

Стили пишутся по принципу прогрессивного улучшения:
- **Базовые стили компонента** создаются для мобильных экранов (до 576px).
- Расширение для больших экранов оформляется через нативный **Range Syntax**:

```scss
// Базовые стили для мобильных (от 0px)
.grid {
  display: flex;
  flex-direction: column;
  gap: 16px;

  // Планшеты (от 768px)
  @media (width >= 768px) {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 24px;
  }

  // Десктоп (от 1024px)
  @media (width >= 1024px) {
    grid-template-columns: repeat(3, 1fr);
  }

  // Широкий экран (от 1280px)
  @media (width >= 1280px) {
    grid-template-columns: repeat(4, 1fr);
    gap: 32px;
  }
}
```

#### Стандартная шкала контрольных точек (Breakpoints):
- `576px` — большие смартфоны (горизонтальная ориентация);
- `768px` — планшеты (появление колоночной сетки);
- `1024px` — ноутбуки и небольшие экраны (десктопная шапка и меню);
- `1280px` — стандартный десктоп (максимальная ширина контентного контейнера);
- `1440px` — широкие мониторы (опционально).

---

### 2. Единицы измерения: px и адаптивные единицы

- **`em` и `rem` категорически запрещены:** исключает накопление ошибок масштабирования шрифтов и непредсказуемые скачки размеров.
- **`px`:** основной стандарт для размеров шрифтов, отступов (padding, margin, gap), границ (border), радиусов скругления (border-radius).
- **Адаптивные единицы:**
  - `%` — для относительной ширины контейнеров и колонок.
  - `vw` / `vh` — для привязки к размеру экрана.
  - `dvh` (Dynamic Viewport Height) и `svh` — для высоты полноэкранных блоков на мобильных браузерах (учитывают появление/скрытие адресной строки).
  - `cqw` / `cqh` — для Container Queries при изоляции карточек.

---

### 3. Именование классов в SCSS Modules

- Имя файла уже изолирует компонент. **БЭМ-префиксы внутри модулей запрещены.**
- Используются короткие, читаемые, плоские имена классов:
```scss
// Hero.module.scss
.root {
  padding-block: 60px;
}

.title {
  font-size: 32px;
  font-weight: 700;
}

.button {
  display: inline-flex;

  &.active {
    background-color: var(--color-primary);
  }
}
```
В JSX: `className={s.title}`, `className={clsx(s.button, isActive && s.active)}`.

---

### 4. Дизайн-токены: CSS Custom Properties

Все переменные оформляются через нативные CSS Custom Properties в `src/styles/_tokens.scss`:

- **Цвета (HEX / RGB):** объявляются в формате HEX или RGB для прямого копирования из Figma.
- **Прозрачность:** формируется на лету через современную нативную функцию `color-mix()`:
  ```scss
  background-color: color-mix(in srgb, var(--color-primary) 12%, transparent);
  ```
- **Sass-переменные (`$var`):** разрешены только для build-time вычислений (значения в медиазапросах, если требуется).

Пример токенов (`_tokens.scss`):
```scss
:root {
  /* Цветовая палитра (HEX / RGB) */
  --color-page: #ffffff;
  --color-surface: #f8fafc;
  --color-surface-raised: #ffffff;
  --color-text: #0f172a;
  --color-text-muted: #64748b;
  --color-border: #e2e8f0;
  --color-primary: #2563eb;
  --color-primary-hover: #1d4ed8;

  /* Радиусы */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 16px;
  --radius-full: 9999px;

  /* Отступы сетки и контейнер */
  --container-max-width: 1280px;
  --container-padding: 16px;

  /* Переходы и анимации */
  --transition-fast: 150ms ease;
  --transition-base: 250ms ease;
}
```

---

### 5. Обязательный миксин для :hover на смартфонах

Для исключения «залипания» hover-эффектов при тапе пальцем все наведения оборачиваются в миксин `@include hover`:

```scss
// src/styles/_tools.scss
@mixin hover {
  @media (hover: hover) and (pointer: fine) {
    @content;
  }
}
```

Применение в компонентах:
```scss
.link {
  color: var(--color-text);
  transition: color var(--transition-fast);

  @include hover {
    &:hover {
      color: var(--color-primary);
    }
  }

  &:active {
    color: var(--color-primary-hover);
  }
}
```

---

### 6. Уважение к доступности: prefers-reduced-motion

Все анимации, плавные переходы и скроллы обязаны отключаться для пользователей, чувствительных к движению:

```scss
// src/styles/_tools.scss
@mixin reduced-motion {
  @media (prefers-reduced-motion: reduce) {
    @content;
  }
}
```

---

### 7. Запрет !important

Использование `!important` категорически **запрещено**, кроме:
1. Утилитного класса `.visually-hidden` (для гарантии скрытия от экрана).
2. Вынужденного перебивания жестких инлайн-стилей сторонних виджетов (карты, плееры).

---

### 8. Каскадные слои CSS (@layer)

Для полного контроля специфичности селекторов и предотвращения конфликтов глобальные стили в `src/styles/global.scss` организуются через директиву `@layer`:

```scss
@layer reset, base, components, utilities;

@layer reset {
  /* Нормализация, box-sizing: border-box, сброс отступов у списков/заголовков */
  *, *::before, *::after {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }
}

@layer base {
  /* Стили html, body, типографика h1-h6, оформление ссылок */
  html {
    min-width: 320px;
    background-color: var(--color-page);
    color: var(--color-text);
    overflow-x: clip; /* Защита от горизонтального скролла без поломки sticky */
  }
}

@layer components {
  /* Место для стилей компонентов (CSS Modules автоматически попадают сюда) */
}

@layer utilities {
  /* Сервисные утилиты с наивысшим приоритетом: .container, .visually-hidden, .skip-link */
  .container {
    width: min(var(--container-max), calc(100% - (var(--container-gutter) * 2)));
    margin-inline: auto;
  }
}
```

---

### 9. Порядок сортировки CSS-свойств (Концентрический: Снаружи-Внутрь)

Все свойства внутри селектора записываются строго по группам от внешнего позиционирования к внутреннему содержанию:

1. **Позиционирование:** `position`, `top`, `right`, `bottom`, `left`, `z-index`, `inset`.
2. **Дисплей и геометрия коробки:** `display`, `flex-direction`, `justify-content`, `align-items`, `grid-template-*`, `gap`, `width`, `min-width`, `max-width`, `height`, `padding`, `margin`, `overflow`.
3. **Визуальное оформление:** `background`, `border`, `border-radius`, `box-shadow`, `opacity`.
4. **Типографика:** `font-family`, `font-size`, `font-weight`, `line-height`, `color`, `text-align`, `text-decoration`.
5. **Анимации и интерактив:** `transition`, `transform`, `cursor`, `pointer-events`, `user-select`.
6. **Вложенные псевдоклассы и медиазапросы:** `&:hover`, `&:focus-visible`, `&.active`, `@media (width >= ...)`.

Пример эталонного оформления правила:
```scss
.card {
  /* 1. Позиционирование */
  position: relative;
  z-index: 1;

  /* 2. Коробка */
  display: flex;
  flex-direction: column;
  gap: 16px;
  width: 100%;
  padding: 24px;

  /* 3. Визуал */
  background-color: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);

  /* 4. Типографика */
  color: var(--color-text);

  /* 5. Интерактив */
  transition: border-color var(--transition-fast), box-shadow var(--transition-fast);

  /* 6. Состояния и адаптивность */
  @include hover {
    &:hover {
      border-color: var(--color-primary);
      box-shadow: var(--shadow-hover);
    }
  }

  @media (width >= 768px) {
    padding: 32px;
  }
}
```

---

### 10. Стандарты типографики и локальных шрифтов

- **Только локальные шрифты:** шрифты скачиваются в репозиторий (`src/assets/fonts/`), использование сторонних CDN (Google Fonts) через внешний `<link>` или CSS `@import` запрещено для защиты от задержек сети и соблюдения приватности.
- **Формат .woff2:** используется исключительно компактный формат `woff2`.
- **Обязательный font-display: swap:** текст отображается мгновенно системным шрифтом до окончания загрузки кастомного (устранение FOIT — Flash of Invisible Text).
- **Ограничение начертаний:** подключаются строго используемые веса (например: Regular 400 и SemiBold 600).

Пример подключения (`_tokens.scss` или отдельный файл шрифтов):
```scss
@font-face {
  font-family: 'Inter';
  font-weight: 400;
  font-style: normal;
  font-display: swap;
  src: url('/fonts/inter-regular.woff2') format('woff2');
}

@font-face {
  font-family: 'Inter';
  font-weight: 600;
  font-style: normal;
  font-display: swap;
  src: url('/fonts/inter-semibold.woff2') format('woff2');
}
```

---

### 11. Флюидная адаптивная типографика (clamp)

Для исключения ручного переписывания размеров шрифтов в десятках медиазапросов крупные заголовки и лид-абзацы настраиваются через нативную функцию `clamp(min, preferred, max)` в токенах `_tokens.scss`:

- **Минимальный порог (`px`):** гарантирует читаемость и предотвращает сплющивание заголовка на узких экранах (320px–375px).
- **Динамический множитель (`vw`):** обеспечивает плавное масштабирование пропорционально экрану без резких скачков.
- **Максимальный порог (`px`):** ограничивает рост шрифта на больших мониторах.
- **Базовый текст (`body`):** фиксируется на стабильных `16px` (для форм и абзацев) ради идеального чтения и предотвращения масштабирования поля ввода на смартфонах (Safari зумит инпуты меньше 16px).

Пример эталонной шкалы типографики (`_tokens.scss`):
```scss
:root {
  /* Основной и акцентный шрифты */
  --font-sans: 'Inter', system-ui, -apple-system, sans-serif;

  /* Флюидные заголовки */
  --font-display: clamp(40px, 4vw, 64px);
  --font-h1: clamp(32px, 3.4vw, 52px);
  --font-h2: clamp(24px, 2.5vw, 38px);
  --font-h3: clamp(20px, 1.8vw, 26px);
  --font-h4: 20px;

  /* Текстовые стили */
  --font-lead: clamp(16px, 1.2vw, 18px);
  --font-body: 16px;
  --font-small: 14px;
  --font-caption: 12px;

  /* Интерлиньяж (Line-Height) */
  --lh-tight: 1.15;   /* Для крупных заголовков h1-h2 */
  --lh-snug: 1.3;    /* Для подзаголовков h3-h4 */
  --lh-normal: 1.5;  /* Для основного чтения */
}
```

---

### 12. Правило выноса стилей в миксины (_tools.scss)

Любой повторяющийся состав CSS-деклараций (от 3–4 строк), встречающийся в двух или более компонентах, обязан выноситься в миксин в `src/styles/_tools.scss`.

Обязательный базовый пул переиспользуемых миксинов:

```scss
// 1. Доступное кольцо фокуса клавиатуры
@mixin focus-ring {
  outline: 2px solid var(--color-primary);
  outline-offset: 2px;
}

// 2. Обрезка текста с многоточием на N строк
@mixin line-clamp($lines: 1) {
  display: -webkit-box;
  -webkit-line-clamp: $lines;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

// 3. Сброс дефолтных стилей кнопки
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

// 4. Тонкий кастомный скроллбар
@mixin custom-scrollbar {
  scrollbar-width: thin;
  scrollbar-color: var(--color-border) transparent;

  &::-webkit-scrollbar {
    width: 6px;
    height: 6px;
  }

  &::-webkit-scrollbar-track {
    background: transparent;
  }

  &::-webkit-scrollbar-thumb {
    background-color: var(--color-border);
    border-radius: var(--radius-full);
  }
}
```

---

### 13. Системная иерархия Z-Index

Запрещен прямой хардкод чисел `z-index` в компонентах. Значения задаются строго через токены в `_tokens.scss`:

```scss
:root {
  --z-negative: -1;
  --z-normal: 1;
  --z-dropdown: 20;
  --z-sticky: 30;         /* Липкая шапка и фиксированные плашки */
  --z-modal-backdrop: 40; /* Подложка модального окна */
  --z-modal: 50;          /* Модальные окна */
  --z-toast: 100;         /* Всплывающие уведомления (Sonner) */
  --z-tooltip: 110;       /* Тултипы */
}
```

---

### 14. Шкала радиусов и теней

Вместо произвольных значений используются системные токены:

```scss
:root {
  /* Радиусы скругления */
  --radius-sm: 4px;       /* Чекбоксы, бейджи, теги */
  --radius-md: 8px;       /* Кнопки, поля ввода, селекты */
  --radius-lg: 16px;      /* Карточки, выпадающие списки */
  --radius-xl: 24px;      /* Модальные окна, крупные баннеры */
  --radius-full: 9999px;  /* Круглые кнопки, пилюли */

  /* Тени (мягкие многослойные тени) */
  --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 12px 0 rgba(0, 0, 0, 0.08);
  --shadow-lg: 0 12px 32px 0 rgba(0, 0, 0, 0.12);
  --shadow-overlay: 0 20px 48px 0 rgba(0, 0, 0, 0.18);
}
```

---

### 15. Производительность анимаций и переходов

Для исключения лагов и просадки FPS на мобильных устройствах действуют строгие правила:

1. **Запрет `transition: all`:** анимируются только явно перечисленные свойства.
2. **Запрет анимации геометрии:** запрещено анимировать `width`, `height`, `top`, `left`, `margin`, `padding` (вызывают reflow). Изменение положения и масштаба делается строго через `transform: translate(...) / scale(...)`.
3. **Безопасный список для анимаций:** `transform`, `opacity`, `color`, `background-color`, `border-color`, `box-shadow`.
4. **Единые тайминги и функция плавности:**
```scss
:root {
  --motion-fast: 150ms;                               /* Ховеры, активные состояния кнопок */
  --motion-base: 250ms;                               /* Дропдауны, табы, спойлеры */
  --motion-slow: 350ms;                               /* Модальные окна, крупные шторки */
  --ease-out: cubic-bezier(0.22, 1, 0.36, 1);         /* Естественное физическое затухание */
}
```

