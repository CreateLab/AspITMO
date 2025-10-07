# Лабораторная работа №2. Переход на ASP.NET Core и файловое хранилище

## Цель
Мигрировать проект с самописного TCP-сервера на ASP.NET Core Web API и реализовать полноценное файловое хранилище данных с JSON-сериализацией.

---

## Задание

### 1. Создать ASP.NET Core Web API проект с правильной архитектурой

**Выберите один из вариантов архитектуры:**

**Вариант А: N-Layer Architecture**
```
YourProject.API/          — Controllers, Program.cs, Middleware
YourProject.Core/         — Models, DTOs, Interfaces
YourProject.Services/     — Business logic, Service implementations
YourProject.Infrastructure/ — Repositories, File storage
YourProject.Tests/        — Unit tests
```

**Вариант B: Clean Architecture**
```
YourProject.API/          — Controllers, Program.cs, Configuration
YourProject.Application/  — Services, DTOs, Interfaces, Validators
YourProject.Domain/       — Entities, Domain logic
YourProject.Infrastructure/ — Repositories, External services
YourProject.Tests/        — Unit tests
```

Выберите архитектуру, которая больше подходит для вашего проекта. Главное — чёткое разделение ответственности между слоями.

---

2. Реализовать файловое хранилище данных
Создать базовый класс FileStorageRepository<T> для работы с JSON-файлами:

GetAllAsync() — чтение всех записей из файла
GetByIdAsync(id) — поиск по ID
CreateAsync(entity) — добавление новой записи
UpdateAsync(entity) — обновление существующей записи
DeleteAsync(id) — удаление записи
SaveChangesAsync() — сохранение изменений в файл
FindAsync(predicate) — поиск по условию

Требования:

использовать System.Text.Json для сериализации
реализовать thread-safe операции чтения/записи (SemaphoreSlim)
обеспечить атомарность операций записи (запись во временный файл + переименование)
обработка ошибок при работе с файлами (пусть исключения пробрасываются наверх)


### 3. Создать минимум 4 связанные модели

**Примеры для разных типов проектов:**
- **Чат**: User, Channel, Message, UserChannelMembership, MessageReaction
- **Игра**: User, Game, Player, GameMove, GameState, GameResult
- **Банк**: User, Account, Transaction, Card, TransactionCategory
- **Магазин**: User, Product, Order, OrderItem, Category, Review
- **Соцсеть**: User, Post, Comment, Like, Friendship, Notification
- **Библиотека**: User, Book, Author, Loan, Reservation, Review
- **Фитнес**: User, Workout, Exercise, WorkoutLog, Goal, Achievement

**Каждая модель должна иметь:**
- уникальный ID (Guid)
- timestamp создания (CreatedAt)
- timestamp обновления (UpdatedAt)
- связи с другими моделями (через ID или коллекции ID)
- бизнес-логику в виде методов (например, IsExpired(), CanBeDeleted())

---

### 4. Реализовать Repository Pattern
Создать специализированные репозитории для каждой модели, наследующиеся от базового FileStorageRepository<T> или использующие композицию:

UserRepository : FileStorageRepository<User> с дополнительными методами:

FindByEmailAsync(email)
FindByUsernameAsync(username)
ExistsByEmailAsync(email)


MessageRepository : FileStorageRepository<Message>:

GetByChannelIdAsync(channelId, page, pageSize)
GetUnreadAsync(userId)
MarkAsReadAsync(messageId)


И т.д. для каждой сущности с её специфичными методами

### 5. Создать полноценный RESTful API для всех моделей

**Базовые endpoints:**
- `GET /api/[resource]` — список с query parameters:
  - `?page=1&pageSize=10` — пагинация
  - `?sortBy=createdAt&sortOrder=desc` — сортировка
  - `?search=query` — поиск
  - специфичные фильтры для вашего проекта
