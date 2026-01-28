# 04. Thread & ThreadPool

## 개요

Thread는 프로그램 실행의 가장 기본적인 단위입니다. .NET의 `System.Threading.Thread`와 `ThreadPool`을 이해하면 Unity에서의 백그라운드 작업 처리와 멀티스레딩의 기초를 다질 수 있습니다.

---

## 1. Thread 기본 개념

### Thread란?

```
┌─────────────────────────────────────────────────────────────────┐
│                        Process vs Thread                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Process (프로세스)                                               │
│  ├─ 실행 중인 프로그램의 인스턴스                                  │
│  ├─ 독립적인 메모리 공간                                          │
│  ├─ 최소 1개 이상의 Thread 포함                                   │
│  └─ 예: Unity Editor, Unity 빌드된 게임                          │
│                                                                  │
│  Thread (스레드)                                                  │
│  ├─ 프로세스 내에서 실행되는 실행 단위                             │
│  ├─ 프로세스의 메모리 공간 공유                                    │
│  ├─ 독립적인 스택, 공유 힙                                        │
│  └─ 예: Unity 메인 스레드, 렌더 스레드, Job Worker 스레드           │
│                                                                  │
│  ┌────────────────────────────────────────────────┐             │
│  │                  Unity Process                   │             │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐       │             │
│  │  │  Main    │ │ Render   │ │  Job     │ ...   │             │
│  │  │ Thread   │ │ Thread   │ │ Workers  │       │             │
│  │  └──────────┘ └──────────┘ └──────────┘       │             │
│  │         │           │            │             │             │
│  │         └───────────┼────────────┘             │             │
│  │                     │                          │             │
│  │              Shared Memory                     │             │
│  └────────────────────────────────────────────────┘             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Thread 생성과 기본 사용

```csharp
using UnityEngine;
using System;
using System.Threading;

public class ThreadBasicsExample : MonoBehaviour
{
    // =============================================
    // 기본 Thread 생성
    // =============================================

    private void Start()
    {
        BasicThreadCreation();
        ThreadWithParameter();
        ThreadWithLambda();
    }

    /// <summary>
    /// 가장 기본적인 Thread 생성 방법
    /// </summary>
    private void BasicThreadCreation()
    {
        // ThreadStart 델리게이트로 스레드 생성
        Thread thread = new Thread(new ThreadStart(DoWork));

        // 스레드 시작
        thread.Start();

        Debug.Log($"[Main] Thread 시작됨. Main Thread ID: {Thread.CurrentThread.ManagedThreadId}");
    }

    private void DoWork()
    {
        int threadId = Thread.CurrentThread.ManagedThreadId;
        Debug.Log($"[Worker] DoWork 실행 중. Thread ID: {threadId}");

        // 작업 시뮬레이션
        for (int i = 0; i < 5; i++)
        {
            Thread.Sleep(100);
            Debug.Log($"[Worker] 작업 {i + 1}/5 완료");
        }

        Debug.Log("[Worker] DoWork 완료");
    }

    /// <summary>
    /// 매개변수가 있는 Thread 생성
    /// </summary>
    private void ThreadWithParameter()
    {
        // ParameterizedThreadStart 사용
        Thread thread = new Thread(new ParameterizedThreadStart(DoWorkWithParam));
        thread.Start("Hello from parameter!");
    }

    private void DoWorkWithParam(object parameter)
    {
        string message = parameter as string;
        Debug.Log($"[Worker] 받은 매개변수: {message}");
    }

    /// <summary>
    /// Lambda 표현식으로 Thread 생성 (가장 많이 사용)
    /// </summary>
    private void ThreadWithLambda()
    {
        int localValue = 42;

        Thread thread = new Thread(() =>
        {
            // 클로저로 외부 변수 캡처
            Debug.Log($"[Lambda] 캡처된 값: {localValue}");
            Debug.Log($"[Lambda] Thread ID: {Thread.CurrentThread.ManagedThreadId}");
        });

        thread.Start();
    }
}
```

### Thread 속성과 메서드

```csharp
using UnityEngine;
using System;
using System.Threading;

public class ThreadPropertiesExample : MonoBehaviour
{
    private void Start()
    {
        DemonstrateThreadProperties();
    }

