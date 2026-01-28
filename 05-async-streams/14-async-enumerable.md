# 14. IAsyncEnumerable

## 개요

`IAsyncEnumerable<T>`는 **비동기 스트림**을 표현하는 인터페이스입니다. `await foreach`를 사용하여 비동기적으로 생성되는 데이터를 하나씩 처리할 수 있습니다. 대용량 데이터나 스트리밍 API에서 유용합니다.

---

## 기본 개념

### 동기 vs 비동기 열거

```csharp
// 동기 열거: 모든 데이터가 메모리에 있어야 함
IEnumerable<int> GetNumbers()
{
    yield return 1;
    yield return 2;
    yield return 3;
}

foreach (var num in GetNumbers())
{
    Debug.Log(num);
}

// 비동기 열거: 데이터를 비동기적으로 하나씩 가져옴
async IAsyncEnumerable<int> GetNumbersAsync()
{
    await Task.Delay(100);
    yield return 1;
    await Task.Delay(100);
    yield return 2;
    await Task.Delay(100);
    yield return 3;
}

await foreach (var num in GetNumbersAsync())
{
    Debug.Log(num);
}
```

### IAsyncEnumerable 인터페이스

```csharp
// 인터페이스 정의
public interface IAsyncEnumerable<out T>
{
    IAsyncEnumerator<T> GetAsyncEnumerator(CancellationToken cancellationToken = default);
}

public interface IAsyncEnumerator<out T> : IAsyncDisposable
{
    T Current { get; }
    ValueTask<bool> MoveNextAsync();
}
```

---

## 기본 사용법

### async iterator 메서드

```csharp
public class AsyncEnumerableBasics : MonoBehaviour
{
    async void Start()
    {
        // await foreach로 비동기 스트림 소비
        await foreach (var item in GenerateItemsAsync())
        {
            Debug.Log($"Received: {item}");
        }

        Debug.Log("스트림 완료");
    }

    // async iterator 메서드
    async IAsyncEnumerable<string> GenerateItemsAsync()
    {
        for (int i = 1; i <= 5; i++)
        {
            // 비동기 작업 (네트워크, 파일 등)
            await Task.Delay(500);

            // yield return으로 값 생성
            yield return $"Item {i}";
        }
    }
}
```

### CancellationToken 지원

```csharp
using System.Runtime.CompilerServices;

public class AsyncEnumerableCancellation : MonoBehaviour
{
    private CancellationTokenSource _cts;

    async void Start()
    {
        _cts = new CancellationTokenSource();

        try
        {
            // WithCancellation으로 취소 토큰 전달
            await foreach (var item in GenerateItemsAsync()
                .WithCancellation(_cts.Token))
            {
                Debug.Log($"Received: {item}");
            }
        }
        catch (OperationCanceledException)
        {
            Debug.Log("스트림 취소됨");
        }
    }

    void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }

    // EnumeratorCancellation 특성으로 취소 토큰 받기
    async IAsyncEnumerable<string> GenerateItemsAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        int i = 0;
        while (!cancellationToken.IsCancellationRequested)
        {
            await Task.Delay(500, cancellationToken);
            yield return $"Item {++i}";
        }
    }
}
```

---

## Unity에서의 활용

### 리소스 점진적 로딩

```csharp
public class ProgressiveResourceLoader : MonoBehaviour
{
    [SerializeField] private Slider _progressBar;
    [SerializeField] private Text _statusText;

    async void Start()
    {
        int loadedCount = 0;
        int totalCount = 10;

        await foreach (var resource in LoadResourcesAsync(totalCount))
        {
            loadedCount++;
            _progressBar.value = (float)loadedCount / totalCount;
            _statusText.text = $"Loaded: {resource.Name}";

            Debug.Log($"Resource loaded: {resource.Name}");
        }

        _statusText.text = "All resources loaded!";
    }

    async IAsyncEnumerable<Resource> LoadResourcesAsync(
        int count,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        var resourcePaths = Enumerable.Range(1, count)
            .Select(i => $"Prefabs/Resource_{i}");

        foreach (var path in resourcePaths)
        {
            cancellationToken.ThrowIfCancellationRequested();

            var request = Resources.LoadAsync<GameObject>(path);

            while (!request.isDone)
            {
                await Task.Yield();
            }

            yield return new Resource
            {
                Name = path,
                Asset = request.asset
            };
        }
    }
}

public class Resource
{
    public string Name { get; set; }
    public UnityEngine.Object Asset { get; set; }
}
```

### 서버 이벤트 스트리밍 (SSE)

