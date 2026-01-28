# 02. 유니티 엔진 라이프 사이클과 동시성

## 개요

Unity 엔진은 단일 메인 스레드에서 게임 로직을 실행하는 프레임 기반 실행 모델을 사용합니다. 동시성 프로그래밍을 효과적으로 활용하려면 Unity의 실행 흐름(PlayerLoop)과 라이프사이클을 이해해야 합니다.

---

## 1. Unity Main Thread의 이해

### Main Thread란?

Unity의 메인 스레드는 게임의 핵심 로직이 실행되는 단일 스레드입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Unity Main Thread                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  • 모든 MonoBehaviour 콜백 (Start, Update, etc.)                  │
│  • 모든 Unity API 호출 (Transform, GameObject, etc.)              │
│  • 렌더링 커맨드 생성                                              │
│  • 물리 시뮬레이션 (FixedUpdate)                                   │
│  • UI 이벤트 처리                                                  │
│  • Coroutine 실행                                                 │
│  • Animation 업데이트                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 왜 Single Thread인가?

Unity가 메인 스레드 모델을 사용하는 이유:

1. **결정론적 실행**: 매 프레임 동일한 순서로 실행 → 디버깅 용이
2. **단순성**: 멀티스레드 동기화 문제 회피
3. **안전성**: 레이스 컨디션, 데드락 위험 감소
4. **호환성**: 다양한 플랫폼에서 일관된 동작

### Unity 예제: 메인 스레드 확인

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;

public class MainThreadExample : MonoBehaviour
{
    private int mainThreadId;

    private void Awake()
    {
        // 메인 스레드 ID 저장
        mainThreadId = Thread.CurrentThread.ManagedThreadId;
        Debug.Log($"Main Thread ID: {mainThreadId}");
    }

    private void Start()
    {
        // 메인 스레드에서 실행됨
        Debug.Log($"Start() - Thread ID: {Thread.CurrentThread.ManagedThreadId}");
        CheckMainThread("Start");

        // 백그라운드 스레드에서 실행
        Task.Run(() =>
        {
            Debug.Log($"Task.Run - Thread ID: {Thread.CurrentThread.ManagedThreadId}");
            CheckMainThread("Task.Run");

            // ❌ 이것은 실패함!
            // transform.position = Vector3.zero; // UnityException 발생
        });

        // 코루틴 시작
        StartCoroutine(CoroutineExample());
    }

    private void Update()
    {
        // 매 프레임 메인 스레드에서 실행
        // Debug.Log($"Update() - Thread ID: {Thread.CurrentThread.ManagedThreadId}");
    }

    private System.Collections.IEnumerator CoroutineExample()
    {
        Debug.Log($"Coroutine - Thread ID: {Thread.CurrentThread.ManagedThreadId}");
        CheckMainThread("Coroutine");

        yield return new WaitForSeconds(1f);

        // yield 후에도 여전히 메인 스레드
        Debug.Log($"Coroutine after yield - Thread ID: {Thread.CurrentThread.ManagedThreadId}");
        CheckMainThread("Coroutine after yield");
    }

    private void CheckMainThread(string context)
    {
        bool isMainThread = Thread.CurrentThread.ManagedThreadId == mainThreadId;
        Debug.Log($"[{context}] Is Main Thread: {isMainThread}");
    }
}
```

---

## 2. PlayerLoop 시스템

### PlayerLoop란?

PlayerLoop는 Unity 엔진이 매 프레임 실행하는 시스템들의 순서를 정의합니다.

### 기본 PlayerLoop 순서

```
┌─────────────────────────────────────────────────────────────────┐
│                     Unity Frame Lifecycle                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  프레임 시작                                                      │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────┐                        │
│  │  Initialization                      │                        │
│  │  • 새 오브젝트 초기화                  │                        │
│  │  • Awake() 호출                       │                        │
│  └─────────────────────────────────────┘                        │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────┐                        │
│  │  EarlyUpdate                         │                        │
│  │  • 입력 처리                          │                        │
│  │  • 물리 시뮬레이션 준비               │                        │
│  └─────────────────────────────────────┘                        │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────┐                        │
│  │  FixedUpdate (0~N회 실행)            │ ◄── 고정 시간 간격     │
│  │  • FixedUpdate() 호출                │                        │
│  │  • 물리 시뮬레이션                    │                        │
│  │  • yield WaitForFixedUpdate         │                        │
│  └─────────────────────────────────────┘                        │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────┐                        │
│  │  Update                              │                        │
│  │  • Update() 호출                     │                        │
│  │  • yield null                        │                        │
│  │  • yield WaitForSeconds             │                        │
│  └─────────────────────────────────────┘                        │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────┐                        │
│  │  PreLateUpdate                       │                        │
│  │  • Animation 업데이트                 │                        │
│  │  • AI/NavMesh 업데이트               │                        │
│  └─────────────────────────────────────┘                        │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────┐                        │
│  │  LateUpdate                          │                        │
│  │  • LateUpdate() 호출                 │                        │
│  │  • 카메라 추적 등                     │                        │
│  └─────────────────────────────────────┘                        │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────┐                        │
│  │  PostLateUpdate                      │                        │
│  │  • yield WaitForEndOfFrame          │                        │
│  │  • 렌더링 준비                        │                        │
│  └─────────────────────────────────────┘                        │
│      │                                                           │
│      ▼                                                           │
│  렌더링 → 프레임 종료                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Unity 예제: PlayerLoop 확인 및 커스터마이징

