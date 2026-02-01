# 40. 프로파일링 & 디버깅

## 개요

동시성 및 비동기 코드는 단일 스레드 코드에 비해 디버깅과 프로파일링이 훨씬 어렵습니다. 비결정적 실행 순서, 스레드 간 상호작용, 비동기 스택 트레이스 손실 등의 문제가 복합적으로 작용하기 때문입니다. 이 섹션에서는 Unity 환경에서 동시성 코드를 효과적으로 프로파일링하고 디버깅하는 전략과 도구를 다룹니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│              동시성 코드 프로파일링 & 디버깅 도구                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  프로파일링                    │  디버깅                              │
│  ─────────────────────────────┼──────────────────────────────────── │
│  Unity Profiler (CPU/Memory)  │  async 스택 트레이스                 │
│  ProfilerMarker               │  데드락 진단                         │
│  CustomSampler                │  Race Condition 탐지                 │
│  Deep Profiling               │  Thread Safety 검증                  │
│  Frame Debugger               │  Rider / Visual Studio 디버거        │
│  Memory Profiler              │  구조화된 로깅                       │
│  UniTask Tracker              │  조건부 브레이크포인트                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Unity Profiler 활용

### CPU Usage 모듈

```csharp
using UnityEngine;
using UnityEngine.Profiling;
using System.Threading.Tasks;

public class CpuProfilingExample : MonoBehaviour
{
    // =============================================
    // ProfilerMarker를 이용한 구간 측정
    // =============================================

    // ProfilerMarker는 구조체 기반으로 GC 할당 없음
    private static readonly ProfilerMarker s_ProcessDataMarker =
        new ProfilerMarker("MyGame.ProcessData");

    private static readonly ProfilerMarker s_AsyncOperationMarker =
        new ProfilerMarker("MyGame.AsyncOperation");

    private static readonly ProfilerMarker s_ParallelWorkMarker =
        new ProfilerMarker("MyGame.ParallelWork");

    private void Update()
    {
        // 동기 작업 프로파일링
        using (s_ProcessDataMarker.Auto())
        {
            ProcessGameData();
        }
    }

    private void ProcessGameData()
    {
        // 게임 데이터 처리 로직
        // Profiler에서 "MyGame.ProcessData"로 표시됨
        for (int i = 0; i < 1000; i++)
        {
            // 시뮬레이션 작업
            _ = Mathf.Sqrt(i);
        }
    }

    // =============================================
    // 비동기 작업에 ProfilerMarker 적용
    // =============================================

    private async void Start()
    {
        // 비동기 작업의 시작과 끝을 별도로 마킹
        s_AsyncOperationMarker.Begin();

        try
        {
            await LoadDataAsync();
        }
        finally
        {
            s_AsyncOperationMarker.End();
        }
    }

    private async Task LoadDataAsync()
    {
        // 비동기 로딩 시뮬레이션
        await Task.Delay(100);
        Debug.Log("데이터 로드 완료");
    }
}
```

### Timeline 뷰 분석

```csharp
using UnityEngine;
using UnityEngine.Profiling;
using Unity.Profiling;
using System.Threading;
using System.Threading.Tasks;

public class TimelineProfilingExample : MonoBehaviour
{
    // =============================================
    // ProfilerMarker로 Timeline에 구간 표시
    // =============================================

    // 카테고리를 지정하면 Profiler에서 색상으로 구분 가능
    private static readonly ProfilerMarker s_MainThreadWork =
        new ProfilerMarker(ProfilerCategory.Scripts, "MainThread.GameLogic");

    private static readonly ProfilerMarker s_BackgroundWork =
        new ProfilerMarker(ProfilerCategory.Loading, "Background.DataLoad");

    private static readonly ProfilerMarker s_RenderPrep =
        new ProfilerMarker(ProfilerCategory.Render, "Render.AsyncPrep");

    private void Update()
    {
        // Timeline 뷰에서 메인 스레드 구간 확인 가능
        using (s_MainThreadWork.Auto())
        {
            RunGameLogic();
        }
    }

    private void RunGameLogic()
    {
        // 메인 스레드 작업
        // Profiler Timeline에서 "MainThread.GameLogic" 블록으로 표시
        Thread.Sleep(1); // 시뮬레이션
    }

    // =============================================
    // 백그라운드 스레드 작업 프로파일링
    // =============================================

    private async Task ProcessInBackgroundAsync()
    {
        await Task.Run(() =>
        {
            // 백그라운드 스레드에서도 ProfilerMarker 사용 가능
            // Timeline 뷰에서 Worker Thread에 표시됨
            using (s_BackgroundWork.Auto())
            {
                // 무거운 계산 작업
                for (int i = 0; i < 10000; i++)
                {
                    _ = Mathf.Sqrt(i);
                }
            }
        });
    }
}
```

### Memory 프로파일링

```csharp
using UnityEngine;
using UnityEngine.Profiling;
using Unity.Profiling;

public class MemoryProfilingExample : MonoBehaviour
{
    // =============================================
    // ProfilerCounter로 커스텀 메모리 메트릭 추적
    // =============================================

    private static readonly ProfilerCounter<int> s_ActiveTaskCount =
        new ProfilerCounter<int>(
            ProfilerCategory.Scripts,
            "Active Async Tasks",
            ProfilerMarkerDataUnit.Count);

    private static readonly ProfilerCounter<long> s_AsyncAllocBytes =
        new ProfilerCounter<long>(
            ProfilerCategory.Memory,
            "Async Allocation Bytes",
            ProfilerMarkerDataUnit.Bytes);

    private int activeTaskCount = 0;

    private void Update()
    {
        // 매 프레임 Profiler에 커스텀 카운터 보고
        s_ActiveTaskCount.Sample(activeTaskCount);

        // 관리 힙 메모리 체크
        long totalMemory = System.GC.GetTotalMemory(false);
        s_AsyncAllocBytes.Sample(totalMemory);
    }

    // =============================================
    // 비동기 할당 추적
    // =============================================

    public void TrackAsyncAllocation()
    {
        // GC 할당 전후 비교
        long before = Profiler.GetTotalAllocatedMemoryLong();

        // 비동기 작업 시작
        Interlocked.Increment(ref activeTaskCount);

        long after = Profiler.GetTotalAllocatedMemoryLong();
        long allocated = after - before;

        if (allocated > 1024) // 1KB 이상 할당 시 경고
        {
            Debug.LogWarning($"비동기 작업에서 큰 할당 감지: {allocated} bytes");
        }
    }

    private System.Threading.Interlocked Interlocked;
}
```

---

## 2. 비동기 작업 프로파일링

### ProfilerMarker 심화 활용

