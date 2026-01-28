# 06. APM (Asynchronous Programming Model)

## 개요

APM(Asynchronous Programming Model)은 .NET Framework 1.0부터 도입된 가장 오래된 비동기 프로그래밍 패턴입니다. `BeginXxx`/`EndXxx` 메서드 쌍과 `IAsyncResult` 인터페이스를 사용합니다. 현재는 레거시 패턴이지만, 오래된 API를 이해하고 마이그레이션하기 위해 알아두면 좋습니다.

---

## 1. APM 패턴의 기본 구조

### BeginXxx / EndXxx 패턴

```csharp
using System;
using System.IO;
using System.Net;
using System.Threading;
using UnityEngine;

public class APMBasicsExample : MonoBehaviour
{
    /*
    APM 패턴 구조:

    // 동기 메서드
    TResult Operation(TArgs args);

    // 비동기 메서드 쌍
    IAsyncResult BeginOperation(TArgs args, AsyncCallback callback, object state);
    TResult EndOperation(IAsyncResult asyncResult);
    */

    private void Start()
    {
        DemonstrateAPM();
    }

    // =============================================
    // 파일 읽기 예제 (APM)
    // =============================================

    private void DemonstrateAPM()
    {
        string filePath = Application.dataPath + "/test.txt";

        // 테스트 파일 생성
        File.WriteAllText(filePath, "Hello APM World!");

        // 파일 열기
        FileStream fs = new FileStream(
            filePath,
            FileMode.Open,
            FileAccess.Read,
            FileShare.Read,
            4096,
            FileOptions.Asynchronous // 비동기 I/O 활성화
        );

        byte[] buffer = new byte[1024];

        // BeginRead: 비동기 읽기 시작
        IAsyncResult asyncResult = fs.BeginRead(
            buffer,                     // 버퍼
            0,                          // 오프셋
            buffer.Length,              // 읽을 바이트 수
            new AsyncCallback(ReadCallback), // 완료 콜백
            new ReadState { FileStream = fs, Buffer = buffer } // 상태 객체
        );

        Debug.Log($"BeginRead 호출됨, IsCompleted: {asyncResult.IsCompleted}");
        Debug.Log("메인 스레드는 계속 실행...");
    }

    // 상태 객체
    private class ReadState
    {
        public FileStream FileStream;
        public byte[] Buffer;
    }

    // 콜백 (I/O 완료 시 호출됨)
    private void ReadCallback(IAsyncResult ar)
    {
        try
        {
            ReadState state = (ReadState)ar.AsyncState;

            // EndRead: 비동기 읽기 완료 및 결과 받기
            int bytesRead = state.FileStream.EndRead(ar);

            string content = System.Text.Encoding.UTF8.GetString(state.Buffer, 0, bytesRead);
            Debug.Log($"[콜백] 읽기 완료: {bytesRead} bytes");
            Debug.Log($"[콜백] 내용: {content}");
            Debug.Log($"[콜백] Thread ID: {Thread.CurrentThread.ManagedThreadId}");

            state.FileStream.Close();
        }
        catch (Exception ex)
        {
            Debug.LogError($"읽기 에러: {ex.Message}");
        }
    }
}
```

### IAsyncResult 인터페이스

