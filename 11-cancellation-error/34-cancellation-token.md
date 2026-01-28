# 34. CancellationToken

## 개요

`CancellationToken`은 **비동기 작업의 협력적 취소**를 위한 메커니즘입니다. Unity에서는 Scene 전환, GameObject 파괴, 앱 종료 시 진행 중인 비동기 작업을 안전하게 중단하는 데 필수적입니다.

---

## 기본 개념

### 구성 요소

```
CancellationTokenSource (생산자)
        │
        │  Token 발급
        ↓
  CancellationToken (토큰)
        │
        │  취소 확인
        ↓
    비동기 작업 (소비자)
```

### 기본 사용법

```csharp
using System.Threading;
using UnityEngine;

public class CancellationBasics : MonoBehaviour
{
    private CancellationTokenSource _cts;

    async void Start()
    {
        // 1. CancellationTokenSource 생성
        _cts = new CancellationTokenSource();

        try
        {
            // 2. Token을 비동기 작업에 전달
            await LongRunningTaskAsync(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 취소되었습니다");
        }
    }

    async Task LongRunningTaskAsync(CancellationToken cancellationToken)
    {
        for (int i = 0; i < 100; i++)
        {
            // 3. 취소 확인
            cancellationToken.ThrowIfCancellationRequested();

            Debug.Log($"Processing {i}...");
            await Task.Delay(100, cancellationToken);
        }
    }

    void OnDestroy()
    {
        // 4. 취소 신호 발생
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

---

## CancellationTokenSource

### 생성 및 취소

```csharp
public class CTSExamples : MonoBehaviour
{
    void Start()
    {
        // 기본 생성
        var cts1 = new CancellationTokenSource();

        // 타임아웃과 함께 생성
        var cts2 = new CancellationTokenSource(TimeSpan.FromSeconds(5));

        // 밀리초로 타임아웃
        var cts3 = new CancellationTokenSource(5000);

        // 취소
        cts1.Cancel();

        // 지연 취소
        cts1.CancelAfter(TimeSpan.FromSeconds(3));
        cts1.CancelAfter(3000);

        // 정리
        cts1.Dispose();
        cts2.Dispose();
        cts3.Dispose();
    }
}
```

### 연결된 토큰

```csharp
public class LinkedTokenExample : MonoBehaviour
{
    private CancellationTokenSource _globalCts;
    private CancellationTokenSource _localCts;

    async void Start()
    {
        _globalCts = new CancellationTokenSource();
        _localCts = new CancellationTokenSource();

        // 여러 CTS를 연결 - 하나라도 취소되면 취소됨
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            _globalCts.Token,
            _localCts.Token);

        try
        {
            await SomeOperationAsync(linkedCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("취소됨 (global 또는 local)");
        }
    }

    async Task SomeOperationAsync(CancellationToken ct)
    {
        await Task.Delay(10000, ct);
    }

    // 로컬 작업만 취소
    public void CancelLocalOperation()
    {
        _localCts?.Cancel();
    }

    // 모든 작업 취소
    public void CancelAllOperations()
    {
        _globalCts?.Cancel();
    }
}
```

---

## 취소 확인 방법

### ThrowIfCancellationRequested

```csharp
public class ThrowIfCancelledExample : MonoBehaviour
{
    async Task ProcessAsync(CancellationToken cancellationToken)
    {
        for (int i = 0; i < 1000; i++)
        {
            // 취소 시 OperationCanceledException 발생
            cancellationToken.ThrowIfCancellationRequested();

            // 처리 로직
            await Task.Yield();
        }
    }
}
```

### IsCancellationRequested

```csharp
public class IsCancelledExample : MonoBehaviour
{
    async Task ProcessWithCleanupAsync(CancellationToken cancellationToken)
    {
        for (int i = 0; i < 1000; i++)
        {
            // 예외 없이 취소 확인
            if (cancellationToken.IsCancellationRequested)
            {
                // 정리 작업 수행
                Debug.Log("Cleaning up before cancellation...");
                CleanupResources();
                return; // 또는 throw
            }

            await Task.Yield();
        }
    }

    void CleanupResources()
    {
        // 정리 로직
    }
}
```

### Register 콜백

```csharp
public class RegisterCallbackExample : MonoBehaviour
{
    private CancellationTokenSource _cts;
    private CancellationTokenRegistration _registration;

