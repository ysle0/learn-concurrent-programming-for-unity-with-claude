# 11. ValueTask

## 개요

`ValueTask<T>`는 .NET Core 2.1에서 도입된 **allocation-free 비동기 반환 타입**입니다. 동기적으로 완료되는 경우가 많은 비동기 메서드에서 힙 할당을 줄여 성능을 개선합니다. Unity에서는 특히 **GC 최적화**가 중요하기 때문에 적절한 사용이 권장됩니다.

---

## Task vs ValueTask

### Task의 힙 할당 문제

```csharp
public class TaskAllocationExample : MonoBehaviour
{
    private Dictionary<string, byte[]> _cache = new Dictionary<string, byte[]>();

    // Task는 항상 힙에 할당됨
    public async Task<byte[]> GetDataAsync(string key)
    {
        // 캐시에 있으면 즉시 반환 - 그런데도 Task 객체 생성!
        if (_cache.TryGetValue(key, out var data))
        {
            return data; // Task.FromResult 호출 (힙 할당)
        }

        // 실제 비동기 작업
        data = await LoadFromServerAsync(key);
        _cache[key] = data;
        return data;
    }

    // 90%가 캐시 히트라면 90%의 호출에서 불필요한 Task 할당!
}
```

### ValueTask로 할당 제거

```csharp
public class ValueTaskExample : MonoBehaviour
{
    private Dictionary<string, byte[]> _cache = new Dictionary<string, byte[]>();

    // ValueTask는 값 타입 - 스택에 할당
    public ValueTask<byte[]> GetDataAsync(string key)
    {
        // 캐시 히트: 힙 할당 없음!
        if (_cache.TryGetValue(key, out var data))
        {
            return new ValueTask<byte[]>(data); // 스택 할당만
        }

        // 캐시 미스: 실제 비동기 작업으로 전환
        return new ValueTask<byte[]>(LoadFromServerCoreAsync(key));
    }

    private async Task<byte[]> LoadFromServerCoreAsync(string key)
    {
        var data = await LoadFromServerAsync(key);
        _cache[key] = data;
        return data;
    }
}
```

### 성능 비교

| 시나리오 | Task | ValueTask |
|----------|------|-----------|
| 동기 완료 (캐시 히트) | ~56 bytes 힙 할당 | 0 bytes (스택) |
| 비동기 완료 | ~72 bytes 힙 할당 | ~72 bytes 힙 할당 |
| GC 부담 | 높음 | 낮음 |

---

## ValueTask 기본 사용법

### 생성 방법

```csharp
public class ValueTaskCreation : MonoBehaviour
{
    // 1. 동기 값으로 생성
    public ValueTask<int> GetCachedValue()
    {
        return new ValueTask<int>(42); // 즉시 완료, 힙 할당 없음
    }

    // 2. Task로부터 생성
    public ValueTask<string> GetDataAsync()
    {
        Task<string> task = FetchFromServerAsync();
        return new ValueTask<string>(task);
    }

    // 3. async 메서드의 반환 타입으로
    public async ValueTask<byte[]> LoadAsync()
    {
        await Task.Delay(100);
        return new byte[1024];
    }

    // 4. 제네릭이 아닌 ValueTask (반환값 없음)
    public ValueTask ProcessAsync()
    {
        if (IsAlreadyProcessed())
        {
            return ValueTask.CompletedTask; // 즉시 완료
        }

        return new ValueTask(ProcessCoreAsync());
    }

    private bool IsAlreadyProcessed() => false;
    private Task ProcessCoreAsync() => Task.CompletedTask;
    private Task<string> FetchFromServerAsync() => Task.FromResult("data");
}
```

### await 사용

```csharp
public class ValueTaskAwait : MonoBehaviour
{
    async void Start()
    {
        // ValueTask도 await 가능
        var result = await GetValueAsync();
        Debug.Log($"Result: {result}");
    }

    public ValueTask<int> GetValueAsync()
    {
        return new ValueTask<int>(42);
    }
}
```

---

## ValueTask 사용 규칙 (중요!)

### 규칙 1: 한 번만 await 가능

