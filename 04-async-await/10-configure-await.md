# 10. ConfigureAwait & 컨텍스트 제어

## 개요

`ConfigureAwait`는 async/await에서 **await 후 어떤 컨텍스트에서 코드를 계속 실행할지** 제어하는 메서드입니다. Unity 환경에서는 Main Thread 복귀와 데드락 방지에 핵심적인 역할을 합니다.

---

## SynchronizationContext 복습

### 컨텍스트 캡처란?

```csharp
public class ContextCaptureExample : MonoBehaviour
{
    async void Start()
    {
        Debug.Log($"Start - Thread: {Thread.CurrentThread.ManagedThreadId}"); // Main Thread

        // 이 시점에서 SynchronizationContext가 캡처됨
        await Task.Delay(1000);

        // 캡처된 컨텍스트(UnitySynchronizationContext)로 복귀
        Debug.Log($"After await - Thread: {Thread.CurrentThread.ManagedThreadId}"); // Main Thread

        // Unity API 사용 가능!
        transform.position = Vector3.one;
    }
}
```

### 기본 동작

| 상황 | SynchronizationContext | await 후 |
|------|------------------------|----------|
| Unity Main Thread | UnitySynchronizationContext | Main Thread로 복귀 |
| ThreadPool Thread | null | 아무 ThreadPool Thread에서 계속 |
| WPF/WinForms | DispatcherSynchronizationContext | UI Thread로 복귀 |

---

## ConfigureAwait 기본 사용법

### ConfigureAwait(true) - 기본값

```csharp
public class ConfigureAwaitTrueExample : MonoBehaviour
{
    async void Start()
    {
        Debug.Log($"Before: Thread {Thread.CurrentThread.ManagedThreadId}");

        // ConfigureAwait(true)는 기본 동작 (생략 가능)
        await Task.Delay(100).ConfigureAwait(true);

        // 캡처된 컨텍스트(Main Thread)로 복귀
        Debug.Log($"After: Thread {Thread.CurrentThread.ManagedThreadId}");

        // Unity API 안전하게 사용 가능
        GetComponent<Renderer>().material.color = Color.red;
    }
}
```

### ConfigureAwait(false) - 컨텍스트 무시

```csharp
public class ConfigureAwaitFalseExample : MonoBehaviour
{
    async void Start()
    {
        Debug.Log($"Before: Thread {Thread.CurrentThread.ManagedThreadId}");

        // ConfigureAwait(false): 컨텍스트 캡처 안 함
        await Task.Delay(100).ConfigureAwait(false);

        // ThreadPool의 아무 스레드에서 계속 실행 가능
        Debug.Log($"After: Thread {Thread.CurrentThread.ManagedThreadId}");

        // ⚠️ 위험! Main Thread가 아닐 수 있음
        // transform.position = Vector3.zero; // 런타임 에러 가능!
    }
}
```

---

## ConfigureAwait(false)의 용도

### 1. 성능 최적화

```csharp
public class PerformanceOptimization : MonoBehaviour
{
    // CPU 집약적 작업에서 ConfigureAwait(false) 사용
    async Task<byte[]> ProcessDataAsync(byte[] data)
    {
        // 첫 번째 await - 스레드풀로 이동
        await Task.Yield().ConfigureAwait(false);

        // 이후 모든 작업은 ThreadPool에서 실행
        // Main Thread로 복귀하는 오버헤드 없음
        var result = new byte[data.Length];

        for (int i = 0; i < data.Length; i++)
        {
            result[i] = (byte)(data[i] ^ 0xFF);

            // 추가 비동기 작업도 ThreadPool에서 계속
            if (i % 10000 == 0)
            {
                await Task.Yield().ConfigureAwait(false);
            }
        }

        return result;
    }

    async void Start()
    {
        var data = new byte[1000000];

        // 무거운 작업 호출
        var result = await ProcessDataAsync(data);

        // Start()에서는 ConfigureAwait 지정 안 함 → Main Thread로 복귀
        Debug.Log($"완료: {result.Length} bytes");
    }
}
```

