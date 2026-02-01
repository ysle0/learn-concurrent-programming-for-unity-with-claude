# 41. 테스트 작성

## 개요

비동기/동시성 코드는 **비결정적(non-deterministic)** 특성으로 인해 테스트가 매우 까다롭습니다. 실행 순서가 매번 달라질 수 있고, 타이밍에 의존하는 버그는 재현이 어렵습니다. Unity 환경에서는 메인 스레드 제약, Coroutine 기반 비동기, 프레임 단위 실행 등 추가적인 복잡성이 존재합니다.

### 비동기 코드 테스트가 어려운 이유

```
동기 코드 테스트:   Input → 함수 호출 → Output → Assert (즉시 검증)
비동기 코드 테스트: Input → 함수 호출 → ... 대기 ... → Output → Assert
                                        ↑
                                   언제 완료? 어떤 스레드? 예외는 어디로?
```

```csharp
// ❌ 비동기 코드를 동기적으로 테스트하면 실패
[Test]
public void NaiveAsyncTest()
{
    var service = new DataService();
    service.LoadDataAsync(); // 비동기 시작만 하고 즉시 반환
    Assert.IsNotNull(service.Data); // 아직 로드되지 않아 실패!
}

// ✅ 비동기 완료를 올바르게 대기
[Test]
public async Task ProperAsyncTest()
{
    var service = new DataService();
    await service.LoadDataAsync(); // 완료 대기
    Assert.IsNotNull(service.Data);
}
```

---

## 1. Unity Test Framework 기초

Unity Test Framework는 **NUnit 3**을 기반으로 하며, Unity 특화 확장을 제공합니다.

```
프로젝트 구조:
Assets/
├── Scripts/
│   └── MyGame/
│       └── Services/DataService.cs
└── Tests/
    ├── EditMode/
    │   ├── EditMode.asmdef          ← EditMode 테스트 어셈블리
    │   └── DataServiceTests.cs
    └── PlayMode/
        ├── PlayMode.asmdef          ← PlayMode 테스트 어셈블리
        └── DataServicePlayTests.cs
```

### 어셈블리 정의 설정

```json
// EditMode.asmdef
{
    "name": "Tests.EditMode",
    "references": [ "MyGame" ],
    "includePlatforms": [ "Editor" ],
    "defineConstraints": [ "UNITY_INCLUDE_TESTS" ],
    "overrideReferences": true,
    "precompiledReferences": [ "nunit.framework.dll" ]
}

// PlayMode.asmdef — includePlatforms를 비워서 모든 플랫폼 포함
{
    "name": "Tests.PlayMode",
    "references": [ "MyGame", "UnityEngine.TestRunner", "UnityEditor.TestRunner" ],
    "includePlatforms": [],
    "defineConstraints": [ "UNITY_INCLUDE_TESTS" ],
    "overrideReferences": true,
    "precompiledReferences": [ "nunit.framework.dll" ]
}
```

### 기본 테스트 어트리뷰트

```csharp
using NUnit.Framework;

[TestFixture]  // 테스트 클래스 표시 (생략 가능)
public class BasicTestExample
{
    [OneTimeSetUp]  public void OneTimeSetUp()  { /* 클래스 전체 1회 */ }
    [SetUp]         public void SetUp()         { /* 각 테스트 전 */ }
    [TearDown]      public void TearDown()      { /* 각 테스트 후 */ }
    [OneTimeTearDown] public void OneTimeTearDown() { /* 클래스 전체 1회 */ }

    [Test]
    public void SimpleTest() => Assert.AreEqual(4, 2 + 2);

    [TestCase(1, 2, 3)]
    [TestCase(10, 20, 30)]
    public void ParameterizedTest(int a, int b, int expected)
        => Assert.AreEqual(expected, a + b);
}
```

---

## 2. EditMode 테스트

EditMode 테스트는 **에디터 환경에서 동기적으로** 실행됩니다. MonoBehaviour 라이프사이클 없이 순수 로직을 테스트할 때 적합합니다.

