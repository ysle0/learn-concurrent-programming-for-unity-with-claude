# 24. NativeContainer

## 개요

**NativeContainer**는 Unity Job System에서 사용하는 **Unmanaged 메모리 컬렉션**입니다. C# 관리 힙 대신 네이티브 메모리를 사용하여 GC 오버헤드 없이 고성능 데이터 처리가 가능합니다.

---

## 설치

### Unity Package Manager

```
com.unity.collections
```

---

## Allocator 종류

### 메모리 할당 타입

```csharp
using Unity.Collections;

public class AllocatorExample : MonoBehaviour
{
    void Start()
    {
        // Temp: 1프레임 이내 사용 (가장 빠름)
        var tempArray = new NativeArray<int>(100, Allocator.Temp);
        // 프레임 끝에 자동 해제 (수동 Dispose 권장)

        // TempJob: Job 완료까지 사용
        var tempJobArray = new NativeArray<int>(100, Allocator.TempJob);
        // 4프레임 후 자동 해제

        // Persistent: 명시적 해제까지 유지
        var persistentArray = new NativeArray<int>(100, Allocator.Persistent);
        // 반드시 수동 Dispose 필요

        // 정리
        tempArray.Dispose();
        tempJobArray.Dispose();
        persistentArray.Dispose();
    }
}
```

### Allocator 선택 가이드

| Allocator | 수명 | 속도 | 용도 |
|-----------|------|------|------|
| **Temp** | 1 프레임 | 가장 빠름 | 임시 계산 |
| **TempJob** | ~4 프레임 | 빠름 | Job 데이터 |
| **Persistent** | 무제한 | 느림 | 장기 저장 |

---

## NativeArray<T>

### 기본 사용법

```csharp
using Unity.Collections;

public class NativeArrayExample : MonoBehaviour
{
    private NativeArray<float> _data;

    void Start()
    {
        // 생성
        _data = new NativeArray<float>(1000, Allocator.Persistent);

        // 값 설정
        for (int i = 0; i < _data.Length; i++)
        {
            _data[i] = i * 0.1f;
        }

        // 값 읽기
        float first = _data[0];
        float last = _data[^1]; // C# 8.0+ 인덱서

        // 배열로 복사
        float[] managedArray = _data.ToArray();

        // 관리 배열에서 복사
        float[] source = new float[] { 1, 2, 3, 4, 5 };
        var fromManaged = new NativeArray<float>(source, Allocator.Temp);
        fromManaged.Dispose();
    }

    void OnDestroy()
    {
        // 반드시 Dispose 호출
        if (_data.IsCreated)
        {
            _data.Dispose();
        }
    }
}
```

### Slice와 SubArray

```csharp
public class NativeSliceExample : MonoBehaviour
{
    void Start()
    {
        var array = new NativeArray<int>(100, Allocator.Temp);

        // Slice: 범위 참조 (복사 없음)
        NativeSlice<int> slice = array.Slice(10, 20);
        slice[0] = 999; // array[10] = 999

        // 전체 Slice
        NativeSlice<int> fullSlice = array.Slice();

        // SubArray: 읽기 전용 Slice
        // NativeArray<int>.ReadOnly readOnly = array.AsReadOnly();

        array.Dispose();
    }
}
```

---

## NativeList<T>

### 동적 배열

```csharp
using Unity.Collections;

public class NativeListExample : MonoBehaviour
{
    private NativeList<int> _list;

    void Start()
    {
        // 초기 용량으로 생성
        _list = new NativeList<int>(100, Allocator.Persistent);

        // 추가
        _list.Add(1);
        _list.Add(2);
        _list.Add(3);

        // 범위 추가
        _list.AddRange(new NativeArray<int>(new[] { 4, 5, 6 }, Allocator.Temp));

        // 접근
        int first = _list[0];
        int length = _list.Length;
        int capacity = _list.Capacity;

        // 삭제
        _list.RemoveAt(0);
        _list.RemoveAtSwapBack(0); // 빠른 삭제 (순서 무시)

        // 정리
        _list.Clear();

        Debug.Log($"Length: {_list.Length}");
    }

    void OnDestroy()
    {
        if (_list.IsCreated)
        {
            _list.Dispose();
        }
    }
}
```

### Job에서 NativeList 사용