    private void DemonstrateThreadProperties()
    {
        // 현재 스레드 정보
        Thread currentThread = Thread.CurrentThread;
        Debug.Log($"=== 현재 스레드 정보 ===");
        Debug.Log($"ManagedThreadId: {currentThread.ManagedThreadId}");
        Debug.Log($"Name: {currentThread.Name ?? "(none)"}");
        Debug.Log($"IsBackground: {currentThread.IsBackground}");
        Debug.Log($"IsAlive: {currentThread.IsAlive}");
        Debug.Log($"Priority: {currentThread.Priority}");
        Debug.Log($"ThreadState: {currentThread.ThreadState}");
        Debug.Log($"IsThreadPoolThread: {currentThread.IsThreadPoolThread}");

        // 새 스레드 생성 및 설정
        Thread workerThread = new Thread(() =>
        {
            Thread.CurrentThread.Name = "MyWorkerThread";
            Debug.Log($"[Worker] Name: {Thread.CurrentThread.Name}");
            Thread.Sleep(2000);
        });

        // 스레드 속성 설정
        workerThread.Name = "CustomWorker";
        workerThread.Priority = ThreadPriority.BelowNormal;
        workerThread.IsBackground = true; // 앱 종료 시 자동으로 종료됨

        workerThread.Start();

        // 스레드 상태 확인
        Debug.Log($"\n=== Worker 스레드 정보 ===");
        Debug.Log($"Name: {workerThread.Name}");
        Debug.Log($"IsBackground: {workerThread.IsBackground}");
        Debug.Log($"Priority: {workerThread.Priority}");
        Debug.Log($"IsAlive: {workerThread.IsAlive}");
    }
}
```

### Thread 제어 메서드

```csharp
using UnityEngine;
using System;
using System.Threading;

public class ThreadControlExample : MonoBehaviour
{
    private Thread workerThread;
    private volatile bool shouldStop = false;

    private void Start()
    {
        DemonstrateThreadControl();
    }

    private void DemonstrateThreadControl()
    {
        workerThread = new Thread(WorkerMethod);
        workerThread.Start();

        // 3초 후 스레드 중지 요청
        Invoke(nameof(RequestStop), 3f);
    }

    private void WorkerMethod()
    {
        int iteration = 0;

        while (!shouldStop)
        {
            Debug.Log($"[Worker] 반복 {++iteration}");
            Thread.Sleep(500);

            // Thread.Sleep vs Task.Delay 비교:
            // Thread.Sleep: 현재 스레드 블로킹 (CPU 사용 안 함)
            // Task.Delay: 비동기 대기 (스레드 반환)
        }

        Debug.Log("[Worker] 스레드 종료됨");
    }

    private void RequestStop()
    {
        Debug.Log("[Main] 스레드 중지 요청");
        shouldStop = true;
    }

    // =============================================
    // Thread.Join - 스레드 완료 대기
    // =============================================

    private void ThreadJoinExample()
    {
        Thread thread = new Thread(() =>
        {
            Thread.Sleep(1000);
            Debug.Log("[Worker] 작업 완료");
        });

        thread.Start();

        Debug.Log("[Main] Join 호출 - 스레드 완료 대기");
        thread.Join(); // 스레드가 완료될 때까지 현재 스레드 블로킹
        Debug.Log("[Main] 스레드 완료됨");

        // 타임아웃 있는 Join
        Thread thread2 = new Thread(() => Thread.Sleep(5000));
        thread2.Start();

        bool completed = thread2.Join(1000); // 1초 타임아웃
        Debug.Log($"1초 내 완료: {completed}"); // false
    }

    // =============================================
    // ⚠️ Thread.Abort - 사용하지 말 것!
    // =============================================

    /*
    // ❌ Thread.Abort()는 .NET Core / .NET 5+에서 제거됨
    // Unity 2021+에서도 지원되지 않을 수 있음

    private void DangerousAbortExample()
    {
        Thread thread = new Thread(() =>
        {
            while (true)
            {
                // 무한 루프
            }
        });
        thread.Start();

        // ❌ 위험! ThreadAbortException이 발생하며 강제 종료
        // 리소스 정리가 제대로 되지 않을 수 있음
        thread.Abort();
    }
    */

    // =============================================
    // ✅ 올바른 스레드 종료 패턴
    // =============================================

    private CancellationTokenSource cts;

