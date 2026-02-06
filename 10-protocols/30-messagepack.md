# Section 30: MessagePack

## 개요

MessagePack은 효율적인 바이너리 직렬화 포맷입니다. JSON과 유사한 구조를 가지지만 더 작고 빠릅니다. Unity에서는 MessagePack-CSharp 라이브러리를 통해 사용하며, 네트워크 통신 및 데이터 저장에 적합합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    MessagePack vs JSON                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   JSON (텍스트):                                                 │
│   {"name":"Hero","level":42,"hp":100.5}                         │
│   크기: 38 bytes                                                 │
│                                                                  │
│   MessagePack (바이너리):                                        │
│   83 A4 6E 61 6D 65 A4 48 65 72 6F A5 6C 65 76 65               │
│   6C 2A A2 68 70 CB 40 59 20 00 00 00 00 00                     │
│   크기: 26 bytes (32% 절감)                                      │
│                                                                  │
│   비교:                                                          │
│   ┌───────────────┬──────────────┬──────────────┐               │
│   │     항목      │     JSON     │  MessagePack │               │
│   ├───────────────┼──────────────┼──────────────┤               │
│   │ 크기          │    100%      │    50-70%    │               │
│   │ 직렬화 속도   │    1x        │    2-10x     │               │
│   │ 역직렬화 속도 │    1x        │    2-10x     │               │
│   │ 가독성        │    높음      │    낮음      │               │
│   │ 스키마 필요   │    아니오    │    아니오    │               │
│   └───────────────┴──────────────┴──────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 설치

```
Unity Package Manager 또는 NuGet:

1. MessagePack-CSharp (권장)
   - https://github.com/neuecc/MessagePack-CSharp
   - UPM: https://github.com/neuecc/MessagePack-CSharp.git?path=src/MessagePack.UnityClient/Assets/Scripts/MessagePack

2. 필요한 패키지:
   - MessagePack
   - MessagePack.Annotations
   - MessagePack.Unity (Unity 전용 Resolver)
```

---

## 기본 사용법

### 기본 직렬화/역직렬화

```csharp
using MessagePack;
using UnityEngine;

/// <summary>
/// MessagePack 기본 예제
/// </summary>
public class MessagePackBasicExample : MonoBehaviour
{
    // [MessagePackObject]로 직렬화 가능 표시
    [MessagePackObject]
    public class PlayerData
    {
        // [Key]로 속성 인덱스 지정 (정수 키 사용 - 더 효율적)
        [Key(0)]
        public string PlayerName { get; set; }

        [Key(1)]
        public int Level { get; set; }

        [Key(2)]
        public float Health { get; set; }

        [Key(3)]
        public long Experience { get; set; }

        [Key(4)]
        public int[] Inventory { get; set; }

        // [IgnoreMember]로 직렬화 제외
        [IgnoreMember]
        public string TempData { get; set; }
    }

    private void Start()
    {
        // 객체 생성
        var player = new PlayerData
        {
            PlayerName = "Hero",
            Level = 42,
            Health = 100.5f,
            Experience = 1234567890L,
            Inventory = new[] { 1, 2, 3, 4, 5 }
        };

        // 직렬화 (객체 → byte[])
        byte[] bytes = MessagePackSerializer.Serialize(player);
        Debug.Log($"직렬화 크기: {bytes.Length} bytes");

        // 역직렬화 (byte[] → 객체)
        PlayerData loaded = MessagePackSerializer.Deserialize<PlayerData>(bytes);
        Debug.Log($"플레이어: {loaded.PlayerName}, Lv.{loaded.Level}");

        // JSON으로 변환 (디버깅용)
        string json = MessagePackSerializer.ConvertToJson(bytes);
        Debug.Log($"JSON: {json}");
    }
}
```

### 문자열 키 사용