    void Start()
    {
        _cts = new CancellationTokenSource();

        // 취소 시 실행될 콜백 등록
        _registration = _cts.Token.Register(() =>
        {
            Debug.Log("취소 콜백 실행됨!");
            // 정리 작업
        });

        // 상태 객체와 함께
        _cts.Token.Register(state =>
        {
            Debug.Log($"취소됨: {state}");
        }, "My State");
    }

    void OnDestroy()
    {
        // 등록 해제
        _registration.Dispose();
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

---

## Unity에서의 활용

### MonoBehaviour 생명주기

```csharp
public class LifecycleCancellation : MonoBehaviour
{
    private CancellationTokenSource _cts;

    async void Start()
    {
        _cts = new CancellationTokenSource();

        try
        {
            await LoadDataAsync(_cts.Token);
            await ProcessDataAsync(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 취소됨 (오브젝트 파괴)");
        }
    }

    void OnDisable()
    {
        // 비활성화 시 취소 (선택적)
        // _cts?.Cancel();
    }

    void OnDestroy()
    {
        // 파괴 시 반드시 취소
        _cts?.Cancel();
        _cts?.Dispose();
    }

    async Task LoadDataAsync(CancellationToken ct)
    {
        await Task.Delay(1000, ct);
    }

    async Task ProcessDataAsync(CancellationToken ct)
    {
        await Task.Delay(1000, ct);
    }
}
```

### destroyCancellationToken (Unity 2022.2+)

```csharp
public class DestroyCancellationTokenExample : MonoBehaviour
{
    async void Start()
    {
        try
        {
            // MonoBehaviour의 destroyCancellationToken 사용
            // GameObject 파괴 시 자동 취소
            await LongOperationAsync(destroyCancellationToken);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("오브젝트 파괴로 취소됨");
        }
    }

    async Task LongOperationAsync(CancellationToken ct)
    {
        while (true)
        {
            ct.ThrowIfCancellationRequested();
            await Task.Delay(1000, ct);
            Debug.Log("Still running...");
        }
    }

    // OnDestroy에서 별도 취소 불필요!
}
```

### Scene 전환

```csharp
public class SceneTransitionCancellation : MonoBehaviour
{
    private static CancellationTokenSource _sceneLoadCts;

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    static void Initialize()
    {
        SceneManager.sceneUnloaded += OnSceneUnloaded;
    }

    static void OnSceneUnloaded(Scene scene)
    {
        // Scene 언로드 시 모든 작업 취소
        _sceneLoadCts?.Cancel();
        _sceneLoadCts?.Dispose();
        _sceneLoadCts = new CancellationTokenSource();
    }

    public static CancellationToken SceneToken => _sceneLoadCts?.Token ?? CancellationToken.None;

    // 사용 예
    async void Start()
    {
        try
        {
            await LoadSceneDataAsync(SceneToken);
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

---

## UniTask와 CancellationToken

### UniTask에서의 사용

```csharp
using Cysharp.Threading.Tasks;

public class UniTaskCancellation : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        try
        {
            // destroyCancellationToken 자동 지원
            await UniTask.Delay(5000, cancellationToken: destroyCancellationToken);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("취소됨");
        }
    }
}
```

### GetCancellationTokenOnDestroy

```csharp
using Cysharp.Threading.Tasks;

public class UniTaskCancellationExtension : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // UniTask의 확장 메서드
        var ct = this.GetCancellationTokenOnDestroy();

        try
        {
            await LongOperationAsync(ct);
        }
        catch (OperationCanceledException)
        {
            return; // 정상 종료
        }
    }

    async UniTask LongOperationAsync(CancellationToken ct)
    {
        await UniTask.Delay(10000, cancellationToken: ct);
    }
}
```

### SuppressCancellationThrow

```csharp
using Cysharp.Threading.Tasks;

