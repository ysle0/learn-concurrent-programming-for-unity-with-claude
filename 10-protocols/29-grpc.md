# Section 29: gRPC

## 개요

gRPC는 Google이 개발한 고성능 RPC(Remote Procedure Call) 프레임워크입니다. HTTP/2 기반으로 양방향 스트리밍, 멀티플렉싱을 지원하며, Protocol Buffers를 사용해 효율적인 직렬화를 제공합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                      gRPC Architecture                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Unity Client                         Game Server               │
│   ┌──────────────┐                    ┌──────────────┐          │
│   │   Generated  │     HTTP/2         │   Generated  │          │
│   │   gRPC Stub  │◀══════════════════▶│   gRPC Stub  │          │
│   └──────────────┘     Protobuf       └──────────────┘          │
│          │                                    │                  │
│          ▼                                    ▼                  │
│   ┌──────────────┐                    ┌──────────────┐          │
│   │    Your      │                    │    Your      │          │
│   │  Game Code   │                    │Server Logic  │          │
│   └──────────────┘                    └──────────────┘          │
│                                                                  │
│   gRPC 통신 유형:                                                 │
│   ┌────────────────────────────────────────────────────┐        │
│   │ 1. Unary          : 단일 요청 → 단일 응답           │        │
│   │ 2. Server Stream  : 단일 요청 → 스트림 응답         │        │
│   │ 3. Client Stream  : 스트림 요청 → 단일 응답         │        │
│   │ 4. Bidirectional  : 스트림 요청 ↔ 스트림 응답       │        │
│   └────────────────────────────────────────────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## gRPC vs REST 비교

| 특성 | gRPC | REST |
|-----|------|------|
| **프로토콜** | HTTP/2 | HTTP/1.1 또는 HTTP/2 |
| **직렬화** | Protocol Buffers (바이너리) | JSON (텍스트) |
| **스트리밍** | 양방향 스트리밍 지원 | 제한적 |
| **코드 생성** | .proto에서 자동 생성 | 수동 또는 OpenAPI |
| **브라우저 지원** | grpc-web 필요 | 네이티브 |
| **성능** | 높음 | 중간 |
| **가독성** | 바이너리 (디버깅 어려움) | 텍스트 (디버깅 쉬움) |
| **Unity WebGL** | 미지원 | 지원 |

---

## Unity에서 gRPC 설정

### 패키지 설치

```
필요한 패키지:
1. Grpc.Net.Client (NuGet)
2. Google.Protobuf (NuGet)
3. Grpc.Tools (코드 생성용)

Unity Package Manager를 통해 설치하거나,
Plugins 폴더에 DLL 직접 추가
```

### Protocol Buffers 정의 (.proto)