    private void CorrectThreadTermination()
    {
        cts = new CancellationTokenSource();
        var token = cts.Token;

        Thread thread = new Thread(() =>
        {
            while (!token.IsCancellationRequested)
            {
                // 작업 수행
                Thread.Sleep(100);
            }

            Debug.Log("[Worker] 정상적으로 종료됨");
        });

        thread.Start();

        // 종료 요청
        Invoke(nameof(CancelThread), 2f);
    }

    private void CancelThread()
    {
        cts?.Cancel();
    }

    private void OnDestroy()
    {
        // 정리
        shouldStop = true;
        cts?.Cancel();
        cts?.Dispose();
    }
}
```

---

## 2. Foreground vs Background Thread

### 차이점

```csharp
using UnityEngine;
using System.Threading;

public class ForegroundBackgroundExample : MonoBehaviour
{
    private void Start()
    {
        DemonstrateDifference();
    }

    private void DemonstrateDifference()
    {
        // Foreground Thread (기본값)
        Thread foregroundThread = new Thread(() =>
        {
            for (int i = 0; i < 10; i++)
            {
                Debug.Log($"[Foreground] {i}");
                Thread.Sleep(500);
            }
        });
        foregroundThread.IsBackground = false; // 기본값
        // 앱이 종료되어도 이 스레드가 완료될 때까지 프로세스 유지

        // Background Thread
        Thread backgroundThread = new Thread(() =>
        {
            for (int i = 0; i < 10; i++)
            {
                Debug.Log($"[Background] {i}");
                Thread.Sleep(500);
            }
        });
        backgroundThread.IsBackground = true;
        // 앱이 종료되면 즉시 중단됨

        foregroundThread.Start();
        backgroundThread.Start();
    }
}

/*
┌─────────────────────────────────────────────────────────────────┐
│           Foreground vs Background Thread                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Foreground Thread (IsBackground = false, 기본값)                │
│  ├─ 프로세스 종료를 막음                                          │
│  ├─ 모든 Foreground 스레드가 완료되어야 프로세스 종료              │
│  ├─ 중요한 작업에 사용                                            │
│  └─ Unity 메인 스레드는 Foreground                               │
│                                                                  │
│  Background Thread (IsBackground = true)                         │
│  ├─ 프로세스 종료를 막지 않음                                     │
│  ├─ 모든 Foreground 스레드 종료 시 자동으로 종료됨                 │
│  ├─ 중요하지 않은 작업에 사용                                     │
│  └─ ThreadPool 스레드는 Background                               │
│                                                                  │
│  Unity 권장:                                                      │
│  └─ 대부분의 경우 Background Thread 사용 권장                     │
│     (게임 종료 시 깔끔하게 종료되도록)                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
*/
```

---

## 3. ThreadPool

### ThreadPool이란?

ThreadPool은 미리 생성된 스레드들의 집합으로, 스레드 생성/삭제 비용을 줄여줍니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                        ThreadPool 동작                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  직접 Thread 생성:                                               │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                   │
│  │ Create  │ →  │  Work   │ →  │ Destroy │  (비용 큼)          │
│  │ Thread  │     │         │     │ Thread  │                    │
│  └─────────┘     └─────────┘     └─────────┘                   │
│                                                                  │
│  ThreadPool 사용:                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     Thread Pool                          │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐           │   │
│  │  │Worker 1│ │Worker 2│ │Worker 3│ │Worker N│  (대기)    │   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘           │   │
│  └─────────────────────────────────────────────────────────┘   │
│         ▲                                                       │
│         │ 작업 요청                                              │
│         │                                                       │
│  ┌──────────────┐                                               │
│  │  Work Item   │  → Worker가 작업 수행 → 작업 완료 → 대기 복귀   │
│  └──────────────┘                                               │
│                                                                  │
│  장점:                                                           │
│  ├─ 스레드 생성/삭제 비용 절감                                    │
│  ├─ 스레드 수 자동 관리                                          │
│  ├─ CPU 코어 수에 따른 최적화                                    │
│  └─ 짧은 작업에 적합                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### ThreadPool 기본 사용

```csharp
using UnityEngine;
using System;
using System.Threading;

