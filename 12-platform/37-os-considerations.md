# 37. OS별 주의사항

## 개요

Unity는 "한 번 작성, 여러 플랫폼 배포"를 지향하지만, **동시성 프로그래밍**에서는 플랫폼마다 근본적으로 다른 제약과 동작이 존재합니다. WebGL은 멀티스레딩 자체가 불가능하고, iOS는 백그라운드 실행에 엄격한 제한이 있으며, 콘솔 플랫폼은 독자적인 스레드 모델을 갖습니다. 이 섹션에서는 각 플랫폼의 동시성 관련 차이를 체계적으로 분석하고, 크로스 플랫폼 호환 코드를 작성하기 위한 전략을 제시합니다.

---

## 1. 플랫폼별 동시성 차이 개요

### 플랫폼 특성 비교표

```
┌─────────────┬────────────┬────────────┬────────────┬──────────────┐
│   플랫폼     │ 멀티스레딩  │ Thread Pool │ async/await│ Job System   │
├─────────────┼────────────┼────────────┼────────────┼──────────────┤
│ Windows     │     ✅     │     ✅     │     ✅     │     ✅       │
│ macOS       │     ✅     │     ✅     │     ✅     │     ✅       │
│ Linux       │     ✅     │     ✅     │     ✅     │     ✅       │
│ iOS         │     ✅     │   ✅(제한) │     ✅     │     ✅       │
│ Android     │     ✅     │     ✅     │     ✅     │     ✅       │
│ WebGL       │     ❌     │     ❌     │   ⚠️(제한) │     ❌       │
│ PS4/PS5     │     ✅     │   ✅(제한) │     ✅     │     ✅       │
│ Xbox        │     ✅     │   ✅(제한) │     ✅     │     ✅       │
│ Switch      │     ✅     │   ✅(제한) │     ✅     │     ✅       │
└─────────────┴────────────┴────────────┴────────────┴──────────────┘
```

### 핵심 차이 요약

```csharp
using UnityEngine;

/// <summary>
/// 플랫폼별 동시성 지원 수준을 런타임에 확인하는 유틸리티
/// </summary>
public static class PlatformConcurrencyInfo
{
    // =============================================
    // 현재 플랫폼의 동시성 지원 수준 확인
    // =============================================

    public static bool SupportsMultithreading
    {
        get
        {
#if UNITY_WEBGL && !UNITY_EDITOR
            return false;
#else
            return true;
#endif
        }
    }

    public static bool SupportsThreadPool
    {
        get
        {
#if UNITY_WEBGL && !UNITY_EDITOR
            return false;
#else
            return true;
#endif
        }
    }

    public static bool SupportsJobSystem
    {
        get
        {
#if UNITY_WEBGL && !UNITY_EDITOR
            return false;
#else
            return true;
#endif
        }
    }

    public static int RecommendedMaxConcurrency
    {
        get
        {
#if UNITY_WEBGL && !UNITY_EDITOR
            return 1;
#elif UNITY_IOS
            // iOS는 보수적으로 코어 수 - 1 사용
            return Mathf.Max(1, SystemInfo.processorCount - 1);
#elif UNITY_SWITCH
            // Switch는 제한된 코어 사용 권장
            return Mathf.Min(3, SystemInfo.processorCount);
#else
            return SystemInfo.processorCount;
#endif
        }
    }

    public static void LogPlatformInfo()
    {
        Debug.Log($"[PlatformConcurrency] 프로세서 수: {SystemInfo.processorCount}");
        Debug.Log($"[PlatformConcurrency] 멀티스레딩 지원: {SupportsMultithreading}");
        Debug.Log($"[PlatformConcurrency] 스레드 풀 지원: {SupportsThreadPool}");
        Debug.Log($"[PlatformConcurrency] Job System 지원: {SupportsJobSystem}");
        Debug.Log($"[PlatformConcurrency] 권장 동시성 수준: {RecommendedMaxConcurrency}");
    }
}
```

---

## 2. iOS 고려사항

### 2.1 백그라운드 실행 제한

iOS는 앱이 백그라운드로 전환되면 **약 5~10초 후에 모든 실행이 일시 중지**됩니다. 장시간 비동기 작업은 반드시 이를 고려해야 합니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// iOS 백그라운드 전환 시 비동기 작업을 안전하게 관리하는 매니저
/// </summary>
public class iOSBackgroundTaskManager : MonoBehaviour
{
    private CancellationTokenSource _backgroundCts;
    private bool _isInBackground = false;

    // =============================================
    // 백그라운드 전환 감지
    // =============================================

    private void OnApplicationPause(bool pauseStatus)
    {
        if (pauseStatus)
        {
            // 앱이 백그라운드로 진입
            _isInBackground = true;
            OnEnterBackground();
        }
        else
        {
            // 앱이 포그라운드로 복귀
            _isInBackground = false;
            OnEnterForeground();
        }
    }

    private void OnEnterBackground()
    {
        Debug.Log("[iOS] 백그라운드 진입 - 비동기 작업 일시 중지");

        // 진행 중인 비필수 작업 취소
        _backgroundCts?.Cancel();

#if UNITY_IOS && !UNITY_EDITOR
        // iOS 백그라운드 작업 시간 요청 (약 30초 추가 실행 시간)
        UnityEngine.iOS.Device.SetNoBackupFlag("SaveData");
#endif
    }

    private void OnEnterForeground()
    {
        Debug.Log("[iOS] 포그라운드 복귀 - 비동기 작업 재개");

        // 새 CancellationTokenSource 생성
        _backgroundCts?.Dispose();
        _backgroundCts = new CancellationTokenSource();

        // 중단된 작업 재시작
        _ = ResumeNetworkSyncAsync(_backgroundCts.Token);
    }

    // =============================================
    // 백그라운드 상태를 고려한 비동기 작업
    // =============================================

    // ✅ 올바른 패턴: 백그라운드 상태 확인
    public async Task SafeDownloadAsync(string url, CancellationToken token)
    {
        // 백그라운드 상태에서는 시작하지 않음
        if (_isInBackground)
        {
            Debug.LogWarning("[iOS] 백그라운드에서 다운로드 시작 불가");
            return;
        }

        using var linkedCts = CancellationTokenSource
            .CreateLinkedTokenSource(token, _backgroundCts?.Token ?? CancellationToken.None);

        try
        {
            // 작업 수행 (백그라운드 전환 시 자동 취소됨)
            await PerformDownloadAsync(url, linkedCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("[iOS] 백그라운드 전환으로 다운로드 취소됨");
            // 포그라운드 복귀 시 자동 재시도를 위해 상태 저장
            SaveDownloadState(url);
        }
    }

    private async Task PerformDownloadAsync(string url, CancellationToken token)
    {
        // 실제 다운로드 로직
        await Task.Delay(1000, token);
        Debug.Log($"[iOS] 다운로드 완료: {url}");
    }

    private async Task ResumeNetworkSyncAsync(CancellationToken token)
    {
        await Task.Delay(500, token);
        Debug.Log("[iOS] 네트워크 동기화 재개 완료");
    }

    private void SaveDownloadState(string url)
    {
        PlayerPrefs.SetString("PendingDownload", url);
        PlayerPrefs.Save();
    }

    private void OnDestroy()
    {
        _backgroundCts?.Cancel();
        _backgroundCts?.Dispose();
    }
}
```

### 2.2 스레드 제한

iOS에서는 과도한 스레드 생성이 앱 종료로 이어질 수 있습니다.

```csharp
// ❌ 잘못된 패턴: iOS에서 무제한 스레드 생성
public class BadiOSThreading : MonoBehaviour
{
    void Start()
    {
        // iOS에서 수십 개의 스레드를 직접 생성하면 메모리 부족 발생 가능
        for (int i = 0; i < 50; i++)
        {
            new Thread(() =>
            {
                Thread.Sleep(10000);
            }).Start();
        }
    }
}

// ✅ 올바른 패턴: SemaphoreSlim으로 동시 실행 수 제한
public class GoodiOSThreading : MonoBehaviour
{
    // iOS에서는 동시 스레드 수를 보수적으로 설정
    private static readonly SemaphoreSlim _throttle = new SemaphoreSlim(
#if UNITY_IOS && !UNITY_EDITOR
        3  // iOS: 최대 3개 동시 실행
#else
        Environment.ProcessorCount  // 기타 플랫폼: 코어 수만큼
#endif
    );