```csharp
using Unity.Jobs;
using Unity.Collections;

// Job에서 NativeList 쓰기
public struct ListWriteJob : IJob
{
    public NativeList<int> List;

    public void Execute()
    {
        for (int i = 0; i < 100; i++)
        {
            List.Add(i);
        }
    }
}

// 병렬 쓰기는 ParallelWriter 사용
public struct ParallelListWriteJob : IJobParallelFor
{
    public NativeList<int>.ParallelWriter Writer;

    public void Execute(int index)
    {
        Writer.AddNoResize(index); // 용량 미리 확보 필요
    }
}

public class ListJobExample : MonoBehaviour
{
    void Start()
    {
        // 일반 Job
        var list1 = new NativeList<int>(Allocator.TempJob);
        var job1 = new ListWriteJob { List = list1 };
        job1.Schedule().Complete();
        Debug.Log($"List1 count: {list1.Length}");
        list1.Dispose();

        // 병렬 Job
        var list2 = new NativeList<int>(1000, Allocator.TempJob);
        list2.Capacity = 1000; // 용량 미리 확보

        var job2 = new ParallelListWriteJob { Writer = list2.AsParallelWriter() };
        job2.Schedule(1000, 64).Complete();
        Debug.Log($"List2 count: {list2.Length}");
        list2.Dispose();
    }
}
```

---

## NativeHashMap<TKey, TValue>

### 기본 사용법

```csharp
using Unity.Collections;

public class NativeHashMapExample : MonoBehaviour
{
    private NativeHashMap<int, float> _map;

    void Start()
    {
        // 초기 용량으로 생성
        _map = new NativeHashMap<int, float>(100, Allocator.Persistent);

        // 추가
        _map.Add(1, 1.5f);
        _map.Add(2, 2.5f);
        _map[3] = 3.5f; // 또는 인덱서

        // TryAdd: 이미 있으면 false 반환
        bool added = _map.TryAdd(1, 9.9f); // false

        // 조회
        if (_map.TryGetValue(1, out float value))
        {
            Debug.Log($"Key 1: {value}");
        }

        // ContainsKey
        bool hasKey = _map.ContainsKey(2);

        // 삭제
        _map.Remove(1);

        // 개수
        int count = _map.Count();

        // 모든 키/값 순회
        foreach (var kvp in _map)
        {
            Debug.Log($"{kvp.Key}: {kvp.Value}");
        }
    }

    void OnDestroy()
    {
        if (_map.IsCreated)
        {
            _map.Dispose();
        }
    }
}
```

### 병렬 쓰기

```csharp
public struct ParallelHashMapJob : IJobParallelFor
{
    public NativeHashMap<int, float>.ParallelWriter Writer;

    public void Execute(int index)
    {
        Writer.TryAdd(index, index * 1.5f);
    }
}

public class ParallelHashMapExample : MonoBehaviour
{
    void Start()
    {
        var map = new NativeHashMap<int, float>(1000, Allocator.TempJob);

        var job = new ParallelHashMapJob { Writer = map.AsParallelWriter() };
        job.Schedule(1000, 64).Complete();

        Debug.Log($"Map count: {map.Count()}");
        map.Dispose();
    }
}
```

---

## NativeMultiHashMap<TKey, TValue>

### 중복 키 지원

```csharp
using Unity.Collections;

public class NativeMultiHashMapExample : MonoBehaviour
{
    void Start()
    {
        // 하나의 키에 여러 값 저장
        var multiMap = new NativeMultiHashMap<int, string>(100, Allocator.Temp);

        // 같은 키로 여러 값 추가
        multiMap.Add(1, "A");
        multiMap.Add(1, "B");
        multiMap.Add(1, "C");
        multiMap.Add(2, "X");

        // 특정 키의 모든 값 순회
        if (multiMap.TryGetFirstValue(1, out string value, out var iterator))
        {
            do
            {
                Debug.Log($"Key 1: {value}");
            }
            while (multiMap.TryGetNextValue(out value, ref iterator));
        }

        // 특정 키의 값 개수
        int count = multiMap.CountValuesForKey(1);
        Debug.Log($"Key 1 has {count} values");

        multiMap.Dispose();
    }
}
```

---

## NativeHashSet<T>

### 고유 값 집합

```csharp
using Unity.Collections;

public class NativeHashSetExample : MonoBehaviour
{
    void Start()
    {
        var set = new NativeHashSet<int>(100, Allocator.Temp);

        // 추가
        set.Add(1);
        set.Add(2);
        set.Add(3);
        set.Add(1); // 무시됨 (중복)

        // 포함 여부
        bool contains = set.Contains(2); // true

        // 삭제
        set.Remove(2);

        // 개수
        int count = set.Count();

        // 순회
        foreach (int item in set)
        {
            Debug.Log($"Item: {item}");
        }

        set.Dispose();
    }
}
```

---

## NativeQueue<T>

