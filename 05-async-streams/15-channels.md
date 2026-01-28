# 15. Channel & Pipeline

## 개요

`System.Threading.Channels`는 **생산자-소비자 패턴**을 위한 고성능 비동기 데이터 통신 채널입니다. Unity에서 스레드 간 안전한 데이터 전달, 작업 큐, 이벤트 스트리밍 등에 활용됩니다.

---

## Channel 기본 개념

### 생산자-소비자 패턴

```
┌──────────────┐    ┌─────────┐    ┌──────────────┐
│   Producer   │ -> │ Channel │ -> │   Consumer   │
│   (Writer)   │    │ (Queue) │    │   (Reader)   │
└──────────────┘    └─────────┘    └──────────────┘
```

### Channel 종류

| 타입 | 설명 | 사용 시기 |
|------|------|-----------|
| Unbounded | 무제한 버퍼 | 생산 속도가 일정하지 않을 때 |
| Bounded | 제한된 버퍼 | 메모리 제한, 배압(Backpressure) 필요 |
| SingleReader | 단일 소비자 최적화 | 소비자가 1개일 때 |
| SingleWriter | 단일 생산자 최적화 | 생산자가 1개일 때 |

---

## 기본 사용법

### Unbounded Channel

```csharp
using System.Threading.Channels;

public class UnboundedChannelExample : MonoBehaviour
{
    private Channel<string> _channel;

    async void Start()
    {
        // Unbounded Channel 생성
        _channel = Channel.CreateUnbounded<string>();

        // 생산자와 소비자 동시 실행
        var producerTask = ProduceAsync();
        var consumerTask = ConsumeAsync();

        await Task.WhenAll(producerTask, consumerTask);
    }

    async Task ProduceAsync()
    {
        var writer = _channel.Writer;

        for (int i = 0; i < 10; i++)
        {
            await writer.WriteAsync($"Message {i}");
            Debug.Log($"Produced: Message {i}");
            await Task.Delay(100);
        }

        // 생산 완료 신호
        writer.Complete();
    }

    async Task ConsumeAsync()
    {
        var reader = _channel.Reader;

        // 채널이 완료될 때까지 읽기
        await foreach (var message in reader.ReadAllAsync())
        {
            Debug.Log($"Consumed: {message}");
            await Task.Delay(200); // 처리 시뮬레이션
        }
    }
}
```

### Bounded Channel

```csharp
public class BoundedChannelExample : MonoBehaviour
{
    private Channel<WorkItem> _workQueue;

    async void Start()
    {
        // Bounded Channel: 최대 100개 항목
        _workQueue = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait, // 꽉 차면 대기
            SingleReader = false,
            SingleWriter = false
        });

        // 여러 생산자
        var producers = Enumerable.Range(0, 3)
            .Select(i => ProduceWorkAsync(i))
            .ToArray();

        // 여러 소비자
        var consumers = Enumerable.Range(0, 2)
            .Select(i => ProcessWorkAsync(i))
            .ToArray();

        // 생산 완료 대기
        await Task.WhenAll(producers);
        _workQueue.Writer.Complete();

        // 소비 완료 대기
        await Task.WhenAll(consumers);
    }

    async Task ProduceWorkAsync(int producerId)
    {
        for (int i = 0; i < 50; i++)
        {
            var work = new WorkItem { Id = $"P{producerId}-{i}", Data = $"Data {i}" };
            await _workQueue.Writer.WriteAsync(work);
            Debug.Log($"Producer {producerId}: Created {work.Id}");
        }
    }

    async Task ProcessWorkAsync(int consumerId)
    {
        await foreach (var work in _workQueue.Reader.ReadAllAsync())
        {
            Debug.Log($"Consumer {consumerId}: Processing {work.Id}");
            await Task.Delay(50); // 처리 시뮬레이션
        }
    }
}

public class WorkItem
{
    public string Id { get; set; }
    public string Data { get; set; }
}
```

---

## BoundedChannelFullMode 옵션

### 채널이 가득 찼을 때의 동작