```csharp
using MessagePack;

/// <summary>
/// 문자열 키를 사용한 직렬화
/// 정수 키보다 약간 느리지만 가독성 좋음
/// </summary>
[MessagePackObject(keyAsPropertyName: true)]
public class GameSettings
{
    public float MusicVolume { get; set; }
    public float SfxVolume { get; set; }
    public bool Fullscreen { get; set; }
    public string Language { get; set; }
    public int QualityLevel { get; set; }
}

/// <summary>
/// 명시적 문자열 키 지정
/// </summary>
[MessagePackObject]
public class UserProfile
{
    [Key("user_id")]
    public string UserId { get; set; }

    [Key("display_name")]
    public string DisplayName { get; set; }

    [Key("avatar_url")]
    public string AvatarUrl { get; set; }

    [Key("created_at")]
    public long CreatedAt { get; set; }
}
```

---

## Unity 타입 지원

### Unity Resolver 설정

```csharp
using MessagePack;
using MessagePack.Resolvers;
using MessagePack.Unity;
using UnityEngine;

/// <summary>
/// MessagePack Unity 초기화
/// </summary>
public static class MessagePackConfig
{
    private static bool _initialized;

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    public static void Initialize()
    {
        if (_initialized) return;

        // Unity용 Resolver 설정
        var resolver = CompositeResolver.Create(
            // Unity 타입 지원 (Vector3, Quaternion, Color 등)
            UnityResolver.Instance,

            // 기본 Resolver
            StandardResolver.Instance
        );

        var options = MessagePackSerializerOptions.Standard.WithResolver(resolver);
        MessagePackSerializer.DefaultOptions = options;

        _initialized = true;
    }
}
```

### Unity 타입 직렬화

```csharp
using MessagePack;
using UnityEngine;

/// <summary>
/// Unity 타입을 포함한 데이터 클래스
/// </summary>
[MessagePackObject]
public class TransformData
{
    [Key(0)]
    public Vector3 Position { get; set; }

    [Key(1)]
    public Quaternion Rotation { get; set; }

    [Key(2)]
    public Vector3 Scale { get; set; }
}

[MessagePackObject]
public class RenderData
{
    [Key(0)]
    public Color Color { get; set; }

    [Key(1)]
    public Vector2 UVOffset { get; set; }

    [Key(2)]
    public Bounds Bounds { get; set; }
}

/// <summary>
/// 게임 오브젝트 상태 저장/로드
/// </summary>
public class GameObjectSerializer : MonoBehaviour
{
    [MessagePackObject]
    public class GameObjectState
    {
        [Key(0)]
        public string Name { get; set; }

        [Key(1)]
        public Vector3 Position { get; set; }

        [Key(2)]
        public Quaternion Rotation { get; set; }

        [Key(3)]
        public Vector3 LocalScale { get; set; }

        [Key(4)]
        public bool IsActive { get; set; }
    }

    /// <summary>
    /// 현재 상태를 바이트 배열로 직렬화
    /// </summary>
    public byte[] SerializeState()
    {
        var state = new GameObjectState
        {
            Name = gameObject.name,
            Position = transform.position,
            Rotation = transform.rotation,
            LocalScale = transform.localScale,
            IsActive = gameObject.activeSelf
        };

        return MessagePackSerializer.Serialize(state);
    }

    /// <summary>
    /// 바이트 배열에서 상태 복원
    /// </summary>
    public void DeserializeState(byte[] data)
    {
        var state = MessagePackSerializer.Deserialize<GameObjectState>(data);

        gameObject.name = state.Name;
        transform.position = state.Position;
        transform.rotation = state.Rotation;
        transform.localScale = state.LocalScale;
        gameObject.SetActive(state.IsActive);
    }
}
```

---

## 네트워크 통신 예제

### MessagePack 기반 네트워크 프로토콜

