# 16. UniRx

## 개요

**UniRx**(Reactive Extensions for Unity)는 Unity를 위한 **Reactive Programming** 라이브러리입니다. Observable 시퀀스와 LINQ 스타일 쿼리 연산자를 사용하여 비동기 및 이벤트 기반 프로그램을 구성합니다.

---

## 설치

### Package Manager로 설치

```
// Unity Package Manager - Git URL
https://github.com/neuecc/UniRx.git?path=Assets/Plugins/UniRx/Scripts
```

### Asset Store
```
Asset Store에서 "UniRx" 검색하여 무료 설치
```

---

## 핵심 개념

### Observable과 Observer

```csharp
using UniRx;
using UnityEngine;

public class ObservableBasics : MonoBehaviour
{
    void Start()
    {
        // Observable: 데이터 스트림을 발행
        // Observer: 데이터 스트림을 구독하여 처리

        // 간단한 Observable 생성 및 구독
        Observable.Return("Hello UniRx!")
            .Subscribe(message => Debug.Log(message));

        // 여러 값 방출
        Observable.Create<int>(observer =>
        {
            observer.OnNext(1);
            observer.OnNext(2);
            observer.OnNext(3);
            observer.OnCompleted();
            return Disposable.Empty;
        })
        .Subscribe(
            value => Debug.Log($"OnNext: {value}"),
            error => Debug.LogError($"OnError: {error}"),
            () => Debug.Log("OnCompleted")
        );
    }
}
```

### Hot vs Cold Observable

```csharp
public class HotColdObservable : MonoBehaviour
{
    void Start()
    {
        // Cold Observable: 구독 시 실행 시작, 각 구독자가 독립적
        var cold = Observable.Interval(TimeSpan.FromSeconds(1));

        cold.Subscribe(x => Debug.Log($"Sub1: {x}"));
        // 2초 후 구독
        Observable.Timer(TimeSpan.FromSeconds(2))
            .Subscribe(_ => cold.Subscribe(x => Debug.Log($"Sub2: {x}")));

        // Hot Observable: 이미 실행 중, 구독자들이 같은 스트림 공유
        var hot = Observable.Interval(TimeSpan.FromSeconds(1))
            .Publish()
            .RefCount();

        hot.Subscribe(x => Debug.Log($"HotSub1: {x}"));
        Observable.Timer(TimeSpan.FromSeconds(2))
            .Subscribe(_ => hot.Subscribe(x => Debug.Log($"HotSub2: {x}")));
    }
}
```

---

## Subject 종류

### Subject<T>

```csharp
public class SubjectExample : MonoBehaviour
{
    // Subject: Observable이자 Observer
    private Subject<string> _messageSubject = new Subject<string>();

    void Start()
    {
        // 구독
        _messageSubject.Subscribe(msg => Debug.Log($"Received: {msg}"));

        // 값 발행
        _messageSubject.OnNext("Hello");
        _messageSubject.OnNext("World");
    }

    void OnDestroy()
    {
        _messageSubject.OnCompleted();
        _messageSubject.Dispose();
    }
}
```

### BehaviorSubject<T>

```csharp
public class BehaviorSubjectExample : MonoBehaviour
{
    // 최신 값을 가지고 있음, 새 구독자에게 즉시 전달
    private BehaviorSubject<int> _scoreSubject = new BehaviorSubject<int>(0);

    void Start()
    {
        // 첫 번째 구독자: 초기값 0을 즉시 받음
        _scoreSubject.Subscribe(score => Debug.Log($"Sub1 Score: {score}"));

        _scoreSubject.OnNext(100);
        _scoreSubject.OnNext(200);

        // 두 번째 구독자: 최신값 200을 즉시 받음
        _scoreSubject.Subscribe(score => Debug.Log($"Sub2 Score: {score}"));
    }
}
```

### ReplaySubject<T>

