# 26. HttpClient

## 개요

`System.Net.Http.HttpClient`는 .NET 표준 HTTP 클라이언트로, Unity에서도 사용할 수 있습니다. `UnityWebRequest`와 달리 메인 스레드에 종속되지 않으며, `async/await` 패턴과 자연스럽게 통합됩니다. 미들웨어 파이프라인(`DelegatingHandler`), 세밀한 타임아웃 제어, 풍부한 .NET 에코시스템 활용이 가능하지만, Unity 플랫폼별 호환성에 주의가 필요합니다.

```
┌──────────────────────────────────────────────────────────────────┐
│              HttpClient vs UnityWebRequest 핵심 차이               │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  특성              │ HttpClient          │ UnityWebRequest         │
│  ─────────────────┼─────────────────────┼─────────────────────────│
│  스레드            │ 백그라운드 스레드    │ 메인 스레드 (코루틴)     │
│  async/await       │ 네이티브 지원        │ Unity 2023+ / UniTask   │
│  미들웨어          │ DelegatingHandler    │ 없음                    │
│  재사용            │ 싱글턴 권장          │ 요청마다 생성/폐기       │
│  플랫폼 호환       │ IL2CPP 주의 필요     │ 모든 플랫폼 안정         │
│  에셋 다운로드     │ 수동 처리            │ 텍스처/오디오/번들 내장   │
│  인증서 관리       │ HttpClientHandler    │ CertificateHandler      │
│  Cookie 관리       │ CookieContainer 내장 │ 수동 관리               │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. HttpClient 재사용과 소켓 고갈 문제

### 왜 HttpClient를 재사용해야 하는가?

`HttpClient`는 내부적으로 TCP 연결 풀을 관리합니다. 요청마다 새로운 인스턴스를 생성하면 소켓이 `TIME_WAIT` 상태로 남아 **소켓 고갈(Socket Exhaustion)** 문제가 발생합니다.

```csharp
using System;
using System.Net.Http;
using System.Threading.Tasks;
using UnityEngine;

public class HttpClientReuseExample : MonoBehaviour
{
    // =============================================
    // ❌ 잘못된 사용: 요청마다 새 인스턴스 생성
    // =============================================

    private async Task BadPatternAsync()
    {
        for (int i = 0; i < 100; i++)
        {
            // 매번 새로운 HttpClient 생성 → 소켓 고갈 위험!
            using (var client = new HttpClient())
            {
                var response = await client.GetStringAsync("https://api.example.com/data");
                Debug.Log(response);
            }
            // Dispose 후에도 소켓이 TIME_WAIT 상태로 약 240초 유지
            // 100회 반복 시 100개의 소켓이 TIME_WAIT에 걸림
        }
    }

    // =============================================
    // ✅ 올바른 사용: 싱글턴으로 재사용
    // =============================================

    // 정적 인스턴스로 애플리케이션 수명 동안 유지
    private static readonly HttpClient sharedClient = new HttpClient
    {
        BaseAddress = new Uri("https://api.example.com/"),
        Timeout = TimeSpan.FromSeconds(30)
    };

    private async Task GoodPatternAsync()
    {
        for (int i = 0; i < 100; i++)
        {
            // 동일한 HttpClient 인스턴스 재사용 → 연결 풀 활용
            var response = await sharedClient.GetStringAsync("data");
            Debug.Log(response);
        }
    }
}
```

### 소켓 고갈의 원리

```
❌ 요청마다 새 인스턴스:

  요청 1 → [HttpClient 생성] → [TCP 연결] → [응답] → [Dispose]
                                                          ↓
                                              소켓: TIME_WAIT (240초)
  요청 2 → [HttpClient 생성] → [TCP 연결] → [응답] → [Dispose]
                                                          ↓
                                              소켓: TIME_WAIT (240초)
  ...반복 → 소켓 고갈!

✅ 싱글턴 재사용:

  요청 1 ─┐
  요청 2 ─┼→ [HttpClient 싱글턴] → [TCP 연결 풀] → 연결 재사용
  요청 3 ─┘                        (Keep-Alive)
```

### Unity 전용 싱글턴 패턴

```csharp
using System;
using System.Net.Http;
using System.Net.Http.Headers;
using UnityEngine;

/// <summary>
/// Unity 환경에서 HttpClient를 안전하게 관리하는 싱글턴.
/// 애플리케이션 수명 동안 하나의 인스턴스만 유지하여 소켓 고갈을 방지합니다.
/// </summary>
public static class HttpClientProvider
{
    private static HttpClient instance;
    private static readonly object lockObj = new object();

    public static HttpClient Client
    {
        get
        {
            if (instance == null)
            {
                lock (lockObj)
                {
                    if (instance == null)
                    {
                        instance = CreateClient();
                    }
                }
            }
            return instance;
        }
    }

    private static HttpClient CreateClient()
    {
        var handler = new HttpClientHandler
        {
            // 연결 풀 설정
            MaxConnectionsPerServer = 10,

            // 자동 압축
            AutomaticDecompression =
                System.Net.DecompressionMethods.GZip |
                System.Net.DecompressionMethods.Deflate,

            // 리다이렉트 설정
            AllowAutoRedirect = true,
            MaxAutomaticRedirections = 5
        };

        var client = new HttpClient(handler)
        {
            Timeout = TimeSpan.FromSeconds(30)
        };

        // 기본 헤더 설정
        client.DefaultRequestHeaders.Accept.Add(
            new MediaTypeWithQualityHeaderValue("application/json"));
        client.DefaultRequestHeaders.UserAgent.ParseAdd(
            $"UnityGame/{Application.version}");

        return client;
    }

    /// <summary>
    /// 애플리케이션 종료 시 호출하여 리소스를 정리합니다.
    /// </summary>
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    private static void Reset()
    {
        // Domain Reload 대비 (Enter Play Mode Options)
        instance?.Dispose();
        instance = null;
    }
}
```

---

## 2. Unity에서의 HttpClient 설정

### 기본 설정

```csharp
using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

public class HttpClientSetupExample : MonoBehaviour
{
    private static HttpClient httpClient;

    // =============================================
    // 초기화 (Awake에서 한 번만)
    // =============================================

    private void Awake()
    {
        if (httpClient == null)
        {
            var handler = new HttpClientHandler
            {
                // 서버당 최대 동시 연결 수
                MaxConnectionsPerServer = 6,

                // gzip/deflate 자동 해제
                AutomaticDecompression =
                    System.Net.DecompressionMethods.GZip |
                    System.Net.DecompressionMethods.Deflate,

                // 프록시 설정 (필요 시)
                UseProxy = false,

                // 쿠키 관리
                UseCookies = true,
                CookieContainer = new System.Net.CookieContainer()
            };

            httpClient = new HttpClient(handler)
            {
                BaseAddress = new Uri("https://api.mygame.com/v1/"),
                Timeout = TimeSpan.FromSeconds(30)
            };

            // 기본 요청 헤더
            httpClient.DefaultRequestHeaders.Accept.Add(
                new MediaTypeWithQualityHeaderValue("application/json"));
            httpClient.DefaultRequestHeaders.AcceptEncoding.Add(
                new StringWithQualityHeaderValue("gzip"));
            httpClient.DefaultRequestHeaders.Add("X-Game-Version", Application.version);
            httpClient.DefaultRequestHeaders.Add("X-Platform", Application.platform.ToString());
        }
    }

    // =============================================
    // GET 요청 예시
    // =============================================

    public async Task<string> GetPlayerDataAsync(string playerId, CancellationToken token)
    {
        try
        {
            // BaseAddress를 기반으로 상대 경로 사용
            HttpResponseMessage response = await httpClient.GetAsync(
                $"players/{playerId}", token);

            response.EnsureSuccessStatusCode();

            string json = await response.Content.ReadAsStringAsync();
            return json;
        }
        catch (HttpRequestException e)
        {
            Debug.LogError($"HTTP 요청 실패: {e.Message}");
            throw;
        }
    }

