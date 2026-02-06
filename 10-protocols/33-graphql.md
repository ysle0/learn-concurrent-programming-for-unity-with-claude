# Section 33: GraphQL

## 개요

GraphQL은 Facebook이 개발한 쿼리 언어로, 클라이언트가 필요한 데이터를 정확히 요청할 수 있습니다. REST API와 달리 단일 엔드포인트에서 유연한 데이터 조회가 가능하며, Over-fetching과 Under-fetching 문제를 해결합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    GraphQL vs REST                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   REST API:                                                      │
│   GET /players/123                  → 전체 플레이어 데이터        │
│   GET /players/123/inventory        → 인벤토리만                 │
│   GET /players/123/stats            → 스탯만                     │
│   ▲ 여러 요청 필요 (Under-fetching)                              │
│   ▲ 불필요한 데이터 포함 (Over-fetching)                          │
│                                                                  │
│   GraphQL:                                                       │
│   POST /graphql                                                  │
│   query {                           → 필요한 것만 정확히 요청     │
│     player(id: "123") {                                          │
│       name                                                       │
│       level                                                      │
│       inventory { name quantity }                                │
│     }                                                            │
│   }                                                              │
│   ▲ 단일 요청으로 필요한 데이터만 조회                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## GraphQL 핵심 개념

| 개념 | 설명 | 예시 |
|-----|------|------|
| **Query** | 데이터 조회 (읽기) | `query { player { name } }` |
| **Mutation** | 데이터 수정 (쓰기) | `mutation { updatePlayer(...) }` |
| **Subscription** | 실시간 업데이트 | `subscription { onPlayerUpdate }` |
| **Schema** | 타입 정의 | `type Player { name: String! }` |
| **Resolver** | 데이터 반환 로직 | 서버 측 함수 |

---

## Unity에서 GraphQL 클라이언트

### 기본 GraphQL 클라이언트

```csharp
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// Unity용 GraphQL 클라이언트
/// </summary>
public class GraphQLClient : IDisposable
{
    private readonly HttpClient _httpClient;
    private readonly string _endpoint;

    public GraphQLClient(string endpoint)
    {
        _endpoint = endpoint;
        _httpClient = new HttpClient();
        _httpClient.DefaultRequestHeaders.Accept.Add(
            new System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("application/json"));
    }

    /// <summary>
    /// 인증 토큰 설정
    /// </summary>
    public void SetAuthToken(string token)
    {
        _httpClient.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
    }

    /// <summary>
    /// GraphQL 쿼리 실행
    /// </summary>
    public async Task<GraphQLResponse<T>> QueryAsync<T>(
        string query,
        object variables = null,
        CancellationToken ct = default)
    {
        return await ExecuteAsync<T>(query, variables, ct);
    }

    /// <summary>
    /// GraphQL Mutation 실행
    /// </summary>
    public async Task<GraphQLResponse<T>> MutateAsync<T>(
        string mutation,
        object variables = null,
        CancellationToken ct = default)
    {
        return await ExecuteAsync<T>(mutation, variables, ct);
    }

    /// <summary>
    /// GraphQL 요청 실행
    /// </summary>
    private async Task<GraphQLResponse<T>> ExecuteAsync<T>(
        string query,
        object variables,
        CancellationToken ct)
    {
        var request = new GraphQLRequest
        {
            query = query,
            variables = variables
        };

        string json = JsonUtility.ToJson(request);
        using var content = new StringContent(json, Encoding.UTF8, "application/json");

        var response = await _httpClient.PostAsync(_endpoint, content, ct);
        string responseJson = await response.Content.ReadAsStringAsync();

        if (!response.IsSuccessStatusCode)
        {
            throw new GraphQLException($"HTTP {response.StatusCode}: {responseJson}");
        }

        return JsonUtility.FromJson<GraphQLResponse<T>>(responseJson);
    }

    public void Dispose()
    {
        _httpClient?.Dispose();
    }
}

/// <summary>
/// GraphQL 요청 구조
/// </summary>
[Serializable]
public class GraphQLRequest
{
    public string query;
    public object variables;
    public string operationName;
}

/// <summary>
/// GraphQL 응답 구조
/// </summary>
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
}

[Serializable]
public class GraphQLErrorLocation
{
    public int line;
    public int column;
}

/// <summary>
/// GraphQL 예외
/// </summary>
public class GraphQLException : Exception
{
    public GraphQLError[] Errors { get; }

    public GraphQLException(string message) : base(message) { }

    public GraphQLException(GraphQLError[] errors)
        : base(errors?[0]?.message ?? "Unknown GraphQL error")
    {
        Errors = errors;
    }
}
```

