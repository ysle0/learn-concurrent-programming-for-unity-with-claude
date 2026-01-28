# 01. 동시성 프로그래밍 기초 개념

## 개요

동시성 프로그래밍은 여러 작업을 효율적으로 처리하기 위한 프로그래밍 패러다임입니다. Unity 게임 개발에서 부드러운 프레임 레이트를 유지하면서 네트워크 통신, 파일 I/O, 복잡한 연산을 처리하려면 동시성 프로그래밍의 기본 개념을 이해해야 합니다.

---

## 1. 동기(Synchronous) vs 비동기(Asynchronous)

### 동기(Synchronous)

동기 처리는 작업이 순차적으로 실행되며, 현재 작업이 완료될 때까지 다음 작업이 대기합니다.

```
작업A 시작 → 작업A 완료 → 작업B 시작 → 작업B 완료 → 작업C 시작 → 작업C 완료
```

### 비동기(Asynchronous)

비동기 처리는 작업의 완료를 기다리지 않고 다음 작업을 시작합니다. 작업이 완료되면 콜백, 이벤트, 또는 await를 통해 결과를 받습니다.

```
작업A 시작 → 작업B 시작 → 작업C 시작
     ↓           ↓           ↓
  작업A 완료   작업B 완료   작업C 완료
```

### Unity 예제: 동기 vs 비동기

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;
using System.Threading.Tasks;
using System.Net.Http;
using System.IO;

public class SyncVsAsyncExample : MonoBehaviour
{
    // ❌ 동기 방식 - 게임이 멈춤 (절대 사용하지 말 것!)
    private void SynchronousFileRead()
    {
        Debug.Log("파일 읽기 시작");

        // 이 작업이 완료될 때까지 메인 스레드가 블로킹됨
        // Unity 에디터나 게임이 완전히 멈춤!
        string content = File.ReadAllText("very_large_file.txt");

        Debug.Log($"파일 읽기 완료: {content.Length} 글자");
    }

