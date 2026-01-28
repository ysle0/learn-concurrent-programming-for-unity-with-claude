# 20. Concurrent Collections

## 개요

`System.Collections.Concurrent` 네임스페이스는 **스레드 안전한 컬렉션**을 제공합니다. 멀티스레드 환경에서 락 없이도 안전하게 데이터를 추가, 제거, 조회할 수 있습니다.

---

## 왜 Concurrent Collections인가?

### 일반 컬렉션의 문제

```csharp
public class ThreadUnsafeExample : MonoBehaviour
{
    void Start()
    {
        var list = new List<int>();

        // ❌ 위험: 여러 스레드에서 동시 접근
        Parallel.For(0, 10000, i =>
        {
            list.Add(i); // 런타임 에러 또는 데이터 손상!
        });

        // 결과: 예외 발생 또는 일부 데이터 누락
        Debug.Log($"Count: {list.Count}"); // 10000이 아닐 수 있음
    }
}
```

### 수동 락의 문제

```csharp
public class ManualLockProblem : MonoBehaviour
{
    void Start()
    {
        var list = new List<int>();
        var lockObj = new object();

        // 동작하지만 성능 저하
        Parallel.For(0, 10000, i =>
        {
            lock (lockObj)
            {
                list.Add(i); // 한 번에 하나의 스레드만 접근
            }
        });

        // 병렬의 이점이 거의 없음
    }
}
```

### Concurrent Collections 해결책

```csharp
using System.Collections.Concurrent;

public class ConcurrentSolution : MonoBehaviour
{
    void Start()
    {
        var bag = new ConcurrentBag<int>();

        // ✅ 안전하고 효율적
        Parallel.For(0, 10000, i =>
        {
            bag.Add(i); // 내부적으로 최적화된 동기화
        });

        Debug.Log($"Count: {bag.Count}"); // 정확히 10000
    }
}
```

---

## ConcurrentDictionary<TKey, TValue>

### 기본 사용법

```csharp
using System.Collections.Concurrent;

public class ConcurrentDictionaryBasics : MonoBehaviour
{
    private ConcurrentDictionary<string, int> _playerScores =
        new ConcurrentDictionary<string, int>();

    void Start()
    {
        // 추가
        _playerScores.TryAdd("Player1", 100);
        _playerScores.TryAdd("Player2", 200);

        // 또는 인덱서 사용 (덮어쓰기)
        _playerScores["Player3"] = 300;

        // 조회
        if (_playerScores.TryGetValue("Player1", out int score))
        {
            Debug.Log($"Player1 score: {score}");
        }

        // 업데이트
        _playerScores.TryUpdate("Player1", 150, 100); // oldValue가 100일 때만 업데이트

        // 제거
        _playerScores.TryRemove("Player2", out _);

        // 모든 항목
        foreach (var kvp in _playerScores)
        {
            Debug.Log($"{kvp.Key}: {kvp.Value}");
        }
    }
}
```

### GetOrAdd / AddOrUpdate

```csharp
public class ConcurrentDictionaryAdvanced : MonoBehaviour
{
    private ConcurrentDictionary<string, int> _cache =
        new ConcurrentDictionary<string, int>();

    void Start()
    {
        // GetOrAdd: 없으면 추가하고 반환
        int value1 = _cache.GetOrAdd("key1", key =>
        {
            // 이 팩토리는 키가 없을 때만 호출됨
            return ExpensiveCalculation(key);
        });

        // AddOrUpdate: 있으면 업데이트, 없으면 추가
        int value2 = _cache.AddOrUpdate(
            "score",
            key => 10,                    // 추가 시 값
            (key, oldValue) => oldValue + 10  // 업데이트 시 값
        );

        Debug.Log($"Value1: {value1}, Value2: {value2}");
    }

    int ExpensiveCalculation(string key)
    {
        Thread.Sleep(100); // 비싼 계산 시뮬레이션
        return key.GetHashCode();
    }
}
```

### Unity에서의 활용: 오브젝트 캐시