### 쿼리 빌더

```csharp
using System.Text;

/// <summary>
/// GraphQL 쿼리 빌더
/// </summary>
public class GraphQLQueryBuilder
{
    private readonly StringBuilder _sb = new StringBuilder();
    private int _indentLevel = 0;

    public GraphQLQueryBuilder Query(string name = null)
    {
        _sb.Append("query");
        if (!string.IsNullOrEmpty(name))
        {
            _sb.Append(" ").Append(name);
        }
        return this;
    }

    public GraphQLQueryBuilder Mutation(string name = null)
    {
        _sb.Append("mutation");
        if (!string.IsNullOrEmpty(name))
        {
            _sb.Append(" ").Append(name);
        }
        return this;
    }

    public GraphQLQueryBuilder Variables(string variables)
    {
        _sb.Append("(").Append(variables).Append(")");
        return this;
    }

    public GraphQLQueryBuilder Begin()
    {
        _sb.AppendLine(" {");
        _indentLevel++;
        return this;
    }

    public GraphQLQueryBuilder End()
    {
        _indentLevel--;
        AppendIndent();
        _sb.AppendLine("}");
        return this;
    }

    public GraphQLQueryBuilder Field(string name, string args = null)
    {
        AppendIndent();
        _sb.Append(name);
        if (!string.IsNullOrEmpty(args))
        {
            _sb.Append("(").Append(args).Append(")");
        }
        _sb.AppendLine();
        return this;
    }

    public GraphQLQueryBuilder Object(string name, string args = null)
    {
        AppendIndent();
        _sb.Append(name);
        if (!string.IsNullOrEmpty(args))
        {
            _sb.Append("(").Append(args).Append(")");
        }
        _sb.AppendLine(" {");
        _indentLevel++;
        return this;
    }

    public GraphQLQueryBuilder EndObject()
    {
        _indentLevel--;
        AppendIndent();
        _sb.AppendLine("}");
        return this;
    }

    public GraphQLQueryBuilder Fragment(string name, string onType)
    {
        _sb.Append("fragment ").Append(name).Append(" on ").Append(onType);
        return this;
    }

    public GraphQLQueryBuilder UseFragment(string name)
    {
        AppendIndent();
        _sb.Append("...").AppendLine(name);
        return this;
    }

    private void AppendIndent()
    {
        for (int i = 0; i < _indentLevel; i++)
        {
            _sb.Append("  ");
        }
    }

    public override string ToString() => _sb.ToString();
}
```

---

## 게임 API 예제

### 데이터 타입 정의

```csharp
using System;
using UnityEngine;

/// <summary>
/// GraphQL 응답 데이터 타입들
/// </summary>
namespace Game.GraphQL
{
    [Serializable]
    public class PlayerQueryResponse
    {
        public PlayerData player;
    }

    [Serializable]
    public class PlayersQueryResponse
    {
        public PlayerData[] players;
    }

    [Serializable]
    public class PlayerData
    {
        public string id;
        public string name;
        public int level;
        public long experience;
        public int gold;
        public int gems;
        public PlayerStats stats;
        public InventoryItem[] inventory;
        public Achievement[] achievements;
    }

    [Serializable]
    public class PlayerStats
    {
        public float health;
        public float maxHealth;
        public float mana;
        public float maxMana;
        public int attack;
        public int defense;
    }

    [Serializable]
    public class InventoryItem
    {
        public string id;
        public string name;
        public string description;
        public int quantity;
        public string rarity;
        public ItemStats stats;
    }

    [Serializable]
    public class ItemStats
    {
        public int attack;
        public int defense;
        public float critChance;
    }

    [Serializable]
    public class Achievement
    {
        public string id;
        public string name;
        public string description;
        public bool unlocked;
        public long unlockedAt;
    }

    [Serializable]
    public class LeaderboardResponse
    {
        public LeaderboardData leaderboard;
    }

    [Serializable]
    public class LeaderboardData
    {
        public string id;
        public string name;
        public LeaderboardEntry[] entries;
        public LeaderboardEntry myRank;
    }

    [Serializable]
    public class LeaderboardEntry
    {
        public int rank;
        public string playerId;
        public string playerName;
        public long score;
    }

    // Mutation 응답
    [Serializable]
    public class LoginResponse
    {
        public LoginResult login;
    }

    [Serializable]
    public class LoginResult
    {
        public bool success;
        public string token;
        public PlayerData player;
        public string errorMessage;
    }

    [Serializable]
    public class UpdatePlayerResponse
    {
        public PlayerData updatePlayer;
    }

    [Serializable]
    public class UseItemResponse
    {
        public UseItemResult useItem;
    }

    [Serializable]
    public class UseItemResult
    {
        public bool success;
        public InventoryItem item;
        public PlayerStats updatedStats;
    }
}
```