```csharp
using System;
using System.Threading;
using UnityEngine;

public class IAsyncResultExample : MonoBehaviour
{
    /*
    public interface IAsyncResult
    {
        object AsyncState { get; }           // BeginXxx에 전달된 state 객체
        WaitHandle AsyncWaitHandle { get; }  // 완료 대기용 핸들
        bool CompletedSynchronously { get; } // 동기적으로 완료되었는지
        bool IsCompleted { get; }            // 완료 여부
    }
    */

    private void Start()
    {
        DemonstrateIAsyncResult();
    }

    private void DemonstrateIAsyncResult()
    {
        // 델리게이트를 이용한 APM
        Func<int, int, int> add = (a, b) =>
        {
            Thread.Sleep(1000); // 시뮬레이션
            return a + b;
        };

        // 비동기 호출 시작
        IAsyncResult ar = add.BeginInvoke(10, 20, null, "myState");

        Debug.Log($"AsyncState: {ar.AsyncState}");
        Debug.Log($"IsCompleted: {ar.IsCompleted}");
        Debug.Log($"CompletedSynchronously: {ar.CompletedSynchronously}");

        // 방법 1: 폴링
        while (!ar.IsCompleted)
        {
            Debug.Log("대기 중...");
            Thread.Sleep(100);
        }

        // 방법 2: WaitHandle 사용
        // ar.AsyncWaitHandle.WaitOne();

        // 방법 3: 타임아웃 대기
        // bool completed = ar.AsyncWaitHandle.WaitOne(TimeSpan.FromSeconds(5));

        // 결과 받기
        int result = add.EndInvoke(ar);
        Debug.Log($"결과: {result}");
    }
}
```

---

## 2. APM의 세 가지 사용 패턴

### 패턴 1: 콜백 패턴

```csharp
using System;
using System.IO;
using System.Threading;
using UnityEngine;

public class APMCallbackPattern : MonoBehaviour
{
    // 가장 일반적인 APM 사용 방법
    // 콜백에서 결과를 처리

    private void Start()
    {
        CallbackPatternExample();
    }

    private void CallbackPatternExample()
    {
        Func<string, string> process = (input) =>
        {
            Thread.Sleep(500);
            return input.ToUpper();
        };

        Debug.Log("비동기 작업 시작");

        // 콜백 함수 전달
        process.BeginInvoke("hello world", ar =>
        {
            // 이 콜백은 ThreadPool 스레드에서 실행됨
            try
            {
                string result = ((Func<string, string>)ar.AsyncState).EndInvoke(ar);
                Debug.Log($"[콜백] 결과: {result}, Thread: {Thread.CurrentThread.ManagedThreadId}");

                // ⚠️ Unity API 직접 호출 불가!
                // transform.position = Vector3.zero; // 에러!
            }
            catch (Exception ex)
            {
                Debug.LogError($"에러: {ex.Message}");
            }
        }, process);

        Debug.Log("메인 스레드 계속 실행");
    }
}
```

### 패턴 2: Wait 패턴

```csharp
using System;
using System.Threading;
using UnityEngine;

public class APMWaitPattern : MonoBehaviour
{
    // WaitHandle을 사용하여 완료 대기
    // 다른 작업을 하다가 결과가 필요할 때 대기

    private void Start()
    {
        WaitPatternExample();
    }

    private void WaitPatternExample()
    {
        Func<int> heavyWork = () =>
        {
            Thread.Sleep(1000);
            return 42;
        };

        Debug.Log("작업 시작");
        IAsyncResult ar = heavyWork.BeginInvoke(null, null);

        // 다른 작업 수행
        Debug.Log("다른 작업 수행 중...");
        Thread.Sleep(200);

        // 결과 필요 - 완료될 때까지 대기
        Debug.Log("결과 대기 중...");
        ar.AsyncWaitHandle.WaitOne();

        int result = heavyWork.EndInvoke(ar);
        Debug.Log($"결과: {result}");
    }

    // 타임아웃 포함 대기
    private void WaitWithTimeoutExample()
    {
        Func<int> slowWork = () =>
        {
            Thread.Sleep(5000);
            return 100;
        };

        IAsyncResult ar = slowWork.BeginInvoke(null, null);

        // 2초 타임아웃
        if (ar.AsyncWaitHandle.WaitOne(TimeSpan.FromSeconds(2)))
        {
            int result = slowWork.EndInvoke(ar);
            Debug.Log($"완료: {result}");
        }
        else
        {
            Debug.Log("타임아웃 - 작업이 아직 실행 중");
            // 주의: 작업은 계속 실행 중이며 취소되지 않음
        }
    }
}
```

### 패턴 3: 폴링 패턴

