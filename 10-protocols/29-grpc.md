# 29. gRPC

## 개요

gRPC(Google Remote Procedure Call)는 Google이 개발한 고성능 오픈소스 RPC 프레임워크입니다. HTTP/2를 전송 프로토콜로 사용하고, Protocol Buffers(protobuf)를 인터페이스 정의 언어(IDL) 및 직렬화 형식으로 사용합니다. REST API 대비 더 낮은 지연 시간, 양방향 스트리밍, 강력한 타입 안전성을 제공하며, Unity에서 실시간 멀티플레이어 게임, 서버-클라이언트 통신, 마이크로서비스 연동에 활용됩니다.

### gRPC vs REST 비교

| 특성 | gRPC | REST (JSON) |
|------|------|-------------|
| 프로토콜 | HTTP/2 | HTTP/1.1 또는 HTTP/2 |
| 직렬화 | Protocol Buffers (바이너리) | JSON (텍스트) |
| 스트리밍 | 양방향 스트리밍 지원 | 제한적 (SSE, WebSocket 별도) |
| 코드 생성 | .proto에서 자동 생성 | OpenAPI/Swagger (선택적) |
| 성능 | 높음 (바이너리, 멀티플렉싱) | 보통 (텍스트, 파싱 오버헤드) |
| 브라우저 지원 | gRPC-Web 필요 | 네이티브 지원 |
| Unity 호환 | 추가 설정 필요 | UnityWebRequest 기본 지원 |

---

## 1. Protocol Buffers (.proto 파일 정의)

Protocol Buffers는 gRPC의 핵심 직렬화 포맷입니다. `.proto` 파일에서 메시지와 서비스를 정의하고, 이를 기반으로 C# 코드를 자동 생성합니다.

### .proto 파일 기본 구조

```protobuf
// game_service.proto
syntax = "proto3";

package game;

option csharp_namespace = "Game.Grpc";

// =============================================
// 메시지 정의
// =============================================

message PlayerInfo {
    string player_id = 1;
    string display_name = 2;
    int32 level = 3;
    float health = 4;
    Vector3Proto position = 5;
    repeated ItemInfo inventory = 6;  // 배열/리스트
    map<string, string> metadata = 7; // 딕셔너리
    PlayerState state = 8;
}

message Vector3Proto {
    float x = 1;
    float y = 2;
    float z = 3;
}

message ItemInfo {
    string item_id = 1;
    string name = 2;
    int32 quantity = 3;
    ItemType type = 4;
}

// 열거형 정의
enum PlayerState {
    PLAYER_STATE_UNKNOWN = 0;
    PLAYER_STATE_IDLE = 1;
    PLAYER_STATE_MOVING = 2;
    PLAYER_STATE_ATTACKING = 3;
    PLAYER_STATE_DEAD = 4;
}

enum ItemType {
    ITEM_TYPE_UNKNOWN = 0;
    ITEM_TYPE_WEAPON = 1;
    ITEM_TYPE_ARMOR = 2;
    ITEM_TYPE_CONSUMABLE = 3;
}

// =============================================
// 서비스 정의 - 4가지 RPC 패턴
// =============================================

service GameService {
    // Unary RPC: 단일 요청 → 단일 응답
    rpc Login (LoginRequest) returns (LoginResponse);
    rpc GetPlayerInfo (GetPlayerRequest) returns (PlayerInfo);

    // Server Streaming RPC: 단일 요청 → 스트림 응답
    rpc SubscribeWorldEvents (WorldEventRequest) returns (stream WorldEvent);

    // Client Streaming RPC: 스트림 요청 → 단일 응답
    rpc UploadPlayerActions (stream PlayerAction) returns (ActionSummary);

    // Bidirectional Streaming RPC: 스트림 요청 ↔ 스트림 응답
    rpc GameChat (stream ChatMessage) returns (stream ChatMessage);
}

// =============================================
// 요청/응답 메시지
// =============================================

message LoginRequest {
    string username = 1;
    string password = 2;
    string client_version = 3;
}

message LoginResponse {
    bool success = 1;
    string auth_token = 2;
    string error_message = 3;
    PlayerInfo player = 4;
}

message GetPlayerRequest {
    string player_id = 1;
}

message WorldEventRequest {
    string zone_id = 1;
    repeated string event_types = 2;
}

message WorldEvent {
    string event_id = 1;
    string event_type = 2;
    string payload = 3;
    int64 timestamp = 4;
}

message PlayerAction {
    string action_type = 1;
    Vector3Proto position = 2;
    int64 timestamp = 3;
}

message ActionSummary {
    int32 actions_processed = 1;
    float total_distance = 2;
    int64 duration_ms = 3;
}

message ChatMessage {
    string sender_id = 1;
    string sender_name = 2;
    string content = 3;
    int64 timestamp = 4;
    string channel = 5;
}
```

### 코드 생성

```bash
# protoc를 사용한 C# 코드 생성
protoc --csharp_out=./Generated \
       --grpc_out=./Generated \
       --plugin=protoc-gen-grpc=grpc_csharp_plugin \
       game_service.proto

# .csproj에서 자동 생성 설정 (grpc-dotnet 사용 시)
# <Protobuf Include="Protos/game_service.proto" GrpcServices="Client" />
```

```csharp
using UnityEngine;
using Game.Grpc;

/// <summary>
/// 생성된 protobuf 메시지 사용 예시
/// </summary>
public class ProtobufUsageExample : MonoBehaviour
{
    private void Start()
    {
        // =============================================
        // protobuf 메시지 생성 및 사용
        // =============================================

        var player = new PlayerInfo
        {
            PlayerId = "player_001",
            DisplayName = "용사",
            Level = 42,
            Health = 100.0f,
            Position = new Vector3Proto { X = 10, Y = 0, Z = 20 },
            State = PlayerState.Idle
        };

        // 인벤토리 추가 (repeated 필드)
        player.Inventory.Add(new ItemInfo
        {
            ItemId = "sword_01",
            Name = "불꽃 검",
            Quantity = 1,
            Type = ItemType.Weapon
        });

        // 메타데이터 추가 (map 필드)
        player.Metadata["guild"] = "드래곤 슬레이어";
        player.Metadata["region"] = "kr";

        // 바이너리 직렬화 (네트워크 전송용)
        byte[] bytes = player.ToByteArray();
        Debug.Log($"직렬화 크기: {bytes.Length} 바이트");

        // 역직렬화
        PlayerInfo deserialized = PlayerInfo.Parser.ParseFrom(bytes);
        Debug.Log($"플레이어: {deserialized.DisplayName}, Lv.{deserialized.Level}");

        // Unity Vector3 변환
        Vector3 unityPosition = ToUnityVector3(deserialized.Position);
        Debug.Log($"위치: {unityPosition}");
    }

    // =============================================
    // protobuf ↔ Unity 변환 유틸리티
    // =============================================

    public static Vector3 ToUnityVector3(Vector3Proto proto)
    {
        return new Vector3(proto.X, proto.Y, proto.Z);
    }

    public static Vector3Proto ToProtoVector3(Vector3 v)
    {
        return new Vector3Proto { X = v.x, Y = v.y, Z = v.z };
    }
}
```

---

## 2. Unity에서 gRPC 설정

### NuGet 패키지 설치 (grpc-dotnet)

```csharp
// Unity Package Manager 또는 NuGetForUnity를 통해 설치:
// - Google.Protobuf
// - Grpc.Net.Client
// - Grpc.Tools (코드 생성용)

// 참고: Unity 2021+ 에서는 .NET Standard 2.1 또는 .NET Framework를
// 타겟으로 설정해야 합니다.
// Project Settings > Player > Api Compatibility Level > .NET Standard 2.1
```

### gRPC 채널 및 클라이언트 기본 설정

