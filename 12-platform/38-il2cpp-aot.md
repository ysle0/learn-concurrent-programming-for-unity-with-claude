# Section 38: IL2CPP & AOT

## 개요

IL2CPP(Intermediate Language To C++)는 Unity의 스크립팅 백엔드로, C# 코드를 C++로 변환하여 네이티브 코드로 컴파일합니다. AOT(Ahead-Of-Time) 컴파일은 런타임 전에 코드를 미리 컴파일하는 방식으로, iOS와 같은 플랫폼에서 필수입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    IL2CPP 컴파일 파이프라인                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   C# 코드                                                        │
│      │                                                           │
│      ▼                                                           │
│   ┌────────────────┐                                             │
│   │   C# Compiler  │  (.cs → .dll)                               │
│   └────────────────┘                                             │
│      │                                                           │
│      ▼                                                           │
│   ┌────────────────┐                                             │
│   │    IL2CPP      │  (.dll → .cpp)                              │
│   └────────────────┘                                             │
│      │                                                           │
│      ▼                                                           │
│   ┌────────────────┐                                             │
│   │  C++ Compiler  │  (.cpp → 네이티브 코드)                      │
│   └────────────────┘                                             │
│      │                                                           │
│      ▼                                                           │
│   네이티브 실행 파일                                              │
│                                                                  │
│   Mono vs IL2CPP:                                                │
│   ┌───────────────┬─────────────┬─────────────────┐             │
│   │     특성      │    Mono     │     IL2CPP      │             │
│   ├───────────────┼─────────────┼─────────────────┤             │
│   │ 컴파일 방식   │ JIT         │ AOT             │             │
│   │ 빌드 속도     │ 빠름        │ 느림            │             │
│   │ 런타임 성능   │ 보통        │ 빠름            │             │
│   │ 코드 크기     │ 작음        │ 큼              │             │
│   │ 리플렉션      │ 완전 지원   │ 제한적          │             │
│   │ 제네릭       │ 완전 지원   │ 제한적          │             │
│   └───────────────┴─────────────┴─────────────────┘             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## IL2CPP 제한사항

### 주요 제한사항

| 기능 | Mono | IL2CPP | 해결 방법 |
|-----|------|--------|----------|
| **동적 코드 생성** | 지원 | 미지원 | link.xml, Preserve |
| **Reflection.Emit** | 지원 | 미지원 | Source Generator |
| **제네릭 가상 메서드** | 지원 | 제한적 | 명시적 인스턴스화 |
| **System.Reflection** | 지원 | 제한적 | 타입 보존 필요 |
| **dynamic 키워드** | 지원 | 미지원 | 정적 타입 사용 |
| **타입 추론** | 런타임 | 컴파일 타임 | 명시적 타입 지정 |

---

## 코드 스트리핑 방지

### link.xml 설정

```xml
<!-- Assets/link.xml -->
<linker>
    <!-- 전체 어셈블리 보존 -->
    <assembly fullname="MyGameAssembly" preserve="all"/>

    <!-- 특정 네임스페이스 보존 -->
    <assembly fullname="Assembly-CSharp">
        <namespace fullname="Game.Network" preserve="all"/>
        <namespace fullname="Game.Data" preserve="all"/>
    </assembly>

    <!-- 특정 타입만 보존 -->
    <assembly fullname="Assembly-CSharp">
        <type fullname="Game.Player.PlayerData" preserve="all"/>
        <type fullname="Game.Items.ItemDatabase" preserve="all"/>
    </assembly>

    <!-- 특정 메서드만 보존 -->
    <assembly fullname="Assembly-CSharp">
        <type fullname="Game.Network.MessageHandler">
            <method name="HandleMessage"/>
            <method name="ProcessEvent"/>
        </type>
    </assembly>

    <!-- 제네릭 타입 보존 -->
    <assembly fullname="mscorlib">
        <type fullname="System.Collections.Generic.List`1" preserve="all"/>
        <type fullname="System.Collections.Generic.Dictionary`2" preserve="all"/>
    </assembly>

    <!-- JSON 직렬화 라이브러리 -->
    <assembly fullname="Newtonsoft.Json" preserve="all"/>

    <!-- MessagePack -->
    <assembly fullname="MessagePack" preserve="all"/>
    <assembly fullname="MessagePack.Annotations" preserve="all"/>

    <!-- System.Memory -->
    <assembly fullname="System.Memory" preserve="all"/>
    <assembly fullname="System.Buffers" preserve="all"/>
