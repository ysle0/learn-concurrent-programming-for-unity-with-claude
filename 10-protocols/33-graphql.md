# 33. GraphQL

## 개요

GraphQL은 Facebook이 개발한 API 쿼리 언어로, 클라이언트가 필요한 데이터를 정확히 명시하여 요청할 수 있습니다. REST API와 달리 단일 엔드포인트에서 유연한 데이터 조회가 가능하며, 게임 클라이언트의 네트워크 효율성을 극대화할 수 있습니다.

### GraphQL vs REST 비교

| 특성 | REST | GraphQL |
|------|------|---------|
| 엔드포인트 | 리소스별 다수 (`/users`, `/items`) | 단일 (`/graphql`) |
| 데이터 페칭 | 서버가 응답 구조 결정 | 클라이언트가 필요한 필드 명시 |
| Over-fetching | 불필요한 데이터 포함 가능 | 요청한 필드만 반환 |
| Under-fetching | 여러 요청 필요할 수 있음 | 한 번의 요청으로 관련 데이터 조회 |
| 실시간 | 별도 구현 필요 (WebSocket 등) | Subscription 내장 지원 |
| 타입 시스템 | 없음 (OpenAPI 별도) | 스키마로 강타입 보장 |

---

## 1. GraphQL 기본 개념 (Query, Mutation, Subscription)

```csharp
using UnityEngine;

/// <summary>
/// GraphQL의 세 가지 주요 오퍼레이션 타입
/// </summary>
public class GraphQLConceptsExample : MonoBehaviour
{
    // Query: 데이터 조회 (READ)
    private const string PlayerQuery = @"
        query GetPlayer($playerId: ID!) {
            player(id: $playerId) {
                name
                level
                inventory { itemId name quantity }
                guild { name memberCount }
            }
        }";

    // Mutation: 데이터 변경 (CREATE, UPDATE, DELETE)
    private const string EquipItemMutation = @"
        mutation EquipItem($playerId: ID!, $itemId: ID!, $slot: EquipSlot!) {
            equipItem(playerId: $playerId, itemId: $itemId, slot: $slot) {
                success
                player {
                    equippedItems { slot item { name } }
                }
                error
            }
        }";

    // Subscription: 실시간 데이터 스트림 (WebSocket 기반)
    private const string ChatSubscription = @"
        subscription OnChatMessage($channelId: ID!) {
            chatMessageReceived(channelId: $channelId) {
                id
                sender { name avatar }
                content
                timestamp
            }
        }";
}
```

---

## 2. GraphQL 요청/응답 모델

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;

[Serializable]
public class GraphQLRequest
{
    public string query;
    public string operationName;
    public string variables;

    public GraphQLRequest(string query, string operationName = null,
                          Dictionary<string, object> variables = null)
    {
        this.query = query;
        this.operationName = operationName;
        if (variables != null)
            this.variables = DictionaryToJson(variables);
    }

    public string ToJson() => JsonUtility.ToJson(this);

    private string DictionaryToJson(Dictionary<string, object> dict)
    {
        var parts = new List<string>();
        foreach (var kvp in dict)
        {
            string value = kvp.Value is string s ? $"\"{s}\""
                         : kvp.Value is bool b ? b.ToString().ToLower()
                         : kvp.Value.ToString();
            parts.Add($"\"{kvp.Key}\":{value}");
        }
        return "{" + string.Join(",", parts) + "}";
    }
}

[Serializable]
public class GraphQLResponse<T>
{
    public T data;
    public GraphQLError[] errors;
    public bool HasErrors => errors != null && errors.Length > 0;
}

[Serializable]
public class GraphQLError
{
    public string message;
    public GraphQLErrorLocation[] locations;
    public string[] path;
    public GraphQLErrorExtensions extensions;
}

[Serializable]
public class GraphQLErrorLocation { public int line; public int column; }