    // ✅ 비동기 방식 (Coroutine) - Unity 전통적인 방법
    private IEnumerator AsynchronousWebRequest()
    {
        Debug.Log("웹 요청 시작");

        using (UnityWebRequest request = UnityWebRequest.Get("https://api.example.com/data"))
        {
            // 요청을 보내고 완료될 때까지 프레임마다 체크
            // 이 동안 게임은 계속 실행됨
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"응답 받음: {request.downloadHandler.text}");
            }
            else
            {
                Debug.LogError($"에러: {request.error}");
            }
        }

        Debug.Log("웹 요청 완료");
    }

    // ✅ 비동기 방식 (async/await) - 현대적인 방법
    private async void AsynchronousFileReadAsync()
    {
        Debug.Log("비동기 파일 읽기 시작");

        // await 키워드로 비동기 대기
        // 이 동안 메인 스레드는 다른 작업을 수행할 수 있음
        string content = await File.ReadAllTextAsync("very_large_file.txt");

        Debug.Log($"파일 읽기 완료: {content.Length} 글자");
    }

    private void Start()
    {
        // Coroutine 시작
        StartCoroutine(AsynchronousWebRequest());

        // async 메서드 호출
        AsynchronousFileReadAsync();

        // 이 로그는 위의 비동기 작업들이 완료되기 전에 즉시 출력됨
        Debug.Log("Start 메서드 완료 - 비동기 작업은 백그라운드에서 진행 중");
    }
}
```

### 핵심 포인트

| 구분 | 동기 | 비동기 |
|------|------|--------|
| 실행 흐름 | 순차적, 블로킹 | 비순차적, 논블로킹 |
| 코드 복잡도 | 단순함 | 상대적으로 복잡 |
| 성능 | I/O 대기 시 비효율적 | I/O 대기 시 효율적 |
| Unity에서 | 프레임 드랍 발생 | 부드러운 실행 유지 |

---

## 2. 병렬(Parallelism) vs 동시성(Concurrency)

이 두 개념은 자주 혼동되지만 명확히 다릅니다.

### 동시성(Concurrency)

- **정의**: 여러 작업을 번갈아가며 처리하는 것처럼 보이게 하는 것
- **비유**: 요리사 1명이 여러 요리를 번갈아가며 조리
- **핵심**: 단일 코어에서도 가능, 작업 전환(Context Switching)을 통해 구현

```
시간 →
CPU: [작업A][작업B][작업A][작업C][작업B][작업A]...
```

### 병렬(Parallelism)

- **정의**: 여러 작업을 실제로 동시에 처리하는 것
- **비유**: 요리사 4명이 각각 다른 요리를 동시에 조리
- **핵심**: 멀티 코어 필수, 실제로 동시 실행

```
시간 →
CPU 코어1: [작업A][작업A][작업A]...
CPU 코어2: [작업B][작업B][작업B]...
CPU 코어3: [작업C][작업C][작업C]...
CPU 코어4: [작업D][작업D][작업D]...
```

### Unity 예제: 동시성 vs 병렬성

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;
using System.Collections;
using System.Collections.Generic;

public class ConcurrencyVsParallelismExample : MonoBehaviour
{
    // =============================================
    // 동시성 (Concurrency) 예제 - 단일 스레드, 협력적 멀티태스킹
    // =============================================

    // Coroutine을 사용한 동시성 - 모든 작업이 메인 스레드에서 실행
    private IEnumerator ConcurrentTask(string taskName, int iterations)
    {
        for (int i = 0; i < iterations; i++)
        {
            Debug.Log($"[Concurrency] {taskName}: 반복 {i + 1}/{iterations} " +
                     $"(Thread: {Thread.CurrentThread.ManagedThreadId})");

            // yield return으로 제어권을 양보 - 다른 코루틴이 실행될 수 있음
            yield return null; // 다음 프레임까지 대기
        }
        Debug.Log($"[Concurrency] {taskName} 완료!");
    }

    private void StartConcurrentTasks()
    {
        Debug.Log("=== 동시성 작업 시작 (모두 메인 스레드에서 실행) ===");

        // 3개의 코루틴이 "동시에" 실행되는 것처럼 보이지만
        // 실제로는 메인 스레드에서 번갈아가며 실행됨
        StartCoroutine(ConcurrentTask("작업A", 3));
        StartCoroutine(ConcurrentTask("작업B", 3));
        StartCoroutine(ConcurrentTask("작업C", 3));
    }

    // =============================================
    // 병렬 (Parallelism) 예제 - 멀티 스레드, 실제 동시 실행
    // =============================================

    private async void StartParallelTasks()
    {
        Debug.Log("=== 병렬 작업 시작 (여러 스레드에서 동시 실행) ===");

        // Task.Run은 ThreadPool의 다른 스레드에서 작업을 실행
        var tasks = new List<Task>();

        tasks.Add(Task.Run(() => ParallelTask("작업A", 1000000)));
        tasks.Add(Task.Run(() => ParallelTask("작업B", 1000000)));
        tasks.Add(Task.Run(() => ParallelTask("작업C", 1000000)));
        tasks.Add(Task.Run(() => ParallelTask("작업D", 1000000)));

        // 모든 병렬 작업이 완료될 때까지 대기
        await Task.WhenAll(tasks);

        Debug.Log("=== 모든 병렬 작업 완료 ===");
    }

    private void ParallelTask(string taskName, int iterations)
    {
        int threadId = Thread.CurrentThread.ManagedThreadId;
        Debug.Log($"[Parallelism] {taskName} 시작 (Thread: {threadId})");

        // CPU 연산 시뮬레이션
        double result = 0;
        for (int i = 0; i < iterations; i++)
        {
            result += System.Math.Sqrt(i);
        }

        Debug.Log($"[Parallelism] {taskName} 완료 (Thread: {threadId}, Result: {result:F2})");
    }

    // =============================================
    // Parallel.For를 사용한 데이터 병렬 처리
    // =============================================

    private void ParallelDataProcessing()
    {
        Debug.Log("=== 데이터 병렬 처리 시작 ===");

        int[] data = new int[10000];
        int[] results = new int[10000];

        // 초기화
        for (int i = 0; i < data.Length; i++)
        {
            data[i] = i;
        }

        // Parallel.For - 여러 코어에서 동시에 처리
        Parallel.For(0, data.Length, i =>
        {
            // 각 요소를 독립적으로 처리 (여러 스레드에서 동시 실행)
            results[i] = data[i] * data[i];
        });

        Debug.Log($"병렬 처리 완료. 첫 번째 결과: {results[0]}, 마지막 결과: {results[9999]}");
    }

    private void Start()
    {
        // 동시성 예제 실행
        StartConcurrentTasks();

        // 병렬성 예제 실행
        Invoke(nameof(RunParallelExamples), 2f);
    }

    private void RunParallelExamples()
    {
        StartParallelTasks();
        ParallelDataProcessing();
    }
}
```