```csharp
using UnityEngine;
using Grpc.Net.Client;
using Grpc.Core;
using System;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using Game.Grpc;

/// <summary>
/// Unity에서 gRPC 채널 설정 및 기본 연결 관리
/// </summary>
public class GrpcConnectionManager : MonoBehaviour
{
    [SerializeField] private string serverAddress = "https://localhost:5001";
    [SerializeField] private int deadlineSeconds = 30;

    private GrpcChannel channel;
    private GameService.GameServiceClient client;
    private CancellationTokenSource lifetimeCts;

    // =============================================
    // 초기화
    // =============================================

    private void Awake()
    {
        lifetimeCts = new CancellationTokenSource();
        InitializeChannel();
    }

    private void InitializeChannel()
    {
        // ✅ 올바른 채널 설정
        var httpHandler = new SocketsHttpHandler
        {
            // 연결 풀링 설정
            PooledConnectionIdleTimeout = TimeSpan.FromMinutes(5),
            KeepAlivePingDelay = TimeSpan.FromSeconds(60),
            KeepAlivePingTimeout = TimeSpan.FromSeconds(30),

            // HTTP/2 전용 설정
            EnableMultipleHttp2Connections = true
        };

        channel = GrpcChannel.ForAddress(serverAddress, new GrpcChannelOptions
        {
            HttpHandler = httpHandler,
            MaxReceiveMessageSize = 16 * 1024 * 1024,  // 16MB
            MaxSendMessageSize = 16 * 1024 * 1024,
            MaxRetryAttempts = 3,

            // 서비스 설정 (재시도 정책 등)
            ServiceConfig = new ServiceConfig
            {
                MethodConfigs =
                {
                    new MethodConfig
                    {
                        Names = { MethodName.Default },
                        RetryPolicy = new RetryPolicy
                        {
                            MaxAttempts = 3,
                            InitialBackoff = TimeSpan.FromSeconds(1),
                            MaxBackoff = TimeSpan.FromSeconds(5),
                            BackoffMultiplier = 1.5,
                            RetryableStatusCodes =
                            {
                                StatusCode.Unavailable,
                                StatusCode.DeadlineExceeded
                            }
                        }
                    }
                }
            }
        });

        client = new GameService.GameServiceClient(channel);
        Debug.Log($"gRPC 채널 생성: {serverAddress}");
    }

    // =============================================
    // 채널 상태 모니터링
    // =============================================

    public GrpcChannel Channel => channel;
    public GameService.GameServiceClient Client => client;

    public CallOptions CreateCallOptions(int? deadlineOverride = null)
    {
        int seconds = deadlineOverride ?? deadlineSeconds;

        return new CallOptions(
            deadline: DateTime.UtcNow.AddSeconds(seconds),
            cancellationToken: lifetimeCts.Token
        );
    }

    // =============================================
    // 정리
    // =============================================

    private async void OnDestroy()
    {
        lifetimeCts?.Cancel();
        lifetimeCts?.Dispose();

        if (channel != null)
        {
            await channel.ShutdownAsync();
            channel.Dispose();
            Debug.Log("gRPC 채널 종료");
        }
    }
}
```

---

## 3. Unary RPC (단일 요청-응답)

가장 기본적인 RPC 패턴으로, REST API의 요청-응답과 유사합니다.

```csharp
using UnityEngine;
using Grpc.Core;
using System;
using System.Threading;
using System.Threading.Tasks;
using Cysharp.Threading.Tasks;
using Game.Grpc;

/// <summary>
/// Unary RPC 호출 예시 - 단일 요청에 대한 단일 응답
/// </summary>
public class UnaryRpcExample : MonoBehaviour
{
    [SerializeField] private GrpcConnectionManager connectionManager;

    private async void Start()
    {
        var token = this.GetCancellationTokenOnDestroy();

        try
        {
            // 로그인
            var loginResult = await LoginAsync("player1", "password123", token);
            if (loginResult.Success)
            {
                Debug.Log($"로그인 성공: {loginResult.Player.DisplayName}");

                // 플레이어 정보 조회
                var playerInfo = await GetPlayerInfoAsync(loginResult.Player.PlayerId, token);
                Debug.Log($"레벨: {playerInfo.Level}, HP: {playerInfo.Health}");
            }
        }
        catch (RpcException ex)
        {
            HandleRpcError(ex);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("요청 취소됨");
        }
    }

    // =============================================
    // Unary RPC 호출
    // =============================================

    private async UniTask<LoginResponse> LoginAsync(
        string username, string password, CancellationToken token)
    {
        var request = new LoginRequest
        {
            Username = username,
            Password = password,
            ClientVersion = Application.version
        };

        // ✅ 데드라인과 취소 토큰 설정
        var options = new CallOptions(
            deadline: DateTime.UtcNow.AddSeconds(10),
            cancellationToken: token
        );

        // Unary 호출: 하나의 요청 → 하나의 응답
        LoginResponse response = await client.LoginAsync(request, options);
        return response;
    }

    private async UniTask<PlayerInfo> GetPlayerInfoAsync(
        string playerId, CancellationToken token)
    {
        var request = new GetPlayerRequest { PlayerId = playerId };

        // 메타데이터(헤더)와 함께 호출
        var headers = new Metadata
        {
            { "authorization", $"Bearer {AuthToken}" },
            { "x-client-version", Application.version }
        };

        var options = new CallOptions(
            headers: headers,
            deadline: DateTime.UtcNow.AddSeconds(15),
            cancellationToken: token
        );

        return await client.GetPlayerInfoAsync(request, options);
    }

    private GameService.GameServiceClient client =>
        connectionManager.Client;

    private string AuthToken { get; set; }

    // =============================================
    // 에러 처리
    // =============================================

    private void HandleRpcError(RpcException ex)
    {
        switch (ex.StatusCode)
        {
            case StatusCode.Unauthenticated:
                Debug.LogError("인증 실패. 다시 로그인하세요.");
                break;
            case StatusCode.DeadlineExceeded:
                Debug.LogError("요청 시간 초과");
                break;
            case StatusCode.Unavailable:
                Debug.LogError("서버에 연결할 수 없습니다");
                break;
            default:
                Debug.LogError($"gRPC 에러 [{ex.StatusCode}]: {ex.Status.Detail}");
                break;
        }
    }
}
```

---

## 4. Server Streaming RPC

서버가 클라이언트의 단일 요청에 대해 여러 응답을 스트리밍합니다. 실시간 이벤트 구독, 대량 데이터 전송 등에 적합합니다.

```csharp
using UnityEngine;
using Grpc.Core;
using System;
using System.Threading;
using Cysharp.Threading.Tasks;
using Game.Grpc;

/// <summary>
/// Server Streaming RPC - 월드 이벤트 실시간 구독
/// </summary>
public class ServerStreamingExample : MonoBehaviour
{
    [SerializeField] private GrpcConnectionManager connectionManager;

    private CancellationTokenSource streamCts;

    private async void Start()
    {
        var destroyToken = this.GetCancellationTokenOnDestroy();
        await SubscribeToWorldEventsAsync(destroyToken);
    }

    // =============================================
    // Server Streaming: 월드 이벤트 구독
    // =============================================

    private async UniTask SubscribeToWorldEventsAsync(CancellationToken token)
    {
        streamCts = CancellationTokenSource.CreateLinkedTokenSource(token);

        var request = new WorldEventRequest
        {
            ZoneId = "zone_01",
            EventTypes = { "monster_spawn", "boss_appear", "weather_change" }
        };

        try
        {
            // 서버 스트리밍 호출 시작
            using var call = connectionManager.Client.SubscribeWorldEvents(
                request,
                new CallOptions(cancellationToken: streamCts.Token));

            // ✅ 응답 스트림을 비동기 반복으로 읽기
            await foreach (var worldEvent in
                call.ResponseStream.ReadAllAsync(streamCts.Token))
            {
                ProcessWorldEvent(worldEvent);
            }

            Debug.Log("서버 스트림 정상 종료");
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.Cancelled)
        {
            Debug.Log("이벤트 구독 취소됨");
        }
        catch (RpcException ex)
        {
            Debug.LogError($"스트리밍 에러: {ex.Status.Detail}");

            // 재연결 시도
            await ReconnectWithBackoff(token);
        }
    }

    // =============================================
    // 이벤트 처리 (메인 스레드에서 실행 보장)
    // =============================================

    private void ProcessWorldEvent(WorldEvent worldEvent)
    {
        // UniTask의 SwitchToMainThread 또는
        // UnitySynchronizationContext를 통해 메인 스레드 보장
        switch (worldEvent.EventType)
        {
            case "monster_spawn":
                Debug.Log($"몬스터 출현: {worldEvent.Payload}");
                // SpawnMonster(worldEvent.Payload);
                break;

            case "boss_appear":
                Debug.Log($"보스 등장: {worldEvent.Payload}");
                // ShowBossAlert(worldEvent.Payload);
                break;

            case "weather_change":
                Debug.Log($"날씨 변경: {worldEvent.Payload}");
                // ChangeWeather(worldEvent.Payload);
                break;

            default:
                Debug.Log($"알 수 없는 이벤트: {worldEvent.EventType}");
                break;
        }
    }

    // =============================================
    // 지수 백오프 재연결
    // =============================================

    private async UniTask ReconnectWithBackoff(CancellationToken token)
    {
        int maxRetries = 5;
        int baseDelayMs = 1000;

        for (int attempt = 1; attempt <= maxRetries; attempt++)
        {
            if (token.IsCancellationRequested) return;

            int delay = baseDelayMs * (int)Math.Pow(2, attempt - 1);
            Debug.Log($"재연결 시도 {attempt}/{maxRetries} ({delay}ms 후)");

            await UniTask.Delay(delay, cancellationToken: token);

            try
            {
                await SubscribeToWorldEventsAsync(token);
                return; // 성공 시 종료
            }
            catch (Exception ex)
            {
                Debug.LogWarning($"재연결 실패: {ex.Message}");
            }
        }

        Debug.LogError("최대 재연결 시도 횟수 초과");
    }

    // =============================================
    // 구독 중지
    // =============================================

    public void StopSubscription()
    {
        streamCts?.Cancel();
        Debug.Log("이벤트 구독 중지 요청");
    }

    private void OnDestroy()
    {
        streamCts?.Cancel();
        streamCts?.Dispose();
    }
}
```