[Serializable]
public class GraphQLErrorExtensions { public string code; public string classification; }
```

---

## 3. UnityWebRequest로 GraphQL 쿼리 전송

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System;
using System.Collections;
using System.Collections.Generic;
using System.Text;

public class GraphQLClient : MonoBehaviour
{
    [SerializeField] private string endpoint = "https://api.example.com/graphql";
    [SerializeField] private int timeoutSeconds = 30;
    private Dictionary<string, string> headers = new Dictionary<string, string>();

    public void SetAuthToken(string token) =>
        headers["Authorization"] = $"Bearer {token}";

    public void Query<T>(string query, Dictionary<string, object> variables,
                         Action<T> onSuccess, Action<GraphQLError[]> onError,
                         string operationName = null)
    {
        StartCoroutine(ExecuteRequest(query, variables, operationName, onSuccess, onError));
    }

    public void Mutate<T>(string mutation, Dictionary<string, object> variables,
                          Action<T> onSuccess, Action<GraphQLError[]> onError,
                          string operationName = null)
    {
        StartCoroutine(ExecuteRequest(mutation, variables, operationName, onSuccess, onError));
    }

    private IEnumerator ExecuteRequest<T>(string query,
        Dictionary<string, object> variables, string operationName,
        Action<T> onSuccess, Action<GraphQLError[]> onError)
    {
        var request = new GraphQLRequest(query, operationName, variables);
        byte[] bodyRaw = Encoding.UTF8.GetBytes(request.ToJson());

        using (UnityWebRequest webRequest = new UnityWebRequest(endpoint, "POST"))
        {
            webRequest.uploadHandler = new UploadHandlerRaw(bodyRaw);
            webRequest.downloadHandler = new DownloadHandlerBuffer();
            webRequest.timeout = timeoutSeconds;
            webRequest.SetRequestHeader("Content-Type", "application/json");
            webRequest.SetRequestHeader("Accept", "application/json");

            foreach (var header in headers)
                webRequest.SetRequestHeader(header.Key, header.Value);

            yield return webRequest.SendWebRequest();

            if (webRequest.result != UnityWebRequest.Result.Success)
            {
                var networkError = new GraphQLError
                {
                    message = $"네트워크 에러: {webRequest.error} (코드: {webRequest.responseCode})"
                };
                onError?.Invoke(new[] { networkError });
                yield break;
            }

            try
            {
                string responseText = webRequest.downloadHandler.text;
                var response = JsonUtility.FromJson<GraphQLResponse<T>>(responseText);

                if (response.HasErrors)
                    onError?.Invoke(response.errors);
                else
                    onSuccess?.Invoke(response.data);
            }
            catch (Exception ex)
            {
                onError?.Invoke(new[] { new GraphQLError { message = $"파싱 실패: {ex.Message}" } });
            }
        }
    }
}

// 사용 예시
public class GraphQLUsageExample : MonoBehaviour
{
    [SerializeField] private GraphQLClient graphQL;

    [Serializable] public class PlayerData { public PlayerInfo player; }
    [Serializable] public class PlayerInfo { public string name; public int level; }

    private void Start()
    {
        graphQL.SetAuthToken("my-jwt-token");

        var variables = new Dictionary<string, object> { { "playerId", "player-123" } };

        graphQL.Query<PlayerData>(
            @"query GetPlayer($playerId: ID!) {
                player(id: $playerId) { name level }
            }",
            variables,
            data => Debug.Log($"플레이어: {data.player.name}, Lv.{data.player.level}"),
            errors => { foreach (var e in errors) Debug.LogError($"에러: {e.message}"); },
            operationName: "GetPlayer"
        );
    }
}
```

---

## 4. async/await 기반 클라이언트