```csharp
public class FullModeExamples : MonoBehaviour
{
    void CreateChannelsWithDifferentModes()
    {
        // Wait: 공간이 생길 때까지 대기 (기본값)
        var waitChannel = Channel.CreateBounded<int>(new BoundedChannelOptions(10)
        {
            FullMode = BoundedChannelFullMode.Wait
        });

        // DropNewest: 새 항목 버림
        var dropNewestChannel = Channel.CreateBounded<int>(new BoundedChannelOptions(10)
        {
            FullMode = BoundedChannelFullMode.DropNewest
        });

        // DropOldest: 가장 오래된 항목 버림
        var dropOldestChannel = Channel.CreateBounded<int>(new BoundedChannelOptions(10)
        {
            FullMode = BoundedChannelFullMode.DropOldest
        });

        // DropWrite: 쓰기 즉시 실패 (버림)
        var dropWriteChannel = Channel.CreateBounded<int>(new BoundedChannelOptions(10)
        {
            FullMode = BoundedChannelFullMode.DropWrite
        });
    }
}
```

### 모드별 사용 예

```csharp
public class FullModeUseCases : MonoBehaviour
{
    // 로깅: 오래된 로그 버리기
    private Channel<LogEntry> _logChannel;

    // 최신 가격만 중요: 새 데이터가 오면 이전 것 버리기
    private Channel<PriceUpdate> _priceChannel;

    void Awake()
    {
        // 로그 채널: 가장 오래된 것 버림
        _logChannel = Channel.CreateBounded<LogEntry>(new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.DropOldest
        });

        // 가격 채널: 새 데이터가 오면 기존 것 버림
        _priceChannel = Channel.CreateBounded<PriceUpdate>(new BoundedChannelOptions(1)
        {
            FullMode = BoundedChannelFullMode.DropOldest
        });
    }
}

public class LogEntry { }
public class PriceUpdate { }
```

---

## Unity에서의 활용

### 메인 스레드로 작업 전달

```csharp
public class MainThreadDispatcher : MonoBehaviour
{
    private static MainThreadDispatcher _instance;
    private Channel<Action> _actionChannel;

    public static MainThreadDispatcher Instance => _instance;

    void Awake()
    {
        _instance = this;
        _actionChannel = Channel.CreateUnbounded<Action>(new UnboundedChannelOptions
        {
            SingleReader = true
        });
    }

    void Update()
    {
        // 매 프레임 큐에 있는 작업 처리
        while (_actionChannel.Reader.TryRead(out var action))
        {
            try
            {
                action?.Invoke();
            }
            catch (Exception ex)
            {
                Debug.LogException(ex);
            }
        }
    }

    // 백그라운드 스레드에서 호출
    public void Dispatch(Action action)
    {
        _actionChannel.Writer.TryWrite(action);
    }

    // 비동기 버전
    public async Task DispatchAsync(Action action)
    {
        await _actionChannel.Writer.WriteAsync(action);
    }
}

// 사용 예
public class BackgroundWorker : MonoBehaviour
{
    async void Start()
    {
        await Task.Run(async () =>
        {
            // 백그라운드에서 계산
            var result = HeavyCalculation();

            // 메인 스레드에서 UI 업데이트
            MainThreadDispatcher.Instance.Dispatch(() =>
            {
                Debug.Log($"Result: {result}");
                transform.position = new Vector3(result, 0, 0);
            });
        });
    }

    float HeavyCalculation()
    {
        // 무거운 계산
        return 42f;
    }
}
```

### 게임 이벤트 시스템

```csharp
public class EventChannel<T>
{
    private readonly Channel<T> _channel;
    private readonly List<Func<T, UniTask>> _handlers = new List<Func<T, UniTask>>();
    private CancellationTokenSource _cts;

    public EventChannel()
    {
        _channel = Channel.CreateUnbounded<T>();
    }

    public void Subscribe(Func<T, UniTask> handler)
    {
        _handlers.Add(handler);
    }

    public void Unsubscribe(Func<T, UniTask> handler)
    {
        _handlers.Remove(handler);
    }

    public void Publish(T eventData)
    {
        _channel.Writer.TryWrite(eventData);
    }

    public async UniTask StartProcessingAsync(CancellationToken cancellationToken)
    {
        await foreach (var eventData in _channel.Reader.ReadAllAsync(cancellationToken))
        {
            foreach (var handler in _handlers.ToArray()) // 복사본으로 순회
            {
                try
                {
                    await handler(eventData);
                }
                catch (Exception ex)
                {
                    Debug.LogException(ex);
                }
            }
        }
    }

    public void Complete()
    {
        _channel.Writer.Complete();
    }
}

// 사용 예
public class GameEventSystem : MonoBehaviour
{
    public static EventChannel<PlayerDamageEvent> OnPlayerDamage = new EventChannel<PlayerDamageEvent>();
    public static EventChannel<ItemPickupEvent> OnItemPickup = new EventChannel<ItemPickupEvent>();

    async void Start()
    {
        // 이벤트 처리 시작
        _ = OnPlayerDamage.StartProcessingAsync(destroyCancellationToken);
        _ = OnItemPickup.StartProcessingAsync(destroyCancellationToken);
    }

    void OnDestroy()
    {
        OnPlayerDamage.Complete();
        OnItemPickup.Complete();
    }
}

public class PlayerDamageEvent
{
    public int PlayerId { get; set; }
    public float Damage { get; set; }
}

public class ItemPickupEvent
{
    public int PlayerId { get; set; }
    public string ItemId { get; set; }
}
```

