# 19. PLINQ (Parallel LINQ)

## 개요

**PLINQ**(Parallel LINQ)는 LINQ 쿼리를 **자동으로 병렬화**하는 기술입니다. `AsParallel()` 호출 하나로 기존 LINQ 쿼리를 멀티코어에서 병렬 실행할 수 있습니다.

---

## 기본 사용법

### AsParallel()

```csharp
using System.Linq;
using UnityEngine;

public class PLINQBasics : MonoBehaviour
{
    void Start()
    {
        var numbers = Enumerable.Range(0, 10000).ToArray();

        // 순차 LINQ
        var sequentialResult = numbers
            .Where(n => n % 2 == 0)
            .Select(n => ComputeValue(n))
            .ToList();

        // 병렬 PLINQ - AsParallel() 추가만 하면 됨
        var parallelResult = numbers
            .AsParallel()
            .Where(n => n % 2 == 0)
            .Select(n => ComputeValue(n))
            .ToList();

        Debug.Log($"Results: {parallelResult.Count}");
    }

    int ComputeValue(int n)
    {
        // CPU 집약적 계산
        double result = 0;
        for (int i = 0; i < 1000; i++)
        {
            result += Math.Sin(n + i);
        }
        return (int)result;
    }
}
```

### 순서 유지

```csharp
public class PLINQOrdering : MonoBehaviour
{
    void Start()
    {
        var numbers = Enumerable.Range(0, 100).ToArray();

        // 순서 보장 안 됨 (더 빠름)
        var unordered = numbers
            .AsParallel()
            .Select(n => n * 2)
            .ToList();
        // 결과: [0, 4, 2, 8, 6, ...] - 순서 무작위

        // 순서 보장 (약간 느림)
        var ordered = numbers
            .AsParallel()
            .AsOrdered()
            .Select(n => n * 2)
            .ToList();
        // 결과: [0, 2, 4, 6, 8, ...] - 원래 순서

        Debug.Log($"Unordered first 5: {string.Join(", ", unordered.Take(5))}");
        Debug.Log($"Ordered first 5: {string.Join(", ", ordered.Take(5))}");
    }
}
```

---

## 병렬화 제어

### WithDegreeOfParallelism

```csharp
public class PLINQParallelismControl : MonoBehaviour
{
    void Start()
    {
        var data = Enumerable.Range(0, 1000);

        // 병렬 수준 제한
        var result = data
            .AsParallel()
            .WithDegreeOfParallelism(4) // 최대 4개 스레드
            .Select(n => ProcessItem(n))
            .ToList();

        // 코어 수에 맞게 설정
        var autoResult = data
            .AsParallel()
            .WithDegreeOfParallelism(Environment.ProcessorCount)
            .Select(n => ProcessItem(n))
            .ToList();
    }

    int ProcessItem(int n) => n * 2;
}
```

### WithExecutionMode

```csharp
public class PLINQExecutionMode : MonoBehaviour
{
    void Start()
    {
        var data = Enumerable.Range(0, 100);

        // 기본: 런타임이 병렬화 여부 결정
        var defaultMode = data
            .AsParallel()
            .WithExecutionMode(ParallelExecutionMode.Default)
            .Select(n => n * 2);

        // 강제 병렬 실행 (작은 데이터셋에서도)
        var forceParallel = data
            .AsParallel()
            .WithExecutionMode(ParallelExecutionMode.ForceParallelism)
            .Select(n => n * 2);
    }
}
```

### WithMergeOptions

```csharp
public class PLINQMergeOptions : MonoBehaviour
{
    void Start()
    {
        var data = Enumerable.Range(0, 10000);

        // NotBuffered: 결과 즉시 반환 (지연 없음)
        var notBuffered = data
            .AsParallel()
            .WithMergeOptions(ParallelMergeOptions.NotBuffered)
            .Select(n => n * 2);

        // AutoBuffered: 부분 버퍼링 (기본값)
        var autoBuffered = data
            .AsParallel()
            .WithMergeOptions(ParallelMergeOptions.AutoBuffered)
            .Select(n => n * 2);

        // FullyBuffered: 모든 결과 수집 후 반환 (메모리 사용 큼)
        var fullyBuffered = data
            .AsParallel()
            .WithMergeOptions(ParallelMergeOptions.FullyBuffered)
            .Select(n => n * 2);
    }
}
```

---

## 취소 처리

### WithCancellation

