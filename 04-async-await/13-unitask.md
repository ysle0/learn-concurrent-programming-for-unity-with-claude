# 13. UniTask

## 개요

UniTask는 Cysharp에서 개발한 Unity 전용 고성능 async/await 라이브러리입니다. Unity의 PlayerLoop와 완벽하게 통합되며, Zero-allocation을 목표로 설계되어 GC 부담 없이 비동기 프로그래밍을 할 수 있습니다. Unity 프로젝트에서 가장 권장되는 비동기 솔루션입니다.

---

## 1. UniTask 소개

### 왜 UniTask인가?

```
┌─────────────────────────────────────────────────────────────────┐
│                  Task vs UniTask 비교                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  특성              │  Task           │  UniTask                  │
│  ─────────────────┼─────────────────┼───────────────────────────│
│  GC Allocation    │  있음           │  Zero (구조체 기반)        │
│  PlayerLoop 통합  │  없음           │  완벽한 통합               │
│  Coroutine 변환   │  수동           │  자동 지원                 │
│  취소 처리        │  복잡           │  간편 (자동 연동)           │
│  트래커/디버깅    │  제한적         │  UniTaskTracker 제공       │
│  WebGL 지원       │  문제 있음      │  완벽 지원                 │
│  DOTween 연동     │  수동           │  내장 지원                 │
│  Addressables     │  AsyncOpHandle  │  ToUniTask 확장           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 설치 방법

```
Package Manager (권장):
1. Window > Package Manager
2. + 버튼 > Add package from git URL
3. https://github.com/Cysharp/UniTask.git?path=src/UniTask/Assets/Plugins/UniTask

또는 manifest.json에 추가:
{
  "dependencies": {
    "com.cysharp.unitask": "https://github.com/Cysharp/UniTask.git?path=src/UniTask/Assets/Plugins/UniTask"
  }
}
```

---

## 2. UniTask 기본 사용법

### 기본 구문

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using System.Threading;

public class UniTaskBasicsExample : MonoBehaviour
{
    private CancellationTokenSource cts;

    private void Start()
    {
        cts = new CancellationTokenSource();

        // UniTask 시작
        RunExamplesAsync().Forget(); // Forget()으로 fire-and-forget
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }

    // =============================================
    // UniTask 기본 반환 타입
    // =============================================

    // UniTask - 결과 없음 (Task 대응)
    private async UniTask SimpleUniTaskAsync()
    {
        Debug.Log("UniTask 시작");
        await UniTask.Delay(1000); // 1초 대기
        Debug.Log("UniTask 완료");
    }

    // UniTask<T> - 결과 반환 (Task<T> 대응)
    private async UniTask<int> UniTaskWithResultAsync()
    {
        await UniTask.Delay(500);
        return 42;
    }

    // UniTaskVoid - fire-and-forget (async void 대체)
    private async UniTaskVoid FireAndForgetAsync()
    {
        await UniTask.Delay(100);
        Debug.Log("Fire and forget 완료");
        // 예외 발생 시 UniTaskScheduler.UnobservedTaskException으로 전달
    }

    private async UniTask RunExamplesAsync()
    {
        // 기본 UniTask
        await SimpleUniTaskAsync();

        // 결과 받기
        int result = await UniTaskWithResultAsync();
        Debug.Log($"결과: {result}");

        // Fire and forget
        FireAndForgetAsync().Forget();
    }
}
```

### UniTask.Delay와 대기 메서드

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using System;
using System.Threading;

public class UniTaskDelayExample : MonoBehaviour
{
    private CancellationTokenSource cts;

    private async void Start()
    {
        cts = new CancellationTokenSource();
        await DemonstrateDelayMethods(cts.Token);
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }

    private async UniTask DemonstrateDelayMethods(CancellationToken token)
    {
        // =============================================
        // 시간 기반 대기
        // =============================================

        // 밀리초 대기
        await UniTask.Delay(1000, cancellationToken: token);

        // TimeSpan 대기
        await UniTask.Delay(TimeSpan.FromSeconds(0.5f), cancellationToken: token);

        // DelayType 지정 (기본: DeltaTime)
        await UniTask.Delay(100, DelayType.DeltaTime, cancellationToken: token);      // Time.deltaTime 기반
        await UniTask.Delay(100, DelayType.UnscaledDeltaTime, cancellationToken: token); // Time.unscaledDeltaTime 기반
        await UniTask.Delay(100, DelayType.Realtime, cancellationToken: token);       // 실제 시간 기반

        // =============================================
        // 프레임 기반 대기
        // =============================================

        // 다음 프레임까지 대기 (yield return null 대응)
        await UniTask.Yield();

        // 지정된 프레임 수만큼 대기
        await UniTask.DelayFrame(10, cancellationToken: token);

        // 다음 FixedUpdate까지 대기
        await UniTask.WaitForFixedUpdate(token);

        // 다음 LateUpdate까지 대기
        await UniTask.Yield(PlayerLoopTiming.LastPostLateUpdate);

        // 프레임 끝까지 대기 (WaitForEndOfFrame 대응)
        await UniTask.WaitForEndOfFrame(this, token);

        // =============================================
        // 조건 기반 대기
        // =============================================

        bool condition = false;

        // 5초 후 condition을 true로 설정
        UniTask.Delay(5000, cancellationToken: token).ContinueWith(() => condition = true).Forget();

        // 조건이 true가 될 때까지 대기
        await UniTask.WaitUntil(() => condition, cancellationToken: token);
        Debug.Log("조건 충족!");

        // 조건이 false가 될 때까지 대기
        await UniTask.WaitWhile(() => condition, cancellationToken: token);
    }
}
```

---

## 3. PlayerLoop 통합

### PlayerLoopTiming

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using System.Threading;

public class PlayerLoopTimingExample : MonoBehaviour
{
    private CancellationTokenSource cts;

    private async void Start()
    {
        cts = new CancellationTokenSource();
        await DemonstratePlayerLoopTiming(cts.Token);
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }

    private async UniTask DemonstratePlayerLoopTiming(CancellationToken token)
    {
        /*
        PlayerLoopTiming 종류:

        Initialization 단계:
        - Initialization
        - LastInitialization

        EarlyUpdate 단계:
        - EarlyUpdate
        - LastEarlyUpdate

        FixedUpdate 단계:
        - FixedUpdate
        - LastFixedUpdate

        PreUpdate 단계:
        - PreUpdate
        - LastPreUpdate

        Update 단계:
        - Update (기본값)
        - LastUpdate

        PreLateUpdate 단계:
        - PreLateUpdate
        - LastPreLateUpdate

        PostLateUpdate 단계:
        - PostLateUpdate
        - LastPostLateUpdate

        TimeUpdate 단계:
        - TimeUpdate
        - LastTimeUpdate
        */

        // FixedUpdate에서 실행
        await UniTask.Yield(PlayerLoopTiming.FixedUpdate);
        Debug.Log($"FixedUpdate 시점 - Frame: {Time.frameCount}");

        // Update 후에 실행
        await UniTask.Yield(PlayerLoopTiming.LastUpdate);
        Debug.Log($"LastUpdate 시점 - Frame: {Time.frameCount}");

        // LateUpdate 후에 실행
        await UniTask.Yield(PlayerLoopTiming.PostLateUpdate);
        Debug.Log($"PostLateUpdate 시점 - Frame: {Time.frameCount}");
    }

    // =============================================
    // 실전 예제: 카메라 추적
    // =============================================

    [SerializeField] private Transform target;
    [SerializeField] private float smoothTime = 0.3f;

    private async UniTaskVoid FollowTargetAsync(CancellationToken token)
    {
        Vector3 velocity = Vector3.zero;

        while (!token.IsCancellationRequested)
        {
            // LateUpdate 타이밍에 실행 (카메라 추적에 적합)
            await UniTask.Yield(PlayerLoopTiming.PostLateUpdate, token);

            if (target != null)
            {
                transform.position = Vector3.SmoothDamp(
                    transform.position,
                    target.position,
                    ref velocity,
                    smoothTime
                );
            }
        }
    }
}
```