```csharp
using System;
using System.Threading;
using UnityEngine;

public class APMPollingPattern : MonoBehaviour
{
    // IsCompleted를 주기적으로 확인
    // 완료 전까지 다른 작업 수행 가능

    private IAsyncResult currentOperation;
    private Func<int> workDelegate;

    private void Start()
    {
        workDelegate = () =>
        {
            Thread.Sleep(2000);
            return 42;
        };

        currentOperation = workDelegate.BeginInvoke(null, null);
        Debug.Log("폴링 시작");
    }

    private void Update()
    {
        if (currentOperation != null && !currentOperation.IsCompleted)
        {
            // 완료 전 - 진행 표시 등
            Debug.Log("작업 진행 중...");
        }
        else if (currentOperation != null && currentOperation.IsCompleted)
        {
            // 완료됨
            int result = workDelegate.EndInvoke(currentOperation);
            Debug.Log($"완료! 결과: {result}");
            currentOperation = null;
        }
    }
}
```

---

## 3. 실제 .NET API의 APM 예제

### WebRequest APM

```csharp
using System;
using System.IO;
using System.Net;
using System.Threading;
using UnityEngine;

public class WebRequestAPMExample : MonoBehaviour
{
    // .NET의 WebRequest는 APM을 지원함
    // (Unity에서는 UnityWebRequest 사용 권장)

    private void Start()
    {
        // 실제 네트워크 요청 예제
        FetchDataWithAPM("https://httpbin.org/get");
    }

    private void FetchDataWithAPM(string url)
    {
        try
        {
            WebRequest request = WebRequest.Create(url);
            request.Method = "GET";

            Debug.Log("요청 시작...");

            // 비동기 요청 시작
            request.BeginGetResponse(ResponseCallback, request);

            Debug.Log("BeginGetResponse 호출됨, 메인 스레드 계속");
        }
        catch (Exception ex)
        {
            Debug.LogError($"요청 에러: {ex.Message}");
        }
    }

    private void ResponseCallback(IAsyncResult ar)
    {
        try
        {
            WebRequest request = (WebRequest)ar.AsyncState;

            // 응답 받기
            using (WebResponse response = request.EndGetResponse(ar))
            using (Stream stream = response.GetResponseStream())
            using (StreamReader reader = new StreamReader(stream))
            {
                string content = reader.ReadToEnd();
                Debug.Log($"[콜백] 응답 받음 ({content.Length} chars)");
                Debug.Log($"[콜백] Thread: {Thread.CurrentThread.ManagedThreadId}");
            }
        }
        catch (Exception ex)
        {
            Debug.LogError($"응답 에러: {ex.Message}");
        }
    }
}
```

---

## 4. APM에서 TAP로 변환

### TaskFactory.FromAsync 사용

```csharp
using System;
using System.IO;
using System.Threading.Tasks;
using UnityEngine;

public class APMToTAPConversionExample : MonoBehaviour
{
    private void Start()
    {
        ConvertAPMToTAP();
    }

    private async void ConvertAPMToTAP()
    {
        string filePath = Application.dataPath + "/test.txt";
        File.WriteAllText(filePath, "APM to TAP conversion test!");

        // APM 방식의 FileStream
        using (FileStream fs = new FileStream(
            filePath,
            FileMode.Open,
            FileAccess.Read,
            FileShare.Read,
            4096,
            FileOptions.Asynchronous))
        {
            byte[] buffer = new byte[1024];

            // APM을 TAP으로 변환
            int bytesRead = await Task.Factory.FromAsync(
                fs.BeginRead,   // Begin 메서드
                fs.EndRead,     // End 메서드
                buffer,         // 첫 번째 매개변수
                0,              // 두 번째 매개변수
                buffer.Length,  // 세 번째 매개변수
                null            // state (사용 안 함)
            );

            string content = System.Text.Encoding.UTF8.GetString(buffer, 0, bytesRead);
            Debug.Log($"TAP으로 변환된 결과: {content}");
        }
    }

    // 일반화된 APM → TAP 변환 패턴
    private Task<TResult> ConvertAPMToTask<TResult>(
        Func<AsyncCallback, object, IAsyncResult> beginMethod,
        Func<IAsyncResult, TResult> endMethod)
    {
        return Task.Factory.FromAsync(beginMethod, endMethod, null);
    }
}
```