```csharp
using UnityEngine;
using UnityEngine.Profiling;
using Unity.Profiling;
using System;
using System.Threading;
using System.Threading.Tasks;

public class AsyncProfilingAdvancedExample : MonoBehaviour
{
    // =============================================
    // 메타데이터가 포함된 ProfilerMarker
    // =============================================

    private static readonly ProfilerMarker s_LoadAsset =
        new ProfilerMarker("AsyncLoad.Asset");

    private static readonly ProfilerMarker s_NetworkRequest =
        new ProfilerMarker("AsyncLoad.Network");

    // =============================================
    // 비동기 작업 래퍼로 자동 프로파일링
    // =============================================

    /// <summary>
    /// 비동기 작업을 ProfilerMarker로 감싸는 유틸리티
    /// </summary>
    public static async Task<T> ProfileAsync<T>(
        ProfilerMarker marker,
        Func<Task<T>> taskFactory)
    {
        marker.Begin();
        try
        {
            return await taskFactory();
        }
        finally
        {
            marker.End();
        }
    }

    public static async Task ProfileAsync(
        ProfilerMarker marker,
        Func<Task> taskFactory)
    {
        marker.Begin();
        try
        {
            await taskFactory();
        }
        finally
        {
            marker.End();
        }
    }

    // =============================================
    // 사용 예시
    // =============================================

    private async void Start()
    {
        // ProfilerMarker로 감싼 비동기 작업
        string data = await ProfileAsync(s_LoadAsset, async () =>
        {
            await Task.Delay(100);
            return "에셋 데이터";
        });

        await ProfileAsync(s_NetworkRequest, async () =>
        {
            await Task.Delay(200);
            Debug.Log("네트워크 요청 완료");
        });
    }

    // =============================================
    // CustomSampler를 이용한 세밀한 측정
    // =============================================

    private CustomSampler asyncSampler;
    private Recorder asyncRecorder;

    private void Awake()
    {
        // CustomSampler 생성
        asyncSampler = CustomSampler.Create("MyAsyncOperation");
        asyncRecorder = asyncSampler.GetRecorder();
        asyncRecorder.enabled = true;
    }

    private void MeasureWithCustomSampler()
    {
        asyncSampler.Begin();

        // 측정할 작업 수행
        PerformWork();

        asyncSampler.End();

        // Recorder에서 시간 측정 결과 읽기
        if (asyncRecorder.isValid)
        {
            long elapsedNanoseconds = asyncRecorder.elapsedNanoseconds;
            int sampleCount = asyncRecorder.sampleBlockCount;

            Debug.Log($"소요 시간: {elapsedNanoseconds / 1_000_000.0:F2}ms, " +
                      $"샘플 수: {sampleCount}");
        }
    }

    private void PerformWork()
    {
        // 작업 시뮬레이션
        Thread.SpinWait(10000);
    }
}
```

### 비동기 작업 타이밍 수집기

```csharp
using UnityEngine;
using System;
using System.Collections.Concurrent;
using System.Diagnostics;
using System.Threading.Tasks;
using Debug = UnityEngine.Debug;

/// <summary>
/// 비동기 작업의 실행 시간을 수집하고 통계를 제공하는 유틸리티
/// </summary>
public class AsyncTimingCollector : MonoBehaviour
{
    private static AsyncTimingCollector instance;
    public static AsyncTimingCollector Instance => instance;

    // 스레드 안전한 컬렉션 사용
    private readonly ConcurrentDictionary<string, ConcurrentBag<double>> timings
        = new ConcurrentDictionary<string, ConcurrentBag<double>>();

    private void Awake()
    {
        instance = this;
    }

    // =============================================
    // 비동기 작업 시간 측정
    // =============================================

    public async Task<T> MeasureAsync<T>(string operationName, Func<Task<T>> operation)
    {
        var sw = Stopwatch.StartNew();

        try
        {
            return await operation();
        }
        finally
        {
            sw.Stop();
            RecordTiming(operationName, sw.Elapsed.TotalMilliseconds);
        }
    }

    public async Task MeasureAsync(string operationName, Func<Task> operation)
    {
        var sw = Stopwatch.StartNew();

        try
        {
            await operation();
        }
        finally
        {
            sw.Stop();
            RecordTiming(operationName, sw.Elapsed.TotalMilliseconds);
        }
    }

    private void RecordTiming(string name, double ms)
    {
        var bag = timings.GetOrAdd(name, _ => new ConcurrentBag<double>());
        bag.Add(ms);
    }

    // =============================================
    // 통계 출력
    // =============================================

    public void PrintStatistics()
    {
        Debug.Log("=== 비동기 작업 타이밍 통계 ===");

        foreach (var kvp in timings)
        {
            var values = kvp.Value.ToArray();
            if (values.Length == 0) continue;

            Array.Sort(values);

            double sum = 0;
            foreach (double v in values) sum += v;

            double avg = sum / values.Length;
            double min = values[0];
            double max = values[values.Length - 1];
            double median = values[values.Length / 2];

            // P95 (95번째 백분위수)
            int p95Index = (int)(values.Length * 0.95);
            double p95 = values[Math.Min(p95Index, values.Length - 1)];

            Debug.Log($"[{kvp.Key}] " +
                      $"횟수: {values.Length}, " +
                      $"평균: {avg:F2}ms, " +
                      $"중간값: {median:F2}ms, " +
                      $"최소: {min:F2}ms, " +
                      $"최대: {max:F2}ms, " +
                      $"P95: {p95:F2}ms");
        }
    }

    private void OnDestroy()
    {
        PrintStatistics();
    }
}
```

---

## 3. async 스택 트레이스 디버깅

### 문제점: 스택 트레이스 손실

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;

public class AsyncStackTraceIssueExample : MonoBehaviour
{
    // =============================================
    // 문제: 비동기 호출 시 스택 트레이스가 단절됨
    // =============================================

    private async void Start()
    {
        try
        {
            await MethodA();
        }
        catch (Exception ex)
        {
            // ❌ 스택 트레이스에 MethodA -> MethodB -> MethodC 전체 경로가 보이지 않을 수 있음
            // 특히 ConfigureAwait(false) 사용 시 더 심함
            Debug.LogError($"스택 트레이스:\n{ex.StackTrace}");

            // 출력 예시:
            // at MethodC() in Script.cs:line 42
            // at MethodB() in Script.cs:line 35
            // --- End of stack trace from previous location ---
            // (MethodA 정보가 누락될 수 있음)
        }
    }

    private async Task MethodA()
    {
        await MethodB();
    }

    private async Task MethodB()
    {
        await MethodC();
    }

    private async Task MethodC()
    {
        await Task.Delay(10);
        throw new InvalidOperationException("비동기 작업에서 에러 발생!");
    }
}
```

### 해결 방법: AsyncStackTrace 유틸리티

```csharp
using UnityEngine;
using System;
using System.Collections.Concurrent;
using System.Diagnostics;
using System.Runtime.CompilerServices;
using System.Threading.Tasks;
using Debug = UnityEngine.Debug;

/// <summary>
/// 비동기 호출 체인의 스택 트레이스를 보존하는 유틸리티
/// </summary>
public static class AsyncDiagnostics
{
    // 개발 빌드에서만 활성화
    public static bool IsEnabled { get; set; }
#if UNITY_EDITOR || DEVELOPMENT_BUILD
        = true;
#else
        = false;
#endif

    // 비동기 작업 ID별 호출 체인 저장
    private static readonly ConcurrentDictionary<int, string> callChains
        = new ConcurrentDictionary<int, string>();

    // =============================================
    // 비동기 호출 추적
    // =============================================

    /// <summary>
    /// 비동기 메서드의 호출 지점을 기록합니다.
    /// </summary>
    public static Task<T> TraceAsync<T>(
        Func<Task<T>> operation,
        [CallerMemberName] string caller = "",
        [CallerFilePath] string filePath = "",
        [CallerLineNumber] int lineNumber = 0)
    {
        if (!IsEnabled)
            return operation();

        string callSite = $"  at {caller} in {filePath}:line {lineNumber}";
        int taskId = Task.CurrentId ?? Environment.CurrentManagedThreadId;

        callChains.AddOrUpdate(taskId,
            callSite,
            (_, existing) => existing + "\n" + callSite);

        return ExecuteWithDiagnostics(operation, taskId);
    }

    private static async Task<T> ExecuteWithDiagnostics<T>(
        Func<Task<T>> operation, int taskId)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex)
        {
            if (callChains.TryRemove(taskId, out string chain))
            {
                // 원래 예외에 비동기 호출 체인 정보 추가
                throw new AsyncDiagnosticException(
                    $"비동기 호출 체인:\n{chain}\n\n원본 예외: {ex.Message}",
                    ex);
            }
            throw;
        }
    }

    /// <summary>
    /// 현재 비동기 컨텍스트의 호출 체인을 반환합니다.
    /// </summary>
    public static string GetCurrentCallChain()
    {
        int taskId = Task.CurrentId ?? Environment.CurrentManagedThreadId;
        return callChains.TryGetValue(taskId, out string chain)
            ? chain
            : "(호출 체인 정보 없음)";
    }

    /// <summary>
    /// 추적 정보 정리
    /// </summary>
    public static void Clear()
    {
        callChains.Clear();
    }
}

