# Лабораторная работа №4: Аутентификация и авторизация

## Цель работы
Реализовать систему аутентификации на основе JWT токенов, многоуровневую авторизацию с ролями и политиками.

---

## Задание 1. Система пользователей

### Модель User
Создайте модель пользователя со следующими полями:
- `Id` — Guid, первичный ключ
- `Email` — string, уникальный, required
- `Username` — string, уникальный, required
- `PasswordHash` — string, хеш пароля
- `PasswordSalt` — string, соль для хеша
- `FirstName` — string
- `LastName` — string
- `DateOfBirth` — DateTime (nullable)
- `PhoneNumber` — string (nullable)
- `CreatedAt` — DateTime
- `UpdatedAt` — DateTime
- `LastLoginAt` — DateTime (nullable)
- `IsEmailConfirmed` — bool
- `IsActive` — bool
- `IsDeleted` — bool (для soft delete)

### Модель UserSession
Для управления сессиями:
- `Id` — Guid
- `UserId` — Guid, FK к User  
- `RefreshTokenHash` — string
- `CreatedAt` — DateTime
- `ExpiresAt` — DateTime
- `DeviceInfo` — string
- `IpAddress` — string
- `IsRevoked` — bool

**Преимущества подхода:**
- Поддержка нескольких активных сессий (несколько устройств)
- Безопасное хеширование refresh токенов
- Улучшенный аудит и мониторинг сессий

### Модель Role
Создайте модель роли:
- `Id` — Guid
- `Name` — string (Admin, User, Moderator + специфичные для проекта)
- `Description` — string
- `CreatedAt` — DateTime

Добавьте минимум 3-4 роли для вашего проекта.

### Модель UserRole (Many-to-Many)
Промежуточная таблица для связи пользователей и ролей:
- `UserId` — Guid, FK к User
- `RoleId` — Guid, FK к Role
- `AssignedAt` — DateTime
- `AssignedBy` — Guid (nullable), кто назначил роль

### Модель Permission
Для детального управления правами:
- `Id` — Guid
- `Name` — string (CanEditPost, CanDeleteUser, CanViewReports и т.д.)
- `Description` — string
- `Category` — string (для группировки)

Создайте минимум 5-7 permissions для вашего проекта.

### Модель RolePermission (Many-to-Many)
Связь ролей и прав:
- `RoleId` — Guid
- `PermissionId` — Guid

### Модель UserClaim
Для дополнительных claims пользователя:
- `Id` — Guid
- `UserId` — Guid, FK к User
- `ClaimType` — string (SubscriptionLevel, Department, etc.)
- `ClaimValue` — string

### Конфигурация в DbContext
- Настройте все связи через Fluent API
- Уникальные индексы на Email и Username
- Каскадное удаление для UserRole и UserClaim
- Seed data для ролей и permissions

---

## Задание 2. Регистрация пользователей

### Endpoint POST /api/auth/register

**DTO для регистрации (RegisterDTO):**
- Email
- Username
- Password
- ConfirmPassword
- FirstName
- LastName
- DateOfBirth (optional)
- PhoneNumber (optional)

### Валидация через FluentValidation
Создайте валидатор со следующими правилами:

**Email:**
- NotEmpty
- EmailAddress
- MaxLength(200)
- Асинхронная проверка уникальности в БД

**Username:**
- NotEmpty
- Length(3, 50)
- Matches regex (только буквы, цифры, подчёркивание)
- Асинхронная проверка уникальности в БД

**Password:**
- NotEmpty
- MinimumLength(8)
- RequireUppercase
- RequireLowercase
- RequireDigit
- RequireSpecialCharacter

**ConfirmPassword:**
- Equal(Password)

**DateOfBirth (если обязательно):**
- NotEmpty
- Возраст минимум 18 лет
- Не в будущем

**PhoneNumber (если есть):**
- Matches regex для формата телефона

### Бизнес-логика регистрации
Реализуйте следующий flow:

1. **Валидация данных** — через FluentValidation
2. **Хеширование пароля:**
   - Используйте BCrypt.Net-Next или PBKDF2 или Argon2
   - Генерация соли
   - Хеширование с солью
3. **Создание пользователя:**
   - Заполнение всех полей
   - IsActive = true
   - IsEmailConfirmed = false
   - Роль "User" по умолчанию
4. **Генерация confirmation token:**
   - Криптографически стойкий random token
   - Сохранение в БД или кодирование в JWT
5. **Отправка welcome email:** (по желанию)
   - Mock реализация или реальная
   - Ссылка для подтверждения email

---

## Задание 3. JWT аутентификация

### Настройка JWT в appsettings.json
Добавьте секцию конфигурации:
- Secret — секретный ключ (минимум 32 символа)
- Issuer — издатель токена
- Audience — получатель токена
- AccessTokenExpirationMinutes — срок жизни access token (15-30)
- RefreshTokenExpirationDays — срок жизни refresh token (7-30)