### 작업 큐 시스템

```csharp
public class WorkQueue<T> : IDisposable
{
    private readonly Channel<T> _channel;
    private readonly Func<T, CancellationToken, Task> _processor;
    private readonly int _workerCount;
    private CancellationTokenSource _cts;
    private Task[] _workers;

    public WorkQueue(
        Func<T, CancellationToken, Task> processor,
        int workerCount = 4,
        int maxQueueSize = 1000)
    {
        _processor = processor;
        _workerCount = workerCount;

        _channel = Channel.CreateBounded<T>(new BoundedChannelOptions(maxQueueSize)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = workerCount == 1
        });
    }

    public void Start()
    {
        _cts = new CancellationTokenSource();

        _workers = Enumerable.Range(0, _workerCount)
            .Select(_ => WorkerLoopAsync(_cts.Token))
            .ToArray();
    }

    public async Task EnqueueAsync(T item, CancellationToken cancellationToken = default)
    {
        await _channel.Writer.WriteAsync(item, cancellationToken);
    }

    public bool TryEnqueue(T item)
    {
        return _channel.Writer.TryWrite(item);
    }

    private async Task WorkerLoopAsync(CancellationToken cancellationToken)
    {
        await foreach (var item in _channel.Reader.ReadAllAsync(cancellationToken))
        {
            try
            {
                await _processor(item, cancellationToken);
            }
            catch (OperationCanceledException)
            {
                break;
            }
            catch (Exception ex)
            {
                Debug.LogException(ex);
            }
        }
    }

    public async Task StopAsync()
    {
        _channel.Writer.Complete();

        if (_workers != null)
        {
            await Task.WhenAll(_workers);
        }
    }

    public void Dispose()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }
}

// 사용 예
public class DownloadManager : MonoBehaviour
{
    private WorkQueue<DownloadTask> _downloadQueue;

    void Awake()
    {
        _downloadQueue = new WorkQueue<DownloadTask>(
            ProcessDownloadAsync,
            workerCount: 3,
            maxQueueSize: 100);

        _downloadQueue.Start();
    }

    async void Start()
    {
        // 다운로드 작업 추가
        for (int i = 0; i < 20; i++)
        {
            await _downloadQueue.EnqueueAsync(new DownloadTask
            {
                Url = $"https://example.com/file{i}.dat",
                DestPath = Path.Combine(Application.persistentDataPath, $"file{i}.dat")
            });
        }
    }

    async Task ProcessDownloadAsync(DownloadTask task, CancellationToken cancellationToken)
    {
        Debug.Log($"Downloading: {task.Url}");
        await Task.Delay(1000, cancellationToken); // 다운로드 시뮬레이션
        Debug.Log($"Completed: {task.Url}");
    }

    async void OnDestroy()
    {
        await _downloadQueue.StopAsync();
        _downloadQueue.Dispose();
    }
}

public class DownloadTask
{
    public string Url { get; set; }
    public string DestPath { get; set; }
}
```

---

## Pipeline 패턴

### 단계별 처리 파이프라인

