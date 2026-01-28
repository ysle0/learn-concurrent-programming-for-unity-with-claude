# 35. 예외 처리 패턴

## 개요

비동기 프로그래밍에서의 예외 처리는 동기 코드와 다른 특성을 가집니다. **AggregateException**, 예외 전파, Try 패턴 등을 이해하고 Unity 환경에 맞게 적용해야 합니다.

---

## async/await 예외 처리

### 기본 예외 처리

```csharp
public class AsyncExceptionBasics : MonoBehaviour
{
    async void Start()
    {
        // async 메서드의 예외는 await에서 발생
        try
        {
            await MayThrowAsync();
        }
        catch (InvalidOperationException ex)
        {
            Debug.LogError($"예외 발생: {ex.Message}");
        }
    }

    async Task MayThrowAsync()
    {
        await Task.Delay(100);
        throw new InvalidOperationException("Something went wrong!");
    }
}
```

### async void vs async Task 예외

```csharp
public class AsyncVoidException : MonoBehaviour
{
    void Start()
    {
        // async void: 예외가 SynchronizationContext로 전파
        // Unity에서는 메인 스레드에서 처리됨
        AsyncVoidMethod(); // 예외 catch 불가!

        // async Task: await로 예외 받을 수 있음
        _ = AsyncTaskMethod(); // Fire-and-forget (예외 유실 가능)

        // 올바른 방법
        HandleAsync();
    }

    async void AsyncVoidMethod()
    {
        await Task.Delay(100);
        throw new Exception("async void exception");
        // 이 예외는 Application.logMessageReceived로 감지
    }

    async Task AsyncTaskMethod()
    {
        await Task.Delay(100);
        throw new Exception("async Task exception");
    }

    async void HandleAsync()
    {
        try
        {
            await AsyncTaskMethod();
        }
        catch (Exception ex)
        {
            Debug.LogError($"Caught: {ex.Message}");
        }
    }
}
```

---

## AggregateException

### Task.WhenAll 예외

```csharp
public class AggregateExceptionExample : MonoBehaviour
{
    async void Start()
    {
        var tasks = new[]
        {
            ThrowAsync("Error 1"),
            ThrowAsync("Error 2"),
            Task.CompletedTask
        };

        try
        {
            await Task.WhenAll(tasks);
        }
        catch (Exception ex)
        {
            // await는 첫 번째 예외만 전파
            Debug.LogError($"First exception: {ex.Message}");
        }

        // 모든 예외 확인
        try
        {
            Task.WhenAll(tasks).Wait();
        }
        catch (AggregateException ae)
        {
            foreach (var innerEx in ae.InnerExceptions)
            {
                Debug.LogError($"Inner: {innerEx.Message}");
            }
        }
    }

    // 더 나은 방법: 모든 예외 수집
    async void BetterApproach()
    {
        var tasks = new[]
        {
            ThrowAsync("Error 1"),
            ThrowAsync("Error 2"),
            Task.CompletedTask
        };

        var allTasks = Task.WhenAll(tasks);

        try
        {
            await allTasks;
        }
        catch
        {
            // allTasks.Exception에서 AggregateException 접근
            if (allTasks.Exception != null)
            {
                foreach (var ex in allTasks.Exception.InnerExceptions)
                {
                    Debug.LogError($"Exception: {ex.Message}");
                }
            }
        }
    }

    async Task ThrowAsync(string message)
    {
        await Task.Delay(100);
        throw new Exception(message);
    }
}
```

### Flatten

```csharp
public class FlattenExample : MonoBehaviour
{
    void Start()
    {
        try
        {
            var task = Task.Run(() =>
            {
                throw new AggregateException(
                    new Exception("Inner 1"),
                    new AggregateException(
                        new Exception("Nested 1"),
                        new Exception("Nested 2")));
            });

            task.Wait();
        }
        catch (AggregateException ae)
        {
            // Flatten: 중첩된 AggregateException 평탄화
            foreach (var ex in ae.Flatten().InnerExceptions)
            {
                Debug.LogError($"Flattened: {ex.Message}");
            }
        }
    }
}
```

