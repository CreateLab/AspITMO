# Лабораторная работа №5 (дополнительная). Real-time или gRPC

## Цель работы
Изучить продвинутые технологии коммуникации: реализовать либо real-time взаимодействие через SignalR, либо высокопроизводительное API через gRPC. Лабораторная работа выполняется на дополнительные баллы.

---

## Выбор технологии

Выберите **ОДИН** из двух вариантов для реализации:
- **Вариант A:** SignalR для Real-time коммуникации
- **Вариант B:** gRPC для высокопроизводительного API

**Нельзя реализовывать оба варианта** — выберите тот, который больше подходит для вашего проекта.

---

# Вариант A: SignalR для Real-time коммуникации

## Теоретическая часть

### Что такое SignalR?
SignalR — это библиотека для добавления real-time функциональности в приложения. Позволяет серверу отправлять обновления клиентам мгновенно, без polling.

### Транспорты
SignalR автоматически выбирает лучший транспорт:
1. WebSockets (предпочтительный)
2. Server-Sent Events
3. Long Polling (fallback)

### Hubs
Hub — это класс на сервере, через который клиенты вызывают методы и получают уведомления.

### Группы
Механизм для broadcast сообщений определённому набору клиентов (каналы, комнаты).

---

## Задание A1. Настройка SignalR

### Установка пакетов
Установите:
- `Microsoft.AspNetCore.SignalR`

### Настройка в Program.cs
Добавьте сервисы и middleware:
- `builder.Services.AddSignalR()`
- `app.MapHub<YourHub>("/hubname")`

### CORS для SignalR
Настройте CORS:
- AllowCredentials() — обязательно для SignalR
- WithOrigins() — конкретные origins клиентов
- AllowAnyHeader, AllowAnyMethod

### Настройка reconnection
Настройте параметры:
- KeepAliveInterval — интервал keep-alive пакетов
- ClientTimeoutInterval — таймаут отключения клиента
- HandshakeTimeout — таймаут для handshake

### Логирование SignalR
Настройте логирование для:
- OnConnectedAsync события
- OnDisconnectedAsync события
- Ошибки в Hub методах
- Broadcast операции

---

## Задание A2. Создание базового Hub

### Класс Hub
Создайте Hub класс, наследующийся от `Hub` или `Hub<T>`:
- Определите публичные методы для вызова клиентами
- Используйте async методы

### Типизированный Hub (Strongly-typed)
Создайте интерфейс клиента:
- Определите методы, которые сервер может вызывать на клиенте
- Используйте `Hub<IYourClient>`
- Обеспечивает type-safety

### Авторизация Hub
Защитите Hub:
- Атрибут `[Authorize]` на классе Hub
- ИЛИ на конкретных методах
- Доступ к User.Identity в Hub методах

### Context информация
Используйте Context для получения:
- ConnectionId — уникальный ID соединения
- User — информация о пользователе (если авторизован)
- Items — хранилище данных соединения

### OnConnectedAsync
Переопределите метод:
- Логирование подключения
- Добавление в группы автоматически
- Уведомление других клиентов
- Инициализация состояния соединения

### OnDisconnectedAsync
Переопределите метод:
- Логирование отключения
- Удаление из групп
- Очистка ресурсов
- Уведомление других клиентов

---

## Задание A3. Работа с группами

### Добавление в группы
Реализуйте методы:
- `JoinGroup(string groupName)` — присоединиться к группе
- Используйте `Groups.AddToGroupAsync(Context.ConnectionId, groupName)`
- Проверка прав доступа к группе
- Уведомление участников группы

### Удаление из групп
Реализуйте методы:
- `LeaveGroup(string groupName)` — покинуть группу
- `Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName)`
- Уведомление участников

### Broadcasting в группу
Отправка сообщений всем участникам:
- `Clients.Group(groupName).SendAsync("MethodName", data)`
- Исключение отправителя: `Clients.OthersInGroup(groupName)`