```csharp
public class ProcessingPipeline : MonoBehaviour
{
    private Channel<RawData> _rawDataChannel;
    private Channel<ProcessedData> _processedChannel;
    private Channel<FinalResult> _resultChannel;

    async void Start()
    {
        // 파이프라인 채널 생성
        _rawDataChannel = Channel.CreateBounded<RawData>(100);
        _processedChannel = Channel.CreateBounded<ProcessedData>(100);
        _resultChannel = Channel.CreateBounded<FinalResult>(100);

        // 파이프라인 단계 시작
        var stage1 = ProcessStage1Async(destroyCancellationToken);
        var stage2 = ProcessStage2Async(destroyCancellationToken);
        var consumer = ConsumeResultsAsync(destroyCancellationToken);

        // 데이터 생산
        await ProduceDataAsync();

        // 완료 신호 전파
        _rawDataChannel.Writer.Complete();
        await stage1;

        _processedChannel.Writer.Complete();
        await stage2;

        _resultChannel.Writer.Complete();
        await consumer;

        Debug.Log("Pipeline completed");
    }

    async Task ProduceDataAsync()
    {
        for (int i = 0; i < 100; i++)
        {
            await _rawDataChannel.Writer.WriteAsync(new RawData { Id = i, Value = i * 10 });
        }
    }

    async Task ProcessStage1Async(CancellationToken cancellationToken)
    {
        await foreach (var raw in _rawDataChannel.Reader.ReadAllAsync(cancellationToken))
        {
            // 1단계: 필터링 및 변환
            if (raw.Value > 50)
            {
                var processed = new ProcessedData
                {
                    Id = raw.Id,
                    TransformedValue = raw.Value * 2
                };
                await _processedChannel.Writer.WriteAsync(processed, cancellationToken);
            }
        }
    }

    async Task ProcessStage2Async(CancellationToken cancellationToken)
    {
        await foreach (var processed in _processedChannel.Reader.ReadAllAsync(cancellationToken))
        {
            // 2단계: 추가 처리
            var result = new FinalResult
            {
                Id = processed.Id,
                Score = processed.TransformedValue / 10f
            };
            await _resultChannel.Writer.WriteAsync(result, cancellationToken);
        }
    }

    async Task ConsumeResultsAsync(CancellationToken cancellationToken)
    {
        await foreach (var result in _resultChannel.Reader.ReadAllAsync(cancellationToken))
        {
            Debug.Log($"Final Result: Id={result.Id}, Score={result.Score}");
        }
    }
}

public class RawData { public int Id; public int Value; }
public class ProcessedData { public int Id; public float TransformedValue; }
public class FinalResult { public int Id; public float Score; }
```

### 분기(Fan-out) / 병합(Fan-in) 패턴

```csharp
public class FanOutFanIn : MonoBehaviour
{
    async void Start()
    {
        // 입력 채널
        var inputChannel = Channel.CreateUnbounded<int>();

        // 작업자별 채널 (Fan-out)
        var workerChannels = Enumerable.Range(0, 4)
            .Select(_ => Channel.CreateUnbounded<int>())
            .ToArray();

        // 결과 채널 (Fan-in)
        var resultChannel = Channel.CreateUnbounded<int>();

        // Fan-out: 입력을 작업자들에게 분배
        var distributor = DistributeAsync(inputChannel.Reader, workerChannels.Select(c => c.Writer).ToArray());

        // 작업자들 시작
        var workers = workerChannels.Select((c, i) =>
            ProcessWorkerAsync(i, c.Reader, resultChannel.Writer, destroyCancellationToken))
            .ToArray();

        // 결과 수집 시작
        var collector = CollectResultsAsync(resultChannel.Reader, destroyCancellationToken);

        // 입력 데이터 생성
        for (int i = 0; i < 100; i++)
        {
            await inputChannel.Writer.WriteAsync(i);
        }
        inputChannel.Writer.Complete();

        // 분배 완료 대기
        await distributor;

        // 작업자 채널 완료
        foreach (var wc in workerChannels)
        {
            wc.Writer.Complete();
        }

        // 작업자 완료 대기
        await Task.WhenAll(workers);

        // 결과 채널 완료
        resultChannel.Writer.Complete();

        // 수집 완료 대기
        await collector;
    }

    async Task DistributeAsync(ChannelReader<int> reader, ChannelWriter<int>[] writers)
    {
        int index = 0;
        await foreach (var item in reader.ReadAllAsync())
        {
            // 라운드 로빈 분배
            await writers[index % writers.Length].WriteAsync(item);
            index++;
        }
    }

    async Task ProcessWorkerAsync(
        int workerId,
        ChannelReader<int> input,
        ChannelWriter<int> output,
        CancellationToken cancellationToken)
    {
        await foreach (var item in input.ReadAllAsync(cancellationToken))
        {
            // 처리
            var result = item * 2;
            Debug.Log($"Worker {workerId}: {item} -> {result}");
            await output.WriteAsync(result, cancellationToken);
        }
    }

    async Task CollectResultsAsync(ChannelReader<int> reader, CancellationToken cancellationToken)
    {
        var results = new List<int>();

        await foreach (var result in reader.ReadAllAsync(cancellationToken))
        {
            results.Add(result);
        }

        Debug.Log($"Total results: {results.Count}");
    }
}
```

