# 39. 메모리 관리 & GC

## 개요

Unity에서 동시성 프로그래밍을 할 때 **메모리 할당(Allocation)**과 **가비지 컬렉션(GC)**은 성능에 결정적인 영향을 미칩니다. 비동기 작업, 병렬 처리, 코루틴 등 모든 동시성 패턴은 의도치 않은 힙 할당을 유발할 수 있으며, 이는 GC 스파이크로 이어져 프레임 드랍을 발생시킵니다.

이 섹션에서는 Unity 동시성 프로그래밍에서 발생하는 메모리 문제를 진단하고 최적화하는 실전 기법을 다룹니다.

### Unity GC의 특성

Unity는 **Boehm GC**를 기본으로 사용합니다. 이 GC는 보수적(conservative), 비세대별(non-generational), Stop-the-World 방식으로 동작합니다.

| 특성 | Boehm GC (기본) | Incremental GC (선택) |
|------|----------------|----------------------|
| 수집 방식 | Stop-the-World | 점진적 (분산) |
| 프레임 영향 | 큰 스파이크 가능 | 작은 스파이크 분산 |
| 활성화 | 기본값 | Project Settings에서 활성화 |
| 세대별 수집 | 미지원 | 미지원 |
| 압축(Compaction) | 미지원 | 미지원 |

```csharp
// Incremental GC 확인 및 제어
public class GCInfoExample : MonoBehaviour
{
    void Start()
    {
        // 현재 GC 모드 확인
        Debug.Log($"Incremental GC: {UnityEngine.Scripting.GarbageCollector.isIncremental}");

        // Incremental GC 시간 예산 설정 (나노초)
        // 기본값: 약 3ms
        UnityEngine.Scripting.GarbageCollector.incrementalTimeSliceNanoseconds = 3_000_000; // 3ms
    }
}
```

> **핵심 원칙**: .NET의 세대별 GC와 달리 Unity의 Boehm GC는 **모든 힙 객체를 매번 스캔**합니다. 따라서 힙 할당을 줄이는 것이 .NET 서버 환경보다 훨씬 더 중요합니다.

---

## 1. Allocation 줄이기 - 비동기 작업에서의 힙 할당

비동기 프로그래밍에서 발생하는 대표적인 힙 할당 원인들을 파악합니다.

### 비동기 작업의 숨겨진 할당

```csharp
public class AsyncAllocationExample : MonoBehaviour
{
    // ❌ 매 호출마다 다수의 힙 할당 발생
    async Task BadAsyncPattern()
    {
        // 할당 1: async 상태머신 (IAsyncStateMachine) 박싱
        // 할당 2: Task 객체
        // 할당 3: ExecutionContext 캡처
        // 할당 4: 람다 클로저 (캡처 변수가 있을 경우)

        var data = new byte[1024]; // 할당 5: 바이트 배열
        await Task.Delay(100);

        var result = data.Select(b => b * 2).ToList(); // 할당 6, 7: LINQ iterator + List
        Debug.Log($"Result count: {result.Count}"); // 할당 8: string 보간
    }

    // ✅ 할당을 최소화한 패턴
    private byte[] _reusableBuffer = new byte[1024];
    private readonly StringBuilder _sb = new StringBuilder(64);

    void GoodSyncPattern()
    {
        // 버퍼 재사용
        Array.Clear(_reusableBuffer, 0, _reusableBuffer.Length);

        // LINQ 대신 직접 루프
        for (int i = 0; i < _reusableBuffer.Length; i++)
        {
            _reusableBuffer[i] = (byte)(_reusableBuffer[i] * 2);
        }

        // StringBuilder 재사용
        _sb.Clear();
        _sb.Append("Result count: ");
        _sb.Append(_reusableBuffer.Length);
        Debug.Log(_sb.ToString());
    }
}
```

### 프레임별 비동기 호출의 누적 할당

```csharp
public class FrameAllocationExample : MonoBehaviour
{
    // ❌ 매 프레임 힙 할당 발생
    async void Update()
    {
        // 매 프레임 async 상태머신 할당 + Task 할당
        await CheckConditionAsync();
    }

    private async Task CheckConditionAsync()
    {
        // 대부분 동기적으로 완료되지만 Task 객체는 항상 할당됨
        if (SomeCondition())
        {
            await Task.Yield();
        }
    }

    // ✅ 불필요한 비동기를 제거하고 조건부로만 사용
    private bool _needsAsyncCheck = false;

    void Update_Optimized()
    {
        // 동기 경로: 할당 없음
        if (!SomeCondition()) return;

        // 비동기가 정말 필요할 때만 호출
        if (_needsAsyncCheck)
        {
            _ = CheckConditionAsyncOptimized();
        }
    }

    private bool SomeCondition() => false;
    private async Task CheckConditionAsyncOptimized() { await Task.Yield(); }
}
```

---

## 2. Task vs ValueTask vs UniTask 메모리 비교

### 할당 비교표

| 항목 | Task\<T\> | ValueTask\<T\> | UniTask\<T\> |
|------|-----------|----------------|--------------|
| 동기 완료 시 | ~56-72 bytes | 0 bytes (스택) | 0 bytes (스택) |
| 비동기 완료 시 | ~72+ bytes | ~72+ bytes | ~0 bytes (풀링) |
| 상태머신 박싱 | 항상 | 비동기 시 | 안 함 (자체 풀) |
| CancellationToken 지원 | 추가 할당 | 추가 할당 | 내장 (할당 없음) |
| Unity 최적화 | 없음 | 부분적 | 완전 최적화 |

### 실측 비교 코드

```csharp
using System.Threading.Tasks;
using Cysharp.Threading.Tasks;

public class TaskMemoryComparisonExample : MonoBehaviour
{
    private Dictionary<string, byte[]> _cache = new();

    // =============================================
    // Task<T>: 동기 완료 시에도 힙 할당 발생
    // =============================================

    // ❌ 캐시 히트 시에도 ~56 bytes 할당
    public async Task<byte[]> GetWithTask(string key)
    {
        if (_cache.TryGetValue(key, out var data))
        {
            return data; // Task.FromResult() 내부에서 Task 객체 할당
        }

        data = await LoadDataAsync(key);
        _cache[key] = data;
        return data;
    }

    // =============================================
    // ValueTask<T>: 동기 완료 시 할당 제거
    // =============================================

    // ✅ 캐시 히트 시 0 bytes 할당
    public ValueTask<byte[]> GetWithValueTask(string key)
    {
        if (_cache.TryGetValue(key, out var data))
        {
            return new ValueTask<byte[]>(data); // 스택에만 할당
        }

        return new ValueTask<byte[]>(GetWithValueTaskCore(key));
    }

    private async Task<byte[]> GetWithValueTaskCore(string key)
    {
        var data = await LoadDataAsync(key);
        _cache[key] = data;
        return data;
    }

    // =============================================
    // UniTask<T>: 동기/비동기 모두 최소 할당
    // =============================================

    // ✅ 오브젝트 풀링으로 비동기 완료 시에도 할당 최소화
    public async UniTask<byte[]> GetWithUniTask(string key)
    {
        if (_cache.TryGetValue(key, out var data))
        {
            return data; // 할당 없음
        }

        // UniTask의 상태머신은 자체 풀에서 관리 -> GC 부하 없음
        data = await LoadDataUniTaskAsync(key);
        _cache[key] = data;
        return data;
    }

    private async Task<byte[]> LoadDataAsync(string key)
    {
        await Task.Delay(100);
        return new byte[256];
    }

    private async UniTask<byte[]> LoadDataUniTaskAsync(string key)
    {
        await UniTask.Delay(100);
        return new byte[256];
    }
}
```