- `GET /api/[resource]/{id}` — получение по ID с связанными данными
- `POST /api/[resource]` — создание с валидацией
- `PUT /api/[resource]/{id}` — полное обновление
- `PATCH /api/[resource]/{id}` — частичное обновление (JsonPatchDocument)
- `DELETE /api/[resource]/{id}` — удаление с проверками

**Дополнительные endpoints для бизнес-логики:**
- `GET /api/channels/{id}/messages` — получение сообщений канала
- `POST /api/orders/{id}/cancel` — отмена заказа
- `GET /api/users/{id}/statistics` — статистика пользователя
- и т.д. в зависимости от проекта

---

### 6. Реализовать DTOs и маппинг

**Отдельные DTO для разных сценариев:**
- `CreateDTO` — для создания (только необходимые поля)
- `UpdateDTO` — для обновления (nullable поля)
- `ResponseDTO` — для ответа (с вложенными данными)
- `ListItemDTO` — для списков (упрощённая версия)

**AutoMapper для маппинга:**
- настройка profiles для каждого типа
- custom resolvers для сложных преобразований
- reverse mapping там где нужно
- вложенные DTO для представления связей

---

### 7. Добавить FluentValidation

**Валидаторы для всех Create/Update DTO:**
- Стандартные правила: NotEmpty, NotNull, Length, EmailAddress, Matches (regex)
- Кастомные правила валидации:
  - проверка уникальности (email, username)
  - проверка существования связанных сущностей
  - бизнес-правила (например, дата не в прошлом)
- Валидация на уровне коллекций
- Асинхронная валидация с обращением к репозиториям
- Понятные сообщения об ошибках

---

### 8. Настроить Dependency Injection

**Регистрация всех сервисов с правильными lifetime:**
- `AddScoped` — для репозиториев, UnitOfWork, сервисов с состоянием
- `AddTransient` — для валидаторов, легковесных сервисов
- `AddSingleton` — для конфигурации, кеша

**Options Pattern для конфигурации:**
- путь к файлам хранилища
- настройки пагинации
- другие конфигурационные параметры

**Использование extension methods** для чистоты Program.cs:
- `services.AddRepositories();`
- `services.AddServices();`
- `services.AddValidators();`

---

### 9. Настроить Swagger с расширенной документацией

- Установка пакетов: `Swashbuckle.AspNetCore`, `Swashbuckle.AspNetCore.Annotations`
- XML-комментарии:
  - включение генерации XML файла в .csproj
  - добавление комментариев ко всем контроллерам, методам, моделям
- Настройка Swagger UI:
  - заголовок, описание, версия API
  - группировка endpoints по тегам
  - примеры запросов и ответов
- Аннотации для описания кодов ответов (200, 201, 400, 404, 500)

---

### 10. Реализовать расширенную функциональность API

**Поиск:**
- полнотекстовый поиск по нескольким полям
- поиск с partial matching
- case-insensitive поиск

**Фильтрация:**
- фильтры по датам (диапазоны: from, to)
- фильтры по enum значениям
- фильтры по boolean полям
- множественный выбор (например, `?categories=1,2,3`)

**Сортировка:**
- сортировка по любому полю
- множественная сортировка
- default сортировка

**Пагинация:**
- Page-based: `?page=1&pageSize=10`
- метаданные в ответе: items, pageNumber, pageSize, totalCount, totalPages, hasNextPage, hasPreviousPage

**Bulk операции:**
- массовое создание
- массовое обновление
- массовое удаление
- возврат результатов по каждому элементу (успех/ошибка)

---

### 11. Написать unit-тесты (60% покрытия)

**Тесты для репозиториев:**
- CRUD операции
- сложные запросы с фильтрацией
- thread-safety операций
- обработка ошибок файловой системы

**Тесты для сервисов:**
- бизнес-логика
- валидация данных
- обработка edge cases

**Тесты для валидаторов:**
- все правила валидации
- кастомные правила
- асинхронная валидация

**Используемые инструменты:**
- xUnit или NUnit
- Moq для мокирования зависимостей
- FluentAssertions для читаемых проверок
- AutoFixture для генерации тестовых данных (опционально)
