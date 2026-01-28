# 09. TAP (Task Asynchronous Programming)

## 개요

TAP(Task-based Asynchronous Pattern)은 .NET의 현대적인 비동기 프로그래밍 패턴입니다. `Task`와 `Task<T>` 클래스를 중심으로 비동기 작업을 표현하며, async/await 키워드와 함께 사용됩니다.

---

## 1. Task 클래스 기초

### Task vs Task<T>

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class TaskBasicsExample : MonoBehaviour
{
    private async void Start()
    {
        await DemonstrateTaskTypes();
    }

    private async Task DemonstrateTaskTypes()
    {
        // =============================================
        // Task - 결과 없이 완료만 추적
        // =============================================

        Debug.Log("=== Task (결과 없음) ===");

        Task task = DoWorkAsync();
        Debug.Log($"Task 상태: {task.Status}");

        await task;
        Debug.Log($"Task 완료 상태: {task.Status}");

        // =============================================
        // Task<T> - 결과값 반환
        // =============================================

        Debug.Log("\n=== Task<T> (결과 있음) ===");

        Task<int> taskWithResult = GetValueAsync();
        Debug.Log($"Task<int> 상태: {taskWithResult.Status}");

        int result = await taskWithResult;
        Debug.Log($"결과: {result}, 상태: {taskWithResult.Status}");

        // =============================================
        // Task 속성들
        // =============================================

        Debug.Log("\n=== Task 속성 ===");
        Task demoTask = Task.Delay(100);

        Debug.Log($"Id: {demoTask.Id}");
        Debug.Log($"Status: {demoTask.Status}");
        Debug.Log($"IsCompleted: {demoTask.IsCompleted}");
        Debug.Log($"IsCanceled: {demoTask.IsCanceled}");
        Debug.Log($"IsFaulted: {demoTask.IsFaulted}");
        Debug.Log($"IsCompletedSuccessfully: {demoTask.IsCompletedSuccessfully}");

        await demoTask;

        Debug.Log($"\n완료 후:");
        Debug.Log($"Status: {demoTask.Status}");
        Debug.Log($"IsCompletedSuccessfully: {demoTask.IsCompletedSuccessfully}");
    }

    private async Task DoWorkAsync()
    {
        await Task.Delay(100);
        Debug.Log("작업 완료");
    }

    private async Task<int> GetValueAsync()
    {
        await Task.Delay(100);
        return 42;
    }
}