```csharp
using System;
using System.Collections.Generic;
using MessagePack;
using UnityEngine;

/// <summary>
/// 네트워크 메시지 기본 클래스
/// </summary>
[MessagePackObject]
[Union(0, typeof(LoginRequest))]
[Union(1, typeof(LoginResponse))]
[Union(2, typeof(PlayerStateMessage))]
[Union(3, typeof(ChatMessage))]
[Union(4, typeof(GameEventMessage))]
public abstract class NetworkMessage
{
    [Key(0)]
    public long Timestamp { get; set; }

    [Key(1)]
    public int MessageId { get; set; }

    protected NetworkMessage()
    {
        Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
    }
}

[MessagePackObject]
public class LoginRequest : NetworkMessage
{
    [Key(2)]
    public string Username { get; set; }

    [Key(3)]
    public string Password { get; set; }

    [Key(4)]
    public string DeviceId { get; set; }
}

[MessagePackObject]
public class LoginResponse : NetworkMessage
{
    [Key(2)]
    public bool Success { get; set; }

    [Key(3)]
    public string PlayerId { get; set; }

    [Key(4)]
    public string Token { get; set; }

    [Key(5)]
    public string ErrorMessage { get; set; }
}

[MessagePackObject]
public class PlayerStateMessage : NetworkMessage
{
    [Key(2)]
    public string PlayerId { get; set; }

    [Key(3)]
    public Vector3 Position { get; set; }

    [Key(4)]
    public float RotationY { get; set; }

    [Key(5)]
    public int AnimationState { get; set; }

    [Key(6)]
    public float Health { get; set; }
}

[MessagePackObject]
public class ChatMessage : NetworkMessage
{
    [Key(2)]
    public string SenderId { get; set; }

    [Key(3)]
    public string SenderName { get; set; }

    [Key(4)]
    public string Content { get; set; }

    [Key(5)]
    public int Channel { get; set; } // 0: All, 1: Team, 2: Whisper
}

[MessagePackObject]
public class GameEventMessage : NetworkMessage
{
    [Key(2)]
    public int EventType { get; set; }

    [Key(3)]
    public Dictionary<string, object> EventData { get; set; }
}
```

### 네트워크 직렬화 유틸리티

```csharp
using System;
using System.IO;
using MessagePack;
using UnityEngine;

/// <summary>
/// 네트워크 메시지 직렬화 유틸리티
/// </summary>
public static class NetworkSerializer
{
    /// <summary>
    /// 메시지를 바이트 배열로 직렬화 (길이 프리픽스 포함)
    /// </summary>
    public static byte[] Serialize<T>(T message) where T : NetworkMessage
    {
        // MessagePack 직렬화
        byte[] payload = MessagePackSerializer.Serialize(message);

        // 길이 프리픽스 추가 (4 bytes)
        byte[] result = new byte[4 + payload.Length];
        BitConverter.GetBytes(payload.Length).CopyTo(result, 0);
        payload.CopyTo(result, 4);

        return result;
    }

    /// <summary>
    /// 스트림에서 메시지 역직렬화
    /// </summary>
    public static T Deserialize<T>(byte[] data) where T : NetworkMessage
    {
        // 길이 프리픽스 건너뛰기
        var segment = new ArraySegment<byte>(data, 4, data.Length - 4);
        return MessagePackSerializer.Deserialize<T>(segment);
    }

    /// <summary>
    /// Union 타입으로 역직렬화 (다형성)
    /// </summary>
    public static NetworkMessage DeserializeMessage(byte[] data)
    {
        var segment = new ArraySegment<byte>(data, 4, data.Length - 4);
        return MessagePackSerializer.Deserialize<NetworkMessage>(segment);
    }

    /// <summary>
    /// 압축된 직렬화 (LZ4)
    /// </summary>
    public static byte[] SerializeCompressed<T>(T message)
    {
        var options = MessagePackSerializerOptions.Standard
            .WithCompression(MessagePackCompression.Lz4BlockArray);

        return MessagePackSerializer.Serialize(message, options);
    }

    /// <summary>
    /// 압축 해제 및 역직렬화
    /// </summary>
    public static T DeserializeCompressed<T>(byte[] data)
    {
        var options = MessagePackSerializerOptions.Standard
            .WithCompression(MessagePackCompression.Lz4BlockArray);

        return MessagePackSerializer.Deserialize<T>(data, options);
    }
}

/// <summary>
/// 사용 예시
/// </summary>
public class NetworkSerializerExample : MonoBehaviour
{
    private void Start()
    {
        // 로그인 요청 생성 및 직렬화
        var loginRequest = new LoginRequest
        {
            Username = "player1",
            Password = "secret",
            DeviceId = SystemInfo.deviceUniqueIdentifier
        };

        byte[] data = NetworkSerializer.Serialize(loginRequest);
        Debug.Log($"직렬화 크기: {data.Length} bytes");

        // 역직렬화 (다형성)
        NetworkMessage message = NetworkSerializer.DeserializeMessage(data);

        if (message is LoginRequest login)
        {
            Debug.Log($"로그인 요청: {login.Username}");
        }

        // 압축 테스트
        var playerState = new PlayerStateMessage
        {
            PlayerId = "player123",
            Position = transform.position,
            RotationY = transform.eulerAngles.y,
            AnimationState = 1,
            Health = 100f
        };

        byte[] compressed = NetworkSerializer.SerializeCompressed(playerState);
        byte[] uncompressed = MessagePackSerializer.Serialize(playerState);

        Debug.Log($"압축 전: {uncompressed.Length}, 압축 후: {compressed.Length}");
    }
}
```