```csharp
public class ServerSentEvents : MonoBehaviour
{
    [SerializeField] private Text _messageText;

    async void Start()
    {
        try
        {
            await foreach (var message in StreamServerEventsAsync(destroyCancellationToken))
            {
                _messageText.text = message;
                Debug.Log($"Server event: {message}");
            }
        }
        catch (OperationCanceledException)
        {
            Debug.Log("SSE 스트림 종료");
        }
    }

    async IAsyncEnumerable<string> StreamServerEventsAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        using var client = new HttpClient();
        client.Timeout = TimeSpan.FromMilliseconds(Timeout.Infinite);

        using var response = await client.GetAsync(
            "https://api.example.com/events",
            HttpCompletionOption.ResponseHeadersRead,
            cancellationToken);

        using var stream = await response.Content.ReadAsStreamAsync();
        using var reader = new StreamReader(stream);

        while (!reader.EndOfStream && !cancellationToken.IsCancellationRequested)
        {
            var line = await reader.ReadLineAsync();

            if (!string.IsNullOrEmpty(line) && line.StartsWith("data:"))
            {
                yield return line.Substring(5).Trim();
            }
        }
    }
}
```

### 페이지네이션 API 처리

```csharp
public class PaginatedApiClient : MonoBehaviour
{
    async void Start()
    {
        var allItems = new List<Item>();

        await foreach (var item in FetchAllItemsAsync())
        {
            allItems.Add(item);
            Debug.Log($"Fetched: {item.Name}");
        }

        Debug.Log($"Total items: {allItems.Count}");
    }

    async IAsyncEnumerable<Item> FetchAllItemsAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        int page = 1;
        bool hasMorePages = true;

        while (hasMorePages)
        {
            cancellationToken.ThrowIfCancellationRequested();

            var response = await FetchPageAsync(page, cancellationToken);

            foreach (var item in response.Items)
            {
                yield return item;
            }

            hasMorePages = response.HasNextPage;
            page++;
        }
    }

    async Task<PagedResponse> FetchPageAsync(int page, CancellationToken cancellationToken)
    {
        // 실제 API 호출 시뮬레이션
        await Task.Delay(500, cancellationToken);

        return new PagedResponse
        {
            Items = new List<Item>
            {
                new Item { Name = $"Item {page * 10 - 9}" },
                new Item { Name = $"Item {page * 10 - 8}" },
                new Item { Name = $"Item {page * 10 - 7}" }
            },
            HasNextPage = page < 5
        };
    }
}

public class Item
{
    public string Name { get; set; }
}

public class PagedResponse
{
    public List<Item> Items { get; set; }
    public bool HasNextPage { get; set; }
}
```

---

## LINQ 스타일 연산자

### System.Linq.Async 패키지

```csharp
// NuGet: System.Linq.Async 패키지 필요
using System.Linq;

public class AsyncLinqOperators : MonoBehaviour
{
    async void Start()
    {
        // Select
        await foreach (var doubled in NumbersAsync().SelectAwait(async x =>
        {
            await Task.Delay(10);
            return x * 2;
        }))
        {
            Debug.Log($"Doubled: {doubled}");
        }

        // Where
        await foreach (var even in NumbersAsync().Where(x => x % 2 == 0))
        {
            Debug.Log($"Even: {even}");
        }

        // Take
        await foreach (var first3 in NumbersAsync().Take(3))
        {
            Debug.Log($"First 3: {first3}");
        }

        // ToListAsync
        var list = await NumbersAsync().ToListAsync();
        Debug.Log($"Total count: {list.Count}");
    }

    async IAsyncEnumerable<int> NumbersAsync()
    {
        for (int i = 1; i <= 10; i++)
        {
            await Task.Delay(100);
            yield return i;
        }
    }
}
```

### 커스텀 연산자 구현