public class ThreadPoolBasicsExample : MonoBehaviour
{
    private void Start()
    {
        // ThreadPool 정보 확인
        CheckThreadPoolInfo();

        // 다양한 ThreadPool 사용 방법
        UseQueueUserWorkItem();
        UseThreadPoolWithCallback();
    }

    private void CheckThreadPoolInfo()
    {
        ThreadPool.GetMinThreads(out int minWorker, out int minIO);
        ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);
        ThreadPool.GetAvailableThreads(out int availWorker, out int availIO);

        Debug.Log("=== ThreadPool 정보 ===");
        Debug.Log($"최소 Worker 스레드: {minWorker}");
        Debug.Log($"최대 Worker 스레드: {maxWorker}");
        Debug.Log($"사용 가능한 Worker 스레드: {availWorker}");
        Debug.Log($"최소 I/O 스레드: {minIO}");
        Debug.Log($"최대 I/O 스레드: {maxIO}");
        Debug.Log($"사용 가능한 I/O 스레드: {availIO}");
    }

    // =============================================
    // QueueUserWorkItem - 기본 사용법
    // =============================================

    private void UseQueueUserWorkItem()
    {
        Debug.Log("\n=== QueueUserWorkItem ===");

        // 방법 1: WaitCallback 델리게이트
        ThreadPool.QueueUserWorkItem(new WaitCallback(WorkerCallback));

        // 방법 2: Lambda (권장)
        ThreadPool.QueueUserWorkItem(state =>
        {
            int threadId = Thread.CurrentThread.ManagedThreadId;
            Debug.Log($"[ThreadPool Lambda] Thread ID: {threadId}");
            Debug.Log($"[ThreadPool Lambda] IsThreadPoolThread: {Thread.CurrentThread.IsThreadPoolThread}");
        });

        // 방법 3: 매개변수 전달
        ThreadPool.QueueUserWorkItem(state =>
        {
            string data = state as string;
            Debug.Log($"[ThreadPool] 받은 데이터: {data}");
        }, "Hello ThreadPool!");
    }

    private void WorkerCallback(object state)
    {
        Debug.Log($"[WorkerCallback] Thread ID: {Thread.CurrentThread.ManagedThreadId}");
        Thread.Sleep(100);
        Debug.Log("[WorkerCallback] 완료");
    }

    // =============================================
    // RegisterWaitForSingleObject - 이벤트 기반
    // =============================================

    private void UseThreadPoolWithCallback()
    {
        Debug.Log("\n=== RegisterWaitForSingleObject ===");

        AutoResetEvent waitHandle = new AutoResetEvent(false);
        RegisteredWaitHandle registeredHandle = null;

        registeredHandle = ThreadPool.RegisterWaitForSingleObject(
            waitHandle,
            (state, timedOut) =>
            {
                if (timedOut)
                {
                    Debug.Log("[Register] 타임아웃!");
                }
                else
                {
                    Debug.Log("[Register] 이벤트 수신!");
                }

                // 등록 해제
                registeredHandle?.Unregister(null);
            },
            null,
            TimeSpan.FromSeconds(5), // 타임아웃
            true // 한 번만 실행
        );

        // 2초 후 이벤트 발생
        ThreadPool.QueueUserWorkItem(_ =>
        {
            Thread.Sleep(2000);
            Debug.Log("[Trigger] 이벤트 발생!");
            waitHandle.Set();
        });
    }
}
```

### ThreadPool 설정

```csharp
using UnityEngine;
using System.Threading;

public class ThreadPoolConfigExample : MonoBehaviour
{
    private void Start()
    {
        // ⚠️ ThreadPool 설정 변경은 신중하게!
        // Unity와 .NET 런타임이 이미 최적화된 설정을 사용 중

        ConfigureThreadPool();
    }

    private void ConfigureThreadPool()
    {
        // 현재 설정 확인
        ThreadPool.GetMinThreads(out int minWorker, out int minIO);
        ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);

        Debug.Log($"변경 전 - Min: {minWorker}/{minIO}, Max: {maxWorker}/{maxIO}");

        // 최소 스레드 수 변경
        // 최소값을 높이면 스레드가 미리 생성되어 첫 작업이 빠름
        // 단, 메모리 사용량 증가
        bool success = ThreadPool.SetMinThreads(
            Environment.ProcessorCount * 2, // Worker 스레드
            Environment.ProcessorCount      // I/O 스레드
        );
        Debug.Log($"SetMinThreads 성공: {success}");