### 출력 예시 비교

**동시성 (Concurrency) 출력:**
```
[Concurrency] 작업A: 반복 1/3 (Thread: 1)  // 모두 같은 스레드
[Concurrency] 작업B: 반복 1/3 (Thread: 1)
[Concurrency] 작업C: 반복 1/3 (Thread: 1)
[Concurrency] 작업A: 반복 2/3 (Thread: 1)
[Concurrency] 작업B: 반복 2/3 (Thread: 1)
...
```

**병렬 (Parallelism) 출력:**
```
[Parallelism] 작업A 시작 (Thread: 4)   // 서로 다른 스레드
[Parallelism] 작업B 시작 (Thread: 5)
[Parallelism] 작업C 시작 (Thread: 6)
[Parallelism] 작업D 시작 (Thread: 7)
[Parallelism] 작업C 완료 (Thread: 6)   // 완료 순서는 보장되지 않음
[Parallelism] 작업A 완료 (Thread: 4)
...
```

### 핵심 비교표

| 구분 | 동시성 (Concurrency) | 병렬 (Parallelism) |
|------|---------------------|-------------------|
| 정의 | 여러 작업을 번갈아 처리 | 여러 작업을 동시에 처리 |
| 필요 조건 | 단일 코어로도 가능 | 멀티 코어 필수 |
| 스레드 | 보통 단일 스레드 | 멀티 스레드 |
| Unity 예시 | Coroutine, async/await | Parallel.For, Job System |
| 적합한 작업 | I/O-bound 작업 | CPU-bound 작업 |
| 오버헤드 | 낮음 | 스레드 생성/전환 비용 |

---

## 3. Blocking vs Non-blocking

### Blocking (블로킹)

호출된 함수가 완료될 때까지 호출자의 실행이 멈추는 것.

```csharp
// 블로킹 예제
string data = File.ReadAllText("file.txt");  // 파일을 다 읽을 때까지 멈춤
Console.WriteLine(data);                       // 위 줄이 끝나야 실행됨
```

### Non-blocking (논블로킹)

호출된 함수가 즉시 반환되어 호출자가 계속 실행될 수 있는 것.

```csharp
// 논블로킹 예제
var task = File.ReadAllTextAsync("file.txt");  // 즉시 반환
Console.WriteLine("파일 읽는 중...");           // 바로 실행됨
string data = await task;                       // 필요할 때 결과 대기
```

### Unity 예제: Blocking vs Non-blocking

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;
using System.Net;
using System.IO;
using System.Threading;
using System.Diagnostics;

public class BlockingVsNonBlockingExample : MonoBehaviour
{
    private Stopwatch stopwatch = new Stopwatch();
    private int frameCount = 0;

    // =============================================
    // ❌ Blocking 예제 - 절대 사용하지 말 것!
    // =============================================

