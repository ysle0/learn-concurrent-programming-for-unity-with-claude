# 22. Job System

## 개요

**Unity Job System**은 **멀티스레드 코드를 안전하게 작성**하기 위한 프레임워크입니다. 워커 스레드에서 작업을 실행하면서도 경쟁 조건(Race Condition)을 컴파일 시점에 방지합니다.

---

## 설치

### Unity Package Manager

```
// Unity 2020.1+ 기본 포함
// 또는 명시적 설치
com.unity.jobs
com.unity.collections (NativeContainer용)
com.unity.burst (선택, 성능 최적화)
```

---

## 기본 개념

### 왜 Job System인가?

```csharp
// ❌ 일반 스레드: 위험
public class UnsafeThreading : MonoBehaviour
{
    private List<int> _data = new List<int>();

    void Start()
    {
        // 경쟁 조건 발생 가능
        Task.Run(() =>
        {
            for (int i = 0; i < 1000; i++)
            {
                _data.Add(i); // 위험!
            }
        });

        Task.Run(() =>
        {
            foreach (var item in _data) // 위험!
            {
                Debug.Log(item);
            }
        });
    }
}

// ✅ Job System: 안전
// - 컴파일 시점 안전성 검사
// - 자동 워커 스레드 관리
// - Burst 컴파일러 호환
```

### Job System 아키텍처

```
Main Thread
    │
    ├── Job 스케줄링
    │       ↓
    │   ┌─────────────────────────┐
    │   │     Job Scheduler       │
    │   └─────────────────────────┘
    │           ↓
    │   ┌─────┬─────┬─────┬─────┐
    │   │ W1  │ W2  │ W3  │ W4  │  Worker Threads
    │   └─────┴─────┴─────┴─────┘
    │           ↓
    └── Job 완료 대기 (Complete)
```

---

## IJob 인터페이스

### 기본 Job

```csharp
using Unity.Jobs;
using Unity.Collections;

public struct SimpleJob : IJob
{
    // 입력/출력 데이터는 NativeContainer 사용
    public NativeArray<int> Data;
    public int Multiplier;

    public void Execute()
    {
        for (int i = 0; i < Data.Length; i++)
        {
            Data[i] *= Multiplier;
        }
    }
}

public class SimpleJobExample : MonoBehaviour
{
    void Start()
    {
        // 1. NativeArray 생성
        var data = new NativeArray<int>(100, Allocator.TempJob);
        for (int i = 0; i < data.Length; i++)
        {
            data[i] = i;
        }

        // 2. Job 생성
        var job = new SimpleJob
        {
            Data = data,
            Multiplier = 2
        };

        // 3. Job 스케줄링
        JobHandle handle = job.Schedule();

        // 4. Job 완료 대기
        handle.Complete();

        // 5. 결과 사용
        Debug.Log($"Result: {data[50]}"); // 100

        // 6. 메모리 해제
        data.Dispose();
    }
}
```

### 읽기 전용 데이터

```csharp
public struct ReadOnlyJob : IJob
{
    [ReadOnly] // 읽기 전용 표시 - 병렬 읽기 허용
    public NativeArray<float> InputData;

    public NativeArray<float> OutputData;

    public void Execute()
    {
        for (int i = 0; i < InputData.Length; i++)
        {
            OutputData[i] = InputData[i] * 2f;
        }
    }
}
```

---

## IJobParallelFor

### 병렬 처리

```csharp
using Unity.Jobs;
using Unity.Collections;
using Unity.Burst;

[BurstCompile]
public struct ParallelJob : IJobParallelFor
{
    [ReadOnly]
    public NativeArray<float> Input;

    [WriteOnly]
    public NativeArray<float> Output;

    public float Multiplier;

    public void Execute(int index)
    {
        // 각 인덱스가 병렬로 실행됨
        Output[index] = Input[index] * Multiplier;
    }
}

public class ParallelJobExample : MonoBehaviour
{
    void Start()
    {
        int count = 10000;

        var input = new NativeArray<float>(count, Allocator.TempJob);
        var output = new NativeArray<float>(count, Allocator.TempJob);

        // 입력 데이터 설정
        for (int i = 0; i < count; i++)
        {
            input[i] = i;
        }

        var job = new ParallelJob
        {
            Input = input,
            Output = output,
            Multiplier = 2f
        };

        // innerloopBatchCount: 각 워커가 처리할 배치 크기
        JobHandle handle = job.Schedule(count, 64);
        handle.Complete();

        Debug.Log($"Result: {output[5000]}"); // 10000

        input.Dispose();
        output.Dispose();
    }
}
```