        // 최대 스레드 수 변경
        // 너무 높으면 컨텍스트 스위칭 오버헤드 증가
        // success = ThreadPool.SetMaxThreads(100, 100);

        // 변경 후 확인
        ThreadPool.GetMinThreads(out minWorker, out minIO);
        Debug.Log($"변경 후 - Min: {minWorker}/{minIO}");
    }
}
```

---

## 4. Thread vs ThreadPool vs Task.Run 비교

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Diagnostics;

public class ThreadComparisonExample : MonoBehaviour
{
    private const int WORK_COUNT = 1000;

    private void Start()
    {
        ComparePerformance();
        CompareFeatures();
    }

    private async void ComparePerformance()
    {
        Debug.Log("=== 성능 비교 (1000개 작업) ===\n");

        // 1. 직접 Thread 생성
        var sw = Stopwatch.StartNew();
        Thread[] threads = new Thread[WORK_COUNT];
        for (int i = 0; i < WORK_COUNT; i++)
        {
            threads[i] = new Thread(() => SmallWork());
            threads[i].Start();
        }
        foreach (var t in threads) t.Join();
        sw.Stop();
        Debug.Log($"직접 Thread 생성: {sw.ElapsedMilliseconds}ms");

        // 2. ThreadPool
        sw.Restart();
        CountdownEvent countdown1 = new CountdownEvent(WORK_COUNT);
        for (int i = 0; i < WORK_COUNT; i++)
        {
            ThreadPool.QueueUserWorkItem(_ =>
            {
                SmallWork();
                countdown1.Signal();
            });
        }
        countdown1.Wait();
        sw.Stop();
        Debug.Log($"ThreadPool: {sw.ElapsedMilliseconds}ms");

        // 3. Task.Run
        sw.Restart();
        Task[] tasks = new Task[WORK_COUNT];
        for (int i = 0; i < WORK_COUNT; i++)
        {
            tasks[i] = Task.Run(() => SmallWork());
        }
        await Task.WhenAll(tasks);
        sw.Stop();
        Debug.Log($"Task.Run: {sw.ElapsedMilliseconds}ms");
    }

    private void SmallWork()
    {
        // 매우 짧은 작업
        int sum = 0;
        for (int i = 0; i < 100; i++)
        {
            sum += i;
        }
    }

    private void CompareFeatures()
    {
        Debug.Log("\n=== 기능 비교 ===");

        /*
        ┌──────────────────────────────────────────────────────────────────────────┐
        │                    Thread vs ThreadPool vs Task.Run                       │
        ├─────────────┬───────────────────┬─────────────────┬──────────────────────┤
        │   특성       │   new Thread()   │   ThreadPool    │   Task.Run           │
        ├─────────────┼───────────────────┼─────────────────┼──────────────────────┤
        │ 스레드 생성  │ 매번 새로 생성     │ 재사용          │ 재사용 (ThreadPool)   │
        │ 생성 비용    │ 높음              │ 낮음            │ 낮음                  │
        │ 스레드 제어  │ 완전한 제어        │ 제한적          │ 제한적               │
        │ 반환값       │ 없음              │ 없음            │ Task<T>로 가능       │
        │ 취소         │ 수동 구현          │ 수동 구현        │ CancellationToken    │
        │ 예외 처리    │ 스레드 내부에서    │ 스레드 내부에서  │ await에서 처리 가능   │
        │ 연속 작업    │ 수동 구현          │ 수동 구현        │ ContinueWith, await  │
        │ async/await │ 사용 불가          │ 사용 불가        │ 완벽 지원             │
        │ 진행률 보고  │ 수동 구현          │ 수동 구현        │ IProgress<T>         │
        │ 사용 권장    │ 장기 실행 작업     │ 짧은 작업        │ 대부분의 경우         │
        └─────────────┴───────────────────┴─────────────────┴──────────────────────┘
        */

        // Task.Run의 장점 시연
        DemonstrateTaskRunAdvantages();
    }

    private async void DemonstrateTaskRunAdvantages()
    {
        Debug.Log("\n--- Task.Run 장점 시연 ---");

        // 1. 반환값
        int result = await Task.Run(() =>
        {
            int sum = 0;
            for (int i = 0; i < 1000; i++) sum += i;
            return sum;
        });
        Debug.Log($"반환값: {result}");

        // 2. 취소
        var cts = new CancellationTokenSource();
        cts.CancelAfter(100);
        try
        {
            await Task.Run(() =>
            {
                while (true)
                {
                    cts.Token.ThrowIfCancellationRequested();
                    Thread.Sleep(10);
                }
            }, cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업이 취소됨");
        }

        // 3. 예외 처리
        try
        {
            await Task.Run(() => throw new InvalidOperationException("테스트 에러"));
        }
        catch (InvalidOperationException e)
        {
            Debug.Log($"예외 처리됨: {e.Message}");
        }

        // 4. 연속 작업
        await Task.Run(() => Debug.Log("첫 번째 작업"))
            .ContinueWith(_ => Debug.Log("두 번째 작업"));
    }
}

/*
일반적인 결과 예시:
=== 성능 비교 (1000개 작업) ===
직접 Thread 생성: 1500ms   (스레드 생성 비용)
ThreadPool: 50ms          (재사용으로 빠름)
Task.Run: 55ms            (ThreadPool 기반 + 약간의 오버헤드)
*/
```