public class AsyncDiagnosticException : Exception
{
    public AsyncDiagnosticException(string message, Exception inner)
        : base(message, inner) { }
}

// =============================================
// 사용 예시
// =============================================

public class AsyncStackTraceFixExample : MonoBehaviour
{
    private async void Start()
    {
        try
        {
            // ✅ TraceAsync로 감싸서 호출 체인 보존
            await AsyncDiagnostics.TraceAsync(async () =>
            {
                return await LoadPlayerDataAsync("player123");
            });
        }
        catch (AsyncDiagnosticException ex)
        {
            // 전체 비동기 호출 체인이 포함된 에러 메시지
            Debug.LogError(ex.Message);
        }
    }

    private async Task<string> LoadPlayerDataAsync(string playerId)
    {
        await Task.Delay(10);
        throw new InvalidOperationException($"플레이어 '{playerId}'를 찾을 수 없습니다.");
    }
}
```

---

## 4. 데드락 진단

### 데드락 원인과 탐지

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;
using Debug = UnityEngine.Debug;

public class DeadlockDiagnosticsExample : MonoBehaviour
{
    // =============================================
    // ❌ 가장 흔한 데드락 패턴: .Result / .Wait() 사용
    // =============================================

    private void DeadlockExample_Bad()
    {
        // ❌ 메인 스레드에서 .Result 호출 → 데드락!
        // async 메서드가 메인 스레드로 돌아오려 하지만
        // .Result가 메인 스레드를 블로킹하고 있음
        // string result = LoadDataAsync().Result; // 절대 완료되지 않음!
    }

    private async Task<string> LoadDataAsync()
    {
        await Task.Delay(100); // 메인 스레드로 복귀를 시도
        return "데이터";
    }

    // =============================================
    // ✅ 데드락 방지 방법
    // =============================================

    // 방법 1: async/await 사용 (권장)
    private async void CorrectApproach_Async()
    {
        // ✅ await 사용
        string result = await LoadDataAsync();
        Debug.Log(result);
    }

    // 방법 2: ConfigureAwait(false) 사용
    private async Task<string> LoadDataSafeAsync()
    {
        // ✅ ConfigureAwait(false)로 메인 스레드 복귀를 요구하지 않음
        await Task.Delay(100).ConfigureAwait(false);
        return "데이터";
    }

    // 방법 3: Task.Run으로 감싸기 (최후의 수단)
    private void CorrectApproach_TaskRun()
    {
        // ✅ Task.Run으로 다른 스레드에서 실행
        string result = Task.Run(async () =>
        {
            return await LoadDataAsync().ConfigureAwait(false);
        }).Result;

        Debug.Log(result);
    }
}

/// <summary>
/// 데드락 감지를 위한 타임아웃 래퍼
/// </summary>
public static class DeadlockDetector
{
    // =============================================
    // 타임아웃 기반 데드락 감지
    // =============================================

    /// <summary>
    /// 비동기 작업에 타임아웃을 적용하여 데드락 가능성을 감지합니다.
    /// </summary>
    public static async Task<T> WithDeadlockDetection<T>(
        this Task<T> task,
        int timeoutMs = 5000,
        [System.Runtime.CompilerServices.CallerMemberName] string caller = "")
    {
        using var cts = new CancellationTokenSource();
        var completedTask = await Task.WhenAny(task, Task.Delay(timeoutMs, cts.Token));

        if (completedTask == task)
        {
            cts.Cancel(); // 타임아웃 태스크 취소
            return await task; // 결과 반환 (예외가 있으면 전파)
        }

        // 타임아웃 발생 → 데드락 가능성
        string threadInfo = $"Thread: {Thread.CurrentThread.ManagedThreadId}, " +
                           $"IsThreadPoolThread: {Thread.CurrentThread.IsThreadPoolThread}, " +
                           $"IsBackground: {Thread.CurrentThread.IsBackground}";

        Debug.LogError(
            $"[데드락 감지] '{caller}'에서 {timeoutMs}ms 타임아웃!\n" +
            $"{threadInfo}\n" +
            $"Task 상태: {task.Status}\n" +
            $"호출 스택:\n{new StackTrace()}");

        throw new TimeoutException(
            $"잠재적 데드락 감지: '{caller}'이(가) {timeoutMs}ms 내에 완료되지 않았습니다.");
    }

    // =============================================
    // 사용 예시
    // =============================================

    public static async Task UsageExample()
    {
        try
        {
            // 5초 내에 완료되지 않으면 데드락 경고
            var result = await SomeAsyncOperation()
                .WithDeadlockDetection(timeoutMs: 5000);
        }
        catch (TimeoutException ex)
        {
            Debug.LogError($"데드락 가능성: {ex.Message}");
        }
    }

    private static async Task<string> SomeAsyncOperation()
    {
        await Task.Delay(100);
        return "완료";
    }
}
```

### 락 순서 검증기

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;
using System.Threading;

/// <summary>
/// 락 획득 순서를 추적하여 순환 대기(데드락 조건)를 감지합니다.
/// 개발/디버그 빌드에서만 사용하세요.
/// </summary>
public static class LockOrderValidator
{
    [ThreadStatic] private static List<string> heldLocks;
    private static readonly Dictionary<string, HashSet<string>> lockOrderGraph
        = new Dictionary<string, HashSet<string>>();
    private static readonly object graphLock = new object();

    /// <summary>
    /// 락 획득을 기록하고 순서 위반을 감지합니다.
    /// </summary>
    public static IDisposable AcquireLock(object lockObj, string lockName)
    {
        if (heldLocks == null)
            heldLocks = new List<string>();

        // 현재 스레드가 이미 보유한 락이 있으면 순서 기록
        foreach (string held in heldLocks)
        {
            RecordOrder(held, lockName);
        }

        Monitor.Enter(lockObj);
        heldLocks.Add(lockName);

        return new LockReleaser(lockObj, lockName);
    }

    private static void RecordOrder(string first, string second)
    {
        lock (graphLock)
        {
            if (!lockOrderGraph.TryGetValue(first, out var afterSet))
            {
                afterSet = new HashSet<string>();
                lockOrderGraph[first] = afterSet;
            }
            afterSet.Add(second);

            // 역방향 간선 확인 → 순환 = 데드락 위험
            if (lockOrderGraph.TryGetValue(second, out var reverseSet)
                && reverseSet.Contains(first))
            {
                Debug.LogError(
                    $"[데드락 위험] 락 순서 위반 감지!\n" +
                    $"스레드 {Thread.CurrentThread.ManagedThreadId}에서\n" +
                    $"'{first}' → '{second}' 순서로 획득하지만,\n" +
                    $"다른 경로에서 '{second}' → '{first}' 순서가 발견됨.");
            }
        }
    }

    private class LockReleaser : IDisposable
    {
        private readonly object lockObj;
        private readonly string lockName;
        private bool disposed;

        public LockReleaser(object lockObj, string lockName)
        {
            this.lockObj = lockObj;
            this.lockName = lockName;
        }

        public void Dispose()
        {
            if (disposed) return;
            disposed = true;
            heldLocks?.Remove(lockName);
            Monitor.Exit(lockObj);
        }
    }
}

// 사용 예시
public class LockOrderExample : MonoBehaviour
{
    private readonly object lockA = new object();
    private readonly object lockB = new object();

