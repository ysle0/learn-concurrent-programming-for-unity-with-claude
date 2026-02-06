# Section 27: HTTP/REST & JSON

## 개요

REST(Representational State Transfer)는 웹 API 설계의 표준 아키텍처 스타일입니다. Unity 게임에서 서버와 통신할 때 REST API와 JSON 포맷을 가장 널리 사용합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    REST API Architecture                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Unity Client                         REST Server               │
│   ┌──────────┐                        ┌──────────────┐          │
│   │          │  GET /users/123        │              │          │
│   │  Game    │ ─────────────────────▶ │   API        │          │
│   │  Client  │                        │   Server     │          │
│   │          │ ◀───────────────────── │              │          │
│   └──────────┘  200 OK + JSON         └──────────────┘          │
│                                                                  │
│   HTTP Methods:                                                  │
│   ┌─────────┬──────────────────────────────────────────┐        │
│   │ GET     │ 리소스 조회 (Read)                        │        │
│   │ POST    │ 리소스 생성 (Create)                      │        │
│   │ PUT     │ 리소스 전체 수정 (Update/Replace)          │        │
│   │ PATCH   │ 리소스 부분 수정 (Partial Update)          │        │
│   │ DELETE  │ 리소스 삭제 (Delete)                      │        │
│   └─────────┴──────────────────────────────────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## JSON 직렬화 라이브러리 비교

| 라이브러리 | 장점 | 단점 | Unity 호환성 |
|-----------|------|------|-------------|
| **JsonUtility** | 빠름, 내장, GC 최소화 | 기능 제한적 | 완벽 |
| **Newtonsoft.Json** | 풍부한 기능, 유연함 | 리플렉션 사용, 느림 | IL2CPP 주의 |
| **System.Text.Json** | .NET 표준, 빠름 | Unity 2021+ 필요 | 일부 제한 |

---

## JsonUtility (Unity 내장)

### 기본 사용법

```csharp
using System;
using UnityEngine;

/// <summary>
/// JsonUtility 기본 직렬화/역직렬화 예제
/// </summary>
public class JsonUtilityExample : MonoBehaviour
{
    // JsonUtility는 public 필드 또는 [SerializeField] 속성 필요
    [Serializable]
    public class PlayerData
    {
        public string playerName;
        public int level;
        public float health;
        public int[] inventory;
        public Vector3 position;  // Unity 타입 지원
    }

    [Serializable]
    public class GameSettings
    {
        public float musicVolume;
        public float sfxVolume;
        public bool fullscreen;
        public string language;
    }

    private void Start()
    {
        // 직렬화 (객체 → JSON)
        var player = new PlayerData
        {
            playerName = "Hero",
            level = 42,
            health = 100f,
            inventory = new[] { 1, 2, 3, 4, 5 },
            position = new Vector3(10f, 0f, 20f)
        };

        string json = JsonUtility.ToJson(player);
        Debug.Log($"직렬화: {json}");
        // {"playerName":"Hero","level":42,"health":100.0,"inventory":[1,2,3,4,5],"position":{"x":10.0,"y":0.0,"z":20.0}}

        // 예쁘게 출력
        string prettyJson = JsonUtility.ToJson(player, prettyPrint: true);
        Debug.Log($"Pretty JSON:\n{prettyJson}");

        // 역직렬화 (JSON → 객체)
        PlayerData loadedPlayer = JsonUtility.FromJson<PlayerData>(json);
        Debug.Log($"로드된 플레이어: {loadedPlayer.playerName}, Lv.{loadedPlayer.level}");

        // 기존 객체에 덮어쓰기 (GC 최소화)
        var existingPlayer = new PlayerData();
        JsonUtility.FromJsonOverwrite(json, existingPlayer);
    }
}
```

### JsonUtility 제한사항 및 해결책

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// JsonUtility의 제한사항과 해결책
/// </summary>
public class JsonUtilityLimitations : MonoBehaviour
{
    // ❌ JsonUtility는 Dictionary 미지원
    // public Dictionary<string, int> stats;  // 직렬화 안됨

    // ✅ 해결책 1: 리스트로 변환
    [Serializable]
    public class SerializableDictionary
    {
        public List<string> keys = new List<string>();
        public List<int> values = new List<int>();

        public void Add(string key, int value)
        {
            keys.Add(key);
            values.Add(value);
        }

        public bool TryGetValue(string key, out int value)
        {
            int index = keys.IndexOf(key);
            if (index >= 0)
            {
                value = values[index];
                return true;
            }
            value = default;
            return false;
        }

        public Dictionary<string, int> ToDictionary()
        {
            var dict = new Dictionary<string, int>();
            for (int i = 0; i < keys.Count && i < values.Count; i++)
            {
                dict[keys[i]] = values[i];
            }
            return dict;
        }

        public static SerializableDictionary FromDictionary(Dictionary<string, int> dict)
        {
            var result = new SerializableDictionary();
            foreach (var kvp in dict)
            {
                result.keys.Add(kvp.Key);
                result.values.Add(kvp.Value);
            }
            return result;
        }
    }

    // ✅ 해결책 2: 배열 래퍼 (루트가 배열인 경우)
    [Serializable]
    public class ArrayWrapper<T>
    {
        public T[] items;

        public ArrayWrapper(T[] items)
        {
            this.items = items;
        }
    }

