# Section 37: 플랫폼별 OS 고려사항

## 개요

Unity는 다양한 플랫폼에서 실행되며, 각 플랫폼은 비동기 프로그래밍에서 다른 특성과 제약사항을 가집니다. 이 섹션에서는 Windows, macOS, Linux, Android, iOS, WebGL 등 주요 플랫폼의 특성과 최적화 방법을 다룹니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Unity 지원 플랫폼 특성                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   플랫폼          스레딩     네트워크      파일 I/O   메모리      │
│   ─────────────────────────────────────────────────────────────  │
│   Windows/Mac    풀 지원    무제한        빠름       풍부        │
│   Linux          풀 지원    무제한        빠름       풍부        │
│   Android        제한적     백그라운드    느림       제한적      │
│   iOS            제한적     백그라운드    느림       제한적      │
│   WebGL          싱글스레드  CORS 제한    없음       제한적      │
│   콘솔           플랫폼별    플랫폼별     빠름       고정        │
│                                                                  │
│   스레드 풀 크기:                                                │
│   ┌────────────────────────────────────────────────────────┐    │
│   │ Desktop: CPU 코어 수 × 2~4                              │    │
│   │ Mobile:  CPU 코어 수 (보통 4-8)                         │    │
│   │ WebGL:   1 (싱글 스레드)                                │    │
│   └────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 플랫폼별 특성 비교

| 특성 | Windows/Mac/Linux | Android | iOS | WebGL |
|-----|------------------|---------|-----|-------|
| **멀티스레딩** | 완전 지원 | 지원 | 지원 | 미지원 |
| **백그라운드 실행** | 무제한 | 제한적 | 제한적 | 포커스 의존 |
| **파일 시스템** | 직접 접근 | 샌드박스 | 샌드박스 | 미지원 |
| **네트워크 제한** | 없음 | 백그라운드 제한 | 백그라운드 제한 | CORS |
| **메모리** | 풍부 | 제한적 | 제한적 | 제한적 |
| **Thread.Sleep** | 사용 가능 | 사용 가능 | 사용 가능 | 미지원 |
| **async/await** | 완전 지원 | 완전 지원 | 완전 지원 | 제한적 |

---

## 플랫폼 감지 및 분기

### 런타임 플랫폼 확인

```csharp
using UnityEngine;

/// <summary>
/// 플랫폼 유틸리티
/// </summary>
public static class PlatformUtils
{
    /// <summary>
    /// 현재 플랫폼이 모바일인지 확인
    /// </summary>
    public static bool IsMobile =>
        Application.platform == RuntimePlatform.Android ||
        Application.platform == RuntimePlatform.IPhonePlayer;

    /// <summary>
    /// 현재 플랫폼이 데스크톱인지 확인
    /// </summary>
    public static bool IsDesktop =>
        Application.platform == RuntimePlatform.WindowsPlayer ||
        Application.platform == RuntimePlatform.OSXPlayer ||
        Application.platform == RuntimePlatform.LinuxPlayer ||
        Application.isEditor;

    /// <summary>
    /// WebGL 플랫폼인지 확인
    /// </summary>
    public static bool IsWebGL =>
        Application.platform == RuntimePlatform.WebGLPlayer;

    /// <summary>
    /// 콘솔 플랫폼인지 확인
    /// </summary>
    public static bool IsConsole =>
        Application.platform == RuntimePlatform.PS4 ||
        Application.platform == RuntimePlatform.PS5 ||
        Application.platform == RuntimePlatform.XboxOne ||
        Application.platform == RuntimePlatform.Switch;

    /// <summary>
    /// 멀티스레딩 지원 여부
    /// </summary>
    public static bool SupportsMultithreading =>
        !IsWebGL;

    /// <summary>
    /// 백그라운드 실행 지원 여부
    /// </summary>
    public static bool SupportsBackgroundExecution =>
        IsDesktop || IsConsole;

    /// <summary>
    /// 권장 스레드 풀 크기
    /// </summary>
    public static int RecommendedThreadPoolSize
    {
        get
        {
            if (IsWebGL) return 1;
            if (IsMobile) return Mathf.Max(2, SystemInfo.processorCount);
            return SystemInfo.processorCount * 2;
        }
    }
}
```

### 컴파일 타임 플랫폼 분기