```csharp
using NUnit.Framework;
using System.Threading;
using System.Threading.Tasks;

[TestFixture]
public class AsyncUtilityEditTests
{
    [Test]
    public void TaskResult_ReturnsValue_WhenCompleted()
    {
        var task = Task.FromResult(42);
        Assert.AreEqual(42, task.Result); // 이미 완료된 Task
    }

    [Test]
    public void CancellationToken_ThrowsWhenCancelled()
    {
        var cts = new CancellationTokenSource();
        cts.Cancel();
        Assert.Throws<OperationCanceledException>(() =>
            cts.Token.ThrowIfCancellationRequested());
    }

    [Test]
    public void SemaphoreSlim_LimitsAccess()
    {
        var semaphore = new SemaphoreSlim(2, 2);
        semaphore.Wait();
        semaphore.Wait();
        Assert.AreEqual(0, semaphore.CurrentCount);

        bool acquired = semaphore.Wait(0); // 즉시 반환
        Assert.IsFalse(acquired);

        semaphore.Release(2);
        Assert.AreEqual(2, semaphore.CurrentCount);
    }

    // ✅ EditMode에서도 async Task 테스트 가능 (Unity 2021.2+)
    [Test]
    public async Task AsyncMethod_CompletesSuccessfully()
    {
        var result = await ComputeAsync(5);
        Assert.AreEqual(25, result);
    }

    [Test]
    public void AsyncMethod_ThrowsException()
    {
        Assert.ThrowsAsync<System.ArgumentException>(async () =>
            await ComputeAsync(-1));
    }

    private async Task<int> ComputeAsync(int value)
    {
        if (value < 0) throw new System.ArgumentException("음수 불가");
        await Task.Yield();
        return value * value;
    }
}
```

---

## 3. PlayMode 테스트

PlayMode 테스트는 **런타임 환경을 시뮬레이션**합니다. MonoBehaviour, Coroutine, 프레임 진행이 필요한 테스트에 사용합니다.

```csharp
using System.Collections;
using NUnit.Framework;
using UnityEngine;
using UnityEngine.TestTools;

[TestFixture]
public class CoroutinePlayModeTests
{
    // [UnityTest]는 IEnumerator를 반환 → Coroutine처럼 동작
    [UnityTest]
    public IEnumerator Coroutine_CompletesAfterDelay()
    {
        var go = new GameObject("TestObject");
        var component = go.AddComponent<DelayedInitializer>();

        Assert.IsFalse(component.IsInitialized);
        yield return new WaitForSeconds(2.1f);
        Assert.IsTrue(component.IsInitialized);

        Object.Destroy(go);
    }

    [UnityTest]
    public IEnumerator WaitForFrames_CorrectFrameCount()
    {
        int startFrame = Time.frameCount;
        for (int i = 0; i < 5; i++) yield return null;
        Assert.AreEqual(5, Time.frameCount - startFrame);
    }

    [UnityTest]
    public IEnumerator MonoBehaviour_LifecycleOrder_IsCorrect()
    {
        var go = new GameObject("LifecycleTest");
        var tracker = go.AddComponent<LifecycleTracker>();

        Assert.IsTrue(tracker.AwakeCalled);     // Awake는 즉시 호출
        Assert.IsFalse(tracker.StartCalled);     // Start는 다음 프레임
        yield return null;
        Assert.IsTrue(tracker.StartCalled);

        int prevCount = tracker.UpdateCount;
        yield return null;
        Assert.Greater(tracker.UpdateCount, prevCount);

        Object.Destroy(go);
    }
}

public class DelayedInitializer : MonoBehaviour
{
    public bool IsInitialized { get; private set; }
    private IEnumerator Start()
    {
        yield return new WaitForSeconds(2f);
        IsInitialized = true;
    }
}

public class LifecycleTracker : MonoBehaviour
{
    public bool AwakeCalled { get; private set; }
    public bool StartCalled { get; private set; }
    public int UpdateCount { get; private set; }
    void Awake() => AwakeCalled = true;
    void Start() => StartCalled = true;
    void Update() => UpdateCount++;
}
```

---

## 4. async 테스트 메서드

