# 25. UnityWebRequest

## 개요

`UnityWebRequest`는 Unity에서 HTTP/HTTPS 통신을 위한 네이티브 API입니다. Coroutine, async/await, UniTask와 함께 사용할 수 있으며, 다양한 플랫폼에서 일관된 동작을 제공합니다.

---

## 1. UnityWebRequest 기본 사용법

### GET 요청

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;

public class UnityWebRequestBasicsExample : MonoBehaviour
{
    private void Start()
    {
        // Coroutine 방식
        StartCoroutine(GetRequestCoroutine());
    }

    // =============================================
    // Coroutine을 이용한 GET 요청
    // =============================================

    private IEnumerator GetRequestCoroutine()
    {
        string url = "https://httpbin.org/get";

        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            // 요청 전송 및 완료 대기
            yield return request.SendWebRequest();

            // 결과 확인
            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"응답 코드: {request.responseCode}");
                Debug.Log($"응답 본문: {request.downloadHandler.text}");

                // 응답 헤더
                Debug.Log($"Content-Type: {request.GetResponseHeader("Content-Type")}");
            }
            else
            {
                Debug.LogError($"에러: {request.error}");
                Debug.LogError($"결과: {request.result}");
            }
        }
    }

    // =============================================
    // async/await 방식 (Unity 2023+ 또는 UniTask)
    // =============================================

    #if UNITY_2023_1_OR_NEWER
    private async void GetRequestAsync()
    {
        string url = "https://httpbin.org/get";

        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            await request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"응답: {request.downloadHandler.text}");
            }
            else
            {
                Debug.LogError($"에러: {request.error}");
            }
        }
    }
    #endif
}
```

### POST 요청

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;
using System.Text;

public class PostRequestExample : MonoBehaviour
{
    private void Start()
    {
        // 다양한 POST 방식
        StartCoroutine(PostFormData());
        StartCoroutine(PostJson());
        StartCoroutine(PostRawData());
    }

    // =============================================
    // Form 데이터 POST
    // =============================================

    private IEnumerator PostFormData()
    {
        string url = "https://httpbin.org/post";

        WWWForm form = new WWWForm();
        form.AddField("username", "testuser");
        form.AddField("password", "testpass");
        form.AddField("level", 42);

        using (UnityWebRequest request = UnityWebRequest.Post(url, form))
        {
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"Form POST 응답: {request.downloadHandler.text}");
            }
            else
            {
                Debug.LogError($"에러: {request.error}");
            }
        }
    }

    // =============================================
    // JSON POST
    // =============================================

    [System.Serializable]
    public class UserData
    {
        public string name;
        public int age;
        public string email;
    }

    private IEnumerator PostJson()
    {
        string url = "https://httpbin.org/post";

        // 데이터 객체 생성
        UserData data = new UserData
        {
            name = "홍길동",
            age = 25,
            email = "hong@example.com"
        };

        // JSON 직렬화
        string json = JsonUtility.ToJson(data);
        byte[] bodyRaw = Encoding.UTF8.GetBytes(json);

        using (UnityWebRequest request = new UnityWebRequest(url, "POST"))
        {
            request.uploadHandler = new UploadHandlerRaw(bodyRaw);
            request.downloadHandler = new DownloadHandlerBuffer();
            request.SetRequestHeader("Content-Type", "application/json");

            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"JSON POST 응답: {request.downloadHandler.text}");
            }
            else
            {
                Debug.LogError($"에러: {request.error}");
            }
        }
    }

    // =============================================
    // Raw 바이너리 데이터 POST
    // =============================================

    private IEnumerator PostRawData()
    {
        string url = "https://httpbin.org/post";

        byte[] data = new byte[] { 0x01, 0x02, 0x03, 0x04 };

        using (UnityWebRequest request = new UnityWebRequest(url, "POST"))
        {
            request.uploadHandler = new UploadHandlerRaw(data);
            request.downloadHandler = new DownloadHandlerBuffer();
            request.SetRequestHeader("Content-Type", "application/octet-stream");

            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"Raw POST 응답 크기: {request.downloadHandler.data.Length} bytes");
            }
        }
    }
}
```

---

## 2. PUT, DELETE, PATCH 요청

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;
using System.Text;