```csharp
using UnityEngine;
using UnityEngine.Networking;
using System;
using System.Collections.Generic;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

public class AsyncGraphQLClient
{
    private readonly string endpoint;
    private readonly Dictionary<string, string> headers = new Dictionary<string, string>();

    public AsyncGraphQLClient(string endpoint) { this.endpoint = endpoint; }
    public void SetAuthToken(string token) => headers["Authorization"] = $"Bearer {token}";

    public async Task<T> QueryAsync<T>(string query,
        Dictionary<string, object> variables = null,
        string operationName = null,
        CancellationToken ct = default)
    {
        var request = new GraphQLRequest(query, operationName, variables);
        byte[] bodyRaw = Encoding.UTF8.GetBytes(request.ToJson());

        using var webRequest = new UnityWebRequest(endpoint, "POST");
        webRequest.uploadHandler = new UploadHandlerRaw(bodyRaw);
        webRequest.downloadHandler = new DownloadHandlerBuffer();
        webRequest.timeout = 30;
        webRequest.SetRequestHeader("Content-Type", "application/json");
        foreach (var h in headers) webRequest.SetRequestHeader(h.Key, h.Value);

        var op = webRequest.SendWebRequest();
        while (!op.isDone) { ct.ThrowIfCancellationRequested(); await Task.Yield(); }

        if (webRequest.result != UnityWebRequest.Result.Success)
            throw new GraphQLNetworkException(webRequest.error, webRequest.responseCode);

        var response = JsonUtility.FromJson<GraphQLResponse<T>>(webRequest.downloadHandler.text);
        if (response.HasErrors) throw new GraphQLQueryException(response.errors);
        return response.data;
    }
}

public class GraphQLNetworkException : Exception
{
    public long StatusCode { get; }
    public GraphQLNetworkException(string msg, long code) : base(msg) { StatusCode = code; }
}

public class GraphQLQueryException : Exception
{
    public GraphQLError[] Errors { get; }
    public GraphQLQueryException(GraphQLError[] errors)
        : base(errors.Length > 0 ? errors[0].message : "GraphQL query failed")
    { Errors = errors; }
}
```

---

## 5. 쿼리 작성법 (변수, 프래그먼트, 인라인 프래그먼트)

```csharp
using UnityEngine;

public class GraphQLQueryPatternsExample : MonoBehaviour
{
    // =============================================
    // 변수(Variables) - 재사용 및 인젝션 방지
    // =============================================

    private const string ItemSearchQuery = @"
        query SearchItems(
            $keyword: String!
            $category: ItemCategory
            $limit: Int = 20
            $offset: Int = 0
        ) {
            items(filter: { keyword: $keyword, category: $category }
                  pagination: { limit: $limit, offset: $offset }) {
                totalCount
                nodes { id name rarity level price }
                pageInfo { hasNextPage endCursor }
            }
        }";

    // =============================================
    // 프래그먼트(Fragment) - 공통 필드 세트 재사용
    // =============================================

    private const string FragmentQuery = @"
        fragment PlayerBasicInfo on Player {
            id name level avatar
        }

        query GetBattleResult($battleId: ID!) {
            battle(id: $battleId) {
                winner { ...PlayerBasicInfo stats { damageDealt } }
                loser { ...PlayerBasicInfo stats { damageDealt } }
            }
        }";

    // =============================================
    // 인라인 프래그먼트 - 유니온/인터페이스 타입 처리
    // =============================================

    // union Reward = GoldReward | ItemReward | ExperienceReward
    private const string InlineFragmentQuery = @"
        query GetQuestRewards($questId: ID!) {
            quest(id: $questId) {
                name
                rewards {
                    ... on GoldReward { __typename amount bonusMultiplier }
                    ... on ItemReward { __typename item { id name rarity } quantity }
                    ... on ExperienceReward { __typename amount skillType }
                }
            }
        }";

    // =============================================
    // 디렉티브(@include, @skip) - 조건부 필드 요청
    // =============================================

    private const string ConditionalQuery = @"
        query GetPlayerProfile(
            $playerId: ID!
            $includeInventory: Boolean!
            $skipAchievements: Boolean!
        ) {
            player(id: $playerId) {
                name level
                inventory @include(if: $includeInventory) { items { name quantity } }
                achievements @skip(if: $skipAchievements) { title unlockedAt }
            }
        }";

    // =============================================
    // 별칭(Alias) - 동일 필드 다중 조회
    // =============================================

    private const string AliasQuery = @"
        query CompareWeapons($id1: ID!, $id2: ID!) {
            weapon1: item(id: $id1) { name stats { attack defense } }
            weapon2: item(id: $id2) { name stats { attack defense } }
        }";
}
```

---