</linker>
```

### Preserve 속성 사용

```csharp
using System;
using UnityEngine;
using UnityEngine.Scripting;

/// <summary>
/// [Preserve] 속성으로 코드 스트리핑 방지
/// </summary>
[Preserve]
public class PreservedClass
{
    [Preserve]
    public string Data { get; set; }

    [Preserve]
    public void PreservedMethod()
    {
        Debug.Log("이 메서드는 스트리핑되지 않습니다.");
    }
}

/// <summary>
/// 리플렉션으로 사용되는 타입 보존
/// </summary>
[Preserve]
public class ReflectionTarget
{
    [Preserve]
    public int Id { get; set; }

    [Preserve]
    public string Name { get; set; }

    [Preserve]
    public void Execute(string param)
    {
        Debug.Log($"Execute: {param}");
    }
}

/// <summary>
/// 제네릭 타입 힌트
/// </summary>
public static class GenericTypePreserver
{
    // AOT 컴파일러에게 제네릭 타입 힌트 제공
    [Preserve]
    private static void PreserveGenericTypes()
    {
        // 실제로 호출되지 않지만 컴파일러가 타입을 인식
        _ = new System.Collections.Generic.List<PlayerData>();
        _ = new System.Collections.Generic.Dictionary<string, ItemData>();
        _ = new System.Collections.Generic.HashSet<int>();

        // Nullable 타입
        _ = default(int?);
        _ = default(float?);
        _ = default(DateTime?);

        // Action/Func
        _ = default(Action<string>);
        _ = default(Func<int, string>);
        _ = default(Func<Task<bool>>);
    }

    [Preserve]
    public class PlayerData { }

    [Preserve]
    public class ItemData { }
}
```

---

## 리플렉션 호환 코드

### 안전한 리플렉션 사용

```csharp
using System;
using System.Reflection;
using UnityEngine;
using UnityEngine.Scripting;

/// <summary>
/// IL2CPP 호환 리플렉션 유틸리티
/// </summary>
public static class IL2CPPReflection
{
    /// <summary>
    /// 안전한 타입 조회
    /// </summary>
    public static Type GetTypeSafe(string typeName)
    {
        try
        {
            return Type.GetType(typeName, throwOnError: false);
        }
        catch (Exception ex)
        {
            Debug.LogWarning($"타입을 찾을 수 없음: {typeName}, {ex.Message}");
            return null;
        }
    }

    /// <summary>
    /// 안전한 메서드 호출
    /// </summary>
    public static object InvokeSafe(object target, string methodName, params object[] args)
    {
        try
        {
            var type = target.GetType();
            var method = type.GetMethod(methodName,
                BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance);

            if (method == null)
            {
                Debug.LogWarning($"메서드를 찾을 수 없음: {methodName}");
                return null;
            }

            return method.Invoke(target, args);
        }
        catch (Exception ex)
        {
            Debug.LogError($"메서드 호출 실패: {methodName}, {ex.Message}");
            return null;
        }
    }

    /// <summary>
    /// 안전한 프로퍼티 값 가져오기
    /// </summary>
    public static T GetPropertySafe<T>(object target, string propertyName, T defaultValue = default)
    {
        try
        {
            var type = target.GetType();
            var property = type.GetProperty(propertyName);

            if (property == null)
            {
                return defaultValue;
            }

            return (T)property.GetValue(target);
        }
        catch
        {
            return defaultValue;
        }
    }

