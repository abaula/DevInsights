# Эксклюзивный доступ к объектам в .NET: разбираем LockedValueFactory на пальцах

> *Для начинающих разработчиков, которые хотят понять, как безопасно работать с общими данными в многопоточных приложениях*

Привет, коллега! Если ты только начинаешь погружаться в мир асинхронного программирования в .NET, то наверняка уже сталкивался с ситуацией: несколько фоновых задач пытаются одновременно изменить один и тот же объект, и… всё ломается. Знакомо?

Сегодня разберём элегантное решение этой проблемы — механизм **LockedValueFactory<T>**, который помогает организовать эксклюзивный доступ к объектам без головной боли. Поехали! 🚀

---

## Зачем вообще нужна блокировка?

Представь, что у тебя есть общая коллекция заданий `IJobCollection`, которую читают и обновляют несколько фоновых процессов. Если два процесса одновременно попытаются изменить коллекцию, может произойти:

- ❌ Потеря данных
- ❌ Исключения `InvalidOperationException`
- ❌ Непредсказуемое поведение приложения

**Решение:** дать доступ к объекту только одному потоку за раз. Но как сделать это удобно и безопасно?

---

## Знакомьтесь: LockedValueFactory<T>

Это универсальная «фабрика», которая выдаёт специальные обёртки над объектами — **LockedValue<T>**. Работает это примерно так:

```
🔒 Запросил объект → поставил блокировку и получил объект → поработал → отпустил блокировку → следующий может получить объект.
```

Всё построено на двух простых классах:

### 1️⃣ LockedValueFactory<T> — создаёт «заблокированные» объекты

```csharp
public class LockedValueFactory<T> where T : class
{
    private readonly SemaphoreSlim _semaphoreSlim; // = new SemaphoreSlim(1)
    private readonly IServiceProvider _serviceProvider;

    public LockedValueFactory(IServiceProvider serviceProvider)
    {
        _semaphoreSlim = new SemaphoreSlim(1); // Разрешаем только 1 поток одновременно
        _serviceProvider = serviceProvider;
    }

    // Получить объект может только один клиент,
    // остальные будут ждать освобождения блокировки.
    public async Task<LockedValue<T>> Create()
    {
        await _semaphoreSlim.WaitAsync(); // Ждём, пока освободится «ключ»
        // Блокировка установлена, оборачиваекм нужный объект в LockedValue<T> и возвращаем.
        var value = _serviceProvider.GetRequiredService<T>();
        return new LockedValue<T>(_semaphoreSlim, value);
    }
}
```

🔹 **SemaphoreSlim(1)** — это как один ключ от комнаты: кто взял, тот и работает, остальные ждут.
🔹 **WaitAsync()** — не блокирует поток, пока ждёт, что важно для асинхронных приложений.

### 2️⃣ LockedValue<T> — обёртка, которая «отпускает блокировку с объекта» при вызове Dispose()

```csharp
public class LockedValue<T> : IDisposable
{
    private readonly SemaphoreSlim _semaphoreSlim;
    private readonly T _value;
    private bool _disposed;

    public LockedValue(SemaphoreSlim semaphoreSlim, T value)
    {
        _semaphoreSlim = semaphoreSlim;
        _value = value;
    }

    public T Value => _value; // Доступ к самому объекту

    public void Dispose()
    {
        if (_disposed) return;
        _semaphoreSlim.Release(); // 🔓 Отпускаем ключ!
        _disposed = true;
    }
}
```

✨ **Магия в том**, что `LockedValue<T>` реализует `IDisposable`. Это значит, что его можно использовать с конструкцией `using`, и блокировка освободится **автоматически**, даже если в коде возникнет ошибка.

---

## Как это использовать на практике?

Допустим, у тебя есть задача `FetchJobsToMemory`, которая обновляет коллекцию заданий. Вот как выглядит безопасный код:

```csharp
class FetchJobsToMemory : IFetchJobsToMemory
{
    private readonly Lazy<LockedValueFactory<IJobCollection>> _jobCollectionFactory;

    public async Task Execute(int batchSize)
    {
        // Получаем эксклюзивный доступ к коллекции Job
        using var jobCollectionValue = await _jobCollectionFactory.Value.Create();

        // Этот код выполняется только в одном экземпляре одновременно
        // ... Получаем список Job
        var processingJobs = await _selectJobsInStatusPage.Value.Execute(...);
        // ... Обновляем коллекцию
        jobCollectionValue.Value.Update(processingJobs);

        // Выход из using → автоматический Dispose() → блокирока снята, `jobCollection` свободна для других потоков.
    }
}
```

