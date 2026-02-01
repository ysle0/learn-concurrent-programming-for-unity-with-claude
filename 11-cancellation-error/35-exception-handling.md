# 35. 예외 처리 패턴

## 개요

비동기 및 동시성 프로그래밍에서 예외 처리는 동기 코드보다 훨씬 복잡합니다. 예외가 발생하는 스레드와 관찰하는 스레드가 다를 수 있고, 여러 작업에서 동시에 예외가 발생할 수 있으며, fire-and-forget 패턴에서는 예외가 조용히 유실될 수 있습니다. Unity 환경에서는 Coroutine의 try-catch 제한, UniTask의 독자적 예외 모델 등 추가적인 고려사항이 있습니다.

이 섹션에서는 Task, async/await, UniTask, Coroutine 각각의 예외 처리 패턴과 전역 핸들러, 커스텀 예외, 로깅 통합까지 포괄적으로 다룹니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    비동기 예외 처리 전체 구조                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  발생 계층          │  예외 래핑                │  관찰 방법           │
│  ──────────────────┼─────────────────────────┼────────────────────  │
│  Task.Run           │  AggregateException     │  .Wait() / .Result   │
│  async/await        │  자동 언래핑             │  try-catch           │
│  Task.WhenAll       │  첫 번째만 전파          │  .Exception 프로퍼티 │
│  UniTask            │  직접 전파               │  try-catch / Forget  │
│  Coroutine          │  try-catch 불가(yield)   │  수동 콜백           │
│  fire-and-forget    │  유실 가능               │  전역 핸들러 필요     │
│                                                                      │
│  전역 안전망:                                                        │
│  ├─ TaskScheduler.UnobservedTaskException                           │
│  ├─ UniTaskScheduler.UnobservedTaskException                        │
│  └─ Application.logMessageReceived                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. AggregateException — Task 기반 예외

`Task`가 내부에서 예외를 발생시키면, 해당 예외는 `AggregateException`으로 래핑됩니다. 이는 하나의 Task에서 여러 예외가 발생할 수 있는 구조(자식 Task 등)를 지원하기 위한 설계입니다.

### AggregateException 기본 구조

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;

public class AggregateExceptionExample : MonoBehaviour
{
    private void Start()
    {
        DemonstrateAggregateException();
    }

    // =============================================
    // Task.Wait()에서 AggregateException 발생
    // =============================================

    private void DemonstrateAggregateException()
    {
        Task task = Task.Run(() =>
        {
            throw new InvalidOperationException("비동기 작업에서 오류 발생");
        });

        // ❌ .Wait()는 AggregateException을 던짐
        try
        {
            task.Wait();
        }
        catch (AggregateException ae)
        {
            Debug.Log($"AggregateException 포착: {ae.InnerExceptions.Count}개의 내부 예외");

            foreach (Exception inner in ae.InnerExceptions)
            {
                Debug.LogError($"내부 예외: {inner.GetType().Name} - {inner.Message}");
            }
        }

        // ❌ .Result도 마찬가지로 AggregateException 발생
        Task<int> valueTask = Task.Run<int>(() =>
        {
            throw new ArgumentException("잘못된 인수");
            return 0; // 도달하지 않음
        });

        try
        {
            int result = valueTask.Result; // AggregateException 발생
        }
        catch (AggregateException ae)
        {
            Debug.LogError($"Result 접근 시 예외: {ae.InnerException?.Message}");
        }
    }
}
```

### AggregateException 평탄화 (Flatten)

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;

public class AggregateExceptionFlattenExample : MonoBehaviour
{
    // =============================================
    // 중첩된 AggregateException 평탄화
    // =============================================

    private void Start()
    {
        Task parent = Task.Factory.StartNew(() =>
        {
            // 자식 Task들이 각각 예외를 발생
            Task child1 = Task.Factory.StartNew(
                () => throw new InvalidOperationException("자식 1 오류"),
                TaskCreationOptions.AttachedToParent
            );

            Task child2 = Task.Factory.StartNew(
                () => throw new ArgumentException("자식 2 오류"),
                TaskCreationOptions.AttachedToParent
            );
        });

        try
        {
            parent.Wait();
        }
        catch (AggregateException ae)
        {
            // Flatten()으로 중첩 구조를 한 레벨로 펼침
            AggregateException flattened = ae.Flatten();

            Debug.Log($"평탄화 후 예외 수: {flattened.InnerExceptions.Count}");

            foreach (Exception ex in flattened.InnerExceptions)
            {
                Debug.LogError($"[Flatten] {ex.GetType().Name}: {ex.Message}");
            }
        }
    }

    // =============================================
    // Handle()을 이용한 선택적 예외 처리
    // =============================================

    private void HandleSelectiveExceptions()
    {
        Task task = Task.Run(() =>
        {
            throw new InvalidOperationException("처리 가능한 오류");
        });

        try
        {
            task.Wait();
        }
        catch (AggregateException ae)
        {
            // Handle()은 처리된 예외에 대해 true 반환
            // 처리되지 않은 예외가 있으면 새 AggregateException을 던짐
            ae.Handle(ex =>
            {
                if (ex is InvalidOperationException ioe)
                {
                    Debug.LogWarning($"처리됨: {ioe.Message}");
                    return true; // 이 예외는 처리 완료
                }
                return false; // 이 예외는 재throw
            });
        }
    }
}
```

---

## 2. async/await에서의 예외 전파

async/await 패턴에서는 `AggregateException`이 자동으로 **언래핑(unwrap)** 되어, 첫 번째 내부 예외가 직접 전파됩니다. 이로 인해 동기 코드와 거의 동일한 방식으로 try-catch를 사용할 수 있습니다.

### 자동 언래핑 메커니즘

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;

public class AsyncAwaitExceptionExample : MonoBehaviour
{
    private async void Start()
    {
        await DemonstrateUnwrapping();
    }

    // =============================================
    // async/await의 자동 예외 언래핑
    // =============================================

    private async Task DemonstrateUnwrapping()
    {
        // ✅ async/await는 AggregateException을 자동으로 언래핑
        try
        {
            await Task.Run(() =>
            {
                throw new InvalidOperationException("내부 오류 발생");
            });
        }
        catch (InvalidOperationException ex)
        {
            // AggregateException이 아닌 원본 예외를 직접 포착
            Debug.Log($"원본 예외 직접 포착: {ex.Message}");
        }

        // ✅ 여러 단계의 async 호출에서도 예외가 올바르게 전파됨
        try
        {
            await Level1Async();
        }
        catch (ApplicationException ex)
        {
            Debug.Log($"다단계 전파된 예외: {ex.Message}");
        }
    }

    private async Task Level1Async()
    {
        await Level2Async();
    }

    private async Task Level2Async()
    {
        await Task.Delay(10);
        throw new ApplicationException("Level2에서 발생한 오류");
    }

    // =============================================
    // 예외 전파와 스택 트레이스
    // =============================================

    private async Task DemonstrateStackTrace()
    {
        try
        {
            await FailingOperationAsync();
        }
        catch (Exception ex)
        {
            // async/await에서는 ExceptionDispatchInfo를 통해
            // 원래 스택 트레이스가 보존됨
            Debug.LogError($"예외 타입: {ex.GetType().Name}");
            Debug.LogError($"메시지: {ex.Message}");
            Debug.LogError($"스택 트레이스:\n{ex.StackTrace}");
        }
    }