### Список участников группы
Реализуйте хранение участников:
- Таблица GroupMembership в БД: GroupId, ConnectionId, UserId, JoinedAt
- ИЛИ в памяти: `ConcurrentDictionary<string, HashSet<string>>`
- Метод `GetGroupMembers(string groupName)` — список пользователей

### Приватные группы
Реализуйте проверку доступа:
- Проверка что пользователь имеет право присоединиться
- Инвайты в приватные группы
- Роли в группах (admin, moderator, member)

### Dynamic groups
Автоматическое создание групп:
- При первом присоединении
- Автоматическое удаление пустых групп

---

## Задание A4. Real-time функциональность (в зависимости от проекта)

Реализуйте минимум **5-7 real-time фич** специфичных для вашего проекта.

### Для чата:

**Отправка сообщений:**
- Hub метод `SendMessage(string channelId, string content)`
- Сохранение в БД
- Broadcast в группу канала
- Возврат MessageDTO с timestamp и ID

**Typing indicators:**
- Hub метод `UserTyping(string channelId)`
- Broadcast другим участникам (кроме отправителя)
- Timeout для автоматического сброса (3-5 секунд)

**Онлайн статус:**
- Автоматическое обновление при подключении/отключении
- Список онлайн пользователей в канале
- Broadcast изменений статуса

**Read receipts:**
- Hub метод `MarkAsRead(string messageId)`
- Обновление в БД
- Broadcast отправителю сообщения

**Реакции на сообщения:**
- Hub метод `AddReaction(string messageId, string emoji)`
- Сохранение в БД
- Broadcast всем участникам

**Редактирование/удаление:**
- Hub методы `EditMessage(id, newContent)` и `DeleteMessage(id)`
- Проверка прав (только автор)
- Broadcast изменений

### Для игры:

**Обновление состояния игры:**
- Hub метод `MakeMove(string gameId, moveData)`
- Валидация хода
- Обновление GameState в БД
- Broadcast всем игрокам

**Ходы в реальном времени:**
- Очередь ходов
- Таймер на ход
- Автоматический пропуск при таймауте

**Система таймеров:**
- Countdown для хода
- Broadcast оставшегося времени
- Background timer в Hub

**Очки и лидерборд:**
- Обновление очков игроков
- Real-time обновление таблицы лидеров
- Broadcast при изменении позиций

**Игровые события:**
- Победа/поражение
- Специальные события (bonus, power-up)
- Достижения

**Lobby система:**
- Создание/присоединение к игре
- Список ожидающих игроков
- Автоматический старт при достаточном количестве

### Для банка:

**Уведомления о транзакциях:**
- Real-time уведомление при входящем платеже
- Детали транзакции
- Push в группу пользователя

**Изменения баланса:**
- Broadcast обновлённого баланса
- При любых операциях со счётом

**Алерты безопасности:**
- Подозрительная активность
- Вход с нового устройства
- Крупные транзакции

### Для магазина:

**Уведомления о статусе заказа:**
- Изменение статуса (обработка, отправка, доставка)
- Real-time обновление для покупателя

**Изменение наличия товаров:**
- Broadcast когда товар появился в наличии
- Для пользователей в wishlist

**Flash sales:**
- Countdown таймеры
- Обновление количества товара
- Уведомления о начале/окончании

### Универсальные функции:

**Система уведомлений:**
- Hub метод для отправки уведомления конкретному пользователю
- Broadcast в группу пользователя
- Типы: info, warning, error, success

**Live updates данных:**
- Broadcast изменений любых сущностей
- Подписка на конкретные типы обновлений

---

## Задание A5. Система уведомлений через SignalR

### Модель Notification в БД
Расширьте модель:
- Id, UserId, Type, Title, Message, Link
- IsRead, CreatedAt, ReadAt
- Priority (Low, Normal, High, Urgent)
- Category (System, User, Security, etc.)