public class HttpMethodsExample : MonoBehaviour
{
    private void Start()
    {
        StartCoroutine(DemonstrateHttpMethods());
    }

    private IEnumerator DemonstrateHttpMethods()
    {
        // PUT
        yield return PutRequest();

        // DELETE
        yield return DeleteRequest();

        // PATCH
        yield return PatchRequest();

        // HEAD
        yield return HeadRequest();
    }

    // =============================================
    // PUT 요청
    // =============================================

    private IEnumerator PutRequest()
    {
        string url = "https://httpbin.org/put";
        string json = "{\"name\":\"updated\",\"value\":123}";

        using (UnityWebRequest request = UnityWebRequest.Put(url, json))
        {
            request.SetRequestHeader("Content-Type", "application/json");

            yield return request.SendWebRequest();

            Debug.Log($"PUT 응답: {request.result}");
        }
    }

    // =============================================
    // DELETE 요청
    // =============================================

    private IEnumerator DeleteRequest()
    {
        string url = "https://httpbin.org/delete";

        using (UnityWebRequest request = UnityWebRequest.Delete(url))
        {
            yield return request.SendWebRequest();

            Debug.Log($"DELETE 응답: {request.result}");
        }
    }

    // =============================================
    // PATCH 요청 (커스텀 메서드)
    // =============================================

    private IEnumerator PatchRequest()
    {
        string url = "https://httpbin.org/patch";
        string json = "{\"field\":\"patched\"}";
        byte[] bodyRaw = Encoding.UTF8.GetBytes(json);

        using (UnityWebRequest request = new UnityWebRequest(url, "PATCH"))
        {
            request.uploadHandler = new UploadHandlerRaw(bodyRaw);
            request.downloadHandler = new DownloadHandlerBuffer();
            request.SetRequestHeader("Content-Type", "application/json");

            yield return request.SendWebRequest();

            Debug.Log($"PATCH 응답: {request.result}");
        }
    }

    // =============================================
    // HEAD 요청 (본문 없이 헤더만)
    // =============================================

    private IEnumerator HeadRequest()
    {
        string url = "https://httpbin.org/get";

        using (UnityWebRequest request = UnityWebRequest.Head(url))
        {
            yield return request.SendWebRequest();

            Debug.Log($"HEAD 응답 코드: {request.responseCode}");
            Debug.Log($"Content-Length: {request.GetResponseHeader("Content-Length")}");
        }
    }
}
```

---

## 3. 파일 다운로드

### 텍스처, 오디오, 에셋번들

```csharp
using UnityEngine;
using UnityEngine.Networking;
using UnityEngine.UI;
using System.Collections;
using System.IO;

public class FileDownloadExample : MonoBehaviour
{
    [SerializeField] private RawImage targetImage;
    [SerializeField] private AudioSource audioSource;

    // =============================================
    // 텍스처 다운로드
    // =============================================