```csharp
using UnityEngine;

/// <summary>
/// 플랫폼별 조건부 컴파일 예제
/// </summary>
public class PlatformConditionalExample : MonoBehaviour
{
    private void Start()
    {
#if UNITY_EDITOR
        Debug.Log("에디터에서 실행 중");
#elif UNITY_STANDALONE_WIN
        Debug.Log("Windows 빌드");
#elif UNITY_STANDALONE_OSX
        Debug.Log("macOS 빌드");
#elif UNITY_STANDALONE_LINUX
        Debug.Log("Linux 빌드");
#elif UNITY_ANDROID
        Debug.Log("Android 빌드");
#elif UNITY_IOS
        Debug.Log("iOS 빌드");
#elif UNITY_WEBGL
        Debug.Log("WebGL 빌드");
#elif UNITY_PS4 || UNITY_PS5
        Debug.Log("PlayStation 빌드");
#elif UNITY_XBOXONE
        Debug.Log("Xbox 빌드");
#elif UNITY_SWITCH
        Debug.Log("Switch 빌드");
#else
        Debug.Log("기타 플랫폼");
#endif
    }
}
```

---

## 모바일 플랫폼 (Android/iOS)

### 백그라운드 실행 제한

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 모바일 백그라운드 실행 관리자
/// </summary>
public class MobileBackgroundManager : MonoBehaviour
{
    private CancellationTokenSource _backgroundCts;
    private bool _isInBackground;

    public event Action OnEnterBackground;
    public event Action OnEnterForeground;

    private void OnApplicationFocus(bool hasFocus)
    {
        if (hasFocus)
        {
            OnReturnToForeground();
        }
        else
        {
            OnGoToBackground();
        }
    }

    private void OnApplicationPause(bool pauseStatus)
    {
        if (pauseStatus)
        {
            OnGoToBackground();
        }
        else
        {
            OnReturnToForeground();
        }
    }

    private void OnGoToBackground()
    {
        if (_isInBackground) return;
        _isInBackground = true;

        Debug.Log("앱이 백그라운드로 전환됨");

        // 백그라운드 작업 취소
        _backgroundCts?.Cancel();

        // 네트워크 연결 일시 중지
        PauseNetworkConnections();

        // 오디오 중지
        AudioListener.pause = true;

        OnEnterBackground?.Invoke();
    }

    private void OnReturnToForeground()
    {
        if (!_isInBackground) return;
        _isInBackground = false;

        Debug.Log("앱이 포그라운드로 복귀");

        // 새 CancellationTokenSource 생성
        _backgroundCts = new CancellationTokenSource();

        // 네트워크 연결 재개
        ResumeNetworkConnections();

        // 오디오 재개
        AudioListener.pause = false;

        // 서버와 상태 동기화
        SyncWithServer();

        OnEnterForeground?.Invoke();
    }

    private void PauseNetworkConnections()
    {
        // WebSocket 연결 일시 중지
        // 진행 중인 다운로드 일시 중지
    }

    private void ResumeNetworkConnections()
    {
        // WebSocket 재연결
        // 다운로드 재개
    }

    private async void SyncWithServer()
    {
        try
        {
            // 백그라운드 동안 놓친 이벤트 동기화
            await Task.Delay(100, _backgroundCts.Token);
            // 서버에서 최신 상태 가져오기
        }
        catch (OperationCanceledException)
        {
            // 다시 백그라운드로 전환됨
        }
    }

    private void OnDestroy()
    {
        _backgroundCts?.Cancel();
        _backgroundCts?.Dispose();
    }
}
```

### 모바일 메모리 관리

```csharp
using System;
using UnityEngine;

/// <summary>
/// 모바일 메모리 관리자
/// </summary>
public class MobileMemoryManager : MonoBehaviour
{
    [SerializeField] private float memoryWarningThreshold = 0.8f;

    private long _lastMemoryCheck;
    private const long CheckIntervalMs = 5000;

    public event Action OnLowMemoryWarning;

    private void Start()
    {
        // iOS 저메모리 경고 등록
        Application.lowMemory += HandleLowMemory;
    }

    private void Update()
    {
        // 주기적 메모리 체크 (모바일)
        if (PlatformUtils.IsMobile)
        {
            long now = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
            if (now - _lastMemoryCheck > CheckIntervalMs)
            {
                _lastMemoryCheck = now;
                CheckMemoryUsage();
            }
        }
    }