```csharp
using NUnit.Framework;
using System;
using System.Threading;
using System.Threading.Tasks;

[TestFixture]
public class AsyncTaskTests
{
    [Test]
    public async Task FetchData_ReturnsValidResult()
    {
        var service = new MockDataService();
        var data = await service.FetchDataAsync("test-key");
        Assert.AreEqual("test-value", data);
    }

    [Test]
    public void AsyncMethod_ThrowsOnInvalidInput()
    {
        var service = new MockDataService();
        Assert.ThrowsAsync<ArgumentNullException>(async () =>
            await service.FetchDataAsync(null));
    }

    [Test]
    public async Task AsyncMethod_RespectsCancellation()
    {
        var cts = new CancellationTokenSource();
        cts.Cancel();

        try
        {
            await new MockDataService()
                .FetchWithCancellationAsync("key", cts.Token);
            Assert.Fail("OperationCanceledException이 발생해야 합니다");
        }
        catch (OperationCanceledException) { Assert.Pass(); }
    }

    [Test]
    [Timeout(3000)] // 3초 타임아웃
    public async Task SlowOperation_CompletesWithinTimeout()
    {
        var result = await new MockDataService().SlowOperationAsync();
        Assert.IsNotNull(result);
    }

    [Test]
    public async Task ParallelOperations_AllComplete()
    {
        var svc = new MockDataService();
        var results = await Task.WhenAll(
            svc.FetchDataAsync("k1"),
            svc.FetchDataAsync("k2"),
            svc.FetchDataAsync("k3"));

        Assert.AreEqual(3, results.Length);
        foreach (var r in results) Assert.IsNotNull(r);
    }
}

public class MockDataService
{
    public async Task<string> FetchDataAsync(string key)
    {
        if (key == null) throw new ArgumentNullException(nameof(key));
        await Task.Delay(10);
        return "test-value";
    }

    public async Task<string> FetchWithCancellationAsync(string key, CancellationToken ct)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(1000, ct);
        return "value";
    }

    public async Task<string> SlowOperationAsync()
    {
        await Task.Delay(500);
        return "done";
    }
}
```

---

## 5. UniTask 테스트 패턴

```csharp
using NUnit.Framework;
using Cysharp.Threading.Tasks;
using System;
using System.Threading;
using UnityEngine;
using UnityEngine.TestTools;

[TestFixture]
public class UniTaskTestExamples
{
    // ✅ UniTask를 async Task로 변환하여 테스트
    [Test]
    public async Task UniTask_BasicTest()
    {
        var result = await LoadDataUniTask().AsTask();
        Assert.AreEqual(42, result);
    }

    // ✅ [UnityTest]와 UniTask.ToCoroutine 조합
    [UnityTest]
    public System.Collections.IEnumerator UniTask_WithUnityTest()
        => UniTask.ToCoroutine(async () =>
    {
        var result = await LoadDataUniTask();
        Assert.AreEqual(42, result);
    });

    [UnityTest]
    public System.Collections.IEnumerator UniTask_CancellationTest()
        => UniTask.ToCoroutine(async () =>
    {
        var cts = new CancellationTokenSource();
        cts.Cancel();

        bool cancelled = false;
        try { await UniTask.Delay(1000, cancellationToken: cts.Token); }
        catch (OperationCanceledException) { cancelled = true; }
        Assert.IsTrue(cancelled);
    });

    [UnityTest]
    public System.Collections.IEnumerator UniTask_WhenAll_CompletesAll()
        => UniTask.ToCoroutine(async () =>
    {
        var (r1, r2, r3) = await UniTask.WhenAll(
            LoadDataUniTask(), LoadDataUniTask(), LoadDataUniTask());
        Assert.AreEqual(42, r1);
        Assert.AreEqual(42, r2);
        Assert.AreEqual(42, r3);
    });

    [UnityTest]
    public System.Collections.IEnumerator UniTask_Timeout_ThrowsOnExpiry()
        => UniTask.ToCoroutine(async () =>
    {
        bool timedOut = false;
        try
        {
            await SlowUniTaskOperation()
                .Timeout(TimeSpan.FromMilliseconds(100));
        }
        catch (TimeoutException) { timedOut = true; }
        Assert.IsTrue(timedOut);
    });

    [UnityTest]
    public System.Collections.IEnumerator UniTask_DelayFrame_Test()
        => UniTask.ToCoroutine(async () =>
    {
        int startFrame = Time.frameCount;
        await UniTask.DelayFrame(3);
        Assert.AreEqual(3, Time.frameCount - startFrame);
    });

    private async UniTask<int> LoadDataUniTask()
    {
        await UniTask.Delay(10);
        return 42;
    }

    private async UniTask SlowUniTaskOperation()
        => await UniTask.Delay(5000);
}
```

---

## 6. Mock/Stub 패턴

### 비동기 의존성 모킹

