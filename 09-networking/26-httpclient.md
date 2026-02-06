# Section 26: HttpClient

## 개요

`HttpClient`는 .NET의 표준 HTTP 클라이언트 라이브러리로, Unity에서도 사용 가능합니다. `UnityWebRequest`와 달리 .NET Standard API를 그대로 사용할 수 있어 서버-클라이언트 코드 공유에 유리합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    HttpClient Architecture                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│   │  HttpClient  │───▶│ HttpHandler  │───▶│   Network    │     │
│   │  (Singleton) │    │   Pipeline   │    │    Layer     │     │
│   └──────────────┘    └──────────────┘    └──────────────┘     │
│          │                    │                                  │
│          ▼                    ▼                                  │
│   ┌──────────────┐    ┌──────────────┐                          │
│   │ HttpRequest  │    │  Delegating  │                          │
│   │   Message    │    │   Handlers   │                          │
│   └──────────────┘    └──────────────┘                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## HttpClient vs UnityWebRequest 비교

| 특성 | HttpClient | UnityWebRequest |
|------|-----------|-----------------|
| **네임스페이스** | System.Net.Http | UnityEngine.Networking |
| **async/await** | 네이티브 지원 | Coroutine 또는 AsyncOperation 변환 필요 |
| **인스턴스 관리** | Singleton 권장 | 매 요청마다 생성/해제 |
| **스레드** | 백그라운드 스레드 | 메인 스레드 |
| **플랫폼 지원** | WebGL 미지원 | 모든 Unity 플랫폼 |
| **코드 공유** | 서버와 공유 가능 | Unity 전용 |
| **인증서 처리** | 표준 .NET 방식 | Unity 전용 핸들러 |
| **스트리밍** | 우수 | 제한적 |
| **메모리 제어** | 세밀한 제어 가능 | 제한적 |

---

## 기본 사용법

### HttpClient 인스턴스 관리 (싱글톤 패턴)

```csharp
using System;
using System.Net.Http;
using System.Net.Http.Headers;
using UnityEngine;

/// <summary>
/// HttpClient 싱글톤 관리자
/// HttpClient는 재사용이 권장되므로 싱글톤으로 관리
/// </summary>
public static class HttpClientProvider
{
    private static HttpClient _client;
    private static readonly object _lock = new object();

    /// <summary>
    /// 기본 타임아웃 (30초)
    /// </summary>
    public static TimeSpan DefaultTimeout { get; set; } = TimeSpan.FromSeconds(30);

    /// <summary>
    /// 기본 Base Address
    /// </summary>
    public static Uri BaseAddress { get; set; }

    /// <summary>
    /// HttpClient 인스턴스 반환
    /// 스레드 안전하게 초기화
    /// </summary>
    public static HttpClient Client
    {
        get
        {
            if (_client == null)
            {
                lock (_lock)
                {
                    if (_client == null)
                    {
                        _client = CreateClient();
                    }
                }
            }
            return _client;
        }
    }

    private static HttpClient CreateClient()
    {
        var handler = new HttpClientHandler
        {
            // 자동 압축 해제
            AutomaticDecompression = System.Net.DecompressionMethods.GZip
                                   | System.Net.DecompressionMethods.Deflate,
            // 쿠키 자동 처리
            UseCookies = true,
            // 리다이렉트 자동 처리
            AllowAutoRedirect = true,
            MaxAutomaticRedirections = 5
        };

        var client = new HttpClient(handler)
        {
            Timeout = DefaultTimeout
        };

        if (BaseAddress != null)
        {
            client.BaseAddress = BaseAddress;
        }

        // 기본 헤더 설정
        client.DefaultRequestHeaders.Accept.Add(
            new MediaTypeWithQualityHeaderValue("application/json"));
        client.DefaultRequestHeaders.AcceptEncoding.Add(
            new StringWithQualityHeaderValue("gzip"));
        client.DefaultRequestHeaders.UserAgent.ParseAdd(
            $"Unity/{Application.unityVersion} ({Application.platform})");

        return client;
    }

    /// <summary>
    /// 인증 토큰 설정
    /// </summary>
    public static void SetBearerToken(string token)
    {
        Client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token);
    }

    /// <summary>
    /// 인증 토큰 제거
    /// </summary>
    public static void ClearBearerToken()
    {
        Client.DefaultRequestHeaders.Authorization = null;
    }

    /// <summary>
    /// 앱 종료 시 정리
    /// RuntimeInitializeOnLoadMethod로 자동 등록
    /// </summary>
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    private static void Initialize()
    {
        Application.quitting += () =>
        {
            _client?.Dispose();
            _client = null;
        };
    }
}
```

### 기본 GET 요청

