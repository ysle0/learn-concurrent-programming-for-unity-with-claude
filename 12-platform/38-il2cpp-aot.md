# 38. IL2CPP & AOT

## 개요

**IL2CPP**(Intermediate Language To C++)는 Unity의 스크립팅 백엔드로, C# 코드를 C++ 소스 코드로 변환한 뒤 네이티브 바이너리로 컴파일합니다. **AOT(Ahead-Of-Time) 컴파일** 방식을 사용하므로 런타임에 동적 코드 생성이 불가능하며, 이로 인해 동시성 프로그래밍에서도 다양한 제약과 고려사항이 발생합니다.

### Mono vs IL2CPP 비교

| 항목 | Mono | IL2CPP |
|------|------|--------|
| 컴파일 방식 | JIT (Just-In-Time) | AOT (Ahead-Of-Time) |
| 런타임 코드 생성 | 가능 | **불가능** |
| 실행 성능 | 보통 | 우수 (네이티브 코드) |
| 빌드 시간 | 빠름 | 느림 (C++ 컴파일 포함) |
| 바이너리 크기 | 작음 | 큼 (C++ 런타임 포함) |
| 플랫폼 지원 | 제한적 | iOS, 콘솔 등 필수 |
| 리플렉션 | 완전 지원 | 제한적 |
| 제네릭 | 완전 지원 | 일부 제한 |

### AOT 컴파일 파이프라인

```
C# 소스 코드
    ↓ (Roslyn 컴파일러)
IL (Intermediate Language) / .NET 어셈블리
    ↓ (IL2CPP 변환기)
C++ 소스 코드
    ↓ (네이티브 C++ 컴파일러: clang, MSVC 등)
네이티브 바이너리 (.so, .a, .dll)
```

> **중요**: iOS, WebGL, 대부분의 콘솔 플랫폼은 IL2CPP가 **필수**입니다. 동시성 코드를 작성할 때 반드시 IL2CPP 호환성을 고려해야 합니다.

---

## 1. 코드 스트리핑 (Managed Code Stripping)

IL2CPP 빌드 시 사용되지 않는 코드를 제거하여 바이너리 크기를 줄이는 과정입니다. 동시성 라이브러리를 사용할 때 스트리핑으로 인해 필수 코드가 제거되는 문제가 빈번히 발생합니다.

### 스트리핑 레벨

```
Project Settings → Player → Other Settings → Managed Stripping Level
```

| 레벨 | 설명 | 제거 범위 |
|------|------|-----------|
| Minimal | 최소한의 스트리핑 | 명백히 사용되지 않는 코드만 제거 |
| Low | 낮은 수준 | 도달 불가능한 코드 제거 |
| Medium | 중간 수준 | 미사용 타입 멤버 추가 제거 |
| High | 높은 수준 (기본값) | 적극적 제거, 리플렉션 코드 포함 |

### 스트리핑 문제 발생 예시

```csharp
using System;
using System.Threading.Tasks;
using UnityEngine;

public class StrippingIssueExample : MonoBehaviour
{
    // ❌ 리플렉션으로만 사용되는 타입 - 스트리핑될 수 있음
    private async Task ProcessDataAsync()
    {
        var data = await FetchDataAsync();

        // 리플렉션 기반 역직렬화 - IL2CPP에서 문제 가능
        var type = Type.GetType("MyNamespace.DataProcessor");
        if (type != null)
        {
            var processor = Activator.CreateInstance(type);
            var method = type.GetMethod("Process");
            method?.Invoke(processor, new object[] { data });
        }
    }

    // ✅ 직접 참조로 변경 - 스트리핑 방지
    private async Task ProcessDataSafeAsync()
    {
        var data = await FetchDataAsync();
        var processor = new DataProcessor();
        processor.Process(data);
    }

    private Task<byte[]> FetchDataAsync()
        => Task.FromResult(new byte[0]);
}

public class DataProcessor
{
    public void Process(byte[] data)
        => Debug.Log($"Processing {data.Length} bytes");
}
```

---

## 2. link.xml 설정 (스트리핑 방지)

`link.xml` 파일을 사용하여 특정 어셈블리, 타입, 메서드가 스트리핑되지 않도록 보호할 수 있습니다.

### 동시성 라이브러리용 link.xml

