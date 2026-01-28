# 07. Coroutine

## 개요

Coroutine은 Unity의 전통적인 비동기 처리 방식입니다. `IEnumerator`와 `yield` 키워드를 사용하여 작업을 여러 프레임에 걸쳐 분산 실행할 수 있습니다. 메인 스레드에서 실행되므로 Unity API를 안전하게 사용할 수 있다는 장점이 있습니다.

---

## 1. Coroutine 기본 개념

### 동작 원리

```
┌─────────────────────────────────────────────────────────────────┐
│                    Coroutine 동작 원리                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Frame 1                                                        │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Update() → Coroutine 실행 시작                              │ │
│  │            │                                                │ │
│  │            ▼                                                │ │
│  │     yield return null ──▶ 여기서 일시 정지                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Frame 2                                                        │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Update() → Coroutine 재개                                   │ │
│  │            │                                                │ │
│  │            ▼                                                │ │
│  │     yield return null ──▶ 다시 일시 정지                     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Frame 3                                                        │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Update() → Coroutine 재개                                   │ │
│  │            │                                                │ │
│  │            ▼                                                │ │
│  │     Coroutine 완료                                          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  핵심:                                                           │
│  • 모든 코드는 메인 스레드에서 실행                                │
│  • yield에서 실행을 "양보"하고 다음 프레임에 재개                   │
│  • 병렬 실행이 아닌 협력적 멀티태스킹                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 기본 구문

```csharp
using UnityEngine;
using System.Collections;

public class CoroutineBasicsExample : MonoBehaviour
{
    private void Start()
    {
        // Coroutine 시작 방법들
        StartCoroutine(SimpleCoroutine());
        StartCoroutine("SimpleCoroutineByName"); // 문자열 버전
        StartCoroutine(CoroutineWithParameter(5));
    }

    // =============================================
    // 기본 Coroutine
    // =============================================

    private IEnumerator SimpleCoroutine()
    {
        Debug.Log("Coroutine 시작");

        // 다음 프레임까지 대기
        yield return null;

        Debug.Log("1프레임 후");

        // 다시 대기
        yield return null;

        Debug.Log("2프레임 후 - 완료");
    }

    // 문자열로 호출할 수 있는 Coroutine
    private IEnumerator SimpleCoroutineByName()
    {
        yield return null;
        Debug.Log("문자열로 시작된 Coroutine");
    }

    // 매개변수가 있는 Coroutine
    private IEnumerator CoroutineWithParameter(int count)
    {
        for (int i = 0; i < count; i++)
        {
            Debug.Log($"카운트: {i + 1}/{count}");
            yield return null;
        }
        Debug.Log("카운트 완료");
    }

    // =============================================
    // Coroutine 중지
    // =============================================

    private Coroutine runningCoroutine;

    private void StartAndStop()
    {
        // 참조 저장
        runningCoroutine = StartCoroutine(LongRunningCoroutine());

        // 특정 Coroutine 중지
        StopCoroutine(runningCoroutine);

        // 문자열로 중지 (문자열로 시작한 경우만)
        StopCoroutine("SimpleCoroutineByName");

        // 이 오브젝트의 모든 Coroutine 중지
        StopAllCoroutines();
    }