```csharp
using UnityEngine;
using UnityEngine.LowLevel;
using UnityEngine.PlayerLoop;
using System;
using System.Text;

public class PlayerLoopExample : MonoBehaviour
{
    // =============================================
    // PlayerLoop 구조 출력
    // =============================================

    [ContextMenu("Print PlayerLoop")]
    private void PrintPlayerLoop()
    {
        var playerLoop = PlayerLoop.GetCurrentPlayerLoop();
        StringBuilder sb = new StringBuilder();

        PrintLoopRecursive(playerLoop, sb, 0);
        Debug.Log(sb.ToString());
    }

    private void PrintLoopRecursive(PlayerLoopSystem system, StringBuilder sb, int depth)
    {
        string indent = new string(' ', depth * 2);

        if (system.type != null)
        {
            sb.AppendLine($"{indent}├─ {system.type.Name}");
        }

        if (system.subSystemList != null)
        {
            foreach (var subSystem in system.subSystemList)
            {
                PrintLoopRecursive(subSystem, sb, depth + 1);
            }
        }
    }

    // =============================================
    // 커스텀 PlayerLoop 시스템 추가
    // =============================================

    private void Awake()
    {
        // 커스텀 업데이트 시스템 등록
        RegisterCustomUpdate();
    }

    private void RegisterCustomUpdate()
    {
        var playerLoop = PlayerLoop.GetCurrentPlayerLoop();

        // Update 단계 찾기
        for (int i = 0; i < playerLoop.subSystemList.Length; i++)
        {
            if (playerLoop.subSystemList[i].type == typeof(Update))
            {
                // 커스텀 시스템 삽입
                var updateSystem = playerLoop.subSystemList[i];
                var subSystems = new PlayerLoopSystem[updateSystem.subSystemList.Length + 1];

                // 기존 시스템 복사
                Array.Copy(updateSystem.subSystemList, subSystems, updateSystem.subSystemList.Length);

                // 커스텀 시스템 추가
                subSystems[subSystems.Length - 1] = new PlayerLoopSystem
                {
                    type = typeof(CustomUpdateSystem),
                    updateDelegate = CustomUpdate
                };

                updateSystem.subSystemList = subSystems;
                playerLoop.subSystemList[i] = updateSystem;

                break;
            }
        }

        PlayerLoop.SetPlayerLoop(playerLoop);
        Debug.Log("Custom PlayerLoop system registered!");
    }

    // 커스텀 업데이트 시스템 마커 타입
    private struct CustomUpdateSystem { }

    // 커스텀 업데이트 함수 (매 프레임 Update 단계에서 실행)
    private static void CustomUpdate()
    {
        // 이 코드는 매 프레임 Update 단계에서 실행됨
        // Debug.Log("Custom Update Running!");
    }

    private void OnDestroy()
    {
        // 원래 PlayerLoop로 복원
        PlayerLoop.SetPlayerLoop(PlayerLoop.GetDefaultPlayerLoop());
    }
}
```

---

## 3. MonoBehaviour 라이프사이클 콜백

### 전체 라이프사이클 순서