    private async Task FailingOperationAsync()
    {
        await Task.Delay(100);
        throw new InvalidOperationException("비동기 스택 트레이스 테스트");
    }
}
```

### async void의 예외 처리 위험

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;

public class AsyncVoidDangerExample : MonoBehaviour
{
    // =============================================
    // async void는 예외를 호출자에게 전파할 수 없음
    // =============================================

    private void Start()
    {
        // ❌ async void의 예외는 SynchronizationContext로 직접 전달됨
        // Unity에서는 UnitySynchronizationContext가 처리 → 크래시 가능
        try
        {
            DangerousAsyncVoid(); // 이 try-catch는 예외를 잡지 못함!
        }
        catch (Exception ex)
        {
            // 여기에 절대 도달하지 않음
            Debug.Log($"포착 불가: {ex.Message}");
        }

        // ✅ async Task를 사용하고 await하면 예외를 정상적으로 포착
        SafeCallAsync().ContinueWith(t =>
        {
            if (t.IsFaulted)
            {
                Debug.LogError($"안전하게 포착: {t.Exception?.InnerException?.Message}");
            }
        });
    }

    // ❌ async void — 예외가 호출자에게 전파되지 않음
    private async void DangerousAsyncVoid()
    {
        await Task.Delay(100);
        throw new InvalidOperationException("async void에서 발생한 예외");
        // 이 예외는 SynchronizationContext.Post로 전달되어
        // UnityMainThread에서 처리되지 않은 예외로 나타남
    }

    // ✅ async Task — 예외를 반환된 Task에 저장
    private async Task SafeCallAsync()
    {
        await Task.Delay(100);
        throw new InvalidOperationException("async Task에서 발생한 예외");
    }
}
```

---

## 3. Task.WhenAll에서의 다중 예외 처리

`Task.WhenAll`은 여러 Task를 동시에 실행하며, 여러 Task에서 예외가 발생할 수 있습니다. await 시에는 **첫 번째 예외만** 전파되므로, 나머지 예외를 확인하려면 추가 처리가 필요합니다.

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

public class WhenAllExceptionExample : MonoBehaviour
{
    private async void Start()
    {
        await DemonstrateWhenAllExceptions();
    }

    // =============================================
    // Task.WhenAll 다중 예외 처리
    // =============================================

    private async Task DemonstrateWhenAllExceptions()
    {
        Task task1 = Task.Run(() =>
            throw new InvalidOperationException("작업 1 실패"));

        Task task2 = Task.Run(() =>
            throw new ArgumentException("작업 2 실패"));

        Task task3 = Task.Run(() =>
            throw new TimeoutException("작업 3 실패"));

        Task allTasks = Task.WhenAll(task1, task2, task3);

        // ❌ await는 첫 번째 예외만 전파 — 나머지를 놓칠 수 있음
        try
        {
            await allTasks;
        }
        catch (Exception ex)
        {
            // 첫 번째 예외만 잡힘 (보통 InvalidOperationException)
            Debug.Log($"await로 잡힌 예외: {ex.GetType().Name} - {ex.Message}");
        }

        // ✅ 모든 예외를 확인하려면 Task.Exception을 사용
        if (allTasks.Exception != null)
        {
            Debug.Log($"전체 예외 수: {allTasks.Exception.InnerExceptions.Count}");

            foreach (Exception inner in allTasks.Exception.InnerExceptions)
            {
                Debug.LogError($"[WhenAll] {inner.GetType().Name}: {inner.Message}");
            }
        }
    }

    // =============================================
    // 안전한 WhenAll 패턴 — 모든 예외 수집
    // =============================================

    private async Task SafeWhenAllAsync()
    {
        Task[] tasks = new Task[]
        {
            ProcessItemAsync("A"),
            ProcessItemAsync("B"),
            ProcessItemAsync("C"),
        };

        Task allTasks = Task.WhenAll(tasks);

        try
        {
            await allTasks;
        }
        catch
        {
            // 첫 번째 예외를 무시하고 전체를 처리
        }

        // ✅ 개별 Task의 상태를 확인하여 부분 실패 처리
        var errors = new List<string>();
        var succeeded = new List<string>();

        for (int i = 0; i < tasks.Length; i++)
        {
            if (tasks[i].IsFaulted)
            {
                string errorMsg = tasks[i].Exception?.InnerException?.Message ?? "알 수 없는 오류";
                errors.Add($"작업 {i}: {errorMsg}");
                Debug.LogWarning($"작업 {i} 실패: {errorMsg}");
            }
            else if (tasks[i].IsCompletedSuccessfully)
            {
                succeeded.Add($"작업 {i}");
            }
        }

        Debug.Log($"성공: {succeeded.Count}, 실패: {errors.Count}");
    }

    // =============================================
    // 결과값이 있는 WhenAll 예외 처리
    // =============================================

    private async Task<int[]> SafeWhenAllWithResultsAsync()
    {
        Task<int>[] tasks = new Task<int>[]
        {
            ComputeAsync(1),
            ComputeAsync(2),
            ComputeAsync(3),
        };

        try
        {
            return await Task.WhenAll(tasks);
        }
        catch
        {
            // 실패한 항목은 기본값으로 대체
            int[] results = new int[tasks.Length];
            for (int i = 0; i < tasks.Length; i++)
            {
                if (tasks[i].IsCompletedSuccessfully)
                {
                    results[i] = tasks[i].Result;
                }
                else
                {
                    results[i] = -1; // 기본값
                    Debug.LogWarning($"Task {i} 실패, 기본값 사용");
                }
            }
            return results;
        }
    }

    private async Task ProcessItemAsync(string item)
    {
        await Task.Delay(100);
        if (item == "B")
            throw new Exception($"항목 '{item}' 처리 실패");
    }

    private async Task<int> ComputeAsync(int value)
    {
        await Task.Delay(50);
        if (value == 2) throw new Exception("계산 오류");
        return value * 10;
    }
}
```

---

## 4. Fire-and-Forget에서의 예외 유실 문제

Task를 await하지 않고 실행하면 (fire-and-forget), 내부에서 발생한 예외가 관찰되지 않아 **조용히 유실**됩니다. 이는 버그를 감추고 디버깅을 어렵게 만드는 주요 원인입니다.

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;

public class FireAndForgetExceptionExample : MonoBehaviour
{
    // =============================================
    // 예외 유실 시나리오
    // =============================================

    private void Start()
    {
        // ❌ await 없이 실행 — 예외가 유실됨
        SilentlyFailingTask();
        Debug.Log("이 로그는 정상 출력됨 — 예외를 알 수 없음");

        // ❌ Task.Run 결과를 무시 — 예외 유실
        _ = Task.Run(() =>
        {
            throw new Exception("이 예외는 아무도 보지 못함");
        });

        // ✅ 안전한 fire-and-forget 패턴
        SafeFireAndForget(RiskyOperationAsync());
    }

    private async Task SilentlyFailingTask()
    {
        await Task.Delay(100);
        throw new Exception("유실되는 예외");
        // GC가 Task를 수집할 때 UnobservedTaskException 발생 가능
        // 하지만 이는 비결정적이며 .NET 4.x에서는 프로세스를 종료하지 않음
    }

    // =============================================
    // 안전한 fire-and-forget 유틸리티
    // =============================================

    // ✅ 예외를 로깅하는 fire-and-forget 래퍼
    private static async void SafeFireAndForget(
        Task task,
        Action<Exception> onError = null,
        bool continueOnCapturedContext = false)
    {
        try
        {
            await task.ConfigureAwait(continueOnCapturedContext);
        }
        catch (Exception ex)
        {
            if (onError != null)
            {
                onError(ex);
            }
            else
            {
                Debug.LogError($"[FireAndForget] 예외 포착: {ex}");
            }
        }
    }

    // ✅ 제네릭 버전
    private static async void SafeFireAndForget<TException>(
        Task task,
        Action<TException> onError = null,
        bool continueOnCapturedContext = false)
        where TException : Exception
    {
        try
        {
            await task.ConfigureAwait(continueOnCapturedContext);
        }
        catch (TException ex)
        {
            onError?.Invoke(ex);
        }
        catch (Exception ex)
        {
            Debug.LogError($"[FireAndForget] 예상치 못한 예외: {ex}");
        }
    }

    private async Task RiskyOperationAsync()
    {
        await Task.Delay(200);
        throw new InvalidOperationException("위험한 작업 실패");
    }

    // =============================================
    // 확장 메서드로 구현한 패턴
    // =============================================
}

// ✅ 프로젝트 전체에서 사용할 확장 메서드
public static class TaskExtensions
{
    public static async void SafeFireAndForget(
        this Task task,
        Action<Exception> onError = null)
    {
        try
        {
            await task;
        }
        catch (Exception ex)
        {
            onError?.Invoke(ex);
            Debug.LogError($"[SafeFireAndForget] {ex}");
        }
    }
}
```