    public async Task ProcessItemsAsync(string[] items, CancellationToken token)
    {
        var tasks = new List<Task>();
        foreach (var item in items)
        {
            tasks.Add(ProcessWithThrottleAsync(item, token));
        }
        await Task.WhenAll(tasks);
    }

    private async Task ProcessWithThrottleAsync(string item, CancellationToken token)
    {
        await _throttle.WaitAsync(token);
        try
        {
            await Task.Run(() => HeavyComputation(item), token);
        }
        finally
        {
            _throttle.Release();
        }
    }

    private void HeavyComputation(string item)
    {
        // 무거운 연산
    }
}
```

### 2.3 App Store 규정 관련

```csharp
// App Store 심사 시 주의해야 할 동시성 관련 사항:
//
// 1. 백그라운드에서 과도한 CPU 사용 금지
//    - 백그라운드 음악 재생, 위치 추적 등 명시적 이유 없이 백그라운드 실행 불가
//
// 2. 네트워크 작업은 반드시 타임아웃 설정
//    - 무한 대기 상태는 앱 거절 사유
//
// 3. 메인 스레드 차단 금지
//    - UI가 5초 이상 응답하지 않으면 워치독에 의해 강제 종료

// ✅ iOS 준수 패턴
public class iOSCompliantNetworking : MonoBehaviour
{
    private const float iOS_TIMEOUT_SECONDS = 30f;

    public async Task<string> FetchDataAsync(string url, CancellationToken token)
    {
        using var timeoutCts = new CancellationTokenSource(
            TimeSpan.FromSeconds(iOS_TIMEOUT_SECONDS));
        using var linkedCts = CancellationTokenSource
            .CreateLinkedTokenSource(token, timeoutCts.Token);

        try
        {
            var request = UnityEngine.Networking.UnityWebRequest.Get(url);
            request.timeout = (int)iOS_TIMEOUT_SECONDS;

            var operation = request.SendWebRequest();

            // Unity 2023+에서 await 가능
            while (!operation.isDone)
            {
                linkedCts.Token.ThrowIfCancellationRequested();
                await Task.Yield();
            }

            return request.downloadHandler.text;
        }
        catch (OperationCanceledException)
        {
            Debug.LogWarning("[iOS] 네트워크 요청 타임아웃 또는 취소");
            throw;
        }
    }
}
```

---

## 3. Android 고려사항

### 3.1 백그라운드 서비스와 Doze 모드

Android 6.0(API 23) 이상에서 **Doze 모드**는 장시간 미사용 시 네트워크 접근과 백그라운드 작업을 극도로 제한합니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// Android Doze 모드와 배터리 최적화를 고려한 비동기 작업 관리자
/// </summary>
public class AndroidBackgroundManager : MonoBehaviour
{
    private bool _isInDozeMode = false;
    private float _lastActiveTime;
    private CancellationTokenSource _dozeCts;

    // =============================================
    // Doze 모드 감지 및 대응
    // =============================================

    private void OnApplicationFocus(bool hasFocus)
    {
        if (hasFocus)
        {
            _lastActiveTime = Time.realtimeSinceStartup;
            OnExitDoze();
        }
    }

    private void OnApplicationPause(bool pauseStatus)
    {
        if (pauseStatus)
        {
            OnEnterDoze();
        }
        else
        {
            OnExitDoze();
        }
    }

    private void OnEnterDoze()
    {
        _isInDozeMode = true;
        _dozeCts?.Cancel();
        Debug.Log("[Android] Doze 모드 감지 - 비필수 작업 중단");

        // 중요 데이터 즉시 저장
        SaveCriticalData();
    }

    private void OnExitDoze()
    {
        if (_isInDozeMode)
        {
            _isInDozeMode = false;
            _dozeCts?.Dispose();
            _dozeCts = new CancellationTokenSource();
            Debug.Log("[Android] Doze 모드 해제 - 작업 재개");

            // 대기 중이던 네트워크 작업 재시도
            _ = RetryPendingOperationsAsync(_dozeCts.Token);
        }
    }

    // =============================================
    // Doze 모드 대응 네트워크 동기화
    // =============================================

    /// <summary>
    /// Doze 모드를 고려한 주기적 서버 동기화
    /// </summary>
    public async Task PeriodicSyncAsync(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            if (!_isInDozeMode)
            {
                try
                {
                    await SyncWithServerAsync(token);
                    Debug.Log("[Android] 서버 동기화 성공");
                }
                catch (Exception ex) when (!(ex is OperationCanceledException))
                {
                    Debug.LogWarning($"[Android] 동기화 실패 (Doze 모드 가능성): {ex.Message}");
                }
            }

            // Doze 모드에서는 네트워크가 차단되므로 대기 간격 증가
            float delaySeconds = _isInDozeMode ? 300f : 60f;
            await Task.Delay(TimeSpan.FromSeconds(delaySeconds), token);
        }
    }

    private async Task SyncWithServerAsync(CancellationToken token)
    {
        await Task.Delay(500, token); // 동기화 로직 대체
    }

    private async Task RetryPendingOperationsAsync(CancellationToken token)
    {
        await Task.Delay(1000, token);
        Debug.Log("[Android] 보류된 작업 재시도 완료");
    }

    private void SaveCriticalData()
    {
        PlayerPrefs.Save();
    }

    private void OnDestroy()
    {
        _dozeCts?.Cancel();
        _dozeCts?.Dispose();
    }
}
```

### 3.2 배터리 최적화와 스레드 관리

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// Android 배터리 상태에 따라 동시성 수준을 자동 조절하는 시스템
/// </summary>
public class AndroidBatteryAwareScheduler : MonoBehaviour
{
    private int _maxConcurrency;

    private void Start()
    {
        UpdateConcurrencyForBattery();
    }

    // =============================================
    // 배터리 수준에 따른 동시성 조절
    // =============================================

    private void UpdateConcurrencyForBattery()
    {
#if UNITY_ANDROID && !UNITY_EDITOR
        float batteryLevel = SystemInfo.batteryLevel;             // 0.0 ~ 1.0
        BatteryStatus status = SystemInfo.batteryStatus;

        if (status == BatteryStatus.Charging || status == BatteryStatus.Full)
        {
            // 충전 중에는 최대 성능 사용
            _maxConcurrency = SystemInfo.processorCount;
            Debug.Log($"[Android] 충전 중 - 최대 동시성: {_maxConcurrency}");
        }
        else if (batteryLevel > 0.5f)
        {
            // 배터리 50% 이상: 적당한 동시성
            _maxConcurrency = Mathf.Max(2, SystemInfo.processorCount / 2);
        }
        else if (batteryLevel > 0.2f)
        {
            // 배터리 20~50%: 최소한의 동시성
            _maxConcurrency = 2;
        }
        else
        {
            // 배터리 20% 이하: 가능하면 단일 스레드
            _maxConcurrency = 1;
            Debug.LogWarning("[Android] 배터리 부족 - 최소 동시성 모드");
        }
#else
        _maxConcurrency = SystemInfo.processorCount;
#endif
    }

    // ✅ 배터리 상태 기반 작업 스케줄링
    public async Task ProcessBatchAsync<T>(T[] items, Func<T, CancellationToken, Task> processor,
        CancellationToken token)
    {
        UpdateConcurrencyForBattery();
        var semaphore = new SemaphoreSlim(_maxConcurrency);

        var tasks = new Task[items.Length];
        for (int i = 0; i < items.Length; i++)
        {
            var item = items[i];
            tasks[i] = Task.Run(async () =>
            {
                await semaphore.WaitAsync(token);
                try
                {
                    await processor(item, token);
                }
                finally
                {
                    semaphore.Release();
                }
            }, token);
        }

        await Task.WhenAll(tasks);
    }
}
```

---

## 4. WebGL 제약사항

### 4.1 단일 스레드 환경

WebGL은 브라우저의 **메인 스레드**에서만 실행됩니다. `System.Threading.Thread`, `Task.Run`, `ThreadPool`을 사용할 수 없습니다.

```csharp
using UnityEngine;
using System.Collections;
using System.Threading.Tasks;

public class WebGLConcurrencyExample : MonoBehaviour
{
    // =============================================
    // ❌ WebGL에서 사용 불가능한 패턴들
    // =============================================

#if !UNITY_WEBGL
    void ThreadingNotAvailableOnWebGL()
    {
        // 이 코드들은 WebGL에서 컴파일은 되지만 런타임에 예외 발생
        // 또는 아무 동작도 하지 않음

        // Thread 생성 불가
        var thread = new System.Threading.Thread(() =>
        {
            Debug.Log("이 코드는 WebGL에서 실행되지 않음");
        });
        thread.Start(); // WebGL에서 예외 발생

        // Task.Run 불가
        Task.Run(() =>
        {
            Debug.Log("ThreadPool 없음");
        }); // WebGL에서 동기적으로 실행되거나 실패

        // Parallel 불가
        System.Threading.Tasks.Parallel.For(0, 100, i =>
        {
            Debug.Log(i);
        }); // WebGL에서 작동하지 않음
    }
#endif