    private IEnumerator LongRunningCoroutine()
    {
        while (true)
        {
            Debug.Log("실행 중...");
            yield return new WaitForSeconds(1f);
        }
    }
}
```

---

## 2. yield return 종류

### 모든 yield 타입

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;

public class YieldTypesExample : MonoBehaviour
{
    private void Start()
    {
        StartCoroutine(DemonstrateAllYieldTypes());
    }

    private IEnumerator DemonstrateAllYieldTypes()
    {
        Debug.Log($"=== yield 타입 데모 시작 (Frame: {Time.frameCount}) ===");

        // =============================================
        // 1. yield return null - 다음 프레임까지 대기
        // =============================================
        Debug.Log($"yield return null 전 (Frame: {Time.frameCount})");
        yield return null;
        Debug.Log($"yield return null 후 (Frame: {Time.frameCount})");

        // =============================================
        // 2. WaitForSeconds - 지정 시간(게임 시간) 대기
        // =============================================
        Debug.Log($"WaitForSeconds(0.5) 전 (Time: {Time.time:F2})");
        yield return new WaitForSeconds(0.5f);
        Debug.Log($"WaitForSeconds(0.5) 후 (Time: {Time.time:F2})");

        // =============================================
        // 3. WaitForSecondsRealtime - 실제 시간 대기
        // =============================================
        // Time.timeScale의 영향을 받지 않음
        Debug.Log($"WaitForSecondsRealtime(0.5) 전");
        yield return new WaitForSecondsRealtime(0.5f);
        Debug.Log($"WaitForSecondsRealtime(0.5) 후");

        // =============================================
        // 4. WaitForEndOfFrame - 프레임 렌더링 완료 후
        // =============================================
        Debug.Log($"WaitForEndOfFrame 전 (Frame: {Time.frameCount})");
        yield return new WaitForEndOfFrame();
        Debug.Log($"WaitForEndOfFrame 후 (Frame: {Time.frameCount})"); // 같은 프레임

        // =============================================
        // 5. WaitForFixedUpdate - 다음 FixedUpdate 후
        // =============================================
        Debug.Log($"WaitForFixedUpdate 전");
        yield return new WaitForFixedUpdate();
        Debug.Log($"WaitForFixedUpdate 후");

        // =============================================
        // 6. WaitUntil - 조건이 true가 될 때까지
        // =============================================
        bool condition = false;
        StartCoroutine(SetConditionAfterDelay(() => condition = true, 1f));
        Debug.Log("WaitUntil 대기 시작...");
        yield return new WaitUntil(() => condition);
        Debug.Log("WaitUntil 완료 - condition이 true가 됨");

        // =============================================
        // 7. WaitWhile - 조건이 false가 될 때까지
        // =============================================
        bool waiting = true;
        StartCoroutine(SetConditionAfterDelay(() => waiting = false, 1f));
        Debug.Log("WaitWhile 대기 시작...");
        yield return new WaitWhile(() => waiting);
        Debug.Log("WaitWhile 완료 - waiting이 false가 됨");

        // =============================================
        // 8. 다른 Coroutine 대기
        // =============================================
        Debug.Log("중첩 Coroutine 시작...");
        yield return StartCoroutine(NestedCoroutine());
        Debug.Log("중첩 Coroutine 완료");

        // =============================================
        // 9. AsyncOperation 대기 (씬 로드, 에셋번들 등)
        // =============================================
        Debug.Log("AsyncOperation 예제 (주석 처리됨)");
        // yield return SceneManager.LoadSceneAsync("SceneName");
        // yield return AssetBundle.LoadFromFileAsync(path);

        // =============================================
        // 10. UnityWebRequest
        // =============================================
        Debug.Log("웹 요청 시작...");
        using (UnityWebRequest request = UnityWebRequest.Get("https://httpbin.org/get"))
        {
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"웹 요청 성공: {request.downloadHandler.text.Substring(0, 50)}...");
            }
            else
            {
                Debug.LogError($"웹 요청 실패: {request.error}");
            }
        }

        Debug.Log("=== 모든 yield 타입 데모 완료 ===");
    }

    private IEnumerator NestedCoroutine()
    {
        Debug.Log("  중첩 Coroutine 실행 중...");
        yield return new WaitForSeconds(0.5f);
        Debug.Log("  중첩 Coroutine 종료");
    }

    private IEnumerator SetConditionAfterDelay(System.Action action, float delay)
    {
        yield return new WaitForSeconds(delay);
        action();
    }
}
```

### 실행 시점 정리