    private void CheckMemoryUsage()
    {
        // 현재 메모리 사용량
        long totalMemory = GC.GetTotalMemory(false);

        // 시스템 메모리
        long systemMemory = SystemInfo.systemMemorySize * 1024 * 1024L;

        float usageRatio = (float)totalMemory / systemMemory;

        if (usageRatio > memoryWarningThreshold)
        {
            Debug.LogWarning($"메모리 사용량 높음: {usageRatio:P1}");
            HandleLowMemory();
        }
    }

    private void HandleLowMemory()
    {
        Debug.LogWarning("저메모리 경고!");

        // 캐시 정리
        ClearCaches();

        // 사용하지 않는 에셋 언로드
        Resources.UnloadUnusedAssets();

        // GC 강제 실행
        GC.Collect();

        OnLowMemoryWarning?.Invoke();
    }

    private void ClearCaches()
    {
        // 텍스처 캐시 정리
        // 오디오 캐시 정리
        // 오브젝트 풀 축소
    }

    private void OnDestroy()
    {
        Application.lowMemory -= HandleLowMemory;
    }
}
```

### 모바일 네트워크 상태

```csharp
using System;
using UnityEngine;

/// <summary>
/// 모바일 네트워크 상태 모니터
/// </summary>
public class NetworkReachabilityMonitor : MonoBehaviour
{
    private NetworkReachability _lastReachability;
    private float _checkInterval = 1f;
    private float _lastCheckTime;

    public event Action<NetworkReachability> OnReachabilityChanged;

    public NetworkReachability CurrentReachability => Application.internetReachability;

    public bool IsConnected =>
        Application.internetReachability != NetworkReachability.NotReachable;

    public bool IsOnWifi =>
        Application.internetReachability == NetworkReachability.ReachableViaLocalAreaNetwork;

    public bool IsOnMobileData =>
        Application.internetReachability == NetworkReachability.ReachableViaCarrierDataNetwork;

    private void Start()
    {
        _lastReachability = Application.internetReachability;
    }

    private void Update()
    {
        if (Time.time - _lastCheckTime > _checkInterval)
        {
            _lastCheckTime = Time.time;
            CheckReachability();
        }
    }

    private void CheckReachability()
    {
        var current = Application.internetReachability;

        if (current != _lastReachability)
        {
            Debug.Log($"네트워크 상태 변경: {_lastReachability} → {current}");
            _lastReachability = current;
            OnReachabilityChanged?.Invoke(current);
        }
    }

    /// <summary>
    /// 대용량 다운로드 허용 여부 (WiFi 전용 정책)
    /// </summary>
    public bool ShouldAllowLargeDownload(bool wifiOnly = true)
    {
        if (!IsConnected) return false;
        if (wifiOnly && !IsOnWifi) return false;
        return true;
    }
}

/// <summary>
/// 네트워크 인식 다운로드 관리자
/// </summary>
public class NetworkAwareDownloader : MonoBehaviour
{
    [SerializeField] private NetworkReachabilityMonitor networkMonitor;
    [SerializeField] private bool requireWifiForLargeFiles = true;
    [SerializeField] private long largeFileSizeThreshold = 10 * 1024 * 1024; // 10MB

    public async System.Threading.Tasks.Task<byte[]> DownloadAsync(
        string url,
        long expectedSize,
        System.Threading.CancellationToken ct)
    {
        // 대용량 파일 WiFi 확인
        if (expectedSize > largeFileSizeThreshold && requireWifiForLargeFiles)
        {
            if (!networkMonitor.IsOnWifi)
            {
                throw new InvalidOperationException(
                    "대용량 파일은 WiFi 연결에서만 다운로드할 수 있습니다.");
            }
        }

        // 네트워크 연결 확인
        if (!networkMonitor.IsConnected)
        {
            throw new InvalidOperationException("네트워크에 연결되어 있지 않습니다.");
        }

        // 다운로드 실행
        using var client = new System.Net.Http.HttpClient();
        return await client.GetByteArrayAsync(url);
    }
}
```

---

## WebGL 플랫폼

### WebGL 제한사항 및 대안

```csharp
using System;
using System.Collections;
using UnityEngine;
using UnityEngine.Networking;

