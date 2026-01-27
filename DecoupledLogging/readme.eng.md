# Shadow Logging via Events - Complete Decoupling of Business Logic and Logging in DI Environment

Hello, colleagues!

I want to share an approach to logging that radically simplifies architecture and strengthens SOLID principles.

I've created a code example on [GitHub](https://github.com/abaula/MixedCode/blob/master/DecoupledLogging/src/Program.cs) to demonstrate how shadow decoupled logging works through C# events.

## What is Decoupled Logging?

Decoupled logging is an architectural approach where business classes **do not contain direct logger calls** (like `ILogger.LogInformation`). Instead:
- Business classes generate **events**.
- **Specialized logger classes** handle the events.
- Business classes know nothing about loggers.
- Logging is configured **centrally** at the application level, without polluting domain logic.

## Pros and Cons of Decoupled Logging

There are opinions both "for" and "against" decoupled logging.

Typical arguments "for":

- **SRP purity**: The class focuses on business logic; logging is an external concern.
- **Testability**: No `ILogger` mocks needed; classes work standalone.
- **Flexibility and scalability**: One event can be handled in all required ways: logs, metrics, audit. Easy to change event handling logic and logging libraries.
- **Decoupling for libraries**: Consumers of business logic classes decide themselves whether to log events and which ones. No hard dependencies.
- **Performance**: Business class does not format data for logs. When implementing event handlers, use smart fast caching of incoming events with subsequent post-processing - it doesn't block business logic.

Critics' objections boil down to a simple thesis: don't complicate things if not really needed - "inject ILogger and don't worry." This opinion sounds reasonable; I agree with such criticism - if you have a simple application, don't complicate it.

## Extracting Logging from the Class

The simplest way to separate business logic and logging is to write a Wrapper for the business class.

Decorator/wrapper for logging is convenient but **imposes usage rules**:
- Client code must work through the wrapper.
- Refactoring becomes more complex.
- Duplication and inheritance issues arise.

This approach makes logging **not "shadow"** - consumers indirectly know about it.

## Complete Separation

Complete separation of business logic and logging is possible only using two independent objects: business class and log handler.

Simple example:

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

This approach is straightforward, but if the application uses a DI container, it requires adaptation.

## Shadowing Logging

DI containers perform 2 important tasks:
- Object factory
- Object lifetime management

Object creation is simple: the DI container will return 2 ready objects, with the log handler receiving the business class instance and logger instance upon creation.

```csharp
var services = new ServiceCollection();
services.AddScoped<FileLogger>();
services.AddScoped<OrderServiceLogger>();
services.AddScoped<OrderService>();
var serviceProvider = services.BuildServiceProvider();

var orderService = serviceProvider.GetRequiredService<OrderService>();
var orderServiceLogger = serviceProvider.GetRequiredService<OrderServiceLogger>();
```

The problem is managing the lifetime of the log handler `OrderServiceLogger`, i.e., explicitly storing a reference to the created object and synchronizing its lifetime with the business class `OrderService` instance.

If we do nothing else, we'll have to explicitly create a new `OrderServiceLogger` instance wherever we create an `OrderService` instance and ensure their lifetimes match - that's not the behavior we want.

**What we need:**
- Use only business logic object instances in business logic, in our example `OrderService`.
- Business logic should know nothing about objects performing other tasks in the application, in our example logging via `OrderServiceLogger`.
- When creating a business logic object, the application must guarantee all implemented service functions for it - if `OrderServiceLogger` is implemented for `OrderService`, it must be created in time and handle events.
- Correct service function operation includes optimal application resource management - the `OrderServiceLogger` instance must be removed from memory after the associated `OrderService` object is destroyed.

These requirements are easy to implement, even within a DI container. We've sorted object creation; now we need to implement lifetime synchronization using weak references.

We need to ensure the created `OrderServiceLogger` object lives no less than the `OrderService` instance and is removed when no longer needed.

For this, we need an application-level object that:
- Stores references to both dependent objects.
- Monitors their lifetimes.
- Removes `OrderServiceLogger` as soon as `OrderService` is removed.

We can implement such a class ourselves, where there is a **key object** and **dependent objects**. The architecture of such a class is simple:
- Key object(s) are stored as weak references, which do not prevent the object from being garbage collected.
- Dependent objects are stored as strong references, which prevent the garbage collector from destroying them.
- The state of key objects is periodically checked - if they are removed, dependent objects are also removed.