### 2. 라이브러리 코드

```csharp
// 라이브러리나 유틸리티 코드에서는 ConfigureAwait(false) 사용
public static class AsyncUtilities
{
    // 라이브러리 코드: 호출자의 컨텍스트를 알 수 없음
    public static async Task<string> FetchDataAsync(string url)
    {
        using var client = new HttpClient();

        // 라이브러리에서는 항상 ConfigureAwait(false)
        var response = await client.GetAsync(url).ConfigureAwait(false);
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadAsStringAsync().ConfigureAwait(false);
    }

    public static async Task<T> RetryAsync<T>(
        Func<Task<T>> operation,
        int maxRetries = 3)
    {
        for (int i = 0; i < maxRetries; i++)
        {
            try
            {
                return await operation().ConfigureAwait(false);
            }
            catch when (i < maxRetries - 1)
            {
                await Task.Delay(1000 * (i + 1)).ConfigureAwait(false);
            }
        }

        return await operation().ConfigureAwait(false);
    }
}
```

### 3. 백그라운드 서비스

```csharp
public class BackgroundService
{
    private CancellationTokenSource _cts;

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        _cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);

        while (!_cts.Token.IsCancellationRequested)
        {
            try
            {
                // 백그라운드에서 계속 실행 - Main Thread 필요 없음
                await ProcessQueueAsync().ConfigureAwait(false);
                await Task.Delay(1000, _cts.Token).ConfigureAwait(false);
            }
            catch (OperationCanceledException)
            {
                break;
            }
        }
    }

    private async Task ProcessQueueAsync()
    {
        // 모든 작업이 ThreadPool에서 실행
        await Task.Yield().ConfigureAwait(false);
        // ... 처리 로직
    }
}
```

---

## Unity에서의 데드락 문제

### 데드락 시나리오

```csharp
public class DeadlockExample : MonoBehaviour
{
    void Start()
    {
        // ⚠️ 데드락 발생!
        var result = LoadDataAsync().Result; // .Result 또는 .Wait() 사용
        Debug.Log(result);
    }

    async Task<string> LoadDataAsync()
    {
        // Main Thread에서 호출되어 UnitySynchronizationContext 캡처
        await Task.Delay(1000); // ConfigureAwait(true)가 기본값

        // Main Thread로 복귀하려 하지만...
        // Main Thread는 .Result에서 블로킹 중!
        // → 데드락!

        return "Data";
    }
}
```

### 데드락 발생 원리

```
1. Main Thread: LoadDataAsync().Result 호출
   → Main Thread가 블로킹되어 대기

2. LoadDataAsync: await Task.Delay(1000) 실행
   → UnitySynchronizationContext 캡처
   → Task.Delay 완료 대기

3. Task.Delay 완료
   → UnitySynchronizationContext.Post() 호출
   → Main Thread에서 continuation 실행 요청

4. 데드락!
   → Main Thread는 .Result에서 블로킹 중
   → continuation은 Main Thread 필요
   → 서로 대기하며 영원히 멈춤
```

### 해결 방법 1: ConfigureAwait(false)

```csharp
public class DeadlockSolution1 : MonoBehaviour
{
    void Start()
    {
        // ConfigureAwait(false)로 데드락 방지
        var result = LoadDataAsync().Result;
        Debug.Log(result);
    }

    async Task<string> LoadDataAsync()
    {
        // Main Thread로 복귀하지 않음 → 데드락 없음
        await Task.Delay(1000).ConfigureAwait(false);
        return "Data";
    }
}
```

### 해결 방법 2: async 전파 (권장)

```csharp
public class DeadlockSolution2 : MonoBehaviour
{
    async void Start()
    {
        // 비동기로 호출 - 데드락 없음
        var result = await LoadDataAsync();
        Debug.Log(result);
    }

    async Task<string> LoadDataAsync()
    {
        await Task.Delay(1000); // 기본값(true) 사용 가능
        return "Data";
    }
}
```