/// <summary>
/// WebGL 호환 비동기 작업
/// </summary>
public class WebGLCompatibleAsync : MonoBehaviour
{
    // WebGL에서는 Thread, Task.Run 사용 불가
    // Coroutine 또는 UniTask 사용

#if UNITY_WEBGL && !UNITY_EDITOR
    /// <summary>
    /// WebGL용 HTTP 요청 (Coroutine)
    /// </summary>
    public IEnumerator GetRequestCoroutine(
        string url,
        Action<string> onSuccess,
        Action<string> onError)
    {
        using var request = UnityWebRequest.Get(url);

        yield return request.SendWebRequest();

        if (request.result == UnityWebRequest.Result.Success)
        {
            onSuccess?.Invoke(request.downloadHandler.text);
        }
        else
        {
            onError?.Invoke(request.error);
        }
    }

    /// <summary>
    /// WebGL용 딜레이 (Thread.Sleep 불가)
    /// </summary>
    public IEnumerator DelayCoroutine(float seconds, Action callback)
    {
        yield return new WaitForSeconds(seconds);
        callback?.Invoke();
    }

#else
    /// <summary>
    /// 네이티브 플랫폼용 HTTP 요청 (Task)
    /// </summary>
    public async System.Threading.Tasks.Task<string> GetRequestAsync(
        string url,
        System.Threading.CancellationToken ct = default)
    {
        using var client = new System.Net.Http.HttpClient();
        return await client.GetStringAsync(url);
    }
#endif

    /// <summary>
    /// 플랫폼 독립적인 HTTP 요청
    /// </summary>
    public void Request(string url, Action<string> onSuccess, Action<string> onError)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        StartCoroutine(GetRequestCoroutine(url, onSuccess, onError));
#else
        System.Threading.Tasks.Task.Run(async () =>
        {
            try
            {
                var result = await GetRequestAsync(url);

                // 메인 스레드로 콜백
                UnityMainThreadDispatcher.Enqueue(() => onSuccess?.Invoke(result));
            }
            catch (Exception ex)
            {
                UnityMainThreadDispatcher.Enqueue(() => onError?.Invoke(ex.Message));
            }
        });
#endif
    }
}

/// <summary>
/// 메인 스레드 디스패처
/// </summary>
public class UnityMainThreadDispatcher : MonoBehaviour
{
    private static UnityMainThreadDispatcher _instance;
    private readonly System.Collections.Concurrent.ConcurrentQueue<Action> _queue = new();

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void Initialize()
    {
        if (_instance == null)
        {
            var go = new GameObject("MainThreadDispatcher");
            _instance = go.AddComponent<UnityMainThreadDispatcher>();
            DontDestroyOnLoad(go);
        }
    }

    public static void Enqueue(Action action)
    {
        _instance?._queue.Enqueue(action);
    }

    private void Update()
    {
        while (_queue.TryDequeue(out var action))
        {
            action?.Invoke();
        }
    }
}
```

### WebGL JavaScript 인터롭

```csharp
using System.Runtime.InteropServices;
using AOT;
using UnityEngine;

/// <summary>
/// WebGL JavaScript 브릿지
/// </summary>
public class WebGLBridge : MonoBehaviour
{
#if UNITY_WEBGL && !UNITY_EDITOR
    // JavaScript 함수 선언
    [DllImport("__Internal")]
    private static extern void JS_SaveToLocalStorage(string key, string value);

    [DllImport("__Internal")]
    private static extern string JS_LoadFromLocalStorage(string key);

    [DllImport("__Internal")]
    private static extern void JS_OpenUrl(string url);

    [DllImport("__Internal")]
    private static extern bool JS_IsMobileBrowser();

    [DllImport("__Internal")]
    private static extern void JS_ShowAlert(string message);
#endif

    /// <summary>
    /// 로컬 스토리지에 저장
    /// </summary>
    public static void SaveToLocalStorage(string key, string value)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        JS_SaveToLocalStorage(key, value);
#else
        PlayerPrefs.SetString(key, value);
        PlayerPrefs.Save();
#endif
    }

    /// <summary>
    /// 로컬 스토리지에서 로드
    /// </summary>
    public static string LoadFromLocalStorage(string key)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        return JS_LoadFromLocalStorage(key);
#else
        return PlayerPrefs.GetString(key, "");
