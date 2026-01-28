# 36. 생명주기 관리

## 개요

Unity에서 비동기 작업의 **생명주기 관리**는 매우 중요합니다. GameObject 파괴, Scene 전환, 앱 종료 시 진행 중인 작업을 안전하게 정리하지 않으면 예외, 메모리 누수, 크래시가 발생할 수 있습니다.

---

## Unity 생명주기 이벤트

### MonoBehaviour 생명주기

```
Awake → OnEnable → Start → Update → OnDisable → OnDestroy
                    ↑                    ↓
                    └──────────────────────┘ (재활성화)
```

### 비동기 작업과 생명주기

```csharp
public class LifecycleAwareness : MonoBehaviour
{
    private CancellationTokenSource _cts;
    private bool _isDestroyed;

    void Awake()
    {
        _cts = new CancellationTokenSource();
    }

    async void Start()
    {
        try
        {
            await ProcessAsync(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업 취소됨");
        }
    }

    async Task ProcessAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await Task.Delay(1000, ct);

            // ⚠️ 파괴 후 Unity API 호출 방지
            if (_isDestroyed) return;

            transform.position += Vector3.up;
        }
    }

    void OnDisable()
    {
        // 비활성화 시 작업 일시 중지 (선택적)
    }

    void OnDestroy()
    {
        _isDestroyed = true;
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

---

## destroyCancellationToken

### Unity 2022.2+ 내장 기능

```csharp
public class DestroyCancellationTokenExample : MonoBehaviour
{
    async void Start()
    {
        try
        {
            // MonoBehaviour의 destroyCancellationToken
            // GameObject 파괴 시 자동으로 취소됨
            await LongRunningOperationAsync(destroyCancellationToken);
        }
        catch (OperationCanceledException)
        {
            // 정상적인 취소
            Debug.Log("오브젝트 파괴로 작업 종료");
        }
    }

    async Task LongRunningOperationAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await Task.Delay(100, ct);
            Debug.Log("Working...");
        }
    }

    // OnDestroy에서 별도 취소 처리 불필요!
}
```

### Unity 2022.1 이하: 수동 구현

```csharp
public abstract class CancellableMonoBehaviour : MonoBehaviour
{
    private CancellationTokenSource _destroyCts;

    protected CancellationToken destroyCancellationToken
    {
        get
        {
            if (_destroyCts == null)
            {
                _destroyCts = new CancellationTokenSource();
            }
            return _destroyCts.Token;
        }
    }

    protected virtual void OnDestroy()
    {
        _destroyCts?.Cancel();
        _destroyCts?.Dispose();
        _destroyCts = null;
    }
}

// 사용
public class MyBehaviour : CancellableMonoBehaviour
{
    async void Start()
    {
        try
        {
            await DoWorkAsync(destroyCancellationToken);
        }
        catch (OperationCanceledException) { }
    }

    async Task DoWorkAsync(CancellationToken ct)
    {
        await Task.Delay(10000, ct);
    }
}
```

---

## DisposableBag 패턴

### 여러 리소스 관리

```csharp
public class DisposableBag : IDisposable
{
    private readonly List<IDisposable> _disposables = new List<IDisposable>();
    private bool _disposed;

    public void Add(IDisposable disposable)
    {
        if (_disposed)
        {
            disposable.Dispose();
            return;
        }
        _disposables.Add(disposable);
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;

        foreach (var disposable in _disposables)
        {
            try
            {
                disposable?.Dispose();
            }
            catch (Exception ex)
            {
                Debug.LogException(ex);
            }
        }
        _disposables.Clear();
    }
}

// 확장 메서드
public static class DisposableExtensions
{
    public static T AddTo<T>(this T disposable, DisposableBag bag) where T : IDisposable
    {
        bag.Add(disposable);
        return disposable;
    }
}

// 사용
public class DisposableBagExample : MonoBehaviour
{
    private DisposableBag _disposables = new DisposableBag();

