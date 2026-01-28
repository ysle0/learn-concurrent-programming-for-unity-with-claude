# 05. 동기화 기법 (Synchronization Primitives)

## 개요

멀티스레드 환경에서 여러 스레드가 동시에 공유 리소스에 접근하면 데이터 손상, 레이스 컨디션 등의 문제가 발생할 수 있습니다. 동기화 기법은 이러한 문제를 해결하여 스레드 안전성을 보장합니다.

---

## 1. 왜 동기화가 필요한가?

### 레이스 컨디션 (Race Condition)

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class RaceConditionExample : MonoBehaviour
{
    private int counter = 0;
    private const int ITERATIONS = 100000;

    private void Start()
    {
        DemonstrateRaceCondition();
    }

    private async void DemonstrateRaceCondition()
    {
        counter = 0;

        // 두 태스크가 동시에 counter를 증가
        var task1 = Task.Run(() => IncrementCounter());
        var task2 = Task.Run(() => IncrementCounter());

        await Task.WhenAll(task1, task2);

        // 예상: 200000
        // 실제: 무작위 값 (예: 156234, 189012 등)
        Debug.Log($"최종 counter 값: {counter}");
        Debug.Log($"예상 값: {ITERATIONS * 2}");
        Debug.Log($"손실된 증가: {ITERATIONS * 2 - counter}");
    }

    private void IncrementCounter()
    {
        for (int i = 0; i < ITERATIONS; i++)
        {
            // ❌ 레이스 컨디션 발생!
            // counter++ 는 실제로 3단계:
            // 1. counter 값 읽기
            // 2. 값에 1 더하기
            // 3. counter에 저장
            counter++;
        }
    }
}

/*
레이스 컨디션 발생 원리:

Thread 1                    Thread 2
─────────                   ─────────
읽기: counter = 100
                           읽기: counter = 100
더하기: 100 + 1 = 101
저장: counter = 101
                           더하기: 100 + 1 = 101
                           저장: counter = 101

→ 두 번 증가했지만 counter는 101 (1회 손실!)
*/
```

### 데이터 손상 예제

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;
using System.Collections.Generic;

public class DataCorruptionExample : MonoBehaviour
{
    private List<int> sharedList = new List<int>();

    private async void Start()
    {
        // ❌ List<T>는 스레드 안전하지 않음!
        sharedList.Clear();

        var tasks = new Task[10];
        for (int i = 0; i < 10; i++)
        {
            int taskId = i;
            tasks[i] = Task.Run(() =>
            {
                for (int j = 0; j < 1000; j++)
                {
                    // 동시에 여러 스레드가 Add 호출 시 예외 또는 데이터 손상
                    try
                    {
                        sharedList.Add(taskId * 1000 + j);
                    }
                    catch (System.Exception e)
                    {
                        Debug.LogError($"예외 발생: {e.Message}");
                    }
                }
            });
        }

        await Task.WhenAll(tasks);

        // 예상: 10000, 실제: 무작위 (또는 예외)
        Debug.Log($"리스트 크기: {sharedList.Count}");
    }
}
```

---

## 2. lock 키워드

`lock`은 C#에서 가장 기본적이고 많이 사용되는 동기화 메커니즘입니다.

### 기본 사용법

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class LockBasicsExample : MonoBehaviour
{
    private int counter = 0;
    private readonly object lockObject = new object();
    private const int ITERATIONS = 100000;

    private void Start()
    {
        DemonstrateLock();
    }

    private async void DemonstrateLock()
    {
        counter = 0;

        var task1 = Task.Run(() => SafeIncrementCounter());
        var task2 = Task.Run(() => SafeIncrementCounter());

        await Task.WhenAll(task1, task2);

        // ✅ 항상 200000
        Debug.Log($"최종 counter 값: {counter}");
        Debug.Log($"예상 값: {ITERATIONS * 2}");
    }

    private void SafeIncrementCounter()
    {
        for (int i = 0; i < ITERATIONS; i++)
        {
            // ✅ lock으로 동기화
            lock (lockObject)
            {
                counter++;
            }
        }
    }
}

/*
lock의 동작:

Thread 1                    Thread 2
─────────                   ─────────
lock 획득
  읽기: counter = 100
  더하기: 100 + 1 = 101
  저장: counter = 101
lock 해제
                           lock 획득
                             읽기: counter = 101
                             더하기: 101 + 1 = 102
                             저장: counter = 102
                           lock 해제

→ 순차적으로 실행되어 데이터 일관성 보장
*/
```

### lock 객체 선택 가이드

```csharp
using UnityEngine;
using System;

public class LockObjectGuideExample : MonoBehaviour
{
    // ✅ 좋은 예: 전용 lock 객체
    private readonly object _lock = new object();

    // ✅ 좋은 예: static 리소스용 static lock
    private static readonly object _staticLock = new object();

    private int counter = 0;
    private static int staticCounter = 0;

