# Advanced C# / .NET Development Tactics
### Comprehensive Production-Ready Guide with Full Examples

**Target**: .NET 8 / 9 · C# 12 / 13  
**Style**: Modern, idiomatic, production-oriented

---

## Table of Contents

1. Modern C# Language Features
2. Asynchronous Programming & Concurrency
3. Performance & Memory (Span, Memory, Pools)
4. Resilience & Error Handling
5. Observability
6. Configuration & Options Pattern
7. Validation
8. Caching
9. Dependency Injection & Architecture
10. Testing
11. Source Generators
12. Native AOT & Trimming
13. Security
14. High-Performance Networking
15. Domain-Driven Design Tactics
16. Channels & High-Throughput Pipelines
17. Interceptors (C# 12+)
18. Generic Math & Numeric Abstractions
19. Minimal APIs Advanced Patterns
20. gRPC & HTTP/3
21. Hot Reload & Developer Productivity
22. Advanced Memory Marshaling

---

## 1. Modern C# Language Features

```csharp
public class OrderService(IOrderRepository repo, ILogger<OrderService> logger)
{
    public async Task<Order> GetAsync(Guid id, CancellationToken ct = default)
    {
        logger.LogInformation("Fetching order {OrderId}", id);
        return await repo.GetAsync(id, ct) 
            ?? throw new NotFoundException($"Order {id} not found");
    }
}

List<int> numbers = [1, 2, 3, 4, 5];
int[] array = [..numbers, 6, 7];

public class User
{
    public required Guid Id { get; init; }
    public required string Email { get; init; }
    public string? FullName { get; init; }
}
```

---

## 2. Asynchronous Programming & Concurrency

```csharp
public ValueTask<Order?> GetFromCacheAsync(Guid id)
{
    if (_cache.TryGetValue(id, out var order))
        return new ValueTask<Order?>(order);
    return new ValueTask<Order?>(LoadFromDbAsync(id));
}
```

---

## 3. Performance & Memory

```csharp
public int ParseNumbers(ReadOnlySpan<char> input)
{
    int sum = 0;
    foreach (var segment in input.Split(','))
    {
        if (int.TryParse(segment, out var number))
            sum += number;
    }
    return sum;
}

var pool = ArrayPool<byte>.Shared;
byte[] buffer = pool.Rent(4096);
try { /* use buffer */ }
finally { pool.Return(buffer); }
```

---

## 4. Resilience & Error Handling (Polly v8)

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(200),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 10,
        BreakDuration = TimeSpan.FromSeconds(15)
    })
    .AddTimeout(TimeSpan.FromSeconds(5))
    .Build();
```

---

## 5–15. Core Topics (Observability, Options, Validation, Caching, DI, Testing, Source Generators, AOT, Security, Networking, DDD)

(Retained from previous version — structured logging, OpenTelemetry, IOptions validation, FluentValidation, HybridCache, Keyed services, xUnit + WebApplicationFactory, LoggerMessage / JSON source gen, Native AOT considerations, secret management, HttpClientFactory + resilience, Value Objects.)

---

## 16. Channels & High-Throughput Pipelines

```csharp
public async Task RunPipelineAsync(CancellationToken ct)
{
    var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(10_000)
    {
        FullMode = BoundedChannelFullMode.Wait,
        SingleReader = false,
        SingleWriter = false
    });

    var producers = Enumerable.Range(0, 4).Select(i => Task.Run(async () =>
    {
        await foreach (var item in GetWorkItemsAsync(i, ct))
            await channel.Writer.WriteAsync(item, ct);
    }, ct));

    var consumers = Enumerable.Range(0, Environment.ProcessorCount).Select(_ => Task.Run(async () =>
    {
        await foreach (var item in channel.Reader.ReadAllAsync(ct))
            await ProcessAsync(item, ct);
    }, ct));

    await Task.WhenAll(producers);
    channel.Writer.Complete();
    await Task.WhenAll(consumers);
}
```

---

## 17. Interceptors (C# 12+)

Compile-time interception for cross-cutting concerns (logging, metrics, caching) without runtime proxies.

```csharp
using System.Runtime.CompilerServices;

public static class OrderInterceptors
{
    [InterceptsLocation("Services/OrderService.cs", line: 25, character: 16)]
    public static async Task<Order> GetAsync_Interceptor(
        this OrderService svc, Guid id, CancellationToken ct)
    {
        // pre-logic
        var result = await svc.GetAsync(id, ct);
        // post-logic
        return result;
    }
}
```

---

## 18. Generic Math & Numeric Abstractions

```csharp
public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T result = T.Zero;
    foreach (var v in values)
        result += v;
    return result;
}

public static T Clamp<T>(T value, T min, T max) where T : IComparable<T>
{
    if (value.CompareTo(min) < 0) return min;
    if (value.CompareTo(max) > 0) return max;
    return value;
}
```

---

## 19. Minimal APIs Advanced Patterns

```csharp
var orders = app.MapGroup("/orders").RequireAuthorization();

orders.MapPost("/", async (CreateOrderRequest req, OrderService svc) =>
{
    var order = await svc.CreateAsync(req);
    return Results.Created($"/orders/{order.Id}", order);
})
.AddEndpointFilter(async (ctx, next) =>
{
    var logger = ctx.HttpContext.RequestServices.GetRequiredService<ILoggerFactory>()
        .CreateLogger("Orders");
    logger.LogInformation("Creating order");
    return await next(ctx);
})
.WithOpenApi()
.Produces<Order>(StatusCodes.Status201Created)
.ProducesProblem(StatusCodes.Status400BadRequest);
```

---

## 20. gRPC & HTTP/3

```csharp
builder.Services.AddGrpc();
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5001, listen =>
    {
        listen.Protocols = HttpProtocols.Http1AndHttp2AndHttp3;
        listen.UseHttps();
    });
});

app.MapGrpcService<OrderGrpcService>();
```

---

## 21. Hot Reload & Developer Productivity

- `dotnet watch run`
- Prefer designs that allow method-body changes without restart
- Source generators and interceptors that cooperate with Hot Reload
- Avoid changing public surface or generic constraints during a session

---

## 22. Advanced Memory Marshaling

```csharp
public static partial class Native
{
    [LibraryImport("native", EntryPoint = "process_buffer")]
    public static partial int ProcessBuffer(ReadOnlySpan<byte> input, Span<byte> output);
}

// Zero-copy friendly interop
Span<byte> input = stackalloc byte[256];
Span<byte> output = stackalloc byte[256];
int written = Native.ProcessBuffer(input, output);
```

Prefer source-generated `LibraryImport` over classic `DllImport`.

---

**Key Recommendations (Updated)**

- Channels for high-throughput work queues
- Interceptors + Source Generators over runtime proxies
- Generic Math for reusable numeric code
- Minimal APIs + Endpoint Filters for new HTTP services
- HTTP/3 where latency and multiplexing matter
- Design with Native AOT and trimming in mind from the start

*Document version: 2.0 — Expanded*
