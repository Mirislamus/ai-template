# ADR 0016: Архитектура интернационализации и переводов (i18n)

## Контекст и цели

Многоязычность сайта влияет на охват аудитории, конверсию и региональное поисковое ранжирование.
Цели архитектурного стандарта:
1. **100% типобезопасность словарей:** константные TypeScript-объекты вместо сырых JSON для автокомплита и проверки отсутствующих ключей на этапе компиляции.
2. **Нулевой лишний вес в клиенте:** переводы компилируются в статический HTML на этапе сборки (Astro SSG), в интерактивные острова передается только срез активной локали.
3. **Отказ от тяжелых библиотек:** вместо `react-i18next` — легковесная типизированная утилита `t()` и нативный `Intl.PluralRules`.
4. **Четкая маршрутизация:** встроенный модуль `astro:i18n` в `astro.config.ts` с бесшовными URL (`/` для основного языка, `/en/` для альтернативного).
5. **Нативная локализация чисел и дат:** `Intl.NumberFormat` и `Intl.DateTimeFormat`.

---

## Принятые стандарты

### 1. Формат и структура словарей переводов

- **Размещение:** `src/shared/i18n/locales/` (`ru.ts`, `en.ts`, `uz.ts`).
- **Все переводы хранятся в TypeScript-файлах и входят в сборку.** Загрузка словарей с сервера в рантайме не используется; на статическую страницу и в остров попадают только строки активной локали (см. ниже).
- **Константные объекты `as const`:**
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
  ```
- **Схема выводится из основного языка с расширением литералов до `string`** (иначе альтернативные языки пришлось бы писать русскими литералами):
  ```ts
  // src/shared/i18n/schema.ts
  import type { ru } from './locales/ru';

  type Widen<TValue> = TValue extends string
    ? string
    : { -readonly [TKey in keyof TValue]: Widen<TValue[TKey]> };

  export type TranslationSchema = Widen<typeof ru>;

  type Paths<TValue> = TValue extends string
    ? never
    : {
        [TKey in keyof TValue & string]: TValue[TKey] extends string ? TKey : `${TKey}.${Paths<TValue[TKey]>}`;
      }[keyof TValue & string];

  export type TranslationKey = Paths<TranslationSchema>;
  ```
- **Контроль целостности:** файлы альтернативных языков типизируются `TranslationSchema`; пропущенный ключ или опечатка роняет сборку:
  ```ts
  // src/shared/i18n/locales/en.ts
  import type { TranslationSchema } from '../schema';

  export const en: TranslationSchema = {
    common: { submit: 'Send request', loading: 'Sending...', success: 'Request sent', error: 'Something went wrong' },
    nav: { services: 'Services', about: 'About', contacts: 'Contacts' },
  };
  ```
- **Изоляция размера бандла:** в статическом рендере Astro на странице `/services` вшит только русский текст, на `/en/services` — только английский; JS-файлы словарей в браузер не отправляются. В интерактивные острова передается срез конкретного раздела активной локали (`<ContactForm dict={getDictionary(lang).common} />`).

---

### 2. Типизированные утилиты перевода

- **Размещение:** `src/shared/i18n/utils.ts`.
- **Легковесные хелперы без сторонних библиотек и без приведений типов:**
  ```ts
  import { en } from './locales/en';
  import { ru } from './locales/ru';
  import type { TranslationKey, TranslationSchema } from './schema';

  const locales = { ru, en } as const;

  export type SupportedLocale = keyof typeof locales;
  export const DEFAULT_LOCALE: SupportedLocale = 'ru';

  export function getDictionary(lang: SupportedLocale): TranslationSchema {
    return locales[lang];
  }

  function resolve(dict: TranslationSchema, key: string): string | undefined {
    let current: unknown = dict;
    for (const part of key.split('.')) {
      if (typeof current !== 'object' || current === null || !(part in current)) return undefined;
      current = Reflect.get(current, part);
    }
    return typeof current === 'string' ? current : undefined;
  }

  export function getTranslations(lang: SupportedLocale = DEFAULT_LOCALE) {
    const dict = getDictionary(lang);

    return function t(key: TranslationKey, params?: Record<string, string | number>): string {
      const text = resolve(dict, key) ?? key;
      if (!params) return text;
      // Простая интерполяция {paramName}
      return text.replaceAll(/\{(\w+)\}/g, (match, name: string) => String(params[name] ?? match));
    };
  }
  ```
- Использование: `t('nav.services')`; несуществующий ключ не проходит проверку типов.
- **Нативная плюрализация:** для окончаний числительных используется `new Intl.PluralRules(locale)` без сторонних пакетов.

---

### 3. Маршрутизация и переключение языка (`astro.config.ts`)

- **Конфигурация:**
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
  - `src/pages/index.astro` — главная (RU), `src/pages/services.astro` — услуги (RU).
  - `src/pages/en/index.astro` — главная (EN), `src/pages/en/services.astro` — услуги (EN).
- **Переключатель языков** реализуется нативными HTML-ссылками `<a>`:
  ```html
  <nav aria-label="Язык / Language">
    <a href="/services" class:list={[{ active: currentLang === 'ru' }]}>RU</a>
    <a href="/en/services" class:list={[{ active: currentLang === 'en' }]}>EN</a>
  </nav>
  ```
  - Переключение работает без JavaScript и сразу индексируется поисковыми роботами; связка `hreflang` — [ADR 0013](0013-seo-standards.md) §5.

---

### 4. Локализация чисел, цен и дат

- **Категорический запрет сторонних библиотек:** `moment`, `dayjs`, `numeral` не используются.
- **Соответствие локали и `Intl`-тега** задается одним объектом:
  ```ts
  const INTL_LOCALES = { ru: 'ru-RU', en: 'en-US' } as const satisfies Record<SupportedLocale, string>;
  ```
- **Цены и валюты (`Intl.NumberFormat`):**
  ```ts
  export function formatPrice(amount: number, locale: SupportedLocale, currency = 'RUB'): string {
    return new Intl.NumberFormat(INTL_LOCALES[locale], {
      style: 'currency',
      currency,
      maximumFractionDigits: 0,
    }).format(amount);
  }
  ```
- **Календарные даты (`Intl.DateTimeFormat`):**
  ```ts
  export function formatDate(date: Date, locale: SupportedLocale): string {
    return new Intl.DateTimeFormat(INTL_LOCALES[locale], {
      day: 'numeric',
      month: 'long',
      year: 'numeric',
    }).format(date);
  }
  ```