    private void GoodLockExamples()
    {
        // ✅ 전용 객체 사용
        lock (_lock)
        {
            counter++;
        }

        // ✅ static 데이터는 static lock 사용
        lock (_staticLock)
        {
            staticCounter++;
        }
    }

    private void BadLockExamples()
    {
        // ❌ 나쁜 예 1: this 사용
        // 외부에서 같은 인스턴스로 lock 할 수 있어 데드락 위험
        lock (this)
        {
            counter++;
        }

        // ❌ 나쁜 예 2: Type 객체 사용
        // 전체 앱에서 공유되어 성능 저하 및 데드락 위험
        lock (typeof(LockObjectGuideExample))
        {
            counter++;
        }

        // ❌ 나쁜 예 3: string 리터럴 사용
        // string interning으로 예상치 못한 공유 발생
        lock ("myLock")
        {
            counter++;
        }

        // ❌ 나쁜 예 4: 값 타입 사용 (컴파일 에러)
        // int lockValue = 0;
        // lock (lockValue) { } // 에러: 값 타입은 lock 불가

        // ❌ 나쁜 예 5: null 가능 객체 사용
        object maybeNull = null;
        // lock (maybeNull) { } // NullReferenceException
    }
}

/*
┌─────────────────────────────────────────────────────────────────┐
│                    lock 객체 선택 가이드                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ✅ 권장                                                         │
│  ├─ private readonly object _lock = new object();               │
│  ├─ private static readonly object _staticLock = new object();  │
│  └─ 전용 락 객체 사용                                            │
│                                                                  │
│  ❌ 비권장                                                       │
│  ├─ lock(this) - 외부에서 접근 가능                              │
│  ├─ lock(typeof(T)) - 전역 공유                                  │
│  ├─ lock("string") - string interning                          │
│  └─ lock(공개 객체) - 외부 간섭 가능                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
*/
```

### lock과 예외 처리

```csharp
using UnityEngine;
using System;
using System.Threading;

public class LockExceptionExample : MonoBehaviour
{
    private readonly object _lock = new object();
    private int value = 0;

    private void SafeWithException()
    {
        lock (_lock)
        {
            try
            {
                value++;
                // 예외가 발생해도...
                throw new InvalidOperationException("테스트 예외");
            }
            catch (InvalidOperationException)
            {
                // 예외 처리
                Debug.Log("예외 처리됨");
            }
            // lock은 자동으로 해제됨 (finally에서 Monitor.Exit 호출)
        }
    }

    // lock은 내부적으로 다음과 같이 동작:
    private void LockInternals()
    {
        bool lockTaken = false;
        try
        {
            Monitor.Enter(_lock, ref lockTaken);

            // critical section
            value++;
        }
        finally
        {
            if (lockTaken)
            {
                Monitor.Exit(_lock);
            }
        }
    }
}
```

---

## 3. Monitor 클래스

`Monitor`는 `lock` 키워드의 기반이 되는 클래스로, 더 세밀한 제어가 가능합니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class MonitorExample : MonoBehaviour
{
    private readonly object _lock = new object();
    private int sharedResource = 0;

    private void Start()
    {
        BasicMonitorUsage();
        TryEnterExample();
        WaitPulseExample();
    }

    // =============================================
    // 기본 사용법
    // =============================================

    private void BasicMonitorUsage()
    {
        bool lockTaken = false;
        try
        {
            // lock 획득
            Monitor.Enter(_lock, ref lockTaken);

            // Critical section
            sharedResource++;
        }
        finally
        {
            // lock 해제 (반드시 해제해야 함)
            if (lockTaken)
            {
                Monitor.Exit(_lock);
            }
        }
    }

    // =============================================
    // TryEnter - 타임아웃 지원
    // =============================================

    private void TryEnterExample()
    {
        // lock 획득 시도 (최대 1초 대기)
        bool lockTaken = Monitor.TryEnter(_lock, TimeSpan.FromSeconds(1));

        if (lockTaken)
        {
            try
            {
                Debug.Log("lock 획득 성공!");
                sharedResource++;
            }
            finally
            {
                Monitor.Exit(_lock);
            }
        }
        else
        {
            Debug.Log("lock 획득 실패 (타임아웃)");
        }
    }

    // =============================================
    // Wait/Pulse - 스레드 간 신호 전달
    // =============================================

    private bool dataReady = false;

    private void WaitPulseExample()
    {
        // Producer
        Task.Run(() =>
        {
            Thread.Sleep(1000); // 데이터 준비 시뮬레이션

            lock (_lock)
            {
                dataReady = true;
                Debug.Log("[Producer] 데이터 준비 완료, Pulse 전송");
                Monitor.Pulse(_lock); // 대기 중인 스레드에 신호
            }
        });

        // Consumer
        Task.Run(() =>
        {
            lock (_lock)
            {
                while (!dataReady)
                {
                    Debug.Log("[Consumer] 데이터 대기 중...");
                    Monitor.Wait(_lock); // lock 해제하고 대기
                }
                Debug.Log("[Consumer] 데이터 처리!");
            }
        });
    }
}
```