### FIFO 큐

```csharp
using Unity.Collections;

public class NativeQueueExample : MonoBehaviour
{
    void Start()
    {
        var queue = new NativeQueue<int>(Allocator.Temp);

        // 추가
        queue.Enqueue(1);
        queue.Enqueue(2);
        queue.Enqueue(3);

        // 꺼내기
        if (queue.TryDequeue(out int value))
        {
            Debug.Log($"Dequeued: {value}"); // 1
        }

        // 확인만 (제거 안 함)
        if (queue.TryPeek(out int peeked))
        {
            Debug.Log($"Peeked: {peeked}"); // 2
        }

        // 개수
        int count = queue.Count;

        queue.Dispose();
    }
}

// 병렬 쓰기
public struct ParallelQueueJob : IJobParallelFor
{
    public NativeQueue<int>.ParallelWriter Writer;

    public void Execute(int index)
    {
        Writer.Enqueue(index);
    }
}
```

---

## NativeStream

### 고성능 병렬 쓰기

```csharp
using Unity.Collections;
using Unity.Jobs;

public class NativeStreamExample : MonoBehaviour
{
    void Start()
    {
        int threadCount = 4;
        var stream = new NativeStream(threadCount, Allocator.TempJob);

        // 병렬 쓰기 Job
        var writeJob = new StreamWriteJob { Stream = stream.AsWriter() };
        var writeHandle = writeJob.Schedule(threadCount, 1);

        // 읽기 Job (쓰기 완료 후)
        var readJob = new StreamReadJob { Stream = stream.AsReader() };
        var readHandle = readJob.Schedule(writeHandle);

        readHandle.Complete();

        stream.Dispose();
    }
}

public struct StreamWriteJob : IJobParallelFor
{
    public NativeStream.Writer Stream;

    public void Execute(int index)
    {
        Stream.BeginForEachIndex(index);

        // 여러 값 쓰기
        for (int i = 0; i < 10; i++)
        {
            Stream.Write(index * 100 + i);
        }

        Stream.EndForEachIndex();
    }
}

public struct StreamReadJob : IJob
{
    public NativeStream.Reader Stream;

    public void Execute()
    {
        int total = 0;

        for (int index = 0; index < Stream.ForEachCount; index++)
        {
            int count = Stream.BeginForEachIndex(index);

            for (int i = 0; i < count; i++)
            {
                int value = Stream.Read<int>();
                total += value;
            }

            Stream.EndForEachIndex();
        }

        // Debug.Log($"Total: {total}");
    }
}
```

---

## 안전성 속성

### [ReadOnly], [WriteOnly]

```csharp
using Unity.Collections;
using Unity.Jobs;

public struct SafetyAttributeJob : IJobParallelFor
{
    [ReadOnly]
    public NativeArray<float> Input; // 병렬 읽기 허용

    [WriteOnly]
    public NativeArray<float> Output; // 쓰기만 (읽기 불가)

    // [NativeDisableParallelForRestriction]
    // public NativeArray<float> Shared; // 병렬 제한 해제 (위험!)

    public void Execute(int index)
    {
        Output[index] = Input[index] * 2f;
    }
}
```

### [NativeDisableContainerSafetyRestriction]

```csharp
// 안전성 검사 비활성화 (주의 필요)
public struct UnsafeJob : IJob
{
    [NativeDisableContainerSafetyRestriction]
    public NativeArray<int> Data;

    public void Execute()
    {
        // 안전성 검사 없이 접근
        // 경쟁 조건 발생 가능!
    }
}
```

---

## 실전 패턴

### 패턴 1: 공간 해시맵