---

## UniTask Channel 통합

### UniTask와 함께 사용

```csharp
using Cysharp.Threading.Tasks;

public class UniTaskChannelIntegration : MonoBehaviour
{
    private Channel<string> _messageChannel;

    async UniTaskVoid Start()
    {
        _messageChannel = Channel.CreateUnbounded<string>();

        // 생산자
        _ = ProduceMessagesAsync(destroyCancellationToken);

        // 소비자 (UniTask)
        await ConsumeMessagesAsync(destroyCancellationToken);
    }

    async UniTask ProduceMessagesAsync(CancellationToken cancellationToken)
    {
        for (int i = 0; i < 10; i++)
        {
            await _messageChannel.Writer.WriteAsync($"Message {i}", cancellationToken);
            await UniTask.Delay(500, cancellationToken: cancellationToken);
        }
        _messageChannel.Writer.Complete();
    }

    async UniTask ConsumeMessagesAsync(CancellationToken cancellationToken)
    {
        await foreach (var message in _messageChannel.Reader.ReadAllAsync(cancellationToken))
        {
            Debug.Log($"Received: {message}");
            // Main Thread에서 UI 업데이트 가능
        }
    }
}
```

### AsyncReactiveProperty와 Channel 조합

```csharp
using Cysharp.Threading.Tasks;

public class ChannelWithReactiveProperty : MonoBehaviour
{
    private Channel<int> _scoreChannel;
    private readonly AsyncReactiveProperty<int> _totalScore = new AsyncReactiveProperty<int>(0);

    public IReadOnlyAsyncReactiveProperty<int> TotalScore => _totalScore;

    async UniTaskVoid Start()
    {
        _scoreChannel = Channel.CreateUnbounded<int>();

        // 점수 처리
        _ = ProcessScoresAsync(destroyCancellationToken);

        // 점수 변경 감지
        _totalScore.Subscribe(score =>
        {
            Debug.Log($"Total score updated: {score}");
        });

        // 테스트: 점수 추가
        await AddScoreAsync(10);
        await AddScoreAsync(20);
        await AddScoreAsync(30);
    }

    public async UniTask AddScoreAsync(int score)
    {
        await _scoreChannel.Writer.WriteAsync(score);
    }

    async UniTask ProcessScoresAsync(CancellationToken cancellationToken)
    {
        await foreach (var score in _scoreChannel.Reader.ReadAllAsync(cancellationToken))
        {
            _totalScore.Value += score;
        }
    }
}
```

---

## 실전 패턴

### 패턴 1: 요청-응답 채널

```csharp
public class RequestResponseChannel<TRequest, TResponse>
{
    private readonly Channel<(TRequest Request, TaskCompletionSource<TResponse> Tcs)> _channel;

    public RequestResponseChannel()
    {
        _channel = Channel.CreateUnbounded<(TRequest, TaskCompletionSource<TResponse>)>();
    }

    public async Task<TResponse> SendRequestAsync(
        TRequest request,
        CancellationToken cancellationToken = default)
    {
        var tcs = new TaskCompletionSource<TResponse>();

        using (cancellationToken.Register(() => tcs.TrySetCanceled()))
        {
            await _channel.Writer.WriteAsync((request, tcs), cancellationToken);
            return await tcs.Task;
        }
    }

    public async Task StartProcessingAsync(
        Func<TRequest, Task<TResponse>> handler,
        CancellationToken cancellationToken)
    {
        await foreach (var (request, tcs) in _channel.Reader.ReadAllAsync(cancellationToken))
        {
            try
            {
                var response = await handler(request);
                tcs.TrySetResult(response);
            }
            catch (Exception ex)
            {
                tcs.TrySetException(ex);
            }
        }
    }

    public void Complete()
    {
        _channel.Writer.Complete();
    }
}

// 사용 예
public class ApiService : MonoBehaviour
{
    private RequestResponseChannel<ApiRequest, ApiResponse> _apiChannel;

    async void Start()
    {
        _apiChannel = new RequestResponseChannel<ApiRequest, ApiResponse>();

        // 요청 처리 시작
        _ = _apiChannel.StartProcessingAsync(HandleApiRequestAsync, destroyCancellationToken);

        // 요청 보내기
        var response = await _apiChannel.SendRequestAsync(
            new ApiRequest { Endpoint = "/users", Method = "GET" });

        Debug.Log($"Response: {response.Data}");
    }

    async Task<ApiResponse> HandleApiRequestAsync(ApiRequest request)
    {
        await Task.Delay(500); // API 호출 시뮬레이션
        return new ApiResponse { Data = $"Response for {request.Endpoint}" };
    }

    void OnDestroy()
    {
        _apiChannel.Complete();
    }
}

public class ApiRequest { public string Endpoint; public string Method; }
public class ApiResponse { public string Data; }
```