## 6. Subscription (실시간 데이터, WebSocket 기반)

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// graphql-transport-ws 프로토콜 기반 Subscription 클라이언트
/// 실제 WebSocket 구현은 NativeWebSocket 또는 System.Net.WebSockets 사용
/// </summary>
public class GraphQLSubscriptionClient : IDisposable
{
    private readonly string wsEndpoint;
    private CancellationTokenSource cts;
    private readonly Dictionary<string, Action<string>> subscriptions
        = new Dictionary<string, Action<string>>();
    private int nextId = 1;

    public event Action OnConnected;
    public event Action<string> OnError;

    public GraphQLSubscriptionClient(string wsEndpoint)
    {
        this.wsEndpoint = wsEndpoint;
    }

    // graphql-transport-ws 프로토콜 흐름:
    // 1. WebSocket 연결
    // 2. connection_init 메시지 전송 (인증 정보 포함)
    // 3. connection_ack 수신 대기
    // 4. subscribe 메시지로 구독 등록
    // 5. next 메시지로 데이터 수신
    // 6. complete 메시지로 구독 해제

    public async Task ConnectAsync(Dictionary<string, string> connectionParams = null)
    {
        cts = new CancellationTokenSource();
        // WebSocket 연결 및 connection_init 전송
        // connection_ack 수신 시 OnConnected 발행
        OnConnected?.Invoke();
    }

    public async Task<string> Subscribe(string query,
        Dictionary<string, object> variables,
        Action<string> onData, Action<string> onError = null)
    {
        string id = (nextId++).ToString();
        // { "id": "1", "type": "subscribe", "payload": { "query": "...", "variables": {} } }
        subscriptions[id] = onData;
        return id;
    }

    public async Task Unsubscribe(string subscriptionId)
    {
        // { "id": "1", "type": "complete" } 전송
        subscriptions.Remove(subscriptionId);
    }

    public void Dispose()
    {
        cts?.Cancel();
        cts?.Dispose();
    }
}

// Subscription 사용 예시
public class SubscriptionUsageExample : MonoBehaviour
{
    private GraphQLSubscriptionClient subClient;
    private string chatSubId;

    private async void Start()
    {
        subClient = new GraphQLSubscriptionClient("wss://api.mygame.com/graphql");
        await subClient.ConnectAsync(new Dictionary<string, string>
        {
            { "authorization", "Bearer my-jwt-token" }
        });

        chatSubId = await subClient.Subscribe(
            @"subscription OnChat($channelId: ID!) {
                chatMessage(channelId: $channelId) { sender content timestamp }
            }",
            new Dictionary<string, object> { { "channelId", "general" } },
            onData: msg => Debug.Log($"새 메시지: {msg}"),
            onError: err => Debug.LogError($"채팅 에러: {err}")
        );
    }

    private async void OnDestroy()
    {
        if (chatSubId != null) await subClient.Unsubscribe(chatSubId);
        subClient?.Dispose();
    }
}
```

---

## 7. 에러 핸들링

```csharp
using UnityEngine;
using System;

/// <summary>
/// GraphQL 에러 핸들링 전략
/// 핵심: GraphQL은 HTTP 200으로 에러를 반환할 수 있음
/// </summary>
public class GraphQLErrorHandlingExample : MonoBehaviour
{
    // GraphQL 에러 응답 예시:
    // HTTP 200 OK
    // {
    //   "data": { "player": null },
    //   "errors": [{
    //     "message": "Player not found",
    //     "path": ["player"],
    //     "extensions": { "code": "NOT_FOUND" }
    //   }]
    // }
    //
    // 부분 성공 (data + errors 동시 존재 가능):
    // {
    //   "data": { "player": { "name": "Warrior" }, "guild": null },
    //   "errors": [{ "message": "Guild service unavailable", "path": ["guild"] }]
    // }

    public enum GraphQLErrorCode
    {
        Unknown, NotFound, Unauthorized, Forbidden,
        ValidationError, RateLimited, ServiceUnavailable, InternalError
    }