    /// <summary>
    /// 블로킹 방식의 웹 요청 - 게임이 완전히 멈춤
    /// </summary>
    private void BlockingWebRequest()
    {
        stopwatch.Restart();

        // WebClient.DownloadString은 동기 메서드
        // 응답이 올 때까지 메인 스레드가 완전히 멈춤!
        using (var client = new WebClient())
        {
            // ⚠️ 이 줄에서 게임이 멈춤 (프레임 드랍!)
            string result = client.DownloadString("https://httpbin.org/delay/2");
            UnityEngine.Debug.Log($"[Blocking] 응답 받음 (소요 시간: {stopwatch.ElapsedMilliseconds}ms)");
        }
    }

    /// <summary>
    /// 블로킹 방식의 파일 읽기 - 대용량 파일에서 문제 발생
    /// </summary>
    private void BlockingFileRead()
    {
        stopwatch.Restart();

        // File.ReadAllBytes는 동기 메서드
        // 파일을 다 읽을 때까지 메인 스레드 블로킹
        byte[] data = File.ReadAllBytes("large_file.bin");

        UnityEngine.Debug.Log($"[Blocking] 파일 읽기 완료: {data.Length} bytes " +
                             $"(소요 시간: {stopwatch.ElapsedMilliseconds}ms)");
    }

    /// <summary>
    /// 블로킹 방식의 Sleep - 절대 메인 스레드에서 사용하지 말 것!
    /// </summary>
    private void BlockingSleep()
    {
        UnityEngine.Debug.Log("[Blocking] Sleep 시작...");

        // Thread.Sleep은 현재 스레드를 완전히 멈춤
        // 메인 스레드에서 호출하면 게임 전체가 멈춤!
        Thread.Sleep(2000); // 2초 동안 게임 정지

        UnityEngine.Debug.Log("[Blocking] Sleep 종료");
    }

    // =============================================
    // ✅ Non-blocking 예제 - 권장 방법
    // =============================================

    /// <summary>
    /// 논블로킹 방식의 웹 요청 (Coroutine)
    /// </summary>
    private IEnumerator NonBlockingWebRequest()
    {
        stopwatch.Restart();

        using (var request = UnityWebRequest.Get("https://httpbin.org/delay/2"))
        {
            // SendWebRequest()는 즉시 반환됨 (논블로킹)
            // yield return으로 매 프레임 완료 여부만 체크
            yield return request.SendWebRequest();

            UnityEngine.Debug.Log($"[Non-Blocking] 응답 받음 (소요 시간: {stopwatch.ElapsedMilliseconds}ms)");
            UnityEngine.Debug.Log($"[Non-Blocking] 요청 중 업데이트된 프레임 수: {frameCount}");
        }
    }

    /// <summary>
    /// 논블로킹 방식의 파일 읽기 (async/await)
    /// </summary>
    private async void NonBlockingFileReadAsync()
    {
        stopwatch.Restart();

        // ReadAllBytesAsync는 비동기 메서드 - 즉시 반환
        byte[] data = await File.ReadAllBytesAsync("large_file.bin");

        UnityEngine.Debug.Log($"[Non-Blocking] 파일 읽기 완료: {data.Length} bytes " +
                             $"(소요 시간: {stopwatch.ElapsedMilliseconds}ms)");
    }

    /// <summary>
    /// 논블로킹 방식의 대기 (Coroutine)
    /// </summary>
    private IEnumerator NonBlockingWait()
    {
        UnityEngine.Debug.Log("[Non-Blocking] 대기 시작...");

        // WaitForSeconds는 논블로킹 - 게임은 계속 실행됨
        yield return new WaitForSeconds(2f);

        UnityEngine.Debug.Log("[Non-Blocking] 대기 종료");
        UnityEngine.Debug.Log($"[Non-Blocking] 대기 중 업데이트된 프레임 수: {frameCount}");
    }

    // =============================================
    // 프레임 카운터로 블로킹 여부 확인
    // =============================================

    private void Update()
    {
        frameCount++;

        // 정상적으로 논블로킹이면 이 로그가 계속 출력됨
        // 블로킹이면 이 로그가 멈춤
        if (frameCount % 60 == 0)
        {
            UnityEngine.Debug.Log($"Frame: {frameCount} (게임 정상 실행 중)");
        }
    }