```
┌─────────────────────────────────────────────────────────────────┐
│                  yield return 실행 시점 정리                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  yield return           │  재개 시점                             │
│  ──────────────────────┼─────────────────────────────────────── │
│  null                   │  다음 프레임의 Update 후               │
│  WaitForSeconds(n)      │  n초(게임시간) 후의 Update 후          │
│  WaitForSecondsRealtime │  n초(실제시간) 후                      │
│  WaitForEndOfFrame      │  현재 프레임의 렌더링 완료 후          │
│  WaitForFixedUpdate     │  다음 FixedUpdate 후                   │
│  WaitUntil(condition)   │  condition이 true가 된 후 Update 후   │
│  WaitWhile(condition)   │  condition이 false가 된 후 Update 후  │
│  Coroutine              │  해당 Coroutine 완료 후                 │
│  AsyncOperation         │  비동기 작업 완료 후                   │
│  CustomYieldInstruction │  keepWaiting이 false가 된 후          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. CustomYieldInstruction

### 커스텀 대기 조건 만들기

```csharp
using UnityEngine;
using System.Collections;

public class CustomYieldExample : MonoBehaviour
{
    private void Start()
    {
        StartCoroutine(TestCustomYield());
    }

    private IEnumerator TestCustomYield()
    {
        Debug.Log("커스텀 yield 테스트 시작");

        // 키 입력 대기
        yield return new WaitForKeyPress(KeyCode.Space);
        Debug.Log("스페이스바 눌림!");

        // 오브젝트가 특정 위치에 도달할 때까지 대기
        yield return new WaitForPosition(transform, new Vector3(10, 0, 0), 0.5f);
        Debug.Log("목표 위치 도달!");

        // 여러 조건 조합
        yield return new WaitForAll(
            new WaitForSeconds(1f),
            new WaitForKeyPress(KeyCode.Return)
        );
        Debug.Log("1초 + 엔터키 완료!");
    }
}

// =============================================
// CustomYieldInstruction 예제들
// =============================================

/// <summary>
/// 특정 키가 눌릴 때까지 대기
/// </summary>
public class WaitForKeyPress : CustomYieldInstruction
{
    private KeyCode targetKey;

    public WaitForKeyPress(KeyCode key)
    {
        targetKey = key;
    }

    public override bool keepWaiting
    {
        get { return !Input.GetKeyDown(targetKey); }
    }
}

/// <summary>
/// Transform이 목표 위치에 도달할 때까지 대기
/// </summary>
public class WaitForPosition : CustomYieldInstruction
{
    private Transform transform;
    private Vector3 targetPosition;
    private float threshold;

    public WaitForPosition(Transform t, Vector3 target, float thresh = 0.1f)
    {
        transform = t;
        targetPosition = target;
        threshold = thresh;
    }

    public override bool keepWaiting
    {
        get
        {
            return Vector3.Distance(transform.position, targetPosition) > threshold;
        }
    }
}

/// <summary>
/// 모든 조건이 완료될 때까지 대기
/// </summary>
public class WaitForAll : CustomYieldInstruction
{
    private CustomYieldInstruction[] instructions;

    public WaitForAll(params CustomYieldInstruction[] instr)
    {
        instructions = instr;
    }

    public override bool keepWaiting
    {
        get
        {
            foreach (var instruction in instructions)
            {
                if (instruction.keepWaiting)
                    return true;
            }
            return false;
        }
    }
}

/// <summary>
/// 조건 중 하나라도 완료되면 진행
/// </summary>
public class WaitForAny : CustomYieldInstruction
{
    private CustomYieldInstruction[] instructions;

    public WaitForAny(params CustomYieldInstruction[] instr)
    {
        instructions = instr;
    }

    public override bool keepWaiting
    {
        get
        {
            foreach (var instruction in instructions)
            {
                if (!instruction.keepWaiting)
                    return false;
            }
            return true;
        }
    }
}

/// <summary>
/// 애니메이션 완료 대기
/// </summary>
public class WaitForAnimation : CustomYieldInstruction
{
    private Animator animator;
    private string stateName;
    private int layer;

    public WaitForAnimation(Animator anim, string state, int layerIndex = 0)
    {
        animator = anim;
        stateName = state;
        layer = layerIndex;
    }

    public override bool keepWaiting
    {
        get
        {
            var stateInfo = animator.GetCurrentAnimatorStateInfo(layer);
            return stateInfo.IsName(stateName) && stateInfo.normalizedTime < 1f;
        }
    }
}
```

---

## 4. Coroutine 활용 패턴

### 패턴 1: 시간 기반 애니메이션

```csharp
using UnityEngine;
using System.Collections;