### UniTask의 풀링 아키텍처

```csharp
// UniTask 내부 풀링 원리 이해 (개념 코드)
public class UniTaskPoolingConceptExample : MonoBehaviour
{
    // UniTask는 내부적으로 IStateMachineRunnerPromise를 풀링함
    // 일반 Task: async 호출마다 new StateMachine() -> 힙 할당
    // UniTask: 풀에서 빌려옴 -> 완료 후 반환 -> 할당 없음

    async UniTask DemonstratePooling()
    {
        // 첫 호출: 풀에서 StateMachineRunner 할당 (1회)
        await UniTask.Delay(100);
        // 완료 후: StateMachineRunner를 풀에 반환

        // 두 번째 호출: 풀에서 재사용 -> 할당 없음!
        await UniTask.Delay(100);

        // 1000번 호출해도 추가 할당이 거의 없음
        for (int i = 0; i < 1000; i++)
        {
            await UniTask.Yield();
        }
    }

    void Start()
    {
        // UniTask의 TaskPool 진단 정보
        // TaskPool.GetCacheSizeInfo()로 풀 상태 확인 가능
        var (count, size) = TaskPool.GetCacheSizeInfo();
        Debug.Log($"UniTask Pool - Types: {count}, Total cached: {size}");
    }
}
```

---

## 3. Object Pooling 패턴

### Unity의 ObjectPool\<T\> API

```csharp
using UnityEngine;
using UnityEngine.Pool;

// =============================================
// Unity 내장 ObjectPool<T> 사용
// =============================================

public class PoolingExample : MonoBehaviour
{
    // ObjectPool<T> - 스레드 안전하지 않음 (메인 스레드 전용)
    private ObjectPool<ParticleSystem> _particlePool;

    // LinkedPool<T> - 연결 리스트 기반 (메모리 효율적)
    private LinkedPool<List<int>> _listPool;

    void Awake()
    {
        _particlePool = new ObjectPool<ParticleSystem>(
            createFunc: () =>
            {
                var go = new GameObject("PooledParticle");
                return go.AddComponent<ParticleSystem>();
            },
            actionOnGet: ps =>
            {
                ps.gameObject.SetActive(true);
                ps.Play();
            },
            actionOnRelease: ps =>
            {
                ps.Stop(true, ParticleSystemStopBehavior.StopEmittingAndClear);
                ps.gameObject.SetActive(false);
            },
            actionOnDestroy: ps =>
            {
                Destroy(ps.gameObject);
            },
            collectionCheck: true,  // 중복 반환 검사 (디버그용)
            defaultCapacity: 10,
            maxSize: 100
        );

        _listPool = new LinkedPool<List<int>>(
            createFunc: () => new List<int>(32),
            actionOnGet: list => list.Clear(),
            actionOnRelease: list => list.Clear(),
            actionOnDestroy: list => { },
            collectionCheck: false,
            maxSize: 50
        );
    }

    void SpawnEffect(Vector3 position)
    {
        var ps = _particlePool.Get();
        ps.transform.position = position;

        // 일정 시간 후 반환
        StartCoroutine(ReturnAfterDelay(ps, 2f));
    }

    private System.Collections.IEnumerator ReturnAfterDelay(ParticleSystem ps, float delay)
    {
        yield return new WaitForSeconds(delay);
        _particlePool.Release(ps);
    }

    // 리스트 풀 활용 예
    void ProcessData()
    {
        // ✅ 풀에서 리스트를 가져와 사용 후 반환
        var tempList = _listPool.Get();
        try
        {
            for (int i = 0; i < 100; i++)
                tempList.Add(i);

            ProcessList(tempList);
        }
        finally
        {
            _listPool.Release(tempList);
        }
    }

    void ProcessList(List<int> list) { /* ... */ }
}
```

### 비동기 작업용 커스텀 풀

```csharp
using System.Collections.Concurrent;

// =============================================
// 스레드 안전한 오브젝트 풀
// =============================================

public class ThreadSafePool<T> where T : class, new()
{
    private readonly ConcurrentBag<T> _pool = new();
    private readonly Action<T> _resetAction;
    private readonly int _maxSize;
    private int _count;

    public ThreadSafePool(Action<T> resetAction = null, int maxSize = 100)
    {
        _resetAction = resetAction;
        _maxSize = maxSize;
    }

    public T Rent()
    {
        if (_pool.TryTake(out var item))
        {
            Interlocked.Decrement(ref _count);
            return item;
        }
        return new T();
    }

    public void Return(T item)
    {
        if (Interlocked.Increment(ref _count) <= _maxSize)
        {
            _resetAction?.Invoke(item);
            _pool.Add(item);
        }
        else
        {
            Interlocked.Decrement(ref _count);
        }
    }
}

// 사용 예
public class AsyncPoolUsageExample : MonoBehaviour
{
    private static readonly ThreadSafePool<List<Vector3>> _vectorListPool =
        new(list => list.Clear(), maxSize: 20);

    async UniTask ProcessEnemiesAsync(CancellationToken ct)
    {
        var positions = _vectorListPool.Rent();
        try
        {
            // 백그라운드에서 처리
            await UniTask.RunOnThreadPool(() =>
            {
                for (int i = 0; i < 1000; i++)
                {
                    positions.Add(CalculatePosition(i));
                }
            }, cancellationToken: ct);

            // 메인 스레드에서 적용
            await UniTask.SwitchToMainThread();
            ApplyPositions(positions);
        }
        finally
        {
            _vectorListPool.Return(positions);
        }
    }

    Vector3 CalculatePosition(int i) => new Vector3(i, 0, i);
    void ApplyPositions(List<Vector3> positions) { /* ... */ }
}
```

### GenericPool과 ListPool (Unity 내장)

```csharp
using UnityEngine.Pool;

public class BuiltInPoolExample : MonoBehaviour
{
    void Example()
    {
        // =============================================
        // GenericPool<T> - static 풀 (편의 API)
        // =============================================

        var list = ListPool<int>.Get();
        try
        {
            list.Add(1);
            list.Add(2);
            list.Add(3);
        }
        finally
        {
            ListPool<int>.Release(list);
        }

        // =============================================
        // DictionaryPool
        // =============================================

        var dict = DictionaryPool<string, int>.Get();
        try
        {
            dict["score"] = 100;
            dict["health"] = 80;
        }
        finally
        {
            DictionaryPool<string, int>.Release(dict);
        }

        // =============================================
        // HashSetPool
        // =============================================

        var set = HashSetPool<int>.Get();
        try
        {
            set.Add(1);
            set.Add(2);
        }
        finally
        {
            HashSetPool<int>.Release(set);
        }
    }
}
```

---

## 4. Span\<T\>, Memory\<T\>, ReadOnlySpan\<T\>

`Span<T>`와 `Memory<T>`는 힙 할당 없이 메모리 슬라이스를 다루는 타입입니다. 배열 복사를 제거하고 GC 부하를 줄일 수 있습니다.

### 기본 개념