```xml
<!-- Assets/link.xml -->
<linker>
    <!-- 핵심 동시성 타입 보존 -->
    <assembly fullname="mscorlib">
        <!-- async/await 인프라 -->
        <type fullname="System.Runtime.CompilerServices.AsyncTaskMethodBuilder" preserve="all"/>
        <type fullname="System.Runtime.CompilerServices.AsyncTaskMethodBuilder`1" preserve="all"/>
        <type fullname="System.Runtime.CompilerServices.TaskAwaiter" preserve="all"/>
        <type fullname="System.Runtime.CompilerServices.TaskAwaiter`1" preserve="all"/>

        <!-- 동기화 프리미티브 -->
        <type fullname="System.Threading.SemaphoreSlim" preserve="all"/>
        <type fullname="System.Threading.CancellationTokenSource" preserve="all"/>
    </assembly>

    <!-- 확장 라이브러리 -->
    <assembly fullname="System.Threading.Tasks" preserve="all"/>
    <assembly fullname="System.Threading.Channels" preserve="all"/>
    <assembly fullname="System.Collections.Concurrent" preserve="all"/>

    <!-- UniTask 관련 보존 -->
    <assembly fullname="UniTask" preserve="all"/>
    <assembly fullname="UniTask.Linq" preserve="all"/>

    <!-- 직렬화 라이브러리 보존 -->
    <assembly fullname="MessagePack" preserve="all"/>
    <assembly fullname="MessagePack.Resolvers.GeneratedResolver" preserve="all"/>

    <!-- 프로젝트 어셈블리 (리플렉션 사용 시) -->
    <assembly fullname="Assembly-CSharp">
        <type fullname="MyGame.Networking.*" preserve="all"/>
    </assembly>
</linker>
```

### [Preserve] 어트리뷰트 활용

```csharp
using UnityEngine;
using UnityEngine.Scripting;

// 클래스 전체 보존
[Preserve]
public class NetworkMessageHandler
{
    [Preserve]
    public async void HandleMessageAsync(byte[] data)
    {
        await ProcessAsync(data);
    }

    [Preserve]
    private async System.Threading.Tasks.Task ProcessAsync(byte[] data)
    {
        await System.Threading.Tasks.Task.Delay(100);
        Debug.Log($"Processed {data.Length} bytes");
    }
}

// 제네릭 타입 보존
[Preserve]
public class AsyncMessageQueue<T> where T : struct
{
    private readonly System.Collections.Concurrent.ConcurrentQueue<T> _queue = new();

    [Preserve]
    public void Enqueue(T item) => _queue.Enqueue(item);

    [Preserve]
    public bool TryDequeue(out T result) => _queue.TryDequeue(out result);
}
```

---

## 3. 리플렉션 제한

AOT 환경에서는 컴파일 시점에 모든 타입 정보가 결정되어야 합니다. 리플렉션을 통한 동적 타입 접근은 심각한 제한을 받습니다.

### 문제가 되는 패턴

```csharp
using System;
using System.Threading.Tasks;
using UnityEngine;

public class ReflectionIssuesExample : MonoBehaviour
{
    // ❌ 동적 메서드 호출 - AOT에서 실패 가능
    public async Task InvokeHandlerBad(string handlerTypeName, object data)
    {
        var type = Type.GetType(handlerTypeName);
        var instance = Activator.CreateInstance(type);
        var method = type.GetMethod("HandleAsync");

        // MakeGenericMethod는 AOT에서 특히 위험
        var genericMethod = method.MakeGenericMethod(data.GetType());
        var task = (Task)genericMethod.Invoke(instance, new[] { data });
        await task;
    }

    // ❌ Expression 트리 컴파일 - AOT에서 불가
    public void CreateDynamicDelegate()
    {
        var param = System.Linq.Expressions.Expression.Parameter(typeof(int));
        var lambda = System.Linq.Expressions.Expression.Lambda<Func<int, int>>(
            System.Linq.Expressions.Expression.Multiply(param, param),
            param
        );
        // var compiled = lambda.Compile(); // ❌ IL2CPP에서 런타임 에러!
    }
}
```

### AOT-safe 리플렉션 대안

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using UnityEngine;

// ✅ 인터페이스 기반 핸들러 패턴 (리플렉션 불필요)
public interface IAsyncMessageHandler<T>
{
    Task HandleAsync(T message);
}

// ✅ 수동 등록 기반 디스패처
public class AotSafeMessageDispatcher
{
    private readonly Dictionary<Type, object> _handlers = new();

    public void Register<T>(IAsyncMessageHandler<T> handler)
    {
        _handlers[typeof(T)] = handler;
    }

    public async Task DispatchAsync<T>(T message)
    {
        if (_handlers.TryGetValue(typeof(T), out var handler))
        {
            await ((IAsyncMessageHandler<T>)handler).HandleAsync(message);
        }
        else
        {
            Debug.LogWarning($"No handler for {typeof(T).Name}");
        }
    }
}

// 사용 예시
public struct ChatMessage { public string Content; public string Sender; }

public class ChatMessageHandler : IAsyncMessageHandler<ChatMessage>
{
    public async Task HandleAsync(ChatMessage message)
    {
        await Task.Delay(10);
        Debug.Log($"Chat from {message.Sender}: {message.Content}");
    }
}

public class GameBootstrap : MonoBehaviour
{
    private readonly AotSafeMessageDispatcher _dispatcher = new();

    private void Awake()
    {
        // ✅ 명시적 등록 - AOT 안전
        _dispatcher.Register(new ChatMessageHandler());
    }

    private async void Start()
    {
        await _dispatcher.DispatchAsync(new ChatMessage
        {
            Content = "Hello!", Sender = "Player1"
        });
    }
}
```