```csharp
using UnityEngine;
using System.Collections;

public class LifecycleCallbacksExample : MonoBehaviour
{
    // =============================================
    // 초기화 단계 (Initialization Phase)
    // =============================================

    /// <summary>
    /// 스크립트 인스턴스가 로드될 때 호출 (가장 먼저)
    /// - 다른 오브젝트 참조 설정
    /// - 컴포넌트 캐싱
    /// - 오브젝트가 비활성화 상태여도 호출됨
    /// </summary>
    private void Awake()
    {
        Debug.Log("1. Awake() - 인스턴스 로드");
    }

    /// <summary>
    /// 오브젝트가 활성화될 때 호출
    /// - 여러 번 호출될 수 있음 (활성화/비활성화 반복 시)
    /// </summary>
    private void OnEnable()
    {
        Debug.Log("2. OnEnable() - 활성화됨");
    }

    /// <summary>
    /// 첫 번째 Update 전에 호출
    /// - 오브젝트가 비활성화 상태면 호출되지 않음
    /// - 코루틴 시작 가능
    /// </summary>
    private void Start()
    {
        Debug.Log("3. Start() - 첫 Update 전");
        StartCoroutine(CoroutineLifecycle());
    }

    // =============================================
    // 물리 업데이트 단계 (Physics Phase)
    // =============================================

    /// <summary>
    /// 고정 시간 간격으로 호출 (기본 0.02초 = 50Hz)
    /// - 물리 연산에 사용
    /// - 프레임 레이트와 무관하게 일정한 간격
    /// - 한 프레임에 여러 번 또는 0번 호출될 수 있음
    /// </summary>
    private void FixedUpdate()
    {
        // 물리 기반 이동
        // Rigidbody.AddForce 등은 여기서 호출
        // Debug.Log("FixedUpdate() - 물리 업데이트");
    }

    // =============================================
    // 게임 로직 업데이트 단계 (Game Logic Phase)
    // =============================================

    /// <summary>
    /// 매 프레임 호출
    /// - 대부분의 게임 로직 처리
    /// - 입력 처리
    /// - 비물리 기반 이동
    /// </summary>
    private void Update()
    {
        // 입력 처리
        // if (Input.GetKeyDown(KeyCode.Space)) { }

        // 비물리 이동
        // transform.Translate(Vector3.forward * Time.deltaTime);

        // Debug.Log("Update() - 매 프레임");
    }

    /// <summary>
    /// 모든 Update() 호출 후 실행
    /// - 카메라 추적에 적합
    /// - 다른 오브젝트의 Update 결과에 의존하는 로직
    /// </summary>
    private void LateUpdate()
    {
        // 카메라가 플레이어를 따라가는 로직
        // transform.position = player.position + offset;

        // Debug.Log("LateUpdate() - Update 후");
    }

    // =============================================
    // 렌더링 단계 (Rendering Phase)
    // =============================================

    /// <summary>
    /// GUI 렌더링 시 호출 (IMGUI용)
    /// - 한 프레임에 여러 번 호출될 수 있음
    /// - 현재는 UI Toolkit이나 uGUI 권장
    /// </summary>
    private void OnGUI()
    {
        // 레거시 GUI 렌더링
        // GUI.Label(new Rect(10, 10, 100, 20), "Hello");
    }

    /// <summary>
    /// 카메라가 씬을 렌더링할 때 호출
    /// </summary>
    private void OnRenderObject()
    {
        // 커스텀 렌더링 로직
    }

    // =============================================
    // 종료 단계 (Decommissioning Phase)
    // =============================================

    /// <summary>
    /// 오브젝트가 비활성화될 때 호출
    /// </summary>
    private void OnDisable()
    {
        Debug.Log("OnDisable() - 비활성화됨");
    }

    /// <summary>
    /// 오브젝트가 파괴될 때 호출
    /// - 정리 로직 (이벤트 구독 해제, 리소스 해제 등)
    /// </summary>
    private void OnDestroy()
    {
        Debug.Log("OnDestroy() - 파괴됨");
    }

    /// <summary>
    /// 애플리케이션 종료 시 호출
    /// </summary>
    private void OnApplicationQuit()
    {
        Debug.Log("OnApplicationQuit() - 앱 종료");
    }

    // =============================================
    // Coroutine 실행 시점
    // =============================================

    private IEnumerator CoroutineLifecycle()
    {
        Debug.Log("Coroutine: 시작");

        // yield return null - Update() 후에 재개
        yield return null;
        Debug.Log("Coroutine: yield null 후 (Update 다음)");

        // yield return WaitForFixedUpdate - FixedUpdate() 후에 재개
        yield return new WaitForFixedUpdate();
        Debug.Log("Coroutine: WaitForFixedUpdate 후");

        // yield return WaitForEndOfFrame - 프레임 렌더링 후에 재개
        yield return new WaitForEndOfFrame();
        Debug.Log("Coroutine: WaitForEndOfFrame 후");

        // yield return WaitForSeconds - 지정된 시간 후에 재개
        yield return new WaitForSeconds(1f);
        Debug.Log("Coroutine: WaitForSeconds(1) 후");
    }
}
```

### 라이프사이클 다이어그램