```csharp
public class ReplaySubjectExample : MonoBehaviour
{
    // 지정된 개수의 이전 값을 새 구독자에게 재생
    private ReplaySubject<string> _logSubject = new ReplaySubject<string>(bufferSize: 5);

    void Start()
    {
        _logSubject.OnNext("Log 1");
        _logSubject.OnNext("Log 2");
        _logSubject.OnNext("Log 3");

        // 새 구독자가 모든 이전 값(최대 5개)을 받음
        _logSubject.Subscribe(log => Debug.Log($"Replayed: {log}"));

        _logSubject.OnNext("Log 4");
    }
}
```

### AsyncSubject<T>

```csharp
public class AsyncSubjectExample : MonoBehaviour
{
    // 완료 시 마지막 값만 전달
    private AsyncSubject<int> _resultSubject = new AsyncSubject<int>();

    void Start()
    {
        _resultSubject.Subscribe(result => Debug.Log($"Final Result: {result}"));

        _resultSubject.OnNext(1);
        _resultSubject.OnNext(2);
        _resultSubject.OnNext(3); // 이 값만 전달됨

        _resultSubject.OnCompleted(); // 완료 시 마지막 값 전달
    }
}
```

---

## ReactiveProperty

### 기본 사용법

```csharp
using UniRx;
using UnityEngine;
using UnityEngine.UI;

public class ReactivePropertyExample : MonoBehaviour
{
    // 값 변경을 자동으로 알림
    public ReactiveProperty<int> Health = new ReactiveProperty<int>(100);
    public ReactiveProperty<string> PlayerName = new ReactiveProperty<string>("Player1");

    [SerializeField] private Text _healthText;
    [SerializeField] private Slider _healthSlider;

    void Start()
    {
        // 값 변경 구독
        Health.Subscribe(hp =>
        {
            Debug.Log($"Health changed: {hp}");
            _healthText.text = $"HP: {hp}";
            _healthSlider.value = hp / 100f;
        }).AddTo(this);

        // 조건부 구독
        Health.Where(hp => hp <= 20)
            .Subscribe(_ => Debug.Log("Low health warning!"))
            .AddTo(this);

        // 값 변경
        Health.Value = 80;
        Health.Value = 15; // "Low health warning!" 출력
    }
}
```

### ReactiveCollection

```csharp
public class ReactiveCollectionExample : MonoBehaviour
{
    private ReactiveCollection<string> _inventory = new ReactiveCollection<string>();

    void Start()
    {
        // 추가 이벤트
        _inventory.ObserveAdd()
            .Subscribe(e => Debug.Log($"Added: {e.Value} at index {e.Index}"))
            .AddTo(this);

        // 제거 이벤트
        _inventory.ObserveRemove()
            .Subscribe(e => Debug.Log($"Removed: {e.Value}"))
            .AddTo(this);

        // 개수 변경
        _inventory.ObserveCountChanged()
            .Subscribe(count => Debug.Log($"Count: {count}"))
            .AddTo(this);

        _inventory.Add("Sword");
        _inventory.Add("Shield");
        _inventory.Remove("Sword");
    }
}
```

### ReactiveDictionary

```csharp
public class ReactiveDictionaryExample : MonoBehaviour
{
    private ReactiveDictionary<string, int> _stats = new ReactiveDictionary<string, int>();

    void Start()
    {
        _stats.ObserveAdd()
            .Subscribe(e => Debug.Log($"Stat added: {e.Key} = {e.Value}"))
            .AddTo(this);

        _stats.ObserveReplace()
            .Subscribe(e => Debug.Log($"Stat changed: {e.Key} {e.OldValue} -> {e.NewValue}"))
            .AddTo(this);

        _stats["Attack"] = 10;
        _stats["Defense"] = 5;
        _stats["Attack"] = 15; // Replace 이벤트
    }
}
```

---

## Unity 통합

### Update를 Observable로

```csharp
public class UpdateAsObservable : MonoBehaviour
{
    void Start()
    {
        // Update를 Observable로 변환
        this.UpdateAsObservable()
            .Subscribe(_ =>
            {
                // 매 프레임 실행
                transform.Rotate(Vector3.up * Time.deltaTime * 30f);
            })
            .AddTo(this);

        // FixedUpdate
        this.FixedUpdateAsObservable()
            .Subscribe(_ =>
            {
                // 물리 업데이트
            })
            .AddTo(this);

        // LateUpdate
        this.LateUpdateAsObservable()
            .Subscribe(_ =>
            {
                // 카메라 따라가기 등
            })
            .AddTo(this);
    }
}
```

