# 28. WebSocket

## 개요

WebSocket은 클라이언트와 서버 간의 **전이중(Full-Duplex) 양방향 통신**을 제공하는 프로토콜입니다. HTTP와 달리 한번 연결이 수립되면 양쪽 모두 자유롭게 메시지를 보내고 받을 수 있어, 실시간 게임 통신, 채팅, 매칭 시스템 등에 널리 사용됩니다. C#에서는 `System.Net.WebSockets.ClientWebSocket`을 기본 제공하며, Unity에서는 WebGL 플랫폼 지원을 위해 NativeWebSocket 같은 서드파티 라이브러리도 활용됩니다.

---

## 1. WebSocket 프로토콜 기초

### HTTP vs WebSocket 비교

```
┌─────────────────────────────────────────────────────────┐
│                  HTTP (요청-응답 모델)                    │
│                                                         │
│  Client ──── Request ────▶ Server                       │
│  Client ◀─── Response ─── Server                        │
│  (연결 종료)                                             │
│                                                         │
│  Client ──── Request ────▶ Server                       │
│  Client ◀─── Response ─── Server                        │
│  (연결 종료)                                             │
├─────────────────────────────────────────────────────────┤
│               WebSocket (양방향 모델)                    │
│                                                         │
│  Client ──── HTTP Upgrade ──▶ Server                    │
│  Client ◀─── 101 Switching ── Server                    │
│  ═══════════ 연결 유지 ═══════════                       │
│  Client ◀──▶ 메시지 자유 교환 ◀──▶ Server               │
│  ═══════════ Close ═══════════                           │
└─────────────────────────────────────────────────────────┘
```

### 프로토콜 특성 비교

```csharp
// WebSocket 연결은 HTTP Upgrade 요청으로 시작됩니다.
//
// 클라이언트 요청:
// GET /chat HTTP/1.1
// Upgrade: websocket
// Connection: Upgrade
// Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
// Sec-WebSocket-Version: 13
//
// 서버 응답:
// HTTP/1.1 101 Switching Protocols
// Upgrade: websocket
// Connection: Upgrade
// Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
//
// =============================================
// HTTP 특성
// =============================================
// - 단방향: 클라이언트 요청 → 서버 응답
// - Stateless: 매 요청이 독립적
// - 오버헤드: 매 요청마다 헤더 전송 (수백 bytes)
// - 적합: REST API, 파일 다운로드, 일반 웹 요청
//
// =============================================
// WebSocket 특성
// =============================================
// - 양방향: 서버도 능동적으로 메시지 전송 가능
// - Stateful: 연결이 유지되는 동안 상태 보존
// - 저오버헤드: 핸드셰이크 후 프레임 헤더만 추가 (2~14 bytes)
// - 적합: 실시간 게임, 채팅, 알림, 라이브 데이터
```

---

## 2. ClientWebSocket 기본 사용법

### 연결, 전송, 수신, 종료

```csharp
using System;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

public class WebSocketBasicsExample : MonoBehaviour
{
    private ClientWebSocket _webSocket;
    private CancellationTokenSource _cts;

    // =============================================
    // 기본 연결 및 메시지 교환
    // =============================================

    private async void Start()
    {
        _cts = new CancellationTokenSource();
        _webSocket = new ClientWebSocket();

        try
        {
            // 1. 서버에 연결
            Uri serverUri = new Uri("wss://echo.websocket.org");
            await _webSocket.ConnectAsync(serverUri, _cts.Token);
            Debug.Log($"WebSocket 연결 성공! 상태: {_webSocket.State}");

            // 2. 메시지 전송
            string message = "Hello, WebSocket!";
            byte[] sendBuffer = Encoding.UTF8.GetBytes(message);
            await _webSocket.SendAsync(
                new ArraySegment<byte>(sendBuffer),
                WebSocketMessageType.Text,
                endOfMessage: true,
                cancellationToken: _cts.Token
            );
            Debug.Log($"전송: {message}");

            // 3. 메시지 수신
            byte[] receiveBuffer = new byte[4096];
            WebSocketReceiveResult result = await _webSocket.ReceiveAsync(
                new ArraySegment<byte>(receiveBuffer), _cts.Token
            );
            string received = Encoding.UTF8.GetString(receiveBuffer, 0, result.Count);
            Debug.Log($"수신: {received} (타입: {result.MessageType})");

            // 4. 연결 종료
            await _webSocket.CloseAsync(
                WebSocketCloseStatus.NormalClosure,
                "클라이언트 정상 종료",
                _cts.Token
            );
            Debug.Log($"연결 종료. 상태: {_webSocket.State}");
        }
        catch (WebSocketException ex)
        {
            Debug.LogError($"WebSocket 에러: {ex.Message}");
        }
        catch (OperationCanceledException)
        {
            Debug.Log("WebSocket 작업이 취소되었습니다.");
        }
    }

    // =============================================
    // WebSocket 상태: None → Connecting → Open → CloseSent → Closed
    // Aborted = 비정상 종료
    // =============================================

    private void OnDestroy()
    {
        _cts?.Cancel();
        _webSocket?.Dispose();
        _cts?.Dispose();
    }
}
```