```csharp
using System;

public class SpanBasicsExample : MonoBehaviour
{
    void Start()
    {
        int[] numbers = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

        // ❌ 배열 슬라이스 - 새 배열 할당
        int[] slice = new int[5];
        Array.Copy(numbers, 2, slice, 0, 5); // 힙 할당!

        // ✅ Span 슬라이스 - 할당 없음
        Span<int> spanSlice = numbers.AsSpan(2, 5); // 힙 할당 없음!

        // Span으로 원본 수정 가능
        spanSlice[0] = 99; // numbers[2]가 99로 변경됨

        // ReadOnlySpan - 읽기 전용 뷰
        ReadOnlySpan<int> readOnly = numbers.AsSpan();
        int sum = SumSpan(readOnly);
        Debug.Log($"Sum: {sum}");
    }

    // Span을 매개변수로 받는 메서드 - 할당 없이 배열 부분 처리
    static int SumSpan(ReadOnlySpan<int> span)
    {
        int sum = 0;
        for (int i = 0; i < span.Length; i++)
        {
            sum += span[i];
        }
        return sum;
    }
}
```

### Memory\<T\> - 비동기에서 사용 가능한 Span

```csharp
using System;
using System.Threading.Tasks;

public class MemoryExample : MonoBehaviour
{
    // Span<T>는 ref struct이므로 async 메서드에서 사용 불가
    // Memory<T>는 async 메서드에서 안전하게 사용 가능

    private byte[] _largeBuffer = new byte[1024 * 1024]; // 1MB

    // ❌ Span은 async 메서드에서 사용 불가
    // async Task ProcessSpanAsync(Span<byte> span) { } // 컴파일 에러!

    // ✅ Memory<T>는 async에서 사용 가능
    async Task ProcessMemoryAsync(Memory<byte> memory)
    {
        // 비동기 작업에서 Memory 사용
        await Task.Run(() =>
        {
            // Memory에서 Span을 얻어 동기 처리
            Span<byte> span = memory.Span;
            for (int i = 0; i < span.Length; i++)
            {
                span[i] = (byte)(i % 256);
            }
        });
    }

    async Task ExampleUsage()
    {
        // 대용량 버퍼를 청크로 나누어 비동기 처리
        int chunkSize = 1024;

        for (int offset = 0; offset < _largeBuffer.Length; offset += chunkSize)
        {
            int length = Math.Min(chunkSize, _largeBuffer.Length - offset);
            Memory<byte> chunk = _largeBuffer.AsMemory(offset, length);

            await ProcessMemoryAsync(chunk); // 할당 없이 슬라이스 전달
        }
    }
}
```

### Unity 데이터 처리에서의 Span 활용

```csharp
public class UnitySpanExample : MonoBehaviour
{
    // 물리 레이캐스트 결과 처리
    private RaycastHit[] _hitBuffer = new RaycastHit[32];

    void FixedUpdate()
    {
        int hitCount = Physics.RaycastNonAlloc(
            transform.position, transform.forward, _hitBuffer, 100f);

        if (hitCount > 0)
        {
            // ✅ 할당 없이 결과 슬라이스 처리
            ProcessHits(_hitBuffer.AsSpan(0, hitCount));
        }
    }

    void ProcessHits(ReadOnlySpan<RaycastHit> hits)
    {
        float closestDistance = float.MaxValue;
        int closestIndex = -1;

        for (int i = 0; i < hits.Length; i++)
        {
            if (hits[i].distance < closestDistance)
            {
                closestDistance = hits[i].distance;
                closestIndex = i;
            }
        }

        if (closestIndex >= 0)
        {
            Debug.Log($"Closest hit: {hits[closestIndex].collider.name}");
        }
    }
}
```

---

## 5. ArrayPool\<T\> 활용

`ArrayPool<T>`는 배열을 풀링하여 반복적인 배열 할당을 제거합니다.

```csharp
using System;
using System.Buffers;
using UnityEngine;

public class ArrayPoolExample : MonoBehaviour
{
    // =============================================
    // ArrayPool 기본 사용법
    // =============================================

    void ProcessFrameData()
    {
        // ❌ 매 프레임 배열 할당
        // byte[] buffer = new byte[4096]; // 힙 할당!

        // ✅ ArrayPool에서 대여
        byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
        // 주의: 실제 반환 배열 크기는 4096 이상일 수 있음 (2의 거듭제곱)

        try
        {
            // buffer.Length >= 4096 (정확한 크기가 아닐 수 있음)
            ProcessBuffer(buffer, actualLength: 4096);
        }
        finally
        {
            // 반드시 반환! clearArray: true면 보안상 데이터 초기화
            ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
        }
    }

    // =============================================
    // 커스텀 ArrayPool 생성
    // =============================================

    // 특정 크기 범위에 최적화된 풀
    private static readonly ArrayPool<float> _floatPool =
        ArrayPool<float>.Create(maxArrayLength: 1024 * 64, maxArraysPerBucket: 10);

    void ProcessPhysicsData(int entityCount)
    {
        float[] velocities = _floatPool.Rent(entityCount * 3); // xyz per entity
        float[] positions = _floatPool.Rent(entityCount * 3);

        try
        {
            // 물리 계산
            for (int i = 0; i < entityCount; i++)
            {
                int idx = i * 3;
                positions[idx]     += velocities[idx]     * Time.deltaTime;
                positions[idx + 1] += velocities[idx + 1] * Time.deltaTime;
                positions[idx + 2] += velocities[idx + 2] * Time.deltaTime;
            }
        }
        finally
        {
            _floatPool.Return(velocities);
            _floatPool.Return(positions);
        }
    }

    // =============================================
    // IMemoryOwner<T> 패턴 - 자동 반환
    // =============================================

    void MemoryOwnerPattern()
    {
        // using과 함께 사용하면 자동 반환
        using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(1024);
        Memory<byte> memory = owner.Memory;

        // memory를 사용한 처리...
        Span<byte> span = memory.Span;
        for (int i = 0; i < span.Length; i++)
        {
            span[i] = (byte)(i % 256);
        }
    } // 여기서 자동으로 풀에 반환됨

    void ProcessBuffer(byte[] buffer, int actualLength) { /* ... */ }
}
```

### ArrayPool과 비동기 조합

```csharp
public class ArrayPoolAsyncExample : MonoBehaviour
{
    async UniTask<int> ProcessLargeDataAsync(CancellationToken ct)
    {
        // 풀에서 대여
        float[] buffer = ArrayPool<float>.Shared.Rent(10000);

        try
        {
            // 백그라운드 스레드에서 데이터 처리
            int result = await UniTask.RunOnThreadPool(() =>
            {
                int sum = 0;
                for (int i = 0; i < 10000; i++)
                {
                    buffer[i] = MathF.Sin(i * 0.01f);
                    sum += (int)(buffer[i] * 100);
                }
                return sum;
            }, cancellationToken: ct);

            return result;
        }
        finally
        {
            // 비동기 완료 후 반환
            ArrayPool<float>.Shared.Return(buffer);
        }
    }
}
```

---

## 6. stackalloc과 Span

`stackalloc`은 스택에 메모리를 할당하여 GC 부하를 완전히 제거합니다.

