# 34. CancellationToken

## 개요

`CancellationToken`은 .NET에서 제공하는 **협력적 취소(Cooperative Cancellation)** 패턴의 핵심 구성 요소입니다. 비동기 작업이나 장시간 실행되는 작업을 안전하고 구조적으로 취소할 수 있는 메커니즘을 제공합니다.

"협력적"이라는 의미는, 취소를 요청하는 쪽과 취소를 수행하는 쪽이 **서로 협력**한다는 뜻입니다. 작업을 즉시 강제 종료하는 것이 아니라, 작업 수행 코드가 스스로 취소 여부를 확인하고 적절한 시점에 정리 작업을 수행한 뒤 종료합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                   협력적 취소 패턴 구조                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CancellationTokenSource          CancellationToken              │
│  (취소 요청자)                     (취소 감시자)                  │
│  ┌──────────────┐                ┌──────────────────┐           │
│  │  Cancel()    │───생성──────▶  │ IsCancellation   │           │
│  │  CancelAfter│                │ Requested         │           │
│  │  Dispose()  │                │ ThrowIfCancell... │           │
│  └──────────────┘                │ Register()        │           │
│                                   └────────┬─────────┘           │
│                                            │                     │
│                                   ┌────────▼─────────┐           │
│                                   │  비동기 작업       │           │
│                                   │  (토큰을 전달받아  │           │
│                                   │   취소 여부 확인)  │           │
│                                   └──────────────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 핵심 구성 요소

| 구성 요소 | 역할 | 설명 |
|-----------|------|------|
| **CancellationTokenSource** | 취소 신호 발생 | `Cancel()` 호출로 취소 요청 |
| **CancellationToken** | 취소 신호 전달 | 작업에 전달되어 취소 여부를 모니터링 |
| **OperationCanceledException** | 취소 통보 | 취소 시 발생하는 예외 |

---

## 1. CancellationTokenSource 생성 및 관리