    public static GraphQLErrorCode ClassifyError(GraphQLError error)
    {
        if (error.extensions == null) return GraphQLErrorCode.Unknown;

        return error.extensions.code switch
        {
            "NOT_FOUND" => GraphQLErrorCode.NotFound,
            "UNAUTHENTICATED" => GraphQLErrorCode.Unauthorized,
            "FORBIDDEN" => GraphQLErrorCode.Forbidden,
            "BAD_USER_INPUT" or "GRAPHQL_VALIDATION_FAILED" => GraphQLErrorCode.ValidationError,
            "RATE_LIMITED" => GraphQLErrorCode.RateLimited,
            "SERVICE_UNAVAILABLE" => GraphQLErrorCode.ServiceUnavailable,
            "INTERNAL_SERVER_ERROR" => GraphQLErrorCode.InternalError,
            _ => GraphQLErrorCode.Unknown
        };
    }

    // 종합 에러 핸들러
    public class GraphQLErrorHandler
    {
        public Action OnAuthError;
        public Action<string> OnNotFound;
        public Action<string> OnGenericError;

        public void HandleErrors(GraphQLError[] errors)
        {
            foreach (var error in errors)
            {
                var code = ClassifyError(error);
                switch (code)
                {
                    case GraphQLErrorCode.Unauthorized:
                        Debug.LogWarning("[GraphQL] 인증 만료 - 재로그인 필요");
                        OnAuthError?.Invoke();
                        break;
                    case GraphQLErrorCode.NotFound:
                        OnNotFound?.Invoke(error.message);
                        break;
                    default:
                        Debug.LogError($"[GraphQL] 에러: {error.message}");
                        OnGenericError?.Invoke(error.message);
                        break;
                }
            }
        }
    }

    // 부분 성공(Partial Success) 처리
    [Serializable] public class DashboardData
    {
        public PlayerInfo player;
        public GuildInfo guild; // null일 수 있음 (부분 에러)
    }
    [Serializable] public class PlayerInfo { public string name; public int level; }
    [Serializable] public class GuildInfo { public string name; }

    public void HandlePartialResponse(GraphQLResponse<DashboardData> response)
    {
        if (response.data?.player != null)
            Debug.Log($"플레이어 로드 성공: {response.data.player.name}");

        if (response.data?.guild == null)
            Debug.LogWarning("길드 정보를 불러올 수 없습니다");

        if (response.HasErrors)
        {
            foreach (var error in response.errors)
                Debug.LogWarning($"부분 에러 [{string.Join(".", error.path ?? new string[0])}]: {error.message}");
        }
    }
}
```

---

## 8. 캐싱 전략

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;

public class GraphQLCache
{
    private class CacheEntry
    {
        public string Data;
        public DateTime ExpiresAt;
        public bool IsExpired => DateTime.UtcNow > ExpiresAt;
    }

    private readonly Dictionary<string, CacheEntry> cache = new Dictionary<string, CacheEntry>();
    private readonly float defaultTtl;

    public GraphQLCache(float defaultTtlSeconds = 300f) { defaultTtl = defaultTtlSeconds; }

    public string GenerateCacheKey(string query, Dictionary<string, object> variables)
    {
        string vars = variables != null ? string.Join(",", variables) : "";
        return (query.Trim() + "|" + vars).GetHashCode().ToString("x8");
    }

    public bool TryGet(string key, out string data)
    {
        data = null;
        if (cache.TryGetValue(key, out var entry) && !entry.IsExpired)
        {
            data = entry.Data;
            return true;
        }
        cache.Remove(key);
        return false;
    }

    public void Set(string key, string data, float? ttl = null)
    {
        cache[key] = new CacheEntry
        {
            Data = data,
            ExpiresAt = DateTime.UtcNow.AddSeconds(ttl ?? defaultTtl)
        };
    }

    public void Invalidate(string key) => cache.Remove(key);

    public void InvalidateByPattern(string pattern)
    {
        var toRemove = new List<string>();
        foreach (var key in cache.Keys)
            if (key.Contains(pattern)) toRemove.Add(key);
        foreach (var key in toRemove) cache.Remove(key);
    }

    public void ClearAll() => cache.Clear();
}

// 캐시 통합 클라이언트
public class CachedGraphQLClient : MonoBehaviour
{
    [SerializeField] private GraphQLClient innerClient;
    private GraphQLCache cache = new GraphQLCache(120f);

    // 오퍼레이션별 캐시 TTL 정책
    private static readonly Dictionary<string, float> CachePolicies = new()
    {
        { "GetPlayerProfile", 60f },
        { "GetLeaderboard", 30f },
        { "GetItemCatalog", 3600f },
    };

    public void QueryWithCache<T>(string query, Dictionary<string, object> variables,
        Action<T> onSuccess, Action<GraphQLError[]> onError,
        string operationName = null)
    {
        string cacheKey = cache.GenerateCacheKey(query, variables);

        // 캐시 히트 시 즉시 반환
        if (cache.TryGet(cacheKey, out string cached))
        {
            onSuccess?.Invoke(JsonUtility.FromJson<T>(cached));
            return;
        }

        // 캐시 미스: 네트워크 요청 후 캐시 저장
        float ttl = operationName != null && CachePolicies.TryGetValue(operationName, out float t)
            ? t : 120f;

        innerClient.Query<T>(query, variables,
            data => { cache.Set(cacheKey, JsonUtility.ToJson(data), ttl); onSuccess?.Invoke(data); },
            onError, operationName);
    }

    public void MutateAndInvalidate<T>(string mutation, Dictionary<string, object> variables,
        string[] invalidateOps, Action<T> onSuccess, Action<GraphQLError[]> onError)
    {
        innerClient.Mutate<T>(mutation, variables,
            data =>
            {
                foreach (var op in invalidateOps) cache.InvalidateByPattern(op);
                onSuccess?.Invoke(data);
            },
            onError);
    }
}
```

