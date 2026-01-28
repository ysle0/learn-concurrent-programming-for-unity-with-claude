# 08. async/await 기초

## 개요

`async`와 `await`는 C# 5.0(2012)에 도입된 비동기 프로그래밍의 핵심 키워드입니다. 복잡한 콜백 패턴 없이 동기 코드처럼 읽히는 비동기 코드를 작성할 수 있게 해줍니다. Unity에서도 2017 버전부터 지원하며, 현재는 필수적인 기술입니다.

---

## 1. async/await 기본 문법

### 기본 구조

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;
using System.Threading;

public class AsyncAwaitBasicsExample : MonoBehaviour
{
    /*
    async/await 기본 구조:

    async 반환타입 메서드이름()
    {
        // 동기 코드
        var result = await 비동기작업();
        // await 이후 코드 (continuation)
    }

    반환 타입:
    - void: fire-and-forget (예외 처리 어려움)
    - Task: 완료만 추적
    - Task<T>: 결과값 반환
    - ValueTask<T>: 최적화된 버전
    */

    private async void Start()
    {
        Debug.Log("Start 시작");

        // await로 비동기 작업 대기
        string result = await GetDataAsync();
        Debug.Log($"결과: {result}");

        Debug.Log("Start 완료");
    }

    // async 메서드 정의
    private async Task<string> GetDataAsync()
    {
        Debug.Log("GetDataAsync 시작");

        // await: 비동기 작업이 완료될 때까지 대기
        // 이 동안 스레드는 다른 작업을 수행할 수 있음
        await Task.Delay(1000); // 1초 대기

        Debug.Log("GetDataAsync 완료");
        return "Hello Async World!";
    }
}
```

### async 키워드

```csharp
using UnityEngine;
using System.Threading.Tasks;

public class AsyncKeywordExample : MonoBehaviour
{
    // =============================================
    // async의 역할
    // =============================================

    /*
    async 키워드의 역할:
    1. 메서드 내에서 await 사용을 허용
    2. 컴파일러가 상태 머신(State Machine) 생성
    3. 반환 타입을 Task로 래핑 (void 제외)
    */

    // async void - 이벤트 핸들러용 (권장하지 않음)
    private async void AsyncVoidMethod()
    {
        await Task.Delay(100);
        // 예외가 발생하면 잡기 어려움!
    }

    // async Task - 완료 추적 가능
    private async Task AsyncTaskMethod()
    {
        await Task.Delay(100);
        // 호출자가 await로 완료 대기 가능
    }

    // async Task<T> - 결과 반환
    private async Task<int> AsyncTaskWithResultMethod()
    {
        await Task.Delay(100);
        return 42;
    }

    // =============================================
    // async 없이 Task 반환
    // =============================================

    // async 없이도 Task 반환 가능 (await 미사용 시)
    private Task<int> ReturnTaskWithoutAsync()
    {
        // Task.FromResult: 이미 완료된 Task 생성
        return Task.FromResult(42);
    }

    private Task ReturnCompletedTask()
    {
        // Task.CompletedTask: 이미 완료된 Task
        return Task.CompletedTask;
    }

    // =============================================
    // async 메서드 사용
    // =============================================