    // =============================================
    // POST 요청 예시
    // =============================================

    public async Task<string> SaveScoreAsync(int score, CancellationToken token)
    {
        var content = new StringContent(
            JsonUtility.ToJson(new ScoreData { score = score, timestamp = DateTime.UtcNow.ToString("o") }),
            Encoding.UTF8,
            "application/json"
        );

        HttpResponseMessage response = await httpClient.PostAsync("scores", content, token);
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadAsStringAsync();
    }

    [Serializable]
    private struct ScoreData
    {
        public int score;
        public string timestamp;
    }
}
```

### 인증 토큰 관리

```csharp
using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 인증 토큰을 관리하며, 토큰 만료 시 자동 갱신하는 HttpClient 래퍼.
/// </summary>
public class AuthenticatedHttpClient
{
    private readonly HttpClient client;
    private string accessToken;
    private string refreshToken;
    private DateTime tokenExpiry;

    public AuthenticatedHttpClient(HttpClient client)
    {
        this.client = client;
    }

    public void SetTokens(string access, string refresh, DateTime expiry)
    {
        accessToken = access;
        refreshToken = refresh;
        tokenExpiry = expiry;
    }

    public async Task<HttpResponseMessage> SendWithAuthAsync(
        HttpRequestMessage request, CancellationToken token = default)
    {
        // 토큰 만료 확인 및 갱신
        if (DateTime.UtcNow >= tokenExpiry.AddMinutes(-1))
        {
            await RefreshTokenAsync(token);
        }

        // Authorization 헤더 설정
        request.Headers.Authorization =
            new AuthenticationHeaderValue("Bearer", accessToken);

        HttpResponseMessage response = await client.SendAsync(request, token);

        // 401이면 토큰 갱신 후 재시도
        if (response.StatusCode == System.Net.HttpStatusCode.Unauthorized)
        {
            await RefreshTokenAsync(token);
            request.Headers.Authorization =
                new AuthenticationHeaderValue("Bearer", accessToken);

            // HttpRequestMessage는 재사용 불가 → 새로 생성 필요
            var retryRequest = await CloneRequestAsync(request);
            response = await client.SendAsync(retryRequest, token);
        }

        return response;
    }

    private async Task RefreshTokenAsync(CancellationToken token)
    {
        Debug.Log("토큰 갱신 중...");

        var content = new StringContent(
            $"{{\"refresh_token\":\"{refreshToken}\"}}",
            System.Text.Encoding.UTF8,
            "application/json"
        );

        var response = await client.PostAsync("auth/refresh", content, token);
        response.EnsureSuccessStatusCode();

        string json = await response.Content.ReadAsStringAsync();
        // 토큰 파싱 및 업데이트 (프로젝트의 JSON 라이브러리에 맞게 구현)
        Debug.Log("토큰 갱신 완료");
    }

    private async Task<HttpRequestMessage> CloneRequestAsync(HttpRequestMessage original)
    {
        var clone = new HttpRequestMessage(original.Method, original.RequestUri);

        if (original.Content != null)
        {
            var content = await original.Content.ReadAsByteArrayAsync();
            clone.Content = new ByteArrayContent(content);

            foreach (var header in original.Content.Headers)
            {
                clone.Content.Headers.TryAddWithoutValidation(header.Key, header.Value);
            }
        }

        foreach (var header in original.Headers)
        {
            clone.Headers.TryAddWithoutValidation(header.Key, header.Value);
        }

        return clone;
    }
}
```

---

## 3. 타임아웃 설정 및 관리

### HttpClient의 타임아웃 계층

`HttpClient`에서 타임아웃은 여러 계층에서 설정할 수 있습니다. 각 계층의 역할과 우선순위를 이해하는 것이 중요합니다.

```
┌─────────────────────────────────────────────────┐
│                타임아웃 계층 구조                  │
├─────────────────────────────────────────────────┤
│                                                   │
│  HttpClient.Timeout (전체 요청 타임아웃)           │
│    └─ CancellationTokenSource (개별 요청)         │
│        └─ HttpRequestMessage 단위 제어            │
│                                                   │
│  우선순위:                                        │
│  CancellationToken > HttpClient.Timeout           │
│                                                   │
└─────────────────────────────────────────────────┘
```

```csharp
using System;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

public class TimeoutManagementExample : MonoBehaviour
{
    private static readonly HttpClient client = new HttpClient
    {
        // 전체 기본 타임아웃 (모든 요청에 적용)
        Timeout = TimeSpan.FromSeconds(30)
    };

    // =============================================
    // 기본 타임아웃 (HttpClient.Timeout)
    // =============================================

    public async Task BasicTimeoutAsync()
    {
        try
        {
            // HttpClient.Timeout(30초)이 적용됨
            string result = await client.GetStringAsync("https://api.example.com/data");
            Debug.Log(result);
        }
        catch (TaskCanceledException)
        {
            // HttpClient.Timeout 초과 시 TaskCanceledException 발생
            Debug.LogError("요청 타임아웃 (30초 초과)");
        }
        catch (HttpRequestException e)
        {
            Debug.LogError($"HTTP 에러: {e.Message}");
        }
    }

    // =============================================
    // 개별 요청 타임아웃 (CancellationToken)
    // =============================================

    public async Task PerRequestTimeoutAsync()
    {
        // 이 요청만 5초 타임아웃 적용
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));

        try
        {
            HttpResponseMessage response = await client.GetAsync(
                "https://api.example.com/slow-endpoint",
                cts.Token
            );

            response.EnsureSuccessStatusCode();
            string body = await response.Content.ReadAsStringAsync();
            Debug.Log(body);
        }
        catch (OperationCanceledException) when (cts.IsCancellationRequested)
        {
            Debug.LogError("개별 요청 타임아웃 (5초 초과)");
        }
    }

    // =============================================
    // 연결 타임아웃과 읽기 타임아웃 분리
    // =============================================

    public async Task SeparateTimeoutsAsync(CancellationToken externalToken)
    {
        // 연결 타임아웃: 5초
        // 전체 타임아웃: 30초
        using var connectCts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            connectCts.Token, externalToken);

        try
        {
            // HttpCompletionOption.ResponseHeadersRead:
            // 헤더를 받으면 즉시 반환 → 연결 타임아웃 역할
            HttpResponseMessage response = await client.GetAsync(
                "https://api.example.com/large-data",
                HttpCompletionOption.ResponseHeadersRead,
                linkedCts.Token
            );

            response.EnsureSuccessStatusCode();

            // 연결 성공 → 연결 타임아웃 해제
            connectCts.Dispose();

            // 읽기 타임아웃: 60초 (대용량 데이터 수신)
            using var readCts = new CancellationTokenSource(TimeSpan.FromSeconds(60));
            using var readLinkedCts = CancellationTokenSource.CreateLinkedTokenSource(
                readCts.Token, externalToken);

            string body = await response.Content.ReadAsStringAsync();
            Debug.Log($"수신 완료: {body.Length} chars");
        }
        catch (OperationCanceledException e)
        {
            if (connectCts.IsCancellationRequested)
                Debug.LogError("연결 타임아웃");
            else if (externalToken.IsCancellationRequested)
                Debug.Log("외부에서 취소됨");
            else
                Debug.LogError("읽기 타임아웃");
        }
    }

    // =============================================
    // Infinite Timeout + 개별 제어 패턴
    // =============================================

    // HttpClient.Timeout을 무한으로 설정하고,
    // 모든 타임아웃을 CancellationToken으로 제어하는 패턴
    private static readonly HttpClient flexibleClient = new HttpClient
    {
        Timeout = System.Threading.Timeout.InfiniteTimeSpan
    };

    public async Task FlexibleTimeoutAsync(CancellationToken token)
    {
        // API 호출: 10초 타임아웃
        using var apiCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            apiCts.Token, token);

        try
        {
            var response = await flexibleClient.GetAsync(
                "https://api.example.com/data", linkedCts.Token);
            response.EnsureSuccessStatusCode();
            Debug.Log("API 호출 성공");
        }
        catch (OperationCanceledException)
        {
            Debug.LogError(apiCts.IsCancellationRequested
                ? "API 호출 타임아웃" : "외부 취소");
        }
    }
}
```

---

## 4. HttpClient vs UnityWebRequest 비교

### 상세 비교표

```
┌──────────────────────────────────────────────────────────────────────┐
│                   HttpClient vs UnityWebRequest 상세 비교              │
├──────────────┬───────────────────────┬────────────────────────────────┤
│ 항목          │ HttpClient            │ UnityWebRequest                │
├──────────────┼───────────────────────┼────────────────────────────────┤
│ 네임스페이스  │ System.Net.Http       │ UnityEngine.Networking         │
│ 재사용        │ 싱글턴 필수           │ 요청마다 생성/폐기              │
│ 스레딩        │ ThreadPool 기반       │ Unity 메인 스레드               │
│ async/await   │ 네이티브 지원         │ Unity 2023+ 또는 UniTask       │
│ 미들웨어      │ DelegatingHandler     │ 없음                           │
│ JSON 직렬화   │ System.Text.Json 등   │ JsonUtility (제한적)            │
│ 스트리밍      │ Stream 기반           │ DownloadHandler                │
│ 진행률        │ 수동 구현             │ downloadProgress 내장           │
│ 텍스처        │ 수동 변환 필요        │ UnityWebRequestTexture         │
│ 에셋번들      │ 지원 안 함            │ UnityWebRequestAssetBundle     │
│ WebGL         │ 제한적               │ 완벽 지원                       │
│ IL2CPP        │ 일부 제약             │ 안정적                          │
│ 쿠키          │ CookieContainer      │ 수동 관리                       │
│ 리다이렉트    │ 자동 (설정 가능)      │ 자동 (설정 가능)                │
│ 압축          │ 자동 해제             │ 수동 처리                       │
│ 테스트        │ 모킹 용이             │ 모킹 어려움                     │
│ HTTP/2        │ .NET 지원 시 가능     │ 미지원                          │
│ 인증서 핀닝   │ HttpClientHandler     │ CertificateHandler             │
└──────────────┴───────────────────────┴────────────────────────────────┘
```

### 언제 무엇을 사용할까?

```csharp
using System;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;