---

## 5. Client Streaming RPC

클라이언트가 여러 메시지를 스트리밍하고, 서버가 모든 메시지를 받은 후 단일 응답을 반환합니다.

```csharp
using UnityEngine;
using Grpc.Core;
using System;
using System.Threading;
using Cysharp.Threading.Tasks;
using Game.Grpc;

/// <summary>
/// Client Streaming RPC - 플레이어 행동 로그 업로드
/// </summary>
public class ClientStreamingExample : MonoBehaviour
{
    [SerializeField] private GrpcConnectionManager connectionManager;
    [SerializeField] private float actionSendInterval = 0.1f;

    private AsyncClientStreamingCall<PlayerAction, ActionSummary> streamCall;
    private bool isStreaming;

    // =============================================
    // Client Streaming: 플레이어 행동 데이터 전송
    // =============================================

    public async UniTask StartActionStreamAsync(CancellationToken token)
    {
        try
        {
            var options = new CallOptions(
                deadline: DateTime.UtcNow.AddMinutes(30),
                cancellationToken: token);

            // 클라이언트 스트리밍 호출 시작
            streamCall = connectionManager.Client.UploadPlayerActions(options);
            isStreaming = true;

            Debug.Log("행동 스트림 시작");

            // 주기적으로 행동 데이터 전송
            while (isStreaming && !token.IsCancellationRequested)
            {
                var action = CaptureCurrentAction();
                await streamCall.RequestStream.WriteAsync(action);

                await UniTask.Delay(
                    TimeSpan.FromSeconds(actionSendInterval),
                    cancellationToken: token);
            }

            // ✅ 스트림 완료 신호 전송
            await streamCall.RequestStream.CompleteAsync();

            // 서버로부터 최종 응답 받기
            ActionSummary summary = await streamCall.ResponseAsync;
            Debug.Log($"행동 요약 - 처리: {summary.ActionsProcessed}건, " +
                      $"이동 거리: {summary.TotalDistance:F1}m, " +
                      $"소요 시간: {summary.DurationMs}ms");
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.Cancelled)
        {
            Debug.Log("행동 스트림 취소됨");
        }
        catch (RpcException ex)
        {
            Debug.LogError($"스트리밍 에러: {ex.Status.Detail}");
        }
        finally
        {
            isStreaming = false;
            streamCall?.Dispose();
        }
    }

    // =============================================
    // 현재 플레이어 행동 캡처
    // =============================================

    private PlayerAction CaptureCurrentAction()
    {
        Vector3 pos = transform.position;

        return new PlayerAction
        {
            ActionType = GetCurrentActionType(),
            Position = new Vector3Proto { X = pos.x, Y = pos.y, Z = pos.z },
            Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
        };
    }

    private string GetCurrentActionType()
    {
        // 실제 게임에서는 현재 플레이어 상태에 따라 반환
        if (Input.GetMouseButton(0)) return "attack";
        if (Input.GetAxis("Horizontal") != 0 || Input.GetAxis("Vertical") != 0)
            return "move";
        return "idle";
    }

    // =============================================
    // 스트림 중지
    // =============================================

    public void StopStreaming()
    {
        isStreaming = false;
    }
}
```

---

## 6. Bidirectional Streaming RPC

클라이언트와 서버가 동시에 메시지를 주고받는 양방향 스트리밍입니다. 실시간 채팅, 멀티플레이어 동기화 등에 적합합니다.

```csharp
using UnityEngine;
using UnityEngine.UI;
using Grpc.Core;
using System;
using System.Collections.Generic;
using System.Threading;
using Cysharp.Threading.Tasks;
using Game.Grpc;

/// <summary>
/// Bidirectional Streaming RPC - 실시간 게임 채팅
/// </summary>
public class BidirectionalStreamingExample : MonoBehaviour
{
    [SerializeField] private GrpcConnectionManager connectionManager;
    [SerializeField] private Text chatDisplay;
    [SerializeField] private InputField chatInput;

    private AsyncDuplexStreamingCall<ChatMessage, ChatMessage> chatStream;
    private CancellationTokenSource chatCts;
    private readonly Queue<string> chatHistory = new Queue<string>();
    private const int MaxChatHistory = 100;

    // =============================================
    // 채팅 스트림 시작
    // =============================================

    public async UniTask StartChatAsync(string playerId, string playerName,
        CancellationToken token)
    {
        chatCts = CancellationTokenSource.CreateLinkedTokenSource(token);

        try
        {
            var options = new CallOptions(cancellationToken: chatCts.Token);

            // 양방향 스트리밍 호출
            chatStream = connectionManager.Client.GameChat(options);

            // ✅ 수신과 송신을 병렬로 처리
            var receiveTask = ReceiveMessagesAsync(chatCts.Token);
            var inputTask = HandleUserInputAsync(playerId, playerName, chatCts.Token);

            // 둘 중 하나가 끝날 때까지 대기
            await UniTask.WhenAny(receiveTask, inputTask);
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.Cancelled)
        {
            Debug.Log("채팅 종료");
        }
        catch (RpcException ex)
        {
            Debug.LogError($"채팅 에러: {ex.Status.Detail}");
        }
        finally
        {
            chatStream?.Dispose();
        }
    }

    // =============================================
    // 메시지 수신 루프
    // =============================================

    private async UniTask ReceiveMessagesAsync(CancellationToken token)
    {
        try
        {
            await foreach (var message in
                chatStream.ResponseStream.ReadAllAsync(token))
            {
                // 메인 스레드에서 UI 업데이트
                await UniTask.SwitchToMainThread();
                DisplayChatMessage(message);
            }
        }
        catch (OperationCanceledException)
        {
            // 정상 취소
        }
    }

    // =============================================
    // 사용자 입력 처리 루프
    // =============================================

    private async UniTask HandleUserInputAsync(
        string playerId, string playerName, CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            // 프레임마다 입력 확인
            await UniTask.Yield(PlayerLoopTiming.Update, token);

            if (Input.GetKeyDown(KeyCode.Return) &&
                !string.IsNullOrWhiteSpace(chatInput.text))
            {
                var message = new ChatMessage
                {
                    SenderId = playerId,
                    SenderName = playerName,
                    Content = chatInput.text,
                    Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
                    Channel = "general"
                };

                // 메시지 전송
                await chatStream.RequestStream.WriteAsync(message);
                chatInput.text = "";
                chatInput.ActivateInputField();
            }
        }
    }

    // =============================================
    // UI 업데이트
    // =============================================

    private void DisplayChatMessage(ChatMessage message)
    {
        string formattedTime = DateTimeOffset
            .FromUnixTimeMilliseconds(message.Timestamp)
            .ToLocalTime()
            .ToString("HH:mm:ss");

        string chatLine = $"[{formattedTime}] {message.SenderName}: {message.Content}";
        chatHistory.Enqueue(chatLine);

        while (chatHistory.Count > MaxChatHistory)
            chatHistory.Dequeue();

        if (chatDisplay != null)
            chatDisplay.text = string.Join("\n", chatHistory);
    }

    // =============================================
    // 정리
    // =============================================

    public async UniTask DisconnectChatAsync()
    {
        if (chatStream != null)
        {
            await chatStream.RequestStream.CompleteAsync();
        }
        chatCts?.Cancel();
    }

    private void OnDestroy()
    {
        chatCts?.Cancel();
        chatCts?.Dispose();
    }
}
```

---

## 7. gRPC-Web (브라우저/Unity WebGL 호환)

표준 gRPC는 HTTP/2 트레일러를 필요로 하기 때문에 브라우저 환경에서 직접 사용할 수 없습니다. gRPC-Web은 이를 해결하기 위한 프로토콜 변환 계층입니다.