### Input 처리

```csharp
public class InputObservable : MonoBehaviour
{
    void Start()
    {
        // 키 입력
        Observable.EveryUpdate()
            .Where(_ => Input.GetKeyDown(KeyCode.Space))
            .Subscribe(_ => Debug.Log("Space pressed!"))
            .AddTo(this);

        // 마우스 클릭
        Observable.EveryUpdate()
            .Where(_ => Input.GetMouseButtonDown(0))
            .Select(_ => Input.mousePosition)
            .Subscribe(pos => Debug.Log($"Clicked at: {pos}"))
            .AddTo(this);

        // 축 입력
        Observable.EveryUpdate()
            .Select(_ => new Vector2(Input.GetAxis("Horizontal"), Input.GetAxis("Vertical")))
            .Where(v => v.magnitude > 0.1f)
            .Subscribe(v => Debug.Log($"Moving: {v}"))
            .AddTo(this);
    }
}
```

### UI 이벤트 바인딩

```csharp
using UnityEngine.UI;

public class UIBinding : MonoBehaviour
{
    [SerializeField] private Button _startButton;
    [SerializeField] private InputField _nameInput;
    [SerializeField] private Toggle _soundToggle;
    [SerializeField] private Slider _volumeSlider;

    void Start()
    {
        // Button 클릭
        _startButton.OnClickAsObservable()
            .Subscribe(_ => Debug.Log("Start clicked!"))
            .AddTo(this);

        // InputField 값 변경
        _nameInput.OnValueChangedAsObservable()
            .Where(text => text.Length >= 3)
            .Subscribe(text => Debug.Log($"Valid name: {text}"))
            .AddTo(this);

        // Toggle 상태
        _soundToggle.OnValueChangedAsObservable()
            .Subscribe(isOn => Debug.Log($"Sound: {isOn}"))
            .AddTo(this);

        // Slider 값
        _volumeSlider.OnValueChangedAsObservable()
            .Subscribe(value => Debug.Log($"Volume: {value}"))
            .AddTo(this);

        // 더블 클릭 감지
        _startButton.OnClickAsObservable()
            .Buffer(_startButton.OnClickAsObservable().Throttle(TimeSpan.FromMilliseconds(300)))
            .Where(clicks => clicks.Count >= 2)
            .Subscribe(_ => Debug.Log("Double clicked!"))
            .AddTo(this);
    }
}
```

### Trigger 이벤트

```csharp
public class TriggerObservable : MonoBehaviour
{
    void Start()
    {
        // OnTriggerEnter
        this.OnTriggerEnterAsObservable()
            .Where(col => col.CompareTag("Player"))
            .Subscribe(col => Debug.Log($"Player entered: {col.name}"))
            .AddTo(this);

        // OnCollisionEnter
        this.OnCollisionEnterAsObservable()
            .Subscribe(collision => Debug.Log($"Collision: {collision.gameObject.name}"))
            .AddTo(this);

        // OnMouseDown
        this.OnMouseDownAsObservable()
            .Subscribe(_ => Debug.Log("Mouse down on this object"))
            .AddTo(this);
    }
}
```

---

## 연산자

### 필터링 연산자

```csharp
public class FilterOperators : MonoBehaviour
{
    void Start()
    {
        var numbers = Observable.Range(1, 20);

        // Where: 조건 필터링
        numbers.Where(x => x % 2 == 0)
            .Subscribe(x => Debug.Log($"Even: {x}"));

        // Distinct: 중복 제거
        Observable.Return(1).Concat(Observable.Return(2))
            .Concat(Observable.Return(1))
            .Distinct()
            .Subscribe(x => Debug.Log($"Distinct: {x}"));

        // DistinctUntilChanged: 연속 중복 제거
        Observable.Return(1).Concat(Observable.Return(1))
            .Concat(Observable.Return(2))
            .Concat(Observable.Return(2))
            .DistinctUntilChanged()
            .Subscribe(x => Debug.Log($"Changed: {x}"));

        // Take: 처음 N개만
        numbers.Take(5)
            .Subscribe(x => Debug.Log($"First 5: {x}"));

        // Skip: 처음 N개 건너뛰기
        numbers.Skip(15)
            .Subscribe(x => Debug.Log($"Skip 15: {x}"));

        // First, Last
        numbers.First().Subscribe(x => Debug.Log($"First: {x}"));
        numbers.Last().Subscribe(x => Debug.Log($"Last: {x}"));
    }
}
```