public class WhenToUseWhatExample : MonoBehaviour
{
    // =============================================
    // ✅ HttpClient가 적합한 경우
    // =============================================

    // 1. 복잡한 REST API 통신 (인증, 미들웨어, 재시도)
    // 2. 백그라운드 스레드에서의 대량 네트워크 작업
    // 3. 테스트 가능한 코드 작성 (DI/모킹)
    // 4. .NET 라이브러리 호환이 필요한 경우
    // 5. HTTP/2 또는 고급 HTTP 기능이 필요한 경우

    private static readonly HttpClient httpClient = new HttpClient();

    public async Task FetchLeaderboardAsync(CancellationToken token)
    {
        // 복잡한 API 호출 + 인증 + 재시도 = HttpClient
        httpClient.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", "token");

        var response = await httpClient.GetAsync("https://api.game.com/leaderboard", token);
        response.EnsureSuccessStatusCode();
        string json = await response.Content.ReadAsStringAsync();
        Debug.Log($"리더보드: {json}");
    }

    // =============================================
    // ✅ UnityWebRequest가 적합한 경우
    // =============================================

    // 1. 텍스처, 오디오, 에셋번들 다운로드
    // 2. WebGL 빌드가 필수인 프로젝트
    // 3. Unity 에디터 도구 개발
    // 4. 간단한 REST API 호출
    // 5. 진행률 표시가 필요한 다운로드

    public IEnumerator DownloadTextureCoroutine(string url)
    {
        // 텍스처 다운로드 = UnityWebRequest
        using (var request = UnityWebRequestTexture.GetTexture(url))
        {
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Texture2D tex = DownloadHandlerTexture.GetContent(request);
                Debug.Log($"텍스처: {tex.width}x{tex.height}");
            }
        }
    }

    // =============================================
    // 하이브리드 접근: 둘 다 사용
    // =============================================

    // 게임 API 통신 → HttpClient
    // 에셋 다운로드 → UnityWebRequest
    // 이렇게 역할을 분리하는 것이 현실적인 최적의 접근입니다.
}
```

---

## 5. HttpClientHandler와 DelegatingHandler

### HttpClientHandler 설정

`HttpClientHandler`는 HTTP 요청의 저수준 동작을 제어합니다.

```csharp
using System;
using System.Net;
using System.Net.Http;
using System.Net.Security;
using System.Security.Cryptography.X509Certificates;
using UnityEngine;

public class HttpClientHandlerExample : MonoBehaviour
{
    // =============================================
    // HttpClientHandler 상세 설정
    // =============================================

    public static HttpClient CreateConfiguredClient()
    {
        var handler = new HttpClientHandler
        {
            // ── 연결 설정 ──
            MaxConnectionsPerServer = 10,
            MaxAutomaticRedirections = 3,
            AllowAutoRedirect = true,

            // ── 압축 ──
            AutomaticDecompression =
                DecompressionMethods.GZip | DecompressionMethods.Deflate,

            // ── 쿠키 ──
            UseCookies = true,
            CookieContainer = new CookieContainer(),

            // ── 프록시 ──
            UseProxy = false,

            // ── SSL/TLS 설정 ──
            // 프로덕션에서는 반드시 기본 인증서 검증 사용!
            // 개발 환경에서만 커스텀 검증 사용
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            ServerCertificateCustomValidationCallback = (message, cert, chain, errors) =>
            {
                // 개발 환경: 인증서 에러 로그 출력 후 허용
                if (errors != SslPolicyErrors.None)
                {
                    Debug.LogWarning($"[DEV] 인증서 경고: {errors}");
                }
                return true;
            },
#endif

            // ── 자격 증명 ──
            UseDefaultCredentials = false,
            PreAuthenticate = false
        };

        var client = new HttpClient(handler, disposeHandler: true)
        {
            Timeout = TimeSpan.FromSeconds(30)
        };

        return client;
    }
}
```

### DelegatingHandler: 미들웨어 파이프라인

`DelegatingHandler`는 HTTP 요청/응답 파이프라인에 미들웨어를 삽입하는 패턴입니다. 로깅, 인증, 재시도 등의 횡단 관심사를 분리할 수 있습니다.

```
요청 흐름:

  [HttpClient]
      ↓
  [LoggingHandler]  ← DelegatingHandler
      ↓
  [AuthHandler]     ← DelegatingHandler
      ↓
  [RetryHandler]    ← DelegatingHandler
      ↓
  [HttpClientHandler]  ← 실제 HTTP 전송
      ↓
  [서버]
```

```csharp
using System;
using System.Diagnostics;
using System.Net;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;
using Debug = UnityEngine.Debug;

// =============================================
// 로깅 핸들러
// =============================================

public class LoggingHandler : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        Debug.Log($"[HTTP 요청] {request.Method} {request.RequestUri}");

        var stopwatch = Stopwatch.StartNew();

        HttpResponseMessage response = await base.SendAsync(request, cancellationToken);

        stopwatch.Stop();

        Debug.Log($"[HTTP 응답] {(int)response.StatusCode} {response.StatusCode} " +
                  $"({stopwatch.ElapsedMilliseconds}ms)");

        return response;
    }
}

// =============================================
// 인증 헤더 자동 삽입 핸들러
// =============================================

public class AuthenticationHandler : DelegatingHandler
{
    private readonly Func<Task<string>> getTokenAsync;