### 해결 방법 3: Task.Run으로 우회

```csharp
public class DeadlockSolution3 : MonoBehaviour
{
    void Start()
    {
        // ThreadPool에서 실행 - 데드락 없음
        var result = Task.Run(() => LoadDataAsync()).Result;
        Debug.Log(result);
    }

    async Task<string> LoadDataAsync()
    {
        // Task.Run 내부에서는 SynchronizationContext가 null
        await Task.Delay(1000);
        return "Data";
    }
}
```

---

## ConfigureAwait 사용 가이드라인

### Unity 프로젝트에서의 권장 사항

```csharp
public class ConfigureAwaitGuidelines : MonoBehaviour
{
    // ✅ 좋음: MonoBehaviour 메서드에서는 기본값 사용
    async void Start()
    {
        var data = await LoadDataAsync();
        transform.position = new Vector3(data.x, data.y, data.z); // Unity API 사용
    }

    // ✅ 좋음: 내부 헬퍼 메서드에서 ConfigureAwait(false)
    private async Task<(float x, float y, float z)> LoadDataAsync()
    {
        var json = await FetchJsonAsync().ConfigureAwait(false);
        var parsed = ParseJson(json);
        return (parsed.x, parsed.y, parsed.z);
    }

    private async Task<string> FetchJsonAsync()
    {
        using var client = new HttpClient();
        return await client.GetStringAsync("https://api.example.com/data")
            .ConfigureAwait(false);
    }

    // ❌ 나쁨: ConfigureAwait(false) 후 Unity API 사용
    private async Task BadExample()
    {
        await Task.Delay(100).ConfigureAwait(false);
        // 위험! Main Thread가 아닐 수 있음
        // GetComponent<Renderer>().enabled = true;
    }
}
```

### 어디에 ConfigureAwait(false)를 사용해야 하는가?

| 코드 위치 | ConfigureAwait | 이유 |
|-----------|---------------|------|
| MonoBehaviour 메서드 | 기본값(true) | Unity API 사용 필요 |
| UI 업데이트 직전 | 기본값(true) | Main Thread 필요 |
| 순수 계산/처리 로직 | false | 성능 최적화 |
| 라이브러리/유틸리티 | false | 컨텍스트 독립적 |
| 백그라운드 서비스 | false | Main Thread 불필요 |

---

## 연속된 await에서의 ConfigureAwait

### 모든 await에 적용해야 함

```csharp
public class ChainedConfigureAwait : MonoBehaviour
{
    async Task ProcessAsync()
    {
        // 첫 번째만 ConfigureAwait(false)하면 안 됨!
        await Step1Async().ConfigureAwait(false);

        // 이후 await에도 모두 적용해야 함
        await Step2Async().ConfigureAwait(false);
        await Step3Async().ConfigureAwait(false);

        // 하나라도 빠지면 해당 지점에서 컨텍스트 복귀 시도
    }

    // 더 나은 방법: 메서드 전체를 ThreadPool에서 실행
    async Task ProcessAsyncBetter()
    {
        await Task.Run(async () =>
        {
            // ThreadPool 내부에서는 SynchronizationContext가 null
            // ConfigureAwait 불필요
            await Step1Async();
            await Step2Async();
            await Step3Async();
        });
    }
}
```

### ConfigureAwait 헬퍼 확장 메서드

```csharp
public static class ConfigureAwaitExtensions
{
    // 여러 Task에 한번에 ConfigureAwait(false) 적용
    public static ConfiguredTaskAwaitable[] ConfigureAwaitAll(
        this IEnumerable<Task> tasks,
        bool continueOnCapturedContext = false)
    {
        return tasks
            .Select(t => t.ConfigureAwait(continueOnCapturedContext))
            .ToArray();
    }

    // Task.WhenAll과 함께 사용
    public static async Task WhenAllNoContext(params Task[] tasks)
    {
        await Task.WhenAll(tasks).ConfigureAwait(false);
    }

    public static async Task<T[]> WhenAllNoContext<T>(params Task<T>[] tasks)
    {
        return await Task.WhenAll(tasks).ConfigureAwait(false);
    }
}

// 사용 예
public class ExtensionUsage : MonoBehaviour
{
    async Task Example()
    {
        var results = await ConfigureAwaitExtensions.WhenAllNoContext(
            LoadData1Async(),
            LoadData2Async(),
            LoadData3Async()
        );
    }
}
```

