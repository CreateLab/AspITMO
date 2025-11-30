# Лабораторная работа №4. Аутентификация, авторизация и работа с файлами

## Цель работы
Реализовать полноценную систему аутентификации на основе JWT токенов, многоуровневую авторизацию с ролями и политиками, а также систему загрузки и обработки файлов с поддержкой streaming.

---

## Задание 1. Система пользователей

### Модель User
Расширьте модель пользователя следующими полями:
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
- Соответствие принципам нормализации БД

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

### Конфигурируемая политика паролей

**Настройка в appsettings.json:**
```json
{
  "PasswordPolicy": {
    "MinLength": 8,
    "RequireUppercase": true,
    "RequireLowercase": true,
    "RequireDigit": true,
    "RequireSpecialCharacter": true,
    "AllowedSpecialCharacters": "!@#$%^&*()_+-=[]{}|;:'\",.<>?/~",
    "MaxPasswordAgeInDays": 90
  }
}
Валидация должна использовать конфигурируемые правила:

Password:

NotEmpty
Minimum length из конфигурации
Проверка требований к символам на основе настроек
Проверка допустимых специальных символов


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
   - Используйте BCrypt.Net-Next или PBKDF2
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
5. **Отправка welcome email:**
   - Mock реализация или реальная
   - Ссылка для подтверждения email
6. **Возврат ответа:**
   - Созданный пользователь (без пароля)
   - Сообщение об успешной регистрации

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
   - Verify через BCrypt или PBKDF2
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
   - Сохранение в User.RefreshToken
   - Expiry: 7-30 дней
6. **Обновление LastLoginAt**
7. **Возврат LoginResponseDTO:**
   - AccessToken
   - RefreshToken
   - AccessTokenExpiry
   - RefreshTokenExpiry
   - User (без пароля)

### Endpoint POST /api/auth/refresh
Обновление истёкшего access token:

**DTO (RefreshTokenDTO):**
- AccessToken (истёкший)
- RefreshToken

**Логика:**
1. Извлечь UserId из истёкшего access token (без валидации expiry)
2. Найти пользователя в БД
3. Проверить сохранённый RefreshToken
4. Проверить срок действия RefreshToken
5. Сгенерировать новую пару токенов
6. **Ротация:** старый RefreshToken становится недействительным
7. Сохранить новый RefreshToken в БД
8. Вернуть новые токены

### Endpoint POST /api/auth/logout
Выход из системы:
- Получить UserId из текущего пользователя
- Обнулить RefreshToken в БД
- Установить RefreshTokenExpiryTime в прошлое
- Вернуть 200 OK

### Endpoint POST /api/auth/revoke
Отзыв конкретного refresh token:
- То же что и logout
- Опционально: можно добавить blacklist для access токенов (сложно)

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

### PUT /api/auth/profile
Обновление профиля:

**DTO (UpdateProfileDTO):**
- FirstName
- LastName
- DateOfBirth
- PhoneNumber

**Требования:**
- `[Authorize]`
- Валидация данных
- Обновить только разрешённые поля
- Нельзя менять Email, Username, Password через этот endpoint

---

## Задание 5. JWT Middleware настройка

### Конфигурация в Program.cs
Настройте Authentication:
- DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme
- DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme

### TokenValidationParameters
Настройте параметры валидации:
- ValidateIssuer = true
- ValidateAudience = true
- ValidateLifetime = true
- ValidateIssuerSigningKey = true
- ValidIssuer — из конфигурации
- ValidAudience — из конфигурации
- IssuerSigningKey — из Secret
- ClockSkew — минимальное (например, TimeSpan.Zero)

### JwtBearerEvents
Настройте события:

**OnAuthenticationFailed:**
- Логировать ошибки валидации
- Добавить header "Token-Expired" если токен истёк
- Логировать попытки с невалидными токенами

**OnTokenValidated:**
- Дополнительная валидация (опционально)
- Проверка IsActive пользователя из БД
- Логирование успешной аутентификации

**OnMessageReceived:**
- Извлечение токена из custom headers (опционально)

### Middleware Pipeline
Добавьте в правильном порядке:
1. app.UseAuthentication();
2. app.UseAuthorization();

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

### Claims-based Authorization
Добавьте custom claims при генерации токена:
- SubscriptionLevel (Free, Premium, Enterprise)
- Department (IT, Sales, HR)
- CanEditPost, CanDeleteUser (из permissions)

Используйте в контроллерах через User.Claims.

### Policy-based Authorization
Создайте минимум **5 policies** в Program.cs:

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

### Resource-based Authorization
Используйте IAuthorizationService для проверки прав на конкретный ресурс:

**Сценарии:**
- Пользователь может редактировать только свои посты
- Пользователь может удалять только свои комментарии
- Админ может редактировать всё

**Реализация:**
1. Inject IAuthorizationService в контроллер
2. Загрузить ресурс (пост, комментарий)
3. Вызвать AuthorizeAsync с resource и policy
4. Проверить result.Succeeded
5. Вернуть 403 если forbidden

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

## Задание 8. Защита от атак

### Rate Limiting для auth endpoints
Ограничьте частоту запросов:

**Лимиты:**
- `/api/auth/login` — 5 попыток в минуту на IP
- `/api/auth/register` — 3 попытки в минуту на IP
- `/api/auth/forgot-password` — 3 попытки в час на IP

**Реализация:**
- Используйте AspNetCoreRateLimit пакет
- ИЛИ собственная реализация с кешем
- Возврат 429 Too Many Requests
- Retry-After header

### CORS настройка
Настройте CORS для конкретных origins:
- Не используйте AllowAnyOrigin в production
- Укажите конкретные allowed origins
- Allowed methods: GET, POST, PUT, DELETE, PATCH
- Allowed headers: Authorization, Content-Type
- AllowCredentials если нужны cookies

### Блокировка после неудачных попыток
Реализуйте защиту от brute force:

**Модель LoginAttempt:**
- UserId или Email
- IpAddress
- Success
- Timestamp

**Логика:**
1. При неудачной попытке входа — сохранить в таблицу
2. Подсчитать неудачные попытки за последние 15 минут
3. Если >= 5 попыток:
   - Заблокировать пользователя (IsActive = false)
   - ИЛИ временная блокировка (LockoutEnd)
4. Отправить уведомление пользователю
5. Вернуть понятное сообщение об ошибке

**Разблокировка:**
- Автоматическая через N минут
- Админ может разблокировать вручную
- Пользователь через forgot-password

### Валидация и санитизация
- Используйте FluentValidation для всех входных данных
- Escape HTML в текстовых полях (если отображаются)
- Проверяйте длину всех строк
- Проверяйте допустимые символы

---

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

## Задание 10. Admin панель endpoints

Создайте endpoints для управления пользователями (только для админов):

### GET /api/admin/users
Список всех пользователей:
- `[Authorize(Roles = "Admin")]`
- Пагинация (page, pageSize)
- Фильтрация: isActive, isDeleted, role
- Сортировка: createdAt, lastLoginAt, email
- Поиск по email, username, name
- Возврат UserListDTO с ролями

### GET /api/admin/users/{id}
Детальная информация о пользователе:
- `[Authorize(Roles = "Admin")]`
- Загрузить с ролями, claims, permissions
- История изменений (опционально)
- Статистика активности

### PUT /api/admin/users/{id}/roles
Управление ролями пользователя:
- `[Authorize(Roles = "Admin")]`
- DTO: список RoleIds
- Удалить все текущие роли
- Добавить новые
- Логирование изменений
- Обнулить refresh токены (пересоздать сессию)

### POST /api/admin/users/{id}/ban
Блокировка пользователя:
- `[Authorize(Roles = "Admin")]`
- DTO: Reason, Duration (permanent или временная)
- Установить IsActive = false
- Обнулить refresh токены
- Логирование
- Отправить уведомление пользователю

### POST /api/admin/users/{id}/unban
Разблокировка:
- `[Authorize(Roles = "Admin")]`
- Установить IsActive = true
- Логирование

### DELETE /api/admin/users/{id}
Удаление пользователя:
- `[Authorize(Roles = "Admin")]`
- Soft delete (IsDeleted = true)
- Обнулить токены
- Логирование
- Каскадное удаление связанных данных (или архивация)

### GET /api/admin/statistics
Статистика системы:
- `[Authorize(Roles = "Admin")]`
- Общее количество пользователей
- Активные пользователи (за последние 30 дней)
- Количество по ролям
- Регистрации за период
- Неудачные попытки входа
- Заблокированные пользователи

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

## Задание 12. Система загрузки файлов

### Модель FileMetadata
Создайте таблицу для метаданных файлов:
- `Id` — Guid
- `FileName` — string, имя файла в хранилище
- `OriginalFileName` — string, оригинальное имя
- `ContentType` — string, MIME type
- `Size` — long, размер в байтах
- `UploadedBy` — Guid, FK к User
- `UploadedAt` — DateTime
- `Path` — string, относительный путь
- `Hash` — string, SHA256 хеш для дедупликации
- `IsPublic` — bool
- `ExpiresAt` — DateTime (nullable), для временных файлов
- `DownloadCount` — int

### POST /api/files/upload
Загрузка одного файла:

**Параметры:**
- `IFormFile file` — файл из multipart/form-data
- `bool isPublic` — публичный доступ
- `DateTime? expiresAt` — время истечения

**Валидация файла:**
1. **Размер:** максимум 50MB (настраиваемый)
2. **Тип файла (whitelist):**
   - Изображения: .jpg, .jpeg, .png, .gif, .webp
   - Документы: .pdf, .doc, .docx, .xls, .xlsx
   - Другие разрешённые типы для вашего проекта
3. **Magic bytes проверка:**
   - Прочитать первые байты файла
   - Проверить соответствие ContentType
   - Защита от подделки расширения
4. **Имя файла:**
   - Проверка на path traversal (../, ..\)
   - Запрещённые символы
   - Длина имени

**Обработка файла:**
1. Генерация безопасного имени:
   - Guid.NewGuid() + оригинальное расширение
   - Сохранение оригинального имени в БД
2. Структурированное хранение:
   - По годам/месяцам/дням: 2025/01/15/
   - ИЛИ по UserId: users/{userId}/
3. Вычисление SHA256 хеша
4. Проверка дубликатов по хешу (опционально)
5. Сохранение на диск
6. Создание записи в БД
7. Для изображений: генерация thumbnails (см. задание 15)

**Авторизация:**
- `[Authorize]` — только авторизованные
- UploadedBy = текущий UserId

**Возврат:**
- FileMetadataDTO с Id, OriginalFileName, Size, Url

### POST /api/files/upload-multiple
Загрузка нескольких файлов:

**Параметры:**
- `List<IFormFile> files`

**Обработка:**
- Валидация каждого файла
- Сохранение всех файлов
- Возврат списка FileMetadataDTO
- Если один файл не прошёл валидацию — отклонить все (транзакция)

**Ограничения:**
- Максимум 10 файлов за раз
- Общий размер не более 100MB

### GET /api/files
Список файлов пользователя:
- `[Authorize]`
- Пагинация
- Фильтр по ContentType (изображения, документы)
- Сортировка по дате загрузки
- Поиск по OriginalFileName
- Админ видит все файлы

### GET /api/files/{id}
Скачивание файла:

**Проверки:**
1. Файл существует
2. Не истёк (ExpiresAt)
3. Права доступа:
   - IsPublic == true — всем
   - UploadedBy == currentUser — владельцу
   - Admin — всем

**Возврат:**
- File() с содержимым
- Правильный Content-Type
- Content-Disposition: attachment или inline
- Имя файла из OriginalFileName

**Дополнительно:**
- Инкремент DownloadCount
- Логирование скачивания

### GET /api/files/{id}/info
Метаданные файла без скачивания:
- Возврат FileMetadataDTO
- Проверка прав доступа

### DELETE /api/files/{id}
Удаление файла:

**Проверки:**
- Владелец (UploadedBy == currentUser)
- ИЛИ Admin

**Действия:**
1. Удалить файл с диска
2. Удалить thumbnails (если есть)
3. Удалить запись из БД
4. ИЛИ soft delete (IsDeleted = true)

---

## Задание 13. Streaming больших файлов

### GET /api/files/{id}/stream
Потоковая отдача файла:

**Отличие от обычного скачивания:**
- Не загружает весь файл в память
- Отправляет chunks по мере чтения с диска
- Поддерживает Range requests

**Реализация:**
- Используйте FileStream
- Установите BufferSize (например, 4KB)
- Копируйте в Response.Body асинхронно

**Range Requests поддержка:**
1. Проверить наличие Range header
2. Парсить диапазон (bytes=0-1023)
3. Установить статус 206 Partial Content
4. Установить headers:
   - Accept-Ranges: bytes
   - Content-Range: bytes 0-1023/total
   - Content-Length: размер диапазона
5. Отправить только запрошенный диапазон

**Применение:**
- Для файлов >10MB обязательно используйте streaming
- Видео и аудио (для поддержки seek)
- Докачка прерванных загрузок

### POST /api/files/upload/chunked
Chunk upload для очень больших файлов (>10MB):

**Параметры:**
- `IFormFile chunk` — часть файла
- `string uploadId` — уникальный ID загрузки
- `int chunkIndex` — номер части
- `int totalChunks` — общее количество частей
- `string fileName` — оригинальное имя

**Логика:**
1. Сохранить chunk во временную папку
2. Записать метаданные о прогрессе
3. Когда все chunks загружены:
   - Собрать файл из частей
   - Валидировать полный файл
   - Переместить в постоянное хранилище
   - Очистить временные файлы
4. Возврат прогресса: загружено chunks / totalChunks

**Возврат:**
- При неполной загрузке: прогресс
- При завершении: FileMetadataDTO

---

## Задание 14. Работа с изображениями

### Автоматическая генерация thumbnails
При загрузке изображения создавайте превью:

**Размеры:**
- Small: 200x200 (для списков, аватары)
- Medium: 800x600 (для галерей)
- ИЛИ другие размеры для вашего проекта

**Реализация:**
Используйте библиотеку (выбрать одну):
- SixLabors.ImageSharp (рекомендуется)
- System.Drawing.Common (только Windows)
- SkiaSharp

**Процесс:**
1. Проверить что файл — изображение
2. Загрузить в Image объект
3. Resize с сохранением пропорций:
   - Fit — вписать в размер
   - Crop — обрезать до размера
   - Pad — добавить поля
4. Оптимизация качества (JPEG quality 85-90%)
5. Сохранить рядом с оригиналом:
   - original.jpg
   - original_small.jpg
   - original_medium.jpg
6. Сохранить пути в БД (или генерировать динамически)

### GET /api/files/{id}/thumbnail
Получение превью изображения:

**Query параметры:**
- `size` — small, medium (или конкретные размеры)

**Логика:**
1. Проверить что файл — изображение
2. Проверить права доступа
3. Проверить существование thumbnail
4. Если нет — сгенерировать на лету (и сохранить)
5. Вернуть thumbnail файл

**Cache headers:**
- Cache-Control: public, max-age=31536000
- ETag для кеширования на клиенте

### Оптимизация изображений
При загрузке оптимизируйте оригинал:
- Уменьшение качества JPEG (90%)
- Удаление EXIF данных (для приватности)
- Конвертация в WebP (опционально)
- Удаление метаданных

### Определение размеров
Сохраняйте в FileMetadata:
- Width — ширина в пикселях
- Height — высота в пикселях

Используется для:
- Отображение на фронтенде без загрузки
- Валидация размеров (мин/макс)

### EXIF данные (опционально)
Извлечение и сохранение:
- DateTaken — дата съёмки
- CameraModel — модель камеры
- Location — GPS координаты
- Orientation — ориентация

**Важно:** удалите перед публикацией если IsPublic!

### Поддержка форматов
Обрабатывайте форматы:
- JPEG (.jpg, .jpeg)
- PNG (.png)
- GIF (.gif) — без анимации для thumbnails
- WebP (.webp)

---

## Задание 15. Интеграция файлов с основным функционалом

### Прикрепление файлов к сущностям
Создайте связи с вашими основными сущностями.

**Примеры:**

**Для чата — MessageAttachment:**
- MessageId — FK к Message
- FileId — FK к FileMetadata
- AttachedAt — DateTime

**Для соцсети — PostAttachment:**
- PostId — FK к Post
- FileId — FK к FileMetadata
- Order — int (порядок отображения)

**Для магазина — ProductImage:**
- ProductId — FK к Product
- FileId — FK к FileMetadata
- IsMain — bool (главное фото)
- Order — int

### Endpoints с файлами
Расширьте существующие endpoints:

**POST /api/messages** (для чата):**
- Добавьте `List<Guid> attachmentIds` в DTO
- Проверьте что файлы существуют
- Проверьте что файлы принадлежат пользователю
- Создайте MessageAttachment записи

**POST /api/posts** (для соцсети):**
- `List<Guid> imageIds`
- Проверки
- Создание PostAttachment

**GET /api/messages/{id}:**
- Include attachments
- В DTO добавить список FileMetadataDTO

### Галерея изображений
Для сущностей с изображениями:

**GET /api/products/{id}/images:**
- Все изображения продукта
- Отсортированные по Order
- С URL для thumbnail и полного размера

**POST /api/products/{id}/images:**
- Загрузка и привязка изображения
- Установка Order

**DELETE /api/products/{id}/images/{imageId}:**
- Отвязка изображения
- Опционально: удаление файла

### Preview в API ответах
В ResponseDTO добавьте:
- `ThumbnailUrl` — для быстрого отображения
- `FileUrl` — для полного файла
- `FileSize` — для отображения размера
- `FileType` — для иконки

### Каскадное удаление
При удалении родительской сущности:

**Вариант А — удалить файлы:**
- Установите OnDelete(DeleteBehavior.Cascade)
- В обработчике удаления сущности удалите физические файлы

**Вариант B — архивировать:**
- Soft delete для FileMetadata
- Периодическая очистка старых файлов (background job)

---

## Задание 16. Авторизация для файлов

### Проверка прав доступа
Реализуйте логику в FileService:

**Для скачивания:**
1. Загрузить FileMetadata
2. Если IsPublic — доступ всем
3. Если !IsPublic:
   - Владелец (UploadedBy == currentUserId)
   - ИЛИ Admin
   - ИЛИ есть связь с доступной сущностью

**Пример:**
- Файл приложен к сообщению в приватном канале
- Проверить членство пользователя в канале
- Разрешить доступ участникам

### Policy для файлов
Создайте policy "CanAccessFile":
- Custom FileAccessRequirement
- FileAccessHandler проверяет:
  - IsPublic
  - Владелец
  - Роль Admin
  - Связанные сущности

### Временные ссылки (опционально)
Для публичного доступа без авторизации:

**POST /api/files/{id}/public-link:**
- Генерация secure token
- Сохранение в таблице PublicFileLink:
  - FileId, Token, ExpiresAt, MaxDownloads
- Возврат URL: /api/files/public/{token}

**GET /api/files/public/{token}:**
- Проверка токена
- Проверка expiry
- Проверка лимита скачиваний
- Отдача файла
- Инкремент счётчика

**DELETE /api/files/{id}/public-link:**
- Отзыв ссылки

---

## Задание 17. Unit-тесты (60% покрытия)

### Тесты для Auth сервисов
**Регистрация:**
- Успешная регистрация
- Дубликат email
- Дубликат username
- Невалидный пароль
- Невалидный email

**Вход:**
- Успешный вход
- Неверный пароль
- Несуществующий пользователь
- Неактивный пользователь
- Генерация токенов

**Refresh Token:**
- Успешное обновление
- Невалидный refresh token
- Истёкший refresh token
- Ротация токенов

### Тесты для Authorization handlers
**Policy handlers:**
- Успешная авторизация
- Отказ при отсутствии роли
- Отказ при отсутствии claim
- Custom requirements

**Resource-based:**
- Владелец может редактировать
- Не-владелец не может
- Админ может всё

### Тесты для валидаторов паролей
**Password validator:**
- Все правила валидации
- Минимальная длина
- Заглавные буквы
- Цифры
- Специальные символы

**Email validator:**
- Формат email
- Уникальность (async)

### Тесты для JWT генерации
**Token generation:**
- Корректные claims
- Корректный expiry
- Signature валидация

### Тесты для File сервиса
**Upload:**
- Успешная загрузка
- Превышение размера
- Запрещённый тип файла
- Path traversal попытка
- Magic bytes несоответствие

**Download:**
- Успешное скачивание
- Проверка прав доступа
- Несуществующий файл
- Истёкший файл

**Thumbnail generation:**
- Генерация всех размеров
- Для изображений
- Ошибка для неизображений

### Тесты streaming логики
- Range requests
- Partial content
- Chunk upload

### Используемые инструменты
- xUnit или NUnit
- Moq для мокирования
- FluentAssertions
- Microsoft.AspNetCore.Mvc.Testing (опционально для integration тестов)