```csharp
using UnityEngine;
using Grpc.Net.Client;
using Grpc.Net.Client.Web;
using System;
using System.Net.Http;
using System.Threading;
using Cysharp.Threading.Tasks;
using Game.Grpc;

/// <summary>
/// gRPC-Web 클라이언트 설정 - WebGL 빌드와 브라우저 호환
/// </summary>
public class GrpcWebClientExample : MonoBehaviour
{
    [SerializeField] private string serverAddress = "https://localhost:5001";

    private GrpcChannel channel;
    private GameService.GameServiceClient client;

    // =============================================
    // gRPC-Web 채널 설정
    // =============================================

    private void Awake()
    {
        InitializeGrpcWebChannel();
    }

    private void InitializeGrpcWebChannel()
    {
        // ✅ gRPC-Web 핸들러로 HTTP/1.1 환경에서도 gRPC 사용
        var grpcWebHandler = new GrpcWebHandler(
            GrpcWebMode.GrpcWeb,        // 또는 GrpcWebText (Base64 인코딩)
            new HttpClientHandler()
        );

        channel = GrpcChannel.ForAddress(serverAddress, new GrpcChannelOptions
        {
            HttpHandler = grpcWebHandler,
            MaxReceiveMessageSize = 4 * 1024 * 1024 // 4MB
        });

        client = new GameService.GameServiceClient(channel);
        Debug.Log("gRPC-Web 채널 초기화 완료");
    }

    // =============================================
    // gRPC-Web Unary 호출
    // =============================================

    public async UniTask<LoginResponse> LoginAsync(
        string username, string password, CancellationToken token)
    {
        var request = new LoginRequest
        {
            Username = username,
            Password = password,
            ClientVersion = Application.version
        };

        // gRPC-Web에서도 일반 gRPC와 동일한 API 사용
        return await client.LoginAsync(request,
            deadline: DateTime.UtcNow.AddSeconds(10),
            cancellationToken: token);
    }

    // =============================================
    // gRPC-Web Server Streaming
    // (gRPC-Web은 Client/Bidirectional Streaming 미지원)
    // =============================================

    public async UniTask SubscribeEventsAsync(CancellationToken token)
    {
        var request = new WorldEventRequest { ZoneId = "zone_01" };

        // ✅ Server Streaming은 gRPC-Web에서 지원됨
        using var call = client.SubscribeWorldEvents(
            request, cancellationToken: token);

        await foreach (var evt in call.ResponseStream.ReadAllAsync(token))
        {
            Debug.Log($"[gRPC-Web] 이벤트: {evt.EventType}");
        }
    }

    // =============================================
    // 플랫폼별 채널 팩토리
    // =============================================

    /// <summary>
    /// 플랫폼에 따라 적절한 gRPC 채널 생성
    /// </summary>
    public static GrpcChannel CreatePlatformChannel(string address)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        // WebGL: gRPC-Web 필수
        var handler = new GrpcWebHandler(
            GrpcWebMode.GrpcWebText,
            new HttpClientHandler());

        return GrpcChannel.ForAddress(address, new GrpcChannelOptions
        {
            HttpHandler = handler
        });
#else
        // 네이티브 플랫폼: 표준 gRPC (HTTP/2)
        return GrpcChannel.ForAddress(address, new GrpcChannelOptions
        {
            HttpHandler = new SocketsHttpHandler
            {
                EnableMultipleHttp2Connections = true
            }
        });
#endif
    }

    private async void OnDestroy()
    {
        if (channel != null)
        {
            await channel.ShutdownAsync();
            channel.Dispose();
        }
    }
}
```

### gRPC-Web 제한사항

```
┌──────────────────────────────────────────────────────────────┐
│                    gRPC-Web 지원 매트릭스                     │
├──────────────────────┬──────────────┬────────────────────────┤
│ RPC 패턴             │ gRPC-Web     │ 비고                   │
├──────────────────────┼──────────────┼────────────────────────┤
│ Unary                │ ✅ 지원      │ 완전 지원               │
│ Server Streaming     │ ✅ 지원      │ 완전 지원               │
│ Client Streaming     │ ❌ 미지원    │ HTTP/1.1 제한           │
│ Bidi Streaming       │ ❌ 미지원    │ HTTP/1.1 제한           │
└──────────────────────┴──────────────┴────────────────────────┘
```

---

## 8. MagicOnion (Unity-friendly gRPC 프레임워크)

MagicOnion은 .proto 파일 없이 C# 인터페이스만으로 gRPC 서비스를 정의할 수 있는 Unity 친화적 프레임워크입니다. 서버와 클라이언트 간에 인터페이스를 공유하여 타입 안전성을 보장합니다.

### 서비스 인터페이스 정의 (공유 프로젝트)

```csharp
using MagicOnion;
using MessagePack;
using System.Threading.Tasks;

// =============================================
// 공유 인터페이스 정의 (서버/클라이언트 공통)
// =============================================

/// <summary>
/// Unary 서비스 인터페이스
/// IService<T>를 상속하면 Unary RPC가 됨
/// </summary>
public interface IAccountService : IService<IAccountService>
{
    UnaryResult<LoginResult> LoginAsync(string username, string password);
    UnaryResult<PlayerData> GetPlayerDataAsync(string playerId);
    UnaryResult<bool> UpdatePositionAsync(PositionData position);
}

/// <summary>
/// 실시간 양방향 스트리밍 Hub 인터페이스
/// IStreamingHub<THub, TReceiver>로 양방향 통신 정의
/// </summary>
public interface IGameHub : IStreamingHub<IGameHub, IGameHubReceiver>
{
    // 클라이언트 → 서버 메서드
    Task JoinRoomAsync(string roomName, string playerName);
    Task LeaveRoomAsync();
    Task SendMoveAsync(PositionData position);
    Task SendChatAsync(string message);
    Task SendAttackAsync(string targetId, int damage);
}

/// <summary>
/// 서버 → 클라이언트 콜백 인터페이스
/// </summary>
public interface IGameHubReceiver
{
    void OnPlayerJoined(string playerName);
    void OnPlayerLeft(string playerName);
    void OnPlayerMoved(string playerId, PositionData position);
    void OnChatReceived(string playerName, string message);
    void OnPlayerAttacked(string attackerId, string targetId, int damage);
    void OnRoomInfo(RoomInfo info);
}

// =============================================
// 공유 데이터 클래스 (MessagePack 직렬화)
// =============================================

[MessagePackObject]
public class LoginResult
{
    [Key(0)] public bool Success { get; set; }
    [Key(1)] public string Token { get; set; }
    [Key(2)] public string ErrorMessage { get; set; }
    [Key(3)] public PlayerData Player { get; set; }
}

[MessagePackObject]
public class PlayerData
{
    [Key(0)] public string PlayerId { get; set; }
    [Key(1)] public string DisplayName { get; set; }
    [Key(2)] public int Level { get; set; }
    [Key(3)] public float Health { get; set; }
    [Key(4)] public PositionData Position { get; set; }
}

[MessagePackObject]
public class PositionData
{
    [Key(0)] public float X { get; set; }
    [Key(1)] public float Y { get; set; }
    [Key(2)] public float Z { get; set; }
}

[MessagePackObject]
public class RoomInfo
{
    [Key(0)] public string RoomName { get; set; }
    [Key(1)] public int PlayerCount { get; set; }
    [Key(2)] public string[] PlayerNames { get; set; }
}
```

### Unity 클라이언트 구현

```csharp
using UnityEngine;
using MagicOnion.Client;
using Grpc.Net.Client;
using System;
using System.Threading;
using Cysharp.Threading.Tasks;

/// <summary>
/// MagicOnion을 사용한 Unity gRPC 클라이언트
/// </summary>
public class MagicOnionClientExample : MonoBehaviour, IGameHubReceiver
{
    [SerializeField] private string serverAddress = "http://localhost:5000";
    [SerializeField] private string playerName = "Player1";

    private GrpcChannel channel;
    private IGameHub gameHub;
    private IAccountService accountService;

    // =============================================
    // 초기화 및 연결
    // =============================================

    private async void Start()
    {
        var token = this.GetCancellationTokenOnDestroy();

        try
        {
            await ConnectAsync(token);
        }
        catch (Exception ex)
        {
            Debug.LogError($"연결 실패: {ex.Message}");
        }
    }

    private async UniTask ConnectAsync(CancellationToken token)
    {
        // gRPC 채널 생성
        channel = GrpcChannel.ForAddress(serverAddress);

        // ✅ Unary 서비스 클라이언트 생성
        accountService = MagicOnionClient.Create<IAccountService>(channel);

        // 로그인
        var loginResult = await accountService.LoginAsync(playerName, "password");
        if (!loginResult.Success)
        {
            Debug.LogError($"로그인 실패: {loginResult.ErrorMessage}");
            return;
        }
        Debug.Log($"로그인 성공: {loginResult.Player.DisplayName}");

        // ✅ StreamingHub 연결 (양방향 스트리밍)
        // this를 IGameHubReceiver로 전달하여 서버 콜백 수신
        gameHub = await StreamingHubClient.ConnectAsync<IGameHub, IGameHubReceiver>(
            channel, this, cancellationToken: token);

        // 방 참가
        await gameHub.JoinRoomAsync("room_01", playerName);
        Debug.Log("방 참가 완료");
    }

    // =============================================
    // 클라이언트 → 서버 메서드 호출
    // =============================================

    private async void Update()
    {
        if (gameHub == null) return;

        // 이동 데이터 전송
        if (Input.GetAxis("Horizontal") != 0 || Input.GetAxis("Vertical") != 0)
        {
            var pos = transform.position;
            await gameHub.SendMoveAsync(new PositionData
            {
                X = pos.x, Y = pos.y, Z = pos.z
            });
        }

        // 채팅 전송
        if (Input.GetKeyDown(KeyCode.Return))
        {
            await gameHub.SendChatAsync("안녕하세요!");
        }

        // 공격
        if (Input.GetMouseButtonDown(0))
        {
            await gameHub.SendAttackAsync("target_001", 25);
        }
    }

    // =============================================
    // IGameHubReceiver 구현 (서버 → 클라이언트 콜백)
    // =============================================

    public void OnPlayerJoined(string name)
    {
        Debug.Log($"{name}님이 참가했습니다.");
    }

    public void OnPlayerLeft(string name)
    {
        Debug.Log($"{name}님이 퇴장했습니다.");
    }

    public void OnPlayerMoved(string playerId, PositionData position)
    {
        // 다른 플레이어의 위치 업데이트
        Vector3 pos = new Vector3(position.X, position.Y, position.Z);
        // UpdateRemotePlayerPosition(playerId, pos);
    }

    public void OnChatReceived(string name, string message)
    {
        Debug.Log($"[채팅] {name}: {message}");
    }

    public void OnPlayerAttacked(string attackerId, string targetId, int damage)
    {
        Debug.Log($"{attackerId} → {targetId}: {damage} 데미지");
    }

    public void OnRoomInfo(RoomInfo info)
    {
        Debug.Log($"방: {info.RoomName}, 인원: {info.PlayerCount}명");
    }

    // =============================================
    // 정리
    // =============================================

    private async void OnDestroy()
    {
        if (gameHub != null)
        {
            await gameHub.LeaveRoomAsync();
            await gameHub.DisposeAsync();
        }

        if (channel != null)
        {
            await channel.ShutdownAsync();
            channel.Dispose();
        }
    }
}
```