    private async void Start()
    {
        // async void 호출 (완료 대기 불가)
        AsyncVoidMethod();
        Debug.Log("AsyncVoidMethod 호출됨 (대기 안 함)");

        // async Task 호출 (완료 대기)
        await AsyncTaskMethod();
        Debug.Log("AsyncTaskMethod 완료됨");

        // async Task<T> 호출 (결과 받기)
        int result = await AsyncTaskWithResultMethod();
        Debug.Log($"결과: {result}");
    }
}
```

### await 키워드

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;
using System.Threading;

public class AwaitKeywordExample : MonoBehaviour
{
    /*
    await 키워드의 역할:
    1. 비동기 작업의 완료를 비동기적으로 대기
    2. 완료되면 결과를 추출
    3. 현재 스레드를 블로킹하지 않음
    4. 완료 후 원래 컨텍스트로 복귀 (기본 설정)
    */

    private async void Start()
    {
        Debug.Log($"Start 시작 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        // await 사용 가능한 것들 (awaitable)
        await AwaitExamples();

        Debug.Log($"Start 완료 - Thread: {Thread.CurrentThread.ManagedThreadId}");
    }

    private async Task AwaitExamples()
    {
        // =============================================
        // 1. Task 대기
        // =============================================
        await Task.Delay(100);
        Debug.Log("Task.Delay 완료");

        // =============================================
        // 2. Task<T> 대기 및 결과 받기
        // =============================================
        int result = await GetNumberAsync();
        Debug.Log($"GetNumberAsync 결과: {result}");

        // =============================================
        // 3. Task.Run으로 백그라운드 작업
        // =============================================
        int computed = await Task.Run(() =>
        {
            // 이 코드는 ThreadPool에서 실행됨
            Debug.Log($"Task.Run 내부 - Thread: {Thread.CurrentThread.ManagedThreadId}");
            return HeavyComputation();
        });
        // await 후에는 메인 스레드로 복귀
        Debug.Log($"Task.Run 후 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        // =============================================
        // 4. 여러 Task 동시 대기
        // =============================================
        Task<int> task1 = GetNumberAsync();
        Task<int> task2 = GetNumberAsync();
        Task<int> task3 = GetNumberAsync();

        // 모두 완료될 때까지 대기
        int[] results = await Task.WhenAll(task1, task2, task3);
        Debug.Log($"WhenAll 결과: {string.Join(", ", results)}");

        // 하나라도 완료되면 진행
        Task<int> firstCompleted = await Task.WhenAny(task1, task2, task3);
        Debug.Log($"WhenAny 결과: {firstCompleted.Result}");
    }

    private async Task<int> GetNumberAsync()
    {
        await Task.Delay(100);
        return UnityEngine.Random.Range(1, 100);
    }

    private int HeavyComputation()
    {
        int sum = 0;
        for (int i = 0; i < 1000000; i++)
        {
            sum += i;
        }
        return sum;
    }
}
```

---

## 2. 상태 머신 (State Machine)

### 컴파일러가 생성하는 코드

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;
using System.Runtime.CompilerServices;

public class StateMachineExample : MonoBehaviour
{
    /*
    async 메서드는 컴파일러에 의해 상태 머신으로 변환됩니다.

    원본 코드:
    async Task<int> CalculateAsync()
    {
        int a = await GetAAsync();
        int b = await GetBAsync();
        return a + b;
    }

    컴파일러 변환 (개념적):
    - IAsyncStateMachine 구현 구조체 생성
    - 각 await 지점이 하나의 상태(state)가 됨
    - MoveNext() 메서드에서 상태에 따라 분기
    */

    private async void Start()
    {
        int result = await CalculateAsync();
        Debug.Log($"결과: {result}");
    }

    // 원본 async 메서드
    private async Task<int> CalculateAsync()
    {
        Debug.Log("State 0: 시작");

        int a = await GetValueAsync(10);
        Debug.Log($"State 1: a = {a}");

        int b = await GetValueAsync(20);
        Debug.Log($"State 2: b = {b}");

        return a + b;
    }

    private async Task<int> GetValueAsync(int value)
    {
        await Task.Delay(100);
        return value;
    }

    // =============================================
    // 컴파일러가 생성하는 상태 머신 (개념적 표현)
    // =============================================