---

## 4. Unity 기능 통합

### UnityWebRequest 통합

```csharp
using UnityEngine;
using UnityEngine.Networking;
using Cysharp.Threading.Tasks;
using System;
using System.Threading;

public class UniTaskWebRequestExample : MonoBehaviour
{
    private CancellationTokenSource cts;

    private async void Start()
    {
        cts = new CancellationTokenSource();

        try
        {
            await TestWebRequests(cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("요청 취소됨");
        }
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }

    private async UniTask TestWebRequests(CancellationToken token)
    {
        // =============================================
        // 기본 GET 요청
        // =============================================

        string response = await UnityWebRequest.Get("https://httpbin.org/get")
            .SendWebRequest()
            .WithCancellation(token);

        Debug.Log($"GET 응답: {response.Substring(0, 100)}...");

        // =============================================
        // POST 요청
        // =============================================

        var postData = new WWWForm();
        postData.AddField("name", "UniTask");

        string postResponse = await UnityWebRequest.Post("https://httpbin.org/post", postData)
            .SendWebRequest()
            .WithCancellation(token);

        Debug.Log($"POST 응답: {postResponse.Substring(0, 100)}...");

        // =============================================
        // 타임아웃 설정
        // =============================================

        try
        {
            // 1초 타임아웃
            using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(1));
            using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(token, timeoutCts.Token);

            await UnityWebRequest.Get("https://httpbin.org/delay/5")
                .SendWebRequest()
                .WithCancellation(linkedCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("타임아웃!");
        }

        // =============================================
        // 텍스처 다운로드
        // =============================================

        Texture2D texture = await UnityWebRequestTexture.GetTexture("https://httpbin.org/image/png")
            .SendWebRequest()
            .WithCancellation(token) as Texture2D;

        if (texture != null)
        {
            Debug.Log($"텍스처 다운로드: {texture.width}x{texture.height}");
        }

        // =============================================
        // 에셋번들 다운로드
        // =============================================

        // var bundle = await UnityWebRequestAssetBundle.GetAssetBundle("url")
        //     .SendWebRequest()
        //     .WithCancellation(token) as AssetBundle;
    }
}
```

### Addressables 통합

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using System.Threading;
// using UnityEngine.AddressableAssets;
// using UnityEngine.ResourceManagement.AsyncOperations;

public class UniTaskAddressablesExample : MonoBehaviour
{
    /*
    Addressables 사용 시:

    // 에셋 로드
    var prefab = await Addressables.LoadAssetAsync<GameObject>("MyPrefab")
        .WithCancellation(cancellationToken);

    // 씬 로드
    await Addressables.LoadSceneAsync("MyScene")
        .WithCancellation(cancellationToken);

    // 인스턴스 생성
    var instance = await Addressables.InstantiateAsync("MyPrefab")
        .WithCancellation(cancellationToken);
    */
}
```

### DOTween 통합

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using System.Threading;
// using DG.Tweening;

public class UniTaskDOTweenExample : MonoBehaviour
{
    /*
    DOTween 사용 시 (DOTween Pro 또는 UniTask.DOTween 패키지 필요):

    // 이동 애니메이션 대기
    await transform.DOMove(targetPosition, 1f)
        .WithCancellation(cancellationToken);

    // 페이드 애니메이션 대기
    await spriteRenderer.DOFade(0f, 0.5f)
        .WithCancellation(cancellationToken);

    // 시퀀스 대기
    var sequence = DOTween.Sequence()
        .Append(transform.DOMove(pos1, 1f))
        .Append(transform.DORotate(rot1, 0.5f))
        .Append(transform.DOScale(2f, 0.3f));

    await sequence.WithCancellation(cancellationToken);
    */
}
```