    public AuthenticationHandler(Func<Task<string>> getTokenAsync)
    {
        this.getTokenAsync = getTokenAsync;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        string token = await getTokenAsync();

        if (!string.IsNullOrEmpty(token))
        {
            request.Headers.Authorization =
                new AuthenticationHeaderValue("Bearer", token);
        }

        HttpResponseMessage response = await base.SendAsync(request, cancellationToken);

        // 401 응답 시 토큰 갱신 후 재시도
        if (response.StatusCode == HttpStatusCode.Unauthorized)
        {
            Debug.Log("401 수신 → 토큰 갱신 후 재시도");
            string newToken = await getTokenAsync();
            request.Headers.Authorization =
                new AuthenticationHeaderValue("Bearer", newToken);

            response = await base.SendAsync(request, cancellationToken);
        }

        return response;
    }
}

// =============================================
// 재시도 핸들러
// =============================================

public class RetryHandler : DelegatingHandler
{
    private readonly int maxRetries;
    private readonly TimeSpan initialDelay;

    public RetryHandler(int maxRetries = 3, double initialDelaySeconds = 1.0)
    {
        this.maxRetries = maxRetries;
        this.initialDelay = TimeSpan.FromSeconds(initialDelaySeconds);
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        HttpResponseMessage response = null;
        TimeSpan delay = initialDelay;

        for (int attempt = 0; attempt <= maxRetries; attempt++)
        {
            if (attempt > 0)
            {
                Debug.Log($"재시도 {attempt}/{maxRetries} ({delay.TotalSeconds}초 후)");
                await Task.Delay(delay, cancellationToken);
                delay *= 2; // 지수 백오프
            }

            try
            {
                response = await base.SendAsync(request, cancellationToken);

                // 성공 또는 재시도 불필요한 응답이면 반환
                if (response.IsSuccessStatusCode || !IsRetryableStatusCode(response.StatusCode))
                {
                    return response;
                }

                Debug.LogWarning($"재시도 가능한 응답: {(int)response.StatusCode}");
            }
            catch (HttpRequestException e) when (attempt < maxRetries)
            {
                Debug.LogWarning($"요청 실패, 재시도 예정: {e.Message}");
            }
        }

        return response;
    }

    private bool IsRetryableStatusCode(HttpStatusCode code)
    {
        return code == HttpStatusCode.RequestTimeout ||       // 408
               code == HttpStatusCode.TooManyRequests ||      // 429
               code == HttpStatusCode.InternalServerError ||  // 500
               code == HttpStatusCode.BadGateway ||           // 502
               code == HttpStatusCode.ServiceUnavailable ||   // 503
               code == HttpStatusCode.GatewayTimeout;         // 504
    }
}

// =============================================
// 핸들러 체인 조립
// =============================================

public class HandlerChainSetup : MonoBehaviour
{
    private HttpClient client;

    private void Awake()
    {
        // 핸들러 체인: Logging → Auth → Retry → HttpClientHandler
        var retryHandler = new RetryHandler(maxRetries: 3)
        {
            InnerHandler = new HttpClientHandler
            {
                AutomaticDecompression =
                    DecompressionMethods.GZip | DecompressionMethods.Deflate
            }
        };

        var authHandler = new AuthenticationHandler(GetAccessTokenAsync)
        {
            InnerHandler = retryHandler
        };

        var loggingHandler = new LoggingHandler
        {
            InnerHandler = authHandler
        };

        client = new HttpClient(loggingHandler)
        {
            BaseAddress = new Uri("https://api.mygame.com/"),
            Timeout = TimeSpan.FromSeconds(30)
        };
    }

    private Task<string> GetAccessTokenAsync()
    {
        // 토큰 저장소에서 가져오기 (구현은 프로젝트에 따라 다름)
        return Task.FromResult(PlayerPrefs.GetString("access_token", ""));
    }

    public async Task<string> GetDataAsync(string endpoint, CancellationToken token)
    {
        // 요청만 하면 로깅, 인증, 재시도가 자동 적용
        var response = await client.GetAsync(endpoint, token);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }
}
```

---

## 6. HttpClient with async/await 패턴

### 기본 async/await 패턴

```csharp
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

public class HttpClientAsyncPatternsExample : MonoBehaviour
{
    private static readonly HttpClient client = new HttpClient
    {
        BaseAddress = new Uri("https://api.example.com/"),
        Timeout = TimeSpan.FromSeconds(30)
    };

    private CancellationTokenSource cts;

    private void Start()
    {
        cts = new CancellationTokenSource();
        _ = RunExamplesAsync(cts.Token);
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }

    // =============================================
    // 기본 GET/POST/PUT/DELETE
    // =============================================

    public async Task<T> GetAsync<T>(string endpoint, CancellationToken token)
    {
        HttpResponseMessage response = await client.GetAsync(endpoint, token);
        response.EnsureSuccessStatusCode();
        string json = await response.Content.ReadAsStringAsync();
        return JsonUtility.FromJson<T>(json);
    }

    public async Task<TResponse> PostAsync<TRequest, TResponse>(
        string endpoint, TRequest data, CancellationToken token)
    {
        string json = JsonUtility.ToJson(data);
        var content = new StringContent(json, Encoding.UTF8, "application/json");

        HttpResponseMessage response = await client.PostAsync(endpoint, content, token);
        response.EnsureSuccessStatusCode();
        string responseJson = await response.Content.ReadAsStringAsync();
        return JsonUtility.FromJson<TResponse>(responseJson);
    }

    public async Task PutAsync<T>(string endpoint, T data, CancellationToken token)
    {
        string json = JsonUtility.ToJson(data);
        var content = new StringContent(json, Encoding.UTF8, "application/json");

        HttpResponseMessage response = await client.PutAsync(endpoint, content, token);
        response.EnsureSuccessStatusCode();
    }

    public async Task DeleteAsync(string endpoint, CancellationToken token)
    {
        HttpResponseMessage response = await client.DeleteAsync(endpoint, token);
        response.EnsureSuccessStatusCode();
    }

    // =============================================
    // 병렬 요청 (WhenAll)
    // =============================================

    public async Task ParallelRequestsAsync(CancellationToken token)
    {
        // 여러 엔드포인트에 동시 요청
        Task<string>[] tasks = new[]
        {
            client.GetStringAsync("users/1"),
            client.GetStringAsync("users/2"),
            client.GetStringAsync("users/3"),
            client.GetStringAsync("settings"),
            client.GetStringAsync("notifications")
        };

        try
        {
            string[] results = await Task.WhenAll(tasks);

            for (int i = 0; i < results.Length; i++)
            {
                Debug.Log($"응답 {i}: {results[i].Substring(0, 50)}...");
            }
        }
        catch (Exception e)
        {
            Debug.LogError($"하나 이상의 요청 실패: {e.Message}");

            // 개별 Task 상태 확인
            for (int i = 0; i < tasks.Length; i++)
            {
                if (tasks[i].IsFaulted)
                    Debug.LogError($"  요청 {i} 실패: {tasks[i].Exception?.InnerException?.Message}");
                else if (tasks[i].IsCompleted)
                    Debug.Log($"  요청 {i} 성공");
            }
        }
    }

    // =============================================
    // 순차 요청 (의존 관계)
    // =============================================

    [Serializable]
    public class UserInfo
    {
        public string id;
        public string name;
    }

    [Serializable]
    public class UserProfile
    {
        public string bio;
        public int level;
    }

    public async Task SequentialRequestsAsync(CancellationToken token)
    {
        // 1단계: 사용자 정보 가져오기
        HttpResponseMessage userResponse = await client.GetAsync("users/me", token);
        userResponse.EnsureSuccessStatusCode();
        string userJson = await userResponse.Content.ReadAsStringAsync();
        var user = JsonUtility.FromJson<UserInfo>(userJson);

        // 2단계: 사용자 ID로 프로필 가져오기 (1단계에 의존)
        HttpResponseMessage profileResponse = await client.GetAsync(
            $"profiles/{user.id}", token);
        profileResponse.EnsureSuccessStatusCode();
        string profileJson = await profileResponse.Content.ReadAsStringAsync();
        var profile = JsonUtility.FromJson<UserProfile>(profileJson);

        Debug.Log($"사용자: {user.name}, 레벨: {profile.level}");
    }

