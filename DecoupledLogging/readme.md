# Теневое логирование через события - полное разъединение бизнес-логики и логирования

# Теневое логирование через события: полный decoupling бизнес-логики

Привет, коллеги! Как senior .NET developer с опытом построения распределённых систем, хочу поделиться подходом к логированию, который радикально упрощает архитектуру и усиливает SOLID. Рассмотрим пример из репозитория [abaula/MixedCode](https://github.com/abaula/MixedCode), где показан decoupled logging через события C#.

## Ключевые принципы подхода

Класс **не занимается логированием напрямую**, а генерирует события для ключевых операций: `OrderCreated`, `OrderFailed`, `PaymentProcessed`. Бизнес-логика остаётся чистой, без зависимостей от `ILogger` или любых логгеров.

Решение **"как именно писать в лог и какие события слушать"** выносится на уровень приложения — в `Program.cs` или DI-контейнер. Там настраивается обработчик, который подписывается на события и пишет структурированные логи.

## Усиление SRP и SOLID

Этот подход **максимально выражает принцип единственной ответственности (SRP)**:
- Бизнес-класс отвечает ТОЛЬКО за доменную логику
- Логирование — отдельная обязанность внешнего обработчика

DIP соблюдается идеально — класс не зависит от абстракций логирования. OCP тоже выигрывает: новые способы обработки событий (метрики, аудит, уведомления) добавляются без изменения класса.

## Проблемы традиционных wrapper'ов

Decorator/wrapper для логирования удобен, но **навязывает правила использования**:
- Клиентский код должен работать через wrapper
- Рефакторинг усложняется
- Появляется дублирование и проблемы с наследованием

Такой подход делает логирование **не "теневым"** — потребители косвенно знают о нём.

## Теневое логирование через события

**Суть**: использование методов класса остаётся неизменным независимо от логирования. В `Program.cs` создаётся инстанс класса, и сразу подписываются обработчики:

```csharp
var orderService = new OrderService();
orderService.OrderCreated += (s, e) =>
    logger.LogInformation("Order {OrderId} created", e.Order.Id);
```

Класс остаётся **прозрачным** для логирования — события генерируются "про себя".

## Хитрая регистрация с ConditionalWeakTable

В примере реализован **специальный класс-подписчик**, регистрируемый в DI-контейнере через `ConditionalWeakTable<TClass, ILogger<TClass>>`.

**Как это работает**:
```
var loggerTable = new ConditionalWeakTable<OrderService, ILogger<OrderService>>();
orderService.OrderCreated += (s, e) => {
    if (loggerTable.TryGetValue((OrderService)s, out var logger))
        logger.LogInformation("Order created: {OrderId}", e.Order.Id);
};
```

**Ключевые преимущества**:
- Логгер **автоматически привязывается** к конкретному инстансу класса
- **Слабые ссылки** гарантируют корректный lifetime: логгер живёт не дольше класса
- Работает с transient/scoped сервисами без утечек памяти
- **Централизованная подписка** в DI-контейнере

## Плюсы подхода

- **Полный decoupling**: класс testable без моков логгеров
- **Мультиназначенность событий**: логи + метрики + аудит одним махом
- **SOLID на максимум**: SRP/DIP/OCP в идеале
- **Теневое подключение**: API класса не меняется
- **Автоматический lifetime** через WeakTable
- **DI-friendly**: прозрачная интеграция с Microsoft.Extensions.DependencyInjection

## Минусы подхода

- **Overhead событий**: аллокации в высоконагруженных системах (решается async void обработчиками)
- **Сложность отладки**: логи не в стек-трейсе бизнес-метода
- **Магия WeakTable**: добавляет уровень абстракции, усложняет понимание
- **Performance lookup**: небольшой overhead поиска в таблице
- **.NET-специфичность**: WeakTable есть не везде
- **Verbose подписки**: без extension methods может быть много boilerplate

## Когда использовать

**Идеально для**:
- Библиотек и доменных моделей
- Микросервисов с множественными потребителями
- Систем, где логирование/аудит/метрики настраиваются по-разному в разных окружениях

**Менее подходит для**:
- Простых CRUD-приложений (overengineering)
- Критически горячих путей (overhead событий)

## Итоговая архитектура в DI

```csharp
// Регистрация в Program.cs
services.AddScoped<ILoggerTable>(provider =>
    new LoggerTable(new ConditionalWeakTableFactory()));
services.AddScoped<OrderService>();

// Фабрика обработчиков
services.AddTransient<EventLogger<OrderService>>();
```

Такой подход даёт **максимальную гибкость** при минимальных компромиссах. Рекомендую для серьёзных проектов, где архитектура важнее сиюминутной простоты.

Пишите в комментариях ваш опыт с подобными паттернами! 🚀


# Вынос логирования из класса: Decoupled Logging

## Что такое decoupled logging?

Decoupled logging (разделённое логирование) — архитектурный подход, при котором бизнес-классы **не содержат прямых вызовов логгеров** (типа `ILogger.LogInformation`). Вместо этого они:
- Генерируют **события** (`EventHandler<OrderCreatedEventArgs>`)
- Или используют **внешние обработчики** через DI/события/синглтоны

Логирование настраивается **централизованно** на уровне приложения, без загрязнения доменной логики. Классический пример — события C# с `ConditionalWeakTable` для lifetime-менеджмента, как в [MixedCode](https://github.com/abaula/MixedCode). [stackoverflow](https://stackoverflow.com/questions/550785/c-events-or-an-observer-interface-pros-cons)

## За: Аргументы сторонников

**Сторонники** (DDD-энтузиасты, библиотечные разработчики) подчёркивают:
- **Чистота SRP**: Класс фокусируется на бизнес-логике, логи — внешняя забота. Усиливает SOLID (DIP, OCP). [blog.stackademic](https://blog.stackademic.com/the-hidden-power-of-c-events-how-to-build-decoupled-scalable-systems-like-a-pro-8a62d166d3a2)
- **Тестируемость**: Нет моков `ILogger`, классы работают standalone. [reddit](https://www.reddit.com/r/csharp/comments/swvyvs/inject_logger_vs_raising_events_in_console_app/)
- **Гибкость**: Одно событие → логи + метрики + аудит. Легко менять логгеры (Serilog → NLog). [mol-tech](https://www.mol-tech.us/blog/events-and-delegates-csharp-dotnet-development)
- **Decoupling для библиотек**: Потребитель сам решает, логировать ли. Нет жёстких зависимостей. [stackoverflow](https://stackoverflow.com/questions/1020967/using-delegates-or-interfaces-to-decouple-the-logging-best-practices-c-sharp)
- **Масштабируемость**: События async, не блокируют горячие пути.

**Типичный комментарий**: "Логирование — cross-cutting concern, не для домена!" [reddit](https://www.reddit.com/r/csharp/comments/swvyvs/inject_logger_vs_raising_events_in_console_app/)

## Против: Аргументы оппонентов

**Противники** (прагматики, enterprise-разрабы) видят overengineering:
- **Производительность**: События + WeakTable = аллокации, GC-pressure, overhead lookup. В hot paths критично. [jacksondunstan](https://www.jacksondunstan.com/articles/3621)
- **Сложность**: Магия событий усложняет дебаг (стек-трейс размыт), отладку, понимание кода. [mol-tech](https://www.mol-tech.us/blog/events-and-delegates-csharp-dotnet-development)
- **Memory leaks**: Неправильная отписка/WeakRef → утечки. Требует экспертизы. [jacksondunstan](https://www.jacksondunstan.com/articles/3621)
- **Boilerplate**: Подписки, фабрики, таблицы — verbose код для простых случаев. [stackoverflow](https://stackoverflow.com/questions/1020967/using-delegates-or-interfaces-to-decouple-the-logging-best-practices-c-sharp)
- **Простота важнее**: `ILogger` via DI — mature, zero-cost абстракция. Зачем изобретать велосипед? [stackoverflow](https://stackoverflow.com/questions/5646820/logger-wrapper-best-practice)

**Типичный комментарий**: "Для консольных apps/монолитов — inject ILogger и не парьтесь. Моки просты!" [reddit](https://www.reddit.com/r/csharp/comments/swvyvs/inject_logger_vs_raising_events_in_console_app/)

## Сравнение подходов

| Аспект          | Decoupled (события)                  | Coupled (ILogger в классе)          |
|-----------------|--------------------------------------|-------------------------------------|
| **SRP/SOLID**  | Высокий (полный decoupling)         | Средний (cross-cutting в домене)   |
| **Perf**       | Overhead событий (~10-20% в hot)    | Минимальный                        |
| **Тестирование**| Без моков                          | Моки ILogger (просто)              |
| **Гибкость**   | Максимум (много обработчиков)       | Зависит от DI-конфига              |
| **Сложность**  | Высокая (магия, WeakTable)          | Низкая (стандарт MS.Extensions)    |
| **Использование**| Библиотеки, DDD, микросервисы     | CRUD, монолиты, простые apps       |

Данные из дебатов на SO/Reddit. [stackoverflow](https://stackoverflow.com/questions/550785/c-events-or-an-observer-interface-pros-cons)

## Когда выбирать decoupled?

- **Да**: Библиотеки, shared домены, сложные события (логи+метрики).
- **Нет**: Простые apps, perf-critical код, команды новичков.

**Рекомендация**: Начните с `ILogger` DI. Переходите на события, если нужны мульти-обработчики или полная независимость. Баланс — ключ к хорошей архитектуре! 🚀