    public IEnumerator DownloadTexture(string url)
    {
        using (UnityWebRequest request = UnityWebRequestTexture.GetTexture(url))
        {
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Texture2D texture = DownloadHandlerTexture.GetContent(request);
                Debug.Log($"텍스처 다운로드 완료: {texture.width}x{texture.height}");

                if (targetImage != null)
                {
                    targetImage.texture = texture;
                }
            }
            else
            {
                Debug.LogError($"텍스처 다운로드 실패: {request.error}");
            }
        }
    }

    // =============================================
    // 오디오 클립 다운로드
    // =============================================

    public IEnumerator DownloadAudioClip(string url, AudioType audioType = AudioType.MPEG)
    {
        using (UnityWebRequest request = UnityWebRequestMultimedia.GetAudioClip(url, audioType))
        {
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                AudioClip clip = DownloadHandlerAudioClip.GetContent(request);
                Debug.Log($"오디오 다운로드 완료: {clip.length}초");

                if (audioSource != null)
                {
                    audioSource.clip = clip;
                    audioSource.Play();
                }
            }
            else
            {
                Debug.LogError($"오디오 다운로드 실패: {request.error}");
            }
        }
    }

    // =============================================
    // 에셋번들 다운로드
    // =============================================

    public IEnumerator DownloadAssetBundle(string url)
    {
        using (UnityWebRequest request = UnityWebRequestAssetBundle.GetAssetBundle(url))
        {
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                AssetBundle bundle = DownloadHandlerAssetBundle.GetContent(request);
                Debug.Log($"에셋번들 다운로드 완료: {bundle.name}");

                // 에셋 로드
                // var prefab = bundle.LoadAsset<GameObject>("MyPrefab");

                // 사용 후 해제
                // bundle.Unload(false);
            }
            else
            {
                Debug.LogError($"에셋번들 다운로드 실패: {request.error}");
            }
        }
    }

    // =============================================
    // 파일로 직접 저장
    // =============================================

    public IEnumerator DownloadFile(string url, string localPath)
    {
        using (UnityWebRequest request = new UnityWebRequest(url, "GET"))
        {
            // 파일로 직접 저장하는 다운로드 핸들러
            request.downloadHandler = new DownloadHandlerFile(localPath);

            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"파일 저장 완료: {localPath}");
                Debug.Log($"파일 크기: {new FileInfo(localPath).Length} bytes");
            }
            else
            {
                Debug.LogError($"파일 다운로드 실패: {request.error}");
            }
        }
    }
}
```

---

## 4. 파일 업로드

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;
using System.Collections.Generic;
using System.IO;

public class FileUploadExample : MonoBehaviour
{
    // =============================================
    // 단일 파일 업로드
    // =============================================

    public IEnumerator UploadFile(string url, string filePath)
    {
        byte[] fileData = File.ReadAllBytes(filePath);
        string fileName = Path.GetFileName(filePath);

        List<IMultipartFormSection> formData = new List<IMultipartFormSection>
        {
            new MultipartFormFileSection("file", fileData, fileName, "application/octet-stream")
        };

        using (UnityWebRequest request = UnityWebRequest.Post(url, formData))
        {
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"업로드 성공: {request.downloadHandler.text}");
            }
            else
            {
                Debug.LogError($"업로드 실패: {request.error}");
            }
        }
    }

    // =============================================
    // 여러 파일 및 필드 업로드
    // =============================================

    public IEnumerator UploadMultiple(string url, Dictionary<string, byte[]> files, Dictionary<string, string> fields)
    {
        List<IMultipartFormSection> formData = new List<IMultipartFormSection>();

        // 텍스트 필드 추가
        foreach (var field in fields)
        {
            formData.Add(new MultipartFormDataSection(field.Key, field.Value));
        }

        // 파일 추가
        foreach (var file in files)
        {
            formData.Add(new MultipartFormFileSection("files", file.Value, file.Key, "application/octet-stream"));
        }

        using (UnityWebRequest request = UnityWebRequest.Post(url, formData))
        {
            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log("다중 업로드 성공");
            }
            else
            {
                Debug.LogError($"업로드 실패: {request.error}");
            }
        }
    }

    // =============================================
    // 스트리밍 업로드 (대용량 파일)
    // =============================================

    public IEnumerator UploadLargeFile(string url, string filePath)
    {
        using (UnityWebRequest request = new UnityWebRequest(url, "POST"))
        {
            // 파일에서 직접 스트리밍
            request.uploadHandler = new UploadHandlerFile(filePath);
            request.downloadHandler = new DownloadHandlerBuffer();
            request.SetRequestHeader("Content-Type", "application/octet-stream");

            // 진행률 모니터링을 위한 비동기 전송
            var operation = request.SendWebRequest();

            while (!operation.isDone)
            {
                Debug.Log($"업로드 진행률: {request.uploadProgress:P0}");
                yield return null;
            }

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log("대용량 파일 업로드 성공");
            }
            else
            {
                Debug.LogError($"업로드 실패: {request.error}");
            }
        }
    }
}
```

---

## 5. 진행률 및 타임아웃