```protobuf
// game_service.proto

syntax = "proto3";

package game;

option csharp_namespace = "Game.Proto";

// 게임 서비스 정의
service GameService {
    // 단일 요청-응답 (Unary)
    rpc Login(LoginRequest) returns (LoginResponse);
    rpc GetPlayerInfo(PlayerInfoRequest) returns (PlayerInfo);

    // 서버 스트리밍
    rpc WatchLeaderboard(LeaderboardRequest) returns (stream LeaderboardUpdate);

    // 클라이언트 스트리밍
    rpc UploadReplay(stream ReplayChunk) returns (UploadResult);

    // 양방향 스트리밍
    rpc GameSession(stream GameInput) returns (stream GameState);
}

// 메시지 정의
message LoginRequest {
    string username = 1;
    string password = 2;
    string device_id = 3;
}

message LoginResponse {
    bool success = 1;
    string access_token = 2;
    string player_id = 3;
    string error_message = 4;
}

message PlayerInfoRequest {
    string player_id = 1;
}

message PlayerInfo {
    string player_id = 1;
    string nickname = 2;
    int32 level = 3;
    int64 experience = 4;
    int64 gold = 5;
    repeated Item inventory = 6;
}

message Item {
    string item_id = 1;
    string name = 2;
    int32 quantity = 3;
    ItemRarity rarity = 4;
}

enum ItemRarity {
    COMMON = 0;
    UNCOMMON = 1;
    RARE = 2;
    EPIC = 3;
    LEGENDARY = 4;
}

message LeaderboardRequest {
    string leaderboard_id = 1;
    int32 top_count = 2;
}

message LeaderboardUpdate {
    int64 timestamp = 1;
    repeated LeaderboardEntry entries = 2;
}

message LeaderboardEntry {
    int32 rank = 1;
    string player_id = 2;
    string nickname = 3;
    int64 score = 4;
}

message ReplayChunk {
    int32 chunk_index = 1;
    bytes data = 2;
    bool is_last = 3;
}

message UploadResult {
    bool success = 1;
    string replay_id = 2;
    int64 total_bytes = 3;
}

message GameInput {
    int32 sequence = 1;
    int64 timestamp = 2;
    float move_x = 3;
    float move_y = 4;
    bool jump = 5;
    bool fire = 6;
}

message GameState {
    int64 server_time = 1;
    int32 last_processed_input = 2;
    repeated PlayerState players = 3;
}

message PlayerState {
    string player_id = 1;
    float pos_x = 2;
    float pos_y = 3;
    float pos_z = 4;
    float rot_y = 5;
    float health = 6;
    int32 score = 7;
}
```

---

## gRPC 클라이언트 구현

### 기본 gRPC 클라이언트

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using Grpc.Core;
using Grpc.Net.Client;
using Game.Proto;
using UnityEngine;

/// <summary>
/// gRPC 게임 클라이언트
/// </summary>
public class GrpcGameClient : IDisposable
{
    private readonly GrpcChannel _channel;
    private readonly GameService.GameServiceClient _client;
    private string _accessToken;

    public bool IsConnected => _channel.State == ConnectivityState.Ready;

    public GrpcGameClient(string serverAddress)
    {
        // HTTP/2 채널 생성
        _channel = GrpcChannel.ForAddress(serverAddress, new GrpcChannelOptions
        {
            // 최대 메시지 크기
            MaxReceiveMessageSize = 16 * 1024 * 1024, // 16MB
            MaxSendMessageSize = 16 * 1024 * 1024,

            // 연결 유지
            HttpHandler = new System.Net.Http.SocketsHttpHandler
            {
                PooledConnectionIdleTimeout = TimeSpan.FromMinutes(5),
                KeepAlivePingDelay = TimeSpan.FromSeconds(30),
                KeepAlivePingTimeout = TimeSpan.FromSeconds(10),
                EnableMultipleHttp2Connections = true
            }
        });

        _client = new GameService.GameServiceClient(_channel);
    }

    /// <summary>
    /// 인증 헤더 생성
    /// </summary>
    private Metadata CreateAuthHeaders()
    {
        var headers = new Metadata();
        if (!string.IsNullOrEmpty(_accessToken))
        {
            headers.Add("Authorization", $"Bearer {_accessToken}");
        }
        return headers;
    }

    #region Unary RPC

    /// <summary>
    /// 로그인 (Unary RPC)
    /// </summary>
    public async Task<LoginResponse> LoginAsync(
        string username,
        string password,
        string deviceId,
        CancellationToken ct = default)
    {
        var request = new LoginRequest
        {
            Username = username,
            Password = password,
            DeviceId = deviceId
        };

        try
        {
            var response = await _client.LoginAsync(
                request,
                cancellationToken: ct);

            if (response.Success)
            {
                _accessToken = response.AccessToken;
            }

            return response;
        }
        catch (RpcException ex)
        {
            Debug.LogError($"Login failed: {ex.Status.StatusCode} - {ex.Message}");
            throw;
        }
    }

    /// <summary>
    /// 플레이어 정보 조회 (Unary RPC)
    /// </summary>
    public async Task<PlayerInfo> GetPlayerInfoAsync(
        string playerId,
        CancellationToken ct = default)
    {
        var request = new PlayerInfoRequest { PlayerId = playerId };

        var response = await _client.GetPlayerInfoAsync(
            request,
            headers: CreateAuthHeaders(),
            cancellationToken: ct);

        return response;
    }