/*
TaskStatus 열거형:
- Created: 생성됨, 아직 시작 안 됨
- WaitingForActivation: 스케줄링 대기
- WaitingToRun: 실행 대기
- Running: 실행 중
- WaitingForChildrenToComplete: 자식 Task 완료 대기
- RanToCompletion: 정상 완료
- Canceled: 취소됨
- Faulted: 예외 발생
*/
```

---

## 2. Task 생성 방법

### Task.Run

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class TaskCreationExample : MonoBehaviour
{
    private async void Start()
    {
        await DemonstrateTaskCreation();
    }

    private async Task DemonstrateTaskCreation()
    {
        // =============================================
        // 1. Task.Run - 가장 일반적인 방법
        // =============================================

        Debug.Log("=== Task.Run ===");

        // Action (결과 없음)
        await Task.Run(() =>
        {
            Debug.Log($"Task.Run 실행 - Thread: {Thread.CurrentThread.ManagedThreadId}");
        });

        // Func<T> (결과 있음)
        int result = await Task.Run(() =>
        {
            return 42;
        });
        Debug.Log($"결과: {result}");

        // async lambda
        await Task.Run(async () =>
        {
            await Task.Delay(100);
            Debug.Log("async lambda 완료");
        });

        // =============================================
        // 2. Task.Factory.StartNew - 더 많은 옵션
        // =============================================

        Debug.Log("\n=== Task.Factory.StartNew ===");

        // 기본 사용
        await Task.Factory.StartNew(() =>
        {
            Debug.Log("Factory.StartNew 실행");
        });

        // 옵션 지정
        await Task.Factory.StartNew(() =>
        {
            Debug.Log("LongRunning Task - 별도 스레드에서 실행");
        }, TaskCreationOptions.LongRunning);

        // ⚠️ async lambda 사용 시 주의!
        // Task.Factory.StartNew(async () => ...) 는 Task<Task> 반환
        Task<Task> nestedTask = Task.Factory.StartNew(async () =>
        {
            await Task.Delay(100);
        });
        await nestedTask.Unwrap(); // Unwrap 필요

        // =============================================
        // 3. new Task() - 직접 생성 (권장하지 않음)
        // =============================================

        Debug.Log("\n=== new Task() ===");

        Task manualTask = new Task(() =>
        {
            Debug.Log("수동 생성 Task");
        });
        manualTask.Start(); // 명시적 시작 필요
        await manualTask;

        // =============================================
        // 4. 이미 완료된 Task
        // =============================================

        Debug.Log("\n=== 완료된 Task ===");

        // Task.CompletedTask - 이미 완료된 Task
        Task completed = Task.CompletedTask;
        Debug.Log($"CompletedTask 상태: {completed.Status}");

        // Task.FromResult - 이미 완료된 Task<T>
        Task<int> fromResult = Task.FromResult(100);
        Debug.Log($"FromResult 값: {fromResult.Result}");

        // Task.FromCanceled - 취소된 Task
        var cts = new CancellationTokenSource();
        cts.Cancel();
        Task fromCanceled = Task.FromCanceled(cts.Token);
        Debug.Log($"FromCanceled 상태: {fromCanceled.Status}");

        // Task.FromException - 실패한 Task
        Task fromException = Task.FromException(new InvalidOperationException("테스트"));
        Debug.Log($"FromException 상태: {fromException.Status}");
    }
}

/*
┌─────────────────────────────────────────────────────────────────┐
│              Task 생성 방법 선택 가이드                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Task.Run                                                        │
│  ├─ 가장 일반적인 선택                                           │
│  ├─ CPU-bound 작업을 ThreadPool에서 실행                        │
│  └─ async lambda 자동 처리                                      │
│                                                                  │
│  Task.Factory.StartNew                                          │
│  ├─ 더 많은 옵션 필요할 때                                       │
│  ├─ TaskCreationOptions 지정                                    │
│  ├─ TaskScheduler 지정                                          │
│  └─ ⚠️ async lambda는 Task.Run 사용 권장                        │
│                                                                  │
│  Task.FromResult / CompletedTask                                │
│  ├─ 동기적으로 완료된 결과 반환                                  │
│  ├─ 캐싱, 조건부 비동기에 유용                                   │
│  └─ 할당 없음 (성능 최적화)                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
*/
```

---

## 3. Task 조합

### WhenAll / WhenAny

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;
using System.Linq;

public class TaskCombinationExample : MonoBehaviour
{
    private async void Start()
    {
        await DemonstrateTaskCombination();
    }

    private async Task DemonstrateTaskCombination()
    {
        // =============================================
        // Task.WhenAll - 모든 Task 완료 대기
        // =============================================

        Debug.Log("=== Task.WhenAll ===");

        var task1 = DelayAndReturn("Task 1", 300, 10);
        var task2 = DelayAndReturn("Task 2", 100, 20);
        var task3 = DelayAndReturn("Task 3", 200, 30);

        // 모든 Task가 완료될 때까지 대기
        int[] results = await Task.WhenAll(task1, task2, task3);

        Debug.Log($"모든 결과: {string.Join(", ", results)}");
        Debug.Log($"합계: {results.Sum()}");

        // 결과 없는 Task들의 WhenAll
        await Task.WhenAll(
            Task.Delay(100),
            Task.Delay(200),
            Task.Delay(300)
        );
        Debug.Log("모든 Delay 완료");

        // =============================================
        // Task.WhenAny - 하나라도 완료되면 진행
        // =============================================

        Debug.Log("\n=== Task.WhenAny ===");

        var fastTask = DelayAndReturn("Fast", 100, 1);
        var slowTask = DelayAndReturn("Slow", 500, 2);

        Task<int> firstCompleted = await Task.WhenAny(fastTask, slowTask);
        Debug.Log($"먼저 완료된 Task 결과: {firstCompleted.Result}");

        // 나머지도 완료 대기 (이미 실행 중)
        await Task.WhenAll(fastTask, slowTask);
        Debug.Log("모든 Task 완료");

        // =============================================
        // 타임아웃 패턴 with WhenAny
        // =============================================

        Debug.Log("\n=== 타임아웃 패턴 ===");

        var workTask = DelayAndReturn("Work", 2000, 100);
        var timeoutTask = Task.Delay(500);

        if (await Task.WhenAny(workTask, timeoutTask) == timeoutTask)
        {
            Debug.Log("타임아웃! 작업이 500ms 내에 완료되지 않음");
            // 주의: workTask는 여전히 실행 중!
        }
        else
        {
            Debug.Log($"시간 내 완료: {workTask.Result}");
        }
    }