    void Start()
    {
        // CancellationTokenSource
        var cts = new CancellationTokenSource().AddTo(_disposables);

        // 기타 IDisposable
        var stream = new MemoryStream().AddTo(_disposables);

        StartOperation(cts.Token);
    }

    async void StartOperation(CancellationToken ct)
    {
        try
        {
            await Task.Delay(10000, ct);
        }
        catch (OperationCanceledException) { }
    }

    void OnDestroy()
    {
        _disposables.Dispose();
    }
}
```

---

## Scene 전환 관리

### Scene 단위 취소

```csharp
public class SceneLifecycleManager : MonoBehaviour
{
    private static SceneLifecycleManager _instance;
    private CancellationTokenSource _sceneCts;

    public static CancellationToken SceneToken =>
        _instance?._sceneCts?.Token ?? CancellationToken.None;

    void Awake()
    {
        if (_instance != null)
        {
            Destroy(gameObject);
            return;
        }

        _instance = this;
        DontDestroyOnLoad(gameObject);

        _sceneCts = new CancellationTokenSource();
        SceneManager.sceneUnloaded += OnSceneUnloaded;
    }

    void OnSceneUnloaded(Scene scene)
    {
        // 이전 Scene의 작업 취소
        _sceneCts?.Cancel();
        _sceneCts?.Dispose();
        _sceneCts = new CancellationTokenSource();
    }

    void OnDestroy()
    {
        SceneManager.sceneUnloaded -= OnSceneUnloaded;
        _sceneCts?.Cancel();
        _sceneCts?.Dispose();
    }
}

// 사용
public class SceneAwareTask : MonoBehaviour
{
    async void Start()
    {
        try
        {
            // Scene 전환 시 자동 취소
            await LoadSceneDataAsync(SceneLifecycleManager.SceneToken);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("Scene 전환으로 작업 취소");
        }
    }

    async Task LoadSceneDataAsync(CancellationToken ct)
    {
        await Task.Delay(5000, ct);
    }
}
```

### 안전한 Scene 전환

```csharp
public class SafeSceneTransition : MonoBehaviour
{
    private CancellationTokenSource _transitionCts;

    public async Task TransitionToSceneAsync(string sceneName)
    {
        // 이전 전환 취소
        _transitionCts?.Cancel();
        _transitionCts?.Dispose();
        _transitionCts = new CancellationTokenSource();

        var ct = _transitionCts.Token;

        try
        {
            // 페이드 아웃
            await FadeOutAsync(ct);

            // Scene 로드
            var operation = SceneManager.LoadSceneAsync(sceneName);
            operation.allowSceneActivation = false;

            while (operation.progress < 0.9f)
            {
                ct.ThrowIfCancellationRequested();
                await Task.Yield();
            }

            operation.allowSceneActivation = true;

            // 로드 완료 대기
            while (!operation.isDone)
            {
                ct.ThrowIfCancellationRequested();
                await Task.Yield();
            }

            // 페이드 인
            await FadeInAsync(ct);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("Scene 전환 취소됨");
        }
    }

    async Task FadeOutAsync(CancellationToken ct)
    {
        await Task.Delay(500, ct);
    }

    async Task FadeInAsync(CancellationToken ct)
    {
        await Task.Delay(500, ct);
    }

    void OnDestroy()
    {
        _transitionCts?.Cancel();
        _transitionCts?.Dispose();
    }
}
```

---

## UniTask 통합

### GetCancellationTokenOnDestroy

```csharp
using Cysharp.Threading.Tasks;

public class UniTaskLifecycle : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // UniTask의 확장 메서드
        var ct = this.GetCancellationTokenOnDestroy();

        try
        {
            await ProcessAsync(ct);
        }
        catch (OperationCanceledException)
        {
            // 정상 종료
        }
    }

    async UniTask ProcessAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await UniTask.Delay(1000, cancellationToken: ct);
            Debug.Log("Processing...");
        }
    }
}
```

### CancellationTokenSource.RegisterTo

```csharp
using Cysharp.Threading.Tasks;