```csharp
// 공간 분할을 위한 해시맵
public class SpatialHashMap : MonoBehaviour
{
    private NativeMultiHashMap<int, int> _spatialHash;
    private NativeArray<float3> _positions;

    private float _cellSize = 10f;

    void Start()
    {
        int entityCount = 10000;
        _positions = new NativeArray<float3>(entityCount, Allocator.Persistent);
        _spatialHash = new NativeMultiHashMap<int, int>(entityCount * 4, Allocator.Persistent);

        // 위치 초기화
        for (int i = 0; i < entityCount; i++)
        {
            _positions[i] = new float3(
                UnityEngine.Random.Range(-100f, 100f),
                0,
                UnityEngine.Random.Range(-100f, 100f));
        }
    }

    void Update()
    {
        // 해시맵 재구성
        _spatialHash.Clear();

        new BuildSpatialHashJob
        {
            Positions = _positions,
            SpatialHash = _spatialHash.AsParallelWriter(),
            CellSize = _cellSize
        }.Schedule(_positions.Length, 64).Complete();
    }

    int GetCellHash(float3 position)
    {
        int x = (int)math.floor(position.x / _cellSize);
        int z = (int)math.floor(position.z / _cellSize);
        return x * 73856093 ^ z * 19349663; // 해시 함수
    }

    void OnDestroy()
    {
        _positions.Dispose();
        _spatialHash.Dispose();
    }
}

[BurstCompile]
public struct BuildSpatialHashJob : IJobParallelFor
{
    [ReadOnly]
    public NativeArray<float3> Positions;

    public NativeMultiHashMap<int, int>.ParallelWriter SpatialHash;

    public float CellSize;

    public void Execute(int index)
    {
        float3 pos = Positions[index];
        int hash = GetCellHash(pos);
        SpatialHash.Add(hash, index);
    }

    int GetCellHash(float3 position)
    {
        int x = (int)math.floor(position.x / CellSize);
        int z = (int)math.floor(position.z / CellSize);
        return x * 73856093 ^ z * 19349663;
    }
}
```

### 패턴 2: 더블 버퍼링

```csharp
public class DoubleBuffering : MonoBehaviour
{
    private NativeArray<float3> _positionsA;
    private NativeArray<float3> _positionsB;
    private bool _useA = true;

    void Start()
    {
        int count = 10000;
        _positionsA = new NativeArray<float3>(count, Allocator.Persistent);
        _positionsB = new NativeArray<float3>(count, Allocator.Persistent);
    }

    void Update()
    {
        var readBuffer = _useA ? _positionsA : _positionsB;
        var writeBuffer = _useA ? _positionsB : _positionsA;

        new UpdatePositionJob
        {
            Input = readBuffer,
            Output = writeBuffer,
            DeltaTime = Time.deltaTime
        }.Schedule(readBuffer.Length, 64).Complete();

        _useA = !_useA; // 버퍼 스왑
    }

    void OnDestroy()
    {
        _positionsA.Dispose();
        _positionsB.Dispose();
    }
}

[BurstCompile]
public struct UpdatePositionJob : IJobParallelFor
{
    [ReadOnly]
    public NativeArray<float3> Input;

    [WriteOnly]
    public NativeArray<float3> Output;

    public float DeltaTime;

    public void Execute(int index)
    {
        Output[index] = Input[index] + new float3(0, 0, 1) * DeltaTime;
    }
}
```

---

## 메모리 관리

### Dispose 패턴

```csharp
public class DisposableComponent : MonoBehaviour, System.IDisposable
{
    private NativeArray<int> _array;
    private NativeList<float> _list;
    private bool _disposed;

    void Awake()
    {
        _array = new NativeArray<int>(100, Allocator.Persistent);
        _list = new NativeList<float>(100, Allocator.Persistent);
    }

    public void Dispose()
    {
        if (_disposed) return;

        if (_array.IsCreated) _array.Dispose();
        if (_list.IsCreated) _list.Dispose();

        _disposed = true;
    }

    void OnDestroy()
    {
        Dispose();
    }
}
```

### 조건부 Dispose

```csharp
void SafeDispose()
{
    // IsCreated 체크 후 Dispose
    if (_array.IsCreated)
    {
        _array.Dispose();
    }

    // null 체크 (NativeContainer는 struct이므로 불필요하지만 안전을 위해)
}
```

---

## 정리

### NativeContainer 요약

| 컨테이너 | 용도 | 특징 |
|----------|------|------|
| **NativeArray** | 고정 배열 | 가장 빠름 |
| **NativeList** | 동적 배열 | Add 지원 |
| **NativeHashMap** | 키-값 | 빠른 조회 |
| **NativeMultiHashMap** | 중복 키 | 그룹화 |
| **NativeHashSet** | 고유 값 | 중복 방지 |
| **NativeQueue** | FIFO 큐 | 순서 보장 |
| **NativeStream** | 병렬 쓰기 | 고성능 |

### 체크리스트

- [ ] 적절한 Allocator 선택했는가?
- [ ] Dispose 호출 보장하는가?
- [ ] [ReadOnly]/[WriteOnly] 적용했는가?
- [ ] 병렬 쓰기 시 ParallelWriter 사용하는가?

---

## 참고 자료

- [Unity.Collections Documentation](https://docs.unity3d.com/Packages/com.unity.collections@latest)
- [NativeContainer Safety](https://docs.unity3d.com/Manual/JobSystemNativeContainer.html)
- [Custom NativeContainers](https://docs.unity3d.com/Packages/com.unity.collections@latest/manual/custom-containers.html)
