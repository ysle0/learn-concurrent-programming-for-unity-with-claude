# 18. Parallel 클래스

## 개요

`System.Threading.Tasks.Parallel` 클래스는 **데이터 병렬 처리**를 위한 API를 제공합니다. 컬렉션의 각 요소를 여러 스레드에서 동시에 처리하여 CPU 집약적 작업의 성능을 크게 향상시킬 수 있습니다.

---

## 기본 개념

### 병렬 처리란?

```
순차 처리:
[작업1] -> [작업2] -> [작업3] -> [작업4]
총 시간: 4초 (1초 × 4)

병렬 처리 (4 코어):
[작업1]
[작업2]  -> 동시 실행
[작업3]
[작업4]
총 시간: 1초
```

### Unity에서의 주의사항

```csharp
public class ParallelInUnity : MonoBehaviour
{
    void Start()
    {
        // ⚠️ 중요: Parallel은 ThreadPool 스레드 사용
        // Unity API는 Main Thread에서만 호출 가능!

        // ❌ 잘못된 사용
        Parallel.For(0, 10, i =>
        {
            // 이 코드는 ThreadPool에서 실행됨
            // transform.position = Vector3.zero; // 런타임 에러!
        });

        // ✅ 올바른 사용: 순수 계산만 수행
        var results = new float[100];
        Parallel.For(0, 100, i =>
        {
            results[i] = Mathf.Sin(i * 0.1f) * Mathf.Cos(i * 0.2f);
        });

        // Main Thread에서 결과 사용
        Debug.Log($"Results: {results[0]}, {results[50]}, {results[99]}");
    }
}
```

---

## Parallel.For

### 기본 사용법

```csharp
using System.Threading.Tasks;

public class ParallelForBasics : MonoBehaviour
{
    void Start()
    {
        int[] data = new int[1000];

        // 순차 처리
        for (int i = 0; i < data.Length; i++)
        {
            data[i] = ComputeValue(i);
        }

        // 병렬 처리
        Parallel.For(0, data.Length, i =>
        {
            data[i] = ComputeValue(i);
        });
    }

    int ComputeValue(int index)
    {
        // CPU 집약적 계산 시뮬레이션
        double result = 0;
        for (int j = 0; j < 1000; j++)
        {
            result += Math.Sin(index + j);
        }
        return (int)result;
    }
}
```

### ParallelLoopResult

```csharp
public class ParallelLoopResultExample : MonoBehaviour
{
    void Start()
    {
        var result = Parallel.For(0, 100, (i, loopState) =>
        {
            if (i == 50)
            {
                loopState.Break(); // 50 이후 새 반복 시작 안 함
            }

            // 처리
            Debug.Log($"Processing {i}");
        });

        Debug.Log($"IsCompleted: {result.IsCompleted}");
        Debug.Log($"LowestBreakIteration: {result.LowestBreakIteration}");
    }
}
```

### 로컬 상태 사용

```csharp
public class ParallelForLocalState : MonoBehaviour
{
    void Start()
    {
        long totalSum = 0;

        // Thread-local 상태로 경쟁 상태 방지
        Parallel.For(0, 1000,
            // 초기화: 각 스레드의 로컬 합계
            () => 0L,
            // 본문: 로컬 합계에 추가
            (i, loopState, localSum) =>
            {
                return localSum + ComputeValue(i);
            },
            // 최종화: 로컬 합계를 전역 합계에 추가
            localSum =>
            {
                Interlocked.Add(ref totalSum, localSum);
            });

        Debug.Log($"Total Sum: {totalSum}");
    }

    long ComputeValue(int i) => (long)i * i;
}
```

---

## Parallel.ForEach

### 기본 사용법

```csharp
public class ParallelForEachBasics : MonoBehaviour
{
    void Start()
    {
        var items = new List<string> { "A", "B", "C", "D", "E" };

        // 순차 처리
        foreach (var item in items)
        {
            ProcessItem(item);
        }

        // 병렬 처리
        Parallel.ForEach(items, item =>
        {
            ProcessItem(item);
        });
    }

    void ProcessItem(string item)
    {
        // 처리 로직
        Thread.Sleep(100); // 시뮬레이션
        Debug.Log($"Processed: {item} on Thread {Thread.CurrentThread.ManagedThreadId}");
    }
}
```

