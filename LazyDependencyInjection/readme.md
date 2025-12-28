# Использование `Lazy<T>` с Dependency Injection в .NET

Dependency Injection (DI) — это популярный механизм внедрения зависимостей, который идеально соответствует принципам SOLID (Dependency Inversion Principle). В .NET использование DI (`Microsoft.Extensions.DependencyInjection`) стало стандартом де-факто.

Однако у DI есть важный недостаток: при создании корневого объекта (например, контроллера) контейнер резолвит **всё дерево зависимостей**, включая глубоко вложенные. В крупных системах с микросервисами или тяжелыми сервисами (БД, внешние API, ML-модели) это приводит к:

- **Замедлению startup времени** — создание объекта сервиса, даже если он не будет использоваться - не будет вызван по условиям в коде;
- **Увеличению потребления памяти** — все без исключения объекты создаются заранее.

## Решение: использование DI с `Lazy<T>` — ленивая инициализация зависимостей

Класс `Lazy<T>` создает объект только при первом обращении к свойству `.Value`,
`Microsoft.Extensions.DependencyInjection.ServiceProvider` **нативно поддерживает** инъекцию `Lazy<T>`.

Всё что нужно сделать это зарегестрировать в контейнере `Lazy<T>` вместе с оборачиваемым сервисом.

```csharp
builder.Services.AddScoped<IDatabaseService, SqlDatabaseService>();
builder.Services.AddScoped<Lazy<IDatabaseService>>();
```

После этого можно внедрять обёрнутый сервис.

```csharp
public class ProductController : ControllerBase
{
    private readonly Lazy<IDatabaseService> _db;

    public ProductController(Lazy<IDatabaseService> db)
    {
        _db = db;
    }
}
```

## `Lazy<T>` vs `Func<T>`

Есть большая разница, если вместо `Lazy<T>` использовать `Func<T>`.

При использовании `Lazy<T>` создание обёрнутого объекта произойдёт один раз при первом вызове `.Value`, повторные вызовы `.Value` всегда будут возвращать готовый объект из `Lazy<T>`.

Не забываем, что сам `Lazy<T>` является контейнером и его тип lifetime повлияет, какой экземпляр объекта получат сервисы, например:

```csharp
builder.Services.AddScoped<IDatabaseService, SqlDatabaseService>();
builder.Services.AddScoped<Lazy<IDatabaseService>>();
```

Тут оба сервиса имеют lifetime `scoped` и во всех внедрениях внутри одного `scope` будет внедрён один и тот же экземпляр `Lazy<IDatabaseService>` и это ожидаемое поведение.

Следующий пример сложнее:

```csharp
builder.Services.AddTransient<IDatabaseService, SqlDatabaseService>();
builder.Services.AddScoped<Lazy<IDatabaseService>>();
```

Тут очевидно, что `Lazy<IDatabaseService>` инкапсулирует созданный объект `IDatabaseService` внутри `scope`.
Такое поведение может оказатся неожиданным, так как отличается от указанного жизненного цикла `transient` при регистрации `IDatabaseService`. **Будьте внимательны**.

`Func<T>` ведёт себя совершенно иначе - он будет запрашивать получение объекта из контейнера при каждом вызове `()` и конечное поведение будет зависить с каким lifetime был зарегистрирован обёрнутый тип - `AddScoped`, `AddSingleton` или `AddTransient`.

Это принципиально разное поведение, которое нужно учитывать.

**`Func<T>`** регистрируется так:
```csharp
builder.Services.AddScoped<Func<IDatabaseService>>(sp => () => sp.GetRequiredService<IDatabaseService>());
```

Вот хороший пример, поясняющий разницу в поведении и показывающий преимущества использования `Lazy<T>`.

```scharp
public class ProductController : ControllerBase
{
    private readonly Func<IDatabaseService> _db;

    public ProductController(Func<IDatabaseService> db)
    {
        _db = db;
    }

    public void DoSomthing()
    {
        _db().CheckConnection();
        _db().SelectSomthing();
    }
}
```

Как будет работать этот код, если `IDatabaseService` зарегистрирован в контейнере через `AddTransient`? Каждый вызов `_db()` будет возвращать новый экземпляр объекта `IDatabaseService`. Скорее всего это не то поведение, которое вам нужно.

Использование `Lazy<T>` лишено такого недостатка, так как он сам является контейнером.

## Практические рекомендации

1. **Используйте `Lazy<T>` для**:
   - Сервисов с тяжёлой инициализацией затрагивающей DB, FileSystem, External API;
   - Опциональных зависимостей, что часто актуально для бизнес логики.

2. **Опционально избегайте использование в**:
   - Singleton без опциональной зависимости, тут ленивость теряет функциональный смысл, но для чистого кода использовать её - ок;
   - Критических путях, где ленивость добавляет небольшой overhead в виде вызова `.Value`, но обычно в бизнес логике это не оказывает существенного влияния на производительность, но в отдельных критических к скорости алгоритмах нужно взвесить за и против.


**Вывод**: `Lazy<T>` — простой и мощный инструмент для оптимизации DI. Он не требует сторонних библиотек и работает из коробки с Microsoft DI. Для Autofac или Unity доступны расширения вроде `LazyProxy`.

**Код с примером использования `Lazy<T>` с DI можно посмотреть на** [GitHub](https://github.com/abaula/guess_number)