    #endregion

    #region Server Streaming

    /// <summary>
    /// 리더보드 실시간 구독 (Server Streaming)
    /// </summary>
    public async Task WatchLeaderboardAsync(
        string leaderboardId,
        int topCount,
        Action<LeaderboardUpdate> onUpdate,
        CancellationToken ct = default)
    {
        var request = new LeaderboardRequest
        {
            LeaderboardId = leaderboardId,
            TopCount = topCount
        };

        using var call = _client.WatchLeaderboard(
            request,
            headers: CreateAuthHeaders(),
            cancellationToken: ct);

        try
        {
            await foreach (var update in call.ResponseStream.ReadAllAsync(ct))
            {
                onUpdate?.Invoke(update);
            }
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.Cancelled)
        {
            Debug.Log("Leaderboard subscription cancelled");
        }
    }

    #endregion

    #region Client Streaming

    /// <summary>
    /// 리플레이 업로드 (Client Streaming)
    /// </summary>
    public async Task<UploadResult> UploadReplayAsync(
        byte[] replayData,
        int chunkSize = 64 * 1024,
        IProgress<float> progress = null,
        CancellationToken ct = default)
    {
        using var call = _client.UploadReplay(
            headers: CreateAuthHeaders(),
            cancellationToken: ct);

        int totalChunks = (int)Math.Ceiling((double)replayData.Length / chunkSize);

        for (int i = 0; i < totalChunks; i++)
        {
            int offset = i * chunkSize;
            int size = Math.Min(chunkSize, replayData.Length - offset);

            var chunk = new ReplayChunk
            {
                ChunkIndex = i,
                Data = Google.Protobuf.ByteString.CopyFrom(replayData, offset, size),
                IsLast = (i == totalChunks - 1)
            };

            await call.RequestStream.WriteAsync(chunk);
            progress?.Report((float)(i + 1) / totalChunks);
        }

        await call.RequestStream.CompleteAsync();
        return await call.ResponseAsync;
    }

    #endregion

    #region Bidirectional Streaming

    /// <summary>
    /// 게임 세션 (Bidirectional Streaming)
    /// </summary>
    public async Task GameSessionAsync(
        IAsyncEnumerable<GameInput> inputs,
        Action<GameState> onState,
        CancellationToken ct = default)
    {
        using var call = _client.GameSession(
            headers: CreateAuthHeaders(),
            cancellationToken: ct);

        // 입력 전송 태스크
        var sendTask = Task.Run(async () =>
        {
            await foreach (var input in inputs.WithCancellation(ct))
            {
                await call.RequestStream.WriteAsync(input);
            }
            await call.RequestStream.CompleteAsync();
        }, ct);

        // 상태 수신 태스크
        var receiveTask = Task.Run(async () =>
        {
            await foreach (var state in call.ResponseStream.ReadAllAsync(ct))
            {
                onState?.Invoke(state);
            }
        }, ct);

        await Task.WhenAll(sendTask, receiveTask);
    }

    #endregion