```csharp
public class ObjectCache : MonoBehaviour
{
    private ConcurrentDictionary<string, GameObject> _prefabCache =
        new ConcurrentDictionary<string, GameObject>();

    // 스레드 안전한 프리팹 캐시
    public GameObject GetPrefab(string path)
    {
        return _prefabCache.GetOrAdd(path, p =>
        {
            // 주의: Resources.Load는 Main Thread 필요
            // 실제로는 로딩을 Main Thread에서 해야 함
            return Resources.Load<GameObject>(p);
        });
    }

    // 스레드 안전한 데이터 캐시 (더 적합한 예)
    private ConcurrentDictionary<int, EnemyData> _enemyDataCache =
        new ConcurrentDictionary<int, EnemyData>();

    public EnemyData GetEnemyData(int id)
    {
        return _enemyDataCache.GetOrAdd(id, LoadEnemyData);
    }

    private EnemyData LoadEnemyData(int id)
    {
        // JSON 파싱 등 스레드 안전한 작업
        return new EnemyData { Id = id, Name = $"Enemy_{id}" };
    }
}

public class EnemyData
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

---

## ConcurrentQueue<T>

### 기본 사용법

```csharp
public class ConcurrentQueueBasics : MonoBehaviour
{
    private ConcurrentQueue<string> _messageQueue = new ConcurrentQueue<string>();

    void Start()
    {
        // 여러 스레드에서 추가
        Parallel.For(0, 100, i =>
        {
            _messageQueue.Enqueue($"Message {i}");
        });

        Debug.Log($"Queue count: {_messageQueue.Count}");

        // 하나씩 처리
        while (_messageQueue.TryDequeue(out string message))
        {
            Debug.Log($"Processing: {message}");
        }
    }
}
```

### 작업 큐 패턴

```csharp
public class WorkQueue : MonoBehaviour
{
    private ConcurrentQueue<Action> _workQueue = new ConcurrentQueue<Action>();
    private bool _isRunning = true;

    void Start()
    {
        // 작업자 스레드 시작
        Task.Run(ProcessWorkQueue);

        // 작업 추가
        for (int i = 0; i < 10; i++)
        {
            int taskId = i;
            EnqueueWork(() => ProcessTask(taskId));
        }
    }

    public void EnqueueWork(Action work)
    {
        _workQueue.Enqueue(work);
    }

    async Task ProcessWorkQueue()
    {
        while (_isRunning)
        {
            if (_workQueue.TryDequeue(out Action work))
            {
                try
                {
                    work?.Invoke();
                }
                catch (Exception ex)
                {
                    Debug.LogError($"Work failed: {ex.Message}");
                }
            }
            else
            {
                await Task.Delay(10); // 큐가 비었으면 대기
            }
        }
    }

    void ProcessTask(int id)
    {
        Debug.Log($"Processing task {id} on thread {Thread.CurrentThread.ManagedThreadId}");
        Thread.Sleep(100);
    }

    void OnDestroy()
    {
        _isRunning = false;
    }
}
```

### Main Thread Dispatcher

```csharp
public class MainThreadDispatcher : MonoBehaviour
{
    private static MainThreadDispatcher _instance;
    private ConcurrentQueue<Action> _mainThreadQueue = new ConcurrentQueue<Action>();

    public static MainThreadDispatcher Instance => _instance;

    void Awake()
    {
        _instance = this;
    }

    void Update()
    {
        // 매 프레임 큐의 작업 처리
        int processedCount = 0;
        while (_mainThreadQueue.TryDequeue(out Action action) && processedCount < 100)
        {
            try
            {
                action?.Invoke();
            }
            catch (Exception ex)
            {
                Debug.LogException(ex);
            }
            processedCount++;
        }
    }

    public void Enqueue(Action action)
    {
        _mainThreadQueue.Enqueue(action);
    }

    public Task EnqueueAsync(Action action)
    {
        var tcs = new TaskCompletionSource<bool>();
        _mainThreadQueue.Enqueue(() =>
        {
            try
            {
                action();
                tcs.SetResult(true);
            }
            catch (Exception ex)
            {
                tcs.SetException(ex);
            }
        });
        return tcs.Task;
    }
}