---

## 고급 기능

### 커스텀 Formatter

```csharp
using System;
using System.Buffers;
using MessagePack;
using MessagePack.Formatters;
using UnityEngine;

/// <summary>
/// DateTime을 Unix timestamp로 직렬화하는 Formatter
/// </summary>
public class UnixTimestampDateTimeFormatter : IMessagePackFormatter<DateTime>
{
    public static readonly UnixTimestampDateTimeFormatter Instance = new();

    public void Serialize(
        ref MessagePackWriter writer,
        DateTime value,
        MessagePackSerializerOptions options)
    {
        long timestamp = new DateTimeOffset(value).ToUnixTimeSeconds();
        writer.Write(timestamp);
    }

    public DateTime Deserialize(
        ref MessagePackReader reader,
        MessagePackSerializerOptions options)
    {
        long timestamp = reader.ReadInt64();
        return DateTimeOffset.FromUnixTimeSeconds(timestamp).DateTime;
    }
}

/// <summary>
/// Color32를 4바이트로 직렬화하는 Formatter
/// </summary>
public class CompactColor32Formatter : IMessagePackFormatter<Color32>
{
    public static readonly CompactColor32Formatter Instance = new();

    public void Serialize(
        ref MessagePackWriter writer,
        Color32 value,
        MessagePackSerializerOptions options)
    {
        // RGBA를 단일 uint로 패킹
        uint packed = (uint)value.r << 24 | (uint)value.g << 16 |
                     (uint)value.b << 8 | value.a;
        writer.Write(packed);
    }

    public Color32 Deserialize(
        ref MessagePackReader reader,
        MessagePackSerializerOptions options)
    {
        uint packed = reader.ReadUInt32();
        return new Color32(
            (byte)(packed >> 24),
            (byte)(packed >> 16),
            (byte)(packed >> 8),
            (byte)packed);
    }
}

/// <summary>
/// 커스텀 Resolver
/// </summary>
public class GameResolver : IFormatterResolver
{
    public static readonly GameResolver Instance = new();

    public IMessagePackFormatter<T> GetFormatter<T>()
    {
        return FormatterCache<T>.Formatter;
    }

    private static class FormatterCache<T>
    {
        public static readonly IMessagePackFormatter<T> Formatter;

        static FormatterCache()
        {
            Formatter = (IMessagePackFormatter<T>)GetFormatter(typeof(T));
        }

        private static object GetFormatter(Type type)
        {
            if (type == typeof(DateTime))
                return UnixTimestampDateTimeFormatter.Instance;

            if (type == typeof(Color32))
                return CompactColor32Formatter.Instance;

            return null;
        }
    }
}

/// <summary>
/// 커스텀 Resolver 등록
/// </summary>
public static class CustomMessagePackConfig
{
    public static void Initialize()
    {
        var resolver = CompositeResolver.Create(
            GameResolver.Instance,
            MessagePack.Unity.UnityResolver.Instance,
            StandardResolver.Instance
        );

        MessagePackSerializer.DefaultOptions =
            MessagePackSerializerOptions.Standard.WithResolver(resolver);
    }
}
```

### 스키마 버전 관리