    public void Dispose()
    {
        _channel?.ShutdownAsync().Wait();
        _channel?.Dispose();
    }
}
```

### Unity MonoBehaviour 통합

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Cysharp.Threading.Tasks;
using Game.Proto;
using UnityEngine;

/// <summary>
/// Unity에서 gRPC 사용 예시
/// </summary>
public class GrpcManager : MonoBehaviour
{
    [SerializeField] private string serverAddress = "https://game.example.com:5001";

    private GrpcGameClient _client;
    private CancellationTokenSource _cts;

    // 이벤트
    public event Action<PlayerInfo> OnPlayerInfoReceived;
    public event Action<LeaderboardUpdate> OnLeaderboardUpdated;
    public event Action<GameState> OnGameStateReceived;

    private async void Awake()
    {
        _cts = new CancellationTokenSource();

        try
        {
            _client = new GrpcGameClient(serverAddress);
            Debug.Log("gRPC 클라이언트 초기화 완료");
        }
        catch (Exception ex)
        {
            Debug.LogError($"gRPC 초기화 실패: {ex.Message}");
        }
    }

    /// <summary>
    /// 로그인
    /// </summary>
    public async UniTask<bool> LoginAsync(string username, string password)
    {
        try
        {
            var response = await _client.LoginAsync(
                username,
                password,
                SystemInfo.deviceUniqueIdentifier,
                _cts.Token);

            if (response.Success)
            {
                Debug.Log($"로그인 성공! Player ID: {response.PlayerId}");
                return true;
            }
            else
            {
                Debug.LogWarning($"로그인 실패: {response.ErrorMessage}");
                return false;
            }
        }
        catch (Exception ex)
        {
            Debug.LogError($"로그인 오류: {ex.Message}");
            return false;
        }
    }

    /// <summary>
    /// 플레이어 정보 조회
    /// </summary>
    public async UniTask<PlayerInfo> GetPlayerInfoAsync(string playerId)
    {
        var info = await _client.GetPlayerInfoAsync(playerId, _cts.Token);

        // 메인 스레드에서 이벤트 발생
        await UniTask.SwitchToMainThread();
        OnPlayerInfoReceived?.Invoke(info);

        return info;
    }

    /// <summary>
    /// 리더보드 구독 시작
    /// </summary>
    public async UniTaskVoid StartLeaderboardSubscriptionAsync(string leaderboardId)
    {
        try
        {
            await _client.WatchLeaderboardAsync(
                leaderboardId,
                100,
                async update =>
                {
                    await UniTask.SwitchToMainThread();
                    OnLeaderboardUpdated?.Invoke(update);
                },
                _cts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("리더보드 구독 취소됨");
        }
    }

    /// <summary>
    /// 리플레이 업로드
    /// </summary>
    public async UniTask<string> UploadReplayAsync(byte[] replayData)
    {
        var progress = new Progress<float>(p =>
        {
            Debug.Log($"업로드 진행: {p * 100:F1}%");
        });

        var result = await _client.UploadReplayAsync(
            replayData,
            progress: progress,
            ct: _cts.Token);

        if (result.Success)
        {
            Debug.Log($"리플레이 업로드 완료: {result.ReplayId}");
            return result.ReplayId;
        }

        return null;
    }

    private void OnDestroy()
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _client?.Dispose();
    }
}
```

---

## 양방향 스트리밍 게임 세션

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;
using Cysharp.Threading.Tasks;
using Game.Proto;
using UnityEngine;

/// <summary>
/// 양방향 스트리밍을 사용한 실시간 게임 세션
/// </summary>
public class RealtimeGameSession : MonoBehaviour
{
    [SerializeField] private GrpcManager grpcManager;
    [SerializeField] private float inputSendRate = 20f; // 초당 20회

    private GrpcGameClient _client;
    private CancellationTokenSource _sessionCts;
    private Channel<GameInput> _inputChannel;
    private int _inputSequence;

    public event Action<GameState> OnGameStateReceived;

    private void Awake()
    {
        // 무제한 채널 생성
        _inputChannel = Channel.CreateUnbounded<GameInput>();
    }

    /// <summary>
    /// 게임 세션 시작
    /// </summary>
    public async UniTask StartSessionAsync()
    {
        _sessionCts = new CancellationTokenSource();
        _inputSequence = 0;

        try
        {
            // 입력 수집 루프 시작
            InputCollectionLoopAsync(_sessionCts.Token).Forget();

            // 양방향 스트리밍 시작
            await _client.GameSessionAsync(
                ReadInputsAsync(_sessionCts.Token),
                HandleGameState,
                _sessionCts.Token);
        }
        catch (OperationCanceledException)
        {
            Debug.Log("게임 세션 종료");
        }
        catch (Exception ex)
        {
            Debug.LogError($"세션 오류: {ex.Message}");
        }
    }