public class CoroutineAnimationExample : MonoBehaviour
{
    [SerializeField] private float duration = 2f;

    private void Start()
    {
        // 페이드 인
        StartCoroutine(FadeIn());

        // 이동 애니메이션
        StartCoroutine(MoveTo(new Vector3(10, 0, 0), duration));

        // 크기 변경
        StartCoroutine(ScaleTo(Vector3.one * 2f, duration));
    }

    // =============================================
    // 페이드 애니메이션
    // =============================================

    private IEnumerator FadeIn()
    {
        SpriteRenderer sr = GetComponent<SpriteRenderer>();
        if (sr == null) yield break;

        Color color = sr.color;
        color.a = 0f;
        sr.color = color;

        float elapsed = 0f;
        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            float t = elapsed / duration;

            color.a = Mathf.Lerp(0f, 1f, t);
            sr.color = color;

            yield return null;
        }

        color.a = 1f;
        sr.color = color;
    }

    // =============================================
    // 이동 애니메이션
    // =============================================

    private IEnumerator MoveTo(Vector3 targetPosition, float time)
    {
        Vector3 startPosition = transform.position;
        float elapsed = 0f;

        while (elapsed < time)
        {
            elapsed += Time.deltaTime;
            float t = elapsed / time;

            // 이징 적용
            float easedT = EaseOutQuad(t);

            transform.position = Vector3.Lerp(startPosition, targetPosition, easedT);
            yield return null;
        }

        transform.position = targetPosition;
    }

    // 이징 함수
    private float EaseOutQuad(float t)
    {
        return 1f - (1f - t) * (1f - t);
    }

    // =============================================
    // 스케일 애니메이션
    // =============================================

    private IEnumerator ScaleTo(Vector3 targetScale, float time)
    {
        Vector3 startScale = transform.localScale;
        float elapsed = 0f;

        while (elapsed < time)
        {
            elapsed += Time.deltaTime;
            float t = Mathf.Clamp01(elapsed / time);

            transform.localScale = Vector3.Lerp(startScale, targetScale, t);
            yield return null;
        }

        transform.localScale = targetScale;
    }

    // =============================================
    // 범용 트윈 유틸리티
    // =============================================

    public static IEnumerator Tween(float duration, System.Action<float> onUpdate, System.Func<float, float> easing = null)
    {
        float elapsed = 0f;
        easing = easing ?? (t => t); // 기본: 선형

        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            float t = Mathf.Clamp01(elapsed / duration);
            float easedT = easing(t);

            onUpdate(easedT);
            yield return null;
        }

        onUpdate(1f);
    }
}
```

### 패턴 2: 순차적 작업 실행

```csharp
using UnityEngine;
using System.Collections;
using System.Collections.Generic;

public class SequentialTasksExample : MonoBehaviour
{
    private void Start()
    {
        StartCoroutine(SequentialTasks());
        StartCoroutine(ChainedCoroutines());
    }

    // =============================================
    // 순차 실행
    // =============================================

    private IEnumerator SequentialTasks()
    {
        Debug.Log("=== 순차 작업 시작 ===");

        yield return StartCoroutine(Task("Task 1", 1f));
        yield return StartCoroutine(Task("Task 2", 0.5f));
        yield return StartCoroutine(Task("Task 3", 0.8f));

        Debug.Log("=== 모든 순차 작업 완료 ===");
    }

    private IEnumerator Task(string name, float duration)
    {
        Debug.Log($"{name} 시작");
        yield return new WaitForSeconds(duration);
        Debug.Log($"{name} 완료");
    }

    // =============================================
    // 체이닝 패턴
    // =============================================

    private IEnumerator ChainedCoroutines()
    {
        yield return new WaitForSeconds(5f); // 이전 예제 후 실행

        Debug.Log("=== 체인 실행 시작 ===");

        yield return LoadData()
            .Then(() => ProcessData())
            .Then(() => SaveData())
            .Then(() => Debug.Log("체인 완료!"));
    }

    private IEnumerator LoadData()
    {
        Debug.Log("데이터 로딩...");
        yield return new WaitForSeconds(0.5f);
        Debug.Log("데이터 로딩 완료");
    }