    // =============================================
    // ✅ WebGL에서 사용 가능한 패턴들
    // =============================================

    // 패턴 1: Coroutine (모든 플랫폼에서 작동)
    private IEnumerator LoadDataCoroutine()
    {
        var request = UnityEngine.Networking.UnityWebRequest.Get("https://api.example.com/data");
        yield return request.SendWebRequest();

        if (request.result == UnityEngine.Networking.UnityWebRequest.Result.Success)
        {
            Debug.Log($"데이터 수신: {request.downloadHandler.text}");
        }
    }

    // 패턴 2: 프레임 분산 처리 (수동 시간 분할)
    private IEnumerator ProcessLargeDataCoroutine(int[] data)
    {
        int processed = 0;
        float frameTimeLimit = 0.008f; // 8ms 이내로 제한 (60fps 유지)

        while (processed < data.Length)
        {
            float frameStart = Time.realtimeSinceStartup;

            while (processed < data.Length &&
                   (Time.realtimeSinceStartup - frameStart) < frameTimeLimit)
            {
                // 데이터 처리 (한 프레임에 시간 제한만큼만)
                data[processed] = data[processed] * 2;
                processed++;
            }

            // 진행률 보고
            float progress = (float)processed / data.Length;
            Debug.Log($"처리 진행률: {progress:P0}");

            // 다음 프레임까지 대기
            yield return null;
        }

        Debug.Log("대량 데이터 처리 완료 (WebGL 호환)");
    }

    private void Start()
    {
        StartCoroutine(LoadDataCoroutine());
    }
}
```

### 4.2 async/await의 WebGL 제한

```csharp
using UnityEngine;
using System.Threading.Tasks;

/// <summary>
/// WebGL에서의 async/await 사용 시 주의사항
/// </summary>
public class WebGLAsyncLimitations : MonoBehaviour
{
    // =============================================
    // ⚠️ WebGL에서 async/await은 제한적으로 동작
    // =============================================

    // ❌ Task.Delay는 WebGL에서 정확하지 않거나 동작하지 않을 수 있음
    async void BadWebGLAsync()
    {
        await Task.Delay(1000);  // WebGL에서는 정확한 타이밍 보장 불가
        Debug.Log("지연 후 실행");
    }

    // ❌ Task.Run은 WebGL에서 작동하지 않음
    async void BadWebGLTaskRun()
    {
        var result = await Task.Run(() =>
        {
            return HeavyComputation();
        });
        // WebGL에서 예외 발생 또는 메인 스레드에서 동기 실행
    }

    // ✅ WebGL 호환 async 패턴: UniTask 사용 권장
    // UniTask의 DelayFrame, Yield 등은 WebGL에서 안전하게 작동
    /*
    async UniTaskVoid GoodWebGLAsync()
    {
        await UniTask.DelayFrame(60);  // 60프레임 대기 (WebGL 안전)
        await UniTask.Yield();          // 다음 프레임까지 양보 (WebGL 안전)
        Debug.Log("WebGL에서 안전하게 실행");
    }
    */

    private int HeavyComputation()
    {
        int sum = 0;
        for (int i = 0; i < 1000000; i++) sum += i;
        return sum;
    }
}
```

### 4.3 WebGL 전용 비동기 전략

```csharp
using UnityEngine;
using System;
using System.Collections;
using System.Collections.Generic;

/// <summary>
/// WebGL에서 무거운 작업을 프레임에 분산 처리하는 배치 프로세서
/// </summary>
public class WebGLBatchProcessor : MonoBehaviour
{
    private readonly Queue<Action> _workQueue = new Queue<Action>();
    private bool _isProcessing = false;

    // =============================================
    // 작업 큐 기반 프레임 분산 처리
    // =============================================

    /// <summary>
    /// 작업을 큐에 추가하고 프레임 분산 처리 시작
    /// </summary>
    public void EnqueueWork(Action work)
    {
        _workQueue.Enqueue(work);

        if (!_isProcessing)
        {
            _isProcessing = true;
            StartCoroutine(ProcessWorkQueue());
        }
    }

    /// <summary>
    /// 복수의 작업 항목을 일괄 등록
    /// </summary>
    public void EnqueueBatch<T>(T[] items, Action<T> processor)
    {
        foreach (var item in items)
        {
            var capturedItem = item;
            _workQueue.Enqueue(() => processor(capturedItem));
        }

        if (!_isProcessing)
        {
            _isProcessing = true;
            StartCoroutine(ProcessWorkQueue());
        }
    }

    private IEnumerator ProcessWorkQueue()
    {
        while (_workQueue.Count > 0)
        {
            float frameStart = Time.realtimeSinceStartup;
            float budgetMs = 0.012f; // 프레임당 12ms 예산

            while (_workQueue.Count > 0 &&
                   (Time.realtimeSinceStartup - frameStart) < budgetMs)
            {
                var work = _workQueue.Dequeue();
                try
                {
                    work.Invoke();
                }
                catch (Exception ex)
                {
                    Debug.LogError($"[WebGL] 작업 처리 오류: {ex.Message}");
                }
            }

            // 남은 작업이 있으면 다음 프레임에 계속
            if (_workQueue.Count > 0)
            {
                yield return null;
            }
        }

        _isProcessing = false;
    }
}
```

---

## 5. Windows / macOS / Linux 데스크톱 차이점

### 5.1 데스크톱 플랫폼 공통 특성

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Runtime.InteropServices;

/// <summary>
/// 데스크톱 플랫폼별 차이를 고려한 동시성 유틸리티
/// </summary>
public static class DesktopConcurrencyUtils
{
    // =============================================
    // 플랫폼별 스레드 풀 설정
    // =============================================

    /// <summary>
    /// 데스크톱 플랫폼에 맞게 스레드 풀을 최적화
    /// </summary>
    public static void OptimizeThreadPool()
    {
        int processorCount = Environment.ProcessorCount;

#if UNITY_STANDALONE_WIN
        // Windows: IOCP (I/O Completion Port) 기반
        // I/O 바운드 작업에 강점이 있어 I/O 스레드를 넉넉히 설정
        ThreadPool.SetMinThreads(processorCount * 2, processorCount * 4);
        Debug.Log("[Windows] IOCP 기반 스레드 풀 최적화 적용");

#elif UNITY_STANDALONE_OSX
        // macOS: Grand Central Dispatch(GCD)와 유사한 스레드 관리
        // Mono/IL2CPP 환경에서는 기본 스레드 풀 사용
        ThreadPool.SetMinThreads(processorCount, processorCount * 2);
        Debug.Log("[macOS] 스레드 풀 최적화 적용");

#elif UNITY_STANDALONE_LINUX
        // Linux: epoll 기반 I/O 멀티플렉싱
        ThreadPool.SetMinThreads(processorCount, processorCount * 2);
        Debug.Log("[Linux] 스레드 풀 최적화 적용");
#endif
    }

    // =============================================
    // 파일 I/O 차이
    // =============================================

    /// <summary>
    /// 플랫폼별 최적 파일 I/O 전략
    /// </summary>
    public static async Task<byte[]> ReadFileOptimalAsync(string path, CancellationToken token)
    {
        // 데스크톱에서는 비동기 파일 I/O를 완전히 활용 가능
        // 단, OS별로 내부 구현 방식이 다름
        //   - Windows: Overlapped I/O (진정한 비동기)
        //   - macOS/Linux: 스레드 풀 기반 시뮬레이션

        using var stream = new System.IO.FileStream(
            path,
            System.IO.FileMode.Open,
            System.IO.FileAccess.Read,
            System.IO.FileShare.Read,
            bufferSize: 4096,
            useAsync: true  // 비동기 모드 활성화
        );

        byte[] buffer = new byte[stream.Length];
        await stream.ReadAsync(buffer, 0, buffer.Length, token);
        return buffer;
    }
}
```

### 5.2 OS별 프로세스 우선순위와 스레드 어피니티

```csharp
using UnityEngine;
using System;
using System.Threading;

/// <summary>
/// 데스크톱 플랫폼에서의 스레드 우선순위 관리
/// </summary>
public class DesktopThreadPriorityManager : MonoBehaviour
{
    // =============================================
    // 스레드 우선순위 설정 (데스크톱 전용)
    // =============================================

    public void ConfigureBackgroundThread(Thread thread)
    {
#if UNITY_STANDALONE
        // 데스크톱에서는 스레드 우선순위 조절이 효과적
        thread.Priority = System.Threading.ThreadPriority.BelowNormal;
        thread.IsBackground = true;
        Debug.Log("[Desktop] 백그라운드 스레드 우선순위 설정: BelowNormal");
#endif
    }

    // =============================================
    // 데스크톱에서의 고성능 병렬 처리
    // =============================================

    // ✅ 데스크톱은 코어 수가 많아 병렬 처리에 유리
    public void ProcessWithFullParallelism(float[] data)
    {
#if UNITY_STANDALONE
        int coreCount = Environment.ProcessorCount;
        Debug.Log($"[Desktop] {coreCount}코어로 병렬 처리 시작");

        // 데스크톱에서는 Parallel.For를 적극 활용 가능
        System.Threading.Tasks.Parallel.For(0, data.Length,
            new ParallelOptions
            {
                MaxDegreeOfParallelism = coreCount
            },
            i =>
            {
                data[i] = Mathf.Sin(data[i]) * Mathf.Cos(data[i]);
            });
#else
        // 비데스크톱 플랫폼에서는 순차 처리 또는 제한적 병렬 처리
        for (int i = 0; i < data.Length; i++)
        {
            data[i] = Mathf.Sin(data[i]) * Mathf.Cos(data[i]);
        }
#endif
    }
}
```

---

## 6. 콘솔 플랫폼 (PS, Xbox, Switch) 고려사항

### 6.1 콘솔 공통 제약

콘솔 플랫폼은 NDA(비밀유지계약) 하에 정보가 제한되지만, 일반적인 주의사항은 다음과 같습니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// 콘솔 플랫폼의 동시성 제약을 고려한 가이드라인
/// 주의: 구체적인 콘솔 API는 NDA 대상이므로 일반 원칙만 설명
/// </summary>
public class ConsoleConcurrencyGuidelines : MonoBehaviour
{
    // =============================================
    // 콘솔 플랫폼 공통 원칙
    // =============================================
    //
    // 1. 코어 수가 고정되어 있으므로 정확히 알 수 있음
    //    - PS4: 6 가용 코어 (8코어 중 2개는 OS용)
    //    - PS5: 7 가용 코어 (8코어 중 1개는 OS용)
    //    - Xbox One: 6 가용 코어 (8코어 중 2개는 OS용)
    //    - Xbox Series X: 7 가용 코어
    //    - Switch: 3 가용 코어 (4코어 중 1개는 OS용)
    //
    // 2. 메모리가 고정되어 있으므로 스레드 스택 크기에 주의
    //
    // 3. 독점 타이틀은 하드웨어를 완전히 점유 가능
    //    - 스레드 어피니티(Thread Affinity) 직접 설정 가능
    //
    // 4. Unity Job System이 콘솔에 최적화되어 있으므로 권장