```csharp
public class PLINQCancellation : MonoBehaviour
{
    private CancellationTokenSource _cts;

    void Start()
    {
        _cts = new CancellationTokenSource();
        ProcessDataAsync();
    }

    async void ProcessDataAsync()
    {
        var data = Enumerable.Range(0, 100000);

        try
        {
            var result = await Task.Run(() =>
                data
                    .AsParallel()
                    .WithCancellation(_cts.Token)
                    .Select(n =>
                    {
                        // 무거운 작업
                        Thread.Sleep(1);
                        return n * 2;
                    })
                    .ToList()
            );

            Debug.Log($"Completed: {result.Count} items");
        }
        catch (OperationCanceledException)
        {
            Debug.Log("Query cancelled");
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

## PLINQ 연산자

### ForAll (부작용 실행)

```csharp
public class PLINQForAll : MonoBehaviour
{
    void Start()
    {
        var numbers = Enumerable.Range(1, 100);

        // ForAll: ToList 없이 각 요소에 액션 실행
        numbers
            .AsParallel()
            .Where(n => n % 2 == 0)
            .ForAll(n =>
            {
                // 병렬로 실행됨
                Debug.Log($"Processing {n} on thread {Thread.CurrentThread.ManagedThreadId}");
            });

        // 주의: ForAll은 순서 보장 안 함
    }
}
```

### Aggregate (병렬 집계)

```csharp
public class PLINQAggregate : MonoBehaviour
{
    void Start()
    {
        var numbers = Enumerable.Range(1, 10000);

        // 순차 합계
        var sequentialSum = numbers.Sum();

        // 병렬 합계
        var parallelSum = numbers
            .AsParallel()
            .Sum();

        // 커스텀 병렬 집계
        var customAggregate = numbers
            .AsParallel()
            .Aggregate(
                seed: 0,
                func: (subtotal, item) => subtotal + item,
                resultSelector: result => result
            );

        // 스레드별 시드가 필요한 집계
        var threadLocalAggregate = numbers
            .AsParallel()
            .Aggregate(
                seedFactory: () => 0, // 각 스레드의 초기값
                updateAccumulatorFunc: (subtotal, item) => subtotal + item,
                combineAccumulatorsFunc: (total, subtotal) => total + subtotal,
                resultSelector: result => result
            );

        Debug.Log($"Sequential: {sequentialSum}");
        Debug.Log($"Parallel: {parallelSum}");
        Debug.Log($"Custom: {customAggregate}");
        Debug.Log($"Thread-local: {threadLocalAggregate}");
    }
}
```

---

## Unity에서의 활용

### 대량 데이터 필터링

```csharp
public class PLINQDataFiltering : MonoBehaviour
{
    [System.Serializable]
    public class GameEntity
    {
        public int Id;
        public Vector3 Position;
        public int Health;
        public bool IsActive;
    }

    private List<GameEntity> _entities = new List<GameEntity>();

    void Start()
    {
        // 테스트 데이터 생성
        for (int i = 0; i < 10000; i++)
        {
            _entities.Add(new GameEntity
            {
                Id = i,
                Position = new Vector3(
                    UnityEngine.Random.Range(-100f, 100f),
                    0,
                    UnityEngine.Random.Range(-100f, 100f)),
                Health = UnityEngine.Random.Range(0, 100),
                IsActive = UnityEngine.Random.value > 0.5f
            });
        }

        // PLINQ로 필터링
        var playerPosition = Vector3.zero;
        float range = 50f;

        var nearbyActiveEntities = _entities
            .AsParallel()
            .Where(e => e.IsActive)
            .Where(e => e.Health > 0)
            .Where(e => Vector3.Distance(e.Position, playerPosition) < range)
            .ToList();

        Debug.Log($"Found {nearbyActiveEntities.Count} nearby active entities");
    }
}
```

### 텍스처 분석

```csharp
public class PLINQTextureAnalysis : MonoBehaviour
{
    [SerializeField] private Texture2D _texture;

    void Start()
    {
        var pixels = _texture.GetPixels();

        // 병렬로 색상 분석
        var colorStats = pixels
            .AsParallel()
            .GroupBy(p => GetColorCategory(p))
            .Select(g => new
            {
                Category = g.Key,
                Count = g.Count(),
                AverageR = g.Average(p => p.r),
                AverageG = g.Average(p => p.g),
                AverageB = g.Average(p => p.b)
            })
            .ToList();

        foreach (var stat in colorStats)
        {
            Debug.Log($"{stat.Category}: {stat.Count} pixels, " +
                     $"Avg RGB: ({stat.AverageR:F2}, {stat.AverageG:F2}, {stat.AverageB:F2})");
        }
    }