```csharp
using System;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

public class BasicHttpClientExample : MonoBehaviour
{
    private CancellationTokenSource _cts;

    private void OnEnable()
    {
        _cts = new CancellationTokenSource();
    }

    private void OnDisable()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }

    /// <summary>
    /// 간단한 GET 요청
    /// </summary>
    public async Task<string> GetAsync(string url, CancellationToken ct = default)
    {
        try
        {
            // HttpClient는 싱글톤 사용
            var response = await HttpClientProvider.Client.GetAsync(url, ct);

            // 상태 코드 확인
            response.EnsureSuccessStatusCode();

            // 응답 본문 읽기
            string content = await response.Content.ReadAsStringAsync();

            return content;
        }
        catch (HttpRequestException ex)
        {
            Debug.LogError($"HTTP 요청 실패: {ex.Message}");
            throw;
        }
        catch (TaskCanceledException) when (ct.IsCancellationRequested)
        {
            Debug.Log("요청이 취소되었습니다.");
            throw;
        }
        catch (TaskCanceledException)
        {
            // 타임아웃
            Debug.LogError("요청 시간이 초과되었습니다.");
            throw;
        }
    }

    /// <summary>
    /// JSON 데이터 GET
    /// </summary>
    public async Task<T> GetJsonAsync<T>(string url, CancellationToken ct = default)
    {
        string json = await GetAsync(url, ct);
        return JsonUtility.FromJson<T>(json);
    }

    /// <summary>
    /// 바이트 배열 다운로드
    /// </summary>
    public async Task<byte[]> GetBytesAsync(string url, CancellationToken ct = default)
    {
        var response = await HttpClientProvider.Client.GetAsync(url, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsByteArrayAsync();
    }

    /// <summary>
    /// 스트림으로 다운로드 (대용량 파일)
    /// </summary>
    public async Task<byte[]> GetStreamAsync(string url,
        IProgress<float> progress = null,
        CancellationToken ct = default)
    {
        using var response = await HttpClientProvider.Client.GetAsync(
            url,
            HttpCompletionOption.ResponseHeadersRead,
            ct);

        response.EnsureSuccessStatusCode();

        var totalBytes = response.Content.Headers.ContentLength ?? -1L;

        using var stream = await response.Content.ReadAsStreamAsync();
        using var memoryStream = new System.IO.MemoryStream();

        var buffer = new byte[8192];
        var bytesRead = 0L;
        int read;

        while ((read = await stream.ReadAsync(buffer, 0, buffer.Length, ct)) > 0)
        {
            await memoryStream.WriteAsync(buffer, 0, read, ct);
            bytesRead += read;

            if (totalBytes > 0)
            {
                progress?.Report((float)bytesRead / totalBytes);
            }
        }

        return memoryStream.ToArray();
    }

    // 사용 예시
    private async void Start()
    {
        try
        {
            string data = await GetAsync("https://api.example.com/data", _cts.Token);
            Debug.Log($"받은 데이터: {data}");
        }
        catch (Exception ex)
        {
            Debug.LogError($"오류 발생: {ex.Message}");
        }
    }
}
```

### POST 요청

```csharp
using System;
using System.Net.Http;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

public class HttpPostExample : MonoBehaviour
{
    [Serializable]
    public class LoginRequest
    {
        public string username;
        public string password;
    }

    [Serializable]
    public class LoginResponse
    {
        public string token;
        public string userId;
        public long expiresAt;
    }

    /// <summary>
    /// JSON POST 요청
    /// </summary>
    public async Task<TResponse> PostJsonAsync<TRequest, TResponse>(
        string url,
        TRequest data,
        CancellationToken ct = default)
    {
        // 객체를 JSON으로 직렬화
        string json = JsonUtility.ToJson(data);

        using var content = new StringContent(
            json,
            Encoding.UTF8,
            "application/json");

        var response = await HttpClientProvider.Client.PostAsync(url, content, ct);
        response.EnsureSuccessStatusCode();

        string responseJson = await response.Content.ReadAsStringAsync();
        return JsonUtility.FromJson<TResponse>(responseJson);
    }

    /// <summary>
    /// Form 데이터 POST
    /// </summary>
    public async Task<string> PostFormAsync(
        string url,
        System.Collections.Generic.Dictionary<string, string> formData,
        CancellationToken ct = default)
    {
        using var content = new FormUrlEncodedContent(formData);

        var response = await HttpClientProvider.Client.PostAsync(url, content, ct);
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadAsStringAsync();
    }

    /// <summary>
    /// 멀티파트 폼 데이터 (파일 업로드)
    /// </summary>
    public async Task<string> UploadFileAsync(
        string url,
        byte[] fileData,
        string fileName,
        string fieldName = "file",
        CancellationToken ct = default)
    {
        using var content = new MultipartFormDataContent();
        using var fileContent = new ByteArrayContent(fileData);

        // Content-Type 설정
        fileContent.Headers.ContentType =
            new System.Net.Http.Headers.MediaTypeHeaderValue("application/octet-stream");

        content.Add(fileContent, fieldName, fileName);

        var response = await HttpClientProvider.Client.PostAsync(url, content, ct);
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadAsStringAsync();
    }

    // 사용 예시
    private async void Start()
    {
        var cts = new CancellationTokenSource();

        try
        {
            // JSON POST
            var loginRequest = new LoginRequest
            {
                username = "player1",
                password = "securepassword"
            };

            var loginResponse = await PostJsonAsync<LoginRequest, LoginResponse>(
                "https://api.example.com/auth/login",
                loginRequest,
                cts.Token);

            Debug.Log($"로그인 성공! 토큰: {loginResponse.token}");

            // 인증 토큰 저장
            HttpClientProvider.SetBearerToken(loginResponse.token);
        }
        catch (Exception ex)
        {
            Debug.LogError($"로그인 실패: {ex.Message}");
        }
    }

    private void OnDestroy()
    {
        // 로그아웃 시 토큰 제거
        HttpClientProvider.ClearBearerToken();
    }
}
```

---

## 고급 패턴

### 재시도 로직이 포함된 HTTP 클라이언트