---

## UniTask에서의 컨텍스트 제어

### UniTask와 SynchronizationContext

```csharp
using Cysharp.Threading.Tasks;

public class UniTaskContextControl : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        Debug.Log($"Start: Thread {Thread.CurrentThread.ManagedThreadId}");

        // UniTask는 기본적으로 PlayerLoop 기반
        await UniTask.Delay(1000);

        Debug.Log($"After delay: Thread {Thread.CurrentThread.ManagedThreadId}");
        // Main Thread (PlayerLoop에서 실행)
    }
}
```

### SwitchToMainThread / SwitchToThreadPool

```csharp
using Cysharp.Threading.Tasks;

public class UniTaskThreadSwitching : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        Debug.Log($"1. Main Thread: {Thread.CurrentThread.ManagedThreadId}");

        // ThreadPool로 전환
        await UniTask.SwitchToThreadPool();
        Debug.Log($"2. ThreadPool: {Thread.CurrentThread.ManagedThreadId}");

        // 무거운 작업 수행
        await HeavyWorkAsync();

        // Main Thread로 복귀
        await UniTask.SwitchToMainThread();
        Debug.Log($"3. Main Thread: {Thread.CurrentThread.ManagedThreadId}");

        // Unity API 안전하게 사용
        transform.position = Vector3.zero;
    }

    async UniTask HeavyWorkAsync()
    {
        // ThreadPool에서 실행 중
        await UniTask.Delay(100);
        // 계속 ThreadPool에서 실행
    }
}
```

### PlayerLoopTiming으로 정밀 제어

```csharp
using Cysharp.Threading.Tasks;

public class PlayerLoopTimingExample : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // Update 직후에 실행
        await UniTask.Yield(PlayerLoopTiming.Update);
        Debug.Log("After Update");

        // FixedUpdate 직후에 실행
        await UniTask.Yield(PlayerLoopTiming.FixedUpdate);
        Debug.Log("After FixedUpdate");

        // 렌더링 직전에 실행
        await UniTask.Yield(PlayerLoopTiming.PreLateUpdate);
        Debug.Log("Before LateUpdate");
    }
}
```

---

## Unity 2023+ Awaitable과 컨텍스트

### Awaitable의 기본 동작

```csharp
// Unity 2023.1+
public class AwaitableContextExample : MonoBehaviour
{
    async void Start()
    {
        Debug.Log($"Start: Thread {Thread.CurrentThread.ManagedThreadId}");

        // Awaitable은 자동으로 Main Thread로 복귀
        await Awaitable.WaitForSecondsAsync(1f);

        Debug.Log($"After: Thread {Thread.CurrentThread.ManagedThreadId}");
        // Main Thread

        transform.position = Vector3.one; // 안전!
    }
}
```

### Awaitable.BackgroundThreadAsync

```csharp
// Unity 2023.1+
public class AwaitableBackgroundExample : MonoBehaviour
{
    async void Start()
    {
        Debug.Log($"1. Main: {Thread.CurrentThread.ManagedThreadId}");

        // 백그라운드 스레드로 전환
        await Awaitable.BackgroundThreadAsync();
        Debug.Log($"2. Background: {Thread.CurrentThread.ManagedThreadId}");

        // 무거운 작업
        var result = PerformHeavyCalculation();

        // Main Thread로 복귀
        await Awaitable.MainThreadAsync();
        Debug.Log($"3. Main: {Thread.CurrentThread.ManagedThreadId}");

        // 결과 적용
        UpdateUI(result);
    }

    int PerformHeavyCalculation()
    {
        // CPU 집약적 작업
        return 42;
    }

    void UpdateUI(int result)
    {
        // UI 업데이트
    }
}
```