    // =============================================
    // 테스트 실행
    // =============================================

    private void Start()
    {
        // 논블로킹 방식 테스트 (권장)
        StartCoroutine(NonBlockingWait());
        StartCoroutine(NonBlockingWebRequest());

        // ⚠️ 아래 메서드들은 주석을 해제하면 게임이 멈춤!
        // BlockingSleep();
        // BlockingWebRequest();
    }
}
```

### 핵심 비교표

| 구분 | Blocking | Non-blocking |
|------|----------|--------------|
| 호출 후 | 완료까지 대기 | 즉시 반환 |
| 스레드 상태 | 대기 (멈춤) | 다른 작업 가능 |
| Unity 영향 | 프레임 드랍, 게임 정지 | 부드러운 실행 유지 |
| 예시 | Thread.Sleep, File.ReadAllText | yield return, await, Task |

---

## 4. CPU-bound vs I/O-bound

작업의 특성에 따라 적절한 비동기 패턴이 달라집니다.

### CPU-bound 작업

- **정의**: CPU 연산이 주된 작업 (계산, 알고리즘, 데이터 처리)
- **특징**: 더 빠른 CPU가 있으면 더 빨리 완료
- **해결책**: 병렬 처리 (멀티 스레드), Job System
- **예시**: 경로 탐색, 물리 연산, 이미지 처리, AI 계산

### I/O-bound 작업

- **정의**: 외부 시스템 대기가 주된 작업 (네트워크, 파일, DB)
- **특징**: 더 빠른 CPU가 있어도 크게 빨라지지 않음
- **해결책**: 비동기 처리 (async/await, Coroutine)
- **예시**: 웹 요청, 파일 읽기/쓰기, 데이터베이스 쿼리

### Unity 예제: CPU-bound vs I/O-bound

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;
using System.Threading.Tasks;
using System.Diagnostics;
using System.IO;
using Unity.Jobs;
using Unity.Collections;
using Unity.Burst;

public class CpuBoundVsIoBoundExample : MonoBehaviour
{
    private Stopwatch stopwatch = new Stopwatch();

    // =============================================
    // CPU-bound 작업 예제
    // =============================================

    /// <summary>
    /// CPU-bound: 경로 탐색 시뮬레이션 (A* 알고리즘 등)
    /// </summary>
    private int[] CalculatePathfinding(int gridSize)
    {
        // CPU 집약적인 연산 시뮬레이션
        int[] path = new int[gridSize * gridSize];

        for (int i = 0; i < path.Length; i++)
        {
            // 복잡한 계산 시뮬레이션
            for (int j = 0; j < 100; j++)
            {
                path[i] = (int)(Mathf.Sqrt(i * j) + Mathf.Sin(i) * 100);
            }
        }

        return path;
    }

    /// <summary>
    /// ❌ 잘못된 방법: 메인 스레드에서 CPU-bound 작업 실행
    /// </summary>
    private void CpuBoundOnMainThread()
    {
        stopwatch.Restart();

        // 메인 스레드에서 무거운 연산 → 프레임 드랍!
        int[] result = CalculatePathfinding(500);

        UnityEngine.Debug.Log($"[CPU-bound 메인스레드] 완료: {stopwatch.ElapsedMilliseconds}ms, " +
                             $"결과 길이: {result.Length}");
    }

    /// <summary>
    /// ✅ 올바른 방법 1: Task.Run으로 백그라운드 스레드에서 실행
    /// </summary>
    private async void CpuBoundWithTaskRun()
    {
        stopwatch.Restart();

        // Task.Run으로 ThreadPool 스레드에서 실행
        int[] result = await Task.Run(() => CalculatePathfinding(500));

        UnityEngine.Debug.Log($"[CPU-bound Task.Run] 완료: {stopwatch.ElapsedMilliseconds}ms, " +
                             $"결과 길이: {result.Length}");
    }

    /// <summary>
    /// ✅ 올바른 방법 2: Unity Job System 사용 (가장 효율적)
    /// </summary>
    [BurstCompile]
    private struct PathfindingJob : IJobParallelFor
    {
        [WriteOnly]
        public NativeArray<int> Results;

        public void Execute(int index)
        {
            // 각 인덱스를 병렬로 처리
            int value = 0;
            for (int j = 0; j < 100; j++)
            {
                value = (int)(Mathf.Sqrt(index * j) + Mathf.Sin(index) * 100);
            }
            Results[index] = value;
        }
    }

    private void CpuBoundWithJobSystem()
    {
        stopwatch.Restart();

        int size = 500 * 500;
        var results = new NativeArray<int>(size, Allocator.TempJob);

        var job = new PathfindingJob
        {
            Results = results
        };

        // 병렬 실행 (모든 CPU 코어 활용)
        JobHandle handle = job.Schedule(size, 64);
        handle.Complete();

        UnityEngine.Debug.Log($"[CPU-bound Job System] 완료: {stopwatch.ElapsedMilliseconds}ms, " +
                             $"결과 길이: {results.Length}");

        results.Dispose();
    }

    // =============================================
    // I/O-bound 작업 예제
    // =============================================

    /// <summary>
    /// I/O-bound: 웹 요청 (서버 응답 대기가 대부분)
    /// </summary>
    private IEnumerator IoBoundWebRequest()
    {
        stopwatch.Restart();

        using (var request = UnityWebRequest.Get("https://httpbin.org/delay/1"))
        {
            // I/O 대기 - CPU는 거의 사용하지 않음
            yield return request.SendWebRequest();

            UnityEngine.Debug.Log($"[I/O-bound 웹요청] 완료: {stopwatch.ElapsedMilliseconds}ms");
        }
    }

    /// <summary>
    /// I/O-bound: 파일 읽기 (디스크 대기가 대부분)
    /// </summary>
    private async void IoBoundFileRead()
    {
        stopwatch.Restart();

        // 비동기 파일 읽기 - 디스크 I/O 대기
        if (File.Exists("test_file.txt"))
        {
            string content = await File.ReadAllTextAsync("test_file.txt");
            UnityEngine.Debug.Log($"[I/O-bound 파일읽기] 완료: {stopwatch.ElapsedMilliseconds}ms, " +
                                 $"길이: {content.Length}");
        }
    }

    // =============================================
    // 잘못된 패턴 예제 (하지 말아야 할 것)
    // =============================================

    /// <summary>
    /// ❌ 잘못된 패턴: I/O-bound 작업에 Task.Run 사용
    /// (불필요한 스레드 점유, 오히려 비효율적)
    /// </summary>
    private async void WrongPatternIoBoundWithTaskRun()
    {
        // ❌ I/O-bound 작업을 Task.Run으로 감싸면 스레드를 낭비
        string result = await Task.Run(() =>
        {
            // 이 스레드는 대부분의 시간을 I/O 대기에 사용
            // 다른 작업을 할 수 있는 스레드를 불필요하게 점유
            return File.ReadAllText("file.txt");
        });
    }

    /// <summary>
    /// ✅ 올바른 패턴: I/O-bound 작업은 async API 직접 사용
    /// </summary>
    private async void CorrectPatternIoBound()
    {
        // ✅ 비동기 API를 직접 사용 - 스레드 점유 없이 I/O 완료 대기
        string result = await File.ReadAllTextAsync("file.txt");
    }

    // =============================================
    // 테스트 실행
    // =============================================

    private void Start()
    {
        // CPU-bound 예제들
        UnityEngine.Debug.Log("=== CPU-bound 작업 비교 ===");
        CpuBoundWithTaskRun();
        CpuBoundWithJobSystem();

        // I/O-bound 예제들
        UnityEngine.Debug.Log("=== I/O-bound 작업 비교 ===");
        StartCoroutine(IoBoundWebRequest());
        IoBoundFileRead();
    }
}
```