### 게임 API 서비스

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using Game.GraphQL;
using UnityEngine;

/// <summary>
/// 게임 GraphQL API 서비스
/// </summary>
public class GameGraphQLService : IDisposable
{
    private readonly GraphQLClient _client;

    public GameGraphQLService(string endpoint)
    {
        _client = new GraphQLClient(endpoint);
    }

    public void SetAuthToken(string token)
    {
        _client.SetAuthToken(token);
    }

    #region Queries

    /// <summary>
    /// 플레이어 정보 조회
    /// </summary>
    public async Task<PlayerData> GetPlayerAsync(
        string playerId,
        CancellationToken ct = default)
    {
        const string query = @"
            query GetPlayer($id: ID!) {
                player(id: $id) {
                    id
                    name
                    level
                    experience
                    gold
                    gems
                    stats {
                        health
                        maxHealth
                        mana
                        maxMana
                        attack
                        defense
                    }
                }
            }";

        var response = await _client.QueryAsync<PlayerQueryResponse>(
            query,
            new { id = playerId },
            ct);

        if (response.HasErrors)
        {
            throw new GraphQLException(response.errors);
        }

        return response.data.player;
    }

    /// <summary>
    /// 플레이어 인벤토리 조회
    /// </summary>
    public async Task<InventoryItem[]> GetInventoryAsync(
        string playerId,
        int limit = 100,
        CancellationToken ct = default)
    {
        const string query = @"
            query GetInventory($playerId: ID!, $limit: Int) {
                player(id: $playerId) {
                    inventory(limit: $limit) {
                        id
                        name
                        description
                        quantity
                        rarity
                        stats {
                            attack
                            defense
                            critChance
                        }
                    }
                }
            }";

        var response = await _client.QueryAsync<PlayerQueryResponse>(
            query,
            new { playerId, limit },
            ct);

        if (response.HasErrors)
        {
            throw new GraphQLException(response.errors);
        }

        return response.data.player?.inventory ?? new InventoryItem[0];
    }

    /// <summary>
    /// 리더보드 조회
    /// </summary>
    public async Task<LeaderboardData> GetLeaderboardAsync(
        string leaderboardId,
        int top = 100,
        CancellationToken ct = default)
    {
        const string query = @"
            query GetLeaderboard($id: ID!, $top: Int) {
                leaderboard(id: $id) {
                    id
                    name
                    entries(top: $top) {
                        rank
                        playerId
                        playerName
                        score
                    }
                    myRank {
                        rank
                        score
                    }
                }
            }";

        var response = await _client.QueryAsync<LeaderboardResponse>(
            query,
            new { id = leaderboardId, top },
            ct);

        if (response.HasErrors)
        {
            throw new GraphQLException(response.errors);
        }

        return response.data.leaderboard;
    }

    #endregion

    #region Mutations

    /// <summary>
    /// 로그인
    /// </summary>
    public async Task<LoginResult> LoginAsync(
        string username,
        string password,
        CancellationToken ct = default)
    {
        const string mutation = @"
            mutation Login($username: String!, $password: String!) {
                login(username: $username, password: $password) {
                    success
                    token
                    player {
                        id
                        name
                        level
                    }
                    errorMessage
                }
            }";

        var response = await _client.MutateAsync<LoginResponse>(
            mutation,
            new { username, password },
            ct);

        if (response.HasErrors)
        {
            throw new GraphQLException(response.errors);
        }

        var result = response.data.login;
        if (result.success && !string.IsNullOrEmpty(result.token))
        {
            SetAuthToken(result.token);
        }

        return result;
    }