### Producer-Consumer 패턴

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Collections.Generic;

public class ProducerConsumerExample : MonoBehaviour
{
    private Queue<int> queue = new Queue<int>();
    private readonly object _lock = new object();
    private const int MAX_SIZE = 10;
    private bool isRunning = true;

    private void Start()
    {
        // Producer 시작
        Task.Run(() => Producer());

        // Consumer 시작
        Task.Run(() => Consumer());

        // 5초 후 종료
        Invoke(nameof(Stop), 5f);
    }

    private void Producer()
    {
        int item = 0;
        while (isRunning)
        {
            lock (_lock)
            {
                // 큐가 가득 차면 대기
                while (queue.Count >= MAX_SIZE && isRunning)
                {
                    Debug.Log("[Producer] 큐가 가득 참, 대기...");
                    Monitor.Wait(_lock);
                }

                if (!isRunning) break;

                queue.Enqueue(item);
                Debug.Log($"[Producer] 생산: {item}, 큐 크기: {queue.Count}");
                item++;

                // Consumer에게 알림
                Monitor.PulseAll(_lock);
            }

            Thread.Sleep(100); // 생산 속도 조절
        }
    }

    private void Consumer()
    {
        while (isRunning)
        {
            lock (_lock)
            {
                // 큐가 비어있으면 대기
                while (queue.Count == 0 && isRunning)
                {
                    Debug.Log("[Consumer] 큐가 비어있음, 대기...");
                    Monitor.Wait(_lock);
                }

                if (!isRunning && queue.Count == 0) break;

                int item = queue.Dequeue();
                Debug.Log($"[Consumer] 소비: {item}, 큐 크기: {queue.Count}");

                // Producer에게 알림
                Monitor.PulseAll(_lock);
            }

            Thread.Sleep(200); // 소비 속도 조절
        }
    }

    private void Stop()
    {
        lock (_lock)
        {
            isRunning = false;
            Monitor.PulseAll(_lock);
        }
    }
}
```

---

## 4. Mutex

`Mutex`는 프로세스 간 동기화도 지원하는 동기화 프리미티브입니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class MutexExample : MonoBehaviour
{
    // =============================================
    // 로컬 Mutex (프로세스 내)
    // =============================================

    private Mutex localMutex = new Mutex();

    private void LocalMutexExample()
    {
        try
        {
            // Mutex 획득 (무한 대기)
            localMutex.WaitOne();

            Debug.Log("Mutex 획득됨");
            // Critical section

        }
        finally
        {
            localMutex.ReleaseMutex();
        }
    }

    // 타임아웃 지원
    private void MutexWithTimeout()
    {
        // 1초 동안 대기
        if (localMutex.WaitOne(TimeSpan.FromSeconds(1)))
        {
            try
            {
                Debug.Log("Mutex 획득 성공");
            }
            finally
            {
                localMutex.ReleaseMutex();
            }
        }
        else
        {
            Debug.Log("Mutex 획득 실패 (타임아웃)");
        }
    }

    // =============================================
    // 명명된 Mutex (프로세스 간 동기화)
    // =============================================

    private void NamedMutexExample()
    {
        // 전역 이름을 가진 Mutex - 다른 프로세스에서도 접근 가능
        using (var namedMutex = new Mutex(false, "Global\\MyUnityGameMutex"))
        {
            try
            {
                if (namedMutex.WaitOne(TimeSpan.FromSeconds(5)))
                {
                    Debug.Log("명명된 Mutex 획득 - 단일 인스턴스 실행");
                    // 게임 로직...
                    Thread.Sleep(2000);
                }
                else
                {
                    Debug.Log("이미 다른 인스턴스가 실행 중");
                }
            }
            finally
            {
                namedMutex.ReleaseMutex();
            }
        }
    }

    // =============================================
    // 단일 인스턴스 게임 보장
    // =============================================

    private static Mutex singleInstanceMutex;

    private void Awake()
    {
        // 게임 시작 시 단일 인스턴스 확인
        bool createdNew;
        singleInstanceMutex = new Mutex(true, "MyUniqueGameIdentifier", out createdNew);

        if (!createdNew)
        {
            Debug.LogError("게임이 이미 실행 중입니다!");
            Application.Quit();
            return;
        }

        Debug.Log("게임 단일 인스턴스 확인 완료");
    }

    private void OnApplicationQuit()
    {
        singleInstanceMutex?.ReleaseMutex();
        singleInstanceMutex?.Dispose();
    }
}

/*
┌─────────────────────────────────────────────────────────────────┐
│                   lock vs Mutex 비교                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  특성              │     lock        │      Mutex                │
│  ─────────────────┼─────────────────┼───────────────────────────│
│  범위              │ 프로세스 내     │ 프로세스 간 가능           │
│  성능              │ 빠름            │ 상대적으로 느림            │
│  사용 편의성       │ 간단            │ 복잡                       │
│  타임아웃          │ Monitor 필요    │ 기본 지원                  │
│  이름 지정         │ 불가            │ 가능 (전역 동기화)          │
│  IDisposable      │ 아니오          │ 예                         │
│                                                                  │
│  권장 사용:                                                       │
│  └─ 대부분의 경우 lock 사용                                       │
│  └─ 프로세스 간 동기화가 필요할 때만 Mutex 사용                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
*/
```