    // JsonUtility는 루트 배열 미지원
    // string json = "[1,2,3]";  // 직렬화 안됨

    public static T[] FromJsonArray<T>(string json)
    {
        // {"items":[...]} 형태로 감싸기
        string wrappedJson = $"{{\"items\":{json}}}";
        var wrapper = JsonUtility.FromJson<ArrayWrapper<T>>(wrappedJson);
        return wrapper.items;
    }

    // ✅ 해결책 3: 널 처리
    [Serializable]
    public class NullableData
    {
        public string name;
        public bool hasScore;  // null 대신 플래그 사용
        public int score;

        // 값 설정 헬퍼
        public void SetScore(int? value)
        {
            if (value.HasValue)
            {
                hasScore = true;
                score = value.Value;
            }
            else
            {
                hasScore = false;
                score = 0;
            }
        }

        public int? GetScore()
        {
            return hasScore ? score : (int?)null;
        }
    }

    private void Start()
    {
        // 배열 파싱 예시
        string arrayJson = "[{\"name\":\"item1\"},{\"name\":\"item2\"}]";
        var items = FromJsonArray<SimpleItem>(arrayJson);
        Debug.Log($"파싱된 아이템 수: {items.Length}");
    }

    [Serializable]
    public class SimpleItem
    {
        public string name;
    }
}
```

---

## Newtonsoft.Json (Json.NET)

### 설치 및 기본 사용

```csharp
// Package Manager에서 설치:
// com.unity.nuget.newtonsoft-json

using System;
using System.Collections.Generic;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;
using UnityEngine;

/// <summary>
/// Newtonsoft.Json 사용 예제
/// </summary>
public class NewtonsoftJsonExample : MonoBehaviour
{
    // 다양한 속성 지원
    public class ComplexData
    {
        // 속성 이름 변경
        [JsonProperty("player_name")]
        public string PlayerName { get; set; }

        // 직렬화 제외
        [JsonIgnore]
        public string SecretKey { get; set; }

        // 널이면 무시
        [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
        public string OptionalField { get; set; }

        // Dictionary 지원
        public Dictionary<string, int> Stats { get; set; }

        // DateTime 지원
        public DateTime CreatedAt { get; set; }

        // 다형성 지원
        [JsonProperty(TypeNameHandling = TypeNameHandling.Auto)]
        public List<IItem> Inventory { get; set; }
    }

    public interface IItem
    {
        string Name { get; }
    }

    public class Weapon : IItem
    {
        public string Name { get; set; }
        public int Damage { get; set; }
    }

    public class Potion : IItem
    {
        public string Name { get; set; }
        public int HealAmount { get; set; }
    }

    private void Start()
    {
        var data = new ComplexData
        {
            PlayerName = "Hero",
            SecretKey = "should_not_serialize",
            OptionalField = null,
            Stats = new Dictionary<string, int>
            {
                { "strength", 10 },
                { "dexterity", 8 },
                { "intelligence", 12 }
            },
            CreatedAt = DateTime.UtcNow,
            Inventory = new List<IItem>
            {
                new Weapon { Name = "Sword", Damage = 50 },
                new Potion { Name = "Health Potion", HealAmount = 100 }
            }
        };

        // 직렬화 설정
        var settings = new JsonSerializerSettings
        {
            Formatting = Formatting.Indented,
            NullValueHandling = NullValueHandling.Ignore,
            DateFormatString = "yyyy-MM-ddTHH:mm:ssZ"
        };

        string json = JsonConvert.SerializeObject(data, settings);
        Debug.Log($"Newtonsoft JSON:\n{json}");

        // 역직렬화
        var loadedData = JsonConvert.DeserializeObject<ComplexData>(json, settings);
        Debug.Log($"로드된 플레이어: {loadedData.PlayerName}");
        Debug.Log($"Stats: {string.Join(", ", loadedData.Stats)}");
    }
}
```

### 동적 JSON 처리 (JObject)

```csharp
using Newtonsoft.Json.Linq;
using UnityEngine;

/// <summary>
/// JObject를 사용한 동적 JSON 처리
/// </summary>
public class DynamicJsonExample : MonoBehaviour
{
    private void Start()
    {
        // 알 수 없는 구조의 JSON 파싱
        string unknownJson = @"{
            ""type"": ""player"",
            ""data"": {
                ""name"": ""Hero"",
                ""stats"": {
                    ""hp"": 100,
                    ""mp"": 50
                }
            },
            ""metadata"": [1, 2, 3]
        }";

        JObject root = JObject.Parse(unknownJson);

        // 경로로 접근
        string type = (string)root["type"];
        string name = (string)root["data"]["name"];
        int hp = (int)root["data"]["stats"]["hp"];

        Debug.Log($"Type: {type}, Name: {name}, HP: {hp}");

        // SelectToken으로 깊은 경로 접근
        var mp = root.SelectToken("data.stats.mp");
        Debug.Log($"MP: {mp}");

        // 배열 순회
        JArray metadata = (JArray)root["metadata"];
        foreach (var item in metadata)
        {
            Debug.Log($"Metadata item: {item}");
        }

        // 동적으로 JSON 생성
        var newJson = new JObject
        {
            ["type"] = "response",
            ["success"] = true,
            ["data"] = new JObject
            {
                ["items"] = new JArray { "sword", "shield", "potion" }
            }
        };

