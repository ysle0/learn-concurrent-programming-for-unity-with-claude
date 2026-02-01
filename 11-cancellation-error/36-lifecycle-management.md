# 36. 생명주기 관리

## 개요

Unity에서 비동기 작업을 사용할 때 가장 빈번하게 발생하는 문제 중 하나는 **오브젝트 생명주기와 비동기 작업의 불일치**입니다. MonoBehaviour가 파괴되었는데 비동기 작업이 계속 실행되거나, 씬이 전환되었는데 이전 씬의 구독이 남아있는 경우가 대표적입니다.

생명주기 관리란, 비동기 작업(Task, UniTask, Observable 구독 등)이 **소유 오브젝트의 생존 범위를 벗어나지 않도록** 체계적으로 관리하는 것을 의미합니다.

```
[Unity 오브젝트 생명주기와 비동기 작업]

Awake → Start → 비동기 작업 시작 ─────────────────→ 작업 완료
                     │                                    │
                     │    OnDestroy (오브젝트 파괴)        │
                     │        ↓                           │
                     │    ❌ 작업이 아직 실행 중!          │
                     │    → NullReferenceException         │
                     │    → MissingReferenceException      │
                     │                                    │
                     │    ✅ CancellationToken으로 취소     │
                     │    → 안전한 정리                     │
```

---

## 1. Unity 오브젝트 생명주기와 비동기 작업의 문제

### 문제가 발생하는 전형적인 패턴

```csharp
using UnityEngine;
using System.Threading.Tasks;

// ❌ 잘못된 패턴: 생명주기를 고려하지 않은 비동기 작업
public class BadAsyncExample : MonoBehaviour
{
    private async void Start()
    {
        // 3초 후 데이터 로드
        string data = await LoadDataAsync();

        // 이 시점에 오브젝트가 이미 파괴되었을 수 있음!
        // MissingReferenceException 발생 가능
        GetComponent<TextMesh>().text = data;
    }

    private async Task<string> LoadDataAsync()
    {
        await Task.Delay(3000);
        return "Loaded Data";
    }

    // 사용자가 3초 이내에 씬을 전환하면?
    // → Start의 await 이후 코드가 파괴된 오브젝트에 접근
}
```

### 생명주기 불일치로 인한 주요 문제들

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class LifecycleProblemsExample : MonoBehaviour
{
    // =============================================
    // 문제 1: 파괴된 오브젝트 접근
    // =============================================

    // ❌ 오브젝트 파괴 후 컴포넌트 접근 시도
    private async void StartWithoutGuard()
    {
        await Task.Delay(5000);

        // this가 파괴되었을 수 있음
        if (this == null) return; // Unity에서 null 체크는 가능하지만 불안정

        transform.position = Vector3.zero; // MissingReferenceException 위험
    }

    // =============================================
    // 문제 2: 메모리 누수
    // =============================================

    // ❌ 이벤트 구독 해제 없이 오브젝트 파괴
    private void SubscribeWithoutCleanup()
    {
        // 정적 이벤트에 인스턴스 메서드 구독
        // → 오브젝트가 파괴되어도 GC 수집 불가 (메모리 누수)
        GameEvents.OnScoreChanged += HandleScoreChanged;
    }

    private void HandleScoreChanged(int score)
    {
        Debug.Log($"Score: {score}");
    }

    // =============================================
    // 문제 3: 백그라운드 스레드에서 Unity API 접근
    // =============================================

    // ❌ 비동기 작업에서 메인 스레드 컨텍스트 손실
    private async void LoadAndApplyAsync()
    {
        // ConfigureAwait(false)로 스레드 컨텍스트를 잃은 경우
        var data = await Task.Run(() => HeavyComputation());

        // 메인 스레드가 아닐 수 있음!
        // UnityException: ... can only be called from the main thread
        gameObject.SetActive(true);
    }

    private int HeavyComputation() => 42;
}

// 예시용 정적 이벤트 클래스
public static class GameEvents
{
    public static event Action<int> OnScoreChanged;
}
```

---

## 2. OnDestroy에서의 취소 패턴

### 전통적인 CancellationTokenSource 패턴

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

// ✅ CancellationTokenSource를 사용한 기본 취소 패턴
public class ManualCancellationExample : MonoBehaviour
{
    private CancellationTokenSource _cts;

    private void Awake()
    {
        _cts = new CancellationTokenSource();
    }

    private async void Start()
    {
        try
        {
            await PeriodicUpdateAsync(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("비동기 작업이 취소되었습니다.");
        }
    }

    private async Task PeriodicUpdateAsync(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            // 주기적으로 서버에서 데이터 가져오기
            await FetchDataAsync(token);
            await Task.Delay(5000, token);
        }
    }

    private async Task FetchDataAsync(CancellationToken token)
    {
        // 네트워크 요청 시뮬레이션
        await Task.Delay(1000, token);
        Debug.Log("데이터 갱신 완료");
    }

    private void OnDestroy()
    {
        // 오브젝트 파괴 시 모든 비동기 작업 취소
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

### Unity 2022.2+ destroyCancellationToken

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

// ✅ Unity 2022.2+ 내장 destroyCancellationToken 활용
public class DestroyCancellationTokenExample : MonoBehaviour
{
    // Unity 2022.2부터 MonoBehaviour에 내장된 CancellationToken
    // OnDestroy 시 자동으로 취소됨 → 직접 CancellationTokenSource를 관리할 필요 없음

    private async void Start()
    {
        try
        {
            // destroyCancellationToken은 MonoBehaviour의 내장 속성
            await LongRunningOperationAsync(destroyCancellationToken);
        }
        catch (OperationCanceledException)
        {
            // 오브젝트 파괴 시 자동으로 여기로 진입
            Debug.Log("오브젝트 파괴로 인해 작업이 취소되었습니다.");
        }
    }

    private async Task LongRunningOperationAsync(CancellationToken token)
    {
        for (int i = 0; i < 100; i++)
        {
            token.ThrowIfCancellationRequested();

            await Task.Delay(1000, token);
            Debug.Log($"진행 중: {i + 1}/100");
        }
    }
}

// ✅ destroyCancellationToken과 추가 취소 조건을 결합하는 패턴
public class LinkedCancellationExample : MonoBehaviour
{
    public async void StartOperation(CancellationToken externalToken)
    {
        // destroyCancellationToken과 외부 토큰을 결합
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            destroyCancellationToken,
            externalToken
        );

        try
        {
            await DoWorkAsync(linkedCts.Token);
        }
        catch (OperationCanceledException)
        {
            if (destroyCancellationToken.IsCancellationRequested)
                Debug.Log("오브젝트 파괴로 취소");
            else if (externalToken.IsCancellationRequested)
                Debug.Log("외부 요청으로 취소");
        }
    }

    private async Task DoWorkAsync(CancellationToken token)
    {
        await Task.Delay(10000, token);
    }
}
```

### UniTask에서의 destroyCancellationToken 활용

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using System.Threading;

// ✅ UniTask + destroyCancellationToken 조합
public class UniTaskDestroyCancellationExample : MonoBehaviour
{
    private async void Start()
    {
        // UniTask는 destroyCancellationToken과 자연스럽게 통합
        await RepeatActionAsync(destroyCancellationToken);
    }

    private async UniTask RepeatActionAsync(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            Debug.Log("반복 작업 실행 중...");

            // UniTask의 Delay는 CancellationToken을 직접 지원
            bool cancelled = await UniTask.Delay(
                1000,
                cancellationToken: token
            ).SuppressCancellationThrow();

            if (cancelled) break;
        }
    }
}