```csharp
using NUnit.Framework;
using System;
using System.Threading;
using System.Threading.Tasks;

// 서비스 인터페이스
public interface IDataRepository
{
    Task<string> GetByIdAsync(string id, CancellationToken ct = default);
    Task SaveAsync(string id, string data, CancellationToken ct = default);
    Task<bool> ExistsAsync(string id, CancellationToken ct = default);
}

// ✅ 수동 Mock 구현
public class MockDataRepository : IDataRepository
{
    public int GetByIdCallCount { get; private set; }
    public int SaveCallCount { get; private set; }
    public string LastSavedId { get; private set; }

    public string GetByIdResult { get; set; } = "default-data";
    public bool ExistsResult { get; set; } = true;
    public Exception ExceptionToThrow { get; set; }
    public int DelayMilliseconds { get; set; } = 0;

    public async Task<string> GetByIdAsync(string id, CancellationToken ct = default)
    {
        GetByIdCallCount++;
        if (ExceptionToThrow != null) throw ExceptionToThrow;
        if (DelayMilliseconds > 0) await Task.Delay(DelayMilliseconds, ct);
        ct.ThrowIfCancellationRequested();
        return GetByIdResult;
    }

    public async Task SaveAsync(string id, string data, CancellationToken ct = default)
    {
        SaveCallCount++;
        LastSavedId = id;
        if (DelayMilliseconds > 0) await Task.Delay(DelayMilliseconds, ct);
    }

    public Task<bool> ExistsAsync(string id, CancellationToken ct = default)
        => Task.FromResult(ExistsResult);
}

// 테스트 대상
public class CacheService
{
    private readonly IDataRepository _repository;
    public CacheService(IDataRepository repository) => _repository = repository;

    public async Task<string> GetOrLoadAsync(string id, CancellationToken ct = default)
    {
        if (await _repository.ExistsAsync(id, ct))
            return await _repository.GetByIdAsync(id, ct);
        return null;
    }
}

// 테스트
[TestFixture]
public class CacheServiceTests
{
    private MockDataRepository _mockRepo;
    private CacheService _service;

    [SetUp]
    public void SetUp()
    {
        _mockRepo = new MockDataRepository();
        _service = new CacheService(_mockRepo);
    }

    [Test]
    public async Task GetOrLoad_WhenExists_ReturnsData()
    {
        _mockRepo.ExistsResult = true;
        _mockRepo.GetByIdResult = "cached-data";

        var result = await _service.GetOrLoadAsync("key1");

        Assert.AreEqual("cached-data", result);
        Assert.AreEqual(1, _mockRepo.GetByIdCallCount);
    }

    [Test]
    public async Task GetOrLoad_WhenNotExists_ReturnsNull()
    {
        _mockRepo.ExistsResult = false;
        var result = await _service.GetOrLoadAsync("key1");
        Assert.IsNull(result);
        Assert.AreEqual(0, _mockRepo.GetByIdCallCount);
    }

    [Test]
    public void GetOrLoad_WhenRepoThrows_PropagatesException()
    {
        _mockRepo.ExistsResult = true;
        _mockRepo.ExceptionToThrow = new InvalidOperationException("DB 연결 실패");
        Assert.ThrowsAsync<InvalidOperationException>(
            async () => await _service.GetOrLoadAsync("key1"));
    }
}
```

---

## 7. 타이밍 기반 테스트

### 시간 추상화를 통한 테스트 가능한 설계

```csharp
using NUnit.Framework;
using System;

// ✅ 시간 추상화 인터페이스
public interface ITimeProvider
{
    float DeltaTime { get; }
    float Time { get; }
}

// 프로덕션 구현
public class UnityTimeProvider : ITimeProvider
{
    public float DeltaTime => UnityEngine.Time.deltaTime;
    public float Time => UnityEngine.Time.time;
}

// ✅ 테스트용 Fake
public class FakeTimeProvider : ITimeProvider
{
    public float DeltaTime { get; set; } = 0.016f;
    public float Time { get; set; } = 0f;
    public void Advance(float seconds) => Time += seconds;
}

// 시간 의존 클래스
public class Cooldown
{
    private readonly ITimeProvider _time;
    private float _lastUseTime = float.MinValue;

    public float Duration { get; }
    public bool IsReady => _time.Time - _lastUseTime >= Duration;
    public float RemainingTime => Math.Max(0, Duration - (_time.Time - _lastUseTime));

    public Cooldown(float duration, ITimeProvider time)
    {
        Duration = duration;
        _time = time;
    }

    public bool TryUse()
    {
        if (!IsReady) return false;
        _lastUseTime = _time.Time;
        return true;
    }
}

[TestFixture]
public class CooldownTests
{
    private FakeTimeProvider _fakeTime;
    private Cooldown _cooldown;

    [SetUp]
    public void SetUp()
    {
        _fakeTime = new FakeTimeProvider { Time = 0f };
        _cooldown = new Cooldown(2.0f, _fakeTime);
    }

    [Test] public void IsReadyInitially() => Assert.IsTrue(_cooldown.IsReady);

    [Test]
    public void NotReadyAfterUse()
    {
        _cooldown.TryUse();
        Assert.IsFalse(_cooldown.IsReady);
    }

    [Test]
    public void ReadyAfterDuration()
    {
        _cooldown.TryUse();
        _fakeTime.Advance(2.0f); // 실제 대기 없이 시간 진행!
        Assert.IsTrue(_cooldown.IsReady);
    }

    [Test]
    public void RemainingTime_IsAccurate()
    {
        _cooldown.TryUse();
        _fakeTime.Advance(0.5f);
        Assert.AreEqual(1.5f, _cooldown.RemainingTime, 0.001f);
    }
}
```

