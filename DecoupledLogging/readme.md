# Теневое логирование через события - полное разъединение бизнес-логики и логирования в DI среде

Привет, коллеги!

Хочу поделиться подходом к логированию, который радикально упрощает архитектуру и усиливает SOLID.

Сделал пример кода [GitHub](https://github.com/abaula/MixedCode/blob/master/DecoupledLogging/src/Program.cs), чтобы показать как работает теневое логирование (shadow decoupled logging) через события C#.

## Что такое decoupled logging?

Decoupled logging (разделённое логирование) - архитектурный подход, при котором бизнес-классы **не содержат прямых вызовов логгеров** (типа `ILogger.LogInformation`). Вместо этого:
- Бизнес-классы генерируют **события**.
- Обработкой событий занимаются **специализированные классы-логеры**.
- Бизнес-классы ничего не знают о логгерах.
- Логирование настраивается **централизованно** на уровне приложения, без загрязнения доменной логики.

## "За" и "против" decoupled logging

Существуют мнения "за" и "против" decoupled logging.

Типичные аргументы "за":

- **Чистота SRP**: класс фокусируется на бизнес-логике, логи - внешняя забота.
- **Тестируемость**: нет моков `ILogger`, классы работают standalone.
- **Гибкость и масштабируемость**: одно событие можно обрабатывать всем нужными способами: логи, метрики, аудит. Легко менять логику обработки событий и библиотеки логирования.
- **Decoupling для библиотек**: потребитель классов бизнес-логики сам решает, логировать события и какие именно. Нет жёстких зависимостей.
- **Производительность**: бизнес-класс не форматирует данные для лога. При реализации обработчиков событий применяй мозг и быстрое кэширование входящих событий с последующей пост-обработкой - не блокирует бизнес логику.

Возражения критиков сводятся к простому тезису - не усложняйте, если этого действительно не требуется - "inject ILogger и не парьтесь". Это мнение звучит разумно, я согласен с такой критикой - если у вас простое приложение, то не усложняйте его.

## Вынос логирования из класса

Самый простой способ разделить бизнес-логику и логирование, это написать Wrapper для бизнес-класса.

Decorator/wrapper для логирования удобен, но **навязывает правила использования**:
- Клиентский код должен работать через wrapper.
- Рефакторинг усложняется.
- Появляется дублирование и проблемы с наследованием.

Такой подход делает логирование **не "теневым"** - потребители косвенно знают о нём.

## Полное разделение

Полное разделение бизнес-логики и логирования возможно только при использовании двух независимых объектов: бизнес-класс, обработчик логов.

Простой пример:

```csharp
class OrderServiceLogger
{
    public OrderServiceLogger(FileLogger logger, OrderService orderService)
    {
        orderService.OrderCreated += (s, e) => logger.LogInformation($"Order {e.OrderId} created.");
    }
}

var orderService = new OrderService();
var fileLogger = new FileLogger();
var orderServiceLogger = new OrderServiceLogger(fileLogger, orderService);
orderService.CreateOrder(...);
```

Этот подход очевиден, но если приложение построено с использованием DI контейнера, то подход требует адаптации.

## Уводим логирование в тень

DI контейнеры выполняют 2 важных задачи:
- Фабрика объектов
- Управление временем жизни объектов

С задачей создания объектов всё просто, DI контейнер вернёт нам 2 готовых объекта, при чём обработчик логов получит при создании экземпляр бизнес-класса и экземпляр логгера.

```csharp
var services = new ServiceCollection();
services.AddScoped<FileLogger>();
services.AddScoped<OrderServiceLogger>();
services.AddScoped<OrderService>();
var serviceProvider = services.BuildServiceProvider();

var orderService = serviceProvider.GetRequiredService<OrderService>();
var orderServiceLogger = serviceProvider.GetRequiredService<OrderServiceLogger>();
```

Проблема заключается в том, что теперь нам нужно управлять временем жизни обработчика логов `OrderServiceLogger`, т.е. явно хранить ссылку на созданный объект и синхронизировать его время жизни с временем жизни экземпляра бизнес-класса `OrderService`.

Если больше ничего не делать, то нам придётся явно создавать новый экземпляр `OrderServiceLogger` везде, где мы создаём экземпляр `OrderService`, и следить чтобы время их жизни совпадало - это совсем не то поведение, которое нам нужно.

**Нам нужно:**
- Использовать в бизнес логике только экземпляры объектов бизнес логики, в нашем примере `OrderService`.
- Бизнес логика ничего не должна знать об объектах выполняющих иные задачи в рамках приложения, в нашем примере это логирование через объект `OrderServiceLogger`.
- При создании объекта бизнес логики, приложение должно гарантированно обеспечивать все реализованные для него сервисные функции - если для `OrderService` реализован `OrderServiceLogger`, то он должен вовремя создаватся и обрабатывать события.
- Корректная работа сервисных функций включает оптимальное управление ресурсами приложения - экземпляр `OrderServiceLogger` должен удалятся из памяти после уничтожения связанного с ним объекта `OrderService`.

Эти требования легко реализовать, даже в рамках DI контейнера. С созданием объектов мы уже разобрались, осталось реализовать функционал синхронизации их времени жизни, в этом нам помогут weak reference.

Нам нужно обеспечить, чтобы созданный объект `OrderServiceLogger` жил не меньше чем экземпляр `OrderService` и удалялся когда он больше не нужен.

Для этого нам требуется некий объект уровня приложения, который:
- хранит ссылки на оба зависимых объекта.
- следит за временем их жизни.
- удалит `OrderServiceLogger` как только `OrderService` будет удалён.

Мы можем сами реализовать такой класс, в котором есть **ключевой объект** и **зависимые объекты**. Архитектура такого класса проста:
- ключевой объект (или несколько) хранится в виде слабых ссылок (weak reference), которые не блокируют объект от сборки мусора.
- зависимые объекты хранятся в виде сильных ссылок, которые не дают сборщику мусора уничтожить их.
- периодически проверяется состояние ключевых объектов - если они удалены, то удаляются и зависимые объекты.

Для простого случая, можно использовать класс [ConditionalWeakTable<TKey,TValue>](https://learn.microsoft.com/ru-ru/dotnet/api/system.runtime.compilerservices.conditionalweaktable-2?view=net-8.0) из пространства имен `System.Runtime.CompilerServices`, который уже реализует такую логику.

## Пишем логику для DI

Реализуем метод расширения для `ServiceCollection`, и рассмотрим как он работает.

```csharp
public static class ServiceCollectionExtensions
{
    public static ServiceCollection AddScopedWithLogger<TService, TServiceInstance, TServiceLogger>(
        this ServiceCollection services)
        where TService : class
        where TServiceInstance : class, TService
        where TServiceLogger : class
    {
        // Регистрация TServiceInstance.
        services.AddScoped<TServiceInstance>();
        // Регистрация TServiceLogger.
        services.AddScoped<TServiceLogger>();
        // Регистрация TService.
        services.AddScoped<TService>(sp =>
        {
            var instance = sp.GetRequiredService<TServiceInstance>();
            var logger = sp.GetRequiredService<TServiceLogger>();
            var conditionalWeakTable = sp.GetRequiredService<ConditionalWeakTable<object, object>>();
            // Помещаем instance и logger в ConditionalWeakTable.
            conditionalWeakTable.Add(instance, logger);

            return instance;
        });

        return services;
    }
}

```

Метод `AddScopedWithLogger` выполняет всю необходимую работу:
- регистрирует все типы в DI.
- реализует логику создания и увязывания бизнес-класса и класса обработчика его событий.

**Важно** - в DI контейнере необходимо разделить логику создания самого объекта бизнес-класса от логики создания экземпляра со всеми его теневыми объектами. Для этого лучше всего использовать контракты бизнес-классов (интерфейсы).


```csharp
public class OrderEventArgs : EventArgs
{
    public int OrderId { get; set; }
}

public interface IOrderService
{
    event EventHandler<OrderEventArgs> OrderCreated;
    void CreateOrder(int id, string customer);
}
```

Таким образом DI контейнер передаст в конструктор `OrderServiceLogger` экземпляр `OrderService`.

**В бизнес логике необходимо использовать только контракты**, что является рекомендуемым подходом.

```csharp
class OrderManager : IOrderManager
{
    public OrderManager(IOrderService orderService)
    {
        ...
    }
}
```

Корректно регистрируем все типы в контейнере:

```csharp
var services = new ServiceCollection();
services.AddScopedWithLogger<IOrderService, OrderService, OrderServiceLogger>();
```

Теперь создание объекта для его контракта `IOrderService` приведёт к вызову следующего кода из метода расширения

```csharp
...
// Регистрация TService.
services.AddScoped<TService>(sp =>
{
    var instance = sp.GetRequiredService<TServiceInstance>();
    var logger = sp.GetRequiredService<TServiceLogger>();
    var conditionalWeakTable = sp.GetRequiredService<ConditionalWeakTable<object, object>>();
    // Помещаем instance и logger в ConditionalWeakTable.
    conditionalWeakTable.Add(instance, logger);

    return instance;
});
...
```

Сделаю расшифровку для сочетания параметров `IOrderService, OrderService, OrderServiceLogger`.

```csharp
services.AddScoped<IOrderService>(sp =>
{
    var instance = sp.GetRequiredService<OrderService>();
    var logger = sp.GetRequiredService<OrderServiceLogger>();
    var conditionalWeakTable = sp.GetRequiredService<ConditionalWeakTable<object, object>>();
    // Помещаем instance и logger в ConditionalWeakTable.
    conditionalWeakTable.Add(instance, logger);

    return instance;
});
```

Как видите, всё просто. Создаются объекты `OrderService`, `OrderServiceLogger` со всеми зависимостями,
далее оба объекта сохраняются в таблице `ConditionalWeakTable<object, object>`.

```csharp
...
var conditionalWeakTable = sp.GetRequiredService<ConditionalWeakTable<object, object>>();
    // Помещаем instance и logger в ConditionalWeakTable.
    conditionalWeakTable.Add(instance, logger);
...
```

Сам объект `ConditionalWeakTable<object, object>` должен быть зарегистрирован в DI контейнере со временем жизни
равным или большим чем `OrderService`, `OrderServiceLogger`.

Рекомендую использовать `Scoped` если регистрируемые объекты живут не дольше. `Singleton` использовать не обязательно.

Ну и последний элемент пазла - необходимо на уровне приложения создать экземпляр `ConditionalWeakTable<object, object>`, который живет не меньше чем хранимые в нём объекты.

Самый простой пример:

```csharp
class Program
{
    private static void Main()
    {
        var services = new ServiceCollection();
        services.AddScoped<ConditionalWeakTable<object, object>>();

        // регистрация всех типов и другой сервисный код приложения
        ...
        ...

        // Instance ConditionalWeakTable, который держит ссылки на теневые объекты.
        var conditionalWeakTable = serviceProvider.GetRequiredService<ConditionalWeakTable<object, object>>();
        // Запускаем работу приложения.
        Run(...);
    }
}
```

## Заключение

**Какие преимущества подхода вижу я:**
- Логгер автоматически привязывается к конкретному инстансу класса.
- Слабые ссылки гарантируют работу без утечек памяти.
- Централизованная подписка в DI-контейнере.
- Возможность гибкого расширения количества теневых сервисов и управления ими.
- Крепкий SOILD с минимумом компромиссов.

Рекомендую использовать для серьёзных проектов, где качественная архитектура даёт ощутимое преимущество.