```
                    ┌──────────────────────────────────────────┐
                    │              Scene Load                   │
                    └──────────────────────────────────────────┘
                                        │
                    ┌───────────────────▼───────────────────┐
                    │               Awake()                  │
                    │      (모든 오브젝트에서 호출)            │
                    └───────────────────────────────────────┘
                                        │
                    ┌───────────────────▼───────────────────┐
                    │             OnEnable()                 │
                    │        (활성화된 오브젝트만)             │
                    └───────────────────────────────────────┘
                                        │
                    ┌───────────────────▼───────────────────┐
                    │               Start()                  │
                    │         (첫 프레임 전에 1회)            │
                    └───────────────────────────────────────┘
                                        │
            ┌───────────────────────────┼───────────────────────────┐
            │                    Game Loop                           │
            │  ┌────────────────────────────────────────────────┐   │
            │  │                                                 │   │
            │  │  ┌─────────────────────────────────────────┐  │   │
            │  │  │  FixedUpdate() [0~N회]                   │  │   │
            │  │  │  • 물리 시뮬레이션                        │  │   │
            │  │  │  • yield WaitForFixedUpdate 재개        │  │   │
            │  │  └─────────────────────────────────────────┘  │   │
            │  │                      │                         │   │
            │  │  ┌─────────────────────────────────────────┐  │   │
            │  │  │  Update() [1회]                          │  │   │
            │  │  │  • 입력 처리, 게임 로직                   │  │   │
            │  │  │  • yield null 재개                       │  │   │
            │  │  │  • yield WaitForSeconds 재개            │  │   │
            │  │  └─────────────────────────────────────────┘  │   │
            │  │                      │                         │   │
            │  │  ┌─────────────────────────────────────────┐  │   │
            │  │  │  LateUpdate() [1회]                      │  │   │
            │  │  │  • 카메라 추적 등                         │  │   │
            │  │  └─────────────────────────────────────────┘  │   │
            │  │                      │                         │   │
            │  │  ┌─────────────────────────────────────────┐  │   │
            │  │  │  Rendering                               │  │   │
            │  │  │  • yield WaitForEndOfFrame 재개         │  │   │
            │  │  └─────────────────────────────────────────┘  │   │
            │  │                      │                         │   │
            │  └──────────────────────┼─────────────────────────┘   │
            │                    다음 프레임                         │
            └───────────────────────────────────────────────────────┘
```

---

## 4. 동시성과 라이프사이클의 관계

### Coroutine 실행 시점

```csharp
using UnityEngine;
using System.Collections;

public class CoroutineTimingExample : MonoBehaviour
{
    private void Start()
    {
        StartCoroutine(DemonstrateCoroutineTiming());
    }

    private IEnumerator DemonstrateCoroutineTiming()
    {
        Debug.Log("=== Coroutine Yield 타이밍 테스트 ===");

        // 1. yield return null
        // 실행 시점: 다음 프레임의 Update() 후
        Debug.Log($"Frame {Time.frameCount}: yield null 전");
        yield return null;
        Debug.Log($"Frame {Time.frameCount}: yield null 후 (Update 다음)");

        // 2. yield return WaitForFixedUpdate
        // 실행 시점: 다음 FixedUpdate() 후
        Debug.Log($"Frame {Time.frameCount}: WaitForFixedUpdate 전");
        yield return new WaitForFixedUpdate();
        Debug.Log($"Frame {Time.frameCount}: WaitForFixedUpdate 후");

        // 3. yield return WaitForEndOfFrame
        // 실행 시점: 현재 프레임 렌더링 완료 후
        Debug.Log($"Frame {Time.frameCount}: WaitForEndOfFrame 전");
        yield return new WaitForEndOfFrame();
        Debug.Log($"Frame {Time.frameCount}: WaitForEndOfFrame 후 (같은 프레임)");

        // 4. yield return WaitForSeconds
        // 실행 시점: 지정된 시간(게임 시간) 후의 Update() 다음
        Debug.Log($"Frame {Time.frameCount}: WaitForSeconds(0.5) 전, Time: {Time.time}");
        yield return new WaitForSeconds(0.5f);
        Debug.Log($"Frame {Time.frameCount}: WaitForSeconds(0.5) 후, Time: {Time.time}");

        // 5. yield return WaitForSecondsRealtime
        // 실행 시점: 지정된 실제 시간 후 (Time.timeScale 무시)
        Time.timeScale = 0.5f; // 게임 시간 느리게
        Debug.Log($"Frame {Time.frameCount}: WaitForSecondsRealtime(1) 전");
        yield return new WaitForSecondsRealtime(1f);
        Debug.Log($"Frame {Time.frameCount}: WaitForSecondsRealtime(1) 후");
        Time.timeScale = 1f;

        // 6. yield return WaitUntil
        // 실행 시점: 조건이 true가 될 때
        int counter = 0;
        Debug.Log("WaitUntil 시작 (counter >= 3 될 때까지)");
        StartCoroutine(IncrementCounter(() => counter++));
        yield return new WaitUntil(() => counter >= 3);
        Debug.Log($"WaitUntil 완료: counter = {counter}");

        // 7. yield return WaitWhile
        // 실행 시점: 조건이 false가 될 때
        bool isWaiting = true;
        Debug.Log("WaitWhile 시작");
        Invoke(nameof(StopWaiting), 0.5f);
        yield return new WaitWhile(() => isWaiting);
        Debug.Log("WaitWhile 완료");

        void StopWaiting() => isWaiting = false;
    }

    private IEnumerator IncrementCounter(System.Action increment)
    {
        for (int i = 0; i < 5; i++)
        {
            yield return new WaitForSeconds(0.2f);
            increment();
        }
    }
}
```