    // =============================================
    // 스트리밍 응답 (대용량 데이터)
    // =============================================

    public async Task StreamResponseAsync(string url, CancellationToken token)
    {
        using HttpResponseMessage response = await client.GetAsync(
            url, HttpCompletionOption.ResponseHeadersRead, token);

        response.EnsureSuccessStatusCode();

        long? totalBytes = response.Content.Headers.ContentLength;

        using var stream = await response.Content.ReadAsStreamAsync();
        byte[] buffer = new byte[8192];
        long totalRead = 0;
        int bytesRead;

        while ((bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length, token)) > 0)
        {
            totalRead += bytesRead;

            // 진행률 계산
            if (totalBytes.HasValue)
            {
                float progress = (float)totalRead / totalBytes.Value;
                Debug.Log($"다운로드 진행률: {progress:P0}");
            }

            // 여기서 데이터 처리 (파일 저장, 파싱 등)
        }

        Debug.Log($"다운로드 완료: {totalRead} bytes");
    }

    // =============================================
    // 메인 스레드로 결과 전달
    // =============================================

    public async Task FetchAndUpdateUIAsync(CancellationToken token)
    {
        // 백그라운드 스레드에서 HTTP 요청
        string data = await client.GetStringAsync("game/state");

        // Unity 메인 스레드 컨텍스트가 캡처되어 있다면
        // await 이후 자동으로 메인 스레드로 복귀
        // (SynchronizationContext가 설정된 경우)

        // 메인 스레드에서 UI 업데이트
        Debug.Log($"게임 상태: {data}");
    }

    private async Task RunExamplesAsync(CancellationToken token)
    {
        try
        {
            await ParallelRequestsAsync(token);
            await SequentialRequestsAsync(token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("요청이 취소되었습니다.");
        }
        catch (Exception e)
        {
            Debug.LogError($"예외 발생: {e.Message}");
        }
    }
}
```

### UniTask와 함께 사용

```csharp
#if UNITASK_ENABLED
using System;
using System.Net.Http;
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine;

public class HttpClientWithUniTaskExample : MonoBehaviour
{
    private static readonly HttpClient client = new HttpClient
    {
        BaseAddress = new Uri("https://api.example.com/"),
        Timeout = TimeSpan.FromSeconds(30)
    };

    private async void Start()
    {
        var token = this.GetCancellationTokenOnDestroy();

        try
        {
            await FetchDataAsync(token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("GameObject 파괴로 인한 취소");
        }
    }

    // =============================================
    // UniTask로 HttpClient 사용
    // =============================================

    private async UniTask FetchDataAsync(CancellationToken token)
    {
        // HttpClient는 Task를 반환하지만 UniTask에서 await 가능
        HttpResponseMessage response = await client.GetAsync("data", token);
        response.EnsureSuccessStatusCode();
        string json = await response.Content.ReadAsStringAsync();

        // 메인 스레드로 전환 (UI 업데이트용)
        await UniTask.SwitchToMainThread(token);
        Debug.Log($"메인 스레드에서 처리: {json}");
    }

    // =============================================
    // UniTask.WhenAll로 병렬 HTTP 요청
    // =============================================

    private async UniTask<string[]> FetchMultipleAsync(
        string[] endpoints, CancellationToken token)
    {
        var tasks = new UniTask<string>[endpoints.Length];

        for (int i = 0; i < endpoints.Length; i++)
        {
            tasks[i] = FetchStringAsync(endpoints[i], token);
        }

        return await UniTask.WhenAll(tasks);
    }

    private async UniTask<string> FetchStringAsync(
        string endpoint, CancellationToken token)
    {
        var response = await client.GetAsync(endpoint, token);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }
}
#endif
```

---

## 7. Retry 패턴과 Polly

### 직접 구현한 Retry 유틸리티

```csharp
using System;
using System.Net;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// Unity 환경에서 사용하는 HTTP 재시도 유틸리티.
/// 지수 백오프와 지터를 적용하여 서버 부하를 분산합니다.
/// </summary>
public static class HttpRetryHelper
{
    // =============================================
    // 기본 재시도
    // =============================================

    public static async Task<HttpResponseMessage> SendWithRetryAsync(
        HttpClient client,
        Func<HttpRequestMessage> requestFactory,
        int maxRetries = 3,
        CancellationToken token = default)
    {
        HttpResponseMessage response = null;
        Exception lastException = null;

        for (int attempt = 0; attempt <= maxRetries; attempt++)
        {
            if (attempt > 0)
            {
                // 지수 백오프 + 지터
                TimeSpan delay = CalculateDelay(attempt);
                Debug.Log($"[Retry] 재시도 {attempt}/{maxRetries} " +
                          $"({delay.TotalSeconds:F1}초 후)");
                await Task.Delay(delay, token);
            }

            try
            {
                // HttpRequestMessage는 재사용 불가 → 팩토리 패턴 사용
                using HttpRequestMessage request = requestFactory();
                response = await client.SendAsync(request, token);

                if (response.IsSuccessStatusCode)
                    return response;

                if (!IsTransientError(response.StatusCode))
                {
                    Debug.LogWarning($"[Retry] 비일시적 에러: {(int)response.StatusCode}");
                    return response; // 재시도 의미 없음
                }

                // 429 Too Many Requests: Retry-After 헤더 존중
                if (response.StatusCode == HttpStatusCode.TooManyRequests)
                {
                    if (response.Headers.RetryAfter?.Delta.HasValue == true)
                    {
                        TimeSpan retryAfter = response.Headers.RetryAfter.Delta.Value;
                        Debug.Log($"[Retry] Retry-After: {retryAfter.TotalSeconds}초");
                        await Task.Delay(retryAfter, token);
                    }
                }

                Debug.LogWarning($"[Retry] 일시적 에러: {(int)response.StatusCode}");
            }
            catch (HttpRequestException e)
            {
                lastException = e;
                Debug.LogWarning($"[Retry] 네트워크 에러: {e.Message}");

                if (attempt == maxRetries)
                    throw;
            }
            catch (TaskCanceledException) when (!token.IsCancellationRequested)
            {
                lastException = new TimeoutException("HTTP 요청 타임아웃");
                Debug.LogWarning("[Retry] 타임아웃");

                if (attempt == maxRetries)
                    throw lastException;
            }
        }

        return response;
    }

    // =============================================
    // 지수 백오프 + 지터 계산
    // =============================================

    private static TimeSpan CalculateDelay(int attempt)
    {
        // 기본 지수 백오프: 2^attempt 초
        double baseDelay = Math.Pow(2, attempt);

        // 지터: 0~1 사이 랜덤 값을 곱해서 서버 부하 분산
        double jitter = UnityEngine.Random.Range(0f, 1f);
        double delay = baseDelay * (0.5 + jitter);

        // 최대 30초
        return TimeSpan.FromSeconds(Math.Min(delay, 30));
    }

    private static bool IsTransientError(HttpStatusCode code)
    {
        return code == HttpStatusCode.RequestTimeout ||
               code == HttpStatusCode.TooManyRequests ||
               (int)code >= 500;
    }
}

// =============================================
// 사용 예시
// =============================================

public class RetryUsageExample : MonoBehaviour
{
    private static readonly HttpClient client = new HttpClient
    {
        BaseAddress = new Uri("https://api.example.com/"),
        Timeout = TimeSpan.FromSeconds(10)
    };