    /// <summary>
    /// 안전한 인스턴스 생성
    /// </summary>
    public static T CreateInstanceSafe<T>() where T : class, new()
    {
        try
        {
            return Activator.CreateInstance<T>();
        }
        catch (Exception ex)
        {
            Debug.LogError($"인스턴스 생성 실패: {typeof(T).Name}, {ex.Message}");
            return null;
        }
    }
}

/// <summary>
/// 리플렉션 대신 인터페이스 사용
/// </summary>
public interface ISerializable
{
    void Serialize(System.IO.BinaryWriter writer);
    void Deserialize(System.IO.BinaryReader reader);
}

[Preserve]
public class PlayerState : ISerializable
{
    public int Level { get; set; }
    public float Health { get; set; }

    public void Serialize(System.IO.BinaryWriter writer)
    {
        writer.Write(Level);
        writer.Write(Health);
    }

    public void Deserialize(System.IO.BinaryReader reader)
    {
        Level = reader.ReadInt32();
        Health = reader.ReadSingle();
    }
}
```

### 타입 레지스트리 패턴

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Scripting;

/// <summary>
/// IL2CPP 호환 타입 레지스트리
/// 리플렉션 대신 명시적 등록 사용
/// </summary>
public class TypeRegistry
{
    private static readonly Dictionary<string, Func<object>> _factories = new();
    private static readonly Dictionary<Type, string> _typeNames = new();

    /// <summary>
    /// 타입 등록
    /// </summary>
    public static void Register<T>(string typeName = null) where T : new()
    {
        var type = typeof(T);
        typeName ??= type.FullName;

        _factories[typeName] = () => new T();
        _typeNames[type] = typeName;
    }

    /// <summary>
    /// 인스턴스 생성
    /// </summary>
    public static object Create(string typeName)
    {
        if (_factories.TryGetValue(typeName, out var factory))
        {
            return factory();
        }

        Debug.LogError($"등록되지 않은 타입: {typeName}");
        return null;
    }

    /// <summary>
    /// 타입 이름 조회
    /// </summary>
    public static string GetTypeName(object instance)
    {
        var type = instance.GetType();
        return _typeNames.TryGetValue(type, out var name) ? name : type.FullName;
    }

    /// <summary>
    /// 초기화 (앱 시작 시 호출)
    /// </summary>
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    [Preserve]
    private static void Initialize()
    {
        // 모든 직렬화 가능 타입 등록
        Register<PlayerData>();
        Register<GameSettings>();
        Register<InventoryItem>();
        Register<QuestProgress>();
        // ... 추가 타입
    }

    [Preserve] public class PlayerData { public int level; }
    [Preserve] public class GameSettings { public float volume; }
    [Preserve] public class InventoryItem { public string id; }
    [Preserve] public class QuestProgress { public bool completed; }
}

/// <summary>
/// 타입 레지스트리 사용 예시
/// </summary>
public class RegistryExample : MonoBehaviour
{
    private void Start()
    {
        // 타입 이름으로 인스턴스 생성 (리플렉션 없이)
        var player = TypeRegistry.Create("Game.PlayerData") as TypeRegistry.PlayerData;
        if (player != null)
        {
            player.level = 42;
            Debug.Log($"플레이어 레벨: {player.level}");
        }
    }
}
```

---

## 제네릭 제한사항 처리

### 제네릭 타입 명시적 인스턴스화

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Scripting;

/// <summary>
/// 제네릭 메서드의 AOT 호환성 확보
/// </summary>
public class GenericAOTCompatibility
{
    /// <summary>
    /// 제네릭 메서드
    /// </summary>
    public T Deserialize<T>(byte[] data)
    {
        // 실제 역직렬화 로직
        return default;
    }

    /// <summary>
    /// AOT 힌트 - 사용될 모든 타입에 대해 명시적 호출
    /// </summary>
    [Preserve]
    private void AOTHints()
    {
        // 이 코드는 실행되지 않지만 컴파일러가 필요한 코드를 생성
        Deserialize<int>(null);
        Deserialize<float>(null);
        Deserialize<string>(null);
        Deserialize<Vector3>(null);
        Deserialize<PlayerData>(null);
        Deserialize<List<int>>(null);
        Deserialize<Dictionary<string, int>>(null);
    }