### Создание уведомлений
Сервис для создания:
- Метод `CreateNotificationAsync(userId, type, title, message)`
- Сохранение в БД
- Отправка через SignalR

### Hub метод для отправки
В NotificationHub:
- Определите группы по userId: `"user_{userId}"`
- При подключении добавляйте в группу
- Метод `SendNotificationToUser(userId, notification)`
- Используйте `Clients.Group($"user_{userId}")`

### Endpoints для уведомлений
**GET /api/notifications:**
- `[Authorize]`
- Список уведомлений текущего пользователя
- Пагинация
- Фильтр: isRead, type, category
- Сортировка по дате

**GET /api/notifications/unread-count:**
- Количество непрочитанных
- Для badge

**PUT /api/notifications/{id}/read:**
- Пометить как прочитанное
- Установить ReadAt
- Broadcast обновление счётчика

**POST /api/notifications/mark-all-read:**
- Пометить все как прочитанные
- Broadcast обновление

**DELETE /api/notifications/{id}:**
- Удаление уведомления

### Client события
Определите методы для клиента:
- `ReceiveNotification(notification)` — новое уведомление
- `NotificationRead(notificationId)` — прочитано
- `UnreadCountUpdated(count)` — обновление счётчика

### Badge с количеством
- Real-time обновление badge
- При подключении отправить текущий count
- При каждом изменении broadcast новый count

### Группировка уведомлений
- По типу/категории
- Складывание похожих (5 новых сообщений вместо 5 отдельных)

---

## Задание A6. Интеграция SignalR с существующим функционалом

### Уведомления при изменениях
Добавьте SignalR уведомления в существующие операции:

**При создании сущности:**
- Пост создан → уведомить подписчиков
- Заказ создан → уведомить администратора
- Сообщение отправлено → уведомить получателя

**При обновлении:**
- Статус заказа изменён → уведомить покупателя
- Комментарий получил ответ → уведомить автора
- Пост отредактирован → уведомить читателей

**При удалении:**
- Пост удалён модератором → уведомить автора

### Inject IHubContext в сервисы
Используйте IHubContext для отправки из сервисов:
- `IHubContext<YourHub, IYourClient>`
- Вызов клиентских методов из сервисов
- Без создания экземпляра Hub

### Domain Events интеграция
При возникновении Domain Event:
- Обработчик события отправляет SignalR уведомление
- Broadcast изменений всем заинтересованным

### Background jobs интеграция
При выполнении фоновых задач:
- Прогресс выполнения через SignalR
- Уведомление о завершении
- Ошибки в real-time

---

## Задание A7. Масштабирование SignalR

### Redis Backplane
Для нескольких инстансов приложения:

**Установка пакета:**
- `Microsoft.AspNetCore.SignalR.StackExchangeRedis`

**Настройка:**
- `services.AddSignalR().AddStackExchangeRedis("connection-string")`
- Все инстансы подключены к одному Redis
- Сообщения реплицируются между серверами

**Когда нужно:**
- Horizontal scaling с load balancer
- Несколько серверов обрабатывают WebSocket соединения
- Broadcast должен доходить до клиентов на всех серверах

### Sticky Sessions
Альтернатива Redis:
- Настройка load balancer для sticky sessions
- Один клиент всегда подключается к одному серверу
- Проще но менее гибко

### Connection management
Хранение активных соединений:
- Таблица ActiveConnections: UserId, ConnectionId, ServerInstance, ConnectedAt
- Очистка при disconnect
- Для отправки уведомлений конкретному пользователю

---

## Задание A8. Unit-тесты для SignalR (60%)

### Тесты Hub методов
Моки для тестирования:
- Mock IHubCallerClients
- Mock IGroupManager
- Mock HubCallerContext

**Тесты:**
- Вызов методов Hub
- Broadcast сообщений
- Добавление/удаление из групп
- Проверка вызовов клиентских методов

