# 12. Awaitable (Unity 2023+)

## 개요

`Awaitable`은 **Unity 2023.1**에서 도입된 **Unity 네이티브 async/await 지원**입니다. Unity 엔진에 최적화된 비동기 프로그래밍을 별도의 외부 라이브러리 없이 사용할 수 있습니다.

---

## Awaitable의 특징

### 핵심 특징

| 특징 | 설명 |
|------|------|
| **Unity 네이티브** | 엔진에 내장, 외부 패키지 불필요 |
| **풀링 기반** | 객체 풀을 사용해 GC 부담 최소화 |
| **Main Thread 자동 복귀** | await 후 자동으로 Main Thread에서 계속 |
| **PlayerLoop 통합** | Unity 라이프사이클과 자연스럽게 연동 |
| **Coroutine 대체** | 기존 Coroutine의 현대적 대안 |

### Task/UniTask와 비교

| 기능 | Task | UniTask | Awaitable |
|------|------|---------|-----------|
| Unity 최적화 | ❌ | ✅ | ✅ |
| GC 할당 | 많음 | 최소화 | 최소화 |
| 외부 패키지 | 불필요 | 필요 | 불필요 |
| Main Thread 복귀 | 수동 | 자동 | 자동 |
| 취소 지원 | ✅ | ✅ | ✅ |
| 지원 버전 | 모든 버전 | 모든 버전 | 2023.1+ |

---

## 기본 사용법

### Awaitable 반환 메서드

```csharp
// Unity 2023.1+
using UnityEngine;

public class AwaitableBasics : MonoBehaviour
{
    async void Start()
    {
        Debug.Log("Start 시작");

        // Awaitable 대기
        await DoSomethingAsync();

        Debug.Log("완료 - 여기는 Main Thread");
        transform.position = Vector3.one; // 안전!
    }

    // Awaitable 반환 타입
    async Awaitable DoSomethingAsync()
    {
        Debug.Log("작업 시작");
        await Awaitable.WaitForSecondsAsync(1f);
        Debug.Log("1초 후");
    }
}
```

### Awaitable<T> (값 반환)

```csharp
public class AwaitableWithResult : MonoBehaviour
{
    async void Start()
    {
        // 값을 반환하는 Awaitable
        int result = await CalculateAsync();
        Debug.Log($"결과: {result}");

        // 여러 값 가져오기
        var playerData = await LoadPlayerDataAsync();
        Debug.Log($"Player: {playerData.Name}, Level: {playerData.Level}");
    }

    async Awaitable<int> CalculateAsync()
    {
        await Awaitable.WaitForSecondsAsync(0.5f);
        return 42;
    }

    async Awaitable<PlayerData> LoadPlayerDataAsync()
    {
        await Awaitable.WaitForSecondsAsync(1f);
        return new PlayerData { Name = "Hero", Level = 10 };
    }
}

public class PlayerData
{
    public string Name { get; set; }
    public int Level { get; set; }
}
```

---

## Awaitable 정적 메서드

### 시간 대기

```csharp
public class AwaitableTimeWait : MonoBehaviour
{
    async void Start()
    {
        // 실제 시간(Unscaled) 대기
        Debug.Log("시작: " + Time.time);
        await Awaitable.WaitForSecondsAsync(2f);
        Debug.Log("2초 후: " + Time.time);

        // 취소 토큰과 함께
        var cts = new CancellationTokenSource();
        cts.CancelAfter(1000); // 1초 후 취소

        try
        {
            await Awaitable.WaitForSecondsAsync(5f, cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("대기가 취소됨");
        }
    }
}
```

### 다음 프레임 대기

```csharp
public class AwaitableNextFrame : MonoBehaviour
{
    async void Start()
    {
        Debug.Log($"Frame {Time.frameCount}");

        // 다음 프레임까지 대기
        await Awaitable.NextFrameAsync();

        Debug.Log($"Frame {Time.frameCount}"); // +1

        // 여러 프레임 대기
        for (int i = 0; i < 10; i++)
        {
            await Awaitable.NextFrameAsync();
            Debug.Log($"Frame {Time.frameCount}");
        }
    }
}
```

### EndOfFrame 대기