---

## 5. Unity에서의 Thread 제약사항

### Unity API와 Thread

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;
using System.Collections.Concurrent;

public class UnityThreadRestrictionsExample : MonoBehaviour
{
    private ConcurrentQueue<System.Action> mainThreadQueue = new ConcurrentQueue<System.Action>();
    private int mainThreadId;

    private void Awake()
    {
        mainThreadId = Thread.CurrentThread.ManagedThreadId;
    }

    private void Update()
    {
        // 메인 스레드에서 큐 처리
        while (mainThreadQueue.TryDequeue(out var action))
        {
            action?.Invoke();
        }
    }

    private void Start()
    {
        DemonstrateRestrictions();
        DemonstrateWorkarounds();
    }

    // =============================================
    // Unity API 제약사항
    // =============================================

    private void DemonstrateRestrictions()
    {
        Debug.Log("=== Unity Thread 제약사항 ===\n");

        Task.Run(() =>
        {
            // ❌ 모두 UnityException 발생!

            // Transform 접근
            // var pos = transform.position;

            // GameObject 접근
            // var go = gameObject;
            // var name = gameObject.name;
            // gameObject.SetActive(false);

            // GameObject 찾기/생성
            // var player = GameObject.Find("Player");
            // var newObj = new GameObject("Test");
            // Instantiate(prefab);
            // Destroy(gameObject);

            // Component 접근
            // var rb = GetComponent<Rigidbody>();

            // Physics
            // Physics.Raycast(Vector3.zero, Vector3.forward);

            // Scene 관리
            // SceneManager.LoadScene("MyScene");

            // Resources
            // var asset = Resources.Load<Texture2D>("texture");

            // Debug.Log는 스레드 안전하지만 과도한 사용은 비권장
            UnityEngine.Debug.Log("[Task.Run] 이것은 동작함 (권장하지 않음)");
        });
    }

    // =============================================
    // 해결 방법
    // =============================================

    private async void DemonstrateWorkarounds()
    {
        Debug.Log("\n=== 해결 방법 ===\n");

        // 방법 1: 메인 스레드에서 데이터 준비 → 백그라운드 처리 → 메인 스레드에서 적용
        Vector3 currentPos = transform.position; // 메인 스레드에서 읽기

        Vector3 newPos = await Task.Run(() =>
        {
            // 백그라운드에서 계산 (Unity API 사용 안 함)
            float x = Mathf.Sin(Time.time) * 5f; // ⚠️ Time.time은 캡처된 값 사용
            float z = Mathf.Cos(Time.time) * 5f;
            return new Vector3(x, currentPos.y, z);
        });

        transform.position = newPos; // 메인 스레드에서 적용

        // 방법 2: 액션 큐 사용
        Task.Run(() =>
        {
            // 백그라운드 연산
            int result = HeavyCalculation();

            // 메인 스레드에서 실행할 작업 큐에 추가
            mainThreadQueue.Enqueue(() =>
            {
                // ✅ 메인 스레드에서 실행됨
                transform.position = new Vector3(result, 0, 0);
                Debug.Log($"큐를 통한 업데이트: {result}");
            });
        });

        // 방법 3: SynchronizationContext 사용 (이전 섹션 참조)
    }