### Тесты authorization
- Доступ авторизованных пользователей
- Отказ неавторизованным
- Проверка ролей

### Тесты бизнес-логики
- Валидация входных данных
- Обработка ошибок
- Проверка сохранения в БД (mock repository)

### Integration тесты (опционально)
- Используйте TestServer
- Подключение SignalR клиента
- Отправка и получение сообщений
- Проверка real-time обновлений



# Вариант B: gRPC для высокопроизводительного API

## Теоретическая часть

### Что такое gRPC?
gRPC — это высокопроизводительный RPC (Remote Procedure Call) фреймворк от Google. Использует Protocol Buffers для сериализации и HTTP/2 для транспорта.

### Преимущества gRPC
- Высокая производительность (бинарный протокол)
- Строгая типизация (contract-first)
- Поддержка streaming (server, client, bidirectional)
- Генерация клиентского кода
- Кроссплатформенность

### Protocol Buffers
Язык описания данных (.proto файлы):
- Определение сообщений (messages)
- Определение сервисов (services)
- Генерация кода для разных языков

### Типы вызовов
1. **Unary** — один запрос, один ответ
2. **Server streaming** — один запрос, поток ответов
3. **Client streaming** — поток запросов, один ответ
4. **Bidirectional streaming** — поток запросов и ответов

---

## Задание B1. Настройка gRPC

### Установка пакетов
Установите:
- `Grpc.AspNetCore`
- `Grpc.Tools` (для генерации кода из .proto)
- `Google.Protobuf` (runtime для Protocol Buffers)

### Создание .proto файлов
Создайте папку `Protos/` в проекте.

**Базовая структура .proto файла:**
- `syntax = "proto3";`
- package название
- C# namespace
- Определение сервисов
- Определение сообщений

### Настройка в .csproj
Добавьте ItemGroup:
- Protobuf Include для каждого .proto файла
- GrpcServices="Server" (или Client, или Both)

### Настройка в Program.cs
Добавьте:
- `builder.Services.AddGrpc()`
- `app.MapGrpcService<YourService>()`

### gRPC-Web (опционально)
Для браузерных клиентов:
- `services.AddGrpc().AddGrpcWeb()`
- `app.UseGrpcWeb()`
- `endpoints.MapGrpcService<YourService>().EnableGrpcWeb()`

---

## Задание B2. Создание gRPC сервисов

Создайте минимум **3-4 gRPC сервиса** для основных операций вашего проекта.

### Определение сервисов в .proto

**Unary вызовы (основные CRUD):**
```
service UserService {
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
  rpc UpdateUser (UpdateUserRequest) returns (UserResponse);
  rpc DeleteUser (DeleteUserRequest) returns (DeleteUserResponse);
}
```

**Определение сообщений:**
```
message GetUserRequest {
  string user_id = 1;
}

message UserResponse {
  string id = 1;
  string email = 2;
  string username = 3;
  string first_name = 4;
  string last_name = 5;
  google.protobuf.Timestamp created_at = 6;
}
```

### Реализация сервисов в C#
Создайте класс, наследующийся от сгенерированного базового класса:
- Переопределите методы из .proto
- Inject зависимости (репозитории, сервисы)
- Реализуйте бизнес-логику
- Маппинг между domain моделями и protobuf сообщениями

### Обработка ошибок
Используйте RpcException:
- StatusCode.NotFound — сущность не найдена
- StatusCode.InvalidArgument — невалидные данные
- StatusCode.PermissionDenied — нет прав
- StatusCode.Unauthenticated — не авторизован
- Detail и Metadata для дополнительной информации

### Валидация входных данных
- Проверка обязательных полей
- Проверка форматов (email, etc.)
- Бизнес-правила валидации
- Выброс RpcException при ошибке

---

## Задание B3. Server Streaming

Реализуйте минимум **2 server streaming** метода.