    private async Task<int> DelayAndReturn(string name, int delayMs, int value)
    {
        Debug.Log($"{name} 시작");
        await Task.Delay(delayMs);
        Debug.Log($"{name} 완료 ({delayMs}ms)");
        return value;
    }
}
```

### 순차 실행 vs 병렬 실행

```csharp
using UnityEngine;
using System;
using System.Diagnostics;
using System.Threading.Tasks;

public class SequentialVsParallelExample : MonoBehaviour
{
    private async void Start()
    {
        await CompareExecutionPatterns();
    }

    private async Task CompareExecutionPatterns()
    {
        var sw = new Stopwatch();

        // =============================================
        // 순차 실행 (Sequential)
        // =============================================

        UnityEngine.Debug.Log("=== 순차 실행 ===");
        sw.Start();

        // 각 await가 완료될 때까지 다음으로 넘어가지 않음
        int a = await GetValueAsync("A", 300);
        int b = await GetValueAsync("B", 200);
        int c = await GetValueAsync("C", 100);

        sw.Stop();
        UnityEngine.Debug.Log($"순차 결과: {a + b + c}, 소요 시간: {sw.ElapsedMilliseconds}ms");
        // 예상 시간: 300 + 200 + 100 = 600ms

        // =============================================
        // 병렬 실행 (Parallel)
        // =============================================

        UnityEngine.Debug.Log("\n=== 병렬 실행 ===");
        sw.Restart();

        // Task를 먼저 시작하고 나중에 await
        Task<int> taskA = GetValueAsync("A", 300);
        Task<int> taskB = GetValueAsync("B", 200);
        Task<int> taskC = GetValueAsync("C", 100);

        // 모두 완료될 때까지 대기
        int[] results = await Task.WhenAll(taskA, taskB, taskC);

        sw.Stop();
        UnityEngine.Debug.Log($"병렬 결과: {results[0] + results[1] + results[2]}, 소요 시간: {sw.ElapsedMilliseconds}ms");
        // 예상 시간: max(300, 200, 100) = 300ms

        // =============================================
        // 혼합: 일부 순차, 일부 병렬
        // =============================================

        UnityEngine.Debug.Log("\n=== 혼합 실행 ===");
        sw.Restart();

        // Step 1: 병렬
        var step1Tasks = new[]
        {
            GetValueAsync("1A", 100),
            GetValueAsync("1B", 100)
        };
        int[] step1Results = await Task.WhenAll(step1Tasks);

        // Step 2: Step 1 결과에 의존 (순차)
        int step2Result = await GetValueAsync("2", step1Results[0] + step1Results[1]);

        sw.Stop();
        UnityEngine.Debug.Log($"혼합 결과: {step2Result}, 소요 시간: {sw.ElapsedMilliseconds}ms");
    }