// ✅ GetCancellationTokenOnDestroy() (Unity 2022.2 이전 버전 호환)
public class LegacyCancellationExample : MonoBehaviour
{
    private async void Start()
    {
        // UniTask의 확장 메서드: 모든 Unity 버전에서 사용 가능
        var token = this.GetCancellationTokenOnDestroy();

        await UniTask.Delay(5000, cancellationToken: token);
        Debug.Log("작업 완료!");
    }
}
```

---

## 3. Scene 전환 시 비동기 작업 정리

### 씬 이벤트 기반 정리

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

// ✅ 씬 전환 시 비동기 작업을 체계적으로 정리하는 매니저
public class SceneLifecycleManager : MonoBehaviour
{
    private static SceneLifecycleManager _instance;
    private CancellationTokenSource _sceneCts;
    private readonly List<IDisposable> _sceneDisposables = new List<IDisposable>();

    // 현재 씬에 묶인 CancellationToken
    public static CancellationToken SceneCancellationToken =>
        _instance?._sceneCts?.Token ?? CancellationToken.None;

    private void Awake()
    {
        if (_instance != null && _instance != this)
        {
            Destroy(gameObject);
            return;
        }

        _instance = this;
        DontDestroyOnLoad(gameObject);

        // 씬 관련 이벤트 등록
        SceneManager.sceneUnloaded += OnSceneUnloaded;
        SceneManager.sceneLoaded += OnSceneLoaded;

        CreateNewSceneCts();
    }

    private void OnSceneLoaded(Scene scene, LoadSceneMode mode)
    {
        // Additive 로딩이 아닌 경우에만 새 토큰 생성
        if (mode == LoadSceneMode.Single)
        {
            CreateNewSceneCts();
            Debug.Log($"[SceneLifecycle] 씬 '{scene.name}' 로드됨. 새 CancellationToken 생성.");
        }
    }

    private void OnSceneUnloaded(Scene scene)
    {
        Debug.Log($"[SceneLifecycle] 씬 '{scene.name}' 언로드됨. 비동기 작업 정리 중...");

        // 현재 씬의 모든 비동기 작업 취소
        _sceneCts?.Cancel();
        _sceneCts?.Dispose();

        // 등록된 Disposable 정리
        foreach (var disposable in _sceneDisposables)
        {
            disposable?.Dispose();
        }
        _sceneDisposables.Clear();
    }

    private void CreateNewSceneCts()
    {
        _sceneCts = new CancellationTokenSource();
    }

    // 외부에서 씬 범위의 Disposable을 등록
    public static void RegisterDisposable(IDisposable disposable)
    {
        _instance?._sceneDisposables.Add(disposable);
    }

    private void OnDestroy()
    {
        SceneManager.sceneUnloaded -= OnSceneUnloaded;
        SceneManager.sceneLoaded -= OnSceneLoaded;
        _sceneCts?.Cancel();
        _sceneCts?.Dispose();
    }
}
```

### 씬 전환 시 비동기 작업 사용 예제

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

// ✅ 씬 범위의 CancellationToken을 활용하는 컴포넌트
public class SceneAwareComponent : MonoBehaviour
{
    private async void Start()
    {
        // 씬 전환 시 자동으로 취소되는 토큰 사용
        var token = SceneLifecycleManager.SceneCancellationToken;

        try
        {
            await PeriodicSaveAsync(token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("씬 전환으로 자동 저장 중단됨.");
        }
    }

    private async Task PeriodicSaveAsync(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            await Task.Delay(10000, token); // 10초마다 자동 저장
            SaveProgress();
        }
    }

    private void SaveProgress()
    {
        Debug.Log("진행 상황 저장 완료");
    }
}
```

---

## 4. DisposableBag 패턴

### R3에서의 DisposableBag

```csharp
using R3;
using R3.Triggers;
using UnityEngine;

// ✅ R3의 DisposableBag를 이용한 깔끔한 구독 관리
public class R3DisposableBagExample : MonoBehaviour
{
    private DisposableBag _disposableBag;

    private void Start()
    {
        _disposableBag = new DisposableBag();

        // 여러 구독을 DisposableBag에 등록
        Observable.EveryUpdate()
            .Subscribe(_ => UpdateUI())
            .AddTo(ref _disposableBag);

        Observable.Timer(TimeSpan.FromSeconds(5))
            .Subscribe(_ => Debug.Log("5초 경과"))
            .AddTo(ref _disposableBag);

        Observable.Interval(TimeSpan.FromSeconds(1))
            .Subscribe(count => Debug.Log($"Tick: {count}"))
            .AddTo(ref _disposableBag);
    }

    private void UpdateUI()
    {
        // UI 갱신 로직
    }

    private void OnDestroy()
    {
        // 한 번의 호출로 모든 구독 해제
        _disposableBag.Dispose();
    }
}

// ✅ R3의 AddTo(MonoBehaviour) 확장 메서드
public class R3AddToExample : MonoBehaviour
{
    private void Start()
    {
        // AddTo(this)를 사용하면 MonoBehaviour 파괴 시 자동 정리
        Observable.Interval(TimeSpan.FromSeconds(1))
            .Subscribe(count => Debug.Log($"카운트: {count}"))
            .AddTo(this); // OnDestroy 시 자동 해제

        Observable.EveryUpdate()
            .Where(_ => Input.GetKeyDown(KeyCode.Space))
            .Subscribe(_ => Debug.Log("스페이스 키 눌림"))
            .AddTo(this); // OnDestroy 시 자동 해제
    }
    // OnDestroy에서 별도 정리 불필요!
}
```

### UniRx에서의 CompositeDisposable

```csharp
using UniRx;
using UnityEngine;
using System;

// ✅ UniRx의 CompositeDisposable 패턴
public class UniRxDisposableExample : MonoBehaviour
{
    private readonly CompositeDisposable _compositeDisposable = new CompositeDisposable();

    private void Start()
    {
        // 구독을 CompositeDisposable에 추가
        Observable.EveryUpdate()
            .Subscribe(_ => HandleUpdate())
            .AddTo(_compositeDisposable);

        Observable.Interval(TimeSpan.FromSeconds(2))
            .Subscribe(x => Debug.Log($"Interval: {x}"))
            .AddTo(_compositeDisposable);

        // 또는 AddTo(this)로 자동 관리
        Observable.Timer(TimeSpan.FromSeconds(10))
            .Subscribe(_ => Debug.Log("10초 타이머"))
            .AddTo(this);
    }

    private void HandleUpdate()
    {
        // 매 프레임 로직
    }

    private void OnDestroy()
    {
        _compositeDisposable.Dispose();
    }
}
```

---

## 5. IDisposable 구현과 비동기 리소스 정리

### 기본 IDisposable 패턴

```csharp
using UnityEngine;
using System;
using System.Threading;

// ✅ IDisposable을 올바르게 구현하는 비동기 서비스
public class AsyncDataService : IDisposable
{
    private CancellationTokenSource _cts;
    private bool _disposed;
    private Timer _refreshTimer;

    public AsyncDataService()
    {
        _cts = new CancellationTokenSource();
        _refreshTimer = new Timer(
            callback: _ => RefreshData(),
            state: null,
            dueTime: TimeSpan.Zero,
            period: TimeSpan.FromSeconds(30)
        );
    }

    public CancellationToken Token => _cts.Token;

    private void RefreshData()
    {
        if (_disposed) return;
        Debug.Log("데이터 갱신 중...");
    }

    // IDisposable 구현
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // 관리되는 리소스 정리
            _cts?.Cancel();
            _cts?.Dispose();
            _cts = null;