```csharp
using System;
using System.Net;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 재시도 로직이 포함된 HTTP 서비스
/// </summary>
public class ResilientHttpService
{
    private readonly HttpClient _client;
    private readonly RetryPolicy _retryPolicy;

    public ResilientHttpService(HttpClient client, RetryPolicy retryPolicy = null)
    {
        _client = client;
        _retryPolicy = retryPolicy ?? RetryPolicy.Default;
    }

    /// <summary>
    /// 재시도가 적용된 GET 요청
    /// </summary>
    public async Task<HttpResponseMessage> GetWithRetryAsync(
        string url,
        CancellationToken ct = default)
    {
        return await ExecuteWithRetryAsync(
            () => _client.GetAsync(url, ct),
            ct);
    }

    /// <summary>
    /// 재시도가 적용된 POST 요청
    /// </summary>
    public async Task<HttpResponseMessage> PostWithRetryAsync(
        string url,
        HttpContent content,
        CancellationToken ct = default)
    {
        return await ExecuteWithRetryAsync(
            () => _client.PostAsync(url, content, ct),
            ct);
    }

    /// <summary>
    /// 재시도 로직 실행
    /// </summary>
    private async Task<HttpResponseMessage> ExecuteWithRetryAsync(
        Func<Task<HttpResponseMessage>> operation,
        CancellationToken ct)
    {
        Exception lastException = null;

        for (int attempt = 0; attempt <= _retryPolicy.MaxRetries; attempt++)
        {
            try
            {
                var response = await operation();

                // 재시도 가능한 상태 코드 확인
                if (IsRetryableStatusCode(response.StatusCode) &&
                    attempt < _retryPolicy.MaxRetries)
                {
                    Debug.LogWarning($"재시도 가능한 상태 코드: {response.StatusCode}, " +
                                   $"시도 {attempt + 1}/{_retryPolicy.MaxRetries + 1}");

                    await WaitBeforeRetry(attempt, ct);
                    continue;
                }

                return response;
            }
            catch (HttpRequestException ex)
            {
                lastException = ex;
                Debug.LogWarning($"HTTP 요청 실패: {ex.Message}, " +
                               $"시도 {attempt + 1}/{_retryPolicy.MaxRetries + 1}");

                if (attempt < _retryPolicy.MaxRetries)
                {
                    await WaitBeforeRetry(attempt, ct);
                }
            }
            catch (TaskCanceledException) when (!ct.IsCancellationRequested)
            {
                // 타임아웃
                lastException = new TimeoutException("요청 시간 초과");
                Debug.LogWarning($"요청 시간 초과, 시도 {attempt + 1}/{_retryPolicy.MaxRetries + 1}");

                if (attempt < _retryPolicy.MaxRetries)
                {
                    await WaitBeforeRetry(attempt, ct);
                }
            }
        }

        throw new HttpRequestException(
            $"모든 재시도 실패 ({_retryPolicy.MaxRetries + 1}회 시도)",
            lastException);
    }

    private bool IsRetryableStatusCode(HttpStatusCode statusCode)
    {
        return statusCode switch
        {
            HttpStatusCode.RequestTimeout => true,        // 408
            HttpStatusCode.TooManyRequests => true,       // 429
            HttpStatusCode.InternalServerError => true,   // 500
            HttpStatusCode.BadGateway => true,            // 502
            HttpStatusCode.ServiceUnavailable => true,    // 503
            HttpStatusCode.GatewayTimeout => true,        // 504
            _ => false
        };
    }

    private async Task WaitBeforeRetry(int attempt, CancellationToken ct)
    {
        // 지수 백오프 + 지터
        var baseDelay = _retryPolicy.BaseDelay;
        var maxDelay = _retryPolicy.MaxDelay;

        var delay = TimeSpan.FromMilliseconds(
            Math.Min(
                baseDelay.TotalMilliseconds * Math.Pow(2, attempt),
                maxDelay.TotalMilliseconds));

        // 지터 추가 (0-25% 랜덤 추가)
        var jitter = TimeSpan.FromMilliseconds(
            delay.TotalMilliseconds * UnityEngine.Random.Range(0f, 0.25f));

        await Task.Delay(delay + jitter, ct);
    }
}

/// <summary>
/// 재시도 정책 설정
/// </summary>
public class RetryPolicy
{
    public int MaxRetries { get; set; } = 3;
    public TimeSpan BaseDelay { get; set; } = TimeSpan.FromSeconds(1);
    public TimeSpan MaxDelay { get; set; } = TimeSpan.FromSeconds(30);

    public static RetryPolicy Default => new RetryPolicy();

    public static RetryPolicy Aggressive => new RetryPolicy
    {
        MaxRetries = 5,
        BaseDelay = TimeSpan.FromMilliseconds(500),
        MaxDelay = TimeSpan.FromSeconds(60)
    };

    public static RetryPolicy Conservative => new RetryPolicy
    {
        MaxRetries = 2,
        BaseDelay = TimeSpan.FromSeconds(2),
        MaxDelay = TimeSpan.FromSeconds(10)
    };
}
```

### DelegatingHandler를 활용한 파이프라인