    public async Task<string> FetchWithRetryAsync(CancellationToken token)
    {
        HttpResponseMessage response = await HttpRetryHelper.SendWithRetryAsync(
            client,
            requestFactory: () => new HttpRequestMessage(HttpMethod.Get, "data"),
            maxRetries: 3,
            token: token
        );

        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }
}
```

### Polly 라이브러리 활용

Polly는 .NET의 대표적인 복원력(Resilience) 라이브러리입니다. Unity에서 NuGet 패키지로 사용할 수 있습니다.

```csharp
// =============================================
// Polly 설치 (NuGetForUnity 또는 수동)
// =============================================
//
// 방법 1: NuGetForUnity 패키지 매니저로 "Polly" 검색 후 설치
// 방법 2: NuGet에서 Polly DLL 다운로드 후 Assets/Plugins에 배치
// 방법 3: manifest.json에 OpenUPM 레지스트리 추가

#if POLLY_ENABLED
using System;
using System.Net;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using Polly;
using Polly.Retry;
using Polly.Timeout;
using Polly.CircuitBreaker;
using Polly.Wrap;
using UnityEngine;

public class PollyIntegrationExample : MonoBehaviour
{
    private HttpClient client;
    private AsyncPolicyWrap<HttpResponseMessage> resiliencePolicy;

    private void Awake()
    {
        SetupPolicies();
        client = new HttpClient
        {
            BaseAddress = new Uri("https://api.example.com/"),
            Timeout = System.Threading.Timeout.InfiniteTimeSpan // Polly가 관리
        };
    }

    // =============================================
    // Polly 정책 설정
    // =============================================

    private void SetupPolicies()
    {
        // 1. 재시도 정책 (지수 백오프)
        AsyncRetryPolicy<HttpResponseMessage> retryPolicy =
            Policy<HttpResponseMessage>
                .HandleResult(r => IsTransientError(r.StatusCode))
                .Or<HttpRequestException>()
                .WaitAndRetryAsync(
                    retryCount: 3,
                    sleepDurationProvider: attempt =>
                        TimeSpan.FromSeconds(Math.Pow(2, attempt))
                        + TimeSpan.FromMilliseconds(UnityEngine.Random.Range(0, 1000)),
                    onRetry: (outcome, delay, attempt, context) =>
                    {
                        Debug.Log($"[Polly] 재시도 {attempt}: " +
                                  $"{delay.TotalSeconds:F1}초 후 " +
                                  $"(사유: {outcome.Result?.StatusCode ?? 0})");
                    }
                );

        // 2. 서킷 브레이커 정책
        AsyncCircuitBreakerPolicy<HttpResponseMessage> circuitBreakerPolicy =
            Policy<HttpResponseMessage>
                .HandleResult(r => !r.IsSuccessStatusCode)
                .Or<HttpRequestException>()
                .CircuitBreakerAsync(
                    handledEventsAllowedBeforeBreaking: 5,  // 5번 실패 시
                    durationOfBreak: TimeSpan.FromSeconds(30), // 30초 차단
                    onBreak: (outcome, duration) =>
                    {
                        Debug.LogWarning($"[Polly] 서킷 OPEN: " +
                                         $"{duration.TotalSeconds}초 동안 요청 차단");
                    },
                    onReset: () =>
                    {
                        Debug.Log("[Polly] 서킷 CLOSED: 정상 복구");
                    },
                    onHalfOpen: () =>
                    {
                        Debug.Log("[Polly] 서킷 HALF-OPEN: 테스트 요청 허용");
                    }
                );

        // 3. 타임아웃 정책
        AsyncTimeoutPolicy<HttpResponseMessage> timeoutPolicy =
            Policy.TimeoutAsync<HttpResponseMessage>(
                seconds: 10,
                timeoutStrategy: TimeoutStrategy.Optimistic,
                onTimeoutAsync: (context, timespan, task) =>
                {
                    Debug.LogWarning($"[Polly] 타임아웃: {timespan.TotalSeconds}초 초과");
                    return Task.CompletedTask;
                }
            );

        // 정책 조합: 타임아웃 → 서킷 브레이커 → 재시도
        // 바깥에서 안쪽 순서로 실행됨
        resiliencePolicy = Policy.WrapAsync(
            retryPolicy,
            circuitBreakerPolicy,
            timeoutPolicy
        );
    }

    // =============================================
    // Polly 정책 적용 요청
    // =============================================

    public async Task<string> ResilientGetAsync(string endpoint, CancellationToken token)
    {
        try
        {
            HttpResponseMessage response = await resiliencePolicy.ExecuteAsync(
                async ct =>
                {
                    var request = new HttpRequestMessage(HttpMethod.Get, endpoint);
                    return await client.SendAsync(request, ct);
                },
                token
            );

            response.EnsureSuccessStatusCode();
            return await response.Content.ReadAsStringAsync();
        }
        catch (BrokenCircuitException)
        {
            Debug.LogError("서킷 브레이커가 열려 있어 요청을 차단했습니다.");
            throw;
        }
        catch (TimeoutRejectedException)
        {
            Debug.LogError("Polly 타임아웃으로 요청이 거부되었습니다.");
            throw;
        }
    }

    private static bool IsTransientError(HttpStatusCode code)
    {
        return code == HttpStatusCode.RequestTimeout ||
               code == HttpStatusCode.TooManyRequests ||
               (int)code >= 500;
    }
}

// =============================================
// Polly + DelegatingHandler 통합
// =============================================

public class PollyHandler : DelegatingHandler
{
    private readonly IAsyncPolicy<HttpResponseMessage> policy;

    public PollyHandler(IAsyncPolicy<HttpResponseMessage> policy)
    {
        this.policy = policy;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        return await policy.ExecuteAsync(
            ct => base.SendAsync(request, ct),
            cancellationToken
        );
    }
}
#endif
```

---

## 8. IL2CPP/플랫폼별 주의사항

### 플랫폼별 호환성

```
┌──────────────────────────────────────────────────────────────────────┐
│                  플랫폼별 HttpClient 호환성                           │
├──────────────┬───────────────┬──────────────────────────────────────┤
│ 플랫폼        │ 지원 상태      │ 주의사항                             │
├──────────────┼───────────────┼──────────────────────────────────────┤
│ Windows       │ ✅ 완전 지원   │ 제한 없음                           │
│ macOS         │ ✅ 완전 지원   │ 제한 없음                           │
│ Linux         │ ✅ 완전 지원   │ 제한 없음                           │
│ Android       │ ⚠️ 조건부     │ IL2CPP 시 리플렉션 제한              │
│ iOS           │ ⚠️ 조건부     │ NSUrlSession 핸들러 권장             │
│ WebGL         │ ❌ 미지원      │ UnityWebRequest만 사용 가능          │
│ 콘솔(PS/Xbox) │ ❌ 미지원      │ 플랫폼 전용 API 사용                │
│ UWP           │ ⚠️ 조건부     │ Windows.Web.Http 고려               │
└──────────────┴───────────────┴──────────────────────────────────────┘
```

### IL2CPP 빌드 대응

```csharp
using System;
using System.Net.Http;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.Scripting;

// =============================================
// IL2CPP 빌드 시 주의사항
// =============================================

// 1. 제네릭 타입의 AOT 문제
// IL2CPP는 AOT(Ahead-of-Time) 컴파일이므로
// 런타임에 제네릭 타입을 생성할 수 없습니다.

// ✅ link.xml을 사용하여 필요한 타입 보존
// Assets/link.xml:
// <linker>
//   <assembly fullname="System.Net.Http">
//     <type fullname="System.Net.Http.HttpClientHandler" preserve="all"/>
//     <type fullname="System.Net.Http.HttpClient" preserve="all"/>
//     <type fullname="System.Net.Http.StringContent" preserve="all"/>
//     <type fullname="System.Net.Http.ByteArrayContent" preserve="all"/>
//     <type fullname="System.Net.Http.HttpResponseMessage" preserve="all"/>
//   </assembly>
//   <assembly fullname="System.Net.Primitives">
//     <type fullname="System.Net.CookieContainer" preserve="all"/>
//   </assembly>
//   <assembly fullname="System">
//     <type fullname="System.Net.DecompressionMethods" preserve="all"/>
//   </assembly>
// </linker>

public class IL2CPPSafeHttpClient : MonoBehaviour
{
    // =============================================
    // [Preserve] 어트리뷰트로 스트리핑 방지
    // =============================================

