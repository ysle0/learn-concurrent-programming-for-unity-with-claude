# Section 28: WebSocket

## 개요

WebSocket은 클라이언트와 서버 간의 양방향 실시간 통신을 제공하는 프로토콜입니다. HTTP와 달리 연결을 유지하며, 서버에서 클라이언트로 즉시 데이터를 푸시할 수 있어 실시간 게임에 적합합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    WebSocket vs HTTP                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   HTTP (Request-Response)              WebSocket (Full-Duplex)  │
│   ┌────────┐     ┌────────┐           ┌────────┐     ┌────────┐ │
│   │ Client │────▶│ Server │           │ Client │◀───▶│ Server │ │
│   └────────┘◀────└────────┘           └────────┘     └────────┘ │
│                                                                  │
│   클라이언트가 먼저 요청               양방향 언제든 전송 가능     │
│   매 요청마다 연결 생성                연결 유지 (Persistent)      │
│   오버헤드 큼                          오버헤드 작음               │
│                                                                  │
│   WebSocket 연결 수립 과정:                                       │
│   ┌────────┐  HTTP Upgrade Request   ┌────────┐                 │
│   │ Client │─────────────────────────▶│ Server │                 │
│   │        │◀─────────────────────────│        │                 │
│   │        │  101 Switching Protocols │        │                 │
│   │        │◀════════════════════════▶│        │                 │
│   └────────┘  WebSocket Connection    └────────┘                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## WebSocket 사용 사례

| 사용 사례 | 설명 |
|----------|------|
| **실시간 멀티플레이어** | 플레이어 위치, 액션 동기화 |
| **채팅 시스템** | 실시간 메시지 전달 |
| **라이브 알림** | 이벤트, 보상 즉시 알림 |
| **경매/거래소** | 실시간 가격 업데이트 |
| **매치메이킹** | 실시간 매칭 상태 알림 |
| **협동 플레이** | 팀 상태 동기화 |

---

## 기본 WebSocket 클라이언트

### System.Net.WebSockets 사용

```csharp
using System;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 기본 WebSocket 클라이언트
/// </summary>
public class BasicWebSocketClient : IDisposable
{
    private ClientWebSocket _socket;
    private CancellationTokenSource _cts;
    private readonly int _receiveBufferSize;

    public event Action OnConnected;
    public event Action<string> OnMessage;
    public event Action<byte[]> OnBinaryMessage;
    public event Action<string> OnError;
    public event Action<WebSocketCloseStatus?, string> OnClosed;

    public bool IsConnected => _socket?.State == WebSocketState.Open;

    public BasicWebSocketClient(int receiveBufferSize = 8192)
    {
        _receiveBufferSize = receiveBufferSize;
    }

    /// <summary>
    /// WebSocket 서버에 연결
    /// </summary>
    public async Task ConnectAsync(string url, CancellationToken ct = default)
    {
        if (_socket != null)
        {
            throw new InvalidOperationException("이미 연결되어 있습니다.");
        }

        _socket = new ClientWebSocket();
        _cts = CancellationTokenSource.CreateLinkedTokenSource(ct);

        try
        {
            var uri = new Uri(url);
            await _socket.ConnectAsync(uri, _cts.Token);

            OnConnected?.Invoke();

            // 수신 루프 시작
            _ = ReceiveLoopAsync(_cts.Token);
        }
        catch (Exception ex)
        {
            OnError?.Invoke($"연결 실패: {ex.Message}");
            throw;
        }
    }

    /// <summary>
    /// 텍스트 메시지 전송
    /// </summary>
    public async Task SendAsync(string message, CancellationToken ct = default)
    {
        if (!IsConnected)
        {
            throw new InvalidOperationException("연결되지 않았습니다.");
        }

        var bytes = Encoding.UTF8.GetBytes(message);
        var segment = new ArraySegment<byte>(bytes);

        await _socket.SendAsync(
            segment,
            WebSocketMessageType.Text,
            endOfMessage: true,
            cancellationToken: ct);
    }

    /// <summary>
    /// 바이너리 메시지 전송
    /// </summary>
    public async Task SendBinaryAsync(byte[] data, CancellationToken ct = default)
    {
        if (!IsConnected)
        {
            throw new InvalidOperationException("연결되지 않았습니다.");
        }

        var segment = new ArraySegment<byte>(data);

        await _socket.SendAsync(
            segment,
            WebSocketMessageType.Binary,
            endOfMessage: true,
            cancellationToken: ct);
    }

    /// <summary>
    /// 연결 종료
    /// </summary>
    public async Task CloseAsync(
        WebSocketCloseStatus status = WebSocketCloseStatus.NormalClosure,
        string reason = "Client closing",
        CancellationToken ct = default)
    {
        if (_socket == null) return;

        try
        {
            if (_socket.State == WebSocketState.Open)
            {
                await _socket.CloseAsync(status, reason, ct);
            }
        }
        catch (Exception ex)
        {
            Debug.LogWarning($"종료 중 오류: {ex.Message}");
        }
    }

    /// <summary>
    /// 메시지 수신 루프
    /// </summary>
    private async Task ReceiveLoopAsync(CancellationToken ct)
    {
        var buffer = new byte[_receiveBufferSize];
        var messageBuffer = new System.IO.MemoryStream();

        try
        {
            while (_socket.State == WebSocketState.Open && !ct.IsCancellationRequested)
            {
                var segment = new ArraySegment<byte>(buffer);
                var result = await _socket.ReceiveAsync(segment, ct);

                if (result.MessageType == WebSocketMessageType.Close)
                {
                    OnClosed?.Invoke(result.CloseStatus, result.CloseStatusDescription);
                    break;
                }

                messageBuffer.Write(buffer, 0, result.Count);

                if (result.EndOfMessage)
                {
                    var messageBytes = messageBuffer.ToArray();
                    messageBuffer.SetLength(0);

                    if (result.MessageType == WebSocketMessageType.Text)
                    {
                        string message = Encoding.UTF8.GetString(messageBytes);
                        OnMessage?.Invoke(message);
                    }
                    else if (result.MessageType == WebSocketMessageType.Binary)
                    {
                        OnBinaryMessage?.Invoke(messageBytes);
                    }
                }
            }
        }
        catch (OperationCanceledException)
        {
            // 정상 취소
        }
        catch (WebSocketException ex)
        {
            OnError?.Invoke($"WebSocket 오류: {ex.Message}");
        }
        catch (Exception ex)
        {
            OnError?.Invoke($"수신 오류: {ex.Message}");
        }
        finally
        {
            messageBuffer.Dispose();
        }
    }

    public void Dispose()
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _socket?.Dispose();
        _socket = null;
    }
}
```