public class UniTaskCTSManagement : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // CTS를 GameObject에 등록 - 파괴 시 자동 취소 및 Dispose
        var cts = new CancellationTokenSource();
        cts.RegisterRaiseCancelOnDestroy(this);

        try
        {
            await LongOperationAsync(cts.Token);
        }
        catch (OperationCanceledException) { }
    }

    async UniTask LongOperationAsync(CancellationToken ct)
    {
        await UniTask.Delay(10000, cancellationToken: ct);
    }
}
```

---

## 실전 패턴

### 패턴 1: 안전한 비동기 컴포넌트

```csharp
public abstract class AsyncMonoBehaviour : MonoBehaviour
{
    private CancellationTokenSource _cts;
    private readonly List<Task> _runningTasks = new List<Task>();

    protected CancellationToken CancellationToken
    {
        get
        {
            if (_cts == null)
            {
                _cts = new CancellationTokenSource();
            }
            return _cts.Token;
        }
    }

    protected void TrackTask(Task task)
    {
        _runningTasks.Add(task);
        task.ContinueWith(t => _runningTasks.Remove(t));
    }

    protected async Task WaitForAllTasksAsync()
    {
        _cts?.Cancel();
        await Task.WhenAll(_runningTasks.ToArray());
    }

    protected virtual void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }
}

// 사용
public class SafeAsyncComponent : AsyncMonoBehaviour
{
    async void Start()
    {
        var task = ProcessAsync();
        TrackTask(task);
    }

    async Task ProcessAsync()
    {
        try
        {
            while (!CancellationToken.IsCancellationRequested)
            {
                await Task.Delay(1000, CancellationToken);
                Debug.Log("Processing...");
            }
        }
        catch (OperationCanceledException) { }
    }
}
```

### 패턴 2: 서비스 생명주기

```csharp
public interface IAsyncService : IDisposable
{
    Task InitializeAsync(CancellationToken ct);
    Task ShutdownAsync();
}

public class GameService : IAsyncService
{
    private CancellationTokenSource _serviceCts;
    private Task _backgroundTask;

    public async Task InitializeAsync(CancellationToken ct)
    {
        _serviceCts = new CancellationTokenSource();
        using var linked = CancellationTokenSource.CreateLinkedTokenSource(ct, _serviceCts.Token);

        // 초기화
        await LoadConfigAsync(linked.Token);

        // 백그라운드 작업 시작
        _backgroundTask = RunBackgroundLoopAsync(_serviceCts.Token);
    }

    async Task LoadConfigAsync(CancellationToken ct)
    {
        await Task.Delay(1000, ct);
    }

    async Task RunBackgroundLoopAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                await Task.Delay(5000, ct);
                // 주기적 작업
            }
            catch (OperationCanceledException)
            {
                break;
            }
        }
    }

    public async Task ShutdownAsync()
    {
        _serviceCts?.Cancel();

        if (_backgroundTask != null)
        {
            try
            {
                await _backgroundTask;
            }
            catch (OperationCanceledException) { }
        }
    }

    public void Dispose()
    {
        _serviceCts?.Cancel();
        _serviceCts?.Dispose();
    }
}

// 사용
public class ServiceManager : MonoBehaviour
{
    private List<IAsyncService> _services = new List<IAsyncService>();

    async void Start()
    {
        var gameService = new GameService();
        _services.Add(gameService);

        await gameService.InitializeAsync(destroyCancellationToken);
    }