```csharp
using UnityEngine;
using UnityEngine.Networking;
using UnityEngine.UI;
using System.Collections;

public class ProgressAndTimeoutExample : MonoBehaviour
{
    [SerializeField] private Slider progressBar;
    [SerializeField] private Text progressText;

    // =============================================
    // 다운로드 진행률
    // =============================================

    public IEnumerator DownloadWithProgress(string url)
    {
        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            var operation = request.SendWebRequest();

            while (!operation.isDone)
            {
                float progress = request.downloadProgress;

                // UI 업데이트
                if (progressBar != null)
                    progressBar.value = progress;

                if (progressText != null)
                    progressText.text = $"다운로드: {progress:P0}";

                Debug.Log($"다운로드 진행률: {progress:P0}");

                yield return null;
            }

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log($"다운로드 완료: {request.downloadHandler.data.Length} bytes");
            }
        }
    }

    // =============================================
    // 타임아웃 설정
    // =============================================

    public IEnumerator RequestWithTimeout(string url, int timeoutSeconds = 10)
    {
        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            // 타임아웃 설정 (초 단위)
            request.timeout = timeoutSeconds;

            yield return request.SendWebRequest();

            switch (request.result)
            {
                case UnityWebRequest.Result.Success:
                    Debug.Log("요청 성공");
                    break;

                case UnityWebRequest.Result.ConnectionError:
                    // 타임아웃 시 ConnectionError 발생
                    if (request.error.Contains("Request timeout"))
                    {
                        Debug.LogError($"타임아웃! ({timeoutSeconds}초 초과)");
                    }
                    else
                    {
                        Debug.LogError($"연결 에러: {request.error}");
                    }
                    break;

                case UnityWebRequest.Result.ProtocolError:
                    Debug.LogError($"프로토콜 에러: {request.responseCode}");
                    break;

                case UnityWebRequest.Result.DataProcessingError:
                    Debug.LogError($"데이터 처리 에러: {request.error}");
                    break;
            }
        }
    }

    // =============================================
    // 수동 타임아웃 (Coroutine 기반)
    // =============================================

    public IEnumerator RequestWithManualTimeout(string url, float timeoutSeconds)
    {
        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            var operation = request.SendWebRequest();
            float startTime = Time.realtimeSinceStartup;

            while (!operation.isDone)
            {
                // 타임아웃 체크
                if (Time.realtimeSinceStartup - startTime > timeoutSeconds)
                {
                    request.Abort();
                    Debug.LogError("수동 타임아웃!");
                    yield break;
                }

                yield return null;
            }

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log("요청 성공");
            }
        }
    }
}
```

---

## 6. 인증 및 헤더

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System;
using System.Collections;
using System.Text;

public class AuthenticationExample : MonoBehaviour
{
    // =============================================
    // 커스텀 헤더
    // =============================================