### 실행 시간 측정 테스트

```csharp
using System.Diagnostics;
using System.Threading.Tasks;

[TestFixture]
public class PerformanceTimingTests
{
    [Test]
    public async Task RetryWithBackoff_RespectsDelays()
    {
        var sw = Stopwatch.StartNew();
        int attempts = 0;

        async Task<bool> FlakyOperation()
        {
            attempts++;
            if (attempts < 3) throw new System.Exception("일시적 오류");
            return true;
        }

        bool result = await RetryAsync(FlakyOperation, maxRetries: 3, baseDelayMs: 50);
        sw.Stop();

        Assert.IsTrue(result);
        Assert.AreEqual(3, attempts);
        Assert.GreaterOrEqual(sw.ElapsedMilliseconds, 100); // 50 + 100 = 150ms 지연
    }

    private async Task<T> RetryAsync<T>(System.Func<Task<T>> op, int maxRetries, int baseDelayMs)
    {
        for (int i = 0; i < maxRetries; i++)
        {
            try { return await op(); }
            catch when (i < maxRetries - 1) { await Task.Delay(baseDelayMs * (i + 1)); }
        }
        throw new System.Exception("최대 재시도 횟수 초과");
    }
}
```

---

## 8. 스레드 안전성 테스트

```csharp
using NUnit.Framework;
using System.Collections.Concurrent;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

[TestFixture]
public class ThreadSafetyTests
{
    // ✅ 동시 접근 시 데이터 무결성 검증
    [Test]
    public async Task ThreadSafeCounter_MaintainsCorrectCount()
    {
        var counter = new ThreadSafeCounter();
        int total = 10000, concurrency = 10;

        var tasks = Enumerable.Range(0, concurrency)
            .Select(_ => Task.Run(() =>
            {
                for (int i = 0; i < total / concurrency; i++)
                    counter.Increment();
            })).ToArray();

        await Task.WhenAll(tasks);
        Assert.AreEqual(total, counter.Value, "모든 증가가 손실 없이 반영되어야 합니다");
    }

    // ✅ Producer-Consumer 패턴 테스트
    [Test]
    public async Task ProducerConsumer_AllItemsProcessed()
    {
        var queue = new ConcurrentQueue<int>();
        var processed = new ConcurrentBag<int>();
        int itemCount = 1000;
        var producerDone = new ManualResetEventSlim(false);

        var producer = Task.Run(() =>
        {
            for (int i = 0; i < itemCount; i++) queue.Enqueue(i);
            producerDone.Set();
        });

        var consumer = Task.Run(() =>
        {
            while (!producerDone.IsSet || !queue.IsEmpty)
            {
                if (queue.TryDequeue(out int item)) processed.Add(item);
                else Thread.SpinWait(10);
            }
        });

        await Task.WhenAll(producer, consumer);
        Assert.AreEqual(itemCount, processed.Count);
    }

    // ✅ ConcurrentDictionary 동시 읽기/쓰기
    [Test]
    public async Task ConcurrentDictionary_SafeReadWrite()
    {
        var dict = new ConcurrentDictionary<string, int>();

        var writers = Enumerable.Range(0, 5).Select(_ => Task.Run(() =>
        {
            for (int i = 0; i < 5000; i++)
                dict.AddOrUpdate($"key-{i % 100}", 1, (_, old) => old + 1);
        }));

        var readers = Enumerable.Range(0, 5).Select(_ => Task.Run(() =>
        {
            for (int i = 0; i < 5000; i++)
                dict.TryGetValue($"key-{i % 100}", out _);
        }));

        await Task.WhenAll(writers.Concat(readers)); // 예외 없이 완료되면 성공
        Assert.IsTrue(dict.Count > 0 && dict.Count <= 100);
    }
}

public class ThreadSafeCounter
{
    private int _value;
    public int Value => _value;
    public void Increment() => Interlocked.Increment(ref _value);
}
```

---

## 9. 통합 테스트

### 네트워크 호출 테스트 (Fake HTTP 클라이언트)