### 변환 연산자

```csharp
public class TransformOperators : MonoBehaviour
{
    void Start()
    {
        // Select: 값 변환
        Observable.Range(1, 5)
            .Select(x => x * 10)
            .Subscribe(x => Debug.Log($"x10: {x}"));

        // SelectMany: 평탄화
        Observable.Range(1, 3)
            .SelectMany(x => Observable.Range(x, 3))
            .Subscribe(x => Debug.Log($"Flat: {x}"));

        // Buffer: 그룹화
        Observable.Interval(TimeSpan.FromSeconds(0.5))
            .Take(10)
            .Buffer(3)
            .Subscribe(list => Debug.Log($"Buffer: {string.Join(", ", list)}"));

        // Scan: 누적
        Observable.Range(1, 5)
            .Scan((acc, x) => acc + x)
            .Subscribe(x => Debug.Log($"Running sum: {x}"));
    }
}
```

### 결합 연산자

```csharp
public class CombineOperators : MonoBehaviour
{
    void Start()
    {
        var source1 = Observable.Interval(TimeSpan.FromSeconds(1)).Select(x => $"A{x}").Take(5);
        var source2 = Observable.Interval(TimeSpan.FromSeconds(1.5)).Select(x => $"B{x}").Take(5);

        // Merge: 두 스트림 합치기
        source1.Merge(source2)
            .Subscribe(x => Debug.Log($"Merged: {x}"))
            .AddTo(this);

        // Zip: 짝 맞추기
        Observable.Range(1, 5)
            .Zip(Observable.Return("A").Concat(Observable.Return("B"))
                .Concat(Observable.Return("C"))
                .Concat(Observable.Return("D"))
                .Concat(Observable.Return("E")),
                (num, letter) => $"{letter}{num}")
            .Subscribe(x => Debug.Log($"Zipped: {x}"));

        // CombineLatest: 최신 값 결합
        var health = new ReactiveProperty<int>(100);
        var mana = new ReactiveProperty<int>(50);

        health.CombineLatest(mana, (h, m) => $"HP: {h}, MP: {m}")
            .Subscribe(status => Debug.Log(status))
            .AddTo(this);

        health.Value = 80;
        mana.Value = 30;
    }
}
```

### 시간 연산자

```csharp
public class TimeOperators : MonoBehaviour
{
    void Start()
    {
        // Delay: 지연
        Observable.Return("Delayed message")
            .Delay(TimeSpan.FromSeconds(2))
            .Subscribe(msg => Debug.Log(msg))
            .AddTo(this);

        // Throttle: 일정 시간 동안 마지막 값만
        Observable.EveryUpdate()
            .Select(_ => Input.mousePosition)
            .Throttle(TimeSpan.FromSeconds(0.5))
            .Subscribe(pos => Debug.Log($"Throttled pos: {pos}"))
            .AddTo(this);

        // ThrottleFirst: 일정 시간 동안 첫 값만
        Observable.EveryUpdate()
            .Where(_ => Input.GetMouseButtonDown(0))
            .ThrottleFirst(TimeSpan.FromSeconds(1))
            .Subscribe(_ => Debug.Log("Click (throttled)"))
            .AddTo(this);

        // Debounce (= Throttle): 입력 멈춘 후 일정 시간 뒤
        var searchInput = new Subject<string>();
        searchInput
            .Throttle(TimeSpan.FromMilliseconds(500))
            .Subscribe(text => Debug.Log($"Search: {text}"))
            .AddTo(this);

        // Sample: 일정 간격으로 샘플링
        Observable.EveryUpdate()
            .Sample(TimeSpan.FromSeconds(1))
            .Subscribe(_ => Debug.Log($"Sampled at {Time.time}"))
            .AddTo(this);

        // Timeout: 타임아웃
        Observable.Timer(TimeSpan.FromSeconds(5))
            .Timeout(TimeSpan.FromSeconds(3))
            .Subscribe(
                _ => Debug.Log("Completed"),
                ex => Debug.Log($"Timeout: {ex.Message}"))
            .AddTo(this);
    }
}
```