    [Preserve]
    public class PlayerData { }
}

/// <summary>
/// 제네릭 인터페이스 구현
/// </summary>
public interface IMessageHandler<T>
{
    void Handle(T message);
}

// 각 메시지 타입에 대한 구체적 구현
[Preserve]
public class LoginMessageHandler : IMessageHandler<LoginMessage>
{
    public void Handle(LoginMessage message)
    {
        Debug.Log($"로그인: {message.Username}");
    }
}

[Preserve]
public class ChatMessageHandler : IMessageHandler<ChatMessage>
{
    public void Handle(ChatMessage message)
    {
        Debug.Log($"채팅: {message.Content}");
    }
}

[Preserve] public class LoginMessage { public string Username; }
[Preserve] public class ChatMessage { public string Content; }

/// <summary>
/// 메시지 디스패처 (제네릭 대신 타입별 처리)
/// </summary>
public class MessageDispatcher
{
    private readonly Dictionary<Type, object> _handlers = new();

    public void Register<T>(IMessageHandler<T> handler)
    {
        _handlers[typeof(T)] = handler;
    }

    public void Dispatch<T>(T message)
    {
        if (_handlers.TryGetValue(typeof(T), out var handler))
        {
            ((IMessageHandler<T>)handler).Handle(message);
        }
    }

    [RuntimeInitializeOnLoadMethod]
    [Preserve]
    private static void Initialize()
    {
        var dispatcher = new MessageDispatcher();

        // 모든 핸들러 명시적 등록
        dispatcher.Register(new LoginMessageHandler());
        dispatcher.Register(new ChatMessageHandler());
    }
}
```

### ValueType 제네릭 처리

```csharp
using System;
using UnityEngine.Scripting;

/// <summary>
/// 값 타입 제네릭의 AOT 문제 해결
/// </summary>
public static class ValueTypeGenericFix
{
    /// <summary>
    /// Nullable<T>의 AOT 힌트
    /// </summary>
    [Preserve]
    private static void NullableHints()
    {
        // 모든 사용되는 Nullable 타입 명시
        _ = default(int?);
        _ = default(float?);
        _ = default(double?);
        _ = default(bool?);
        _ = default(DateTime?);
        _ = default(Guid?);
        _ = default(Vector3?);
        _ = default(Quaternion?);

        // Nullable 비교
        _ = EqualityComparer<int?>.Default;
        _ = EqualityComparer<float?>.Default;
    }

    /// <summary>
    /// 값 타입 배열/컬렉션 힌트
    /// </summary>
    [Preserve]
    private static void CollectionHints()
    {
        // Array
        _ = new int[0];
        _ = new float[0];
        _ = new Vector3[0];

        // List
        _ = new System.Collections.Generic.List<int>();
        _ = new System.Collections.Generic.List<float>();
        _ = new System.Collections.Generic.List<Vector3>();

        // Dictionary
        _ = new System.Collections.Generic.Dictionary<int, int>();
        _ = new System.Collections.Generic.Dictionary<int, float>();
        _ = new System.Collections.Generic.Dictionary<int, string>();
    }
}

// Vector3? 확장
[Preserve]
public struct Vector3
{
    public float x, y, z;
}

[Preserve]
public struct Quaternion
{
    public float x, y, z, w;
}
```

---

## async/await AOT 호환성

### Task/ValueTask 보존

```csharp
using System;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.Scripting;