---

## 9. 게임에서의 GraphQL 활용 사례

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;

public class GameGraphQLExamples : MonoBehaviour
{
    [SerializeField] private GraphQLClient graphQL;

    // =============================================
    // 사례 1: 리더보드 시스템
    // =============================================

    [Serializable] public class LeaderboardData { public LeaderboardResult leaderboard; }
    [Serializable] public class LeaderboardResult
    {
        public LeaderboardEntry[] entries;
        public int myRank;
    }
    [Serializable] public class LeaderboardEntry
    {
        public int rank; public string playerName; public int score;
    }

    public void FetchLeaderboard(string gameMode, int season)
    {
        string query = @"
            query GetLeaderboard($gameMode: GameMode!, $season: Int!, $pageSize: Int!) {
                leaderboard(gameMode: $gameMode, season: $season, pageSize: $pageSize) {
                    entries { rank playerName score }
                    myRank
                }
            }";

        graphQL.Query<LeaderboardData>(query,
            new Dictionary<string, object>
            {
                { "gameMode", gameMode }, { "season", season }, { "pageSize", 50 }
            },
            data => Debug.Log($"내 순위: #{data.leaderboard.myRank}"),
            errors => Debug.LogError("리더보드 로드 실패")
        );
    }

    // =============================================
    // 사례 2: 인벤토리 아이템 사용 (Mutation)
    // =============================================

    [Serializable] public class UseItemResult { public UseItemPayload useItem; }
    [Serializable] public class UseItemPayload
    {
        public bool success;
        public string message;
        public InventorySnapshot updatedInventory;
    }
    [Serializable] public class InventorySnapshot
    {
        public InventoryItem[] items;
        public int usedSlots; public int maxSlots;
    }
    [Serializable] public class InventoryItem
    {
        public string id; public string name; public int quantity;
    }