### 인덱스와 함께 사용

```csharp
public class ParallelForEachWithIndex : MonoBehaviour
{
    void Start()
    {
        var items = new List<string> { "A", "B", "C", "D", "E" };
        var results = new string[items.Count];

        Parallel.ForEach(items, (item, state, index) =>
        {
            results[index] = $"Processed_{item}";
        });

        foreach (var result in results)
        {
            Debug.Log(result);
        }
    }
}
```

### 파티셔닝

```csharp
public class ParallelPartitioning : MonoBehaviour
{
    void Start()
    {
        var data = Enumerable.Range(0, 10000).ToList();

        // Partitioner를 사용한 최적화
        var partitioner = Partitioner.Create(data, loadBalance: true);

        Parallel.ForEach(partitioner, (item, state) =>
        {
            // 부하 균형이 잡힌 파티션에서 처리
            ProcessItem(item);
        });
    }

    void ProcessItem(int item) { }
}
```

---

## Parallel.Invoke

### 여러 작업 동시 실행

```csharp
public class ParallelInvokeExample : MonoBehaviour
{
    private byte[] _textureData;
    private byte[] _audioData;
    private string _configData;

    void Start()
    {
        // 여러 독립적인 작업을 동시에 실행
        Parallel.Invoke(
            () => _textureData = LoadTextureData(),
            () => _audioData = LoadAudioData(),
            () => _configData = LoadConfigData()
        );

        // 모든 작업 완료 후 실행
        Debug.Log($"All data loaded: Texture={_textureData?.Length}, Audio={_audioData?.Length}");
    }

    byte[] LoadTextureData()
    {
        Thread.Sleep(500);
        return new byte[1024];
    }

    byte[] LoadAudioData()
    {
        Thread.Sleep(300);
        return new byte[512];
    }

    string LoadConfigData()
    {
        Thread.Sleep(200);
        return "config data";
    }
}
```

---

## ParallelOptions

### 병렬 처리 제어

```csharp
public class ParallelOptionsExample : MonoBehaviour
{
    private CancellationTokenSource _cts;

    void Start()
    {
        _cts = new CancellationTokenSource();

        var options = new ParallelOptions
        {
            // 최대 병렬 수준 (동시 실행 스레드 수)
            MaxDegreeOfParallelism = Environment.ProcessorCount / 2,

            // 취소 토큰
            CancellationToken = _cts.Token,

            // 태스크 스케줄러 (기본값 사용 권장)
            TaskScheduler = TaskScheduler.Default
        };

        try
        {
            Parallel.For(0, 1000, options, i =>
            {
                // 무거운 작업
                HeavyComputation(i);
            });
        }
        catch (OperationCanceledException)
        {
            Debug.Log("Parallel operation cancelled");
        }
    }

    void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }

    void HeavyComputation(int i)
    {
        // 계산...
    }
}
```

### 코어 수에 따른 병렬 수준 조정

```csharp
public class AdaptiveParallelism : MonoBehaviour
{
    void Start()
    {
        int coreCount = Environment.ProcessorCount;
        Debug.Log($"Available cores: {coreCount}");

        // 시나리오별 권장 설정
        var cpuBoundOptions = new ParallelOptions
        {
            // CPU 집약적: 코어 수만큼
            MaxDegreeOfParallelism = coreCount
        };

        var ioBoundOptions = new ParallelOptions
        {
            // I/O 집약적: 코어 수의 2배 이상
            MaxDegreeOfParallelism = coreCount * 2
        };

        var limitedOptions = new ParallelOptions
        {
            // 리소스 제한: 최대 4개
            MaxDegreeOfParallelism = Math.Min(coreCount, 4)
        };
    }
}
```

---

## Unity에서의 실전 활용

### 메시 데이터 처리