---

## 9. 인증, 메타데이터, 인터셉터

### 인증 및 메타데이터

```csharp
using UnityEngine;
using Grpc.Core;
using Grpc.Core.Interceptors;
using Grpc.Net.Client;
using System;
using System.Threading;
using System.Threading.Tasks;
using Game.Grpc;

/// <summary>
/// gRPC 인증 및 메타데이터 처리
/// </summary>
public class GrpcAuthExample : MonoBehaviour
{
    private string authToken;

    // =============================================
    // 메타데이터를 통한 인증 토큰 전달
    // =============================================

    public async Task<PlayerInfo> GetPlayerWithAuthAsync(
        GameService.GameServiceClient client,
        string playerId,
        CancellationToken token)
    {
        // ✅ 메타데이터에 인증 정보 포함
        var headers = new Metadata
        {
            { "authorization", $"Bearer {authToken}" },
            { "x-request-id", Guid.NewGuid().ToString() },
            { "x-client-platform", Application.platform.ToString() },
            { "x-client-version", Application.version }
        };

        var request = new GetPlayerRequest { PlayerId = playerId };

        var callOptions = new CallOptions(
            headers: headers,
            deadline: DateTime.UtcNow.AddSeconds(10),
            cancellationToken: token
        );

        // 응답 헤더와 트레일러 접근
        var call = client.GetPlayerInfoAsync(request, callOptions);
        var response = await call.ResponseAsync;

        // 응답 헤더 읽기
        Metadata responseHeaders = await call.ResponseHeadersAsync;
        foreach (var entry in responseHeaders)
        {
            Debug.Log($"응답 헤더: {entry.Key} = {entry.Value}");
        }

        // 트레일러 (응답 완료 후)
        Metadata trailers = call.GetTrailers();
        foreach (var entry in trailers)
        {
            Debug.Log($"트레일러: {entry.Key} = {entry.Value}");
        }

        return response;
    }

    // =============================================
    // CallCredentials를 통한 자동 토큰 주입
    // =============================================

    public GrpcChannel CreateAuthenticatedChannel(string address)
    {
        // 매 요청마다 자동으로 토큰 추가
        var credentials = CallCredentials.FromInterceptor((context, metadata) =>
        {
            if (!string.IsNullOrEmpty(authToken))
            {
                metadata.Add("authorization", $"Bearer {authToken}");
            }
            return Task.CompletedTask;
        });

        var channel = GrpcChannel.ForAddress(address, new GrpcChannelOptions
        {
            Credentials = ChannelCredentials.Create(
                new SslCredentials(), credentials)
        });

        return channel;
    }
}
```

### 클라이언트 인터셉터

```csharp
using Grpc.Core;
using Grpc.Core.Interceptors;
using System;
using System.Diagnostics;
using System.Threading.Tasks;
using Debug = UnityEngine.Debug;

/// <summary>
/// 로깅 인터셉터 - 모든 gRPC 호출을 로깅
/// </summary>
public class LoggingInterceptor : Interceptor
{
    // =============================================
    // Unary 호출 인터셉트
    // =============================================

    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var sw = Stopwatch.StartNew();
        string method = context.Method.FullName;

        Debug.Log($"[gRPC] 호출 시작: {method}");

        var call = continuation(request, context);

        // 응답 래핑하여 로깅 추가
        var responseAsync = HandleResponse(call.ResponseAsync, method, sw);

        return new AsyncUnaryCall<TResponse>(
            responseAsync,
            call.ResponseHeadersAsync,
            call.GetStatus,
            call.GetTrailers,
            call.Dispose);
    }

    private async Task<TResponse> HandleResponse<TResponse>(
        Task<TResponse> responseTask, string method, Stopwatch sw)
    {
        try
        {
            var response = await responseTask;
            sw.Stop();
            Debug.Log($"[gRPC] 호출 완료: {method} ({sw.ElapsedMilliseconds}ms)");
            return response;
        }
        catch (RpcException ex)
        {
            sw.Stop();
            Debug.LogError($"[gRPC] 호출 실패: {method} " +
                           $"({sw.ElapsedMilliseconds}ms) - {ex.StatusCode}");
            throw;
        }
    }

    // =============================================
    // Server Streaming 인터셉트
    // =============================================

    public override AsyncServerStreamingCall<TResponse> AsyncServerStreamingCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncServerStreamingCallContinuation<TRequest, TResponse> continuation)
    {
        Debug.Log($"[gRPC] 서버 스트림 시작: {context.Method.FullName}");
        return continuation(request, context);
    }
}

/// <summary>
/// 인증 토큰 자동 주입 인터셉터
/// </summary>
public class AuthInterceptor : Interceptor
{
    private readonly Func<string> tokenProvider;

    public AuthInterceptor(Func<string> tokenProvider)
    {
        this.tokenProvider = tokenProvider;
    }

    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        // ✅ 메타데이터에 토큰 자동 추가
        var token = tokenProvider();
        if (!string.IsNullOrEmpty(token))
        {
            var headers = context.Options.Headers ?? new Metadata();
            headers.Add("authorization", $"Bearer {token}");

            var newOptions = context.Options.WithHeaders(headers);
            context = new ClientInterceptorContext<TRequest, TResponse>(
                context.Method, context.Host, newOptions);
        }

        return continuation(request, context);
    }
}

/// <summary>
/// 재시도 인터셉터
/// </summary>
public class RetryInterceptor : Interceptor
{
    private readonly int maxRetries;
    private readonly TimeSpan baseDelay;

    public RetryInterceptor(int maxRetries = 3, int baseDelayMs = 500)
    {
        this.maxRetries = maxRetries;
        this.baseDelay = TimeSpan.FromMilliseconds(baseDelayMs);
    }

    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var call = continuation(request, context);

        var retryResponse = RetryAsync(
            call.ResponseAsync, request, context, continuation);

        return new AsyncUnaryCall<TResponse>(
            retryResponse,
            call.ResponseHeadersAsync,
            call.GetStatus,
            call.GetTrailers,
            call.Dispose);
    }

    private async Task<TResponse> RetryAsync<TRequest, TResponse>(
        Task<TResponse> originalResponse,
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
        where TRequest : class
        where TResponse : class
    {
        for (int attempt = 0; attempt <= maxRetries; attempt++)
        {
            try
            {
                if (attempt == 0)
                    return await originalResponse;

                var call = continuation(request, context);
                return await call.ResponseAsync;
            }
            catch (RpcException ex) when (IsRetryable(ex.StatusCode) &&
                                          attempt < maxRetries)
            {
                int delay = (int)(baseDelay.TotalMilliseconds * Math.Pow(2, attempt));
                Debug.LogWarning($"[gRPC] 재시도 {attempt + 1}/{maxRetries}: " +
                                 $"{ex.StatusCode}, {delay}ms 후");
                await Task.Delay(delay);
            }
        }

        throw new RpcException(new Status(StatusCode.Internal, "최대 재시도 횟수 초과"));
    }

    private bool IsRetryable(StatusCode code)
    {
        return code == StatusCode.Unavailable ||
               code == StatusCode.DeadlineExceeded ||
               code == StatusCode.Aborted;
    }
}
```

### 인터셉터 등록