---

## 5. UniTask 예외 처리 패턴

UniTask는 `Task`와 다른 예외 처리 모델을 사용합니다. `AggregateException`으로 래핑하지 않고, 발생한 예외를 그대로 전파합니다. 또한 `OperationCanceledException`을 특별하게 처리합니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using Cysharp.Threading.Tasks;

public class UniTaskExceptionExample : MonoBehaviour
{
    private CancellationTokenSource cts;

    private void Start()
    {
        cts = new CancellationTokenSource();
        RunExamplesAsync().Forget();
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }

    // =============================================
    // UniTask 기본 예외 처리
    // =============================================

    private async UniTaskVoid RunExamplesAsync()
    {
        // ✅ UniTask는 AggregateException 없이 원본 예외를 직접 전파
        try
        {
            await FailingUniTask();
        }
        catch (InvalidOperationException ex)
        {
            Debug.Log($"UniTask 예외 직접 포착: {ex.Message}");
        }

        // ✅ 취소는 OperationCanceledException으로 전파
        try
        {
            cts.Cancel();
            await UniTask.Delay(1000, cancellationToken: cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("취소를 정상적으로 처리");
        }
    }

    private async UniTask FailingUniTask()
    {
        await UniTask.Delay(50);
        throw new InvalidOperationException("UniTask 내부 오류");
    }

    // =============================================
    // UniTask.WhenAll 예외 처리
    // =============================================

    private async UniTask WhenAllExceptionDemo()
    {
        // UniTask.WhenAll도 첫 번째 예외만 전파
        try
        {
            await UniTask.WhenAll(
                FailWithDelay("작업A", 100),
                FailWithDelay("작업B", 200),
                FailWithDelay("작업C", 300)
            );
        }
        catch (Exception ex)
        {
            Debug.LogError($"WhenAll 첫 번째 예외: {ex.Message}");
        }
    }

    private async UniTask FailWithDelay(string name, int delayMs)
    {
        await UniTask.Delay(delayMs);
        throw new Exception($"{name} 실패");
    }

    // =============================================
    // Forget()과 예외 처리
    // =============================================

    private void ForgetExceptionDemo()
    {
        // ❌ Forget()은 예외를 UniTaskScheduler로 전달
        FailingUniTask().Forget();

        // ✅ 예외 핸들러를 명시적으로 지정할 수 있음
        // SuppressCancellationThrow와 함께 사용
    }

    // =============================================
    // SuppressCancellationThrow 패턴
    // =============================================

    private async UniTask CancellationPatternDemo(CancellationToken token)
    {
        // ✅ SuppressCancellationThrow로 취소 예외를 bool로 변환
        bool isCanceled = await UniTask.Delay(1000, cancellationToken: token)
            .SuppressCancellationThrow();

        if (isCanceled)
        {
            Debug.Log("작업이 취소되었습니다 — 예외 없이 처리");
            return;
        }

        Debug.Log("작업 완료");
    }

    // =============================================
    // UniTask에서 여러 예외를 안전하게 수집
    // =============================================

    private async UniTask SafeWhenAllUniTask()
    {
        var tasks = new UniTask[]
        {
            ProcessAsync("A"),
            ProcessAsync("B"),
            ProcessAsync("C"),
        };

        // 개별 작업을 감싸서 예외를 수집
        var results = new (bool success, Exception error)[tasks.Length];

        var wrappedTasks = new UniTask[tasks.Length];
        for (int i = 0; i < tasks.Length; i++)
        {
            int index = i; // 클로저 캡처
            wrappedTasks[i] = WrapWithErrorCapture(tasks[index], index, results);
        }

        await UniTask.WhenAll(wrappedTasks);

        // 결과 분석
        for (int i = 0; i < results.Length; i++)
        {
            if (!results[i].success)
            {
                Debug.LogWarning($"작업 {i} 실패: {results[i].error.Message}");
            }
        }
    }

    private async UniTask WrapWithErrorCapture(
        UniTask task,
        int index,
        (bool success, Exception error)[] results)
    {
        try
        {
            await task;
            results[index] = (true, null);
        }
        catch (Exception ex)
        {
            results[index] = (false, ex);
        }
    }

    private async UniTask ProcessAsync(string item)
    {
        await UniTask.Delay(100);
        if (item == "B") throw new Exception($"{item} 처리 실패");
    }
}
```

---

## 6. Try 패턴 — TryGetResult와 bool 반환

예외를 사용하지 않고 성공/실패를 반환값으로 표현하는 패턴입니다. 성능에 민감한 경로에서 예외 오버헤드를 피하거나, 실패가 **예상 가능한 정상 흐름**일 때 유용합니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;
using Cysharp.Threading.Tasks;

public class TryPatternExample : MonoBehaviour
{
    // =============================================
    // 동기 Try 패턴 (기본)
    // =============================================

    // ✅ bool 반환 + out 파라미터 패턴
    private bool TryParseConfig(string json, out GameConfig config)
    {
        config = default;

        try
        {
            config = JsonUtility.FromJson<GameConfig>(json);
            return config != null;
        }
        catch (Exception ex)
        {
            Debug.LogWarning($"설정 파싱 실패: {ex.Message}");
            return false;
        }
    }

    // =============================================
    // 비동기 Try 패턴 — Result 타입 사용
    // =============================================

    // out 파라미터는 async 메서드에서 사용 불가
    // 따라서 Result 타입을 정의하여 사용

    // ✅ 비동기 결과 래퍼
    public readonly struct AsyncResult<T>
    {
        public bool IsSuccess { get; }
        public T Value { get; }
        public Exception Error { get; }

        private AsyncResult(bool success, T value, Exception error)
        {
            IsSuccess = success;
            Value = value;
            Error = error;
        }

        public static AsyncResult<T> Success(T value)
            => new AsyncResult<T>(true, value, null);

        public static AsyncResult<T> Failure(Exception error)
            => new AsyncResult<T>(false, default, error);

        public void Deconstruct(out bool isSuccess, out T value)
        {
            isSuccess = IsSuccess;
            value = Value;
        }
    }

    // ✅ 비동기 Try 패턴 적용
    private async Task<AsyncResult<string>> TryFetchDataAsync(
        string url, CancellationToken token = default)
    {
        try
        {
            using var www = UnityEngine.Networking.UnityWebRequest.Get(url);
            await www.SendWebRequest();

            if (www.result == UnityEngine.Networking.UnityWebRequest.Result.Success)
            {
                return AsyncResult<string>.Success(www.downloadHandler.text);
            }
            else
            {
                return AsyncResult<string>.Failure(
                    new Exception($"HTTP 오류: {www.responseCode}"));
            }
        }
        catch (Exception ex)
        {
            return AsyncResult<string>.Failure(ex);
        }
    }

    // ✅ 사용 예시
    private async Task UseTryPatternAsync()
    {
        var result = await TryFetchDataAsync("https://api.example.com/data");

        if (result.IsSuccess)
        {
            Debug.Log($"데이터 수신: {result.Value}");
        }
        else
        {
            Debug.LogWarning($"데이터 수신 실패: {result.Error.Message}");
            // 폴백 데이터 사용
        }

        // 구조 분해 할당 활용
        var (success, value) = await TryFetchDataAsync("https://api.example.com/backup");
        if (success)
        {
            Debug.Log($"백업 데이터: {value}");
        }
    }

    // =============================================
    // UniTask용 Try 패턴
    // =============================================

    private async UniTask<(bool success, T result)> TryAsync<T>(
        UniTask<T> task)
    {
        try
        {
            T result = await task;
            return (true, result);
        }
        catch (OperationCanceledException)
        {
            return (false, default);
        }
        catch (Exception ex)
        {
            Debug.LogWarning($"TryAsync 실패: {ex.Message}");
            return (false, default);
        }
    }

    // ✅ 사용 예시
    private async UniTaskVoid UseTryAsyncPattern()
    {
        var (success, score) = await TryAsync(LoadScoreAsync());

        if (success)
        {
            Debug.Log($"점수: {score}");
        }
        else
        {
            Debug.Log("점수 로드 실패, 기본값 사용");
            int fallbackScore = 0;
        }
    }

    private async UniTask<int> LoadScoreAsync()
    {
        await UniTask.Delay(100);
        return 42;
    }

    [Serializable]
    private class GameConfig
    {
        public string name;
        public int version;
    }
}
```

---

## 7. 전역 예외 핸들러

모든 예외를 개별적으로 처리하는 것은 불가능합니다. 전역 예외 핸들러는 관찰되지 않은 예외를 포착하는 **최후의 안전망** 역할을 합니다.

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;
using Cysharp.Threading.Tasks;

public class GlobalExceptionHandlerExample : MonoBehaviour
{
    // =============================================
    // TaskScheduler.UnobservedTaskException
    // =============================================

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void SetupGlobalHandlers()
    {
        // .NET Task의 관찰되지 않은 예외 처리
        // GC가 Task를 수집할 때 예외가 관찰되지 않았으면 발생
        TaskScheduler.UnobservedTaskException += OnUnobservedTaskException;

        // Unity 로그 메시지 수신 (모든 Debug.LogError/LogException 포착)
        Application.logMessageReceived += OnLogMessageReceived;

        // 처리되지 않은 예외 (메인 스레드)
        AppDomain.CurrentDomain.UnhandledException += OnUnhandledException;

        Debug.Log("[전역 핸들러] 설정 완료");
    }