    /// <summary>
    /// 게임 세션 종료
    /// </summary>
    public void StopSession()
    {
        _sessionCts?.Cancel();
        _inputChannel.Writer.TryComplete();
    }

    /// <summary>
    /// 입력 수집 루프
    /// </summary>
    private async UniTaskVoid InputCollectionLoopAsync(CancellationToken ct)
    {
        float interval = 1f / inputSendRate;

        while (!ct.IsCancellationRequested)
        {
            // 현재 입력 수집
            var input = new GameInput
            {
                Sequence = ++_inputSequence,
                Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
                MoveX = Input.GetAxis("Horizontal"),
                MoveY = Input.GetAxis("Vertical"),
                Jump = Input.GetButton("Jump"),
                Fire = Input.GetButton("Fire1")
            };

            // 채널에 입력 전송
            await _inputChannel.Writer.WriteAsync(input, ct);

            await UniTask.Delay(TimeSpan.FromSeconds(interval), cancellationToken: ct);
        }
    }

    /// <summary>
    /// 채널에서 입력 읽기
    /// </summary>
    private async IAsyncEnumerable<GameInput> ReadInputsAsync(
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct)
    {
        await foreach (var input in _inputChannel.Reader.ReadAllAsync(ct))
        {
            yield return input;
        }
    }

    /// <summary>
    /// 게임 상태 수신 처리
    /// </summary>
    private void HandleGameState(GameState state)
    {
        // 메인 스레드에서 실행
        UniTask.Post(() =>
        {
            OnGameStateReceived?.Invoke(state);

            // 플레이어 상태 적용
            foreach (var player in state.Players)
            {
                UpdatePlayerState(player);
            }
        });
    }

    private void UpdatePlayerState(PlayerState state)
    {
        // 플레이어 오브젝트 찾아서 상태 적용
        Debug.Log($"Player {state.PlayerId}: Pos({state.PosX}, {state.PosY}, {state.PosZ})");
    }

    private void OnDestroy()
    {
        StopSession();
        _sessionCts?.Dispose();
    }
}
```

---

## gRPC 에러 핸들링

```csharp
using System;
using System.Threading.Tasks;
using Grpc.Core;
using UnityEngine;

/// <summary>
/// gRPC 에러 핸들링 유틸리티
/// </summary>
public static class GrpcErrorHandler
{
    /// <summary>
    /// RpcException을 게임 친화적인 메시지로 변환
    /// </summary>
    public static string GetUserFriendlyMessage(RpcException ex)
    {
        return ex.StatusCode switch
        {
            StatusCode.Unavailable => "서버에 연결할 수 없습니다. 네트워크를 확인해주세요.",
            StatusCode.DeadlineExceeded => "요청 시간이 초과되었습니다. 다시 시도해주세요.",
            StatusCode.Unauthenticated => "로그인이 필요합니다.",
            StatusCode.PermissionDenied => "권한이 없습니다.",
            StatusCode.NotFound => "요청한 리소스를 찾을 수 없습니다.",
            StatusCode.AlreadyExists => "이미 존재합니다.",
            StatusCode.ResourceExhausted => "서버가 너무 바쁩니다. 잠시 후 다시 시도해주세요.",
            StatusCode.Internal => "서버 내부 오류가 발생했습니다.",
            StatusCode.Cancelled => "요청이 취소되었습니다.",
            _ => $"오류가 발생했습니다: {ex.Status.Detail}"
        };
    }

    /// <summary>
    /// 재시도 가능한 에러인지 확인
    /// </summary>
    public static bool IsRetryable(RpcException ex)
    {
        return ex.StatusCode switch
        {
            StatusCode.Unavailable => true,
            StatusCode.DeadlineExceeded => true,
            StatusCode.ResourceExhausted => true,
            StatusCode.Aborted => true,
            _ => false
        };
    }