---

## 5. Semaphore / SemaphoreSlim

`Semaphore`는 동시에 접근할 수 있는 스레드 수를 제한합니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;

public class SemaphoreExample : MonoBehaviour
{
    // 최대 3개 스레드만 동시 접근 가능
    private SemaphoreSlim semaphore = new SemaphoreSlim(3, 3);

    private void Start()
    {
        DemonstrateSemaphore();
        ResourcePoolExample();
    }

    // =============================================
    // 기본 사용법
    // =============================================

    private async void DemonstrateSemaphore()
    {
        Debug.Log("=== Semaphore 데모 (최대 3개 동시 실행) ===");

        var tasks = new Task[10];

        for (int i = 0; i < 10; i++)
        {
            int taskId = i;
            tasks[i] = ProcessWithSemaphoreAsync(taskId);
        }

        await Task.WhenAll(tasks);
        Debug.Log("모든 작업 완료");
    }

    private async Task ProcessWithSemaphoreAsync(int taskId)
    {
        Debug.Log($"Task {taskId}: 세마포어 대기 중...");

        // 세마포어 획득 (대기)
        await semaphore.WaitAsync();

        try
        {
            Debug.Log($"Task {taskId}: 실행 중... (현재 카운트: {semaphore.CurrentCount})");
            await Task.Delay(1000); // 작업 시뮬레이션
            Debug.Log($"Task {taskId}: 완료");
        }
        finally
        {
            // 세마포어 해제
            semaphore.Release();
        }
    }

    // =============================================
    // 리소스 풀 패턴
    // =============================================

    // 동시에 5개의 네트워크 연결만 허용
    private SemaphoreSlim connectionPool = new SemaphoreSlim(5, 5);

    private async void ResourcePoolExample()
    {
        Debug.Log("\n=== 연결 풀 데모 (최대 5개 연결) ===");

        var downloadTasks = new Task[20];

        for (int i = 0; i < 20; i++)
        {
            int fileId = i;
            downloadTasks[i] = DownloadFileAsync($"file_{fileId}.dat");
        }

        await Task.WhenAll(downloadTasks);
        Debug.Log("모든 다운로드 완료");
    }

    private async Task DownloadFileAsync(string fileName)
    {
        // 연결 획득 대기
        await connectionPool.WaitAsync();

        try
        {
            Debug.Log($"[Download] {fileName} 시작 (활성 연결: {5 - connectionPool.CurrentCount})");
            await Task.Delay(500); // 다운로드 시뮬레이션
            Debug.Log($"[Download] {fileName} 완료");
        }
        finally
        {
            connectionPool.Release();
        }
    }

    // =============================================
    // 타임아웃 지원
    // =============================================

    private async Task WithTimeoutAsync()
    {
        // 2초 타임아웃
        if (await semaphore.WaitAsync(TimeSpan.FromSeconds(2)))
        {
            try
            {
                // 작업 수행
            }
            finally
            {
                semaphore.Release();
            }
        }
        else
        {
            Debug.Log("세마포어 획득 타임아웃");
        }
    }

    private void OnDestroy()
    {
        semaphore?.Dispose();
        connectionPool?.Dispose();
    }
}