```csharp
public class StackallocExample : MonoBehaviour
{
    // =============================================
    // 기본 stackalloc 사용
    // =============================================

    void ProcessSmallData()
    {
        // ✅ 스택 할당 - GC 부하 없음
        Span<int> buffer = stackalloc int[64];

        for (int i = 0; i < buffer.Length; i++)
        {
            buffer[i] = i * i;
        }

        int max = FindMax(buffer);
        Debug.Log($"Max: {max}");
    }

    static int FindMax(ReadOnlySpan<int> span)
    {
        int max = int.MinValue;
        for (int i = 0; i < span.Length; i++)
        {
            if (span[i] > max) max = span[i];
        }
        return max;
    }

    // =============================================
    // 조건부 stackalloc/ArrayPool 패턴
    // =============================================

    void ProcessVariableData(int count)
    {
        // 작은 크기: 스택, 큰 크기: ArrayPool
        const int StackAllocThreshold = 256;

        byte[] rentedArray = null;
        Span<byte> buffer = count <= StackAllocThreshold
            ? stackalloc byte[count]
            : (rentedArray = ArrayPool<byte>.Shared.Rent(count));

        try
        {
            // buffer 사용
            for (int i = 0; i < count; i++)
            {
                buffer[i] = (byte)(i % 256);
            }

            ProcessSpan(buffer.Slice(0, count));
        }
        finally
        {
            // ArrayPool에서 빌렸다면 반환
            if (rentedArray != null)
            {
                ArrayPool<byte>.Shared.Return(rentedArray);
            }
        }
    }

    void ProcessSpan(Span<byte> data) { /* ... */ }

    // =============================================
    // 임시 문자열 비교에 stackalloc 활용
    // =============================================

    bool CompareTagOptimized(string tag)
    {
        // 짧은 문자열의 경우 stackalloc 활용
        Span<char> lowerTag = stackalloc char[tag.Length];
        tag.AsSpan().ToLowerInvariant(lowerTag);

        return lowerTag.SequenceEqual("enemy".AsSpan());
    }
}
```

> **주의**: `stackalloc`은 스택 크기 제한(기본 1MB)이 있으므로, 큰 버퍼에는 사용하지 마세요. 일반적으로 1KB 이하에서 사용을 권장합니다.

---

## 7. string 할당 줄이기

문자열은 불변(immutable) 참조 타입이므로 모든 조작이 새 힙 할당을 유발합니다.

### StringBuilder 활용

```csharp
using System.Text;

public class StringOptimizationExample : MonoBehaviour
{
    // ❌ 문자열 연결마다 힙 할당
    string BadStringBuild(int score, float time, string playerName)
    {
        // 매 연결마다 새 string 객체 생성
        string result = "Player: " + playerName;     // 할당 1
        result += " | Score: " + score.ToString();    // 할당 2, 3
        result += " | Time: " + time.ToString("F2");  // 할당 4, 5
        return result;
    }

    // ✅ StringBuilder 재사용
    private readonly StringBuilder _sb = new StringBuilder(128);

    string GoodStringBuild(int score, float time, string playerName)
    {
        _sb.Clear();
        _sb.Append("Player: ").Append(playerName)
           .Append(" | Score: ").Append(score)
           .Append(" | Time: ").Append(time.ToString("F2"));
        return _sb.ToString(); // 최종 1회만 할당
    }

    // =============================================
    // UI 텍스트 업데이트 최적화
    // =============================================

    private TMPro.TMP_Text _scoreText;
    private int _lastScore = -1;
    private readonly StringBuilder _uiSb = new StringBuilder(32);

    // ❌ 매 프레임 string 할당
    void UpdateScoreBad(int score)
    {
        _scoreText.text = $"Score: {score}"; // 매 프레임 할당!
    }

    // ✅ 값 변경 시에만 업데이트 + StringBuilder
    void UpdateScoreGood(int score)
    {
        if (score == _lastScore) return; // 변경 없으면 스킵
        _lastScore = score;

        _uiSb.Clear();
        _uiSb.Append("Score: ").Append(score);
        _scoreText.SetText(_uiSb); // TMP는 StringBuilder 직접 수용
    }
}
```

### string.Create 활용 (C# 9.0+)

```csharp
public class StringCreateExample : MonoBehaviour
{
    // ✅ string.Create - 중간 할당 없이 문자열 생성
    string FormatPlayerInfo(int id, int score)
    {
        // 정확한 길이를 미리 계산하여 1회 할당만 발생
        return string.Create(32, (id, score), static (span, state) =>
        {
            "Player#".AsSpan().CopyTo(span);
            int pos = 7;

            state.id.TryFormat(span.Slice(pos), out int written);
            pos += written;

            ": ".AsSpan().CopyTo(span.Slice(pos));
            pos += 2;

            state.score.TryFormat(span.Slice(pos), out written);
            pos += written;

            // 남은 공간 정리
            span.Slice(pos).Fill('\0');
        });
    }
}
```

### Interpolated String Handler (C# 10+)

```csharp
using System.Runtime.CompilerServices;
using System.Text;

// =============================================
// 커스텀 Interpolated String Handler - 조건부 문자열 생성
// =============================================

[InterpolatedStringHandler]
public ref struct LogInterpolatedStringHandler
{
    private StringBuilder _builder;
    private readonly bool _enabled;

    public LogInterpolatedStringHandler(
        int literalLength,
        int formattedCount,
        LogLevel level,
        out bool handlerIsValid)
    {
        // 로그 레벨이 비활성화면 문자열을 아예 생성하지 않음
        _enabled = IsLogLevelEnabled(level);
        handlerIsValid = _enabled;

        _builder = _enabled ? new StringBuilder(literalLength + formattedCount * 8) : null;
    }

    public void AppendLiteral(string s)
    {
        if (_enabled) _builder.Append(s);
    }

    public void AppendFormatted<T>(T value)
    {
        if (_enabled) _builder.Append(value);
    }

    public override string ToString() => _builder?.ToString() ?? string.Empty;

    static bool IsLogLevelEnabled(LogLevel level) => level >= LogLevel.Warning;
}

public enum LogLevel { Debug, Info, Warning, Error }

public class InterpolatedStringExample : MonoBehaviour
{
    // ✅ 로그 레벨이 낮으면 문자열 자체를 생성하지 않음
    void LogMessage(LogLevel level, [InterpolatedStringHandlerArgument("level")]
        ref LogInterpolatedStringHandler handler)
    {
        if (level >= LogLevel.Warning)
        {
            Debug.Log(handler.ToString());
        }
    }

    void Example()
    {
        int playerCount = 42;
        float fps = 59.8f;

        // Debug 레벨이 비활성화면 string 포맷팅 자체가 발생하지 않음!
        // → GC 할당 0 bytes
        LogMessage(LogLevel.Debug, $"Players: {playerCount}, FPS: {fps}");
    }
}
```

---

## 8. 클로저와 캡처 변수에 의한 할당

람다와 클로저는 컴파일러가 자동 생성하는 캡처 클래스로 인해 숨겨진 힙 할당을 유발합니다.

### 캡처의 비용