    private IEnumerator ProcessData()
    {
        Debug.Log("데이터 처리...");
        yield return new WaitForSeconds(0.3f);
        Debug.Log("데이터 처리 완료");
    }

    private IEnumerator SaveData()
    {
        Debug.Log("데이터 저장...");
        yield return new WaitForSeconds(0.2f);
        Debug.Log("데이터 저장 완료");
    }
}

// =============================================
// Coroutine 체이닝 확장 메서드
// =============================================

public static class CoroutineChainExtensions
{
    public static IEnumerator Then(this IEnumerator first, System.Func<IEnumerator> next)
    {
        yield return first;
        yield return next();
    }

    public static IEnumerator Then(this IEnumerator first, System.Action action)
    {
        yield return first;
        action();
    }
}
```

### 패턴 3: 병렬 실행 및 대기

```csharp
using UnityEngine;
using System.Collections;
using System.Collections.Generic;

public class ParallelCoroutinesExample : MonoBehaviour
{
    private void Start()
    {
        StartCoroutine(ParallelExecution());
        StartCoroutine(WaitForAllCoroutines());
    }

    // =============================================
    // 병렬 실행 (fire-and-forget)
    // =============================================

    private IEnumerator ParallelExecution()
    {
        Debug.Log("=== 병렬 실행 시작 ===");

        // 여러 Coroutine을 동시에 시작 (병렬처럼 보임)
        StartCoroutine(DownloadAsset("Asset A", 2f));
        StartCoroutine(DownloadAsset("Asset B", 1f));
        StartCoroutine(DownloadAsset("Asset C", 1.5f));

        Debug.Log("모든 다운로드 시작됨 (병렬 실행)");

        yield return null;
    }

    private IEnumerator DownloadAsset(string name, float time)
    {
        Debug.Log($"{name} 다운로드 시작");
        yield return new WaitForSeconds(time);
        Debug.Log($"{name} 다운로드 완료 ({time}초 소요)");
    }

    // =============================================
    // 모든 Coroutine 완료 대기
    // =============================================

    private IEnumerator WaitForAllCoroutines()
    {
        yield return new WaitForSeconds(5f); // 이전 예제 후

        Debug.Log("=== 모두 완료 대기 시작 ===");

        List<Coroutine> coroutines = new List<Coroutine>();

        coroutines.Add(StartCoroutine(DownloadAsset("File 1", 1f)));
        coroutines.Add(StartCoroutine(DownloadAsset("File 2", 2f)));
        coroutines.Add(StartCoroutine(DownloadAsset("File 3", 0.5f)));

        // 모든 Coroutine이 완료될 때까지 대기
        foreach (var coroutine in coroutines)
        {
            yield return coroutine;
        }

        Debug.Log("=== 모든 다운로드 완료 ===");
    }

    // =============================================
    // 병렬 실행 유틸리티
    // =============================================

    /// <summary>
    /// 여러 Coroutine을 병렬로 실행하고 모두 완료될 때까지 대기
    /// </summary>
    public static IEnumerator WhenAll(MonoBehaviour runner, params IEnumerator[] coroutines)
    {
        int remaining = coroutines.Length;

        foreach (var coroutine in coroutines)
        {
            runner.StartCoroutine(RunAndDecrement(coroutine, () => remaining--));
        }

        yield return new WaitUntil(() => remaining == 0);
    }

    private static IEnumerator RunAndDecrement(IEnumerator coroutine, System.Action onComplete)
    {
        yield return coroutine;
        onComplete();
    }

    /// <summary>
    /// 하나라도 완료되면 진행
    /// </summary>
    public static IEnumerator WhenAny(MonoBehaviour runner, params IEnumerator[] coroutines)
    {
        bool anyCompleted = false;

        foreach (var coroutine in coroutines)
        {
            runner.StartCoroutine(RunAndSignal(coroutine, () => anyCompleted = true));
        }

        yield return new WaitUntil(() => anyCompleted);
    }

    private static IEnumerator RunAndSignal(IEnumerator coroutine, System.Action onComplete)
    {
        yield return coroutine;
        onComplete();
    }
}
```

### 패턴 4: 반복 실행

```csharp
using UnityEngine;
using System.Collections;