---

## 4. 제네릭 제한

IL2CPP는 AOT 컴파일 특성상 제네릭 인스턴스화에 제한이 있습니다. 특히 **값 타입(Value Type) 제네릭**은 각 타입 조합마다 별도의 네이티브 코드가 생성되어야 하므로, 컴파일 시점에 사용될 모든 조합을 알아야 합니다.

### 문제 발생 시나리오

```csharp
using System;
using System.Threading.Tasks;
using System.Collections.Concurrent;
using UnityEngine;

public class GenericLimitationsExample : MonoBehaviour
{
    // ❌ 런타임에 결정되는 값 타입 제네릭 - AOT에서 실패 가능
    public async Task ProcessGenericBad(Type valueType)
    {
        var queueType = typeof(ConcurrentQueue<>).MakeGenericType(valueType);
        var queue = Activator.CreateInstance(queueType);
        await Task.CompletedTask;
    }

    // ✅ 컴파일 타임에 타입이 결정되는 제네릭
    public async Task ProcessGenericGood()
    {
        var intQueue = new ConcurrentQueue<int>();
        var floatQueue = new ConcurrentQueue<float>();
        var vectorQueue = new ConcurrentQueue<Vector3>();

        intQueue.Enqueue(42);
        floatQueue.Enqueue(3.14f);
        vectorQueue.Enqueue(Vector3.zero);

        await Task.CompletedTask;
    }
}
```

### 제네릭 인스턴스 강제 생성 (Generic Forcing)

```csharp
using System.Collections.Concurrent;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.Scripting;

// 이 클래스는 실제로 호출되지 않지만, IL2CPP가 제네릭 인스턴스를
// 생성하도록 강제합니다.
[Preserve]
public static class AotGenericForcer
{
    [Preserve]
    private static void ForceGenericInstantiations()
    {
        // ConcurrentQueue 값 타입 인스턴스
        ForceType<ConcurrentQueue<int>>();
        ForceType<ConcurrentQueue<float>>();
        ForceType<ConcurrentQueue<Vector3>>();
        ForceType<ConcurrentQueue<Quaternion>>();

        // ConcurrentDictionary 인스턴스
        ForceType<ConcurrentDictionary<int, Vector3>>();
        ForceType<ConcurrentDictionary<string, int>>();

        // Task<T> 및 TaskCompletionSource<T> 인스턴스
        ForceType<Task<int>>();
        ForceType<Task<float>>();
        ForceType<Task<bool>>();
        ForceType<Task<Vector3>>();
        ForceType<TaskCompletionSource<int>>();
        ForceType<TaskCompletionSource<bool>>();
    }

    [Preserve]
    private static void ForceType<T>()
    {
        System.Runtime.CompilerServices.RuntimeHelpers
            .RunClassConstructor(typeof(T).TypeHandle);
    }
}
```

### 자주 발생하는 제네릭 관련 오류

```
// 1. ExecutionEngineException
"Attempting to call method
'ConcurrentQueue`1<MyStruct>::Enqueue'
for which no ahead of time (AOT) code was generated."

// 2. TypeInitializationException
"The type initializer for
'Task`1<MyValueType>' threw an exception."

// 해결: AotGenericForcer에 해당 타입 조합 추가
```

---

## 5. 동적 코드 생성 불가

IL2CPP 환경에서는 `System.Reflection.Emit` 네임스페이스의 모든 기능이 **사용 불가능**합니다.

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using UnityEngine;

public class DynamicCodeAlternatives : MonoBehaviour
{
    // ❌ DynamicMethod, ILGenerator - IL2CPP에서 PlatformNotSupportedException
    // ❌ Expression.Compile() - IL2CPP에서 런타임 에러

    // ✅ 대안: 컴파일 타임에 결정되는 delegate 사용
    private static readonly Func<int, int, int> Add = (a, b) => a + b;
    private static readonly Func<int, int, int> Multiply = (a, b) => a * b;

    // ✅ 대안: Strategy 패턴으로 동적 행동 구현
    private readonly Dictionary<string, Func<int, int, Task<int>>> _operations = new();

    private void SetupOperations()
    {
        _operations["add"] = async (a, b) =>
        {
            await Task.Yield();
            return a + b;
        };
        _operations["multiply"] = async (a, b) =>
        {
            await Task.Yield();
            return a * b;
        };
    }

    private async Task<int> ExecuteOperation(string op, int a, int b)
    {
        if (_operations.TryGetValue(op, out var operation))
            return await operation(a, b);
        throw new InvalidOperationException($"Unknown operation: {op}");
    }
}
```