```csharp
using System;
using System.Diagnostics;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;
using Debug = UnityEngine.Debug;

/// <summary>
/// 로깅을 위한 DelegatingHandler
/// </summary>
public class LoggingHandler : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        var stopwatch = Stopwatch.StartNew();

        Debug.Log($"[HTTP] {request.Method} {request.RequestUri}");

        try
        {
            var response = await base.SendAsync(request, cancellationToken);

            stopwatch.Stop();
            Debug.Log($"[HTTP] {response.StatusCode} ({stopwatch.ElapsedMilliseconds}ms)");

            return response;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            Debug.LogError($"[HTTP] 오류: {ex.Message} ({stopwatch.ElapsedMilliseconds}ms)");
            throw;
        }
    }
}

/// <summary>
/// 인증 토큰 자동 갱신 Handler
/// </summary>
public class AuthenticationHandler : DelegatingHandler
{
    private readonly ITokenProvider _tokenProvider;
    private readonly SemaphoreSlim _refreshLock = new SemaphoreSlim(1, 1);

    public AuthenticationHandler(ITokenProvider tokenProvider)
    {
        _tokenProvider = tokenProvider;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        // 현재 토큰 가져오기
        var token = await _tokenProvider.GetTokenAsync();

        if (!string.IsNullOrEmpty(token))
        {
            request.Headers.Authorization =
                new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
        }

        var response = await base.SendAsync(request, cancellationToken);

        // 401 Unauthorized인 경우 토큰 갱신 시도
        if (response.StatusCode == System.Net.HttpStatusCode.Unauthorized)
        {
            await _refreshLock.WaitAsync(cancellationToken);
            try
            {
                // 토큰 갱신
                if (await _tokenProvider.RefreshTokenAsync())
                {
                    // 새 토큰으로 재시도
                    token = await _tokenProvider.GetTokenAsync();
                    request.Headers.Authorization =
                        new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);

                    response = await base.SendAsync(request, cancellationToken);
                }
            }
            finally
            {
                _refreshLock.Release();
            }
        }

        return response;
    }
}

/// <summary>
/// 토큰 제공자 인터페이스
/// </summary>
public interface ITokenProvider
{
    Task<string> GetTokenAsync();
    Task<bool> RefreshTokenAsync();
}

/// <summary>
/// Handler 파이프라인으로 HttpClient 생성
/// </summary>
public static class HttpClientFactory
{
    public static HttpClient CreateWithPipeline(ITokenProvider tokenProvider)
    {
        var innerHandler = new HttpClientHandler
        {
            AutomaticDecompression = System.Net.DecompressionMethods.GZip
        };

        // 핸들러 체인 구성 (역순으로 실행됨)
        var loggingHandler = new LoggingHandler
        {
            InnerHandler = innerHandler
        };

        var authHandler = new AuthenticationHandler(tokenProvider)
        {
            InnerHandler = loggingHandler
        };

        return new HttpClient(authHandler);
    }
}
```

### 요청 풀링 및 동시성 제어

```csharp
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 동시 요청 수를 제한하는 HTTP 클라이언트 래퍼
/// </summary>
public class ThrottledHttpClient
{
    private readonly HttpClient _client;
    private readonly SemaphoreSlim _semaphore;
    private readonly int _maxConcurrentRequests;

    public ThrottledHttpClient(HttpClient client, int maxConcurrentRequests = 4)
    {
        _client = client;
        _maxConcurrentRequests = maxConcurrentRequests;
        _semaphore = new SemaphoreSlim(maxConcurrentRequests, maxConcurrentRequests);
    }

    /// <summary>
    /// 동시성이 제한된 GET 요청
    /// </summary>
    public async Task<HttpResponseMessage> GetAsync(
        string url,
        CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            return await _client.GetAsync(url, ct);
        }
        finally
        {
            _semaphore.Release();
        }
    }

    /// <summary>
    /// 여러 URL을 병렬로 요청 (동시성 제한 적용)
    /// </summary>
    public async Task<List<HttpResponseMessage>> GetAllAsync(
        IEnumerable<string> urls,
        CancellationToken ct = default)
    {
        var tasks = new List<Task<HttpResponseMessage>>();

        foreach (var url in urls)
        {
            tasks.Add(GetAsync(url, ct));
        }

        var responses = await Task.WhenAll(tasks);
        return new List<HttpResponseMessage>(responses);
    }

    /// <summary>
    /// 현재 대기 중인 요청 수
    /// </summary>
    public int PendingRequests => _maxConcurrentRequests - _semaphore.CurrentCount;
}

/// <summary>
/// 사용 예시
/// </summary>
public class ThrottledClientExample : MonoBehaviour
{
    private ThrottledHttpClient _throttledClient;

    private void Start()
    {
        // 최대 4개의 동시 요청만 허용
        _throttledClient = new ThrottledHttpClient(
            HttpClientProvider.Client,
            maxConcurrentRequests: 4);
    }

    public async Task DownloadMultipleAsync()
    {
        var urls = new[]
        {
            "https://api.example.com/data/1",
            "https://api.example.com/data/2",
            "https://api.example.com/data/3",
            "https://api.example.com/data/4",
            "https://api.example.com/data/5",
            "https://api.example.com/data/6",
            "https://api.example.com/data/7",
            "https://api.example.com/data/8"
        };

        // 8개 URL이지만 동시에 4개씩만 요청
        var responses = await _throttledClient.GetAllAsync(urls);

        Debug.Log($"모든 요청 완료: {responses.Count}개");
    }
}
```

---

## UniTask와 통합