    private int HeavyCalculation()
    {
        int sum = 0;
        for (int i = 0; i < 1000000; i++)
        {
            sum += i % 100;
        }
        return sum;
    }
}

/*
┌─────────────────────────────────────────────────────────────────┐
│              Unity에서 백그라운드 스레드 사용 패턴                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  메인 스레드                    백그라운드 스레드                  │
│  ─────────────                  ────────────────                  │
│                                                                  │
│  1. 필요한 데이터 준비                                           │
│     (Unity API로 값 읽기)                                        │
│          │                                                       │
│          ▼                                                       │
│  2. Task.Run으로 작업 전달 ──────▶ 3. 순수 C# 연산 수행          │
│                                       (Unity API 사용 금지)      │
│          │                                   │                   │
│          │◀──────────────── 결과 반환 ◀──────┘                   │
│          ▼                                                       │
│  4. await 후 결과 적용                                           │
│     (Unity API로 값 설정)                                        │
│                                                                  │
│  허용되는 것:                                                    │
│  ├─ 수학 연산 (Mathf 일부)                                       │
│  ├─ 순수 C# 데이터 처리                                          │
│  ├─ 파일 I/O (System.IO)                                        │
│  ├─ 네트워크 (System.Net)                                       │
│  └─ 컬렉션 조작                                                  │
│                                                                  │
│  금지되는 것:                                                    │
│  ├─ Transform, GameObject                                       │
│  ├─ Component 접근                                               │
│  ├─ Physics, Raycast                                            │
│  ├─ Scene 관리                                                   │
│  ├─ Resources 로드                                               │
│  └─ 대부분의 UnityEngine API                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
*/
```

---

## 6. 실전 예제: 백그라운드 경로 탐색

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Collections.Generic;
using System.Collections.Concurrent;

public class BackgroundPathfindingExample : MonoBehaviour
{
    [SerializeField] private Transform target;

    private CancellationTokenSource cts;
    private ConcurrentQueue<Vector3[]> pathResults = new ConcurrentQueue<Vector3[]>();
    private Vector3[] currentPath;

    private void Start()
    {
        cts = new CancellationTokenSource();
        StartPeriodicPathfinding(cts.Token);
    }

    private async void StartPeriodicPathfinding(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            try
            {
                // 메인 스레드에서 현재 위치 캡처
                Vector3 start = transform.position;
                Vector3 end = target != null ? target.position : Vector3.zero;

                // 백그라운드에서 경로 계산
                Vector3[] path = await Task.Run(() =>
                    CalculatePath(start, end, token), token);

                // 메인 스레드에서 결과 적용
                if (path != null && !token.IsCancellationRequested)
                {
                    currentPath = path;
                    Debug.Log($"경로 계산 완료: {path.Length} 포인트");
                }

                await Task.Delay(500, token); // 0.5초마다 재계산
            }
            catch (OperationCanceledException)
            {
                Debug.Log("경로 탐색 취소됨");
                break;
            }
        }
    }

    // 백그라운드에서 실행되는 경로 계산 (Unity API 사용 안 함)
    private Vector3[] CalculatePath(Vector3 start, Vector3 end, CancellationToken token)
    {
        List<Vector3> path = new List<Vector3>();
        path.Add(start);

        // 간단한 직선 경로 시뮬레이션 (실제로는 A* 등 사용)
        int steps = 10;
        for (int i = 1; i <= steps; i++)
        {
            token.ThrowIfCancellationRequested();

            float t = (float)i / steps;
            Vector3 point = Vector3.Lerp(start, end, t);
            path.Add(point);

            // 무거운 연산 시뮬레이션
            Thread.Sleep(10);
        }

        return path.ToArray();
    }

    private void OnDrawGizmos()
    {
        // 디버그용 경로 시각화
        if (currentPath != null && currentPath.Length > 1)
        {
            Gizmos.color = Color.green;
            for (int i = 0; i < currentPath.Length - 1; i++)
            {
                Gizmos.DrawLine(currentPath[i], currentPath[i + 1]);
            }
        }
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }
}
```

---