/*
출력 예시:
=== Semaphore 데모 (최대 3개 동시 실행) ===
Task 0: 세마포어 대기 중...
Task 1: 세마포어 대기 중...
Task 2: 세마포어 대기 중...
Task 0: 실행 중... (현재 카운트: 0)
Task 1: 실행 중... (현재 카운트: 0)
Task 2: 실행 중... (현재 카운트: 0)
Task 3: 세마포어 대기 중...  ← 대기
Task 4: 세마포어 대기 중...  ← 대기
...
Task 0: 완료
Task 3: 실행 중... (현재 카운트: 0)  ← Task 0 완료 후 실행
...
*/
```

---

## 6. Interlocked

`Interlocked`는 원자적 연산을 제공하여 lock 없이 간단한 동기화를 수행합니다.

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class InterlockedExample : MonoBehaviour
{
    private int counter = 0;
    private long longCounter = 0;

    private void Start()
    {
        ComparePerformance();
        DemonstrateInterlockedOperations();
    }

    // =============================================
    // 성능 비교: lock vs Interlocked
    // =============================================

    private async void ComparePerformance()
    {
        const int iterations = 1000000;
        var sw = new System.Diagnostics.Stopwatch();

        // 1. lock 사용
        counter = 0;
        object lockObj = new object();
        sw.Start();

        var lockTasks = new Task[4];
        for (int i = 0; i < 4; i++)
        {
            lockTasks[i] = Task.Run(() =>
            {
                for (int j = 0; j < iterations; j++)
                {
                    lock (lockObj) { counter++; }
                }
            });
        }
        await Task.WhenAll(lockTasks);
        sw.Stop();
        Debug.Log($"lock: {sw.ElapsedMilliseconds}ms, 결과: {counter}");

        // 2. Interlocked 사용
        counter = 0;
        sw.Restart();

        var interlockedTasks = new Task[4];
        for (int i = 0; i < 4; i++)
        {
            interlockedTasks[i] = Task.Run(() =>
            {
                for (int j = 0; j < iterations; j++)
                {
                    Interlocked.Increment(ref counter);
                }
            });
        }
        await Task.WhenAll(interlockedTasks);
        sw.Stop();
        Debug.Log($"Interlocked: {sw.ElapsedMilliseconds}ms, 결과: {counter}");
    }

    // =============================================
    // Interlocked 메서드들
    // =============================================

    private void DemonstrateInterlockedOperations()
    {
        Debug.Log("\n=== Interlocked 연산들 ===");

        int value = 0;

        // Increment: value++
        int incremented = Interlocked.Increment(ref value);
        Debug.Log($"Increment: {incremented}"); // 1

        // Decrement: value--
        int decremented = Interlocked.Decrement(ref value);
        Debug.Log($"Decrement: {decremented}"); // 0

        // Add: value += n
        int added = Interlocked.Add(ref value, 10);
        Debug.Log($"Add 10: {added}"); // 10

        // Exchange: 값 교환하고 이전 값 반환
        int previous = Interlocked.Exchange(ref value, 100);
        Debug.Log($"Exchange to 100, previous: {previous}"); // 10

        // CompareExchange: 조건부 교환
        // value가 100이면 200으로 변경
        int original = Interlocked.CompareExchange(ref value, 200, 100);
        Debug.Log($"CompareExchange: original={original}, current={value}"); // 100, 200

        // value가 100이 아니므로 변경되지 않음
        original = Interlocked.CompareExchange(ref value, 300, 100);
        Debug.Log($"CompareExchange (not matched): original={original}, current={value}"); // 200, 200

        // Read: 64비트 값의 원자적 읽기 (32비트 플랫폼에서 필요)
        long longValue = Interlocked.Read(ref longCounter);
        Debug.Log($"Read long: {longValue}");
    }

    // =============================================
    // CompareExchange를 이용한 락프리 패턴
    // =============================================

    private int state = 0;
    private const int IDLE = 0;
    private const int RUNNING = 1;

    /// <summary>
    /// 락프리 상태 변경 (원자적 check-and-set)
    /// </summary>
    private bool TryStartOperation()
    {
        // state가 IDLE이면 RUNNING으로 변경
        int original = Interlocked.CompareExchange(ref state, RUNNING, IDLE);
        return original == IDLE; // 변경 성공 여부
    }

    private void CompleteOperation()
    {
        Interlocked.Exchange(ref state, IDLE);
    }

    // =============================================
    // 락프리 카운터 (스핀 방식)
    // =============================================

    private int spinCounter = 0;

    /// <summary>
    /// CompareExchange를 이용한 안전한 증가
    /// </summary>
    private int SafeIncrement()
    {
        int original, incremented;
        do
        {
            original = spinCounter;
            incremented = original + 1;
            // original과 같으면 incremented로 교환
        }
        while (Interlocked.CompareExchange(ref spinCounter, incremented, original) != original);

        return incremented;
    }
}

/*
┌─────────────────────────────────────────────────────────────────┐
│                 Interlocked 메서드 정리                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  메서드                    │ 동작                │ 반환값         │
│  ────────────────────────┼───────────────────┼────────────────│
│  Increment(ref x)        │ x += 1            │ 새 값           │
│  Decrement(ref x)        │ x -= 1            │ 새 값           │
│  Add(ref x, n)           │ x += n            │ 새 값           │
│  Exchange(ref x, n)      │ x = n             │ 이전 값         │
│  CompareExchange(ref x,  │ if(x==c) x=n      │ x의 원래 값     │
│      newVal, comparand)  │                   │                │
│  Read(ref long)          │ 64비트 원자적 읽기  │ 값             │
│                                                                  │
│  사용 권장:                                                       │
│  └─ 단순 카운터 증가/감소                                         │
│  └─ 플래그 설정/해제                                              │
│  └─ 락프리 알고리즘                                               │
│                                                                  │
│  lock 대비 장점:                                                  │
│  └─ 빠름 (커널 전환 없음)                                         │
│  └─ 데드락 없음                                                   │
│                                                                  │
│  제한사항:                                                        │
│  └─ 복잡한 연산에는 부적합                                        │
│  └─ 여러 변수 동시 업데이트 불가                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
*/
```

---

## 7. SpinLock