```csharp
public class ValueTaskRules : MonoBehaviour
{
    async void Start()
    {
        var valueTask = GetDataAsync();

        // ✅ 첫 번째 await
        var result1 = await valueTask;

        // ❌ 두 번째 await - 정의되지 않은 동작!
        // var result2 = await valueTask; // 위험!
    }

    // Task는 여러 번 await 가능
    async void TaskCanAwaitMultipleTimes()
    {
        var task = Task.FromResult(42);

        var result1 = await task;
        var result2 = await task; // ✅ Task는 괜찮음
    }

    public ValueTask<int> GetDataAsync() => new ValueTask<int>(42);
}
```

### 규칙 2: 병렬 대기 금지

```csharp
public class ValueTaskNoParallel : MonoBehaviour
{
    async void Start()
    {
        // ❌ 잘못된 사용 - 같은 ValueTask를 병렬로 대기
        var vt = GetDataAsync();
        // await Task.WhenAll(vt.AsTask(), vt.AsTask()); // 위험!

        // ✅ 올바른 사용 - 각각 별도의 ValueTask 생성
        var vt1 = GetDataAsync();
        var vt2 = GetDataAsync();
        await Task.WhenAll(vt1.AsTask(), vt2.AsTask());
    }

    public ValueTask<int> GetDataAsync() => new ValueTask<int>(42);
}
```

### 규칙 3: 즉시 await 하거나 AsTask() 호출

```csharp
public class ValueTaskImmediateAwait : MonoBehaviour
{
    async void Start()
    {
        // ✅ 패턴 1: 즉시 await
        var result = await GetDataAsync();

        // ✅ 패턴 2: 저장이 필요하면 AsTask() 사용
        var task = GetDataAsync().AsTask();
        // 이제 task는 여러 번 await 가능
        await task;

        // ❌ ValueTask를 필드에 저장하지 말 것
        // _cachedValueTask = GetDataAsync(); // 위험!
    }

    public ValueTask<int> GetDataAsync() => new ValueTask<int>(42);
}
```

---

## 언제 ValueTask를 사용해야 하는가?

### 사용해야 하는 경우

```csharp
public class WhenToUseValueTask : MonoBehaviour
{
    private Dictionary<int, Enemy> _enemyCache = new Dictionary<int, Enemy>();

    // ✅ 1. 자주 동기적으로 완료되는 메서드
    public ValueTask<Enemy> GetEnemyAsync(int id)
    {
        // 대부분 캐시 히트 (80%+)
        if (_enemyCache.TryGetValue(id, out var enemy))
        {
            return new ValueTask<Enemy>(enemy); // 힙 할당 없음
        }

        return new ValueTask<Enemy>(LoadEnemyAsync(id));
    }

    // ✅ 2. 핫 패스(Hot Path)에서 호출되는 메서드
    public ValueTask<bool> CheckConditionAsync()
    {
        // 매 프레임 호출 - 할당 최소화 중요
        if (IsConditionMet())
        {
            return new ValueTask<bool>(true);
        }

        return new ValueTask<bool>(CheckConditionCoreAsync());
    }

    // ✅ 3. 인터페이스 구현에서 대부분 동기 완료
    public ValueTask InitializeAsync()
    {
        if (IsInitialized)
        {
            return ValueTask.CompletedTask;
        }

        return new ValueTask(InitializeCoreAsync());
    }

    private bool IsConditionMet() => true;
    private Task<bool> CheckConditionCoreAsync() => Task.FromResult(true);
    private Task<Enemy> LoadEnemyAsync(int id) => Task.FromResult(new Enemy());
    private Task InitializeCoreAsync() => Task.CompletedTask;
    private bool IsInitialized => false;
}

public class Enemy { }
```

### 사용하지 말아야 하는 경우

```csharp
public class WhenNotToUseValueTask : MonoBehaviour
{
    // ❌ 1. 항상 비동기로 실행되는 메서드
    public async Task<byte[]> DownloadFileAsync(string url)
    {
        // 네트워크 호출은 항상 비동기
        using var client = new HttpClient();
        return await client.GetByteArrayAsync(url);
    }

    // ❌ 2. 결과를 저장하거나 여러 곳에서 사용해야 하는 경우
    public Task<GameConfig> LoadConfigAsync()
    {
        // 결과를 캐시하고 공유해야 함
        return _configTask ??= LoadConfigCoreAsync();
    }
    private Task<GameConfig> _configTask;

    // ❌ 3. Task.WhenAll/WhenAny와 함께 사용
    public async Task LoadAllAsync()
    {
        // Task가 필요한 API
        await Task.WhenAll(
            LoadEnemiesAsync(),
            LoadItemsAsync(),
            LoadMapsAsync()
        );
    }

    private Task LoadEnemiesAsync() => Task.CompletedTask;
    private Task LoadItemsAsync() => Task.CompletedTask;
    private Task LoadMapsAsync() => Task.CompletedTask;
    private Task<GameConfig> LoadConfigCoreAsync() => Task.FromResult(new GameConfig());
}

public class GameConfig { }
```