### async/await와 라이프사이클

```csharp
using UnityEngine;
using System.Threading.Tasks;

public class AsyncAwaitTimingExample : MonoBehaviour
{
    private void Start()
    {
        // async void는 fire-and-forget
        TestAsyncTiming();
    }

    private async void TestAsyncTiming()
    {
        Debug.Log($"Frame {Time.frameCount}: async 시작");

        // Task.Yield - 다음 프레임까지 대기 (Unity에서는 권장하지 않음)
        // ⚠️ Task.Yield는 Unity의 SynchronizationContext에 의해
        // 메인 스레드로 돌아오지만, 정확한 타이밍은 보장되지 않음
        await Task.Yield();
        Debug.Log($"Frame {Time.frameCount}: Task.Yield 후");

        // Task.Delay - 지정된 시간만큼 대기 (실제 시간)
        // ⚠️ Task.Delay는 ThreadPool 타이머 사용, 정확도 낮음
        Debug.Log($"Frame {Time.frameCount}: Task.Delay(100) 전");
        await Task.Delay(100);
        Debug.Log($"Frame {Time.frameCount}: Task.Delay(100) 후");

        // ✅ Unity 2023+에서는 Awaitable 사용 권장
        // await Awaitable.NextFrameAsync();
        // await Awaitable.WaitForSecondsAsync(0.5f);
    }

    // =============================================
    // async/await에서 Unity API 호출
    // =============================================

    private async void AsyncUnityAPIExample()
    {
        Debug.Log("async 메서드 시작 - 메인 스레드");

        // ConfigureAwait(true) 또는 기본값
        // Unity의 SynchronizationContext가 메인 스레드로 복귀시킴
        await Task.Delay(100);
        // ✅ 메인 스레드이므로 Unity API 사용 가능
        transform.position = Vector3.zero;

        // Task.Run 내부는 백그라운드 스레드
        await Task.Run(() =>
        {
            // ❌ 여기서는 Unity API 사용 불가!
            // transform.position = Vector3.one; // 에러 발생!

            // 순수 C# 연산만 가능
            int result = HeavyCalculation();
            return result;
        });

        // Task.Run 완료 후 다시 메인 스레드
        // ✅ Unity API 사용 가능
        Debug.Log("Task.Run 완료 후 - 메인 스레드");
        transform.position = Vector3.one;
    }

    private int HeavyCalculation()
    {
        int sum = 0;
        for (int i = 0; i < 1000000; i++)
        {
            sum += i;
        }
        return sum;
    }
}
```

---

## 5. 메인 스레드 제약사항

### Unity API의 메인 스레드 요구사항