    /// <summary>
    /// 재시도 지연 시간 계산
    /// </summary>
    public static TimeSpan GetRetryDelay(RpcException ex, int attemptCount)
    {
        // Retry-After 헤더 확인
        var retryAfter = ex.Trailers.Get("retry-after")?.Value;
        if (retryAfter != null && int.TryParse(retryAfter, out int seconds))
        {
            return TimeSpan.FromSeconds(seconds);
        }

        // 지수 백오프
        return TimeSpan.FromSeconds(Math.Min(Math.Pow(2, attemptCount), 30));
    }
}

/// <summary>
/// 재시도 로직이 포함된 gRPC 호출 래퍼
/// </summary>
public class ResilientGrpcCaller
{
    private readonly int _maxRetries;
    private readonly TimeSpan _baseDelay;

    public ResilientGrpcCaller(int maxRetries = 3, TimeSpan? baseDelay = null)
    {
        _maxRetries = maxRetries;
        _baseDelay = baseDelay ?? TimeSpan.FromSeconds(1);
    }

    /// <summary>
    /// 재시도가 적용된 Unary 호출
    /// </summary>
    public async Task<TResponse> CallAsync<TResponse>(
        Func<Task<TResponse>> rpcCall,
        CancellationToken ct = default)
    {
        int attempt = 0;
        Exception lastException = null;

        while (attempt <= _maxRetries)
        {
            try
            {
                return await rpcCall();
            }
            catch (RpcException ex) when (GrpcErrorHandler.IsRetryable(ex) &&
                                          attempt < _maxRetries)
            {
                lastException = ex;
                var delay = GrpcErrorHandler.GetRetryDelay(ex, attempt);

                Debug.LogWarning($"gRPC 호출 실패, 재시도 {attempt + 1}/{_maxRetries}: " +
                               $"{ex.StatusCode} (대기: {delay.TotalSeconds}초)");

                await Task.Delay(delay, ct);
                attempt++;
            }
            catch (RpcException ex)
            {
                Debug.LogError($"gRPC 호출 실패: {GrpcErrorHandler.GetUserFriendlyMessage(ex)}");
                throw;
            }
        }

        throw new Exception($"모든 재시도 실패", lastException);
    }
}
```

---

## gRPC 연결 관리

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using Grpc.Core;
using Grpc.Net.Client;
using UnityEngine;

/// <summary>
/// gRPC 연결 상태 관리자
/// </summary>
public class GrpcConnectionManager : IDisposable
{
    private readonly string _address;
    private GrpcChannel _channel;
    private readonly object _lock = new object();
    private Timer _healthCheckTimer;

    public event Action OnConnected;
    public event Action OnDisconnected;
    public event Action<ConnectivityState> OnStateChanged;

    public ConnectivityState CurrentState => _channel?.State ?? ConnectivityState.Shutdown;
    public bool IsConnected => CurrentState == ConnectivityState.Ready;

    public GrpcConnectionManager(string address)
    {
        _address = address;
    }

    /// <summary>
    /// 연결 시작
    /// </summary>
    public async Task ConnectAsync(CancellationToken ct = default)
    {
        lock (_lock)
        {
            if (_channel != null)
            {
                throw new InvalidOperationException("이미 연결됨");
            }

            _channel = GrpcChannel.ForAddress(_address);
        }

        // 연결 대기
        await _channel.ConnectAsync(ct);

        OnConnected?.Invoke();
        OnStateChanged?.Invoke(ConnectivityState.Ready);

        // 상태 모니터링 시작
        StartHealthCheck();
    }

    /// <summary>
    /// 채널 가져오기
    /// </summary>
    public GrpcChannel GetChannel()
    {
        lock (_lock)
        {
            if (_channel == null)
            {
                throw new InvalidOperationException("연결되지 않음");
            }
            return _channel;
        }
    }

    /// <summary>
    /// 연결 상태 확인 타이머
    /// </summary>
    private void StartHealthCheck()
    {
        _healthCheckTimer = new Timer(async _ =>
        {
            try
            {
                var state = _channel.State;
                OnStateChanged?.Invoke(state);

                if (state == ConnectivityState.TransientFailure)
                {
                    Debug.LogWarning("gRPC 연결 불안정, 재연결 시도...");
                    await _channel.ConnectAsync();
                }
            }
            catch (Exception ex)
            {
                Debug.LogError($"Health check 오류: {ex.Message}");
            }
        }, null, TimeSpan.FromSeconds(5), TimeSpan.FromSeconds(5));
    }

    /// <summary>
    /// 연결 종료
    /// </summary>
    public async Task DisconnectAsync()
    {
        _healthCheckTimer?.Dispose();
        _healthCheckTimer = null;

        lock (_lock)
        {
            if (_channel != null)
            {
                _channel.ShutdownAsync().Wait();
                _channel.Dispose();
                _channel = null;
            }
        }

        OnDisconnected?.Invoke();
    }

    public void Dispose()
    {
        DisconnectAsync().Wait();
    }
}
```