```csharp
public class AwaitableEndOfFrame : MonoBehaviour
{
    async void Start()
    {
        // 렌더링 후까지 대기 (스크린샷 촬영에 유용)
        await Awaitable.EndOfFrameAsync();

        // 스크린샷 촬영
        var texture = ScreenCapture.CaptureScreenshotAsTexture();
        Debug.Log($"Screenshot captured: {texture.width}x{texture.height}");
    }
}
```

### FixedUpdate 대기

```csharp
public class AwaitableFixedUpdate : MonoBehaviour
{
    [SerializeField] private Rigidbody _rigidbody;

    async void Start()
    {
        // FixedUpdate까지 대기
        await Awaitable.FixedUpdateAsync();

        // 물리 연산 안전하게 수행
        _rigidbody.AddForce(Vector3.up * 10f, ForceMode.Impulse);
    }

    async Awaitable MoveWithPhysicsAsync()
    {
        for (int i = 0; i < 100; i++)
        {
            await Awaitable.FixedUpdateAsync();
            _rigidbody.MovePosition(_rigidbody.position + Vector3.forward * 0.1f);
        }
    }
}
```

---

## 스레드 전환

### BackgroundThreadAsync / MainThreadAsync

```csharp
public class AwaitableThreadSwitching : MonoBehaviour
{
    async void Start()
    {
        Debug.Log($"1. Main Thread: {Thread.CurrentThread.ManagedThreadId}");

        // 백그라운드 스레드로 전환
        await Awaitable.BackgroundThreadAsync();
        Debug.Log($"2. Background: {Thread.CurrentThread.ManagedThreadId}");

        // CPU 집약적 작업
        var result = PerformHeavyCalculation();

        // Main Thread로 복귀
        await Awaitable.MainThreadAsync();
        Debug.Log($"3. Main Thread: {Thread.CurrentThread.ManagedThreadId}");

        // Unity API 사용 가능
        transform.position = new Vector3(result, 0, 0);
    }

    float PerformHeavyCalculation()
    {
        // 무거운 계산 시뮬레이션
        float sum = 0;
        for (int i = 0; i < 10000000; i++)
        {
            sum += Mathf.Sin(i * 0.001f);
        }
        return sum;
    }
}
```

### 스레드 전환 패턴

```csharp
public class ThreadSwitchingPatterns : MonoBehaviour
{
    [SerializeField] private Text statusText;

    async void Start()
    {
        statusText.text = "Processing...";

        // 병렬 처리
        var results = await ProcessDataInBackgroundAsync();

        // 결과 적용 (Main Thread)
        statusText.text = $"Completed: {results.Length} items";
    }

    async Awaitable<int[]> ProcessDataInBackgroundAsync()
    {
        // 백그라운드로 전환
        await Awaitable.BackgroundThreadAsync();

        // 병렬 처리
        var data = Enumerable.Range(0, 1000).ToArray();
        var results = new int[data.Length];

        Parallel.For(0, data.Length, i =>
        {
            results[i] = ExpensiveCalculation(data[i]);
        });

        // 결과 반환 전에 Main Thread로 복귀할 필요 없음
        // 호출자가 await 후 자동으로 Main Thread
        return results;
    }

    int ExpensiveCalculation(int value)
    {
        // 무거운 계산
        return value * value;
    }
}
```

---

## 취소 처리

### CancellationToken 사용

```csharp
public class AwaitableCancellation : MonoBehaviour
{
    private CancellationTokenSource _cts;

    async void Start()
    {
        _cts = new CancellationTokenSource();

        try
        {
            await LongRunningOperationAsync(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 취소되었습니다");
        }
    }

    void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }

    async Awaitable LongRunningOperationAsync(CancellationToken cancellationToken)
    {
        for (int i = 0; i < 100; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();

            Debug.Log($"Processing {i}...");
            await Awaitable.WaitForSecondsAsync(0.1f, cancellationToken);
        }
    }
}
```

### destroyCancellationToken

```csharp
public class AwaitableDestroyCancellation : MonoBehaviour
{
    async void Start()
    {
        try
        {
            // MonoBehaviour의 destroyCancellationToken 사용
            await ProcessAsync(destroyCancellationToken);
        }
        catch (OperationCanceledException)
        {
            // 오브젝트 파괴 시 자동 취소
            Debug.Log("GameObject가 파괴되어 작업 취소됨");
        }
    }

    async Awaitable ProcessAsync(CancellationToken cancellationToken)
    {
        while (true)
        {
            await Awaitable.WaitForSecondsAsync(1f, cancellationToken);
            Debug.Log("Processing...");
        }
    }
}
```