public class SuppressCancellation : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // 취소 시 예외 대신 false 반환
        var (isCanceled, result) = await LoadDataAsync()
            .SuppressCancellationThrow();

        if (isCanceled)
        {
            Debug.Log("취소됨 - 예외 없이 처리");
            return;
        }

        Debug.Log($"결과: {result}");
    }

    async UniTask<string> LoadDataAsync()
    {
        await UniTask.Delay(1000, cancellationToken: destroyCancellationToken);
        return "Data";
    }
}
```

---

## 타임아웃 패턴

### 타임아웃 구현

```csharp
public class TimeoutPatterns : MonoBehaviour
{
    async void Start()
    {
        // 패턴 1: CancellationTokenSource 타임아웃
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        try
        {
            await SlowOperationAsync(cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("타임아웃!");
        }

        // 패턴 2: Task.WhenAny
        try
        {
            await WithTimeout(SlowOperationAsync(CancellationToken.None), TimeSpan.FromSeconds(5));
        }
        catch (TimeoutException)
        {
            Debug.Log("타임아웃!");
        }
    }

    async Task<T> WithTimeout<T>(Task<T> task, TimeSpan timeout)
    {
        var timeoutTask = Task.Delay(timeout);
        var completed = await Task.WhenAny(task, timeoutTask);

        if (completed == timeoutTask)
        {
            throw new TimeoutException();
        }

        return await task;
    }

    async Task WithTimeout(Task task, TimeSpan timeout)
    {
        var timeoutTask = Task.Delay(timeout);
        var completed = await Task.WhenAny(task, timeoutTask);

        if (completed == timeoutTask)
        {
            throw new TimeoutException();
        }

        await task;
    }

    async Task SlowOperationAsync(CancellationToken ct)
    {
        await Task.Delay(10000, ct);
    }

    async Task<string> SlowOperationAsync(CancellationToken ct, bool dummy)
    {
        await Task.Delay(10000, ct);
        return "result";
    }
}
```

### 연결된 타임아웃

```csharp
public class LinkedTimeout : MonoBehaviour
{
    private CancellationTokenSource _operationCts;

    async void Start()
    {
        _operationCts = new CancellationTokenSource();

        // 작업 취소 토큰 + 타임아웃
        using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            _operationCts.Token,
            timeoutCts.Token);

        try
        {
            await OperationAsync(linkedCts.Token);
        }
        catch (OperationCanceledException)
        {
            if (timeoutCts.IsCancellationRequested)
            {
                Debug.Log("타임아웃으로 취소");
            }
            else
            {
                Debug.Log("사용자가 취소");
            }
        }
    }

    async Task OperationAsync(CancellationToken ct)
    {
        await Task.Delay(15000, ct);
    }

    public void CancelOperation()
    {
        _operationCts?.Cancel();
    }
}
```

---

## 실전 패턴

### 패턴 1: 재사용 가능한 CTS

```csharp
public class ReusableCTS : MonoBehaviour
{
    private CancellationTokenSource _cts;

    void RestartOperation()
    {
        // 기존 작업 취소
        _cts?.Cancel();
        _cts?.Dispose();

        // 새 CTS 생성
        _cts = new CancellationTokenSource();

        // 새 작업 시작
        _ = DoWorkAsync(_cts.Token);
    }

    async Task DoWorkAsync(CancellationToken ct)
    {
        try
        {
            await Task.Delay(10000, ct);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("이전 작업 취소됨");
        }
    }