```csharp
using System;
using MessagePack;
using UnityEngine;

/// <summary>
/// 버전이 있는 세이브 데이터
/// </summary>
[MessagePackObject]
public class VersionedSaveData
{
    [Key(0)]
    public int Version { get; set; } = CurrentVersion;

    [Key(1)]
    public PlayerSaveData Player { get; set; }

    [Key(2)]
    public GameProgressData Progress { get; set; }

    [Key(3)]
    public SettingsSaveData Settings { get; set; }

    public const int CurrentVersion = 3;
}

[MessagePackObject]
public class PlayerSaveData
{
    [Key(0)]
    public string PlayerId { get; set; }

    [Key(1)]
    public string Nickname { get; set; }

    [Key(2)]
    public int Level { get; set; }

    [Key(3)]
    public long Experience { get; set; }

    // v2에서 추가된 필드
    [Key(4)]
    public int[] EquippedItems { get; set; }

    // v3에서 추가된 필드
    [Key(5)]
    public DateTime LastPlayTime { get; set; }
}

[MessagePackObject]
public class GameProgressData
{
    [Key(0)]
    public int CurrentStage { get; set; }

    [Key(1)]
    public bool[] CompletedLevels { get; set; }

    [Key(2)]
    public int[] HighScores { get; set; }
}

[MessagePackObject]
public class SettingsSaveData
{
    [Key(0)]
    public float MusicVolume { get; set; }

    [Key(1)]
    public float SfxVolume { get; set; }

    [Key(2)]
    public string Language { get; set; }
}

/// <summary>
/// 버전 마이그레이션이 포함된 세이브 시스템
/// </summary>
public class VersionedSaveSystem : MonoBehaviour
{
    private const string SaveFileName = "save.dat";

    /// <summary>
    /// 저장
    /// </summary>
    public void Save(VersionedSaveData data)
    {
        data.Version = VersionedSaveData.CurrentVersion;
        byte[] bytes = MessagePackSerializer.Serialize(data);

        string path = System.IO.Path.Combine(
            Application.persistentDataPath,
            SaveFileName);

        System.IO.File.WriteAllBytes(path, bytes);
        Debug.Log($"저장 완료: {bytes.Length} bytes");
    }

    /// <summary>
    /// 로드 (버전 마이그레이션 포함)
    /// </summary>
    public VersionedSaveData Load()
    {
        string path = System.IO.Path.Combine(
            Application.persistentDataPath,
            SaveFileName);

        if (!System.IO.File.Exists(path))
        {
            return CreateNewSaveData();
        }

        byte[] bytes = System.IO.File.ReadAllBytes(path);
        var data = MessagePackSerializer.Deserialize<VersionedSaveData>(bytes);

        // 버전 마이그레이션
        if (data.Version < VersionedSaveData.CurrentVersion)
        {
            data = Migrate(data);
        }

        return data;
    }

    /// <summary>
    /// 버전 마이그레이션
    /// </summary>
    private VersionedSaveData Migrate(VersionedSaveData data)
    {
        // v1 → v2: EquippedItems 추가
        if (data.Version < 2)
        {
            data.Player.EquippedItems = new int[0];
            Debug.Log("세이브 데이터 마이그레이션: v1 → v2");
        }

        // v2 → v3: LastPlayTime 추가
        if (data.Version < 3)
        {
            data.Player.LastPlayTime = DateTime.UtcNow;
            Debug.Log("세이브 데이터 마이그레이션: v2 → v3");
        }

        data.Version = VersionedSaveData.CurrentVersion;

        // 마이그레이션 후 저장
        Save(data);

        return data;
    }

    private VersionedSaveData CreateNewSaveData()
    {
        return new VersionedSaveData
        {
            Player = new PlayerSaveData
            {
                PlayerId = Guid.NewGuid().ToString(),
                Nickname = "Player",
                Level = 1,
                Experience = 0,
                EquippedItems = new int[0],
                LastPlayTime = DateTime.UtcNow
            },
            Progress = new GameProgressData
            {
                CurrentStage = 1,
                CompletedLevels = new bool[100],
                HighScores = new int[100]
            },
            Settings = new SettingsSaveData
            {
                MusicVolume = 1f,
                SfxVolume = 1f,
                Language = "en"
            }
        };
    }
}
```

---

## 성능 최적화

### 메모리 효율적인 직렬화