```csharp
using UnityEngine;
using System.Threading;
using System.Threading.Tasks;
using System.Collections.Concurrent;

public class MainThreadRestrictionExample : MonoBehaviour
{
    // 메인 스레드에서만 호출 가능한 Unity API 목록:
    /*
    ┌─────────────────────────────────────────────────────────────┐
    │              메인 스레드 전용 Unity API                       │
    ├─────────────────────────────────────────────────────────────┤
    │                                                              │
    │  GameObject 관련:                                            │
    │    • GameObject.Find, FindWithTag                           │
    │    • Instantiate, Destroy                                   │
    │    • SetActive                                               │
    │    • AddComponent, GetComponent                             │
    │                                                              │
    │  Transform 관련:                                             │
    │    • position, rotation, scale                              │
    │    • parent, SetParent                                      │
    │    • Translate, Rotate                                      │
    │                                                              │
    │  렌더링 관련:                                                 │
    │    • Material 속성 변경                                       │
    │    • Renderer.enabled                                        │
    │    • Camera 속성                                             │
    │                                                              │
    │  물리 관련:                                                   │
    │    • Rigidbody.AddForce                                     │
    │    • Physics.Raycast                                        │
    │                                                              │
    │  UI 관련:                                                    │
    │    • Canvas, UI 컴포넌트                                     │
    │    • Text, Image 등                                         │
    │                                                              │
    │  기타:                                                       │
    │    • Debug.Log (스레드 안전하지만 권장하지 않음)                │
    │    • Resources.Load                                         │
    │    • SceneManager                                           │
    │                                                              │
    └─────────────────────────────────────────────────────────────┘
    */

    // 작업 큐를 사용한 메인 스레드 마샬링
    private ConcurrentQueue<System.Action> mainThreadActions = new ConcurrentQueue<System.Action>();

    private void Update()
    {
        // 매 프레임 큐에 있는 액션들을 메인 스레드에서 실행
        while (mainThreadActions.TryDequeue(out var action))
        {
            action?.Invoke();
        }
    }

    // =============================================
    // 잘못된 예제: 백그라운드 스레드에서 Unity API 호출
    // =============================================

    private void WrongExample()
    {
        Task.Run(() =>
        {
            // ❌ UnityException: get_transform can only be called from the main thread.
            // transform.position = Vector3.zero;

            // ❌ 마찬가지로 실패
            // gameObject.SetActive(false);

            // ❌ 이것도 실패
            // var obj = GameObject.Find("Player");
        });
    }

    // =============================================
    // 올바른 예제: 작업 큐를 통한 메인 스레드 마샬링
    // =============================================

    private async void CorrectExample()
    {
        Debug.Log("백그라운드 작업 시작");

        // 무거운 연산은 백그라운드에서
        Vector3 calculatedPosition = await Task.Run(() =>
        {
            // CPU 집약적 연산
            float x = 0, y = 0, z = 0;
            for (int i = 0; i < 1000000; i++)
            {
                x += Mathf.Sin(i * 0.001f);
                y += Mathf.Cos(i * 0.001f);
                z += i * 0.0001f;
            }
            return new Vector3(x, y, z);
        });

        // await 이후에는 자동으로 메인 스레드로 복귀 (SynchronizationContext 덕분)
        // ✅ 이제 Unity API 사용 가능
        transform.position = calculatedPosition;
        Debug.Log($"위치 설정 완료: {calculatedPosition}");
    }

    // =============================================
    // 수동 마샬링 예제 (콜백 방식)
    // =============================================

    private void ManualMarshalingExample()
    {
        // 별도 스레드에서 작업 수행
        new Thread(() =>
        {
            // 무거운 연산 수행
            int result = 0;
            for (int i = 0; i < 10000000; i++)
            {
                result += i;
            }

            // 결과를 메인 스레드에서 처리하도록 큐에 추가
            mainThreadActions.Enqueue(() =>
            {
                // ✅ 이 코드는 메인 스레드(Update)에서 실행됨
                Debug.Log($"계산 결과: {result}");
                transform.position = new Vector3(result % 10, 0, 0);
            });

        }).Start();
    }

    // =============================================
    // UnitySynchronizationContext 활용 예제
    // =============================================

    private SynchronizationContext unitySyncContext;

    private void Awake()
    {
        // 메인 스레드의 SynchronizationContext 캡처
        unitySyncContext = SynchronizationContext.Current;
    }

    private void SyncContextExample()
    {
        Task.Run(() =>
        {
            // 백그라운드 스레드에서 연산
            int result = CalculateHeavyTask();

            // SynchronizationContext.Post로 메인 스레드에 작업 전달
            unitySyncContext.Post(_ =>
            {
                // ✅ 메인 스레드에서 실행됨
                transform.position = new Vector3(result, 0, 0);
                Debug.Log($"SyncContext로 마샬링된 결과: {result}");
            }, null);
        });
    }

    private int CalculateHeavyTask()
    {
        int sum = 0;
        for (int i = 0; i < 1000000; i++)
        {
            sum += i % 100;
        }
        return sum;
    }
}
```

---

## 6. FixedUpdate vs Update: 시간 관리

### Time.deltaTime vs Time.fixedDeltaTime