```csharp
using NUnit.Framework;
using System;
using System.Collections.Generic;
using System.Net;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

// ✅ HTTP 클라이언트 추상화
public interface IHttpClient
{
    Task<string> GetStringAsync(string url, CancellationToken ct = default);
}

// ✅ 테스트용 Fake
public class FakeHttpClient : IHttpClient
{
    private readonly Dictionary<string, string> _responses = new();
    public int RequestCount { get; private set; }
    public int SimulatedLatencyMs { get; set; } = 0;

    public void SetupResponse(string url, string response)
        => _responses[url] = response;

    public async Task<string> GetStringAsync(string url, CancellationToken ct = default)
    {
        RequestCount++;
        if (SimulatedLatencyMs > 0) await Task.Delay(SimulatedLatencyMs, ct);
        if (_responses.TryGetValue(url, out var resp)) return resp;
        throw new HttpRequestException($"404: {url}");
    }
}

// API 서비스
public class GameApiService
{
    private readonly IHttpClient _http;
    private readonly string _baseUrl;

    public GameApiService(IHttpClient http, string baseUrl)
    {
        _http = http;
        _baseUrl = baseUrl;
    }

    public async Task<PlayerData> GetPlayerAsync(string id, CancellationToken ct = default)
    {
        var json = await _http.GetStringAsync($"{_baseUrl}/players/{id}", ct);
        return JsonUtility.FromJson<PlayerData>(json);
    }
}

[Serializable]
public class PlayerData { public string id; public string name; public int score; }

// 테스트
[TestFixture]
public class GameApiServiceTests
{
    private FakeHttpClient _fakeHttp;
    private GameApiService _service;

    [SetUp]
    public void SetUp()
    {
        _fakeHttp = new FakeHttpClient();
        _service = new GameApiService(_fakeHttp, "https://api.game.com");
    }

    [Test]
    public async Task GetPlayer_ReturnsDeserializedData()
    {
        _fakeHttp.SetupResponse("https://api.game.com/players/p123",
            "{\"id\":\"p123\",\"name\":\"TestPlayer\",\"score\":100}");

        var player = await _service.GetPlayerAsync("p123");
        Assert.AreEqual("p123", player.id);
        Assert.AreEqual("TestPlayer", player.name);
        Assert.AreEqual(100, player.score);
    }

    [Test]
    public void GetPlayer_WhenServerError_ThrowsException()
    {
        Assert.ThrowsAsync<HttpRequestException>(
            async () => await _service.GetPlayerAsync("unknown"));
    }
}
```

---

## 10. CI/CD에서의 Unity 테스트

### GitHub Actions 설정

```yaml
# .github/workflows/unity-tests.yml
name: Unity Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        testMode: [ EditMode, PlayMode ]

    steps:
      - uses: actions/checkout@v4
        with: { lfs: true }

      - uses: actions/cache@v4
        with:
          path: Library
          key: Library-${{ hashFiles('Assets/**', 'Packages/**') }}

      - uses: game-ci/unity-test-runner@v4
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
        with:
          testMode: ${{ matrix.testMode }}
          artifactsPath: TestResults
          customParameters: -nographics -batchmode

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: Test Results (${{ matrix.testMode }})
          path: TestResults
```

### 커맨드라인 테스트 실행

```bash
#!/bin/bash
UNITY="/Applications/Unity/Hub/Editor/2022.3.0f1/Unity.app/Contents/MacOS/Unity"

# EditMode 테스트
"$UNITY" -runTests -batchmode -nographics \
    -projectPath "$(pwd)" \
    -testPlatform EditMode \
    -testResults ./TestResults/editmode-results.xml
EDIT_EXIT=$?

# PlayMode 테스트
"$UNITY" -runTests -batchmode -nographics \
    -projectPath "$(pwd)" \
    -testPlatform PlayMode \
    -testResults ./TestResults/playmode-results.xml
PLAY_EXIT=$?

echo "EditMode: $([ $EDIT_EXIT -eq 0 ] && echo 'PASSED' || echo 'FAILED')"
echo "PlayMode: $([ $PLAY_EXIT -eq 0 ] && echo 'PASSED' || echo 'FAILED')"
exit $(( EDIT_EXIT + PLAY_EXIT ))
```

```bash
# 필터링 실행
Unity -runTests -testCategory "AsyncTests"
Unity -runTests -testFilter ".*Async.*"
Unity -runTests -assemblyNames "Tests.EditMode"
```

---

## 11. 테스트 가능한 비동기 코드 작성 패턴

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