## 7. 베스트 프랙티스

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class ThreadBestPracticesExample : MonoBehaviour
{
    /*
    ┌─────────────────────────────────────────────────────────────────┐
    │                   Thread 사용 베스트 프랙티스                     │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  1. Task.Run 우선 사용                                          │
    │     └─ new Thread() 대신 Task.Run 사용                          │
    │     └─ 취소, 예외 처리, 연속 작업 지원                           │
    │                                                                  │
    │  2. 항상 취소 지원                                               │
    │     └─ CancellationToken 전달                                   │
    │     └─ OnDestroy에서 취소 호출                                   │
    │                                                                  │
    │  3. Background Thread 설정                                      │
    │     └─ IsBackground = true 설정                                 │
    │     └─ 앱 종료 시 정상 종료 보장                                 │
    │                                                                  │
    │  4. Unity API 호출 금지                                         │
    │     └─ 백그라운드 스레드에서 Unity API 사용 금지                  │
    │     └─ 필요 시 메인 스레드로 마샬링                              │
    │                                                                  │
    │  5. 공유 리소스 동기화                                          │
    │     └─ lock, Interlocked 사용                                  │
    │     └─ Concurrent 컬렉션 활용                                   │
    │                                                                  │
    │  6. 예외 처리                                                    │
    │     └─ try-catch로 감싸기                                       │
    │     └─ 예외가 스레드를 종료시키지 않도록                          │
    │                                                                  │
    │  7. 스레드 풀 설정 변경 자제                                     │
    │     └─ 기본 설정이 대부분의 경우 최적                            │
    │                                                                  │
    │  8. 장기 실행 작업 분리                                         │
    │     └─ ThreadPool 대신 전용 Thread 사용                         │
    │     └─ TaskCreationOptions.LongRunning 사용                    │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
    */

    private CancellationTokenSource cts;

    private void Start()
    {
        cts = new CancellationTokenSource();

        // 좋은 예: Task.Run + CancellationToken
        GoodExample(cts.Token);

        // 장기 실행 작업
        LongRunningTaskExample(cts.Token);
    }

    private async void GoodExample(CancellationToken token)
    {
        try
        {
            // 메인 스레드에서 데이터 준비
            Vector3 pos = transform.position;

            // 백그라운드 연산
            var result = await Task.Run(() =>
            {
                // 취소 확인
                token.ThrowIfCancellationRequested();

                // 순수 C# 연산만
                return PerformCalculation(pos);

            }, token);

            // 결과 적용 (메인 스레드)
            transform.position = result;
        }
        catch (OperationCanceledException)
        {
            Debug.Log("작업 취소됨");
        }
        catch (Exception e)
        {
            Debug.LogError($"에러: {e.Message}");
        }
    }

    private Vector3 PerformCalculation(Vector3 input)
    {
        // 무거운 연산
        Thread.Sleep(100);
        return input + Vector3.up;
    }

    private void LongRunningTaskExample(CancellationToken token)
    {
        // 장기 실행 작업은 LongRunning 옵션 사용
        // ThreadPool을 점유하지 않고 전용 스레드 생성
        Task.Factory.StartNew(() =>
        {
            while (!token.IsCancellationRequested)
            {
                // 긴 작업...
                Thread.Sleep(1000);
            }
        }, token, TaskCreationOptions.LongRunning, TaskScheduler.Default);
    }

    private void OnDestroy()
    {
        // 필수: 취소 및 정리
        cts?.Cancel();
        cts?.Dispose();
    }
}
```

---

## 주의사항

1. **Unity API 제약**: 백그라운드 스레드에서 Unity API 호출 불가
2. **스레드 풀 고갈**: 너무 많은 장기 작업을 ThreadPool에 넣으면 고갈 위험
3. **메모리 누수**: 취소하지 않은 스레드가 계속 실행될 수 있음
4. **예외 처리**: 스레드에서 발생한 예외는 별도 처리 필요
5. **데드락 주의**: 메인 스레드를 블로킹하면서 메인 스레드 작업 대기 시 데드락

---

## 참고 자료

- [Microsoft: Threading in C#](https://docs.microsoft.com/en-us/dotnet/standard/threading/)
- [Microsoft: ThreadPool Class](https://docs.microsoft.com/en-us/dotnet/api/system.threading.threadpool)
- [Unity: Scripting Backends](https://docs.unity3d.com/Manual/scripting-backends.html)

---

## 다음 섹션

[05. 동기화 기법](./05-synchronization-primitives.md)