    string GetColorCategory(Color pixel)
    {
        float max = Mathf.Max(pixel.r, pixel.g, pixel.b);
        if (max < 0.2f) return "Dark";
        if (max > 0.8f) return "Bright";
        if (pixel.r > pixel.g && pixel.r > pixel.b) return "Red-ish";
        if (pixel.g > pixel.r && pixel.g > pixel.b) return "Green-ish";
        return "Blue-ish";
    }
}
```

### 점수 계산

```csharp
public class PLINQScoreCalculation : MonoBehaviour
{
    [System.Serializable]
    public class PlayerScore
    {
        public string PlayerId;
        public int[] RoundScores;
    }

    private List<PlayerScore> _playerScores = new List<PlayerScore>();

    void Start()
    {
        // 테스트 데이터
        for (int i = 0; i < 1000; i++)
        {
            var scores = new int[10];
            for (int j = 0; j < 10; j++)
            {
                scores[j] = UnityEngine.Random.Range(0, 100);
            }
            _playerScores.Add(new PlayerScore
            {
                PlayerId = $"Player_{i}",
                RoundScores = scores
            });
        }

        // 병렬로 총점 계산 및 정렬
        var leaderboard = _playerScores
            .AsParallel()
            .Select(p => new
            {
                Player = p.PlayerId,
                TotalScore = p.RoundScores.Sum(),
                AverageScore = p.RoundScores.Average(),
                BestRound = p.RoundScores.Max()
            })
            .OrderByDescending(x => x.TotalScore)
            .Take(10)
            .ToList();

        foreach (var entry in leaderboard)
        {
            Debug.Log($"{entry.Player}: Total={entry.TotalScore}, " +
                     $"Avg={entry.AverageScore:F1}, Best={entry.BestRound}");
        }
    }
}
```

---

## 성능 고려사항

### PLINQ가 효과적인 경우

```csharp
public class PLINQEffectiveness : MonoBehaviour
{
    void Start()
    {
        var data = Enumerable.Range(0, 100000).ToArray();

        // ✅ 효과적: CPU 집약적 작업
        var cpuBound = data
            .AsParallel()
            .Select(n => ExpensiveCalculation(n))
            .ToList();

        // ❌ 비효과적: 간단한 작업
        var simple = data
            .AsParallel()
            .Select(n => n * 2) // 너무 빠른 연산
            .ToList();

        // ✅ 효과적: 필터링 후 무거운 처리
        var filtered = data
            .AsParallel()
            .Where(n => n % 100 == 0) // 1%만 통과
            .Select(n => ExpensiveCalculation(n))
            .ToList();
    }

    int ExpensiveCalculation(int n)
    {
        // 무거운 계산
        double result = 0;
        for (int i = 0; i < 10000; i++)
        {
            result += Math.Sin(n * i * 0.0001);
        }
        return (int)result;
    }
}
```

### 오버헤드 측정

```csharp
public class PLINQBenchmark : MonoBehaviour
{
    void Start()
    {
        var data = Enumerable.Range(0, 100000).ToArray();

        // 순차
        var sw1 = System.Diagnostics.Stopwatch.StartNew();
        var seqResult = data.Select(n => n * 2).ToList();
        sw1.Stop();

        // 병렬
        var sw2 = System.Diagnostics.Stopwatch.StartNew();
        var parResult = data.AsParallel().Select(n => n * 2).ToList();
        sw2.Stop();

        Debug.Log($"Sequential: {sw1.ElapsedMilliseconds}ms");
        Debug.Log($"Parallel: {sw2.ElapsedMilliseconds}ms");
        // 단순 연산에서는 순차가 더 빠를 수 있음!
    }
}
```

---

## PLINQ vs Parallel

### 비교

| 항목 | PLINQ | Parallel |
|------|-------|----------|
| **스타일** | 선언적 (쿼리) | 명령형 (루프) |
| **용도** | 데이터 변환/필터링 | 각 요소에 작업 실행 |
| **결과** | IEnumerable 반환 | void 또는 ParallelLoopResult |
| **순서** | AsOrdered()로 제어 | 기본 무순서 |
| **취소** | WithCancellation | ParallelOptions |

### 선택 가이드

```csharp
public class PLINQvsParallelChoice : MonoBehaviour
{
    void Start()
    {
        var data = Enumerable.Range(0, 10000).ToArray();

        // PLINQ 적합: 데이터 변환, 필터링, 집계
        var transformed = data
            .AsParallel()
            .Where(n => n % 2 == 0)
            .Select(n => n * 2)
            .Sum();

        // Parallel 적합: 부작용 있는 작업
        var results = new ConcurrentBag<int>();
        Parallel.ForEach(data, n =>
        {
            var processed = ProcessWithSideEffect(n);
            results.Add(processed);
        });
    }