---

## 6. async/await와 IL2CPP

C#의 `async/await`는 컴파일러가 **상태 머신(State Machine)**을 자동 생성합니다. IL2CPP에서는 이 상태 머신이 C++ 코드로 변환되며, 몇 가지 특수한 고려사항이 있습니다.

### 상태 머신 코드 생성 이해

```csharp
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

public class AsyncStateMachineExample : MonoBehaviour
{
    // 이 async 메서드는 컴파일러에 의해 상태 머신 구조체로 변환됩니다.
    // IL2CPP는 이 구조체를 C++ struct로 변환합니다.
    public async Task<int> LoadAndProcessAsync(CancellationToken ct)
    {
        var rawData = await DownloadDataAsync(ct);     // State 0
        var processed = await ParseDataAsync(rawData, ct);  // State 1
        await SaveResultAsync(processed, ct);               // State 2
        return processed.Length;
    }

    // 컴파일러가 내부적으로 생성하는 구조 (개념적 설명):
    //
    // [CompilerGenerated]
    // struct <LoadAndProcessAsync>d__0 : IAsyncStateMachine
    // {
    //     public int <>1__state;
    //     public AsyncTaskMethodBuilder<int> <>t__builder;
    //     public CancellationToken ct;
    //     private byte[] <rawData>5__1;       // 로컬 변수 캡처
    //     private int[] <processed>5__2;
    //     private TaskAwaiter<byte[]> <>u__1;  // awaiter
    //     public void MoveNext() { /* 상태 머신 로직 */ }
    // }
    //
    // IL2CPP는 이 struct를 C++ struct로 변환합니다.
    // 각 await 지점마다 상태 전이 코드가 생성됩니다.

    private Task<byte[]> DownloadDataAsync(CancellationToken ct)
        => Task.FromResult(new byte[] { 1, 2, 3 });
    private Task<int[]> ParseDataAsync(byte[] data, CancellationToken ct)
        => Task.FromResult(new int[] { data.Length });
    private Task SaveResultAsync(int[] result, CancellationToken ct)
        => Task.CompletedTask;
}
```

### async/await IL2CPP 최적화 팁

```csharp
using System.Threading.Tasks;
using UnityEngine;

public class AsyncIL2CPPOptimization : MonoBehaviour
{
    // ❌ 불필요한 async - 상태 머신 오버헤드
    private async Task<int> GetValueBad()
    {
        return await Task.FromResult(42);
    }

    // ✅ 동기적으로 완료되는 경우 async 제거
    private Task<int> GetValueGood()
    {
        return Task.FromResult(42);
    }

    // ✅ 독립적인 작업은 병렬 실행으로 상태 전이 감소
    private async Task ProcessOptimized()
    {
        var a = await Step1Async();

        // 독립적인 작업은 WhenAll로 병렬화
        var task1 = Step2Async(a);
        var task2 = Step3Async(a);
        await Task.WhenAll(task1, task2);

        await Step4Async(task1.Result + task2.Result);
    }

    // ✅ ValueTask 사용으로 동기 완료 시 할당 감소
    private ValueTask<int> GetCachedValue(int key)
    {
        if (_cache.TryGetValue(key, out var value))
            return new ValueTask<int>(value); // 할당 없음
        return new ValueTask<int>(LoadValueAsync(key));
    }

    private readonly System.Collections.Generic.Dictionary<int, int> _cache = new();

    private async Task<int> LoadValueAsync(int key)
    {
        await Task.Delay(100);
        var value = key * 2;
        _cache[key] = value;
        return value;
    }

    private Task<int> Step1Async() => Task.FromResult(1);
    private Task<int> Step2Async(int v) => Task.FromResult(v + 1);
    private Task<int> Step3Async(int v) => Task.FromResult(v + 2);
    private Task<int> Step4Async(int v) => Task.FromResult(v + 3);
}
```

---

## 7. UniTask의 IL2CPP 호환성

UniTask는 설계 단계부터 IL2CPP/AOT 환경을 고려하여 만들어진 라이브러리입니다. 대부분의 기능이 AOT 안전하지만, 일부 주의사항이 있습니다.