---

## 5. 취소 처리

### 자동 취소 연동

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using System.Threading;

public class UniTaskCancellationExample : MonoBehaviour
{
    // =============================================
    // GetCancellationTokenOnDestroy - 자동 취소
    // =============================================

    private async void Start()
    {
        // 오브젝트 파괴 시 자동으로 취소되는 토큰
        var token = this.GetCancellationTokenOnDestroy();

        try
        {
            await LongRunningOperationAsync(token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("오브젝트 파괴로 인한 취소");
        }
    }

    private async UniTask LongRunningOperationAsync(CancellationToken token)
    {
        while (true)
        {
            // 매 반복마다 취소 확인 불필요 - UniTask가 자동 처리
            await UniTask.Delay(1000, cancellationToken: token);
            Debug.Log("반복 실행 중...");
        }
    }

    // =============================================
    // 취소 처리 패턴
    // =============================================

    private async UniTask CancellationPatternsAsync()
    {
        var cts = new CancellationTokenSource();

        // 방법 1: OperationCanceledException catch
        try
        {
            await UniTask.Delay(10000, cancellationToken: cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("취소됨 (예외)");
        }

        // 방법 2: SuppressCancellationThrow 사용
        var (isCanceled, _) = await UniTask.Delay(10000, cancellationToken: cts.Token)
            .SuppressCancellationThrow();

        if (isCanceled)
        {
            Debug.Log("취소됨 (예외 없음)");
        }

        cts.Dispose();
    }

    // =============================================
    // 타임아웃 패턴
    // =============================================

    private async UniTask TimeoutExample()
    {
        var token = this.GetCancellationTokenOnDestroy();

        try
        {
            // 3초 타임아웃
            await SlowOperationAsync()
                .Timeout(TimeSpan.FromSeconds(3));
        }
        catch (TimeoutException)
        {
            Debug.Log("타임아웃!");
        }

        // 또는 TimeoutWithoutException 사용
        bool isTimeout = await SlowOperationAsync()
            .TimeoutWithoutException(TimeSpan.FromSeconds(3));

        if (isTimeout)
        {
            Debug.Log("타임아웃 (예외 없음)");
        }
    }

    private async UniTask SlowOperationAsync()
    {
        await UniTask.Delay(10000);
    }
}
```

---

## 6. 병렬 처리

### WhenAll / WhenAny

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using System;
using System.Threading;
using System.Linq;

public class UniTaskParallelExample : MonoBehaviour
{
    private async void Start()
    {
        var token = this.GetCancellationTokenOnDestroy();
        await DemonstrateParallelProcessing(token);
    }

    private async UniTask DemonstrateParallelProcessing(CancellationToken token)
    {
        // =============================================
        // WhenAll - 모두 완료 대기
        // =============================================

        Debug.Log("=== WhenAll ===");

        var startTime = Time.realtimeSinceStartup;

        // 모든 작업을 병렬로 실행하고 결과 수집
        var results = await UniTask.WhenAll(
            LoadDataAsync("A", 300, token),
            LoadDataAsync("B", 200, token),
            LoadDataAsync("C", 100, token)
        );

        var elapsed = Time.realtimeSinceStartup - startTime;
        Debug.Log($"WhenAll 결과: {string.Join(", ", results)}");
        Debug.Log($"소요 시간: {elapsed:F2}초 (예상: ~0.3초)");

        // =============================================
        // WhenAny - 하나라도 완료되면 진행
        // =============================================

        Debug.Log("\n=== WhenAny ===");

        startTime = Time.realtimeSinceStartup;

        // 가장 먼저 완료되는 것의 인덱스와 결과
        var (winnerIndex, winnerResult) = await UniTask.WhenAny(
            LoadDataAsync("Fast", 100, token),
            LoadDataAsync("Slow", 500, token)
        );

        elapsed = Time.realtimeSinceStartup - startTime;
        Debug.Log($"WhenAny 승자: 인덱스 {winnerIndex}, 결과: {winnerResult}");
        Debug.Log($"소요 시간: {elapsed:F2}초 (예상: ~0.1초)");

        // =============================================
        // 대량 병렬 처리
        // =============================================

        Debug.Log("\n=== 대량 병렬 처리 ===");

        string[] urls = Enumerable.Range(1, 10)
            .Select(i => $"Data_{i}")
            .ToArray();

        var allResults = await UniTask.WhenAll(
            urls.Select(url => LoadDataAsync(url, 100, token))
        );

        Debug.Log($"총 {allResults.Length}개 로드 완료");
    }

    private async UniTask<string> LoadDataAsync(string name, int delayMs, CancellationToken token)
    {
        Debug.Log($"[{name}] 시작");
        await UniTask.Delay(delayMs, cancellationToken: token);
        Debug.Log($"[{name}] 완료");
        return $"{name}_Result";
    }
}
```

### 제한된 동시성

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using System;
using System.Threading;
using System.Collections.Generic;
using System.Linq;

public class UniTaskThrottlingExample : MonoBehaviour
{
    private async void Start()
    {
        var token = this.GetCancellationTokenOnDestroy();

        // 최대 3개의 동시 작업으로 제한
        await ProcessWithLimitedConcurrency(
            Enumerable.Range(1, 10).ToArray(),
            maxConcurrency: 3,
            token
        );
    }

    private async UniTask ProcessWithLimitedConcurrency(
        int[] items,
        int maxConcurrency,
        CancellationToken token)
    {
        var semaphore = new SemaphoreSlim(maxConcurrency);
        var tasks = new List<UniTask>();

        foreach (var item in items)
        {
            await semaphore.WaitAsync(token);

            tasks.Add(ProcessItemAsync(item, semaphore, token));
        }

        await UniTask.WhenAll(tasks);
        semaphore.Dispose();
    }

    private async UniTask ProcessItemAsync(int item, SemaphoreSlim semaphore, CancellationToken token)
    {
        try
        {
            Debug.Log($"[{item}] 처리 시작");
            await UniTask.Delay(500, cancellationToken: token);
            Debug.Log($"[{item}] 처리 완료");
        }
        finally
        {
            semaphore.Release();
        }
    }
}
```

---

## 7. 채널 (Channel)

### Producer-Consumer 패턴

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Linq;
using System;
using System.Threading;

public class UniTaskChannelExample : MonoBehaviour
{
    private async void Start()
    {
        var token = this.GetCancellationTokenOnDestroy();

        // 채널 생성 (용량 제한)
        var channel = Channel.CreateSingleConsumerUnbounded<int>();

        // Producer와 Consumer 병렬 실행
        await UniTask.WhenAll(
            ProducerAsync(channel.Writer, token),
            ConsumerAsync(channel.Reader, token)
        );
    }

    private async UniTask ProducerAsync(ChannelWriter<int> writer, CancellationToken token)
    {
        for (int i = 0; i < 10; i++)
        {
            await UniTask.Delay(100, cancellationToken: token);
            await writer.WriteAsync(i, token);
            Debug.Log($"[Producer] 생산: {i}");
        }

        writer.Complete();
        Debug.Log("[Producer] 완료");
    }

    private async UniTask ConsumerAsync(ChannelReader<int> reader, CancellationToken token)
    {
        await foreach (var item in reader.ReadAllAsync(token))
        {
            Debug.Log($"[Consumer] 소비: {item}");
            await UniTask.Delay(200, cancellationToken: token); // 처리 시뮬레이션
        }

        Debug.Log("[Consumer] 완료");
    }
}
```

---

## 8. AsyncReactiveProperty

### 반응형 속성

```csharp
using UnityEngine;
using UnityEngine.UI;
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Linq;
using System.Threading;

public class AsyncReactivePropertyExample : MonoBehaviour
{
    [SerializeField] private Text healthText;
    [SerializeField] private Slider healthSlider;

    private AsyncReactiveProperty<int> health;

    private void Start()
    {
        var token = this.GetCancellationTokenOnDestroy();

        // AsyncReactiveProperty 생성
        health = new AsyncReactiveProperty<int>(100);

        // 값 변경 구독
        SubscribeToHealth(token).Forget();

        // 테스트: 1초마다 체력 감소
        DecreaseHealthOverTime(token).Forget();
    }

    private void OnDestroy()
    {
        health?.Dispose();
    }

    private async UniTaskVoid SubscribeToHealth(CancellationToken token)
    {
        // 값이 변경될 때마다 UI 업데이트
        await health.ForEachAsync(value =>
        {
            if (healthText != null)
                healthText.text = $"HP: {value}";

            if (healthSlider != null)
                healthSlider.value = value / 100f;

            Debug.Log($"체력 변경: {value}");

        }, token);
    }

    private async UniTaskVoid DecreaseHealthOverTime(CancellationToken token)
    {
        while (!token.IsCancellationRequested && health.Value > 0)
        {
            await UniTask.Delay(1000, cancellationToken: token);
            health.Value -= 10;
        }
    }

    public void TakeDamage(int damage)
    {
        health.Value = Mathf.Max(0, health.Value - damage);
    }

    public void Heal(int amount)
    {
        health.Value = Mathf.Min(100, health.Value + amount);
    }
}
```

---

## 9. UniTaskTracker

### 디버깅 도구

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;

public class UniTaskTrackerExample : MonoBehaviour
{
    /*
    UniTaskTracker 사용법:

    1. Window > UniTask Tracker 메뉴로 창 열기
    2. Enable Tracking 체크
    3. Enable StackTrace 체크 (성능 영향 있음)

    기능:
    - 현재 실행 중인 UniTask 목록 표시
    - 누수된 Task 감지
    - 생성 위치 스택 트레이스
    - GC Alloc 모니터링
    */

    private void Start()
    {
        // 에디터에서 트래킹 활성화
#if UNITY_EDITOR
        Cysharp.Threading.Tasks.TaskTracker.EnableTracking = true;
        Cysharp.Threading.Tasks.TaskTracker.EnableStackTrace = true;
#endif

        // 테스트용 Task 생성
        TestLeakedTask().Forget();
    }

    private async UniTaskVoid TestLeakedTask()
    {
        // 이 Task는 취소되지 않으면 트래커에 계속 표시됨
        while (true)
        {
            await UniTask.Delay(1000);
            Debug.Log("실행 중...");
        }
    }
}
```

---

## 10. Coroutine에서 마이그레이션

### 변환 가이드

```csharp
using UnityEngine;
using UnityEngine.Networking;
using Cysharp.Threading.Tasks;
using System.Collections;
using System.Threading;

public class CoroutineToUniTaskMigration : MonoBehaviour
{
    // =============================================
    // Before: Coroutine
    // =============================================

    private IEnumerator LoadDataCoroutine()
    {
        Debug.Log("로딩 시작");

        yield return new WaitForSeconds(1f);

        using (var request = UnityWebRequest.Get("https://api.example.com"))
        {
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"결과: {request.downloadHandler.text}");
            }
        }

        Debug.Log("로딩 완료");
    }