        Debug.Log($"생성된 JSON:\n{newJson.ToString(Newtonsoft.Json.Formatting.Indented)}");

        // 값 수정
        root["data"]["stats"]["hp"] = 150;
        Debug.Log($"수정된 HP: {root["data"]["stats"]["hp"]}");
    }

    /// <summary>
    /// 서버 응답에서 특정 필드만 추출
    /// </summary>
    public T ExtractField<T>(string json, string path)
    {
        var root = JObject.Parse(json);
        var token = root.SelectToken(path);

        if (token == null)
        {
            throw new System.Exception($"Path not found: {path}");
        }

        return token.ToObject<T>();
    }
}
```

### IL2CPP 호환성 설정

```csharp
using Newtonsoft.Json;
using Newtonsoft.Json.Serialization;
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// IL2CPP 빌드를 위한 Newtonsoft.Json 설정
/// </summary>
public static class JsonNetConfig
{
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void Initialize()
    {
        // IL2CPP에서 리플렉션 문제 방지를 위한 설정
        JsonConvert.DefaultSettings = () => new JsonSerializerSettings
        {
            // 생성자 처리
            ConstructorHandling = ConstructorHandling.AllowNonPublicDefaultConstructor,

            // 컨트랙트 리졸버 (필요시 커스텀)
            ContractResolver = new DefaultContractResolver
            {
                NamingStrategy = new CamelCaseNamingStrategy()
            },

            // 타입 정보 (다형성 사용 시)
            TypeNameHandling = TypeNameHandling.Auto,

            // 오류 처리
            Error = (sender, args) =>
            {
                Debug.LogWarning($"JSON 오류: {args.ErrorContext.Error.Message}");
                args.ErrorContext.Handled = true;
            }
        };
    }
}

/// <summary>
/// link.xml 예시 (Assets 폴더에 생성)
/// </summary>
/*
<linker>
  <assembly fullname="Newtonsoft.Json" preserve="all"/>
  <assembly fullname="System.Core">
    <type fullname="System.Linq.Expressions.Interpreter.LightLambda" preserve="all" />
  </assembly>
</linker>
*/
```

---

## REST API 클라이언트 구현

### 범용 REST 클라이언트

```csharp
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using Newtonsoft.Json;
using UnityEngine;

/// <summary>
/// 범용 REST API 클라이언트
/// </summary>
public class RestClient : IDisposable
{
    private readonly HttpClient _httpClient;
    private readonly string _baseUrl;
    private readonly JsonSerializerSettings _jsonSettings;

    public RestClient(string baseUrl, TimeSpan? timeout = null)
    {
        _baseUrl = baseUrl.TrimEnd('/');

        _httpClient = new HttpClient
        {
            Timeout = timeout ?? TimeSpan.FromSeconds(30)
        };

        _httpClient.DefaultRequestHeaders.Accept.Add(
            new System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("application/json"));

        _jsonSettings = new JsonSerializerSettings
        {
            NullValueHandling = NullValueHandling.Ignore,
            DateFormatString = "yyyy-MM-ddTHH:mm:ssZ"
        };
    }

    #region Authentication

    public void SetBearerToken(string token)
    {
        _httpClient.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
    }

    public void SetApiKey(string headerName, string apiKey)
    {
        _httpClient.DefaultRequestHeaders.Remove(headerName);
        _httpClient.DefaultRequestHeaders.Add(headerName, apiKey);
    }

    public void ClearAuthentication()
    {
        _httpClient.DefaultRequestHeaders.Authorization = null;
    }

    #endregion

    #region HTTP Methods

    /// <summary>
    /// GET 요청
    /// </summary>
    public async Task<T> GetAsync<T>(
        string endpoint,
        Dictionary<string, string> queryParams = null,
        CancellationToken ct = default)
    {
        string url = BuildUrl(endpoint, queryParams);
        var response = await _httpClient.GetAsync(url, ct);
        return await HandleResponseAsync<T>(response);
    }

    /// <summary>
    /// POST 요청
    /// </summary>
    public async Task<TResponse> PostAsync<TRequest, TResponse>(
        string endpoint,
        TRequest body,
        CancellationToken ct = default)
    {
        string url = BuildUrl(endpoint);
        var content = CreateJsonContent(body);
        var response = await _httpClient.PostAsync(url, content, ct);
        return await HandleResponseAsync<TResponse>(response);
    }

    /// <summary>
    /// POST (응답 없음)
    /// </summary>
    public async Task PostAsync<TRequest>(
        string endpoint,
        TRequest body,
        CancellationToken ct = default)
    {
        string url = BuildUrl(endpoint);
        var content = CreateJsonContent(body);
        var response = await _httpClient.PostAsync(url, content, ct);
        await EnsureSuccessAsync(response);
    }

    /// <summary>
    /// PUT 요청
    /// </summary>
    public async Task<TResponse> PutAsync<TRequest, TResponse>(
        string endpoint,
        TRequest body,
        CancellationToken ct = default)
    {
        string url = BuildUrl(endpoint);
        var content = CreateJsonContent(body);
        var response = await _httpClient.PutAsync(url, content, ct);
        return await HandleResponseAsync<TResponse>(response);
    }