#endif
    }

    /// <summary>
    /// URL 열기
    /// </summary>
    public static void OpenUrl(string url)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        JS_OpenUrl(url);
#else
        Application.OpenURL(url);
#endif
    }
}
```

**JavaScript 플러그인 (Plugins/WebGL/Bridge.jslib)**:
```javascript
mergeInto(LibraryManager.library, {
    JS_SaveToLocalStorage: function(keyPtr, valuePtr) {
        var key = UTF8ToString(keyPtr);
        var value = UTF8ToString(valuePtr);
        localStorage.setItem(key, value);
    },

    JS_LoadFromLocalStorage: function(keyPtr) {
        var key = UTF8ToString(keyPtr);
        var value = localStorage.getItem(key) || "";
        var bufferSize = lengthBytesUTF8(value) + 1;
        var buffer = _malloc(bufferSize);
        stringToUTF8(value, buffer, bufferSize);
        return buffer;
    },

    JS_OpenUrl: function(urlPtr) {
        var url = UTF8ToString(urlPtr);
        window.open(url, '_blank');
    },

    JS_IsMobileBrowser: function() {
        return /Android|iPhone|iPad|iPod/i.test(navigator.userAgent);
    },

    JS_ShowAlert: function(messagePtr) {
        var message = UTF8ToString(messagePtr);
        alert(message);
    }
});
```

---

## 데스크톱 플랫폼

### 스레드 풀 최적화

```csharp
using System;
using System.Threading;
using UnityEngine;

/// <summary>
/// 데스크톱 스레드 풀 설정
/// </summary>
public static class DesktopThreadPoolConfig
{
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void Initialize()
    {
        if (!PlatformUtils.IsDesktop) return;

        // 현재 스레드 풀 설정 확인
        ThreadPool.GetMinThreads(out int minWorker, out int minIO);
        ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);

        Debug.Log($"스레드 풀 - Worker: {minWorker}/{maxWorker}, IO: {minIO}/{maxIO}");

        // 필요시 최소 스레드 수 조정
        int recommended = Environment.ProcessorCount * 2;
        if (minWorker < recommended)
        {
            ThreadPool.SetMinThreads(recommended, recommended);
            Debug.Log($"스레드 풀 최소값 조정: {recommended}");
        }
    }
}
```

### 파일 시스템 접근

```csharp
using System;
using System.IO;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 플랫폼별 파일 시스템 유틸리티
/// </summary>
public static class PlatformFileSystem
{
    /// <summary>
    /// 플랫폼별 저장 경로
    /// </summary>
    public static string GetSavePath(string filename)
    {
#if UNITY_EDITOR
        return Path.Combine(Application.dataPath, "..", "Saves", filename);
#elif UNITY_STANDALONE
        return Path.Combine(Application.persistentDataPath, filename);
#elif UNITY_ANDROID || UNITY_IOS
        return Path.Combine(Application.persistentDataPath, filename);
#elif UNITY_WEBGL
        // WebGL은 파일 시스템 미지원 - PlayerPrefs 또는 IndexedDB 사용
        return filename;
#else
        return Path.Combine(Application.persistentDataPath, filename);
#endif
    }

    /// <summary>
    /// 비동기 파일 쓰기
    /// </summary>
    public static async Task WriteAllBytesAsync(
        string path,
        byte[] data,
        CancellationToken ct = default)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        // WebGL: PlayerPrefs 사용
        string base64 = Convert.ToBase64String(data);
        PlayerPrefs.SetString(path, base64);
        PlayerPrefs.Save();
        await Task.CompletedTask;
#else
        // 디렉토리 생성
        string dir = Path.GetDirectoryName(path);
        if (!string.IsNullOrEmpty(dir) && !Directory.Exists(dir))
        {
            Directory.CreateDirectory(dir);
        }

        // 비동기 쓰기
        using var stream = new FileStream(
            path,
            FileMode.Create,
            FileAccess.Write,
            FileShare.None,
            bufferSize: 4096,
            useAsync: true);

        await stream.WriteAsync(data, 0, data.Length, ct);
#endif
    }

    /// <summary>
    /// 비동기 파일 읽기
    /// </summary>
    public static async Task<byte[]> ReadAllBytesAsync(
        string path,
        CancellationToken ct = default)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        // WebGL: PlayerPrefs 사용
        string base64 = PlayerPrefs.GetString(path, "");
        if (string.IsNullOrEmpty(base64))
        {
            throw new FileNotFoundException("Data not found", path);
        }
        return Convert.FromBase64String(base64);