`CancellationTokenSource`(이하 CTS)는 취소 신호를 생성하고 제어하는 객체입니다. `IDisposable`을 구현하므로 사용 후 반드시 해제해야 합니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class CancellationTokenSourceBasicsExample : MonoBehaviour
{
    private CancellationTokenSource _cts;

    private async void Start()
    {
        _cts = new CancellationTokenSource();
        CancellationToken token = _cts.Token;

        // 취소 전 상태 확인
        Debug.Log($"취소 요청됨: {token.IsCancellationRequested}"); // false
        Debug.Log($"취소 가능: {token.CanBeCanceled}");             // true

        try
        {
            await LongRunningOperationAsync(token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 취소되었습니다.");
        }
    }

    private async Task LongRunningOperationAsync(CancellationToken token)
    {
        for (int i = 0; i < 100; i++)
        {
            // 취소 요청 확인 - 취소 시 OperationCanceledException 발생
            token.ThrowIfCancellationRequested();

            Debug.Log($"작업 진행 중: {i}%");
            await Task.Delay(100, token);
        }
    }

    // 버튼 등에서 호출하여 취소
    public void CancelOperation() => _cts?.Cancel();

    private void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

> **참고:** `CancellationToken.None`은 절대 취소되지 않는 토큰입니다. `CanBeCanceled`가 `false`를 반환합니다.

---

## 2. Token 전파 패턴

CancellationToken은 메서드 호출 체인을 통해 전파됩니다. 비동기 메서드의 **마지막 매개변수**로 `CancellationToken`을 받는 것이 표준 패턴입니다.

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class TokenPropagationExample : MonoBehaviour
{
    private CancellationTokenSource _cts;

    private async void Start()
    {
        _cts = new CancellationTokenSource();

        try
        {
            // 최상위에서 토큰을 생성하고 체인 전체에 전파
            await ProcessGameDataAsync(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("전체 작업이 취소되었습니다.");
        }
    }

    // 레벨 1: 최상위 작업 - 토큰을 하위로 전달
    private async Task ProcessGameDataAsync(CancellationToken token)
    {
        var playerData = await LoadPlayerDataAsync(token);
        var worldData = await FetchFromServerAsync("/world/data", token);
        Debug.Log("모든 데이터 로드 완료");
    }

    // 레벨 2: 중간 작업 - 토큰을 더 하위에 전달
    private async Task<string> LoadPlayerDataAsync(CancellationToken token)
    {
        var inventory = await FetchFromServerAsync("/player/inventory", token);
        var stats = await FetchFromServerAsync("/player/stats", token);
        return $"{inventory}, {stats}";
    }

    // 레벨 3: 최하위 작업 - 실제 비동기 호출에 토큰 사용
    private async Task<string> FetchFromServerAsync(string endpoint, CancellationToken token)
    {
        token.ThrowIfCancellationRequested();
        await Task.Delay(500, token);
        return $"Data from {endpoint}";
    }

    private void OnDestroy()
    {
        // 하나의 Cancel()로 전체 체인이 취소됨
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

### 기본값 패턴

```csharp
// ✅ 올바른 패턴: CancellationToken에 기본값 제공
public async Task<string> LoadDataAsync(string url, CancellationToken token = default)
{
    await Task.Delay(1000, token);
    return "data";
}

// 호출 시 토큰 생략 가능
// await LoadDataAsync("url");           // 취소 불가
// await LoadDataAsync("url", myToken);  // 취소 가능
```

---

## 3. 연결된 토큰 (Linked Token Source)

`CancellationTokenSource.CreateLinkedTokenSource`를 사용하면 여러 취소 소스를 결합할 수 있습니다. 연결된 소스 중 **하나라도** 취소되면 결합된 토큰도 취소됩니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class LinkedTokenSourceExample : MonoBehaviour
{
    private CancellationTokenSource _globalCts;
    private CancellationTokenSource _operationCts;

    private async void Start()
    {
        _globalCts = new CancellationTokenSource();     // 앱 수준 취소
        _operationCts = new CancellationTokenSource();  // 개별 작업 취소

        // 두 소스를 연결 - 둘 중 하나라도 취소되면 linkedToken이 취소됨
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            _globalCts.Token,
            _operationCts.Token
        );

        try
        {
            await PerformOperationAsync(linkedCts.Token);
        }
        catch (OperationCanceledException)
        {
            // 어떤 소스가 취소를 트리거했는지 확인
            if (_globalCts.IsCancellationRequested)
                Debug.Log("글로벌 취소로 인해 작업이 중단되었습니다.");
            else if (_operationCts.IsCancellationRequested)
                Debug.Log("개별 작업 취소 요청입니다.");
        }
    }

    private async Task PerformOperationAsync(CancellationToken token)
    {
        for (int i = 0; i < 50; i++)
        {
            token.ThrowIfCancellationRequested();
            await Task.Delay(200, token);
        }
    }

    public void CancelCurrentOperation() => _operationCts?.Cancel();

    private void OnDestroy()
    {
        _globalCts?.Cancel();   _globalCts?.Dispose();
        _operationCts?.Cancel(); _operationCts?.Dispose();
    }
}
```

### 3개 이상의 토큰 연결

```csharp
// 여러 취소 조건을 하나로 결합
using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
    globalToken,       // 앱 종료
    sceneToken,        // 씬 전환
    operationToken     // 개별 작업
);
await SomeOperationAsync(linkedCts.Token);
```

---

## 4. 타임아웃 패턴

CancellationTokenSource는 생성 시 또는 이후에 타임아웃을 설정할 수 있습니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class TimeoutCancellationExample : MonoBehaviour
{
    private async void Start()
    {
        // 패턴 1: 생성자에서 타임아웃 설정
        using (var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5)))
        {
            try { await SlowOperationAsync(cts.Token); }
            catch (OperationCanceledException) { Debug.Log("5초 타임아웃"); }
        }

        // 패턴 2: CancelAfter로 타임아웃 설정
        using (var cts = new CancellationTokenSource())
        {
            cts.CancelAfter(TimeSpan.FromSeconds(3));
            try { await SlowOperationAsync(cts.Token); }
            catch (OperationCanceledException) { Debug.Log("3초 타임아웃"); }
        }

        // 패턴 3: 밀리초 단위
        using (var cts = new CancellationTokenSource(2000))
        {
            try { await SlowOperationAsync(cts.Token); }
            catch (OperationCanceledException) { Debug.Log("2초 타임아웃"); }
        }
    }

    private async Task SlowOperationAsync(CancellationToken token)
    {
        await Task.Delay(10000, token);
    }
}
```

### 타임아웃 + 수동 취소 결합 및 구분

```csharp
// ✅ 타임아웃과 사용자 취소를 구분하는 패턴
public async Task<string> FetchWithTimeoutAsync(
    string url, TimeSpan timeout, CancellationToken cancellationToken)
{
    using var timeoutCts = new CancellationTokenSource(timeout);
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        cancellationToken, timeoutCts.Token);

    try
    {
        return await FetchAsync(url, linkedCts.Token);
    }
    catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested
                                              && !cancellationToken.IsCancellationRequested)
    {
        // 외부 취소가 아닌 타임아웃인 경우
        throw new TimeoutException($"요청이 {timeout.TotalSeconds}초 내에 완료되지 않았습니다.");
    }
    // 외부 취소인 경우 OperationCanceledException이 그대로 전파됨
}

private async Task<string> FetchAsync(string url, CancellationToken token)
{
    await Task.Delay(5000, token);
    return "result";
}
```

---

## 5. Unity에서의 CancellationToken

Unity는 자체 생명주기에 맞는 CancellationToken을 제공합니다.

### MonoBehaviour.destroyCancellationToken (Unity 2022.2+)

```csharp
using UnityEngine;
using System.Threading.Tasks;

public class DestroyCancellationTokenExample : MonoBehaviour
{
    // destroyCancellationToken은 GameObject 파괴 시 자동으로 취소됨
    private async void Start()
    {
        try
        {
            // CTS를 직접 만들 필요 없음!
            await RepeatActionAsync(destroyCancellationToken);
        }
        catch (System.OperationCanceledException)
        {
            Debug.Log("오브젝트 파괴로 작업 취소");
        }
    }

    private async Task RepeatActionAsync(System.Threading.CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            Debug.Log("반복 작업 실행 중...");
            await Task.Delay(1000, token);
        }
    }
}
```

### Application.exitCancellationToken (Unity 2022.2+)

```csharp
using UnityEngine;
using System.Threading;

public class ExitCancellationTokenExample : MonoBehaviour
{
    // 오브젝트 파괴 + 앱 종료 양쪽 모두 대응
    private async void Start()
    {
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            destroyCancellationToken,
            Application.exitCancellationToken
        );

        try
        {
            // DontDestroyOnLoad 오브젝트에서 특히 유용
            while (true)
            {
                linkedCts.Token.ThrowIfCancellationRequested();
                Debug.Log("백그라운드 동기화 중...");
                await System.Threading.Tasks.Task.Delay(5000, linkedCts.Token);
            }
        }
        catch (System.OperationCanceledException)
        {
            Debug.Log("작업 취소됨");
        }
    }
}
```

### Unity 2022.2 이전 버전 호환 패턴

```csharp
using UnityEngine;
using System.Threading;

public class LegacyCancellationExample : MonoBehaviour
{
    private CancellationTokenSource _destroyCts;

    private CancellationToken DestroyCancellationToken
    {
        get
        {
            _destroyCts ??= new CancellationTokenSource();
            return _destroyCts.Token;
        }
    }

    private async void Start()
    {
        try
        {
            await System.Threading.Tasks.Task.Delay(5000, DestroyCancellationToken);
        }
        catch (System.OperationCanceledException) { Debug.Log("취소됨"); }
    }

    private void OnDestroy()
    {
        _destroyCts?.Cancel();
        _destroyCts?.Dispose();
    }
}
```

---

## 6. UniTask와 CancellationToken 통합

UniTask는 CancellationToken과 긴밀하게 통합되어 Unity 환경에 최적화된 취소 처리를 제공합니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using Cysharp.Threading.Tasks;

public class UniTaskCancellationExample : MonoBehaviour
{
    private async void Start()
    {
        // UniTask의 GetCancellationTokenOnDestroy() 확장 메서드
        var token = this.GetCancellationTokenOnDestroy();

        // =============================================
        // SuppressCancellationThrow - 예외 없이 취소 여부 확인
        // =============================================
        bool isCanceled = await UniTask.Delay(5000, cancellationToken: token)
            .SuppressCancellationThrow();

        if (isCanceled)
        {
            Debug.Log("취소됨 (예외 없이)");
            return;
        }

        // =============================================
        // Timeout 확장 메서드
        // =============================================
        try
        {
            await SlowOperationAsync(token)
                .Timeout(TimeSpan.FromSeconds(3));
        }
        catch (TimeoutException)
        {
            Debug.Log("3초 타임아웃 발생!");
        }

        // =============================================
        // TimeoutWithoutException - 예외 없는 타임아웃
        // =============================================
        bool isTimeout = await SlowOperationAsync(token)
            .TimeoutWithoutException(TimeSpan.FromSeconds(3));

        if (isTimeout)
            Debug.Log("타임아웃 (예외 없이)");
    }

    private async UniTask SlowOperationAsync(CancellationToken token)
    {
        await UniTask.Delay(10000, cancellationToken: token);
    }
}
```

### UniTask WhenAll / WhenAny와 취소

```csharp
using UnityEngine;
using System;
using System.Threading;
using Cysharp.Threading.Tasks;

public class UniTaskConcurrentCancellationExample : MonoBehaviour
{
    private async void Start()
    {
        var token = this.GetCancellationTokenOnDestroy();

        // WhenAll - 토큰이 취소되면 전체 취소
        try
        {
            var (r1, r2, r3) = await UniTask.WhenAll(
                LoadAssetAsync("asset1", token),
                LoadAssetAsync("asset2", token),
                LoadAssetAsync("asset3", token)
            );
            Debug.Log($"모든 에셋 로드 완료: {r1}, {r2}, {r3}");
        }
        catch (OperationCanceledException)
        {
            Debug.Log("에셋 로드 취소됨");
        }
    }

    private async UniTask<string> LoadAssetAsync(string name, CancellationToken token)
    {
        await UniTask.Delay(1000, cancellationToken: token);
        return name;
    }
}
```

---

## 7. Awaitable과 CancellationToken

Unity 2023.1+의 Awaitable도 CancellationToken을 지원합니다.

```csharp
#if UNITY_2023_1_OR_NEWER
using UnityEngine;
using System;
using System.Threading;

public class AwaitableCancellationExample : MonoBehaviour
{
    private async void Start()
    {
        try
        {
            // Awaitable API에 destroyCancellationToken 전달
            await Awaitable.WaitForSecondsAsync(5f, destroyCancellationToken);
            await Awaitable.NextFrameAsync(destroyCancellationToken);
            await Awaitable.EndOfFrameAsync(destroyCancellationToken);
            await Awaitable.FixedUpdateAsync(destroyCancellationToken);

            // 값을 반환하는 Awaitable에서도 사용
            int score = await CalculateScoreAsync(destroyCancellationToken);
            Debug.Log($"점수: {score}");
        }
        catch (OperationCanceledException)
        {
            Debug.Log("Awaitable 작업 취소됨");
        }
    }

    private async Awaitable<int> CalculateScoreAsync(CancellationToken token)
    {
        await Awaitable.BackgroundThreadAsync();
        token.ThrowIfCancellationRequested();
        int score = 42; // 무거운 계산
        await Awaitable.MainThreadAsync();
        return score;
    }

    // 타임아웃과 Awaitable 조합
    private async Awaitable LoadWithTimeoutAsync()
    {
        using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            destroyCancellationToken, timeoutCts.Token);

        try
        {
            await Awaitable.WaitForSecondsAsync(15f, linkedCts.Token);
        }
        catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested)
        {
            Debug.LogWarning("리소스 로드 타임아웃");
        }
    }
}
#endif
```

---

## 8. Token 등록 (CancellationToken.Register)

`CancellationToken.Register`를 사용하면 토큰이 취소될 때 실행할 콜백을 등록할 수 있습니다. 취소를 지원하지 않는 레거시 API와 연동할 때 특히 유용합니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine.Networking;

public class TokenRegistrationExample : MonoBehaviour
{
    private void Start()
    {
        var cts = new CancellationTokenSource();

        // 취소 시 실행할 콜백 등록
        CancellationTokenRegistration reg1 = cts.Token.Register(() =>
            Debug.Log("콜백 1"));
        cts.Token.Register(() => Debug.Log("콜백 2"));
        cts.Token.Register(() => Debug.Log("콜백 3"));

        // Cancel() 호출 시 등록된 콜백이 역순으로 실행됨
        // 출력: 콜백 3 → 콜백 2 → 콜백 1
        cts.Cancel();

        // Registration은 IDisposable - Dispose하면 콜백 해제
        reg1.Dispose();
        cts.Dispose();
    }

    // =============================================
    // UnityWebRequest와 Register 연동
    // =============================================
    private async Task FetchWithCancellationAsync(string url, CancellationToken token)
    {
        var request = UnityWebRequest.Get(url);
        var operation = request.SendWebRequest();

        // 취소 시 UnityWebRequest.Abort() 자동 호출
        using var registration = token.Register(() =>
        {
            request.Abort();
            request.Dispose();
        });

        while (!operation.isDone)
        {
            token.ThrowIfCancellationRequested();
            await Task.Yield();
        }

        if (request.result == UnityWebRequest.Result.Success)
            Debug.Log($"응답: {request.downloadHandler.text}");

        request.Dispose();
    }

    // =============================================
    // 상태를 함께 전달 (클로저 할당 방지)
    // =============================================
    private void RegisterWithState()
    {
        var cts = new CancellationTokenSource();
        string resourceId = "player-save-123";

        cts.Token.Register(static (state) =>
        {
            Debug.Log($"리소스 정리: {(string)state}");
        }, resourceId);

        cts.Cancel();
        cts.Dispose();
    }
}
```

---

## 9. 올바른 취소 처리 패턴

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;
using Cysharp.Threading.Tasks;

public class CancellationPatternsExample : MonoBehaviour
{
    // =============================================
    // ✅ 올바른 패턴: ThrowIfCancellationRequested로 즉시 중단
    // =============================================
    private async Task GoodPattern1Async(CancellationToken token)
    {
        foreach (var item in new int[100])
        {
            token.ThrowIfCancellationRequested(); // 루프마다 확인
            await ProcessItemAsync(item, token);
        }
    }

    // =============================================
    // ✅ 올바른 패턴: IsCancellationRequested로 우아한 종료
    // =============================================
    private async Task GoodPattern2Async(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            await Task.Delay(1000, token);
            Debug.Log("작업 수행");
        }
        // 취소 후 정리 작업
        Debug.Log("정리 완료");
    }

    // =============================================
    // ✅ 올바른 패턴: 취소와 에러를 구분
    // =============================================
    private async Task GoodPattern3Async(CancellationToken token)
    {
        try
        {
            await Task.Delay(10000, token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("취소됨 - 정상");
            throw; // 재전파
        }
        catch (Exception ex)
        {
            Debug.LogError($"실제 에러: {ex.Message}");
            throw;
        }
    }

    // =============================================
    // ✅ 올바른 패턴: 취소 후 정리 + 재전파
    // =============================================
    private async Task GoodPattern4Async(CancellationToken token)
    {
        string tempFile = "temp.dat";
        try
        {
            await Task.Delay(5000, token);
        }
        catch (OperationCanceledException)
        {
            CleanupTempFile(tempFile); // 정리 수행
            throw;                     // 예외 재전파
        }
    }

    // =============================================
    // ❌ 잘못된 패턴: 토큰을 받았지만 사용하지 않음
    // =============================================
    private async Task BadPattern1Async(CancellationToken token)
    {
        await Task.Delay(10000); // token을 전달하지 않음 - 취소 불가!
    }

    // =============================================
    // ❌ 잘못된 패턴: OperationCanceledException을 삼킴
    // =============================================
    private async Task BadPattern2Async(CancellationToken token)
    {
        try
        {
            await Task.Delay(10000, token);
        }
        catch (OperationCanceledException)
        {
            // 호출자가 취소 여부를 알 수 없음!
            Debug.Log("무시하고 계속...");
        }
    }

    // =============================================
    // ❌ 잘못된 패턴: Exception으로 모두 catch
    // =============================================
    private async Task BadPattern3Async(CancellationToken token)
    {
        try
        {
            await Task.Delay(10000, token);
        }
        catch (Exception ex)
        {
            // OperationCanceledException과 실제 에러를 구분 못함
            Debug.LogError($"에러: {ex.Message}");
        }
    }

    // 헬퍼 메서드
    private Task ProcessItemAsync(int item, CancellationToken t) => Task.Delay(10, t);
    private void CleanupTempFile(string f) { }
}
```

### async void에서의 취소 처리

```csharp
using UnityEngine;
using System;
using System.Threading;
using Cysharp.Threading.Tasks;

public class AsyncVoidCancellationExample : MonoBehaviour
{
    // ❌ 위험: async void에서 OperationCanceledException 미처리 → 크래시
    private async void DangerousAsyncVoid()
    {
        var cts = new CancellationTokenSource();
        cts.Cancel();
        await System.Threading.Tasks.Task.Delay(1000, cts.Token); // 크래시!
    }

    // ✅ 올바른 패턴: async void에서 반드시 try-catch
    private async void SafeAsyncVoid()
    {
        try
        {
            await System.Threading.Tasks.Task.Delay(1000, destroyCancellationToken);
        }
        catch (OperationCanceledException) { /* async void에서 필수 */ }
        catch (Exception ex) { Debug.LogError($"오류: {ex}"); }
    }

    // ✅ 더 나은 패턴: UniTask의 Forget
    private void Start()
    {
        GameLoopAsync(this.GetCancellationTokenOnDestroy()).Forget();
    }

    private async UniTaskVoid GameLoopAsync(CancellationToken token)
    {
        // UniTaskVoid + Forget은 OperationCanceledException을 자동 무시
        while (true)
        {
            await UniTask.Delay(1000, cancellationToken: token);
            Debug.Log("틱");
        }
    }
}
```

---

## 10. IDisposable과 CancellationTokenSource

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class CtsDisposableExample : MonoBehaviour
{
    // ✅ 패턴 1: using 문 (메서드 범위)
    private async Task MethodScopedAsync()
    {
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        await DoWorkAsync(cts.Token);
        // 메서드 종료 시 자동 Dispose
    }

    // ✅ 패턴 2: 필드 CTS (수동 관리)
    private CancellationTokenSource _fieldCts;

    public void StartOperation()
    {
        // 이전 CTS 정리 후 새로 생성
        _fieldCts?.Cancel();
        _fieldCts?.Dispose();
        _fieldCts = new CancellationTokenSource();
    }

    // ✅ 패턴 3: 재시작 가능한 작업
    private CancellationTokenSource _repeatableCts;

    public async void RestartOperation()
    {
        _repeatableCts?.Cancel();
        _repeatableCts?.Dispose();
        _repeatableCts = new CancellationTokenSource();

        try
        {
            await DoWorkAsync(_repeatableCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("이전 작업 취소됨");
        }
    }

    // ❌ 잘못된 패턴: Dispose 누락 → 내부 리소스 누수!
    private async void LeakyPattern()
    {
        var cts = new CancellationTokenSource();
        await DoWorkAsync(cts.Token);
        // cts.Dispose() 누락!
    }

    // ❌ 잘못된 패턴: Dispose 후 재사용 → ObjectDisposedException!
    private void ReuseAfterDispose()
    {
        var cts = new CancellationTokenSource();
        cts.Dispose();
        // cts.Cancel();     // ObjectDisposedException!
        // var t = cts.Token; // ObjectDisposedException!
    }

    private async Task DoWorkAsync(CancellationToken token)
    {
        await Task.Delay(3000, token);
    }

    private void OnDestroy()
    {
        _fieldCts?.Cancel();     _fieldCts?.Dispose();
        _repeatableCts?.Cancel(); _repeatableCts?.Dispose();
    }
}
```

### CTS 관리 유틸리티 클래스

```csharp
using System;
using System.Threading;

/// <summary>
/// CancellationTokenSource를 안전하게 관리하는 유틸리티.
/// 재시작 가능한 비동기 작업에 유용합니다.
/// </summary>
public class CancellationTokenSourceManager : IDisposable
{
    private CancellationTokenSource _cts;
    private readonly object _lock = new object();
    private bool _disposed;

    public CancellationToken Token
    {
        get
        {
            lock (_lock)
            {
                if (_disposed) throw new ObjectDisposedException(nameof(CancellationTokenSourceManager));
                _cts ??= new CancellationTokenSource();
                return _cts.Token;
            }
        }
    }

    public void CancelAndReset()
    {
        lock (_lock)
        {
            if (_disposed) return;
            _cts?.Cancel();
            _cts?.Dispose();
            _cts = new CancellationTokenSource();
        }
    }

    public void Cancel()
    {
        lock (_lock) { _cts?.Cancel(); }
    }

    public void Dispose()
    {
        lock (_lock)
        {
            if (_disposed) return;
            _disposed = true;
            _cts?.Cancel();
            _cts?.Dispose();
            _cts = null;
        }
    }
}
```

---

## 11. 실전 종합 예제: 검색 디바운싱

```csharp
using UnityEngine;
using System;
using System.Threading;
using Cysharp.Threading.Tasks;

public class SearchWithDebounceExample : MonoBehaviour
{
    private CancellationTokenSource _searchCts;

    // UI 입력이 변경될 때마다 호출
    public void OnSearchInputChanged(string query)
    {
        // 이전 검색 즉시 취소
        _searchCts?.Cancel();
        _searchCts?.Dispose();
        _searchCts = new CancellationTokenSource();

        SearchWithDebounceAsync(query, _searchCts.Token).Forget();
    }

    private async UniTask SearchWithDebounceAsync(string query, CancellationToken token)
    {
        // 300ms 디바운스 - 타이핑이 멈출 때까지 대기
        await UniTask.Delay(300, cancellationToken: token);

        if (string.IsNullOrWhiteSpace(query)) return;

        Debug.Log($"검색 실행: {query}");

        try
        {
            var results = await PerformSearchAsync(query, token);
            foreach (var r in results) Debug.Log(r);
        }
        catch (OperationCanceledException) { /* 새 검색어 입력으로 인한 취소 */ }
        catch (Exception ex) { Debug.LogError($"검색 오류: {ex.Message}"); }
    }

    private async UniTask<string[]> PerformSearchAsync(string query, CancellationToken token)
    {
        await UniTask.Delay(500, cancellationToken: token);
        return new[] { $"Result 1: {query}", $"Result 2: {query}" };
    }

    private void OnDestroy()
    {
        _searchCts?.Cancel();
        _searchCts?.Dispose();
    }
}
```

---

## 주의사항

1. **CancellationTokenSource는 반드시 Dispose해야 합니다.** 내부에 타이머와 콜백 리스트를 보유하고 있어 Dispose하지 않으면 메모리 누수가 발생합니다.

2. **취소된 CTS는 재사용할 수 없습니다.** `Cancel()` 호출 후 `IsCancellationRequested`가 영원히 `true`이므로, 새 작업에는 새 CTS를 생성해야 합니다.

3. **`CancellationToken.None`을 취소하려고 하면 안 됩니다.** 취소 불가능한 토큰으로, 취소가 필요 없는 경우에만 사용합니다.

4. **`async void`에서 `OperationCanceledException`을 반드시 처리해야 합니다.** 처리하지 않으면 앱이 크래시할 수 있습니다.

5. **`CreateLinkedTokenSource`로 생성한 CTS도 반드시 Dispose해야 합니다.** 원본 토큰의 콜백을 등록하므로 Dispose하지 않으면 원본이 해제되지 않을 수 있습니다.

6. **Register 콜백은 동기적으로 실행됩니다.** `Cancel()` 호출 스레드에서 모든 콜백이 실행되므로, 콜백 안에서 오래 걸리는 작업을 피해야 합니다.

7. **ThrowIfCancellationRequested vs IsCancellationRequested:** 즉시 중단이 필요하면 전자, 정리 작업 후 종료가 필요하면 후자를 사용합니다.

8. **Unity API는 메인 스레드에서만 호출해야 합니다.** `Register` 콜백이 백그라운드 스레드에서 실행될 수 있으므로 주의가 필요합니다.

---

## 베스트 프랙티스

### 설계 원칙

```
✅ 비동기 메서드에는 항상 CancellationToken 매개변수를 추가하세요.
✅ CancellationToken은 메서드의 마지막 매개변수로 배치하세요.
✅ 선택적 취소를 지원하려면 default 값을 사용하세요.
✅ CancellationTokenSource는 사용 후 반드시 Dispose하세요.
✅ Unity 2022.2+에서는 destroyCancellationToken을 적극 활용하세요.
✅ 타임아웃에는 CreateLinkedTokenSource를 활용하세요.
✅ OperationCanceledException은 catch 후 적절히 재전파하세요.
✅ UniTask의 SuppressCancellationThrow로 코드를 간결하게 작성하세요.

❌ CancellationToken을 받았으면서 사용하지 않는 것은 피하세요.
❌ OperationCanceledException을 삼키지 마세요.
❌ 취소된 CancellationTokenSource를 재사용하지 마세요.
❌ async void에서 OperationCanceledException을 미처리 상태로 두지 마세요.
❌ Register 콜백에서 긴 작업을 수행하지 마세요.
❌ Dispose 후 CancellationTokenSource에 접근하지 마세요.
```

### 빠른 참조표

| 상황 | 권장 패턴 |
|------|-----------|
| MonoBehaviour 수명 | `destroyCancellationToken` (2022.2+) |
| MonoBehaviour 수명 (구버전) | 필드 CTS + OnDestroy에서 Cancel/Dispose |
| 앱 종료 | `Application.exitCancellationToken` |
| UniTask MonoBehaviour 수명 | `GetCancellationTokenOnDestroy()` |
| 메서드 범위 작업 | `using var cts = new CTS()` |
| 타임아웃 | `new CTS(TimeSpan)` + LinkedTokenSource |
| 재시작 가능한 작업 | 이전 CTS Cancel/Dispose 후 새 CTS 생성 |
| 디바운싱 | 이전 CTS Cancel/Dispose + 딜레이 |
| 복합 취소 조건 | `CreateLinkedTokenSource` |
| 예외 없는 취소 확인 | UniTask `SuppressCancellationThrow` |

### 메서드 시그니처 권장 패턴

```csharp
// 필수 취소
public async Task<T> FetchDataAsync(string url, CancellationToken cancellationToken)

// 선택적 취소
public async Task<T> ProcessAsync(int data, CancellationToken cancellationToken = default)

// Unity MonoBehaviour 진입점
private async void Start()
{
    try { await InitializeAsync(destroyCancellationToken); }
    catch (OperationCanceledException) { }
}
```

---

## 참고 자료

- [Microsoft: Cancellation in Managed Threads](https://learn.microsoft.com/dotnet/standard/threading/cancellation-in-managed-threads)
- [Microsoft: CancellationTokenSource Class](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtokensource)
- [Microsoft: CancellationToken Struct](https://learn.microsoft.com/dotnet/api/system.threading.cancellationtoken)
- [Unity: MonoBehaviour.destroyCancellationToken](https://docs.unity3d.com/ScriptReference/MonoBehaviour-destroyCancellationToken.html)
- [Unity: Application.exitCancellationToken](https://docs.unity3d.com/ScriptReference/Application-exitCancellationToken.html)
- [Unity: Awaitable](https://docs.unity3d.com/ScriptReference/Awaitable.html)
- [UniTask GitHub](https://github.com/Cysharp/UniTask)

---

## 다음 섹션

[35. 예외 처리 패턴](./35-exception-handling.md)