---

## 3. 연결 관리 클래스 설계

```csharp
using System;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// Unity에서 사용하기 위한 WebSocket 연결 관리 클래스.
/// 연결, 수신 루프, 종료를 일관된 인터페이스로 제공합니다.
/// </summary>
public class WebSocketManager : MonoBehaviour
{
    [SerializeField] private string serverUrl = "wss://example.com/ws";

    private ClientWebSocket _webSocket;
    private CancellationTokenSource _cts;
    private readonly byte[] _receiveBuffer = new byte[8192];

    public event Action OnConnected;
    public event Action<string> OnTextMessage;
    public event Action<byte[]> OnBinaryMessage;
    public event Action<string> OnError;
    public event Action<WebSocketCloseStatus?, string> OnClosed;

    public bool IsConnected => _webSocket?.State == WebSocketState.Open;

    // =============================================
    // 연결
    // =============================================

    public async Task ConnectAsync()
    {
        if (IsConnected) return;

        _cts = new CancellationTokenSource();
        _webSocket = new ClientWebSocket();
        _webSocket.Options.KeepAliveInterval = TimeSpan.FromSeconds(30);
        // _webSocket.Options.SetRequestHeader("Authorization", "Bearer token");

        try
        {
            await _webSocket.ConnectAsync(new Uri(serverUrl), _cts.Token);
            OnConnected?.Invoke();
            _ = ReceiveLoopAsync(_cts.Token);
        }
        catch (Exception ex)
        {
            OnError?.Invoke(ex.Message);
        }
    }

    // =============================================
    // 수신 루프 (큰 메시지를 여러 프레임으로 조립)
    // =============================================

    private async Task ReceiveLoopAsync(CancellationToken ct)
    {
        try
        {
            while (_webSocket.State == WebSocketState.Open && !ct.IsCancellationRequested)
            {
                using var ms = new System.IO.MemoryStream();
                WebSocketReceiveResult result;

                do
                {
                    result = await _webSocket.ReceiveAsync(
                        new ArraySegment<byte>(_receiveBuffer), ct);

                    if (result.MessageType == WebSocketMessageType.Close)
                    {
                        await _webSocket.CloseOutputAsync(
                            WebSocketCloseStatus.NormalClosure, "서버 요청 종료", ct);
                        OnClosed?.Invoke(result.CloseStatus, result.CloseStatusDescription);
                        return;
                    }
                    ms.Write(_receiveBuffer, 0, result.Count);
                }
                while (!result.EndOfMessage);

                byte[] messageData = ms.ToArray();
                if (result.MessageType == WebSocketMessageType.Text)
                    OnTextMessage?.Invoke(Encoding.UTF8.GetString(messageData));
                else if (result.MessageType == WebSocketMessageType.Binary)
                    OnBinaryMessage?.Invoke(messageData);
            }
        }
        catch (OperationCanceledException) { }
        catch (WebSocketException ex) { OnError?.Invoke(ex.Message); }
    }

    // =============================================
    // 메시지 전송
    // =============================================

    public async Task SendTextAsync(string message)
    {
        if (!IsConnected) return;
        byte[] buffer = Encoding.UTF8.GetBytes(message);
        await _webSocket.SendAsync(
            new ArraySegment<byte>(buffer),
            WebSocketMessageType.Text, true, _cts.Token);
    }

    public async Task SendBinaryAsync(byte[] data)
    {
        if (!IsConnected) return;
        await _webSocket.SendAsync(
            new ArraySegment<byte>(data),
            WebSocketMessageType.Binary, true, _cts.Token);
    }

    // =============================================
    // 종료 및 정리
    // =============================================

    public async Task DisconnectAsync()
    {
        if (_webSocket?.State == WebSocketState.Open)
        {
            try
            {
                await _webSocket.CloseAsync(
                    WebSocketCloseStatus.NormalClosure, "클라이언트 종료",
                    CancellationToken.None);
            }
            catch { }
        }
        Cleanup();
    }

    private void Cleanup()
    {
        _cts?.Cancel();
        _webSocket?.Dispose();
        _cts?.Dispose();
        _webSocket = null;
        _cts = null;
    }

    private void OnDestroy() => Cleanup();
}
```

---

## 4. 재연결 전략

### Exponential Backoff