### 타임아웃 구현

```csharp
public class AwaitableTimeout : MonoBehaviour
{
    async void Start()
    {
        try
        {
            await WithTimeoutAsync(LoadDataAsync(), TimeSpan.FromSeconds(5));
            Debug.Log("데이터 로드 완료");
        }
        catch (TimeoutException)
        {
            Debug.LogError("타임아웃 발생!");
        }
    }

    async Awaitable WithTimeoutAsync(Awaitable operation, TimeSpan timeout)
    {
        using var cts = new CancellationTokenSource();
        cts.CancelAfter(timeout);

        try
        {
            // 타임아웃과 함께 대기
            var timeoutTask = Awaitable.WaitForSecondsAsync((float)timeout.TotalSeconds, cts.Token);

            // 먼저 완료되는 것 확인
            // 참고: Awaitable에는 WhenAny가 없으므로 다른 방식 필요
            await operation;
            cts.Cancel(); // 작업 완료 시 타임아웃 취소
        }
        catch (OperationCanceledException)
        {
            throw new TimeoutException($"작업이 {timeout.TotalSeconds}초 내에 완료되지 않았습니다.");
        }
    }

    async Awaitable LoadDataAsync()
    {
        await Awaitable.WaitForSecondsAsync(3f); // 3초 소요
    }
}
```

---

## AwaitableCompletionSource

### 기본 사용법

```csharp
public class AwaitableCompletionSourceExample : MonoBehaviour
{
    private AwaitableCompletionSource _completionSource;

    void Start()
    {
        StartCoroutine(WaitAndComplete());
        WaitForCompletionAsync();
    }

    async void WaitForCompletionAsync()
    {
        _completionSource = new AwaitableCompletionSource();

        Debug.Log("완료 대기 중...");
        await _completionSource.Awaitable;
        Debug.Log("완료됨!");
    }

    IEnumerator WaitAndComplete()
    {
        yield return new WaitForSeconds(2f);

        // 외부에서 완료 신호
        _completionSource.SetResult();
    }
}
```

### 값 반환

```csharp
public class AwaitableCompletionSourceWithValue : MonoBehaviour
{
    private AwaitableCompletionSource<int> _completionSource;

    async void Start()
    {
        _completionSource = new AwaitableCompletionSource<int>();

        // 다른 곳에서 결과 설정
        Invoke(nameof(SetResultDelayed), 2f);

        // 결과 대기
        int result = await _completionSource.Awaitable;
        Debug.Log($"받은 결과: {result}");
    }

    void SetResultDelayed()
    {
        _completionSource.SetResult(42);
    }
}
```

### 예외 전파

```csharp
public class AwaitableCompletionSourceException : MonoBehaviour
{
    private AwaitableCompletionSource<string> _completionSource;

    async void Start()
    {
        _completionSource = new AwaitableCompletionSource<string>();

        // 예외 설정
        Invoke(nameof(SetExceptionDelayed), 1f);

        try
        {
            var result = await _completionSource.Awaitable;
        }
        catch (InvalidOperationException ex)
        {
            Debug.LogError($"예외 발생: {ex.Message}");
        }
    }

    void SetExceptionDelayed()
    {
        _completionSource.SetException(new InvalidOperationException("작업 실패!"));
    }
}
```

---

## Coroutine에서 Awaitable로 마이그레이션

### 기존 Coroutine 코드

```csharp
// 기존 방식 (Coroutine)
public class CoroutineExample : MonoBehaviour
{
    private Coroutine _runningCoroutine;

    void Start()
    {
        _runningCoroutine = StartCoroutine(LoadResourcesCoroutine());
    }

    IEnumerator LoadResourcesCoroutine()
    {
        Debug.Log("로딩 시작...");

        yield return new WaitForSeconds(1f);
        Debug.Log("리소스 1 로드");

        yield return new WaitForSeconds(1f);
        Debug.Log("리소스 2 로드");

        yield return new WaitForEndOfFrame();
        Debug.Log("프레임 끝에서 처리");
    }

    void OnDestroy()
    {
        if (_runningCoroutine != null)
        {
            StopCoroutine(_runningCoroutine);
        }
    }
}
```

### Awaitable로 변환