### Unity MonoBehaviour 통합

```csharp
using System;
using System.Threading;
using UnityEngine;

/// <summary>
/// Unity에서 WebSocket 사용 예시
/// </summary>
public class WebSocketManager : MonoBehaviour
{
    [SerializeField] private string serverUrl = "wss://game.example.com/ws";

    private BasicWebSocketClient _client;
    private CancellationTokenSource _cts;

    // 메인 스레드에서 실행할 액션 큐
    private readonly System.Collections.Concurrent.ConcurrentQueue<Action> _mainThreadQueue
        = new();

    public event Action<string> OnMessageReceived;

    private void Awake()
    {
        _client = new BasicWebSocketClient();
        _cts = new CancellationTokenSource();

        // 이벤트 핸들러 등록
        _client.OnConnected += HandleConnected;
        _client.OnMessage += HandleMessage;
        _client.OnError += HandleError;
        _client.OnClosed += HandleClosed;
    }

    private async void Start()
    {
        await ConnectAsync();
    }

    private void Update()
    {
        // 메인 스레드에서 큐에 있는 액션 실행
        while (_mainThreadQueue.TryDequeue(out var action))
        {
            action?.Invoke();
        }
    }

    private async System.Threading.Tasks.Task ConnectAsync()
    {
        try
        {
            Debug.Log($"WebSocket 연결 중: {serverUrl}");
            await _client.ConnectAsync(serverUrl, _cts.Token);
        }
        catch (Exception ex)
        {
            Debug.LogError($"연결 실패: {ex.Message}");
            // 재연결 시도
            await RetryConnectAsync();
        }
    }

    private async System.Threading.Tasks.Task RetryConnectAsync()
    {
        int retryCount = 0;
        int maxRetries = 5;
        int baseDelay = 1000;

        while (retryCount < maxRetries && !_cts.IsCancellationRequested)
        {
            retryCount++;
            int delay = baseDelay * (int)Math.Pow(2, retryCount - 1);

            Debug.Log($"재연결 시도 {retryCount}/{maxRetries} ({delay}ms 후)");

            await System.Threading.Tasks.Task.Delay(delay, _cts.Token);

            try
            {
                _client?.Dispose();
                _client = new BasicWebSocketClient();
                _client.OnConnected += HandleConnected;
                _client.OnMessage += HandleMessage;
                _client.OnError += HandleError;
                _client.OnClosed += HandleClosed;

                await _client.ConnectAsync(serverUrl, _cts.Token);
                return; // 성공
            }
            catch (Exception ex)
            {
                Debug.LogWarning($"재연결 실패: {ex.Message}");
            }
        }

        Debug.LogError("모든 재연결 시도 실패");
    }

    public async void SendMessage(string message)
    {
        try
        {
            await _client.SendAsync(message, _cts.Token);
        }
        catch (Exception ex)
        {
            Debug.LogError($"전송 실패: {ex.Message}");
        }
    }

    // 이벤트를 메인 스레드로 마샬링
    private void HandleConnected()
    {
        _mainThreadQueue.Enqueue(() =>
        {
            Debug.Log("WebSocket 연결됨!");
        });
    }

    private void HandleMessage(string message)
    {
        _mainThreadQueue.Enqueue(() =>
        {
            Debug.Log($"메시지 수신: {message}");
            OnMessageReceived?.Invoke(message);
        });
    }

    private void HandleError(string error)
    {
        _mainThreadQueue.Enqueue(() =>
        {
            Debug.LogError($"WebSocket 오류: {error}");
        });
    }

    private void HandleClosed(System.Net.WebSockets.WebSocketCloseStatus? status, string reason)
    {
        _mainThreadQueue.Enqueue(() =>
        {
            Debug.Log($"WebSocket 종료: {status} - {reason}");
            // 예기치 않은 종료 시 재연결
            if (status != System.Net.WebSockets.WebSocketCloseStatus.NormalClosure)
            {
                _ = RetryConnectAsync();
            }
        });
    }

    private async void OnDestroy()
    {
        _cts?.Cancel();

        if (_client != null)
        {
            await _client.CloseAsync();
            _client.Dispose();
        }

        _cts?.Dispose();
    }
}
```