```csharp
using System;
using System.Net.Http;
using System.Text;
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine;

/// <summary>
/// UniTask와 HttpClient 통합 서비스
/// </summary>
public class UniTaskHttpService
{
    private readonly HttpClient _client;

    public UniTaskHttpService(HttpClient client = null)
    {
        _client = client ?? HttpClientProvider.Client;
    }

    /// <summary>
    /// UniTask 기반 GET 요청
    /// </summary>
    public async UniTask<string> GetStringAsync(
        string url,
        CancellationToken ct = default)
    {
        // HttpClient 작업을 UniTask로 래핑
        var response = await _client.GetAsync(url, ct).AsUniTask();
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadAsStringAsync().AsUniTask();
    }

    /// <summary>
    /// JSON GET 및 역직렬화
    /// </summary>
    public async UniTask<T> GetJsonAsync<T>(
        string url,
        CancellationToken ct = default)
    {
        string json = await GetStringAsync(url, ct);
        return JsonUtility.FromJson<T>(json);
    }

    /// <summary>
    /// JSON POST
    /// </summary>
    public async UniTask<TResponse> PostJsonAsync<TRequest, TResponse>(
        string url,
        TRequest data,
        CancellationToken ct = default)
    {
        string json = JsonUtility.ToJson(data);
        using var content = new StringContent(json, Encoding.UTF8, "application/json");

        var response = await _client.PostAsync(url, content, ct).AsUniTask();
        response.EnsureSuccessStatusCode();

        string responseJson = await response.Content.ReadAsStringAsync().AsUniTask();
        return JsonUtility.FromJson<TResponse>(responseJson);
    }

    /// <summary>
    /// 타임아웃이 적용된 요청
    /// </summary>
    public async UniTask<string> GetWithTimeoutAsync(
        string url,
        TimeSpan timeout,
        CancellationToken ct = default)
    {
        using var timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        timeoutCts.CancelAfter(timeout);

        try
        {
            return await GetStringAsync(url, timeoutCts.Token);
        }
        catch (OperationCanceledException) when (!ct.IsCancellationRequested)
        {
            throw new TimeoutException($"요청 시간 초과: {timeout.TotalSeconds}초");
        }
    }

    /// <summary>
    /// 진행률과 함께 다운로드
    /// </summary>
    public async UniTask<byte[]> DownloadWithProgressAsync(
        string url,
        IProgress<float> progress,
        CancellationToken ct = default)
    {
        using var response = await _client.GetAsync(
            url,
            HttpCompletionOption.ResponseHeadersRead,
            ct).AsUniTask();

        response.EnsureSuccessStatusCode();

        var totalBytes = response.Content.Headers.ContentLength ?? -1L;
        using var stream = await response.Content.ReadAsStreamAsync().AsUniTask();

        var buffer = new byte[8192];
        var result = new System.IO.MemoryStream();
        long bytesRead = 0;
        int read;

        while ((read = await stream.ReadAsync(buffer, 0, buffer.Length, ct).AsUniTask()) > 0)
        {
            await result.WriteAsync(buffer, 0, read, ct).AsUniTask();
            bytesRead += read;

            if (totalBytes > 0)
            {
                progress?.Report((float)bytesRead / totalBytes);
            }

            // 프레임 양보 (메인 스레드 블로킹 방지)
            await UniTask.Yield();
        }

        progress?.Report(1f);
        return result.ToArray();
    }
}

/// <summary>
/// UniTask HTTP 서비스 사용 예시
/// </summary>
public class UniTaskHttpExample : MonoBehaviour
{
    [Serializable]
    public class GameData
    {
        public int level;
        public int score;
        public string playerName;
    }

    private UniTaskHttpService _httpService;

    private void Start()
    {
        _httpService = new UniTaskHttpService();

        // destroyCancellationToken 활용
        LoadGameDataAsync(destroyCancellationToken).Forget();
    }

    private async UniTaskVoid LoadGameDataAsync(CancellationToken ct)
    {
        try
        {
            // 5초 타임아웃
            var data = await _httpService.GetWithTimeoutAsync(
                "https://api.example.com/game/data",
                TimeSpan.FromSeconds(5),
                ct);

            Debug.Log($"게임 데이터 로드 완료: {data}");
        }
        catch (TimeoutException)
        {
            Debug.LogWarning("서버 응답 시간 초과, 오프라인 데이터 사용");
            // 오프라인 폴백 로직
        }
        catch (HttpRequestException ex)
        {
            Debug.LogError($"네트워크 오류: {ex.Message}");
        }
        catch (OperationCanceledException)
        {
            Debug.Log("요청 취소됨");
        }
    }

    public async UniTask SaveGameDataAsync(GameData data, CancellationToken ct)
    {
        await _httpService.PostJsonAsync<GameData, object>(
            "https://api.example.com/game/save",
            data,
            ct);

        Debug.Log("게임 데이터 저장 완료");
    }
}
```

---

## 실용적인 API 클라이언트 구현