    private async Task<int> GetValueAsync(string name, int delayMs)
    {
        UnityEngine.Debug.Log($"[{name}] 시작");
        await Task.Delay(delayMs);
        UnityEngine.Debug.Log($"[{name}] 완료");
        return delayMs;
    }
}
```

---

## 4. Task 연속 작업 (Continuation)

### ContinueWith

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class TaskContinuationExample : MonoBehaviour
{
    private async void Start()
    {
        await DemonstrateContinuation();
    }

    private async Task DemonstrateContinuation()
    {
        // =============================================
        // ContinueWith - 저수준 연속 작업
        // =============================================

        Debug.Log("=== ContinueWith ===");

        Task<int> originalTask = Task.Run(() =>
        {
            Debug.Log("원본 Task 실행");
            return 42;
        });

        // ContinueWith로 연속 작업 등록
        Task<string> continuationTask = originalTask.ContinueWith(antecedent =>
        {
            Debug.Log($"Continuation 실행, 이전 결과: {antecedent.Result}");
            return $"결과는 {antecedent.Result}";
        });

        string result = await continuationTask;
        Debug.Log($"최종 결과: {result}");

        // =============================================
        // TaskContinuationOptions
        // =============================================

        Debug.Log("\n=== TaskContinuationOptions ===");

        // 성공 시에만 실행
        Task successTask = Task.Run(() => 100)
            .ContinueWith(t => Debug.Log($"성공: {t.Result}"),
                TaskContinuationOptions.OnlyOnRanToCompletion);

        // 실패 시에만 실행
        Task failTask = Task.Run(() => { throw new Exception("에러!"); })
            .ContinueWith(t => Debug.Log($"실패: {t.Exception?.InnerException?.Message}"),
                TaskContinuationOptions.OnlyOnFaulted);

        // 취소 시에만 실행
        var cts = new CancellationTokenSource();
        cts.Cancel();
        Task cancelTask = Task.Run(() => { }, cts.Token)
            .ContinueWith(t => Debug.Log("취소됨"),
                TaskContinuationOptions.OnlyOnCanceled);

        await Task.WhenAll(successTask, failTask, cancelTask);

        // =============================================
        // ⚠️ ContinueWith vs await (권장: await 사용)
        // =============================================

        Debug.Log("\n=== await 권장 패턴 ===");

        // ❌ ContinueWith (복잡하고 컨텍스트 문제 있음)
        await Task.Run(() => 42)
            .ContinueWith(t =>
            {
                // 기본적으로 ThreadPool에서 실행
                // Unity API 사용 불가!
                return t.Result * 2;
            });

        // ✅ await (간단하고 컨텍스트 자동 처리)
        int value = await Task.Run(() => 42);
        int doubled = value * 2;
        // 메인 스레드에서 실행, Unity API 사용 가능
    }
}

/*
TaskContinuationOptions 주요 옵션:
- None: 기본값
- OnlyOnRanToCompletion: 성공 시에만
- OnlyOnFaulted: 실패 시에만
- OnlyOnCanceled: 취소 시에만
- NotOnRanToCompletion: 성공이 아닐 때만
- ExecuteSynchronously: 가능하면 동기 실행
- LazyCancellation: 지연된 취소 확인
*/
```

---

## 5. 취소 (Cancellation)