---

## 예외 전파 패턴

### 예외 래핑

```csharp
public class ExceptionWrapping : MonoBehaviour
{
    async void Start()
    {
        try
        {
            await LoadGameDataAsync();
        }
        catch (GameDataException ex)
        {
            Debug.LogError($"Game data error: {ex.Message}");
            Debug.LogError($"Original: {ex.InnerException?.Message}");
        }
    }

    async Task LoadGameDataAsync()
    {
        try
        {
            await FetchFromServerAsync();
        }
        catch (HttpRequestException ex)
        {
            // 도메인 예외로 래핑
            throw new GameDataException("Failed to load game data", ex);
        }
    }

    async Task FetchFromServerAsync()
    {
        await Task.Delay(100);
        throw new HttpRequestException("Network error");
    }
}

public class GameDataException : Exception
{
    public GameDataException(string message, Exception inner)
        : base(message, inner) { }
}
```

### 예외 필터링

```csharp
public class ExceptionFiltering : MonoBehaviour
{
    async void Start()
    {
        try
        {
            await MakeRequestAsync();
        }
        catch (HttpRequestException ex) when (ex.Message.Contains("timeout"))
        {
            Debug.Log("타임아웃 발생 - 재시도");
            await MakeRequestAsync();
        }
        catch (HttpRequestException ex) when (ex.Message.Contains("404"))
        {
            Debug.LogError("리소스를 찾을 수 없음");
        }
        catch (HttpRequestException ex)
        {
            Debug.LogError($"기타 HTTP 에러: {ex.Message}");
        }
    }

    async Task MakeRequestAsync()
    {
        await Task.Delay(100);
        throw new HttpRequestException("timeout occurred");
    }
}
```

---

## Try 패턴

### 예외 없는 결과 반환

```csharp
public class TryPattern : MonoBehaviour
{
    async void Start()
    {
        // 패턴 1: Tuple 반환
        var (success, data) = await TryLoadDataAsync();
        if (success)
        {
            Debug.Log($"Data: {data}");
        }
        else
        {
            Debug.Log("Failed to load data");
        }

        // 패턴 2: Result 타입
        var result = await LoadDataWithResultAsync();
        if (result.IsSuccess)
        {
            Debug.Log($"Data: {result.Value}");
        }
        else
        {
            Debug.LogError($"Error: {result.Error}");
        }
    }

    async Task<(bool Success, string Data)> TryLoadDataAsync()
    {
        try
        {
            await Task.Delay(100);
            return (true, "Loaded data");
        }
        catch
        {
            return (false, null);
        }
    }

    async Task<Result<string>> LoadDataWithResultAsync()
    {
        try
        {
            await Task.Delay(100);
            return Result<string>.Ok("Loaded data");
        }
        catch (Exception ex)
        {
            return Result<string>.Fail(ex.Message);
        }
    }
}

// Result 타입 정의
public class Result<T>
{
    public bool IsSuccess { get; private set; }
    public T Value { get; private set; }
    public string Error { get; private set; }

    public static Result<T> Ok(T value) =>
        new Result<T> { IsSuccess = true, Value = value };

    public static Result<T> Fail(string error) =>
        new Result<T> { IsSuccess = false, Error = error };
}
```

### UniTask SuppressCancellationThrow

```csharp
using Cysharp.Threading.Tasks;

public class UniTaskTryPattern : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // 취소 예외 억제
        var (isCanceled, result) = await LoadAsync()
            .SuppressCancellationThrow();

        if (!isCanceled)
        {
            Debug.Log($"Result: {result}");
        }
    }

    async UniTask<string> LoadAsync()
    {
        await UniTask.Delay(1000, cancellationToken: destroyCancellationToken);
        return "data";
    }
}
```

---

## 재시도 패턴

### 간단한 재시도