### 커스텀 APM 래퍼

```csharp
using System;
using System.Threading.Tasks;
using UnityEngine;

public class CustomAPMWrapper : MonoBehaviour
{
    // 레거시 APM API를 async/await로 사용하기 위한 래퍼

    private void Start()
    {
        UseWrappedAPI();
    }

    private async void UseWrappedAPI()
    {
        // 레거시 APM API가 있다고 가정
        LegacyService service = new LegacyService();

        // 래퍼를 통해 async/await 사용
        string result = await service.ProcessDataTaskAsync("input");
        Debug.Log($"결과: {result}");
    }

    // 레거시 서비스 시뮬레이션
    private class LegacyService
    {
        // APM 패턴의 레거시 메서드
        public IAsyncResult BeginProcessData(string input, AsyncCallback callback, object state)
        {
            Func<string, string> process = i =>
            {
                System.Threading.Thread.Sleep(500);
                return i.ToUpper();
            };

            return process.BeginInvoke(input, callback, state);
        }

        public string EndProcessData(IAsyncResult ar)
        {
            return ((Func<string, string>)((System.Runtime.Remoting.Messaging.AsyncResult)ar).AsyncDelegate).EndInvoke(ar);
        }

        // TAP 래퍼 메서드
        public Task<string> ProcessDataTaskAsync(string input)
        {
            var tcs = new TaskCompletionSource<string>();

            BeginProcessData(input, ar =>
            {
                try
                {
                    string result = EndProcessData(ar);
                    tcs.SetResult(result);
                }
                catch (Exception ex)
                {
                    tcs.SetException(ex);
                }
            }, null);

            return tcs.Task;
        }
    }
}
```

---

## 5. APM의 문제점

```csharp
using UnityEngine;
using System;
using System.Threading;

public class APMProblemsExample : MonoBehaviour
{
    /*
    ┌─────────────────────────────────────────────────────────────────┐
    │                      APM의 문제점                                │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  1. 콜백 지옥 (Callback Hell)                                   │
    │     └─ 중첩된 비동기 호출이 복잡해짐                              │
    │     └─ 코드 가독성 저하                                          │
    │                                                                  │
    │  2. 예외 처리의 어려움                                           │
    │     └─ EndXxx에서만 예외 발생                                    │
    │     └─ 콜백에서 예외 처리 필수                                   │
    │                                                                  │
    │  3. 취소 지원 부재                                               │
    │     └─ CancellationToken 미지원                                 │
    │     └─ 취소 로직 직접 구현 필요                                  │
    │                                                                  │
    │  4. 진행률 보고 어려움                                           │
    │     └─ IProgress<T> 미지원                                      │
    │     └─ 별도 구현 필요                                           │
    │                                                                  │
    │  5. 메서드 쌍 관리                                               │
    │     └─ Begin/End 쌍을 올바르게 호출해야 함                        │
    │     └─ EndXxx 미호출 시 리소스 누수                              │
    │                                                                  │
    │  6. 콜백 스레드                                                  │
    │     └─ 콜백이 ThreadPool에서 실행됨                              │
    │     └─ Unity API 사용 불가                                       │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
    */

    private void Start()
    {
        // 콜백 지옥 예제
        CallbackHellExample();
    }

    // =============================================
    // 콜백 지옥 예제
    // =============================================

    private void CallbackHellExample()
    {
        // 3개의 연속 비동기 작업을 APM으로 구현하면...
        Step1((result1) =>
        {
            Step2(result1, (result2) =>
            {
                Step3(result2, (result3) =>
                {
                    // 점점 깊어지는 중첩...
                    Debug.Log($"최종 결과: {result3}");
                });
            });
        });

        // 비교: async/await 사용 시
        // var result1 = await Step1Async();
        // var result2 = await Step2Async(result1);
        // var result3 = await Step3Async(result2);
    }

    private void Step1(Action<int> callback)
    {
        // 비동기 작업 시뮬레이션
        ThreadPool.QueueUserWorkItem(_ =>
        {
            Thread.Sleep(100);
            callback(1);
        });
    }

    private void Step2(int input, Action<int> callback)
    {
        ThreadPool.QueueUserWorkItem(_ =>
        {
            Thread.Sleep(100);
            callback(input + 1);
        });
    }

    private void Step3(int input, Action<int> callback)
    {
        ThreadPool.QueueUserWorkItem(_ =>
        {
            Thread.Sleep(100);
            callback(input + 1);
        });
    }
}
```