```csharp
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 게임 서버 API 클라이언트
/// </summary>
public class GameApiClient : IDisposable
{
    private readonly HttpClient _client;
    private readonly string _baseUrl;
    private string _authToken;
    private bool _disposed;

    public GameApiClient(string baseUrl)
    {
        _baseUrl = baseUrl.TrimEnd('/');

        var handler = new HttpClientHandler
        {
            AutomaticDecompression = System.Net.DecompressionMethods.GZip
        };

        _client = new HttpClient(handler)
        {
            Timeout = TimeSpan.FromSeconds(30)
        };

        _client.DefaultRequestHeaders.Accept.Add(
            new System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("application/json"));
    }

    #region Authentication

    [Serializable]
    public class AuthRequest
    {
        public string email;
        public string password;
    }

    [Serializable]
    public class AuthResponse
    {
        public string accessToken;
        public string refreshToken;
        public long expiresIn;
    }

    /// <summary>
    /// 로그인
    /// </summary>
    public async Task<AuthResponse> LoginAsync(
        string email,
        string password,
        CancellationToken ct = default)
    {
        var request = new AuthRequest { email = email, password = password };
        var response = await PostAsync<AuthRequest, AuthResponse>(
            "/auth/login",
            request,
            ct);

        SetAuthToken(response.accessToken);
        return response;
    }

    /// <summary>
    /// 인증 토큰 설정
    /// </summary>
    public void SetAuthToken(string token)
    {
        _authToken = token;
        _client.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
    }

    /// <summary>
    /// 로그아웃
    /// </summary>
    public void Logout()
    {
        _authToken = null;
        _client.DefaultRequestHeaders.Authorization = null;
    }

    #endregion

    #region Player Data

    [Serializable]
    public class PlayerProfile
    {
        public string id;
        public string nickname;
        public int level;
        public int experience;
        public int gold;
        public int gems;
    }

    /// <summary>
    /// 플레이어 프로필 조회
    /// </summary>
    public Task<PlayerProfile> GetProfileAsync(CancellationToken ct = default)
    {
        return GetAsync<PlayerProfile>("/player/profile", ct);
    }

    /// <summary>
    /// 플레이어 프로필 업데이트
    /// </summary>
    public Task<PlayerProfile> UpdateProfileAsync(
        PlayerProfile profile,
        CancellationToken ct = default)
    {
        return PutAsync<PlayerProfile, PlayerProfile>("/player/profile", profile, ct);
    }

    #endregion

    #region Leaderboard

    [Serializable]
    public class LeaderboardEntry
    {
        public int rank;
        public string playerId;
        public string nickname;
        public int score;
    }

    [Serializable]
    public class LeaderboardResponse
    {
        public LeaderboardEntry[] entries;
        public LeaderboardEntry myRank;
    }

    /// <summary>
    /// 리더보드 조회
    /// </summary>
    public Task<LeaderboardResponse> GetLeaderboardAsync(
        string leaderboardId,
        int limit = 100,
        CancellationToken ct = default)
    {
        return GetAsync<LeaderboardResponse>(
            $"/leaderboard/{leaderboardId}?limit={limit}",
            ct);
    }

    /// <summary>
    /// 점수 제출
    /// </summary>
    public Task<LeaderboardEntry> SubmitScoreAsync(
        string leaderboardId,
        int score,
        CancellationToken ct = default)
    {
        return PostAsync<object, LeaderboardEntry>(
            $"/leaderboard/{leaderboardId}/submit",
            new { score },
            ct);
    }

    #endregion

    #region HTTP Methods

    private async Task<T> GetAsync<T>(string endpoint, CancellationToken ct)
    {
        var url = _baseUrl + endpoint;
        var response = await _client.GetAsync(url, ct);
        return await HandleResponseAsync<T>(response);
    }

    private async Task<TResponse> PostAsync<TRequest, TResponse>(
        string endpoint,
        TRequest data,
        CancellationToken ct)
    {
        var url = _baseUrl + endpoint;
        var json = JsonUtility.ToJson(data);
        using var content = new StringContent(json, Encoding.UTF8, "application/json");

        var response = await _client.PostAsync(url, content, ct);
        return await HandleResponseAsync<TResponse>(response);
    }

    private async Task<TResponse> PutAsync<TRequest, TResponse>(
        string endpoint,
        TRequest data,
        CancellationToken ct)
    {
        var url = _baseUrl + endpoint;
        var json = JsonUtility.ToJson(data);
        using var content = new StringContent(json, Encoding.UTF8, "application/json");

        var response = await _client.PutAsync(url, content, ct);
        return await HandleResponseAsync<TResponse>(response);
    }

    private async Task DeleteAsync(string endpoint, CancellationToken ct)
    {
        var url = _baseUrl + endpoint;
        var response = await _client.DeleteAsync(url, ct);
        response.EnsureSuccessStatusCode();
    }

    private async Task<T> HandleResponseAsync<T>(HttpResponseMessage response)
    {
        var content = await response.Content.ReadAsStringAsync();

        if (!response.IsSuccessStatusCode)
        {
            throw new ApiException(
                response.StatusCode,
                content,
                $"API 오류: {response.StatusCode}");
        }

        return JsonUtility.FromJson<T>(content);
    }

    #endregion

    public void Dispose()
    {
        if (!_disposed)
        {
            _client?.Dispose();
            _disposed = true;
        }
    }
}

/// <summary>
/// API 예외
/// </summary>
public class ApiException : Exception
{
    public System.Net.HttpStatusCode StatusCode { get; }
    public string ResponseContent { get; }

    public ApiException(
        System.Net.HttpStatusCode statusCode,
        string responseContent,
        string message)
        : base(message)
    {
        StatusCode = statusCode;
        ResponseContent = responseContent;
    }
}

/// <summary>
/// API 클라이언트 사용 예시
/// </summary>
public class GameApiExample : MonoBehaviour
{
    private GameApiClient _api;
    private CancellationTokenSource _cts;

    private void Awake()
    {
        _api = new GameApiClient("https://api.mygame.com/v1");
        _cts = new CancellationTokenSource();
    }

    private async void Start()
    {
        try
        {
            // 로그인
            var auth = await _api.LoginAsync(
                "player@example.com",
                "password123",
                _cts.Token);

            Debug.Log($"로그인 성공, 토큰 만료: {auth.expiresIn}초");

            // 프로필 조회
            var profile = await _api.GetProfileAsync(_cts.Token);
            Debug.Log($"환영합니다, {profile.nickname}! (Lv.{profile.level})");

            // 리더보드 조회
            var leaderboard = await _api.GetLeaderboardAsync("weekly", 10, _cts.Token);
            Debug.Log($"내 순위: {leaderboard.myRank?.rank ?? -1}");
        }
        catch (ApiException ex)
        {
            Debug.LogError($"API 오류 ({ex.StatusCode}): {ex.ResponseContent}");
        }
        catch (Exception ex)
        {
            Debug.LogError($"오류: {ex.Message}");
        }
    }

    private void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _api?.Dispose();
    }
}
```