```csharp
public class SimpleRetry : MonoBehaviour
{
    async void Start()
    {
        try
        {
            var result = await RetryAsync(
                LoadDataAsync,
                maxRetries: 3,
                delay: TimeSpan.FromSeconds(1));

            Debug.Log($"Result: {result}");
        }
        catch (Exception ex)
        {
            Debug.LogError($"All retries failed: {ex.Message}");
        }
    }

    async Task<T> RetryAsync<T>(
        Func<Task<T>> operation,
        int maxRetries,
        TimeSpan delay)
    {
        for (int i = 0; i < maxRetries; i++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (i < maxRetries - 1)
            {
                Debug.LogWarning($"Retry {i + 1}/{maxRetries}: {ex.Message}");
                await Task.Delay(delay);
            }
        }

        return await operation(); // 마지막 시도
    }

    async Task<string> LoadDataAsync()
    {
        await Task.Delay(100);
        if (UnityEngine.Random.value < 0.7f)
            throw new Exception("Random failure");
        return "Success!";
    }
}
```

### 지수 백오프

```csharp
public class ExponentialBackoff : MonoBehaviour
{
    async void Start()
    {
        try
        {
            var result = await RetryWithBackoffAsync(
                LoadDataAsync,
                maxRetries: 5,
                initialDelay: TimeSpan.FromMilliseconds(100),
                maxDelay: TimeSpan.FromSeconds(5));

            Debug.Log($"Result: {result}");
        }
        catch (Exception ex)
        {
            Debug.LogError($"Failed: {ex.Message}");
        }
    }

    async Task<T> RetryWithBackoffAsync<T>(
        Func<Task<T>> operation,
        int maxRetries,
        TimeSpan initialDelay,
        TimeSpan maxDelay)
    {
        TimeSpan delay = initialDelay;

        for (int i = 0; i < maxRetries; i++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (i < maxRetries - 1)
            {
                Debug.LogWarning($"Retry {i + 1}/{maxRetries} after {delay.TotalMilliseconds}ms");

                await Task.Delay(delay);

                // 지수 증가 + 지터
                delay = TimeSpan.FromMilliseconds(
                    Math.Min(delay.TotalMilliseconds * 2 + UnityEngine.Random.Range(0, 100),
                             maxDelay.TotalMilliseconds));
            }
        }

        return await operation();
    }

    async Task<string> LoadDataAsync()
    {
        await Task.Delay(50);
        if (UnityEngine.Random.value < 0.8f)
            throw new HttpRequestException("Server busy");
        return "Data";
    }
}
```

### 조건부 재시도

```csharp
public class ConditionalRetry : MonoBehaviour
{
    async void Start()
    {
        try
        {
            var result = await RetryOnConditionAsync(
                LoadDataAsync,
                shouldRetry: ex => ex is HttpRequestException httpEx &&
                                   (httpEx.Message.Contains("503") ||
                                    httpEx.Message.Contains("timeout")),
                maxRetries: 3,
                delay: TimeSpan.FromSeconds(1));

            Debug.Log($"Result: {result}");
        }
        catch (Exception ex)
        {
            Debug.LogError($"Non-retryable error: {ex.Message}");
        }
    }

    async Task<T> RetryOnConditionAsync<T>(
        Func<Task<T>> operation,
        Func<Exception, bool> shouldRetry,
        int maxRetries,
        TimeSpan delay)
    {
        for (int i = 0; i < maxRetries; i++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (i < maxRetries - 1 && shouldRetry(ex))
            {
                Debug.LogWarning($"Retrying due to: {ex.Message}");
                await Task.Delay(delay);
            }
        }

        return await operation();
    }

    async Task<string> LoadDataAsync()
    {
        await Task.Delay(50);
        throw new HttpRequestException("503 Service Unavailable");
    }
}
```

---

## 전역 예외 처리

### Application.logMessageReceived