```csharp
public static class AsyncEnumerableExtensions
{
    // Batch: 여러 항목을 묶어서 처리
    public static async IAsyncEnumerable<List<T>> Batch<T>(
        this IAsyncEnumerable<T> source,
        int batchSize,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        var batch = new List<T>(batchSize);

        await foreach (var item in source.WithCancellation(cancellationToken))
        {
            batch.Add(item);

            if (batch.Count >= batchSize)
            {
                yield return batch;
                batch = new List<T>(batchSize);
            }
        }

        if (batch.Count > 0)
        {
            yield return batch;
        }
    }

    // Throttle: 일정 시간 간격으로만 값 방출
    public static async IAsyncEnumerable<T> Throttle<T>(
        this IAsyncEnumerable<T> source,
        TimeSpan interval,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        DateTime lastEmit = DateTime.MinValue;

        await foreach (var item in source.WithCancellation(cancellationToken))
        {
            var now = DateTime.UtcNow;
            if (now - lastEmit >= interval)
            {
                lastEmit = now;
                yield return item;
            }
        }
    }

    // Buffer with timeout: 시간 또는 개수로 버퍼링
    public static async IAsyncEnumerable<List<T>> BufferWithTimeout<T>(
        this IAsyncEnumerable<T> source,
        int maxCount,
        TimeSpan timeout,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        var buffer = new List<T>();
        var startTime = DateTime.UtcNow;

        await foreach (var item in source.WithCancellation(cancellationToken))
        {
            buffer.Add(item);

            if (buffer.Count >= maxCount || DateTime.UtcNow - startTime >= timeout)
            {
                yield return buffer;
                buffer = new List<T>();
                startTime = DateTime.UtcNow;
            }
        }

        if (buffer.Count > 0)
        {
            yield return buffer;
        }
    }
}

// 사용 예
public class CustomOperatorUsage : MonoBehaviour
{
    async void Start()
    {
        // Batch 사용
        await foreach (var batch in GetDataAsync().Batch(5))
        {
            Debug.Log($"Processing batch of {batch.Count} items");
        }

        // Throttle 사용
        await foreach (var item in GetFastDataAsync().Throttle(TimeSpan.FromSeconds(1)))
        {
            Debug.Log($"Throttled: {item}");
        }
    }

    async IAsyncEnumerable<int> GetDataAsync()
    {
        for (int i = 0; i < 23; i++)
        {
            yield return i;
        }
    }

    async IAsyncEnumerable<int> GetFastDataAsync()
    {
        for (int i = 0; i < 100; i++)
        {
            await Task.Delay(100);
            yield return i;
        }
    }
}
```

---

## 병렬 처리

### 제한된 병렬성으로 처리

```csharp
public class ParallelAsyncEnumerable : MonoBehaviour
{
    async void Start()
    {
        // 동시에 최대 3개씩 처리
        await foreach (var result in ProcessInParallelAsync(GetUrlsAsync(), 3))
        {
            Debug.Log($"Completed: {result}");
        }
    }

    async IAsyncEnumerable<string> ProcessInParallelAsync(
        IAsyncEnumerable<string> urls,
        int maxConcurrency,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        var semaphore = new SemaphoreSlim(maxConcurrency);
        var tasks = new List<Task<string>>();

        await foreach (var url in urls.WithCancellation(cancellationToken))
        {
            await semaphore.WaitAsync(cancellationToken);

            tasks.Add(Task.Run(async () =>
            {
                try
                {
                    return await ProcessUrlAsync(url, cancellationToken);
                }
                finally
                {
                    semaphore.Release();
                }
            }, cancellationToken));

            // 완료된 작업 yield
            while (tasks.Count > 0)
            {
                var completed = tasks.FirstOrDefault(t => t.IsCompleted);
                if (completed != null)
                {
                    tasks.Remove(completed);
                    yield return await completed;
                }
                else
                {
                    break;
                }
            }
        }

        // 남은 작업 완료 대기
        foreach (var task in tasks)
        {
            yield return await task;
        }
    }

    async IAsyncEnumerable<string> GetUrlsAsync()
    {
        for (int i = 0; i < 10; i++)
        {
            yield return $"https://api.example.com/item/{i}";
        }
    }

    async Task<string> ProcessUrlAsync(string url, CancellationToken cancellationToken)
    {
        await Task.Delay(Random.Range(100, 500), cancellationToken);
        return $"Processed: {url}";
    }
}
```

---

## 실전 패턴

### 패턴 1: 게임 이벤트 스트림

```csharp
public class GameEventStream : MonoBehaviour
{
    private Channel<GameEvent> _eventChannel;

    void Awake()
    {
        _eventChannel = Channel.CreateUnbounded<GameEvent>();
    }

    // 이벤트 발행
    public void PublishEvent(GameEvent gameEvent)
    {
        _eventChannel.Writer.TryWrite(gameEvent);
    }

    // 이벤트 스트림 구독
    public async IAsyncEnumerable<GameEvent> GetEventStreamAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (var evt in _eventChannel.Reader.ReadAllAsync(cancellationToken))
        {
            yield return evt;
        }
    }

    // 사용 예
    async void Start()
    {
        // 이벤트 핸들러
        _ = HandleEventsAsync(destroyCancellationToken);

        // 테스트 이벤트 발행
        PublishEvent(new GameEvent { Type = "PlayerSpawn", Data = "Player1" });
    }

    async Task HandleEventsAsync(CancellationToken cancellationToken)
    {
        await foreach (var evt in GetEventStreamAsync(cancellationToken))
        {
            switch (evt.Type)
            {
                case "PlayerSpawn":
                    Debug.Log($"Player spawned: {evt.Data}");
                    break;
                case "EnemyKilled":
                    Debug.Log($"Enemy killed: {evt.Data}");
                    break;
            }
        }
    }

    void OnDestroy()
    {
        _eventChannel.Writer.Complete();
    }
}

public class GameEvent
{
    public string Type { get; set; }
    public string Data { get; set; }
}
```