### CancellationToken 사용

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class TaskCancellationExample : MonoBehaviour
{
    private CancellationTokenSource cts;

    private void OnEnable()
    {
        cts = new CancellationTokenSource();
        StartLongRunningTask(cts.Token);
    }

    private void OnDisable()
    {
        cts?.Cancel();
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }

    // 버튼에서 호출 (테스트용)
    public void CancelTask()
    {
        cts?.Cancel();
    }

    // =============================================
    // 취소 가능한 Task
    // =============================================

    private async void StartLongRunningTask(CancellationToken token)
    {
        try
        {
            await LongRunningOperationAsync(token);
            Debug.Log("작업 완료!");
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
            // 방법 1: 명시적 취소 확인 및 예외
            token.ThrowIfCancellationRequested();

            // 방법 2: 조건 확인
            if (token.IsCancellationRequested)
            {
                // 정리 작업 가능
                Debug.Log("취소 감지, 정리 중...");
                throw new OperationCanceledException(token);
            }

            Debug.Log($"진행 중: {i + 1}%");

            // 취소 토큰을 Delay에도 전달
            await Task.Delay(100, token);
        }
    }

    // =============================================
    // 여러 취소 패턴
    // =============================================

    private async Task CancellationPatternsAsync()
    {
        // 타임아웃 기반 취소
        using (var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(5)))
        {
            try
            {
                await LongRunningOperationAsync(timeoutCts.Token);
            }
            catch (OperationCanceledException)
            {
                Debug.Log("5초 타임아웃!");
            }
        }

        // 연결된 토큰 (여러 소스 중 하나라도 취소되면)
        var cts1 = new CancellationTokenSource();
        var cts2 = new CancellationTokenSource();

        using (var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(cts1.Token, cts2.Token))
        {
            // cts1 또는 cts2 중 하나라도 취소되면 linkedCts.Token도 취소됨
            var linkedToken = linkedCts.Token;

            // 3초 후 타임아웃
            cts1.CancelAfter(TimeSpan.FromSeconds(3));
        }
    }

    // =============================================
    // Task.Run에 CancellationToken 전달
    // =============================================

    private async Task TaskRunWithCancellation(CancellationToken token)
    {
        // Task.Run의 두 번째 인자로 취소 토큰 전달
        int result = await Task.Run(() =>
        {
            int sum = 0;
            for (int i = 0; i < 1000000; i++)
            {
                // 주기적으로 취소 확인
                if (i % 10000 == 0)
                {
                    token.ThrowIfCancellationRequested();
                }
                sum += i;
            }
            return sum;
        }, token); // 취소 토큰 전달

        Debug.Log($"결과: {result}");
    }
}
```

### 취소 콜백 등록

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class CancellationCallbackExample : MonoBehaviour
{
    private async void Start()
    {
        var cts = new CancellationTokenSource();

        // 취소 시 호출될 콜백 등록
        cts.Token.Register(() =>
        {
            Debug.Log("취소 콜백 1 호출됨!");
        });

        cts.Token.Register(() =>
        {
            Debug.Log("취소 콜백 2 호출됨!");
        });

        // 1초 후 취소
        cts.CancelAfter(1000);

        try
        {
            await Task.Delay(5000, cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("Task가 취소됨");
        }

        cts.Dispose();
    }
}
```

---

## 6. 예외 처리

### Task 예외 처리 패턴

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;

public class TaskExceptionHandlingExample : MonoBehaviour
{
    private async void Start()
    {
        await DemonstrateExceptionHandling();
    }

    private async Task DemonstrateExceptionHandling()
    {
        // =============================================
        // 기본 예외 처리
        // =============================================

        Debug.Log("=== 기본 예외 처리 ===");

        try
        {
            await FailingTaskAsync();
        }
        catch (InvalidOperationException ex)
        {
            Debug.Log($"예외 잡힘: {ex.Message}");
        }

        // =============================================
        // WhenAll에서의 예외
        // =============================================

        Debug.Log("\n=== WhenAll 예외 ===");

        Task task1 = ThrowAsync("예외 1");
        Task task2 = ThrowAsync("예외 2");
        Task allTasks = Task.WhenAll(task1, task2);

        try
        {
            await allTasks;
        }
        catch (Exception ex)
        {
            // 첫 번째 예외만 직접 잡힘
            Debug.Log($"직접 잡힌 예외: {ex.Message}");
        }

        // 모든 예외 확인
        if (allTasks.Exception != null)
        {
            foreach (var innerEx in allTasks.Exception.InnerExceptions)
            {
                Debug.Log($"내부 예외: {innerEx.Message}");
            }
        }

        // =============================================
        // 예외 관찰 (Observe)
        // =============================================

        Debug.Log("\n=== 예외 관찰 ===");

        Task fireAndForget = Task.Run(() => throw new Exception("관찰되지 않은 예외"));

        // 예외 관찰 (UnobservedTaskException 방지)
        _ = fireAndForget.ContinueWith(t =>
        {
            if (t.Exception != null)
            {
                Debug.Log($"관찰된 예외: {t.Exception.InnerException?.Message}");
            }
        }, TaskContinuationOptions.OnlyOnFaulted);

        await Task.Delay(100);
    }