```csharp
public class GlobalExceptionHandler : MonoBehaviour
{
    void Awake()
    {
        Application.logMessageReceived += OnLogMessageReceived;
        AppDomain.CurrentDomain.UnhandledExceptionHandler += OnUnhandledException;
    }

    void OnLogMessageReceived(string condition, string stackTrace, LogType type)
    {
        if (type == LogType.Exception)
        {
            Debug.Log($"Exception caught: {condition}");
            // 분석 서버로 전송 등
        }
    }

    void OnUnhandledException(object sender, UnhandledExceptionEventArgs e)
    {
        var ex = e.ExceptionObject as Exception;
        Debug.LogError($"Unhandled exception: {ex?.Message}");
    }

    void OnDestroy()
    {
        Application.logMessageReceived -= OnLogMessageReceived;
    }
}
```

### TaskScheduler.UnobservedTaskException

```csharp
public class UnobservedExceptionHandler : MonoBehaviour
{
    void Awake()
    {
        TaskScheduler.UnobservedTaskException += OnUnobservedTaskException;
    }

    void OnUnobservedTaskException(object sender, UnobservedTaskExceptionEventArgs e)
    {
        Debug.LogError($"Unobserved task exception: {e.Exception.Message}");

        // 예외를 관찰된 것으로 표시 (앱 크래시 방지)
        e.SetObserved();
    }

    void OnDestroy()
    {
        TaskScheduler.UnobservedTaskException -= OnUnobservedTaskException;
    }
}
```

---

## 실전 패턴

### 패턴 1: 서비스 호출 래퍼

```csharp
public class ServiceCallWrapper
{
    public async Task<T> CallWithHandlingAsync<T>(
        Func<Task<T>> serviceCall,
        CancellationToken cancellationToken = default)
    {
        try
        {
            return await serviceCall();
        }
        catch (OperationCanceledException) when (cancellationToken.IsCancellationRequested)
        {
            // 정상 취소
            throw;
        }
        catch (HttpRequestException ex)
        {
            Debug.LogError($"Network error: {ex.Message}");
            throw new ServiceException("서버 연결 실패", ex);
        }
        catch (JsonException ex)
        {
            Debug.LogError($"Parse error: {ex.Message}");
            throw new ServiceException("데이터 파싱 실패", ex);
        }
        catch (Exception ex)
        {
            Debug.LogError($"Unexpected error: {ex.Message}");
            throw new ServiceException("알 수 없는 오류", ex);
        }
    }
}

public class ServiceException : Exception
{
    public ServiceException(string message, Exception inner) : base(message, inner) { }
}
```

### 패턴 2: 안전한 Fire-and-Forget

```csharp
public static class TaskExtensions
{
    public static async void FireAndForget(
        this Task task,
        Action<Exception> onException = null)
    {
        try
        {
            await task;
        }
        catch (Exception ex)
        {
            if (onException != null)
            {
                onException(ex);
            }
            else
            {
                Debug.LogException(ex);
            }
        }
    }
}

// 사용 예
public class FireAndForgetExample : MonoBehaviour
{
    void Start()
    {
        // 예외 무시
        DoSomethingAsync().FireAndForget();

        // 예외 처리
        DoSomethingAsync().FireAndForget(ex =>
        {
            Debug.LogError($"Background task failed: {ex.Message}");
        });
    }

    async Task DoSomethingAsync()
    {
        await Task.Delay(100);
    }
}
```

---

## 정리

### 예외 처리 요약

| 패턴 | 용도 |
|------|------|
| **try-catch** | 기본 예외 처리 |
| **AggregateException** | WhenAll 다중 예외 |
| **when 필터** | 조건부 예외 처리 |
| **Try 패턴** | 예외 없는 결과 반환 |
| **재시도** | 일시적 오류 복구 |

### 체크리스트

- [ ] async void 대신 async Task 사용하는가?
- [ ] OperationCanceledException 별도 처리하는가?
- [ ] 재시도 가능한 예외를 식별했는가?
- [ ] 전역 예외 핸들러 설정했는가?
- [ ] 사용자 친화적 에러 메시지 제공하는가?

---

## 참고 자료

- [Exception Handling in Async](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/handling-exceptions-in-async)
- [AggregateException](https://docs.microsoft.com/en-us/dotnet/api/system.aggregateexception)
- [Best Practices for Exceptions](https://docs.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions)