    /*
    [CompilerGenerated]
    private struct CalculateAsyncStateMachine : IAsyncStateMachine
    {
        public int state; // 현재 상태
        public AsyncTaskMethodBuilder<int> builder;

        // 지역 변수들
        public int a;
        public int b;

        // 대기 중인 awaiter
        private TaskAwaiter<int> awaiter;

        public void MoveNext()
        {
            int result;
            try
            {
                switch (state)
                {
                    case 0: // 초기 상태
                        awaiter = GetValueAsync(10).GetAwaiter();
                        if (!awaiter.IsCompleted)
                        {
                            state = 1;
                            builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);
                            return;
                        }
                        goto case 1;

                    case 1: // GetValueAsync(10) 완료 후
                        a = awaiter.GetResult();
                        awaiter = GetValueAsync(20).GetAwaiter();
                        if (!awaiter.IsCompleted)
                        {
                            state = 2;
                            builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);
                            return;
                        }
                        goto case 2;

                    case 2: // GetValueAsync(20) 완료 후
                        b = awaiter.GetResult();
                        result = a + b;
                        break;
                }
            }
            catch (Exception ex)
            {
                builder.SetException(ex);
                return;
            }

            builder.SetResult(result);
        }

        public void SetStateMachine(IAsyncStateMachine stateMachine) { }
    }
    */
}
```

### 상태 머신 시각화

```
┌─────────────────────────────────────────────────────────────────┐
│                    async 메서드 상태 머신                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  async Task<int> CalculateAsync()                               │
│  {                                                               │
│      int a = await GetAAsync();  ← State 0 → State 1            │
│      int b = await GetBAsync();  ← State 1 → State 2            │
│      return a + b;               ← State 2 → 완료               │
│  }                                                               │
│                                                                  │
│  실행 흐름:                                                      │
│                                                                  │
│  ┌────────┐    GetAAsync()    ┌────────┐    GetBAsync()         │
│  │State 0 │ ───완료 안됨────▶ │ 대기   │ ◀──── 콜백 ◀───┐       │
│  └────────┘                   └────────┘                │       │
│      │                                                   │       │
│      │ 완료됨                                            │       │
│      ▼                                                   │       │
│  ┌────────┐    GetBAsync()    ┌────────┐               │       │
│  │State 1 │ ───완료 안됨────▶ │ 대기   │ ◀──── 콜백 ───┘       │
│  └────────┘                   └────────┘                        │
│      │                                                          │
│      │ 완료됨                                                   │
│      ▼                                                          │
│  ┌────────┐                                                     │
│  │State 2 │ ──▶ return a + b ──▶ 완료                          │
│  └────────┘                                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 실행 흐름 이해

### 동기적 완료 vs 비동기적 완료

```csharp
using UnityEngine;
using System.Threading.Tasks;
using System.Threading;

public class ExecutionFlowExample : MonoBehaviour
{
    private async void Start()
    {
        Debug.Log($"[1] Start 시작 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        // 동기적 완료 (이미 완료된 Task)
        await SynchronousCompletionAsync();

        // 비동기적 완료 (실제 대기 발생)
        await AsynchronousCompletionAsync();

        Debug.Log($"[8] Start 완료 - Thread: {Thread.CurrentThread.ManagedThreadId}");
    }

    // =============================================
    // 동기적 완료 - await가 즉시 반환
    // =============================================

    private async Task SynchronousCompletionAsync()
    {
        Debug.Log($"[2] SynchronousCompletion 시작");

        // Task.FromResult는 이미 완료된 Task 반환
        // await는 즉시 완료되어 스레드 전환 없음
        int result = await Task.FromResult(42);

        Debug.Log($"[3] SynchronousCompletion 완료 (result: {result})");
        // 동기적으로 실행됨 - 같은 스택 프레임
    }

    // =============================================
    // 비동기적 완료 - 실제 대기 발생
    // =============================================

    private async Task AsynchronousCompletionAsync()
    {
        Debug.Log($"[4] AsynchronousCompletion 시작 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        // Task.Delay는 실제로 비동기 대기
        // 이 시점에서 메서드는 "일시 중지"되고 호출자에게 반환
        await Task.Delay(100);

        // 100ms 후 여기서 재개됨
        // UnitySynchronizationContext가 메인 스레드로 복귀시킴
        Debug.Log($"[5] Task.Delay 후 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        // Task.Run은 ThreadPool에서 실행
        int result = await Task.Run(() =>
        {
            Debug.Log($"[6] Task.Run 내부 - Thread: {Thread.CurrentThread.ManagedThreadId}");
            return 100;
        });

        // await 후 다시 메인 스레드로 복귀
        Debug.Log($"[7] Task.Run 후 - Thread: {Thread.CurrentThread.ManagedThreadId}");
    }
}

/*
출력 예시:
[1] Start 시작 - Thread: 1
[2] SynchronousCompletion 시작
[3] SynchronousCompletion 완료 (result: 42)  ← 동기적 (같은 호출)
[4] AsynchronousCompletion 시작 - Thread: 1
[5] Task.Delay 후 - Thread: 1               ← 메인 스레드로 복귀
[6] Task.Run 내부 - Thread: 4               ← ThreadPool 스레드
[7] Task.Run 후 - Thread: 1                 ← 메인 스레드로 복귀
[8] Start 완료 - Thread: 1
*/
```

### await 전후의 컨텍스트