    private static void OnUnobservedTaskException(
        object sender, UnobservedTaskExceptionEventArgs e)
    {
        Debug.LogError($"[UnobservedTask] 관찰되지 않은 Task 예외:\n{e.Exception}");

        // SetObserved()를 호출하면 예외가 "관찰됨"으로 처리됨
        // 호출하지 않으면 .NET 4.0에서는 프로세스 종료
        // .NET 4.5+에서는 기본적으로 무시
        e.SetObserved();

        // 텔레메트리로 전송
        ReportToAnalytics("UnobservedTaskException", e.Exception);
    }

    private static void OnLogMessageReceived(
        string condition, string stackTrace, LogType type)
    {
        if (type == LogType.Exception)
        {
            // 런타임에서 잡히지 않은 예외 기록
            Debug.Log($"[전역] 예외 감지: {condition}");
        }
    }

    private static void OnUnhandledException(
        object sender, UnhandledExceptionEventArgs e)
    {
        if (e.ExceptionObject is Exception ex)
        {
            Debug.LogError($"[Unhandled] 처리되지 않은 예외: {ex}");
        }
    }

    // =============================================
    // UniTaskScheduler 전역 예외 핸들러
    // =============================================

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    private static void SetupUniTaskHandler()
    {
        // UniTask의 관찰되지 않은 예외 핸들러 설정
        UniTaskScheduler.UnobservedTaskException += OnUniTaskUnobservedException;

        // OperationCanceledException 전파 여부 설정
        // true로 설정하면 취소 예외도 전역 핸들러로 전달
        UniTaskScheduler.PropagateOperationCanceledException = false;

        // UnobservedTaskException 발생 시 동작 설정
        // true면 예외를 로그에 출력 (기본값: true)
        UniTaskScheduler.DispatchUnityMainThread = true;

        Debug.Log("[UniTask 전역 핸들러] 설정 완료");
    }

    private static void OnUniTaskUnobservedException(Exception ex)
    {
        // UniTask에서 Forget() 등으로 관찰되지 않은 예외 처리
        if (ex is OperationCanceledException)
        {
            // 취소는 보통 정상 흐름이므로 경고 레벨로 로깅
            Debug.LogWarning($"[UniTask] 관찰되지 않은 취소: {ex.Message}");
            return;
        }

        Debug.LogError($"[UniTask] 관찰되지 않은 예외: {ex}");
        ReportToAnalytics("UniTaskUnobservedException", ex);
    }

    // =============================================
    // 전역 핸들러 해제 (에디터 모드 대비)
    // =============================================

    private void OnApplicationQuit()
    {
        TaskScheduler.UnobservedTaskException -= OnUnobservedTaskException;
        Application.logMessageReceived -= OnLogMessageReceived;
        AppDomain.CurrentDomain.UnhandledException -= OnUnhandledException;
        UniTaskScheduler.UnobservedTaskException -= OnUniTaskUnobservedException;

        Debug.Log("[전역 핸들러] 해제 완료");
    }

    // =============================================
    // 분석/텔레메트리 통합 헬퍼
    // =============================================