    public IEnumerator RequestWithCustomHeaders(string url)
    {
        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            // 일반 헤더 설정
            request.SetRequestHeader("Accept", "application/json");
            request.SetRequestHeader("Accept-Language", "ko-KR");
            request.SetRequestHeader("X-Custom-Header", "custom-value");

            yield return request.SendWebRequest();

            Debug.Log($"응답: {request.downloadHandler.text}");
        }
    }

    // =============================================
    // Bearer Token 인증
    // =============================================

    public IEnumerator RequestWithBearerToken(string url, string token)
    {
        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            request.SetRequestHeader("Authorization", $"Bearer {token}");

            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log("인증 성공");
            }
            else if (request.responseCode == 401)
            {
                Debug.LogError("인증 실패: 토큰이 유효하지 않음");
            }
        }
    }

    // =============================================
    // Basic 인증
    // =============================================

    public IEnumerator RequestWithBasicAuth(string url, string username, string password)
    {
        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            // Base64로 인코딩
            string credentials = $"{username}:{password}";
            string base64Credentials = Convert.ToBase64String(Encoding.UTF8.GetBytes(credentials));

            request.SetRequestHeader("Authorization", $"Basic {base64Credentials}");

            yield return request.SendWebRequest();

            Debug.Log($"Basic Auth 응답: {request.result}");
        }
    }

    // =============================================
    // API Key 인증
    // =============================================

    public IEnumerator RequestWithApiKey(string url, string apiKey)
    {
        // 쿼리 파라미터로
        string urlWithKey = $"{url}?api_key={apiKey}";

        using (UnityWebRequest request = UnityWebRequest.Get(urlWithKey))
        {
            // 또는 헤더로
            request.SetRequestHeader("X-API-Key", apiKey);

            yield return request.SendWebRequest();

            Debug.Log($"API Key 응답: {request.result}");
        }
    }
}
```

---

## 7. HTTPS 및 인증서

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;

public class HttpsCertificateExample : MonoBehaviour
{
    // =============================================
    // 인증서 무시 (개발용 - 프로덕션에서 사용 금지!)
    // =============================================

    public IEnumerator RequestIgnoringCertificate(string url)
    {
        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            // 모든 인증서 허용 (보안 위험!)
            request.certificateHandler = new AcceptAllCertificates();

            yield return request.SendWebRequest();

            Debug.Log($"응답: {request.result}");
        }
    }

    // 모든 인증서 허용 핸들러 (개발용)
    private class AcceptAllCertificates : CertificateHandler
    {
        protected override bool ValidateCertificate(byte[] certificateData)
        {
            // 항상 true 반환 - 모든 인증서 허용
            // ⚠️ 프로덕션에서 절대 사용 금지!
            return true;
        }
    }

    // =============================================
    // 특정 인증서만 허용 (Certificate Pinning)
    // =============================================

    [SerializeField] private string expectedCertificateHash;

    public IEnumerator RequestWithCertificatePinning(string url)
    {
        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            request.certificateHandler = new PinnedCertificateHandler(expectedCertificateHash);

            yield return request.SendWebRequest();

            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log("인증서 검증 성공");
            }
            else
            {
                Debug.LogError($"요청 실패: {request.error}");
            }
        }
    }

    private class PinnedCertificateHandler : CertificateHandler
    {
        private string expectedHash;

        public PinnedCertificateHandler(string hash)
        {
            expectedHash = hash;
        }

        protected override bool ValidateCertificate(byte[] certificateData)
        {
            // 인증서 해시 비교
            string actualHash = ComputeHash(certificateData);
            return actualHash == expectedHash;
        }

        private string ComputeHash(byte[] data)
        {
            using (var sha256 = System.Security.Cryptography.SHA256.Create())
            {
                byte[] hash = sha256.ComputeHash(data);
                return System.BitConverter.ToString(hash).Replace("-", "").ToLowerInvariant();
            }
        }
    }
}
```

---

## 8. 재시도 및 에러 처리

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System;
using System.Collections;

public class RetryAndErrorHandlingExample : MonoBehaviour
{
    // =============================================
    // 자동 재시도
    // =============================================

    public IEnumerator RequestWithRetry(string url, int maxRetries = 3, float retryDelay = 1f)
    {
        int attempt = 0;
        UnityWebRequest.Result lastResult = UnityWebRequest.Result.InProgress;
        string lastError = "";

        while (attempt < maxRetries)
        {
            attempt++;
            Debug.Log($"요청 시도 {attempt}/{maxRetries}");

            using (UnityWebRequest request = UnityWebRequest.Get(url))
            {
                request.timeout = 10;

                yield return request.SendWebRequest();

                lastResult = request.result;
                lastError = request.error;

                if (request.result == UnityWebRequest.Result.Success)
                {
                    Debug.Log($"성공 (시도 {attempt})");
                    Debug.Log($"응답: {request.downloadHandler.text}");
                    yield break;
                }

                // 재시도 가능한 에러인지 확인
                if (!IsRetryable(request))
                {
                    Debug.LogError($"재시도 불가능한 에러: {request.error}");
                    yield break;
                }

                Debug.LogWarning($"시도 {attempt} 실패: {request.error}. {retryDelay}초 후 재시도...");
            }

            yield return new WaitForSeconds(retryDelay);

            // 지수 백오프 (선택적)
            retryDelay *= 2;
        }

        Debug.LogError($"최대 재시도 횟수 초과. 마지막 에러: {lastError}");
    }

    private bool IsRetryable(UnityWebRequest request)
    {
        // 네트워크 에러는 재시도 가능
        if (request.result == UnityWebRequest.Result.ConnectionError)
            return true;

        // 5xx 서버 에러는 재시도 가능
        if (request.responseCode >= 500 && request.responseCode < 600)
            return true;

        // 429 Too Many Requests는 재시도 가능
        if (request.responseCode == 429)
            return true;

        // 4xx 클라이언트 에러는 재시도해도 동일한 결과
        return false;
    }

    // =============================================
    // 상세 에러 처리
    // =============================================