### 패턴 2: 배치 처리

```csharp
public class BatchProcessor<T>
{
    private readonly Channel<T> _channel;
    private readonly int _batchSize;
    private readonly TimeSpan _batchTimeout;
    private readonly Func<List<T>, Task> _processBatch;

    public BatchProcessor(
        int batchSize,
        TimeSpan batchTimeout,
        Func<List<T>, Task> processBatch)
    {
        _batchSize = batchSize;
        _batchTimeout = batchTimeout;
        _processBatch = processBatch;
        _channel = Channel.CreateUnbounded<T>();
    }

    public async Task AddAsync(T item, CancellationToken cancellationToken = default)
    {
        await _channel.Writer.WriteAsync(item, cancellationToken);
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        var batch = new List<T>(_batchSize);
        var timer = Task.Delay(_batchTimeout, cancellationToken);

        while (!cancellationToken.IsCancellationRequested)
        {
            var readTask = _channel.Reader.ReadAsync(cancellationToken).AsTask();

            var completed = await Task.WhenAny(readTask, timer);

            if (completed == readTask && !readTask.IsCanceled)
            {
                batch.Add(await readTask);

                if (batch.Count >= _batchSize)
                {
                    await ProcessBatchAsync(batch);
                    batch.Clear();
                    timer = Task.Delay(_batchTimeout, cancellationToken);
                }
            }
            else // 타이머 완료
            {
                if (batch.Count > 0)
                {
                    await ProcessBatchAsync(batch);
                    batch.Clear();
                }
                timer = Task.Delay(_batchTimeout, cancellationToken);
            }
        }

        // 남은 항목 처리
        if (batch.Count > 0)
        {
            await ProcessBatchAsync(batch);
        }
    }

    private async Task ProcessBatchAsync(List<T> batch)
    {
        try
        {
            await _processBatch(new List<T>(batch));
        }
        catch (Exception ex)
        {
            Debug.LogException(ex);
        }
    }

    public void Complete()
    {
        _channel.Writer.Complete();
    }
}

// 사용 예
public class TelemetryService : MonoBehaviour
{
    private BatchProcessor<TelemetryEvent> _batchProcessor;

    void Awake()
    {
        _batchProcessor = new BatchProcessor<TelemetryEvent>(
            batchSize: 100,
            batchTimeout: TimeSpan.FromSeconds(5),
            SendTelemetryBatchAsync);

        _ = _batchProcessor.StartAsync(destroyCancellationToken);
    }

    public async Task LogEventAsync(string eventName, Dictionary<string, object> properties)
    {
        await _batchProcessor.AddAsync(new TelemetryEvent
        {
            Name = eventName,
            Properties = properties,
            Timestamp = DateTime.UtcNow
        });
    }

    async Task SendTelemetryBatchAsync(List<TelemetryEvent> batch)
    {
        Debug.Log($"Sending telemetry batch: {batch.Count} events");
        await Task.Delay(100); // 전송 시뮬레이션
    }

    void OnDestroy()
    {
        _batchProcessor.Complete();
    }
}

public class TelemetryEvent
{
    public string Name { get; set; }
    public Dictionary<string, object> Properties { get; set; }
    public DateTime Timestamp { get; set; }
}
```

### 패턴 3: 우선순위 큐

