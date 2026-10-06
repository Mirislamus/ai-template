# ADR 0022: Авторизация, сессии и права доступа

## Контекст и цели

Собственная реализация входа, хранения паролей и сессий — частый источник уязвимостей. Вторая частая уязвимость — доступ к чужим данным подменой идентификатора в адресе (IDOR).
Цели архитектурного стандарта:
1. **Проверенная библиотека вместо своей криптографии:** Better Auth для входа, сессий и паролей.
2. **Сессии в httpOnly-cookie:** токены недоступны JavaScript страницы.
3. **Способы входа под проект:** набор включается по решению в [PRODUCT.md](../../../PRODUCT.md).
4. **Защита админки:** обязательная двухфакторная аутентификация для администраторов.
5. **Проверка владельца в каждом запросе:** пользователь видит и меняет только свои данные.

---

## Принятые стандарты

### 1. Better Auth

- Авторизация реализуется [Better Auth](https://www.better-auth.com/) с адаптером Drizzle (`@better-auth/drizzle-adapter`, провайдер `pg`). Конфигурация — `src/shared/auth.ts`.
- Обработчик монтируется в Elysia (`.mount(auth.handler)`, [интеграция](https://www.better-auth.com/docs/integrations/elysia)) и обслуживает пути `/api/auth/*`.
- Таблицы Better Auth генерируются командой `bunx auth@latest generate` и проходят обычный цикл миграций ([ADR 0021](0021-database-and-migrations.md) §5).
- Хеширование паролей, выпуск и проверку сессий выполняет только Better Auth. Собственное хеширование паролей и собственные токены запрещены.
- `BETTER_AUTH_SECRET` — случайная строка не короче 32 байт, своя для каждого окружения ([ADR 0023](0023-security-logging-and-config.md) §4).

### 2. Сессии

- Сессия хранится в БД и передается httpOnly-cookie. Атрибуты cookie по умолчанию (`HttpOnly`, `Secure` в продакшене, `SameSite=Lax`) не ослабляются.
- Срок жизни — значения Better Auth по умолчанию: 7 дней с продлением раз в сутки при активности ([документация](https://www.better-auth.com/docs/concepts/session-management)).
- Выход из аккаунта и смена пароля завершают сессию на сервере, а не только удаляют cookie.

### 3. Способы входа

Шаблон описывает четыре способа; проект включает нужные по решению из PRODUCT.md.

| Способ | Реализация | Обязательные условия |
|---|---|---|
| Email + пароль | `emailAndPassword` | SMTP для подтверждения и сброса пароля |
| Телефон + SMS-код | Плагин `phoneNumber` ([документация](https://www.better-auth.com/docs/plugins/phone-number)), отправка через `sendOTP` | SMS-провайдер выбирается в проекте; лимит отправки кодов на номер и IP (ADR 0023 §3) |
| Google | `socialProviders.google` | OAuth-клиент Google Cloud |
| Telegram | Плагин `genericOAuth` ([документация](https://www.better-auth.com/docs/plugins/generic-oauth)) с `discoveryUrl: 'https://oauth.telegram.org/.well-known/openid-configuration'` и PKCE ([Telegram Login](https://core.telegram.org/bots/telegram-login)) | Client ID и Secret от BotFather |

- Сторонние неофициальные плагины входа не подключаются.
- Попытки ввода SMS-кода ограничены плагином: 3 попытки на код, затем нужен новый код.

### 4. Пароли и подтверждение

- **Пароль:** от 8 до 128 символов, без правил вида «заглавная, цифра, символ».
- **Утекшие пароли** отклоняются плагином `haveIBeenPwned`: во внешний сервис уходят только первые 5 символов хеша пароля ([документация](https://www.better-auth.com/docs/plugins/have-i-been-pwned)).
- **Подтверждение email или телефона** обязательно до важных действий: оплата, оформление заказа, смена контактных данных и пароля. Войти и просматривать данные можно без подтверждения. Неподтвержденный пользователь получает `403` с кодом `VERIFICATION_REQUIRED`.

### 5. Роли и проверка владельца

- **Роли:** поле `role` пользователя (`user` | `admin`), объявленное в `user.additionalFields` с `defaultValue: 'user'` и `input: false`: пользователь не может назначить себе роль через API. Значение по умолчанию и `check`-ограничение дублируются в миграции БД.
- **Первый администратор** назначается скриптом на сервере, не через публичный API.
- **Проверка доступа в маршрутах** — макросы Elysia в `src/shared/auth.ts`:
  - `auth: true` — нужна сессия, иначе `401 UNAUTHORIZED`;
  - `verified: true` — нужен подтвержденный email или телефон, иначе `403 VERIFICATION_REQUIRED`;
  - `role: 'admin'` — нужна роль, иначе `403 FORBIDDEN`.
  ```ts
  .get('/orders/:id', ({ user, params }) => ordersService.getById(user.id, params.id), {
    auth: true,
    params: orderParamsSchema,
    response: orderSchema,
  })
  ```
- **Проверка владельца** выполняется в сервисе условием `where` запроса, а не фильтрацией после выборки:
  ```ts
  export async function getById(userId: string, orderId: string) {
    const [order] = await db
      .select()
      .from(orders)
      .where(and(eq(orders.id, orderId), eq(orders.userId, userId)));
    if (!order) throw new AppError(404, 'NOT_FOUND');
    return order;
  }
  ```
- Чужой ресурс возвращает `404 NOT_FOUND`, а не `403`: ответ не подтверждает, что ресурс существует.

### 6. Двухфакторная аутентификация

- Плагин `twoFactor` с TOTP-кодами из приложения-аутентификатора и резервными кодами ([документация](https://www.better-auth.com/docs/plugins/2fa)).
- **Для `admin` 2FA обязательна:** макрос `role: 'admin'` дополнительно проверяет `user.twoFactorEnabled`; без включенной 2FA ответ `403` с кодом `TWO_FACTOR_REQUIRED`, и интерфейс ведет администратора на включение 2FA.
- Для обычных пользователей 2FA включается по желанию.

### 7. Защита от перебора

- Встроенный rate limit Better Auth включен в продакшене по умолчанию (100 запросов за 10 секунд, вход — 3 попытки за 10 секунд, [документация](https://www.better-auth.com/docs/concepts/rate-limit)) и не отключается.
- Хранилище лимитов — память процесса, пока API работает в одном экземпляре. При нескольких экземплярах хранилище переключается на БД (`storage: 'database'`).
- Общие лимиты Nginx и лимиты форм — [ADR 0023](0023-security-logging-and-config.md) §3.