// 사용 예
public class BackgroundWorker : MonoBehaviour
{
    async void Start()
    {
        // 백그라운드에서 계산
        await Task.Run(() =>
        {
            var result = HeavyCalculation();

            // Main Thread에서 UI 업데이트
            MainThreadDispatcher.Instance.Enqueue(() =>
            {
                Debug.Log($"Result: {result}");
                transform.position = new Vector3(result, 0, 0);
            });
        });
    }

    float HeavyCalculation()
    {
        Thread.Sleep(1000);
        return 42f;
    }
}
```

---

## ConcurrentStack<T>

### 기본 사용법

```csharp
public class ConcurrentStackBasics : MonoBehaviour
{
    private ConcurrentStack<int> _stack = new ConcurrentStack<int>();

    void Start()
    {
        // 추가 (LIFO)
        _stack.Push(1);
        _stack.Push(2);
        _stack.Push(3);

        // 여러 개 한번에 추가
        _stack.PushRange(new[] { 4, 5, 6 });

        // 꺼내기
        if (_stack.TryPop(out int value))
        {
            Debug.Log($"Popped: {value}"); // 6
        }

        // 확인만 (제거 안 함)
        if (_stack.TryPeek(out int top))
        {
            Debug.Log($"Top: {top}"); // 5
        }

        // 여러 개 한번에 꺼내기
        int[] items = new int[3];
        int count = _stack.TryPopRange(items);
        Debug.Log($"Popped {count} items: {string.Join(", ", items.Take(count))}");
    }
}
```

### Undo 시스템

```csharp
public class UndoSystem : MonoBehaviour
{
    private ConcurrentStack<ICommand> _undoStack = new ConcurrentStack<ICommand>();
    private ConcurrentStack<ICommand> _redoStack = new ConcurrentStack<ICommand>();

    public void ExecuteCommand(ICommand command)
    {
        command.Execute();
        _undoStack.Push(command);
        _redoStack.Clear(); // 새 명령 실행 시 redo 스택 클리어
    }

    public void Undo()
    {
        if (_undoStack.TryPop(out ICommand command))
        {
            command.Undo();
            _redoStack.Push(command);
        }
    }

    public void Redo()
    {
        if (_redoStack.TryPop(out ICommand command))
        {
            command.Execute();
            _undoStack.Push(command);
        }
    }
}

public interface ICommand
{
    void Execute();
    void Undo();
}

public class MoveCommand : ICommand
{
    private Transform _target;
    private Vector3 _oldPosition;
    private Vector3 _newPosition;

    public MoveCommand(Transform target, Vector3 newPosition)
    {
        _target = target;
        _oldPosition = target.position;
        _newPosition = newPosition;
    }

    public void Execute() => _target.position = _newPosition;
    public void Undo() => _target.position = _oldPosition;
}
```

---

## ConcurrentBag<T>

### 기본 사용법

```csharp
public class ConcurrentBagBasics : MonoBehaviour
{
    private ConcurrentBag<int> _bag = new ConcurrentBag<int>();

    void Start()
    {
        // 순서 보장 없는 컬렉션 (가장 빠름)
        Parallel.For(0, 1000, i =>
        {
            _bag.Add(i);
        });

        Debug.Log($"Bag count: {_bag.Count}");

        // 꺼내기 (순서 보장 안 됨)
        if (_bag.TryTake(out int value))
        {
            Debug.Log($"Took: {value}");
        }

        // 확인만
        if (_bag.TryPeek(out int peeked))
        {
            Debug.Log($"Peeked: {peeked}");
        }
    }
}
```

### 결과 수집

```csharp
public class ResultCollection : MonoBehaviour
{
    void Start()
    {
        var results = new ConcurrentBag<ProcessResult>();

        // 병렬로 처리하고 결과 수집
        Parallel.For(0, 100, i =>
        {
            var result = ProcessItem(i);
            results.Add(result);
        });

        // 결과 정리
        var sortedResults = results.OrderBy(r => r.Index).ToList();
        Debug.Log($"Collected {sortedResults.Count} results");
    }