    /// <summary>
    /// PATCH 요청
    /// </summary>
    public async Task<TResponse> PatchAsync<TRequest, TResponse>(
        string endpoint,
        TRequest body,
        CancellationToken ct = default)
    {
        string url = BuildUrl(endpoint);
        var content = CreateJsonContent(body);
        var request = new HttpRequestMessage(new HttpMethod("PATCH"), url)
        {
            Content = content
        };
        var response = await _httpClient.SendAsync(request, ct);
        return await HandleResponseAsync<TResponse>(response);
    }

    /// <summary>
    /// DELETE 요청
    /// </summary>
    public async Task DeleteAsync(string endpoint, CancellationToken ct = default)
    {
        string url = BuildUrl(endpoint);
        var response = await _httpClient.DeleteAsync(url, ct);
        await EnsureSuccessAsync(response);
    }

    #endregion

    #region Helpers

    private string BuildUrl(string endpoint, Dictionary<string, string> queryParams = null)
    {
        var url = $"{_baseUrl}/{endpoint.TrimStart('/')}";

        if (queryParams != null && queryParams.Count > 0)
        {
            var queryString = new StringBuilder("?");
            foreach (var param in queryParams)
            {
                if (queryString.Length > 1) queryString.Append("&");
                queryString.Append($"{Uri.EscapeDataString(param.Key)}={Uri.EscapeDataString(param.Value)}");
            }
            url += queryString.ToString();
        }

        return url;
    }

    private StringContent CreateJsonContent<T>(T body)
    {
        string json = JsonConvert.SerializeObject(body, _jsonSettings);
        return new StringContent(json, Encoding.UTF8, "application/json");
    }

    private async Task<T> HandleResponseAsync<T>(HttpResponseMessage response)
    {
        string content = await response.Content.ReadAsStringAsync();

        if (!response.IsSuccessStatusCode)
        {
            throw new RestException(response.StatusCode, content);
        }

        return JsonConvert.DeserializeObject<T>(content, _jsonSettings);
    }

    private async Task EnsureSuccessAsync(HttpResponseMessage response)
    {
        if (!response.IsSuccessStatusCode)
        {
            string content = await response.Content.ReadAsStringAsync();
            throw new RestException(response.StatusCode, content);
        }
    }

    #endregion

    public void Dispose()
    {
        _httpClient?.Dispose();
    }
}

/// <summary>
/// REST API 예외
/// </summary>
public class RestException : Exception
{
    public System.Net.HttpStatusCode StatusCode { get; }
    public string ResponseContent { get; }

