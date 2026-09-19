# ADR 0016: Архитектура интернационализации и переводов (i18n)

## Контекст и цели

Многоязычность сайта напрямую влияет на охват аудитории, конверсию и региональное поисковое ранжирование.
Цели архитектурного стандарта:
1. **100% типобезопасность словарей:** использование константных TypeScript-объектов вместо сырых JSON для автокомплита и проверки отсутствующих ключей на этапе компиляции.
2. **Нулевой лишний вес в клиенте (Zero Runtime Bloat):** компиляция переводов в статический HTML на этапе сборки SSG (в Astro), проброс в интерактивные острова строго активной локали.
3. **Отказ от тяжелых библиотек:** замена `react-i18next` на легковесную типизированную утилиту `t()` и нативный `Intl.PluralRules`.
4. **Четкая маршрутизация:** использование встроенного модуля `astro:i18n` в `astro.config.ts` с бесшовными URL (`/` для основного языка, `/en/` для альтернативного).
5. **Нативная локализация чисел и дат:** форматирование через стандарты `Intl.NumberFormat` и `Intl.DateTimeFormat`.

---

## Принятые стандарты

### 1. Формат и структура словарей переводов

- **Размещение:** `src/shared/i18n/locales/` (`ru.ts`, `en.ts`, `uz.ts`).
- **Константные объекты `as const` с автовыводом схемы:**
  ```ts
  // src/shared/i18n/locales/ru.ts
  export const ru = {
    common: {
      submit: 'Отправить заявку',
      loading: 'Отправка...',
      success: 'Заявка успешно отправлена',
      error: 'Произошла ошибка, попробуйте снова',
    },
    nav: {
      services: 'Услуги',
      about: 'О компании',
      contacts: 'Контакты',
    },
  } as const;

  // Базовая схема выводится автоматически из основного языка:
  export type TranslationSchema = typeof ru;
  ```
- **Контроль целостности:**
  - Файлы альтернативных языков строго типизируются интерфейсом `TranslationSchema`:
    ```ts
    // src/shared/i18n/locales/en.ts
    import type { TranslationSchema } from './ru';

    export const en: TranslationSchema = {
      // Если пропущен ключ или допущена опечатка — билд немедленно падает
      common: { ... },
      nav: { ... },
    };
    ```
- **Изоляция размера бандла:**
  - В статическом рендере Astro страницы собираются изолированно: на странице `/services` в разметку вшит только русский текст, на `/en/services` — только английский. В браузер клиенту JS-файлы словарей не отправляются (0 Кб оверхеда).
  - В интерактивные острова передается срез переводов конкретного компонента через пропсы (`<ContactForm dict={t.common} />`).

---

### 2. Типизированная утилита перевода: `getTranslations`

- **Размещение:** `src/shared/i18n/utils.ts`.
- **Легковесный хелпер без сторонних библиотек:**
  ```ts
  import { ru } from './locales/ru';
  import { en } from './locales/en';

  const locales = { ru, en } as const;
  export type SupportedLocale = keyof typeof locales;
  export const DEFAULT_LOCALE: SupportedLocale = 'ru';

  export function getTranslations(lang: SupportedLocale = DEFAULT_LOCALE) {
    const dict = locales[lang] ?? locales[DEFAULT_LOCALE];

    return function t(section: keyof typeof dict, key: string, params?: Record<string, string | number>): string {
      const text = (dict[section] as Record<string, string>)?.[key] ?? key;
      if (!params) return text;

      // Простая интерполяция {paramName}
      return Object.entries(params).reduce(
        (acc, [k, v]) => acc.replace(new RegExp(`\\{${k}\\}`, 'g'), String(v)),
        text,
      );
    };
  }
  ```
- **Нативная плюрализация:**
  - Для правильных окончаний числительных в русском языке используется нативный объект `new Intl.PluralRules(locale)` без сторонних пакетов.

---

### 3. Маршрутизация и переключение языка (`astro.config.ts`)

- **Конфигурация в `astro.config.ts`:**
  ```ts
  import { defineConfig } from 'astro/config';

  export default defineConfig({
    i18n: {
      defaultLocale: 'ru',
      locales: ['ru', 'en'],
      routing: {
        prefixDefaultLocale: false, // основной язык без префикса (чистый URL)
      },
    },
  });
  ```
- **Структура страниц в `src/pages/`:**
  - `src/pages/index.astro` — главная страница (RU).
  - `src/pages/services.astro` — страница услуг (RU).
  - `src/pages/en/index.astro` — главная страница (EN).
  - `src/pages/en/services.astro` — страница услуг (EN).
- **Переключатель языков (Language Switcher):**
  - Реализуется через **нативные HTML-ссылки `<a>`**:
    ```html
    <nav aria-label="Язык / Language">
      <a href="/services" class:list={[{ active: currentLang === 'ru' }]}>RU</a>
      <a href="/en/services" class:list={[{ active: currentLang === 'en' }]}>EN</a>
    </nav>
    ```
  - Переключение доступно без JavaScript и моментально индексируется поисковыми роботами.

---

### 4. Локализация чисел, цен и дат

- **Категорический запрет сторонних библиотек:** отказ от `moment`, `dayjs`, `numeral`.
- **Цены и валюты (`Intl.NumberFormat`):**
  ```ts
  export function formatPrice(amount: number, locale: SupportedLocale, currency = 'RUB'): string {
    return new Intl.NumberFormat(locale === 'ru' ? 'ru-RU' : 'en-US', {
      style: 'currency',
      currency,
      maximumFractionDigits: 0,
    }).format(amount);
  }
  ```
- **Календарные даты (`Intl.DateTimeFormat`):**
  ```ts
  export function formatDate(date: Date, locale: SupportedLocale): string {
    return new Intl.DateTimeFormat(locale === 'ru' ? 'ru-RU' : 'en-US', {
      day: 'numeric',
      month: 'long',
      year: 'numeric',
    }).format(date);
  }
  ```