```csharp
using UnityEngine;
using System.Threading.Tasks;
using System.Threading;

public class AwaitContextExample : MonoBehaviour
{
    private async void Start()
    {
        await DemonstrateContext();
    }

    private async Task DemonstrateContext()
    {
        // =============================================
        // Unity의 UnitySynchronizationContext
        // =============================================

        var syncContext = SynchronizationContext.Current;
        Debug.Log($"SyncContext: {syncContext?.GetType().Name}");
        // 출력: UnitySynchronizationContext

        Debug.Log($"await 전 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        // await 시점에 현재 SynchronizationContext가 캡처됨
        await Task.Delay(100);

        // await 완료 후 캡처된 컨텍스트(메인 스레드)로 복귀
        Debug.Log($"await 후 - Thread: {Thread.CurrentThread.ManagedThreadId}");

        // ✅ Unity API 안전하게 사용 가능
        transform.position = Vector3.one;

        // =============================================
        // Task.Run 내부는 다른 컨텍스트
        // =============================================

        await Task.Run(() =>
        {
            var innerContext = SynchronizationContext.Current;
            Debug.Log($"Task.Run 내 SyncContext: {innerContext?.GetType().Name ?? "null"}");
            // 출력: null (ThreadPool에는 SyncContext 없음)

            // ❌ 여기서 Unity API 사용 불가!
            // transform.position = Vector3.zero; // 예외 발생
        });

        // Task.Run 완료 후 다시 메인 스레드
        Debug.Log($"Task.Run 후 - Thread: {Thread.CurrentThread.ManagedThreadId}");
    }
}
```

---

## 4. 반환 타입

### void, Task, Task<T> 비교

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;

public class ReturnTypesExample : MonoBehaviour
{
    private async void Start()
    {
        // =============================================
        // 1. async void - 사용 주의!
        // =============================================

        // fire-and-forget (완료 대기 불가)
        AsyncVoidMethod();
        Debug.Log("AsyncVoidMethod 호출 후 (대기 안 함)");

        // 예외 처리가 어려움
        try
        {
            AsyncVoidWithException();
        }
        catch (Exception)
        {
            // 이 catch는 동작하지 않음!
            Debug.Log("이 메시지는 출력되지 않음");
        }

        await Task.Delay(500); // 예외가 발생할 시간 줌

        // =============================================
        // 2. async Task - 권장
        // =============================================

        // 완료 대기 가능
        await AsyncTaskMethod();
        Debug.Log("AsyncTaskMethod 완료됨");

        // 예외 처리 가능
        try
        {
            await AsyncTaskWithException();
        }
        catch (InvalidOperationException ex)
        {
            Debug.Log($"예외 잡힘: {ex.Message}");
        }

        // =============================================
        // 3. async Task<T> - 결과가 있을 때
        // =============================================

        int result = await AsyncTaskWithResult();
        Debug.Log($"결과: {result}");

        // 복잡한 결과
        var data = await GetUserDataAsync();
        Debug.Log($"사용자: {data.Name}, {data.Age}세");
    }

    // =============================================
    // async void - 이벤트 핸들러용
    // =============================================

    private async void AsyncVoidMethod()
    {
        await Task.Delay(100);
        Debug.Log("AsyncVoidMethod 완료");
    }

    private async void AsyncVoidWithException()
    {
        await Task.Delay(100);
        throw new InvalidOperationException("async void 예외!");
        // 이 예외는 앱을 크래시시킬 수 있음!
    }

    // =============================================
    // async Task - 권장 패턴
    // =============================================

    private async Task AsyncTaskMethod()
    {
        await Task.Delay(100);
        Debug.Log("AsyncTaskMethod 내부 완료");
    }

    private async Task AsyncTaskWithException()
    {
        await Task.Delay(100);
        throw new InvalidOperationException("async Task 예외!");
        // 이 예외는 await에서 잡을 수 있음
    }

    // =============================================
    // async Task<T> - 결과 반환
    // =============================================

    private async Task<int> AsyncTaskWithResult()
    {
        await Task.Delay(100);
        return 42;
    }

    private async Task<UserData> GetUserDataAsync()
    {
        await Task.Delay(100);
        return new UserData { Name = "홍길동", Age = 25 };
    }