    // =============================================
    // After: UniTask
    // =============================================

    private async UniTask LoadDataUniTaskAsync(CancellationToken token)
    {
        Debug.Log("로딩 시작");

        await UniTask.Delay(1000, cancellationToken: token);

        var result = await UnityWebRequest.Get("https://api.example.com")
            .SendWebRequest()
            .WithCancellation(token);

        Debug.Log($"결과: {result}");

        Debug.Log("로딩 완료");
    }

    // =============================================
    // 변환 규칙
    // =============================================

    /*
    Coroutine                        →  UniTask
    ─────────────────────────────────────────────────────────
    IEnumerator                      →  async UniTask
    yield return null                →  await UniTask.Yield()
    yield return new WaitForSeconds  →  await UniTask.Delay()
    yield return WaitForEndOfFrame   →  await UniTask.WaitForEndOfFrame()
    yield return WaitForFixedUpdate  →  await UniTask.WaitForFixedUpdate()
    yield return WaitUntil           →  await UniTask.WaitUntil()
    yield return WaitWhile           →  await UniTask.WaitWhile()
    yield return request.SendWebRequest →  await request.SendWebRequest()
                                           .WithCancellation(token)
    yield return StartCoroutine      →  await (UniTask)
    StopCoroutine                    →  CancellationToken.Cancel()
    */