---

## 플랫폼별 주의사항

### WebGL 제한

```csharp
/// <summary>
/// 플랫폼별 HTTP 클라이언트 팩토리
/// </summary>
public static class PlatformHttpClientFactory
{
    public static bool IsHttpClientSupported
    {
        get
        {
#if UNITY_WEBGL && !UNITY_EDITOR
            // WebGL에서는 HttpClient 미지원
            return false;
#else
            return true;
#endif
        }
    }

    /// <summary>
    /// 플랫폼에 적합한 HTTP 서비스 생성
    /// </summary>
    public static IHttpService CreateHttpService()
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        // WebGL용 UnityWebRequest 기반 서비스 반환
        return new UnityWebRequestHttpService();
#else
        // 기타 플랫폼용 HttpClient 기반 서비스 반환
        return new HttpClientService();
#endif
    }
}

/// <summary>
/// 플랫폼 독립적인 HTTP 서비스 인터페이스
/// </summary>
public interface IHttpService
{
    System.Threading.Tasks.Task<string> GetAsync(string url, CancellationToken ct);
    System.Threading.Tasks.Task<string> PostAsync(string url, string json, CancellationToken ct);
}
```

### SSL/TLS 인증서 처리

```csharp
using System.Net.Http;
using System.Net.Security;
using System.Security.Cryptography.X509Certificates;
using UnityEngine;

/// <summary>
/// 개발 환경용 인증서 무시 설정
/// 프로덕션에서는 절대 사용하지 마세요!
/// </summary>
public static class DevelopmentHttpClient
{
    public static HttpClient CreateInsecureClient()
    {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
        var handler = new HttpClientHandler
        {
            // 개발 환경에서만 인증서 검증 우회
            ServerCertificateCustomValidationCallback =
                (message, cert, chain, errors) =>
                {
                    Debug.LogWarning($"개발 모드: SSL 인증서 검증 건너뜀 - {cert?.Subject}");
                    return true;
                }
        };

        return new HttpClient(handler);
#else
        // 프로덕션에서는 일반 클라이언트 사용
        return new HttpClient();
#endif
    }
}
```

---

## 성능 최적화

```csharp
using System;
using System.Buffers;
using System.IO;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 메모리 효율적인 HTTP 다운로드
/// </summary>
public class OptimizedHttpDownloader
{
    private readonly HttpClient _client;

    // ArrayPool을 사용하여 버퍼 재사용
    private static readonly ArrayPool<byte> BufferPool = ArrayPool<byte>.Shared;

    public OptimizedHttpDownloader(HttpClient client)
    {
        _client = client;
    }

    /// <summary>
    /// 메모리 효율적인 대용량 파일 다운로드
    /// </summary>
    public async Task DownloadToFileAsync(
        string url,
        string filePath,
        IProgress<DownloadProgress> progress = null,
        CancellationToken ct = default)
    {
        using var response = await _client.GetAsync(
            url,
            HttpCompletionOption.ResponseHeadersRead,
            ct);

        response.EnsureSuccessStatusCode();

        var totalBytes = response.Content.Headers.ContentLength ?? -1L;

        using var contentStream = await response.Content.ReadAsStreamAsync();
        using var fileStream = new FileStream(
            filePath,
            FileMode.Create,
            FileAccess.Write,
            FileShare.None,
            bufferSize: 8192,
            useAsync: true);

        // ArrayPool에서 버퍼 대여
        var buffer = BufferPool.Rent(8192);

        try
        {
            long bytesDownloaded = 0;
            int bytesRead;

            while ((bytesRead = await contentStream.ReadAsync(
                buffer, 0, buffer.Length, ct)) > 0)
            {
                await fileStream.WriteAsync(buffer, 0, bytesRead, ct);
                bytesDownloaded += bytesRead;

                progress?.Report(new DownloadProgress
                {
                    BytesDownloaded = bytesDownloaded,
                    TotalBytes = totalBytes,
                    Progress = totalBytes > 0
                        ? (float)bytesDownloaded / totalBytes
                        : 0
                });
            }
        }
        finally
        {
            // 버퍼 반환
            BufferPool.Return(buffer);
        }
    }

    /// <summary>
    /// 청크 단위 다운로드 (Range 헤더 사용)
    /// </summary>
    public async Task DownloadWithResumeAsync(
        string url,
        string filePath,
        CancellationToken ct = default)
    {
        long existingLength = 0;
        FileMode mode = FileMode.Create;

        // 기존 파일이 있으면 이어받기
        if (File.Exists(filePath))
        {
            existingLength = new FileInfo(filePath).Length;
            mode = FileMode.Append;
        }

        using var request = new HttpRequestMessage(HttpMethod.Get, url);

        if (existingLength > 0)
        {
            request.Headers.Range =
                new System.Net.Http.Headers.RangeHeaderValue(existingLength, null);
        }

        using var response = await _client.SendAsync(
            request,
            HttpCompletionOption.ResponseHeadersRead,
            ct);

        // 206 Partial Content 또는 200 OK
        if (response.StatusCode != System.Net.HttpStatusCode.PartialContent &&
            response.StatusCode != System.Net.HttpStatusCode.OK)
        {
            throw new HttpRequestException($"예상치 못한 상태 코드: {response.StatusCode}");
        }

        using var contentStream = await response.Content.ReadAsStreamAsync();
        using var fileStream = new FileStream(filePath, mode, FileAccess.Write);

        await contentStream.CopyToAsync(fileStream, 8192, ct);
    }
}

public struct DownloadProgress
{
    public long BytesDownloaded;
    public long TotalBytes;
    public float Progress;
}
```