            _refreshTimer?.Dispose();
            _refreshTimer = null;
        }

        _disposed = true;
    }

    ~AsyncDataService()
    {
        Dispose(false);
    }
}
```

### MonoBehaviour에서 IDisposable 서비스 관리

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

// ✅ MonoBehaviour가 IDisposable 서비스들의 생명주기를 관리
public class ServiceManagerComponent : MonoBehaviour
{
    private readonly List<IDisposable> _services = new List<IDisposable>();

    private AsyncDataService _dataService;
    private AsyncCacheService _cacheService;

    private void Awake()
    {
        _dataService = new AsyncDataService();
        _cacheService = new AsyncCacheService();

        RegisterService(_dataService);
        RegisterService(_cacheService);
    }

    private void RegisterService(IDisposable service)
    {
        _services.Add(service);
    }

    private async void Start()
    {
        try
        {
            // 서비스의 CancellationToken을 사용하여 비동기 작업 수행
            await LoadInitialDataAsync(_dataService.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("초기 데이터 로드가 취소되었습니다.");
        }
    }

    private async Task LoadInitialDataAsync(System.Threading.CancellationToken token)
    {
        await Task.Delay(2000, token);
        Debug.Log("초기 데이터 로드 완료");
    }

    private void OnDestroy()
    {
        // 등록된 모든 서비스 정리 (역순)
        for (int i = _services.Count - 1; i >= 0; i--)
        {
            try
            {
                _services[i]?.Dispose();
            }
            catch (Exception ex)
            {
                Debug.LogError($"서비스 Dispose 중 오류: {ex.Message}");
            }
        }
        _services.Clear();
    }
}

// 예시 캐시 서비스
public class AsyncCacheService : IDisposable
{
    private bool _disposed;

    public void Dispose()
    {
        if (_disposed) return;
        Debug.Log("캐시 서비스 정리 완료");
        _disposed = true;
    }
}
```

---

## 6. IAsyncDisposable 패턴

### IAsyncDisposable 기본 구현

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

// ✅ IAsyncDisposable: 비동기 정리가 필요한 리소스
public class AsyncConnectionManager : IAsyncDisposable, IDisposable
{
    private CancellationTokenSource _cts;
    private bool _disposed;
    private bool _isConnected;

    public AsyncConnectionManager()
    {
        _cts = new CancellationTokenSource();
    }

    public async Task ConnectAsync(CancellationToken token = default)
    {
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            _cts.Token, token);

        // 연결 시뮬레이션
        await Task.Delay(1000, linkedCts.Token);
        _isConnected = true;
        Debug.Log("연결 성공");
    }

    // =============================================
    // IAsyncDisposable 구현
    // 비동기 정리 작업이 필요한 경우 사용
    // =============================================
    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;

        if (_isConnected)
        {
            try
            {
                // 서버에 정상적인 종료 알림 (graceful shutdown)
                await SendDisconnectMessageAsync();
                Debug.Log("정상적으로 연결 종료됨");
            }
            catch (Exception ex)
            {
                Debug.LogWarning($"연결 종료 중 오류: {ex.Message}");
            }
        }

        _cts?.Cancel();
        _cts?.Dispose();
        _cts = null;
        _disposed = true;
        _isConnected = false;

        GC.SuppressFinalize(this);
    }

    // 동기 Dispose도 함께 구현 (폴백용)
    public void Dispose()
    {
        if (_disposed) return;

        _cts?.Cancel();
        _cts?.Dispose();
        _cts = null;
        _disposed = true;
        _isConnected = false;

        GC.SuppressFinalize(this);
    }

    private async Task SendDisconnectMessageAsync()
    {
        await Task.Delay(500); // 종료 메시지 전송 시뮬레이션
    }
}
```

### await using 패턴

```csharp
using System;
using System.Threading.Tasks;
using UnityEngine;

public class AsyncDisposableUsageExample : MonoBehaviour
{
    // ✅ await using으로 비동기 리소스 자동 정리
    private async Task UseConnectionAsync()
    {
        await using var connection = new AsyncConnectionManager();

        await connection.ConnectAsync(destroyCancellationToken);

        // 연결을 사용하는 작업...
        await Task.Delay(5000, destroyCancellationToken);

        // 블록을 벗어나면 DisposeAsync()가 자동 호출됨
    }

    // ✅ 여러 비동기 리소스를 순차적으로 정리
    private async Task UseMultipleResourcesAsync()
    {
        await using var db = new AsyncDatabaseConnection();
        await using var cache = new AsyncCacheConnection();

        await db.OpenAsync(destroyCancellationToken);
        await cache.OpenAsync(destroyCancellationToken);

        // 작업 수행...

        // cache → db 순서로 DisposeAsync 호출 (역순)
    }

    private async void Start()
    {
        try
        {
            await UseConnectionAsync();
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 취소되었습니다.");
        }
    }
}

// 예시: 비동기 데이터베이스 연결
public class AsyncDatabaseConnection : IAsyncDisposable
{
    public async Task OpenAsync(System.Threading.CancellationToken token = default)
    {
        await Task.Delay(500, token);
        Debug.Log("DB 연결 열림");
    }

    public async ValueTask DisposeAsync()
    {
        await Task.Delay(200);
        Debug.Log("DB 연결 닫힘");
    }
}

// 예시: 비동기 캐시 연결
public class AsyncCacheConnection : IAsyncDisposable
{
    public async Task OpenAsync(System.Threading.CancellationToken token = default)
    {
        await Task.Delay(300, token);
        Debug.Log("캐시 연결 열림");
    }

    public async ValueTask DisposeAsync()
    {
        await Task.Delay(100);
        Debug.Log("캐시 연결 닫힘");
    }
}
```

---

## 7. SafeFireAndForget 패턴

### 기본 SafeFireAndForget

```csharp
using System;
using System.Threading.Tasks;
using UnityEngine;

// ✅ Fire-and-forget 비동기 작업을 안전하게 실행하는 확장 메서드
public static class TaskExtensions
{
    /// <summary>
    /// 비동기 작업을 fire-and-forget으로 실행하되, 예외를 안전하게 처리합니다.
    /// async void의 위험성을 완화하는 패턴입니다.
    /// </summary>
    public static async void SafeFireAndForget(
        this Task task,
        Action<Exception> onException = null,
        bool continueOnCapturedContext = true)
    {
        try
        {
            await task.ConfigureAwait(continueOnCapturedContext);
        }
        catch (OperationCanceledException)
        {
            // 취소는 정상 동작이므로 무시
        }
        catch (Exception ex)
        {
            if (onException != null)
                onException.Invoke(ex);
            else
                Debug.LogException(ex);
        }
    }

    // ValueTask 버전
    public static async void SafeFireAndForget(
        this ValueTask task,
        Action<Exception> onException = null,
        bool continueOnCapturedContext = true)
    {
        try
        {
            await task.ConfigureAwait(continueOnCapturedContext);
        }
        catch (OperationCanceledException)
        {
            // 취소는 무시
        }
        catch (Exception ex)
        {
            if (onException != null)
                onException.Invoke(ex);
            else
                Debug.LogException(ex);
        }
    }
}
```

### 사용 예제

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class SafeFireAndForgetExample : MonoBehaviour
{
    // ❌ async void는 예외가 전파되지 않아 위험
    private async void DangerousMethod()
    {
        await Task.Delay(1000);
        throw new InvalidOperationException("이 예외는 추적이 어렵습니다!");
    }

    // ✅ SafeFireAndForget으로 안전하게 실행
    private void Start()
    {
        // 기본: 예외 발생 시 Debug.LogException으로 출력
        LoadDataAsync(destroyCancellationToken).SafeFireAndForget();

        // 커스텀 에러 핸들러 지정
        SendAnalyticsAsync().SafeFireAndForget(
            onException: ex => Debug.LogWarning($"Analytics 전송 실패: {ex.Message}")
        );

        // 여러 fire-and-forget 작업 실행
        PreloadResourcesAsync(destroyCancellationToken).SafeFireAndForget();
        WarmUpCacheAsync(destroyCancellationToken).SafeFireAndForget();
    }

    private async Task LoadDataAsync(CancellationToken token)
    {
        await Task.Delay(2000, token);
        Debug.Log("데이터 로드 완료");
    }

    private async Task SendAnalyticsAsync()
    {
        await Task.Delay(1000);
        Debug.Log("Analytics 전송 완료");
    }

    private async Task PreloadResourcesAsync(CancellationToken token)
    {
        await Task.Delay(3000, token);
        Debug.Log("리소스 프리로드 완료");
    }

    private async Task WarmUpCacheAsync(CancellationToken token)
    {
        await Task.Delay(1500, token);
        Debug.Log("캐시 워밍업 완료");
    }
}
```

### UniTask의 Forget 메서드