```csharp
using UnityEngine;
using Grpc.Net.Client;
using Grpc.Core.Interceptors;
using Game.Grpc;

/// <summary>
/// 인터셉터 체인 등록 예시
/// </summary>
public class InterceptorSetupExample : MonoBehaviour
{
    private string currentToken = "";

    private GameService.GameServiceClient CreateClientWithInterceptors()
    {
        var channel = GrpcChannel.ForAddress("https://localhost:5001");

        // ✅ 인터셉터를 체인으로 등록 (실행 순서: 등록 역순)
        var invoker = channel
            .Intercept(new RetryInterceptor(maxRetries: 3))
            .Intercept(new LoggingInterceptor())
            .Intercept(new AuthInterceptor(() => currentToken));

        // 인터셉터가 적용된 클라이언트 생성
        return new GameService.GameServiceClient(invoker);
    }
}
```

---

## 10. 에러 핸들링 (gRPC Status Codes)

### gRPC 상태 코드 매핑

```csharp
using UnityEngine;
using Grpc.Core;
using System;
using System.Threading;
using Cysharp.Threading.Tasks;

/// <summary>
/// gRPC 에러 처리 - 상태 코드별 분류 및 대응
/// </summary>
public class GrpcErrorHandlingExample : MonoBehaviour
{
    /*
    ┌───────────────────────────────────────────────────────────────────┐
    │                     gRPC 상태 코드 참조표                         │
    ├──────────────────┬────┬──────────────────────────────────────────┤
    │ 코드             │ 값 │ 설명                                     │
    ├──────────────────┼────┼──────────────────────────────────────────┤
    │ OK               │  0 │ 성공                                     │
    │ Cancelled        │  1 │ 클라이언트가 요청 취소                   │
    │ Unknown          │  2 │ 알 수 없는 에러                          │
    │ InvalidArgument  │  3 │ 잘못된 인자                              │
    │ DeadlineExceeded │  4 │ 데드라인 초과 (타임아웃)                 │
    │ NotFound         │  5 │ 리소스 없음                              │
    │ AlreadyExists    │  6 │ 이미 존재                                │
    │ PermissionDenied │  7 │ 권한 거부                                │
    │ ResourceExhausted│  8 │ 리소스 고갈 (rate limit 등)              │
    │ FailedPrecondition│ 9 │ 전제 조건 실패                           │
    │ Aborted          │ 10 │ 트랜잭션 충돌 등으로 중단                │
    │ OutOfRange       │ 11 │ 범위 초과                                │
    │ Unimplemented    │ 12 │ 구현되지 않은 메서드                     │
    │ Internal         │ 13 │ 내부 서버 에러                           │
    │ Unavailable      │ 14 │ 서비스 불가 (일시적)                     │
    │ DataLoss         │ 15 │ 데이터 손실                              │
    │ Unauthenticated  │ 16 │ 인증 필요                                │
    └──────────────────┴────┴──────────────────────────────────────────┘
    */

    // =============================================
    // 종합 에러 처리 래퍼
    // =============================================

    /// <summary>
    /// gRPC 호출을 래핑하여 에러를 체계적으로 처리
    /// </summary>
    public async UniTask<GrpcResult<T>> SafeCallAsync<T>(
        Func<UniTask<T>> grpcCall,
        CancellationToken token = default)
    {
        try
        {
            T result = await grpcCall();
            return GrpcResult<T>.Success(result);
        }
        catch (RpcException ex)
        {
            return HandleRpcException<T>(ex);
        }
        catch (OperationCanceledException)
        {
            return GrpcResult<T>.Failure(StatusCode.Cancelled, "요청이 취소되었습니다.");
        }
        catch (Exception ex)
        {
            Debug.LogError($"예상치 못한 에러: {ex}");
            return GrpcResult<T>.Failure(StatusCode.Unknown, ex.Message);
        }
    }

    // =============================================
    // 상태 코드별 에러 분류
    // =============================================

    private GrpcResult<T> HandleRpcException<T>(RpcException ex)
    {
        string detail = ex.Status.Detail;

        switch (ex.StatusCode)
        {
            // --- 클라이언트 에러 (재시도 불필요) ---
            case StatusCode.InvalidArgument:
                Debug.LogError($"잘못된 요청 인자: {detail}");
                return GrpcResult<T>.ClientError(ex.StatusCode, detail);

            case StatusCode.NotFound:
                Debug.LogWarning($"리소스를 찾을 수 없음: {detail}");
                return GrpcResult<T>.ClientError(ex.StatusCode, detail);

            case StatusCode.AlreadyExists:
                Debug.LogWarning($"이미 존재: {detail}");
                return GrpcResult<T>.ClientError(ex.StatusCode, detail);

            case StatusCode.PermissionDenied:
                Debug.LogError($"권한 거부: {detail}");
                return GrpcResult<T>.ClientError(ex.StatusCode, detail);

            case StatusCode.OutOfRange:
                Debug.LogError($"범위 초과: {detail}");
                return GrpcResult<T>.ClientError(ex.StatusCode, detail);

            // --- 인증 에러 ---
            case StatusCode.Unauthenticated:
                Debug.LogWarning("인증 만료 또는 유효하지 않은 토큰");
                OnAuthenticationRequired?.Invoke();
                return GrpcResult<T>.AuthError(detail);

            // --- 서버 에러 (재시도 가능) ---
            case StatusCode.Unavailable:
                Debug.LogWarning($"서비스 불가 (일시적): {detail}");
                return GrpcResult<T>.RetryableError(ex.StatusCode, detail);

            case StatusCode.DeadlineExceeded:
                Debug.LogWarning($"요청 시간 초과: {detail}");
                return GrpcResult<T>.RetryableError(ex.StatusCode, detail);

            case StatusCode.ResourceExhausted:
                Debug.LogWarning($"리소스 초과 (rate limit): {detail}");
                return GrpcResult<T>.RetryableError(ex.StatusCode, detail);

            case StatusCode.Aborted:
                Debug.LogWarning($"요청 중단 (충돌): {detail}");
                return GrpcResult<T>.RetryableError(ex.StatusCode, detail);

            case StatusCode.Internal:
                Debug.LogError($"서버 내부 에러: {detail}");
                return GrpcResult<T>.ServerError(ex.StatusCode, detail);

            case StatusCode.Unimplemented:
                Debug.LogError($"미구현 메서드: {detail}");
                return GrpcResult<T>.ServerError(ex.StatusCode, detail);

            case StatusCode.DataLoss:
                Debug.LogError($"데이터 손실: {detail}");
                return GrpcResult<T>.ServerError(ex.StatusCode, detail);

            default:
                Debug.LogError($"알 수 없는 gRPC 에러 [{ex.StatusCode}]: {detail}");
                return GrpcResult<T>.Failure(ex.StatusCode, detail);
        }
    }

    public event Action OnAuthenticationRequired;
}

// =============================================
// gRPC 호출 결과 타입
// =============================================

public class GrpcResult<T>
{
    public bool IsSuccess { get; private set; }
    public T Value { get; private set; }
    public StatusCode ErrorCode { get; private set; }
    public string ErrorMessage { get; private set; }
    public GrpcErrorCategory Category { get; private set; }

    // ✅ 성공 결과
    public static GrpcResult<T> Success(T value) => new GrpcResult<T>
    {
        IsSuccess = true,
        Value = value,
        Category = GrpcErrorCategory.None
    };

    // ❌ 실패 결과
    public static GrpcResult<T> Failure(StatusCode code, string message) =>
        new GrpcResult<T>
        {
            IsSuccess = false,
            ErrorCode = code,
            ErrorMessage = message,
            Category = GrpcErrorCategory.Unknown
        };

    public static GrpcResult<T> ClientError(StatusCode code, string message) =>
        new GrpcResult<T>
        {
            IsSuccess = false,
            ErrorCode = code,
            ErrorMessage = message,
            Category = GrpcErrorCategory.ClientError
        };

    public static GrpcResult<T> ServerError(StatusCode code, string message) =>
        new GrpcResult<T>
        {
            IsSuccess = false,
            ErrorCode = code,
            ErrorMessage = message,
            Category = GrpcErrorCategory.ServerError
        };

    public static GrpcResult<T> RetryableError(StatusCode code, string message) =>
        new GrpcResult<T>
        {
            IsSuccess = false,
            ErrorCode = code,
            ErrorMessage = message,
            Category = GrpcErrorCategory.Retryable
        };

    public static GrpcResult<T> AuthError(string message) =>
        new GrpcResult<T>
        {
            IsSuccess = false,
            ErrorCode = StatusCode.Unauthenticated,
            ErrorMessage = message,
            Category = GrpcErrorCategory.Authentication
        };
}

public enum GrpcErrorCategory
{
    None,
    ClientError,
    ServerError,
    Retryable,
    Authentication,
    Unknown
}
```

### 사용 예시