    public void UseItem(string itemId)
    {
        graphQL.Mutate<UseItemResult>(
            @"mutation UseItem($itemId: ID!) {
                useItem(itemId: $itemId) {
                    success message
                    updatedInventory {
                        items { id name quantity }
                        usedSlots maxSlots
                    }
                }
            }",
            new Dictionary<string, object> { { "itemId", itemId } },
            data => Debug.Log($"슬롯: {data.useItem.updatedInventory.usedSlots}" +
                              $"/{data.useItem.updatedInventory.maxSlots}"),
            errors => Debug.LogError("아이템 사용 실패")
        );
    }

    // =============================================
    // 사례 3: 게임 시작 시 초기 데이터 일괄 로드
    // =============================================
    // REST였다면 최소 4개의 API 호출이 필요하지만
    // GraphQL은 단일 쿼리로 모든 데이터를 가져옵니다

    [Serializable] public class GameStartPayload
    {
        public PlayerProfile player;
        public DailyReward[] dailyRewards;
        public Announcement[] announcements;
        public ServerStatus serverStatus;
    }
    [Serializable] public class PlayerProfile { public string name; public int level; }
    [Serializable] public class DailyReward { public int day; public bool claimed; }
    [Serializable] public class Announcement { public string id; public string title; }
    [Serializable] public class ServerStatus { public bool maintenance; public string version; }

    public void LoadGameStartData()
    {
        string query = @"
            query GameStartData($playerId: ID!) {
                player(id: $playerId) { name level }
                dailyRewards(playerId: $playerId) { day claimed }
                announcements(limit: 5) { id title }
                serverStatus { maintenance version }
            }";

        graphQL.Query<GameStartPayload>(query,
            new Dictionary<string, object> { { "playerId", "current-player-id" } },
            data => Debug.Log($"초기 데이터 로드 완료: v{data.serverStatus.version}"),
            errors => Debug.LogError("초기 데이터 로드 실패"),
            operationName: "GameStartData"
        );
    }

    // =============================================
    // 사례 4: 상점 복합 쿼리
    // =============================================

    public void LoadShopPage()
    {
        string query = @"
            query LoadShop($playerId: ID!) {
                shop {
                    dailyDeals { itemId name originalPrice discountedPrice expiresAt }
                    featured { itemId name price }
                }
                player(id: $playerId) {
                    currencies { gold gems tickets }
                }
            }";

        graphQL.Query<ShopPageData>(query,
            new Dictionary<string, object> { { "playerId", "current-player-id" } },
            data => Debug.Log("상점 로드 완료"),
            errors => Debug.LogError("상점 로드 실패"),
            operationName: "LoadShop"
        );
    }

    [Serializable] public class ShopPageData { public ShopInfo shop; }
    [Serializable] public class ShopInfo { public DealItem[] dailyDeals; }
    [Serializable] public class DealItem
    {
        public string itemId; public string name;
        public int originalPrice; public int discountedPrice;
    }
}
```

---

## 주의사항

### 1. HTTP 200이어도 에러가 존재할 수 있다

```csharp
// ❌ HTTP 상태 코드만으로 성공/실패 판단
if (webRequest.result == UnityWebRequest.Result.Success)
{
    var data = JsonUtility.FromJson<MyData>(webRequest.downloadHandler.text);
    UseData(data); // errors를 확인하지 않아 null 데이터 사용 위험
}

// ✅ HTTP 상태 + GraphQL errors 모두 확인
if (webRequest.result == UnityWebRequest.Result.Success)
{
    var response = JsonUtility.FromJson<GraphQLResponse<MyData>>(webRequest.downloadHandler.text);
    if (response.HasErrors) HandleGraphQLErrors(response.errors);
    if (response.data != null) UseData(response.data);
}
```

### 2. 필요한 필드만 요청하라

```csharp
// ❌ 모든 필드를 항상 요청 (Over-fetching)
string query = @"query {
    player(id: ""123"") {
        id name level experience health mana
        inventory { id name description rarity level stats weight icon }
        guild { id name description members { id name level } }
    }
}";

// ✅ 화면에 표시할 필드만 요청
string query = @"query GetPlayerCard($id: ID!) {
    player(id: $id) { name level guild { name } }
}";
```

### 3. 변수를 쿼리에 직접 삽입하지 마라

```csharp
// ❌ 인젝션 위험
string query = $@"query {{ player(id: ""{userInput}"") {{ name }} }}";

// ✅ GraphQL 변수 사용
string query = @"query GetPlayer($id: ID!) { player(id: $id) { name } }";
var variables = new Dictionary<string, object> { { "id", userInput } };
```

### 4. Subscription 생명주기를 관리하라

```csharp
// ❌ 연결 해제 없이 방치 (메모리 누수, 유령 연결)
void Start() { subClient.Subscribe(query, variables, OnData); }

// ✅ 생명주기에 맞춰 정리
private string subId;
async void Start() { subId = await subClient.Subscribe(query, variables, OnData); }
async void OnDestroy()
{
    if (subId != null) await subClient.Unsubscribe(subId);
    subClient?.Dispose();
}
```

### 5. Mutation 후 캐시를 갱신하라

```csharp
// ❌ Mutation 후 캐시를 갱신하지 않음 (stale 데이터)
graphQL.Mutate<Result>(buyMutation, vars, data => Debug.Log("구매 완료"), onError);