---

## UniTask 통합 WebSocket 클라이언트

```csharp
using System;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine;

/// <summary>
/// UniTask 기반 WebSocket 클라이언트
/// </summary>
public class UniTaskWebSocketClient : IDisposable
{
    private ClientWebSocket _socket;
    private CancellationTokenSource _receiveCts;
    private readonly int _bufferSize;

    public event Action OnConnected;
    public event Action<string> OnTextMessage;
    public event Action<byte[]> OnBinaryMessage;
    public event Action<Exception> OnError;
    public event Action<WebSocketCloseStatus?, string> OnDisconnected;

    public bool IsConnected => _socket?.State == WebSocketState.Open;
    public WebSocketState State => _socket?.State ?? WebSocketState.None;

    public UniTaskWebSocketClient(int bufferSize = 8192)
    {
        _bufferSize = bufferSize;
    }

    /// <summary>
    /// 연결
    /// </summary>
    public async UniTask ConnectAsync(
        string url,
        CancellationToken ct = default)
    {
        if (_socket != null && _socket.State == WebSocketState.Open)
        {
            throw new InvalidOperationException("이미 연결됨");
        }

        _socket = new ClientWebSocket();
        _receiveCts = CancellationTokenSource.CreateLinkedTokenSource(ct);

        try
        {
            await _socket.ConnectAsync(new Uri(url), ct);
            OnConnected?.Invoke();

            // 백그라운드 수신 시작
            ReceiveLoopAsync(_receiveCts.Token).Forget();
        }
        catch (Exception ex)
        {
            OnError?.Invoke(ex);
            throw;
        }
    }

    /// <summary>
    /// 텍스트 전송
    /// </summary>
    public async UniTask SendTextAsync(string message, CancellationToken ct = default)
    {
        ThrowIfNotConnected();

        var bytes = Encoding.UTF8.GetBytes(message);
        await _socket.SendAsync(
            new ArraySegment<byte>(bytes),
            WebSocketMessageType.Text,
            true,
            ct);
    }

    /// <summary>
    /// 바이너리 전송
    /// </summary>
    public async UniTask SendBinaryAsync(byte[] data, CancellationToken ct = default)
    {
        ThrowIfNotConnected();

        await _socket.SendAsync(
            new ArraySegment<byte>(data),
            WebSocketMessageType.Binary,
            true,
            ct);
    }

    /// <summary>
    /// JSON 객체 전송
    /// </summary>
    public UniTask SendJsonAsync<T>(T data, CancellationToken ct = default)
    {
        string json = JsonUtility.ToJson(data);
        return SendTextAsync(json, ct);
    }

    /// <summary>
    /// 연결 종료
    /// </summary>
    public async UniTask DisconnectAsync(
        WebSocketCloseStatus status = WebSocketCloseStatus.NormalClosure,
        string reason = "Client disconnect",
        CancellationToken ct = default)
    {
        _receiveCts?.Cancel();

        if (_socket?.State == WebSocketState.Open)
        {
            try
            {
                await _socket.CloseAsync(status, reason, ct);
            }
            catch (Exception ex)
            {
                Debug.LogWarning($"Disconnect error: {ex.Message}");
            }
        }
    }

    /// <summary>
    /// 수신 루프
    /// </summary>
    private async UniTaskVoid ReceiveLoopAsync(CancellationToken ct)
    {
        var buffer = new byte[_bufferSize];
        var messageBuffer = new System.IO.MemoryStream();

        try
        {
            while (_socket.State == WebSocketState.Open && !ct.IsCancellationRequested)
            {
                var result = await _socket.ReceiveAsync(
                    new ArraySegment<byte>(buffer),
                    ct);

                if (result.MessageType == WebSocketMessageType.Close)
                {
                    // PlayerLoop에서 실행
                    await UniTask.SwitchToMainThread();
                    OnDisconnected?.Invoke(result.CloseStatus, result.CloseStatusDescription);
                    break;
                }

                messageBuffer.Write(buffer, 0, result.Count);

                if (result.EndOfMessage)
                {
                    var data = messageBuffer.ToArray();
                    messageBuffer.SetLength(0);

                    // 메인 스레드에서 이벤트 발생
                    await UniTask.SwitchToMainThread();

                    if (result.MessageType == WebSocketMessageType.Text)
                    {
                        OnTextMessage?.Invoke(Encoding.UTF8.GetString(data));
                    }
                    else
                    {
                        OnBinaryMessage?.Invoke(data);
                    }
                }
            }
        }
        catch (OperationCanceledException)
        {
            // 정상 취소
        }
        catch (Exception ex)
        {
            await UniTask.SwitchToMainThread();
            OnError?.Invoke(ex);
        }
        finally
        {
            messageBuffer.Dispose();
        }
    }

    private void ThrowIfNotConnected()
    {
        if (_socket?.State != WebSocketState.Open)
        {
            throw new InvalidOperationException("WebSocket not connected");
        }
    }

    public void Dispose()
    {
        _receiveCts?.Cancel();
        _receiveCts?.Dispose();
        _socket?.Dispose();
    }
}

/// <summary>
/// UniTask WebSocket 사용 예시
/// </summary>
public class UniTaskWebSocketExample : MonoBehaviour
{
    [Serializable]
    public class ChatMessage
    {
        public string type;
        public string sender;
        public string content;
        public long timestamp;
    }

    private UniTaskWebSocketClient _ws;

    private async void Start()
    {
        _ws = new UniTaskWebSocketClient();

        _ws.OnConnected += () => Debug.Log("연결됨!");
        _ws.OnTextMessage += HandleMessage;
        _ws.OnError += ex => Debug.LogError($"오류: {ex.Message}");
        _ws.OnDisconnected += (status, reason) =>
            Debug.Log($"연결 종료: {status} - {reason}");

        try
        {
            await _ws.ConnectAsync("wss://chat.example.com/ws", destroyCancellationToken);

            // 채팅 메시지 전송
            var message = new ChatMessage
            {
                type = "chat",
                sender = "Player1",
                content = "Hello, World!",
                timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
            };

            await _ws.SendJsonAsync(message, destroyCancellationToken);
        }
        catch (Exception ex)
        {
            Debug.LogError($"연결 실패: {ex.Message}");
        }
    }

    private void HandleMessage(string json)
    {
        var message = JsonUtility.FromJson<ChatMessage>(json);
        Debug.Log($"[{message.sender}]: {message.content}");
    }

    private async void OnDestroy()
    {
        if (_ws != null)
        {
            await _ws.DisconnectAsync();
            _ws.Dispose();
        }
    }
}
```