```csharp
// 현대적 방식 (Awaitable)
public class AwaitableExample : MonoBehaviour
{
    private CancellationTokenSource _cts;

    async void Start()
    {
        _cts = new CancellationTokenSource();

        try
        {
            await LoadResourcesAsync(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("로딩 취소됨");
        }
    }

    async Awaitable LoadResourcesAsync(CancellationToken cancellationToken)
    {
        Debug.Log("로딩 시작...");

        await Awaitable.WaitForSecondsAsync(1f, cancellationToken);
        Debug.Log("리소스 1 로드");

        await Awaitable.WaitForSecondsAsync(1f, cancellationToken);
        Debug.Log("리소스 2 로드");

        await Awaitable.EndOfFrameAsync();
        Debug.Log("프레임 끝에서 처리");
    }

    void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

### 마이그레이션 매핑 테이블

| Coroutine | Awaitable |
|-----------|-----------|
| `yield return null` | `await Awaitable.NextFrameAsync()` |
| `yield return new WaitForSeconds(n)` | `await Awaitable.WaitForSecondsAsync(n)` |
| `yield return new WaitForSecondsRealtime(n)` | `await Awaitable.WaitForSecondsAsync(n)` |
| `yield return new WaitForEndOfFrame()` | `await Awaitable.EndOfFrameAsync()` |
| `yield return new WaitForFixedUpdate()` | `await Awaitable.FixedUpdateAsync()` |
| `yield return new WaitUntil(() => cond)` | `while (!cond) await Awaitable.NextFrameAsync()` |
| `yield return new WaitWhile(() => cond)` | `while (cond) await Awaitable.NextFrameAsync()` |
| `StartCoroutine()` 취소 | `CancellationToken` 사용 |

---

## 실전 패턴

### 패턴 1: 시퀀스 애니메이션

```csharp
public class SequenceAnimation : MonoBehaviour
{
    [SerializeField] private Transform _target;
    [SerializeField] private float _duration = 1f;

    async void Start()
    {
        try
        {
            // 연속 애니메이션
            await MoveToAsync(new Vector3(5, 0, 0), destroyCancellationToken);
            await ScaleToAsync(Vector3.one * 2f, destroyCancellationToken);
            await RotateToAsync(Quaternion.Euler(0, 180, 0), destroyCancellationToken);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("애니메이션 취소됨");
        }
    }

    async Awaitable MoveToAsync(Vector3 target, CancellationToken cancellationToken)
    {
        Vector3 startPos = _target.position;
        float elapsed = 0f;

        while (elapsed < _duration)
        {
            cancellationToken.ThrowIfCancellationRequested();

            elapsed += Time.deltaTime;
            float t = Mathf.Clamp01(elapsed / _duration);
            t = Mathf.SmoothStep(0, 1, t); // 이징

            _target.position = Vector3.Lerp(startPos, target, t);
            await Awaitable.NextFrameAsync();
        }

        _target.position = target;
    }

    async Awaitable ScaleToAsync(Vector3 target, CancellationToken cancellationToken)
    {
        Vector3 startScale = _target.localScale;
        float elapsed = 0f;

        while (elapsed < _duration)
        {
            cancellationToken.ThrowIfCancellationRequested();

            elapsed += Time.deltaTime;
            float t = Mathf.Clamp01(elapsed / _duration);

            _target.localScale = Vector3.Lerp(startScale, target, t);
            await Awaitable.NextFrameAsync();
        }

        _target.localScale = target;
    }

    async Awaitable RotateToAsync(Quaternion target, CancellationToken cancellationToken)
    {
        Quaternion startRot = _target.rotation;
        float elapsed = 0f;

        while (elapsed < _duration)
        {
            cancellationToken.ThrowIfCancellationRequested();

            elapsed += Time.deltaTime;
            float t = Mathf.Clamp01(elapsed / _duration);

            _target.rotation = Quaternion.Slerp(startRot, target, t);
            await Awaitable.NextFrameAsync();
        }

        _target.rotation = target;
    }
}
```

### 패턴 2: 리소스 로딩

```csharp
public class ResourceLoader : MonoBehaviour
{
    [SerializeField] private Slider _progressBar;
    [SerializeField] private Text _statusText;

    async void Start()
    {
        try
        {
            await LoadAllResourcesAsync(destroyCancellationToken);
            _statusText.text = "로딩 완료!";
        }
        catch (OperationCanceledException)
        {
            _statusText.text = "로딩 취소됨";
        }
    }