    /// <summary>
    /// 닉네임 변경
    /// </summary>
    public async Task<PlayerData> UpdateNicknameAsync(
        string newNickname,
        CancellationToken ct = default)
    {
        const string mutation = @"
            mutation UpdateNickname($name: String!) {
                updatePlayer(input: { name: $name }) {
                    id
                    name
                    level
                }
            }";

        var response = await _client.MutateAsync<UpdatePlayerResponse>(
            mutation,
            new { name = newNickname },
            ct);

        if (response.HasErrors)
        {
            throw new GraphQLException(response.errors);
        }

        return response.data.updatePlayer;
    }

    /// <summary>
    /// 아이템 사용
    /// </summary>
    public async Task<UseItemResult> UseItemAsync(
        string itemId,
        int quantity = 1,
        CancellationToken ct = default)
    {
        const string mutation = @"
            mutation UseItem($itemId: ID!, $quantity: Int!) {
                useItem(itemId: $itemId, quantity: $quantity) {
                    success
                    item {
                        id
                        name
                        quantity
                    }
                    updatedStats {
                        health
                        mana
                    }
                }
            }";

        var response = await _client.MutateAsync<UseItemResponse>(
            mutation,
            new { itemId, quantity },
            ct);

        if (response.HasErrors)
        {
            throw new GraphQLException(response.errors);
        }

        return response.data.useItem;
    }

    /// <summary>
    /// 점수 제출
    /// </summary>
    public async Task<LeaderboardEntry> SubmitScoreAsync(
        string leaderboardId,
        long score,
        CancellationToken ct = default)
    {
        const string mutation = @"
            mutation SubmitScore($leaderboardId: ID!, $score: Long!) {
                submitScore(leaderboardId: $leaderboardId, score: $score) {
                    rank
                    playerId
                    playerName
                    score
                }
            }";

        var response = await _client.MutateAsync<SubmitScoreResponse>(
            mutation,
            new { leaderboardId, score },
            ct);

        if (response.HasErrors)
        {
            throw new GraphQLException(response.errors);
        }

        return response.data.submitScore;
    }

    [Serializable]
    private class SubmitScoreResponse
    {
        public LeaderboardEntry submitScore;
    }

    #endregion

    public void Dispose()
    {
        _client?.Dispose();
    }
}
```

### Unity 통합 예제

```csharp
using System;
using System.Threading;
using Cysharp.Threading.Tasks;
using Game.GraphQL;
using UnityEngine;
using UnityEngine.UI;

/// <summary>
/// GraphQL 사용 예시 (Unity MonoBehaviour)
/// </summary>
public class GraphQLExample : MonoBehaviour
{
    [SerializeField] private string graphqlEndpoint = "https://api.mygame.com/graphql";
    [SerializeField] private Text statusText;

    private GameGraphQLService _api;
    private CancellationTokenSource _cts;

    private void Awake()
    {
        _api = new GameGraphQLService(graphqlEndpoint);
        _cts = new CancellationTokenSource();
    }

    private async void Start()
    {
        await LoginAndLoadDataAsync();
    }

    private async UniTask LoginAndLoadDataAsync()
    {
        try
        {
            UpdateStatus("로그인 중...");

            // 로그인
            var loginResult = await _api.LoginAsync(
                "player@example.com",
                "password123",
                _cts.Token);

            if (!loginResult.success)
            {
                UpdateStatus($"로그인 실패: {loginResult.errorMessage}");
                return;
            }

            UpdateStatus($"환영합니다, {loginResult.player.name}!");

            // 상세 정보 로드
            await LoadPlayerDetailsAsync(loginResult.player.id);
        }
        catch (GraphQLException ex)
        {
            UpdateStatus($"GraphQL 오류: {ex.Message}");
            Debug.LogError($"GraphQL 오류: {ex}");
        }
        catch (Exception ex)
        {
            UpdateStatus($"오류: {ex.Message}");
            Debug.LogError(ex);
        }
    }