#else
        if (!File.Exists(path))
        {
            throw new FileNotFoundException("File not found", path);
        }

        using var stream = new FileStream(
            path,
            FileMode.Open,
            FileAccess.Read,
            FileShare.Read,
            bufferSize: 4096,
            useAsync: true);

        var data = new byte[stream.Length];
        await stream.ReadAsync(data, 0, data.Length, ct);
        return data;
#endif
    }
}
```

---

## 크로스 플랫폼 비동기 패턴

### 플랫폼 추상화 레이어

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// 플랫폼 독립적인 비동기 서비스 인터페이스
/// </summary>
public interface IAsyncService
{
    Task<string> GetAsync(string url, CancellationToken ct = default);
    Task<byte[]> DownloadAsync(string url, IProgress<float> progress = null, CancellationToken ct = default);
    Task SaveAsync(string key, byte[] data, CancellationToken ct = default);
    Task<byte[]> LoadAsync(string key, CancellationToken ct = default);
}

/// <summary>
/// 데스크톱/모바일용 구현
/// </summary>
public class NativeAsyncService : IAsyncService
{
    public async Task<string> GetAsync(string url, CancellationToken ct = default)
    {
        using var client = new System.Net.Http.HttpClient();
        return await client.GetStringAsync(url);
    }

    public async Task<byte[]> DownloadAsync(
        string url,
        IProgress<float> progress = null,
        CancellationToken ct = default)
    {
        using var client = new System.Net.Http.HttpClient();
        using var response = await client.GetAsync(
            url,
            System.Net.Http.HttpCompletionOption.ResponseHeadersRead,
            ct);

        var totalBytes = response.Content.Headers.ContentLength ?? -1L;
        using var stream = await response.Content.ReadAsStreamAsync();

        var buffer = new byte[8192];
        var result = new System.IO.MemoryStream();
        long bytesRead = 0;
        int read;

        while ((read = await stream.ReadAsync(buffer, 0, buffer.Length, ct)) > 0)
        {
            await result.WriteAsync(buffer, 0, read, ct);
            bytesRead += read;

            if (totalBytes > 0)
            {
                progress?.Report((float)bytesRead / totalBytes);
            }
        }

        return result.ToArray();
    }

    public async Task SaveAsync(string key, byte[] data, CancellationToken ct = default)
    {
        string path = PlatformFileSystem.GetSavePath(key);
        await PlatformFileSystem.WriteAllBytesAsync(path, data, ct);
    }

    public async Task<byte[]> LoadAsync(string key, CancellationToken ct = default)
    {
        string path = PlatformFileSystem.GetSavePath(key);
        return await PlatformFileSystem.ReadAllBytesAsync(path, ct);
    }
}

/// <summary>
/// WebGL용 구현
/// </summary>
public class WebGLAsyncService : IAsyncService
{
    public Task<string> GetAsync(string url, CancellationToken ct = default)
    {
        var tcs = new TaskCompletionSource<string>();

        // UnityWebRequest를 Coroutine으로 처리
        CoroutineRunner.Instance.StartCoroutine(GetCoroutine(url, tcs, ct));

        return tcs.Task;
    }

    private System.Collections.IEnumerator GetCoroutine(
        string url,
        TaskCompletionSource<string> tcs,
        CancellationToken ct)
    {
        using var request = UnityEngine.Networking.UnityWebRequest.Get(url);

        var operation = request.SendWebRequest();

        while (!operation.isDone)
        {
            if (ct.IsCancellationRequested)
            {
                request.Abort();
                tcs.TrySetCanceled();
                yield break;
            }
            yield return null;
        }

        if (request.result == UnityEngine.Networking.UnityWebRequest.Result.Success)
        {
            tcs.TrySetResult(request.downloadHandler.text);
        }
        else
        {
            tcs.TrySetException(new Exception(request.error));
        }
    }

    public Task<byte[]> DownloadAsync(
        string url,
        IProgress<float> progress = null,
        CancellationToken ct = default)
    {
        var tcs = new TaskCompletionSource<byte[]>();
        CoroutineRunner.Instance.StartCoroutine(DownloadCoroutine(url, progress, tcs, ct));
        return tcs.Task;
    }

    private System.Collections.IEnumerator DownloadCoroutine(
        string url,
        IProgress<float> progress,
        TaskCompletionSource<byte[]> tcs,
        CancellationToken ct)
    {
        using var request = UnityEngine.Networking.UnityWebRequest.Get(url);

        var operation = request.SendWebRequest();

        while (!operation.isDone)
        {
            if (ct.IsCancellationRequested)
            {
                request.Abort();
                tcs.TrySetCanceled();
                yield break;
            }
            progress?.Report(operation.progress);
            yield return null;
        }

        if (request.result == UnityEngine.Networking.UnityWebRequest.Result.Success)
        {
            tcs.TrySetResult(request.downloadHandler.data);
        }
        else
        {
            tcs.TrySetException(new Exception(request.error));
        }
    }

    public Task SaveAsync(string key, byte[] data, CancellationToken ct = default)
    {
        string base64 = Convert.ToBase64String(data);
        UnityEngine.PlayerPrefs.SetString(key, base64);
        UnityEngine.PlayerPrefs.Save();
        return Task.CompletedTask;
    }

    public Task<byte[]> LoadAsync(string key, CancellationToken ct = default)
    {
        string base64 = UnityEngine.PlayerPrefs.GetString(key, "");
        if (string.IsNullOrEmpty(base64))
        {
            return Task.FromException<byte[]>(new System.IO.FileNotFoundException());
        }
        return Task.FromResult(Convert.FromBase64String(base64));
    }
}

/// <summary>
/// Coroutine 실행을 위한 싱글톤
/// </summary>
public class CoroutineRunner : UnityEngine.MonoBehaviour
{
    private static CoroutineRunner _instance;
    public static CoroutineRunner Instance
    {
        get
        {
            if (_instance == null)
            {
                var go = new UnityEngine.GameObject("CoroutineRunner");
                _instance = go.AddComponent<CoroutineRunner>();
                DontDestroyOnLoad(go);
            }
            return _instance;
        }
    }
}

/// <summary>
/// 서비스 팩토리
/// </summary>
public static class AsyncServiceFactory
{
    public static IAsyncService Create()
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        return new WebGLAsyncService();
#else
        return new NativeAsyncService();
#endif
    }
}
```