### 패턴 2: 파일 스트리밍

```csharp
public class FileStreamProcessor : MonoBehaviour
{
    async void Start()
    {
        string filePath = Path.Combine(Application.persistentDataPath, "large_data.txt");

        // 파일을 라인 단위로 비동기 처리
        await foreach (var line in ReadLinesAsync(filePath, destroyCancellationToken))
        {
            ProcessLine(line);
        }
    }

    async IAsyncEnumerable<string> ReadLinesAsync(
        string filePath,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        using var reader = new StreamReader(filePath);

        while (!reader.EndOfStream)
        {
            cancellationToken.ThrowIfCancellationRequested();

            var line = await reader.ReadLineAsync();
            if (line != null)
            {
                yield return line;
            }
        }
    }

    void ProcessLine(string line)
    {
        // 라인 처리 로직
        Debug.Log($"Processing: {line}");
    }
}
```

### 패턴 3: 실시간 데이터 처리

```csharp
public class RealtimeDataProcessor : MonoBehaviour
{
    [SerializeField] private LineRenderer _chartLine;

    private List<Vector3> _dataPoints = new List<Vector3>();
    private float _xPosition = 0f;

    async void Start()
    {
        // 실시간 데이터 스트림 처리
        await foreach (var value in GetSensorDataAsync(destroyCancellationToken))
        {
            AddDataPoint(value);
        }
    }

    async IAsyncEnumerable<float> GetSensorDataAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            // 센서 데이터 시뮬레이션
            await Task.Delay(100, cancellationToken);

            float noise = Mathf.PerlinNoise(Time.time, 0f);
            float value = Mathf.Sin(Time.time * 2f) + noise * 0.3f;

            yield return value;
        }
    }

    void AddDataPoint(float value)
    {
        _xPosition += 0.1f;
        _dataPoints.Add(new Vector3(_xPosition, value, 0f));

        // 최대 100개 포인트 유지
        if (_dataPoints.Count > 100)
        {
            _dataPoints.RemoveAt(0);
            // X 좌표 재조정
            for (int i = 0; i < _dataPoints.Count; i++)
            {
                _dataPoints[i] = new Vector3(i * 0.1f, _dataPoints[i].y, 0f);
            }
            _xPosition = _dataPoints.Count * 0.1f;
        }

        UpdateChart();
    }

    void UpdateChart()
    {
        _chartLine.positionCount = _dataPoints.Count;
        _chartLine.SetPositions(_dataPoints.ToArray());
    }
}
```

### 패턴 4: 재시도가 포함된 데이터 페칭

```csharp
public class RetryableDataFetcher : MonoBehaviour
{
    async void Start()
    {
        await foreach (var data in FetchWithRetryAsync(
            GetDataUrlsAsync(),
            maxRetries: 3,
            cancellationToken: destroyCancellationToken))
        {
            Debug.Log($"Fetched: {data}");
        }
    }

    async IAsyncEnumerable<string> FetchWithRetryAsync(
        IAsyncEnumerable<string> urls,
        int maxRetries,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (var url in urls.WithCancellation(cancellationToken))
        {
            int retries = 0;
            while (retries <= maxRetries)
            {
                try
                {
                    var data = await FetchDataAsync(url, cancellationToken);
                    yield return data;
                    break;
                }
                catch (HttpRequestException) when (retries < maxRetries)
                {
                    retries++;
                    Debug.LogWarning($"Retry {retries}/{maxRetries} for {url}");
                    await Task.Delay(1000 * retries, cancellationToken);
                }
            }
        }
    }

    async IAsyncEnumerable<string> GetDataUrlsAsync()
    {
        for (int i = 0; i < 10; i++)
        {
            yield return $"https://api.example.com/data/{i}";
        }
    }

    async Task<string> FetchDataAsync(string url, CancellationToken cancellationToken)
    {
        using var client = new HttpClient();
        return await client.GetStringAsync(url, cancellationToken);
    }
}
```

---

## UniTask와 함께 사용