public class RepeatingCoroutineExample : MonoBehaviour
{
    private bool isRunning = true;

    private void Start()
    {
        StartCoroutine(RepeatingTask());
        StartCoroutine(IntervalTask(0.5f));
    }

    // =============================================
    // 무한 반복
    // =============================================

    private IEnumerator RepeatingTask()
    {
        int count = 0;

        while (isRunning)
        {
            count++;
            Debug.Log($"반복 {count}");

            yield return new WaitForSeconds(1f);
        }

        Debug.Log("반복 종료");
    }

    // =============================================
    // 인터벌 기반 실행
    // =============================================

    private IEnumerator IntervalTask(float interval)
    {
        WaitForSeconds wait = new WaitForSeconds(interval);

        while (isRunning)
        {
            // 상태 업데이트, 체크 등
            Debug.Log($"인터벌 체크 (Time: {Time.time:F1})");

            yield return wait; // 객체 재사용 (GC 감소)
        }
    }

    private void OnDestroy()
    {
        isRunning = false;
    }
}
```

---

## 5. Coroutine 생명주기 관리

```csharp
using UnityEngine;
using System.Collections;
using System.Collections.Generic;

public class CoroutineLifecycleExample : MonoBehaviour
{
    private List<Coroutine> activeCoroutines = new List<Coroutine>();
    private bool isDestroyed = false;

    // =============================================
    // 안전한 Coroutine 관리
    // =============================================

    /// <summary>
    /// 추적 가능한 Coroutine 시작
    /// </summary>
    private Coroutine StartTrackedCoroutine(IEnumerator routine)
    {
        var coroutine = StartCoroutine(TrackedRoutine(routine));
        activeCoroutines.Add(coroutine);
        return coroutine;
    }

    private IEnumerator TrackedRoutine(IEnumerator routine)
    {
        yield return routine;
        // 완료 후 리스트에서 제거 로직 추가 가능
    }

    // =============================================
    // 안전한 Coroutine (파괴 확인)
    // =============================================

    private IEnumerator SafeCoroutine()
    {
        while (!isDestroyed)
        {
            // 오브젝트가 파괴되지 않았는지 확인
            if (this == null) yield break;

            // 작업 수행
            Debug.Log("안전한 Coroutine 실행 중");

            yield return new WaitForSeconds(1f);
        }
    }

    // =============================================
    // OnDisable / OnDestroy 처리
    // =============================================

    private void OnDisable()
    {
        // 오브젝트가 비활성화되면 모든 Coroutine이 자동 중지됨
        Debug.Log("OnDisable - Coroutine들이 중지됨");
    }

    private void OnDestroy()
    {
        isDestroyed = true;
        StopAllCoroutines();
        activeCoroutines.Clear();
        Debug.Log("OnDestroy - 정리 완료");
    }

    // =============================================
    // Scene 전환 시 주의사항
    // =============================================