### 결정 플로우차트

```
메서드가 자주 동기적으로 완료되는가?
    ├── No → Task 사용
    └── Yes → 핫 패스인가? (매 프레임/고빈도 호출)
                ├── No → Task 사용 (단순성 우선)
                └── Yes → 결과를 저장/공유해야 하는가?
                            ├── Yes → Task 사용
                            └── No → ValueTask 사용
```

---

## Unity에서의 ValueTask 활용

### 오브젝트 풀과 ValueTask

```csharp
public class ObjectPoolWithValueTask : MonoBehaviour
{
    private Queue<GameObject> _pool = new Queue<GameObject>();
    private GameObject _prefab;
    private Transform _parent;

    public void Initialize(GameObject prefab, int initialSize)
    {
        _prefab = prefab;
        _parent = transform;

        for (int i = 0; i < initialSize; i++)
        {
            var obj = CreateNew();
            obj.SetActive(false);
            _pool.Enqueue(obj);
        }
    }

    // 대부분 풀에서 즉시 반환 - ValueTask 적합
    public ValueTask<GameObject> GetAsync()
    {
        if (_pool.Count > 0)
        {
            var obj = _pool.Dequeue();
            obj.SetActive(true);
            return new ValueTask<GameObject>(obj); // 힙 할당 없음
        }

        // 풀이 비었으면 비동기로 확장
        return new ValueTask<GameObject>(ExpandPoolAndGetAsync());
    }

    private async Task<GameObject> ExpandPoolAndGetAsync()
    {
        // 프레임 스파이크 방지를 위해 분산 생성
        for (int i = 0; i < 10; i++)
        {
            var obj = CreateNew();
            obj.SetActive(false);
            _pool.Enqueue(obj);
            await Task.Yield();
        }

        return _pool.Dequeue();
    }

    public void Return(GameObject obj)
    {
        obj.SetActive(false);
        _pool.Enqueue(obj);
    }

    private GameObject CreateNew()
    {
        return Instantiate(_prefab, _parent);
    }
}
```

### 캐시된 데이터 접근

```csharp
public class CachedDataAccess : MonoBehaviour
{
    private Dictionary<string, Sprite> _spriteCache = new Dictionary<string, Sprite>();

    // 대부분 캐시 히트 - ValueTask 적합
    public ValueTask<Sprite> GetSpriteAsync(string spriteName)
    {
        if (_spriteCache.TryGetValue(spriteName, out var sprite))
        {
            return new ValueTask<Sprite>(sprite);
        }

        return new ValueTask<Sprite>(LoadSpriteAsync(spriteName));
    }

    private async Task<Sprite> LoadSpriteAsync(string spriteName)
    {
        var path = $"Sprites/{spriteName}";
        var request = Resources.LoadAsync<Sprite>(path);

        while (!request.isDone)
        {
            await Task.Yield();
        }

        var sprite = request.asset as Sprite;
        _spriteCache[spriteName] = sprite;
        return sprite;
    }
}
```

### 상태 체크 메서드

```csharp
public class StateCheckMethods : MonoBehaviour
{
    private bool _isReady;
    private TaskCompletionSource<bool> _readyTcs;

    public void Initialize()
    {
        _readyTcs = new TaskCompletionSource<bool>();
        StartCoroutine(PrepareResources());
    }

    // 대부분 즉시 true/false 반환
    public ValueTask<bool> IsReadyAsync()
    {
        if (_isReady)
        {
            return new ValueTask<bool>(true);
        }

        if (_readyTcs == null)
        {
            return new ValueTask<bool>(false);
        }

        return new ValueTask<bool>(_readyTcs.Task);
    }

    private IEnumerator PrepareResources()
    {
        yield return new WaitForSeconds(2f);
        _isReady = true;
        _readyTcs?.SetResult(true);
    }
}
```

---

## IValueTaskSource로 커스텀 구현

### IValueTaskSource 이해