```csharp
using Cysharp.Threading.Tasks;
using UnityEngine;
using System;

public class UniTaskForgetExample : MonoBehaviour
{
    private void Start()
    {
        // ✅ UniTask의 Forget(): 예외를 UniTaskScheduler로 전달
        LoadAsync(destroyCancellationToken).Forget();

        // ✅ SuppressCancellationThrow()와 결합
        LoadWithCancellationAsync(destroyCancellationToken).Forget();
    }

    private async UniTask LoadAsync(System.Threading.CancellationToken token)
    {
        await UniTask.Delay(2000, cancellationToken: token);
        Debug.Log("로드 완료");
    }

    private async UniTask LoadWithCancellationAsync(System.Threading.CancellationToken token)
    {
        bool cancelled = await UniTask.Delay(
            2000,
            cancellationToken: token
        ).SuppressCancellationThrow();

        if (!cancelled)
        {
            Debug.Log("로드 완료");
        }
    }
}
```

---

## 8. DontDestroyOnLoad와 비동기 작업

### 영속 오브젝트의 비동기 작업 관리

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

// ✅ DontDestroyOnLoad 오브젝트의 생명주기 관리
public class PersistentAsyncManager : MonoBehaviour
{
    private static PersistentAsyncManager _instance;

    // 앱 전체 수명의 CancellationTokenSource
    private CancellationTokenSource _appLifetimeCts;

    // 개별 작업 취소를 위한 CancellationTokenSource
    private CancellationTokenSource _currentOperationCts;

    public static PersistentAsyncManager Instance => _instance;

    private void Awake()
    {
        if (_instance != null && _instance != this)
        {
            Destroy(gameObject);
            return;
        }

        _instance = this;
        DontDestroyOnLoad(gameObject);

        _appLifetimeCts = new CancellationTokenSource();
    }

    /// <summary>
    /// 씬 전환과 무관하게 지속되는 비동기 작업 시작
    /// </summary>
    public async Task StartPersistentOperationAsync()
    {
        try
        {
            await HeartbeatLoopAsync(_appLifetimeCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("영속 작업이 종료되었습니다.");
        }
    }

    /// <summary>
    /// 현재 씬에서만 유효한 작업 시작 (씬 전환 시 취소 가능)
    /// </summary>
    public async Task StartSceneScopedOperationAsync(CancellationToken sceneToken)
    {
        // 앱 수명 토큰과 씬 토큰을 결합
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            _appLifetimeCts.Token,
            sceneToken
        );

        try
        {
            await SceneScopedWorkAsync(linkedCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("씬 범위 작업이 취소되었습니다.");
        }
    }

    /// <summary>
    /// 진행 중인 개별 작업만 취소 (새 작업으로 교체 시)
    /// </summary>
    public async Task ReplaceCurrentOperationAsync()
    {
        // 기존 작업 취소
        _currentOperationCts?.Cancel();
        _currentOperationCts?.Dispose();

        _currentOperationCts = CancellationTokenSource.CreateLinkedTokenSource(
            _appLifetimeCts.Token
        );

        try
        {
            await SomeOperationAsync(_currentOperationCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 교체되었습니다.");
        }
    }

    private async Task HeartbeatLoopAsync(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            Debug.Log("Heartbeat...");
            await Task.Delay(30000, token);
        }
    }

    private async Task SceneScopedWorkAsync(CancellationToken token)
    {
        await Task.Delay(5000, token);
        Debug.Log("씬 범위 작업 완료");
    }

    private async Task SomeOperationAsync(CancellationToken token)
    {
        await Task.Delay(3000, token);
        Debug.Log("작업 완료");
    }

    private void OnDestroy()
    {
        _appLifetimeCts?.Cancel();
        _appLifetimeCts?.Dispose();
        _currentOperationCts?.Cancel();
        _currentOperationCts?.Dispose();
    }
}
```

### DontDestroyOnLoad + 씬 인식 패턴

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using System;
using System.Threading;
using System.Threading.Tasks;

// ✅ 씬 변경을 인식하는 영속 매니저
public class SceneAwarePersistentManager : MonoBehaviour
{
    private CancellationTokenSource _sceneCts;

    private void Awake()
    {
        DontDestroyOnLoad(gameObject);
        SceneManager.activeSceneChanged += OnActiveSceneChanged;
        CreateSceneCts();
    }

    private void OnActiveSceneChanged(Scene oldScene, Scene newScene)
    {
        Debug.Log($"[PersistentManager] 씬 전환: {oldScene.name} → {newScene.name}");

        // 이전 씬의 작업 취소
        _sceneCts?.Cancel();
        _sceneCts?.Dispose();

        // 새 씬용 토큰 생성
        CreateSceneCts();

        // 새 씬에 맞는 작업 시작
        OnNewSceneLoaded(newScene.name, _sceneCts.Token);
    }

    private void CreateSceneCts()
    {
        _sceneCts = new CancellationTokenSource();
    }

    private async void OnNewSceneLoaded(string sceneName, CancellationToken token)
    {
        try
        {
            switch (sceneName)
            {
                case "MainMenu":
                    await LoadMenuDataAsync(token);
                    break;
                case "GameScene":
                    await InitializeGameAsync(token);
                    break;
                case "LoadingScene":
                    // 로딩 씬에서는 추가 작업 없음
                    break;
            }
        }
        catch (OperationCanceledException)
        {
            Debug.Log($"씬 '{sceneName}'의 초기화가 취소되었습니다.");
        }
    }

    private async Task LoadMenuDataAsync(CancellationToken token)
    {
        await Task.Delay(1000, token);
        Debug.Log("메뉴 데이터 로드 완료");
    }

    private async Task InitializeGameAsync(CancellationToken token)
    {
        await Task.Delay(2000, token);
        Debug.Log("게임 초기화 완료");
    }

    private void OnDestroy()
    {
        SceneManager.activeSceneChanged -= OnActiveSceneChanged;
        _sceneCts?.Cancel();
        _sceneCts?.Dispose();
    }
}
```

---

## 9. 씬 로딩 중 비동기 작업 관리

### 비동기 씬 로딩과 작업 조율

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using System;
using System.Threading;
using System.Threading.Tasks;

// ✅ 씬 로딩 중 비동기 작업을 조율하는 로더
public class AsyncSceneLoader : MonoBehaviour
{
    [SerializeField] private CanvasGroup _loadingScreen;

    public async Task LoadSceneWithPreparationAsync(
        string sceneName,
        CancellationToken token = default)
    {
        // 1단계: 로딩 화면 표시
        ShowLoadingScreen();

        try
        {
            // 2단계: 현재 씬 데이터 저장 (비동기)
            await SaveCurrentSceneDataAsync(token);

            // 3단계: 새 씬에 필요한 데이터 프리로드 (비동기)
            var preloadTask = PreloadSceneDataAsync(sceneName, token);

            // 4단계: 씬 비동기 로드 시작
            var loadOperation = SceneManager.LoadSceneAsync(sceneName);
            loadOperation.allowSceneActivation = false;

            // 5단계: 씬 로드 90%와 데이터 프리로드 완료를 모두 대기
            while (loadOperation.progress < 0.9f)
            {
                token.ThrowIfCancellationRequested();
                await Task.Yield();
            }

            // 프리로드 완료 대기
            await preloadTask;

            // 6단계: 씬 활성화
            loadOperation.allowSceneActivation = true;

            // 7단계: 씬 완전 로드 대기
            while (!loadOperation.isDone)
            {
                await Task.Yield();
            }

            // 8단계: 로딩 화면 숨김
            HideLoadingScreen();
        }
        catch (OperationCanceledException)
        {
            Debug.Log("씬 로딩이 취소되었습니다.");
            HideLoadingScreen();
            throw;
        }
    }

    private async Task SaveCurrentSceneDataAsync(CancellationToken token)
    {
        await Task.Delay(500, token);
        Debug.Log("현재 씬 데이터 저장 완료");
    }

    private async Task PreloadSceneDataAsync(string sceneName, CancellationToken token)
    {
        await Task.Delay(1000, token);
        Debug.Log($"'{sceneName}' 프리로드 데이터 준비 완료");
    }

