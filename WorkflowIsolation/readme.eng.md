# Implementing an In-Process Actor Model in .NET via System.Threading.Channels and Events

> Workflow Isolation: Building In-Process Actors in .NET Without Frameworks

[Example code for the article](https://github.com/abaula/WorkflowIsolation)

## Abstract

In the Go world, the CSP (Communicating Sequential Processes) concept and goroutines communicating via channels have become the de facto standard for building scalable systems. In the .NET ecosystem, developers more often rely on classic async/await, which, while solving the problem of CPU thread blocking, imposes hidden architectural limitations: the code of the called class is often executed in the context of the caller's thread.

In this article, we will explore how to elegantly port the philosophy of Go channels to C# classes without using heavy third-party frameworks like Akka.NET. We will break down the "Isolated Actor" pattern, which combines lightweight built-in `System.Threading.Channels` queues with the standard C# event mechanism. This approach allows us to implement asynchronous message passing within a single process, ensuring strict adherence to the principles of Workflow Isolation and Workflow Single Responsibility at the level of individual system components.

## Introduction: Rethinking Multithreading in .NET

Are you familiar with the situation: you open the logs of a high-load service and see how the business logic of a single class is executed across five different Thread Pool threads, replacing each other? Classic async/await blurs the execution context. When ActorA calls a method of ActorB, the receiver's code unceremoniously "hijacks" the current pool thread for its own needs. As a result, the mental model of predictable execution is violated, and debugging turns into a detective story.

The famous manifesto by the creators of the Go language states: "Do not communicate by sharing memory; instead, share memory by communicating." But what if we apply this principle within the C# ecosystem?

>**Important Architectural Note:**
>
>The terms in this article are based on the concept of Workflow Isolation. Since in modern .NET we operate with Task abstractions, under the hood the scheduler (TaskScheduler) can switch physical OS threads on every asynchronous transition (await). We focus specifically on the isolation of the *workflow*: the workflow (an abstraction) allocated for processing the tasks of a specific actor will never be used to execute code of classes outside the actor's area of responsibility.

Using the modern .NET stack, we can design an architecture where each class exclusively owns its own isolated workflow, and interaction between them is built on asynchronous FIFO buffers. After reading this article, you will learn how to create autonomous in-process actors that exchange messages via Go-style events, maintaining strict ordering and purity of the execution pipeline.

## The Illusion of Isolation in Async C# and the "Thread Hijacking" Problem

Most .NET developers are used to perceiving the async/await keywords as synonymous with safe multithreading. We write asynchronous code to avoid blocking threads, and it intuitively seems that since methods execute asynchronously, the system components themselves operate independently of each other.

However, this is an illusion. Architecturally, async/await solves the problem of efficient CPU resource utilization, but it does not solve the problem of execution context isolation.

## The Problem: How Standard async/await "Hijacks" Context

Consider a classic situation. We have two services: OrderService (processes orders) and NotificationService (sends notifications). They are written according to standard canons:

```csharp
public class OrderService
{
    private readonly NotificationService _notificationService;

    public async Task CreateOrderAsync(Order order)
    {
        // 1. Order creation logic (executed in Thread No. 1)
        await SaveToDbAsync(order);

        // 2. Calling a method of another service
        await _notificationService.SendEmailAsync(order.UserId, "Order created");
    }
}
```

What happens at the OS and Thread Pool level when SendEmailAsync is called?

Since calling an asynchronous method in C# involves unfolding a state machine, Thread No. 1, which was just processing the order logic inside OrderService, freely enters the SendEmailAsync method and starts executing the notification logic.

If a real asynchronous wait occurs inside SendEmailAsync (e.g., a network request to an SMTP server), Thread No. 1 is released. But when the network response returns, the continuation of the SendEmailAsync method is picked up by any random thread from the Thread Pool (let's say Thread No. 2). And it is this Thread No. 2 that, having finished sending the email, will return back to OrderService and continue executing the remaining code there.

From a hardware perspective, this is efficient. From an architectural perspective, it's chaos:

* Violation of responsibility boundaries: The code of an external service (NotificationService), which is far outside the boundaries of OrderService's responsibility, is executed within the current order creation workflow.
* Implicit delays and blockages: Instead of doing its direct job, the OrderService workflow is forced to idle and wait for the complete completion of someone else's computations. The component completely loses control over which specific tasks are currently consuming the resource allocated to its workflow.

## The Solution: Architectural Shift to Workflow Isolation and Workflow Single Responsibility

To bring order to complex distributed systems within a single process, we need to rethink object interaction. Instead of direct method calls, we transition to the concept of Isolated Actors.

We introduce two key rules:

   1. Workflow Isolation: The workflow allocated for processing the tasks of a specific actor will never be used to execute code of classes outside the actor's area of responsibility.
   2. Workflow Single Responsibility: The allocated workflow of an actor has the right to execute only the business logic for which this specific actor is responsible.

Two actors no longer call each other's methods directly. They throw data structures into channels. Our actor no longer exposes heavy methods like SendEmailAsync. Instead, it has:

* An incoming buffer (Mailbox): A fast, thread-safe FIFO queue based on System.Threading.Channels.
* Its own worker: An infinite loop launched in Task.Run that sequentially reads this buffer.

When OrderService wants to send a notification, it no longer executes the sending code. It simply calls the instantaneous, non-blocking SendAsync() method into the NotificationService buffer (equivalent to the `ch <- message` operation in Go) and immediately goes back to its own business. The order workflow is completely isolated from foreign contexts, and the ordering of messages is guaranteed by the channel structure.

## Anatomy of an "Isolated Actor" Based on System.Threading.Channels

Let's move on to the practical implementation. To build an efficient in-process message passing system, a standard `ConcurrentQueue<T>` won't do—you'd have to build complex locking and signaling logic around it using `SemaphoreSlim` or `TaskCompletionSource`.

In modern .NET, the `System.Threading.Channels` library is perfectly suited for these purposes. It is an ultra-fast, thread-safe data structure implementing the "Producer-Consumer" pattern with near-zero memory allocation.

Below is the complete code for our SequentialActor. It strictly preserves ordering, isolates the workflow, and correctly releases resources.

```csharp
using System;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;

// Event argument class for passing data between actors
public class ActorMessageEventArgs : EventArgs
{
    public string Payload { get; }
    public string SenderName { get; }

    public ActorMessageEventArgs(string payload, string senderName)
    {
        Payload = payload;
        SenderName = senderName;
    }
}

public class SequentialActor : IDisposable, IAsyncDisposable
{
    private readonly Channel<string> _buffer;
    public string Name { get; }

    private readonly CancellationTokenSource _cts;
    private Task? _processingTask;
    private bool _isDisposed;

    // Feedback event for other actors
    public event EventHandler<ActorMessageEventArgs>? MessageProcessed;

    public SequentialActor(string name)
    {
        Name = name;
        _cts = new CancellationTokenSource();

        // Configuring channel optimization for the actor architecture
        _buffer = Channel.CreateUnbounded<string>(new UnboundedChannelOptions
        {
            // FIFO GUARANTEE: We explicitly specify that only ONE worker will read from the buffer.
            // This disables internal read contention and guarantees strict ordering.
            SingleReader = true,

            // Disallow synchronous continuations so that the calling workflow thread does not get
            // pulled into the pipeline processing logic when the buffer is filled/freed.
            AllowSynchronousContinuations = false
        });
    }

    /// <summary>
    /// Non-blocking method to write a message to the actor's buffer.
    /// Can be safely called concurrently from any other threads.
    /// </summary>
    public async ValueTask SendAsync(string message)
    {
        ObjectDisposedException.ThrowIf(_isDisposed, this);
        await _buffer.Writer.WriteAsync(message);
    }

    /// <summary>
    /// Starts the actor's lifecycle. Creates an isolated workflow.
    /// </summary>
    public void StartProcessing()
    {
        ObjectDisposedException.ThrowIf(_isDisposed, this);
        if (_processingTask != null) return;

        _processingTask = Task.Run(async () =>
        {
            try
            {
                // The ReadAllAsync method returns an async stream (IAsyncEnumerable).
                // The loop will extract messages strictly in FIFO order.
                await foreach (var message in _buffer.Reader.ReadAllAsync(_cts.Token))
                {
                    // Workflow Isolation point: processing code is executed STRICTLY sequentially.
                    // The actor's workflow performs exclusively its internal tasks,
                    // adhering to Workflow Single Responsibility. The next message will not start
                    // processing until the current await completes.
                    await ProcessMessageInternalAsync(message);

                    // Notify subscribers via event
                    OnMessageProcessed(new ActorMessageEventArgs(message, Name));
                }
            }
            catch (OperationCanceledException)
            {
                // Standard cancellation via CancellationToken
            }
            catch (Exception ex)
            {
                // Business logic errors should not crash the entire application lifecycle
                LogError(ex);
            }
        });
    }

    private async Task ProcessMessageInternalAsync(string message)
    {
        // Simulating useful class work (DB operations, HTTP requests, etc.)
        await Task.Delay(100, _cts.Token);
    }

    protected virtual void OnMessageProcessed(ActorMessageEventArgs e)
    {
        MessageProcessed?.Invoke(this, e);
    }

    private void LogError(Exception ex) => Console.WriteLine($"[{Name}] Error: {ex.Message}");

    #region Resource Cleanup Interfaces Implementation (Graceful Shutdown)

    public async ValueTask DisposeAsync()
    {
        if (_isDisposed) return;
        _isDisposed = true;

        // 1. Gracefully close the buffer for writing.
        // New messages are not accepted, but the ReadAllAsync loop will continue working
        // until it processes ALL messages already in the queue.
        _buffer.Writer.TryComplete();

        // 2. Wait for the workflow to completely finish draining the queue
        if (_processingTask != null)
        {
            try { await _processingTask; } catch { }
        }

        MessageProcessed = null; // Protect against memory leaks via events
        _cts.Dispose();
        GC.SuppressFinalize(this);
    }

    public void Dispose()
    {
        if (_isDisposed) return;
        _isDisposed = true;

        _buffer.Writer.TryComplete();
        _cts.Cancel(); // In a synchronous scenario, we have to forcefully interrupt processing

        _processingTask?.GetAwaiter().GetResult(); // Safe blocking wait for Task
        MessageProcessed = null;
        _cts.Dispose();
        GC.SuppressFinalize(this);
    }

    #endregion
}
```

## Architectural Accents of the Code:

   1. The `SingleReader = true` flag: Under the hood, System.Threading.Channels switches to a highly optimized data structure. It doesn't need to check if another thread is trying to grab the element simultaneously. This provides a colossal speed boost and reduces CPU load.
   2. The `AllowSynchronousContinuations = false` parameter: If not set, the thread writing a message via SendAsync might be forcibly involved by the scheduler in executing the worker's logic itself. By setting it to false, we strictly separate contexts: the sender thread only puts data in and leaves instantly.
   3. Dual Dispose implementation: In the async world, it's important to give the system a chance to finish gracefully (Graceful Shutdown). Closing the writer via TryComplete() signals the code "finish what you managed to grab and shut down smoothly."


## Asynchronous Binding of Actors via Events Without Breaking Isolation

When we isolated the computations of each actor within its own incoming buffer, the next architectural question arises: how to organize feedback? For example, how can NotificationService report back to OrderService that the notification was successfully sent?

The most natural way for C# is to use standard events. However, the standard event mechanism in .NET works synchronously. When an actor generates an event (calls Invoke), the code of all subscriber handlers is executed in the exact same workflow thread that triggered the event.

If we simply subscribe one class to the event of another and start executing heavy logic there, we will instantly destroy our entire architecture. The workflow of the first actor will "fly off" to execute the code of the second, violating both Workflow Isolation and Workflow Single Responsibility.

## The "Fast Forward" Rule

To prevent events from destroying isolation, we impose a strict architectural regulation on the handler code:

The subscriber has no right to execute business logic inside another actor's event handler. The only thing it is allowed to do is take the data from the event arguments and instantly forward (write) it to its own incoming buffer via SendAsync().

Since writing to an unbounded `System.Threading.Channels` (CreateUnbounded) happens instantly in memory without any locks, the calling actor's workflow performs this operation in microseconds and immediately returns to its queue. Heavy processing of the event by the subscriber will only begin when its own isolated workflow gets to this message in the FIFO queue.

## Implementation of Inter-Actor Communication

Let's see what the secure binding of two actors via events looks like in the main application module:

```csharp
class Program
{
    static async Task Main(string[] args)
    {
        // Create two isolated actors
        await using var orderActor = new SequentialActor("OrderProcessor");
        await using var notificationActor = new SequentialActor("NotificationWorker");

        // SUBSCRIPTION (Observing Workflow Isolation):
        // When OrderActor finishes processing, we notify NotificationActor.
        orderActor.MessageProcessed += (sender, e) =>
        {
            // Note: this lambda expression is executed in the OrderActor's workflow!
            // Therefore, there should be no heavy logic here.
            string taskForNotification = $"Send email for order: {e.Payload}";

            // Instant forward to the neighbor's buffer.
            // The OrderActor's workflow is released in fractions of a microsecond.
            _ = notificationActor.SendAsync(taskForNotification);
        };

        // Launch independent workflows for each actor
        orderActor.StartProcessing();
        notificationActor.StartProcessing();

        // Send a start signal from the external context
        await orderActor.SendAsync("Order No. 1024");

        // Give actors time to sequentially execute their tasks
        await Task.Delay(500);
    }
}
```

## Why is the `_ = notificationActor.SendAsync()` call absolutely safe here?

The Fire-and-Forget construct (via ignoring ValueTask) is usually considered an anti-pattern, but here it is justified and safe:

   1. Specifics of the unbounded buffer: The SendAsync method writes to Channel.CreateUnbounded. Such an in-memory queue is always ready to accept an element. The buffer addition operation executes synchronously and instantly. Under the hood, WriteAsync immediately returns an already completed ValueTask (with the IsCompleted = true flag). No allocations for the async/await state machine and no context switches occur.
   2. Isolation guarantee from the neighbor: Thanks to the `AllowSynchronousContinuations = false` flag, even if the NotificationActor was idle and waiting for data at that moment, its awakening will happen strictly in the thread pool via the TaskScheduler. It physically cannot "hijack" the OrderActor's workflow.
   3. Predictable error interception: If the neighbor's buffer is closed (Dispose is called), the WriteAsync method will throw a ChannelClosedException synchronously at the moment of the call. The error won't hang in the background but will instantly bubble up to the try-catch block of the current actor, where it will be correctly handled without risking crashing the process.


## Conclusion: The Architectural Manifesto of the "Isolated Actor"

Implementing the "Isolated Actor" pattern based on `System.Threading.Channels` and C# Events is a powerful way to bring order to the asynchronous code of .NET applications. Porting the philosophy of Go channels to C# classes allows us to move away from the chaotic task distribution inherent in standard async/await and transition to a strictly deterministic, predictable execution model.

## Main Takeaways of Applying the Pattern

* Workflow Isolation in action: Each actor turns into an impregnable fortress. The workflow allocated for its tasks is completely protected from the invasion of foreign code, making logging linear and debugging trivial.
* Workflow Single Responsibility at the design level: System components no longer share CPU time within a single end-to-end call. An actor's workflow is completely insured against idleness and blockages: it will never hang waiting for the execution of foreign tasks outside the scope of the current component's responsibility. Instead, it instantly delegates external work via the buffer and immediately proceeds to process the next message in its queue.
* High performance: Message passing occurs within RAM with near-zero allocation and speeds of millions of operations per second, thanks to the low-level optimizations of the Channels library.
* Safe Fire-and-Forget: Binding actors via events with instant data forwarding is an absolutely reliable architectural solution, as the write operation is atomic, synchronous under the hood, and physically cannot cause deadlocks.

## When Should You Apply This Approach?

This pattern is ideal for systems with complex internal data processing logic within a single process:

   1. Data Processing Pipelines: When data must pass through a chain of sequential stages (e.g., validation ➔ enrichment ➔ writing to DB ➔ sending a notification).
   2. Integration Gateways and Parsers: Where it is necessary to quickly read external sources and pass tasks for processing to internal services without blocking the reading thread.
   3. Local Background Services: Implementing complex scenarios inside `BackgroundService` in ASP.NET Core without involving heavy external message queues like RabbitMQ.

## Final Conclusion

You don't always need heavy distributed frameworks like Akka.NET or Proto.Actor to get the benefits of the actor model. Modern .NET provides developers with concise, lightweight, and incredibly fast tools right "out of the box." The "Isolated Actor" pattern proves that adhering to strict workflow hygiene and proper separation of execution contexts allows you to write the most complex multithreaded systems while keeping the code clean, safe, and easy to understand.

---

## Tags for publication:

.NET, C#, Multithreading, Software Architecture, System.Threading.Channels, Design Patterns