    public IEnumerator RequestWithDetailedErrorHandling(string url)
    {
        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            yield return request.SendWebRequest();

            switch (request.result)
            {
                case UnityWebRequest.Result.Success:
                    HandleSuccess(request);
                    break;

                case UnityWebRequest.Result.ConnectionError:
                    HandleConnectionError(request);
                    break;

                case UnityWebRequest.Result.ProtocolError:
                    HandleProtocolError(request);
                    break;

                case UnityWebRequest.Result.DataProcessingError:
                    HandleDataProcessingError(request);
                    break;
            }
        }
    }

    private void HandleSuccess(UnityWebRequest request)
    {
        Debug.Log($"성공: {request.downloadHandler.text}");
    }

    private void HandleConnectionError(UnityWebRequest request)
    {
        if (request.error.Contains("timeout"))
        {
            Debug.LogError("연결 타임아웃. 네트워크 상태를 확인하세요.");
        }
        else if (request.error.Contains("Cannot resolve"))
        {
            Debug.LogError("DNS 해석 실패. 인터넷 연결을 확인하세요.");
        }
        else if (request.error.Contains("Unable to connect"))
        {
            Debug.LogError("서버에 연결할 수 없습니다.");
        }
        else
        {
            Debug.LogError($"연결 에러: {request.error}");
        }
    }

    private void HandleProtocolError(UnityWebRequest request)
    {
        long code = request.responseCode;

        switch (code)
        {
            case 400:
                Debug.LogError("잘못된 요청 (400)");
                break;
            case 401:
                Debug.LogError("인증 필요 (401)");
                break;
            case 403:
                Debug.LogError("접근 거부 (403)");
                break;
            case 404:
                Debug.LogError("리소스를 찾을 수 없음 (404)");
                break;
            case 429:
                Debug.LogError("요청 과다 (429). 잠시 후 다시 시도하세요.");
                break;
            case 500:
                Debug.LogError("서버 내부 오류 (500)");
                break;
            case 502:
                Debug.LogError("게이트웨이 오류 (502)");
                break;
            case 503:
                Debug.LogError("서비스 일시 중단 (503)");
                break;
            default:
                Debug.LogError($"HTTP 오류: {code}");
                break;
        }
    }

    private void HandleDataProcessingError(UnityWebRequest request)
    {
        Debug.LogError($"데이터 처리 오류: {request.error}");
    }
}
```

---

## 9. UniTask와 함께 사용

```csharp
using UnityEngine;
using UnityEngine.Networking;
using Cysharp.Threading.Tasks;
using System;
using System.Threading;