    async Awaitable LoadAllResourcesAsync(CancellationToken cancellationToken)
    {
        var resources = new[] { "Textures/UI", "Prefabs/Characters", "Audio/Music" };

        for (int i = 0; i < resources.Length; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();

            _statusText.text = $"Loading {resources[i]}...";
            _progressBar.value = (float)i / resources.Length;

            await LoadResourceAsync(resources[i], cancellationToken);
        }

        _progressBar.value = 1f;
    }

    async Awaitable LoadResourceAsync(string path, CancellationToken cancellationToken)
    {
        var request = Resources.LoadAsync(path);

        while (!request.isDone)
        {
            cancellationToken.ThrowIfCancellationRequested();
            await Awaitable.NextFrameAsync();
        }
    }
}
```

### 패턴 3: 씬 전환

```csharp
using UnityEngine.SceneManagement;

public class SceneTransition : MonoBehaviour
{
    [SerializeField] private CanvasGroup _fadeOverlay;
    [SerializeField] private float _fadeDuration = 0.5f;

    public async Awaitable TransitionToSceneAsync(string sceneName)
    {
        // Fade Out
        await FadeAsync(0f, 1f, destroyCancellationToken);

        // 씬 로드
        var operation = SceneManager.LoadSceneAsync(sceneName);
        operation.allowSceneActivation = false;

        // 로딩 대기
        while (operation.progress < 0.9f)
        {
            await Awaitable.NextFrameAsync();
        }

        // 씬 활성화
        operation.allowSceneActivation = true;

        // 씬 로드 완료 대기
        while (!operation.isDone)
        {
            await Awaitable.NextFrameAsync();
        }

        // Fade In
        await FadeAsync(1f, 0f, destroyCancellationToken);
    }

    async Awaitable FadeAsync(float from, float to, CancellationToken cancellationToken)
    {
        float elapsed = 0f;
        _fadeOverlay.alpha = from;

        while (elapsed < _fadeDuration)
        {
            cancellationToken.ThrowIfCancellationRequested();

            elapsed += Time.deltaTime;
            float t = elapsed / _fadeDuration;
            _fadeOverlay.alpha = Mathf.Lerp(from, to, t);

            await Awaitable.NextFrameAsync();
        }

        _fadeOverlay.alpha = to;
    }
}
```

### 패턴 4: 입력 대기

```csharp
public class InputWaiter : MonoBehaviour
{
    async void Start()
    {
        Debug.Log("아무 키나 누르세요...");
        await WaitForAnyKeyAsync(destroyCancellationToken);

        Debug.Log("키 입력 감지! 마우스 클릭을 기다립니다...");
        var clickPosition = await WaitForMouseClickAsync(destroyCancellationToken);

        Debug.Log($"클릭 위치: {clickPosition}");
    }

    async Awaitable WaitForAnyKeyAsync(CancellationToken cancellationToken)
    {
        while (!Input.anyKeyDown)
        {
            cancellationToken.ThrowIfCancellationRequested();
            await Awaitable.NextFrameAsync();
        }
    }

    async Awaitable<Vector3> WaitForMouseClickAsync(CancellationToken cancellationToken)
    {
        while (!Input.GetMouseButtonDown(0))
        {
            cancellationToken.ThrowIfCancellationRequested();
            await Awaitable.NextFrameAsync();
        }

        return Input.mousePosition;
    }
}
```

---

## Awaitable vs UniTask 비교

### 기능 비교

| 기능 | Awaitable | UniTask |
|------|-----------|---------|
| Unity 내장 | ✅ | ❌ (외부 패키지) |
| 지원 버전 | 2023.1+ | 모든 버전 |
| PlayerLoopTiming | 제한적 | 세밀한 제어 |
| WhenAll/WhenAny | ❌ | ✅ |
| Channel | ❌ | ✅ |
| AsyncReactiveProperty | ❌ | ✅ |
| DOTween 통합 | ❌ | ✅ |
| 기능 풍부함 | 기본적 | 매우 풍부 |

### 선택 기준

```csharp
// Awaitable 선택 시
// - Unity 2023.1 이상
// - 간단한 비동기 작업
// - 외부 의존성 최소화 원할 때
// - 기본적인 async/await만 필요할 때