    /*
    ┌─────────────────────────────────────────────────────────────────┐
    │              Scene 전환과 Coroutine                              │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  1. Coroutine은 MonoBehaviour에 종속됨                          │
    │     └─ 오브젝트가 파괴되면 Coroutine도 중지                      │
    │                                                                  │
    │  2. DontDestroyOnLoad 오브젝트의 Coroutine                       │
    │     └─ Scene이 바뀌어도 계속 실행                                │
    │     └─ 파괴된 오브젝트 참조 주의!                                │
    │                                                                  │
    │  3. 권장 패턴                                                    │
    │     └─ Scene 전환 전 명시적으로 StopAllCoroutines() 호출        │
    │     └─ null 체크 또는 파괴 플래그 확인                           │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘
    */
}
```

---

## 6. Coroutine vs async/await 비교

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;
using System.Threading.Tasks;

public class CoroutineVsAsyncExample : MonoBehaviour
{
    private void Start()
    {
        // Coroutine 방식
        StartCoroutine(CoroutineVersion());

        // async/await 방식
        AsyncVersion();
    }

    // =============================================
    // Coroutine 버전
    // =============================================

    private IEnumerator CoroutineVersion()
    {
        Debug.Log("[Coroutine] 시작");

        // 웹 요청
        using (var request = UnityWebRequest.Get("https://httpbin.org/get"))
        {
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"[Coroutine] 성공: {request.downloadHandler.text.Length} chars");
            }
        }

        // 대기
        yield return new WaitForSeconds(1f);

        // Unity API 사용 (안전)
        transform.position = Vector3.one;

        Debug.Log("[Coroutine] 완료");
    }

    // =============================================
    // async/await 버전 (Unity 2023+ Awaitable 또는 UniTask 권장)
    // =============================================

    private async void AsyncVersion()
    {
        Debug.Log("[Async] 시작");

        // 웹 요청 (Task 기반)
        using (var client = new System.Net.Http.HttpClient())
        {
            var response = await client.GetStringAsync("https://httpbin.org/get");
            Debug.Log($"[Async] 성공: {response.Length} chars");
        }

        // 대기
        await Task.Delay(1000);

        // ✅ Unity API 사용 (UnitySynchronizationContext 덕분에 안전)
        transform.position = Vector3.one * 2;

        Debug.Log("[Async] 완료");
    }
}

/*
┌─────────────────────────────────────────────────────────────────┐
│              Coroutine vs async/await 비교                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  특성              │  Coroutine         │  async/await          │
│  ─────────────────┼────────────────────┼──────────────────────  │
│  문법              │  IEnumerator, yield │  async, await         │
│  반환값            │  불가 (out 파라미터)│  Task<T>로 가능        │
│  예외 처리         │  try-catch 제한    │  완전한 try-catch      │
│  취소              │  플래그 직접 구현   │  CancellationToken     │
│  중첩              │  yield return      │  await                 │
│  Unity API         │  안전 (메인스레드)  │  SyncContext로 안전    │
│  GC 할당          │  yield마다 발생     │  상태머신 최적화        │
│  디버깅            │  스택 추적 어려움   │  상대적으로 좋음       │
│  PlayerLoop 통합   │  네이티브 지원     │  별도 구현 필요         │
│                                                                  │
│  권장:                                                           │
│  └─ 단순한 대기, 애니메이션 → Coroutine                          │
│  └─ 복잡한 비동기 로직, 반환값 필요 → async/await (UniTask)       │
│  └─ Unity 2023+ → Awaitable 사용 고려                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
*/
```

---

## 7. 성능 최적화

```csharp
using UnityEngine;
using System.Collections;

public class CoroutineOptimizationExample : MonoBehaviour
{
    // =============================================
    // ❌ 나쁜 예: 매번 new 생성
    // =============================================

    private IEnumerator BadPerformance()
    {
        while (true)
        {
            // 매 반복마다 새 객체 생성 → GC 부담
            yield return new WaitForSeconds(0.1f);
        }
    }

    // =============================================
    // ✅ 좋은 예: 객체 재사용
    // =============================================

    private IEnumerator GoodPerformance()
    {
        // 한 번만 생성하고 재사용
        WaitForSeconds wait = new WaitForSeconds(0.1f);

        while (true)
        {
            yield return wait; // 동일 객체 재사용
        }
    }

    // =============================================
    // 캐싱 패턴
    // =============================================

    // 자주 사용하는 WaitFor 객체들을 캐싱
    private static readonly WaitForEndOfFrame WaitForEndOfFrame = new WaitForEndOfFrame();
    private static readonly WaitForFixedUpdate WaitForFixedUpdate = new WaitForFixedUpdate();

    // 시간 기반 대기는 Dictionary로 캐싱
    private static readonly System.Collections.Generic.Dictionary<float, WaitForSeconds> WaitCache
        = new System.Collections.Generic.Dictionary<float, WaitForSeconds>();

    public static WaitForSeconds GetWaitForSeconds(float seconds)
    {
        if (!WaitCache.TryGetValue(seconds, out var wait))
        {
            wait = new WaitForSeconds(seconds);
            WaitCache[seconds] = wait;
        }
        return wait;
    }

    private IEnumerator OptimizedCoroutine()
    {
        // 캐시된 객체 사용
        yield return WaitForEndOfFrame;
        yield return WaitForFixedUpdate;
        yield return GetWaitForSeconds(0.5f);
        yield return GetWaitForSeconds(0.5f); // 동일 객체 재사용
    }

    // =============================================
    // 불필요한 Coroutine 피하기
    // =============================================

    // ❌ 매 프레임 실행되는 단순 로직에 Coroutine 불필요
    private IEnumerator UnnecessaryCoroutine()
    {
        while (true)
        {
            transform.Rotate(0, 1, 0);
            yield return null;
        }
    }

    // ✅ Update 사용이 더 적절
    private void Update()
    {
        transform.Rotate(0, 1, 0);
    }
}
```