    private class UserData
    {
        public string Name;
        public int Age;
    }
}

/*
┌─────────────────────────────────────────────────────────────────┐
│                    반환 타입 선택 가이드                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  async void                                                      │
│  ├─ 사용: 이벤트 핸들러 (Button.onClick += async () => ...)       │
│  ├─ 단점: 완료 대기 불가, 예외 처리 어려움                        │
│  └─ 권장: 가능하면 피하기                                        │
│                                                                  │
│  async Task                                                      │
│  ├─ 사용: 결과가 없는 비동기 작업                                 │
│  ├─ 장점: 완료 대기 가능, 예외 처리 가능                          │
│  └─ 권장: 기본 선택                                              │
│                                                                  │
│  async Task<T>                                                   │
│  ├─ 사용: 결과를 반환하는 비동기 작업                             │
│  ├─ 장점: 결과값 반환, 완료 대기, 예외 처리                       │
│  └─ 권장: 결과가 필요할 때                                        │
│                                                                  │
│  ValueTask<T>                                                    │
│  ├─ 사용: 성능 최적화가 필요할 때                                 │
│  ├─ 장점: 동기 완료 시 할당 없음                                  │
│  └─ 권장: 핫 패스, 자주 호출되는 메서드                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
*/
```

---

## 5. 예외 처리

### 기본 예외 처리

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;

public class AsyncExceptionHandlingExample : MonoBehaviour
{
    private async void Start()
    {
        await BasicExceptionHandling();
        await MultipleExceptionHandling();
        await ExceptionInTaskRun();
    }

    // =============================================
    // 기본 예외 처리
    // =============================================

    private async Task BasicExceptionHandling()
    {
        Debug.Log("=== 기본 예외 처리 ===");

        try
        {
            await ThrowingMethodAsync();
        }
        catch (InvalidOperationException ex)
        {
            // await에서 예외가 던져짐
            Debug.Log($"예외 잡힘: {ex.Message}");
        }
        finally
        {
            Debug.Log("finally 블록 실행");
        }
    }

    private async Task ThrowingMethodAsync()
    {
        await Task.Delay(100);
        throw new InvalidOperationException("비동기 예외!");
    }

    // =============================================
    // 여러 Task에서의 예외
    // =============================================

    private async Task MultipleExceptionHandling()
    {
        Debug.Log("\n=== 여러 Task 예외 처리 ===");

        var task1 = ThrowAfterDelay("예외 1", 100);
        var task2 = ThrowAfterDelay("예외 2", 200);
        var task3 = ThrowAfterDelay("예외 3", 300);

        try
        {
            // WhenAll은 모든 Task가 완료될 때까지 대기
            await Task.WhenAll(task1, task2, task3);
        }
        catch (Exception ex)
        {
            // 첫 번째 예외만 전파됨
            Debug.Log($"잡힌 예외: {ex.Message}");
        }

        // 모든 예외 확인
        Debug.Log($"Task1 예외: {task1.Exception?.InnerException?.Message}");
        Debug.Log($"Task2 예외: {task2.Exception?.InnerException?.Message}");
        Debug.Log($"Task3 예외: {task3.Exception?.InnerException?.Message}");
    }

    private async Task ThrowAfterDelay(string message, int delayMs)
    {
        await Task.Delay(delayMs);
        throw new InvalidOperationException(message);
    }

    // =============================================
    // Task.Run 내부 예외
    // =============================================

    private async Task ExceptionInTaskRun()
    {
        Debug.Log("\n=== Task.Run 예외 처리 ===");

        try
        {
            await Task.Run(() =>
            {
                throw new InvalidOperationException("Task.Run 내부 예외!");
            });
        }
        catch (InvalidOperationException ex)
        {
            // Task.Run 내부 예외도 await에서 잡힘
            Debug.Log($"Task.Run 예외 잡힘: {ex.Message}");
        }
    }
}
```

### AggregateException 처리

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;
using System.Collections.Generic;

public class AggregateExceptionExample : MonoBehaviour
{
    private async void Start()
    {
        await HandleAggregateException();
    }