/// <summary>
/// async/await의 AOT 호환성 확보
/// </summary>
public static class AsyncAOTPreservation
{
    [Preserve]
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void PreserveAsyncTypes()
    {
        // Task 관련 타입
        _ = typeof(Task);
        _ = typeof(Task<int>);
        _ = typeof(Task<string>);
        _ = typeof(Task<bool>);
        _ = typeof(Task<byte[]>);

        // ValueTask
        _ = typeof(ValueTask);
        _ = typeof(ValueTask<int>);
        _ = typeof(ValueTask<string>);

        // TaskCompletionSource
        _ = typeof(TaskCompletionSource<bool>);
        _ = typeof(TaskCompletionSource<string>);

        // CancellationToken
        _ = typeof(System.Threading.CancellationToken);
        _ = typeof(System.Threading.CancellationTokenSource);

        // AsyncLocal
        _ = typeof(System.Threading.AsyncLocal<int>);
    }

    /// <summary>
    /// 비동기 메서드 AOT 힌트
    /// </summary>
    [Preserve]
    private static async Task AsyncMethodHints()
    {
        await Task.Delay(1);
        await Task.Yield();
        _ = await Task.FromResult(1);
        _ = await Task.FromResult("test");
    }

    /// <summary>
    /// ConfigureAwait AOT 힌트
    /// </summary>
    [Preserve]
    private static async Task ConfigureAwaitHints()
    {
        await Task.Delay(1).ConfigureAwait(false);
        await Task.FromResult(1).ConfigureAwait(false);
    }
}
```

### UniTask AOT 설정

```csharp
using Cysharp.Threading.Tasks;
using UnityEngine.Scripting;

/// <summary>
/// UniTask AOT 호환성
/// </summary>
public static class UniTaskAOTPreservation
{
    [Preserve]
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void PreserveUniTaskTypes()
    {
        // UniTask 기본 타입
        _ = typeof(UniTask);
        _ = typeof(UniTask<int>);
        _ = typeof(UniTask<string>);
        _ = typeof(UniTask<bool>);
        _ = typeof(UniTask<byte[]>);

        // UniTaskVoid
        _ = typeof(UniTaskVoid);

        // UniTask 컬렉션
        _ = typeof(UniTask<int[]>);
        _ = typeof(UniTask<System.Collections.Generic.List<string>>);
    }

    [Preserve]
    private static async UniTask UniTaskMethodHints()
    {
        await UniTask.Delay(1);
        await UniTask.Yield();
        await UniTask.SwitchToMainThread();
        await UniTask.SwitchToThreadPool();
    }

    [Preserve]
    private static async UniTask<int> UniTaskGenericHints()
    {
        await UniTask.Delay(1);
        return 42;
    }
}
```

---

## 직렬화 라이브러리 AOT 설정

### MessagePack AOT

```csharp
using MessagePack;
using MessagePack.Resolvers;
using UnityEngine.Scripting;

/// <summary>
/// MessagePack AOT 설정
/// </summary>
public static class MessagePackAOTConfig
{
    [Preserve]
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void Initialize()
    {
        // 컴포지트 리졸버 생성
        var resolver = CompositeResolver.Create(
            // 생성된 리졸버 (Source Generator)
            // GeneratedResolver.Instance,

            // 빌트인 리졸버
            BuiltinResolver.Instance,
            AttributeFormatterResolver.Instance,
            PrimitiveObjectResolver.Instance,

            // Unity 타입 (필요시)
            MessagePack.Unity.UnityResolver.Instance,

            // 기본 리졸버
            StandardResolver.Instance
        );

        var options = MessagePackSerializerOptions.Standard.WithResolver(resolver);
        MessagePackSerializer.DefaultOptions = options;
    }
}
```

### MemoryPack AOT

```csharp
using MemoryPack;
using UnityEngine.Scripting;

/// <summary>
/// MemoryPack은 Source Generator 사용으로 AOT 자동 호환
/// </summary>
[MemoryPackable]
[Preserve]
public partial class AOTCompatibleData
{
    [Preserve]
    public int Id { get; set; }

    [Preserve]
    public string Name { get; set; }

    [Preserve]
    public byte[] CustomData { get; set; }
}