For the simple case, we can use the [ConditionalWeakTable<TKey,TValue>](https://learn.microsoft.com/ru-ru/dotnet/api/system.runtime.compilerservices.conditionalweaktable-2?view=net-8.0) class from the `System.Runtime.CompilerServices` namespace, which already implements this logic.

## Writing DI Logic

Let's implement an extension method for `ServiceCollection` and examine how it works.

```csharp
public static class ServiceCollectionExtensions
{
    public static ServiceCollection AddScopedWithLogger<TService, TServiceInstance, TServiceLogger>(
        this ServiceCollection services)
        where TService : class
        where TServiceInstance : class, TService
        where TServiceLogger : class
    {
        // Register TServiceInstance.
        services.AddScoped<TServiceInstance>();
        // Register TServiceLogger.
        services.AddScoped<TServiceLogger>();
        // Register TService.
        services.AddScoped<TService>(sp =>
        {
            var instance = sp.GetRequiredService<TServiceInstance>();
            var logger = sp.GetRequiredService<TServiceLogger>();
            var conditionalWeakTable = sp.GetRequiredService<ConditionalWeakTable<object, object>>();
            // Put instance and logger into ConditionalWeakTable.
            conditionalWeakTable.Add(instance, logger);

            return instance;
        });

        return services;
    }
}
```

The `AddScopedWithLogger` method does all the necessary work:
- Registers all types in DI.
- Implements the logic for creating and linking the business class and its event handler class.

**Important** - in the DI container, it is necessary to separate the logic for creating the business class object itself from the logic for creating the instance with all its shadow objects. For this, it is best to use business class contracts (interfaces).

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

Thus, the DI container will pass the `OrderService` instance to the `OrderServiceLogger` constructor.

**In business logic, only contracts must be used**, which is a recommended approach.

```csharp
class OrderManager : IOrderManager
{
    public OrderManager(IOrderService orderService)
    {
        ...
    }
}
```

Correctly register all types in the container:

```csharp
var services = new ServiceCollection();
services.AddScopedWithLogger<IOrderService, OrderService, OrderServiceLogger>();
```

Now, creating an object for its contract `IOrderService` will trigger the following code from the extension method:

```csharp
...
// Register TService.
services.AddScoped<TService>(sp =>
{
    var instance = sp.GetRequiredService<TServiceInstance>();
    var logger = sp.GetRequiredService<TServiceLogger>();
    var conditionalWeakTable = sp.GetRequiredService<ConditionalWeakTable<object, object>>();
    // Put instance and logger into ConditionalWeakTable.
    conditionalWeakTable.Add(instance, logger);

    return instance;
});
...
```

Here's the breakdown for the parameter combination `IOrderService, OrderService, OrderServiceLogger`:

```csharp
services.AddScoped<IOrderService>(sp =>
{
    var instance = sp.GetRequiredService<OrderService>();
    var logger = sp.GetRequiredService<OrderServiceLogger>();
    var conditionalWeakTable = sp.GetRequiredService<ConditionalWeakTable<object, object>>();
    // Put instance and logger into ConditionalWeakTable.
    conditionalWeakTable.Add(instance, logger);

    return instance;
});
```

As you can see, it's simple. `OrderService` and `OrderServiceLogger` objects are created with all dependencies, then both objects are saved in the `ConditionalWeakTable<object, object>`.

```csharp
...
var conditionalWeakTable = sp.GetRequiredService<ConditionalWeakTable<object, object>>();
// Put instance and logger into ConditionalWeakTable.
conditionalWeakTable.Add(instance, logger);
...
```

The `ConditionalWeakTable<object, object>` object itself must be registered in the DI container with a lifetime equal to or greater than `OrderService` and `OrderServiceLogger`.

I recommend using `Scoped` if the registered objects live no longer. `Singleton` is not necessary.

And the last piece of the puzzle - at the application level, create a `ConditionalWeakTable<object, object>` instance that lives no less than the objects stored in it.

Simplest example:

```csharp
class Program
{
    private static void Main()
    {
        var services = new ServiceCollection();
        services.AddScoped<ConditionalWeakTable<object, object>>();

        // Registration of all types and other application service code
        ...
        ...

        // Instance of ConditionalWeakTable that holds references to shadow objects.
        var conditionalWeakTable = serviceProvider.GetRequiredService<ConditionalWeakTable<object, object>>();
        // Start application work.
        Run(...);
    }
}
```

## Conclusion

**Advantages of the approach as I see them:**
- Logger is automatically bound to a specific class instance.
- Weak references guarantee operation without memory leaks.
- Centralized subscription in the DI container.
- Ability to flexibly extend the number of shadow services and manage them.
- Strong SOLID with minimal compromises.

I recommend using it for serious projects where quality architecture provides a tangible advantage.