    private async Task HandleAggregateException()
    {
        var tasks = new List<Task>
        {
            ThrowAsync("Error A"),
            ThrowAsync("Error B"),
            ThrowAsync("Error C")
        };

        Task allTasks = Task.WhenAll(tasks);

        try
        {
            await allTasks;
        }
        catch (Exception ex)
        {
            Debug.Log($"직접 잡힌 예외: {ex.Message}");

            // AggregateException을 통해 모든 예외 접근
            if (allTasks.Exception != null)
            {
                Debug.Log($"\n모든 예외 ({allTasks.Exception.InnerExceptions.Count}개):");
                foreach (var innerEx in allTasks.Exception.InnerExceptions)
                {
                    Debug.Log($"  - {innerEx.Message}");
                }
            }
        }
    }

    private async Task ThrowAsync(string message)
    {
        await Task.Delay(100);
        throw new InvalidOperationException(message);
    }
}
```

---

## 6. Unity 실전 예제

### 안전한 비동기 패턴

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class UnityAsyncPatternsExample : MonoBehaviour
{
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

    private async void Start()
    {
        try
        {
            await SafeAsyncOperation(cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 취소되었습니다.");
        }
        catch (Exception ex)
        {
            Debug.LogError($"에러: {ex.Message}");
        }
    }

    // =============================================
    // 안전한 비동기 작업 패턴
    // =============================================

    private async Task SafeAsyncOperation(CancellationToken token)
    {
        // 1. 취소 확인
        token.ThrowIfCancellationRequested();

        // 2. 메인 스레드에서 Unity 데이터 준비
        Vector3 startPosition = transform.position;
        string objectName = gameObject.name;

        // 3. 백그라운드 작업
        var result = await Task.Run(() =>
        {
            token.ThrowIfCancellationRequested();

            // 순수 계산만 수행
            return HeavyCalculation(startPosition);

        }, token);

        // 4. 취소 및 파괴 확인
        token.ThrowIfCancellationRequested();
        if (this == null) return;

        // 5. 메인 스레드에서 결과 적용
        transform.position = result;
        Debug.Log($"{objectName} 위치 업데이트 완료");
    }

    private Vector3 HeavyCalculation(Vector3 input)
    {
        // CPU 집약적 계산 시뮬레이션
        Thread.Sleep(500);
        return input + Vector3.up * 10f;
    }

    // =============================================
    // 반복 작업 패턴
    // =============================================

    private async Task RepeatingAsyncTask(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            try
            {
                // 작업 수행
                await DoWorkAsync(token);

                // 간격 대기
                await Task.Delay(1000, token);
            }
            catch (OperationCanceledException)
            {
                break;
            }
        }
    }

    private async Task DoWorkAsync(CancellationToken token)
    {
        await Task.Delay(100, token);
        Debug.Log($"작업 완료: {Time.time:F1}");
    }
}
```

### 진행률 보고

```csharp
using UnityEngine;
using UnityEngine.UI;
using System;
using System.Threading;
using System.Threading.Tasks;

public class ProgressReportingExample : MonoBehaviour
{
    [SerializeField] private Slider progressBar;
    [SerializeField] private Text statusText;

    private async void Start()
    {
        // IProgress<T>를 통한 진행률 보고
        var progress = new Progress<float>(value =>
        {
            // 이 콜백은 메인 스레드에서 실행됨
            if (progressBar != null)
                progressBar.value = value;
            if (statusText != null)
                statusText.text = $"진행률: {value:P0}";
        });

        await ProcessWithProgress(progress);
        Debug.Log("처리 완료!");
    }

    private async Task ProcessWithProgress(IProgress<float> progress)
    {
        int totalSteps = 100;

        for (int i = 0; i <= totalSteps; i++)
        {
            // 백그라운드 작업
            await Task.Run(() =>
            {
                Thread.Sleep(50); // 작업 시뮬레이션
            });

            // 진행률 보고 (메인 스레드로 마샬링됨)
            progress?.Report((float)i / totalSteps);
        }
    }
}
```

---

## 7. 일반적인 실수와 해결책