---

## 실전 패턴

### 패턴 1: 데이터 로딩 + UI 업데이트

```csharp
public class DataLoadingPattern : MonoBehaviour
{
    [SerializeField] private Text statusText;
    [SerializeField] private Image progressBar;

    async void Start()
    {
        try
        {
            statusText.text = "Loading...";

            // 데이터 로딩 (내부에서 ConfigureAwait(false) 사용)
            var data = await LoadGameDataAsync();

            // 여기는 Main Thread - UI 업데이트 안전
            statusText.text = "Processing...";

            // 데이터 처리 (ThreadPool에서)
            var processed = await Task.Run(() => ProcessData(data));

            // 다시 Main Thread - UI 업데이트
            statusText.text = $"Loaded: {processed.ItemCount} items";
            progressBar.fillAmount = 1f;
        }
        catch (Exception ex)
        {
            statusText.text = $"Error: {ex.Message}";
        }
    }

    private async Task<byte[]> LoadGameDataAsync()
    {
        using var client = new HttpClient();

        // 라이브러리 코드 스타일로 ConfigureAwait(false) 사용
        var response = await client.GetAsync("https://api.game.com/data")
            .ConfigureAwait(false);

        return await response.Content.ReadAsByteArrayAsync()
            .ConfigureAwait(false);
    }

    private GameData ProcessData(byte[] data)
    {
        // CPU 집약적 처리
        return new GameData { ItemCount = data.Length };
    }
}

public class GameData
{
    public int ItemCount { get; set; }
}
```

### 패턴 2: 병렬 다운로드 with 진행률

```csharp
public class ParallelDownloadPattern : MonoBehaviour
{
    [SerializeField] private Slider progressSlider;

    private int _completedCount;
    private int _totalCount;

    async void Start()
    {
        var urls = new[]
        {
            "https://example.com/file1",
            "https://example.com/file2",
            "https://example.com/file3"
        };

        _totalCount = urls.Length;
        _completedCount = 0;

        // 병렬 다운로드 시작
        var downloads = urls.Select(url => DownloadFileAsync(url, OnFileCompleted));

        await Task.WhenAll(downloads);

        // 모든 다운로드 완료 - Main Thread
        Debug.Log("All downloads complete!");
    }

    private async Task DownloadFileAsync(string url, Action onComplete)
    {
        try
        {
            using var client = new HttpClient();

            // 다운로드 - ConfigureAwait(false)로 ThreadPool에서
            var data = await client.GetByteArrayAsync(url).ConfigureAwait(false);

            // 파일 저장도 ThreadPool에서
            var fileName = Path.GetFileName(new Uri(url).LocalPath);
            var path = Path.Combine(Application.persistentDataPath, fileName);
            await File.WriteAllBytesAsync(path, data).ConfigureAwait(false);

            // 완료 콜백 - 주의: 여전히 ThreadPool
            onComplete?.Invoke();
        }
        catch (Exception ex)
        {
            Debug.LogError($"Download failed: {url}, {ex.Message}");
        }
    }

    private void OnFileCompleted()
    {
        // Thread-safe하게 카운트 증가
        var completed = Interlocked.Increment(ref _completedCount);

        // UI 업데이트는 Main Thread에서 해야 함
        // UniTask 사용 시
        UniTask.Post(() =>
        {
            progressSlider.value = (float)completed / _totalCount;
        });
    }
}
```

### 패턴 3: 취소 가능한 백그라운드 작업