### Определение в .proto
```
service DataService {
  rpc StreamData (StreamRequest) returns (stream DataResponse);
}
```

### Примеры использования

**Для чата — поток сообщений:**
```
rpc StreamMessages (StreamMessagesRequest) returns (stream Message);
```
- Клиент подписывается на канал
- Сервер отправляет каждое новое сообщение
- Поток открыт пока клиент подключен

**Для игры — поток состояний:**
```
rpc StreamGameUpdates (GameIdRequest) returns (stream GameState);
```
- Обновления состояния игры в реальном времени
- Ходы игроков
- События игры

**Для любого проекта — уведомления:**
```
rpc StreamNotifications (Empty) returns (stream Notification);
```
- Поток уведомлений для пользователя
- Push уведомления

**Экспорт данных большого объёма:**
```
rpc ExportUsers (ExportRequest) returns (stream UserData);
```
- Постраничная выгрузка
- Избежание OutOfMemory

### Реализация на сервере
- Метод принимает request и IServerStreamWriter
- Цикл отправки: `await responseStream.WriteAsync(data)`
- Проверка `context.CancellationToken.IsCancellationRequested`
- Завершение при disconnect клиента

### Управление подписками
- Хранение активных streams
- Уведомление всех подписчиков при событии
- Очистка при disconnect

---

## Задание B4. Client Streaming

Реализуйте минимум **1 client streaming** метод.

### Определение в .proto
```
service UploadService {
  rpc UploadFile (stream FileChunk) returns (UploadResponse);
}
```

### Примеры использования

**Загрузка файла по частям:**
```
message FileChunk {
  bytes content = 1;
  string filename = 2;
  int32 chunk_number = 3;
}

message UploadResponse {
  string file_id = 1;
  int64 total_size = 2;
}
```
- Клиент отправляет файл chunks
- Сервер собирает и сохраняет
- Возвращает результат

**Batch операции:**
```
rpc BatchCreateUsers (stream CreateUserRequest) returns (BatchResponse);
```
- Создание множества пользователей
- Клиент стримит запросы
- Сервер обрабатывает batch

**Метрики/логи от клиента:**
```
rpc SendMetrics (stream Metric) returns (MetricsResponse);
```
- Клиент отправляет метрики
- Сервер агрегирует

### Реализация на сервере
- Метод принимает IAsyncStreamReader
- Цикл чтения: `await requestStream.MoveNext()`
- `requestStream.Current` для получения данных
- Обработка каждого chunk
- Возврат результата после завершения stream

---

## Задание B5. Bidirectional Streaming

Реализуйте минимум **1 bidirectional streaming** метод.

### Определение в .proto
```
service ChatService {
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}
```

### Примеры использования

**Чат в реальном времени:**
- Клиент отправляет сообщения (stream)
- Сервер отправляет сообщения всех участников (stream)
- Двусторонняя коммуникация

**Игровая сессия:**
```
rpc GameSession (stream PlayerAction) returns (stream GameEvent);
```
- Игрок отправляет действия
- Сервер отправляет события игры

**Live trading/auction:**
```
rpc LiveAuction (stream Bid) returns (stream AuctionUpdate);
```

### Реализация на сервере
- Метод принимает IAsyncStreamReader и IServerStreamWriter
- Два параллельных потока:
  - Чтение входящих сообщений
  - Отправка исходящих
- Использование Task.Run для асинхронности
- Координация через CancellationToken

### Управление соединениями
- Хранение активных streams
- Broadcast сообщений всем участникам
- Очистка при disconnect

---

## Задание B6. Авторизация в gRPC

### JWT авторизация
Настройте JWT для gRPC:
- Та же конфигурация как для REST API
- gRPC автоматически использует auth middleware

### Атрибуты авторизации
Используйте:
- `[Authorize]` на сервисах или методах
- `[Authorize(Roles = "...")]` для ролей
- `[Authorize(Policy = "...")]` для policies