/// <summary>
/// MemoryPack 초기화 (필요시)
/// </summary>
public static class MemoryPackAOTConfig
{
    [Preserve]
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void Initialize()
    {
        // 커스텀 Formatter 등록 (필요시)
        // MemoryPackFormatterProvider.Register<CustomType>(new CustomFormatter());
    }
}
```

---

## IL2CPP 빌드 최적화

### 빌드 설정

```csharp
// Editor 스크립트
#if UNITY_EDITOR
using UnityEditor;
using UnityEditor.Build;
using UnityEditor.Build.Reporting;

/// <summary>
/// IL2CPP 빌드 설정
/// </summary>
public class IL2CPPBuildSettings : IPreprocessBuildWithReport
{
    public int callbackOrder => 0;

    public void OnPreprocessBuild(BuildReport report)
    {
        // IL2CPP 설정
        PlayerSettings.SetScriptingBackend(
            report.summary.platformGroup,
            ScriptingImplementation.IL2CPP);

        // 코드 최적화 레벨
        PlayerSettings.SetIl2CppCompilerConfiguration(
            report.summary.platformGroup,
            Il2CppCompilerConfiguration.Release);

        // 스트리핑 레벨 (주의: 너무 높으면 필요한 코드 제거됨)
        PlayerSettings.stripEngineCode = true;

        // 증분 GC (성능 향상)
        PlayerSettings.gcIncremental = true;
    }
}
#endif
```

### 코드 크기 최적화

```csharp
using UnityEngine;

/// <summary>
/// IL2CPP 코드 크기 최적화 팁
/// </summary>
public class IL2CPPOptimizationTips
{
    // 1. 사용하지 않는 제네릭 피하기
    // ❌ 불필요한 제네릭
    // public class GenericClass<T> where T : class { }

    // ✅ 필요한 경우만 제네릭 사용
    // public class SpecificClass { }

    // 2. 람다/클로저 최소화
    // ❌ 매번 새 람다 생성
    // void Update() {
    //     SomeMethod(() => DoSomething());
    // }

    // ✅ 캐시된 델리게이트 사용
    private System.Action _cachedAction;
    void Start()
    {
        _cachedAction = DoSomething;
    }
    void Update()
    {
        SomeMethod(_cachedAction);
    }
    void SomeMethod(System.Action action) { }
    void DoSomething() { }

    // 3. 문자열 연결 최소화
    // ❌ + 연산자로 문자열 연결
    // string result = "Hello" + name + "!";

    // ✅ StringBuilder 또는 보간 사용
    private readonly System.Text.StringBuilder _sb = new();
    string BuildString(string name)
    {
        _sb.Clear();
        _sb.Append("Hello ").Append(name).Append("!");
        return _sb.ToString();
    }

    // 4. LINQ 주의해서 사용
    // ❌ 복잡한 LINQ (많은 코드 생성)
    // var result = items.Where(x => x.Value > 10)
    //                   .Select(x => x.Name)
    //                   .OrderBy(x => x);

    // ✅ 직접 루프 (코드 크기 작음)
    System.Collections.Generic.List<string> GetFilteredNames(
        System.Collections.Generic.List<Item> items)
    {
        var result = new System.Collections.Generic.List<string>();
        foreach (var item in items)
        {
            if (item.Value > 10)
            {
                result.Add(item.Name);
            }
        }
        result.Sort();
        return result;
    }

    class Item { public int Value; public string Name; }
}
```

---

## 디버깅 및 문제 해결

### 일반적인 오류 및 해결

```csharp
using System;
using UnityEngine;

/// <summary>
/// IL2CPP 일반 오류 및 해결
/// </summary>
public static class IL2CPPTroubleshooting
{
    /*
    오류 1: ExecutionEngineException
    - 원인: 필요한 타입이 스트리핑됨
    - 해결: link.xml에 타입 추가 또는 [Preserve] 속성

    오류 2: NotSupportedException (Reflection.Emit)
    - 원인: 동적 코드 생성 시도
    - 해결: Source Generator 사용 또는 정적 코드로 대체

    오류 3: TypeLoadException
    - 원인: 제네릭 타입 코드가 생성되지 않음
    - 해결: AOT 힌트 추가

    오류 4: MissingMethodException
    - 원인: 메서드가 스트리핑됨
    - 해결: link.xml에 메서드 추가
    */