---

## 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────┐
│               플랫폼별 비동기 베스트 프랙티스                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 모바일                                                       │
│     ├── 백그라운드 전환 시 작업 취소/일시중지                      │
│     ├── 메모리 경고 대응                                          │
│     ├── 네트워크 상태 모니터링                                    │
│     └── 배터리 소모 최소화                                        │
│                                                                  │
│  2. WebGL                                                        │
│     ├── Thread, Task.Run 사용 불가 - Coroutine/UniTask 사용       │
│     ├── Thread.Sleep 불가 - yield/await 사용                     │
│     ├── 파일 시스템 미지원 - IndexedDB/PlayerPrefs 사용           │
│     └── CORS 정책 준수                                           │
│                                                                  │
│  3. 데스크톱                                                     │
│     ├── 스레드 풀 최적 활용                                       │
│     ├── 비동기 파일 I/O 사용                                      │
│     └── 시스템 리소스 효율적 사용                                 │
│                                                                  │
│  4. 크로스 플랫폼                                                 │
│     ├── 플랫폼 추상화 레이어 구현                                  │
│     ├── #if 지시문으로 플랫폼별 분기                              │
│     ├── 런타임 플랫폼 체크 활용                                   │
│     └── 공통 인터페이스 정의                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [Unity Platform Dependent Compilation](https://docs.unity3d.com/Manual/PlatformDependentCompilation.html)
- [Unity WebGL Networking](https://docs.unity3d.com/Manual/webgl-networking.html)
- [Android App Lifecycle](https://developer.android.com/guide/components/activities/activity-lifecycle)
- [iOS Background Execution](https://developer.apple.com/documentation/uikit/app_and_environment/scenes/preparing_your_ui_to_run_in_the_background)

---

## 다음 단계

- [Section 38: IL2CPP & AOT](./38-il2cpp-aot.md) - IL2CPP 빌드 최적화
- [Section 39: 메모리 & GC 최적화](../13-optimization/39-memory-gc.md)