// ❌ 테스트하기 어려운 코드
public class HardToTestService : MonoBehaviour
{
    private HttpClient _client = new HttpClient();    // 직접 생성 → Mock 불가
    private float _time => Time.time;                  // static 의존 → 교체 불가

    async void Start()                                 // async void → 예외 추적 불가
    {
        var data = await _client.GetStringAsync(
            "https://api.game.com/data");              // 하드코딩된 URL
        Debug.Log(data);
    }
}

// ✅ 테스트하기 쉬운 코드 — 인터페이스 + 의존성 주입
public interface IGameDataService
{
    Task<GameData> LoadAsync(CancellationToken ct = default);
    Task SaveAsync(GameData data, CancellationToken ct = default);
}

public class GameDataService : IGameDataService
{
    private readonly IHttpClient _httpClient;
    private readonly string _baseUrl;

    public GameDataService(IHttpClient httpClient, string baseUrl)
    {
        _httpClient = httpClient;
        _baseUrl = baseUrl;
    }

    public async Task<GameData> LoadAsync(CancellationToken ct = default)
    {
        var json = await _httpClient.GetStringAsync($"{_baseUrl}/gamedata", ct);
        return JsonUtility.FromJson<GameData>(json);
    }

    public async Task SaveAsync(GameData data, CancellationToken ct = default)
    {
        // 저장 로직
    }
}

[Serializable]
public class GameData { public int level; public int score; public string playerName; }

// MonoBehaviour는 얇은 래퍼로만 사용
public class GameManager : MonoBehaviour
{
    private IGameDataService _dataService;

    public void Initialize(IGameDataService dataService)
        => _dataService = dataService;

    public async Task LoadGameAsync()
    {
        try
        {
            var data = await _dataService.LoadAsync(destroyCancellationToken);
            ApplyGameData(data);
        }
        catch (OperationCanceledException) { Debug.Log("로딩 취소됨"); }
        catch (Exception ex) { Debug.LogError($"로딩 실패: {ex.Message}"); }
    }

    private void ApplyGameData(GameData data) { /* 적용 */ }
}
```

### 상태 전이 검증 패턴

```csharp
public class StateMachine
{
    public enum State { Idle, Loading, Ready, Error }
    public State CurrentState { get; private set; } = State.Idle;
    public event Action<State> StateChanged;

    private readonly IGameDataService _service;
    public StateMachine(IGameDataService service) => _service = service;

    public async Task InitializeAsync(CancellationToken ct = default)
    {
        SetState(State.Loading);
        try { await _service.LoadAsync(ct); SetState(State.Ready); }
        catch { SetState(State.Error); }
    }

    private void SetState(State s) { CurrentState = s; StateChanged?.Invoke(s); }
}

[TestFixture]
public class StateMachineTests
{
    [Test]
    public async Task Initialize_TransitionsToReady()
    {
        var mock = new MockGameDataService();
        var sm = new StateMachine(mock);
        var transitions = new System.Collections.Generic.List<StateMachine.State>();
        sm.StateChanged += s => transitions.Add(s);

        await sm.InitializeAsync();

        Assert.AreEqual(StateMachine.State.Ready, sm.CurrentState);
        Assert.AreEqual(new[] {
            StateMachine.State.Loading,
            StateMachine.State.Ready
        }, transitions.ToArray());
    }
}

public class MockGameDataService : IGameDataService
{
    public bool ShouldFail { get; set; }
    public async Task<GameData> LoadAsync(CancellationToken ct = default)
    {
        await Task.Delay(10, ct);
        if (ShouldFail) throw new Exception("Mock failure");
        return new GameData { level = 1 };
    }
    public Task SaveAsync(GameData data, CancellationToken ct = default)
        => Task.CompletedTask;
}
```

---

## 주의사항

```csharp
// ❌ 실수 1: async void 테스트 — NUnit이 완료를 추적하지 못함
[Test]
public async void BrokenTest()
{
    await Task.Delay(100);
    Assert.IsTrue(false); // 이 실패가 보고되지 않을 수 있음
}

// ✅ async Task 반환
[Test]
public async Task CorrectTest()
{
    await Task.Delay(100);
    Assert.IsTrue(true);
}

// ❌ 실수 2: 타이밍에 의존하는 불안정한 테스트
[Test]
public async Task FlakyTest()
{
    var task = SomeAsyncOperation();
    await Task.Delay(100);              // "충분히" 기다렸겠지?
    Assert.IsTrue(task.IsCompleted);    // CI에서 실패할 수 있음
}

// ✅ 완료를 직접 await
[Test]
public async Task StableTest()
{
    var result = await SomeAsyncOperation();
    Assert.IsNotNull(result);
}