    private async Task FailingTaskAsync()
    {
        await Task.Delay(100);
        throw new InvalidOperationException("의도적인 예외");
    }

    private async Task ThrowAsync(string message)
    {
        await Task.Delay(100);
        throw new Exception(message);
    }
}
```

---

## 7. TaskCompletionSource

### 수동으로 Task 완료 제어

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class TaskCompletionSourceExample : MonoBehaviour
{
    // =============================================
    // 콜백 기반 API를 Task로 래핑
    // =============================================

    private void Start()
    {
        WrapCallbackBasedAPI();
        WrapCoroutineInTask();
        TimeoutPatternExample();
    }

    // 콜백 기반 API
    private void LegacyApiWithCallback(Action<string> onSuccess, Action<Exception> onError)
    {
        // 시뮬레이션: 1초 후 결과 반환
        Task.Delay(1000).ContinueWith(_ =>
        {
            onSuccess("레거시 API 결과");
        });
    }

    // Task로 래핑
    private Task<string> LegacyApiAsTask()
    {
        var tcs = new TaskCompletionSource<string>();

        LegacyApiWithCallback(
            result => tcs.SetResult(result),
            error => tcs.SetException(error)
        );

        return tcs.Task;
    }

    private async void WrapCallbackBasedAPI()
    {
        string result = await LegacyApiAsTask();
        Debug.Log($"래핑된 API 결과: {result}");
    }

    // =============================================
    // Coroutine을 Task로 래핑
    // =============================================

    private Task WaitForSecondsTask(float seconds)
    {
        var tcs = new TaskCompletionSource<bool>();

        StartCoroutine(WaitAndComplete(seconds, tcs));

        return tcs.Task;
    }

    private System.Collections.IEnumerator WaitAndComplete(float seconds, TaskCompletionSource<bool> tcs)
    {
        yield return new WaitForSeconds(seconds);
        tcs.SetResult(true);
    }

    private async void WrapCoroutineInTask()
    {
        Debug.Log("Coroutine Task 시작");
        await WaitForSecondsTask(1f);
        Debug.Log("Coroutine Task 완료");
    }

    // =============================================
    // 타임아웃 패턴
    // =============================================

    private async Task<T> WithTimeout<T>(Task<T> task, TimeSpan timeout)
    {
        var tcs = new TaskCompletionSource<bool>();

        using (var cts = new CancellationTokenSource())
        {
            var timeoutTask = Task.Delay(timeout, cts.Token);

            var completedTask = await Task.WhenAny(task, timeoutTask);

            if (completedTask == timeoutTask)
            {
                throw new TimeoutException();
            }

            cts.Cancel(); // 타임아웃 타이머 취소
            return await task;
        }
    }

    private async void TimeoutPatternExample()
    {
        try
        {
            int result = await WithTimeout(
                GetSlowResult(),
                TimeSpan.FromSeconds(1)
            );
            Debug.Log($"타임아웃 내 완료: {result}");
        }
        catch (TimeoutException)
        {
            Debug.Log("타임아웃 발생!");
        }
    }

    private async Task<int> GetSlowResult()
    {
        await Task.Delay(2000);
        return 42;
    }
}
```

---

## 8. Unity 실전 패턴