---

## 에러 처리

### Catch와 Retry

```csharp
public class ErrorHandling : MonoBehaviour
{
    void Start()
    {
        // Catch: 에러 처리 후 다른 Observable로 대체
        Observable.Throw<int>(new Exception("Error!"))
            .Catch<int, Exception>(ex =>
            {
                Debug.LogWarning($"Caught: {ex.Message}");
                return Observable.Return(-1);
            })
            .Subscribe(x => Debug.Log($"Value: {x}"))
            .AddTo(this);

        // Retry: 에러 시 재시도
        var retryCount = 0;
        Observable.Create<int>(observer =>
        {
            retryCount++;
            if (retryCount < 3)
            {
                observer.OnError(new Exception($"Attempt {retryCount} failed"));
            }
            else
            {
                observer.OnNext(42);
                observer.OnCompleted();
            }
            return Disposable.Empty;
        })
        .Retry(3)
        .Subscribe(
            x => Debug.Log($"Success: {x}"),
            ex => Debug.LogError($"Failed: {ex.Message}"))
        .AddTo(this);

        // OnErrorRetry: 지연을 두고 재시도
        var attemptCount = 0;
        Observable.Create<string>(observer =>
        {
            attemptCount++;
            Debug.Log($"Attempt {attemptCount}");
            observer.OnError(new Exception("Network error"));
            return Disposable.Empty;
        })
        .OnErrorRetry((Exception ex) =>
        {
            Debug.Log($"Error: {ex.Message}, retrying...");
        }, retryCount: 3, delay: TimeSpan.FromSeconds(1))
        .Subscribe(
            x => Debug.Log($"Result: {x}"),
            ex => Debug.Log("All retries failed"))
        .AddTo(this);
    }
}
```

### Finally와 DoOnError

```csharp
public class FinallyExample : MonoBehaviour
{
    void Start()
    {
        Observable.Range(1, 5)
            .Do(x => Debug.Log($"Processing: {x}"))
            .DoOnError(ex => Debug.LogError($"Error occurred: {ex}"))
            .DoOnCompleted(() => Debug.Log("Stream completed"))
            .Finally(() => Debug.Log("Finally - cleanup"))
            .Subscribe(x => Debug.Log($"Received: {x}"))
            .AddTo(this);
    }
}
```

---

## 리소스 관리

### AddTo로 자동 구독 해제

```csharp
public class ResourceManagement : MonoBehaviour
{
    void Start()
    {
        // AddTo(this): GameObject 파괴 시 자동 구독 해제
        Observable.Interval(TimeSpan.FromSeconds(1))
            .Subscribe(x => Debug.Log($"Tick: {x}"))
            .AddTo(this);

        // AddTo(gameObject)도 동일
        Observable.EveryUpdate()
            .Subscribe(_ => { })
            .AddTo(gameObject);
    }
}
```

### CompositeDisposable

```csharp
public class CompositeDisposableExample : MonoBehaviour
{
    private CompositeDisposable _disposables = new CompositeDisposable();

    void Start()
    {
        Observable.Interval(TimeSpan.FromSeconds(1))
            .Subscribe(x => Debug.Log($"Stream1: {x}"))
            .AddTo(_disposables);

        Observable.Interval(TimeSpan.FromSeconds(2))
            .Subscribe(x => Debug.Log($"Stream2: {x}"))
            .AddTo(_disposables);
    }

    void OnDestroy()
    {
        // 모든 구독 한번에 해제
        _disposables.Dispose();
    }

    public void ClearSubscriptions()
    {
        _disposables.Clear(); // 기존 구독 해제, 새 구독 가능
    }
}
```