    private static void ReportToAnalytics(string category, Exception ex)
    {
        // 실제 구현에서는 Analytics 서비스로 전송
        string report = $"[{category}] {ex.GetType().Name}: {ex.Message}\n{ex.StackTrace}";
        Debug.Log($"[Analytics] 전송: {report.Substring(0, Mathf.Min(200, report.Length))}...");
    }
}
```

---

## 8. Coroutine에서의 예외 처리

Unity Coroutine은 `yield return` 문에서 try-catch를 사용할 수 없다는 근본적인 제약이 있습니다. 이는 C# 이터레이터의 언어 사양에 의한 제한입니다.

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System;
using System.Collections;

public class CoroutineExceptionExample : MonoBehaviour
{
    // =============================================
    // Coroutine의 try-catch 제한
    // =============================================

    // ❌ 컴파일 오류 — yield return은 try-catch 블록 안에 있을 수 없음
    /*
    private IEnumerator BrokenCoroutine()
    {
        try
        {
            yield return new WaitForSeconds(1f); // CS1626 컴파일 오류
        }
        catch (Exception ex)
        {
            Debug.LogError(ex);
        }
    }
    */

    // ✅ yield return이 없는 부분에서만 try-catch 사용 가능
    private IEnumerator PartialTryCatchCoroutine()
    {
        yield return new WaitForSeconds(1f);

        // yield return이 아닌 일반 코드는 try-catch 가능
        try
        {
            int result = int.Parse("not_a_number");
        }
        catch (FormatException ex)
        {
            Debug.LogError($"파싱 오류: {ex.Message}");
        }

        yield return null;
    }

    // =============================================
    // 콜백 패턴으로 Coroutine 예외 처리
    // =============================================

    // ✅ Action 콜백으로 성공/실패 전달
    private IEnumerator FetchDataCoroutine(
        string url,
        Action<string> onSuccess,
        Action<string> onError)
    {
        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                onSuccess?.Invoke(request.downloadHandler.text);
            }
            else
            {
                onError?.Invoke(request.error);
            }
        }
    }

    // 사용 예시
    private void Start()
    {
        StartCoroutine(FetchDataCoroutine(
            "https://api.example.com/data",
            onSuccess: data => Debug.Log($"수신 성공: {data}"),
            onError: error => Debug.LogError($"수신 실패: {error}")
        ));
    }

    // =============================================
    // 래퍼 패턴으로 Coroutine 예외 포착
    // =============================================

    // ✅ 예외를 저장하는 Coroutine 래퍼
    private class CoroutineResult
    {
        public bool IsCompleted { get; set; }
        public bool IsSuccess { get; set; }
        public Exception Error { get; set; }
        public string Data { get; set; }
    }

    private IEnumerator SafeCoroutineWrapper(
        IEnumerator coroutine,
        CoroutineResult result)
    {
        while (true)
        {
            object current;
            try
            {
                if (!coroutine.MoveNext())
                {
                    result.IsCompleted = true;
                    result.IsSuccess = true;
                    yield break;
                }
                current = coroutine.Current;
            }
            catch (Exception ex)
            {
                result.IsCompleted = true;
                result.IsSuccess = false;
                result.Error = ex;
                Debug.LogError($"[CoroutineWrapper] 예외 포착: {ex}");
                yield break;
            }

            yield return current;
        }
    }

    // 사용 예시
    private IEnumerator RunWithSafeWrapper()
    {
        var result = new CoroutineResult();

        yield return StartCoroutine(
            SafeCoroutineWrapper(RiskyCoroutine(), result));

        if (result.IsSuccess)
        {
            Debug.Log("Coroutine 성공적으로 완료");
        }
        else
        {
            Debug.LogError($"Coroutine 실패: {result.Error?.Message}");
        }
    }

    private IEnumerator RiskyCoroutine()
    {
        Debug.Log("위험한 작업 시작...");
        yield return new WaitForSeconds(0.5f);

        // MoveNext() 호출 시 이 예외가 SafeCoroutineWrapper에서 포착됨
        throw new InvalidOperationException("Coroutine 내부 오류");
    }

    // =============================================
    // UniTask 변환으로 예외 처리 개선
    // =============================================

    /*
    // ✅ Coroutine을 UniTask로 변환하면 try-catch 사용 가능
    private async UniTaskVoid CoroutineToUniTaskExample()
    {
        try
        {
            await FetchDataCoroutine("https://api.example.com/data",
                null, null).ToUniTask();
        }
        catch (Exception ex)
        {
            Debug.LogError($"변환된 Coroutine 예외: {ex.Message}");
        }
    }
    */
}
```

---

## 9. 예외 필터 (when 절)

C# 6.0에서 도입된 예외 필터(`when` 절)를 사용하면, 예외 타입뿐 아니라 **조건에 따라** catch 블록의 실행 여부를 결정할 수 있습니다. 스택 트레이스를 보존하면서 조건부 처리가 가능합니다.

```csharp
using UnityEngine;
using System;
using System.Net;
using System.Threading.Tasks;
using Cysharp.Threading.Tasks;

public class ExceptionFilterExample : MonoBehaviour
{
    // =============================================
    // when 절 기본 사용법
    // =============================================

    private async UniTaskVoid Start()
    {
        await DemonstrateExceptionFilters();
    }

    private async UniTask DemonstrateExceptionFilters()
    {
        // ✅ HTTP 상태 코드별 분기 처리
        try
        {
            await FetchWithRetryAsync("https://api.example.com/data");
        }
        catch (HttpRequestException ex) when (ex.StatusCode == 404)
        {
            Debug.LogWarning("리소스를 찾을 수 없습니다 (404)");
        }
        catch (HttpRequestException ex) when (ex.StatusCode == 429)
        {
            Debug.LogWarning("요청 한도 초과 (429) — 잠시 후 재시도하세요");
        }
        catch (HttpRequestException ex) when (ex.StatusCode >= 500)
        {
            Debug.LogError($"서버 오류 ({ex.StatusCode}) — 관리자에게 문의하세요");
        }
        catch (HttpRequestException ex)
        {
            Debug.LogError($"기타 HTTP 오류: {ex.StatusCode} - {ex.Message}");
        }
    }

    // =============================================
    // 재시도 가능 여부 판단에 when 절 활용
    // =============================================

    private async UniTask<string> FetchWithRetryAsync(
        string url, int maxRetries = 3)
    {
        int attempt = 0;

        while (true)
        {
            try
            {
                attempt++;
                return await FetchDataAsync(url);
            }
            // ✅ 재시도 가능한 예외만 필터링
            catch (Exception ex) when (IsTransient(ex) && attempt < maxRetries)
            {
                int delay = (int)Math.Pow(2, attempt) * 1000; // 지수 백오프
                Debug.LogWarning(
                    $"일시적 오류 (시도 {attempt}/{maxRetries}), " +
                    $"{delay}ms 후 재시도: {ex.Message}");
                await UniTask.Delay(delay);
            }
            // 재시도 불가능한 예외 또는 최대 재시도 초과 시 전파
        }
    }

    private bool IsTransient(Exception ex)
    {
        return ex is TimeoutException
            || ex is HttpRequestException httpEx && httpEx.StatusCode >= 500
            || ex.Message.Contains("network")
            || ex.Message.Contains("timeout");
    }

    // =============================================
    // 로깅 전용 when 절 (부수 효과)
    // =============================================

    private async UniTask LoggingFilterExample()
    {
        try
        {
            await SomeOperationAsync();
        }
        // ✅ when 절에서 로깅 후 false 반환 → catch하지 않고 전파
        catch (Exception ex) when (LogException(ex))
        {
            // LogException이 항상 false를 반환하므로 여기에 도달하지 않음
        }
        catch (InvalidOperationException ex)
        {
            Debug.LogError($"InvalidOperation 처리: {ex.Message}");
        }
        catch (Exception ex)
        {
            Debug.LogError($"일반 예외 처리: {ex.Message}");
        }
    }

    // ✅ 로깅 후 항상 false 반환 — 스택 트레이스를 보존하면서 로깅
    private static bool LogException(Exception ex)
    {
        Debug.Log($"[필터 로깅] 예외 통과: {ex.GetType().Name} - {ex.Message}");
        return false; // catch하지 않고 다음 catch로 전달
    }

    // =============================================
    // 환경별 예외 처리
    // =============================================

    private async UniTask EnvironmentSpecificHandling()
    {
        try
        {
            await SomeOperationAsync();
        }
        // 에디터 모드에서만 상세 로깅
        catch (Exception ex) when (Application.isEditor)
        {
            Debug.LogError($"[에디터] 상세 오류:\n{ex}");
            // 에디터에서는 전체 스택 트레이스 출력
        }
        catch (Exception ex)
        {
            Debug.LogError($"[빌드] 오류: {ex.Message}");
            // 빌드에서는 간결한 메시지만
        }
    }

    private async UniTask SomeOperationAsync()
    {
        await UniTask.Delay(100);
        throw new InvalidOperationException("테스트 예외");
    }

    private async UniTask<string> FetchDataAsync(string url)
    {
        await UniTask.Delay(100);
        throw new TimeoutException("연결 시간 초과");
    }

    // HTTP 예외를 위한 간단한 커스텀 클래스
    public class HttpRequestException : Exception
    {
        public int StatusCode { get; }

        public HttpRequestException(string message, int statusCode)
            : base(message)
        {
            StatusCode = statusCode;
        }
    }
}
```

---

## 10. 커스텀 예외 타입 정의

게임 도메인에 특화된 예외 타입을 정의하면, 예외의 종류를 명확히 구분하고 필요한 컨텍스트 정보를 함께 전달할 수 있습니다.

```csharp
using UnityEngine;
using System;
using System.Runtime.Serialization;
using Cysharp.Threading.Tasks;

// =============================================
// 게임 도메인 예외 계층 구조
// =============================================

/// <summary>
/// 모든 게임 관련 예외의 기본 클래스
/// </summary>
[Serializable]
public class GameException : Exception
{
    /// <summary>오류 코드 (서버 통신용)</summary>
    public int ErrorCode { get; }

    /// <summary>사용자에게 표시할 메시지</summary>
    public string UserFriendlyMessage { get; }

    /// <summary>복구 가능 여부</summary>
    public bool IsRecoverable { get; }

    public GameException(
        string message,
        int errorCode = 0,
        string userFriendlyMessage = null,
        bool isRecoverable = false,
        Exception innerException = null)
        : base(message, innerException)
    {
        ErrorCode = errorCode;
        UserFriendlyMessage = userFriendlyMessage ?? "오류가 발생했습니다.";
        IsRecoverable = isRecoverable;
    }

    protected GameException(SerializationInfo info, StreamingContext context)
        : base(info, context)
    {
        ErrorCode = info.GetInt32(nameof(ErrorCode));
        UserFriendlyMessage = info.GetString(nameof(UserFriendlyMessage));
        IsRecoverable = info.GetBoolean(nameof(IsRecoverable));
    }

    public override void GetObjectData(SerializationInfo info, StreamingContext context)
    {
        base.GetObjectData(info, context);
        info.AddValue(nameof(ErrorCode), ErrorCode);
        info.AddValue(nameof(UserFriendlyMessage), UserFriendlyMessage);
        info.AddValue(nameof(IsRecoverable), IsRecoverable);
    }
}

/// <summary>
/// 네트워크 관련 예외
/// </summary>
[Serializable]
public class NetworkException : GameException
{
    public string Url { get; }
    public int HttpStatusCode { get; }

    public NetworkException(
        string message,
        string url,
        int httpStatusCode = 0,
        Exception innerException = null)
        : base(
            message,
            errorCode: 1000 + httpStatusCode,
            userFriendlyMessage: "네트워크 연결에 문제가 있습니다. 잠시 후 다시 시도해주세요.",
            isRecoverable: true,
            innerException: innerException)
    {
        Url = url;
        HttpStatusCode = httpStatusCode;
    }
}

/// <summary>
/// 게임 상태 관련 예외
/// </summary>
[Serializable]
public class GameStateException : GameException
{
    public string ExpectedState { get; }
    public string ActualState { get; }

    public GameStateException(string expectedState, string actualState)
        : base(
            $"잘못된 게임 상태: 예상={expectedState}, 실제={actualState}",
            errorCode: 2000,
            userFriendlyMessage: "잠시 후 다시 시도해주세요.",
            isRecoverable: true)
    {
        ExpectedState = expectedState;
        ActualState = actualState;
    }
}

/// <summary>
/// 리소스 로딩 예외
/// </summary>
[Serializable]
public class ResourceLoadException : GameException
{
    public string ResourcePath { get; }
    public Type ResourceType { get; }

    public ResourceLoadException(string resourcePath, Type resourceType)
        : base(
            $"리소스 로드 실패: {resourcePath} ({resourceType.Name})",
            errorCode: 3000,
            userFriendlyMessage: "게임 데이터를 불러올 수 없습니다.",
            isRecoverable: false)
    {
        ResourcePath = resourcePath;
        ResourceType = resourceType;
    }
}

// =============================================
// 커스텀 예외 사용 예시
// =============================================

public class CustomExceptionUsageExample : MonoBehaviour
{
    private async UniTaskVoid Start()
    {
        await DemonstrateCustomExceptions();
    }

    private async UniTask DemonstrateCustomExceptions()
    {
        try
        {
            await LoadGameDataAsync();
        }
        catch (NetworkException ex)
        {
            Debug.LogWarning($"네트워크 오류 (HTTP {ex.HttpStatusCode}): {ex.Url}");
            ShowUserMessage(ex.UserFriendlyMessage);

            if (ex.IsRecoverable)
            {
                Debug.Log("재시도 가능 — 재시도 버튼 표시");
            }
        }
        catch (ResourceLoadException ex)
        {
            Debug.LogError($"리소스 오류: {ex.ResourcePath} ({ex.ResourceType.Name})");
            ShowUserMessage(ex.UserFriendlyMessage);
        }
        catch (GameStateException ex)
        {
            Debug.LogError($"상태 오류: {ex.ExpectedState} → {ex.ActualState}");
            ShowUserMessage(ex.UserFriendlyMessage);
        }
        catch (GameException ex)
        {
            // 모든 게임 예외의 기본 처리
            Debug.LogError($"게임 오류 [{ex.ErrorCode}]: {ex.Message}");
            ShowUserMessage(ex.UserFriendlyMessage);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 취소되었습니다.");
        }
        catch (Exception ex)
        {
            // 예상치 못한 예외
            Debug.LogError($"알 수 없는 오류: {ex}");
            ShowUserMessage("알 수 없는 오류가 발생했습니다.");
        }
    }

    private async UniTask LoadGameDataAsync()
    {
        // 시뮬레이션: 네트워크 오류 발생
        throw new NetworkException(
            "서버 응답 타임아웃",
            url: "https://api.game.com/player/data",
            httpStatusCode: 504);
    }

    private void ShowUserMessage(string message)
    {
        // UI에 사용자 친화적 메시지 표시
        Debug.Log($"[UI] {message}");
    }
}
```

---

## 11. 로깅 및 텔레메트리 통합

프로덕션 환경에서는 예외를 단순히 콘솔에 출력하는 것으로는 부족합니다. 구조화된 로깅과 원격 텔레메트리를 통해 예외를 수집하고 분석해야 합니다.

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Threading.Tasks;
using Cysharp.Threading.Tasks;
using Debug = UnityEngine.Debug;

// =============================================
// 구조화된 예외 로거
// =============================================

public static class ExceptionLogger
{
    public enum Severity
    {
        Info,
        Warning,
        Error,
        Critical
    }