    void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

### 패턴 2: 취소 가능한 다운로드

```csharp
public class CancellableDownload : MonoBehaviour
{
    [SerializeField] private Button _downloadButton;
    [SerializeField] private Button _cancelButton;
    [SerializeField] private Slider _progressBar;

    private CancellationTokenSource _downloadCts;

    void Start()
    {
        _downloadButton.onClick.AddListener(StartDownload);
        _cancelButton.onClick.AddListener(CancelDownload);
    }

    async void StartDownload()
    {
        _downloadCts?.Cancel();
        _downloadCts?.Dispose();
        _downloadCts = new CancellationTokenSource();

        _downloadButton.interactable = false;
        _cancelButton.interactable = true;

        try
        {
            await DownloadFileAsync(
                "https://example.com/largefile.zip",
                _downloadCts.Token);

            Debug.Log("다운로드 완료!");
        }
        catch (OperationCanceledException)
        {
            Debug.Log("다운로드 취소됨");
            _progressBar.value = 0;
        }
        finally
        {
            _downloadButton.interactable = true;
            _cancelButton.interactable = false;
        }
    }

    void CancelDownload()
    {
        _downloadCts?.Cancel();
    }

    async Task DownloadFileAsync(string url, CancellationToken ct)
    {
        using var client = new HttpClient();

        using var response = await client.GetAsync(
            url,
            HttpCompletionOption.ResponseHeadersRead,
            ct);

        var totalBytes = response.Content.Headers.ContentLength ?? -1;
        var receivedBytes = 0L;

        using var stream = await response.Content.ReadAsStreamAsync();
        var buffer = new byte[8192];
        int bytesRead;

        while ((bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length, ct)) > 0)
        {
            receivedBytes += bytesRead;

            if (totalBytes > 0)
            {
                _progressBar.value = (float)receivedBytes / totalBytes;
            }
        }
    }

    void OnDestroy()
    {
        _downloadCts?.Cancel();
        _downloadCts?.Dispose();
    }
}
```

### 패턴 3: 디바운스 검색

```csharp
public class DebouncedSearch : MonoBehaviour
{
    [SerializeField] private InputField _searchInput;

    private CancellationTokenSource _searchCts;

    void Start()
    {
        _searchInput.onValueChanged.AddListener(OnSearchTextChanged);
    }

    async void OnSearchTextChanged(string text)
    {
        // 이전 검색 취소
        _searchCts?.Cancel();
        _searchCts?.Dispose();
        _searchCts = new CancellationTokenSource();

        try
        {
            // 디바운스 대기
            await Task.Delay(300, _searchCts.Token);

            // 검색 실행
            if (text.Length >= 2)
            {
                var results = await SearchAsync(text, _searchCts.Token);
                DisplayResults(results);
            }
        }
        catch (OperationCanceledException)
        {
            // 새 입력으로 취소됨 - 정상
        }
    }

    async Task<List<string>> SearchAsync(string query, CancellationToken ct)
    {
        await Task.Delay(500, ct); // API 호출 시뮬레이션
        return new List<string> { $"Result for {query}" };
    }

    void DisplayResults(List<string> results)
    {
        foreach (var result in results)
        {
            Debug.Log(result);
        }
    }

    void OnDestroy()
    {
        _searchCts?.Cancel();
        _searchCts?.Dispose();
    }
}
```

---

## 주의사항

### Dispose 패턴

```csharp
public class CTSDisposal : MonoBehaviour
{
    private CancellationTokenSource _cts;

    // ✅ 올바른 패턴
    void OnDestroy()
    {
        _cts?.Cancel();  // 먼저 취소
        _cts?.Dispose(); // 그 다음 해제
    }

    // ❌ 잘못된 패턴 - Cancel 없이 Dispose만
    void BadOnDestroy()
    {
        _cts?.Dispose(); // 진행 중인 작업이 완료될 때까지 블로킹될 수 있음
    }
}
```

### 재사용 금지

```csharp
public class CTSReuse : MonoBehaviour
{
    private CancellationTokenSource _cts;

    // ❌ 취소된 CTS 재사용 금지
    void Bad()
    {
        _cts = new CancellationTokenSource();
        _cts.Cancel();

        // 이미 취소된 CTS의 토큰 - 즉시 취소됨
        var token = _cts.Token; // token.IsCancellationRequested == true
    }

    // ✅ 새 CTS 생성
    void Good()
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _cts = new CancellationTokenSource(); // 새로 생성
    }
}
```

---

## 정리

### CancellationToken 요약

| 항목 | 내용 |
|------|------|
| **CancellationTokenSource** | 취소 신호 발생 |
| **CancellationToken** | 취소 확인용 토큰 |
| **ThrowIfCancellationRequested** | 취소 시 예외 발생 |
| **IsCancellationRequested** | 취소 여부 확인 |
| **Register** | 취소 콜백 등록 |

### 체크리스트

- [ ] CancellationTokenSource를 필드로 저장하는가?
- [ ] OnDestroy에서 Cancel + Dispose 호출하는가?
- [ ] 비동기 메서드에 CancellationToken 전달하는가?
- [ ] OperationCanceledException을 적절히 처리하는가?
- [ ] 타임아웃이 필요한 경우 설정했는가?

---

## 참고 자료

- [CancellationToken - Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)
- [Task Cancellation](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-cancellation)
- [UniTask Cancellation](https://github.com/Cysharp/UniTask#cancellation-and-exception-handling)