    private void Start()
    {
        // ✅ 일관된 순서로 락 획득
        using (LockOrderValidator.AcquireLock(lockA, "LockA"))
        using (LockOrderValidator.AcquireLock(lockB, "LockB"))
        {
            // 안전한 작업
        }

        // ❌ 역순으로 락 획득 시 경고 발생
        // using (LockOrderValidator.AcquireLock(lockB, "LockB"))
        // using (LockOrderValidator.AcquireLock(lockA, "LockA"))
        // {
        //     // 데드락 위험!
        // }
    }
}
```

---

## 5. Race Condition 디버깅

```csharp
using UnityEngine;
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;
using Debug = UnityEngine.Debug;

/// <summary>
/// Race Condition을 감지하기 위한 공유 변수 접근 추적기
/// </summary>
public class RaceConditionDetector<T>
{
    private T value;
    private readonly string name;
    private int lastWriteThread = -1;
    private string lastWriteStack = "";
    private int concurrentAccessCount = 0;

    public RaceConditionDetector(string name, T initialValue = default)
    {
        this.name = name;
        this.value = initialValue;
    }

    // =============================================
    // 스레드 안전하지 않은 접근 감지
    // =============================================

    public T Read()
    {
        int current = Interlocked.Increment(ref concurrentAccessCount);

        if (current > 1)
        {
            Debug.LogWarning(
                $"[Race Condition 가능성] '{name}'에 동시 접근 감지!\n" +
                $"현재 스레드: {Thread.CurrentThread.ManagedThreadId}\n" +
                $"동시 접근 수: {current}\n" +
                $"마지막 쓰기 스레드: {lastWriteThread}\n" +
                $"마지막 쓰기 위치:\n{lastWriteStack}");
        }

        T result = value;
        Interlocked.Decrement(ref concurrentAccessCount);
        return result;
    }

    public void Write(T newValue)
    {
        int current = Interlocked.Increment(ref concurrentAccessCount);
        int threadId = Thread.CurrentThread.ManagedThreadId;

        if (current > 1)
        {
            Debug.LogError(
                $"[Race Condition!] '{name}'에 동시 쓰기 감지!\n" +
                $"현재 스레드: {threadId}\n" +
                $"동시 접근 수: {current}\n" +
                $"마지막 쓰기 스레드: {lastWriteThread}");
        }

        value = newValue;
        lastWriteThread = threadId;
        lastWriteStack = new StackTrace().ToString();

        Interlocked.Decrement(ref concurrentAccessCount);
    }
}

// =============================================
// Race Condition 감지 사용 예시
// =============================================

public class RaceConditionExample : MonoBehaviour
{
    // ❌ 보호되지 않은 공유 변수
    // private int playerScore = 0;

    // ✅ Race Condition 감지가 가능한 래퍼 사용 (디버그 빌드)
    private RaceConditionDetector<int> playerScore
        = new RaceConditionDetector<int>("PlayerScore", 0);

    private async void Start()
    {
        // 여러 비동기 작업에서 동시에 접근
        var tasks = new List<Task>();

        for (int i = 0; i < 10; i++)
        {
            int index = i;
            tasks.Add(Task.Run(() =>
            {
                // 동시 접근 시 경고 발생
                int current = playerScore.Read();
                playerScore.Write(current + index);
            }));
        }

        await Task.WhenAll(tasks);
    }
}
```

---

## 6. Thread Safety 검증

```csharp
using UnityEngine;
using System;
using System.Threading;

/// <summary>
/// 메인 스레드 전용 접근을 강제하는 어트리뷰트와 검증기
/// </summary>
public static class ThreadSafetyValidator
{
    private static int mainThreadId;

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void Initialize()
    {
        mainThreadId = Thread.CurrentThread.ManagedThreadId;
    }

    // =============================================
    // 메인 스레드 검증
    // =============================================

    /// <summary>
    /// 현재 코드가 메인 스레드에서 실행 중인지 확인합니다.
    /// </summary>
    public static void AssertMainThread(
        [System.Runtime.CompilerServices.CallerMemberName] string caller = "")
    {
        if (Thread.CurrentThread.ManagedThreadId != mainThreadId)
        {
            string message =
                $"[Thread Safety 위반] '{caller}'이(가) 메인 스레드가 아닌 " +
                $"스레드 {Thread.CurrentThread.ManagedThreadId}에서 호출되었습니다.\n" +
                $"Unity API는 메인 스레드에서만 호출해야 합니다.";

            Debug.LogError(message);
            throw new InvalidOperationException(message);
        }
    }

    /// <summary>
    /// 현재 코드가 백그라운드 스레드에서 실행 중인지 확인합니다.
    /// </summary>
    public static void AssertBackgroundThread(
        [System.Runtime.CompilerServices.CallerMemberName] string caller = "")
    {
        if (Thread.CurrentThread.ManagedThreadId == mainThreadId)
        {
            Debug.LogWarning(
                $"[성능 경고] '{caller}'이(가) 메인 스레드에서 호출되었습니다.\n" +
                $"이 작업은 백그라운드 스레드에서 실행하는 것이 권장됩니다.");
        }
    }

    public static bool IsMainThread =>
        Thread.CurrentThread.ManagedThreadId == mainThreadId;
}

// =============================================
// Thread Safety 검증 사용 예시
// =============================================

public class ThreadSafetyExample : MonoBehaviour
{
    private int health = 100;
    private readonly object healthLock = new object();

    // =============================================
    // ✅ 메인 스레드 전용 메서드
    // =============================================

    public void UpdateUI()
    {
        ThreadSafetyValidator.AssertMainThread();

        // Unity UI 업데이트 (메인 스레드 필수)
        // uiText.text = $"HP: {health}";
    }

    // =============================================
    // ✅ 스레드 안전한 프로퍼티
    // =============================================

    public int Health
    {
        get
        {
            lock (healthLock)
            {
                return health;
            }
        }
        set
        {
            lock (healthLock)
            {
                health = value;
            }
        }
    }

    // =============================================
    // ✅ Interlocked를 이용한 원자적 연산
    // =============================================

    private int score = 0;

    public void AddScore(int amount)
    {
        // 락 없이 스레드 안전한 덧셈
        Interlocked.Add(ref score, amount);
    }

    public int GetScore()
    {
        return Interlocked.CompareExchange(ref score, 0, 0); // 원자적 읽기
    }
}
```

---

## 7. Deep Profiling vs Instrumentation

```csharp
using UnityEngine;
using UnityEngine.Profiling;
using Unity.Profiling;
using System.Diagnostics;
using Debug = UnityEngine.Debug;

/// <summary>
/// Deep Profiling과 수동 Instrumentation 비교 및 활용
/// </summary>
public class ProfilingStrategyExample : MonoBehaviour
{
    /*
    ┌─────────────────────────────────────────────────────────────────┐
    │          Deep Profiling vs Instrumentation 비교                   │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  특성              │  Deep Profiling    │  Instrumentation       │
    │  ─────────────────┼────────────────────┼──────────────────────  │
    │  설정              │  체크박스 하나     │  코드에 마커 삽입       │
    │  오버헤드          │  매우 높음 (10x+)  │  최소 (무시 가능)      │
    │  정밀도            │  모든 메서드       │  마킹된 구간만         │
    │  프로덕션 사용     │  불가              │  가능                  │
    │  비동기 지원       │  제한적            │  커스텀 구현 필요       │
    │  빌드 영향         │  없음              │  코드 수정 필요         │
    │                                                                  │
    │  권장: 개발 초기에는 Deep Profiling으로 병목 파악,                │
    │        이후 ProfilerMarker로 핵심 구간만 계측                     │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
    */

    // =============================================
    // 수동 Instrumentation (권장 방식)
    // =============================================

    private static readonly ProfilerMarker s_UpdateAI =
        new ProfilerMarker(ProfilerCategory.Ai, "Game.UpdateAI");

    private static readonly ProfilerMarker s_UpdatePhysicsCustom =
        new ProfilerMarker(ProfilerCategory.Physics, "Game.CustomPhysics");