```csharp
// IValueTaskSource<T>: ValueTask의 실제 비동기 작업을 제공하는 인터페이스
public interface IValueTaskSource<out T>
{
    // 현재 상태 반환
    ValueTaskSourceStatus GetStatus(short token);

    // 완료 시 콜백 등록
    void OnCompleted(
        Action<object> continuation,
        object state,
        short token,
        ValueTaskSourceOnCompletedFlags flags);

    // 결과 가져오기
    T GetResult(short token);
}
```

### 재사용 가능한 ValueTaskSource

```csharp
using System.Threading.Tasks.Sources;

// 재사용 가능한 ValueTaskSource 구현
public class PooledValueTaskSource<T> : IValueTaskSource<T>
{
    private ManualResetValueTaskSourceCore<T> _core;
    private static readonly Stack<PooledValueTaskSource<T>> _pool = new Stack<PooledValueTaskSource<T>>();

    public static PooledValueTaskSource<T> Rent()
    {
        lock (_pool)
        {
            return _pool.Count > 0 ? _pool.Pop() : new PooledValueTaskSource<T>();
        }
    }

    public ValueTask<T> Task => new ValueTask<T>(this, _core.Version);

    public void SetResult(T result)
    {
        _core.SetResult(result);
    }

    public void SetException(Exception error)
    {
        _core.SetException(error);
    }

    public T GetResult(short token)
    {
        try
        {
            return _core.GetResult(token);
        }
        finally
        {
            _core.Reset();
            ReturnToPool();
        }
    }

    public ValueTaskSourceStatus GetStatus(short token)
    {
        return _core.GetStatus(token);
    }

    public void OnCompleted(
        Action<object> continuation,
        object state,
        short token,
        ValueTaskSourceOnCompletedFlags flags)
    {
        _core.OnCompleted(continuation, state, token, flags);
    }

    private void ReturnToPool()
    {
        lock (_pool)
        {
            _pool.Push(this);
        }
    }
}
```

### 사용 예제

```csharp
public class CustomValueTaskSourceExample : MonoBehaviour
{
    public ValueTask<int> ComputeAsync()
    {
        var source = PooledValueTaskSource<int>.Rent();

        // 비동기 작업 시뮬레이션
        StartCoroutine(ComputeCoroutine(source));

        return source.Task;
    }

    private IEnumerator ComputeCoroutine(PooledValueTaskSource<int> source)
    {
        yield return new WaitForSeconds(1f);
        source.SetResult(42);
    }

    async void Start()
    {
        var result = await ComputeAsync();
        Debug.Log($"Result: {result}");
    }
}
```

---

## ValueTask vs UniTask

### 비교

| 특성 | ValueTask | UniTask |
|------|-----------|---------|
| 네임스페이스 | System.Threading.Tasks | Cysharp.Threading.Tasks |
| Unity 최적화 | 일반적 | Unity 전용 최적화 |
| PlayerLoop 통합 | 없음 | 있음 |
| 동기 완료 시 할당 | 0 | 0 |
| 비동기 완료 시 할당 | Task 래핑 | 최적화된 풀링 |
| 재사용 규칙 | 엄격함 | 더 유연함 |

### Unity에서의 선택 기준

```csharp
public class ValueTaskVsUniTask : MonoBehaviour
{
    // ValueTask 사용: 표준 .NET 호환성이 필요할 때
    public ValueTask<string> GetDataValueTaskAsync()
    {
        // System.Text.Json, HttpClient 등과 함께 사용
        return new ValueTask<string>("data");
    }

    // UniTask 사용: Unity 전용 최적화가 필요할 때
    public async UniTask<string> GetDataUniTaskAsync()
    {
        // PlayerLoop 기반 대기
        await UniTask.Yield(PlayerLoopTiming.Update);
        return "data";
    }
}
```

### 상호 변환

```csharp
using Cysharp.Threading.Tasks;

public class ValueTaskUniTaskConversion : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // ValueTask → UniTask
        ValueTask<int> valueTask = GetValueTaskAsync();
        int result1 = await valueTask.AsUniTask();

        // UniTask → Task (필요 시)
        UniTask<int> uniTask = GetUniTaskAsync();
        Task<int> task = uniTask.AsTask();

        // Task → UniTask
        Task<int> normalTask = Task.FromResult(42);
        int result2 = await normalTask.AsUniTask();
    }

    ValueTask<int> GetValueTaskAsync() => new ValueTask<int>(42);
    UniTask<int> GetUniTaskAsync() => UniTask.FromResult(42);
}
```

---

## 실전 패턴