    // 예외 컨텍스트 정보를 담는 구조체
    public readonly struct ExceptionContext
    {
        public string Operation { get; }
        public string UserId { get; }
        public Dictionary<string, string> Tags { get; }
        public DateTime Timestamp { get; }

        public ExceptionContext(
            string operation,
            string userId = null,
            Dictionary<string, string> tags = null)
        {
            Operation = operation;
            UserId = userId;
            Tags = tags ?? new Dictionary<string, string>();
            Timestamp = DateTime.UtcNow;
        }
    }

    // ✅ 구조화된 예외 로깅
    public static void Log(
        Exception ex,
        Severity severity,
        ExceptionContext context)
    {
        string logEntry = FormatLogEntry(ex, severity, context);

        switch (severity)
        {
            case Severity.Info:
                Debug.Log(logEntry);
                break;
            case Severity.Warning:
                Debug.LogWarning(logEntry);
                break;
            case Severity.Error:
            case Severity.Critical:
                Debug.LogError(logEntry);
                break;
        }

        // 비동기로 원격 서버에 전송
        SendToTelemetryAsync(ex, severity, context).Forget();
    }

    private static string FormatLogEntry(
        Exception ex,
        Severity severity,
        ExceptionContext context)
    {
        return $"[{severity}] [{context.Timestamp:HH:mm:ss.fff}] " +
               $"작업={context.Operation} | " +
               $"예외={ex.GetType().Name} | " +
               $"메시지={ex.Message} | " +
               $"사용자={context.UserId ?? "N/A"}";
    }

    // 원격 텔레메트리 전송
    private static async UniTaskVoid SendToTelemetryAsync(
        Exception ex,
        Severity severity,
        ExceptionContext context)
    {
        try
        {
            // 실제 구현에서는 Firebase Crashlytics, Sentry 등에 전송
            var payload = new Dictionary<string, object>
            {
                ["exception_type"] = ex.GetType().FullName,
                ["message"] = ex.Message,
                ["stack_trace"] = ex.StackTrace,
                ["severity"] = severity.ToString(),
                ["operation"] = context.Operation,
                ["user_id"] = context.UserId,
                ["timestamp"] = context.Timestamp.ToString("o"),
                ["platform"] = Application.platform.ToString(),
                ["app_version"] = Application.version,
                ["unity_version"] = Application.unityVersion,
            };

            // 태그 추가
            foreach (var tag in context.Tags)
            {
                payload[$"tag_{tag.Key}"] = tag.Value;
            }

            // 네트워크 전송 시뮬레이션
            await UniTask.Delay(10);

            Debug.Log($"[Telemetry] 예외 보고서 전송 완료: {ex.GetType().Name}");
        }
        catch (Exception telemetryEx)
        {
            // 텔레메트리 전송 실패는 무시 (무한 루프 방지)
            Debug.LogWarning($"[Telemetry] 전송 실패: {telemetryEx.Message}");
        }
    }
}

// =============================================
// 예외 로깅 통합 사용 예시
// =============================================

public class LoggingIntegrationExample : MonoBehaviour
{
    private async UniTaskVoid Start()
    {
        await DemonstrateLogging();
    }