---

## 게임용 WebSocket 프로토콜 구현

### 메시지 프로토콜 설계

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// 게임 메시지 타입
/// </summary>
public enum GameMessageType : byte
{
    // 연결 관리
    Ping = 0,
    Pong = 1,
    Auth = 2,
    AuthResponse = 3,

    // 게임 상태
    JoinRoom = 10,
    LeaveRoom = 11,
    RoomState = 12,
    PlayerJoined = 13,
    PlayerLeft = 14,

    // 게임플레이
    PlayerInput = 20,
    PlayerState = 21,
    GameEvent = 22,
    WorldState = 23,

    // 채팅
    ChatMessage = 30,
    SystemMessage = 31,

    // 에러
    Error = 255
}

/// <summary>
/// 기본 메시지 구조
/// </summary>
[Serializable]
public class GameMessage
{
    public int type;
    public long timestamp;
    public string payload;

    public GameMessageType MessageType => (GameMessageType)type;

    public T GetPayload<T>()
    {
        return JsonUtility.FromJson<T>(payload);
    }

    public static GameMessage Create<T>(GameMessageType type, T payload)
    {
        return new GameMessage
        {
            type = (int)type,
            timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
            payload = JsonUtility.ToJson(payload)
        };
    }
}

/// <summary>
/// 플레이어 입력 메시지
/// </summary>
[Serializable]
public class PlayerInputPayload
{
    public int sequence;      // 시퀀스 번호 (클라이언트 예측용)
    public float horizontal;
    public float vertical;
    public bool jump;
    public bool fire;
    public float aimX;
    public float aimY;
}

/// <summary>
/// 플레이어 상태 메시지
/// </summary>
[Serializable]
public class PlayerStatePayload
{
    public string playerId;
    public int lastProcessedInput;  // 서버가 처리한 마지막 입력
    public float posX, posY, posZ;
    public float rotY;
    public float health;
    public int score;
}

/// <summary>
/// 월드 상태 스냅샷
/// </summary>
[Serializable]
public class WorldStatePayload
{
    public long serverTime;
    public PlayerStatePayload[] players;
    public GameObjectState[] objects;
}