### UniTask의 IAsyncEnumerable 지원

```csharp
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Linq;

public class UniTaskAsyncEnumerable : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // UniTask 스타일 async enumerable
        await foreach (var item in UniTaskAsyncEnumerable
            .EveryUpdate()
            .Take(100)
            .Where(_ => Input.GetMouseButton(0))
            .Select(_ => Input.mousePosition)
            .WithCancellation(destroyCancellationToken))
        {
            Debug.Log($"Mouse position while clicked: {item}");
        }
    }
}
```

### UniTask EveryUpdate 패턴

```csharp
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Linq;

public class EveryUpdatePattern : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // 매 프레임 실행
        await foreach (var _ in UniTaskAsyncEnumerable
            .EveryUpdate()
            .WithCancellation(destroyCancellationToken))
        {
            // Update 대체
            transform.Rotate(Vector3.up * Time.deltaTime * 30f);
        }
    }
}
```

---

## 주의사항 및 베스트 프랙티스

### 주의사항

```csharp
public class AsyncEnumerablePrecautions : MonoBehaviour
{
    // ❌ 무한 스트림에서 ToList 호출
    async void Bad_InfiniteToList()
    {
        // 이 코드는 영원히 실행됨!
        // var list = await InfiniteStreamAsync().ToListAsync();
    }

    // ✅ Take로 제한
    async void Good_LimitedCollection()
    {
        var list = await InfiniteStreamAsync()
            .Take(100)
            .ToListAsync(destroyCancellationToken);
    }

    // ❌ 취소 토큰 없이 무한 스트림
    async void Bad_NoCancellation()
    {
        // 오브젝트 파괴 후에도 계속 실행될 수 있음
        // await foreach (var item in InfiniteStreamAsync()) { }
    }

    // ✅ 취소 토큰 사용
    async void Good_WithCancellation()
    {
        await foreach (var item in InfiniteStreamAsync()
            .WithCancellation(destroyCancellationToken))
        {
            // 오브젝트 파괴 시 자동 취소
        }
    }

    async IAsyncEnumerable<int> InfiniteStreamAsync(
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        int i = 0;
        while (!ct.IsCancellationRequested)
        {
            await Task.Delay(100, ct);
            yield return i++;
        }
    }
}
```

### 메모리 관리

```csharp
public class MemoryManagement : MonoBehaviour
{
    // ✅ 스트리밍 처리로 메모리 효율적 사용
    async void ProcessLargeDataEfficiently()
    {
        long totalSize = 0;

        // 한 번에 하나씩 처리 - 메모리 효율적
        await foreach (var chunk in ReadLargeFileAsync())
        {
            totalSize += chunk.Length;
            ProcessChunk(chunk);
            // chunk는 다음 반복에서 GC 대상
        }

        Debug.Log($"Processed {totalSize} bytes");
    }

    // ❌ 전체 로드 - 메모리 비효율적
    async void ProcessLargeDataInefficiently()
    {
        // 모든 데이터를 메모리에 로드
        // var allData = await ReadLargeFileAsync().ToListAsync();
    }

    async IAsyncEnumerable<byte[]> ReadLargeFileAsync()
    {
        // 청크 단위로 읽기
        yield return new byte[4096];
    }

    void ProcessChunk(byte[] chunk) { }
}
```

---

## 정리

### IAsyncEnumerable 요약

| 항목 | 내용 |
|------|------|
| **용도** | 비동기 데이터 스트림 처리 |
| **문법** | `async IAsyncEnumerable<T>`, `yield return`, `await foreach` |
| **취소** | `[EnumeratorCancellation]` 특성, `WithCancellation()` |
| **LINQ** | System.Linq.Async 패키지로 LINQ 연산자 사용 |
| **장점** | 메모리 효율적, 점진적 처리, 자연스러운 비동기 |

### 사용 시기

| 시나리오 | 권장 |
|----------|------|
| 페이지네이션 API | ✅ IAsyncEnumerable |
| 파일 스트리밍 | ✅ IAsyncEnumerable |
| 실시간 데이터 | ✅ IAsyncEnumerable |
| 단일 비동기 작업 | Task/ValueTask |
| 모든 데이터가 필요 | ToListAsync() 후 처리 |

---

## 참고 자료

- [Async Streams in C# 8.0](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-8#asynchronous-streams)
- [IAsyncEnumerable<T> Interface](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1)
- [System.Linq.Async Package](https://www.nuget.org/packages/System.Linq.Async/)
- [UniTask AsyncEnumerable](https://github.com/Cysharp/UniTask#asyncenumerable-and-async-linq)