```csharp
public class ClosureAllocationExample : MonoBehaviour
{
    // =============================================
    // 캡처 변수에 의한 할당
    // =============================================

    void CaptureExamples()
    {
        int localValue = 42;
        string localName = "test";

        // ❌ 지역 변수 캡처 → 컴파일러가 캡처 클래스 생성 (힙 할당)
        Action captureAction = () =>
        {
            Debug.Log($"{localName}: {localValue}");
            // 컴파일러 변환 결과:
            // new <>c__DisplayClass { localValue = 42, localName = "test" }
            // 이 클래스 인스턴스가 힙에 할당됨
        };

        // ❌ this 캡처 → delegate 객체 할당
        Action thisCapture = () =>
        {
            Debug.Log(this.name); // 'this' 캡처
        };

        // ✅ 캡처 없는 static 람다 (C# 9+) → 할당 없음
        Action staticAction = static () =>
        {
            Debug.Log("No capture, no allocation");
        };
    }

    // =============================================
    // 비동기 작업에서의 클로저 비용
    // =============================================

    // ❌ Task.Run에서 클로저 사용 - 매 호출마다 캡처 클래스 할당
    void BadAsyncClosure(int enemyId, Vector3 position)
    {
        Task.Run(() =>
        {
            // enemyId, position을 캡처 → 힙 할당
            ProcessEnemy(enemyId, position);
        });
    }

    // ✅ 상태를 object로 전달하여 클로저 회피
    private static readonly Action<object> _processEnemyAction = ProcessEnemyState;

    void GoodAsyncClosure(int enemyId, Vector3 position)
    {
        var state = EnemyStatePool.Rent();
        state.Id = enemyId;
        state.Position = position;

        Task.Factory.StartNew(_processEnemyAction, state);
    }

    static void ProcessEnemyState(object stateObj)
    {
        var state = (EnemyState)stateObj;
        ProcessEnemy(state.Id, state.Position);
        EnemyStatePool.Return(state);
    }

    static void ProcessEnemy(int id, Vector3 pos) { /* ... */ }
}

// 풀링 가능한 상태 객체
class EnemyState
{
    public int Id;
    public Vector3 Position;
}

static class EnemyStatePool
{
    private static readonly ConcurrentBag<EnemyState> _pool = new();

    public static EnemyState Rent()
    {
        return _pool.TryTake(out var state) ? state : new EnemyState();
    }

    public static void Return(EnemyState state)
    {
        _pool.Add(state);
    }
}
```

### 이벤트 구독에서의 클로저

```csharp
public class EventClosureExample : MonoBehaviour
{
    public event Action<int> OnScoreChanged;

    // ❌ 매번 새 delegate 할당
    void BadSubscribe()
    {
        int multiplier = 2;

        // 람다가 multiplier를 캡처 → 매 호출마다 새 캡처 클래스 + delegate 할당
        OnScoreChanged += (score) => ApplyMultiplier(score, multiplier);
    }

    // ✅ 캡처를 피하는 방법 - 필드로 이동
    private int _multiplier = 2;
    private Action<int> _cachedHandler;

    void GoodSubscribe()
    {
        // 핸들러를 캐싱하여 재사용
        _cachedHandler ??= HandleScoreChanged;
        OnScoreChanged += _cachedHandler;
    }

    void HandleScoreChanged(int score)
    {
        ApplyMultiplier(score, _multiplier);
    }

    void OnDisable()
    {
        if (_cachedHandler != null)
            OnScoreChanged -= _cachedHandler;
    }

    void ApplyMultiplier(int score, int mult) { /* ... */ }
}
```

---

## 9. async 상태머신의 힙 할당

C# 컴파일러는 `async` 메서드를 상태머신(state machine) 구조체로 변환합니다. 이 과정에서 박싱이 발생합니다.

### 상태머신 할당 원리

```csharp
public class AsyncStateMachineExample : MonoBehaviour
{
    // 원본 코드:
    async Task<int> CalculateAsync(int input)
    {
        int a = input * 2;
        await Task.Delay(100);
        int b = a + 10;
        await Task.Delay(100);
        return b;
    }

    // 컴파일러가 대략 이런 구조로 변환:
    //
    // struct <CalculateAsync>d__0 : IAsyncStateMachine  // 구조체로 선언
    // {
    //     public int <>1__state;           // 현재 상태
    //     public AsyncTaskMethodBuilder<int> <>t__builder;
    //     public int input;                // 매개변수 캡처
    //     private int <a>5__1;             // 지역 변수 보존
    //     private int <b>5__2;
    //     private TaskAwaiter <>u__1;      // awaiter 보관
    //
    //     public void MoveNext() { ... }
    //     public void SetStateMachine(IAsyncStateMachine machine) { ... }
    // }
    //
    // 문제: 첫 번째 await에서 미완료 시,
    // 구조체가 IAsyncStateMachine으로 박싱 → 힙 할당 발생!

    // =============================================
    // 할당 발생 시나리오 분석
    // =============================================

    // 시나리오 1: 동기 완료 → 할당 최소
    async Task<int> SyncCompletion()
    {
        // await 없이 즉시 반환하면 상태머신 박싱 없음
        return 42;
        // 하지만 Task<int> 객체는 여전히 할당됨
        // → ValueTask<int>로 해결 가능
    }

    // 시나리오 2: 비동기 완료 → 박싱 발생
    async Task<int> AsyncCompletion()
    {
        await Task.Delay(100); // 상태머신 박싱 발생!
        return 42;
        // 할당: Task<int> 객체 + 상태머신 박싱 + ExecutionContext
    }

    // =============================================
    // 최적화: UniTask로 상태머신 풀링
    // =============================================

    // ✅ UniTask는 상태머신을 풀링하여 할당 제거
    async UniTask<int> OptimizedCalculation(int input)
    {
        int a = input * 2;
        await UniTask.Delay(100);      // 상태머신 풀링 → 할당 없음
        int b = a + 10;
        await UniTask.Delay(100);      // 풀에서 재사용
        return b;
    }
}
```

### ConfigureAwait(false)의 메모리 효과

```csharp
public class ConfigureAwaitMemoryExample : MonoBehaviour
{
    // SynchronizationContext 캡처 회피 → 미세한 할당 절약
    async Task BackgroundWork()
    {
        // ❌ SynchronizationContext를 캡처 (Unity에서 UnitySynchronizationContext)
        await Task.Delay(100);

        // ✅ 컨텍스트 캡처 회피 → 할당 미세 감소
        await Task.Delay(100).ConfigureAwait(false);

        // UniTask에서는 기본적으로 컨텍스트를 캡처하지 않음
    }
}
```

---

## 10. NativeArray와 Unmanaged 메모리

Unity의 `NativeArray<T>`는 관리되지 않는(unmanaged) 메모리에 할당되어 GC에 영향을 주지 않습니다.

### NativeArray 기본