    private void ShowLoadingScreen()
    {
        if (_loadingScreen != null)
        {
            _loadingScreen.alpha = 1;
            _loadingScreen.blocksRaycasts = true;
        }
    }

    private void HideLoadingScreen()
    {
        if (_loadingScreen != null)
        {
            _loadingScreen.alpha = 0;
            _loadingScreen.blocksRaycasts = false;
        }
    }
}
```

### Additive 씬 로딩과 비동기 작업

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

// ✅ Additive 씬 로딩 시 각 씬의 비동기 작업 독립 관리
public class AdditiveSceneManager : MonoBehaviour
{
    private readonly Dictionary<string, CancellationTokenSource> _sceneTokens
        = new Dictionary<string, CancellationTokenSource>();

    public async Task LoadAdditiveSceneAsync(string sceneName, CancellationToken token)
    {
        // 씬별 CancellationTokenSource 생성
        var sceneCts = CancellationTokenSource.CreateLinkedTokenSource(token);
        _sceneTokens[sceneName] = sceneCts;

        // 씬 로드
        var operation = SceneManager.LoadSceneAsync(sceneName, LoadSceneMode.Additive);
        while (!operation.isDone)
        {
            sceneCts.Token.ThrowIfCancellationRequested();
            await Task.Yield();
        }

        Debug.Log($"Additive 씬 '{sceneName}' 로드 완료");
    }

    public async Task UnloadAdditiveSceneAsync(string sceneName)
    {
        // 해당 씬의 비동기 작업 취소
        if (_sceneTokens.TryGetValue(sceneName, out var cts))
        {
            cts.Cancel();
            cts.Dispose();
            _sceneTokens.Remove(sceneName);
        }

        // 씬 언로드
        var operation = SceneManager.UnloadSceneAsync(sceneName);
        while (!operation.isDone)
        {
            await Task.Yield();
        }

        Debug.Log($"Additive 씬 '{sceneName}' 언로드 완료");
    }

    /// <summary>
    /// 특정 Additive 씬에 묶인 CancellationToken 반환
    /// </summary>
    public CancellationToken GetSceneToken(string sceneName)
    {
        return _sceneTokens.TryGetValue(sceneName, out var cts)
            ? cts.Token
            : CancellationToken.None;
    }

    private void OnDestroy()
    {
        foreach (var kvp in _sceneTokens)
        {
            kvp.Value?.Cancel();
            kvp.Value?.Dispose();
        }
        _sceneTokens.Clear();
    }
}
```

---

## 10. Application.quitting 이벤트 활용

### 애플리케이션 종료 시 비동기 정리

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

// ✅ Application.quitting을 활용한 앱 종료 시 정리
public class ApplicationLifecycleManager : MonoBehaviour
{
    private static CancellationTokenSource _appCts;
    private static bool _isQuitting;

    // 앱 전체에서 사용할 수 있는 CancellationToken
    public static CancellationToken AppCancellationToken =>
        _appCts?.Token ?? CancellationToken.None;

    public static bool IsQuitting => _isQuitting;

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void Initialize()
    {
        _appCts = new CancellationTokenSource();
        _isQuitting = false;

        // Application.quitting: 앱이 종료되기 직전에 호출
        Application.quitting += OnApplicationQuitting;

        // Application.wantsToQuit: 종료를 지연시킬 수 있음 (bool 반환)
        Application.wantsToQuit += OnApplicationWantsToQuit;

        Debug.Log("[AppLifecycle] 초기화 완료");
    }

    private static void OnApplicationQuitting()
    {
        Debug.Log("[AppLifecycle] 앱 종료 중...");
        _isQuitting = true;

        // 모든 비동기 작업 취소
        _appCts?.Cancel();
        _appCts?.Dispose();
    }

    private static bool OnApplicationWantsToQuit()
    {
        if (_isQuitting) return true;

        Debug.Log("[AppLifecycle] 종료 전 데이터 저장 시작...");

        // 동기적으로 처리할 수 있는 정리 작업 수행
        SaveCriticalData();

        return true; // false를 반환하면 종료를 막을 수 있음
    }

    private static void SaveCriticalData()
    {
        // 중요 데이터 동기 저장
        PlayerPrefs.Save();
        Debug.Log("[AppLifecycle] 중요 데이터 저장 완료");
    }
}

// ✅ AppCancellationToken을 활용하는 컴포넌트
public class AppAwareComponent : MonoBehaviour
{
    private async void Start()
    {
        try
        {
            await LongRunningWorkAsync(ApplicationLifecycleManager.AppCancellationToken);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("앱 종료로 인해 작업이 취소되었습니다.");
        }
    }

    private async Task LongRunningWorkAsync(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            // 앱 종료 여부 확인
            if (ApplicationLifecycleManager.IsQuitting)
            {
                Debug.Log("앱이 종료 중이므로 작업을 중단합니다.");
                break;
            }

            await Task.Delay(1000, token);
            Debug.Log("작업 진행 중...");
        }
    }
}
```

### 종료 전 비동기 저장 패턴

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;

// ✅ Application.wantsToQuit와 비동기 저장의 조합
public class AsyncSaveOnQuit : MonoBehaviour
{
    private static bool _saveCompleted;
    private static bool _saveInProgress;

    private void Awake()
    {
        DontDestroyOnLoad(gameObject);
        Application.wantsToQuit += HandleWantsToQuit;
    }

    private static bool HandleWantsToQuit()
    {
        if (_saveCompleted) return true;
        if (_saveInProgress) return false;

        // 비동기 저장 시작
        _saveInProgress = true;
        SaveAndQuit();
        return false; // 일단 종료를 막음
    }

    private static async void SaveAndQuit()
    {
        try
        {
            Debug.Log("종료 전 비동기 저장 시작...");
            await SaveAllDataAsync();

            _saveCompleted = true;
            Debug.Log("저장 완료. 앱을 종료합니다.");

            // 저장 완료 후 다시 종료 요청
            Application.Quit();
        }
        catch (Exception ex)
        {
            Debug.LogError($"저장 중 오류: {ex.Message}");
            _saveCompleted = true; // 오류가 있어도 종료 허용
            Application.Quit();
        }
    }

    private static async Task SaveAllDataAsync()
    {
        // 중요 데이터 비동기 저장
        await Task.Delay(1000); // 실제로는 파일 I/O 등
        Debug.Log("모든 데이터 저장 완료");
    }

    private void OnDestroy()
    {
        Application.wantsToQuit -= HandleWantsToQuit;
    }
}
```

---

## 11. CompositeDisposable 패턴

### 직접 구현하는 CompositeDisposable

```csharp
using System;
using System.Collections.Generic;

// ✅ 여러 IDisposable을 하나로 묶어 관리하는 CompositeDisposable
public class CompositeDisposable : IDisposable
{
    private readonly List<IDisposable> _disposables = new List<IDisposable>();
    private bool _disposed;
    private readonly object _lock = new object();

    public int Count
    {
        get
        {
            lock (_lock) { return _disposables.Count; }
        }
    }

    public bool IsDisposed => _disposed;

    /// <summary>
    /// IDisposable을 추가합니다. 이미 Dispose된 경우 즉시 Dispose합니다.
    /// </summary>
    public void Add(IDisposable disposable)
    {
        if (disposable == null) throw new ArgumentNullException(nameof(disposable));

        bool shouldDispose = false;

        lock (_lock)
        {
            if (_disposed)
                shouldDispose = true;
            else
                _disposables.Add(disposable);
        }

        if (shouldDispose)
            disposable.Dispose();
    }

    /// <summary>
    /// 특정 IDisposable을 제거하고 Dispose합니다.
    /// </summary>
    public bool Remove(IDisposable disposable)
    {
        if (disposable == null) throw new ArgumentNullException(nameof(disposable));

        lock (_lock)
        {
            if (_disposed) return false;

            if (_disposables.Remove(disposable))
            {
                disposable.Dispose();
                return true;
            }
        }

        return false;
    }

    /// <summary>
    /// 모든 등록된 IDisposable을 Dispose합니다.
    /// </summary>
    public void Dispose()
    {
        List<IDisposable> toDispose = null;

        lock (_lock)
        {
            if (_disposed) return;
            _disposed = true;
            toDispose = new List<IDisposable>(_disposables);
            _disposables.Clear();
        }

        // 역순으로 Dispose (LIFO)
        for (int i = toDispose.Count - 1; i >= 0; i--)
        {
            try
            {
                toDispose[i]?.Dispose();
            }
            catch (Exception ex)
            {
                UnityEngine.Debug.LogError($"Dispose 중 오류: {ex.Message}");
            }
        }
    }

    /// <summary>
    /// 모든 등록된 IDisposable을 Dispose하되, 새로운 추가를 계속 허용합니다.
    /// </summary>
    public void Clear()
    {
        List<IDisposable> toDispose;

        lock (_lock)
        {
            toDispose = new List<IDisposable>(_disposables);
            _disposables.Clear();
        }

        foreach (var d in toDispose)
        {
            try { d?.Dispose(); }
            catch (Exception ex)
            {
                UnityEngine.Debug.LogError($"Clear 중 Dispose 오류: {ex.Message}");
            }
        }
    }
}
```