### Endpoint POST /api/auth/login

**DTO для входа (LoginDTO):**
- EmailOrUsername — поле для email или username
- Password

### Логика входа
1. **Поиск пользователя:**
   - По email ИЛИ по username
   - Загрузить с ролями и claims
2. **Проверка пароля:**
   - Verify через BCrypt или PBKDF2 или Argon2
3. **Проверки:**
   - IsActive == true
   - IsDeleted == false
   - IsEmailConfirmed == true (опционально)
4. **Генерация Access Token:**
   - Claims: UserId, Email, Username, FirstName, LastName
   - Roles: все роли пользователя
   - Custom claims из UserClaim таблицы
   - Expiry: 15-30 минут
   - Signing с JWT Secret
5. **Генерация Refresh Token:**
   - Криптографически стойкий random string
   - Сохранение хеша в UserSession
   - Expiry: 7-30 дней
6. **Обновление LastLoginAt**
7. **Возврат LoginResponseDTO:**
   - AccessToken
   - RefreshToken
   - AccessTokenExpiry
   - RefreshTokenExpiry
   - User

### Endpoint POST /api/auth/refresh
Обновление истёкшего access token:

**DTO (RefreshTokenDTO):**
- AccessToken (истёкший)
- RefreshToken

**Логика:**
1. Извлечь UserId из истёкшего access token (без валидации expiry)
2. Найти пользователя в БД
3. Проверить сохранённый RefreshToken в UserSession
4. Проверить срок действия RefreshToken
5. Сгенерировать новую пару токенов
6. **Ротация:** старый RefreshToken становится недействительным
7. Сохранить новый RefreshToken в БД
8. Вернуть новые токены

### Endpoint POST /api/auth/logout
Выход из системы:
- Получить UserId из текущего пользователя
- Отозвать все сессии пользователя (IsRevoked = true)
- Вернуть 200 OK

### Endpoint POST /api/auth/revoke
Отзыв конкретного refresh token:
- То же что и logout для конкретной сессии

---

## Задание 4. Дополнительные auth endpoints

### POST /api/auth/forgot-password
Запрос на восстановление пароля:

**DTO (ForgotPasswordDTO):**
- Email

**Логика:**
1. Найти пользователя по email
2. Сгенерировать reset token (secure random или JWT)
3. Сохранить token и expiry в БД (или закодировать в JWT)
4. Отправить email со ссылкой для сброса
5. Вернуть success (не раскрывать существование email)

### POST /api/auth/reset-password
Сброс пароля по токену:

**DTO (ResetPasswordDTO):**
- Token
- NewPassword
- ConfirmNewPassword

**Логика:**
1. Валидировать token
2. Проверить expiry
3. Валидировать новый пароль
4. Хешировать новый пароль
5. Сохранить в БД
6. Отозвать все refresh токены пользователя
7. Вернуть success

### POST /api/auth/change-password
Смена пароля авторизованным пользователем:

**DTO (ChangePasswordDTO):**
- CurrentPassword
- NewPassword
- ConfirmNewPassword

**Требования:**
- `[Authorize]` — только для авторизованных
- Проверка текущего пароля
- Валидация нового пароля
- Хеширование и сохранение

### POST /api/auth/confirm-email
Подтверждение email:

**Query параметры:**
- userId
- token

**Логика:**
1. Найти пользователя
2. Проверить token
3. Установить IsEmailConfirmed = true
4. Вернуть success или redirect

### GET /api/auth/profile
Получение профиля текущего пользователя:
- `[Authorize]`
- Получить UserId из claims
- Загрузить пользователя с ролями
- Вернуть UserResponseDTO
---


## Задание 6. Многоуровневая авторизация

### Role-based Authorization
Используйте атрибут `[Authorize(Roles = "...")]`:

**Примеры применения:**
- `[Authorize(Roles = "Admin")]` — только админы
- `[Authorize(Roles = "Admin,Moderator")]` — админы ИЛИ модераторы
- На уровне контроллера — для всех actions
- На уровне action — для конкретного метода

**Защитите endpoints:**
- Admin панель — только Admin
- Модерация контента — Admin, Moderator
- Личные данные — User, Admin
- Статистика — Admin

### Policy-based Authorization
Создайте минимум **5 policies**:

**Примеры policies:**

**RequireAdminRole:**
- RequireRole("Admin")

**RequireModeratorOrAdmin:**
- RequireRole("Admin", "Moderator")

**RequireEmailConfirmed:**
- RequireClaim("EmailConfirmed", "True")

**RequirePremiumSubscription:**
- RequireClaim("SubscriptionLevel", "Premium", "Enterprise")

**RequireMinimumAge:**
- Custom requirement с проверкой DateOfBirth

**Can{Action}{Entity}:**
- CanEditPost, CanDeleteUser, CanViewReports
- Проверка через permissions в БД