```csharp
using Unity.Collections;
using Unity.Collections.LowLevel.Unsafe;

public class NativeMemoryExample : MonoBehaviour
{
    // =============================================
    // NativeArray - GC 프리 배열
    // =============================================

    void NativeArrayUsage()
    {
        // Allocator 종류:
        // Temp      - 1 프레임 이내 (가장 빠름)
        // TempJob   - 4 프레임 이내 (Job에 적합)
        // Persistent - 수동 Dispose 필요 (장기 보관)

        // ✅ GC에 영향 없는 메모리 할당
        var positions = new NativeArray<Vector3>(10000, Allocator.TempJob);

        // 일반 배열에서 복사
        Vector3[] managedArray = new Vector3[10000];
        positions.CopyFrom(managedArray);

        // NativeArray -> Span 변환 (Unity 2022.2+)
        // 할당 없이 데이터 접근 가능

        // 반드시 Dispose
        positions.Dispose();
    }

    // =============================================
    // UnsafeUtility - 저수준 메모리 관리
    // =============================================

    unsafe void UnsafeMemoryUsage()
    {
        int count = 1000;
        int size = count * sizeof(float);

        // 네이티브 메모리 할당 (GC 무관)
        void* ptr = UnsafeUtility.Malloc(size, UnsafeUtility.AlignOf<float>(), Allocator.Temp);

        try
        {
            float* floats = (float*)ptr;
            for (int i = 0; i < count; i++)
            {
                floats[i] = i * 0.5f;
            }

            // 메모리 복사
            void* dest = UnsafeUtility.Malloc(size, UnsafeUtility.AlignOf<float>(), Allocator.Temp);
            UnsafeUtility.MemCpy(dest, ptr, size);
            UnsafeUtility.Free(dest, Allocator.Temp);
        }
        finally
        {
            UnsafeUtility.Free(ptr, Allocator.Temp);
        }
    }

    // =============================================
    // NativeArray와 관리 배열 간 전환
    // =============================================

    void ManagedNativeBridge()
    {
        // 관리 배열 → NativeArray (복사)
        float[] managed = { 1f, 2f, 3f, 4f, 5f };
        var native = new NativeArray<float>(managed, Allocator.Temp);

        // NativeArray → 관리 배열 (복사)
        float[] backToManaged = native.ToArray(); // 주의: 힙 할당 발생!

        // ✅ 복사 없이 NativeArray 내용 사용
        // NativeSlice로 부분 뷰 (할당 없음)
        NativeSlice<float> slice = new NativeSlice<float>(native, 1, 3);

        native.Dispose();
    }
}
```

### Persistent 할당과 수명 관리

```csharp
public class PersistentNativeArrayExample : MonoBehaviour
{
    // 장기 보관용 네이티브 컨테이너
    private NativeArray<float> _healthData;
    private NativeArray<Vector3> _positionData;
    private bool _initialized;

    void OnEnable()
    {
        int entityCount = 10000;
        _healthData = new NativeArray<float>(entityCount, Allocator.Persistent);
        _positionData = new NativeArray<Vector3>(entityCount, Allocator.Persistent);
        _initialized = true;

        // 초기화
        for (int i = 0; i < entityCount; i++)
        {
            _healthData[i] = 100f;
            _positionData[i] = Vector3.zero;
        }
    }

    void Update()
    {
        if (!_initialized) return;

        // NativeArray는 GC 트래킹 없이 사용 가능
        // 매 프레임 접근해도 GC 부하 없음
        for (int i = 0; i < _healthData.Length; i++)
        {
            if (_healthData[i] <= 0)
            {
                // 엔티티 제거 로직
            }
        }
    }

    void OnDisable()
    {
        // 반드시 수동 해제
        if (_healthData.IsCreated) _healthData.Dispose();
        if (_positionData.IsCreated) _positionData.Dispose();
        _initialized = false;
    }
}
```

---

## 11. GC.Collect 수동 호출 전략

GC를 수동으로 호출하는 것은 일반적으로 권장되지 않지만, 특정 상황에서는 전략적으로 활용할 수 있습니다.

```csharp
using System;
using UnityEngine;
using UnityEngine.SceneManagement;

public class GCStrategyExample : MonoBehaviour
{
    // =============================================
    // 전략적 GC 호출 타이밍
    // =============================================

    void OnEnable()
    {
        SceneManager.sceneLoaded += OnSceneLoaded;
    }

    void OnDisable()
    {
        SceneManager.sceneLoaded -= OnSceneLoaded;
    }

    // ✅ 장면 전환 후: 유저가 로딩을 기대하는 시점
    void OnSceneLoaded(Scene scene, LoadSceneMode mode)
    {
        // 장면 로드 후 GC 수행
        // 이 시점에서 이전 장면의 객체들이 해제 대상
        StartCoroutine(PostSceneLoadGC());
    }

    private System.Collections.IEnumerator PostSceneLoadGC()
    {
        yield return null; // 1프레임 대기 (로딩 UI가 보이는 동안)

        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect(); // 파이널라이저가 해제한 객체도 수집

        // Unload unused assets
        yield return Resources.UnloadUnusedAssets();
    }

    // =============================================
    // 주기적 GC 관리 (Incremental GC와 함께)
    // =============================================

    [SerializeField] private float _gcInterval = 30f; // 30초마다
    private float _lastGCTime;

    void Update()
    {
        // 일정 주기마다 GC 실행 (프레임 시작 시)
        if (Time.realtimeSinceStartup - _lastGCTime > _gcInterval)
        {
            // Incremental GC가 켜져 있으면 스파이크가 분산됨
            if (UnityEngine.Scripting.GarbageCollector.isIncremental)
            {
                // incrementalTimeSliceNanoseconds에 의해 시간 제한
                System.GC.Collect(0, GCCollectionMode.Optimized, false);
            }

            _lastGCTime = Time.realtimeSinceStartup;
        }
    }

    // =============================================
    // GC 비활성화 구간 (위험: 메모리 계속 증가)
    // =============================================

    // ✅ 보스전 등 절대 스파이크가 허용되지 않는 구간
    void EnterCriticalSection()
    {
        // GC를 완전히 비활성화
        GarbageCollector.GCMode = GarbageCollector.Mode.Disabled;
        Debug.LogWarning("GC 비활성화됨 - 메모리 사용량 모니터링 필요");
    }

    void ExitCriticalSection()
    {
        // GC 재활성화
        GarbageCollector.GCMode = GarbageCollector.Mode.Enabled;

        // 즉시 수집 수행
        GC.Collect();
    }

    // =============================================
    // GC 상태 모니터링
    // =============================================

    void LogGCStats()
    {
        long totalMemory = GC.GetTotalMemory(forceFullCollection: false);
        int gen0Collections = GC.CollectionCount(0);

        Debug.Log($"총 관리 메모리: {totalMemory / 1024f / 1024f:F2} MB");
        Debug.Log($"GC 수집 횟수: {gen0Collections}");

        // Unity 전용 메모리 정보
        long totalReserved = UnityEngine.Profiling.Profiler.GetTotalReservedMemoryLong();
        long totalAllocated = UnityEngine.Profiling.Profiler.GetTotalAllocatedMemoryLong();
        long monoHeap = UnityEngine.Profiling.Profiler.GetMonoHeapSizeLong();
        long monoUsed = UnityEngine.Profiling.Profiler.GetMonoUsedSizeLong();

        Debug.Log($"Reserved: {totalReserved / 1024f / 1024f:F2} MB");
        Debug.Log($"Allocated: {totalAllocated / 1024f / 1024f:F2} MB");
        Debug.Log($"Mono Heap: {monoHeap / 1024f / 1024f:F2} MB");
        Debug.Log($"Mono Used: {monoUsed / 1024f / 1024f:F2} MB");
    }
}
```

---

## 12. Memory Profiler 활용

### Unity Profiler로 GC 할당 추적