```csharp
using System;
using System.Net.WebSockets;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// Exponential Backoff 재연결 전략을 구현한 WebSocket 클라이언트.
/// 연결이 끊어지면 점진적으로 대기 시간을 늘려가며 재연결을 시도합니다.
/// </summary>
public class ReconnectingWebSocket : MonoBehaviour
{
    [SerializeField] private string serverUrl = "wss://example.com/ws";
    [SerializeField] private float initialDelaySeconds = 1f;
    [SerializeField] private float maxDelaySeconds = 60f;
    [SerializeField] private int maxRetryCount = 10;
    [SerializeField] private float backoffMultiplier = 2f;

    private ClientWebSocket _webSocket;
    private CancellationTokenSource _cts;
    private int _retryCount;
    private bool _intentionalClose;

    public event Action OnConnected;
    public event Action<int> OnReconnecting;
    public event Action OnGaveUp;

    // =============================================
    // Exponential Backoff 계산 (Jitter 포함)
    // =============================================

    private float CalculateDelay()
    {
        // delay = initial * multiplier^retry + random jitter
        float delay = initialDelaySeconds * Mathf.Pow(backoffMultiplier, _retryCount);
        delay = Mathf.Min(delay, maxDelaySeconds);
        delay += UnityEngine.Random.Range(0f, delay * 0.3f); // Jitter: 동시 재접속 폭주 방지
        return delay;
    }

    // =============================================
    // 연결 및 자동 재연결
    // =============================================

    public async Task ConnectWithRetryAsync()
    {
        _cts = new CancellationTokenSource();
        _intentionalClose = false;
        _retryCount = 0;

        while (!_intentionalClose && _retryCount <= maxRetryCount && !_cts.IsCancellationRequested)
        {
            try
            {
                _webSocket?.Dispose();
                _webSocket = new ClientWebSocket();
                _webSocket.Options.KeepAliveInterval = TimeSpan.FromSeconds(30);

                // 연결 타임아웃
                using var connectCts = CancellationTokenSource.CreateLinkedTokenSource(_cts.Token);
                connectCts.CancelAfter(TimeSpan.FromSeconds(10));

                await _webSocket.ConnectAsync(new Uri(serverUrl), connectCts.Token);

                _retryCount = 0; // 연결 성공 시 카운터 리셋
                OnConnected?.Invoke();

                // 수신 루프 (연결 끊어지면 탈출)
                await ReceiveLoopAsync(_cts.Token);

                if (!_intentionalClose)
                    Debug.Log("연결이 끊어졌습니다. 재연결을 시도합니다.");
            }
            catch (OperationCanceledException) when (_intentionalClose) { return; }
            catch (Exception ex) { Debug.LogWarning($"연결 실패: {ex.Message}"); }

            if (_intentionalClose || _cts.IsCancellationRequested) return;

            _retryCount++;
            if (_retryCount > maxRetryCount) { OnGaveUp?.Invoke(); return; }

            float delay = CalculateDelay();
            Debug.Log($"재연결 대기: {delay:F1}초 ({_retryCount}/{maxRetryCount})");
            OnReconnecting?.Invoke(_retryCount);
            await Task.Delay(TimeSpan.FromSeconds(delay), _cts.Token);
        }
    }

    private async Task ReceiveLoopAsync(CancellationToken ct)
    {
        byte[] buffer = new byte[4096];
        while (_webSocket.State == WebSocketState.Open && !ct.IsCancellationRequested)
        {
            var result = await _webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), ct);
            if (result.MessageType == WebSocketMessageType.Close) return;
        }
    }

    // =============================================
    // 의도적 종료 (재연결 안 함)
    // =============================================

    public async Task DisconnectAsync()
    {
        _intentionalClose = true;
        _cts?.Cancel();
        if (_webSocket?.State == WebSocketState.Open)
        {
            try { await _webSocket.CloseAsync(
                WebSocketCloseStatus.NormalClosure, "정상 종료", CancellationToken.None); }
            catch { }
        }
        _webSocket?.Dispose();
    }

    private void OnDestroy()
    {
        _intentionalClose = true;
        _cts?.Cancel();
        _webSocket?.Dispose();
        _cts?.Dispose();
    }
}
```

### Heartbeat (Ping/Pong)

```csharp
using System;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// Heartbeat를 이용해 연결 상태를 모니터링합니다.
/// 주기적으로 Ping을 보내고, 일정 시간 내 Pong이 오지 않으면 재연결합니다.
/// </summary>
public class HeartbeatWebSocket : MonoBehaviour
{
    [SerializeField] private float heartbeatIntervalSeconds = 15f;
    [SerializeField] private float heartbeatTimeoutSeconds = 5f;

    private ClientWebSocket _webSocket;
    private DateTime _lastPongReceived;

    // =============================================
    // Heartbeat 루프
    // =============================================

    private async Task HeartbeatLoopAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested && _webSocket?.State == WebSocketState.Open)
        {
            // 애플리케이션 레벨 Ping 전송
            string ping = $"{{\"type\":\"ping\",\"ts\":{DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()}}}";
            byte[] pingBytes = Encoding.UTF8.GetBytes(ping);
            await _webSocket.SendAsync(
                new ArraySegment<byte>(pingBytes), WebSocketMessageType.Text, true, ct);

            await Task.Delay(TimeSpan.FromSeconds(heartbeatTimeoutSeconds), ct);

            // Pong 응답 확인
            double elapsed = (DateTime.UtcNow - _lastPongReceived).TotalSeconds;
            if (elapsed > heartbeatIntervalSeconds + heartbeatTimeoutSeconds)
            {
                Debug.LogWarning("[Heartbeat] Pong 타임아웃! 연결 끊김으로 판단.");
                _webSocket.Abort(); // 강제 종료 → 재연결 트리거
                return;
            }

            float remainingWait = heartbeatIntervalSeconds - heartbeatTimeoutSeconds;
            if (remainingWait > 0)
                await Task.Delay(TimeSpan.FromSeconds(remainingWait), ct);
        }
    }

    // =============================================
    // 수신 시 Pong 처리
    // =============================================

    private void HandleReceivedMessage(string message)
    {
        if (message.Contains("\"type\":\"pong\""))
        {
            _lastPongReceived = DateTime.UtcNow;
            return;
        }
        // 일반 메시지 처리...
    }
}
```