### SerialDisposable

```csharp
public class SerialDisposableExample : MonoBehaviour
{
    private SerialDisposable _currentSubscription = new SerialDisposable();

    void Start()
    {
        StartNewStream(1);
    }

    void StartNewStream(int id)
    {
        // 이전 구독 자동 해제, 새 구독으로 교체
        _currentSubscription.Disposable = Observable.Interval(TimeSpan.FromSeconds(1))
            .Subscribe(x => Debug.Log($"Stream {id}: {x}"));
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.Space))
        {
            StartNewStream(Random.Range(1, 100));
        }
    }

    void OnDestroy()
    {
        _currentSubscription.Dispose();
    }
}
```

---

## 실전 패턴

### 패턴 1: MVP 아키텍처

```csharp
// Model
public class PlayerModel
{
    public ReactiveProperty<int> Health = new ReactiveProperty<int>(100);
    public ReactiveProperty<int> MaxHealth = new ReactiveProperty<int>(100);
    public ReactiveProperty<int> Score = new ReactiveProperty<int>(0);

    public void TakeDamage(int damage)
    {
        Health.Value = Mathf.Max(0, Health.Value - damage);
    }

    public void AddScore(int points)
    {
        Score.Value += points;
    }
}

// View
public class PlayerView : MonoBehaviour
{
    [SerializeField] private Slider _healthBar;
    [SerializeField] private Text _healthText;
    [SerializeField] private Text _scoreText;

    public void UpdateHealth(int current, int max)
    {
        _healthBar.value = (float)current / max;
        _healthText.text = $"{current}/{max}";
    }

    public void UpdateScore(int score)
    {
        _scoreText.text = $"Score: {score}";
    }
}

// Presenter
public class PlayerPresenter : MonoBehaviour
{
    [SerializeField] private PlayerView _view;
    private PlayerModel _model;

    void Start()
    {
        _model = new PlayerModel();

        // Model -> View 바인딩
        _model.Health
            .CombineLatest(_model.MaxHealth, (current, max) => (current, max))
            .Subscribe(tuple => _view.UpdateHealth(tuple.current, tuple.max))
            .AddTo(this);

        _model.Score
            .Subscribe(score => _view.UpdateScore(score))
            .AddTo(this);
    }

    public void OnDamageButtonClick()
    {
        _model.TakeDamage(10);
    }

    public void OnScoreButtonClick()
    {
        _model.AddScore(100);
    }
}
```

### 패턴 2: 검색 자동완성

```csharp
public class SearchAutocomplete : MonoBehaviour
{
    [SerializeField] private InputField _searchInput;
    [SerializeField] private Transform _resultsContainer;
    [SerializeField] private GameObject _resultItemPrefab;

    void Start()
    {
        _searchInput.OnValueChangedAsObservable()
            .Where(text => text.Length >= 2) // 최소 2글자
            .Throttle(TimeSpan.FromMilliseconds(500)) // 입력 멈춤 대기
            .DistinctUntilChanged() // 같은 검색어 무시
            .SelectMany(query => SearchAsync(query).ToObservable()) // API 호출
            .ObserveOnMainThread() // Main Thread로
            .Subscribe(results => DisplayResults(results))
            .AddTo(this);
    }

    async Task<List<string>> SearchAsync(string query)
    {
        // API 호출 시뮬레이션
        await Task.Delay(200);
        return new List<string> { $"{query} Result 1", $"{query} Result 2" };
    }

    void DisplayResults(List<string> results)
    {
        // 기존 결과 제거
        foreach (Transform child in _resultsContainer)
        {
            Destroy(child.gameObject);
        }

        // 새 결과 표시
        foreach (var result in results)
        {
            var item = Instantiate(_resultItemPrefab, _resultsContainer);
            item.GetComponentInChildren<Text>().text = result;
        }
    }
}
```

### 패턴 3: 드래그 앤 드롭