    public RestException(System.Net.HttpStatusCode statusCode, string content)
        : base($"REST API 오류: {statusCode}")
    {
        StatusCode = statusCode;
        ResponseContent = content;
    }
}
```

### 게임 서버 API 예제

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 게임 서버 API 클라이언트
/// </summary>
public class GameServerApi : IDisposable
{
    private readonly RestClient _client;

    public GameServerApi(string baseUrl)
    {
        _client = new RestClient(baseUrl);
    }

    #region DTOs (Data Transfer Objects)

    [Serializable]
    public class LoginRequest
    {
        public string username;
        public string password;
    }

    [Serializable]
    public class LoginResponse
    {
        public string accessToken;
        public string refreshToken;
        public long expiresIn;
        public PlayerInfo player;
    }

    [Serializable]
    public class PlayerInfo
    {
        public string id;
        public string nickname;
        public int level;
        public int experience;
        public int gold;
        public int gems;
    }

    [Serializable]
    public class InventoryItem
    {
        public string itemId;
        public string name;
        public int quantity;
        public int rarity;
    }

    [Serializable]
    public class LeaderboardEntry
    {
        public int rank;
        public string playerId;
        public string nickname;
        public long score;
    }

    [Serializable]
    public class PagedResponse<T>
    {
        public T[] items;
        public int total;
        public int page;
        public int pageSize;
        public bool hasMore;
    }

    #endregion

    #region Authentication

    public async Task<LoginResponse> LoginAsync(
        string username,
        string password,
        CancellationToken ct = default)
    {
        var response = await _client.PostAsync<LoginRequest, LoginResponse>(
            "/auth/login",
            new LoginRequest { username = username, password = password },
            ct);

        // 토큰 자동 설정
        _client.SetBearerToken(response.accessToken);

        return response;
    }

    public async Task<LoginResponse> RefreshTokenAsync(
        string refreshToken,
        CancellationToken ct = default)
    {
        var response = await _client.PostAsync<object, LoginResponse>(
            "/auth/refresh",
            new { refreshToken },
            ct);

        _client.SetBearerToken(response.accessToken);
        return response;
    }

    public void Logout()
    {
        _client.ClearAuthentication();
    }

    #endregion

    #region Player

    public Task<PlayerInfo> GetPlayerAsync(CancellationToken ct = default)
    {
        return _client.GetAsync<PlayerInfo>("/player", ct: ct);
    }

    public Task<PlayerInfo> UpdateNicknameAsync(
        string nickname,
        CancellationToken ct = default)
    {
        return _client.PatchAsync<object, PlayerInfo>(
            "/player",
            new { nickname },
            ct);
    }

    #endregion

    #region Inventory

    public Task<PagedResponse<InventoryItem>> GetInventoryAsync(
        int page = 1,
        int pageSize = 20,
        CancellationToken ct = default)
    {
        return _client.GetAsync<PagedResponse<InventoryItem>>(
            "/inventory",
            new Dictionary<string, string>
            {
                { "page", page.ToString() },
                { "pageSize", pageSize.ToString() }
            },
            ct);
    }

    public Task<InventoryItem> UseItemAsync(
        string itemId,
        int quantity = 1,
        CancellationToken ct = default)
    {
        return _client.PostAsync<object, InventoryItem>(
            $"/inventory/{itemId}/use",
            new { quantity },
            ct);
    }

    #endregion

    #region Leaderboard

    public Task<PagedResponse<LeaderboardEntry>> GetLeaderboardAsync(
        string leaderboardId,
        int limit = 100,
        CancellationToken ct = default)
    {
        return _client.GetAsync<PagedResponse<LeaderboardEntry>>(
            $"/leaderboard/{leaderboardId}",
            new Dictionary<string, string>
            {
                { "limit", limit.ToString() }
            },
            ct);
    }

    public Task<LeaderboardEntry> SubmitScoreAsync(
        string leaderboardId,
        long score,
        CancellationToken ct = default)
    {
        return _client.PostAsync<object, LeaderboardEntry>(
            $"/leaderboard/{leaderboardId}/submit",
            new { score },
            ct);
    }

    #endregion

    public void Dispose()
    {
        _client?.Dispose();
    }
}

/// <summary>
/// 게임 서버 API 사용 예시
/// </summary>
public class GameServerApiExample : MonoBehaviour
{
    private GameServerApi _api;
    private CancellationTokenSource _cts;

    private void Awake()
    {
        _api = new GameServerApi("https://api.mygame.com/v1");
        _cts = new CancellationTokenSource();
    }

    private async void Start()
    {
        try
        {
            // 로그인
            var loginResult = await _api.LoginAsync(
                "player@example.com",
                "password123",
                _cts.Token);

            Debug.Log($"환영합니다, {loginResult.player.nickname}!");
            Debug.Log($"레벨: {loginResult.player.level}, 골드: {loginResult.player.gold}");

            // 인벤토리 조회
            var inventory = await _api.GetInventoryAsync(ct: _cts.Token);
            Debug.Log($"아이템 보유: {inventory.total}개");

            foreach (var item in inventory.items)
            {
                Debug.Log($"  - {item.name} x{item.quantity}");
            }

            // 리더보드
            var leaderboard = await _api.GetLeaderboardAsync("weekly", 10, _cts.Token);
            Debug.Log("=== 주간 랭킹 ===");
            foreach (var entry in leaderboard.items)
            {
                Debug.Log($"  {entry.rank}. {entry.nickname}: {entry.score}");
            }
        }
        catch (RestException ex)
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

## UnityWebRequest를 사용한 REST 클라이언트

### Coroutine 기반 구현

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Text;
using UnityEngine;
using UnityEngine.Networking;

/// <summary>
/// UnityWebRequest 기반 REST 클라이언트 (Coroutine)
/// WebGL 호환
/// </summary>
public class UnityRestClient : MonoBehaviour
{
    [SerializeField] private string baseUrl = "https://api.example.com";

    private Dictionary<string, string> _defaultHeaders = new Dictionary<string, string>();

    public void SetBearerToken(string token)
    {
        _defaultHeaders["Authorization"] = $"Bearer {token}";
    }

    /// <summary>
    /// GET 요청
    /// </summary>
    public IEnumerator GetAsync<T>(
        string endpoint,
        Action<T> onSuccess,
        Action<string> onError,
        Dictionary<string, string> queryParams = null)
    {
        string url = BuildUrl(endpoint, queryParams);

        using var request = UnityWebRequest.Get(url);
        ApplyHeaders(request);

        yield return request.SendWebRequest();

        HandleResponse(request, onSuccess, onError);
    }

    /// <summary>
    /// POST 요청
    /// </summary>
    public IEnumerator PostAsync<TRequest, TResponse>(
        string endpoint,
        TRequest body,
        Action<TResponse> onSuccess,
        Action<string> onError)
    {
        string url = BuildUrl(endpoint);
        string jsonBody = JsonUtility.ToJson(body);

        using var request = new UnityWebRequest(url, "POST");
        request.uploadHandler = new UploadHandlerRaw(Encoding.UTF8.GetBytes(jsonBody));
        request.downloadHandler = new DownloadHandlerBuffer();
        request.SetRequestHeader("Content-Type", "application/json");
        ApplyHeaders(request);

        yield return request.SendWebRequest();

        HandleResponse(request, onSuccess, onError);
    }

    /// <summary>
    /// PUT 요청
    /// </summary>
    public IEnumerator PutAsync<TRequest, TResponse>(
        string endpoint,
        TRequest body,
        Action<TResponse> onSuccess,
        Action<string> onError)
    {
        string url = BuildUrl(endpoint);
        string jsonBody = JsonUtility.ToJson(body);

        using var request = UnityWebRequest.Put(url, jsonBody);
        request.SetRequestHeader("Content-Type", "application/json");
        ApplyHeaders(request);

        yield return request.SendWebRequest();

        HandleResponse(request, onSuccess, onError);
    }

    /// <summary>
    /// DELETE 요청
    /// </summary>
    public IEnumerator DeleteAsync(
        string endpoint,
        Action onSuccess,
        Action<string> onError)
    {
        string url = BuildUrl(endpoint);

        using var request = UnityWebRequest.Delete(url);
        ApplyHeaders(request);

        yield return request.SendWebRequest();

        if (request.result == UnityWebRequest.Result.Success)
        {
            onSuccess?.Invoke();
        }
        else
        {
            onError?.Invoke(request.error);
        }
    }

    private string BuildUrl(string endpoint, Dictionary<string, string> queryParams = null)
    {
        string url = $"{baseUrl}/{endpoint.TrimStart('/')}";

        if (queryParams != null && queryParams.Count > 0)
        {
            var sb = new StringBuilder("?");
            foreach (var param in queryParams)
            {
                if (sb.Length > 1) sb.Append("&");
                sb.Append($"{UnityWebRequest.EscapeURL(param.Key)}={UnityWebRequest.EscapeURL(param.Value)}");
            }
            url += sb.ToString();
        }

        return url;
    }

    private void ApplyHeaders(UnityWebRequest request)
    {
        foreach (var header in _defaultHeaders)
        {
            request.SetRequestHeader(header.Key, header.Value);
        }
    }

    private void HandleResponse<T>(
        UnityWebRequest request,
        Action<T> onSuccess,
        Action<string> onError)
    {
        if (request.result == UnityWebRequest.Result.Success)
        {
            try
            {
                T response = JsonUtility.FromJson<T>(request.downloadHandler.text);
                onSuccess?.Invoke(response);
            }
            catch (Exception ex)
            {
                onError?.Invoke($"JSON 파싱 오류: {ex.Message}");
            }
        }
        else
        {
            onError?.Invoke($"{request.responseCode}: {request.error}");
        }
    }
}

/// <summary>
/// 사용 예시
/// </summary>
public class UnityRestClientExample : MonoBehaviour
{
    [Serializable]
    public class UserData
    {
        public string id;
        public string name;
        public string email;
    }

    private UnityRestClient _client;

    private void Awake()
    {
        _client = GetComponent<UnityRestClient>();
    }

    private void Start()
    {
        StartCoroutine(_client.GetAsync<UserData>(
            "/users/1",
            onSuccess: user =>
            {
                Debug.Log($"사용자: {user.name} ({user.email})");
            },
            onError: error =>
            {
                Debug.LogError($"오류: {error}");
            }
        ));
    }
}
```