### Почему `using` — это спасение?

```csharp
// Даже если внутри произойдёт исключение:
using var locked = await factory.Create();
locked.Value.DoSomething(); // Ошибка!
// Dispose() всё равно вызовется, и семафор освободится!
```

Без `using` ты рискуешь «забыть» освободить семафор — и все остальные задачи зависнут навсегда. С `using` — безопасно и надёжно.

---

## Регистрация в Dependency Injection

Чтобы всё работало, нужно правильно зарегистрировать сервисы в контейнере DI:

```csharp
services.AddSingletonWithLazy<IJobCollection, JobCollection>();
services.AddSingletonWithLazy<LockedValueFactory<IJobCollection>>();
```

🔹 **IJobCollection** — `Singleton`: один экземпляр на всё приложение.
🔹 **LockedValueFactory<IJobCollection>** — тоже `Singleton`: один семафор на все запросы.

> **Важно:** если бы фабрика создавалась каждый раз заново, у каждого экземпляра был бы свой семафор, и блокировка не работала бы глобально.

---

## Жизненный цикл блокировки: шаг за шагом

```
1️⃣ Вызывается FetchJobsToMemory.Execute()
        ↓
2️⃣ LockedValueFactory.Create() ждёт, пока семафор станет свободным
        ↓
3️⃣ Семафор захвачен → создаётся LockedValue<IJobCollection>
        ↓
4️⃣ Выполняется работа с объектом IJobCollection (чтение/запись)
        ↓
5️⃣ Завершается блок using → вызывается LockedValue.Dispose()
        ↓
6️⃣ Семафор освобождается → следующая задача может начать работу
```

Всё предсказуемо, прозрачно и безопасно

---

## Почему это решение крутое?

| Преимущество | Что это значит для тебя |
|--------------|-------------------------|
| 🔁 **Автоматическое управление** | `using` гарантирует, что блокировка снимется, даже при исключениях |
| ⚡ **Асинхронность** | `WaitAsync()` не «замораживает» потоки — приложение остаётся отзывчивым |
| 🔐 **Эксклюзивный доступ** | Только одна задача работает с объектом — никаких гонок данных |
| 🧱 **Универсальность** | Работает с любым классом, зарегистрированным в DI |
| 🪶 **Минимум накладных расходов** | Один семафор на тип объекта, нет лишней сложности |

---

## Где ещё можно применить LockedValueFactory<T>?

Этот паттерн полезен везде, где нужен контролируемый доступ к общему ресурсу:

### Кэши в памяти
```csharp
using var cache = await _cacheFactory.Value.Create();
cache.Value.Refresh(); // Только один поток обновляет кэш
```

### Состояния сервисов
```csharp
using var state = await _stateFactory.Value.Create();
state.Value.UpdateStatus(...); // Синхронизация доступа к состоянию
```

### Ограничение параллелизма
```csharp
// Например, не больше 1 экспорта файлов одновременно
using var export = await _exportFactory.Value.Create();
await export.Value.RunExport();
```

### Коллекции в памяти
Как в исходном примере с `IJobCollection` — защита от одновременной модификации.

---

## Советы для начинающих

1. **Всегда используй `using`** с `LockedValue<T>` — это твоя страховка от утечек блокировок.
2. **Не держи блокировку дольше нужного** — выполняй внутри `using` только критическую секцию.
3. **Помни про асинхронность** — используй `WaitAsync()`, а не `Wait()`, чтобы не блокировать потоки.
4. **Тестируй на гонки данных** — добавь нагрузку и убедись, что объект действительно защищён.
5. **Документируй**, где используется блокировка — это поможет команде понимать архитектуру.

---

## Заключение

`LockedValueFactory<T>` — это пример того, как сложную задачу синхронизации можно обернуть в простой и интуитивный API. Благодаря `SemaphoreSlim`, `IDisposable` и DI-контейнеру, ты получаешь:

- ✅ Безопасный доступ к общим объектам
- ✅ Чистый и поддерживаемый код
- ✅ Минимум шансов допустить ошибку

🛡️ Попробуй применить этот паттерн в своём проекте — и спи спокойно, зная, что твои данные под защитой!
