# ADR 0015: Аналитика, отслеживание целей и производительность счетчиков

## Контекст и цели

Подключение счетчиков веб-аналитики (Яндекс.Метрика, Google Analytics 4) необходимо для бизнеса, но классическая синхронная вставка скриптов в `<head>` ухудшает Core Web Vitals и повышает время блокировки потока (TBT).
Цели архитектурного стандарта:
1. **Сохранение метрик Lighthouse при наличии аналитики:** отложенная инициализация счетчиков без блокировки рендера первого экрана.
2. **Защита от сбоев и AdBlock:** централизованный типобезопасный фасад `trackEvent`, исключающий падения приложения при заблокированных скриптах.
3. **Чистый код без разбросанных ID:** идентификаторы счетчиков в переменных окружения, единый список типов целей.
4. **Правовая прозрачность (152-ФЗ / GDPR):** ненавязчивое уведомление об использовании cookie и ссылка на политику конфиденциальности.

---

## Принятые стандарты

### 1. Стратегия загрузки счетчиков аналитики

- **Запрет синхронной вставки в `<head>`:** тяжелые скрипты (`tag.js`, `gtag.js`) не подключаются синхронно на старте документа.
- **Модель согласия проекта:** счетчики загружаются по описанной ниже стратегии **без ожидания нажатия кнопки на баннере**. Баннер информирует пользователя (§3); отдельного opt-in для аналитики в шаблоне нет. Политика конфиденциальности описывает используемые cookie и способ их отключения в браузере.
- **Отложенная загрузка по первому взаимодействию:** инициализация ждет первого пользовательского события. Если действий нет, скрипт загружается в фоне через `requestIdleCallback` (в Safari его нет, поэтому фоллбэк — `setTimeout`):
  ```ts
  // src/shared/lib/analytics/load-analytics.ts
  const INTERACTION_EVENTS = ['scroll', 'mousemove', 'touchstart', 'keydown'] as const;

  function scheduleIdle(callback: () => void): void {
    if (typeof window.requestIdleCallback === 'function') {
      window.requestIdleCallback(callback, { timeout: 4000 });
    } else {
      setTimeout(callback, 3000);
    }
  }

  export function loadAnalyticsDeferred(load: () => void): void {
    let isLoaded = false;

    function runOnce(): void {
      if (isLoaded) return;
      isLoaded = true;
      for (const eventName of INTERACTION_EVENTS) window.removeEventListener(eventName, runOnce);
      load();
    }

    for (const eventName of INTERACTION_EVENTS) {
      window.addEventListener(eventName, runOnce, { once: true, passive: true });
    }
    scheduleIdle(runOnce);
  }
  ```
- **Результат:** браузер успевает отрисовать первый экран (LCP) без участия счетчика, а нагрузка на основной поток появляется уже после первой отрисовки.

---

### 2. Типизированный фасад аналитики: `trackEvent`

- **Размещение:** `src/shared/lib/analytics/track-event.ts`.
- **Категорический запрет прямых вызовов:** `window.ym(...)` и `window.gtag(...)` не пишутся в JSX-компонентах и обработчиках кликов.
- **Типы глобальных функций счетчиков** (расширение `Window` через interface, ADR 0004 §2):
  ```ts
  declare global {
    interface Window {
      ym?: (counterId: number, method: string, ...params: unknown[]) => void;
      gtag?: (command: string, ...params: unknown[]) => void;
    }
  }
  ```
- **Структура фасада с дискриминантными типами:**
  ```ts
  import { env } from '@/shared/config/env';

  export type AnalyticsEvent =
    | { name: 'lead_form_submit'; payload: { formId: string } }
    | { name: 'phone_click'; payload: { location: string } }
    | { name: 'modal_open'; payload: { modalId: string } }
    | { name: 'cta_click'; payload: { location: string } }
    | { name: 'catalog_filter'; payload: { filterId: string; value: string } };

  export function trackEvent(event: AnalyticsEvent): void {
    // Безопасная отправка в Яндекс.Метрику (защита от падения при AdBlock)
    if (typeof window.ym === 'function' && env.PUBLIC_YM_ID) {
      window.ym(Number(env.PUBLIC_YM_ID), 'reachGoal', event.name, event.payload);
    }

    // Безопасная отправка в Google Analytics 4
    if (typeof window.gtag === 'function') {
      window.gtag('event', event.name, event.payload);
    }
  }
  ```
- **Использование в компонентах:**
  ```tsx
  trackEvent({
    name: 'lead_form_submit',
    payload: { formId: 'hero-callback' },
  });
  ```
- Новые цели добавляются в union `AnalyticsEvent`; события, которых в нем нет, использовать нельзя (проверка на этапе типов).

---

### 3. Уведомление об использовании Cookie

- **Размещение:** `src/features/cookie-consent/CookieConsent.astro` — статический компонент с небольшим `<script>` без React-острова (0 Кб лишнего React JS).
- **Поведение:**
  - Ненавязчивая плашка в нижней части экрана: краткий текст, ссылка на [Политику конфиденциальности](../pages/privacy-policy.md) и кнопка «Понятно».
  - Факт закрытия сохраняется в `localStorage` (`cookie_notice_dismissed: 'true'`) через безопасную обертку над Web API (ADR 0004 §9, ADR 0006 §5); повторно плашка не показывается.
  - Нажатие кнопки не влияет на загрузку счетчиков (§1).