public class UnityWebRequestWithUniTaskExample : MonoBehaviour
{
    private async void Start()
    {
        var token = this.GetCancellationTokenOnDestroy();

        try
        {
            await DemonstrateUniTaskUsage(token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("요청 취소됨");
        }
    }

    private async UniTask DemonstrateUniTaskUsage(CancellationToken token)
    {
        // =============================================
        // 기본 사용
        // =============================================

        string response = await UnityWebRequest.Get("https://httpbin.org/get")
            .SendWebRequest()
            .WithCancellation(token);

        Debug.Log($"응답: {response.Substring(0, 100)}");

        // =============================================
        // 타임아웃
        // =============================================

        try
        {
            using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
            using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(token, timeoutCts.Token);

            await UnityWebRequest.Get("https://httpbin.org/delay/10")
                .SendWebRequest()
                .WithCancellation(linkedCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("타임아웃!");
        }

        // =============================================
        // 병렬 요청
        // =============================================

        var results = await UniTask.WhenAll(
            FetchAsync("https://httpbin.org/get", token),
            FetchAsync("https://httpbin.org/ip", token),
            FetchAsync("https://httpbin.org/user-agent", token)
        );

        Debug.Log($"병렬 요청 완료: {results.Length}개");
    }

    private async UniTask<string> FetchAsync(string url, CancellationToken token)
    {
        return await UnityWebRequest.Get(url)
            .SendWebRequest()
            .WithCancellation(token);
    }
}
```

---

## 10. HTTP 클라이언트 래퍼

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System;
using System.Collections;
using System.Text;
using System.Collections.Generic;

/// <summary>
/// UnityWebRequest를 감싼 편의 클래스
/// </summary>
public class HttpClient : MonoBehaviour
{
    private static HttpClient instance;
    public static HttpClient Instance => instance;

    private string baseUrl = "";
    private Dictionary<string, string> defaultHeaders = new Dictionary<string, string>();
    private int defaultTimeout = 30;

    private void Awake()
    {
        if (instance == null)
        {
            instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
        }
    }

    public void Configure(string baseUrl, int timeout = 30)
    {
        this.baseUrl = baseUrl;
        this.defaultTimeout = timeout;
    }

    public void SetDefaultHeader(string key, string value)
    {
        defaultHeaders[key] = value;
    }

    // =============================================
    // GET
    // =============================================

    public void Get(string endpoint, Action<string> onSuccess, Action<string> onError)
    {
        StartCoroutine(GetCoroutine(endpoint, onSuccess, onError));
    }

    private IEnumerator GetCoroutine(string endpoint, Action<string> onSuccess, Action<string> onError)
    {
        string url = baseUrl + endpoint;

        using (UnityWebRequest request = UnityWebRequest.Get(url))
        {
            ApplyDefaults(request);

            yield return request.SendWebRequest();

            HandleResponse(request, onSuccess, onError);
        }
    }

    // =============================================
    // POST JSON
    // =============================================

    public void PostJson<T>(string endpoint, T data, Action<string> onSuccess, Action<string> onError)
    {
        StartCoroutine(PostJsonCoroutine(endpoint, data, onSuccess, onError));
    }

    private IEnumerator PostJsonCoroutine<T>(string endpoint, T data, Action<string> onSuccess, Action<string> onError)
    {
        string url = baseUrl + endpoint;
        string json = JsonUtility.ToJson(data);
        byte[] bodyRaw = Encoding.UTF8.GetBytes(json);

        using (UnityWebRequest request = new UnityWebRequest(url, "POST"))
        {
            request.uploadHandler = new UploadHandlerRaw(bodyRaw);
            request.downloadHandler = new DownloadHandlerBuffer();

            ApplyDefaults(request);
            request.SetRequestHeader("Content-Type", "application/json");

            yield return request.SendWebRequest();

            HandleResponse(request, onSuccess, onError);
        }
    }

    // =============================================
    // Helper
    // =============================================

    private void ApplyDefaults(UnityWebRequest request)
    {
        request.timeout = defaultTimeout;

        foreach (var header in defaultHeaders)
        {
            request.SetRequestHeader(header.Key, header.Value);
        }
    }

    private void HandleResponse(UnityWebRequest request, Action<string> onSuccess, Action<string> onError)
    {
        if (request.result == UnityWebRequest.Result.Success)
        {
            onSuccess?.Invoke(request.downloadHandler.text);
        }
        else
        {
            onError?.Invoke($"{request.responseCode}: {request.error}");
        }
    }
}

// 사용 예시
public class HttpClientUsageExample : MonoBehaviour
{
    private void Start()
    {
        // 설정
        HttpClient.Instance.Configure("https://api.example.com");
        HttpClient.Instance.SetDefaultHeader("Authorization", "Bearer token123");

        // GET 요청
        HttpClient.Instance.Get("/users",
            response => Debug.Log($"사용자 목록: {response}"),
            error => Debug.LogError($"에러: {error}")
        );

        // POST 요청
        var userData = new { name = "홍길동", email = "hong@example.com" };
        HttpClient.Instance.PostJson("/users", userData,
            response => Debug.Log($"생성됨: {response}"),
            error => Debug.LogError($"에러: {error}")
        );
    }
}
```

---

## 주의사항

1. **using 필수**: UnityWebRequest는 IDisposable이므로 using 사용
2. **메인 스레드**: Coroutine 기반이므로 메인 스레드에서 실행
3. **타임아웃 설정**: 기본값이 없으므로 명시적 설정 권장
4. **에러 처리**: result 속성으로 상세 에러 구분
5. **인증서**: 프로덕션에서 인증서 검증 비활성화 금지

---

## 참고 자료

- [Unity: UnityWebRequest](https://docs.unity3d.com/ScriptReference/Networking.UnityWebRequest.html)
- [Unity: Networking](https://docs.unity3d.com/Manual/UNetUsingHLAPI.html)

---

## 다음 섹션

[26. HttpClient (.NET)](./26-httpclient.md)