[Serializable]
public class GameObjectState
{
    public string objectId;
    public int objectType;
    public float posX, posY, posZ;
    public float rotX, rotY, rotZ, rotW;
    public string customData;
}
```

### 실시간 게임 클라이언트

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 실시간 멀티플레이어 게임 클라이언트
/// </summary>
public class RealtimeGameClient : MonoBehaviour
{
    [SerializeField] private string serverUrl = "wss://game.example.com/realtime";
    [SerializeField] private float pingInterval = 5f;
    [SerializeField] private float reconnectDelay = 2f;
    [SerializeField] private int maxReconnectAttempts = 5;

    private UniTaskWebSocketClient _ws;
    private CancellationTokenSource _cts;
    private int _inputSequence;
    private bool _isAuthenticated;

    // 메시지 핸들러
    private Dictionary<GameMessageType, Action<string>> _handlers;

    // 이벤트
    public event Action OnConnected;
    public event Action OnDisconnected;
    public event Action<string> OnPlayerJoined;
    public event Action<string> OnPlayerLeft;
    public event Action<WorldStatePayload> OnWorldStateReceived;
    public event Action<PlayerStatePayload> OnPlayerStateReceived;

    private void Awake()
    {
        _handlers = new Dictionary<GameMessageType, Action<string>>
        {
            { GameMessageType.Pong, HandlePong },
            { GameMessageType.AuthResponse, HandleAuthResponse },
            { GameMessageType.RoomState, HandleRoomState },
            { GameMessageType.PlayerJoined, HandlePlayerJoined },
            { GameMessageType.PlayerLeft, HandlePlayerLeft },
            { GameMessageType.PlayerState, HandlePlayerState },
            { GameMessageType.WorldState, HandleWorldState },
            { GameMessageType.Error, HandleError }
        };
    }

    /// <summary>
    /// 서버에 연결
    /// </summary>
    public async UniTask ConnectAsync(string authToken, CancellationToken ct = default)
    {
        _cts = CancellationTokenSource.CreateLinkedTokenSource(ct);

        _ws = new UniTaskWebSocketClient();
        _ws.OnConnected += () => OnWsConnected(authToken);
        _ws.OnTextMessage += HandleMessage;
        _ws.OnError += HandleWsError;
        _ws.OnDisconnected += HandleWsDisconnected;

        await _ws.ConnectAsync(serverUrl, _cts.Token);
    }

    /// <summary>
    /// 연결 해제
    /// </summary>
    public async UniTask DisconnectAsync()
    {
        _cts?.Cancel();

        if (_ws != null)
        {
            await _ws.DisconnectAsync();
            _ws.Dispose();
            _ws = null;
        }
    }

    /// <summary>
    /// 플레이어 입력 전송
    /// </summary>
    public void SendInput(float horizontal, float vertical, bool jump, bool fire, Vector2 aim)
    {
        if (!_isAuthenticated || _ws?.IsConnected != true) return;

        var input = new PlayerInputPayload
        {
            sequence = ++_inputSequence,
            horizontal = horizontal,
            vertical = vertical,
            jump = jump,
            fire = fire,
            aimX = aim.x,
            aimY = aim.y
        };

        var message = GameMessage.Create(GameMessageType.PlayerInput, input);
        _ws.SendTextAsync(JsonUtility.ToJson(message)).Forget();
    }

    /// <summary>
    /// 방 입장
    /// </summary>
    public async UniTask JoinRoomAsync(string roomId, CancellationToken ct = default)
    {
        var message = GameMessage.Create(GameMessageType.JoinRoom, new { roomId });
        await _ws.SendTextAsync(JsonUtility.ToJson(message), ct);
    }

    /// <summary>
    /// 방 퇴장
    /// </summary>
    public async UniTask LeaveRoomAsync(CancellationToken ct = default)
    {
        var message = GameMessage.Create(GameMessageType.LeaveRoom, new { });
        await _ws.SendTextAsync(JsonUtility.ToJson(message), ct);
    }

    #region WebSocket Event Handlers

    private void OnWsConnected(string authToken)
    {
        Debug.Log("WebSocket 연결됨, 인증 중...");

        // 인증 메시지 전송
        var authMessage = GameMessage.Create(GameMessageType.Auth, new { token = authToken });
        _ws.SendTextAsync(JsonUtility.ToJson(authMessage)).Forget();

        // Ping 루프 시작
        StartPingLoop(_cts.Token).Forget();
    }

    private void HandleMessage(string json)
    {
        try
        {
            var message = JsonUtility.FromJson<GameMessage>(json);

            if (_handlers.TryGetValue(message.MessageType, out var handler))
            {
                handler(message.payload);
            }
            else
            {
                Debug.LogWarning($"알 수 없는 메시지 타입: {message.MessageType}");
            }
        }
        catch (Exception ex)
        {
            Debug.LogError($"메시지 처리 오류: {ex.Message}");
        }
    }

    private void HandleWsError(Exception ex)
    {
        Debug.LogError($"WebSocket 오류: {ex.Message}");
    }

    private void HandleWsDisconnected(
        System.Net.WebSockets.WebSocketCloseStatus? status,
        string reason)
    {
        Debug.Log($"연결 종료: {status} - {reason}");
        _isAuthenticated = false;
        OnDisconnected?.Invoke();

        // 비정상 종료 시 재연결
        if (status != System.Net.WebSockets.WebSocketCloseStatus.NormalClosure)
        {
            TryReconnectAsync(_cts.Token).Forget();
        }
    }

    #endregion

    #region Message Handlers

    private void HandlePong(string payload)
    {
        // Ping-Pong 처리 (연결 상태 확인)
    }

    private void HandleAuthResponse(string payload)
    {
        var response = JsonUtility.FromJson<AuthResponsePayload>(payload);

        if (response.success)
        {
            _isAuthenticated = true;
            Debug.Log("인증 성공!");
            OnConnected?.Invoke();
        }
        else
        {
            Debug.LogError($"인증 실패: {response.error}");
        }
    }

    [Serializable]
    private class AuthResponsePayload
    {
        public bool success;
        public string playerId;
        public string error;
    }

    private void HandleRoomState(string payload)
    {
        var state = JsonUtility.FromJson<RoomStatePayload>(payload);
        Debug.Log($"방 상태: {state.playerCount}명 참가 중");
    }

    [Serializable]
    private class RoomStatePayload
    {
        public string roomId;
        public int playerCount;
        public string[] playerIds;
    }

    private void HandlePlayerJoined(string payload)
    {
        var data = JsonUtility.FromJson<PlayerEventPayload>(payload);
        Debug.Log($"플레이어 입장: {data.playerId}");
        OnPlayerJoined?.Invoke(data.playerId);
    }

    private void HandlePlayerLeft(string payload)
    {
        var data = JsonUtility.FromJson<PlayerEventPayload>(payload);
        Debug.Log($"플레이어 퇴장: {data.playerId}");
        OnPlayerLeft?.Invoke(data.playerId);
    }

    [Serializable]
    private class PlayerEventPayload
    {
        public string playerId;
        public string playerName;
    }

    private void HandlePlayerState(string payload)
    {
        var state = JsonUtility.FromJson<PlayerStatePayload>(payload);
        OnPlayerStateReceived?.Invoke(state);
    }

    private void HandleWorldState(string payload)
    {
        var state = JsonUtility.FromJson<WorldStatePayload>(payload);
        OnWorldStateReceived?.Invoke(state);
    }

    private void HandleError(string payload)
    {
        var error = JsonUtility.FromJson<ErrorPayload>(payload);
        Debug.LogError($"서버 오류: [{error.code}] {error.message}");
    }

    [Serializable]
    private class ErrorPayload
    {
        public string code;
        public string message;
    }

    #endregion

    #region Connection Management

    private async UniTaskVoid StartPingLoop(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested && _ws?.IsConnected == true)
        {
            await UniTask.Delay(TimeSpan.FromSeconds(pingInterval), cancellationToken: ct);

            if (_ws?.IsConnected == true)
            {
                var ping = GameMessage.Create(GameMessageType.Ping, new { });
                await _ws.SendTextAsync(JsonUtility.ToJson(ping), ct);
            }
        }
    }

    private async UniTaskVoid TryReconnectAsync(CancellationToken ct)
    {
        for (int i = 0; i < maxReconnectAttempts; i++)
        {
            if (ct.IsCancellationRequested) break;

            float delay = reconnectDelay * Mathf.Pow(2, i);
            Debug.Log($"재연결 시도 {i + 1}/{maxReconnectAttempts} ({delay}초 후)");

            await UniTask.Delay(TimeSpan.FromSeconds(delay), cancellationToken: ct);

            try
            {
                _ws?.Dispose();
                _ws = new UniTaskWebSocketClient();
                await _ws.ConnectAsync(serverUrl, ct);

                Debug.Log("재연결 성공!");
                return;
            }
            catch (Exception ex)
            {
                Debug.LogWarning($"재연결 실패: {ex.Message}");
            }
        }

        Debug.LogError("모든 재연결 시도 실패");
    }

    #endregion

    private void OnDestroy()
    {
        DisconnectAsync().Forget();
    }
}
```