```csharp
using System;
using System.Buffers;
using MessagePack;
using UnityEngine;

/// <summary>
/// 메모리 효율적인 직렬화
/// </summary>
public static class EfficientSerializer
{
    // 버퍼 풀 사용
    private static readonly ArrayPool<byte> BufferPool = ArrayPool<byte>.Shared;

    /// <summary>
    /// 풀링된 버퍼로 직렬화
    /// </summary>
    public static ArraySegment<byte> SerializePooled<T>(T value)
    {
        // 예상 크기로 버퍼 대여
        var buffer = BufferPool.Rent(1024);

        try
        {
            var writer = new ArrayBufferWriter<byte>();
            MessagePackSerializer.Serialize(writer, value);

            var result = writer.WrittenSpan;

            if (result.Length > buffer.Length)
            {
                BufferPool.Return(buffer);
                buffer = BufferPool.Rent(result.Length);
            }

            result.CopyTo(buffer);
            return new ArraySegment<byte>(buffer, 0, result.Length);
        }
        catch
        {
            BufferPool.Return(buffer);
            throw;
        }
    }

    /// <summary>
    /// 버퍼 반환
    /// </summary>
    public static void ReturnBuffer(ArraySegment<byte> buffer)
    {
        if (buffer.Array != null)
        {
            BufferPool.Return(buffer.Array);
        }
    }

    /// <summary>
    /// ReadOnlyMemory로 역직렬화
    /// </summary>
    public static T DeserializeFromMemory<T>(ReadOnlyMemory<byte> memory)
    {
        return MessagePackSerializer.Deserialize<T>(memory);
    }
}

/// <summary>
/// 객체 풀을 활용한 직렬화
/// </summary>
public class PooledMessageHandler<T> where T : class, new()
{
    private readonly System.Collections.Concurrent.ConcurrentBag<T> _pool = new();

    /// <summary>
    /// 풀에서 객체 가져오기
    /// </summary>
    public T Rent()
    {
        return _pool.TryTake(out var item) ? item : new T();
    }

    /// <summary>
    /// 풀에 객체 반환
    /// </summary>
    public void Return(T item)
    {
        _pool.Add(item);
    }

    /// <summary>
    /// 역직렬화 후 풀링된 객체 반환
    /// </summary>
    public T DeserializeFromPool(byte[] data)
    {
        // 기존 구현에서는 새 객체가 생성됨
        // 완전한 제로 할당을 위해서는 MemoryPack 사용 권장
        return MessagePackSerializer.Deserialize<T>(data);
    }
}
```

### 배치 직렬화

```csharp
using System.Collections.Generic;
using System.IO;
using MessagePack;
using UnityEngine;

/// <summary>
/// 다중 메시지 배치 처리
/// </summary>
public static class BatchSerializer
{
    /// <summary>
    /// 여러 메시지를 하나의 버퍼에 직렬화
    /// </summary>
    public static byte[] SerializeBatch<T>(IEnumerable<T> messages)
    {
        using var stream = new MemoryStream();

        foreach (var message in messages)
        {
            var data = MessagePackSerializer.Serialize(message);
            // 길이 프리픽스 (4 bytes)
            stream.Write(System.BitConverter.GetBytes(data.Length), 0, 4);
            stream.Write(data, 0, data.Length);
        }

        return stream.ToArray();
    }

    /// <summary>
    /// 배치에서 메시지들 역직렬화
    /// </summary>
    public static IEnumerable<T> DeserializeBatch<T>(byte[] batch)
    {
        int offset = 0;

        while (offset < batch.Length)
        {
            // 길이 읽기
            int length = System.BitConverter.ToInt32(batch, offset);
            offset += 4;

            // 메시지 역직렬화
            var segment = new ArraySegment<byte>(batch, offset, length);
            yield return MessagePackSerializer.Deserialize<T>(segment);

            offset += length;
        }
    }
}

/// <summary>
/// 배치 직렬화 사용 예시
/// </summary>
public class BatchSerializerExample : MonoBehaviour
{
    private void Start()
    {
        // 여러 플레이어 상태를 배치로 직렬화
        var states = new List<PlayerStateMessage>
        {
            new PlayerStateMessage { PlayerId = "p1", Position = Vector3.zero },
            new PlayerStateMessage { PlayerId = "p2", Position = Vector3.one },
            new PlayerStateMessage { PlayerId = "p3", Position = Vector3.up }
        };

        byte[] batch = BatchSerializer.SerializeBatch(states);
        Debug.Log($"배치 크기: {batch.Length} bytes ({states.Count}개 메시지)");

        // 역직렬화
        foreach (var state in BatchSerializer.DeserializeBatch<PlayerStateMessage>(batch))
        {
            Debug.Log($"Player: {state.PlayerId} at {state.Position}");
        }
    }
}
```