`SpinLock`은 짧은 시간 동안만 lock이 필요할 때 사용하는 경량 동기화 프리미티브입니다.

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class SpinLockExample : MonoBehaviour
{
    private SpinLock spinLock = new SpinLock();
    private int counter = 0;

    private void Start()
    {
        CompareSpinLockVsLock();
    }

    // =============================================
    // SpinLock 기본 사용법
    // =============================================

    private void BasicSpinLock()
    {
        bool lockTaken = false;
        try
        {
            spinLock.Enter(ref lockTaken);

            // 매우 짧은 critical section
            counter++;
        }
        finally
        {
            if (lockTaken)
            {
                spinLock.Exit();
            }
        }
    }

    // =============================================
    // SpinLock vs lock 성능 비교
    // =============================================

    private async void CompareSpinLockVsLock()
    {
        const int iterations = 1000000;
        var sw = new System.Diagnostics.Stopwatch();

        // 1. 일반 lock (짧은 작업)
        counter = 0;
        object lockObj = new object();
        sw.Start();

        var lockTasks = new Task[4];
        for (int i = 0; i < 4; i++)
        {
            lockTasks[i] = Task.Run(() =>
            {
                for (int j = 0; j < iterations; j++)
                {
                    lock (lockObj)
                    {
                        counter++; // 매우 짧은 작업
                    }
                }
            });
        }
        await Task.WhenAll(lockTasks);
        sw.Stop();
        Debug.Log($"lock (짧은 작업): {sw.ElapsedMilliseconds}ms");

        // 2. SpinLock (짧은 작업)
        counter = 0;
        spinLock = new SpinLock();
        sw.Restart();

        var spinTasks = new Task[4];
        for (int i = 0; i < 4; i++)
        {
            spinTasks[i] = Task.Run(() =>
            {
                for (int j = 0; j < iterations; j++)
                {
                    bool taken = false;
                    try
                    {
                        spinLock.Enter(ref taken);
                        counter++; // 매우 짧은 작업
                    }
                    finally
                    {
                        if (taken) spinLock.Exit();
                    }
                }
            });
        }
        await Task.WhenAll(spinTasks);
        sw.Stop();
        Debug.Log($"SpinLock (짧은 작업): {sw.ElapsedMilliseconds}ms");
    }
}

