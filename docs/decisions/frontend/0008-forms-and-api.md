# ADR 0008: Интеграция с API, обработка форм и веб-безопасность

## Контекст и цели

Формы обратной связи, заявок и сетевые запросы — главный мост между пользователем и бизнесом.
Цели архитектурного стандарта:
1. **Надежность сетевого слоя:** использование легковесного клиента `ky` с автоповтором (retry), таймаутами и интеграцией с Zod.
2. **Управление серверным состоянием:** применение **TanStack Query** для кэширования, синхронизации и мутаций данных без ручных флагов загрузки.
3. **Качество данных форм:** строгая валидация схем через Zod, передача очищенных номеров телефонов (unmasked) из `imask`.
4. **Невидимый антиспам:** защита через Honeypot и временные ловушки (Time-trap) без раздражающих капч (отказ от Google reCAPTCHA).
5. **Доступность и предсказуемость UI:** обязательная реализация 5 состояний формы (Idle, Submitting, Success, Error, Disabled).
6. **Веб-безопасность:** сокрытие секретных ключей через внутренние API-эндпоинты, защита от XSS, Clickjacking и безопасные внешние ссылки.

---

## Принятые стандарты

### 1. Сетевой клиент: `ky` и типизация API

- **Стандартный HTTP-клиент: `ky`:**
  - Для сетевых запросов используется легковесная библиотека `ky` (~3 Кб) поверх нативного Fetch API.
  - **Запрещен Axios:** избыточный размер бандла и устаревшая архитектура.
  - Единый инстанс настраивается в `src/shared/api/client.ts` с хуками:
    ```ts
    import ky, { type HTTPError } from 'ky';
    import { env } from '@/shared/config/env';

    export const api = ky.create({
      prefixUrl: env.PUBLIC_API_URL,
      timeout: 10000, // 10 секунд таймаут по умолчанию
      retry: {
        limit: 2,
        statusCodes: [408, 413, 429, 500, 502, 503, 504],
      },
      hooks: {
        beforeRequest: [
          request => {
            // Автоматическое добавление токена авторизации (если есть)
            const token = getAuthToken();
            if (token) {
              request.headers.set('Authorization', `Bearer ${token}`);
            }
          },
        ],
        afterResponse: [
          async (_request, _options, response) => {
            // Централизованная обработка протухшей сессии
            if (response.status === 401) {
              handleUnauthorizedSession();
            }
          },
        ],
      },
    });
    ```
- **Фабричный хелпер валидации `fetchSchema`:**
  - Для автоматического вывода типов и однострочной рантайм-валидации через Zod:
    ```ts
    import type { ZodType, z } from 'zod';

    export async function fetchSchema<TSchema extends ZodType>(
      request: Promise<unknown>,
      schema: TSchema,
    ): Promise<z.infer<TSchema>> {
      const data = await request;
      return schema.parse(data);
    }
    ```
  - Использование в API-модулях:
    ```ts
    export function getUser(id: string) {
      return fetchSchema(api.get(`users/${id}`).json(), userSchema);
    }
    ```
- **Типизированный парсинг ошибок бэкенда (без приведений типов):**
  ```ts
  import { HTTPError } from 'ky';
  import { z } from 'zod';

  const errorBodySchema = z.object({ message: z.string() });

  export async function getApiErrorMessage(error: unknown): Promise<string> {
    if (error instanceof HTTPError) {
      const fallback = `Ошибка сервера (${error.response.status})`;
      try {
        const body: unknown = await error.response.json();
        const parsed = errorBodySchema.safeParse(body);
        return parsed.success ? parsed.data.message : fallback;
      } catch {
        return fallback; // тело ответа не является JSON
      }
    }
    if (error instanceof Error) return error.message;
    return 'Неизвестная сетевая ошибка';
  }
  ```
- **Базовый URL:** `PUBLIC_API_URL` для собственного серверного эндпоинта форм равен `/api` ([.env.example](../../../.env.example)).
- **Проект с отдельным бэкендом:** свой API вызывается через Eden Treaty, сессия передается httpOnly-cookie без заголовка `Authorization`, ошибки приходят в формате RFC 9457 ([ADR 0020](../backend/0020-api-contract-and-errors.md)). Клиент `ky` из этого раздела остается для внешних API.