---

## 인터셉터를 통한 로깅 및 메트릭

```csharp
using System;
using System.Diagnostics;
using System.Threading.Tasks;
using Grpc.Core;
using Grpc.Core.Interceptors;
using UnityEngine;
using Debug = UnityEngine.Debug;

/// <summary>
/// gRPC 호출 로깅 인터셉터
/// </summary>
public class LoggingInterceptor : Interceptor
{
    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var stopwatch = Stopwatch.StartNew();
        var methodName = context.Method.FullName;

        Debug.Log($"[gRPC] → {methodName}");

        var call = continuation(request, context);

        return new AsyncUnaryCall<TResponse>(
            HandleResponse(call.ResponseAsync, methodName, stopwatch),
            call.ResponseHeadersAsync,
            call.GetStatus,
            call.GetTrailers,
            call.Dispose);
    }

    private async Task<TResponse> HandleResponse<TResponse>(
        Task<TResponse> responseTask,
        string methodName,
        Stopwatch stopwatch)
    {
        try
        {
            var response = await responseTask;
            stopwatch.Stop();
            Debug.Log($"[gRPC] ← {methodName} ({stopwatch.ElapsedMilliseconds}ms)");
            return response;
        }
        catch (RpcException ex)
        {
            stopwatch.Stop();
            Debug.LogError($"[gRPC] ✕ {methodName} ({stopwatch.ElapsedMilliseconds}ms): " +
                         $"{ex.StatusCode}");
            throw;
        }
    }

    public override AsyncServerStreamingCall<TResponse> AsyncServerStreamingCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncServerStreamingCallContinuation<TRequest, TResponse> continuation)
    {
        Debug.Log($"[gRPC Stream] → {context.Method.FullName}");
        return continuation(request, context);
    }
}

/// <summary>
/// 메트릭 수집 인터셉터
/// </summary>
public class MetricsInterceptor : Interceptor
{
    public static int TotalCalls { get; private set; }
    public static int SuccessfulCalls { get; private set; }
    public static int FailedCalls { get; private set; }
    public static double AverageLatencyMs { get; private set; }

    private static double _totalLatency;
    private static readonly object _lock = new object();

    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var stopwatch = Stopwatch.StartNew();
        var call = continuation(request, context);

        return new AsyncUnaryCall<TResponse>(
            TrackMetrics(call.ResponseAsync, stopwatch),
            call.ResponseHeadersAsync,
            call.GetStatus,
            call.GetTrailers,
            call.Dispose);
    }

    private async Task<TResponse> TrackMetrics<TResponse>(
        Task<TResponse> responseTask,
        Stopwatch stopwatch)
    {
        try
        {
            var response = await responseTask;
            stopwatch.Stop();

            lock (_lock)
            {
                TotalCalls++;
                SuccessfulCalls++;
                _totalLatency += stopwatch.ElapsedMilliseconds;
                AverageLatencyMs = _totalLatency / TotalCalls;
            }

            return response;
        }
        catch
        {
            lock (_lock)
            {
                TotalCalls++;
                FailedCalls++;
            }
            throw;
        }
    }

    public static void Reset()
    {
        lock (_lock)
        {
            TotalCalls = 0;
            SuccessfulCalls = 0;
            FailedCalls = 0;
            _totalLatency = 0;
            AverageLatencyMs = 0;
        }
    }
}

/// <summary>
/// 인터셉터 적용 예시
/// </summary>
public class GrpcClientWithInterceptors
{
    public static GameService.GameServiceClient CreateClient(string address)
    {
        var channel = GrpcChannel.ForAddress(address);

        // 인터셉터 체인 적용
        var invoker = channel
            .Intercept(new LoggingInterceptor())
            .Intercept(new MetricsInterceptor());

        return new GameService.GameServiceClient(invoker);
    }
}
```