    private async UniTask DemonstrateLogging()
    {
        try
        {
            await LoadPlayerDataAsync("player_123");
        }
        catch (NetworkException ex)
        {
            ExceptionLogger.Log(
                ex,
                ExceptionLogger.Severity.Warning,
                new ExceptionLogger.ExceptionContext(
                    operation: "LoadPlayerData",
                    userId: "player_123",
                    tags: new Dictionary<string, string>
                    {
                        ["url"] = ex.Url,
                        ["http_code"] = ex.HttpStatusCode.ToString(),
                        ["scene"] = UnityEngine.SceneManagement.SceneManager
                            .GetActiveScene().name,
                    }
                ));
        }
        catch (Exception ex)
        {
            ExceptionLogger.Log(
                ex,
                ExceptionLogger.Severity.Error,
                new ExceptionLogger.ExceptionContext(
                    operation: "LoadPlayerData",
                    userId: "player_123"
                ));
        }
    }

    private async UniTask LoadPlayerDataAsync(string playerId)
    {
        await UniTask.Delay(100);
        throw new NetworkException(
            "서버 응답 없음",
            url: $"https://api.game.com/players/{playerId}",
            httpStatusCode: 503);
    }

    // =============================================
    // 성능 추적과 예외 통합
    // =============================================

    private async UniTask<T> TraceAsync<T>(
        string operationName,
        Func<UniTask<T>> operation)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            T result = await operation();
            stopwatch.Stop();

            Debug.Log($"[Trace] {operationName} 완료: {stopwatch.ElapsedMilliseconds}ms");
            return result;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();

            ExceptionLogger.Log(
                ex,
                ExceptionLogger.Severity.Error,
                new ExceptionLogger.ExceptionContext(
                    operation: operationName,
                    tags: new Dictionary<string, string>
                    {
                        ["duration_ms"] = stopwatch.ElapsedMilliseconds.ToString(),
                        ["failed_at"] = "execution",
                    }
                ));

            throw; // 원본 예외 재전파
        }
    }

    // 사용 예시
    private async UniTask UsageExample()
    {
        string data = await TraceAsync("FetchConfig", async () =>
        {
            await UniTask.Delay(200);
            return "config_data";
        });
    }
}

// =============================================
// 전역 예외 수집기 (MonoBehaviour)
// =============================================

public class GlobalExceptionCollector : MonoBehaviour
{
    private static GlobalExceptionCollector instance;

    private readonly Queue<ExceptionRecord> recentExceptions = new();
    private const int MaxRecords = 50;

    private struct ExceptionRecord
    {
        public DateTime Timestamp;
        public string ExceptionType;
        public string Message;
        public string StackTrace;
    }

    private void Awake()
    {
        if (instance != null)
        {
            Destroy(gameObject);
            return;
        }

        instance = this;
        DontDestroyOnLoad(gameObject);

        Application.logMessageReceived += OnLogMessage;
        TaskScheduler.UnobservedTaskException += OnUnobservedTask;
        UniTaskScheduler.UnobservedTaskException += OnUniTaskException;
    }

    private void OnDestroy()
    {
        Application.logMessageReceived -= OnLogMessage;
        TaskScheduler.UnobservedTaskException -= OnUnobservedTask;
        UniTaskScheduler.UnobservedTaskException -= OnUniTaskException;
    }

    private void OnLogMessage(string condition, string stackTrace, LogType type)
    {
        if (type == LogType.Exception)
        {
            RecordException("UnityException", condition, stackTrace);
        }
    }

    private void OnUnobservedTask(object sender, UnobservedTaskExceptionEventArgs e)
    {
        RecordException(
            e.Exception.GetType().Name,
            e.Exception.Message,
            e.Exception.StackTrace);
        e.SetObserved();
    }

    private void OnUniTaskException(Exception ex)
    {
        RecordException(ex.GetType().Name, ex.Message, ex.StackTrace);
    }

    private void RecordException(string type, string message, string stackTrace)
    {
        if (recentExceptions.Count >= MaxRecords)
        {
            recentExceptions.Dequeue();
        }

        recentExceptions.Enqueue(new ExceptionRecord
        {
            Timestamp = DateTime.UtcNow,
            ExceptionType = type,
            Message = message,
            StackTrace = stackTrace,
        });
    }

    // 디버그 UI나 크래시 리포트에서 최근 예외 목록 조회
    public static IEnumerable<string> GetRecentExceptionSummaries()
    {
        if (instance == null) yield break;

        foreach (var record in instance.recentExceptions)
        {
            yield return $"[{record.Timestamp:HH:mm:ss}] {record.ExceptionType}: {record.Message}";
        }
    }
}
```

---

## 12. 종합 예제 — 예외 처리 실전 패턴

실제 게임 개발에서 자주 사용하는 종합적인 예외 처리 패턴입니다.

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Cysharp.Threading.Tasks;

public class ComprehensiveExceptionExample : MonoBehaviour
{
    private CancellationTokenSource cts;

    private void Start()
    {
        cts = new CancellationTokenSource();
        InitializeGameAsync(cts.Token).Forget();
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }

    // =============================================
    // 초기화 흐름 — 단계별 예외 처리
    // =============================================

    private async UniTaskVoid InitializeGameAsync(CancellationToken token)
    {
        try
        {
            Debug.Log("게임 초기화 시작...");

            // 1단계: 설정 로드 (실패 시 기본값 사용)
            var config = await LoadConfigWithFallbackAsync(token);
            Debug.Log($"설정 로드 완료: {config}");

            // 2단계: 병렬 리소스 로드 (부분 실패 허용)
            var resources = await LoadResourcesParallelAsync(token);
            Debug.Log($"리소스 로드: {resources.Count}개 성공");

            // 3단계: 서버 연결 (재시도 포함)
            await ConnectToServerWithRetryAsync(3, token);
            Debug.Log("서버 연결 완료");

            Debug.Log("게임 초기화 완료!");
        }
        catch (OperationCanceledException)
        {
            Debug.Log("초기화가 취소되었습니다.");
        }
        catch (GameException ex) when (ex.IsRecoverable)
        {
            Debug.LogWarning($"복구 가능한 초기화 오류: {ex.Message}");
            // 오프라인 모드로 전환
        }
        catch (Exception ex)
        {
            Debug.LogError($"초기화 실패: {ex}");
            // 오류 화면 표시
        }
    }

    // ✅ 폴백 패턴 — 실패 시 기본값 사용
    private async UniTask<string> LoadConfigWithFallbackAsync(
        CancellationToken token)
    {
        try
        {
            await UniTask.Delay(100, cancellationToken: token);
            return "서버 설정 데이터";
        }
        catch (OperationCanceledException) { throw; } // 취소는 재전파
        catch (Exception ex)
        {
            Debug.LogWarning($"설정 로드 실패, 기본값 사용: {ex.Message}");
            return "기본 설정 데이터";
        }
    }

    // ✅ 부분 실패 허용 패턴 — 성공한 것만 사용
    private async UniTask<List<string>> LoadResourcesParallelAsync(
        CancellationToken token)
    {
        string[] paths = { "textures/hero", "sounds/bgm", "data/levels" };
        var results = new List<string>();

        var tasks = new List<UniTask>();
        foreach (string path in paths)
        {
            tasks.Add(LoadSingleResourceAsync(path, results, token));
        }

        await UniTask.WhenAll(tasks);
        return results;
    }

    private async UniTask LoadSingleResourceAsync(
        string path, List<string> results, CancellationToken token)
    {
        try
        {
            await UniTask.Delay(100, cancellationToken: token);
            results.Add(path);
        }
        catch (OperationCanceledException) { throw; }
        catch (Exception ex)
        {
            Debug.LogWarning($"리소스 '{path}' 로드 실패 (건너뜀): {ex.Message}");
        }
    }

    // ✅ 지수 백오프 재시도 패턴
    private async UniTask ConnectToServerWithRetryAsync(
        int maxRetries, CancellationToken token)
    {
        Exception lastException = null;

        for (int attempt = 1; attempt <= maxRetries; attempt++)
        {
            try
            {
                await UniTask.Delay(100, cancellationToken: token);
                Debug.Log($"서버 연결 성공 (시도 {attempt})");
                return;
            }
            catch (OperationCanceledException) { throw; }
            catch (Exception ex) when (attempt < maxRetries)
            {
                lastException = ex;
                int delayMs = (int)Math.Pow(2, attempt) * 500;
                Debug.LogWarning(
                    $"연결 실패 (시도 {attempt}/{maxRetries}), " +
                    $"{delayMs}ms 후 재시도");
                await UniTask.Delay(delayMs, cancellationToken: token);
            }
            catch (Exception ex)
            {
                lastException = ex;
            }
        }

        throw new NetworkException(
            $"서버 연결 실패 ({maxRetries}회 시도): {lastException?.Message}",
            url: "wss://game.server.com",
            httpStatusCode: 0,
            innerException: lastException);
    }
}
```