---

## IL2CPP 및 AOT 지원

### Source Generator 사용

```csharp
using MessagePack;

// AOT 환경에서 Source Generator 사용
// .csproj에 다음 추가 필요:
// <ItemGroup>
//   <PackageReference Include="MessagePack.Generator" Version="..." PrivateAssets="all" />
// </ItemGroup>

/// <summary>
/// Source Generator가 코드를 생성할 타입들
/// </summary>
[MessagePackObject]
public partial class GeneratedPlayerData
{
    [Key(0)]
    public string Id { get; set; }

    [Key(1)]
    public int Level { get; set; }
}

/// <summary>
/// 생성된 Resolver 등록
/// </summary>
public static class AotMessagePackConfig
{
    [UnityEngine.RuntimeInitializeOnLoadMethod(
        UnityEngine.RuntimeInitializeLoadType.BeforeSceneLoad)]
    public static void Initialize()
    {
        // Source Generator가 생성한 Resolver 사용
        // GeneratedResolver.Instance가 자동 생성됨

        var resolver = CompositeResolver.Create(
            // GeneratedResolver.Instance,  // Source Generator 생성
            MessagePack.Unity.UnityResolver.Instance,
            StandardResolver.Instance
        );

        MessagePackSerializer.DefaultOptions =
            MessagePackSerializerOptions.Standard.WithResolver(resolver);
    }
}
```

### link.xml 설정

```xml
<!-- Assets/link.xml -->
<linker>
    <!-- MessagePack 코어 -->
    <assembly fullname="MessagePack" preserve="all"/>
    <assembly fullname="MessagePack.Annotations" preserve="all"/>

    <!-- Unity 지원 -->
    <assembly fullname="MessagePack.Unity" preserve="all"/>

    <!-- 생성된 코드 (프로젝트에 따라 수정) -->
    <assembly fullname="Assembly-CSharp" preserve="all">
        <type fullname="*" preserve="all"/>
    </assembly>

    <!-- System.Buffers -->
    <assembly fullname="System.Buffers" preserve="all"/>
    <assembly fullname="System.Memory" preserve="all"/>
</linker>
```

---

## 실용 예제: 멀티플레이어 상태 동기화