### 배치 크기 선택

```csharp
// 배치 크기 가이드라인
// - 작업이 가벼우면: 큰 배치 (64~256)
// - 작업이 무거우면: 작은 배치 (1~16)
// - 기본 권장: 32~64

job.Schedule(count, 64);  // 일반적인 경우
job.Schedule(count, 1);   // 무거운 작업
job.Schedule(count, 256); // 매우 가벼운 작업
```

---

## IJobParallelForTransform

### Transform 병렬 처리

```csharp
using Unity.Jobs;
using Unity.Collections;
using UnityEngine.Jobs;
using Unity.Mathematics;

public struct TransformJob : IJobParallelForTransform
{
    public float DeltaTime;
    public float Speed;

    public void Execute(int index, TransformAccess transform)
    {
        // Transform에 직접 접근 (Main Thread 제한 우회)
        var position = transform.position;
        position += Vector3.forward * Speed * DeltaTime;
        transform.position = position;
    }
}

public class TransformJobExample : MonoBehaviour
{
    [SerializeField] private Transform[] _objects;
    private TransformAccessArray _transformAccessArray;

    void Start()
    {
        // TransformAccessArray 생성
        _transformAccessArray = new TransformAccessArray(_objects);
    }

    void Update()
    {
        var job = new TransformJob
        {
            DeltaTime = Time.deltaTime,
            Speed = 5f
        };

        JobHandle handle = job.Schedule(_transformAccessArray);
        handle.Complete();
    }

    void OnDestroy()
    {
        _transformAccessArray.Dispose();
    }
}
```

---

## JobHandle과 의존성

### 의존성 체인

```csharp
public class JobDependencyExample : MonoBehaviour
{
    void Start()
    {
        var data = new NativeArray<int>(100, Allocator.TempJob);

        // Job 1: 초기화
        var initJob = new InitializeJob { Data = data };
        JobHandle initHandle = initJob.Schedule();

        // Job 2: 처리 (Job 1 완료 후)
        var processJob = new ProcessJob { Data = data };
        JobHandle processHandle = processJob.Schedule(initHandle);

        // Job 3: 최종화 (Job 2 완료 후)
        var finalizeJob = new FinalizeJob { Data = data };
        JobHandle finalHandle = finalizeJob.Schedule(processHandle);

        // 최종 Job만 Complete
        finalHandle.Complete();

        data.Dispose();
    }
}

public struct InitializeJob : IJob
{
    public NativeArray<int> Data;
    public void Execute()
    {
        for (int i = 0; i < Data.Length; i++)
            Data[i] = i;
    }
}

public struct ProcessJob : IJob
{
    public NativeArray<int> Data;
    public void Execute()
    {
        for (int i = 0; i < Data.Length; i++)
            Data[i] *= 2;
    }
}

public struct FinalizeJob : IJob
{
    public NativeArray<int> Data;
    public void Execute()
    {
        // 최종 처리
    }
}
```

### 병렬 의존성

```csharp
public class ParallelDependencyExample : MonoBehaviour
{
    void Start()
    {
        var data1 = new NativeArray<int>(100, Allocator.TempJob);
        var data2 = new NativeArray<int>(100, Allocator.TempJob);
        var combined = new NativeArray<int>(100, Allocator.TempJob);

        // 두 Job을 동시에 실행
        var job1 = new ProcessDataJob { Data = data1 };
        var job2 = new ProcessDataJob { Data = data2 };

        JobHandle handle1 = job1.Schedule();
        JobHandle handle2 = job2.Schedule();

        // 두 Job 모두 완료 후 실행
        JobHandle combinedDependency = JobHandle.CombineDependencies(handle1, handle2);

        var combineJob = new CombineJob
        {
            Input1 = data1,
            Input2 = data2,
            Output = combined
        };

        JobHandle finalHandle = combineJob.Schedule(combinedDependency);
        finalHandle.Complete();

        data1.Dispose();
        data2.Dispose();
        combined.Dispose();
    }
}

public struct ProcessDataJob : IJob
{
    public NativeArray<int> Data;
    public void Execute()
    {
        for (int i = 0; i < Data.Length; i++)
            Data[i] = i * 2;
    }
}

public struct CombineJob : IJob
{
    [ReadOnly] public NativeArray<int> Input1;
    [ReadOnly] public NativeArray<int> Input2;
    public NativeArray<int> Output;

    public void Execute()
    {
        for (int i = 0; i < Output.Length; i++)
            Output[i] = Input1[i] + Input2[i];
    }
}
```