---

## 8. 실전 예제: 로딩 시스템

```csharp
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.SceneManagement;
using System.Collections;
using System.Collections.Generic;

public class LoadingSystemExample : MonoBehaviour
{
    [SerializeField] private Slider progressBar;
    [SerializeField] private Text progressText;

    private void Start()
    {
        StartCoroutine(LoadGameSequence());
    }

    private IEnumerator LoadGameSequence()
    {
        List<IEnumerator> loadingTasks = new List<IEnumerator>
        {
            LoadConfiguration(),
            LoadUserData(),
            LoadAssetBundles(),
            LoadScene("GameScene")
        };

        float totalTasks = loadingTasks.Count;
        int completedTasks = 0;

        foreach (var task in loadingTasks)
        {
            yield return task;
            completedTasks++;

            float progress = completedTasks / totalTasks;
            UpdateProgress(progress);
        }

        Debug.Log("로딩 완료!");
    }

    private void UpdateProgress(float progress)
    {
        if (progressBar != null)
            progressBar.value = progress;

        if (progressText != null)
            progressText.text = $"Loading... {progress:P0}";
    }

    private IEnumerator LoadConfiguration()
    {
        Debug.Log("설정 로딩...");
        yield return new WaitForSeconds(0.5f);
        Debug.Log("설정 로딩 완료");
    }

    private IEnumerator LoadUserData()
    {
        Debug.Log("사용자 데이터 로딩...");
        yield return new WaitForSeconds(0.7f);
        Debug.Log("사용자 데이터 로딩 완료");
    }

    private IEnumerator LoadAssetBundles()
    {
        Debug.Log("에셋번들 로딩...");
        yield return new WaitForSeconds(1f);
        Debug.Log("에셋번들 로딩 완료");
    }

    private IEnumerator LoadScene(string sceneName)
    {
        Debug.Log($"씬 로딩: {sceneName}");

        AsyncOperation asyncLoad = SceneManager.LoadSceneAsync(sceneName, LoadSceneMode.Additive);
        asyncLoad.allowSceneActivation = false;

        while (asyncLoad.progress < 0.9f)
        {
            Debug.Log($"씬 로딩 진행률: {asyncLoad.progress:P0}");
            yield return null;
        }

        Debug.Log("씬 로딩 준비 완료");
        asyncLoad.allowSceneActivation = true;

        yield return asyncLoad;
        Debug.Log($"씬 로딩 완료: {sceneName}");
    }
}
```

---

## 주의사항

1. **메인 스레드**: Coroutine은 항상 메인 스레드에서 실행됨
2. **생명주기**: MonoBehaviour가 비활성화/파괴되면 Coroutine도 중지됨
3. **GC 주의**: yield return new 객체는 매번 할당 → 캐싱 권장
4. **반환값 없음**: Coroutine은 직접적인 반환값 지원 없음
5. **예외 처리**: try-catch가 yield를 포함할 수 없음 (분리 필요)
6. **디버깅 어려움**: 스택 트레이스가 끊김

---

## 참고 자료

- [Unity: Coroutines](https://docs.unity3d.com/Manual/Coroutines.html)
- [Unity: CustomYieldInstruction](https://docs.unity3d.com/ScriptReference/CustomYieldInstruction.html)
- [Unity: WaitForSeconds](https://docs.unity3d.com/ScriptReference/WaitForSeconds.html)

---

## 다음 섹션

[08. async/await 기초](../04-async-await/08-async-await-basics.md)