// ✅ Mutation 응답에 갱신된 데이터를 포함하거나 캐시 무효화
graphQL.Mutate<BuyResult>(buyMutation, vars,
    data => {
        UpdateInventoryUI(data.updatedInventory);
        cache.InvalidateByPattern("GetInventory");
    }, onError);
```

---

## 베스트 프랙티스

### 1. 오퍼레이션 이름을 항상 명시하라

```csharp
// ✅ 서버 로깅, 디버깅, 캐시 키에 활용 가능
graphQL.Query<QuestData>(
    @"query FetchDailyQuests($playerId: ID!) {
        dailyQuests(playerId: $playerId) { id title reward }
    }",
    variables, onSuccess, onError,
    operationName: "FetchDailyQuests");
```

### 2. 프래그먼트 상수로 중복 제거

```csharp
// ✅ 프로젝트 전체에서 공유하는 프래그먼트 정의
public static class GameFragments
{
    public const string ItemBasic = @"
        fragment ItemBasic on Item { id name rarity icon }";
    public const string PlayerSummary = @"
        fragment PlayerSummary on Player { id name level avatar }";
}

string query = GameFragments.ItemBasic + @"
    query GetRewards($questId: ID!) {
        questRewards(questId: $questId) { ...ItemBasic quantity }
    }";
```

### 3. 네트워크 상태에 따라 요청 필드를 조절하라

```csharp
// ✅ 저대역폭 환경에서는 최소한의 데이터만 요청
public string GetPlayerQuery(bool isLowBandwidth) => isLowBandwidth
    ? @"query GetPlayer($id: ID!) { player(id: $id) { name level } }"
    : @"query GetPlayer($id: ID!) {
          player(id: $id) {
              name level experience avatar
              guild { name emblem }
              recentMatches(limit: 5) { result score }
          }
      }";
```

### 4. 재시도 + 캐시 폴백으로 복원력을 확보하라

```csharp
// ✅ 네트워크 실패 시 캐시된 데이터로 폴백
public async Task<T> ResilientQuery<T>(AsyncGraphQLClient client,
    string query, Dictionary<string, object> variables,
    GraphQLCache cache, int maxRetries = 3)
{
    string cacheKey = cache.GenerateCacheKey(query, variables);

    for (int attempt = 0; attempt < maxRetries; attempt++)
    {
        try
        {
            var result = await client.QueryAsync<T>(query, variables);
            cache.Set(cacheKey, JsonUtility.ToJson(result));
            return result;
        }
        catch (GraphQLNetworkException)
        {
            if (attempt == maxRetries - 1 && cache.TryGet(cacheKey, out string cached))
            {
                Debug.LogWarning("[GraphQL] 캐시된 데이터로 폴백");
                return JsonUtility.FromJson<T>(cached);
            }
            await Task.Delay((int)Mathf.Pow(2, attempt) * 1000);
        }
    }
    throw new Exception("최대 재시도 횟수 초과");
}
```

### 5. 쿼리 복잡도를 관리하라

```csharp
// ✅ 깊은 중첩을 피하고 페이지네이션을 적용
string query = @"
    query GetGuildMembers($guildId: ID!, $first: Int!, $after: String) {
        guild(id: $guildId) {
            name
            members(first: $first, after: $after) {
                edges { node { name level } }
                pageInfo { hasNextPage endCursor }
            }
        }
    }";
```

---

## 참고 자료

- [GraphQL 공식 사양](https://spec.graphql.org/)
- [GraphQL 공식 학습 사이트](https://graphql.org/learn/)
- [graphql-transport-ws 프로토콜](https://github.com/enisdenjo/graphql-ws/blob/master/PROTOCOL.md)
- [Unity: UnityWebRequest](https://docs.unity3d.com/ScriptReference/Networking.UnityWebRequest.html)
- [Unity: JsonUtility](https://docs.unity3d.com/ScriptReference/JsonUtility.html)

---

## 다음 섹션

[34. gRPC](./34-grpc.md)