```csharp
using UnityEngine;
using System;
using System.Threading.Tasks;
using System.Threading;

public class CommonAsyncMistakesExample : MonoBehaviour
{
    // =============================================
    // 실수 1: async void 남용
    // =============================================

    // ❌ 잘못된 예
    private async void BadAsyncVoid()
    {
        await Task.Delay(100);
        throw new Exception("이 예외는 잡기 어려움!");
    }

    // ✅ 올바른 예
    private async Task GoodAsyncTask()
    {
        await Task.Delay(100);
        throw new Exception("이 예외는 잡을 수 있음");
    }

    // =============================================
    // 실수 2: .Result 또는 .Wait() 사용
    // =============================================

    // ❌ 잘못된 예 - 데드락 위험!
    private void BadBlockingCall()
    {
        // 메인 스레드에서 .Result 호출 시 데드락!
        // int result = GetValueAsync().Result;
    }

    // ✅ 올바른 예
    private async void GoodAsyncCall()
    {
        int result = await GetValueAsync();
        Debug.Log($"결과: {result}");
    }

    private async Task<int> GetValueAsync()
    {
        await Task.Delay(100);
        return 42;
    }

    // =============================================
    // 실수 3: 취소 미구현
    // =============================================

    // ❌ 잘못된 예
    private async void BadNoCancel()
    {
        while (true)
        {
            await Task.Delay(1000);
            // 오브젝트가 파괴되어도 계속 실행!
        }
    }

    // ✅ 올바른 예
    private CancellationTokenSource cts;

    private void OnEnable()
    {
        cts = new CancellationTokenSource();
        GoodWithCancel(cts.Token);
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }

    private async void GoodWithCancel(CancellationToken token)
    {
        try
        {
            while (!token.IsCancellationRequested)
            {
                await Task.Delay(1000, token);
            }
        }
        catch (OperationCanceledException)
        {
            Debug.Log("정상적으로 취소됨");
        }
    }

    // =============================================
    // 실수 4: 파괴된 오브젝트 접근
    // =============================================

    // ❌ 잘못된 예
    private async void BadAccessAfterDestroy()
    {
        await Task.Delay(5000);
        // 5초 후 오브젝트가 파괴되었을 수 있음!
        transform.position = Vector3.zero; // NullReferenceException 또는 MissingReferenceException
    }

    // ✅ 올바른 예
    private async void GoodCheckDestroy()
    {
        await Task.Delay(5000);

        // 파괴 여부 확인
        if (this == null) return;

        transform.position = Vector3.zero;
    }

    // =============================================
    // 실수 5: async void 이벤트 핸들러에서 예외 미처리
    // =============================================

    // ✅ 올바른 예 - 예외를 내부에서 처리
    private async void OnButtonClick()
    {
        try
        {
            await DoAsyncWork();
        }
        catch (Exception ex)
        {
            Debug.LogError($"버튼 클릭 처리 중 에러: {ex.Message}");
        }
    }

    private async Task DoAsyncWork()
    {
        await Task.Delay(100);
    }
}

/*
┌─────────────────────────────────────────────────────────────────┐
│                    async/await 체크리스트                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ✅ 해야 할 것:                                                  │
│  ├─ async Task 반환 (void 대신)                                  │
│  ├─ CancellationToken 사용                                      │
│  ├─ try-catch로 예외 처리                                        │
│  ├─ 파괴 여부 확인 (this == null)                                │
│  ├─ ConfigureAwait 고려 (라이브러리 코드)                        │
│  └─ 적절한 곳에서 await                                          │
│                                                                  │
│  ❌ 하지 말아야 할 것:                                           │
│  ├─ .Result, .Wait() 사용 (데드락!)                             │
│  ├─ async void 남용                                              │
│  ├─ 취소 무시                                                    │
│  ├─ 예외 무시                                                    │
│  └─ Task.Run 내부에서 Unity API 호출                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
*/
```

---

## 주의사항

1. **async void 주의**: 이벤트 핸들러 외에는 사용 자제
2. **.Result/.Wait() 금지**: 메인 스레드에서 데드락 발생
3. **취소 구현**: CancellationToken으로 정리 보장
4. **파괴 확인**: await 후 오브젝트 파괴 여부 확인
5. **예외 처리**: 모든 async 경로에서 예외 처리

---

## 참고 자료

- [Microsoft: Async/Await Best Practices](https://docs.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming)
- [Stephen Cleary: Async and Await](https://blog.stephencleary.com/2012/02/async-and-await.html)
- [Unity: Async/Await Support](https://docs.unity3d.com/2023.1/Documentation/Manual/async-await-support.html)

---

## 다음 섹션

[09. TAP (Task Asynchronous Programming)](./09-tap.md)