### 작업 유형별 권장 패턴

| 작업 유형 | 특성 | Unity 권장 패턴 |
|-----------|------|-----------------|
| **CPU-bound** | CPU 연산 집약적 | Job System + Burst, Task.Run, Parallel.For |
| **I/O-bound** | 외부 대기 집약적 | Coroutine, async/await, UniTask |
| **혼합** | CPU + I/O 모두 | 상황에 맞게 조합 |

### 판단 기준

```csharp
// 작업 유형 판단 가이드
public class TaskTypeGuide : MonoBehaviour
{
    /*
    ┌─────────────────────────────────────────────────────────────┐
    │                    작업 유형 판단 가이드                      │
    ├─────────────────────────────────────────────────────────────┤
    │                                                             │
    │  Q: 작업이 완료되기를 기다리는 동안 CPU가 바쁜가?              │
    │                                                             │
    │     예 → CPU-bound                                          │
    │          • 경로 탐색 (A*, 다익스트라)                         │
    │          • 물리 시뮬레이션                                   │
    │          • 이미지/오디오 처리                                │
    │          • AI 의사결정                                       │
    │          • 프로시저럴 생성                                   │
    │          → Job System, Task.Run, Parallel.For 사용          │
    │                                                             │
    │     아니오 → I/O-bound                                      │
    │          • 웹 API 호출                                       │
    │          • 파일 읽기/쓰기                                    │
    │          • 데이터베이스 쿼리                                 │
    │          • 에셋번들 다운로드                                 │
    │          • 씬 로딩                                           │
    │          → Coroutine, async/await, UniTask 사용             │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘
    */
}
```