---

## 5. 메시지 프레이밍 및 바이너리/텍스트 메시지

### JSON 텍스트 메시지

```csharp
using System;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// JSON 기반 텍스트 메시지 프로토콜.
/// 메시지 타입별로 직렬화/역직렬화를 수행합니다.
/// </summary>
public class JsonWebSocketProtocol : MonoBehaviour
{
    private ClientWebSocket _webSocket;
    private CancellationTokenSource _cts;

    [Serializable] public class WebSocketMessage
    {
        public string type;
        public string payload;
        public long timestamp;
    }

    [Serializable] public class ChatMessage
    {
        public string sender;
        public string content;
        public string channel;
    }

    [Serializable] public class PlayerPosition
    {
        public string playerId;
        public float x, y, z;
    }

    // =============================================
    // 타입 안전한 메시지 전송
    // =============================================

    public async Task SendMessageAsync<T>(string type, T payload)
    {
        var message = new WebSocketMessage
        {
            type = type,
            payload = JsonUtility.ToJson(payload),
            timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
        };

        string json = JsonUtility.ToJson(message);
        byte[] buffer = Encoding.UTF8.GetBytes(json);
        await _webSocket.SendAsync(
            new ArraySegment<byte>(buffer),
            WebSocketMessageType.Text, true, _cts.Token);
    }

    // =============================================
    // 메시지 수신 및 타입별 분기
    // =============================================

    private void HandleTextMessage(string json)
    {
        var message = JsonUtility.FromJson<WebSocketMessage>(json);

        switch (message.type)
        {
            case "chat":
                var chat = JsonUtility.FromJson<ChatMessage>(message.payload);
                Debug.Log($"[채팅] {chat.sender}: {chat.content}");
                break;
            case "position":
                var pos = JsonUtility.FromJson<PlayerPosition>(message.payload);
                Debug.Log($"[위치] {pos.playerId}: ({pos.x}, {pos.y}, {pos.z})");
                break;
            default:
                Debug.LogWarning($"알 수 없는 메시지 타입: {message.type}");
                break;
        }
    }
}
```

### 바이너리 메시지 (고성능)

```csharp
using System;
using System.IO;
using System.Net.WebSockets;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

/// <summary>
/// 바이너리 프로토콜을 사용하는 WebSocket 통신.
/// JSON보다 적은 대역폭을 사용하며 파싱이 빠릅니다.
/// </summary>
public class BinaryWebSocketProtocol : MonoBehaviour
{
    private ClientWebSocket _webSocket;
    private CancellationTokenSource _cts;

    private enum MessageType : byte
    {
        Ping = 0x00, Pong = 0x01,
        Chat = 0x10,
        PlayerMove = 0x20, PlayerAction = 0x21,
        GameState = 0x30,
    }

    // =============================================
    // 바이너리 직렬화: [1byte 타입][4byte x][4byte y][4byte z][4byte timestamp]
    // 총 17 bytes (JSON 동일 데이터 ~100 bytes 이상)
    // =============================================

    public async Task SendPlayerMoveAsync(Vector3 position)
    {
        using var ms = new MemoryStream(17);
        using var writer = new BinaryWriter(ms);

        writer.Write((byte)MessageType.PlayerMove);
        writer.Write(position.x);
        writer.Write(position.y);
        writer.Write(position.z);
        writer.Write((int)(Time.time * 1000));

        await _webSocket.SendAsync(
            new ArraySegment<byte>(ms.ToArray()),
            WebSocketMessageType.Binary, true, _cts.Token);
    }

    // =============================================
    // 바이너리 역직렬화
    // =============================================

    private void HandleBinaryMessage(byte[] data)
    {
        if (data.Length == 0) return;
        using var ms = new MemoryStream(data);
        using var reader = new BinaryReader(ms);

        var type = (MessageType)reader.ReadByte();
        switch (type)
        {
            case MessageType.PlayerMove:
                float x = reader.ReadSingle(), y = reader.ReadSingle(), z = reader.ReadSingle();
                int ts = reader.ReadInt32();
                Debug.Log($"[바이너리] 이동: ({x},{y},{z}) t={ts}");
                break;
            case MessageType.Chat:
                Debug.Log($"[바이너리] 채팅: {reader.ReadString()}");
                break;
            case MessageType.GameState:
                Debug.Log($"[바이너리] 플레이어 {reader.ReadInt32()}명");
                break;
        }
    }
}
```