```csharp
using UnityEngine;

public class TimeManagementExample : MonoBehaviour
{
    [SerializeField] private Transform physicsObject;
    [SerializeField] private Transform nonPhysicsObject;

    public float moveSpeed = 5f;

    // =============================================
    // Update: 프레임 기반 업데이트
    // =============================================

    private void Update()
    {
        // Time.deltaTime: 이전 프레임과의 시간 간격
        // 프레임 레이트가 변해도 일정한 속도로 이동
        float movement = moveSpeed * Time.deltaTime;

        // 비물리 기반 이동에 적합
        if (nonPhysicsObject != null)
        {
            nonPhysicsObject.Translate(Vector3.forward * movement);
        }

        // ⚠️ Update에서 물리 조작은 권장하지 않음
        // Rigidbody.AddForce 등은 FixedUpdate에서 호출할 것
    }

    // =============================================
    // FixedUpdate: 고정 시간 간격 업데이트
    // =============================================

    private Rigidbody rb;

    private void Awake()
    {
        rb = physicsObject?.GetComponent<Rigidbody>();
    }

    private void FixedUpdate()
    {
        // Time.fixedDeltaTime: 고정 시간 간격 (기본 0.02초)
        // 물리 시뮬레이션의 일관성 보장

        if (rb != null)
        {
            // 물리 기반 이동
            rb.AddForce(Vector3.forward * moveSpeed);
        }

        // 물리 관련 레이캐스트도 FixedUpdate에서
        // Physics.Raycast(...);
    }

    // =============================================
    // 시간 관련 속성 비교
    // =============================================

    private void DebugTimeProperties()
    {
        // Time.time: 게임 시작 후 경과 시간 (timeScale 영향 받음)
        Debug.Log($"Time.time: {Time.time}");

        // Time.unscaledTime: 실제 경과 시간 (timeScale 무시)
        Debug.Log($"Time.unscaledTime: {Time.unscaledTime}");

        // Time.deltaTime: 이전 프레임과의 시간 간격
        Debug.Log($"Time.deltaTime: {Time.deltaTime}");

        // Time.unscaledDeltaTime: 실제 시간 기준 프레임 간격
        Debug.Log($"Time.unscaledDeltaTime: {Time.unscaledDeltaTime}");

        // Time.fixedDeltaTime: 고정 시간 간격 (FixedUpdate 호출 간격)
        Debug.Log($"Time.fixedDeltaTime: {Time.fixedDeltaTime}");

        // Time.timeScale: 게임 시간 배율 (0 = 일시정지)
        Debug.Log($"Time.timeScale: {Time.timeScale}");

        // Time.frameCount: 시작 후 렌더링된 총 프레임 수
        Debug.Log($"Time.frameCount: {Time.frameCount}");
    }

    // =============================================
    // 일시정지 처리 예제
    // =============================================

    private bool isPaused = false;

    private void TogglePause()
    {
        isPaused = !isPaused;
        Time.timeScale = isPaused ? 0f : 1f;
        Debug.Log($"게임 {(isPaused ? "일시정지" : "재개")}");
    }

    // 일시정지 중에도 동작해야 하는 로직
    private void UpdatePauseMenu()
    {
        // Time.unscaledDeltaTime 사용
        // Time.timeScale이 0이어도 동작
        float uiMovement = 100f * Time.unscaledDeltaTime;
    }
}
```

### FixedUpdate 호출 횟수

```csharp
using UnityEngine;

public class FixedUpdateCountExample : MonoBehaviour
{
    private int updateCount = 0;
    private int fixedUpdateCount = 0;
    private float lastLogTime = 0;

    private void Update()
    {
        updateCount++;

        // 1초마다 호출 횟수 출력
        if (Time.time - lastLogTime >= 1f)
        {
            Debug.Log($"지난 1초: Update {updateCount}회, FixedUpdate {fixedUpdateCount}회");
            Debug.Log($"FPS: {1f / Time.deltaTime:F1}, 예상 FixedUpdate: {1f / Time.fixedDeltaTime:F0}회");

            updateCount = 0;
            fixedUpdateCount = 0;
            lastLogTime = Time.time;
        }
    }

    private void FixedUpdate()
    {
        fixedUpdateCount++;
    }

    /*
    예상 출력 (60 FPS, fixedDeltaTime = 0.02 기준):

    지난 1초: Update 60회, FixedUpdate 50회
    FPS: 60.0, 예상 FixedUpdate: 50회

    프레임 레이트가 낮아지면 (30 FPS):
    지난 1초: Update 30회, FixedUpdate 50회
    FPS: 30.0, 예상 FixedUpdate: 50회

    → FixedUpdate는 프레임 레이트와 관계없이 일정한 횟수 유지
    → 한 프레임에 여러 번 또는 0번 호출될 수 있음
    */
}
```

---