### CompositeDisposable 활용 예제

```csharp
using UnityEngine;
using System;
using System.Threading;

// ✅ CompositeDisposable로 다양한 리소스를 통합 관리
public class ResourceManagerExample : MonoBehaviour
{
    private readonly CompositeDisposable _disposables = new CompositeDisposable();

    private void Start()
    {
        // CancellationTokenSource 등록
        var cts = new CancellationTokenSource();
        _disposables.Add(cts);

        // 이벤트 구독을 Disposable로 래핑하여 등록
        var eventSub = new EventSubscription(
            () => GameEvents.OnScoreChanged += HandleScore,
            () => GameEvents.OnScoreChanged -= HandleScore
        );
        _disposables.Add(eventSub);

        // 타이머 등록
        var timer = new System.Threading.Timer(
            _ => Debug.Log("Timer tick"),
            null,
            TimeSpan.Zero,
            TimeSpan.FromSeconds(5)
        );
        _disposables.Add(timer);

        Debug.Log($"등록된 리소스 수: {_disposables.Count}");
    }

    private void HandleScore(int score)
    {
        Debug.Log($"Score: {score}");
    }

    private void OnDestroy()
    {
        // 한 번의 호출로 모든 리소스 정리
        _disposables.Dispose();
        Debug.Log("모든 리소스 정리 완료");
    }
}

// ✅ 이벤트 구독/해제를 IDisposable로 래핑하는 유틸리티
public class EventSubscription : IDisposable
{
    private Action _unsubscribe;
    private bool _disposed;

    public EventSubscription(Action subscribe, Action unsubscribe)
    {
        _unsubscribe = unsubscribe ?? throw new ArgumentNullException(nameof(unsubscribe));
        subscribe?.Invoke();
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        _unsubscribe?.Invoke();
        _unsubscribe = null;
    }
}
```

---

## 12. 실전 예제: 네트워크 매니저의 생명주기 관리

### 종합 예제

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// 실전 네트워크 매니저: 생명주기의 모든 측면을 종합적으로 관리합니다.
///
/// 관리 영역:
/// 1. 앱 전체 수명 (DontDestroyOnLoad)
/// 2. 씬별 요청 취소
/// 3. 개별 요청 취소
/// 4. 연결 풀 관리
/// 5. 정상적인 종료 (Graceful Shutdown)
/// </summary>
public class NetworkManager : MonoBehaviour, IDisposable
{
    // =============================================
    // 싱글톤
    // =============================================
    private static NetworkManager _instance;
    public static NetworkManager Instance => _instance;

    // =============================================
    // CancellationToken 계층 구조
    // =============================================
    private CancellationTokenSource _appLifetimeCts;    // 앱 전체
    private CancellationTokenSource _sceneCts;          // 현재 씬
    private readonly Dictionary<string, CancellationTokenSource> _requestCts
        = new Dictionary<string, CancellationTokenSource>();  // 개별 요청

    // =============================================
    // 리소스 관리
    // =============================================
    private HttpClient _httpClient;
    private readonly CompositeDisposable _disposables = new CompositeDisposable();
    private readonly Queue<Func<Task>> _pendingRequests = new Queue<Func<Task>>();
    private readonly SemaphoreSlim _requestThrottle = new SemaphoreSlim(5); // 동시 요청 5개 제한

    // =============================================
    // 상태
    // =============================================
    private bool _disposed;
    private bool _isInitialized;
    private int _activeRequestCount;

    public int ActiveRequestCount => _activeRequestCount;
    public bool IsInitialized => _isInitialized;

    // =============================================
    // 초기화
    // =============================================

    private void Awake()
    {
        if (_instance != null && _instance != this)
        {
            Destroy(gameObject);
            return;
        }

        _instance = this;
        DontDestroyOnLoad(gameObject);

        Initialize();
    }

    private void Initialize()
    {
        // 앱 전체 수명 토큰
        _appLifetimeCts = new CancellationTokenSource();
        _disposables.Add(_appLifetimeCts);

        // 씬 토큰
        CreateSceneCts();

        // HttpClient 설정
        var handler = new HttpClientHandler
        {
            MaxConnectionsPerServer = 10
        };
        _httpClient = new HttpClient(handler)
        {
            Timeout = TimeSpan.FromSeconds(30)
        };
        _disposables.Add(_httpClient);

        // 이벤트 등록
        SceneManager.activeSceneChanged += OnSceneChanged;
        Application.quitting += OnApplicationQuitting;

        _isInitialized = true;
        Debug.Log("[NetworkManager] 초기화 완료");
    }

    // =============================================
    // 씬 관리
    // =============================================

    private void OnSceneChanged(Scene oldScene, Scene newScene)
    {
        Debug.Log($"[NetworkManager] 씬 전환: {oldScene.name} → {newScene.name}");

        // 이전 씬의 요청 모두 취소
        CancelAllSceneRequests();

        // 새 씬용 토큰 생성
        CreateSceneCts();
    }

    private void CreateSceneCts()
    {
        _sceneCts?.Cancel();
        _sceneCts?.Dispose();

        _sceneCts = CancellationTokenSource.CreateLinkedTokenSource(
            _appLifetimeCts.Token
        );
    }

    private void CancelAllSceneRequests()
    {
        // 씬 범위 요청 취소
        _sceneCts?.Cancel();

        // 개별 요청도 모두 취소
        foreach (var kvp in _requestCts)
        {
            kvp.Value?.Cancel();
            kvp.Value?.Dispose();
        }
        _requestCts.Clear();
    }

    // =============================================
    // 공개 API: HTTP 요청
    // =============================================

    /// <summary>
    /// GET 요청을 보냅니다.
    /// requestId를 지정하면 해당 요청만 개별 취소할 수 있습니다.
    /// </summary>
    public async Task<string> GetAsync(
        string url,
        string requestId = null,
        bool sceneScoped = true,
        CancellationToken additionalToken = default)
    {
        ThrowIfDisposed();

        // CancellationToken 조합
        var tokens = new List<CancellationToken> { _appLifetimeCts.Token };

        if (sceneScoped)
            tokens.Add(_sceneCts.Token);

        if (additionalToken != default)
            tokens.Add(additionalToken);

        // 개별 요청 취소 토큰
        CancellationTokenSource requestCts = null;
        if (!string.IsNullOrEmpty(requestId))
        {
            // 동일 ID의 이전 요청 취소
            CancelRequest(requestId);
            requestCts = new CancellationTokenSource();
            tokens.Add(requestCts.Token);
            _requestCts[requestId] = requestCts;
        }

        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(tokens.ToArray());

        try
        {
            // 동시 요청 수 제한
            await _requestThrottle.WaitAsync(linkedCts.Token);
            Interlocked.Increment(ref _activeRequestCount);

            try
            {
                Debug.Log($"[NetworkManager] GET 요청: {url}");
                var response = await _httpClient.GetAsync(url, linkedCts.Token);
                response.EnsureSuccessStatusCode();

                var content = await response.Content.ReadAsStringAsync();
                Debug.Log($"[NetworkManager] 응답 수신: {url} ({content.Length} bytes)");
                return content;
            }
            finally
            {
                Interlocked.Decrement(ref _activeRequestCount);
                _requestThrottle.Release();
            }
        }
        catch (OperationCanceledException) when (_appLifetimeCts.Token.IsCancellationRequested)
        {
            Debug.Log($"[NetworkManager] 앱 종료로 요청 취소: {url}");
            throw;
        }
        catch (OperationCanceledException) when (_sceneCts.Token.IsCancellationRequested)
        {
            Debug.Log($"[NetworkManager] 씬 전환으로 요청 취소: {url}");
            throw;
        }
        catch (OperationCanceledException)
        {
            Debug.Log($"[NetworkManager] 요청 취소: {url}");
            throw;
        }
        catch (HttpRequestException ex)
        {
            Debug.LogError($"[NetworkManager] HTTP 오류: {ex.Message}");
            throw;
        }
        finally
        {
            // 개별 요청 토큰 정리
            if (!string.IsNullOrEmpty(requestId))
            {
                _requestCts.Remove(requestId);
                requestCts?.Dispose();
            }
        }
    }