```csharp
public class ParallelMeshProcessing : MonoBehaviour
{
    void Start()
    {
        // 메시 정점 데이터 병렬 처리
        var mesh = GetComponent<MeshFilter>().mesh;
        var vertices = mesh.vertices;
        var normals = mesh.normals;

        // 정점 변환 (병렬)
        Parallel.For(0, vertices.Length, i =>
        {
            // 노이즈 적용 예시
            float noise = Mathf.PerlinNoise(vertices[i].x, vertices[i].z);
            vertices[i].y = noise * 2f;
        });

        // 법선 재계산 (Main Thread에서)
        mesh.vertices = vertices;
        mesh.RecalculateNormals();
    }
}
```

### 경로 탐색 병렬화

```csharp
public class ParallelPathfinding : MonoBehaviour
{
    [SerializeField] private Transform[] _targets;
    private Vector3[] _paths;

    void Start()
    {
        _paths = new Vector3[_targets.Length * 100]; // 경로 포인트

        var startPos = transform.position;
        var targetPositions = _targets.Select(t => t.position).ToArray();

        // 여러 목표에 대한 경로를 병렬로 계산
        Parallel.For(0, targetPositions.Length, i =>
        {
            var path = CalculatePath(startPos, targetPositions[i]);

            // 결과 저장
            for (int j = 0; j < path.Length; j++)
            {
                _paths[i * 100 + j] = path[j];
            }
        });

        Debug.Log("All paths calculated");
    }

    Vector3[] CalculatePath(Vector3 start, Vector3 end)
    {
        // A* 또는 다른 경로 탐색 알고리즘
        // Unity NavMesh는 Main Thread 필요 → 순수 계산만 병렬화
        var path = new Vector3[100];
        for (int i = 0; i < 100; i++)
        {
            path[i] = Vector3.Lerp(start, end, i / 99f);
        }
        return path;
    }
}
```

### 이미지 처리

```csharp
public class ParallelImageProcessing : MonoBehaviour
{
    [SerializeField] private Texture2D _sourceTexture;

    void Start()
    {
        var pixels = _sourceTexture.GetPixels32();
        int width = _sourceTexture.width;
        int height = _sourceTexture.height;

        // 그레이스케일 변환 (병렬)
        Parallel.For(0, height, y =>
        {
            for (int x = 0; x < width; x++)
            {
                int index = y * width + x;
                var pixel = pixels[index];

                // 그레이스케일 계산
                byte gray = (byte)(pixel.r * 0.299f + pixel.g * 0.587f + pixel.b * 0.114f);
                pixels[index] = new Color32(gray, gray, gray, pixel.a);
            }
        });

        // 결과 적용 (Main Thread)
        var resultTexture = new Texture2D(width, height);
        resultTexture.SetPixels32(pixels);
        resultTexture.Apply();

        GetComponent<Renderer>().material.mainTexture = resultTexture;
    }
}
```

### 물리 시뮬레이션 보조

```csharp
public class ParallelPhysicsHelper : MonoBehaviour
{
    [SerializeField] private Transform[] _objects;

    private Vector3[] _forces;
    private Vector3[] _positions;

    void FixedUpdate()
    {
        // 위치 캐시 (Main Thread)
        _positions = _objects.Select(o => o.position).ToArray();
        _forces = new Vector3[_objects.Length];

        // 힘 계산 (병렬) - N체 문제 시뮬레이션
        Parallel.For(0, _objects.Length, i =>
        {
            Vector3 totalForce = Vector3.zero;

            for (int j = 0; j < _objects.Length; j++)
            {
                if (i == j) continue;

                Vector3 direction = _positions[j] - _positions[i];
                float distance = direction.magnitude;

                if (distance > 0.1f)
                {
                    // 중력 공식
                    float forceMagnitude = 100f / (distance * distance);
                    totalForce += direction.normalized * forceMagnitude;
                }
            }

            _forces[i] = totalForce;
        });

        // 힘 적용 (Main Thread)
        for (int i = 0; i < _objects.Length; i++)
        {
            var rb = _objects[i].GetComponent<Rigidbody>();
            if (rb != null)
            {
                rb.AddForce(_forces[i]);
            }
        }
    }
}
```