```csharp
public class DragAndDrop : MonoBehaviour
{
    void Start()
    {
        var mouseDown = this.OnMouseDownAsObservable();
        var mouseUp = Observable.EveryUpdate().Where(_ => Input.GetMouseButtonUp(0));

        mouseDown
            .SelectMany(_ => Observable.EveryUpdate()
                .TakeUntil(mouseUp)
                .Select(_ => GetMouseWorldPosition()))
            .Subscribe(pos =>
            {
                transform.position = pos;
            })
            .AddTo(this);
    }

    Vector3 GetMouseWorldPosition()
    {
        var mousePos = Input.mousePosition;
        mousePos.z = 10f;
        return Camera.main.ScreenToWorldPoint(mousePos);
    }
}
```

### 패턴 4: 콤보 시스템

```csharp
public class ComboSystem : MonoBehaviour
{
    public ReactiveProperty<int> ComboCount = new ReactiveProperty<int>(0);

    [SerializeField] private float _comboTimeout = 2f;

    private Subject<Unit> _hitSubject = new Subject<Unit>();

    void Start()
    {
        // 히트 시 콤보 증가, 타임아웃 시 리셋
        _hitSubject
            .Do(_ => ComboCount.Value++)
            .Throttle(TimeSpan.FromSeconds(_comboTimeout))
            .Subscribe(_ => ComboCount.Value = 0)
            .AddTo(this);

        // 콤보 표시
        ComboCount
            .Where(count => count > 0)
            .Subscribe(count => Debug.Log($"Combo: {count}x"))
            .AddTo(this);
    }

    public void OnHit()
    {
        _hitSubject.OnNext(Unit.Default);
    }

    void OnDestroy()
    {
        _hitSubject.Dispose();
    }
}
```

---

## 주의사항

### 구독 해제 잊지 않기

```csharp
public class SubscriptionLeakPrevention : MonoBehaviour
{
    // ❌ 나쁨: 구독 해제 없음 (메모리 누수)
    void Bad()
    {
        Observable.Interval(TimeSpan.FromSeconds(1))
            .Subscribe(x => Debug.Log(x));
    }

    // ✅ 좋음: AddTo 사용
    void Good()
    {
        Observable.Interval(TimeSpan.FromSeconds(1))
            .Subscribe(x => Debug.Log(x))
            .AddTo(this);
    }
}
```

### Hot Observable 주의

```csharp
public class HotObservableCaution : MonoBehaviour
{
    void Start()
    {
        // ⚠️ Publish().RefCount()는 마지막 구독자 해제 시 스트림 종료
        var shared = Observable.Interval(TimeSpan.FromSeconds(1))
            .Publish()
            .RefCount();

        var sub1 = shared.Subscribe(x => Debug.Log($"Sub1: {x}"));
        var sub2 = shared.Subscribe(x => Debug.Log($"Sub2: {x}"));

        // 모든 구독 해제 시 스트림 종료
        Observable.Timer(TimeSpan.FromSeconds(5))
            .Subscribe(_ =>
            {
                sub1.Dispose();
                sub2.Dispose();
                // 새 구독 시 스트림 재시작
            })
            .AddTo(this);
    }
}
```

---

## 정리

### UniRx 요약

| 항목 | 내용 |
|------|------|
| **핵심** | Reactive Extensions for Unity |
| **주요 개념** | Observable, Observer, Subject |
| **ReactiveProperty** | 값 변경 자동 알림 |
| **Unity 통합** | Update, Input, UI, Trigger 등 |
| **리소스 관리** | AddTo, CompositeDisposable |

### 선택 가이드

| 상황 | 권장 |
|------|------|
| 이벤트 스트림 처리 | Observable + 연산자 |
| 상태 관리 | ReactiveProperty |
| 컬렉션 변경 감지 | ReactiveCollection |
| 복잡한 UI 바인딩 | CombineLatest, Zip |
| 입력 처리 | Throttle, DistinctUntilChanged |

---

## 참고 자료

- [UniRx GitHub](https://github.com/neuecc/UniRx)
- [ReactiveX Documentation](http://reactivex.io/documentation)
- [UniRx Operators](https://github.com/neuecc/UniRx#operators)
- [Introduction to Rx](http://introtorx.com/)