/*
┌─────────────────────────────────────────────────────────────────┐
│                    SpinLock vs lock 비교                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SpinLock                                                        │
│  ├─ 동작: lock을 얻을 때까지 계속 시도 (spin)                     │
│  ├─ CPU 사용: lock 대기 중에도 CPU 사용                          │
│  ├─ 적합: 매우 짧은 critical section                             │
│  └─ 부적합: 긴 작업, 많은 경합                                   │
│                                                                  │
│  lock (Monitor)                                                  │
│  ├─ 동작: lock을 못 얻으면 스레드를 대기 상태로                   │
│  ├─ CPU 사용: 대기 중 CPU 사용 안 함                             │
│  ├─ 적합: 긴 critical section, 많은 경합                         │
│  └─ 오버헤드: 커널 전환 비용                                     │
│                                                                  │
│  권장 사용:                                                       │
│  └─ SpinLock: 마이크로초 단위 작업                               │
│  └─ lock: 밀리초 이상 작업                                        │
│                                                                  │
│  ⚠️ 주의:                                                        │
│  └─ SpinLock은 struct이므로 readonly로 선언하면 안 됨!           │
│  └─ SpinLock은 재진입(reentrant) 불가                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
*/
```

---

## 8. ReaderWriterLockSlim

읽기는 동시에 여러 스레드가, 쓰기는 단독으로만 접근할 수 있게 하는 동기화 프리미티브입니다.

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Collections.Generic;

public class ReaderWriterLockExample : MonoBehaviour
{
    private ReaderWriterLockSlim rwLock = new ReaderWriterLockSlim();
    private Dictionary<string, int> cache = new Dictionary<string, int>();

    private void Start()
    {
        DemonstrateReaderWriterLock();
    }

    private async void DemonstrateReaderWriterLock()
    {
        // 초기 데이터
        cache["key1"] = 100;
        cache["key2"] = 200;

        var tasks = new List<Task>();

        // 10개의 Reader
        for (int i = 0; i < 10; i++)
        {
            int readerId = i;
            tasks.Add(Task.Run(() => ReaderTask(readerId)));
        }

        // 2개의 Writer
        for (int i = 0; i < 2; i++)
        {
            int writerId = i;
            tasks.Add(Task.Run(() => WriterTask(writerId)));
        }

        await Task.WhenAll(tasks);
    }

    // =============================================
    // Reader - 동시에 여러 스레드 가능
    // =============================================

    private void ReaderTask(int readerId)
    {
        for (int i = 0; i < 5; i++)
        {
            rwLock.EnterReadLock();
            try
            {
                // 여러 Reader가 동시에 읽기 가능
                Debug.Log($"[Reader {readerId}] 읽기 중... " +
                         $"(Readers: {rwLock.CurrentReadCount})");

                foreach (var kvp in cache)
                {
                    // 읽기 작업
                    var value = kvp.Value;
                }

                Thread.Sleep(100);
            }
            finally
            {
                rwLock.ExitReadLock();
            }

            Thread.Sleep(50);
        }
    }

    // =============================================
    // Writer - 단독 접근만 가능
    // =============================================

    private void WriterTask(int writerId)
    {
        for (int i = 0; i < 3; i++)
        {
            rwLock.EnterWriteLock();
            try
            {
                // Writer는 단독으로만 접근 가능
                Debug.Log($"[Writer {writerId}] 쓰기 중... " +
                         $"(Readers: {rwLock.CurrentReadCount})");

                // 쓰기 작업
                cache[$"key_writer{writerId}_{i}"] = i * 100;

                Thread.Sleep(200);
            }
            finally
            {
                rwLock.ExitWriteLock();
            }

            Thread.Sleep(100);
        }
    }

    // =============================================
    // Upgradeable Lock - 읽기에서 쓰기로 업그레이드
    // =============================================

    private int GetOrAdd(string key, Func<int> valueFactory)
    {
        // 먼저 읽기 시도
        rwLock.EnterUpgradeableReadLock();
        try
        {
            if (cache.TryGetValue(key, out int value))
            {
                return value; // 있으면 반환
            }

            // 없으면 쓰기로 업그레이드
            rwLock.EnterWriteLock();
            try
            {
                // 다시 확인 (다른 스레드가 추가했을 수 있음)
                if (cache.TryGetValue(key, out value))
                {
                    return value;
                }

                // 새 값 추가
                value = valueFactory();
                cache[key] = value;
                return value;
            }
            finally
            {
                rwLock.ExitWriteLock();
            }
        }
        finally
        {
            rwLock.ExitUpgradeableReadLock();
        }
    }

    private void OnDestroy()
    {
        rwLock?.Dispose();
    }
}

/*
출력 예시:
[Reader 0] 읽기 중... (Readers: 3)
[Reader 1] 읽기 중... (Readers: 3)
[Reader 2] 읽기 중... (Readers: 3)  ← 동시 읽기 가능
[Writer 0] 쓰기 중... (Readers: 0)  ← 쓰기 시 독점
[Reader 3] 읽기 중... (Readers: 5)
...

┌─────────────────────────────────────────────────────────────────┐
│              ReaderWriterLockSlim 동작                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Read Lock                                                       │
│  ├─ 여러 Reader 동시 접근 가능                                   │
│  ├─ Writer가 대기 중이면 새 Reader 블로킹                         │
│  └─ 읽기 전용 작업에 사용                                        │
│                                                                  │
│  Write Lock                                                      │
│  ├─ 단 하나의 Writer만 접근                                      │
│  ├─ 모든 Reader와 Writer 블로킹                                  │
│  └─ 쓰기 작업에 사용                                             │
│                                                                  │
│  Upgradeable Read Lock                                           │
│  ├─ 읽기로 시작, 필요 시 쓰기로 업그레이드                        │
│  ├─ 한 번에 하나만 Upgradeable 상태 가능                         │
│  └─ "읽고 필요하면 쓰기" 패턴에 유용                              │
│                                                                  │
│  사용 권장:                                                       │
│  └─ 읽기가 쓰기보다 훨씬 많은 경우                                │
│  └─ 캐시, 설정 데이터 등                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
*/
```

---

## 9. Unity 실전 예제: 스레드 안전한 이벤트 시스템

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Collections.Concurrent;
using System.Collections.Generic;

/// <summary>
/// 스레드 안전한 이벤트 버스 구현
/// </summary>
public class ThreadSafeEventBus : MonoBehaviour
{
    private static ThreadSafeEventBus instance;
    public static ThreadSafeEventBus Instance => instance;

    // 이벤트 구독자 저장 (스레드 안전)
    private ConcurrentDictionary<Type, ConcurrentBag<Delegate>> subscribers
        = new ConcurrentDictionary<Type, ConcurrentBag<Delegate>>();

    // 메인 스레드에서 처리할 이벤트 큐
    private ConcurrentQueue<Action> mainThreadQueue = new ConcurrentQueue<Action>();

    private int mainThreadId;

    private void Awake()
    {
        if (instance != null)
        {
            Destroy(gameObject);
            return;
        }

        instance = this;
        mainThreadId = Thread.CurrentThread.ManagedThreadId;
        DontDestroyOnLoad(gameObject);
    }

    private void Update()
    {
        // 메인 스레드에서 큐 처리
        while (mainThreadQueue.TryDequeue(out var action))
        {
            try
            {
                action?.Invoke();
            }
            catch (Exception e)
            {
                Debug.LogException(e);
            }
        }
    }

    // =============================================
    // 구독
    // =============================================

    public void Subscribe<T>(Action<T> handler)
    {
        var type = typeof(T);
        var bag = subscribers.GetOrAdd(type, _ => new ConcurrentBag<Delegate>());
        bag.Add(handler);
    }

    // =============================================
    // 구독 해제
    // =============================================