```csharp
public class CancellableBackgroundWork : MonoBehaviour
{
    private CancellationTokenSource _cts;

    async void Start()
    {
        _cts = new CancellationTokenSource();

        try
        {
            await ProcessInBackgroundAsync(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("Background work cancelled");
        }
    }

    void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }

    private async Task ProcessInBackgroundAsync(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            // 백그라운드에서 작업 수행
            await Task.Delay(1000, cancellationToken).ConfigureAwait(false);

            // 무거운 처리
            var result = await ComputeAsync(cancellationToken).ConfigureAwait(false);

            // 결과를 Main Thread로 전달
            await UpdateMainThreadAsync(result, cancellationToken);
        }
    }

    private async Task<int> ComputeAsync(CancellationToken cancellationToken)
    {
        // ThreadPool에서 실행
        await Task.Yield();
        cancellationToken.ThrowIfCancellationRequested();
        return 42;
    }

    private async Task UpdateMainThreadAsync(int result, CancellationToken cancellationToken)
    {
        // Main Thread로 전환
        await UniTask.SwitchToMainThread(cancellationToken);

        // UI 업데이트
        Debug.Log($"Result: {result}");
    }
}
```

---

## 일반적인 실수와 해결책

### 실수 1: ConfigureAwait(false) 후 Unity API 사용

```csharp
// ❌ 잘못된 코드
async Task WrongExample()
{
    await Task.Delay(100).ConfigureAwait(false);
    transform.position = Vector3.zero; // 런타임 에러!
}

// ✅ 올바른 코드
async Task CorrectExample()
{
    await Task.Delay(100).ConfigureAwait(false);

    // Main Thread로 복귀
    await UniTask.SwitchToMainThread();
    transform.position = Vector3.zero;
}
```

### 실수 2: 일부 await에만 ConfigureAwait(false) 적용

```csharp
// ❌ 비일관적
async Task InconsistentExample()
{
    await Task.Delay(100).ConfigureAwait(false);
    await Task.Delay(100); // 여기서 컨텍스트 복귀 시도
    await Task.Delay(100).ConfigureAwait(false);
}

// ✅ 일관적
async Task ConsistentExample()
{
    await Task.Delay(100).ConfigureAwait(false);
    await Task.Delay(100).ConfigureAwait(false);
    await Task.Delay(100).ConfigureAwait(false);
}
```

### 실수 3: .Result나 .Wait()으로 데드락

```csharp
// ❌ 데드락 위험
void DeadlockRisk()
{
    var result = SomeAsyncMethod().Result;
}

// ✅ async 전파
async void NoDeadlock()
{
    var result = await SomeAsyncMethod();
}

// ✅ 또는 전체 체인에 ConfigureAwait(false)
void AlsoSafe()
{
    var result = SomeAsyncMethodWithConfigureAwait().Result;
}

async Task<int> SomeAsyncMethodWithConfigureAwait()
{
    await Task.Delay(100).ConfigureAwait(false);
    return 42;
}
```

---

## 정리 및 체크리스트

### ConfigureAwait 결정 플로우차트

```
await 후 Unity API 사용?
    ├── Yes → ConfigureAwait 사용하지 않음 (기본값)
    └── No → 라이브러리/유틸리티 코드?
                ├── Yes → ConfigureAwait(false)
                └── No → 성능이 중요한 루프?
                            ├── Yes → ConfigureAwait(false)
                            └── No → 기본값 사용
```

### 체크리스트

- [ ] MonoBehaviour 메서드에서는 ConfigureAwait 생략 (기본값 사용)
- [ ] 라이브러리/유틸리티 코드에서는 ConfigureAwait(false) 사용
- [ ] .Result나 .Wait() 사용 지양, async 전파 권장
- [ ] ConfigureAwait(false) 후 Unity API 사용 금지
- [ ] 연속된 await에 일관되게 ConfigureAwait 적용
- [ ] UniTask 사용 시 SwitchToMainThread/SwitchToThreadPool 활용

---

## 참고 자료

- [ConfigureAwait FAQ - Stephen Cleary](https://devblogs.microsoft.com/dotnet/configureawait-faq/)
- [Async/Await Best Practices in .NET](https://docs.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/)
- [UniTask - SwitchTo methods](https://github.com/Cysharp/UniTask#switching-thread)
- [Unity 2023 Awaitable](https://docs.unity3d.com/2023.1/Documentation/ScriptReference/Awaitable.html)