// ❌ 실수 3: PlayMode에서 리소스 정리 누락
[UnityTest]
public System.Collections.IEnumerator LeakyTest()
{
    var go = new GameObject("Leak");
    yield return null;
    // GameObject 정리 안 됨! 다음 테스트에 영향
}

// ✅ 반드시 정리
[UnityTest]
public System.Collections.IEnumerator CleanTest()
{
    var go = new GameObject("Clean");
    try { yield return null; }
    finally { Object.DestroyImmediate(go); }
}

// ❌ 실수 4: 메인 스레드에서 .Result → 데드락
[Test]
public void DeadlockTest() { var r = SomeAsync().Result; }

// ✅ async Task 사용
[Test]
public async Task NoDeadlock() { var r = await SomeAsync(); }
```

---

## 베스트 프랙티스

### 비동기 테스트 체크리스트

| 항목 | 설명 |
|------|------|
| **async Task 반환** | async void 대신 async Task 사용 |
| **타임아웃 설정** | `[Timeout(ms)]`으로 무한 대기 방지 |
| **CancellationToken** | 취소 시나리오 반드시 검증 |
| **예외 경로** | 정상/비정상 경로 모두 커버 |
| **리소스 정리** | SetUp/TearDown에서 상태 초기화 |
| **결정적 테스트** | 타이밍이 아닌 완료 상태로 검증 |
| **독립적 테스트** | 테스트 간 공유 상태 제거 |
| **시간 추상화** | ITimeProvider로 시간 의존성 분리 |

### AAA 패턴 (Arrange-Act-Assert)

```csharp
[Test]
public async Task BestPractice_AAAPattern()
{
    // ===== Arrange =====
    var mockRepo = new MockDataRepository { GetByIdResult = "expected" };
    var service = new CacheService(mockRepo);

    // ===== Act =====
    var result = await service.GetOrLoadAsync("key1");

    // ===== Assert =====
    Assert.AreEqual("expected", result);
    Assert.AreEqual(1, mockRepo.GetByIdCallCount);
}
```

### 테스트 명명 규칙

```csharp
// ✅ 좋은 명명: [대상]_[시나리오]_[기대결과]
[Test] public async Task LoadData_WithValidKey_ReturnsData() { }
[Test] public async Task LoadData_WithNullKey_ThrowsArgumentNull() { }
[Test] public async Task LoadData_WhenCancelled_ThrowsOperationCanceled() { }

// ❌ 나쁜 명명
[Test] public async Task Test1() { }
[Test] public async Task ItWorks() { }
```

### 테스트 구성 요약

```
테스트 피라미드 (비동기 코드):

         /  E2E  \           ← 최소한: 전체 워크플로우 통합
        / 통합 테스트 \        ← 적당히: API + 서비스 연동
       / 컴포넌트 테스트 \      ← 많이: Mock 의존성 + 비동기 흐름
      /   단위 테스트    \     ← 가장 많이: 순수 로직, 동기 테스트
     ──────────────────────

    EditMode: 단위 + 컴포넌트 테스트
    PlayMode: 통합 + E2E 테스트
```

---

## 정리

### 테스트 유형별 선택 가이드

| 테스트 대상 | 테스트 유형 | 어트리뷰트 |
|------------|-----------|-----------|
| 순수 로직 (계산, 변환) | EditMode 동기 | `[Test]` |
| async/await 메서드 | EditMode 비동기 | `[Test] async Task` |
| Coroutine 동작 | PlayMode | `[UnityTest] IEnumerator` |
| UniTask 메서드 | PlayMode | `[UnityTest] + ToCoroutine` |
| MonoBehaviour 라이프사이클 | PlayMode | `[UnityTest] IEnumerator` |
| 스레드 안전성 | EditMode 비동기 | `[Test] async Task` |
| 네트워크 통합 | PlayMode + Mock | `[Test] async Task` |
| 타이밍/프레임 의존 | PlayMode + Fake Time | `[UnityTest]` |

---

## 참고 자료

- [Unity Test Framework Documentation](https://docs.unity3d.com/Packages/com.unity.test-framework@latest)
- [NUnit 3 Documentation](https://docs.nunit.org/)
- [Unity Test Runner](https://docs.unity3d.com/Manual/testing-editortestsrunner.html)
- [GameCI - Unity CI/CD](https://game.ci/)
- [UniTask Testing Patterns](https://github.com/Cysharp/UniTask#testing)
- [Microsoft - Unit Testing Async Code](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices)