    public void Unsubscribe<T>(Action<T> handler)
    {
        var type = typeof(T);
        if (subscribers.TryGetValue(type, out var bag))
        {
            // ConcurrentBag은 Remove를 지원하지 않음
            // 실제 구현에서는 ConcurrentDictionary<Delegate, bool> 등 사용 권장
            Debug.LogWarning("ConcurrentBag은 Remove를 지원하지 않습니다. 실제 구현에서는 다른 컬렉션 사용 권장.");
        }
    }

    // =============================================
    // 이벤트 발행
    // =============================================

    /// <summary>
    /// 이벤트 발행 (어떤 스레드에서든 호출 가능)
    /// </summary>
    public void Publish<T>(T eventData, bool executeOnMainThread = true)
    {
        var type = typeof(T);

        if (!subscribers.TryGetValue(type, out var bag))
        {
            return;
        }

        foreach (var subscriber in bag)
        {
            if (subscriber is Action<T> handler)
            {
                if (executeOnMainThread && Thread.CurrentThread.ManagedThreadId != mainThreadId)
                {
                    // 메인 스레드가 아니면 큐에 추가
                    mainThreadQueue.Enqueue(() => handler(eventData));
                }
                else
                {
                    // 메인 스레드면 즉시 실행
                    try
                    {
                        handler(eventData);
                    }
                    catch (Exception e)
                    {
                        Debug.LogException(e);
                    }
                }
            }
        }
    }
}

// =============================================
// 사용 예제
// =============================================

// 이벤트 정의
public struct PlayerDamagedEvent
{
    public int Damage;
    public Vector3 Position;
}

public class EventBusUsageExample : MonoBehaviour
{
    private void Start()
    {
        // 이벤트 구독
        ThreadSafeEventBus.Instance.Subscribe<PlayerDamagedEvent>(OnPlayerDamaged);

        // 백그라운드 스레드에서 이벤트 발행 테스트
        System.Threading.Tasks.Task.Run(() =>
        {
            Thread.Sleep(1000);

            // 백그라운드 스레드에서 발행해도 메인 스레드에서 처리됨
            ThreadSafeEventBus.Instance.Publish(new PlayerDamagedEvent
            {
                Damage = 50,
                Position = Vector3.zero
            });
        });
    }

    private void OnPlayerDamaged(PlayerDamagedEvent e)
    {
        // 메인 스레드에서 실행되므로 Unity API 사용 가능
        Debug.Log($"플레이어 피해: {e.Damage}, 위치: {e.Position}");
        // transform.position = e.Position; // 안전!
    }
}
```

---

## 10. 동기화 선택 가이드

```
┌─────────────────────────────────────────────────────────────────┐
│                    동기화 프리미티브 선택 가이드                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  단순 카운터/플래그?                                              │
│  └─ Yes → Interlocked                                           │
│                                                                  │
│  매우 짧은 critical section? (마이크로초)                         │
│  └─ Yes → SpinLock 고려 (단, 경합이 적을 때)                      │
│                                                                  │
│  읽기가 쓰기보다 훨씬 많은가?                                     │
│  └─ Yes → ReaderWriterLockSlim                                  │
│                                                                  │
│  프로세스 간 동기화 필요?                                         │
│  └─ Yes → Mutex (명명된)                                        │
│                                                                  │
│  동시 접근 수 제한 필요?                                          │
│  └─ Yes → Semaphore / SemaphoreSlim                             │
│                                                                  │
│  스레드 간 신호 전달 필요?                                        │
│  └─ Yes → Monitor.Wait/Pulse 또는 AutoResetEvent                │
│                                                                  │
│  그 외 일반적인 경우                                              │
│  └─ lock (Monitor)                                              │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  성능 순위 (빠른 순):                                             │
│  1. Interlocked (lock-free)                                     │
│  2. SpinLock (짧은 작업)                                         │
│  3. lock / Monitor                                              │
│  4. SemaphoreSlim                                               │
│  5. ReaderWriterLockSlim                                        │
│  6. Mutex, Semaphore (커널 객체)                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 주의사항

1. **데드락 방지**: 여러 lock을 획득할 때 항상 같은 순서로 획득
2. **lock 범위 최소화**: critical section은 최대한 짧게 유지
3. **lock 객체 선택**: private readonly object 사용 권장
4. **재진입 고려**: SpinLock은 재진입 불가
5. **Unity 메인 스레드**: Unity API는 동기화와 무관하게 메인 스레드에서만 호출
6. **Dispose 필수**: IDisposable인 동기화 객체는 반드시 Dispose 호출

---

## 참고 자료

- [Microsoft: Threading in C#](https://docs.microsoft.com/en-us/dotnet/standard/threading/)
- [Microsoft: Lock Statement](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/statements/lock)
- [Microsoft: Interlocked Class](https://docs.microsoft.com/en-us/dotnet/api/system.threading.interlocked)

---

## 다음 섹션

[06. APM (Asynchronous Programming Model)](../03-legacy-patterns/06-apm.md)