    // =============================================
    // 콘솔을 고려한 보수적 스레딩
    // =============================================

    private int GetSafeWorkerCount()
    {
        int processors = SystemInfo.processorCount;

#if UNITY_PS4
        return Mathf.Min(4, processors - 2);  // OS + 렌더 스레드 예약
#elif UNITY_PS5
        return Mathf.Min(5, processors - 2);
#elif UNITY_XBOXONE
        return Mathf.Min(4, processors - 2);
#elif UNITY_GAMECORE_XBOXSERIES
        return Mathf.Min(5, processors - 2);
#elif UNITY_SWITCH
        return Mathf.Min(2, processors - 1);  // Switch는 코어가 적음
#else
        return Mathf.Max(1, processors - 2);
#endif
    }

    // =============================================
    // Switch 특별 고려사항
    // =============================================

    // Switch는 코어 수가 적고 클럭 속도가 낮으므로
    // 동시성보다 작업 분산(시간 분할)이 효과적일 수 있음

    // ✅ Switch 최적화: 작업을 작은 단위로 분할
    public void ProcessForSwitch(float[] data)
    {
#if UNITY_SWITCH
        // Switch에서는 Job System의 배치 크기를 크게 설정하여
        // 스레드 전환 오버헤드 최소화
        int batchSize = 256; // 더 큰 배치로 오버헤드 감소
#else
        int batchSize = 64;  // 일반적인 배치 크기
#endif

        Debug.Log($"배치 크기: {batchSize} (코어 수: {SystemInfo.processorCount})");
    }
}
```

### 6.2 콘솔 메모리 제약과 스레드 스택

```csharp
using System.Threading;

/// <summary>
/// 콘솔 플랫폼의 메모리 제약을 고려한 스레드 생성
/// </summary>
public static class ConsoleThreadFactory
{
    // =============================================
    // 콘솔에서의 스레드 스택 크기 최적화
    // =============================================

    public static Thread CreateOptimizedThread(ThreadStart action, string name = "Worker")
    {
        // 콘솔은 메모리가 제한적이므로 스레드 스택 크기를 최소화
        int stackSize;

#if UNITY_PS4 || UNITY_PS5 || UNITY_XBOXONE || UNITY_GAMECORE_XBOXSERIES
        stackSize = 256 * 1024;  // 256KB (콘솔 표준)
#elif UNITY_SWITCH
        stackSize = 128 * 1024;  // 128KB (Switch는 더 보수적으로)
#else
        stackSize = 0;           // OS 기본값 사용 (보통 1MB)
#endif

        var thread = stackSize > 0
            ? new Thread(action, stackSize)
            : new Thread(action);

        thread.Name = name;
        thread.IsBackground = true;
        return thread;
    }
}
```

---

## 7. 플랫폼별 네트워크 제한

### 7.1 SSL/TLS 및 인증서 차이

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System;
using System.Security.Cryptography.X509Certificates;

/// <summary>
/// 플랫폼별 SSL/TLS 처리 차이를 관리하는 핸들러
/// </summary>
public class PlatformSSLHandler : MonoBehaviour
{
    // =============================================
    // 플랫폼별 TLS 지원 현황
    // =============================================
    //
    // Windows:  TLS 1.2/1.3 (Schannel 사용)
    // macOS:    TLS 1.2/1.3 (SecureTransport 사용)
    // Linux:    TLS 1.2/1.3 (OpenSSL 사용)
    // iOS:      TLS 1.2 필수 (App Transport Security)
    // Android:  TLS 1.2+ (API 29+ 기본 HTTPS 강제)
    // WebGL:    브라우저의 TLS 스택 사용
    // 콘솔:     플랫폼 전용 TLS 라이브러리

    // =============================================
    // 플랫폼별 인증서 검증
    // =============================================

    /// <summary>
    /// 플랫폼에 적합한 인증서 핸들러를 생성
    /// </summary>
    public static CertificateHandler CreatePlatformCertHandler()
    {
#if UNITY_IOS && !UNITY_EDITOR
        // iOS App Transport Security는 TLS 1.2 이상 강제
        // 자체 서명 인증서 사용 시 Info.plist에 예외 설정 필요
        return null; // 기본 시스템 검증 사용
#elif UNITY_ANDROID && !UNITY_EDITOR
        // Android는 Network Security Config로 인증서 핀닝 가능
        // Unity에서는 커스텀 CertificateHandler 활용
        return null; // 기본 시스템 검증 사용
#elif UNITY_WEBGL && !UNITY_EDITOR
        // WebGL은 브라우저의 인증서 검증을 따름
        // CertificateHandler 설정 불가
        return null;
#else
        // 데스크톱/에디터: 기본 검증 사용
        return null;
#endif
    }

    // =============================================
    // 플랫폼별 네트워크 요청 래퍼
    // =============================================

    public static UnityWebRequest CreateSecureRequest(string url)
    {
        var request = UnityWebRequest.Get(url);

#if UNITY_ANDROID && !UNITY_EDITOR
        // Android에서 cleartext HTTP 트래픽은 API 28+에서 기본 차단
        if (url.StartsWith("http://", StringComparison.OrdinalIgnoreCase))
        {
            Debug.LogWarning("[Android] HTTP 평문 통신은 Android 9+에서 차단될 수 있음. HTTPS 사용 권장");
        }
#endif

#if UNITY_IOS && !UNITY_EDITOR
        // iOS ATS(App Transport Security)는 HTTP를 기본 차단
        if (url.StartsWith("http://", StringComparison.OrdinalIgnoreCase))
        {
            Debug.LogWarning("[iOS] ATS에 의해 HTTP가 차단됨. HTTPS 사용 필수");
        }
#endif

        // 타임아웃 설정 (플랫폼별 권장값)
        request.timeout = GetPlatformTimeout();

        return request;
    }

    private static int GetPlatformTimeout()
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        return 30;  // WebGL: 브라우저 제한 고려
#elif UNITY_IOS && !UNITY_EDITOR
        return 30;  // iOS: 워치독 타이머 고려
#elif UNITY_SWITCH
        return 60;  // Switch: 느린 네트워크 고려
#else
        return 30;  // 기본값
#endif
    }
}
```

### 7.2 플랫폼별 동시 연결 수 제한

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// 플랫폼별 동시 네트워크 연결 수 제한을 관리하는 시스템
/// </summary>
public class PlatformNetworkThrottler
{
    private readonly SemaphoreSlim _semaphore;