    private static readonly ProfilerMarker s_SerializeState =
        new ProfilerMarker(ProfilerCategory.Loading, "Game.SerializeState");

    private void Update()
    {
        // 핵심 시스템별로 ProfilerMarker 사용
        using (s_UpdateAI.Auto())
        {
            UpdateAI();
        }

        using (s_UpdatePhysicsCustom.Auto())
        {
            UpdateCustomPhysics();
        }
    }

    private void UpdateAI()
    {
        // AI 로직
    }

    private void UpdateCustomPhysics()
    {
        // 커스텀 물리 시뮬레이션
    }

    // =============================================
    // 조건부 프로파일링 (릴리스 빌드에서 제거)
    // =============================================

    [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
    private static void BeginProfileSample(string name)
    {
        Profiler.BeginSample(name);
    }

    [Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]
    private static void EndProfileSample()
    {
        Profiler.EndSample();
    }

    private void SomeMethod()
    {
        // 릴리스 빌드에서는 자동으로 제거됨
        BeginProfileSample("SomeMethod.HeavyWork");

        // 무거운 작업
        for (int i = 0; i < 1000; i++)
        {
            _ = Mathf.Sqrt(i);
        }

        EndProfileSample();
    }
}
```

---

## 8. Frame Debugger와 비동기 렌더링

```csharp
using UnityEngine;
using UnityEngine.Profiling;
using UnityEngine.Rendering;
using System.Threading.Tasks;

/// <summary>
/// 비동기 렌더링 작업의 프로파일링과 Frame Debugger 활용법
/// </summary>
public class AsyncRenderingDebugExample : MonoBehaviour
{
    /*
    ┌─────────────────────────────────────────────────────────────────┐
    │                Frame Debugger 활용 팁                             │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  1. Window > Analysis > Frame Debugger 열기                      │
    │  2. Enable 버튼으로 프레임 캡처 시작                              │
    │  3. 드로우 콜 목록에서 비동기 렌더링 이벤트 확인                   │
    │  4. CommandBuffer의 비동기 명령 추적                              │
    │                                                                  │
    │  비동기 렌더링 관련 주요 확인 사항:                                │
    │  - AsyncGPUReadback 완료 시점                                    │
    │  - CommandBuffer 실행 순서                                       │
    │  - 렌더 텍스처의 비동기 복사                                      │
    │  - GPU Instancing 배칭 상태                                      │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
    */

    // =============================================
    // AsyncGPUReadback 프로파일링
    // =============================================

    private static readonly ProfilerMarker s_GpuReadback =
        new ProfilerMarker("Rendering.AsyncGPUReadback");

    [SerializeField] private RenderTexture sourceTexture;

    private void RequestGPUReadback()
    {
        if (sourceTexture == null) return;

        s_GpuReadback.Begin();

        AsyncGPUReadback.Request(sourceTexture, 0, TextureFormat.RGBA32,
            (AsyncGPUReadbackRequest request) =>
            {
                s_GpuReadback.End();

                if (request.hasError)
                {
                    Debug.LogError("GPU Readback 실패");
                    return;
                }

                // 읽어온 데이터 처리
                var data = request.GetData<Color32>();
                Debug.Log($"GPU에서 {data.Length}개 픽셀 읽기 완료");
            });
    }

    // =============================================
    // CommandBuffer와 비동기 렌더링 추적
    // =============================================

    private CommandBuffer commandBuffer;

    private void SetupAsyncRendering()
    {
        commandBuffer = new CommandBuffer();
        commandBuffer.name = "AsyncRenderingDebug";

        // Profiler에서 추적 가능한 이름 지정
        commandBuffer.BeginSample("CustomAsyncRendering");

        // 렌더링 명령 추가
        // commandBuffer.Blit(source, destination, material);

        commandBuffer.EndSample("CustomAsyncRendering");

        // 카메라에 CommandBuffer 연결
        Camera.main?.AddCommandBuffer(CameraEvent.AfterEverything, commandBuffer);
    }