    // =============================================
    // 기존 Coroutine을 UniTask로 래핑
    // =============================================

    private UniTask WrapCoroutineAsUniTask()
    {
        return LoadDataCoroutine().ToUniTask();
    }

    // =============================================
    // UniTask에서 Coroutine 호출
    // =============================================

    private async UniTask CallCoroutineFromUniTask()
    {
        // Coroutine을 UniTask로 변환하여 await
        await LoadDataCoroutine().ToUniTask();
    }
}
```

---

## 11. 베스트 프랙티스

```csharp
using UnityEngine;
using Cysharp.Threading.Tasks;
using System;
using System.Threading;

public class UniTaskBestPracticesExample : MonoBehaviour
{
    /*
    ┌─────────────────────────────────────────────────────────────────┐
    │                  UniTask 베스트 프랙티스                         │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  1. 항상 CancellationToken 사용                                  │
    │     └─ GetCancellationTokenOnDestroy() 활용                     │
    │     └─ 오브젝트 생명주기와 연동                                  │
    │                                                                  │
    │  2. Fire-and-forget은 Forget() 명시                             │
    │     └─ 경고 억제 및 의도 명확화                                  │
    │     └─ UniTaskVoid 사용도 고려                                   │
    │                                                                  │
    │  3. 예외 처리                                                    │
    │     └─ try-catch 또는 SuppressCancellationThrow                │
    │     └─ UniTaskScheduler.UnobservedTaskException 핸들링         │
    │                                                                  │
    │  4. PlayerLoopTiming 적절히 선택                                │
    │     └─ 기본값은 Update                                          │
    │     └─ 카메라는 PostLateUpdate                                  │
    │     └─ 물리는 FixedUpdate                                       │
    │                                                                  │
    │  5. 타임아웃 설정                                                │
    │     └─ 네트워크 요청에 필수                                      │
    │     └─ Timeout() 또는 CancellationTokenSource(TimeSpan)        │
    │                                                                  │
    │  6. 디버깅 시 UniTaskTracker 활용                               │
    │     └─ 누수 Task 감지                                           │
    │     └─ 개발 중에만 활성화 (성능 영향)                            │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
    */

    private async UniTask GoodPattern()
    {
        var token = this.GetCancellationTokenOnDestroy();

        try
        {
            await UniTask.Delay(1000, cancellationToken: token);
            // 작업 수행
        }
        catch (OperationCanceledException)
        {
            // 정상적인 취소
            Debug.Log("취소됨");
        }
        catch (Exception ex)
        {
            Debug.LogError($"에러: {ex.Message}");
        }
    }
}
```

---

## 주의사항

1. **취소 토큰 필수**: GetCancellationTokenOnDestroy() 사용 권장
2. **Forget() 명시**: fire-and-forget 시 명시적으로 호출
3. **WebGL 주의**: Task.Run 대신 UniTask 사용
4. **디버깅**: UniTaskTracker로 누수 확인
5. **성능**: 이미 최적화되어 있으나 과도한 생성은 피하기

---

## 참고 자료

- [UniTask GitHub](https://github.com/Cysharp/UniTask)
- [UniTask README (한국어)](https://github.com/Cysharp/UniTask/blob/master/README_KR.md)
- [Cysharp Blog](https://tech.cygames.co.jp/archives/3417/)

---

## 다음 섹션

[14. IAsyncEnumerable](../05-async-streams/14-async-enumerable.md)
