# 03. SynchronizationContext

## 개요

`SynchronizationContext`는 .NET에서 스레드 간 작업 마샬링을 추상화하는 핵심 클래스입니다. Unity는 `UnitySynchronizationContext`를 통해 async/await 패턴이 메인 스레드로 자동 복귀하도록 지원합니다. 이 개념을 이해하면 Unity에서의 비동기 프로그래밍을 더 깊이 이해하고 활용할 수 있습니다.

---

## 1. SynchronizationContext란?

### 기본 개념

`SynchronizationContext`는 코드가 실행되어야 할 "환경" 또는 "컨텍스트"를 나타냅니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                   SynchronizationContext 역할                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  "이 작업을 특정 스레드/환경에서 실행해주세요"                       │
│                                                                  │
│  ┌─────────────┐       Post/Send        ┌─────────────────┐     │
│  │ 백그라운드   │ ─────────────────────▶ │   메인 스레드    │     │
│  │   스레드    │                        │    (UI 스레드)   │     │
│  └─────────────┘                        └─────────────────┘     │
│                                                                  │
│  대표적인 구현:                                                   │
│  • WindowsFormsSynchronizationContext (WinForms)                │
│  • DispatcherSynchronizationContext (WPF)                       │
│  • AspNetSynchronizationContext (ASP.NET)                       │
│  • UnitySynchronizationContext (Unity)                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 핵심 메서드

```csharp
public class SynchronizationContext
{
    // 비동기로 작업 전달 (호출자는 즉시 반환)
    public virtual void Post(SendOrPostCallback d, object state);

    // 동기로 작업 전달 (완료될 때까지 대기)
    public virtual void Send(SendOrPostCallback d, object state);

    // 현재 스레드의 SynchronizationContext 가져오기
    public static SynchronizationContext Current { get; }

    // 현재 스레드에 SynchronizationContext 설정
    public static void SetSynchronizationContext(SynchronizationContext syncContext);
}
```

---

## 2. UnitySynchronizationContext

### Unity에서의 동작 방식