```csharp
using UnityEngine;
using UnityEngine.Profiling;

public class MemoryProfilingExample : MonoBehaviour
{
    // =============================================
    // Profiler.BeginSample로 GC 할당 구간 측정
    // =============================================

    void Update()
    {
        Profiler.BeginSample("MyGame.Update.PhysicsCheck");
        PerformPhysicsCheck();
        Profiler.EndSample();

        Profiler.BeginSample("MyGame.Update.AILogic");
        UpdateAI();
        Profiler.EndSample();

        Profiler.BeginSample("MyGame.Update.Rendering");
        PrepareRendering();
        Profiler.EndSample();
    }

    // =============================================
    // 커스텀 메모리 추적기
    // =============================================

    private long _lastAllocated;

    void TrackAllocationDelta()
    {
        long currentAllocated = Profiler.GetTotalAllocatedMemoryLong();
        long delta = currentAllocated - _lastAllocated;

        if (delta > 1024 * 100) // 100KB 이상 증가 시 경고
        {
            Debug.LogWarning($"큰 할당 감지: {delta / 1024f:F1} KB 증가");
        }

        _lastAllocated = currentAllocated;
    }

    // =============================================
    // ProfilerRecorder로 GC Alloc 실시간 추적 (Unity 2021+)
    // =============================================

    private ProfilerRecorder _gcAllocRecorder;
    private ProfilerRecorder _totalMemoryRecorder;

    void OnEnable()
    {
        _gcAllocRecorder = ProfilerRecorder.StartNew(
            ProfilerCategory.Memory, "GC Allocated In Frame");
        _totalMemoryRecorder = ProfilerRecorder.StartNew(
            ProfilerCategory.Memory, "Total Used Memory");
    }

    void OnDisable()
    {
        _gcAllocRecorder.Dispose();
        _totalMemoryRecorder.Dispose();
    }

    void OnGUI()
    {
        if (!_gcAllocRecorder.Valid) return;

        string text = $"GC Alloc/Frame: {_gcAllocRecorder.LastValue / 1024f:F1} KB\n" +
                      $"Total Memory: {_totalMemoryRecorder.LastValue / (1024f * 1024f):F1} MB";

        GUI.Label(new Rect(10, 10, 300, 50), text);
    }

    void PerformPhysicsCheck() { /* ... */ }
    void UpdateAI() { /* ... */ }
    void PrepareRendering() { /* ... */ }
}
```

### Memory Profiler 패키지 활용

```csharp
// Memory Profiler 패키지 설치:
// Window > Package Manager > Memory Profiler

// 스냅샷 기반 분석 워크플로우:
// 1. Window > Analysis > Memory Profiler 열기
// 2. 게임 실행 중 "Capture" 클릭
// 3. 두 시점의 스냅샷을 비교하여 메모리 누수 탐지
//
// 주요 확인 사항:
// - "All Of Memory" 탭: 전체 메모리 분포
// - "Unity Objects" 탭: Unity 오브젝트별 메모리
// - "Managed Shell Objects": C# 래퍼 오브젝트
// - 스냅샷 비교(Diff)로 누수 패턴 감지

public class MemorySnapshotHelper : MonoBehaviour
{
    // 디버그 메뉴에서 호출하여 메모리 상태 기록
    [ContextMenu("Log Memory State")]
    void LogMemoryState()
    {
        // Managed Memory
        long gcTotal = GC.GetTotalMemory(false);

        // Native Memory (Unity)
        long totalReserved = Profiler.GetTotalReservedMemoryLong();
        long totalAllocated = Profiler.GetTotalAllocatedMemoryLong();
        long unused = Profiler.GetTotalUnusedReservedMemoryLong();

        // Mono/IL2CPP Heap
        long monoHeap = Profiler.GetMonoHeapSizeLong();
        long monoUsed = Profiler.GetMonoUsedSizeLong();
        float monoFragmentation = 1f - (float)monoUsed / monoHeap;

        Debug.Log("=== Memory Snapshot ===");
        Debug.Log($"GC Total: {gcTotal / (1024f * 1024f):F2} MB");
        Debug.Log($"Native Reserved: {totalReserved / (1024f * 1024f):F2} MB");
        Debug.Log($"Native Allocated: {totalAllocated / (1024f * 1024f):F2} MB");
        Debug.Log($"Native Unused: {unused / (1024f * 1024f):F2} MB");
        Debug.Log($"Mono Heap: {monoHeap / (1024f * 1024f):F2} MB");
        Debug.Log($"Mono Used: {monoUsed / (1024f * 1024f):F2} MB");
        Debug.Log($"Mono Fragmentation: {monoFragmentation:P1}");
        Debug.Log("========================");
    }

    // =============================================
    // 프레임별 GC 할당 추적 도구
    // =============================================

    #if DEVELOPMENT_BUILD || UNITY_EDITOR
    private int _frameCount;
    private long _totalGCAllocThisSecond;
    private float _lastReportTime;

    void LateUpdate()
    {
        _frameCount++;

        if (Time.realtimeSinceStartup - _lastReportTime >= 1f)
        {
            if (_totalGCAllocThisSecond > 1024 * 10) // 10KB/s 이상
            {
                Debug.LogWarning(
                    $"[Memory] GC Alloc: {_totalGCAllocThisSecond / 1024f:F1} KB/s " +
                    $"({_frameCount} frames)");
            }

            _totalGCAllocThisSecond = 0;
            _frameCount = 0;
            _lastReportTime = Time.realtimeSinceStartup;
        }
    }
    #endif
}
```

---

## 종합 실전 예제: 제로 할당 게임 루프

```csharp
using System;
using System.Buffers;
using Unity.Collections;
using UnityEngine;
using UnityEngine.Pool;
using Cysharp.Threading.Tasks;

/// <summary>
/// 매 프레임 GC 할당을 최소화한 게임 매니저 예제.
/// 풀링, NativeArray, Span, ArrayPool을 종합 활용합니다.
/// </summary>
public class ZeroAllocGameLoopExample : MonoBehaviour
{
    // =============================================
    // 필드: 모든 버퍼와 풀을 미리 할당
    // =============================================

    [SerializeField] private int _maxEnemies = 1000;

    // NativeArray - GC 프리 데이터 저장소
    private NativeArray<Vector3> _positions;
    private NativeArray<float> _healths;
    private NativeArray<byte> _states; // 0: inactive, 1: active, 2: dead

    // 재사용 버퍼
    private RaycastHit[] _hitBuffer;
    private Collider[] _overlapBuffer;
    private readonly StringBuilder _debugSb = new(256);

    // 오브젝트 풀
    private ObjectPool<DamageEvent> _damageEventPool;

    // ArrayPool 렌탈
    private float[] _distanceBuffer;

    void Awake()
    {
        // 초기화 시 모든 메모리 미리 할당
        _positions = new NativeArray<Vector3>(_maxEnemies, Allocator.Persistent);
        _healths = new NativeArray<float>(_maxEnemies, Allocator.Persistent);
        _states = new NativeArray<byte>(_maxEnemies, Allocator.Persistent);

        _hitBuffer = new RaycastHit[32];
        _overlapBuffer = new Collider[64];

        _damageEventPool = new ObjectPool<DamageEvent>(
            createFunc: () => new DamageEvent(),
            actionOnGet: e => e.Reset(),
            actionOnRelease: e => { },
            defaultCapacity: 50,
            maxSize: 200
        );
    }

    // =============================================
    // Update: 제로 할당 메인 루프
    // =============================================

    void Update()
    {
        float dt = Time.deltaTime;

        // 1. 물리 체크 (NonAlloc API 사용)
        int hitCount = Physics.RaycastNonAlloc(
            Camera.main.transform.position,
            Camera.main.transform.forward,
            _hitBuffer, 100f);

        if (hitCount > 0)
        {
            ProcessHitsNoAlloc(_hitBuffer.AsSpan(0, hitCount));
        }

        // 2. 적 업데이트 (NativeArray 직접 접근)
        for (int i = 0; i < _maxEnemies; i++)
        {
            if (_states[i] != 1) continue; // active만 처리

            // NativeArray에 직접 쓰기
            var pos = _positions[i];
            pos += Vector3.forward * dt;
            _positions[i] = pos;

            // 체력 감소
            float hp = _healths[i];
            hp -= dt; // 예시
            _healths[i] = hp;

            if (hp <= 0)
            {
                _states[i] = 2; // dead
            }
        }

        // 3. 임시 계산에 ArrayPool 사용
        _distanceBuffer = ArrayPool<float>.Shared.Rent(_maxEnemies);
        try
        {
            CalculateDistances(_distanceBuffer, _maxEnemies);
        }
        finally
        {
            ArrayPool<float>.Shared.Return(_distanceBuffer);
            _distanceBuffer = null;
        }
    }

    void ProcessHitsNoAlloc(ReadOnlySpan<RaycastHit> hits)
    {
        for (int i = 0; i < hits.Length; i++)
        {
            // 풀에서 이벤트 가져오기 (할당 없음)
            var evt = _damageEventPool.Get();
            evt.Position = hits[i].point;
            evt.Damage = 10f;

            // 처리 후 반환
            ProcessDamageEvent(evt);
            _damageEventPool.Release(evt);
        }
    }

    void CalculateDistances(float[] buffer, int count)
    {
        Vector3 playerPos = transform.position;
        for (int i = 0; i < count; i++)
        {
            if (_states[i] != 1) { buffer[i] = float.MaxValue; continue; }
            buffer[i] = Vector3.Distance(playerPos, _positions[i]);
        }
    }

    void ProcessDamageEvent(DamageEvent evt) { /* ... */ }

    void OnDestroy()
    {
        // NativeArray 해제 필수
        if (_positions.IsCreated) _positions.Dispose();
        if (_healths.IsCreated) _healths.Dispose();
        if (_states.IsCreated) _states.Dispose();
    }
}

class DamageEvent
{
    public Vector3 Position;
    public float Damage;
    public int TargetId;

    public void Reset()
    {
        Position = Vector3.zero;
        Damage = 0;
        TargetId = -1;
    }
}
```