```csharp
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Linq;
using System.Threading;
using UnityEngine;

public class UniTaskAotExample : MonoBehaviour
{
    // ✅ UniTask는 struct 기반으로 AOT 호환
    private async UniTask LoadGameAsync(CancellationToken ct)
    {
        await UniTask.Delay(1000, cancellationToken: ct);

        // ✅ UniTask.WhenAll - AOT 안전 (최대 15개 오버로드 제공)
        var (result1, result2, result3) = await UniTask.WhenAll(
            LoadAssetAsync("texture", ct),
            LoadAssetAsync("audio", ct),
            LoadAssetAsync("prefab", ct)
        );

        Debug.Log($"Loaded: {result1}, {result2}, {result3}");
    }

    // ✅ UniTask<T>는 값 타입에 대해서도 AOT 안전
    private async UniTask<string> LoadAssetAsync(string name, CancellationToken ct)
    {
        await UniTask.Delay(100, cancellationToken: ct);
        return $"Asset_{name}";
    }

    // ✅ Channel - AOT 안전
    private async UniTask ChannelExample(CancellationToken ct)
    {
        var channel = Channel.CreateSingleConsumerUnbounded<int>();

        UniTask.Void(async () =>
        {
            for (int i = 0; i < 10; i++)
            {
                channel.Writer.TryWrite(i);
                await UniTask.Delay(100, cancellationToken: ct);
            }
            channel.Writer.TryComplete();
        });

        await channel.Reader.ReadAllAsync(ct).ForEachAsync(item =>
        {
            Debug.Log($"Received: {item}");
        }, ct);
    }

    // ❌ 주의: UniTask.Linq에서 복잡한 제네릭 체이닝은
    //    값 타입 셀렉터 사용 시 AOT 미생성 위험 - 빌드 테스트 필요
    private async UniTask ComplexLinqCaution()
    {
        await UniTaskAsyncEnumerable.EveryUpdate()
            .Select((_, index) => new Vector3(index, 0, 0))
            .Where(v => v.x > 10)
            .Take(5)
            .ForEachAsync(v => Debug.Log(v));
    }

    // ✅ 안전한 대안: 명시적 루프 사용
    private async UniTask SafeLinqAlternative()
    {
        int count = 0;
        await foreach (var _ in UniTaskAsyncEnumerable.EveryUpdate()
            .WithCancellation(this.GetCancellationTokenOnDestroy()))
        {
            var position = new Vector3(Time.frameCount, 0, 0);
            if (position.x > 10)
            {
                Debug.Log(position);
                if (++count >= 5) break;
            }
        }
    }
}
```

---

## 8. 직렬화 라이브러리의 AOT 대응

네트워크 통신에서 사용하는 직렬화 라이브러리는 IL2CPP 환경에서 특별한 설정이 필요합니다.

### MessagePack (AOT Code Generation)

```csharp
using System;
using System.Threading.Tasks;
using UnityEngine;

// MessagePack은 mpc(MessagePack Compiler) 또는
// Source Generator를 통해 AOT 코드를 생성합니다.

// 1) 메시지 정의
// [MessagePackObject]
// public class PlayerState
// {
//     [Key(0)] public int PlayerId { get; set; }
//     [Key(1)] public Vector3 Position { get; set; }
//     [Key(2)] public float Health { get; set; }
// }

// 2) AOT Resolver 등록 (Startup에서)
public class MessagePackAotSetup
{
    // [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    // static void Initialize()
    // {
    //     var resolver = MessagePack.Resolvers.CompositeResolver.Create(
    //         MessagePack.Resolvers.GeneratedResolver.Instance, // AOT 생성
    //         MessagePack.Unity.UnityResolver.Instance,         // Unity 타입
    //         MessagePack.Resolvers.StandardResolver.Instance   // 기본 타입
    //     );
    //     var options = MessagePack.MessagePackSerializerOptions.Standard
    //         .WithResolver(resolver);
    //     MessagePack.MessagePackSerializer.DefaultOptions = options;
    // }
}

// 3) 비동기 네트워크 전송과 결합
public class NetworkSerializer
{
    // ✅ AOT-safe 직렬화/역직렬화
    public byte[] Serialize<T>(T obj) => Array.Empty<byte>();
    public T Deserialize<T>(byte[] data) => default;

    public async Task SendAsync<T>(T message)
    {
        var data = Serialize(message);
        await TransmitAsync(data);
    }

    private Task TransmitAsync(byte[] data)
    {
        Debug.Log($"Sending {data.Length} bytes");
        return Task.CompletedTask;
    }
}
```

### MemoryPack (Source Generator 네이티브 지원)