### UniTask 기반 구현

```csharp
using System;
using System.Collections.Generic;
using System.Text;
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine;
using UnityEngine.Networking;

/// <summary>
/// UniTask 기반 REST 클라이언트
/// </summary>
public class UniTaskRestClient
{
    private readonly string _baseUrl;
    private readonly Dictionary<string, string> _defaultHeaders = new();

    public UniTaskRestClient(string baseUrl)
    {
        _baseUrl = baseUrl.TrimEnd('/');
    }

    public void SetBearerToken(string token)
    {
        _defaultHeaders["Authorization"] = $"Bearer {token}";
    }

    /// <summary>
    /// GET 요청
    /// </summary>
    public async UniTask<T> GetAsync<T>(
        string endpoint,
        Dictionary<string, string> queryParams = null,
        CancellationToken ct = default)
    {
        string url = BuildUrl(endpoint, queryParams);

        using var request = UnityWebRequest.Get(url);
        ApplyHeaders(request);

        await request.SendWebRequest().WithCancellation(ct);

        return HandleResponse<T>(request);
    }

    /// <summary>
    /// POST 요청
    /// </summary>
    public async UniTask<TResponse> PostAsync<TRequest, TResponse>(
        string endpoint,
        TRequest body,
        CancellationToken ct = default)
    {
        string url = BuildUrl(endpoint);
        string jsonBody = JsonUtility.ToJson(body);

        using var request = new UnityWebRequest(url, "POST");
        request.uploadHandler = new UploadHandlerRaw(Encoding.UTF8.GetBytes(jsonBody));
        request.downloadHandler = new DownloadHandlerBuffer();
        request.SetRequestHeader("Content-Type", "application/json");
        ApplyHeaders(request);

        await request.SendWebRequest().WithCancellation(ct);

        return HandleResponse<TResponse>(request);
    }

    /// <summary>
    /// 타임아웃이 적용된 요청
    /// </summary>
    public async UniTask<T> GetWithTimeoutAsync<T>(
        string endpoint,
        TimeSpan timeout,
        CancellationToken ct = default)
    {
        using var timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        timeoutCts.CancelAfter(timeout);

        try
        {
            return await GetAsync<T>(endpoint, ct: timeoutCts.Token);
        }
        catch (OperationCanceledException) when (!ct.IsCancellationRequested)
        {
            throw new TimeoutException($"요청 시간 초과: {timeout.TotalSeconds}초");
        }
    }

    private string BuildUrl(string endpoint, Dictionary<string, string> queryParams = null)
    {
        string url = $"{_baseUrl}/{endpoint.TrimStart('/')}";

        if (queryParams != null && queryParams.Count > 0)
        {
            var sb = new StringBuilder("?");
            foreach (var param in queryParams)
            {
                if (sb.Length > 1) sb.Append("&");
                sb.Append($"{UnityWebRequest.EscapeURL(param.Key)}={UnityWebRequest.EscapeURL(param.Value)}");
            }
            url += sb.ToString();
        }

        return url;
    }

    private void ApplyHeaders(UnityWebRequest request)
    {
        foreach (var header in _defaultHeaders)
        {
            request.SetRequestHeader(header.Key, header.Value);
        }
    }

    private T HandleResponse<T>(UnityWebRequest request)
    {
        if (request.result != UnityWebRequest.Result.Success)
        {
            throw new Exception($"HTTP {request.responseCode}: {request.error}");
        }

        return JsonUtility.FromJson<T>(request.downloadHandler.text);
    }
}

/// <summary>
/// UniTask REST 클라이언트 사용 예시
/// </summary>
public class UniTaskRestExample : MonoBehaviour
{
    [Serializable]
    public class Post
    {
        public int id;
        public string title;
        public string body;
    }

    private UniTaskRestClient _client;

    private void Awake()
    {
        _client = new UniTaskRestClient("https://jsonplaceholder.typicode.com");
    }

    private async void Start()
    {
        try
        {
            // 5초 타임아웃으로 요청
            var post = await _client.GetWithTimeoutAsync<Post>(
                "/posts/1",
                TimeSpan.FromSeconds(5),
                destroyCancellationToken);

            Debug.Log($"제목: {post.title}");
            Debug.Log($"내용: {post.body}");
        }
        catch (TimeoutException)
        {
            Debug.LogWarning("서버 응답 시간 초과");
        }
        catch (OperationCanceledException)
        {
            Debug.Log("요청 취소됨");
        }
        catch (Exception ex)
        {
            Debug.LogError($"오류: {ex.Message}");
        }
    }
}
```