## 7. 실전 예제: 라이프사이클과 비동기 조합

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;
using System.Threading;
using System.Threading.Tasks;

public class LifecycleAsyncCombinationExample : MonoBehaviour
{
    private CancellationTokenSource cts;
    private Coroutine runningCoroutine;

    // =============================================
    // 생명주기 관리
    // =============================================

    private void Awake()
    {
        // CancellationTokenSource 생성
        cts = new CancellationTokenSource();
    }

    private void OnEnable()
    {
        // 비동기 작업 시작
        StartAsyncOperations();
    }

    private void OnDisable()
    {
        // Coroutine 정지
        if (runningCoroutine != null)
        {
            StopCoroutine(runningCoroutine);
            runningCoroutine = null;
        }
    }

    private void OnDestroy()
    {
        // 모든 비동기 작업 취소
        cts?.Cancel();
        cts?.Dispose();
        cts = null;
    }

    // =============================================
    // 비동기 작업들
    // =============================================

    private void StartAsyncOperations()
    {
        // 1. Coroutine 시작
        runningCoroutine = StartCoroutine(PeriodicWebRequest());

        // 2. async 작업 시작
        PeriodicAsyncTask(cts.Token);
    }

    // Coroutine 기반 주기적 웹 요청
    private IEnumerator PeriodicWebRequest()
    {
        while (true)
        {
            using (var request = UnityWebRequest.Get("https://api.example.com/status"))
            {
                yield return request.SendWebRequest();

                if (request.result == UnityWebRequest.Result.Success)
                {
                    Debug.Log($"[Coroutine] 응답: {request.downloadHandler.text}");
                }
            }

            yield return new WaitForSeconds(5f);
        }
    }

    // async/await 기반 주기적 작업
    private async void PeriodicAsyncTask(CancellationToken token)
    {
        try
        {
            while (!token.IsCancellationRequested)
            {
                // 백그라운드 스레드에서 무거운 연산
                int result = await Task.Run(() => HeavyCalculation(), token);

                // 메인 스레드로 복귀하여 결과 적용
                Debug.Log($"[Async] 계산 결과: {result}");

                // 대기 (취소 토큰 전달)
                await Task.Delay(3000, token);
            }
        }
        catch (TaskCanceledException)
        {
            Debug.Log("[Async] 작업이 취소되었습니다.");
        }
    }

    private int HeavyCalculation()
    {
        Thread.Sleep(100); // 무거운 연산 시뮬레이션
        return Random.Range(1, 100);
    }

    // =============================================
    // Scene 전환 시 주의사항
    // =============================================

    /*
    Scene 전환 시 발생할 수 있는 문제:

    1. MissingReferenceException
       - Scene이 전환되어 오브젝트가 파괴됨
       - 비동기 작업 완료 후 파괴된 오브젝트 접근 시도

    2. 메모리 누수
       - 취소되지 않은 비동기 작업이 계속 실행
       - 이벤트 구독 해제 안 함

    해결책:
    - OnDestroy에서 CancellationToken 취소
    - null 체크 또는 destroyed 플래그 사용
    - 이벤트 구독 해제
    */

    private bool isDestroyed = false;

    private async void SafeAsyncOperation()
    {
        await Task.Delay(1000);

        // 오브젝트 파괴 여부 확인
        if (isDestroyed || this == null)
        {
            return;
        }

        // 안전하게 Unity API 사용
        transform.position = Vector3.zero;
    }
}
```

---

## 주의사항

1. **Unity API는 메인 스레드에서만 호출**: Transform, GameObject 등은 백그라운드 스레드에서 접근 불가
2. **Coroutine은 메인 스레드에서 실행**: yield 전후로 모두 메인 스레드
3. **async/await의 SynchronizationContext**: Unity의 컨텍스트가 await 후 메인 스레드로 복귀시킴
4. **FixedUpdate의 호출 빈도**: 프레임 레이트와 무관하게 고정 간격으로 호출
5. **생명주기 관리 필수**: OnDestroy에서 비동기 작업 취소, 리소스 해제

---

## 참고 자료

- [Unity: Order of Execution](https://docs.unity3d.com/Manual/ExecutionOrder.html)
- [Unity: PlayerLoop](https://docs.unity3d.com/ScriptReference/LowLevel.PlayerLoop.html)
- [Unity: Coroutines](https://docs.unity3d.com/Manual/Coroutines.html)
- [Unity: Time and Frame Rate Management](https://docs.unity3d.com/Manual/TimeFrameManagement.html)

---

## 다음 섹션

[03. SynchronizationContext](./03-synchronization-context.md)