### 패턴 1: 계층적 캐시

```csharp
public class HierarchicalCache<T> : MonoBehaviour where T : class
{
    private readonly Dictionary<string, T> _l1Cache = new Dictionary<string, T>();
    private readonly LRUCache<string, T> _l2Cache = new LRUCache<string, T>(1000);

    public ValueTask<T> GetAsync(string key, Func<string, Task<T>> loader)
    {
        // L1 캐시 체크 (가장 빠름)
        if (_l1Cache.TryGetValue(key, out var value))
        {
            return new ValueTask<T>(value);
        }

        // L2 캐시 체크
        if (_l2Cache.TryGet(key, out value))
        {
            _l1Cache[key] = value; // L1으로 승격
            return new ValueTask<T>(value);
        }

        // 캐시 미스 - 비동기 로드
        return new ValueTask<T>(LoadAndCacheAsync(key, loader));
    }

    private async Task<T> LoadAndCacheAsync(string key, Func<string, Task<T>> loader)
    {
        var value = await loader(key).ConfigureAwait(false);

        _l2Cache.Set(key, value);
        _l1Cache[key] = value;

        return value;
    }
}

// 간단한 LRU 캐시 구현
public class LRUCache<TKey, TValue>
{
    private readonly int _capacity;
    private readonly Dictionary<TKey, LinkedListNode<(TKey Key, TValue Value)>> _cache;
    private readonly LinkedList<(TKey Key, TValue Value)> _lruList;

    public LRUCache(int capacity)
    {
        _capacity = capacity;
        _cache = new Dictionary<TKey, LinkedListNode<(TKey, TValue)>>(capacity);
        _lruList = new LinkedList<(TKey, TValue)>();
    }

    public bool TryGet(TKey key, out TValue value)
    {
        if (_cache.TryGetValue(key, out var node))
        {
            _lruList.Remove(node);
            _lruList.AddFirst(node);
            value = node.Value.Value;
            return true;
        }

        value = default;
        return false;
    }

    public void Set(TKey key, TValue value)
    {
        if (_cache.TryGetValue(key, out var existingNode))
        {
            _lruList.Remove(existingNode);
            _cache.Remove(key);
        }
        else if (_cache.Count >= _capacity)
        {
            var last = _lruList.Last;
            _lruList.RemoveLast();
            _cache.Remove(last.Value.Key);
        }

        var newNode = _lruList.AddFirst((key, value));
        _cache[key] = newNode;
    }
}
```

### 패턴 2: Try 패턴

```csharp
public class TryPatternWithValueTask : MonoBehaviour
{
    private Dictionary<string, PlayerData> _playerCache = new Dictionary<string, PlayerData>();

    // Try 패턴의 비동기 버전
    public ValueTask<(bool Success, PlayerData Data)> TryGetPlayerAsync(string playerId)
    {
        if (_playerCache.TryGetValue(playerId, out var data))
        {
            return new ValueTask<(bool, PlayerData)>((true, data));
        }

        return new ValueTask<(bool, PlayerData)>(TryLoadPlayerAsync(playerId));
    }

    private async Task<(bool Success, PlayerData Data)> TryLoadPlayerAsync(string playerId)
    {
        try
        {
            var data = await LoadPlayerFromServerAsync(playerId);
            _playerCache[playerId] = data;
            return (true, data);
        }
        catch
        {
            return (false, null);
        }
    }

    async void Start()
    {
        var (success, player) = await TryGetPlayerAsync("player123");

        if (success)
        {
            Debug.Log($"Player: {player.Name}");
        }
        else
        {
            Debug.Log("Player not found");
        }
    }

    private Task<PlayerData> LoadPlayerFromServerAsync(string id)
        => Task.FromResult(new PlayerData { Name = id });
}

public class PlayerData
{
    public string Name { get; set; }
}
```

### 패턴 3: 조건부 비동기