---

## 클라이언트 예측 및 서버 보정

```csharp
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// 클라이언트 예측 및 서버 보정 시스템
/// </summary>
public class ClientPrediction : MonoBehaviour
{
    [SerializeField] private float reconciliationThreshold = 0.5f;
    [SerializeField] private float interpolationSpeed = 10f;

    private RealtimeGameClient _gameClient;

    // 입력 히스토리 (서버 보정용)
    private Queue<PendingInput> _pendingInputs = new Queue<PendingInput>();
    private int _lastAcknowledgedInput;

    // 현재 상태
    private Vector3 _predictedPosition;
    private Vector3 _serverPosition;

    private struct PendingInput
    {
        public int Sequence;
        public float Horizontal;
        public float Vertical;
        public bool Jump;
        public float DeltaTime;
    }

    private void Awake()
    {
        _gameClient = GetComponent<RealtimeGameClient>();
        _gameClient.OnPlayerStateReceived += HandleServerState;
    }

    /// <summary>
    /// 로컬 입력 처리 (즉시 예측)
    /// </summary>
    public void ProcessInput(float horizontal, float vertical, bool jump, Vector2 aim)
    {
        // 1. 서버에 입력 전송
        _gameClient.SendInput(horizontal, vertical, jump, false, aim);

        // 2. 입력 히스토리에 저장
        var input = new PendingInput
        {
            Sequence = _pendingInputs.Count > 0
                ? _pendingInputs.ToArray()[_pendingInputs.Count - 1].Sequence + 1
                : 1,
            Horizontal = horizontal,
            Vertical = vertical,
            Jump = jump,
            DeltaTime = Time.deltaTime
        };
        _pendingInputs.Enqueue(input);

        // 3. 로컬 예측 적용
        ApplyInput(ref _predictedPosition, input);
    }

    /// <summary>
    /// 서버 상태 수신 시 보정
    /// </summary>
    private void HandleServerState(PlayerStatePayload state)
    {
        _serverPosition = new Vector3(state.posX, state.posY, state.posZ);
        _lastAcknowledgedInput = state.lastProcessedInput;

        // 이미 처리된 입력 제거
        while (_pendingInputs.Count > 0 &&
               _pendingInputs.Peek().Sequence <= _lastAcknowledgedInput)
        {
            _pendingInputs.Dequeue();
        }

        // 서버 상태에서 시작하여 미처리 입력 재적용
        Vector3 reconciledPosition = _serverPosition;

        foreach (var input in _pendingInputs)
        {
            ApplyInput(ref reconciledPosition, input);
        }

        // 예측 오차 확인
        float error = Vector3.Distance(reconciledPosition, _predictedPosition);

        if (error > reconciliationThreshold)
        {
            Debug.Log($"서버 보정 적용 (오차: {error:F2})");
            _predictedPosition = reconciledPosition;
        }
    }

    /// <summary>
    /// 입력을 위치에 적용 (예측 로직)
    /// </summary>
    private void ApplyInput(ref Vector3 position, PendingInput input)
    {
        float speed = 5f;
        Vector3 movement = new Vector3(input.Horizontal, 0, input.Vertical);
        movement = movement.normalized * speed * input.DeltaTime;
        position += movement;
    }

    private void Update()
    {
        // 예측된 위치로 부드럽게 이동
        transform.position = Vector3.Lerp(
            transform.position,
            _predictedPosition,
            Time.deltaTime * interpolationSpeed);
    }
}
```