```csharp
using UnityEngine;
using Grpc.Core;
using System.Threading;
using Cysharp.Threading.Tasks;
using Game.Grpc;

/// <summary>
/// GrpcResult를 활용한 안전한 호출 패턴
/// </summary>
public class SafeGrpcCallExample : MonoBehaviour
{
    [SerializeField] private GrpcConnectionManager connectionManager;
    [SerializeField] private GrpcErrorHandlingExample errorHandler;

    private async void Start()
    {
        var token = this.GetCancellationTokenOnDestroy();

        // ✅ 안전한 gRPC 호출
        var result = await errorHandler.SafeCallAsync(async () =>
        {
            var request = new GetPlayerRequest { PlayerId = "player_001" };
            return await connectionManager.Client.GetPlayerInfoAsync(
                request,
                deadline: System.DateTime.UtcNow.AddSeconds(10),
                cancellationToken: token);
        }, token);

        if (result.IsSuccess)
        {
            Debug.Log($"플레이어: {result.Value.DisplayName}");
        }
        else
        {
            switch (result.Category)
            {
                case GrpcErrorCategory.Retryable:
                    Debug.Log("재시도 가능한 에러 - 잠시 후 다시 시도합니다.");
                    break;
                case GrpcErrorCategory.Authentication:
                    Debug.Log("재로그인이 필요합니다.");
                    break;
                case GrpcErrorCategory.ClientError:
                    Debug.Log($"요청 오류: {result.ErrorMessage}");
                    break;
                default:
                    Debug.Log($"에러: {result.ErrorMessage}");
                    break;
            }
        }
    }
}
```

---

## 11. IL2CPP 호환성 주의사항

Unity의 IL2CPP 스크립팅 백엔드에서 gRPC를 사용할 때 여러 제약 사항이 있습니다.

```csharp
using UnityEngine;
using UnityEngine.Scripting;
using System;
using System.Runtime.CompilerServices;

/// <summary>
/// IL2CPP 환경에서 gRPC/Protobuf 호환성 확보
/// </summary>
public class IL2CPPCompatibilityExample : MonoBehaviour
{
    /*
    ┌───────────────────────────────────────────────────────────────────┐
    │                  IL2CPP gRPC 호환성 체크리스트                     │
    ├───────────────────────────────────────────────────────────────────┤
    │                                                                   │
    │  1. 코드 스트리핑 방지                                            │
    │     - link.xml에 gRPC/protobuf 어셈블리 등록                     │
    │     - [Preserve] 어트리뷰트 사용                                 │
    │                                                                   │
    │  2. 리플렉션 제한                                                 │
    │     - protobuf의 리플렉션 기반 기능 주의                         │
    │     - MessagePack의 AOT 코드 생성 사용                           │
    │                                                                   │
    │  3. 제네릭 제한                                                   │
    │     - AOT에서 제네릭 메서드 인스턴스화 필요                      │
    │     - AotHelper로 사전 등록                                      │
    │                                                                   │
    │  4. gRPC 네이티브 라이브러리                                      │
    │     - 플랫폼별 네이티브 바이너리 포함 확인                       │
    │     - grpc-dotnet(관리 코드) 사용 권장                           │
    │                                                                   │
    └───────────────────────────────────────────────────────────────────┘
    */

    // =============================================
    // 1. link.xml - 코드 스트리핑 방지
    // =============================================

    /*
    <!-- Assets/link.xml -->
    <linker>
        <!-- gRPC 관련 -->
        <assembly fullname="Grpc.Core.Api" preserve="all"/>
        <assembly fullname="Grpc.Net.Client" preserve="all"/>
        <assembly fullname="Grpc.Net.Common" preserve="all"/>

        <!-- Protocol Buffers -->
        <assembly fullname="Google.Protobuf" preserve="all"/>

        <!-- MagicOnion 사용 시 -->
        <assembly fullname="MagicOnion.Client" preserve="all"/>
        <assembly fullname="MagicOnion.Shared" preserve="all"/>
        <assembly fullname="MessagePack" preserve="all"/>

        <!-- 생성된 코드 -->
        <assembly fullname="Game.Grpc" preserve="all"/>
    </linker>
    */

    // =============================================
    // 2. [Preserve] 어트리뷰트로 스트리핑 방지
    // =============================================

    [Preserve]
    private static void PreserveGrpcTypes()
    {
        // IL2CPP에서 리플렉션으로 사용되는 타입 보존
        // 실제 호출되지 않아도 컴파일러가 제거하지 않음

        // protobuf 메시지 타입 보존
        _ = typeof(Game.Grpc.PlayerInfo);
        _ = typeof(Game.Grpc.LoginRequest);
        _ = typeof(Game.Grpc.LoginResponse);
        _ = typeof(Game.Grpc.ChatMessage);
        _ = typeof(Game.Grpc.WorldEvent);
    }

    // =============================================
    // 3. AOT 제네릭 사전 등록 (MagicOnion/MessagePack)
    // =============================================

    [Preserve]
    private static void RegisterAotGenerics()
    {
        // MessagePack AOT 코드 생성이 필요한 타입 등록
        // mpc(MessagePack Compiler)로 AOT 코드 자동 생성 가능

        /*
        // MessagePack AOT 코드 생성 명령어:
        // dotnet tool install -g MessagePack.Generator
        // mpc -i ./Assembly-CSharp.dll -o ./Generated/MessagePackGenerated.cs

        // 초기화 시 AOT Resolver 등록
        MessagePack.Resolvers.StaticCompositeResolver.Instance.Register(
            MessagePack.Resolvers.GeneratedResolver.Instance,
            MessagePack.Unity.UnityResolver.Instance,
            MessagePack.Resolvers.StandardResolver.Instance
        );

        var options = MessagePack.MessagePackSerializerOptions.Standard
            .WithResolver(
                MessagePack.Resolvers.StaticCompositeResolver.Instance);
        MessagePack.MessagePackSerializer.DefaultOptions = options;
        */
    }

    // =============================================
    // 4. 플랫폼별 gRPC 설정
    // =============================================

    public static void InitializeGrpcForPlatform()
    {
#if UNITY_IOS
        // iOS: grpc-dotnet 사용 (관리 코드만)
        // Grpc.Core (네이티브)는 iOS에서 문제가 발생할 수 있음
        Debug.Log("iOS: Grpc.Net.Client 사용");

#elif UNITY_ANDROID
        // Android: 네이티브 라이브러리 경로 확인 필요
        // libgrpc_csharp_ext.so가 Plugins/Android/에 있는지 확인
        Debug.Log("Android: 네이티브 라이브러리 확인");

#elif UNITY_WEBGL
        // WebGL: gRPC-Web만 지원
        // HTTP/2 직접 사용 불가
        Debug.Log("WebGL: gRPC-Web 모드");

#elif UNITY_STANDALONE_WIN || UNITY_STANDALONE_OSX || UNITY_STANDALONE_LINUX
        // 데스크톱: 네이티브 gRPC 또는 grpc-dotnet 모두 사용 가능
        Debug.Log("데스크톱: 전체 gRPC 지원");

#endif
    }

    // =============================================
    // 5. IL2CPP 환경 검증 유틸리티
    // =============================================

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void ValidateGrpcSetup()
    {
#if ENABLE_IL2CPP
        Debug.Log("IL2CPP 빌드 - gRPC 호환성 검증 중...");

        try
        {
            // protobuf 직렬화 테스트
            var testMsg = new Game.Grpc.Vector3Proto { X = 1, Y = 2, Z = 3 };
            byte[] bytes = testMsg.ToByteArray();
            var parsed = Game.Grpc.Vector3Proto.Parser.ParseFrom(bytes);

            if (parsed.X == 1 && parsed.Y == 2 && parsed.Z == 3)
            {
                Debug.Log("Protobuf 직렬화 정상 동작");
            }
        }
        catch (Exception ex)
        {
            Debug.LogError($"IL2CPP protobuf 호환성 문제: {ex.Message}");
            Debug.LogError("link.xml에 Google.Protobuf 어셈블리가 등록되어 있는지 확인하세요.");
        }
#endif
    }
}
```

---

## 12. 종합 gRPC 게임 클라이언트