    public PlatformNetworkThrottler()
    {
        int maxConnections = GetMaxConcurrentConnections();
        _semaphore = new SemaphoreSlim(maxConnections);
        Debug.Log($"[Network] 최대 동시 연결 수: {maxConnections}");
    }

    private static int GetMaxConcurrentConnections()
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        return 4;   // WebGL: 브라우저당 동일 호스트 6개 연결 제한 (안전 마진)
#elif UNITY_IOS && !UNITY_EDITOR
        return 4;   // iOS: 배터리/메모리 고려
#elif UNITY_ANDROID && !UNITY_EDITOR
        return 6;   // Android: 적절한 수준
#elif UNITY_SWITCH
        return 2;   // Switch: 네트워크 리소스 제한
#else
        return 10;  // 데스크톱: 넉넉하게
#endif
    }

    /// <summary>
    /// 동시 연결 수를 제한하여 네트워크 요청 실행
    /// </summary>
    public async Task<T> ExecuteThrottledAsync<T>(
        Func<CancellationToken, Task<T>> request,
        CancellationToken token)
    {
        await _semaphore.WaitAsync(token);
        try
        {
            return await request(token);
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

---

## 8. 전처리기 지시문 활용

### 8.1 Unity 플랫폼 심볼 일람

```csharp
// =============================================
// 주요 Unity 전처리기 심볼 정리
// =============================================
//
// ── 플랫폼 ──
// UNITY_STANDALONE         모든 데스크톱 (Win/Mac/Linux)
// UNITY_STANDALONE_WIN     Windows
// UNITY_STANDALONE_OSX     macOS
// UNITY_STANDALONE_LINUX   Linux
// UNITY_IOS                iOS
// UNITY_ANDROID            Android
// UNITY_WEBGL              WebGL
// UNITY_PS4                PlayStation 4
// UNITY_PS5                PlayStation 5
// UNITY_XBOXONE            Xbox One
// UNITY_GAMECORE_XBOXSERIES  Xbox Series X|S
// UNITY_SWITCH             Nintendo Switch
//
// ── 에디터 ──
// UNITY_EDITOR             에디터에서 실행 중
// UNITY_EDITOR_WIN         Windows 에디터
// UNITY_EDITOR_OSX         macOS 에디터
//
// ── 기능 ──
// ENABLE_MONO              Mono 스크립팅 백엔드
// ENABLE_IL2CPP            IL2CPP 스크립팅 백엔드
// NET_STANDARD_2_0         .NET Standard 2.0
// NET_STANDARD_2_1         .NET Standard 2.1
//
// ── 버전 ──
// UNITY_2022_3_OR_NEWER    Unity 2022.3 이상
// UNITY_2023_1_OR_NEWER    Unity 2023.1 이상
// UNITY_6000_0_OR_NEWER    Unity 6 이상
```

### 8.2 플랫폼별 동시성 전략 분기

```csharp
using UnityEngine;
using System;
using System.Collections;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// 전처리기 지시문을 활용한 크로스 플랫폼 비동기 처리기
/// </summary>
public class CrossPlatformAsyncProcessor : MonoBehaviour
{
    // =============================================
    // 전처리기를 활용한 플랫폼별 최적 구현 선택
    // =============================================

    public void StartHeavyWork(float[] data, Action<float[]> onComplete)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        // WebGL: Coroutine 기반 프레임 분산
        StartCoroutine(ProcessCoroutine(data, onComplete));

#elif UNITY_IOS && !UNITY_EDITOR
        // iOS: 제한된 병렬성으로 Task 기반 처리
        _ = ProcessAsyncThrottled(data, onComplete, maxParallelism: 3);

#elif UNITY_SWITCH
        // Switch: Job System 활용 또는 제한적 병렬 처리
        _ = ProcessAsyncThrottled(data, onComplete, maxParallelism: 2);

#elif UNITY_STANDALONE
        // 데스크톱: 전체 코어 활용
        _ = ProcessAsyncFull(data, onComplete);

#else
        // 기본: 안전한 방식
        _ = ProcessAsyncThrottled(data, onComplete,
            maxParallelism: Mathf.Max(1, SystemInfo.processorCount / 2));
#endif
    }

    // =============================================
    // WebGL 전용: Coroutine 기반 처리
    // =============================================

    private IEnumerator ProcessCoroutine(float[] data, Action<float[]> onComplete)
    {
        float timeBudget = 0.008f; // 8ms per frame

        for (int i = 0; i < data.Length;)
        {
            float start = Time.realtimeSinceStartup;
            while (i < data.Length && (Time.realtimeSinceStartup - start) < timeBudget)
            {
                data[i] = Mathf.Sqrt(Mathf.Abs(data[i]));
                i++;
            }
            yield return null;
        }

        onComplete?.Invoke(data);
    }

    // =============================================
    // 모바일/콘솔: 제한된 병렬 처리
    // =============================================

    private async Task ProcessAsyncThrottled(float[] data, Action<float[]> onComplete,
        int maxParallelism)
    {
        var semaphore = new SemaphoreSlim(maxParallelism);
        int chunkSize = data.Length / maxParallelism;
        var tasks = new Task[maxParallelism];

        for (int t = 0; t < maxParallelism; t++)
        {
            int start = t * chunkSize;
            int end = (t == maxParallelism - 1) ? data.Length : start + chunkSize;

            tasks[t] = Task.Run(async () =>
            {
                await semaphore.WaitAsync();
                try
                {
                    for (int i = start; i < end; i++)
                    {
                        data[i] = Mathf.Sqrt(Mathf.Abs(data[i]));
                    }
                }
                finally
                {
                    semaphore.Release();
                }
            });
        }

        await Task.WhenAll(tasks);
        onComplete?.Invoke(data);
    }

    // =============================================
    // 데스크톱: 최대 성능 병렬 처리
    // =============================================

    private async Task ProcessAsyncFull(float[] data, Action<float[]> onComplete)
    {
        await Task.Run(() =>
        {
            System.Threading.Tasks.Parallel.For(0, data.Length, i =>
            {
                data[i] = Mathf.Sqrt(Mathf.Abs(data[i]));
            });
        });

        onComplete?.Invoke(data);
    }
}
```

### 8.3 IL2CPP vs Mono 차이

```csharp
using UnityEngine;
using System;
using System.Threading;

/// <summary>
/// 스크립팅 백엔드별 동시성 동작 차이
/// </summary>
public class ScriptingBackendDifferences : MonoBehaviour
{
    // =============================================
    // IL2CPP vs Mono 동시성 차이점
    // =============================================
    //
    // 1. IL2CPP는 Volatile 시맨틱이 더 엄격함
    // 2. IL2CPP는 Thread.Abort()를 지원하지 않음
    // 3. IL2CPP는 일부 리플렉션 기반 동시성 패턴에 제한 있음
    // 4. IL2CPP는 AOT 컴파일이므로 제네릭 제약이 있음

    // ❌ IL2CPP에서 문제가 되는 패턴
    void BadIL2CPPPattern()
    {
#if ENABLE_MONO
        // Mono에서만 동작: Thread.Abort() 사용
        Thread thread = new Thread(() =>
        {
            while (true) { Thread.Sleep(100); }
        });
        thread.Start();
        thread.Abort(); // IL2CPP에서 NotSupportedException 발생!
#endif
    }

    // ✅ 모든 백엔드에서 안전한 패턴: CancellationToken 사용
    void GoodCancellationPattern()
    {
        var cts = new CancellationTokenSource();

        Thread thread = new Thread(() =>
        {
            while (!cts.Token.IsCancellationRequested)
            {
                Thread.Sleep(100);
            }
            Debug.Log("스레드가 안전하게 종료됨");
        });

        thread.Start();

        // 취소 요청 (Thread.Abort 대신)
        cts.Cancel();
    }
}
```

---

## 9. 플랫폼별 비동기 패턴 선택 가이드

### 9.1 의사결정 플로우차트

```
플랫폼별 비동기 패턴 선택:

Q1. 대상 플랫폼에 WebGL이 포함되는가?
 ├─ YES → Coroutine 또는 UniTask(WebGL 호환 모드) 사용
 │         Thread, Task.Run, Parallel 사용 금지
 │
 └─ NO → Q2. 모바일 플랫폼(iOS/Android)이 포함되는가?
          ├─ YES → Q3. 백그라운드 실행이 필요한가?
          │         ├─ YES → 백그라운드 상태 관리 + CancellationToken 필수
          │         └─ NO  → async/await + 동시성 제한(SemaphoreSlim)
          │
          └─ NO → Q4. 콘솔 플랫폼이 포함되는가?
                   ├─ YES → Job System 우선, 제한적 Thread Pool 사용
                   └─ NO  → 데스크톱 전용: 모든 패턴 자유 사용
```

### 9.2 플랫폼별 권장 패턴 매트릭스

```csharp
using UnityEngine;
using System;

/// <summary>
/// 플랫폼별 권장 비동기 패턴을 안내하는 유틸리티
/// </summary>
public static class PlatformAsyncPatternGuide
{
    // =============================================
    // 각 시나리오별 권장 패턴
    // =============================================

    // ── 네트워크 I/O ──
    // 전 플랫폼: UnityWebRequest + Coroutine (가장 안전)
    // 데스크톱/모바일: UnityWebRequest + async/await 또는 UniTask
    // WebGL: UnityWebRequest + Coroutine만 사용

    // ── CPU 집약 작업 ──
    // 데스크톱: Parallel.For / Task.Run / Job System
    // 모바일: Job System / 제한된 Task.Run
    // 콘솔: Job System (권장)
    // WebGL: Coroutine 프레임 분산 처리

    // ── 파일 I/O ──
    // 데스크톱: async FileStream
    // 모바일: async FileStream (작은 파일) / 스트리밍 (큰 파일)
    // WebGL: IndexedDB via PlayerPrefs 또는 JavaScript Interop
    // 콘솔: 플랫폼 전용 API

    // ── 실시간 데이터 스트리밍 ──
    // 데스크톱: Channel<T> / IAsyncEnumerable
    // 모바일: UniTask AsyncEnumerable
    // WebGL: Coroutine 기반 폴링

    /// <summary>
    /// 현재 플랫폼에서 권장되는 패턴을 로그로 출력
    /// </summary>
    public static void LogRecommendations()
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        Debug.Log("[가이드] WebGL 권장 패턴:");
        Debug.Log("  - 네트워크: Coroutine + UnityWebRequest");
        Debug.Log("  - CPU 작업: 프레임 분산 (시간 슬라이싱)");
        Debug.Log("  - 저장: PlayerPrefs 또는 JS Interop");
        Debug.Log("  - 금지: Thread, Task.Run, Parallel, Job System");

#elif UNITY_IOS && !UNITY_EDITOR
        Debug.Log("[가이드] iOS 권장 패턴:");
        Debug.Log("  - 네트워크: async/await + 타임아웃 필수");
        Debug.Log("  - CPU 작업: Job System 또는 제한된 Task.Run");
        Debug.Log("  - 백그라운드: OnApplicationPause 감지 필수");
        Debug.Log("  - TLS: 1.2 이상 필수 (ATS 규정)");

#elif UNITY_ANDROID && !UNITY_EDITOR
        Debug.Log("[가이드] Android 권장 패턴:");
        Debug.Log("  - 네트워크: async/await + Doze 모드 대응");
        Debug.Log("  - CPU 작업: Job System 또는 배터리 기반 조절");
        Debug.Log("  - 백그라운드: 배터리/Doze 모드 고려");
        Debug.Log("  - TLS: API 28+ HTTPS 강제");

#elif UNITY_SWITCH
        Debug.Log("[가이드] Switch 권장 패턴:");
        Debug.Log("  - CPU 작업: Job System (배치 크기 크게)");
        Debug.Log("  - 스레드: 최대 2개 워커 스레드");
        Debug.Log("  - 메모리: 스레드 스택 128KB 이내");

#elif UNITY_STANDALONE
        Debug.Log("[가이드] 데스크톱 권장 패턴:");
        Debug.Log("  - 모든 비동기 패턴 사용 가능");
        Debug.Log("  - CPU 작업: Parallel.For, Job System 적극 활용");
        Debug.Log("  - 네트워크: HttpClient async/await 가능");
#endif
    }
}
```

---

## 10. 크로스 플랫폼 추상화 레이어 구현

### 10.1 플랫폼 독립적 비동기 실행기

```csharp
using UnityEngine;
using System;
using System.Collections;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// 플랫폼 차이를 추상화하는 비동기 작업 실행기
/// 모든 플랫폼에서 동일한 인터페이스로 비동기 작업을 수행할 수 있음
/// </summary>
public class CrossPlatformAsyncRunner : MonoBehaviour
{
    private static CrossPlatformAsyncRunner _instance;
    public static CrossPlatformAsyncRunner Instance
    {
        get
        {
            if (_instance == null)
            {
                var go = new GameObject("[CrossPlatformAsyncRunner]");
                _instance = go.AddComponent<CrossPlatformAsyncRunner>();
                DontDestroyOnLoad(go);
            }
            return _instance;
        }
    }

    // =============================================
    // 통합 인터페이스: 무거운 작업 실행
    // =============================================

    /// <summary>
    /// 플랫폼에 관계없이 무거운 작업을 안전하게 실행
    /// - WebGL: 프레임 분산 처리
    /// - 모바일: 제한된 병렬 실행
    /// - 데스크톱: 전체 병렬 실행
    /// </summary>
    public void RunHeavyWork<T>(
        IList<T> items,
        Action<T> processor,
        Action onComplete = null,
        Action<float> onProgress = null)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        StartCoroutine(RunCoroutineBased(items, processor, onComplete, onProgress));
#else
        _ = RunTaskBased(items, processor, onComplete, onProgress);
#endif
    }

    // =============================================
    // WebGL 구현: Coroutine 기반 프레임 분산
    // =============================================

    private IEnumerator RunCoroutineBased<T>(
        IList<T> items,
        Action<T> processor,
        Action onComplete,
        Action<float> onProgress)
    {
        float timeBudget = 0.010f; // 10ms per frame
        int processed = 0;

        while (processed < items.Count)
        {
            float frameStart = Time.realtimeSinceStartup;

            while (processed < items.Count &&
                   (Time.realtimeSinceStartup - frameStart) < timeBudget)
            {
                try
                {
                    processor(items[processed]);
                }
                catch (Exception ex)
                {
                    Debug.LogError($"작업 처리 오류 [{processed}]: {ex.Message}");
                }
                processed++;
            }

            onProgress?.Invoke((float)processed / items.Count);
            yield return null;
        }

        onComplete?.Invoke();
    }

    // =============================================
    // 비-WebGL 구현: Task 기반 병렬 처리
    // =============================================

    private async Task RunTaskBased<T>(
        IList<T> items,
        Action<T> processor,
        Action onComplete,
        Action<float> onProgress)
    {
        int maxParallelism = PlatformConcurrencyInfo.RecommendedMaxConcurrency;
        var semaphore = new SemaphoreSlim(maxParallelism);
        int processed = 0;
        int total = items.Count;

        var tasks = new Task[total];
        for (int i = 0; i < total; i++)
        {
            var item = items[i];
            tasks[i] = Task.Run(async () =>
            {
                await semaphore.WaitAsync();
                try
                {
                    processor(item);
                    int current = Interlocked.Increment(ref processed);
                    onProgress?.Invoke((float)current / total);
                }
                finally
                {
                    semaphore.Release();
                }
            });
        }

        await Task.WhenAll(tasks);
        onComplete?.Invoke();
    }
}
```

### 10.2 플랫폼 추상화 네트워크 클라이언트

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System;
using System.Collections;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// 플랫폼 차이를 추상화하는 네트워크 클라이언트
/// 콜백 기반의 통합 인터페이스를 제공
/// </summary>
public class CrossPlatformNetworkClient : MonoBehaviour
{
    private static CrossPlatformNetworkClient _instance;

    public static CrossPlatformNetworkClient Instance
    {
        get
        {
            if (_instance == null)
            {
                var go = new GameObject("[CrossPlatformNetworkClient]");
                _instance = go.AddComponent<CrossPlatformNetworkClient>();
                DontDestroyOnLoad(go);
            }
            return _instance;
        }
    }

    // =============================================
    // 통합 API: 플랫폼 독립적 HTTP 요청
    // =============================================

    /// <summary>
    /// GET 요청 (모든 플랫폼에서 동일하게 동작)
    /// </summary>
    public void Get(string url, Action<string> onSuccess, Action<string> onError,
        int timeoutSeconds = 0)
    {
        if (timeoutSeconds <= 0)
            timeoutSeconds = GetDefaultTimeout();

        StartCoroutine(GetCoroutine(url, onSuccess, onError, timeoutSeconds));
    }

    /// <summary>
    /// POST 요청 (모든 플랫폼에서 동일하게 동작)
    /// </summary>
    public void Post(string url, string jsonBody, Action<string> onSuccess,
        Action<string> onError, int timeoutSeconds = 0)
    {
        if (timeoutSeconds <= 0)
            timeoutSeconds = GetDefaultTimeout();

        StartCoroutine(PostCoroutine(url, jsonBody, onSuccess, onError, timeoutSeconds));
    }

    // =============================================
    // 내부 구현: UnityWebRequest 기반 (전 플랫폼 호환)
    // =============================================

    private IEnumerator GetCoroutine(string url, Action<string> onSuccess,
        Action<string> onError, int timeout)
    {
        using var request = UnityWebRequest.Get(url);
        request.timeout = timeout;

        // 플랫폼별 설정 적용
        ApplyPlatformSettings(request);

        yield return request.SendWebRequest();

        if (request.result == UnityWebRequest.Result.Success)
        {
            onSuccess?.Invoke(request.downloadHandler.text);
        }
        else
        {
            string error = FormatPlatformError(request);
            onError?.Invoke(error);
        }
    }

    private IEnumerator PostCoroutine(string url, string jsonBody,
        Action<string> onSuccess, Action<string> onError, int timeout)
    {
        byte[] bodyRaw = System.Text.Encoding.UTF8.GetBytes(jsonBody);
        using var request = new UnityWebRequest(url, "POST");
        request.uploadHandler = new UploadHandlerRaw(bodyRaw);
        request.downloadHandler = new DownloadHandlerBuffer();
        request.SetRequestHeader("Content-Type", "application/json");
        request.timeout = timeout;

        ApplyPlatformSettings(request);

        yield return request.SendWebRequest();

        if (request.result == UnityWebRequest.Result.Success)
        {
            onSuccess?.Invoke(request.downloadHandler.text);
        }
        else
        {
            string error = FormatPlatformError(request);
            onError?.Invoke(error);
        }
    }

    // =============================================
    // 플랫폼별 설정 적용
    // =============================================

    private void ApplyPlatformSettings(UnityWebRequest request)
    {
#if UNITY_IOS && !UNITY_EDITOR
        // iOS: ATS 호환성을 위한 추가 설정
        request.SetRequestHeader("Accept", "application/json");
#endif

#if UNITY_WEBGL && !UNITY_EDITOR
        // WebGL: CORS 관련 추가 헤더가 필요할 수 있음
        // 서버 측에서 Access-Control-Allow-Origin 설정 필요
#endif
    }

    private int GetDefaultTimeout()
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        return 30;
#elif UNITY_IOS && !UNITY_EDITOR
        return 30;
#elif UNITY_SWITCH
        return 60;
#else
        return 30;
#endif
    }

    private string FormatPlatformError(UnityWebRequest request)
    {
        string baseError = $"[{request.responseCode}] {request.error}";

#if UNITY_IOS && !UNITY_EDITOR
        if (request.error != null && request.error.Contains("SSL"))
        {
            baseError += " (iOS ATS 설정 확인 필요 - Info.plist)";
        }
#elif UNITY_ANDROID && !UNITY_EDITOR
        if (request.error != null && request.error.Contains("cleartext"))
        {
            baseError += " (Android 네트워크 보안 설정 확인 필요 - network_security_config.xml)";
        }
#elif UNITY_WEBGL && !UNITY_EDITOR
        if (request.responseCode == 0)
        {
            baseError += " (CORS 정책 위반 가능성 - 서버 측 설정 확인)";
        }
#endif
        return baseError;
    }
}
```

### 10.3 플랫폼 추상화 팩토리 패턴

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// 플랫폼별 최적 구현을 자동 선택하는 팩토리
/// </summary>
public interface IPlatformWorkScheduler
{
    void Schedule<T>(T[] items, Action<T> work, Action onComplete);
    int MaxConcurrency { get; }
}

// =============================================
// WebGL 구현: 프레임 기반 스케줄러
// =============================================

public class WebGLWorkScheduler : IPlatformWorkScheduler
{
    private readonly MonoBehaviour _host;
    public int MaxConcurrency => 1; // 단일 스레드

    public WebGLWorkScheduler(MonoBehaviour host)
    {
        _host = host;
    }

    public void Schedule<T>(T[] items, Action<T> work, Action onComplete)
    {
        _host.StartCoroutine(ProcessItems(items, work, onComplete));
    }

    private System.Collections.IEnumerator ProcessItems<T>(
        T[] items, Action<T> work, Action onComplete)
    {
        float budget = 0.010f;
        int idx = 0;

        while (idx < items.Length)
        {
            float start = Time.realtimeSinceStartup;
            while (idx < items.Length && (Time.realtimeSinceStartup - start) < budget)
            {
                work(items[idx++]);
            }
            yield return null;
        }
        onComplete?.Invoke();
    }
}

// =============================================
// 스레드 지원 플랫폼 구현: Task 기반 스케줄러
// =============================================

public class ThreadedWorkScheduler : IPlatformWorkScheduler
{
    public int MaxConcurrency { get; }

    public ThreadedWorkScheduler(int maxConcurrency)
    {
        MaxConcurrency = maxConcurrency;
    }

    public void Schedule<T>(T[] items, Action<T> work, Action onComplete)
    {
        Task.Run(async () =>
        {
            var semaphore = new SemaphoreSlim(MaxConcurrency);
            var tasks = new Task[items.Length];

            for (int i = 0; i < items.Length; i++)
            {
                var item = items[i];
                tasks[i] = Task.Run(async () =>
                {
                    await semaphore.WaitAsync();
                    try { work(item); }
                    finally { semaphore.Release(); }
                });
            }

            await Task.WhenAll(tasks);
            onComplete?.Invoke();
        });
    }
}

// =============================================
// 팩토리: 현재 플랫폼에 맞는 스케줄러 생성
// =============================================

public static class PlatformWorkSchedulerFactory
{
    public static IPlatformWorkScheduler Create(MonoBehaviour host)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        Debug.Log("[Factory] WebGL 스케줄러 생성 (프레임 분산)");
        return new WebGLWorkScheduler(host);
#elif UNITY_SWITCH
        Debug.Log("[Factory] Switch 스케줄러 생성 (2 워커)");
        return new ThreadedWorkScheduler(2);
#elif UNITY_IOS && !UNITY_EDITOR
        Debug.Log("[Factory] iOS 스케줄러 생성 (3 워커)");
        return new ThreadedWorkScheduler(3);
#elif UNITY_ANDROID && !UNITY_EDITOR
        int cores = Mathf.Max(2, SystemInfo.processorCount / 2);
        Debug.Log($"[Factory] Android 스케줄러 생성 ({cores} 워커)");
        return new ThreadedWorkScheduler(cores);
#else
        int cores = SystemInfo.processorCount;
        Debug.Log($"[Factory] 데스크톱 스케줄러 생성 ({cores} 워커)");
        return new ThreadedWorkScheduler(cores);
#endif
    }
}
```

### 10.4 사용 예시

```csharp
using UnityEngine;

/// <summary>
/// 크로스 플랫폼 추상화 레이어 사용 예시
/// </summary>
public class CrossPlatformExample : MonoBehaviour
{
    private IPlatformWorkScheduler _scheduler;

    private void Start()
    {
        // 팩토리가 플랫폼에 맞는 스케줄러를 자동 생성
        _scheduler = PlatformWorkSchedulerFactory.Create(this);

        // 플랫폼에 관계없이 동일한 코드로 작업 실행
        float[] data = new float[10000];
        for (int i = 0; i < data.Length; i++) data[i] = i;

        Debug.Log($"[Example] 최대 동시성: {_scheduler.MaxConcurrency}");
        Debug.Log("[Example] 데이터 처리 시작...");

        _scheduler.Schedule(data,
            item =>
            {
                // 무거운 연산 시뮬레이션
                float result = Mathf.Sqrt(Mathf.Abs(item)) * Mathf.PI;
            },
            () =>
            {
                Debug.Log("[Example] 모든 데이터 처리 완료!");
            });
    }
}
```

---

## 주의사항

### 플랫폼별 핵심 주의사항 요약

| 플랫폼 | 주의사항 | 심각도 |
|---------|----------|--------|
| **WebGL** | `Thread`, `Task.Run`, `Parallel` 사용 불가 | 치명적 |
| **WebGL** | `Task.Delay` 정확도 보장 불가 | 높음 |
| **iOS** | 백그라운드 5~10초 후 일시 중지 | 높음 |
| **iOS** | 과도한 스레드 생성 시 앱 종료 | 높음 |
| **iOS** | ATS로 인한 HTTP 차단 | 중간 |
| **Android** | Doze 모드에서 네트워크 차단 | 높음 |
| **Android** | API 28+ 평문 HTTP 차단 | 중간 |
| **Switch** | 가용 코어 3개, 클럭 속도 낮음 | 높음 |
| **콘솔** | 고정 메모리, 스레드 스택 크기 제한 | 중간 |
| **IL2CPP** | `Thread.Abort()` 미지원 | 높음 |
| **IL2CPP** | 일부 리플렉션 기반 패턴 제한 | 중간 |

### 자주 발생하는 실수

```csharp
// =============================================
// ❌ 실수 1: WebGL 빌드에서 Thread 사용
// =============================================
// 에디터에서는 잘 동작하지만 WebGL 빌드에서 런타임 오류 발생
void Mistake1()
{
    // 에디터에서 테스트 시 문제없지만 WebGL 배포 시 실패
    #if !UNITY_WEBGL || UNITY_EDITOR
    Task.Run(() => Debug.Log("Worker thread"));
    #endif
}

// =============================================
// ❌ 실수 2: 모바일에서 백그라운드 상태 무시
// =============================================
// 백그라운드 전환 시 네트워크 요청이 타임아웃되거나 실패
// 반드시 OnApplicationPause/OnApplicationFocus 감지 필요

// =============================================
// ❌ 실수 3: 콘솔에서 기본 스레드 스택 크기 사용
// =============================================
// 데스크톱의 1MB 기본 스택은 콘솔에서 메모리 부족 유발
// 콘솔에서는 128~256KB로 명시적 설정 필요

// =============================================
// ❌ 실수 4: 플랫폼 분기 없이 Parallel.For 사용
// =============================================
// WebGL에서 컴파일은 되지만 런타임 오류 발생
// 반드시 전처리기 지시문으로 분기 처리 필요
```

---

## 베스트 프랙티스

### 1. 최소 공통 분모 원칙

크로스 플랫폼 개발 시, **가장 제약이 많은 플랫폼**을 기준으로 기본 구현을 작성하고, 플랫폼별로 최적화를 추가합니다.

```csharp
// ✅ 최소 공통 분모: Coroutine (전 플랫폼 호환)
// → 그 위에 플랫폼별 최적화 레이어를 추가

// 기본 구현: 모든 플랫폼에서 동작
public interface IAsyncOperation<T>
{
    void Execute(Action<T> onResult, Action<Exception> onError);
}

// 플랫폼별로 내부 구현만 다르게 제공
```

### 2. 전처리기 지시문 최소화

```csharp
// ❌ 코드 전체에 #if를 흩뿌리지 않기
public void DoWork()
{
    #if UNITY_WEBGL
    // WebGL 코드
    #elif UNITY_IOS
    // iOS 코드
    #elif UNITY_ANDROID
    // Android 코드
    #else
    // 기타 코드
    #endif
}

// ✅ 추상화 레이어에 집중하고 호출부는 깔끔하게 유지
public void DoWork()
{
    var scheduler = PlatformWorkSchedulerFactory.Create(this);
    scheduler.Schedule(data, Process, OnComplete);
}
```

### 3. 에디터 테스트 주의

```csharp
// ✅ 에디터에서 WebGL 동작을 시뮬레이션하는 테스트 유틸
#if UNITY_EDITOR
public static class WebGLSimulator
{
    private static bool _simulateWebGL = false;

    public static bool IsSimulating => _simulateWebGL;

    public static void Enable() => _simulateWebGL = true;
    public static void Disable() => _simulateWebGL = false;

    public static bool ShouldUseCoroutines =>
        _simulateWebGL || (Application.platform == RuntimePlatform.WebGLPlayer);
}
#endif
```

### 4. 점진적 성능 향상 (Progressive Enhancement)

```csharp
// ✅ 기본은 안전하게, 플랫폼이 허용하면 성능 강화
public static class ProgressiveEnhancement
{
    public static void ProcessData(float[] data, MonoBehaviour host,
        Action onComplete)
    {
        if (!PlatformConcurrencyInfo.SupportsMultithreading)
        {
            // 폴백: 프레임 분산
            host.StartCoroutine(SlicedProcess(data, onComplete));
        }
        else if (PlatformConcurrencyInfo.RecommendedMaxConcurrency >= 4)
        {
            // 고성능: 풀 병렬
            Task.Run(() =>
            {
                System.Threading.Tasks.Parallel.For(0, data.Length,
                    i => data[i] = Mathf.Sqrt(data[i]));
            }).ContinueWith(_ => onComplete?.Invoke());
        }
        else
        {
            // 제한적 병렬
            Task.Run(() =>
            {
                for (int i = 0; i < data.Length; i++)
                    data[i] = Mathf.Sqrt(data[i]);
            }).ContinueWith(_ => onComplete?.Invoke());
        }
    }

    private static System.Collections.IEnumerator SlicedProcess(
        float[] data, Action onComplete)
    {
        int i = 0;
        while (i < data.Length)
        {
            float start = Time.realtimeSinceStartup;
            while (i < data.Length && (Time.realtimeSinceStartup - start) < 0.008f)
            {
                data[i] = Mathf.Sqrt(data[i]);
                i++;
            }
            yield return null;
        }
        onComplete?.Invoke();
    }
}
```

### 5. 체크리스트

프로젝트 출시 전 플랫폼별 동시성 점검 체크리스트:

- WebGL 빌드에서 `Thread`, `Task.Run`, `Parallel` 사용 코드가 없는지 확인
- iOS/Android에서 `OnApplicationPause` 처리가 되어 있는지 확인
- 모든 네트워크 요청에 타임아웃이 설정되어 있는지 확인
- iOS ATS 규정(HTTPS 강제)을 준수하는지 확인
- Android 네트워크 보안 설정이 올바른지 확인
- 콘솔 플랫폼의 스레드 수와 스택 크기가 적절한지 확인
- IL2CPP 빌드에서 `Thread.Abort()` 사용이 없는지 확인
- 플랫폼별 동시성 수준 제한이 적용되어 있는지 확인

---

## 참고 자료

- [Unity 공식 문서 - Platform-specific Compilation](https://docs.unity3d.com/Manual/PlatformDependentCompilation.html)
- [Unity 공식 문서 - WebGL Networking](https://docs.unity3d.com/Manual/webgl-networking.html)
- [Unity 공식 문서 - Job System](https://docs.unity3d.com/Manual/JobSystem.html)
- [Apple 개발자 문서 - App Transport Security](https://developer.apple.com/documentation/bundleresources/information_property_list/nsapptransportsecurity)
- [Android 개발자 문서 - Doze and App Standby](https://developer.android.com/training/monitoring-device-state/doze-standby)
- [Android 개발자 문서 - Network Security Configuration](https://developer.android.com/training/articles/security-config)
- [Unity 공식 문서 - IL2CPP Scripting Restrictions](https://docs.unity3d.com/Manual/ScriptingRestrictions.html)
- [UniTask GitHub - WebGL 지원](https://github.com/Cysharp/UniTask)

---

[< 이전: 36. 플랫폼별 빌드 설정](./36-platform-build-settings.md) | [다음: 38. 메모리 관리 최적화 >](../13-optimization/38-memory-optimization.md)