---

### 2. Управление серверным состоянием: TanStack Query

- **Область применения:**
  - В Next.js — для любых клиентских выборок, фильтрации каталога и бесконечной пагинации.
  - В интерактивных островах Astro — для динамических виджетов, корзины и отправки заявок (остров оборачивается в `QueryProvider`).
- **Централизованная конфигурация `QueryClient` (`src/shared/api/query-client.ts`):**
  - Разумные дефолты для веб-сайтов, предотвращающие паразитный трафик. **Ретраи выполняет только `ky`** (один слой): повторы в Query поверх повторов `ky` умножают число запросов (до 9 на один вызов), поэтому в `QueryClient` задается `retry: false`:
    ```ts
    import { QueryClient } from '@tanstack/react-query';

    export const queryClient = new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 5 * 60 * 1000, // 5 минут данные считаются свежими
          gcTime: 10 * 60 * 1000, // 10 минут хранятся в памяти
          refetchOnWindowFocus: false, // Не перезапрашивать при клике в окно
          retry: false, // ретраи настроены в ky (src/shared/api/client.ts)
        },
      },
    });
    ```
  - Остров Astro оборачивается в `QueryProvider` с этим экземпляром ([ADR 0005](0005-react-standards.md) §9).
- **Паттерн Query Key Factory:**
  - Исключение опечаток в ключах кеша и централизованная инвалидация:
    ```ts
    // src/features/catalog/api/product.keys.ts
    export const productKeys = {
      all: ['products'] as const,
      lists: () => [...productKeys.all, 'list'] as const,
      list: (filters: ProductFilters) => [...productKeys.lists(), filters] as const,
      details: () => [...productKeys.all, 'detail'] as const,
      detail: (id: string) => [...productKeys.details(), id] as const,
    };
    ```
- **Строгая изоляция в кастомные хуки:**
  - **Запрещено** вызывать `useQuery` / `useMutation` напрямую внутри JSX компонентов.
  - Все сетевые вызовы инкапсулируются в кастомные хуки в слое `api/` фичи или сущности:
    ```ts
    // src/features/catalog/api/use-products.ts
    export function useProducts(filters: ProductFilters) {
      return useQuery({
        queryKey: productKeys.list(filters),
        queryFn: () => getProducts(filters),
      });
    }

    // src/features/contact-form/api/use-submit-contact.ts
    export function useSubmitContact() {
      return useMutation({
        mutationFn: (values: ContactFormValues) => api.post('contact', { json: values }).json(),
        onSuccess: () => {
          toast.success('Заявка успешно отправлена!');
        },
        onError: async error => {
          toast.error(await getApiErrorMessage(error));
        },
      });
    }
    ```

---

### 3. Валидация форм и маски ввода

- **React Hook Form + Zod — обязательный стек для форм от 2 полей:**
  - Схемы выносятся в изолированные файлы модели (`contact-form.schema.ts`).
  - Свойства строгой типизации:
    - Имя: `z.string().trim().min(2, 'Минимум 2 символа').max(50)`.
    - Телефон: проверка на минимальное количество цифр.
    - Email: `z.string().trim().toLowerCase().pipe(z.email('Некорректный email'))`.
    - Чекбокс согласия на обработку персональных данных: `z.literal(true, { error: 'Подтвердите согласие на обработку персональных данных' })`.
- **Маска ввода телефона (`imask`):**
  - В базу и на сервер отправляется **строго очищенное значение (unmaskedValue)** — только цифры (например, `79991234567`), без скобок и дефисов.
  - В чистом Astro библиотека маски подгружается динамически (`import('imask')`) строго по событию `focus` на поле телефона.

---

### 4. Невидимый антиспам (Anti-spam Protection)

- **Тотальный отказ от Google reCAPTCHA:** пазлы и картинки убивают конверсию и раздувают клиентский JS на 150+ Кб.
- **Двухуровневая невидимая защита:**
  1. **Honeypot (Поле-ловушка):**
     - Скрытое поле `<input type="text" name="website" tabIndex={-1} autoComplete="off" className="visually-hidden" />`.
     - Если поле заполнено (боты автоматически заполняют все поля) — запрос реджектится без отправки лида.
  2. **Time-trap (Временная метка):**
     - Клиент передает в запросе момент рендера формы (`startedAt`, мс). Заявки, отправленные быстрее чем за 1.5 секунды, отсекаются как спам-скрипты.
  - **Обе проверки выполняются на сервере** (клиентская проверка обходится). При срабатывании эндпоинт отвечает `200 OK` без отправки лида, чтобы бот не узнал о ловушке.