    private void OnDestroy()
    {
        if (commandBuffer != null)
        {
            Camera.main?.RemoveCommandBuffer(CameraEvent.AfterEverything, commandBuffer);
            commandBuffer.Release();
        }
    }
}
```

---

## 9. UniTask Tracker (메모리 누수 감지)

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using System;
using System.Threading;

/// <summary>
/// UniTask Tracker를 활용한 비동기 작업 누수 감지
/// </summary>
public class UniTaskTrackerExample : MonoBehaviour
{
    /*
    ┌─────────────────────────────────────────────────────────────────┐
    │                UniTask Tracker 사용법                             │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  1. Window > UniTask Tracker 열기                                │
    │  2. "Enable Tracking" 체크                                       │
    │  3. "Enable StackTrace" 체크 (성능 영향 있음)                     │
    │                                                                  │
    │  표시 정보:                                                      │
    │  - 현재 실행 중인 모든 UniTask 목록                              │
    │  - 각 작업의 생성 위치 (스택 트레이스)                            │
    │  - 경과 시간                                                     │
    │  - 작업 상태 (Pending, Completed 등)                             │
    │                                                                  │
    │  누수 감지:                                                      │
    │  - 씬 전환 후에도 남아있는 UniTask → 누수 가능성                  │
    │  - 경과 시간이 비정상적으로 긴 작업 → 미완료 작업                 │
    │  - 동일 위치에서 계속 생성되는 작업 → 정리 누락                   │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
    */

    private CancellationTokenSource cts;

    private void Start()
    {
        cts = new CancellationTokenSource();

        // ✅ 올바른 패턴: CancellationToken 연결
        RunWithProperCancellation(cts.Token).Forget();

        // ❌ 나쁜 패턴: 취소 토큰 없이 실행
        // RunWithoutCancellation().Forget();
    }

    // =============================================
    // ✅ 올바른 비동기 작업 관리
    // =============================================

    private async UniTaskVoid RunWithProperCancellation(CancellationToken token)
    {
        try
        {
            while (!token.IsCancellationRequested)
            {
                await UniTask.Delay(1000, cancellationToken: token);
                Debug.Log("주기적 작업 실행");
            }
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 정상적으로 취소되었습니다.");
        }
    }

    // =============================================
    // ❌ 누수가 발생하는 패턴
    // =============================================

    private async UniTaskVoid RunWithoutCancellation()
    {
        // ❌ 취소 토큰이 없어서 오브젝트 파괴 후에도 계속 실행됨
        while (true)
        {
            await UniTask.Delay(1000);
            // 오브젝트가 파괴되어도 이 루프는 계속됨 → 메모리 누수
            Debug.Log("누수 중인 작업...");
        }
    }

    // =============================================
    // ✅ GetCancellationTokenOnDestroy 활용
    // =============================================

    private async UniTaskVoid SafePeriodicTask()
    {
        // ✅ MonoBehaviour 파괴 시 자동 취소
        var token = this.GetCancellationTokenOnDestroy();

        try
        {
            await UniTask.Delay(5000, cancellationToken: token);
            Debug.Log("5초 후 실행");
        }
        catch (OperationCanceledException)
        {
            // 오브젝트 파괴 시 여기로 옴
        }
    }

    // =============================================
    // 프로그래밍 방식으로 UniTask 추적 정보 확인
    // =============================================

    private void CheckTrackerStatus()
    {
        // UniTask Tracker의 추적 활성화/비활성화
        TaskTracker.EnableTracking = true;
        TaskTracker.EnableStackTrace = true;

        // 현재 추적 중인 작업 목록 출력
        string trackingInfo = "";
        TaskTracker.ForEachActiveTask((trackingData) =>
        {
            // 각 활성 UniTask의 정보
            Debug.Log($"활성 작업: {trackingData}");
        });
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }
}
```

---

## 10. Memory Profiler로 비동기 할당 추적

```csharp
using UnityEngine;
using UnityEngine.Profiling;
using System;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// Memory Profiler를 활용한 비동기 코드의 메모리 할당 추적
/// </summary>
public class AsyncMemoryTrackingExample : MonoBehaviour
{
    /*
    ┌─────────────────────────────────────────────────────────────────┐
    │           Memory Profiler 비동기 할당 분석 가이드                  │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  설치: Package Manager > Memory Profiler (Unity 공식)            │
    │                                                                  │
    │  비동기 코드의 주요 메모리 할당 원인:                              │
    │                                                                  │
    │  1. async 상태 머신 (StateMachine)                               │
    │     - 모든 async 메서드는 상태 머신 객체를 힙에 할당              │
    │     - Task<T> 반환 시 추가 할당 발생                             │
    │                                                                  │
    │  2. 클로저 캡처                                                   │
    │     - 람다에서 외부 변수 캡처 시 클로저 객체 할당                 │
    │                                                                  │
    │  3. Task / Task<T> 객체                                          │
    │     - ValueTask, UniTask로 대체하여 할당 감소 가능               │
    │                                                                  │
    │  4. CancellationTokenSource                                      │
    │     - 매 프레임 생성 시 GC 압박                                  │
    │     - 풀링 또는 재사용 권장                                       │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
    */

    // =============================================
    // ❌ GC 할당이 많은 비동기 패턴
    // =============================================

    private async Task<string> HighAllocationAsync()
    {
        // ❌ Task<string> 할당
        // ❌ async 상태 머신 할당
        // ❌ 클로저 할당 (playerName 캡처)
        string playerName = "Player1";

        await Task.Run(() =>
        {
            // ❌ 람다 클로저 할당
            Debug.Log($"처리 중: {playerName}");
        });

        return playerName;
    }

    // =============================================
    // ✅ GC 할당을 최소화한 패턴 (UniTask 사용)
    // =============================================

    /*
    // UniTask 사용 시:
    private async UniTask<string> LowAllocationAsync()
    {
        // ✅ UniTask는 구조체 → 힙 할당 없음
        await UniTask.Delay(100);
        return "완료";
    }

    private async UniTaskVoid FireAndForgetSafe()
    {
        // ✅ UniTaskVoid: fire-and-forget에 최적화
        await UniTask.Yield();
        Debug.Log("실행됨");
    }
    */

    // =============================================
    // 메모리 할당 측정 유틸리티
    // =============================================

    public static async Task<(T result, long allocatedBytes)> MeasureAllocation<T>(
        Func<Task<T>> operation)
    {
        // GC 할당 전 상태 기록
        long before = GC.GetTotalMemory(false);

        T result = await operation();

        // GC 할당 후 상태 기록
        long after = GC.GetTotalMemory(false);
        long allocated = Math.Max(0, after - before);

        return (result, allocated);
    }

    private async void Start()
    {
        // 메모리 할당 비교 테스트
        var (result1, bytes1) = await MeasureAllocation(async () =>
        {
            await Task.Delay(10);
            return "Task 방식";
        });
        Debug.Log($"Task 방식 할당: {bytes1} bytes");

        // Profiler에서 GC.Alloc 확인 가능
        Profiler.BeginSample("AsyncAllocationTest");

        await Task.Delay(1);

        Profiler.EndSample();
    }
}
```

---

## 11. Rider / Visual Studio 비동기 디버깅

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// IDE의 비동기 디버깅 기능 활용 가이드
/// </summary>
public class IdeAsyncDebuggingExample : MonoBehaviour
{
    /*
    ┌─────────────────────────────────────────────────────────────────┐
    │            JetBrains Rider 비동기 디버깅 기능                     │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  1. Async Call Stack (비동기 호출 스택)                           │
    │     - Debug > Windows > Call Stack에서 확인                      │
    │     - "[Resumption]" 마커로 await 복귀 지점 표시                 │
    │     - "Show External Code" 비활성화로 사용자 코드만 표시          │
    │                                                                  │
    │  2. Tasks 창                                                     │
    │     - Debug > Windows > Tasks                                    │
    │     - 활성 Task 목록, 상태, 대기 중인 스레드 표시                │
    │     - 데드락 시 어떤 Task가 어디서 블로킹 중인지 확인            │
    │                                                                  │
    │  3. Threads 창                                                    │
    │     - 모든 활성 스레드와 현재 실행 위치 표시                     │
    │     - "Freeze" / "Thaw"로 특정 스레드 일시 중지 가능             │
    │                                                                  │
    │  4. 조건부 브레이크포인트                                        │
    │     - 특정 스레드에서만 중단: Thread.CurrentThread.ManagedThreadId == N │
    │     - 특정 조건에서만 중단: 변수 값 조건                         │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────────┐
    │           Visual Studio 비동기 디버깅 기능                       │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  1. Parallel Stacks 창                                           │
    │     - Debug > Windows > Parallel Stacks                          │
    │     - Tasks 뷰: 비동기 작업 간 관계를 시각적으로 표시            │
    │     - Threads 뷰: 스레드별 호출 스택 시각화                      │
    │                                                                  │
    │  2. Parallel Watch 창                                            │
    │     - 모든 스레드/태스크에서 동시에 변수 값 모니터링             │
    │                                                                  │
    │  3. GPU Usage (Visual Studio 전용)                               │
    │     - 비동기 GPU 작업의 타이밍 분석                              │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
    */

    // =============================================
    // 디버깅을 위한 코드 패턴
    // =============================================

    private async Task DebugFriendlyAsync()
    {
        // ✅ 디버거에서 변수 검사가 용이한 패턴
        // await 결과를 변수에 저장 → Watch 창에서 확인 가능
        Task<string> loadTask = LoadDataAsync();
        string result = await loadTask; // 여기에 브레이크포인트 설정

        Task<int> processTask = ProcessDataAsync(result);
        int processed = await processTask; // 여기에 브레이크포인트 설정

        Debug.Log($"처리 완료: {processed}");
    }

    // ❌ 디버깅하기 어려운 패턴
    private async Task HardToDebugAsync()
    {
        // ❌ 체이닝된 await → 중간 상태 확인 불가
        Debug.Log(await ProcessDataAsync(await LoadDataAsync()));
    }

    private async Task<string> LoadDataAsync()
    {
        await Task.Delay(100);
        return "raw_data";
    }

    private async Task<int> ProcessDataAsync(string data)
    {
        await Task.Delay(50);
        return data.Length;
    }

    // =============================================
    // 조건부 브레이크포인트를 위한 헬퍼
    // =============================================

    /// <summary>
    /// 디버깅 시 특정 조건에서 중단하기 위한 헬퍼 메서드.
    /// 브레이크포인트를 설정하고 조건을 지정하세요.
    /// </summary>
    [System.Diagnostics.Conditional("UNITY_EDITOR")]
    private static void DebugBreakIf(bool condition, string message = "")
    {
        if (condition)
        {
            Debug.LogWarning($"[DebugBreak] {message}");
            // 여기에 브레이크포인트를 설정하세요
            System.Diagnostics.Debugger.Break();
        }
    }

    private void Update()
    {
        // 특정 스레드에서만 중단
        DebugBreakIf(
            Thread.CurrentThread.ManagedThreadId != 1,
            "백그라운드 스레드에서 Update 호출됨!");
    }
}
```

---

## 12. 로깅 전략

### 구조화된 로깅

```csharp
using UnityEngine;
using System;
using System.Collections.Concurrent;
using System.Runtime.CompilerServices;
using System.Text;
using System.Threading;

/// <summary>
/// 비동기 컨텍스트를 포함한 구조화된 로깅 시스템
/// </summary>
public static class AsyncLogger
{
    // 로그 레벨
    public enum LogLevel
    {
        Debug,
        Info,
        Warning,
        Error
    }

    // 현재 비동기 컨텍스트 이름 (AsyncLocal로 스레드/Task 간 전파)
    private static readonly AsyncLocal<string> currentContext
        = new AsyncLocal<string>();