```csharp
using System;
using System.Collections.Generic;
using MessagePack;
using UnityEngine;

/// <summary>
/// 월드 상태 스냅샷
/// </summary>
[MessagePackObject]
public class WorldSnapshot
{
    [Key(0)]
    public long ServerTick { get; set; }

    [Key(1)]
    public long Timestamp { get; set; }

    [Key(2)]
    public EntityState[] Entities { get; set; }

    [Key(3)]
    public GameEventData[] Events { get; set; }
}

[MessagePackObject]
public class EntityState
{
    [Key(0)]
    public int EntityId { get; set; }

    [Key(1)]
    public byte EntityType { get; set; }

    [Key(2)]
    public Vector3 Position { get; set; }

    [Key(3)]
    public Quaternion Rotation { get; set; }

    [Key(4)]
    public Vector3 Velocity { get; set; }

    [Key(5)]
    public byte[] CustomData { get; set; }
}

[MessagePackObject]
public class GameEventData
{
    [Key(0)]
    public int EventType { get; set; }

    [Key(1)]
    public int SourceEntityId { get; set; }

    [Key(2)]
    public int TargetEntityId { get; set; }

    [Key(3)]
    public byte[] Parameters { get; set; }
}

/// <summary>
/// 상태 동기화 관리자
/// </summary>
public class StateSynchronizer : MonoBehaviour
{
    // 엔티티 캐시
    private Dictionary<int, EntityState> _entityCache = new();

    // 스냅샷 히스토리 (보간용)
    private Queue<WorldSnapshot> _snapshotHistory = new();
    private const int MaxHistorySize = 30;

    /// <summary>
    /// 서버로부터 스냅샷 수신
    /// </summary>
    public void ReceiveSnapshot(byte[] data)
    {
        // LZ4 압축 해제 및 역직렬화
        var options = MessagePackSerializerOptions.Standard
            .WithCompression(MessagePackCompression.Lz4BlockArray);

        var snapshot = MessagePackSerializer.Deserialize<WorldSnapshot>(data, options);

        // 히스토리에 추가
        _snapshotHistory.Enqueue(snapshot);
        while (_snapshotHistory.Count > MaxHistorySize)
        {
            _snapshotHistory.Dequeue();
        }

        // 엔티티 상태 적용
        ApplySnapshot(snapshot);
    }

    /// <summary>
    /// 스냅샷 적용
    /// </summary>
    private void ApplySnapshot(WorldSnapshot snapshot)
    {
        foreach (var entity in snapshot.Entities)
        {
            _entityCache[entity.EntityId] = entity;

            // 실제 게임 오브젝트에 적용
            ApplyEntityState(entity);
        }

        // 이벤트 처리
        foreach (var evt in snapshot.Events)
        {
            ProcessGameEvent(evt);
        }
    }

    private void ApplyEntityState(EntityState state)
    {
        // 엔티티 찾아서 상태 적용
        // (실제 구현에서는 엔티티 관리자 사용)
    }

    private void ProcessGameEvent(GameEventData evt)
    {
        // 게임 이벤트 처리
        Debug.Log($"Event: {evt.EventType} from {evt.SourceEntityId}");
    }

    /// <summary>
    /// 현재 클라이언트 상태를 서버로 전송
    /// </summary>
    public byte[] CreateClientState(int[] controlledEntityIds)
    {
        var entities = new List<EntityState>();

        foreach (var id in controlledEntityIds)
        {
            if (_entityCache.TryGetValue(id, out var state))
            {
                entities.Add(state);
            }
        }

        var snapshot = new WorldSnapshot
        {
            Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
            Entities = entities.ToArray()
        };

        // LZ4 압축
        var options = MessagePackSerializerOptions.Standard
            .WithCompression(MessagePackCompression.Lz4BlockArray);

        return MessagePackSerializer.Serialize(snapshot, options);
    }
}
```

---

## 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────┐
│                  MessagePack 베스트 프랙티스                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 키 설계                                                      │
│     ├── 정수 키 사용 (문자열보다 효율적)                          │
│     ├── 키 번호는 순차적으로 할당                                 │
│     └── 새 필드는 끝에 추가 (호환성)                              │
│                                                                  │
│  2. 성능 최적화                                                   │
│     ├── LZ4 압축 활용                                            │
│     ├── 버퍼 풀링으로 GC 최소화                                   │
│     └── 배치 직렬화로 오버헤드 감소                               │
│                                                                  │
│  3. 버전 관리                                                    │
│     ├── 버전 필드 포함                                           │
│     ├── 새 필드는 nullable 또는 기본값                            │
│     └── 마이그레이션 로직 구현                                   │
│                                                                  │
│  4. Unity 통합                                                   │
│     ├── UnityResolver 등록                                       │
│     ├── RuntimeInitializeOnLoadMethod 활용                       │
│     └── link.xml로 AOT 지원                                      │
│                                                                  │
│  5. 네트워크 사용                                                 │
│     ├── 길이 프리픽스 추가                                        │
│     ├── Union 타입으로 다형성                                     │
│     └── 대역폭 절약을 위한 압축                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [MessagePack-CSharp GitHub](https://github.com/neuecc/MessagePack-CSharp)
- [MessagePack Specification](https://msgpack.org/index.html)
- [Unity용 MessagePack 가이드](https://github.com/neuecc/MessagePack-CSharp#unity)
- [LZ4 Compression](https://github.com/neuecc/MessagePack-CSharp#lz4-compression)

---

## 다음 단계

- [Section 31: MemoryPack](./31-memorypack.md) - 제로 카피 직렬화
- [Section 32: FlatBuffers](./32-flatbuffers.md) - 제로 파싱 직렬화