---

## 주의사항

1. **async void 최소화**: `async void`는 이벤트 핸들러 전용으로만 사용하고, 예외를 호출자에게 전파할 수 없으므로 반드시 내부에서 try-catch를 감싸야 합니다.

2. **AggregateException 언래핑**: `Task.Wait()`이나 `Task.Result` 사용 시 `AggregateException`이 발생하므로, 가능하면 `await`를 사용하여 자동 언래핑을 활용하세요.

3. **취소와 오류의 구분**: `OperationCanceledException`은 일반적으로 정상 흐름입니다. 오류와 구분하여 처리하고, 전역 핸들러에서 불필요한 로깅을 피하세요.

4. **Coroutine의 try-catch 제한**: `yield return` 문은 try-catch 블록 안에 위치할 수 없습니다. 래퍼 패턴이나 UniTask 변환을 사용하여 이 제한을 우회하세요.

5. **예외 유실 방지**: fire-and-forget 패턴에서는 반드시 안전한 래퍼를 사용하거나 전역 핸들러를 설정하세요.

6. **스택 트레이스 보존**: `throw ex;` 대신 `throw;`를 사용하여 원래 스택 트레이스를 보존하세요.

7. **when 절의 부수 효과**: `when` 절에서 호출하는 메서드는 예외를 발생시키지 않아야 합니다. 예외가 발생하면 해당 catch 블록이 건너뛰어집니다.

8. **전역 핸들러 등록 해제**: 에디터 도메인 리로드 시 이벤트 핸들러가 중복 등록될 수 있으므로, `OnDestroy`나 `OnApplicationQuit`에서 반드시 해제하세요.

---

## 베스트 프랙티스

### ✅ 권장 패턴

```csharp
// ✅ 1. async Task 반환 + await 사용
private async Task GoodAsync()
{
    try
    {
        await SomeOperationAsync();
    }
    catch (OperationCanceledException)
    {
        // 취소는 별도 처리 (정상 흐름)
    }
    catch (Exception ex)
    {
        Debug.LogError($"오류: {ex.Message}");
        throw; // 필요 시 재전파 (스택 트레이스 보존)
    }
}

// ✅ 2. 취소와 오류의 명확한 구분
private async UniTask HandleCancellationProperly(CancellationToken token)
{
    try
    {
        await LongOperationAsync(token);
    }
    catch (OperationCanceledException)
    {
        Debug.Log("정상 취소 처리");
        // 리소스 정리만 수행, 로깅하지 않음
    }
    catch (Exception ex)
    {
        Debug.LogError($"실제 오류: {ex}");
        // 텔레메트리에 보고
    }
}

// ✅ 3. 특정 예외를 먼저, 범용 예외를 나중에 catch
private async UniTask SpecificCatchFirst()
{
    try
    {
        await SomeOperationAsync();
    }
    catch (NetworkException ex)     { /* 네트워크 오류 처리 */ }
    catch (GameStateException ex)   { /* 상태 오류 처리 */ }
    catch (GameException ex)        { /* 기타 게임 오류 */ }
    catch (OperationCanceledException) { /* 취소 */ }
    catch (Exception ex)            { /* 최후의 안전망 */ }
}

// ✅ 4. 안전한 fire-and-forget
private void StartSafeOperation()
{
    SomeOperationAsync()
        .SafeFireAndForget(ex => Debug.LogError($"백그라운드 오류: {ex}"));
}

// ✅ 5. using 문을 통한 리소스 정리 보장
private async UniTask ResourceSafeAsync()
{
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

    try
    {
        await OperationAsync(cts.Token);
    }
    catch (OperationCanceledException)
    {
        Debug.Log("타임아웃으로 취소됨");
    }
    // CancellationTokenSource는 using에 의해 자동 Dispose
}
```

### ❌ 안티패턴

```csharp
// ❌ 1. 예외를 삼키기 (swallowing)
private async Task BadSwallowAsync()
{
    try
    {
        await SomeOperationAsync();
    }
    catch (Exception)
    {
        // 아무것도 하지 않음 — 버그를 감추는 최악의 패턴
    }
}

// ❌ 2. 모든 예외를 동일하게 처리
private async Task BadCatchAllAsync()
{
    try
    {
        await SomeOperationAsync();
    }
    catch (Exception ex)
    {
        Debug.LogError(ex); // 취소, 네트워크 오류, 로직 오류 모두 동일하게 처리
    }
}

// ❌ 3. throw ex — 스택 트레이스 손실
private async Task BadRethrowAsync()
{
    try
    {
        await SomeOperationAsync();
    }
    catch (Exception ex)
    {
        Debug.LogError(ex);
        throw ex; // 원래 스택 트레이스가 사라짐! throw;를 사용할 것
    }
}

// ❌ 4. async void에서 예외 무시
private async void BadAsyncVoid()
{
    // 예외 발생 시 호출자가 포착 불가능
    await SomeOperationAsync();
}

// ❌ 5. fire-and-forget 예외 유실
private void BadFireAndForget()
{
    _ = SomeOperationAsync(); // 예외가 조용히 유실됨
}

// ❌ 6. catch 블록에서 새 예외 발생 (원본 정보 유실)
private async Task BadNewExceptionAsync()
{
    try
    {
        await SomeOperationAsync();
    }
    catch (Exception ex)
    {
        throw new Exception("오류 발생"); // 원본 예외 정보가 사라짐
        // ✅ 대신: throw new GameException("오류 발생", innerException: ex);
    }
}
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                  예외 처리 의사 결정 트리                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  예외 발생 → 복구 가능한가?                                          │
│  │                                                                   │
│  ├─ Yes → 폴백/재시도 로직 실행                                      │
│  │   ├─ 재시도 가능? → 지수 백오프로 재시도                           │
│  │   ├─ 대체 데이터? → 기본값/캐시 사용                               │
│  │   └─ 부분 실패? → 성공한 부분만 활용                               │
│  │                                                                   │
│  └─ No → 상위로 전파                                                 │
│      ├─ 취소 예외? → 정상 흐름으로 처리 (로깅 불필요)                 │
│      ├─ 도메인 예외? → 사용자 메시지 표시                             │
│      └─ 시스템 예외? → 로깅 + 텔레메트리 + 오류 화면                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [Microsoft Docs: Exception handling (Task-based asynchronous pattern)](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/exception-handling-task-parallel-library)
- [Microsoft Docs: AggregateException](https://learn.microsoft.com/en-us/dotnet/api/system.aggregateexception)
- [Microsoft Docs: Exception filters (when clause)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/when)
- [UniTask GitHub: Error handling](https://github.com/Cysharp/UniTask#error-handling)
- [Unity Docs: Coroutines](https://docs.unity3d.com/Manual/Coroutines.html)
- [Stephen Cleary: Async Best Practices](https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming)
- [.NET Task Exception Handling](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/exception-handling-task-parallel-library)