Unity는 메인 스레드에서 `UnitySynchronizationContext`를 제공합니다.

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class UnitySyncContextExample : MonoBehaviour
{
    private void Start()
    {
        // Unity 메인 스레드의 SynchronizationContext 확인
        var context = SynchronizationContext.Current;

        Debug.Log($"SynchronizationContext Type: {context?.GetType().Name}");
        // 출력: UnitySynchronizationContext

        Debug.Log($"Main Thread ID: {Thread.CurrentThread.ManagedThreadId}");

        // async/await 테스트
        TestAsyncAwait();
    }

    private async void TestAsyncAwait()
    {
        Debug.Log($"async 시작 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        // Task.Delay는 ThreadPool 타이머 사용
        await Task.Delay(100);

        // ✅ UnitySynchronizationContext가 메인 스레드로 복귀시킴
        Debug.Log($"await 후 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        // Unity API 안전하게 사용 가능
        transform.position = Vector3.zero;
    }
}
```

### UnitySynchronizationContext의 동작 원리

```
┌─────────────────────────────────────────────────────────────────┐
│              UnitySynchronizationContext 동작 흐름               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. async 메서드 시작 (메인 스레드)                                │
│     └─ SynchronizationContext.Current 캡처                       │
│                                                                  │
│  2. await Task.Delay(100)                                       │
│     └─ continuation(이후 코드)을 SynchronizationContext에 등록    │
│     └─ 메인 스레드는 다른 작업 수행 가능                           │
│                                                                  │
│  3. 100ms 후 타이머 완료 (ThreadPool 스레드)                      │
│     └─ continuation을 UnitySynchronizationContext.Post로 전달    │
│                                                                  │
│  4. Unity PlayerLoop의 다음 프레임                               │
│     └─ UnitySynchronizationContext가 큐에서 작업 꺼냄            │
│     └─ 메인 스레드에서 continuation 실행                          │
│                                                                  │
│  5. await 이후 코드 실행 (메인 스레드)                             │
│     └─ Unity API 안전하게 사용 가능                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. SynchronizationContext 캡처와 복원

### 캡처 (Capture)

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class SyncContextCaptureExample : MonoBehaviour
{
    private SynchronizationContext capturedContext;
    private int mainThreadId;

    private void Start()
    {
        mainThreadId = Thread.CurrentThread.ManagedThreadId;

        // ✅ 메인 스레드에서 SynchronizationContext 캡처
        capturedContext = SynchronizationContext.Current;

        Debug.Log($"캡처된 SyncContext: {capturedContext?.GetType().Name}");
        Debug.Log($"메인 스레드 ID: {mainThreadId}");

        // 백그라운드에서 작업 후 메인 스레드로 결과 전달
        StartBackgroundWork();
    }

    private void StartBackgroundWork()
    {
        // 명시적으로 새 스레드 생성
        new Thread(() =>
        {
            int threadId = Thread.CurrentThread.ManagedThreadId;
            Debug.Log($"[백그라운드] Thread ID: {threadId}");
            Debug.Log($"[백그라운드] SyncContext: {SynchronizationContext.Current?.GetType().Name ?? "null"}");

            // 무거운 연산 수행
            int result = HeavyCalculation();

            // 캡처해둔 SynchronizationContext를 통해 메인 스레드로 전달
            capturedContext.Post(_ =>
            {
                int postThreadId = Thread.CurrentThread.ManagedThreadId;
                Debug.Log($"[Post 콜백] Thread ID: {postThreadId}");
                Debug.Log($"[Post 콜백] 메인 스레드 여부: {postThreadId == mainThreadId}");

                // ✅ 메인 스레드에서 실행되므로 Unity API 사용 가능
                transform.position = new Vector3(result, 0, 0);

            }, null);

        }).Start();
    }

    private int HeavyCalculation()
    {
        Thread.Sleep(500); // 시뮬레이션
        return 42;
    }
}
```

### async/await에서의 자동 캡처

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class AsyncAwaitCaptureExample : MonoBehaviour
{
    // =============================================
    // async/await는 SynchronizationContext를 자동 캡처
    // =============================================

    private async void Start()
    {
        Debug.Log($"[Start] Thread: {Thread.CurrentThread.ManagedThreadId}");
        Debug.Log($"[Start] SyncContext: {SynchronizationContext.Current?.GetType().Name}");

        // await 시점에 현재 SynchronizationContext가 자동으로 캡처됨
        await Task.Run(() =>
        {
            Debug.Log($"[Task.Run 내부] Thread: {Thread.CurrentThread.ManagedThreadId}");
            Debug.Log($"[Task.Run 내부] SyncContext: {SynchronizationContext.Current?.GetType().Name ?? "null"}");

            Thread.Sleep(100);
            return 42;
        });

        // await 완료 후 캡처된 SynchronizationContext로 복귀
        Debug.Log($"[await 후] Thread: {Thread.CurrentThread.ManagedThreadId}");
        Debug.Log($"[await 후] SyncContext: {SynchronizationContext.Current?.GetType().Name}");

        // ✅ 메인 스레드이므로 Unity API 사용 가능
        transform.position = Vector3.one;
    }
}
```

### 출력 예시

```
[Start] Thread: 1
[Start] SyncContext: UnitySynchronizationContext
[Task.Run 내부] Thread: 4
[Task.Run 내부] SyncContext: null
[await 후] Thread: 1
[await 후] SyncContext: UnitySynchronizationContext
```

---

## 4. ConfigureAwait와 SynchronizationContext

### ConfigureAwait(false)의 의미

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class ConfigureAwaitExample : MonoBehaviour
{
    private async void Start()
    {
        Debug.Log($"시작 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        await ConfigureAwaitTrueExample();
        await ConfigureAwaitFalseExample();
    }

    // =============================================
    // ConfigureAwait(true) - 기본값
    // =============================================

    private async Task ConfigureAwaitTrueExample()
    {
        Debug.Log("\n=== ConfigureAwait(true) ===");
        Debug.Log($"await 전 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        // ConfigureAwait(true)는 기본값
        // SynchronizationContext를 캡처하고 복원함
        await Task.Delay(100).ConfigureAwait(true);

        // ✅ 메인 스레드로 복귀
        Debug.Log($"await 후 - Thread: {Thread.CurrentThread.ManagedThreadId}");
        Debug.Log($"SyncContext: {SynchronizationContext.Current?.GetType().Name}");

        // Unity API 사용 가능
        transform.position = Vector3.zero;
    }

    // =============================================
    // ConfigureAwait(false)
    // =============================================

    private async Task ConfigureAwaitFalseExample()
    {
        Debug.Log("\n=== ConfigureAwait(false) ===");
        Debug.Log($"await 전 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        // ConfigureAwait(false)는 SynchronizationContext를 무시
        // 완료된 스레드(보통 ThreadPool)에서 계속 실행
        await Task.Delay(100).ConfigureAwait(false);

        // ⚠️ ThreadPool 스레드에서 실행될 수 있음!
        Debug.Log($"await 후 - Thread: {Thread.CurrentThread.ManagedThreadId}");
        Debug.Log($"SyncContext: {SynchronizationContext.Current?.GetType().Name ?? "null"}");

        // ❌ Unity API 사용 불가! (메인 스레드가 아닐 수 있음)
        // transform.position = Vector3.one; // UnityException 발생 가능!
    }

    // =============================================
    // 실전 패턴: 라이브러리 코드
    // =============================================

    /// <summary>
    /// 라이브러리/유틸리티 메서드에서는 ConfigureAwait(false) 권장
    /// - 불필요한 스레드 전환 방지
    /// - 성능 향상
    /// - 데드락 방지 (특히 .Result 사용 시)
    /// </summary>
    private async Task<string> FetchDataFromServerAsync(string url)
    {
        using (var client = new System.Net.Http.HttpClient())
        {
            // 라이브러리 코드에서는 ConfigureAwait(false) 사용
            var response = await client.GetAsync(url).ConfigureAwait(false);
            var content = await response.Content.ReadAsStringAsync().ConfigureAwait(false);

            // 이 시점에서 ThreadPool 스레드에서 실행될 수 있음
            // 순수 C# 로직만 수행
            return ProcessData(content);
        }
    }

    private string ProcessData(string raw)
    {
        return raw.ToUpper();
    }

    // =============================================
    // Unity 앱 코드에서의 패턴
    // =============================================

    private async void OnButtonClick()
    {
        // UI/Unity 코드에서는 ConfigureAwait 생략 (기본값 true)
        string data = await FetchDataFromServerAsync("https://api.example.com");

        // await 후 메인 스레드로 복귀되어 Unity API 사용 가능
        Debug.Log($"데이터 로드 완료: {data}");
        transform.position = Vector3.one;
    }
}
```

### ConfigureAwait 선택 가이드

```
┌─────────────────────────────────────────────────────────────────┐
│              ConfigureAwait 선택 가이드                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ConfigureAwait(true) 또는 생략 (기본값)                         │
│  ├─ Unity MonoBehaviour 코드                                    │
│  ├─ await 후 Unity API 호출이 필요한 경우                        │
│  ├─ UI 업데이트가 필요한 경우                                    │
│  └─ 메인 스레드 컨텍스트가 필요한 경우                            │
│                                                                  │
│  ConfigureAwait(false)                                          │
│  ├─ 라이브러리/유틸리티 코드                                     │
│  ├─ await 후 Unity API가 필요 없는 경우                          │
│  ├─ 순수 데이터 처리 로직                                        │
│  ├─ 성능 최적화가 필요한 경우                                    │
│  └─ 데드락 방지 (동기 호출과 혼합 시)                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. 커스텀 SynchronizationContext 구현

### 단순한 메인 스레드 디스패처 구현

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Collections.Concurrent;

/// <summary>
/// 메인 스레드로 작업을 마샬링하는 간단한 디스패처
/// UnitySynchronizationContext의 동작 원리 이해용
/// </summary>
public class MainThreadDispatcher : MonoBehaviour
{
    private static MainThreadDispatcher instance;
    private static readonly ConcurrentQueue<Action> actionQueue = new ConcurrentQueue<Action>();
    private static int mainThreadId;

    public static MainThreadDispatcher Instance
    {
        get
        {
            if (instance == null)
            {
                var go = new GameObject("MainThreadDispatcher");
                instance = go.AddComponent<MainThreadDispatcher>();
                DontDestroyOnLoad(go);
            }
            return instance;
        }
    }

    private void Awake()
    {
        if (instance != null && instance != this)
        {
            Destroy(gameObject);
            return;
        }

        instance = this;
        mainThreadId = Thread.CurrentThread.ManagedThreadId;
        DontDestroyOnLoad(gameObject);
    }

    private void Update()
    {
        // 매 프레임 큐에 있는 작업들을 메인 스레드에서 실행
        while (actionQueue.TryDequeue(out var action))
        {
            try
            {
                action?.Invoke();
            }
            catch (Exception e)
            {
                Debug.LogException(e);
            }
        }
    }

    /// <summary>
    /// 메인 스레드에서 실행할 작업 큐에 추가
    /// </summary>
    public static void Enqueue(Action action)
    {
        if (action == null) return;

        // 이미 메인 스레드면 즉시 실행
        if (Thread.CurrentThread.ManagedThreadId == mainThreadId)
        {
            action();
        }
        else
        {
            actionQueue.Enqueue(action);
        }
    }

    /// <summary>
    /// 메인 스레드 여부 확인
    /// </summary>
    public static bool IsMainThread => Thread.CurrentThread.ManagedThreadId == mainThreadId;
}

/// <summary>
/// 커스텀 SynchronizationContext 구현
/// </summary>
public class CustomUnitySyncContext : SynchronizationContext
{
    public override void Post(SendOrPostCallback d, object state)
    {
        // Post는 비동기로 작업 전달
        MainThreadDispatcher.Enqueue(() => d(state));
    }

    public override void Send(SendOrPostCallback d, object state)
    {
        // Send는 동기로 작업 전달 (완료까지 대기)
        if (MainThreadDispatcher.IsMainThread)
        {
            // 이미 메인 스레드면 즉시 실행
            d(state);
        }
        else
        {
            // 다른 스레드면 ManualResetEvent로 대기
            using (var waitHandle = new ManualResetEvent(false))
            {
                MainThreadDispatcher.Enqueue(() =>
                {
                    try
                    {
                        d(state);
                    }
                    finally
                    {
                        waitHandle.Set();
                    }
                });

                waitHandle.WaitOne();
            }
        }
    }

    public override SynchronizationContext CreateCopy()
    {
        return new CustomUnitySyncContext();
    }
}
```

### 커스텀 SynchronizationContext 사용 예제

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class CustomSyncContextUsageExample : MonoBehaviour
{
    private void Start()
    {
        // MainThreadDispatcher 초기화
        var _ = MainThreadDispatcher.Instance;

        TestWithCustomSyncContext();
    }

    private void TestWithCustomSyncContext()
    {
        // 백그라운드 스레드에서 시작
        Task.Run(async () =>
        {
            Debug.Log($"[Task.Run] Thread: {Thread.CurrentThread.ManagedThreadId}");
            Debug.Log($"[Task.Run] SyncContext: {SynchronizationContext.Current?.GetType().Name ?? "null"}");

            // MainThreadDispatcher를 통해 메인 스레드로 작업 전달
            MainThreadDispatcher.Enqueue(() =>
            {
                Debug.Log($"[Enqueue 콜백] Thread: {Thread.CurrentThread.ManagedThreadId}");

                // ✅ 메인 스레드에서 실행
                transform.position = new Vector3(1, 2, 3);
            });
        });
    }
}
```

---

## 6. Post vs Send

### 차이점

```csharp
using UnityEngine;
using System;
using System.Threading;

public class PostVsSendExample : MonoBehaviour
{
    private SynchronizationContext mainContext;

    private void Start()
    {
        mainContext = SynchronizationContext.Current;

        TestPost();
        TestSend();
    }

    // =============================================
    // Post: 비동기, 즉시 반환
    // =============================================

    private void TestPost()
    {
        Debug.Log("=== Post 테스트 ===");

        new Thread(() =>
        {
            Debug.Log($"[스레드] Post 호출 전 - {DateTime.Now:HH:mm:ss.fff}");

            // Post는 즉시 반환됨 (비동기)
            mainContext.Post(_ =>
            {
                Thread.Sleep(500); // 시뮬레이션
                Debug.Log($"[Post 콜백] 실행됨 - {DateTime.Now:HH:mm:ss.fff}");
            }, null);

            // Post 호출 후 즉시 이 줄이 실행됨
            Debug.Log($"[스레드] Post 호출 후 - {DateTime.Now:HH:mm:ss.fff}");

        }).Start();
    }

    // =============================================
    // Send: 동기, 완료까지 대기
    // =============================================

    private void TestSend()
    {
        Debug.Log("\n=== Send 테스트 ===");

        new Thread(() =>
        {
            Debug.Log($"[스레드] Send 호출 전 - {DateTime.Now:HH:mm:ss.fff}");

            // ⚠️ Unity의 UnitySynchronizationContext.Send는
            // 실제로 Post처럼 동작할 수 있음 (구현에 따라 다름)

            // Send는 완료될 때까지 대기 (동기)
            mainContext.Send(_ =>
            {
                Thread.Sleep(500); // 시뮬레이션
                Debug.Log($"[Send 콜백] 실행됨 - {DateTime.Now:HH:mm:ss.fff}");
            }, null);

            // Send 콜백이 완료된 후에야 이 줄이 실행됨
            Debug.Log($"[스레드] Send 호출 후 - {DateTime.Now:HH:mm:ss.fff}");

        }).Start();
    }
}
```

### 출력 예시

```
=== Post 테스트 ===
[스레드] Post 호출 전 - 12:00:00.000
[스레드] Post 호출 후 - 12:00:00.001  // 즉시 반환
[Post 콜백] 실행됨 - 12:00:00.016     // 다음 프레임에서 실행

=== Send 테스트 ===
[스레드] Send 호출 전 - 12:00:00.500
[Send 콜백] 실행됨 - 12:00:01.000     // 콜백 완료 후
[스레드] Send 호출 후 - 12:00:01.001  // 콜백 완료까지 대기
```

---

## 7. 데드락 시나리오와 해결

### 데드락 발생 예제

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class DeadlockExample : MonoBehaviour
{
    // =============================================
    // ❌ 데드락 발생 시나리오
    // =============================================

    private void Start()
    {
        // 이 호출은 데드락을 발생시킵니다!
        // DeadlockScenario();
    }

    private void DeadlockScenario()
    {
        Debug.Log("데드락 시나리오 시작...");

        // ❌ 절대 하지 마세요!
        // .Result 또는 .Wait()는 async 메서드와 함께 사용하면 데드락 발생

        // 1. GetDataAsync 호출
        // 2. await Task.Delay에서 SynchronizationContext 캡처
        // 3. .Result가 메인 스레드를 블로킹
        // 4. Task.Delay 완료 후 SynchronizationContext.Post로 메인 스레드에 복귀 시도
        // 5. 메인 스레드는 .Result로 블로킹되어 있어서 Post 처리 불가
        // 6. 데드락!

        string result = GetDataAsync().Result; // ← 데드락!

        Debug.Log($"결과: {result}"); // 여기 도달 불가
    }

    private async Task<string> GetDataAsync()
    {
        Debug.Log("GetDataAsync 시작");

        // await 시점에 UnitySynchronizationContext 캡처
        await Task.Delay(100);
        // ↑ 이 continuation이 메인 스레드로 Post되어야 하는데
        //   메인 스레드는 .Result로 블로킹되어 있어서 실행 불가

        Debug.Log("GetDataAsync 완료"); // 도달 불가
        return "data";
    }

    // =============================================
    // ✅ 해결책 1: async/await 사용 (권장)
    // =============================================

    private async void CorrectSolution1()
    {
        Debug.Log("해결책 1: async/await");

        // ✅ await를 사용하면 메인 스레드를 블로킹하지 않음
        string result = await GetDataAsync();

        Debug.Log($"결과: {result}");
    }

    // =============================================
    // ✅ 해결책 2: ConfigureAwait(false)
    // =============================================

    private void CorrectSolution2()
    {
        Debug.Log("해결책 2: ConfigureAwait(false)");

        // ConfigureAwait(false)를 사용하면 SynchronizationContext를 무시
        // 따라서 메인 스레드로 복귀하지 않아 데드락 방지
        string result = GetDataWithoutContextAsync().Result;

        Debug.Log($"결과: {result}");

        // ⚠️ 단, 이 경우 GetDataWithoutContextAsync 내부에서
        // Unity API를 사용하면 안 됨!
    }

    private async Task<string> GetDataWithoutContextAsync()
    {
        // ConfigureAwait(false)로 SynchronizationContext 무시
        await Task.Delay(100).ConfigureAwait(false);

        // ⚠️ 여기서는 Unity API 사용 불가!
        return "data";
    }

    // =============================================
    // ✅ 해결책 3: Task.Run으로 감싸기
    // =============================================

    private void CorrectSolution3()
    {
        Debug.Log("해결책 3: Task.Run");

        // Task.Run 내부는 ThreadPool 스레드이므로
        // SynchronizationContext가 null → 데드락 없음
        string result = Task.Run(() => GetDataAsync()).Result;

        Debug.Log($"결과: {result}");

        // ⚠️ 메인 스레드를 블로킹하는 것은 여전히 나쁜 패턴!
    }
}
```

### 데드락 시각화

```
┌─────────────────────────────────────────────────────────────────┐
│                       데드락 시나리오                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  메인 스레드                         ThreadPool                  │
│  ──────────────                     ────────────                 │
│                                                                  │
│  1. GetDataAsync().Result 호출                                   │
│     │                                                            │
│     ▼                                                            │
│  2. await Task.Delay(100)                                       │
│     └─ SyncContext 캡처 (Unity)                                  │
│     │                                                            │
│     ▼                                                            │
│  3. .Result가 스레드 블로킹          4. 100ms 후 타이머 완료       │
│     │                                   │                        │
│     │ ┌─────────────────────────────────┘                        │
│     │ │                                                          │
│     │ ▼                                                          │
│     │ 5. SyncContext.Post 호출                                   │
│     │    "메인 스레드에서 continuation 실행해줘"                   │
│     │                                                            │
│     │                              6. Post된 작업이 큐에 대기     │
│     │                                 하지만 메인 스레드가        │
│     │                                 블로킹되어 처리 불가!       │
│     │                                                            │
│     ▼                                                            │
│  [블로킹 상태 유지]                  [처리 대기 상태]              │
│                                                                  │
│                    💀 데드락! 💀                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. 실전 패턴

### 패턴 1: 백그라운드 연산 후 UI 업데이트

```csharp
using UnityEngine;
using UnityEngine.UI;
using System.Threading;
using System.Threading.Tasks;

public class BackgroundToUIPattern : MonoBehaviour
{
    [SerializeField] private Text statusText;
    [SerializeField] private Slider progressSlider;

    private SynchronizationContext mainContext;

    private void Start()
    {
        mainContext = SynchronizationContext.Current;
        StartHeavyComputation();
    }

    private async void StartHeavyComputation()
    {
        statusText.text = "계산 시작...";

        // IProgress<T>를 통한 진행률 보고
        var progress = new Progress<float>(value =>
        {
            // Progress<T>는 캡처된 SynchronizationContext에서 콜백 실행
            progressSlider.value = value;
            statusText.text = $"진행률: {value:P0}";
        });

        // 백그라운드에서 무거운 연산 수행
        int result = await Task.Run(() => HeavyComputation(progress));

        // await 후 메인 스레드로 복귀
        statusText.text = $"완료! 결과: {result}";
    }

    private int HeavyComputation(IProgress<float> progress)
    {
        int result = 0;
        int total = 10000000;

        for (int i = 0; i < total; i++)
        {
            result += i % 100;

            // 주기적으로 진행률 보고
            if (i % 100000 == 0)
            {
                progress?.Report((float)i / total);
            }
        }

        progress?.Report(1f);
        return result;
    }
}
```

### 패턴 2: 취소 가능한 비동기 작업

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class CancellableAsyncPattern : MonoBehaviour
{
    private CancellationTokenSource cts;

    private void OnEnable()
    {
        cts = new CancellationTokenSource();
        StartPeriodicTask(cts.Token);
    }

    private void OnDisable()
    {
        // 컴포넌트 비활성화 시 취소
        cts?.Cancel();
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }

    private async void StartPeriodicTask(CancellationToken token)
    {
        try
        {
            while (!token.IsCancellationRequested)
            {
                // 백그라운드 작업
                var data = await Task.Run(() => FetchData(), token);

                // 메인 스레드에서 UI 업데이트
                // (UnitySynchronizationContext가 자동으로 복귀시킴)
                UpdateUI(data);

                // 다음 주기까지 대기
                await Task.Delay(1000, token);
            }
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 취소되었습니다.");
        }
    }

    private string FetchData()
    {
        Thread.Sleep(100); // 시뮬레이션
        return $"Data at {System.DateTime.Now}";
    }

    private void UpdateUI(string data)
    {
        Debug.Log($"UI 업데이트: {data}");
    }
}
```

---

## 주의사항

1. **메인 스레드에서만 캡처**: `SynchronizationContext.Current`는 메인 스레드에서만 유효한 값 반환
2. **ConfigureAwait(false) 주의**: 사용 후에는 Unity API 호출 불가
3. **.Result / .Wait() 금지**: 메인 스레드에서 사용 시 데드락 위험
4. **백그라운드 스레드 주의**: Task.Run 내부에서는 `SynchronizationContext.Current`가 null
5. **에디터 vs 런타임**: 에디터에서의 SynchronizationContext 동작이 다를 수 있음

---

## 참고 자료

- [Microsoft: SynchronizationContext](https://docs.microsoft.com/en-us/dotnet/api/system.threading.synchronizationcontext)
- [Stephen Cleary: There Is No Thread](https://blog.stephencleary.com/2013/11/there-is-no-thread.html)
- [Stephen Cleary: Don't Block on Async Code](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html)
- [Unity Forum: UnitySynchronizationContext](https://forum.unity.com/threads/using-async-await-in-unity.472659/)

---

## 다음 섹션

[04. Thread & ThreadPool](../02-threading/04-thread-and-threadpool.md)