---

## IL2CPP 및 AOT 고려사항

```csharp
using System;
using UnityEngine;
using UnityEngine.Scripting;

/// <summary>
/// IL2CPP 빌드를 위한 gRPC 타입 보존
/// link.xml 또는 Preserve 속성 필요
/// </summary>
public static class GrpcAotPreservation
{
    /// <summary>
    /// AOT 컴파일러에게 타입 보존 힌트 제공
    /// </summary>
    [Preserve]
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void PreserveTypes()
    {
        // gRPC 내부 타입 참조
        _ = typeof(Grpc.Core.Channel);
        _ = typeof(Grpc.Core.Metadata);
        _ = typeof(Grpc.Core.Status);

        // Protobuf 타입
        _ = typeof(Google.Protobuf.MessageParser);
    }
}
```

**link.xml 예시**:
```xml
<linker>
    <!-- gRPC -->
    <assembly fullname="Grpc.Core" preserve="all"/>
    <assembly fullname="Grpc.Net.Client" preserve="all"/>

    <!-- Protobuf -->
    <assembly fullname="Google.Protobuf" preserve="all"/>

    <!-- 생성된 코드 -->
    <assembly fullname="Game.Proto" preserve="all"/>
</linker>
```

---

## 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────┐
│                     gRPC 베스트 프랙티스                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 채널 관리                                                    │
│     ├── GrpcChannel은 재사용 (싱글톤 또는 장기 보유)              │
│     ├── 앱 종료 시 ShutdownAsync() 호출                          │
│     └── 연결 상태 모니터링                                       │
│                                                                  │
│  2. 에러 핸들링                                                   │
│     ├── RpcException의 StatusCode로 분기                         │
│     ├── 일시적 오류는 재시도 (Unavailable, DeadlineExceeded)      │
│     └── 사용자 친화적 메시지 변환                                 │
│                                                                  │
│  3. 스트리밍                                                     │
│     ├── 대용량 데이터는 Client Streaming                          │
│     ├── 실시간 업데이트는 Server/Bidirectional Streaming         │
│     └── CancellationToken으로 스트림 종료                        │
│                                                                  │
│  4. 성능                                                         │
│     ├── Protobuf는 JSON보다 10배 빠름                            │
│     ├── HTTP/2 멀티플렉싱으로 연결 효율화                         │
│     └── 압축 활성화 (GzipCompressionProvider)                    │
│                                                                  │
│  5. 플랫폼 제한                                                   │
│     ├── WebGL 미지원 (REST fallback 필요)                        │
│     ├── IL2CPP: link.xml로 타입 보존                             │
│     └── iOS: HTTP/2 지원 확인                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [gRPC Official Documentation](https://grpc.io/docs/)
- [Protocol Buffers](https://developers.google.com/protocol-buffers)
- [gRPC for .NET](https://docs.microsoft.com/aspnet/core/grpc/)
- [Grpc.Net.Client NuGet](https://www.nuget.org/packages/Grpc.Net.Client)

---

## 다음 단계

- [Section 30: MessagePack](./30-messagepack.md) - 바이너리 직렬화
- [Section 31: MemoryPack](./31-memorypack.md) - 제로 카피 직렬화