---

## 고급 JSON 패턴

### JSON 스키마 검증

```csharp
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;
using Newtonsoft.Json.Schema;
using UnityEngine;

/// <summary>
/// JSON 스키마 검증 예제
/// 패키지 필요: Newtonsoft.Json.Schema (NuGet)
/// </summary>
public class JsonSchemaValidation : MonoBehaviour
{
    // 간단한 수동 검증 (외부 패키지 없이)
    public static class SimpleValidator
    {
        public static bool ValidatePlayerData(JObject json, out string error)
        {
            error = null;

            // 필수 필드 검증
            if (json["name"] == null)
            {
                error = "name 필드가 필요합니다";
                return false;
            }

            if (json["level"] == null || json["level"].Type != JTokenType.Integer)
            {
                error = "level 필드는 정수여야 합니다";
                return false;
            }

            int level = (int)json["level"];
            if (level < 1 || level > 100)
            {
                error = "level은 1-100 사이여야 합니다";
                return false;
            }

            return true;
        }
    }

    private void Start()
    {
        string json = @"{ ""name"": ""Hero"", ""level"": 50 }";

        var obj = JObject.Parse(json);
        if (SimpleValidator.ValidatePlayerData(obj, out string error))
        {
            Debug.Log("유효한 플레이어 데이터");
        }
        else
        {
            Debug.LogError($"검증 실패: {error}");
        }
    }
}
```

### 커스텀 JSON 컨버터

```csharp
using System;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;
using UnityEngine;

/// <summary>
/// Unity Vector3를 위한 커스텀 JSON 컨버터
/// </summary>
public class Vector3Converter : JsonConverter<Vector3>
{
    public override Vector3 ReadJson(
        JsonReader reader,
        Type objectType,
        Vector3 existingValue,
        bool hasExistingValue,
        JsonSerializer serializer)
    {
        var obj = JObject.Load(reader);
        return new Vector3(
            (float)(obj["x"] ?? 0),
            (float)(obj["y"] ?? 0),
            (float)(obj["z"] ?? 0)
        );
    }

    public override void WriteJson(
        JsonWriter writer,
        Vector3 value,
        JsonSerializer serializer)
    {
        writer.WriteStartObject();
        writer.WritePropertyName("x");
        writer.WriteValue(value.x);
        writer.WritePropertyName("y");
        writer.WriteValue(value.y);
        writer.WritePropertyName("z");
        writer.WriteValue(value.z);
        writer.WriteEndObject();
    }
}

/// <summary>
/// 색상을 HEX 문자열로 변환하는 컨버터
/// </summary>
public class ColorHexConverter : JsonConverter<Color>
{
    public override Color ReadJson(
        JsonReader reader,
        Type objectType,
        Color existingValue,
        bool hasExistingValue,
        JsonSerializer serializer)
    {
        string hex = (string)reader.Value;
        if (ColorUtility.TryParseHtmlString(hex, out Color color))
        {
            return color;
        }
        return Color.white;
    }

    public override void WriteJson(
        JsonWriter writer,
        Color value,
        JsonSerializer serializer)
    {
        writer.WriteValue($"#{ColorUtility.ToHtmlStringRGBA(value)}");
    }
}

/// <summary>
/// 컨버터 사용 예시
/// </summary>
public class CustomConverterExample : MonoBehaviour
{
    [Serializable]
    public class GameEntity
    {
        [JsonConverter(typeof(Vector3Converter))]
        public Vector3 Position { get; set; }

        [JsonConverter(typeof(ColorHexConverter))]
        public Color Color { get; set; }

        public string Name { get; set; }
    }

    private void Start()
    {
        var entity = new GameEntity
        {
            Name = "Player",
            Position = new Vector3(10, 5, 20),
            Color = Color.red
        };

        var settings = new JsonSerializerSettings
        {
            Formatting = Formatting.Indented,
            Converters = new JsonConverter[]
            {
                new Vector3Converter(),
                new ColorHexConverter()
            }
        };

        string json = JsonConvert.SerializeObject(entity, settings);
        Debug.Log(json);
        // {
        //   "Position": { "x": 10, "y": 5, "z": 20 },
        //   "Color": "#FF0000FF",
        //   "Name": "Player"
        // }

        var loaded = JsonConvert.DeserializeObject<GameEntity>(json, settings);
        Debug.Log($"위치: {loaded.Position}, 색상: {loaded.Color}");
    }
}
```