- **Cloudflare Turnstile (Резервный рубеж):**
  - Бесшовная невидимая капча без решения задач подключается только при фиксации направленной распределенной спам-атаки.
- **Серверный эндпоинт формы (Astro, [ADR 0001](0001-project-architecture.md) §7):**
  ```ts
  // src/pages/api/contact.ts
  import type { APIRoute } from 'astro';
  import { TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID } from 'astro:env/server';

  import { contactRequestSchema } from '@/features/contact-form/model/contact-form.schema';

  export const prerender = false;

  export const POST: APIRoute = async ({ request }) => {
    let body: unknown;
    try {
      body = await request.json();
    } catch {
      return Response.json({ message: 'Некорректный формат запроса' }, { status: 400 });
    }

    const parsed = contactRequestSchema.safeParse(body);
    if (!parsed.success) {
      return Response.json({ message: 'Проверьте заполнение полей формы' }, { status: 400 });
    }

    const { website, startedAt, ...lead } = parsed.data;
    const isBot = website !== '' || Date.now() - startedAt < 1500;
    if (isBot) return Response.json({ ok: true }); // тихо отсекаем спам

    // Отправка лида в интеграцию (Telegram, CRM) с секретами из окружения
    await sendLead(lead, { token: TELEGRAM_BOT_TOKEN, chatId: TELEGRAM_CHAT_ID });
    return Response.json({ ok: true });
  };
  ```
  - `contactRequestSchema` расширяет клиентскую схему полями `website` и `startedAt`. Приватные ключи читаются только на сервере.

---

### 5. Пять обязательных состояний интерфейса формы

Любая форма связи обязана поддерживать 5 состояний формы (не путать с состояниями элемента из [docs/design.md](../../design.md) §7 и состояниями данных Loading / Empty / Error / Success):

1. **Idle (Покой):** поля чистые или содержат плейсхолдеры, кнопка активна.
2. **Submitting (Отправка):**
   - Кнопка заблокирована (`disabled={isSubmitting}`).
   - Отображается спиннер, текст кнопки меняется («Отправка...»).
   - Поля ввода блокируются для изменений.
3. **Success (Успех):**
   - Форма очищается (`reset()`), отображается экран благодарности или всплывает toast-уведомление Sonner.
   - Оповещение скринридеров через `aria-live="polite"`.
4. **Error (Ошибка):**
   - Поля с ошибкой подсвечиваются с установкой `aria-invalid="true"` и ссылкой на текст ошибки `aria-describedby`.
   - При падении сети выводится понятное общее сообщение без технических стектрейсов.
5. **Disabled (Недоступно):**
   - Форма приглушена, если прием заявок временно приостановлен.

---

### 6. Стандарты веб-безопасности

- **Защита приватных секретов и CRM:**
  - **Категорически запрещено** делать запросы к внешним сервисам (Telegram Bot API, CRM API, рассылки) прямо из клиентского браузера.
  - Клиент отправляет форму строго на внутренний серверный эндпоинт (`/api/contact`), где сервер от своего имени безопасно выполняет запрос, используя приватные переменные окружения.
- **Защита от XSS (Межсайтовый скриптинг):**
  - Запрещен вывод сырого HTML через `dangerouslySetInnerHTML` или `set:html` без предварительной санитизации через `dompurify`.
- **Безопасность внешних ссылок:**
  - Все ссылки на внешние ресурсы с `target="_blank"` обязаны содержать атрибут `rel="noopener noreferrer"` для предотвращения доступа к родительскому объекту `window.opener`.
- **Заголовки безопасности веб-сервера (конфигурация — [ADR 0018](../common/0018-deployment-and-caching.md) §3):**
  - `X-Frame-Options: DENY` (защита от встраивания сайта во фреймы / Clickjacking).
  - `X-Content-Type-Options: nosniff` (запрет угадывания MIME-типов).
  - `Referrer-Policy: strict-origin-when-cross-origin`.