```csharp
using UnityEngine;
using UnityEngine.UI;
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Collections.Generic;

public class UnityTAPPatternsExample : MonoBehaviour
{
    [SerializeField] private Text statusText;
    [SerializeField] private Slider progressBar;

    private CancellationTokenSource cts;

    private void OnEnable()
    {
        cts = new CancellationTokenSource();
    }

    private void OnDisable()
    {
        cts?.Cancel();
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }

    // =============================================
    // 패턴 1: 병렬 데이터 로딩
    // =============================================

    public async Task<GameData> LoadGameDataAsync(CancellationToken token)
    {
        UpdateStatus("데이터 로딩 중...");

        // 병렬로 여러 데이터 로드
        var playerTask = LoadPlayerDataAsync(token);
        var worldTask = LoadWorldDataAsync(token);
        var settingsTask = LoadSettingsAsync(token);

        await Task.WhenAll(playerTask, worldTask, settingsTask);

        return new GameData
        {
            Player = playerTask.Result,
            World = worldTask.Result,
            Settings = settingsTask.Result
        };
    }

    // =============================================
    // 패턴 2: 진행률 보고 로딩
    // =============================================

    public async Task LoadWithProgressAsync(IProgress<float> progress, CancellationToken token)
    {
        List<Func<Task>> loadingSteps = new List<Func<Task>>
        {
            () => LoadPlayerDataAsync(token),
            () => LoadWorldDataAsync(token),
            () => LoadSettingsAsync(token),
            () => LoadAssetsAsync(token)
        };

        for (int i = 0; i < loadingSteps.Count; i++)
        {
            token.ThrowIfCancellationRequested();

            await loadingSteps[i]();

            progress?.Report((float)(i + 1) / loadingSteps.Count);
        }
    }

    // =============================================
    // 패턴 3: 재시도 로직
    // =============================================

    public async Task<T> WithRetryAsync<T>(
        Func<Task<T>> operation,
        int maxRetries = 3,
        int delayMs = 1000,
        CancellationToken token = default)
    {
        Exception lastException = null;

        for (int attempt = 1; attempt <= maxRetries; attempt++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (attempt < maxRetries)
            {
                lastException = ex;
                Debug.LogWarning($"시도 {attempt} 실패: {ex.Message}. {delayMs}ms 후 재시도...");
                await Task.Delay(delayMs, token);
            }
        }

        throw lastException!;
    }

    // =============================================
    // 패턴 4: 순차적 UI 애니메이션
    // =============================================

    public async Task PlaySequentialAnimationsAsync(CancellationToken token)
    {
        await FadeInAsync(1f, token);
        await Task.Delay(500, token);
        await ScaleUpAsync(0.5f, token);
        await Task.Delay(200, token);
        await MoveToAsync(new Vector3(100, 0, 0), 1f, token);
    }

    // Helper 메서드들
    private void UpdateStatus(string status)
    {
        if (statusText != null)
            statusText.text = status;
    }

    private async Task<PlayerData> LoadPlayerDataAsync(CancellationToken token)
    {
        await Task.Delay(500, token);
        return new PlayerData();
    }

    private async Task<WorldData> LoadWorldDataAsync(CancellationToken token)
    {
        await Task.Delay(700, token);
        return new WorldData();
    }

    private async Task<SettingsData> LoadSettingsAsync(CancellationToken token)
    {
        await Task.Delay(300, token);
        return new SettingsData();
    }

    private async Task LoadAssetsAsync(CancellationToken token)
    {
        await Task.Delay(1000, token);
    }

    private async Task FadeInAsync(float duration, CancellationToken token)
    {
        await Task.Delay((int)(duration * 1000), token);
    }

    private async Task ScaleUpAsync(float duration, CancellationToken token)
    {
        await Task.Delay((int)(duration * 1000), token);
    }

    private async Task MoveToAsync(Vector3 target, float duration, CancellationToken token)
    {
        await Task.Delay((int)(duration * 1000), token);
    }

    // 데이터 클래스들
    public class GameData
    {
        public PlayerData Player;
        public WorldData World;
        public SettingsData Settings;
    }

    public class PlayerData { }
    public class WorldData { }
    public class SettingsData { }
}
```

---

## 주의사항

1. **Task.Run 과다 사용**: I/O-bound 작업에는 Task.Run 불필요
2. **await 누락**: Task를 await 없이 사용하면 예외가 무시될 수 있음
3. **취소 토큰 전달**: 모든 비동기 메서드에 취소 토큰 전달 권장
4. **.Result/.Wait() 회피**: 메인 스레드에서 데드락 발생 가능
5. **예외 관찰**: await하지 않는 Task의 예외도 처리 필요

---

## 참고 자료

- [Microsoft: Task-based Asynchronous Pattern](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)
- [Microsoft: Task Class](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task)

---

## 다음 섹션

[10. ConfigureAwait & 컨텍스트 제어](./10-configure-await.md)