    [Preserve]
    [Serializable]
    public class ApiResponse
    {
        public int code;
        public string message;
        public string data;
    }

    [Preserve]
    [Serializable]
    public class ApiRequest
    {
        public string action;
        public string payload;
    }

    // =============================================
    // 플랫폼별 HttpClient 생성
    // =============================================

    public static HttpClient CreatePlatformClient()
    {
        HttpClientHandler handler;

#if UNITY_IOS && !UNITY_EDITOR
        // iOS: NSUrlSessionHandler가 가능하면 사용 (Xamarin/MAUI 환경)
        // Unity에서는 기본 HttpClientHandler 사용
        handler = new HttpClientHandler
        {
            // iOS에서 TLS 1.2 강제
            SslProtocols = System.Security.Authentication.SslProtocols.Tls12,
            MaxConnectionsPerServer = 6
        };
#elif UNITY_ANDROID && !UNITY_EDITOR
        // Android: 기본 핸들러 사용
        handler = new HttpClientHandler
        {
            // Android에서 TLS 설정
            SslProtocols = System.Security.Authentication.SslProtocols.Tls12,
            MaxConnectionsPerServer = 6,
            // Android에서는 프록시 이슈가 있을 수 있음
            UseProxy = false
        };
#elif UNITY_WEBGL && !UNITY_EDITOR
        // WebGL: HttpClient 사용 불가!
        // 컴파일 에러 방지를 위한 가드
        Debug.LogError("WebGL에서는 HttpClient를 사용할 수 없습니다. " +
                       "UnityWebRequest를 사용하세요.");
        handler = null;
        return null;
#else
        // Editor / Standalone
        handler = new HttpClientHandler
        {
            AutomaticDecompression =
                System.Net.DecompressionMethods.GZip |
                System.Net.DecompressionMethods.Deflate,
            MaxConnectionsPerServer = 10
        };
#endif

        if (handler == null) return null;

        return new HttpClient(handler)
        {
            Timeout = TimeSpan.FromSeconds(30)
        };
    }

    // =============================================
    // 플랫폼 안전 래퍼
    // =============================================

    /// <summary>
    /// 플랫폼에 따라 HttpClient 또는 UnityWebRequest를 사용하는 래퍼.
    /// WebGL에서는 자동으로 UnityWebRequest로 폴백합니다.
    /// </summary>
    public static async Task<string> SafeGetAsync(string url)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        // WebGL 폴백: UnityWebRequest를 Task로 래핑
        return await WebGLFallbackGetAsync(url);
#else
        using var client = CreatePlatformClient();
        if (client == null)
            throw new PlatformNotSupportedException(
                "이 플랫폼에서 HttpClient를 사용할 수 없습니다.");

        var response = await client.GetAsync(url);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
#endif
    }

#if UNITY_WEBGL
    private static Task<string> WebGLFallbackGetAsync(string url)
    {
        var tcs = new TaskCompletionSource<string>();

        // WebGL에서는 코루틴 기반 UnityWebRequest 사용
        // 실제 구현은 MonoBehaviour에서 코루틴으로 처리
        Debug.Log($"[WebGL] UnityWebRequest 폴백: {url}");

        return tcs.Task;
    }
#endif
}

// =============================================
// AOT 컴파일 힌트 (IL2CPP)
// =============================================

// IL2CPP에서 특정 제네릭 조합이 스트리핑되지 않도록 명시적 참조
internal static class AotTypeEnforcer
{
    // 이 메서드는 실제로 호출되지 않지만,
    // IL2CPP 컴파일러에 타입 힌트를 제공합니다.
    [Preserve]
    private static void EnsureTypes()
    {
        // HttpClient 관련 제네릭 타입 보존
        _ = typeof(Task<HttpResponseMessage>);
        _ = typeof(Task<string>);
        _ = typeof(Task<byte[]>);
    }
}
```

### Enter Play Mode Options 대응

```csharp
using System.Net.Http;
using UnityEngine;

/// <summary>
/// Unity의 Enter Play Mode Options (Domain Reload 비활성화) 시
/// static 필드가 초기화되지 않는 문제를 해결합니다.
/// </summary>
public static class SafeHttpClientManager
{
    private static HttpClient instance;

    public static HttpClient Instance
    {
        get
        {
            if (instance == null)
            {
                instance = new HttpClient
                {
                    Timeout = System.TimeSpan.FromSeconds(30)
                };
            }
            return instance;
        }
    }

    // Domain Reload 시 정적 필드 초기화
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    private static void Init()
    {
        instance?.Dispose();
        instance = null;
    }
}
```

---

## 9. 실전 통합 예제: 게임 API 클라이언트

```csharp
using System;
using System.Net;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 게임 서버 API 통신을 담당하는 종합 클라이언트.
/// 싱글턴 HttpClient, DelegatingHandler 파이프라인,
/// 재시도, 타임아웃, 인증을 모두 통합합니다.
/// </summary>
public class GameApiClient : IDisposable
{
    private readonly HttpClient httpClient;
    private readonly CancellationTokenSource disposeCts;
    private string accessToken;
    private bool disposed;

    public GameApiClient(string baseUrl)
    {
        disposeCts = new CancellationTokenSource();

        // 핸들러 파이프라인 구성
        var innerHandler = new HttpClientHandler
        {
            AutomaticDecompression =
                DecompressionMethods.GZip | DecompressionMethods.Deflate,
            MaxConnectionsPerServer = 10
        };

        var retryHandler = new RetryHandler(maxRetries: 3)
        {
            InnerHandler = innerHandler
        };

        var loggingHandler = new LoggingHandler
        {
            InnerHandler = retryHandler
        };

        httpClient = new HttpClient(loggingHandler)
        {
            BaseAddress = new Uri(baseUrl),
            Timeout = TimeSpan.FromSeconds(30)
        };

        httpClient.DefaultRequestHeaders.Accept.Add(
            new MediaTypeWithQualityHeaderValue("application/json"));
        httpClient.DefaultRequestHeaders.Add(
            "X-Client-Version", Application.version);
    }

    // =============================================
    // 인증
    // =============================================

    public void SetAccessToken(string token)
    {
        accessToken = token;
        httpClient.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token);
    }

    // =============================================
    // CRUD 메서드
    // =============================================

    public async Task<T> GetAsync<T>(string endpoint, CancellationToken token = default)
    {
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            token, disposeCts.Token);

        var response = await httpClient.GetAsync(endpoint, linkedCts.Token);
        return await HandleResponseAsync<T>(response);
    }

    public async Task<TResponse> PostAsync<TRequest, TResponse>(
        string endpoint, TRequest data, CancellationToken token = default)
    {
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            token, disposeCts.Token);

        string json = JsonUtility.ToJson(data);
        var content = new StringContent(json, Encoding.UTF8, "application/json");

        var response = await httpClient.PostAsync(endpoint, content, linkedCts.Token);
        return await HandleResponseAsync<TResponse>(response);
    }

    public async Task PostAsync<TRequest>(
        string endpoint, TRequest data, CancellationToken token = default)
    {
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            token, disposeCts.Token);

        string json = JsonUtility.ToJson(data);
        var content = new StringContent(json, Encoding.UTF8, "application/json");

        var response = await httpClient.PostAsync(endpoint, content, linkedCts.Token);
        response.EnsureSuccessStatusCode();
    }

    // =============================================
    // 응답 처리
    // =============================================

    private async Task<T> HandleResponseAsync<T>(HttpResponseMessage response)
    {
        string responseBody = await response.Content.ReadAsStringAsync();

        if (!response.IsSuccessStatusCode)
        {
            throw new GameApiException(
                (int)response.StatusCode,
                $"API 에러 [{(int)response.StatusCode}]: {responseBody}"
            );
        }

        try
        {
            return JsonUtility.FromJson<T>(responseBody);
        }
        catch (Exception e)
        {
            throw new GameApiException(0, $"JSON 파싱 실패: {e.Message}");
        }
    }

    // =============================================
    // 정리
    // =============================================

    public void Dispose()
    {
        if (!disposed)
        {
            disposed = true;
            disposeCts.Cancel();
            disposeCts.Dispose();
            httpClient.Dispose();
        }
    }
}