---

## 5. 정리: Unity에서의 동시성 프로그래밍 선택 가이드

```
┌────────────────────────────────────────────────────────────────────┐
│                  Unity 동시성 패턴 선택 플로우차트                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  시작: 어떤 작업을 해야 하나요?                                      │
│                                                                    │
│  ├─ 단순한 시간 기반 작업 (대기, 애니메이션)                          │
│  │   └→ Coroutine (yield return)                                  │
│  │                                                                 │
│  ├─ 네트워크 요청, 파일 I/O                                         │
│  │   ├→ Unity 2022 이하: Coroutine + UnityWebRequest               │
│  │   ├→ Unity 2023+: Awaitable                                    │
│  │   └→ 모든 버전: UniTask (권장)                                   │
│  │                                                                 │
│  ├─ 무거운 CPU 연산 (경로탐색, 물리, AI)                             │
│  │   ├→ 단순한 경우: Task.Run                                      │
│  │   ├→ 대량 데이터: Parallel.For                                  │
│  │   └→ 최적화 필요: Job System + Burst (권장)                      │
│  │                                                                 │
│  ├─ 이벤트 스트림 처리 (입력, UI 상태)                               │
│  │   ├→ UniRx                                                      │
│  │   └→ R3 (최신)                                                  │
│  │                                                                 │
│  └─ 복잡한 비동기 흐름                                              │
│      └→ async/await + UniTask                                      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 주의사항

1. **메인 스레드 규칙**: Unity API (Transform, GameObject 등)는 반드시 메인 스레드에서만 호출
2. **블로킹 금지**: `Thread.Sleep()`, 동기 I/O 메서드를 메인 스레드에서 사용 금지
3. **작업 유형 파악**: CPU-bound와 I/O-bound를 구분하여 적절한 패턴 선택
4. **과도한 병렬화 주의**: 너무 많은 스레드/태스크는 오히려 성능 저하

---

## 참고 자료

- [Microsoft: Asynchronous Programming](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/)
- [Unity: Coroutines](https://docs.unity3d.com/Manual/Coroutines.html)
- [Unity: Job System](https://docs.unity3d.com/Manual/JobSystem.html)
- [UniTask GitHub](https://github.com/Cysharp/UniTask)

---

## 다음 섹션

[02. 유니티 엔진 라이프 사이클과 동시성](./02-unity-lifecycle.md)