    /// <summary>
    /// POST 요청을 보냅니다.
    /// </summary>
    public async Task<string> PostAsync(
        string url,
        string jsonBody,
        string requestId = null,
        CancellationToken additionalToken = default)
    {
        ThrowIfDisposed();

        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            _appLifetimeCts.Token,
            _sceneCts.Token,
            additionalToken
        );

        await _requestThrottle.WaitAsync(linkedCts.Token);
        Interlocked.Increment(ref _activeRequestCount);

        try
        {
            var content = new StringContent(
                jsonBody,
                System.Text.Encoding.UTF8,
                "application/json"
            );

            var response = await _httpClient.PostAsync(url, content, linkedCts.Token);
            response.EnsureSuccessStatusCode();

            return await response.Content.ReadAsStringAsync();
        }
        finally
        {
            Interlocked.Decrement(ref _activeRequestCount);
            _requestThrottle.Release();
        }
    }

    // =============================================
    // 개별 요청 취소
    // =============================================

    /// <summary>
    /// 특정 requestId의 요청을 취소합니다.
    /// </summary>
    public void CancelRequest(string requestId)
    {
        if (_requestCts.TryGetValue(requestId, out var cts))
        {
            cts.Cancel();
            cts.Dispose();
            _requestCts.Remove(requestId);
            Debug.Log($"[NetworkManager] 요청 취소됨: {requestId}");
        }
    }

    /// <summary>
    /// 현재 씬의 모든 요청을 취소합니다.
    /// </summary>
    public void CancelCurrentSceneRequests()
    {
        CancelAllSceneRequests();
        CreateSceneCts();
    }

    // =============================================
    // 종료 처리
    // =============================================

    private void OnApplicationQuitting()
    {
        Debug.Log("[NetworkManager] 앱 종료 처리 중...");

        // 모든 진행 중인 요청 취소
        _appLifetimeCts?.Cancel();

        // 활성 요청이 끝날 때까지 짧은 대기
        WaitForActiveRequests(TimeSpan.FromSeconds(2));
    }

    private void WaitForActiveRequests(TimeSpan timeout)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        while (_activeRequestCount > 0 && sw.Elapsed < timeout)
        {
            System.Threading.Thread.Sleep(50);
        }

        if (_activeRequestCount > 0)
        {
            Debug.LogWarning(
                $"[NetworkManager] {_activeRequestCount}개의 요청이 완료되지 않은 채 종료됩니다.");
        }
    }

    // =============================================
    // IDisposable
    // =============================================

    private void ThrowIfDisposed()
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(NetworkManager));
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;

        // 모든 요청 취소
        _appLifetimeCts?.Cancel();

        // 개별 요청 토큰 정리
        foreach (var kvp in _requestCts)
        {
            kvp.Value?.Cancel();
            kvp.Value?.Dispose();
        }
        _requestCts.Clear();

        // 씬 토큰 정리
        _sceneCts?.Cancel();
        _sceneCts?.Dispose();

        // CompositeDisposable 정리
        _disposables.Dispose();

        // SemaphoreSlim 정리
        _requestThrottle.Dispose();

        // 이벤트 해제
        SceneManager.activeSceneChanged -= OnSceneChanged;
        Application.quitting -= OnApplicationQuitting;

        Debug.Log("[NetworkManager] 모든 리소스 정리 완료");
    }

    private void OnDestroy()
    {
        Dispose();
    }
}
```

### NetworkManager 사용 예제

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

// ✅ NetworkManager를 사용하는 게임 컴포넌트
public class PlayerProfileLoader : MonoBehaviour
{
    [SerializeField] private string _playerId;

    private async void Start()
    {
        try
        {
            // 씬 범위 요청: 씬 전환 시 자동 취소
            string profileJson = await NetworkManager.Instance.GetAsync(
                $"https://api.example.com/players/{_playerId}",
                requestId: $"player-profile-{_playerId}",
                sceneScoped: true,
                additionalToken: destroyCancellationToken
            );

            Debug.Log($"프로필 로드 완료: {profileJson}");
            ApplyProfile(profileJson);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("프로필 로드가 취소되었습니다.");
        }
        catch (Exception ex)
        {
            Debug.LogError($"프로필 로드 실패: {ex.Message}");
        }
    }

    // 새 프로필 로드 (기존 요청 자동 취소)
    public async Task ReloadProfileAsync()
    {
        try
        {
            // 동일한 requestId를 사용하면 이전 요청이 자동 취소됨
            string profileJson = await NetworkManager.Instance.GetAsync(
                $"https://api.example.com/players/{_playerId}",
                requestId: $"player-profile-{_playerId}",
                additionalToken: destroyCancellationToken
            );

            ApplyProfile(profileJson);
        }
        catch (OperationCanceledException)
        {
            // 정상적인 취소
        }
    }

    private void ApplyProfile(string json)
    {
        Debug.Log("프로필 적용 완료");
    }
}

// ✅ 검색 자동완성: 이전 요청을 취소하고 새 요청 전송
public class SearchAutocomplete : MonoBehaviour
{
    private const string SearchRequestId = "search-autocomplete";

    public async void OnSearchTextChanged(string query)
    {
        if (string.IsNullOrWhiteSpace(query))
            return;

        try
        {
            // 동일 requestId로 이전 검색 요청 자동 취소
            string results = await NetworkManager.Instance.GetAsync(
                $"https://api.example.com/search?q={Uri.EscapeDataString(query)}",
                requestId: SearchRequestId,
                additionalToken: destroyCancellationToken
            );

            DisplayResults(results);
        }
        catch (OperationCanceledException)
        {
            // 새 검색어 입력으로 이전 요청 취소됨 (정상)
        }
        catch (Exception ex)
        {
            Debug.LogError($"검색 오류: {ex.Message}");
        }
    }

    private void DisplayResults(string results)
    {
        Debug.Log($"검색 결과: {results}");
    }
}
```

---

## 주의사항