---

## 엔티티 보간

```csharp
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// 원격 엔티티 보간 시스템
/// </summary>
public class EntityInterpolation : MonoBehaviour
{
    [SerializeField] private float interpolationDelay = 0.1f; // 100ms

    private Queue<EntitySnapshot> _snapshots = new Queue<EntitySnapshot>();
    private float _renderTime;

    private struct EntitySnapshot
    {
        public float Timestamp;
        public Vector3 Position;
        public Quaternion Rotation;
    }

    /// <summary>
    /// 서버에서 상태 수신
    /// </summary>
    public void ReceiveState(float timestamp, Vector3 position, Quaternion rotation)
    {
        _snapshots.Enqueue(new EntitySnapshot
        {
            Timestamp = timestamp,
            Position = position,
            Rotation = rotation
        });

        // 오래된 스냅샷 제거 (최대 1초)
        while (_snapshots.Count > 0 &&
               _snapshots.Peek().Timestamp < timestamp - 1f)
        {
            _snapshots.Dequeue();
        }
    }

    private void Update()
    {
        // 보간 시점 계산 (현재 시간 - 지연)
        _renderTime = Time.time - interpolationDelay;

        if (_snapshots.Count < 2) return;

        var snapshotsArray = _snapshots.ToArray();

        // 보간할 두 스냅샷 찾기
        EntitySnapshot? before = null;
        EntitySnapshot? after = null;

        for (int i = 0; i < snapshotsArray.Length - 1; i++)
        {
            if (snapshotsArray[i].Timestamp <= _renderTime &&
                snapshotsArray[i + 1].Timestamp >= _renderTime)
            {
                before = snapshotsArray[i];
                after = snapshotsArray[i + 1];
                break;
            }
        }

        if (before.HasValue && after.HasValue)
        {
            // 두 스냅샷 사이 보간
            float t = (_renderTime - before.Value.Timestamp) /
                     (after.Value.Timestamp - before.Value.Timestamp);

            transform.position = Vector3.Lerp(
                before.Value.Position,
                after.Value.Position,
                t);

            transform.rotation = Quaternion.Slerp(
                before.Value.Rotation,
                after.Value.Rotation,
                t);
        }
        else if (snapshotsArray.Length > 0)
        {
            // 스냅샷 부족 시 마지막 위치 사용
            var last = snapshotsArray[snapshotsArray.Length - 1];
            transform.position = last.Position;
            transform.rotation = last.Rotation;
        }
    }
}
```

---

## WebGL 호환 WebSocket