    private async UniTask LoadPlayerDetailsAsync(string playerId)
    {
        // 플레이어 정보
        var player = await _api.GetPlayerAsync(playerId, _cts.Token);
        Debug.Log($"레벨: {player.level}, 골드: {player.gold}");

        // 인벤토리
        var inventory = await _api.GetInventoryAsync(playerId, 50, _cts.Token);
        Debug.Log($"인벤토리 아이템: {inventory.Length}개");

        foreach (var item in inventory)
        {
            Debug.Log($"  - {item.name} x{item.quantity} ({item.rarity})");
        }

        // 리더보드
        var leaderboard = await _api.GetLeaderboardAsync("weekly", 10, _cts.Token);
        Debug.Log($"=== {leaderboard.name} ===");
        foreach (var entry in leaderboard.entries)
        {
            Debug.Log($"  {entry.rank}. {entry.playerName}: {entry.score}");
        }

        if (leaderboard.myRank != null)
        {
            Debug.Log($"내 순위: {leaderboard.myRank.rank}위 ({leaderboard.myRank.score}점)");
        }

        UpdateStatus("데이터 로드 완료");
    }

    public async void OnUsePotion()
    {
        try
        {
            var result = await _api.UseItemAsync("potion_health_001", 1, _cts.Token);

            if (result.success)
            {
                Debug.Log($"포션 사용! 남은 수량: {result.item.quantity}");
                Debug.Log($"현재 HP: {result.updatedStats.health}");
            }
        }
        catch (GraphQLException ex)
        {
            Debug.LogError($"아이템 사용 실패: {ex.Message}");
        }
    }