```csharp
// MemoryPack은 Source Generator를 사용하므로
// IL2CPP에서 추가 설정 없이 동작합니다.

// [MemoryPackable]
// public partial class GameEvent
// {
//     public int EventId { get; set; }
//     public string EventName { get; set; }
//     public byte[] Payload { get; set; }
// }

// [MemoryPackable]
// public partial struct TransformSnapshot
// {
//     public float PosX, PosY, PosZ;
//     public float RotX, RotY, RotZ, RotW;
// }

// ✅ Source Generator가 Serialize/Deserialize 코드를
// 컴파일 타임에 생성하므로 리플렉션이 필요 없습니다.
// var bytes = MemoryPackSerializer.Serialize(snapshot); // AOT 안전
```

---

## 9. Source Generator 활용

Source Generator는 컴파일 타임에 C# 코드를 자동 생성하는 기술로, IL2CPP/AOT 환경에서 리플렉션의 안전한 대안입니다.

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Collections.Generic;
using UnityEngine;

// =============================================
// Source Generator 활용: AOT-safe 이벤트 시스템 개념
// =============================================

// 마커 어트리뷰트 정의
[AttributeUsage(AttributeTargets.Method)]
public class HandleEventAttribute : Attribute
{
    public Type EventType { get; }
    public HandleEventAttribute(Type eventType) => EventType = eventType;
}

// ✅ Source Generator가 생성할 코드의 패턴
public static class EventHandlerRegistry
{
    private static readonly Dictionary<Type, List<Func<object, CancellationToken, Task>>>
        _handlers = new();

    public static void Register(Type eventType,
        Func<object, CancellationToken, Task> handler)
    {
        if (!_handlers.ContainsKey(eventType))
            _handlers[eventType] = new List<Func<object, CancellationToken, Task>>();
        _handlers[eventType].Add(handler);
    }

    public static async Task PublishAsync<T>(T evt, CancellationToken ct = default)
    {
        if (_handlers.TryGetValue(typeof(T), out var handlers))
            foreach (var handler in handlers)
                await handler(evt, ct);
    }
}

// 사용자가 작성하는 핸들러
public struct PlayerDamageEvent { public int PlayerId; public float Damage; }
public struct PlayerHealEvent { public int PlayerId; public float Amount; }

public class PlayerEventHandler
{
    [HandleEvent(typeof(PlayerDamageEvent))]
    public async Task OnPlayerDamage(PlayerDamageEvent evt, CancellationToken ct)
    {
        await Task.Delay(10, ct);
        Debug.Log($"Player {evt.PlayerId} took {evt.Damage} damage");
    }
}

// Source Generator가 자동 생성하는 등록 코드 (개념)
// ✅ 리플렉션 없이 컴파일 타임에 모든 핸들러가 등록됨
public static class GeneratedEventHandlerRegistration
{
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    private static void AutoRegister()
    {
        var handler = new PlayerEventHandler();
        EventHandlerRegistry.Register(typeof(PlayerDamageEvent),
            (evt, ct) => handler.OnPlayerDamage((PlayerDamageEvent)evt, ct));
    }
}
```

### Unity에서 활용 가능한 Source Generator 라이브러리

```
라이브러리               용도                     AOT 지원
────────────────────────────────────────────────────────
MemoryPack              바이너리 직렬화            ✅ 완전
MessagePack v3+         바이너리 직렬화            ✅ Source Gen
System.Text.Json (v8+)  JSON 직렬화               ✅ Source Gen
R3                      Reactive Extensions        ✅ 완전
UniTask                 비동기 처리                ✅ 완전
```

---

## 10. 빌드 시간 최적화

IL2CPP 빌드는 C++ 컴파일 과정이 포함되므로 Mono 대비 상당히 느립니다.

### 빌드 시간 단축 전략

```
1. Player Settings 최적화
   [개발 빌드 시] IL2CPP Code Generation: "Faster (smaller) builds"
   [릴리스 빌드 시] IL2CPP Code Generation: "Faster runtime"

2. 증분 빌드 활용
   같은 빌드 폴더를 유지하면 변경된 파일만 재컴파일

3. Assembly Definition 활용
   코드를 여러 asmdef로 분리하면:
   - 변경되지 않은 어셈블리의 C++ 재생성 방지
   - 병렬 컴파일 효율 향상
```

### Assembly Definition 분리 예시

```
Assets/
├── Scripts/
│   ├── Core/
│   │   ├── Core.asmdef              ← 핵심 시스템 (변경 빈도 낮음)
│   │   ├── AsyncUtilities.cs
│   │   └── ConcurrencyHelpers.cs
│   ├── Networking/
│   │   ├── Networking.asmdef         ← 네트워크 계층
│   │   └── AsyncNetworkClient.cs
│   ├── Gameplay/
│   │   ├── Gameplay.asmdef           ← 게임플레이 (변경 빈도 높음)
│   │   └── PlayerController.cs
│   └── ThirdParty/
│       └── ThirdParty.asmdef         ← 서드파티 (거의 변경 없음)
```

### 빌드 시간 프로파일링

```csharp
#if UNITY_EDITOR
using UnityEngine;
using UnityEditor;
using UnityEditor.Build;
using UnityEditor.Build.Reporting;