---

## 성능 고려사항

### 오버헤드 vs 이득

```csharp
public class ParallelOverhead : MonoBehaviour
{
    void Start()
    {
        // ❌ 너무 작은 작업: 오버헤드가 이득보다 큼
        Parallel.For(0, 100, i =>
        {
            // 1마이크로초 작업 → 병렬화 오버헤드가 더 큼
            int x = i * 2;
        });

        // ✅ 충분히 큰 작업
        Parallel.For(0, 100, i =>
        {
            // 10밀리초+ 작업 → 병렬화 이득
            for (int j = 0; j < 100000; j++)
            {
                Math.Sin(j);
            }
        });
    }
}
```

### 작업 크기 권장

| 작업 시간 | 병렬화 | 이유 |
|----------|--------|------|
| < 1ms | ❌ | 오버헤드가 더 큼 |
| 1ms - 10ms | △ | 상황에 따라 |
| > 10ms | ✅ | 확실한 이득 |
| > 100ms | ✅✅ | 큰 이득 |

### 동기화 비용 최소화

```csharp
public class MinimizeSynchronization : MonoBehaviour
{
    void Start()
    {
        long sum = 0;
        object lockObj = new object();

        // ❌ 나쁨: 매 반복마다 락
        Parallel.For(0, 10000, i =>
        {
            lock (lockObj)
            {
                sum += i;
            }
        });

        // ✅ 좋음: 로컬 상태 사용
        sum = 0;
        Parallel.For(0, 10000,
            () => 0L,
            (i, state, local) => local + i,
            local => Interlocked.Add(ref sum, local));
    }
}
```

---

## Task.Run vs Parallel

### 차이점

```csharp
public class TaskRunVsParallel : MonoBehaviour
{
    async void Start()
    {
        // Task.Run: 단일 작업을 백그라운드에서
        await Task.Run(() =>
        {
            // 하나의 무거운 작업
            HeavyWork();
        });

        // Parallel: 데이터를 여러 스레드로 분할
        Parallel.For(0, 100, i =>
        {
            // 많은 데이터 항목을 병렬 처리
            ProcessItem(i);
        });

        // Task.WhenAll: 독립적인 여러 작업
        await Task.WhenAll(
            Task.Run(() => Work1()),
            Task.Run(() => Work2()),
            Task.Run(() => Work3())
        );

        // Parallel.Invoke: 위와 동일하지만 동기적
        Parallel.Invoke(
            () => Work1(),
            () => Work2(),
            () => Work3()
        );
    }

    void HeavyWork() { }
    void ProcessItem(int i) { }
    void Work1() { }
    void Work2() { }
    void Work3() { }
}
```

### 선택 가이드

| 시나리오 | 권장 |
|----------|------|
| 컬렉션 각 요소 처리 | Parallel.For/ForEach |
| 독립적인 여러 작업 | Parallel.Invoke 또는 Task.WhenAll |
| 비동기 작업 | Task.Run + await |
| 결과 필요 | Task.WhenAll |
| 동기적 완료 대기 | Parallel |

---

## 실전 패턴

### 패턴 1: 프레임 분산 처리

```csharp
public class FrameDistributedProcessing : MonoBehaviour
{
    private List<Enemy> _enemies = new List<Enemy>();
    private int _batchSize = 100;
    private int _currentBatch = 0;

    void Update()
    {
        // 한 프레임에 일부만 처리
        int startIndex = _currentBatch * _batchSize;
        int endIndex = Mathf.Min(startIndex + _batchSize, _enemies.Count);

        if (startIndex < _enemies.Count)
        {
            // 배치 내에서 병렬 처리
            Parallel.For(startIndex, endIndex, i =>
            {
                _enemies[i].CalculateAI();
            });

            _currentBatch++;
        }
        else
        {
            _currentBatch = 0;
        }
    }
}

public class Enemy
{
    public void CalculateAI() { /* AI 로직 */ }
}
```

### 패턴 2: 결과 수집

