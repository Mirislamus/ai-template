# ADR 0020: Контракт API, клиент Eden и формат ошибок

## Контекст и цели

Фронтенд и бэкенд живут в одном монорепо ([ADR 0019](0019-backend-architecture.md) §2), поэтому контракт API можно проверять типами при сборке, а не вручную.
Цели архитектурного стандарта:
1. **Сквозные типы:** клиент получает типы запросов и ответов из кода бэкенда через Eden Treaty.
2. **Предсказуемые ответы:** единые правила для адресов, JSON, дат, денег и пагинации.
3. **Единый формат ошибок:** стандарт RFC 9457 (Problem Details) с машинным кодом ошибки для фронтенда.
4. **Безопасность ответов:** наружу не уходят стек-трейсы, SQL и лишние поля сущностей.

---

## Принятые стандарты

### 1. Клиент: Eden Treaty внутри TanStack Query

- Свой API вызывается через Eden Treaty (`@elysia/eden`, [документация](https://elysiajs.com/eden/treaty/overview)). Сервер экспортирует тип приложения, клиент импортирует его только как тип:
  ```ts
  // apps/api/src/app.ts
  export const app = new Elysia({ prefix: '/api' }) /* .use(...) */;
  export type App = typeof app;
  ```
  ```ts
  // apps/web/src/shared/api/eden.ts
  import { treaty } from '@elysia/eden';
  import type { App } from 'api';

  let client: ReturnType<typeof treaty<App>> | undefined;

  // Клиент создается при первом вызове в браузере: модуль импортируется и при SSR, где нет window
  export function eden() {
    client ??= treaty<App>(window.location.origin);
    return client;
  }
  ```
- Префикс `/api` входит в путь клиента: маршрут `GET /api/orders` вызывается как `eden().api.orders.get()`.
- Запрос с сервера фронтенда (Server Components Next.js, SSR Astro) идет отдельным клиентом `treaty<App>(API_INTERNAL_URL)` из серверного кода; `API_INTERNAL_URL` — серверная переменная ([.env.example](../../../.env.example)). Такой запрос передает в API cookie входящего запроса, иначе сессия пользователя не будет видна.
- **TanStack Query** остается слоем кеша и мутаций ([ADR 0008](../frontend/0008-forms-and-api.md) §2): `queryFn` и `mutationFn` вызывают Eden и выбрасывают ошибку, если она пришла:
  ```ts
  export function useOrders(page: number) {
    return useQuery({
      queryKey: ordersKeys.list(page),
      queryFn: async () => {
        const { data, error } = await eden().api.orders.get({ query: { page, limit: 20 } });
        if (error) throw error;
        return data;
      },
    });
  }
  ```
- `ky` используется только для внешних API (CRM, эквайринг, сторонние сервисы) по [ADR 0008](../frontend/0008-forms-and-api.md) §1.
- Сессия передается httpOnly-cookie (ADR 0022). Заголовок `Authorization: Bearer` из примера ADR 0008 §1 к своему API не применяется.
- **CORS не используется:** API и сайт работают на одном домене, Nginx проксирует `/api` ([ADR 0018](../common/0018-deployment-and-caching.md) §5). В локальной разработке dev-сервер фронтенда проксирует `/api` на API ([docs/setup/backend.md](../../setup/backend.md)). Отдельный домен API требует нового решения со строгим списком разрешенных origin.

### 2. Адреса и методы

- Ресурсы во множественном числе в kebab-case: `/api/orders`, `/api/orders/:id`, `/api/order-items`. Вложенность не глубже одного уровня: `/api/orders/:id/items`.
- `GET` только читает данные. Создание — `POST`, частичное изменение — `PATCH`, удаление — `DELETE`.
- Коды успеха: `200` (данные), `201` (создано, тело — созданная сущность), `204` (без тела).

### 3. Формат данных

- Поля JSON в camelCase. Колонки БД остаются в snake_case, перевод выполняет Drizzle ([ADR 0021](0021-database-and-migrations.md) §2).
- Даты — строки ISO 8601 в UTC (`2026-10-03T09:15:00.000Z`). Eden по умолчанию превращает их в `Date` на клиенте, отображение — через `Intl.DateTimeFormat`.
- Идентификаторы — строки UUID ([ADR 0021](0021-database-and-migrations.md) §2).
- Деньги — целое число в минимальных единицах валюты и код валюты: `{ "amount": 1500000, "currency": "UZS" }`.
- Отсутствующее значение передается как `null`; набор полей ответа не зависит от данных.
- **Каждый маршрут описывает схему ответа** (`response` в Elysia) из `model.ts`. Поля, которых нет в схеме, наружу не уходят (служебные колонки, хеши, внутренние флаги).

### 4. Пагинация

- **Списки по умолчанию:** `?page=1&limit=20`, максимум `limit=100`; значение больше 100 отклоняется валидацией. Ответ:
  ```json
  { "items": [], "total": 0, "page": 1, "limit": 20 }
  ```
- Параметры списка синхронизируются со строкой запроса на фронтенде ([ARCHITECTURE.md](../../../ARCHITECTURE.md), «URL как SSOT»).
- **Бесконечные ленты** используют курсор: `?cursor=<id>&limit=20`, ответ `{ "items": [], "nextCursor": null }`.

### 5. Ошибки: RFC 9457 Problem Details

- Ответ с ошибкой имеет `Content-Type: application/problem+json` и тело:
  ```json
  {
    "status": 422,
    "title": "Validation failed",
    "code": "VALIDATION_FAILED",
    "detail": "Проверьте поля формы",
    "requestId": "c0a8017e-3f2b-4d1a-9b7e-2f6d1c8a9e10",
    "errors": [{ "path": "phone", "message": "Invalid phone" }]
  }
  ```
  Поле `type` опускается, это допустимо по [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) §3.1.1. Машинный идентификатор ошибки — `code`.
- **Базовые коды:**

  | HTTP | `code` | Когда |
  |---|---|---|
  | 401 | `UNAUTHORIZED` | Нет сессии |
  | 403 | `FORBIDDEN` | Не хватает роли или подтверждения (ADR 0022) |
  | 404 | `NOT_FOUND` | Ресурса нет или он принадлежит другому пользователю (ADR 0022 §5) |
  | 409 | `CONFLICT` | Конфликт состояния (дубль, гонка) |
  | 422 | `VALIDATION_FAILED` | Тело, параметры или query не прошли схему; поле `errors` перечисляет поля |
  | 429 | `RATE_LIMITED` | Превышен лимит запросов (ADR 0023 §3) |
  | 500 | `INTERNAL` | Непредвиденная ошибка |

- Ошибки предметной области получают свой `code` в UPPER_SNAKE_CASE (`ORDER_ALREADY_PAID`) и выбрасываются классом `AppError` из `src/shared/errors.ts`.
- **Ошибка 500** наружу отдает только `status`, `title`, `code` и `requestId`. Стек, SQL и текст исключения пишутся в лог (ADR 0023 §5).
- Глобальный обработчик регистрируется в `app.ts` до модулей:
  ```ts
  // apps/api/src/shared/errors.ts
  const STATUS_TITLES: Partial<Record<number, string>> = {
    401: 'Unauthorized',
    403: 'Forbidden',
    404: 'Not found',
    409: 'Conflict',
    422: 'Validation failed',
    429: 'Too many requests',
    500: 'Internal server error',
  };

  export class AppError extends Error {
    constructor(
      readonly status: number,
      readonly code: string,
      readonly detail?: string,
    ) {
      super(code);
    }
  }

  export function problem(status: number, code: string, requestId: string, extra: Record<string, unknown> = {}) {
    return Response.json(
      { status, title: STATUS_TITLES[status] ?? 'Error', code, requestId, ...extra },
      { status, headers: { 'Content-Type': 'application/problem+json' } },
    );
  }
  ```
  ```ts
  // apps/api/src/app.ts (фрагмент)
  .error({ AppError })
  .onError(({ code, error, request }) => {
    const requestId = getRequestId(request);
    if (code === 'AppError') return problem(error.status, error.code, requestId, { detail: error.detail });
    if (code === 'VALIDATION') {
      const errors = error.all.map(issue => ({ path: issue.path, message: issue.message }));
      return problem(422, 'VALIDATION_FAILED', requestId, { errors });
    }
    if (code === 'NOT_FOUND') return problem(404, 'NOT_FOUND', requestId);
    logger.error({ err: error, requestId }, 'unhandled error');
    return problem(500, 'INTERNAL', requestId);
  })
  ```
  `getRequestId` и `logger` — из `src/shared/` (ADR 0023 §5).
- **На фронтенде** тело ошибки разбирается Zod-схемой из `packages/shared`. Текст для пользователя берется из словаря интерфейса по `code` ([ADR 0016](../frontend/0016-i18n-standards.md)), а не из `detail`. Ошибки полей из `errors` выводятся у соответствующих полей формы ([ADR 0008](../frontend/0008-forms-and-api.md) §5).

### 6. Документация API

- Плагин `@elysia/openapi` подключается только вне продакшена (`NODE_ENV !== 'production'`). На проде страница документации и JSON-спецификация недоступны.
- Схемы маршрутов из `model.ts` — единственный источник документации; ручные описания API не ведутся.