---

## 6. Unity에서 APM 사용 시 주의사항

```csharp
using UnityEngine;
using System;
using System.Threading;
using System.Collections.Concurrent;

public class APMUnityConsiderations : MonoBehaviour
{
    private ConcurrentQueue<Action> mainThreadQueue = new ConcurrentQueue<Action>();

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
        SafeAPMUsage();
    }

    // =============================================
    // Unity에서 안전한 APM 사용
    // =============================================

    private void SafeAPMUsage()
    {
        Func<int, int> work = x =>
        {
            Thread.Sleep(500);
            return x * 2;
        };

        work.BeginInvoke(21, ar =>
        {
            try
            {
                int result = work.EndInvoke(ar);

                // ❌ 직접 Unity API 호출 불가
                // transform.position = new Vector3(result, 0, 0);

                // ✅ 메인 스레드 큐에 추가
                mainThreadQueue.Enqueue(() =>
                {
                    // 여기서는 Unity API 사용 가능
                    transform.position = new Vector3(result, 0, 0);
                    Debug.Log($"위치 업데이트: {result}");
                });
            }
            catch (Exception ex)
            {
                mainThreadQueue.Enqueue(() =>
                {
                    Debug.LogError($"에러: {ex.Message}");
                });
            }
        }, null);
    }
}
```

---

## 7. APM vs EAP vs TAP 비교

```
┌─────────────────────────────────────────────────────────────────┐
│              .NET 비동기 패턴 진화                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  APM (Asynchronous Programming Model)                           │
│  └─ .NET 1.0 (2002)                                             │
│  └─ BeginXxx/EndXxx, IAsyncResult                               │
│  └─ 콜백 기반                                                    │
│                                                                  │
│           ↓ 개선                                                 │
│                                                                  │
│  EAP (Event-based Asynchronous Pattern)                         │
│  └─ .NET 2.0 (2005)                                             │
│  └─ XxxAsync 메서드, XxxCompleted 이벤트                        │
│  └─ BackgroundWorker 등                                         │
│                                                                  │
│           ↓ 개선                                                 │
│                                                                  │
│  TAP (Task-based Asynchronous Pattern)                          │
│  └─ .NET 4.0 (2010), C# 5.0 (2012) async/await                 │
│  └─ Task, Task<T>, async/await                                  │
│  └─ 현재 표준                                                    │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Unity 권장:                                                      │
│  └─ Coroutine (전통적)                                           │
│  └─ UniTask (권장)                                               │
│  └─ Awaitable (Unity 2023+)                                     │
│  └─ APM은 레거시 API 호환용으로만 사용                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 주의사항

1. **EndXxx 필수 호출**: EndXxx를 호출하지 않으면 리소스 누수 발생
2. **콜백 스레드**: 콜백은 ThreadPool에서 실행되므로 Unity API 직접 사용 불가
3. **예외 처리**: EndXxx에서 예외가 발생할 수 있으므로 try-catch 필수
4. **취소 미지원**: 기본적으로 취소를 지원하지 않음
5. **레거시 권장**: 새 코드에서는 TAP(async/await) 사용 권장

---

## 참고 자료

- [Microsoft: Asynchronous Programming Model (APM)](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/asynchronous-programming-model-apm)
- [Microsoft: Interop with Other Asynchronous Patterns](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/interop-with-other-asynchronous-patterns-and-types)

---

## 다음 섹션

[07. Coroutine](./07-coroutine.md)