```csharp
using System;
using System.Runtime.InteropServices;
using AOT;
using UnityEngine;

/// <summary>
/// WebGL용 JavaScript WebSocket 브릿지
/// </summary>
public class WebGLWebSocket
{
#if UNITY_WEBGL && !UNITY_EDITOR
    [DllImport("__Internal")]
    private static extern int WebSocketConnect(string url);

    [DllImport("__Internal")]
    private static extern int WebSocketClose(int instanceId);

    [DllImport("__Internal")]
    private static extern int WebSocketSend(int instanceId, string message);

    [DllImport("__Internal")]
    private static extern int WebSocketSendBinary(int instanceId, byte[] data, int length);

    [DllImport("__Internal")]
    private static extern int WebSocketGetState(int instanceId);
#endif

    private int _instanceId = -1;
    private static System.Collections.Generic.Dictionary<int, WebGLWebSocket> _instances
        = new System.Collections.Generic.Dictionary<int, WebGLWebSocket>();

    public event Action OnOpen;
    public event Action<string> OnMessage;
    public event Action<byte[]> OnBinary;
    public event Action<string> OnError;
    public event Action<int> OnClose;

    public void Connect(string url)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        _instanceId = WebSocketConnect(url);
        _instances[_instanceId] = this;
#else
        Debug.LogError("WebGLWebSocket은 WebGL 빌드에서만 사용 가능합니다.");
#endif
    }

    public void Send(string message)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        if (_instanceId >= 0)
        {
            WebSocketSend(_instanceId, message);
        }
#endif
    }

    public void SendBinary(byte[] data)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        if (_instanceId >= 0)
        {
            WebSocketSendBinary(_instanceId, data, data.Length);
        }
#endif
    }

    public void Close()
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        if (_instanceId >= 0)
        {
            WebSocketClose(_instanceId);
            _instances.Remove(_instanceId);
            _instanceId = -1;
        }
#endif
    }

    // JavaScript에서 호출되는 콜백
    [MonoPInvokeCallback(typeof(Action<int>))]
    private static void OnOpenCallback(int instanceId)
    {
        if (_instances.TryGetValue(instanceId, out var ws))
        {
            ws.OnOpen?.Invoke();
        }
    }

    [MonoPInvokeCallback(typeof(Action<int, string>))]
    private static void OnMessageCallback(int instanceId, string message)
    {
        if (_instances.TryGetValue(instanceId, out var ws))
        {
            ws.OnMessage?.Invoke(message);
        }
    }
}
```

**WebGL JavaScript 플러그인 (Plugins/WebGL/WebSocket.jslib)**:
```javascript
var WebSocketPlugin = {
    $webSockets: [],

    WebSocketConnect: function(urlPtr) {
        var url = UTF8ToString(urlPtr);
        var id = webSockets.length;

        var ws = new WebSocket(url);
        ws.binaryType = 'arraybuffer';

        ws.onopen = function() {
            SendMessage('WebSocketManager', 'OnOpen', id);
        };

        ws.onmessage = function(event) {
            if (typeof event.data === 'string') {
                SendMessage('WebSocketManager', 'OnMessage', id + '|' + event.data);
            } else {
                // Binary data handling
                var array = new Uint8Array(event.data);
                // ... binary handling
            }
        };

        ws.onerror = function(error) {
            SendMessage('WebSocketManager', 'OnError', id + '|' + error.message);
        };

        ws.onclose = function(event) {
            SendMessage('WebSocketManager', 'OnClose', id + '|' + event.code);
        };

        webSockets.push(ws);
        return id;
    },

    WebSocketSend: function(instanceId, messagePtr) {
        var message = UTF8ToString(messagePtr);
        if (webSockets[instanceId]) {
            webSockets[instanceId].send(message);
        }
    },

    WebSocketClose: function(instanceId) {
        if (webSockets[instanceId]) {
            webSockets[instanceId].close();
        }
    }
};

autoAddDeps(WebSocketPlugin, '$webSockets');
mergeInto(LibraryManager.library, WebSocketPlugin);
```

---

## 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────┐
│                   WebSocket 베스트 프랙티스                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 연결 관리                                                    │
│     ├── 자동 재연결 (지수 백오프)                                 │
│     ├── Heartbeat/Ping-Pong으로 연결 상태 확인                   │
│     └── 연결 실패 시 사용자에게 명확한 피드백                      │
│                                                                  │
│  2. 메시지 설계                                                   │
│     ├── 메시지 타입 열거형 정의                                   │
│     ├── 시퀀스 번호로 순서 보장                                   │
│     └── 타임스탬프로 지연 시간 측정                               │
│                                                                  │
│  3. 성능 최적화                                                   │
│     ├── 바이너리 프로토콜 사용 (대역폭 절약)                       │
│     ├── 메시지 배칭 (작은 메시지 묶어서 전송)                      │
│     └── 객체 풀링으로 GC 최소화                                   │
│                                                                  │
│  4. 지연 시간 보정                                                │
│     ├── 클라이언트 예측                                          │
│     ├── 서버 보정 (Reconciliation)                               │
│     └── 엔티티 보간 (Interpolation)                              │
│                                                                  │
│  5. 보안                                                         │
│     ├── WSS (WebSocket Secure) 필수                              │
│     ├── 연결 시 토큰 인증                                         │
│     └── 서버에서 모든 입력 검증                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [WebSocket RFC 6455](https://tools.ietf.org/html/rfc6455)
- [ClientWebSocket - Microsoft Docs](https://docs.microsoft.com/dotnet/api/system.net.websockets.clientwebsocket)
- [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
- [Fast-Paced Multiplayer](https://www.gabrielgambetta.com/client-server-game-architecture.html)

---

## 다음 단계

- [Section 29: gRPC](./29-grpc.md) - 고성능 RPC 프로토콜
- [Section 30: MessagePack](./30-messagepack.md) - 바이너리 직렬화