```csharp
using UnityEngine;
using Grpc.Net.Client;
using Grpc.Core;
using Grpc.Core.Interceptors;
using System;
using System.Threading;
using Cysharp.Threading.Tasks;
using Game.Grpc;

/// <summary>
/// 실전 게임에서 사용할 수 있는 종합 gRPC 클라이언트 매니저
/// </summary>
public class GameGrpcManager : MonoBehaviour
{
    [Header("서버 설정")]
    [SerializeField] private string serverAddress = "https://game-server.example.com";
    [SerializeField] private int defaultTimeoutSeconds = 15;
    [SerializeField] private int maxRetries = 3;

    [Header("상태")]
    [SerializeField] private bool isConnected;
    [SerializeField] private string currentPlayerId;

    private GrpcChannel channel;
    private GameService.GameServiceClient client;
    private CancellationTokenSource lifetimeCts;
    private string authToken;

    // 이벤트
    public event Action OnConnected;
    public event Action OnDisconnected;
    public event Action<string> OnError;

    public static GameGrpcManager Instance { get; private set; }

    // =============================================
    // 초기화
    // =============================================

    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
        }
        else
        {
            Destroy(gameObject);
            return;
        }

        lifetimeCts = new CancellationTokenSource();
    }

    public async UniTask InitializeAsync()
    {
        try
        {
            // 채널 생성
            channel = CreateChannel();

            // 인터셉터 체인 구성
            var invoker = channel
                .Intercept(new RetryInterceptor(maxRetries))
                .Intercept(new LoggingInterceptor())
                .Intercept(new AuthInterceptor(() => authToken));

            client = new GameService.GameServiceClient(invoker);

            isConnected = true;
            OnConnected?.Invoke();
            Debug.Log("gRPC 클라이언트 초기화 완료");
        }
        catch (Exception ex)
        {
            Debug.LogError($"gRPC 초기화 실패: {ex.Message}");
            OnError?.Invoke(ex.Message);
        }
    }

    private GrpcChannel CreateChannel()
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        // WebGL: gRPC-Web
        var handler = new Grpc.Net.Client.Web.GrpcWebHandler(
            Grpc.Net.Client.Web.GrpcWebMode.GrpcWeb,
            new System.Net.Http.HttpClientHandler());
        return GrpcChannel.ForAddress(serverAddress,
            new GrpcChannelOptions { HttpHandler = handler });
#else
        // 네이티브 플랫폼
        return GrpcChannel.ForAddress(serverAddress, new GrpcChannelOptions
        {
            HttpHandler = new System.Net.Http.SocketsHttpHandler
            {
                EnableMultipleHttp2Connections = true,
                KeepAlivePingDelay = TimeSpan.FromSeconds(60),
                KeepAlivePingTimeout = TimeSpan.FromSeconds(30)
            },
            MaxReceiveMessageSize = 16 * 1024 * 1024
        });
#endif
    }

    // =============================================
    // API 메서드
    // =============================================

    public async UniTask<LoginResponse> LoginAsync(string username, string password)
    {
        var request = new LoginRequest
        {
            Username = username,
            Password = password,
            ClientVersion = Application.version
        };

        var response = await CallAsync(
            () => client.LoginAsync(request, CreateOptions()));

        if (response.Success)
        {
            authToken = response.AuthToken;
            currentPlayerId = response.Player.PlayerId;
        }

        return response;
    }

    public async UniTask<PlayerInfo> GetPlayerInfoAsync(string playerId)
    {
        var request = new GetPlayerRequest { PlayerId = playerId };
        return await CallAsync(
            () => client.GetPlayerInfoAsync(request, CreateOptions()));
    }

    // =============================================
    // 공통 호출 래퍼
    // =============================================

    private async UniTask<T> CallAsync<T>(Func<AsyncUnaryCall<T>> callFactory)
    {
        try
        {
            return await callFactory();
        }
        catch (RpcException ex)
        {
            HandleError(ex);
            throw;
        }
    }

    private CallOptions CreateOptions(int? timeoutOverride = null)
    {
        int seconds = timeoutOverride ?? defaultTimeoutSeconds;
        return new CallOptions(
            deadline: DateTime.UtcNow.AddSeconds(seconds),
            cancellationToken: lifetimeCts.Token);
    }

    private void HandleError(RpcException ex)
    {
        string message = $"[{ex.StatusCode}] {ex.Status.Detail}";
        Debug.LogError($"gRPC 에러: {message}");
        OnError?.Invoke(message);

        if (ex.StatusCode == StatusCode.Unauthenticated)
        {
            authToken = null;
            // 재로그인 플로우 트리거
        }
    }

    // =============================================
    // 정리
    // =============================================

    private async void OnDestroy()
    {
        lifetimeCts?.Cancel();
        lifetimeCts?.Dispose();

        if (channel != null)
        {
            isConnected = false;
            OnDisconnected?.Invoke();
            await channel.ShutdownAsync();
            channel.Dispose();
        }
    }
}
```

---

## 주의사항

1. **HTTP/2 지원**: gRPC는 HTTP/2를 필요로 하며, WebGL 등 일부 환경에서는 gRPC-Web으로 대체해야 합니다
2. **메인 스레드 접근**: gRPC 콜백은 백그라운드 스레드에서 실행될 수 있으므로, Unity API 접근 시 반드시 메인 스레드로 전환해야 합니다
3. **채널 재사용**: GrpcChannel은 스레드 안전하므로 하나의 채널을 공유하여 사용합니다. 호출마다 채널을 생성하지 마세요
4. **데드라인 설정**: 모든 RPC 호출에 데드라인을 설정하여 무한 대기를 방지합니다
5. **스트림 완료**: Client/Bidirectional Streaming에서 반드시 `CompleteAsync()`로 스트림 종료를 알려야 합니다
6. **IL2CPP 빌드**: link.xml 설정, AOT 코드 생성, [Preserve] 어트리뷰트를 반드시 확인합니다
7. **protobuf 호환성**: 필드 번호를 변경하거나 제거하면 하위 호환성이 깨집니다. 새로운 필드를 추가하는 방식으로 확장하세요

---

## 베스트 프랙티스

### 채널 및 연결 관리

```csharp
// ✅ 올바른 패턴: 채널을 싱글톤으로 재사용
public class GrpcChannelProvider : MonoBehaviour
{
    private static GrpcChannel sharedChannel;

    public static GrpcChannel GetChannel(string address)
    {
        if (sharedChannel == null)
        {
            sharedChannel = GrpcChannel.ForAddress(address);
        }
        return sharedChannel;
    }
}

// ❌ 잘못된 패턴: 호출마다 새 채널 생성
public class BadChannelUsage : MonoBehaviour
{
    public async void MakeCall()
    {
        // 매번 새 채널 생성 → 리소스 낭비, 연결 폭증
        var channel = GrpcChannel.ForAddress("https://server.com");
        var client = new GameService.GameServiceClient(channel);
        // ...
    }
}
```

### 데드라인 및 취소

```csharp
// ✅ 올바른 패턴: 항상 데드라인과 취소 토큰 설정
public async UniTask<PlayerInfo> GetPlayerSafe(
    GameService.GameServiceClient client,
    string playerId,
    CancellationToken token)
{
    var request = new GetPlayerRequest { PlayerId = playerId };
    return await client.GetPlayerInfoAsync(request, new CallOptions(
        deadline: DateTime.UtcNow.AddSeconds(10),
        cancellationToken: token));
}

// ❌ 잘못된 패턴: 데드라인 없이 호출
public async UniTask<PlayerInfo> GetPlayerUnsafe(
    GameService.GameServiceClient client,
    string playerId)
{
    var request = new GetPlayerRequest { PlayerId = playerId };
    // 서버 응답 없으면 영원히 대기할 수 있음
    return await client.GetPlayerInfoAsync(request);
}
```

### 스트리밍 정리

```csharp
// ✅ 올바른 패턴: 스트림을 올바르게 종료하고 정리
public async UniTask StreamWithCleanup(CancellationToken token)
{
    AsyncDuplexStreamingCall<ChatMessage, ChatMessage> call = null;
    try
    {
        call = client.GameChat(new CallOptions(cancellationToken: token));
        // ... 스트리밍 로직 ...

        // 전송 완료 알림
        await call.RequestStream.CompleteAsync();
    }
    finally
    {
        call?.Dispose();
    }
}

// ❌ 잘못된 패턴: 스트림 정리 누락
public async UniTask StreamWithoutCleanup()
{
    var call = client.GameChat();
    // CompleteAsync 호출 없이 종료
    // Dispose 호출 없이 종료
    // → 리소스 누수 발생
}
```

### protobuf 메시지 설계

```csharp
// ✅ 올바른 패턴: 필드 번호 유지, 새 필드 추가로 확장
/*
message PlayerInfo {
    string player_id = 1;     // 기존 필드 유지
    string display_name = 2;  // 기존 필드 유지
    int32 level = 3;          // 기존 필드 유지
    // 필드 4는 삭제됨 → reserved로 표시
    reserved 4;
    string guild_name = 5;    // 새 필드 추가 (OK)
}
*/

// ❌ 잘못된 패턴: 기존 필드 번호 변경 또는 타입 변경
/*
message PlayerInfo {
    string player_id = 1;
    int32 display_name = 2;   // string → int32 타입 변경 → 호환성 깨짐
    int32 level = 4;          // 번호 3 → 4 변경 → 호환성 깨짐
}
*/
```

---

## 참고 자료

- [gRPC 공식 문서](https://grpc.io/docs/)
- [gRPC for .NET (grpc-dotnet)](https://github.com/grpc/grpc-dotnet)
- [Protocol Buffers 언어 가이드](https://protobuf.dev/programming-guides/proto3/)
- [gRPC-Web](https://github.com/grpc/grpc-web)
- [MagicOnion GitHub](https://github.com/Cysharp/MagicOnion)
- [MessagePack for C#](https://github.com/MessagePack-CSharp/MessagePack-CSharp)
- [gRPC Status Codes](https://grpc.github.io/grpc/core/md_doc_statuscodes.html)
- [Unity IL2CPP 코드 스트리핑](https://docs.unity3d.com/Manual/ManagedCodeStripping.html)

---

## 다음 섹션

[30. WebSocket](./30-websocket.md)