---

## 6. NativeWebSocket 라이브러리 (WebGL 지원)

```csharp
// WebGL 빌드에서는 .NET의 System.Net.WebSockets가 지원되지 않습니다.
// 브라우저 환경이므로 JavaScript WebSocket API를 사용해야 합니다.
//
// NativeWebSocket은 이 문제를 해결합니다:
// - 스탠드얼론/모바일: .NET ClientWebSocket 사용
// - WebGL: JavaScript WebSocket API 브릿지
//
// 설치: Package Manager → Add package from git URL
// https://github.com/endel/NativeWebSocket.git#upm

using UnityEngine;
using NativeWebSocket; // NativeWebSocket 패키지 필요

public class NativeWebSocketExample : MonoBehaviour
{
    private WebSocket _webSocket;

    private async void Start()
    {
        _webSocket = new WebSocket("wss://echo.websocket.org");

        _webSocket.OnOpen += () => Debug.Log("연결 성공!");
        _webSocket.OnMessage += (bytes) =>
        {
            string message = System.Text.Encoding.UTF8.GetString(bytes);
            Debug.Log($"수신: {message}");
        };
        _webSocket.OnError += (error) => Debug.LogError($"에러: {error}");
        _webSocket.OnClose += (code) => Debug.Log($"종료: {code}");

        await _webSocket.Connect();
    }

    // =============================================
    // WebGL에서 중요: Update에서 DispatchMessageQueue 호출 필수
    // 브라우저의 JS 이벤트를 Unity 메인 스레드로 전달합니다.
    // =============================================

    private void Update()
    {
        #if !UNITY_WEBGL || UNITY_EDITOR
            _webSocket?.DispatchMessageQueue();
        #endif
    }

    public async void SendText(string msg)
    {
        if (_webSocket.State == WebSocketState.Open)
            await _webSocket.SendText(msg);
    }

    private async void OnApplicationQuit()
    {
        if (_webSocket?.State == WebSocketState.Open)
            await _webSocket.Close();
    }

    private void OnDestroy() => _webSocket?.CancelConnection();
}
```

### 플랫폼 분기 래퍼

```csharp
using System;
using System.Threading.Tasks;

/// <summary>
/// ClientWebSocket과 NativeWebSocket을 추상화하여
/// 플랫폼에 관계없이 동일한 인터페이스를 제공합니다.
/// </summary>
public interface IWebSocketClient
{
    bool IsConnected { get; }
    event Action OnConnected;
    event Action<string> OnTextReceived;
    event Action<byte[]> OnBinaryReceived;
    event Action<string> OnError;
    event Action OnDisconnected;

    Task ConnectAsync(string url);
    Task SendTextAsync(string message);
    Task SendBinaryAsync(byte[] data);
    Task CloseAsync();
    void Tick(); // Update에서 호출 (WebGL용)
}

// 팩토리 메서드로 플랫폼별 인스턴스 생성
public static class WebSocketClientFactory
{
    public static IWebSocketClient Create()
    {
        #if UNITY_WEBGL && !UNITY_EDITOR
            return new WebGLWebSocketClient();   // NativeWebSocket 기반
        #else
            return new StandaloneWebSocketClient(); // ClientWebSocket 기반
        #endif
    }
}
```

---

## 7. WebSocket과 UniTask 통합