---

## 안전성 시스템

### Safety Checks

```csharp
public class SafetyCheckExample : MonoBehaviour
{
    void Start()
    {
        var data = new NativeArray<int>(100, Allocator.TempJob);

        var job = new SimpleJob { Data = data };
        JobHandle handle = job.Schedule();

        // ❌ 컴파일 에러 또는 런타임 에러
        // Job이 완료되기 전에 데이터 접근 시도
        // Debug.Log(data[0]); // InvalidOperationException!

        handle.Complete();

        // ✅ Job 완료 후에는 안전
        Debug.Log(data[0]);

        data.Dispose();
    }
}
```

### DeallocateOnJobCompletion

```csharp
public struct AutoDisposeJob : IJob
{
    [DeallocateOnJobCompletion] // Job 완료 시 자동 해제
    public NativeArray<int> TempData;

    public NativeArray<int> Result;

    public void Execute()
    {
        for (int i = 0; i < TempData.Length; i++)
        {
            Result[i] = TempData[i] * 2;
        }
        // TempData는 자동으로 Dispose됨
    }
}
```

---

## Unity에서의 실전 활용

### 물리 시뮬레이션

```csharp
using Unity.Jobs;
using Unity.Collections;
using Unity.Mathematics;
using Unity.Burst;

[BurstCompile]
public struct GravityJob : IJobParallelFor
{
    [ReadOnly]
    public NativeArray<float3> Positions;

    [ReadOnly]
    public NativeArray<float> Masses;

    public NativeArray<float3> Forces;

    public float GravitationalConstant;

    public void Execute(int i)
    {
        float3 force = float3.zero;

        for (int j = 0; j < Positions.Length; j++)
        {
            if (i == j) continue;

            float3 direction = Positions[j] - Positions[i];
            float distance = math.length(direction);

            if (distance > 0.1f)
            {
                float magnitude = GravitationalConstant * Masses[i] * Masses[j]
                    / (distance * distance);
                force += math.normalize(direction) * magnitude;
            }
        }

        Forces[i] = force;
    }
}

public class GravitySimulation : MonoBehaviour
{
    [SerializeField] private Transform[] _bodies;

    private NativeArray<float3> _positions;
    private NativeArray<float> _masses;
    private NativeArray<float3> _velocities;
    private NativeArray<float3> _forces;

    void Start()
    {
        int count = _bodies.Length;

        _positions = new NativeArray<float3>(count, Allocator.Persistent);
        _masses = new NativeArray<float>(count, Allocator.Persistent);
        _velocities = new NativeArray<float3>(count, Allocator.Persistent);
        _forces = new NativeArray<float3>(count, Allocator.Persistent);

        for (int i = 0; i < count; i++)
        {
            _positions[i] = _bodies[i].position;
            _masses[i] = 1f;
            _velocities[i] = float3.zero;
        }
    }

    void FixedUpdate()
    {
        // 1. 힘 계산
        var gravityJob = new GravityJob
        {
            Positions = _positions,
            Masses = _masses,
            Forces = _forces,
            GravitationalConstant = 10f
        };

        JobHandle gravityHandle = gravityJob.Schedule(_bodies.Length, 32);

        // 2. 속도/위치 업데이트
        var moveJob = new IntegrateJob
        {
            Positions = _positions,
            Velocities = _velocities,
            Forces = _forces,
            Masses = _masses,
            DeltaTime = Time.fixedDeltaTime
        };

        JobHandle moveHandle = moveJob.Schedule(_bodies.Length, 64, gravityHandle);
        moveHandle.Complete();

        // 3. Transform 동기화
        for (int i = 0; i < _bodies.Length; i++)
        {
            _bodies[i].position = _positions[i];
        }
    }

    void OnDestroy()
    {
        _positions.Dispose();
        _masses.Dispose();
        _velocities.Dispose();
        _forces.Dispose();
    }
}

[BurstCompile]
public struct IntegrateJob : IJobParallelFor
{
    public NativeArray<float3> Positions;
    public NativeArray<float3> Velocities;

    [ReadOnly]
    public NativeArray<float3> Forces;

    [ReadOnly]
    public NativeArray<float> Masses;

    public float DeltaTime;

    public void Execute(int index)
    {
        float3 acceleration = Forces[index] / Masses[index];
        Velocities[index] += acceleration * DeltaTime;
        Positions[index] += Velocities[index] * DeltaTime;
    }
}
```