public class BuildTimeProfiler : IPreprocessBuildWithReport, IPostprocessBuildWithReport
{
    public int callbackOrder => 0;
    private static System.Diagnostics.Stopwatch _stopwatch;

    public void OnPreprocessBuild(BuildReport report)
    {
        _stopwatch = System.Diagnostics.Stopwatch.StartNew();
        Debug.Log($"[빌드 시작] {System.DateTime.Now:HH:mm:ss}");
        Debug.Log($"[스크립팅 백엔드] {PlayerSettings.GetScriptingBackend(
            EditorUserBuildSettings.selectedBuildTargetGroup)}");
    }

    public void OnPostprocessBuild(BuildReport report)
    {
        _stopwatch.Stop();
        var elapsed = _stopwatch.Elapsed;
        Debug.Log($"[빌드 완료] 소요 시간: {elapsed.Minutes}분 {elapsed.Seconds}초");
        Debug.Log($"[빌드 크기] {report.summary.totalSize / (1024 * 1024):F1} MB");
    }
}
#endif
```

---

## 11. 디버깅 팁

IL2CPP 빌드에서 동시성 관련 버그를 디버깅하는 것은 까다롭습니다.

### IL2CPP 빌드 디버깅 설정

```
Build Settings:
  ☑ Development Build
  ☑ Script Debugging
  ☑ Wait For Managed Debugger (선택적)

Player Settings → Other Settings:
  - IL2CPP Code Generation: "Faster (smaller) builds"
  - C++ Compiler Configuration: Debug

생성된 C++ 코드 확인 위치:
  - Windows: Library/Il2cppOutputProject/
  - 빌드 폴더: {BuildFolder}/Il2CppOutputProject/
```

### 동시성 버그 디버깅 유틸리티

```csharp
using System.Diagnostics;
using System.Runtime.CompilerServices;
using System.Threading;
using UnityEngine;
using Debug = UnityEngine.Debug;