---

## 주의사항

1. **NativeArray Dispose 누락**: `Allocator.Persistent`로 생성한 NativeArray를 Dispose하지 않으면 네이티브 메모리 누수가 발생합니다. `OnDestroy`에서 반드시 `IsCreated` 체크 후 `Dispose`하세요.

2. **ArrayPool 반환 누수**: `ArrayPool<T>.Shared.Rent()`로 빌린 배열을 반환하지 않으면 풀이 고갈됩니다. `try/finally` 패턴을 반드시 사용하세요.

3. **stackalloc 스택 오버플로우**: 큰 크기의 `stackalloc`은 `StackOverflowException`을 유발합니다. 1KB 이하에서만 사용하고, 가변 크기에는 조건부 fallback 패턴을 적용하세요.

4. **ValueTask 이중 소비**: `ValueTask`는 한 번만 await할 수 있습니다. 결과를 여러 번 사용해야 하면 `.AsTask()`로 변환하거나 결과를 캐싱하세요.

5. **GC.Collect 남용**: 수동 GC 호출은 full collection을 트리거하여 오히려 성능을 악화시킬 수 있습니다. 장면 전환, 로딩 화면 등 사용자가 대기를 기대하는 시점에서만 호출하세요.

6. **Span의 제한**: `Span<T>`는 `ref struct`이므로 클래스 필드로 저장하거나, `async` 메서드 내에서 `await` 경계를 넘겨 사용할 수 없습니다. 비동기 코드에서는 `Memory<T>`를 사용하세요.

7. **GC 비활성화 위험**: `GarbageCollector.GCMode = Mode.Disabled`는 메모리가 계속 증가합니다. 반드시 짧은 구간에서만 사용하고 즉시 재활성화하세요.

8. **Object Pool 과다 보유**: 풀 크기 제한 없이 객체를 보관하면 메모리가 낭비됩니다. 항상 `maxSize`를 설정하고 사용 패턴을 모니터링하세요.

---

## 베스트 프랙티스

### 메모리 할당 최소화 원칙

| 원칙 | 실천 방법 |
|------|-----------|
| **측정 먼저** | Profiler로 실제 할당 지점 확인 후 최적화 |
| **핫 패스 우선** | Update, FixedUpdate 등 매 프레임 호출 경로 우선 최적화 |
| **풀링 활용** | 자주 생성/파괴되는 객체는 반드시 풀링 |
| **값 타입 선호** | 가능하면 struct, NativeArray, Span 사용 |
| **캐싱** | 반복 사용되는 결과는 필드에 캐싱 |

### 상황별 권장 패턴

```
비동기 반환 타입 선택:
├── Unity 전용 프로젝트      → UniTask (최소 할당)
├── 동기 완료가 잦은 경우     → ValueTask
├── 여러 번 await 필요       → Task
└── fire-and-forget         → UniTask.Void / UniTaskVoid

메모리 할당 전략:
├── 작은 임시 버퍼 (< 1KB)   → stackalloc + Span
├── 중간 임시 버퍼 (1KB~1MB) → ArrayPool<T>.Shared.Rent()
├── 대용량 장기 데이터        → NativeArray (Persistent)
├── 반복 생성 객체           → ObjectPool<T>
└── 비동기 슬라이스           → Memory<T>

문자열 처리:
├── 단순 연결               → string.Concat 또는 $"" (소수 인자)
├── 루프 내 연결             → StringBuilder (재사용)
├── 조건부 로깅              → Interpolated String Handler
├── 성능 임계 구간           → Span<char> + stackalloc
└── UI 텍스트               → TMP_Text.SetText(StringBuilder)

GC 관리 전략:
├── 일반 게임플레이           → Incremental GC 활성화
├── 장면 전환 시             → GC.Collect() + Resources.UnloadUnusedAssets()
├── 중요 구간 (보스전 등)     → GC 비활성화 (짧은 구간만)
└── 개발 중                  → Memory Profiler 스냅샷 비교
```

### 체크리스트

- [ ] Update/FixedUpdate에서 `new` 키워드 사용하지 않는가?
- [ ] LINQ를 핫 패스에서 사용하지 않는가?
- [ ] 문자열 보간을 매 프레임 하지 않는가?
- [ ] 람다에서 불필요한 변수를 캡처하지 않는가?
- [ ] NativeArray를 모두 Dispose하는가?
- [ ] ArrayPool에서 빌린 배열을 모두 반환하는가?
- [ ] 비동기 반환 타입으로 UniTask를 사용하는가?
- [ ] 오브젝트 풀의 maxSize를 설정했는가?
- [ ] Profiler에서 GC Alloc이 0에 가까운가?
- [ ] Incremental GC를 활성화했는가?

---

## 참고 자료

- [Unity Manual - Memory in Unity](https://docs.unity3d.com/Manual/performance-memory-overview.html)
- [Unity Manual - Garbage Collection](https://docs.unity3d.com/Manual/performance-garbage-collector.html)
- [Unity Memory Profiler](https://docs.unity3d.com/Packages/com.unity.memoryprofiler@latest)
- [UniTask - Zero Allocation async/await](https://github.com/Cysharp/UniTask)
- [Microsoft - Memory and Span](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/)
- [Microsoft - ArrayPool\<T\>](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1)
- [Microsoft - ObjectPool](https://docs.unity3d.com/ScriptReference/Pool.ObjectPool_1.html)
- [Unity Blog - Incremental Garbage Collection](https://blog.unity.com/technology/feature-preview-incremental-garbage-collection)