    ProcessResult ProcessItem(int index)
    {
        return new ProcessResult
        {
            Index = index,
            Value = index * 2,
            ProcessedAt = DateTime.Now
        };
    }
}

public class ProcessResult
{
    public int Index { get; set; }
    public int Value { get; set; }
    public DateTime ProcessedAt { get; set; }
}
```

---

## BlockingCollection<T>

### 기본 사용법

```csharp
using System.Collections.Concurrent;

public class BlockingCollectionBasics : MonoBehaviour
{
    private BlockingCollection<int> _collection;

    void Start()
    {
        // 기본 ConcurrentQueue 래핑
        _collection = new BlockingCollection<int>();

        // 용량 제한
        var bounded = new BlockingCollection<int>(boundedCapacity: 100);

        // 다른 컬렉션 래핑
        var stackBased = new BlockingCollection<int>(new ConcurrentStack<int>());

        StartProducerConsumer();
    }

    async void StartProducerConsumer()
    {
        // 생산자
        var producer = Task.Run(() =>
        {
            for (int i = 0; i < 10; i++)
            {
                _collection.Add(i);
                Debug.Log($"Produced: {i}");
                Thread.Sleep(100);
            }
            _collection.CompleteAdding(); // 생산 완료
        });

        // 소비자
        var consumer = Task.Run(() =>
        {
            // GetConsumingEnumerable: 블로킹 열거
            foreach (var item in _collection.GetConsumingEnumerable())
            {
                Debug.Log($"Consumed: {item}");
                Thread.Sleep(200);
            }
        });

        await Task.WhenAll(producer, consumer);
        Debug.Log("Producer-Consumer completed");
    }

    void OnDestroy()
    {
        _collection?.Dispose();
    }
}
```

### 생산자-소비자 패턴

```csharp
public class ProducerConsumer : MonoBehaviour
{
    private BlockingCollection<WorkItem> _workItems;
    private CancellationTokenSource _cts;

    void Awake()
    {
        _workItems = new BlockingCollection<WorkItem>(boundedCapacity: 100);
        _cts = new CancellationTokenSource();
    }

    void Start()
    {
        // 여러 소비자 시작
        for (int i = 0; i < 3; i++)
        {
            int consumerId = i;
            Task.Run(() => ConsumeWork(consumerId, _cts.Token));
        }

        // 생산자
        Task.Run(() => ProduceWork(_cts.Token));
    }

    async Task ProduceWork(CancellationToken cancellationToken)
    {
        int itemId = 0;
        while (!cancellationToken.IsCancellationRequested)
        {
            try
            {
                var item = new WorkItem { Id = itemId++, Data = $"Work_{itemId}" };
                _workItems.Add(item, cancellationToken);
                Debug.Log($"Produced: {item.Id}");
                await Task.Delay(50, cancellationToken);
            }
            catch (OperationCanceledException)
            {
                break;
            }
        }
    }

    void ConsumeWork(int consumerId, CancellationToken cancellationToken)
    {
        try
        {
            foreach (var item in _workItems.GetConsumingEnumerable(cancellationToken))
            {
                Debug.Log($"Consumer {consumerId} processing: {item.Id}");
                Thread.Sleep(100); // 처리 시뮬레이션
            }
        }
        catch (OperationCanceledException)
        {
            Debug.Log($"Consumer {consumerId} cancelled");
        }
    }

    void OnDestroy()
    {
        _workItems.CompleteAdding();
        _cts.Cancel();
        _cts.Dispose();
        _workItems.Dispose();
    }
}

public class WorkItem
{
    public int Id { get; set; }
    public string Data { get; set; }
}
```

### 타임아웃과 취소

```csharp
public class BlockingCollectionTimeout : MonoBehaviour
{
    private BlockingCollection<int> _collection = new BlockingCollection<int>();
    private CancellationTokenSource _cts = new CancellationTokenSource();