    int ProcessWithSideEffect(int n)
    {
        // 외부 상태 변경 등
        return n * 2;
    }
}
```

---

## 실전 패턴

### 패턴 1: 청크 처리

```csharp
public class PLINQChunkProcessing : MonoBehaviour
{
    void Start()
    {
        var largeData = Enumerable.Range(0, 1000000).ToArray();

        // 청크로 나눠서 처리
        int chunkSize = 10000;
        var chunks = largeData
            .Select((value, index) => new { value, index })
            .GroupBy(x => x.index / chunkSize)
            .Select(g => g.Select(x => x.value).ToArray())
            .ToList();

        // 각 청크를 병렬로 처리
        var results = chunks
            .AsParallel()
            .SelectMany(chunk => ProcessChunk(chunk))
            .ToList();

        Debug.Log($"Processed {results.Count} items");
    }

    IEnumerable<int> ProcessChunk(int[] chunk)
    {
        return chunk.Select(n => n * 2);
    }
}
```

### 패턴 2: 조건부 병렬화

```csharp
public class ConditionalParallelization : MonoBehaviour
{
    void Start()
    {
        var data = GetData();

        // 데이터 크기에 따라 병렬화 결정
        IEnumerable<int> query = data.Length > 1000
            ? data.AsParallel().Select(n => ProcessItem(n))
            : data.Select(n => ProcessItem(n));

        var result = query.ToList();
    }

    int[] GetData()
    {
        return Enumerable.Range(0, 5000).ToArray();
    }

    int ProcessItem(int n) => n * 2;
}
```

### 패턴 3: 예외 처리

```csharp
public class PLINQExceptionHandling : MonoBehaviour
{
    void Start()
    {
        var data = Enumerable.Range(-10, 20);

        try
        {
            var results = data
                .AsParallel()
                .Select(n =>
                {
                    if (n == 0)
                        throw new DivideByZeroException();
                    return 100 / n;
                })
                .ToList();
        }
        catch (AggregateException ae)
        {
            // 병렬 실행 중 발생한 모든 예외
            foreach (var ex in ae.InnerExceptions)
            {
                Debug.LogError($"Exception: {ex.Message}");
            }
        }
    }
}
```

---

## 주의사항

### 스레드 안전성

```csharp
public class PLINQThreadSafety : MonoBehaviour
{
    void Start()
    {
        var data = Enumerable.Range(0, 1000);

        // ❌ 스레드 안전하지 않음
        var list = new List<int>();
        data.AsParallel().ForAll(n =>
        {
            list.Add(n); // 위험!
        });

        // ✅ ToList() 사용
        var safeList = data
            .AsParallel()
            .Select(n => n * 2)
            .ToList();

        // ✅ 스레드 안전 컬렉션
        var safeBag = new ConcurrentBag<int>();
        data.AsParallel().ForAll(n =>
        {
            safeBag.Add(n * 2);
        });
    }
}
```

### 순서 의존성 주의

```csharp
public class PLINQOrderDependency : MonoBehaviour
{
    void Start()
    {
        var numbers = new[] { 1, 2, 3, 4, 5 };

        // ❌ 순서 의존 연산에 병렬 사용
        int runningTotal = 0;
        var wrongResult = numbers
            .AsParallel()
            .Select(n =>
            {
                runningTotal += n; // 경쟁 조건!
                return runningTotal;
            })
            .ToList();

        // ✅ Scan 패턴 사용 (순차)
        runningTotal = 0;
        var correctResult = numbers
            .Select(n =>
            {
                runningTotal += n;
                return runningTotal;
            })
            .ToList();

        // ✅ 또는 Aggregate 사용
        var aggregateResult = numbers
            .AsParallel()
            .Aggregate(
                new List<int>(),
                (list, n) =>
                {
                    var sum = list.Count > 0 ? list.Last() + n : n;
                    return list.Concat(new[] { sum }).ToList();
                });
    }
}
```

---

## 정리

### PLINQ 요약

| 항목 | 내용 |
|------|------|
| **활성화** | `.AsParallel()` |
| **순서 보장** | `.AsOrdered()` |
| **병렬 수준** | `.WithDegreeOfParallelism(n)` |
| **취소** | `.WithCancellation(token)` |
| **부작용 실행** | `.ForAll(action)` |

### 체크리스트

- [ ] 작업이 CPU 집약적인가?
- [ ] 데이터셋이 충분히 큰가? (>1000 권장)
- [ ] 순서가 중요한가? (AsOrdered 필요?)
- [ ] 스레드 안전성 확보했는가?
- [ ] Unity API 호출이 없는가?

---

## 참고 자료

- [PLINQ - Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/parallel-linq-plinq)
- [Introduction to PLINQ](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/introduction-to-plinq)
- [Order Preservation in PLINQ](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/order-preservation-in-plinq)