```csharp
public class ConditionalAsync : MonoBehaviour
{
    private bool _isInitialized;
    private byte[] _preloadedData;
    private TaskCompletionSource<byte[]> _loadingTcs;

    public void PreloadData(byte[] data)
    {
        _preloadedData = data;
        _isInitialized = true;
    }

    // 상태에 따라 동기/비동기 반환
    public ValueTask<byte[]> GetDataAsync()
    {
        // 이미 초기화됨 - 동기 반환
        if (_isInitialized)
        {
            return new ValueTask<byte[]>(_preloadedData);
        }

        // 로딩 중 - 기존 Task 공유
        if (_loadingTcs != null)
        {
            return new ValueTask<byte[]>(_loadingTcs.Task);
        }

        // 첫 호출 - 로딩 시작
        return new ValueTask<byte[]>(LoadDataAsync());
    }

    private async Task<byte[]> LoadDataAsync()
    {
        _loadingTcs = new TaskCompletionSource<byte[]>();

        try
        {
            var data = await DownloadDataAsync();
            _preloadedData = data;
            _isInitialized = true;
            _loadingTcs.SetResult(data);
            return data;
        }
        catch (Exception ex)
        {
            _loadingTcs.SetException(ex);
            throw;
        }
    }

    private Task<byte[]> DownloadDataAsync()
        => Task.FromResult(new byte[1024]);
}
```

---

## 주의사항 및 베스트 프랙티스

### 주의사항 체크리스트

```csharp
public class ValueTaskBestPractices : MonoBehaviour
{
    // ❌ 여러 번 await 금지
    async void Bad_MultipleAwait()
    {
        var vt = GetValueTaskAsync();
        // await vt;
        // await vt; // 위험!
    }

    // ❌ 필드에 저장 금지
    // private ValueTask<int> _storedValueTask; // 위험!

    // ❌ 동시 대기 금지
    async void Bad_ConcurrentAwait()
    {
        var vt = GetValueTaskAsync();
        // var t1 = vt.AsTask();
        // var t2 = vt.AsTask(); // 위험!
    }

    // ✅ 즉시 await
    async void Good_ImmediateAwait()
    {
        var result = await GetValueTaskAsync();
    }

    // ✅ 저장 필요 시 AsTask()
    void Good_ConvertToTask()
    {
        var task = GetValueTaskAsync().AsTask();
        // 이제 task는 자유롭게 사용 가능
    }

    // ✅ 동기 완료가 많은 경우에만 사용
    ValueTask<int> Good_MostlySynchronous()
    {
        if (IsCached())
        {
            return new ValueTask<int>(GetCachedValue());
        }
        return new ValueTask<int>(LoadAsync());
    }

    private ValueTask<int> GetValueTaskAsync() => new ValueTask<int>(42);
    private bool IsCached() => true;
    private int GetCachedValue() => 42;
    private Task<int> LoadAsync() => Task.FromResult(42);
}
```

### 마이그레이션 가이드

```csharp
// Task에서 ValueTask로 마이그레이션

// 이전 (Task)
public class BeforeMigration
{
    private Dictionary<int, Item> _cache = new Dictionary<int, Item>();

    public async Task<Item> GetItemAsync(int id)
    {
        if (_cache.TryGetValue(id, out var item))
        {
            return item; // Task.FromResult 내부 호출
        }

        item = await LoadItemAsync(id);
        _cache[id] = item;
        return item;
    }

    private Task<Item> LoadItemAsync(int id) => Task.FromResult(new Item());
}

// 이후 (ValueTask)
public class AfterMigration
{
    private Dictionary<int, Item> _cache = new Dictionary<int, Item>();

    public ValueTask<Item> GetItemAsync(int id)
    {
        if (_cache.TryGetValue(id, out var item))
        {
            return new ValueTask<Item>(item); // 힙 할당 없음
        }

        return new ValueTask<Item>(LoadItemCoreAsync(id));
    }

    private async Task<Item> LoadItemCoreAsync(int id)
    {
        var item = await LoadItemAsync(id);
        _cache[id] = item;
        return item;
    }

    private Task<Item> LoadItemAsync(int id) => Task.FromResult(new Item());
}

public class Item { }
```

---

## 정리

| 항목 | 내용 |
|------|------|
| **사용 시기** | 동기 완료가 많은 고빈도 메서드 |
| **피해야 할 때** | 항상 비동기 완료, 결과 공유/저장 필요 |
| **핵심 규칙** | 한 번만 await, 병렬 대기 금지, 필드 저장 금지 |
| **Unity 권장** | UniTask가 더 적합, ValueTask는 .NET 호환 필요 시 |

---

## 참고 자료

- [Understanding ValueTask - Microsoft](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)
- [ValueTask<T> Best Practices](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.valuetask-1)
- [IValueTaskSource](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.sources.ivaluetasksource-1)
- [UniTask - Zero Allocation Alternative](https://github.com/Cysharp/UniTask)