```csharp
using System;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine;

/// <summary>
/// UniTask를 활용한 WebSocket 클라이언트.
/// Zero-allocation과 PlayerLoop 통합으로 Unity에 최적화된 비동기 통신을 구현합니다.
/// </summary>
public class UniTaskWebSocket : MonoBehaviour
{
    [SerializeField] private string serverUrl = "wss://example.com/ws";

    private ClientWebSocket _webSocket;
    private CancellationTokenSource _cts;
    private readonly byte[] _receiveBuffer = new byte[8192];

    public event Action<string> OnMessage;

    // =============================================
    // UniTask 기반 연결 (수신 루프 + Heartbeat 동시 실행)
    // =============================================

    public async UniTask ConnectAsync(CancellationToken ct = default)
    {
        _cts = CancellationTokenSource.CreateLinkedTokenSource(
            ct, this.GetCancellationTokenOnDestroy());

        _webSocket = new ClientWebSocket();
        _webSocket.Options.KeepAliveInterval = TimeSpan.FromSeconds(30);

        await _webSocket.ConnectAsync(new Uri(serverUrl), _cts.Token);
        Debug.Log("UniTask WebSocket 연결 성공!");

        await UniTask.WhenAll(
            ReceiveLoopAsync(_cts.Token),
            HeartbeatAsync(_cts.Token)
        );
    }

    // =============================================
    // UniTask 수신 루프
    // =============================================

    private async UniTask ReceiveLoopAsync(CancellationToken ct)
    {
        try
        {
            while (_webSocket.State == WebSocketState.Open && !ct.IsCancellationRequested)
            {
                var result = await _webSocket.ReceiveAsync(
                    new ArraySegment<byte>(_receiveBuffer), ct);

                if (result.MessageType == WebSocketMessageType.Close) return;

                if (result.MessageType == WebSocketMessageType.Text)
                {
                    string message = Encoding.UTF8.GetString(_receiveBuffer, 0, result.Count);
                    await UniTask.SwitchToMainThread(ct);
                    OnMessage?.Invoke(message);
                }

                await UniTask.Yield(PlayerLoopTiming.Update, ct);
            }
        }
        catch (OperationCanceledException) { }
        catch (Exception ex) { Debug.LogError($"수신 에러: {ex.Message}"); }
    }

    // =============================================
    // UniTask Heartbeat (PlayerLoop 기반, GC 부담 없음)
    // =============================================

    private async UniTask HeartbeatAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested && _webSocket?.State == WebSocketState.Open)
        {
            await UniTask.Delay(TimeSpan.FromSeconds(15), cancellationToken: ct);
            if (_webSocket?.State == WebSocketState.Open)
            {
                byte[] ping = Encoding.UTF8.GetBytes("{\"type\":\"ping\"}");
                await _webSocket.SendAsync(
                    new ArraySegment<byte>(ping), WebSocketMessageType.Text, true, ct);
            }
        }
    }

    // =============================================
    // UniTask 기반 재연결 (Exponential Backoff)
    // =============================================

    public async UniTask ConnectWithRetryAsync(int maxRetries = 5)
    {
        var ct = this.GetCancellationTokenOnDestroy();

        for (int i = 0; i < maxRetries; i++)
        {
            try
            {
                await ConnectAsync(ct);
                return;
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                Debug.LogWarning($"연결 실패 ({i + 1}/{maxRetries}): {ex.Message}");
                float delay = Mathf.Min(1f * Mathf.Pow(2, i), 30f);
                await UniTask.Delay(
                    TimeSpan.FromSeconds(delay),
                    ignoreTimeScale: true,
                    cancellationToken: ct);
            }
        }
        Debug.LogError("최대 재시도 횟수 초과");
    }

    public async UniTask SendAsync(string message)
    {
        if (_webSocket?.State != WebSocketState.Open) return;
        byte[] buffer = Encoding.UTF8.GetBytes(message);
        await _webSocket.SendAsync(
            new ArraySegment<byte>(buffer), WebSocketMessageType.Text, true, _cts.Token);
    }

    private void OnDestroy()
    {
        _cts?.Cancel();
        _webSocket?.Dispose();
        _cts?.Dispose();
    }
}
```

---

## 8. 실시간 게임 통신 패턴

### 채팅 시스템

```csharp
using System;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.UI;

public class WebSocketChatSystem : MonoBehaviour
{
    [SerializeField] private InputField chatInput;
    [SerializeField] private Text chatLog;
    [SerializeField] private string serverUrl = "wss://chat.example.com/ws";
    [SerializeField] private string username = "Player";

    private ClientWebSocket _webSocket;
    private CancellationTokenSource _cts;

    [Serializable] public class ChatProtocol
    {
        public string action;  // "join", "leave", "message", "users"
        public string channel;
        public string sender;
        public string content;
        public string[] users;
    }

    private async void Start()
    {
        _cts = new CancellationTokenSource();
        _webSocket = new ClientWebSocket();
        await _webSocket.ConnectAsync(new Uri(serverUrl), _cts.Token);

        // 채널 참가
        await SendJsonAsync(new ChatProtocol { action = "join", channel = "global", sender = username });
        _ = ReceiveLoopAsync();
    }

    public async void OnSendButtonClick()
    {
        if (string.IsNullOrWhiteSpace(chatInput.text)) return;
        await SendJsonAsync(new ChatProtocol
        {
            action = "message", channel = "global",
            sender = username, content = chatInput.text
        });
        chatInput.text = "";
    }

    private async Task SendJsonAsync(ChatProtocol msg)
    {
        byte[] buffer = Encoding.UTF8.GetBytes(JsonUtility.ToJson(msg));
        await _webSocket.SendAsync(
            new ArraySegment<byte>(buffer), WebSocketMessageType.Text, true, _cts.Token);
    }

    private async Task ReceiveLoopAsync()
    {
        byte[] buffer = new byte[4096];
        while (_webSocket.State == WebSocketState.Open && !_cts.IsCancellationRequested)
        {
            var result = await _webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), _cts.Token);
            if (result.MessageType != WebSocketMessageType.Text) continue;

            var msg = JsonUtility.FromJson<ChatProtocol>(
                Encoding.UTF8.GetString(buffer, 0, result.Count));

            switch (msg.action)
            {
                case "message": AppendChat($"[{msg.sender}] {msg.content}"); break;
                case "join": AppendChat($"*** {msg.sender}님이 입장 ***"); break;
                case "leave": AppendChat($"*** {msg.sender}님이 퇴장 ***"); break;
            }
        }
    }

    private void AppendChat(string text) => chatLog.text += text + "\n";

    private void OnDestroy()
    {
        _cts?.Cancel();
        _webSocket?.Dispose();
        _cts?.Dispose();
    }
}
```