### 경로 탐색

```csharp
[BurstCompile]
public struct PathfindingJob : IJobParallelFor
{
    [ReadOnly]
    public NativeArray<float3> StartPositions;

    [ReadOnly]
    public NativeArray<float3> TargetPositions;

    [ReadOnly]
    public NativeArray<bool> Walkable; // 그리드 맵

    public int GridWidth;
    public int GridHeight;

    public NativeArray<int> PathLengths;

    public void Execute(int index)
    {
        // A* 또는 다른 경로 탐색 알고리즘
        // 각 에이전트별로 병렬 실행

        float3 start = StartPositions[index];
        float3 target = TargetPositions[index];

        // 간단한 직선 거리 (실제로는 A* 구현)
        PathLengths[index] = (int)math.distance(start, target);
    }
}
```

---

## 프레임 분산

### 비동기 Job

```csharp
public class AsyncJobExample : MonoBehaviour
{
    private NativeArray<float> _data;
    private JobHandle _jobHandle;
    private bool _jobScheduled;

    void Update()
    {
        if (!_jobScheduled)
        {
            // Job 스케줄링
            _data = new NativeArray<float>(1000000, Allocator.TempJob);

            var job = new HeavyComputeJob { Data = _data };
            _jobHandle = job.Schedule(_data.Length, 256);

            _jobScheduled = true;
        }
        else if (_jobHandle.IsCompleted)
        {
            // Job 완료 처리
            _jobHandle.Complete();

            Debug.Log($"Job completed! Result: {_data[0]}");

            _data.Dispose();
            _jobScheduled = false;
        }

        // Job 실행 중에도 다른 작업 가능
    }

    void OnDestroy()
    {
        if (_jobScheduled)
        {
            _jobHandle.Complete();
            _data.Dispose();
        }
    }
}

[BurstCompile]
public struct HeavyComputeJob : IJobParallelFor
{
    public NativeArray<float> Data;

    public void Execute(int index)
    {
        // 무거운 계산
        float result = 0;
        for (int i = 0; i < 1000; i++)
        {
            result += math.sin(index + i);
        }
        Data[index] = result;
    }
}
```

---

## 주의사항

### 블로킹 방지

```csharp
// ❌ 나쁨: Update에서 Complete 호출 (블로킹)
void Update()
{
    var job = new MyJob { ... };
    job.Schedule().Complete(); // 매 프레임 블로킹!
}

// ✅ 좋음: 비동기 패턴
private JobHandle _handle;

void Update()
{
    if (_handle.IsCompleted)
    {
        _handle.Complete();
        ProcessResults();

        // 새 Job 스케줄링
        _handle = new MyJob { ... }.Schedule();
    }
}
```

### 메모리 관리

```csharp
// ❌ Dispose 누락
void Bad()
{
    var data = new NativeArray<int>(100, Allocator.TempJob);
    var job = new MyJob { Data = data };
    job.Schedule().Complete();
    // data.Dispose() 누락!
}

// ✅ Dispose 보장
void Good()
{
    var data = new NativeArray<int>(100, Allocator.TempJob);
    try
    {
        var job = new MyJob { Data = data };
        job.Schedule().Complete();
    }
    finally
    {
        data.Dispose();
    }
}
```

---

## 정리

### Job System 요약

| Job 타입 | 용도 | 특징 |
|----------|------|------|
| **IJob** | 단일 작업 | 순차 실행 |
| **IJobParallelFor** | 배열 처리 | 인덱스별 병렬 |
| **IJobParallelForTransform** | Transform 처리 | Main Thread 제한 우회 |

### 체크리스트

- [ ] NativeContainer 사용하는가?
- [ ] [ReadOnly], [WriteOnly] 속성 적용했는가?
- [ ] Complete() 호출 시점 적절한가?
- [ ] Dispose() 보장하는가?
- [ ] Burst 적용했는가?

---

## 참고 자료

- [Unity Job System](https://docs.unity3d.com/Manual/JobSystem.html)
- [Job System Safety](https://docs.unity3d.com/Manual/JobSystemSafetySystem.html)
- [Job Types](https://docs.unity3d.com/Manual/JobSystemCreatingJobs.html)