### Получение User информации
В методах сервиса:
- Доступ через `ServerCallContext.GetHttpContext()`
- User claims из `httpContext.User`
- Проверка прав

### Перехватчики (Interceptors)
Создайте AuthInterceptor:
- Наследование от Interceptor
- Переопределение методов (UnaryServerHandler, etc.)
- Проверка токена
- Логирование auth событий
- Регистрация: `services.AddGrpc(o => o.Interceptors.Add<AuthInterceptor>())`

### Передача токена от клиента
- Metadata в запросе: `Authorization: Bearer {token}`
- Автоматическое извлечение middleware

---

## Задание B7. Interceptors и Middleware

Создайте минимум **3 interceptora** для cross-cutting concerns.

### LoggingInterceptor
Логирование всех gRPC вызовов:
- Имя метода
- Параметры (осторожно с sensitive data)
- Время выполнения
- Результат или ошибка
- ClientIP, User

### ExceptionInterceptor
Глобальная обработка ошибок:
- Перехват всех исключений
- Преобразование в RpcException
- Логирование с контекстом
- Различные коды для разных исключений

### ValidationInterceptor
Валидация входных данных:
- Проверка required полей
- Бизнес-валидация
- FluentValidation интеграция
- RpcException при ошибке

### PerformanceInterceptor
Мониторинг производительности:
- Измерение времени выполнения
- Логирование медленных запросов (>1 сек)
- Метрики производительности
- Алерты при деградации

### Регистрация Interceptors
Порядок имеет значение:
- Logging → Exception → Validation → Auth → Your logic

---

## Задание B8. gRPC vs REST сравнение

### Дублирование функциональности
Реализуйте одни и те же операции в REST и gRPC:
- Минимум 5 операций
- Одинаковая бизнес-логика
- Разные контракты (OpenAPI vs Protobuf)

### Бенчмарк тесты
Напишите тесты производительности:
- Используйте BenchmarkDotNet
- Сравните время выполнения
- Сравните размер payload
- Сравните throughput

**Тестовые сценарии:**
- Простой GET запрос
- Создание сущности (POST)
- Получение списка с 100 элементами
- Streaming vs пагинация
- Загрузка файла

### Документация результатов
В README.md:
- Таблица сравнения производительности
- Графики (опционально)
- Выводы о применимости
- Рекомендации когда использовать что

---

## Задание B9. gRPC клиент

### Создание .NET клиента
Создайте консольное приложение или отдельный проект:
- Добавьте .proto файлы как Client
- Сгенерируйте клиентский код
- Создайте GrpcChannel
- Используйте сгенерированный клиент

### Примеры вызовов
Реализуйте примеры для:
- Unary вызовы
- Server streaming (чтение потока)
- Client streaming (отправка потока)
- Bidirectional streaming

### Обработка ошибок на клиенте
- Try-catch для RpcException
- Проверка StatusCode
- Retry logic с Polly (опционально)

### Конфигурация клиента
- Timeout настройки
- Retry policies
- Deadline для операций

---

## Задание B10. Unit-тесты для gRPC (60%)

### Тесты сервисов
Моки для тестирования:
- Mock ServerCallContext
- Mock зависимостей (repositories)

**Тесты:**
- Unary вызовы (успех и ошибки)
- Валидация входных данных
- Бизнес-логика
- Авторизация

### Тесты streaming
**Server streaming:**
- Отправка нескольких элементов
- Завершение stream
- Отмена (cancellation)

**Client streaming:**
- Получение всех элементов
- Обработка каждого
- Возврат результата

**Bidirectional:**
- Параллельное чтение и запись
- Синхронизация

### Тесты Interceptors
- Логирование вызовов
- Обработка исключений
- Валидация данных
- Модификация контекста

### Integration тесты (опционально)
- Использование TestServer
- Реальные gRPC вызовы
- End-to-end сценарии

---