    async void OnDestroy()
    {
        foreach (var service in _services)
        {
            await service.ShutdownAsync();
            service.Dispose();
        }
    }
}
```

### 패턴 3: 풀링된 비동기 작업

```csharp
public class AsyncTaskPool : MonoBehaviour
{
    private readonly Queue<Func<CancellationToken, Task>> _taskQueue =
        new Queue<Func<CancellationToken, Task>>();
    private CancellationTokenSource _cts;
    private int _runningCount;
    private const int MaxConcurrent = 4;

    void Start()
    {
        _cts = new CancellationTokenSource();
        ProcessQueueAsync(_cts.Token);
    }

    public void Enqueue(Func<CancellationToken, Task> taskFactory)
    {
        _taskQueue.Enqueue(taskFactory);
    }

    async void ProcessQueueAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            while (_runningCount < MaxConcurrent && _taskQueue.Count > 0)
            {
                var taskFactory = _taskQueue.Dequeue();
                RunTaskAsync(taskFactory, ct);
            }

            await Task.Yield();
        }
    }

    async void RunTaskAsync(Func<CancellationToken, Task> taskFactory, CancellationToken ct)
    {
        _runningCount++;
        try
        {
            await taskFactory(ct);
        }
        catch (OperationCanceledException) { }
        catch (Exception ex)
        {
            Debug.LogException(ex);
        }
        finally
        {
            _runningCount--;
        }
    }

    void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

---

## 주의사항

### 파괴 후 Unity API 호출

```csharp
public class SafeUnityAPICall : MonoBehaviour
{
    private bool _isDestroyed;

    async void Start()
    {
        try
        {
            await Task.Delay(1000, destroyCancellationToken);

            // ⚠️ 항상 파괴 여부 확인
            if (_isDestroyed) return;

            transform.position = Vector3.zero;
        }
        catch (OperationCanceledException) { }
    }

    void OnDestroy()
    {
        _isDestroyed = true;
    }
}

// 더 안전한 방법: null 체크
public class SafeWithNullCheck : MonoBehaviour
{
    async void Start()
    {
        try
        {
            await Task.Delay(1000, destroyCancellationToken);

            // gameObject가 null이면 파괴됨
            if (this == null || gameObject == null) return;

            transform.position = Vector3.zero;
        }
        catch (OperationCanceledException) { }
    }
}
```

### 메모리 누수 방지

```csharp
public class PreventMemoryLeak : MonoBehaviour
{
    private CancellationTokenSource _cts;
    private HttpClient _client;

    void Start()
    {
        _cts = new CancellationTokenSource();
        _client = new HttpClient();

        _ = FetchDataAsync(_cts.Token);
    }

    async Task FetchDataAsync(CancellationToken ct)
    {
        try
        {
            // using으로 리소스 관리
            using var response = await _client.GetAsync("https://api.example.com", ct);
            var data = await response.Content.ReadAsStringAsync();
        }
        catch (OperationCanceledException) { }
        catch (Exception ex)
        {
            Debug.LogException(ex);
        }
    }

    void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _client?.Dispose();
    }
}
```

---

## 정리

### 생명주기 관리 요약

| 상황 | 권장 방법 |
|------|-----------|
| **GameObject 파괴** | destroyCancellationToken |
| **Scene 전환** | Scene 단위 CTS |
| **앱 종료** | ApplicationQuit 이벤트 |
| **여러 리소스** | DisposableBag |

### 체크리스트

- [ ] CancellationTokenSource를 OnDestroy에서 Cancel + Dispose 하는가?
- [ ] destroyCancellationToken 사용하는가? (Unity 2022.2+)
- [ ] 파괴 후 Unity API 호출을 방지하는가?
- [ ] Scene 전환 시 작업을 취소하는가?
- [ ] IDisposable 리소스를 정리하는가?

---

## 참고 자료

- [Unity MonoBehaviour Lifecycle](https://docs.unity3d.com/Manual/ExecutionOrder.html)
- [CancellationToken Best Practices](https://docs.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads)
- [UniTask Cancellation](https://github.com/Cysharp/UniTask#cancellation-and-exception-handling)
