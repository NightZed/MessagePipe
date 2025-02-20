# MessagePipe
[![GitHub Actions](https://github.com/Cysharp/MessagePipe/workflows/Build-Debug/badge.svg)](https://github.com/Cysharp/MessagePipe/actions) [![Releases](https://img.shields.io/github/release/Cysharp/MessagePipe.svg)](https://github.com/Cysharp/MessagePipe/releases)

MessagePipe 是一个专为 .NET 和 Unity 设计的高性能内存/分布式消息管道。支持所有 Pub/Sub 场景、CQRS 中介模式、Prism 的 EventAggregator（视图与视图模型解耦）、IPC（进程间通信）-RPC 等。

* 依赖注入优先
* 过滤器管道
* 更高效的事件机制
* 同步/异步支持
* 带键/无键消息
* 缓冲/无缓冲模式
* 单例/作用域生命周期
* 广播/响应（支持多响应）
* 内存内/进程间/分布式通信

MessagePipe 的性能远超标准 C# 事件，比 Prism 的 EventAggregator 快 78 倍。

![](https://user-images.githubusercontent.com/46207/115984507-5d36da80-a5e2-11eb-9942-66602906f499.png)

每次发布操作的内存分配为零。

![](https://user-images.githubusercontent.com/46207/115814615-62542800-a430-11eb-9041-1f31c1ac8464.png)

提供 Roslyn 分析器防止订阅泄漏。

![](https://user-images.githubusercontent.com/46207/117535259-da753d00-b02f-11eb-9818-0ab5ef3049b1.png)

入门指南
---
在 .NET 中使用 NuGet 安装。Unity 用户请参考 [Unity](#unity) 章节。

> PM> Install-Package [MessagePipe](https://www.nuget.org/packages/MessagePipe)

MessagePipe 基于 `Microsoft.Extensions.DependencyInjection`（Unity 中可使用 `VContainer`、`Zenject` 或 `Builtin Tiny DI`），通过 .NET 泛型主机（[.NET Generic Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host)）的 `ConfigureServices` 配置。泛型主机广泛应用于 ASP.NET Core、[MagicOnion](https://github.com/Cysharp/MagicOnion/)、[ConsoleAppFramework](https://github.com/Cysharp/ConsoleAppFramework/)、MAUI、WPF 等场景。

```csharp
using MessagePipe;
using Microsoft.Extensions.DependencyInjection;

Host.CreateDefaultBuilder()
    .ConfigureServices((ctx, services) =>
    {
        services.AddMessagePipe(); // 使用 AddMessagePipe(options => { }) 配置选项
    })
```

通过注入 `IPublisher<T>` 发布消息，注入 `ISubscriber<T>` 订阅消息，类似于 `Logger<T>`。`T` 可以是任意类型（基本类型、结构体、类、枚举等）。

```csharp
using MessagePipe;

public struct MyEvent { }

public class SceneA
{
    readonly IPublisher<MyEvent> publisher;
    
    public SceneA(IPublisher<MyEvent> publisher)
    {
        this.publisher = publisher;
    }

    void Send()
    {
        this.publisher.Publish(new MyEvent());
    }
}

public class SceneB
{
    readonly ISubscriber<MyEvent> subscriber;
    readonly IDisposable disposable;

    public SceneB(ISubscriber<MyEvent> subscriber)
    {
        var bag = DisposableBag.CreateBuilder(); // 组合式 Disposable 管理订阅
        
        subscriber.Subscribe(x => Console.WriteLine("已接收")).AddTo(bag);

        disposable = bag.Build();
    }

    void Close()
    {
        disposable.Dispose(); // 取消订阅，所有订阅必须显式释放
    }
}
```

与事件机制类似，但通过类型解耦。`Subscribe` 返回 `IDisposable`，便于取消订阅。可通过 `DisposableBag`（类似 `CompositeDisposable`）批量管理订阅。详见[管理订阅与诊断](#管理订阅与诊断)章节。

发布者/订阅者（内部称为 MessageBroker）由 DI 管理，支持按作用域隔离。作用域释放时自动取消所有订阅，防止泄漏。

> 默认为单例模式，可通过 `MessagePipeOptions.InstanceLifetime` 配置为 `Singleton` 或 `Scoped`。

`IPublisher<T>/ISubscriber<T>` 为无键接口，MessagePipe 也提供带键接口 `IPublisher<TKey, TMessage>/ISubscriber<TKey, TMessage>`。

例如，在连接 Unity 和 [MagicOnion](https://github.com/Cysharp/MagicOnion/)（类似 SignalR 的实时通信框架） 的实际场景中，通过浏览器 Blazor 传输数据，需要连接 Blazor 的页面（浏览器生命周期）和 MagicOnion 的 Hub（连接生命周期）来传输数据，并按连接 ID 分发消息：

`浏览器 <-> Blazor <- [MessagePipe] -> MagicOnion <-> Unity`

解决方案代码：

```csharp
// MagicOnion（类似 SignalR 的实时通信框架）
public class UnityConnectionHub : StreamingHubBase<IUnityConnectionHub, IUnityConnectionHubReceiver>, IUnityConnectionHub
{
    readonly IPublisher<Guid, UnitEventData> eventPublisher;
    readonly IPublisher<Guid, ConnectionClose> closePublisher;
    Guid id;

    public UnityConnectionHub(IPublisher<Guid, UnitEventData> eventPublisher, IPublisher<Guid, ConnectionClose> closePublisher)
    {
        this.eventPublisher = eventPublisher;
        this.closePublisher = closePublisher;
    }

    override async ValueTask OnConnected()
    {
        this.id = Guid.Parse(Context.Headers["id"]);
    }

    override async ValueTask OnDisconnected()
    {
        this.closePublisher.Publish(id, new ConnectionClose()); / 向浏览器（Blazor）发布
    }

    // 从 Unity 客户端调用
    public Task<UnityEventData> SendEventAsync(UnityEventData data)
    {
        this.eventPublisher.Publish(id, data); // 向浏览器（Blazor）发布
    }
}

// Blazor
public partial class BlazorPage : ComponentBase, IDisposable
{
    [Parameter]
    public Guid ID { get; set; }

    [Inject]
    ISubscriber<Guid, UnitEventData> UnityEventSubscriber { get; set; }

    [Inject]
    ISubscriber<Guid, ConnectionClose> ConnectionCloseSubscriber { get; set; }

    IDisposable subscription;

    protected override void OnInitialized()
    {
        // 从 MagicOnion（即 Unity）接收事件
        var d1 = UnityEventSubscriber.Subscribe(ID, x =>
        {
            // 处理逻辑...
        });

        var d2 = ConnectionCloseSubscriber.Subscribe(ID, _ =>
        {
            // 显示断开状态...
            subscription?.Dispose(); // 取消订阅
        });

        subscription = DisposableBag.Create(d1, d2); // 合并订阅
    }
    
    public void Dispose()
    {
        // 浏览器关闭时取消订阅
        subscription?.Dispose();
    }
}
```

> 与 Reactive Extensions 的 `Subject` 不同，MessagePipe 不提供 `OnCompleted`。`OnCompleted` 的语义模糊，很难确定观察者（订阅者）的意图，且在多事件订阅时难以处理。因此，MessagePipe 只提供了简单的 Publish(OnNext)。若需通知完成，建议通过独立事件处理。

> 也就是说，这相当于  [Relay in RxSwift](https://github.com/ReactiveX/RxSwift/blob/main/Documentation/Subjects.md).

除标准 Pub/Sub 外，MessagePipe 支持异步处理、带返回值的 Mediator 模式、以及通过过滤器定制执行流程。

接口关系图：

![接口关系图](https://user-images.githubusercontent.com/46207/122254092-bf87c980-cf07-11eb-8bdd-039c87309db6.png)

接口虽多，但 API 设计统一，功能相似。

发布/订阅接口
---
发布/订阅接口分为带键（主题）和无键、同步和异步四类：

```csharp
// 无键-同步
public interface IPublisher<TMessage>
{
    void Publish(TMessage message);
}

public interface ISubscriber<TMessage>
{
    IDisposable Subscribe(IMessageHandler<TMessage> handler, params MessageHandlerFilter<TMessage>[] filters);
}

// 无键-异步
public interface IAsyncPublisher<TMessage>
{
    // async 接口的发布即发即弃
    void Publish(TMessage message, CancellationToken cancellationToken = default(CancellationToken));
    ValueTask PublishAsync(TMessage message, CancellationToken cancellationToken = default(CancellationToken));
    ValueTask PublishAsync(TMessage message, AsyncPublishStrategy publishStrategy, CancellationToken cancellationToken = default(CancellationToken));
}

public interface IAsyncSubscriber<TMessage>
{
    IDisposable Subscribe(IAsyncMessageHandler<TMessage> asyncHandler, params AsyncMessageHandlerFilter<TMessage>[] filters);
}

// 带键-同步
public interface IPublisher<TKey, TMessage>
    where TKey : notnull
{
    void Publish(TKey key, TMessage message);
}

public interface ISubscriber<TKey, TMessage>
    where TKey : notnull
{
    IDisposable Subscribe(TKey key, IMessageHandler<TMessage> handler, params MessageHandlerFilter<TMessage>[] filters);
}

// 带键-异步
public interface IAsyncPublisher<TKey, TMessage>
    where TKey : notnull
{
    void Publish(TKey key, TMessage message, CancellationToken cancellationToken = default(CancellationToken));
    ValueTask PublishAsync(TKey key, TMessage message, CancellationToken cancellationToken = default(CancellationToken));
    ValueTask PublishAsync(TKey key, TMessage message, AsyncPublishStrategy publishStrategy, CancellationToken cancellationToken = default(CancellationToken));
}

public interface IAsyncSubscriber<TKey, TMessage>
    where TKey : notnull
{
    IDisposable Subscribe(TKey key, IAsyncMessageHandler<TMessage> asyncHandler, params AsyncMessageHandlerFilter<TMessage>[] filters);
}
```

所有接口都可以通过 DI 以 `IPublisher/Subscribe<T>` 的形式使用。异步接口的 `await PublishAsync` 可等待所有订阅者完成。发布策略（`AsyncPublishStrategy`）默认为并行（`Parallel`），可通过 `MessagePipeOptions` 或发布时指定。若无需等待，可使用 `void Publish` 即发即弃。

通过自定义过滤器可修改执行前后的行为（详见 [过滤器](#过滤器) 章节）。

错误会传播给调用方并终止后续订阅，此行为可通过过滤器修改。

单例与***作用域***（ISingleton***, IScoped***）
---
I(Async)Publisher(Subscriber) 默认生命周期由 `MessagePipeOptions.InstanceLifetime` 决定。但如果声明为 `ISingletonPublisher<TMessage>`/`ISingletonSubscriber<TKey, TMessage>`、`ISingletonAsyncPublisher<TMessage>`/`ISingletonAsyncSubscriber<TKey, TMessage>`，则强制为单例。同理，`IScopedPublisher<TMessage>`/`IScopedSubscriber<TKey, TMessage>`、`IScopedAsyncPublisher<TMessage>`/`IScopedAsyncSubscriber<TKey, TMessage>` 为作用域生命周期。

缓冲接口
---
`IBufferedPublisher<TMessage>/IBufferedSubscriber<TMessage>` 类似于 `BehaviorSubject` 或 RxSwift 的 `BehaviorRelay`， `Subscribe` 订阅时返回最新值。

```csharp
var p = provider.GetRequiredService<IBufferedPublisher<int>>();
var s = provider.GetRequiredService<IBufferedSubscriber<int>>();

p.Publish(999);

var d1 = s.Subscribe(x => Console.WriteLine(x)); // 999
p.Publish(1000); // 1000

var d2 = s.Subscribe(x => Console.WriteLine(x)); // 1000
p.Publish(9999); // 9999, 9999

DisposableBag.Create(d1, d2).Dispose();
```

> 若 `TMessage` 为类且无初始值（null），则订阅时不发送值。

> Keyed buffered publisher/subscriber does not exist because difficult to avoid memory leak of (unused)key and keep latest value.> 不存在带键的缓冲发布者/订阅者，因为难以避免（未使用的）键的内存泄漏，也无法保证值最新。

事件工厂
---
通过 `EventFactory` 可创建类似 C# 事件的泛型接口（`IPublisher/ISubscriber`, `IAsyncPublisher/IAsyncSubscriber`, `IBufferedPublisher/IBufferedSubscriber`, `IBufferedAsyncPublisher/IBufferedAsyncSubscriber`），订阅者与实例绑定而非按类型分组。

相比原生 C# 事件的优势：

* 使用 Subscribe/Dispose 替代 `+=`/`-=`，易于管理
* 同步/异步支持
* 缓冲/无缓冲支持
* 通过 `Dispose` 取消所有订阅
* 通过过滤器定制流程
* 通过 `MessagePipeDiagnosticsInfo` 监控泄漏
* 通过 `MessagePipe.Analyzer` 预防泄漏

```csharp
public class BetterEvent : IDisposable
{
    // 使用 MessagePipe 代替 C# 事件/Rx.Subject
    // 将 Publisher 存储在私有字段中（声明为 IDisposablePublisher/IDisposableAsyncPublisher）
    IDisposablePublisher<int> tickPublisher;

    // Subscriber 从外部使用，因此是公共访问级别
    public ISubscriber<int> OnTick { get; }

    public BetterEvent(EventFactory eventFactory)
    {
        // CreateEvent 可以通过元组解构并一起设置
        (tickPublisher, OnTick) = eventFactory.CreateEvent<int>();

        /// 也可以通过 `CreateAsyncEvent` 创建异步事件（IAsyncSubscriber）
        // eventFactory.CreateAsyncEvent
    }

    int count;
    void Tick()
    {
        tickPublisher.Publish(count++);
    }

    public void Dispose()
    {
        // 释放所有订阅
        tickPublisher.Dispose();
    }
}
```

若需在 DI 外部创建事件，参考[全局服务提供者](#全局服务提供者)章节。

```csharp
IDisposablePublisher<int> tickPublisher;
public ISubscriber<int> OnTick { get; }

ctor()
{
    (tickPublisher, OnTick) = GlobalMessagePipe.CreateEvent<int>();
}
```

请求/响应/全量处理
---
与[MediatR](https://github.com/jbogard/MediatR)类似，实现中介者模式的支持。

```csharp
public interface IRequestHandler<in TRequest, out TResponse>
{
    TResponse Invoke(TRequest request);
}

public interface IAsyncRequestHandler<in TRequest, TResponse>
{
    ValueTask<TResponse> InvokeAsync(TRequest request, CancellationToken cancellationToken = default);
}
```

例如，声明处理`Ping`类型的处理器：

```csharp
public readonly struct Ping { }
public readonly struct Pong { }

public class PingPongHandler : IRequestHandler<Ping, Pong>
{
    public Pong Invoke(Ping request)
    {
        Console.WriteLine("Ping called.");
        return new Pong();
    }
}
```

通过依赖注入获取处理器：

```csharp
class FooController
{
    IRequestHandler<Ping, Pong> requestHandler;

    // automatically instantiate PingPongHandler.
    public FooController(IRequestHandler<Ping, Pong> requestHandler)
    {
        this.requestHandler = requestHandler;
    }

    public void Run()
    {
        var pong = this.requestHandler.Invoke(new Ping());
        Console.WriteLine("PONG");
    }
}
```

更复杂的实现模式可参考 [微软官方文档](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-application-layer-implementation-web-api#implement-the-command-process-pipeline-with-a-mediator-pattern-mediatr)。

若需声明多个请求处理器，可使用 `IRequestAllHandler` 或 `IAsyncRequestAllHandler` 替代单一处理器：

```csharp
public interface IRequestAllHandler<in TRequest, out TResponse>
{
    TResponse[] InvokeAll(TRequest request);
    IEnumerable<TResponse> InvokeAllLazy(TRequest request);
}

public interface IAsyncRequestAllHandler<in TRequest, TResponse>
{
    ValueTask<TResponse[]> InvokeAllAsync(TRequest request, CancellationToken cancellationToken = default);
    ValueTask<TResponse[]> InvokeAllAsync(TRequest request, AsyncPublishStrategy publishStrategy, CancellationToken cancellationToken = default);
    IAsyncEnumerable<TResponse> InvokeAllLazyAsync(TRequest request, CancellationToken cancellationToken = default);
}
```

```csharp
public class PingPongHandler1 : IRequestHandler<Ping, Pong>
{
    public Pong Invoke(Ping request)
    {
        Console.WriteLine("Ping1 called.");
        return new Pong();
    }
}

public class PingPongHandler2 : IRequestHandler<Ping, Pong>
{
    public Pong Invoke(Ping request)
    {
        Console.WriteLine("Ping1 called.");
        return new Pong();
    }
}

class BarController
{
    IRequestAllHandler<Ping, Pong> requestAllHandler;

    public FooController(IRequestAllHandler<Ping, Pong> requestAllHandler)
    {
        this.requestAllHandler = requestAllHandler;
    }

    public void Run()
    {
        var pongs = this.requestAllHandler.InvokeAll(new Ping());
        Console.WriteLine("PONG COUNT:" + pongs.Length);
    }
}
```

订阅的扩展方法
---
`ISubscriber`（或 `IAsyncSubscriber`）接口要求使用 `IMessageHandler<T>` 处理消息：

```csharp
public interface ISubscriber<TMessage>
{
    IDisposable Subscribe(IMessageHandler<TMessage> handler, params MessageHandlerFilter<TMessage>[] filters);
}
```

通过扩展方法可直接使用 `Action<T>` 简化订阅：

```csharp
public static IDisposable Subscribe<TMessage>(this ISubscriber<TMessage> subscriber, Action<TMessage> handler, params MessageHandlerFilter<TMessage>[] filters)
public static IDisposable Subscribe<TMessage>(this ISubscriber<TMessage> subscriber, Action<TMessage> handler, Func<TMessage, bool> predicate, params MessageHandlerFilter<TMessage>[] filters)
public static IObservable<TMessage> AsObservable<TMessage>(this ISubscriber<TMessage> subscriber, params MessageHandlerFilter<TMessage>[] filters)
public static IAsyncEnumerable<TMessage> AsAsyncEnumerable<TMessage>(this IAsyncSubscriber<TMessage> subscriber, params AsyncMessageHandlerFilter<TMessage>[] filters)
public static ValueTask<TMessage> FirstAsync<TMessage>(this ISubscriber<TMessage> subscriber, CancellationToken cancellationToken, params MessageHandlerFilter<TMessage>[] filters)
public static ValueTask<TMessage> FirstAsync<TMessage>(this ISubscriber<TMessage> subscriber, CancellationToken cancellationToken, Func<TMessage, bool> predicate, params MessageHandlerFilter<TMessage>[] filters)
```

`Func<TMessage, bool>` 重载可通过谓词过滤消息（内部使用 `PredicateFilter` 实现，其 `Order` 为`int.MinValue` 且始终优先执行）。

`AsObservable` 将消息管道转换为 `IObservable<T>`，可与Reactive Extensions配合使用（Unity中可使用 `UniRx`）。该方法适用于同步订阅（无键、键控、缓冲）。

`AsAsyncEnumerable` 将消息管道转换为 `IAsyncEnumerable<T>`，支持异步 LINQ 和 `async foreach`。该方法适用于异步订阅（无键、键控、缓冲）。

`FirstAsync` 用于获取第一条消息。类似于 `AsObservable().FirstAsync()` 或 `AsObservable().Where().FirstAsync()`。如果使用 `CancellationTokenSource(TimeSpan)`，则类似于 `AsObservable().Timeout().FirstAsync()`。必须传入 `CancellationToken` 以避免任务泄漏。

```csharp
// Unity 中使用 cts.CancelAfterSlim(TIimeSpan) 代替。
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(1));
var value = await subscriber.FirstAsync(cts.Token);
```

`FirstAsync` 适用于同步和异步订阅（无键、键控、缓冲）。

过滤器
---
过滤器系统可在方法调用前后插入逻辑，采用中间件模式设计，允许以类似的语法编写同步和异步代码。
MessagePipe 提供多种过滤器类型：
- 同步过滤器（`MessageHandlerFilter<T>`）
- 异步过滤器（`AsyncMessageHandlerFilter<T>`）
- 请求过滤器（`RequestHandlerFilter<TReq, TRes>`）
- 异步请求过滤器（`AsyncRequestHandlerFilter<TReq, TRes>`）。
要实现其他具体过滤器，可以扩展上述过滤器类型。

过滤器可通过三种方式应用：
- 全局：通过`MessagePipeOptions.AddGlobalFilter`添加
- 按处理器类型：通过特性标记
- 按订阅：在订阅时直接传入

过滤器的执行顺序由`Order`属性决定，并在订阅时生成。由于过滤器按订阅生成，因此可包含状态。

```csharp
public class ChangedValueFilter<T> : MessageHandlerFilter<T>
{
    T lastValue;

    public override void Handle(T message, Action<T> next)
    {
        if (EqualityComparer<T>.Default.Equals(message, lastValue))
        {
            return;
        }

        lastValue = message;
        next(message);
    }
}

// 按订阅使用
subscribe.Subscribe(x => Console.WriteLine(x), new ChangedValueFilter<int>(){ Order = 100 });

// 按处理器类型使用（需注册泛型过滤器）
[MessageHandlerFilter(typeof(ChangedValueFilter<>), 100)]
public class WriteLineHandler<T> : IMessageHandler<T>
{
    public void Handle(T message) => Console.WriteLine(message);
}

// 全局注册
Host.CreateDefaultBuilder()
    .ConfigureServices((ctx, services) =>
    {
        services.AddMessagePipe(options =>
        {
            options.AddGlobalMessageHandlerFilter(typeof(ChangedValueFilter<>), 100);
        });
    });
```

通过特性标记过滤器时，可使用以下属性：
- `[MessageHandlerFilter(type, order)]`
- `[AsyncMessageHandlerFilter(type, order)]`
- `[RequestHandlerFilter(type, order)]`
- `[AsyncRequestHandlerFilter(type, order)]`

以下为过滤器示例：

```csharp
// 根据谓词过滤消息
public class PredicateFilter<T> : MessageHandlerFilter<T>
{
    private readonly Func<T, bool> predicate;

    public PredicateFilter(Func<T, bool> predicate)
    {
        this.predicate = predicate;
    }

    public override void Handle(T message, Action<T> next)
    {
        if (predicate(message))
        {
            next(message);
        }
    }
}
```

```csharp
// 通过锁确保线程安全
public class LockFilter<T> : MessageHandlerFilter<T>
{
    readonly object gate = new object();

    public override void Handle(T message, Action<T> next)
    {
        lock (gate)
        {
            next(message);
        }
    }
}
```

```csharp
// 忽略异常并记录日志
public class IgnoreErrorFilter<T> : MessageHandlerFilter<T>
{
    readonly ILogger<IgnoreErrorFilter<T>> logger;

    public IgnoreErrorFilter(ILogger<IgnoreErrorFilter<T>> logger)
    {
        this.logger = logger;
    }

    public override void Handle(T message, Action<T> next)
    {
        try
        {
            next(message);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, ""); // 记录错误但不传播
        }
    }
}
```

```csharp
// 通过调度器分发消息
public class DispatcherFilter<T> : MessageHandlerFilter<T>
{
    readonly Dispatcher dispatcher;

    public DispatcherFilter(Dispatcher dispatcher)
    {
        this.dispatcher = dispatcher;
    }

    public override void Handle(T message, Action<T> next)
    {
        dispatcher.BeginInvoke(() =>
        {
            next(message);
        });
    }
}
```

```csharp
// 延迟请求处理
public class DelayRequestFilter : AsyncRequestHandlerFilter<int, int>
{
    public override async ValueTask<int> InvokeAsync(int request, CancellationToken cancellationToken, Func<int, CancellationToken, ValueTask<int>> next)
    {
        await Task.Delay(TimeSpan.FromSeconds(request));
        var response = await next(request, cancellationToken);
        return response;
    }
}
```

管理订阅与诊断
---
订阅方法返回 `IDisposable` 对象，调用 `Dispose` 方法即可取消订阅。相比传统事件机制，这种方式能更便捷地管理订阅。若要管理多个 `IDisposable` 对象，可以使用 Rx（如 UniRx）中的 `CompositeDisposable`，或 MessagePipe 内置的 `DisposableBag`。

```csharp
IDisposable disposable;

void OnInitialize(ISubscriber<int> subscriber)
{
    var d1 = subscriber.Subscribe(_ => { });
    var d2 = subscriber.Subscribe(_ => { });
    var d3 = subscriber.Subscribe(_ => { });

    // 静态方法 DisposableBag.Create 支持 1~7 个参数的优化版本或任意数量
    disposable = DisposableBag.Create(d1, d2, d3);
}

void Close()
{
    // 释放所有订阅
    disposable?.Dispose();
}
```

```csharp
IDisposable disposable;

void OnInitialize(ISubscriber<int> subscriber)
{
    // 使用构造器模式，通过 AddTo 方法添加订阅到集合 subscription.AddTo(bag)
    var bag = DisposableBag.CreateBuilder();

    subscriber.Subscribe(_ => { }).AddTo(bag);
    subscriber.Subscribe(_ => { }).AddTo(bag);
    subscriber.Subscribe(_ => { }).AddTo(bag);

    disposable = bag.Build(); // 生成最终的组合式 IDisposable
}

void Close()
{
    // 释放所有订阅
    disposable?.Dispose();
}
```

```csharp
IDisposable disposable;

void OnInitialize(ISubscriber<int> subscriber)
{
    var bag = DisposableBag.CreateBuilder();

    // 使用 DisposableBag.CreateSingleAssignment 创建单次赋值的可释放对象
    var d = DisposableBag.CreateSingleAssignment();
    
    // 在回调中手动释放订阅
    // 通过 SetTo 和 AddTo 方法绑定到集合
    // 或者使用 d.Disposable = subscriber.Subscribe();
    subscriber.Subscribe(_ => { d.Dispose(); }).SetTo(d).AddTo(bag);

    disposable = bag.Build();
}

void Close()
{
    disposable?.Dispose();
}
```

**必须**妥善处理返回的 `IDisposable` 对象，否则会导致内存泄漏。类似 WPF 中常用的弱引用（Weak Reference）在此场景下属于反模式，所有订阅应显式管理。

可通过 `MessagePipeDiagnosticsInfo` 监控订阅数量。该对象可通过依赖注入（DI）容器获取。

```csharp
public sealed class MessagePipeDiagnosticsInfo
{
    /// <summary>获取当前订阅总数</summary>
    public int SubscribeCount { get; }

    /// <summary>
    /// 当启用 MessagePipeOptions.EnableCaptureStackTrace 时，返回所有订阅的堆栈跟踪信息。
    /// </summary>
    public StackTraceInfo[] GetCapturedStackTraces(bool ascending = true);

    /// <summary>
    /// 当启用 MessagePipeOptions.EnableCaptureStackTrace 时，按订阅调用方分组返回堆栈跟踪。
    /// </summary>
    public ILookup<string, StackTraceInfo> GetGroupedByCaller(bool ascending = true)
}
```

通过监控 `SubscribeCount` 可检查订阅泄漏：

```csharp
public class MonitorTimer : IDisposable
{
    CancellationTokenSource cts = new CancellationTokenSource();

    public MonitorTimer(MessagePipeDiagnosticsInfo diagnosticsInfo)
    {
        RunTimer(diagnosticsInfo);
    }

    async void RunTimer(MessagePipeDiagnosticsInfo diagnosticsInfo)
    {
        while (!cts.IsCancellationRequested)
        {
            // 输出当前订阅总数
            Console.WriteLine("SubscribeCount:" + diagnosticsInfo.SubscribeCount);
            await Task.Delay(TimeSpan.FromSeconds(5), cts.Token);
        }
    }

    public void Dispose()
    {
        cts.Cancel();
    }
}
```

启用 `MessagePipeOptions.EnableCaptureStackTrace`（默认关闭）后，可捕获订阅发生的堆栈信息，便于定位泄漏源头。

若发现 `GroupedByCaller` 中某些调用方的订阅数量异常，可通过堆栈跟踪快速定位未处理的订阅。

在 Unity 中，可通过 `Window -> MessagePipe Diagnostics` 窗口可视化 `MessagePipeDianogsticsInfo` 订阅状态：

![image](https://user-images.githubusercontent.com/46207/116953319-e2e41580-acc7-11eb-88c9-a4704bf3e3c9.png)

启用该窗口需配置 `GlobalMessagePipe`：

```csharp
// VContainer 示例
public class MessagePipeDemo : VContainer.Unity.IStartable
{
    public MessagePipeDemo(IObjectResolver resolver)
    {
        // 要求有如下这行
        GlobalMessagePipe.SetProvider(resolver.AsServiceProvider());
    }
}

// Zenject 示例
void Configure(DiContainer container)
{
    GlobalMessagePipe.SetProvider(container.AsServiceProvider());
}

// 原生 DI 示例
var prodiver = builder.BuildServiceProvider();
GlobalMessagePipe.SetProvider(provider);
```

静态分析工具（Analyzer）
---
在上一节中提到过**必须**妥善处理返回的 `IDisposable` 对象。为防止订阅泄漏，MessagePipe 提供了 Roslyn 分析工具：

> PM> Install-Package [MessagePipe.Analyzer](https://www.nuget.org/packages/MessagePipe.Analyzer)

![](https://user-images.githubusercontent.com/46207/117535259-da753d00-b02f-11eb-9818-0ab5ef3049b1.png)

该工具会检测未处理的 `Subscribe` 调用并报错。

在 Unity 2020.2 及以上版本中，可通过 [Roslyn 分析器文档](https://docs.unity3d.com/2020.2/Documentation/Manual/roslyn-analyzers.html) 配置使用。`MessagePipe.Analyzer.dll` 可从 [发布页面](https://github.com/Cysharp/MessagePipe/releases/) 下载。

![](https://user-images.githubusercontent.com/46207/117535248-d5b08900-b02f-11eb-8add-33101a71033a.png)

由于 Unity 对分析器的支持尚不完善，建议配合 [Cysharp/CsprojModifier](https://github.com/Cysharp/CsprojModifier) 使用：

![](https://github.com/Cysharp/CsprojModifier/raw/master/docs/images/Screen-01.png)

分布式发布订阅与 Redis 集成（IDistributedPubSub / MessagePipe.Redis）
---
若需实现跨网络的消息发布订阅，可使用 `IDistributedPublisher<TKey, TMessage>` 和 `IDistributedSubscriber<TKey, TMessage>` 替代 `IAsyncPublisher`。

```csharp
public interface IDistributedPublisher<TKey, TMessage>
{
    ValueTask PublishAsync(TKey key, TMessage message, CancellationToken cancellationToken = default);
}

public interface IDistributedSubscriber<TKey, TMessage>
{
    // 支持同步与异步处理器
    public ValueTask<IAsyncDisposable> SubscribeAsync(TKey key, IMessageHandler<TMessage> handler, MessageHandlerFilter<TMessage>[] filters, CancellationToken cancellationToken = default);
    public ValueTask<IAsyncDisposable> SubscribeAsync(TKey key, IAsyncMessageHandler<TMessage> handler, AsyncMessageHandlerFilter<TMessage>[] filters, CancellationToken cancellationToken = default);
}
```

`IAsyncPublisher` 仅用于进程内通信。通过网络进行的处理在本质上有区别，因此需要使用不同的接口以避免混淆。

网络通信需通过 Redis 等提供支持：

> PM> Install-Package [MessagePipe.Redis](https://www.nuget.org/packages/MessagePipe.Redis)

调用 `AddMessagePipeRedis` 启用 Redis 支持：

```csharp
Host.CreateDefaultBuilder()
    .ConfigureServices((ctx, services) =>
    {
        services.AddMessagePipe()
            .AddRedis(IConnectionMultiplexer | IConnectionMultiplexerFactory, configure);
    })
```

可直接传入  [StackExchange.Redis](https://github.com/StackExchange/StackExchange.Redis) 的 `ConnectionMultiplexer`，或通过 `IConnectionMultiplexerFactory` 来按键分配和管理连接池。

通过 `MessagePipeRedisOptions` 可配置序列化方式：

```csharp
public sealed class MessagePipeRedisOptions
{
    public IRedisSerializer RedisSerializer { get; set; }
}

public interface IRedisSerializer
{
    byte[] Serialize<T>(T value);
    T Deserialize<T>(byte[] value);
}
```


默认使用  [MessagePack for C#](https://github.com/neuecc/MessagePack-CSharp) 的 `ContractlessStandardResolver`，可通过 `new MessagePackRedisSerializer(options)` 使用其他的 `MessagePackSerializerOptions` 进行更换配置或实现自定义序列化。

本地测试时，可使用内存实现的分布式发布订阅：

```csharp
Host.CreateDefaultBuilder()
    .ConfigureServices((ctx, services) =>
    {
        var config = ctx.Configuration.Get<MyConfig>();

        var builder = services.AddMessagePipe();
        if (config.IsLocal)
        {
            // 本地内存模式
            builder.AddInMemoryDistributedMessageBroker();   
        }
        else
        {
            // 生产环境使用 Redis
            builder.AddRedis();
        }
    });
```

进程间通信与远程调用（InterprocessPubSub, IRemoteAsyncRequest / MessagePipe.Interprocess）
---
若需跨进程通信（如 NamedPipe/UDP/TCP），可使用  `IDistributedPublisher<TKey, TMessage>`, `IDistributedSubscriber<TKey, TMessage>` 类似于 `MessagePipe.Redis`.

> PM> Install-Package MessagePipe.Interprocess

Unity 中支持 MessagePipe.Interprocess（NamedPipe 除外）。

使用 `AddUdpInterprocess`、`AddTcpInterprocess`、`AddNamedPipeInterprocess`、`AddUdpInterprocessUds`、`AddTcpInterprocessUds` 启用进程间提供程序（Uds 是 Unix 域套接字，是性能最好的选项）。

```csharp
Host.CreateDefaultBuilder()
    .ConfigureServices((ctx, services) =>
    {
        services.AddMessagePipe()
            .AddUdpInterprocess("127.0.0.1", 3215, configure); // 配置 IP 和端口
            // .AddTcpInterprocess("127.0.0.1", 3215, configure);
            // .AddNamedPipeInterprocess("messagepipe-namedpipe", configure);
            // .AddUdpInterprocessUds("domainSocketPath")	// Unix 域套接字
            // .AddTcpInterprocessUds("domainSocketPath")
    })
```
发布与订阅示例：
```csharp
public async P(IDistributedPublisher<string, int> publisher)
{
    // 向远端进程发布消息
    await publisher.PublishAsync("foobar", 100);
}

public async S(IDistributedSubscriber<string, int> subscriber)
{
    // 订阅指定键的消息
    await subscriber.SubscribeAsync("foobar", x =>
    {
        Console.WriteLine(x);
    });
}
```

注入 `IDistributedPublisher` 的进程将作为 `server` 服务端监听连接，注入 `IDistributedSubscriber` 的进程作为 `client` 客户端。当 DI 作用域结束时，连接自动关闭。

- **UDP**：无连接协议，消息大小限制为 64KB，适合小数据量场景。
- **NamedPipe**：仅支持 1:1 连接。
- **TCP**：无限制，灵活性最高。

In default uses [MessagePack for C#](https://github.com/neuecc/MessagePack-CSharp)'s `ContractlessStandardResolver` for message serialization. You can change to use other `MessagePackSerializerOptions` by MessagePipeInterprocessOptions.MessagePackSerializerOptions.
默认使用 [MessagePack for C#](https://github.com/neuecc/MessagePack-CSharp) 的 `ContractlessStandardResolver` 进行消息序列化，可通过 MessagePipeInterprocessOptions.MessagePackSerializerOptions 中的 `MessagePipeInterprocessOptions` 更换配置。

```csharp
builder.AddUdpInterprocess("127.0.0.1", 3215, options =>
{
    // 可以配置为其他选项， `InstanceLifetime` 以及 `UnhandledErrorHandler`.
    options.MessagePackSerializerOptions = StandardResolver.Options;
});
```

若需实现 IPC-RPC 功能，可通过 `IRemoteRequestHandler<in TRequest, TResponse>` 调用远程处理器`IAsyncRequestHandler<TRequest, TResponse>` ，使用 `TcpInterprocess` 或 `NamedPipeInterprocess` 开启：

```csharp
Host.CreateDefaultBuilder()
    .ConfigureServices((ctx, services) =>
    {
        services.AddMessagePipe()
            .AddTcpInterprocess("127.0.0.1", 3215, x =>
            {
                x.HostAsServer = true; // 如果远程进程是服务器，则设置为 true（否则为 false（默认值））。
            });
    });
```

```csharp
// 服务端处理器示例：
public class MyAsyncHandler : IAsyncRequestHandler<int, string>
{
    public async ValueTask<string> InvokeAsync(int request, CancellationToken cancellationToken = default)
    {
        await Task.Delay(1);
        if (request == -1)
        {
            throw new Exception("NO -1");
        }
        else
        {
            return "ECHO:" + request.ToString();
        }
    }
}
```

```csharp
// 客户端调用示例：
async void A(IRemoteRequestHandler<int, string> remoteHandler)
{
    var v = await remoteHandler.InvokeAsync(9999);
    Console.WriteLine(v); // ECHO:9999
}
```

在 Unity 中需额外导入 MessagePack 包并调整配置：

```csharp
// VContainer 配置示例
var builder = new ContainerBuilder();
var options = builder.RegisterMessagePipe(configure);

var messagePipeBuilder = builder.ToMessagePipeBuilder(); // 要求转换 ServiceCollection 以启用 Intereprocess

var interprocessOptions = messagePipeBuilder.AddTcpInterprocess();

// 手动注册分布式组件
// IDistributedPublisher/Subscriber
messagePipeBuilder.RegisterTcpInterprocessMessageBroker<int, int>(interprocessOptions);
// 注册远程处理器
builder.RegisterAsyncRequestHandler<int, string, MyAsyncHandler>(options); // 服务端
messagePipeBuilder.RegisterTcpRemoteRequestHandler<int, string>(interprocessOptions); // 客户端
```

MessagePipeOptions
---
可以通过在 `AddMessagePipe(Action<MessagePipeOptions> configure)` 中配置 `MessagePipeOptions` 来调整 MessagePipe 的行为。

```csharp
Host.CreateDefaultBuilder()
    .ConfigureServices((ctx, services) =>
    {
        // var config = ctx.Configuration.Get<MyConfig>(); // 可选：从配置中获取设置（用于选项配置）

        services.AddMessagePipe(options =>
        {
            options.InstanceLifetime = InstanceLifetime.Scoped; // 设置实例生命周期为作用域
#if DEBUG
            // EnableCaptureStackTrace 会降低性能，建议仅在 DEBUG 模式或性能分析时启用
            options.EnableCaptureStackTrace = true;
#endif
        });
    })
```

选项包含以下属性（及方法）：

```csharp
public sealed class MessagePipeOptions
{
    AsyncPublishStrategy DefaultAsyncPublishStrategy; // 默认值：Parallel（并行）
    HandlingSubscribeDisposedPolicy HandlingSubscribeDisposedPolic; // 默认值：Ignore（忽略）
    InstanceLifetime InstanceLifetime; // 默认值：Singleton（单例）
    InstanceLifetime RequestHandlerLifetime; // 默认值：Scoped（作用域）
    bool EnableAutoRegistration;  // 默认值：true
    bool EnableCaptureStackTrace; // 默认值：false

    void SetAutoRegistrationSearchAssemblies(params Assembly[] assemblies); // 设置自动注册的程序集
    void SetAutoRegistrationSearchTypes(params Type[] types); // 设置自动注册的类型
    void AddGlobal***Filter<T>(); // 添加全局过滤器
}

public enum AsyncPublishStrategy
{
    Parallel, // 并行处理
    Sequential // 顺序处理
}

public enum InstanceLifetime
{
    Singleton, Scoped, Transient
}

public enum HandlingSubscribeDisposedPolicy
{
    Ignore, // 忽略已释放的订阅
    Throw // 抛出异常
}
```

### DefaultAsyncPublishStrategy

`IAsyncPublisher` 提供 `PublishAsync` 方法。若策略为 `Sequential`，则按顺序等待每个订阅者；若为 `Parallel`，则使用 `WhenAll` 并行处理。

```csharp
public interface IAsyncPublisher<TMessage>
{
    // 使用默认的 AsyncPublishStrategy
    ValueTask PublishAsync(TMessage message, CancellationToken cancellationToken = default);
    ValueTask PublishAsync(TMessage message, AsyncPublishStrategy publishStrategy, CancellationToken cancellationToken = default);
    // 其他方法省略...
}

public interface IAsyncPublisher<TKey, TMessage>
    where TKey : notnull
{
    // 使用默认的 AsyncPublishStrategy
    ValueTask PublishAsync(TKey key, TMessage message, CancellationToken cancellationToken = default);
    ValueTask PublishAsync(TKey key, TMessage message, AsyncPublishStrategy publishStrategy, CancellationToken cancellationToken = default);
    // 其他方法省略...
}

public interface IAsyncRequestAllHandler<in TRequest, TResponse>
{
    // 使用默认的 AsyncPublishStrategy
    ValueTask<TResponse[]> InvokeAllAsync(TRequest request, CancellationToken cancellationToken = default);
    ValueTask<TResponse[]> InvokeAllAsync(TRequest request, AsyncPublishStrategy publishStrategy, CancellationToken cancellationToken = default);
    // 其他方法省略...
}
```

`MessagePipeOptions.DefaultAsyncPublishStrategy` 的默认值为 `Parallel`。

### HandlingSubscribeDisposedPolicy

当在 MessageBroker（发布/订阅管理器）已被释放（例如作用域结束）后调用 `ISubscriber.Subscribe` 时，可选择 `Ignore`（返回空的 `IDisposable`）或 `Throw` 异常。默认值为 `Ignore`。

### InstanceLifetime

配置 MessageBroker 在 DI 容器中的生命周期，可选 `Singleton` 或 `Scoped`。默认值为 `Singleton`。若选择 `Scoped`，每个作用域内的 MessageBroker 管理独立的订阅者，作用域释放时会取消所有订阅。

### RequestHandlerLifetime

配置 `IRequestHandler`/`IAsyncRequestHandler` 在 DI 容器中的生命周期，可选 `Singleton`、`Scoped` 或 `Transient`。默认值为 `Scoped`。

### EnableAutoRegistration/SetAutoRegistrationSearchAssemblies/SetAutoRegistrationSearchTypes

启动时自动将 `IRequestHandler`、`IAsyncHandler` 及过滤器注册到 DI 容器。默认启用（`true`），搜索范围为当前域的所有程序集和类型。若因程序集裁剪导致自动注册失败，可通过在 `SetAutoRegistrationSearchAssemblies` 或 `SetAutoRegistrationSearchTypes` 中显式添加来指定搜索目标。

通过 `[IgnoreAutoRegistration]` 特性可禁用特定类型的自动注册。

### EnableCaptureStackTrace

详见 [管理订阅与诊断](#管理订阅与诊断) 章节，若启用（`true`），在订阅时捕获堆栈跟踪，便于调试但会影响性能。默认值为 `false`，建议仅在调试时开启。

### AddGlobal***Filter

添加全局过滤器，例如日志过滤器：

```csharp
public class LoggingFilter<T> : MessageHandlerFilter<T>
{
    readonly ILogger<LoggingFilter<T>> logger;

    public LoggingFilter(ILogger<LoggingFilter<T>> logger)
    {
        this.logger = logger;
    }

    public override void Handle(T message, Action<T> next)
    {
        try
        {
            logger.LogDebug("调用前");
            next(message);
            logger.LogDebug("调用完成");
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "错误");
        }
    }
}
```

使用开放泛型注册全局过滤器：

```csharp
Host.CreateDefaultBuilder()
    .ConfigureServices((ctx, services) =>
    {
        services.AddMessagePipe(options =>
        {
            // 使用 typeof(Filter<>, order);
            options.AddGlobalMessageHandlerFilter(typeof(LoggingFilter<>), -10000);
        });
    });
```

全局服务提供者
---
若需从全局作用域获取发布者/订阅者，可在运行前通过静态工具类 `GlobalMessagePipe` 设置全局服务提供者 `IServiceProvider` ：

```csharp
var host = Host.CreateDefaultBuilder()
    .ConfigureServices((ctx, x) =>
    {
        x.AddMessagePipe();
    })
    .Build(); // build host before run.

GlobalMessagePipe.SetProvider(host.Services);  // 设置全局服务提供者

await host.RunAsync(); // run framework.
```

通过 `GlobalMessagePipe` 的静态方法（如 `GetPublisher<T>`, `GetSubscriber<T>`, `CreateEvent<T>`,...）可直接获取全局实例。

![image](https://user-images.githubusercontent.com/46207/116521078-7c00de00-a90e-11eb-85c0-2c62c140c51d.png)

与其他 DI 库集成
---
主流 DI 库均支持 `Microsoft.Extensions.DependencyInjection` 桥接，可通过 MS.E.DI 配置后使用桥接接口。

与 Channels 的对比
---
[System.Threading.Channels](https://docs.microsoft.com/en-us/dotnet/api/system.threading.channels)(for Unity, `UniTask.Channels`) 基于队列实现，生产者不受消费者性能影响，支持流量控制（背压）。这与 MessagePipe 的发布/订阅模式适用场景不同。

Unity
---
需安装 Core 库，并选择 [VContainer](https://github.com/hadashiA/VContainer/)、[Zenject](https://github.com/modesttree/Zenject) 或 `BuiltinContainerBuilder` 作为运行时依赖。可通过 UPM Git URL 或 [发布页面](https://github.com/Cysharp/MessagePipe/releases) 的 Unity 包安装：

* Core `https://github.com/Cysharp/MessagePipe.git?path=src/MessagePipe.Unity/Assets/Plugins/MessagePipe`
* VContainer `https://github.com/Cysharp/MessagePipe.git?path=src/MessagePipe.Unity/Assets/Plugins/MessagePipe.VContainer`
* Zenject `https://github.com/Cysharp/MessagePipe.git?path=src/MessagePipe.Unity/Assets/Plugins/MessagePipe.Zenject`

同时需安装 [UniTask](https://github.com/Cysharp/UniTask)，所有 `.NET` 中的 `ValueTask` 在此实现中均替换为 `UniTask`。

> [!NOTE]
> Unity 版本由于 IL2CPP 限制不支持开放泛型，且无法自动注册。所有消息类型需手动配置。

VContainer 配置示例：

```csharp
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // 注册MessagePipe并获取配置选项
        var options = builder.RegisterMessagePipe(/* configure option */);
        
        // 设置GlobalMessagePipe以启用诊断窗口和全局功能
        builder.RegisterBuildCallback(c => GlobalMessagePipe.SetProvider(c.AsServiceProvider()));

        // RegisterMessageBroker 注册消息代理：IPublisher<int>/ISubscriber<int>（含异步和缓冲类型）
        builder.RegisterMessageBroker<int>(options);

        // 其他支持：RegisterMessageBroker<TKey, TMessage>, RegisterRequestHandler, RegisterAsyncRequestHandler

        // 注册消息过滤器（同步/异步）：RegisterMessageHandlerFilter , 其他支持：RegisterAsyncMessageHandlerFilter, Register(Async)RequestHandlerFilter
        builder.RegisterMessageHandlerFilter<MyFilter<int>>();
        
        builder.RegisterEntryPoint<MessagePipeDemo>(Lifetime.Singleton);
    }
}
// 使用示例
public class MessagePipeDemo : VContainer.Unity.IStartable
{
    readonly IPublisher<int> publisher;
    readonly ISubscriber<int> subscriber;

    public MessagePipeDemo(IPublisher<int> publisher, ISubscriber<int> subscriber)
    {
        this.publisher = publisher;
        this.subscriber = subscriber;
    }

    public void Start()
    {
        var d = DisposableBag.CreateBuilder();
        subscriber.Subscribe(x => Debug.Log("S1:" + x)).AddTo(d); // 订阅消息
        subscriber.Subscribe(x => Debug.Log("S2:" + x)).AddTo(d);

        publisher.Publish(10);
        publisher.Publish(20);
        publisher.Publish(30);

        var disposable = d.Build();
        disposable.Dispose(); // 统一释放订阅
    }
}
```

> [!TIP]
> 若使用 Unity 2022.1+ 和 VContainer 1.14.0+，无需手动调用 `RegisterMessageBroker<>`。
> `ISubscriber<>`、`IPublisher<>` 及其异步版本将自动解析。
> 但 `IRequestHandler<>` 和 `IRequestAllHandler<>` 仍需手动注册。


Unity 版本由于 IL2CPP 限制不支持开放泛型，且无法自动注册。所有消息类型需手动配置。


Zenject 配置示例

```csharp
void Configure(DiContainer builder)
{
    // 绑定MessagePipe并获取配置选项
    var options = builder.BindMessagePipe(/* 配置选项 */);
    
    // RegisterMessageBroker 注册消息代理：IPublisher<int>/ISubscriber<int>（含异步和缓冲类型）
    builder.BindMessageBroker<int>(options);

    // 其他支持：BindMessageBroker<TKey, TMessage>, BindRequestHandler, BindAsyncRequestHandler

    // 绑定消息过滤器：BindMessageHandlerFilter, 其他支持： BindAsyncMessageHandlerFilter, Bind(Async)RequestHandlerFilter
    builder.BindMessageHandlerFilter<MyFilter<int>>();

    // 设置 GlobalMessagePipe 以启用诊断窗口和全局功能
    GlobalMessagePipe.SetProvider(builder.AsServiceProvider());
}
```

> Zenject 版本由于框架限制不支持 `InstanceScope.Singleton`，默认使用 `Scoped` 生命周期且不可更改。

`BuiltinContainerBuilder` 是 MessagePipe 内置的轻量DI容器，使用 MessagePipe 时可以无需其他DI框架。配置示例：

```csharp
var builder = new BuiltinContainerBuilder();

builder.AddMessagePipe(/* 配置选项 */);

// AddMessageBroker: 注册消息代理：IPublisher<int>/ISubscriber<int>（含异步和缓冲类型）
builder.AddMessageBroker<int>(options);

// 其他支持：AddMessageBroker<TKey, TMessage>, AddRequestHandler, AddAsyncRequestHandler

// 注册消息过滤器：AddMessageHandlerFilter: Register for filter, 其他支持：RegisterAsyncMessageHandlerFilter, Register(Async)RequestHandlerFilter
builder.AddMessageHandlerFilter<MyFilter<int>>();

// 创建提供程序并设置为全局（启用诊断窗口和全局功能）
var provider = builder.BuildServiceProvider();
GlobalMessagePipe.SetProvider(provider);

// --- 可以通过 GlobalMessagePipe 使用MessagePipe
var p = GlobalMessagePipe.GetPublisher<int>();
var s = GlobalMessagePipe.GetSubscriber<int>();

var d = s.Subscribe(x => Debug.Log(x));

p.Publish(10);
p.Publish(20);
p.Publish(30);

d.Dispose();
```

> BuiltinContainerBuilder 不支持 scope（总是 `InstanceScope.Singleton`）、`IRequestAllHandler/IAsyncRequestAllHandler` 和其他很多 DI 功能，因此建议在 BuiltinContainerBuilder 中使用 `GlobalMessagePipe`。

由于无法使用开放泛型过滤器，可通过辅助方法简化注册：

```csharp
// 注册 IPublisher<T>/ISubscriber<T> 及全局过滤器
static void RegisterMessageBroker<T>(IContainerBuilder builder, MessagePipeOptions options)
{
    builder.RegisterMessageBroker<T>(options);

    // 设置全局过滤器
    options.AddGlobalMessageHandlerFilter<MyMessageHandlerFilter<T>>();
}

// 注册请求处理器 IRequestHandler<TReq, TRes>/IRequestAllHandler<TReq, TRes> 及全局过滤器
static void RegisterRequest<TRequest, TResponse, THandler>(IContainerBuilder builder, MessagePipeOptions options)
    where THandler : IRequestHandler
{
    builder.RegisterRequestHandler<TRequest, TResponse, THandler>(options);
    
    // 设置全局过滤器
    options.AddGlobalRequestHandlerFilter<MyRequestHandlerFilter<TRequest, TResponse>>();
}
```

可通过 `GlobalMessagePipe` 和 `MessagePipe Diagnostics` 窗口管理订阅状态，详见：[全局服务提供者](#全局服务提供者) and [管理订阅与诊断](#管理订阅与诊断) 章节.

许可证协议
---
基于 MIT 协议开源。