// =============================================
// 커스텀 예외
// =============================================

public class GameApiException : Exception
{
    public int StatusCode { get; }

    public GameApiException(int statusCode, string message)
        : base(message)
    {
        StatusCode = statusCode;
    }
}

// =============================================
// MonoBehaviour에서 사용
// =============================================

public class GameApiClientUsageExample : MonoBehaviour
{
    private GameApiClient apiClient;
    private CancellationTokenSource cts;

    [Serializable]
    public class PlayerData
    {
        public string id;
        public string name;
        public int score;
    }

    [Serializable]
    public class ScoreSubmission
    {
        public int score;
        public string level;
    }

    [Serializable]
    public class ScoreResponse
    {
        public bool success;
        public int rank;
    }

    private void Awake()
    {
        apiClient = new GameApiClient("https://api.mygame.com/v1/");
        cts = new CancellationTokenSource();
    }

    private async void Start()
    {
        try
        {
            // 인증 토큰 설정
            apiClient.SetAccessToken("my-auth-token");

            // 플레이어 데이터 조회
            PlayerData player = await apiClient.GetAsync<PlayerData>(
                "players/me", cts.Token);
            Debug.Log($"플레이어: {player.name}, 점수: {player.score}");

            // 점수 제출
            var submission = new ScoreSubmission
            {
                score = 9999,
                level = "boss-stage"
            };

            ScoreResponse result = await apiClient.PostAsync<ScoreSubmission, ScoreResponse>(
                "scores", submission, cts.Token);
            Debug.Log($"점수 제출 결과: 성공={result.success}, 순위={result.rank}");
        }
        catch (GameApiException e)
        {
            Debug.LogError($"API 에러 [{e.StatusCode}]: {e.Message}");
        }
        catch (OperationCanceledException)
        {
            Debug.Log("요청이 취소되었습니다.");
        }
        catch (Exception e)
        {
            Debug.LogError($"예외: {e.Message}");
        }
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
        apiClient?.Dispose();
    }
}
```

---

## 주의사항

1. **소켓 고갈 방지**: `HttpClient`는 반드시 싱글턴 또는 정적 인스턴스로 재사용합니다. 요청마다 새 인스턴스를 생성하면 `TIME_WAIT` 소켓이 누적되어 시스템 리소스를 고갈시킵니다.

2. **WebGL 미지원**: WebGL 빌드에서 `System.Net.Http.HttpClient`는 사용할 수 없습니다. `UnityWebRequest`로 폴백하는 로직이 필요합니다.

3. **IL2CPP 스트리핑**: IL2CPP 빌드 시 `link.xml`에 `System.Net.Http` 관련 타입을 명시하지 않으면 런타임에 `TypeLoadException`이 발생할 수 있습니다.

4. **메인 스레드 주의**: `HttpClient`의 응답 처리는 백그라운드 스레드에서 실행될 수 있습니다. Unity API(`GameObject`, `Transform` 등)를 호출하려면 반드시 메인 스레드로 전환해야 합니다.

5. **타임아웃과 취소 구분**: `HttpClient.Timeout` 초과 시 `TaskCanceledException`이 발생합니다. 외부 `CancellationToken`에 의한 취소와 구분하려면 `token.IsCancellationRequested`를 확인합니다.

6. **HttpRequestMessage 재사용 불가**: 한번 전송한 `HttpRequestMessage`는 재사용할 수 없습니다. 재시도 로직에서는 팩토리 패턴으로 매번 새 인스턴스를 생성해야 합니다.

7. **DefaultRequestHeaders 스레드 안전**: `HttpClient.DefaultRequestHeaders`는 스레드 안전하지 않습니다. 초기화 후에는 수정하지 않거나, 개별 요청의 `HttpRequestMessage.Headers`를 사용합니다.

8. **DNS 변경 미반영**: 싱글턴 `HttpClient`는 DNS 결과를 캐싱합니다. 서버 IP가 변경되면 반영되지 않을 수 있으므로, `SocketsHttpHandler.PooledConnectionLifetime`을 설정하거나 `HttpClientHandler`의 연결 수명을 관리합니다.

---

## 베스트 프랙티스

### 1. HttpClient 수명 관리

```csharp
// ✅ 정적 싱글턴으로 애플리케이션 수명 동안 유지
private static readonly HttpClient client = new HttpClient();

// ❌ 메서드 내에서 생성/폐기
public async Task BadAsync()
{
    using var client = new HttpClient(); // 소켓 고갈!
    await client.GetAsync("...");
}
```

### 2. 취소 토큰 항상 전달

```csharp
// ✅ CancellationToken을 모든 비동기 메서드에 전달
public async Task<string> GetDataAsync(CancellationToken token)
{
    var response = await client.GetAsync("data", token);
    return await response.Content.ReadAsStringAsync();
}

// ❌ 취소 토큰 없이 호출
public async Task<string> GetDataAsync()
{
    var response = await client.GetAsync("data"); // 취소 불가!
    return await response.Content.ReadAsStringAsync();
}
```

### 3. 응답 상태 확인

```csharp
// ✅ EnsureSuccessStatusCode 또는 명시적 상태 확인
var response = await client.GetAsync("data", token);
if (!response.IsSuccessStatusCode)
{
    string body = await response.Content.ReadAsStringAsync();
    Debug.LogError($"[{(int)response.StatusCode}] {body}");
    return;
}

// ❌ 상태 코드 무시
var response = await client.GetAsync("data");
string result = await response.Content.ReadAsStringAsync(); // 에러 본문일 수도!
```

### 4. 대용량 응답은 스트리밍

```csharp
// ✅ ResponseHeadersRead로 메모리 절약
var response = await client.GetAsync(url,
    HttpCompletionOption.ResponseHeadersRead, token);
using var stream = await response.Content.ReadAsStreamAsync();
// 스트림에서 점진적으로 읽기

// ❌ 전체 응답을 메모리에 로드
string huge = await client.GetStringAsync(url); // 수백 MB 응답 시 OOM 위험
```

### 5. 플랫폼 분기

```csharp
// ✅ 플랫폼별 적절한 API 선택
public interface IHttpService
{
    Task<string> GetAsync(string url, CancellationToken token);
}

// Standalone/Mobile: HttpClient 구현
// WebGL: UnityWebRequest 구현
// 런타임에 적절한 구현체를 주입
```

---

## 참고 자료

- [Microsoft: HttpClient 사용 지침](https://learn.microsoft.com/dotnet/fundamentals/networking/http/httpclient-guidelines)
- [Microsoft: IHttpClientFactory를 사용한 HTTP 요청](https://learn.microsoft.com/dotnet/core/extensions/httpclient-factory)
- [Polly: .NET 복원력 라이브러리](https://github.com/App-vNext/Polly)
- [Unity: IL2CPP 빌드 가이드](https://docs.unity3d.com/Manual/IL2CPP.html)
- [Unity: 관리되는 코드 스트리핑](https://docs.unity3d.com/Manual/ManagedCodeStripping.html)

---

## 다음 섹션

[27. WebSocket](../10-protocols/27-websocket.md)