    void Start()
    {
        // 타임아웃과 함께 추가
        bool added = _collection.TryAdd(1, TimeSpan.FromSeconds(5));
        Debug.Log($"Added with timeout: {added}");

        // 타임아웃과 함께 가져오기
        bool taken = _collection.TryTake(out int item, TimeSpan.FromSeconds(5));
        Debug.Log($"Taken: {taken}, Item: {item}");

        // 취소 토큰과 함께
        try
        {
            _collection.Add(2, _cts.Token);
            var value = _collection.Take(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("Operation cancelled");
        }
    }

    void OnDestroy()
    {
        _cts.Cancel();
        _cts.Dispose();
        _collection.Dispose();
    }
}
```

---

## 컬렉션 선택 가이드

### 비교표

| 컬렉션 | 순서 | 용도 | 특징 |
|--------|------|------|------|
| **ConcurrentDictionary** | 없음 | 키-값 저장 | GetOrAdd, AddOrUpdate |
| **ConcurrentQueue** | FIFO | 작업 큐 | Enqueue/TryDequeue |
| **ConcurrentStack** | LIFO | Undo, 재귀 | Push/TryPop |
| **ConcurrentBag** | 없음 | 결과 수집 | 가장 빠름 |
| **BlockingCollection** | 래핑 | 생산자-소비자 | 블로킹 지원 |

### 선택 플로우차트

```
키-값 필요?
├── Yes → ConcurrentDictionary
└── No → 순서 필요?
           ├── FIFO → ConcurrentQueue
           ├── LIFO → ConcurrentStack
           └── No → 블로킹 필요?
                      ├── Yes → BlockingCollection
                      └── No → ConcurrentBag
```

---

## 성능 팁

### 캐시 활용

```csharp
public class ConcurrentCacheOptimization : MonoBehaviour
{
    // Lazy 초기화와 결합
    private static readonly ConcurrentDictionary<string, Lazy<ExpensiveObject>> _cache =
        new ConcurrentDictionary<string, Lazy<ExpensiveObject>>();

    public ExpensiveObject GetOrCreate(string key)
    {
        // 동시 요청 시 한 번만 생성
        var lazy = _cache.GetOrAdd(key, k =>
            new Lazy<ExpensiveObject>(() => new ExpensiveObject(k)));
        return lazy.Value;
    }
}

public class ExpensiveObject
{
    public ExpensiveObject(string key)
    {
        Debug.Log($"Creating expensive object for {key}");
        Thread.Sleep(100);
    }
}
```

### 불필요한 할당 방지

```csharp
public class AvoidAllocations : MonoBehaviour
{
    private ConcurrentDictionary<int, string> _dict = new ConcurrentDictionary<int, string>();

    void Start()
    {
        // ❌ 매번 새 람다 생성
        for (int i = 0; i < 1000; i++)
        {
            int key = i;
            _dict.GetOrAdd(key, k => $"Value_{k}"); // 람다 할당
        }

        // ✅ 정적 메서드 사용
        for (int i = 0; i < 1000; i++)
        {
            _dict.GetOrAdd(i, CreateValue);
        }
    }

    private static string CreateValue(int key) => $"Value_{key}";
}
```

---

## 정리

### Concurrent Collections 요약

| 컬렉션 | 대체 대상 | 주요 메서드 |
|--------|----------|-------------|
| ConcurrentDictionary | Dictionary | GetOrAdd, AddOrUpdate |
| ConcurrentQueue | Queue | Enqueue, TryDequeue |
| ConcurrentStack | Stack | Push, TryPop |
| ConcurrentBag | List | Add, TryTake |
| BlockingCollection | - | Add, Take, GetConsumingEnumerable |

### 체크리스트

- [ ] 멀티스레드 환경인가?
- [ ] 적절한 컬렉션 타입 선택했는가?
- [ ] TryXxx 메서드로 안전하게 접근하는가?
- [ ] 리소스 해제 (Dispose) 확인했는가?

---

## 참고 자료

- [Thread-Safe Collections](https://docs.microsoft.com/en-us/dotnet/standard/collections/thread-safe/)
- [ConcurrentDictionary Best Practices](https://docs.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2)
- [BlockingCollection Overview](https://docs.microsoft.com/en-us/dotnet/standard/collections/thread-safe/blockingcollection-overview)