```csharp
public class PriorityChannel<T>
{
    private readonly SortedDictionary<int, Channel<T>> _priorityChannels;
    private readonly SemaphoreSlim _semaphore = new SemaphoreSlim(0);

    public PriorityChannel(int priorityLevels = 3)
    {
        _priorityChannels = new SortedDictionary<int, Channel<T>>();

        for (int i = 0; i < priorityLevels; i++)
        {
            _priorityChannels[i] = Channel.CreateUnbounded<T>();
        }
    }

    public async Task WriteAsync(T item, int priority, CancellationToken cancellationToken = default)
    {
        if (!_priorityChannels.ContainsKey(priority))
        {
            priority = _priorityChannels.Keys.Max();
        }

        await _priorityChannels[priority].Writer.WriteAsync(item, cancellationToken);
        _semaphore.Release();
    }

    public async Task<T> ReadAsync(CancellationToken cancellationToken = default)
    {
        await _semaphore.WaitAsync(cancellationToken);

        // 높은 우선순위부터 확인
        foreach (var kvp in _priorityChannels)
        {
            if (kvp.Value.Reader.TryRead(out var item))
            {
                return item;
            }
        }

        throw new InvalidOperationException("No item available");
    }
}

// 사용 예
public class TaskScheduler : MonoBehaviour
{
    private PriorityChannel<GameTask> _taskQueue;

    void Awake()
    {
        _taskQueue = new PriorityChannel<GameTask>(3); // 0: 높음, 1: 중간, 2: 낮음
    }

    async void Start()
    {
        // 작업 추가
        await _taskQueue.WriteAsync(new GameTask { Name = "Low priority" }, 2);
        await _taskQueue.WriteAsync(new GameTask { Name = "High priority" }, 0);
        await _taskQueue.WriteAsync(new GameTask { Name = "Medium priority" }, 1);

        // 작업 처리 (우선순위 순)
        for (int i = 0; i < 3; i++)
        {
            var task = await _taskQueue.ReadAsync(destroyCancellationToken);
            Debug.Log($"Processing: {task.Name}");
        }
    }
}

public class GameTask { public string Name; }
```

---

## 주의사항

### 메모리 누수 방지

```csharp
public class ChannelMemoryManagement : MonoBehaviour
{
    private Channel<byte[]> _dataChannel;

    void Awake()
    {
        // Bounded로 메모리 제한
        _dataChannel = Channel.CreateBounded<byte[]>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.DropOldest // 오래된 데이터 자동 삭제
        });
    }

    void OnDestroy()
    {
        // 반드시 Complete 호출
        _dataChannel.Writer.Complete();
    }
}
```

### 교착 상태 방지

```csharp
public class DeadlockPrevention : MonoBehaviour
{
    // ❌ 동기 대기 - 교착 가능
    void Bad_SyncWait()
    {
        var channel = Channel.CreateBounded<int>(1);
        channel.Writer.WriteAsync(1).AsTask().Wait();
        channel.Writer.WriteAsync(2).AsTask().Wait(); // 교착!
    }

    // ✅ 비동기 사용
    async Task Good_AsyncWrite()
    {
        var channel = Channel.CreateBounded<int>(1);
        await channel.Writer.WriteAsync(1);
        // 소비자가 읽을 때까지 대기
        await channel.Writer.WriteAsync(2);
    }
}
```

---

## 정리

### Channel 요약

| 항목 | 내용 |
|------|------|
| **용도** | 생산자-소비자 패턴, 스레드 간 통신 |
| **종류** | Unbounded, Bounded |
| **특징** | Thread-safe, async 지원, 배압 처리 |
| **사용 시기** | 작업 큐, 이벤트 스트림, 파이프라인 |

### 선택 가이드

| 요구사항 | 권장 |
|----------|------|
| 무제한 버퍼 | Channel.CreateUnbounded |
| 메모리 제한 필요 | Channel.CreateBounded |
| 최신 데이터만 필요 | DropOldest 모드 |
| 단일 소비자 | SingleReader = true |

---

## 참고 자료

- [System.Threading.Channels](https://docs.microsoft.com/en-us/dotnet/api/system.threading.channels)
- [An Introduction to Channels](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/)
- [Producer-Consumer Pattern](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/how-to-implement-a-producer-consumer-dataflow-pattern)
- [UniTask with Channels](https://github.com/Cysharp/UniTask)