---

## 주의사항

### 1. HttpClient 인스턴스 관리

```
⚠️ 잘못된 사용:
┌─────────────────────────────────────────────────────────────┐
│  using (var client = new HttpClient())  // 매 요청마다 생성  │
│  {                                                           │
│      await client.GetAsync(url);  // 소켓 고갈 위험!         │
│  }                                                           │
└─────────────────────────────────────────────────────────────┘

✅ 올바른 사용:
┌─────────────────────────────────────────────────────────────┐
│  // 싱글톤 또는 장기 보유                                     │
│  private static readonly HttpClient _client = new();         │
│                                                              │
│  // 재사용                                                   │
│  await _client.GetAsync(url1);                               │
│  await _client.GetAsync(url2);                               │
└─────────────────────────────────────────────────────────────┘
```

### 2. 타임아웃 처리

```csharp
// HttpClient.Timeout은 전체 요청에 대한 타임아웃
// 개별 요청에 다른 타임아웃을 적용하려면 CancellationToken 사용

public async Task<string> GetWithCustomTimeoutAsync(
    string url,
    TimeSpan timeout,
    CancellationToken ct = default)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(timeout);

    try
    {
        var response = await _client.GetAsync(url, cts.Token);
        return await response.Content.ReadAsStringAsync();
    }
    catch (TaskCanceledException) when (!ct.IsCancellationRequested)
    {
        throw new TimeoutException($"요청 시간 초과: {timeout}");
    }
}
```

### 3. 스레드 안전성

```csharp
// HttpClient 메서드들은 스레드 안전
// 단, DefaultRequestHeaders 수정은 스레드 안전하지 않음

// ❌ 잘못된 예: 동시에 헤더 수정
Parallel.For(0, 10, i =>
{
    // 레이스 컨디션 발생 가능!
    _client.DefaultRequestHeaders.Add("X-Request-Id", i.ToString());
});

// ✅ 올바른 예: HttpRequestMessage 사용
async Task SafeRequestAsync(int requestId)
{
    using var request = new HttpRequestMessage(HttpMethod.Get, url);
    request.Headers.Add("X-Request-Id", requestId.ToString());

    await _client.SendAsync(request);
}
```

---

## 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────┐
│                    HttpClient 베스트 프랙티스                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 싱글톤 패턴                                                  │
│     ├── HttpClient는 재사용하도록 설계됨                         │
│     ├── 매 요청마다 새로 생성하면 소켓 고갈                       │
│     └── 앱 수명 동안 동일 인스턴스 사용                          │
│                                                                  │
│  2. 취소 토큰 활용                                               │
│     ├── 모든 비동기 메서드에 CancellationToken 전달              │
│     ├── 씬 전환, 오브젝트 파괴 시 요청 취소                      │
│     └── 타임아웃에도 CancellationToken 활용                      │
│                                                                  │
│  3. 에러 핸들링                                                  │
│     ├── HttpRequestException: 네트워크 오류                      │
│     ├── TaskCanceledException: 타임아웃 또는 취소                │
│     └── EnsureSuccessStatusCode() 활용                          │
│                                                                  │
│  4. 플랫폼 호환성                                                │
│     ├── WebGL에서는 UnityWebRequest 사용                         │
│     ├── 플랫폼 추상화 레이어 구현                                │
│     └── #if 지시문으로 플랫폼별 분기                             │
│                                                                  │
│  5. 재시도 및 회복력                                             │
│     ├── 일시적 오류에 대해 지수 백오프 재시도                     │
│     ├── 429 Too Many Requests 처리                              │
│     └── Circuit Breaker 패턴 고려                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [Microsoft Docs: HttpClient](https://docs.microsoft.com/dotnet/api/system.net.http.httpclient)
- [HttpClient Guidelines](https://docs.microsoft.com/dotnet/fundamentals/networking/http/httpclient-guidelines)
- [Unity Networking Best Practices](https://docs.unity3d.com/Manual/webgl-networking.html)
- [UniTask GitHub](https://github.com/Cysharp/UniTask)

---

## 다음 단계

- [Section 27: HTTP/REST & JSON](../10-protocols/27-http-rest-json.md) - REST API 패턴 및 JSON 직렬화
- [Section 28: WebSocket](../10-protocols/28-websocket.md) - 실시간 양방향 통신