### 매칭 시스템

```csharp
using System;
using System.Net.WebSockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

public class WebSocketMatchmaking : MonoBehaviour
{
    private ClientWebSocket _webSocket;
    private CancellationTokenSource _cts;

    [Serializable] public class MatchRequest
    {
        public string action;    // "queue", "cancel", "accept"
        public string playerId;
        public string gameMode;  // "1v1", "2v2", "battle_royale"
        public int rating;
    }

    [Serializable] public class MatchResponse
    {
        public string status;    // "queued", "found", "started", "cancelled"
        public string matchId;
        public int estimatedWait;
        public string serverAddress;
        public int serverPort;
    }

    public event Action<int> OnQueueUpdate;
    public event Action<string, int> OnMatchReady;

    public async Task JoinMatchmakingAsync(string gameMode, int rating)
    {
        _cts = new CancellationTokenSource();
        _webSocket = new ClientWebSocket();
        await _webSocket.ConnectAsync(new Uri("wss://matchmaking.example.com/ws"), _cts.Token);

        await SendAsync(JsonUtility.ToJson(new MatchRequest
        {
            action = "queue",
            playerId = SystemInfo.deviceUniqueIdentifier,
            gameMode = gameMode,
            rating = rating
        }));

        // 매칭 결과 수신 루프
        byte[] buffer = new byte[4096];
        while (_webSocket.State == WebSocketState.Open)
        {
            var result = await _webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), _cts.Token);
            if (result.MessageType != WebSocketMessageType.Text) continue;

            var response = JsonUtility.FromJson<MatchResponse>(
                Encoding.UTF8.GetString(buffer, 0, result.Count));

            switch (response.status)
            {
                case "queued":
                    OnQueueUpdate?.Invoke(response.estimatedWait);
                    break;
                case "found":
                    await SendAsync(JsonUtility.ToJson(new MatchRequest
                        { action = "accept", playerId = SystemInfo.deviceUniqueIdentifier }));
                    break;
                case "started":
                    OnMatchReady?.Invoke(response.serverAddress, response.serverPort);
                    return;
                case "cancelled":
                    return;
            }
        }
    }

    public async Task CancelAsync()
    {
        await SendAsync(JsonUtility.ToJson(new MatchRequest
            { action = "cancel", playerId = SystemInfo.deviceUniqueIdentifier }));
        await _webSocket.CloseAsync(WebSocketCloseStatus.NormalClosure, "취소", _cts.Token);
    }

    private async Task SendAsync(string message)
    {
        byte[] buffer = Encoding.UTF8.GetBytes(message);
        await _webSocket.SendAsync(
            new ArraySegment<byte>(buffer), WebSocketMessageType.Text, true, _cts.Token);
    }

    private void OnDestroy()
    {
        _cts?.Cancel();
        _webSocket?.Dispose();
        _cts?.Dispose();
    }
}
```

---

## 9. 올바른 패턴과 잘못된 패턴