    private void UpdateStatus(string message)
    {
        Debug.Log(message);
        if (statusText != null)
        {
            statusText.text = message;
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

## GraphQL Subscription (실시간)

### WebSocket 기반 Subscription

```csharp
using System;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// GraphQL Subscription 클라이언트 (graphql-ws 프로토콜)
/// </summary>
public class GraphQLSubscriptionClient : IDisposable
{
    private ClientWebSocket _socket;
    private CancellationTokenSource _receiveCts;
    private readonly string _endpoint;
    private readonly string _authToken;
    private int _nextId = 1;

    public event Action OnConnected;
    public event Action<string> OnError;

    public GraphQLSubscriptionClient(string wsEndpoint, string authToken = null)
    {
        _endpoint = wsEndpoint;
        _authToken = authToken;
    }

    /// <summary>
    /// WebSocket 연결
    /// </summary>
    public async Task ConnectAsync(CancellationToken ct = default)
    {
        _socket = new ClientWebSocket();
        _socket.Options.AddSubProtocol("graphql-transport-ws");

        if (!string.IsNullOrEmpty(_authToken))
        {
            _socket.Options.SetRequestHeader("Authorization", $"Bearer {_authToken}");
        }

        await _socket.ConnectAsync(new Uri(_endpoint), ct);

        // 연결 초기화 메시지
        await SendMessageAsync(new
        {
            type = "connection_init",
            payload = new { }
        }, ct);

        // 수신 루프 시작
        _receiveCts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        _ = ReceiveLoopAsync(_receiveCts.Token);

        OnConnected?.Invoke();
    }

    /// <summary>
    /// Subscription 구독
    /// </summary>
    public async Task<int> SubscribeAsync<T>(
        string subscription,
        object variables,
        Action<T> onData,
        Action<string> onError = null,
        CancellationToken ct = default)
    {
        var id = _nextId++;

        // Subscribe 메시지 전송
        await SendMessageAsync(new
        {
            id = id.ToString(),
            type = "subscribe",
            payload = new
            {
                query = subscription,
                variables
            }
        }, ct);

        // 메시지 핸들러 등록은 ReceiveLoop에서 처리
        // (실제 구현에서는 ID별 핸들러 딕셔너리 관리 필요)

        return id;
    }

    /// <summary>
    /// Subscription 구독 해제
    /// </summary>
    public async Task UnsubscribeAsync(int subscriptionId, CancellationToken ct = default)
    {
        await SendMessageAsync(new
        {
            id = subscriptionId.ToString(),
            type = "complete"
        }, ct);
    }

    /// <summary>
    /// 메시지 전송
    /// </summary>
    private async Task SendMessageAsync(object message, CancellationToken ct)
    {
        string json = JsonUtility.ToJson(message);
        var bytes = Encoding.UTF8.GetBytes(json);
        await _socket.SendAsync(
            new ArraySegment<byte>(bytes),
            WebSocketMessageType.Text,
            true,
            ct);
    }

    /// <summary>
    /// 수신 루프
    /// </summary>
    private async Task ReceiveLoopAsync(CancellationToken ct)
    {
        var buffer = new byte[4096];

        try
        {
            while (_socket.State == WebSocketState.Open && !ct.IsCancellationRequested)
            {
                var result = await _socket.ReceiveAsync(
                    new ArraySegment<byte>(buffer),
                    ct);

                if (result.MessageType == WebSocketMessageType.Text)
                {
                    string json = Encoding.UTF8.GetString(buffer, 0, result.Count);
                    HandleMessage(json);
                }
                else if (result.MessageType == WebSocketMessageType.Close)
                {
                    break;
                }
            }
        }
        catch (OperationCanceledException)
        {
            // 정상 취소
        }
        catch (Exception ex)
        {
            OnError?.Invoke(ex.Message);
        }
    }

    private void HandleMessage(string json)
    {
        // 메시지 타입에 따라 처리
        // { "type": "next", "id": "1", "payload": { "data": {...} } }
        Debug.Log($"Received: {json}");
    }

    /// <summary>
    /// 연결 종료
    /// </summary>
    public async Task DisconnectAsync()
    {
        _receiveCts?.Cancel();

        if (_socket?.State == WebSocketState.Open)
        {
            await _socket.CloseAsync(
                WebSocketCloseStatus.NormalClosure,
                "Client disconnect",
                CancellationToken.None);
        }
    }

    public void Dispose()
    {
        DisconnectAsync().Wait();
        _receiveCts?.Dispose();
        _socket?.Dispose();
    }
}
```

### Subscription 사용 예시

```csharp
using System;
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine;

/// <summary>
/// GraphQL Subscription 사용 예시
/// </summary>
public class GraphQLSubscriptionExample : MonoBehaviour
{
    [SerializeField] private string wsEndpoint = "wss://api.mygame.com/graphql";

    private GraphQLSubscriptionClient _subscriptionClient;
    private int _playerUpdateSubscriptionId;

    [Serializable]
    public class PlayerUpdateData
    {
        public PlayerUpdate playerUpdate;
    }

    [Serializable]
    public class PlayerUpdate
    {
        public string playerId;
        public string eventType;
        public int newLevel;
        public long newExperience;
    }

    private async void Start()
    {
        _subscriptionClient = new GraphQLSubscriptionClient(wsEndpoint);

        try
        {
            await _subscriptionClient.ConnectAsync(destroyCancellationToken);

            // 플레이어 업데이트 구독
            await SubscribeToPlayerUpdatesAsync();
        }
        catch (Exception ex)
        {
            Debug.LogError($"Subscription 연결 실패: {ex.Message}");
        }
    }

    private async UniTask SubscribeToPlayerUpdatesAsync()
    {
        const string subscription = @"
            subscription OnPlayerUpdate($playerId: ID!) {
                playerUpdate(playerId: $playerId) {
                    playerId
                    eventType
                    newLevel
                    newExperience
                }
            }";

        _playerUpdateSubscriptionId = await _subscriptionClient.SubscribeAsync<PlayerUpdateData>(
            subscription,
            new { playerId = "my-player-id" },
            data =>
            {
                // 메인 스레드에서 실행
                UniTask.Post(() =>
                {
                    var update = data.playerUpdate;
                    Debug.Log($"플레이어 업데이트: {update.eventType}");

                    if (update.eventType == "LEVEL_UP")
                    {
                        Debug.Log($"레벨 업! 새 레벨: {update.newLevel}");
                        // UI 업데이트 등
                    }
                });
            },
            error => Debug.LogError($"Subscription 오류: {error}"),
            destroyCancellationToken);
    }

    private async void OnDestroy()
    {
        if (_subscriptionClient != null)
        {
            await _subscriptionClient.UnsubscribeAsync(_playerUpdateSubscriptionId);
            await _subscriptionClient.DisconnectAsync();
            _subscriptionClient.Dispose();
        }
    }
}
```

---

## 캐싱 및 최적화

### 쿼리 캐싱

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// GraphQL 쿼리 캐시
/// </summary>
public class GraphQLCache
{
    private readonly Dictionary<string, CacheEntry> _cache = new();
    private readonly TimeSpan _defaultTtl;

    private class CacheEntry
    {
        public object Data;
        public DateTime ExpiresAt;

        public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
    }

    public GraphQLCache(TimeSpan? defaultTtl = null)
    {
        _defaultTtl = defaultTtl ?? TimeSpan.FromMinutes(5);
    }

    /// <summary>
    /// 캐시 키 생성
    /// </summary>
    public static string CreateKey(string query, object variables)
    {
        string varsJson = variables != null ? JsonUtility.ToJson(variables) : "";
        return $"{query.GetHashCode()}:{varsJson.GetHashCode()}";
    }

    /// <summary>
    /// 캐시에서 가져오기
    /// </summary>
    public bool TryGet<T>(string key, out T value)
    {
        if (_cache.TryGetValue(key, out var entry) && !entry.IsExpired)
        {
            value = (T)entry.Data;
            return true;
        }

        value = default;
        return false;
    }

    /// <summary>
    /// 캐시에 저장
    /// </summary>
    public void Set<T>(string key, T value, TimeSpan? ttl = null)
    {
        _cache[key] = new CacheEntry
        {
            Data = value,
            ExpiresAt = DateTime.UtcNow + (ttl ?? _defaultTtl)
        };
    }

    /// <summary>
    /// 캐시 무효화
    /// </summary>
    public void Invalidate(string key)
    {
        _cache.Remove(key);
    }

    /// <summary>
    /// 전체 캐시 클리어
    /// </summary>
    public void Clear()
    {
        _cache.Clear();
    }

    /// <summary>
    /// 만료된 항목 정리
    /// </summary>
    public void Cleanup()
    {
        var expiredKeys = new List<string>();
        foreach (var kvp in _cache)
        {
            if (kvp.Value.IsExpired)
            {
                expiredKeys.Add(kvp.Key);
            }
        }

        foreach (var key in expiredKeys)
        {
            _cache.Remove(key);
        }
    }
}

/// <summary>
/// 캐싱이 적용된 GraphQL 클라이언트
/// </summary>
public class CachedGraphQLClient
{
    private readonly GraphQLClient _client;
    private readonly GraphQLCache _cache;

    public CachedGraphQLClient(GraphQLClient client, GraphQLCache cache = null)
    {
        _client = client;
        _cache = cache ?? new GraphQLCache();
    }

    /// <summary>
    /// 캐싱된 쿼리 실행
    /// </summary>
    public async Task<GraphQLResponse<T>> QueryWithCacheAsync<T>(
        string query,
        object variables = null,
        TimeSpan? cacheTtl = null,
        CancellationToken ct = default)
    {
        string cacheKey = GraphQLCache.CreateKey(query, variables);

        // 캐시 확인
        if (_cache.TryGet<GraphQLResponse<T>>(cacheKey, out var cached))
        {
            return cached;
        }

        // API 호출
        var response = await _client.QueryAsync<T>(query, variables, ct);

        // 성공 시 캐시
        if (!response.HasErrors)
        {
            _cache.Set(cacheKey, response, cacheTtl);
        }

        return response;
    }

    /// <summary>
    /// Mutation 후 관련 캐시 무효화
    /// </summary>
    public async Task<GraphQLResponse<T>> MutateAndInvalidateAsync<T>(
        string mutation,
        object variables,
        string[] invalidateQueries,
        CancellationToken ct = default)
    {
        var response = await _client.MutateAsync<T>(mutation, variables, ct);

        if (!response.HasErrors && invalidateQueries != null)
        {
            foreach (var query in invalidateQueries)
            {
                _cache.Invalidate(GraphQLCache.CreateKey(query, null));
            }
        }

        return response;
    }
}
```

---

## 에러 핸들링

```csharp
using System;
using System.Linq;
using UnityEngine;

/// <summary>
/// GraphQL 에러 핸들링 유틸리티
/// </summary>
public static class GraphQLErrorHandler
{
    /// <summary>
    /// 에러를 사용자 친화적 메시지로 변환
    /// </summary>
    public static string GetUserMessage(GraphQLError[] errors)
    {
        if (errors == null || errors.Length == 0)
        {
            return "알 수 없는 오류가 발생했습니다.";
        }

        var error = errors[0];
        var message = error.message;

        // 에러 코드 기반 메시지 변환
        if (message.Contains("UNAUTHENTICATED"))
        {
            return "로그인이 필요합니다.";
        }
        if (message.Contains("NOT_FOUND"))
        {
            return "요청한 데이터를 찾을 수 없습니다.";
        }
        if (message.Contains("PERMISSION_DENIED"))
        {
            return "권한이 없습니다.";
        }
        if (message.Contains("RATE_LIMITED"))
        {
            return "요청이 너무 많습니다. 잠시 후 다시 시도해주세요.";
        }

        return message;
    }

    /// <summary>
    /// 재시도 가능한 에러인지 확인
    /// </summary>
    public static bool IsRetryable(GraphQLError[] errors)
    {
        if (errors == null) return false;

        return errors.Any(e =>
            e.message.Contains("RATE_LIMITED") ||
            e.message.Contains("SERVICE_UNAVAILABLE") ||
            e.message.Contains("TIMEOUT"));
    }
}

/// <summary>
/// GraphQL 에러 핸들링 예시
/// </summary>
public class ErrorHandlingExample : MonoBehaviour
{
    private GameGraphQLService _api;

    private async void LoadData()
    {
        try
        {
            var player = await _api.GetPlayerAsync("123");
            Debug.Log($"플레이어: {player.name}");
        }
        catch (GraphQLException ex) when (ex.Errors?.Any(e =>
            e.message.Contains("UNAUTHENTICATED")) == true)
        {
            Debug.Log("재로그인 필요");
            // 로그인 화면으로 이동
            ShowLoginScreen();
        }
        catch (GraphQLException ex) when (GraphQLErrorHandler.IsRetryable(ex.Errors))
        {
            Debug.Log("재시도 가능한 오류, 재시도 중...");
            // 재시도 로직
            await RetryWithBackoff(() => _api.GetPlayerAsync("123"));
        }
        catch (GraphQLException ex)
        {
            string message = GraphQLErrorHandler.GetUserMessage(ex.Errors);
            ShowErrorDialog(message);
        }
        catch (Exception ex)
        {
            Debug.LogError($"네트워크 오류: {ex.Message}");
            ShowErrorDialog("네트워크 연결을 확인해주세요.");
        }
    }

    private void ShowLoginScreen() { }
    private void ShowErrorDialog(string message) { }
    private async System.Threading.Tasks.Task<T> RetryWithBackoff<T>(
        Func<System.Threading.Tasks.Task<T>> action) => await action();
}
```

---

## 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────┐
│                    GraphQL 베스트 프랙티스                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 쿼리 최적화                                                   │
│     ├── 필요한 필드만 요청                                        │
│     ├── Fragment로 재사용 가능한 필드 그룹화                       │
│     └── 중첩 깊이 제한 (보통 3-4레벨)                             │
│                                                                  │
│  2. 캐싱                                                         │
│     ├── 정적 데이터는 클라이언트 캐싱                              │
│     ├── Mutation 후 관련 캐시 무효화                              │
│     └── TTL 기반 자동 만료                                       │
│                                                                  │
│  3. 에러 핸들링                                                   │
│     ├── 에러 코드 기반 분기 처리                                   │
│     ├── 사용자 친화적 메시지 변환                                  │
│     └── 재시도 로직 (일시적 오류)                                  │
│                                                                  │
│  4. 보안                                                         │
│     ├── 민감한 데이터는 Mutation으로만 처리                        │
│     ├── Rate Limiting 적용                                       │
│     └── Query Complexity 제한                                    │
│                                                                  │
│  5. 실시간 데이터                                                 │
│     ├── Subscription으로 실시간 업데이트                          │
│     ├── 연결 상태 모니터링                                        │
│     └── 재연결 로직 구현                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [GraphQL Official](https://graphql.org/)
- [GraphQL Specification](https://spec.graphql.org/)
- [graphql-ws Protocol](https://github.com/enisdenjo/graphql-ws)
- [Apollo Client Caching](https://www.apollographql.com/docs/react/caching/overview/)

---

## 다음 단계

- [Section 34: CancellationToken](../11-cancellation-error/34-cancellation-token.md) - 작업 취소 패턴
- [Section 37: 플랫폼별 OS 고려사항](../12-platform/37-os-considerations.md)