### Custom Authorization Requirements
Создайте минимум 1-2 custom requirements:

**IAuthorizationRequirement:**
- Пустой интерфейс-маркер

**AuthorizationHandler<TRequirement>:**
- HandleRequirementAsync метод
- Проверка условий
- context.Succeed(requirement) при успехе

**Примеры:**
- MinimumAgeRequirement — проверка возраста
- PermissionRequirement — проверка permission в БД
- ResourceOwnerRequirement — проверка владения ресурсом

**Сценарии:**
- Пользователь может редактировать только свои посты
- Пользователь может удалять только свои комментарии
- Админ может редактировать всё

---

## Задание 7. Swagger с JWT

### Настройка SecurityDefinition
Добавьте Bearer token authentication в Swagger:
- SecurityScheme — Bearer
- BearerFormat — JWT
- Scheme — bearer
- In — Header
- Name — Authorization

### SecurityRequirement
Добавьте глобальное требование токена для защищённых endpoints.

### Тестирование
- Кнопка "Authorize" должна появиться в Swagger UI
- Введите токен в формате: `Bearer {token}`
- Протестируйте защищённые endpoints
- Проверьте ошибку 401 без токена

### Документация endpoints
Добавьте XML комментарии:
- Описание каждого auth endpoint
- Примеры запросов и ответов
- Описание ошибок (401, 403, 400)

---
### CORS настройка
Настройте CORS для конкретных origins:
- Не используйте AllowAnyOrigin в production
- Укажите конкретные allowed origins
- Allowed methods: GET, POST, PUT, DELETE, PATCH
- Allowed headers: Authorization, Content-Type
- AllowCredentials если нужны cookies

## Задание 9. Логирование безопасности

### Таблица SecurityAuditLog
Создайте таблицу для логов безопасности:
- Id — Guid
- EventType — enum (Login, Logout, Register, PasswordChange, RoleChange, etc.)
- UserId — Guid (nullable)
- Email — string (если UserId null)
- IpAddress — string
- UserAgent — string
- Success — bool
- Details — string (JSON)
- Timestamp — DateTime

### События для логирования
Логируйте следующие события:

**Аутентификация:**
- Успешный вход (Login)
- Неудачный вход (FailedLogin) с причиной
- Выход (Logout)
- Обновление токена (TokenRefresh)

**Регистрация:**
- Успешная регистрация (Register)
- Подтверждение email (EmailConfirmed)

**Управление паролями:**
- Запрос сброса пароля (PasswordResetRequested)
- Сброс пароля (PasswordReset)
- Смена пароля (PasswordChanged)

**Управление пользователями:**
- Создание пользователя админом (UserCreated)
- Обновление пользователя (UserUpdated)
- Удаление пользователя (UserDeleted)
- Блокировка/разблокировка (UserBlocked, UserUnblocked)

**Управление ролями:**
- Назначение роли (RoleAssigned)
- Отзыв роли (RoleRevoked)

**Подозрительная активность:**
- Множественные неудачные попытки входа
- Попытка доступа к чужим ресурсам
- Использование истёкшего токена
- Необычный IP адрес или User-Agent

### Реализация логирования
- Создайте AuditService
- Inject в auth контроллеры и сервисы
- Асинхронное сохранение (не блокировать основной поток)
- Background job для очистки старых логов

### Endpoints для просмотра логов (админ)
- `GET /api/admin/audit` — все логи с пагинацией и фильтрами
- `GET /api/admin/audit/user/{userId}` — логи пользователя
- `GET /api/admin/audit/suspicious` — подозрительная активность

---

## Задание 11. Обновление существующих endpoints

### Защита существующих контроллеров
Добавьте `[Authorize]` на:
- Все контроллеры требующие аутентификации
- Оставьте публичными только:
  - GET endpoints для чтения (опционально)
  - Auth endpoints

### Проверка прав доступа к ресурсам
Реализуйте проверки:
- Пользователь может редактировать только свои посты
- Пользователь может удалять только свои комментарии
- Админ/модератор может редактировать всё

Используйте:
- Resource-based authorization
- Проверку UserId == resource.UserId
- ИЛИ HasRole("Admin")

### Фильтрация данных по роли
В GET endpoints:
- Админ видит все записи
- Модератор видит все + может фильтровать
- Обычный пользователь видит только свои

Реализуйте через:
- Where(x => x.UserId == currentUserId) для User
- Без фильтра для Admin

### Добавление UserId в создаваемые сущности
При создании через POST:
- Получить UserId из Claims
- Установить в entity.CreatedBy или entity.UserId
- Нельзя подменить через DTO

---

## Задание 12. Unit-тесты 

### Используемые инструменты
- xUnit или NUnit
- Moq для мокирования
- FluentAssertions
- Microsoft.AspNetCore.Mvc.Testing (опционально для integration тестов)