    /// <summary>
    /// IL2CPP 디버그 로깅
    /// </summary>
    public static void LogIL2CPPInfo()
    {
#if ENABLE_IL2CPP
        Debug.Log("IL2CPP 빌드입니다.");
        Debug.Log($"런타임: {RuntimeInformation.FrameworkDescription}");
#else
        Debug.Log("Mono 빌드입니다.");
#endif
    }

    /// <summary>
    /// 타입 존재 확인
    /// </summary>
    public static bool TypeExists(string typeName)
    {
        try
        {
            var type = Type.GetType(typeName);
            if (type == null)
            {
                Debug.LogWarning($"타입을 찾을 수 없음 (스트리핑됨?): {typeName}");
                return false;
            }
            return true;
        }
        catch (Exception ex)
        {
            Debug.LogError($"타입 확인 오류: {typeName}, {ex.Message}");
            return false;
        }
    }

    /// <summary>
    /// 메서드 존재 확인
    /// </summary>
    public static bool MethodExists(Type type, string methodName)
    {
        try
        {
            var method = type.GetMethod(methodName);
            if (method == null)
            {
                Debug.LogWarning($"메서드를 찾을 수 없음 (스트리핑됨?): {type.Name}.{methodName}");
                return false;
            }
            return true;
        }
        catch (Exception ex)
        {
            Debug.LogError($"메서드 확인 오류: {type.Name}.{methodName}, {ex.Message}");
            return false;
        }
    }
}

#if !ENABLE_IL2CPP
// Mono 환경에서만 사용 가능한 기능들
public static class MonoOnlyFeatures
{
    public static void DynamicCodeExample()
    {
        // Reflection.Emit 등 동적 코드 생성
        // IL2CPP에서는 사용 불가
    }
}
#endif
```

---

## 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────┐
│                  IL2CPP/AOT 베스트 프랙티스                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 코드 보존                                                    │
│     ├── link.xml로 필요한 타입/메서드 보존                        │
│     ├── [Preserve] 속성 활용                                     │
│     └── AOT 힌트 메서드 작성                                     │
│                                                                  │
│  2. 리플렉션 최소화                                               │
│     ├── Source Generator 활용                                    │
│     ├── 타입 레지스트리 패턴 사용                                 │
│     └── 인터페이스 기반 다형성                                    │
│                                                                  │
│  3. 제네릭 주의                                                   │
│     ├── 모든 사용 타입 명시적 인스턴스화                          │
│     ├── 값 타입 제네릭 특히 주의                                  │
│     └── AOT 힌트로 컴파일러 안내                                  │
│                                                                  │
│  4. 직렬화                                                       │
│     ├── Source Generator 기반 라이브러리 선호                     │
│     ├── MemoryPack, MessagePack Source Generator                 │
│     └── 동적 타입 생성 피하기                                     │
│                                                                  │
│  5. 테스트                                                       │
│     ├── 실제 기기에서 IL2CPP 빌드 테스트                          │
│     ├── 개발 초기부터 주기적 빌드                                 │
│     └── 에러 로그 주의 깊게 확인                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [Unity IL2CPP Documentation](https://docs.unity3d.com/Manual/IL2CPP.html)
- [Managed Code Stripping](https://docs.unity3d.com/Manual/ManagedCodeStripping.html)
- [IL2CPP Scripting Restrictions](https://docs.unity3d.com/Manual/ScriptingRestrictions.html)
- [link.xml Reference](https://docs.unity3d.com/Manual/ManagedCodeStripping.html)

---

## 다음 단계

- [Section 39: 메모리 & GC 최적화](../13-optimization/39-memory-gc.md)
- [Section 40: 프로파일링 & 디버깅](../13-optimization/40-profiling-debugging.md)