public static class ConcurrencyDebugUtil
{
    [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
    public static void LogThread(
        string operation,
        [CallerMemberName] string caller = "",
        [CallerFilePath] string file = "",
        [CallerLineNumber] int line = 0)
    {
        var thread = Thread.CurrentThread;
        var isMainThread = thread.ManagedThreadId == 1;
        Debug.Log(
            $"[Thread] {operation}\n" +
            $"  Thread: {thread.ManagedThreadId} " +
            $"({(isMainThread ? "Main" : "Background")})\n" +
            $"  Caller: {caller} at {System.IO.Path.GetFileName(file)}:{line}");
    }

    [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
    public static void AssertMainThread([CallerMemberName] string caller = "")
    {
        if (Thread.CurrentThread.ManagedThreadId != 1)
            Debug.LogError($"[Thread Violation] {caller}은(는) 메인 스레드에서 호출되어야 합니다.");
    }

    [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
    public static void AssertBackgroundThread([CallerMemberName] string caller = "")
    {
        if (Thread.CurrentThread.ManagedThreadId == 1)
            Debug.LogWarning($"[Thread Warning] {caller}이(가) 메인 스레드에서 실행 중입니다.");
    }
}
```

### IL2CPP 관련 에러 대응 가이드

```
에러 1: ExecutionEngineException (제네릭 관련)
  "Attempting to call method ... for which no AOT code was generated"
  → 해결: AotGenericForcer에 해당 타입 추가

에러 2: TypeInitializationException
  "The type initializer for ... threw an exception"
  → 해결: 정적 생성자에서 AOT 비호환 코드 제거

에러 3: MissingMethodException
  "Method not found: ..."
  → 해결: link.xml 또는 [Preserve] 추가

에러 4: NotSupportedException (Reflection.Emit)
  "... is not supported on this platform"
  → 해결: Source Generator 또는 수동 코드로 대체

에러 5: async 메서드에서 NullReferenceException
  → 원인: 상태 머신 구조체의 필드가 스트리핑됨
  → 해결: 관련 타입을 link.xml에 추가
```

---

## 주의사항

### IL2CPP 동시성 프로그래밍 핵심 제약

1. **System.Reflection.Emit 완전 불가**: `DynamicMethod`, `TypeBuilder`, `ILGenerator` 등 동적 코드 생성 API는 사용할 수 없습니다.

2. **Expression.Compile() 불가**: LINQ Expression 트리의 런타임 컴파일이 지원되지 않습니다.

3. **제네릭 값 타입 제한**: 컴파일 시점에 사용되지 않는 값 타입 제네릭 조합은 코드가 생성되지 않습니다.

4. **코드 스트리핑 주의**: `High` 스트리핑 레벨에서는 리플렉션으로만 접근하는 타입이 제거될 수 있습니다.

5. **async 상태 머신 크기**: 복잡한 async 메서드는 큰 C++ 구조체를 생성하여 바이너리 크기를 증가시킵니다.

6. **Thread.Abort() 미지원**: IL2CPP에서 `Thread.Abort()`는 동작하지 않습니다. `CancellationToken`을 사용하세요.

7. **디버깅 어려움**: IL2CPP 빌드의 스택 트레이스는 C++ 변환 후 이름이 변경되어 가독성이 떨어집니다.

---

## 베스트 프랙티스

### AOT-safe 동시성 코드 체크리스트

```
✅ 권장 사항
───────────────────────────────────────────────
☑ CancellationToken을 모든 async 메서드에 전달
☑ UniTask 사용으로 AOT 호환성과 성능 동시 확보
☑ Source Generator 기반 직렬화 라이브러리 사용
☑ link.xml로 핵심 동시성 타입 보존
☑ Assembly Definition으로 코드 분리
☑ 제네릭 값 타입 사용 시 AotGenericForcer 작성
☑ Development Build로 IL2CPP 빌드 주기적 테스트
☑ 인터페이스 기반 추상화로 리플렉션 의존 최소화
☑ ValueTask 활용으로 동기 완료 시 할당 감소
☑ [Preserve] 어트리뷰트로 스트리핑 방지

❌ 금지 사항
───────────────────────────────────────────────
☒ System.Reflection.Emit 사용
☒ Expression.Compile() 호출
☒ MakeGenericType/MakeGenericMethod 런타임 호출 (값 타입)
☒ Type.GetType(string)으로 동적 타입 로딩 의존
☒ Thread.Abort() 사용
☒ 릴리스 빌드에서만 IL2CPP 테스트
☒ link.xml 없이 High 스트리핑 레벨 사용
☒ 불필요한 async 래핑 (상태 머신 오버헤드)
```

### 플랫폼별 빌드 전략

```
플랫폼         스크립팅 백엔드    스트리핑 권장     비고
────────────────────────────────────────────────────────────
iOS            IL2CPP (필수)     Medium~High      App Store 필수
Android        IL2CPP (권장)     Medium           Mono도 가능
WebGL          IL2CPP (필수)     Low~Medium       싱글 스레드 제한
Windows        Mono / IL2CPP    Medium           개발 시 Mono 가능
macOS          Mono / IL2CPP    Medium           개발 시 Mono 가능
콘솔 (PS/Xbox)  IL2CPP (필수)     Medium~High      플랫폼 요구사항
```

### 개발 워크플로 권장 순서

```
1. [개발 단계] Mono 백엔드로 빠른 이터레이션
       ↓
2. [주간 검증] IL2CPP Development Build로 AOT 호환성 확인
       ↓
3. [문제 발견] link.xml, [Preserve], AotGenericForcer 추가
       ↓
4. [QA 단계] IL2CPP Release Build로 전체 테스트
       ↓
5. [출시 준비] 스트리핑 레벨 최적화, 바이너리 크기 확인
```

---

## 참고 자료

- [Unity 공식 문서 - IL2CPP Overview](https://docs.unity3d.com/Manual/IL2CPP.html)
- [Unity 공식 문서 - Managed Code Stripping](https://docs.unity3d.com/Manual/ManagedCodeStripping.html)
- [Unity 공식 문서 - Scripting Restrictions](https://docs.unity3d.com/Manual/ScriptingRestrictions.html)
- [UniTask GitHub - IL2CPP 관련 이슈](https://github.com/Cysharp/UniTask)
- [MessagePack-CSharp - AOT Code Generation](https://github.com/MessagePack-CSharp/MessagePack-CSharp#aot-code-generation)
- [MemoryPack - Source Generator 직렬화](https://github.com/Cysharp/MemoryPack)
- [Unity 블로그 - IL2CPP Internals](https://blog.unity.com/engine-platform/an-introduction-to-ilcpp-internals)
- [.NET Source Generators](https://learn.microsoft.com/dotnet/csharp/roslyn-sdk/source-generators-overview)

---

> **다음 섹션**: [39. WebGL & 싱글 스레드](./39-webgl-single-thread.md) - WebGL 환경의 싱글 스레드 제약과 동시성 대안