    // 로그 큐 (백그라운드 스레드에서 안전하게 로그 추가)
    private static readonly ConcurrentQueue<LogEntry> logQueue
        = new ConcurrentQueue<LogEntry>();

    // =============================================
    // 컨텍스트 설정
    // =============================================

    /// <summary>
    /// 비동기 작업에 컨텍스트 이름을 설정합니다.
    /// using 패턴으로 사용하면 자동으로 복원됩니다.
    /// </summary>
    public static IDisposable BeginContext(string contextName)
    {
        string previous = currentContext.Value;
        currentContext.Value = contextName;
        return new ContextScope(previous);
    }

    // =============================================
    // 로그 메서드
    // =============================================

    public static void LogDebug(
        string message,
        [CallerMemberName] string caller = "",
        [CallerFilePath] string file = "",
        [CallerLineNumber] int line = 0)
    {
        Log(LogLevel.Debug, message, caller, file, line);
    }

    public static void LogInfo(
        string message,
        [CallerMemberName] string caller = "",
        [CallerFilePath] string file = "",
        [CallerLineNumber] int line = 0)
    {
        Log(LogLevel.Info, message, caller, file, line);
    }

    public static void LogWarn(
        string message,
        [CallerMemberName] string caller = "",
        [CallerFilePath] string file = "",
        [CallerLineNumber] int line = 0)
    {
        Log(LogLevel.Warning, message, caller, file, line);
    }

    public static void LogErr(
        string message,
        [CallerMemberName] string caller = "",
        [CallerFilePath] string file = "",
        [CallerLineNumber] int line = 0)
    {
        Log(LogLevel.Error, message, caller, file, line);
    }

    private static void Log(
        LogLevel level,
        string message,
        string caller,
        string file,
        int line)
    {
        var entry = new LogEntry
        {
            Timestamp = DateTime.UtcNow,
            Level = level,
            Message = message,
            Context = currentContext.Value ?? "Global",
            ThreadId = Thread.CurrentThread.ManagedThreadId,
            IsMainThread = ThreadSafetyValidator.IsMainThread,
            Caller = caller,
            File = System.IO.Path.GetFileName(file),
            Line = line
        };

        logQueue.Enqueue(entry);

        // Unity 콘솔에도 출력
        string formatted = FormatLogEntry(entry);

        switch (level)
        {
            case LogLevel.Debug:
            case LogLevel.Info:
                UnityEngine.Debug.Log(formatted);
                break;
            case LogLevel.Warning:
                UnityEngine.Debug.LogWarning(formatted);
                break;
            case LogLevel.Error:
                UnityEngine.Debug.LogError(formatted);
                break;
        }
    }

    private static string FormatLogEntry(LogEntry entry)
    {
        var sb = new StringBuilder();

        sb.Append($"[{entry.Timestamp:HH:mm:ss.fff}]");
        sb.Append($"[{entry.Level}]");
        sb.Append($"[{entry.Context}]");
        sb.Append($"[Thread:{entry.ThreadId}");
        sb.Append(entry.IsMainThread ? "(Main)" : "(BG)");
        sb.Append(']');
        sb.Append($" {entry.Message}");
        sb.Append($" ({entry.File}:{entry.Line} {entry.Caller})");

        return sb.ToString();
    }

    // =============================================
    // 로그 엔트리 구조체
    // =============================================

    private struct LogEntry
    {
        public DateTime Timestamp;
        public LogLevel Level;
        public string Message;
        public string Context;
        public int ThreadId;
        public bool IsMainThread;
        public string Caller;
        public string File;
        public int Line;
    }

    private class ContextScope : IDisposable
    {
        private readonly string previousContext;
        private bool disposed;

        public ContextScope(string previous)
        {
            previousContext = previous;
        }

        public void Dispose()
        {
            if (disposed) return;
            disposed = true;
            currentContext.Value = previousContext;
        }
    }
}

// =============================================
// 구조화된 로깅 사용 예시
// =============================================

public class StructuredLoggingExample : MonoBehaviour
{
    private async void Start()
    {
        // 컨텍스트별 로깅
        using (AsyncLogger.BeginContext("GameInit"))
        {
            AsyncLogger.LogInfo("게임 초기화 시작");

            await InitializeSubsystems();

            AsyncLogger.LogInfo("게임 초기화 완료");
        }
    }

    private async System.Threading.Tasks.Task InitializeSubsystems()
    {
        // 서브시스템별 컨텍스트
        using (AsyncLogger.BeginContext("AudioSystem"))
        {
            AsyncLogger.LogDebug("오디오 시스템 초기화 중...");
            await System.Threading.Tasks.Task.Delay(100);
            AsyncLogger.LogInfo("오디오 시스템 준비 완료");
        }

        using (AsyncLogger.BeginContext("NetworkSystem"))
        {
            AsyncLogger.LogDebug("네트워크 시스템 초기화 중...");
            await System.Threading.Tasks.Task.Delay(200);
            AsyncLogger.LogInfo("네트워크 연결 완료");
        }

        // 출력 예시:
        // [14:23:01.123][Info][GameInit][Thread:1(Main)] 게임 초기화 시작 (Script.cs:10 Start)
        // [14:23:01.124][Debug][AudioSystem][Thread:1(Main)] 오디오 시스템 초기화 중... (Script.cs:20 InitSub)
        // [14:23:01.225][Info][AudioSystem][Thread:1(Main)] 오디오 시스템 준비 완료 (Script.cs:22 InitSub)
        // [14:23:01.226][Debug][NetworkSystem][Thread:1(Main)] 네트워크 시스템 초기화 중... (Script.cs:28 InitSub)
        // [14:23:01.427][Info][NetworkSystem][Thread:1(Main)] 네트워크 연결 완료 (Script.cs:30 InitSub)
        // [14:23:01.428][Info][GameInit][Thread:1(Main)] 게임 초기화 완료 (Script.cs:14 Start)
    }
}
```

### 비동기 작업 추적 로거

```csharp
using UnityEngine;
using System;
using System.Collections.Concurrent;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// 비동기 작업의 생명주기를 추적하는 로거
/// </summary>
public class AsyncOperationTracker
{
    private static int nextOperationId = 0;

    private static readonly ConcurrentDictionary<int, OperationInfo> activeOperations
        = new ConcurrentDictionary<int, OperationInfo>();

    // =============================================
    // 작업 추적
    // =============================================

    public static TrackedOperation Begin(string name)
    {
        int id = Interlocked.Increment(ref nextOperationId);

        var info = new OperationInfo
        {
            Id = id,
            Name = name,
            StartTime = DateTime.UtcNow,
            StartThread = Thread.CurrentThread.ManagedThreadId,
            Status = "Running"
        };

        activeOperations.TryAdd(id, info);

        AsyncLogger.LogDebug($"[OP-{id}] 시작: {name} (Thread: {info.StartThread})");

        return new TrackedOperation(id);
    }

    public static void Complete(int operationId)
    {
        if (activeOperations.TryRemove(operationId, out var info))
        {
            var elapsed = DateTime.UtcNow - info.StartTime;
            int endThread = Thread.CurrentThread.ManagedThreadId;

            AsyncLogger.LogDebug(
                $"[OP-{operationId}] 완료: {info.Name} " +
                $"(소요: {elapsed.TotalMilliseconds:F1}ms, " +
                $"스레드: {info.StartThread} → {endThread})");
        }
    }

    public static void Fail(int operationId, Exception ex)
    {
        if (activeOperations.TryRemove(operationId, out var info))
        {
            var elapsed = DateTime.UtcNow - info.StartTime;

            AsyncLogger.LogErr(
                $"[OP-{operationId}] 실패: {info.Name} " +
                $"(소요: {elapsed.TotalMilliseconds:F1}ms, " +
                $"에러: {ex.Message})");
        }
    }