### 흔한 실수와 해결 방법

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class CommonMistakesExample : MonoBehaviour
{
    // ❌ 실수 1: CancellationTokenSource를 Dispose하지 않음
    private void BadExample1()
    {
        var cts = new CancellationTokenSource();
        DoWorkAsync(cts.Token); // cts가 Dispose되지 않음 → 리소스 누수
    }

    // ✅ 수정: using 또는 OnDestroy에서 Dispose
    private CancellationTokenSource _cts;

    private void GoodExample1()
    {
        _cts = new CancellationTokenSource();
        DoWorkAsync(_cts.Token).SafeFireAndForget();
    }

    private void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }

    // ❌ 실수 2: 취소된 CancellationTokenSource 재사용
    private void BadExample2()
    {
        _cts?.Cancel(); // 취소
        // _cts.Token은 이미 취소 상태이므로 새 작업도 즉시 취소됨!
        DoWorkAsync(_cts.Token); // 바로 OperationCanceledException 발생
    }

    // ✅ 수정: 새 CancellationTokenSource 생성
    private void GoodExample2()
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _cts = new CancellationTokenSource(); // 새로 생성
        DoWorkAsync(_cts.Token).SafeFireAndForget();
    }

    // ❌ 실수 3: async void에서 예외 처리 누락
    private async void BadExample3()
    {
        await Task.Delay(1000);
        throw new Exception("처리되지 않는 예외!"); // 앱 크래시 가능
    }

    // ✅ 수정: try-catch 또는 SafeFireAndForget 사용
    private async void GoodExample3()
    {
        try
        {
            await Task.Delay(1000);
            throw new Exception("안전하게 처리되는 예외");
        }
        catch (Exception ex)
        {
            Debug.LogError($"오류: {ex.Message}");
        }
    }

    // ❌ 실수 4: OnDestroy에서 비동기 작업 시작
    // OnDestroy는 async를 지원하지 않으며, 이 시점에서 시작한 작업은 안정성이 보장되지 않음
    private async void OnDestroyBad()
    {
        await SaveDataAsync(); // 완료가 보장되지 않음!
    }

    // ✅ 수정: 동기적 정리 또는 정적 메서드로 위임
    private void OnDestroyGood()
    {
        _cts?.Cancel();
        _cts?.Dispose();
        // 필요하면 동기 저장 수행
        SaveDataSync();
    }

    private async Task DoWorkAsync(CancellationToken token)
    {
        await Task.Delay(1000, token);
    }

    private async Task SaveDataAsync()
    {
        await Task.Delay(500);
    }

    private void SaveDataSync()
    {
        // 동기적 저장 로직
    }
}
```

### Unity 특화 주의사항

| 상황 | 주의점 | 권장 패턴 |
|------|--------|-----------|
| `async void Start()` | 예외가 전파되지 않음 | `try-catch`로 감싸기 |
| `OnDestroy`에서 정리 | 실행 순서 보장 없음 | 중요 리소스 우선 정리 |
| `DontDestroyOnLoad` | 씬 전환 후에도 살아있음 | 씬별 토큰 분리 |
| `destroyCancellationToken` | Unity 2022.2+ 전용 | 이전 버전: `GetCancellationTokenOnDestroy()` |
| `Application.Quit()` | 에디터에서는 호출 안 됨 | `#if UNITY_EDITOR` 분기 |
| Play Mode 종료 | OnDestroy 호출 보장 안 됨 | `[RuntimeInitializeOnLoadMethod]` 활용 |

---

## 베스트 프랙티스

### 1. CancellationToken 계층 구조 설계

```
Application (최상위)
  └── Scene (씬 범위)
        └── Component (컴포넌트 범위: destroyCancellationToken)
              └── Operation (개별 작업 범위)
```

```csharp
// ✅ 계층적 CancellationToken 사용
public class HierarchicalCancellationExample : MonoBehaviour
{
    public async Task ExecuteOperationAsync(CancellationToken operationToken)
    {
        // 컴포넌트 토큰 + 작업 토큰을 결합
        using var linked = CancellationTokenSource.CreateLinkedTokenSource(
            destroyCancellationToken,
            operationToken
        );

        await DoWorkAsync(linked.Token);
    }

    private async Task DoWorkAsync(CancellationToken token)
    {
        await Task.Delay(1000, token);
    }
}
```

### 2. 리소스 정리 체크리스트

```csharp
// ✅ 모든 비동기 컴포넌트가 따라야 할 패턴
public class WellManagedComponent : MonoBehaviour
{
    // 1. CancellationTokenSource 선언
    private CancellationTokenSource _cts;

    // 2. Disposable 컬렉션
    private readonly CompositeDisposable _disposables = new CompositeDisposable();

    private void Awake()
    {
        // 3. 초기화
        _cts = new CancellationTokenSource();
    }

    private async void Start()
    {
        // 4. 모든 비동기 작업에 CancellationToken 전달
        try
        {
            await RunAsync(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            // 5. 취소 예외 처리
            Debug.Log("작업이 취소되었습니다.");
        }
        catch (Exception ex)
        {
            // 6. 일반 예외 처리
            Debug.LogError($"오류: {ex.Message}");
        }
    }

    private async Task RunAsync(CancellationToken token)
    {
        await Task.Delay(1000, token);
    }

    private void OnDestroy()
    {
        // 7. 체계적 정리
        _cts?.Cancel();
        _cts?.Dispose();
        _disposables.Dispose();
    }
}
```

### 3. 생명주기 관리 패턴 요약

| 패턴 | 사용 시점 | 장점 |
|------|-----------|------|
| `destroyCancellationToken` | MonoBehaviour의 비동기 작업 | 자동 관리, 간결함 |
| `CancellationTokenSource` | 수동 취소가 필요한 경우 | 세밀한 제어 가능 |
| `LinkedTokenSource` | 여러 취소 조건 결합 | 계층적 취소 지원 |
| `CompositeDisposable` | 여러 구독/리소스 관리 | 일괄 정리 |
| `DisposableBag` (R3) | R3 Observable 구독 관리 | 경량, 구조체 기반 |
| `AddTo(this)` | MonoBehaviour에 구독 묶기 | 자동 정리 |
| `IAsyncDisposable` | 비동기 정리가 필요한 리소스 | Graceful shutdown |
| `SafeFireAndForget` | Fire-and-forget 작업 | 예외 안전성 |
| `Application.quitting` | 앱 종료 시 정리 | 전역 종료 처리 |

### 4. 테스트 시 고려사항

```csharp
using System.Threading;
using System.Threading.Tasks;
using NUnit.Framework;

// ✅ 비동기 생명주기 로직을 테스트하는 방법
[TestFixture]
public class LifecycleManagementTests
{
    [Test]
    public async Task CancellationToken_Should_Cancel_Operations()
    {
        var cts = new CancellationTokenSource();

        var task = Task.Delay(10000, cts.Token);

        // 100ms 후 취소
        cts.CancelAfter(100);

        Assert.ThrowsAsync<TaskCanceledException>(async () => await task);

        cts.Dispose();
    }

    [Test]
    public void CompositeDisposable_Should_Dispose_All()
    {
        var disposables = new CompositeDisposable();
        int disposeCount = 0;

        for (int i = 0; i < 5; i++)
        {
            disposables.Add(new ActionDisposable(() => disposeCount++));
        }

        disposables.Dispose();

        Assert.AreEqual(5, disposeCount);
    }
}

// 테스트 유틸리티
public class ActionDisposable : System.IDisposable
{
    private System.Action _action;

    public ActionDisposable(System.Action action)
    {
        _action = action;
    }

    public void Dispose()
    {
        _action?.Invoke();
        _action = null;
    }
}
```

---

## 참고 자료

- [Unity Documentation - MonoBehaviour.destroyCancellationToken](https://docs.unity3d.com/ScriptReference/MonoBehaviour-destroyCancellationToken.html)
- [Unity Documentation - Application.quitting](https://docs.unity3d.com/ScriptReference/Application-quitting.html)
- [Unity Documentation - SceneManager](https://docs.unity3d.com/ScriptReference/SceneManagement.SceneManager.html)
- [Microsoft Docs - CancellationTokenSource](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtokensource)
- [Microsoft Docs - IAsyncDisposable](https://learn.microsoft.com/dotnet/api/system.iasyncdisposable)
- [Microsoft Docs - Implement a DisposeAsync method](https://learn.microsoft.com/dotnet/standard/garbage-collection/implementing-disposeasync)
- [UniTask - GitHub](https://github.com/Cysharp/UniTask)
- [R3 - GitHub](https://github.com/Cysharp/R3)
- [UniRx - GitHub](https://github.com/neuecc/UniRx)

---

[← 이전: 35. 에러 전파와 집계](../11-cancellation-error/35-error-propagation.md) | [다음: 37. 플랫폼별 고려사항 →](../12-platform/37-platform-considerations.md)