// UniTask 선택 시
// - Unity 2022 이하 지원 필요
// - 복잡한 비동기 조합 (WhenAll, WhenAny 등)
// - 세밀한 PlayerLoopTiming 제어 필요
// - AsyncReactiveProperty 등 고급 기능 필요
```

### 함께 사용하기

```csharp
using Cysharp.Threading.Tasks;

public class AwaitableAndUniTask : MonoBehaviour
{
    async void Start()
    {
        // Awaitable 사용
        await Awaitable.WaitForSecondsAsync(1f);

        // UniTask 사용
        await UniTask.Delay(1000);

        // 혼용 가능 (같은 프로젝트에서)
        await LoadWithAwaitableAsync();
        await ProcessWithUniTaskAsync();
    }

    async Awaitable LoadWithAwaitableAsync()
    {
        await Awaitable.WaitForSecondsAsync(1f);
    }

    async UniTask ProcessWithUniTaskAsync()
    {
        await UniTask.Yield(PlayerLoopTiming.PostLateUpdate);
    }
}
```

---

## 주의사항

### Awaitable 풀링 규칙

```csharp
public class AwaitablePrecautions : MonoBehaviour
{
    // ❌ Awaitable을 저장하지 말 것
    // private Awaitable _storedAwaitable; // 위험!

    // ❌ 여러 번 await 하지 말 것
    async void Bad_MultipleAwait()
    {
        var awaitable = DoSomethingAsync();
        // await awaitable;
        // await awaitable; // 위험! 풀로 반환된 객체
    }

    // ✅ 즉시 await
    async void Good_ImmediateAwait()
    {
        await DoSomethingAsync();
    }

    async Awaitable DoSomethingAsync()
    {
        await Awaitable.NextFrameAsync();
    }
}
```

### Main Thread 보장

```csharp
public class MainThreadGuarantee : MonoBehaviour
{
    async void Start()
    {
        // BackgroundThreadAsync 후에는 반드시 MainThreadAsync 호출
        await Awaitable.BackgroundThreadAsync();

        // 여기는 백그라운드 스레드
        // transform.position = Vector3.zero; // 위험!

        await Awaitable.MainThreadAsync();

        // 안전
        transform.position = Vector3.zero;
    }
}
```

### destroyCancellationToken 사용

```csharp
public class DestroyCancellationBestPractice : MonoBehaviour
{
    // ✅ destroyCancellationToken 적극 활용
    async void Start()
    {
        try
        {
            // 오브젝트 파괴 시 자동 취소
            await LongOperationAsync(destroyCancellationToken);
        }
        catch (OperationCanceledException)
        {
            // 정리 로직 (필요 시)
        }
    }

    async Awaitable LongOperationAsync(CancellationToken ct)
    {
        while (true)
        {
            ct.ThrowIfCancellationRequested();
            await Awaitable.WaitForSecondsAsync(1f, ct);
        }
    }
}
```

---

## 정리

### Awaitable 요약

| 항목 | 내용 |
|------|------|
| **도입 버전** | Unity 2023.1 |
| **핵심 장점** | Unity 네이티브, 외부 의존성 없음, GC 최적화 |
| **주요 메서드** | WaitForSecondsAsync, NextFrameAsync, EndOfFrameAsync |
| **스레드 전환** | BackgroundThreadAsync, MainThreadAsync |
| **취소 지원** | CancellationToken, destroyCancellationToken |
| **제한사항** | 한 번만 await, WhenAll/WhenAny 미지원 |

### 체크리스트

- [ ] Unity 2023.1 이상에서만 사용
- [ ] Awaitable을 변수에 저장하지 않기
- [ ] destroyCancellationToken으로 생명주기 관리
- [ ] 백그라운드 스레드에서 Unity API 호출 금지
- [ ] 복잡한 조합이 필요하면 UniTask 고려

---

## 참고 자료

- [Unity 2023.1 Release Notes - Awaitable](https://unity.com/releases/editor/whats-new/2023.1.0)
- [Unity Documentation - Awaitable](https://docs.unity3d.com/2023.1/Documentation/ScriptReference/Awaitable.html)
- [Unity Blog - Async/Await Support](https://blog.unity.com/technology/async-await-support-in-unity)
- [AwaitableCompletionSource API](https://docs.unity3d.com/2023.1/Documentation/ScriptReference/AwaitableCompletionSource.html)