    /// <summary>
    /// 현재 활성 중인 모든 작업을 출력합니다.
    /// </summary>
    public static void DumpActiveOperations()
    {
        Debug.Log($"=== 활성 비동기 작업: {activeOperations.Count}개 ===");

        foreach (var kvp in activeOperations)
        {
            var info = kvp.Value;
            var elapsed = DateTime.UtcNow - info.StartTime;

            Debug.Log(
                $"  [OP-{info.Id}] {info.Name} - " +
                $"실행 중 ({elapsed.TotalSeconds:F1}초 경과, " +
                $"시작 스레드: {info.StartThread})");
        }
    }

    // =============================================
    // 내부 타입
    // =============================================

    private struct OperationInfo
    {
        public int Id;
        public string Name;
        public DateTime StartTime;
        public int StartThread;
        public string Status;
    }

    public struct TrackedOperation : IDisposable
    {
        private readonly int id;
        private bool disposed;

        public TrackedOperation(int id)
        {
            this.id = id;
            this.disposed = false;
        }

        public void Dispose()
        {
            if (disposed) return;
            disposed = true;
            Complete(id);
        }
    }
}

// =============================================
// 사용 예시
// =============================================

public class AsyncTrackingUsageExample : MonoBehaviour
{
    private async void Start()
    {
        // using 패턴으로 자동 완료 추적
        using (AsyncOperationTracker.Begin("데이터 로드"))
        {
            await Task.Delay(500);
        }
        // 출력: [OP-1] 완료: 데이터 로드 (소요: 501.3ms, 스레드: 1 → 1)

        // 수동 추적
        var op = AsyncOperationTracker.Begin("네트워크 요청");
        try
        {
            await Task.Delay(200);
            op.Dispose();
        }
        catch (Exception ex)
        {
            AsyncOperationTracker.Fail(1, ex);
        }

        // 활성 작업 덤프 (디버깅용)
        AsyncOperationTracker.DumpActiveOperations();
    }
}
```

---

## 주의사항

1. **프로파일링 오버헤드**: Deep Profiling은 성능에 큰 영향을 미칩니다. 프로덕션 빌드에서는 반드시 비활성화하세요.

2. **디버그 전용 코드 분리**: `#if UNITY_EDITOR || DEVELOPMENT_BUILD` 또는 `[Conditional]` 어트리뷰트를 사용하여 릴리스 빌드에서 디버그 코드를 제거하세요.

3. **ProfilerMarker vs Profiler.BeginSample**:
   - `ProfilerMarker`는 구조체 기반으로 GC 할당이 없으며, Burst 컴파일러와 호환됩니다.
   - `Profiler.BeginSample`은 문자열 할당이 발생할 수 있으며 레거시 API입니다.
   - 신규 코드에서는 항상 `ProfilerMarker`를 사용하세요.

4. **비동기 스택 트레이스**:
   - `ConfigureAwait(false)` 사용 시 스택 트레이스가 더 많이 단절됩니다.
   - 디버그 빌드에서는 `AsyncDiagnostics`와 같은 유틸리티로 호출 체인을 보존하세요.

5. **스레드 안전성 검증은 개발 중에만**: `ThreadSafetyValidator`, `RaceConditionDetector` 등은 성능 오버헤드가 있으므로 개발 빌드에서만 활성화하세요.

6. **UniTask Tracker**: `EnableStackTrace`는 성능에 상당한 영향을 줍니다. 메모리 누수 의심 시에만 활성화하세요.

7. **로깅 수준 관리**: 프로덕션에서는 Error, Warning만 활성화하고 Debug, Info 레벨은 비활성화하세요.

---

## 베스트 프랙티스

### 프로파일링

```
1. 점진적 접근법 사용
   - 먼저 Unity Profiler의 CPU Usage로 전체적인 병목 파악
   - Timeline 뷰로 스레드 간 작업 분포 확인
   - ProfilerMarker로 의심 구간을 세밀하게 계측
   - Memory Profiler로 비동기 할당 추적

2. ProfilerMarker 네이밍 컨벤션
   ✅ "SystemName.OperationName"  (예: "AI.PathFinding", "Network.SendRequest")
   ❌ "MyMethod"                   (예: 모호한 이름)
   ❌ "DoStuff"                    (예: 의미 없는 이름)

3. 프로덕션 프로파일링
   - ProfilerMarker는 프로덕션 빌드에서도 최소 오버헤드로 사용 가능
   - Development Build 체크 후 Autoconnect Profiler로 실제 기기 프로파일링
   - IL2CPP 빌드에서 프로파일링하여 실제 성능 측정
```

### 디버깅

```
1. 비동기 코드 디버깅 체크리스트
   ✅ 모든 await 결과를 변수에 저장 (중간 상태 검사 가능)
   ✅ CancellationToken을 모든 비동기 메서드에 전달
   ✅ try-catch로 OperationCanceledException 처리
   ✅ ConfigureAwait(false) 사용 시 UI 접근하지 않는지 확인
   ❌ .Result나 .Wait()로 비동기 결과 블로킹
   ❌ async void (이벤트 핸들러 제외)
   ❌ 예외 처리 없는 fire-and-forget

2. 데드락 예방 규칙
   ✅ 메인 스레드에서는 항상 await 사용
   ✅ 라이브러리 코드에서는 ConfigureAwait(false) 사용
   ✅ 락 획득 순서를 일관되게 유지
   ❌ 메인 스레드에서 .Result / .Wait() 호출
   ❌ lock 블록 내에서 await 사용
   ❌ 중첩된 락 (nested locking) 패턴

3. Race Condition 예방
   ✅ 공유 상태에는 lock 또는 Interlocked 사용
   ✅ ConcurrentDictionary 등 스레드 안전한 컬렉션 사용
   ✅ 불변 데이터 구조 선호
   ❌ 보호되지 않은 공유 변수 접근
   ❌ "한 번만 실행"을 bool 플래그로 구현 (Interlocked.CompareExchange 사용)
```

### 로깅

```
1. 비동기 로깅 가이드라인
   ✅ AsyncLocal<T>로 비동기 컨텍스트 전파
   ✅ 스레드 ID와 메인/백그라운드 스레드 구분 포함
   ✅ 타임스탬프에 밀리초 단위 포함
   ✅ 구조화된 형식으로 파싱 가능하게 작성
   ❌ Debug.Log만으로 복잡한 비동기 흐름 추적
   ❌ 로깅에서 Unity API 호출 (메인 스레드가 아닐 수 있음)

2. 성능에 민감한 로깅
   ✅ [Conditional] 어트리뷰트로 릴리스 빌드에서 제거
   ✅ 로그 레벨로 출력량 제어
   ✅ ConcurrentQueue로 스레드 안전한 로그 수집
   ❌ 매 프레임 string 연결 (StringBuilder 사용)
   ❌ 백그라운드 스레드에서 Debug.Log 과다 호출
```

---

## 참고 자료

- [Unity Profiler 공식 문서](https://docs.unity3d.com/Manual/Profiler.html)
- [Unity ProfilerMarker API](https://docs.unity3d.com/ScriptReference/Unity.Profiling.ProfilerMarker.html)
- [Unity Memory Profiler](https://docs.unity3d.com/Packages/com.unity.memoryprofiler@latest)
- [Unity Frame Debugger](https://docs.unity3d.com/Manual/FrameDebugger.html)
- [UniTask Tracker](https://github.com/Cysharp/UniTask#unitask-tracker)
- [Microsoft: Debugging async code](https://learn.microsoft.com/en-us/visualstudio/debugger/debugging-async-code)
- [JetBrains Rider: Async Debugging](https://www.jetbrains.com/help/rider/Debugging_Async_Code.html)

---

## 다음 섹션

[41. 안티패턴 & 함정](./41-anti-patterns.md)