```csharp
public class ParallelResultCollection : MonoBehaviour
{
    void Start()
    {
        var inputs = Enumerable.Range(0, 1000).ToArray();
        var results = new ConcurrentBag<ProcessResult>();

        Parallel.ForEach(inputs, input =>
        {
            var result = Process(input);
            results.Add(result);
        });

        // 결과 정렬 (순서가 중요한 경우)
        var orderedResults = results.OrderBy(r => r.Index).ToList();
        Debug.Log($"Processed {orderedResults.Count} items");
    }

    ProcessResult Process(int input)
    {
        return new ProcessResult { Index = input, Value = input * 2 };
    }
}

public class ProcessResult
{
    public int Index { get; set; }
    public int Value { get; set; }
}
```

### 패턴 3: 취소 가능한 병렬 처리

```csharp
public class CancellableParallel : MonoBehaviour
{
    private CancellationTokenSource _cts;

    void Start()
    {
        _cts = new CancellationTokenSource();
        StartProcessing();
    }

    async void StartProcessing()
    {
        try
        {
            await Task.Run(() =>
            {
                var options = new ParallelOptions
                {
                    CancellationToken = _cts.Token,
                    MaxDegreeOfParallelism = 4
                };

                Parallel.For(0, 10000, options, (i, state) =>
                {
                    // 취소 확인
                    options.CancellationToken.ThrowIfCancellationRequested();

                    // 처리
                    Thread.Sleep(10);
                    Debug.Log($"Processed {i}");
                });
            });
        }
        catch (OperationCanceledException)
        {
            Debug.Log("Processing cancelled");
        }
    }

    void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }
}
```

---

## 주의사항

### Thread-Safety

```csharp
public class ThreadSafetyInParallel : MonoBehaviour
{
    void Start()
    {
        // ❌ 스레드 안전하지 않음
        var list = new List<int>();
        Parallel.For(0, 1000, i =>
        {
            list.Add(i); // 런타임 에러 가능!
        });

        // ✅ 스레드 안전한 컬렉션 사용
        var safeList = new ConcurrentBag<int>();
        Parallel.For(0, 1000, i =>
        {
            safeList.Add(i);
        });

        // ✅ 또는 미리 할당된 배열 사용
        var array = new int[1000];
        Parallel.For(0, 1000, i =>
        {
            array[i] = i; // 각 인덱스에 하나의 스레드만 접근
        });
    }
}
```

### Unity API 호출 금지

```csharp
public class NoUnityAPIInParallel : MonoBehaviour
{
    [SerializeField] private Transform[] _objects;

    void Start()
    {
        // ❌ Unity API 직접 호출
        // Parallel.For(0, _objects.Length, i =>
        // {
        //     _objects[i].position = Vector3.zero; // 에러!
        // });

        // ✅ 데이터만 병렬 처리
        var positions = new Vector3[_objects.Length];
        Parallel.For(0, _objects.Length, i =>
        {
            positions[i] = CalculateNewPosition(i);
        });

        // Main Thread에서 적용
        for (int i = 0; i < _objects.Length; i++)
        {
            _objects[i].position = positions[i];
        }
    }

    Vector3 CalculateNewPosition(int index)
    {
        return new Vector3(index, Mathf.Sin(index), 0);
    }
}
```

---

## 정리

### Parallel 요약

| 항목 | 내용 |
|------|------|
| **For** | 인덱스 기반 병렬 루프 |
| **ForEach** | 컬렉션 병렬 순회 |
| **Invoke** | 여러 액션 동시 실행 |
| **Options** | 병렬 수준, 취소, 스케줄러 제어 |

### 체크리스트

- [ ] 작업이 충분히 무거운가? (>10ms 권장)
- [ ] Unity API 호출이 없는가?
- [ ] 스레드 안전성 확보했는가?
- [ ] 적절한 병렬 수준 설정했는가?
- [ ] 취소 토큰 지원하는가?

---

## 참고 자료

- [Parallel Class - Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.parallel)
- [Data Parallelism](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/data-parallelism-task-parallel-library)
- [Custom Partitioners](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/custom-partitioners-for-plinq-and-tpl)