```csharp
// ✅ 올바른 패턴: CancellationToken으로 생명주기 관리 + OnDestroy 정리
public class GoodWebSocketLifecycle : MonoBehaviour
{
    private ClientWebSocket _ws;
    private CancellationTokenSource _cts;

    private async void Start()
    {
        _cts = new CancellationTokenSource();
        _ws = new ClientWebSocket();
        try
        {
            await _ws.ConnectAsync(new Uri("wss://example.com"), _cts.Token);
        }
        catch (OperationCanceledException) { }
        catch (WebSocketException ex) { Debug.LogError(ex.Message); }
    }

    private void OnDestroy()
    {
        _cts?.Cancel();
        _ws?.Dispose();
        _cts?.Dispose();
    }
}

// ❌ 잘못된 패턴: 리소스 정리 누락 + 취소 불가능
public class BadWebSocketLifecycle : MonoBehaviour
{
    private ClientWebSocket _ws;

    private async void Start()
    {
        _ws = new ClientWebSocket();
        await _ws.ConnectAsync(new Uri("wss://example.com"), CancellationToken.None);
        // OnDestroy 없음 → 리소스 누수!
        // CancellationToken.None → 취소 불가!
    }
}

// ✅ 올바른 패턴: 연결 상태 확인 후 전송
public static class WebSocketExtensions
{
    public static async System.Threading.Tasks.Task SafeSendAsync(
        this ClientWebSocket ws, string message, CancellationToken ct)
    {
        if (ws.State != WebSocketState.Open) return;
        byte[] buffer = System.Text.Encoding.UTF8.GetBytes(message);
        await ws.SendAsync(new ArraySegment<byte>(buffer), WebSocketMessageType.Text, true, ct);
    }
}

// ❌ 잘못된 패턴: 상태 확인 없이 전송 → WebSocketException 발생
// await _ws.SendAsync(...); // State가 Open이 아닐 수 있음!

// ✅ 올바른 패턴: EndOfMessage로 큰 메시지 조립
public static async System.Threading.Tasks.Task<string> ReceiveFullAsync(
    ClientWebSocket ws, CancellationToken ct)
{
    var buffer = new byte[4096];
    using var ms = new System.IO.MemoryStream();
    WebSocketReceiveResult result;
    do
    {
        result = await ws.ReceiveAsync(new ArraySegment<byte>(buffer), ct);
        ms.Write(buffer, 0, result.Count);
    } while (!result.EndOfMessage); // 전체 메시지가 올 때까지

    return System.Text.Encoding.UTF8.GetString(ms.ToArray());
}

// ❌ 잘못된 패턴: EndOfMessage 무시 → 큰 메시지의 일부만 수신
// var result = await ws.ReceiveAsync(buffer, ct);
// return Encoding.UTF8.GetString(buffer, 0, result.Count); // 불완전!
```

---

## 주의사항

1. **WebGL 제약**: `System.Net.WebSockets.ClientWebSocket`은 WebGL에서 사용 불가합니다. WebGL 빌드가 필요하면 NativeWebSocket이나 유사 라이브러리를 사용하세요.

2. **메인 스레드 접근**: `ClientWebSocket`의 수신 루프는 스레드 풀에서 실행될 수 있으므로, Unity API(Transform, GameObject 등)에 접근하려면 반드시 메인 스레드로 디스패치해야 합니다.

3. **리소스 정리**: `ClientWebSocket`은 `IDisposable`입니다. `OnDestroy`에서 반드시 `Dispose()`를 호출하세요. `CancellationTokenSource`도 마찬가지입니다.

4. **Close 핸드셰이크**: `CloseAsync()`는 Close 프레임을 보내고 상대방의 응답을 기다립니다. 즉시 종료가 필요하면 `Abort()`를 사용하세요.

5. **동시 전송 금지**: `ClientWebSocket`은 동시에 여러 `SendAsync`를 호출하면 안 됩니다. Channel 등으로 전송 큐를 직렬화하세요.

6. **네트워크 변경 감지**: 모바일에서 Wi-Fi ↔ 셀룰러 전환 시 연결이 끊어집니다. `Application.internetReachability`를 모니터링하고 재연결하세요.

7. **SSL/TLS**: 프로덕션에서는 반드시 `wss://`(WebSocket Secure)를 사용하세요.

8. **메시지 크기**: 서버에 따라 메시지 크기 제한이 있을 수 있습니다. 큰 데이터는 분할하거나 HTTP를 사용하세요.

---

## 베스트 프랙티스

| 항목 | 권장 사항 |
|------|----------|
| **프로토콜** | 프로덕션에서는 항상 `wss://` 사용 |
| **재연결** | Exponential Backoff + Jitter로 서버 부하 분산 |
| **Heartbeat** | 15~30초 간격으로 Ping/Pong 주기적 전송 |
| **메시지 포맷** | 가독성 필요 시 JSON, 고성능 필요 시 바이너리 |
| **전송 직렬화** | Channel이나 큐를 통해 동시 전송 방지 |
| **수신 버퍼** | 예상 메시지 크기에 맞게 설정 (기본 4~8KB) |
| **큰 메시지** | `EndOfMessage` 플래그로 전체 메시지 조립 |
| **CancellationToken** | `GetCancellationTokenOnDestroy()` 또는 `CancellationTokenSource` 활용 |
| **플랫폼 분기** | `IWebSocketClient` 인터페이스로 WebGL/스탠드얼론 추상화 |
| **에러 처리** | `WebSocketException`, `OperationCanceledException` 구분 처리 |

---

## 참고 자료

- [MDN: WebSocket API](https://developer.mozilla.org/ko/docs/Web/API/WebSocket)
- [RFC 6455: The WebSocket Protocol](https://tools.ietf.org/html/rfc6455)
- [Microsoft: ClientWebSocket Class](https://learn.microsoft.com/dotnet/api/system.net.websockets.clientwebsocket)
- [NativeWebSocket (GitHub)](https://github.com/endel/NativeWebSocket)
- [UniTask (GitHub)](https://github.com/Cysharp/UniTask)

---

## 다음 섹션

[29. gRPC](./29-grpc.md)