---

## 에러 핸들링 패턴

```csharp
using System;
using System.Net;
using System.Net.Http;
using System.Threading.Tasks;
using Newtonsoft.Json;
using UnityEngine;

/// <summary>
/// API 오류 응답 모델
/// </summary>
[Serializable]
public class ApiError
{
    public string code;
    public string message;
    public string[] details;
}

/// <summary>
/// API 예외
/// </summary>
public class ApiException : Exception
{
    public HttpStatusCode StatusCode { get; }
    public ApiError Error { get; }

    public ApiException(HttpStatusCode statusCode, ApiError error)
        : base(error?.message ?? "알 수 없는 API 오류")
    {
        StatusCode = statusCode;
        Error = error;
    }

    public bool IsAuthError => StatusCode == HttpStatusCode.Unauthorized
                            || StatusCode == HttpStatusCode.Forbidden;

    public bool IsNotFound => StatusCode == HttpStatusCode.NotFound;

    public bool IsServerError => (int)StatusCode >= 500;

    public bool IsClientError => (int)StatusCode >= 400 && (int)StatusCode < 500;
}

/// <summary>
/// 에러 핸들링이 포함된 API 호출 예시
/// </summary>
public class ErrorHandlingExample : MonoBehaviour
{
    private RestClient _client;

    private async void Start()
    {
        _client = new RestClient("https://api.example.com");

        try
        {
            var result = await _client.GetAsync<object>("/protected-resource");
            Debug.Log("성공!");
        }
        catch (ApiException ex) when (ex.IsAuthError)
        {
            Debug.LogWarning("인증이 필요합니다. 로그인 화면으로 이동합니다.");
            // 로그인 화면 표시
            ShowLoginScreen();
        }
        catch (ApiException ex) when (ex.IsNotFound)
        {
            Debug.LogWarning("리소스를 찾을 수 없습니다.");
        }
        catch (ApiException ex) when (ex.IsServerError)
        {
            Debug.LogError($"서버 오류: {ex.Error?.message}");
            // 서버 점검 메시지 표시
            ShowServerMaintenanceMessage();
        }
        catch (ApiException ex)
        {
            Debug.LogError($"API 오류 ({ex.StatusCode}): {ex.Error?.message}");
            // 일반 오류 메시지
            ShowErrorDialog(ex.Error?.message ?? "알 수 없는 오류가 발생했습니다.");
        }
        catch (HttpRequestException ex)
        {
            Debug.LogError($"네트워크 오류: {ex.Message}");
            ShowNetworkErrorDialog();
        }
        catch (TaskCanceledException)
        {
            Debug.Log("요청이 취소되었습니다.");
        }
    }

    private void ShowLoginScreen() { /* UI 로직 */ }
    private void ShowServerMaintenanceMessage() { /* UI 로직 */ }
    private void ShowErrorDialog(string message) { /* UI 로직 */ }
    private void ShowNetworkErrorDialog() { /* UI 로직 */ }
}
```

---

## 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────┐
│                  REST API & JSON 베스트 프랙티스                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 직렬화 라이브러리 선택                                        │
│     ├── 성능 중심: JsonUtility                                   │
│     ├── 기능 중심: Newtonsoft.Json                               │
│     └── WebGL 호환: JsonUtility 권장                             │
│                                                                  │
│  2. DTO 설계                                                     │
│     ├── 서버 응답과 1:1 매핑되는 클래스 정의                       │
│     ├── [Serializable] 속성 필수                                  │
│     └── 불필요한 필드는 [JsonIgnore]로 제외                        │
│                                                                  │
│  3. 에러 핸들링                                                   │
│     ├── HTTP 상태 코드별 분기 처리                                │
│     ├── 네트워크 오류와 API 오류 구분                             │
│     └── 사용자 친화적인 오류 메시지                               │
│                                                                  │
│  4. 보안                                                         │
│     ├── HTTPS 필수 사용                                          │
│     ├── 민감한 데이터는 토큰으로 보호                             │
│     └── API 키는 빌드에 하드코딩하지 않음                          │
│                                                                  │
│  5. 성능                                                         │
│     ├── 큰 응답은 페이지네이션 활용                               │
│     ├── 불필요한 필드 요청 줄이기                                  │
│     └── 캐싱 활용 (ETag, Last-Modified)                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [REST API Tutorial](https://restfulapi.net/)
- [JsonUtility - Unity Documentation](https://docs.unity3d.com/ScriptReference/JsonUtility.html)
- [Newtonsoft.Json Documentation](https://www.newtonsoft.com/json/help/html/Introduction.htm)
- [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)

---

## 다음 단계

- [Section 28: WebSocket](./28-websocket.md) - 실시간 양방향 통신
- [Section 29: gRPC](./29-grpc.md) - 고성능 RPC 프로토콜
