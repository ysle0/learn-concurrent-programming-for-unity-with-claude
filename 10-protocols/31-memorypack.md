# Section 31: MemoryPack

## 개요

MemoryPack은 C# 전용 제로 인코딩 고성능 바이너리 직렬화 라이브러리입니다. Source Generator를 활용하여 런타임 리플렉션 없이 최고의 성능을 제공하며, Unity IL2CPP와 완벽하게 호환됩니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    MemoryPack Architecture                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   직렬화 성능 비교 (낮을수록 좋음):                               │
│                                                                  │
│   MemoryPack    ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░  1.0x           │
│   MessagePack   ████████░░░░░░░░░░░░░░░░░░░░░░░░  2.5x           │
│   System.Json   ████████████████░░░░░░░░░░░░░░░░  6.0x           │
│   Newtonsoft    ████████████████████████░░░░░░░░  10.0x          │
│                                                                  │
│   주요 특징:                                                     │
│   ┌────────────────────────────────────────────────────────┐    │
│   │ • Zero-Encoding: 메모리 레이아웃 그대로 직렬화           │    │
│   │ • Source Generator: 컴파일 타임 코드 생성               │    │
│   │ • Zero-Allocation: 버퍼 재사용으로 GC 없음              │    │
│   │ • Unity 최적화: IL2CPP, Burst 호환                      │    │
│   │ • 다형성 지원: Union 타입으로 상속 처리                  │    │
│   └────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 설치

```
Unity Package Manager:
1. Window > Package Manager > + > Add package from git URL
2. https://github.com/Cysharp/MemoryPack.git?path=src/MemoryPack.Unity/Assets/Plugins/MemoryPack

또는 NuGet:
- MemoryPack (Core)
- MemoryPack.Generator (Source Generator)

요구사항:
- Unity 2022.3+ (Source Generator 지원)
- .NET Standard 2.1 또는 .NET 6+
```

---

## 기본 사용법

### 기본 직렬화

```csharp
using MemoryPack;
using UnityEngine;

/// <summary>
/// MemoryPack 기본 예제
/// </summary>
public class MemoryPackBasicExample : MonoBehaviour
{
    // [MemoryPackable]로 직렬화 가능 표시
    // partial 키워드 필수 (Source Generator가 코드 생성)
    [MemoryPackable]
    public partial class PlayerData
    {
        public string PlayerName { get; set; }
        public int Level { get; set; }
        public float Health { get; set; }
        public long Experience { get; set; }
        public int[] Inventory { get; set; }

        // [MemoryPackIgnore]로 직렬화 제외
        [MemoryPackIgnore]
        public string TempData { get; set; }
    }

    private void Start()
    {
        var player = new PlayerData
        {
            PlayerName = "Hero",
            Level = 42,
            Health = 100.5f,
            Experience = 1234567890L,
            Inventory = new[] { 1, 2, 3, 4, 5 }
        };

        // 직렬화
        byte[] bytes = MemoryPackSerializer.Serialize(player);
        Debug.Log($"직렬화 크기: {bytes.Length} bytes");

        // 역직렬화
        var loaded = MemoryPackSerializer.Deserialize<PlayerData>(bytes);
        Debug.Log($"플레이어: {loaded.PlayerName}, Lv.{loaded.Level}");
    }
}
```

### 구조체 직렬화

```csharp
using MemoryPack;
using UnityEngine;

/// <summary>
/// 구조체 직렬화 (값 타입)
/// </summary>
[MemoryPackable]
public partial struct Vector3Data
{
    public float X;
    public float Y;
    public float Z;

    public Vector3Data(float x, float y, float z)
    {
        X = x;
        Y = y;
        Z = z;
    }

    public Vector3 ToVector3() => new Vector3(X, Y, Z);
    public static Vector3Data FromVector3(Vector3 v) => new Vector3Data(v.x, v.y, v.z);
}

[MemoryPackable]
public partial struct TransformSnapshot
{
    public Vector3Data Position;
    public Vector3Data Rotation;
    public Vector3Data Scale;

    public static TransformSnapshot Capture(Transform t)
    {
        return new TransformSnapshot
        {
            Position = Vector3Data.FromVector3(t.position),
            Rotation = Vector3Data.FromVector3(t.eulerAngles),
            Scale = Vector3Data.FromVector3(t.localScale)
        };
    }

    public void Apply(Transform t)
    {
        t.position = Position.ToVector3();
        t.eulerAngles = Rotation.ToVector3();
        t.localScale = Scale.ToVector3();
    }
}
```

---

## 생성자 및 직렬화 순서

### 생성자 지정

```csharp
using MemoryPack;

/// <summary>
/// 직렬화 생성자 지정
/// </summary>
[MemoryPackable]
public partial class ImmutableData
{
    public string Id { get; }
    public string Name { get; }
    public int Value { get; }

    // [MemoryPackConstructor]로 역직렬화에 사용할 생성자 지정
    [MemoryPackConstructor]
    public ImmutableData(string id, string name, int value)
    {
        Id = id;
        Name = name;
        Value = value;
    }
}
```

### 직렬화 순서 지정

```csharp
using MemoryPack;

/// <summary>
/// 명시적 직렬화 순서
/// </summary>
[MemoryPackable]
public partial class OrderedData
{
    // [MemoryPackOrder]로 순서 지정
    [MemoryPackOrder(0)]
    public int Id { get; set; }

    [MemoryPackOrder(1)]
    public string Name { get; set; }

    [MemoryPackOrder(2)]
    public float Value { get; set; }

    // 순서 없는 필드는 마지막에 배치됨
    public string Description { get; set; }
}
```

---

## Union 타입 (다형성)

```csharp
using MemoryPack;
using UnityEngine;

/// <summary>
/// Union 타입으로 다형성 처리
/// </summary>
[MemoryPackable]
[MemoryPackUnion(0, typeof(MoveAction))]
[MemoryPackUnion(1, typeof(AttackAction))]
[MemoryPackUnion(2, typeof(SkillAction))]
[MemoryPackUnion(3, typeof(ItemAction))]
public partial interface IGameAction
{
    int ActionId { get; }
    long Timestamp { get; }
}

[MemoryPackable]
public partial class MoveAction : IGameAction
{
    public int ActionId { get; set; }
    public long Timestamp { get; set; }
    public Vector3Data TargetPosition { get; set; }
    public float Speed { get; set; }
}

[MemoryPackable]
public partial class AttackAction : IGameAction
{
    public int ActionId { get; set; }
    public long Timestamp { get; set; }
    public int TargetEntityId { get; set; }
    public int WeaponId { get; set; }
    public float Damage { get; set; }
}

[MemoryPackable]
public partial class SkillAction : IGameAction
{
    public int ActionId { get; set; }
    public long Timestamp { get; set; }
    public int SkillId { get; set; }
    public Vector3Data TargetPosition { get; set; }
    public int[] AffectedEntities { get; set; }
}

[MemoryPackable]
public partial class ItemAction : IGameAction
{
    public int ActionId { get; set; }
    public long Timestamp { get; set; }
    public int ItemId { get; set; }
    public int TargetSlot { get; set; }
}

/// <summary>
/// Union 사용 예시
/// </summary>
public class UnionExample : MonoBehaviour
{
    private void Start()
    {
        // 다양한 액션 생성
        var actions = new IGameAction[]
        {
            new MoveAction
            {
                ActionId = 1,
                Timestamp = System.DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
                TargetPosition = new Vector3Data(10, 0, 20),
                Speed = 5f
            },
            new AttackAction
            {
                ActionId = 2,
                Timestamp = System.DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
                TargetEntityId = 100,
                WeaponId = 1,
                Damage = 50f
            }
        };

        // 직렬화 (Union 타입으로)
        byte[] bytes = MemoryPackSerializer.Serialize(actions);
        Debug.Log($"직렬화 크기: {bytes.Length} bytes");

        // 역직렬화
        var loadedActions = MemoryPackSerializer.Deserialize<IGameAction[]>(bytes);

        foreach (var action in loadedActions)
        {
            switch (action)
            {
                case MoveAction move:
                    Debug.Log($"이동: {move.TargetPosition.X}, {move.TargetPosition.Z}");
                    break;
                case AttackAction attack:
                    Debug.Log($"공격: 대상 {attack.TargetEntityId}, 데미지 {attack.Damage}");
                    break;
            }
        }
    }
}
```

---

## 제로 할당 직렬화

### 버퍼 재사용

```csharp
using System;
using System.Buffers;
using MemoryPack;
using UnityEngine;

/// <summary>
/// 제로 할당 직렬화/역직렬화
/// </summary>
public class ZeroAllocationSerializer : MonoBehaviour
{
    // 재사용 가능한 버퍼
    private readonly ArrayBufferWriter<byte> _writer = new(1024);

    [MemoryPackable]
    public partial class GameState
    {
        public long Tick { get; set; }
        public EntityState[] Entities { get; set; }
    }

    [MemoryPackable]
    public partial class EntityState
    {
        public int Id { get; set; }
        public Vector3Data Position { get; set; }
        public float Health { get; set; }
    }

    /// <summary>
    /// 버퍼 재사용 직렬화
    /// </summary>
    public ReadOnlySpan<byte> SerializeWithReuse<T>(T value)
    {
        _writer.Clear();
        MemoryPackSerializer.Serialize(_writer, value);
        return _writer.WrittenSpan;
    }

    /// <summary>
    /// ArrayPool 활용
    /// </summary>
    public byte[] SerializeWithPool<T>(T value)
    {
        var writer = new ArrayBufferWriter<byte>();
        MemoryPackSerializer.Serialize(writer, value);

        // ArrayPool에서 버퍼 대여
        var buffer = ArrayPool<byte>.Shared.Rent(writer.WrittenCount);
        writer.WrittenSpan.CopyTo(buffer);

        // 반환 시: ArrayPool<byte>.Shared.Return(buffer);
        return buffer;
    }

    /// <summary>
    /// 스택 할당 활용 (작은 데이터)
    /// </summary>
    public void SerializeSmallData<T>(T value, Action<ReadOnlySpan<byte>> callback)
    {
        // 스택에 버퍼 할당 (GC 없음)
        Span<byte> buffer = stackalloc byte[256];

        var bytesWritten = MemoryPackSerializer.Serialize(buffer, value);

        callback(buffer.Slice(0, bytesWritten));
    }

    /// <summary>
    /// 기존 객체에 역직렬화 (ref 활용)
    /// </summary>
    public void DeserializeInto<T>(ReadOnlySpan<byte> data, ref T target)
    {
        // 새 객체 생성 없이 기존 객체에 덮어쓰기
        MemoryPackSerializer.Deserialize(data, ref target);
    }

    private void Update()
    {
        // 매 프레임 제로 할당 직렬화 예시
        var state = new GameState
        {
            Tick = Time.frameCount,
            Entities = new[]
            {
                new EntityState { Id = 1, Position = Vector3Data.FromVector3(transform.position) }
            }
        };

        // 버퍼 재사용 (GC 없음)
        var bytes = SerializeWithReuse(state);

        // 네트워크 전송 등
        SendToNetwork(bytes);
    }

    private void SendToNetwork(ReadOnlySpan<byte> data)
    {
        // 네트워크 전송 로직
    }
}
```

### 스트리밍 직렬화

```csharp
using System.IO;
using System.IO.Pipelines;
using System.Threading.Tasks;
using MemoryPack;
using UnityEngine;

/// <summary>
/// 스트리밍 직렬화
/// </summary>
public class StreamingSerializer
{
    /// <summary>
    /// Stream에 직렬화
    /// </summary>
    public async Task SerializeToStreamAsync<T>(Stream stream, T value)
    {
        await MemoryPackSerializer.SerializeAsync(stream, value);
    }

    /// <summary>
    /// Stream에서 역직렬화
    /// </summary>
    public async Task<T> DeserializeFromStreamAsync<T>(Stream stream)
    {
        return await MemoryPackSerializer.DeserializeAsync<T>(stream);
    }

    /// <summary>
    /// 파일 저장
    /// </summary>
    public async Task SaveToFileAsync<T>(string path, T value)
    {
        using var stream = new FileStream(
            path,
            FileMode.Create,
            FileAccess.Write,
            FileShare.None,
            bufferSize: 4096,
            useAsync: true);

        await MemoryPackSerializer.SerializeAsync(stream, value);
    }

    /// <summary>
    /// 파일 로드
    /// </summary>
    public async Task<T> LoadFromFileAsync<T>(string path)
    {
        using var stream = new FileStream(
            path,
            FileMode.Open,
            FileAccess.Read,
            FileShare.Read,
            bufferSize: 4096,
            useAsync: true);

        return await MemoryPackSerializer.DeserializeAsync<T>(stream);
    }
}
```

---

## 네트워크 프로토콜

### 네트워크 메시지 시스템

```csharp
using System;
using System.Collections.Generic;
using MemoryPack;
using UnityEngine;

/// <summary>
/// 네트워크 메시지 기본 인터페이스
/// </summary>
[MemoryPackable]
[MemoryPackUnion(0, typeof(PingMessage))]
[MemoryPackUnion(1, typeof(PongMessage))]
[MemoryPackUnion(2, typeof(LoginRequest))]
[MemoryPackUnion(3, typeof(LoginResponse))]
[MemoryPackUnion(4, typeof(PlayerStateSync))]
[MemoryPackUnion(5, typeof(GameEventMessage))]
public partial interface INetworkMessage
{
    ushort MessageType { get; }
    long Timestamp { get; set; }
}

[MemoryPackable]
public partial class PingMessage : INetworkMessage
{
    public ushort MessageType => 0;
    public long Timestamp { get; set; }
    public long ClientTime { get; set; }
}

[MemoryPackable]
public partial class PongMessage : INetworkMessage
{
    public ushort MessageType => 1;
    public long Timestamp { get; set; }
    public long ClientTime { get; set; }
    public long ServerTime { get; set; }
}

[MemoryPackable]
public partial class LoginRequest : INetworkMessage
{
    public ushort MessageType => 2;
    public long Timestamp { get; set; }
    public string Username { get; set; }
    public string Token { get; set; }
    public string DeviceId { get; set; }
}

[MemoryPackable]
public partial class LoginResponse : INetworkMessage
{
    public ushort MessageType => 3;
    public long Timestamp { get; set; }
    public bool Success { get; set; }
    public string PlayerId { get; set; }
    public string SessionToken { get; set; }
    public string ErrorCode { get; set; }
}

[MemoryPackable]
public partial class PlayerStateSync : INetworkMessage
{
    public ushort MessageType => 4;
    public long Timestamp { get; set; }
    public int SequenceNumber { get; set; }
    public PlayerNetState[] Players { get; set; }
}

[MemoryPackable]
public partial struct PlayerNetState
{
    public int PlayerId;
    public Vector3Data Position;
    public float RotationY;
    public byte AnimationState;
    public float Health;
    public ushort Flags;
}

[MemoryPackable]
public partial class GameEventMessage : INetworkMessage
{
    public ushort MessageType => 5;
    public long Timestamp { get; set; }
    public int EventType { get; set; }
    public int SourceId { get; set; }
    public int TargetId { get; set; }
    public byte[] EventData { get; set; }
}
```

### 네트워크 직렬화 서비스

```csharp
using System;
using System.Buffers;
using MemoryPack;
using UnityEngine;

/// <summary>
/// 네트워크 직렬화 서비스
/// </summary>
public class NetworkSerializer
{
    private readonly ArrayBufferWriter<byte> _writeBuffer = new(4096);
    private readonly byte[] _lengthBuffer = new byte[4];

    /// <summary>
    /// 메시지를 패킷으로 직렬화 (길이 프리픽스 포함)
    /// </summary>
    public ReadOnlyMemory<byte> SerializePacket<T>(T message) where T : INetworkMessage
    {
        _writeBuffer.Clear();

        // 길이 자리 확보 (4 bytes)
        var lengthSpan = _writeBuffer.GetSpan(4);
        _writeBuffer.Advance(4);

        // 메시지 직렬화
        var messageStart = _writeBuffer.WrittenCount;
        MemoryPackSerializer.Serialize(_writeBuffer, message);
        var messageLength = _writeBuffer.WrittenCount - messageStart;

        // 길이 기록
        BitConverter.TryWriteBytes(lengthSpan, messageLength);

        return _writeBuffer.WrittenMemory;
    }

    /// <summary>
    /// 패킷에서 메시지 역직렬화
    /// </summary>
    public INetworkMessage DeserializePacket(ReadOnlySpan<byte> packet)
    {
        // 길이 읽기
        var length = BitConverter.ToInt32(packet.Slice(0, 4));

        // 메시지 역직렬화
        var messageSpan = packet.Slice(4, length);
        return MemoryPackSerializer.Deserialize<INetworkMessage>(messageSpan);
    }

    /// <summary>
    /// 여러 메시지 배치 직렬화
    /// </summary>
    public ReadOnlyMemory<byte> SerializeBatch(INetworkMessage[] messages)
    {
        _writeBuffer.Clear();

        // 메시지 수 기록
        var countSpan = _writeBuffer.GetSpan(2);
        BitConverter.TryWriteBytes(countSpan, (ushort)messages.Length);
        _writeBuffer.Advance(2);

        // 각 메시지 직렬화
        foreach (var message in messages)
        {
            // 길이 자리 확보
            var lengthStart = _writeBuffer.WrittenCount;
            var lengthSpan = _writeBuffer.GetSpan(2);
            _writeBuffer.Advance(2);

            // 메시지 직렬화
            var messageStart = _writeBuffer.WrittenCount;
            MemoryPackSerializer.Serialize(_writeBuffer, message);
            var messageLength = _writeBuffer.WrittenCount - messageStart;

            // 길이 기록
            BitConverter.TryWriteBytes(
                _writeBuffer.WrittenSpan.Slice(lengthStart, 2),
                (ushort)messageLength);
        }

        return _writeBuffer.WrittenMemory;
    }

    /// <summary>
    /// 배치 역직렬화
    /// </summary>
    public INetworkMessage[] DeserializeBatch(ReadOnlySpan<byte> batch)
    {
        var count = BitConverter.ToUInt16(batch.Slice(0, 2));
        var messages = new INetworkMessage[count];

        var offset = 2;
        for (int i = 0; i < count; i++)
        {
            var length = BitConverter.ToUInt16(batch.Slice(offset, 2));
            offset += 2;

            messages[i] = MemoryPackSerializer.Deserialize<INetworkMessage>(
                batch.Slice(offset, length));
            offset += length;
        }

        return messages;
    }
}
```

---

## 게임 세이브 시스템

```csharp
using System;
using System.IO;
using System.Threading.Tasks;
using MemoryPack;
using UnityEngine;

/// <summary>
/// MemoryPack 기반 세이브 시스템
/// </summary>
[MemoryPackable]
public partial class SaveData
{
    public int Version { get; set; } = CurrentVersion;
    public long SaveTimestamp { get; set; }
    public PlayerSaveData Player { get; set; }
    public WorldSaveData World { get; set; }
    public SettingsSaveData Settings { get; set; }

    public const int CurrentVersion = 1;
}

[MemoryPackable]
public partial class PlayerSaveData
{
    public string PlayerId { get; set; }
    public string Nickname { get; set; }
    public int Level { get; set; }
    public long Experience { get; set; }
    public int Gold { get; set; }
    public InventorySlot[] Inventory { get; set; }
    public int[] EquippedItems { get; set; }
    public Dictionary<int, int> Skills { get; set; }
}

[MemoryPackable]
public partial struct InventorySlot
{
    public int ItemId;
    public int Quantity;
    public int Durability;
    public byte[] CustomData;
}

[MemoryPackable]
public partial class WorldSaveData
{
    public int CurrentStage { get; set; }
    public Vector3Data PlayerPosition { get; set; }
    public bool[] CompletedQuests { get; set; }
    public Dictionary<string, int> Flags { get; set; }
}

[MemoryPackable]
public partial class SettingsSaveData
{
    public float MasterVolume { get; set; }
    public float MusicVolume { get; set; }
    public float SfxVolume { get; set; }
    public bool Fullscreen { get; set; }
    public int QualityLevel { get; set; }
    public string Language { get; set; }
}

/// <summary>
/// 세이브 관리자
/// </summary>
public class SaveManager : MonoBehaviour
{
    private const string SaveFileName = "save.bin";
    private string SavePath => Path.Combine(Application.persistentDataPath, SaveFileName);

    /// <summary>
    /// 동기 저장
    /// </summary>
    public void Save(SaveData data)
    {
        data.SaveTimestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds();

        byte[] bytes = MemoryPackSerializer.Serialize(data);
        File.WriteAllBytes(SavePath, bytes);

        Debug.Log($"저장 완료: {bytes.Length} bytes");
    }

    /// <summary>
    /// 비동기 저장
    /// </summary>
    public async Task SaveAsync(SaveData data)
    {
        data.SaveTimestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds();

        await using var stream = new FileStream(
            SavePath,
            FileMode.Create,
            FileAccess.Write,
            FileShare.None,
            bufferSize: 4096,
            useAsync: true);

        await MemoryPackSerializer.SerializeAsync(stream, data);

        Debug.Log("비동기 저장 완료");
    }

    /// <summary>
    /// 동기 로드
    /// </summary>
    public SaveData Load()
    {
        if (!File.Exists(SavePath))
        {
            return CreateNewSave();
        }

        byte[] bytes = File.ReadAllBytes(SavePath);
        var data = MemoryPackSerializer.Deserialize<SaveData>(bytes);

        // 버전 마이그레이션
        if (data.Version < SaveData.CurrentVersion)
        {
            data = Migrate(data);
        }

        return data;
    }

    /// <summary>
    /// 비동기 로드
    /// </summary>
    public async Task<SaveData> LoadAsync()
    {
        if (!File.Exists(SavePath))
        {
            return CreateNewSave();
        }

        await using var stream = new FileStream(
            SavePath,
            FileMode.Open,
            FileAccess.Read,
            FileShare.Read,
            bufferSize: 4096,
            useAsync: true);

        var data = await MemoryPackSerializer.DeserializeAsync<SaveData>(stream);

        if (data.Version < SaveData.CurrentVersion)
        {
            data = Migrate(data);
        }

        return data;
    }

    /// <summary>
    /// 백업 저장
    /// </summary>
    public void CreateBackup()
    {
        if (File.Exists(SavePath))
        {
            string backupPath = SavePath + ".backup";
            File.Copy(SavePath, backupPath, overwrite: true);
        }
    }

    /// <summary>
    /// 버전 마이그레이션
    /// </summary>
    private SaveData Migrate(SaveData data)
    {
        // v0 → v1 마이그레이션 예시
        if (data.Version < 1)
        {
            // 새 필드 초기화
            data.Player.Skills ??= new Dictionary<int, int>();
            data.Version = 1;
        }

        return data;
    }

    private SaveData CreateNewSave()
    {
        return new SaveData
        {
            Player = new PlayerSaveData
            {
                PlayerId = Guid.NewGuid().ToString(),
                Nickname = "Player",
                Level = 1,
                Experience = 0,
                Gold = 100,
                Inventory = new InventorySlot[50],
                EquippedItems = new int[10],
                Skills = new Dictionary<int, int>()
            },
            World = new WorldSaveData
            {
                CurrentStage = 1,
                PlayerPosition = new Vector3Data(0, 0, 0),
                CompletedQuests = new bool[100],
                Flags = new Dictionary<string, int>()
            },
            Settings = new SettingsSaveData
            {
                MasterVolume = 1f,
                MusicVolume = 0.8f,
                SfxVolume = 1f,
                Fullscreen = true,
                QualityLevel = 3,
                Language = "en"
            }
        };
    }
}
```

---

## 압축

```csharp
using System.IO;
using System.IO.Compression;
using MemoryPack;
using UnityEngine;

/// <summary>
/// 압축 옵션이 포함된 직렬화
/// </summary>
public static class CompressedSerializer
{
    /// <summary>
    /// Brotli 압축 직렬화
    /// </summary>
    public static byte[] SerializeWithBrotli<T>(T value, CompressionLevel level = CompressionLevel.Optimal)
    {
        // 먼저 MemoryPack 직렬화
        byte[] data = MemoryPackSerializer.Serialize(value);

        // Brotli 압축
        using var output = new MemoryStream();
        using (var brotli = new BrotliStream(output, level))
        {
            brotli.Write(data, 0, data.Length);
        }

        return output.ToArray();
    }

    /// <summary>
    /// Brotli 압축 해제 및 역직렬화
    /// </summary>
    public static T DeserializeWithBrotli<T>(byte[] compressed)
    {
        using var input = new MemoryStream(compressed);
        using var brotli = new BrotliStream(input, CompressionMode.Decompress);
        using var output = new MemoryStream();

        brotli.CopyTo(output);

        return MemoryPackSerializer.Deserialize<T>(output.ToArray());
    }

    /// <summary>
    /// GZip 압축 직렬화
    /// </summary>
    public static byte[] SerializeWithGZip<T>(T value)
    {
        byte[] data = MemoryPackSerializer.Serialize(value);

        using var output = new MemoryStream();
        using (var gzip = new GZipStream(output, CompressionLevel.Optimal))
        {
            gzip.Write(data, 0, data.Length);
        }

        return output.ToArray();
    }

    /// <summary>
    /// GZip 압축 해제 및 역직렬화
    /// </summary>
    public static T DeserializeWithGZip<T>(byte[] compressed)
    {
        using var input = new MemoryStream(compressed);
        using var gzip = new GZipStream(input, CompressionMode.Decompress);
        using var output = new MemoryStream();

        gzip.CopyTo(output);

        return MemoryPackSerializer.Deserialize<T>(output.ToArray());
    }
}

/// <summary>
/// 압축 테스트
/// </summary>
public class CompressionTest : MonoBehaviour
{
    [MemoryPackable]
    public partial class LargeData
    {
        public string[] Strings { get; set; }
        public int[] Numbers { get; set; }
    }

    private void Start()
    {
        // 큰 테스트 데이터 생성
        var data = new LargeData
        {
            Strings = new string[1000],
            Numbers = new int[10000]
        };

        for (int i = 0; i < 1000; i++)
            data.Strings[i] = $"String_{i}";

        for (int i = 0; i < 10000; i++)
            data.Numbers[i] = i;

        // 크기 비교
        byte[] raw = MemoryPackSerializer.Serialize(data);
        byte[] brotli = CompressedSerializer.SerializeWithBrotli(data);
        byte[] gzip = CompressedSerializer.SerializeWithGZip(data);

        Debug.Log($"원본: {raw.Length} bytes");
        Debug.Log($"Brotli: {brotli.Length} bytes ({(float)brotli.Length / raw.Length:P0})");
        Debug.Log($"GZip: {gzip.Length} bytes ({(float)gzip.Length / raw.Length:P0})");
    }
}
```

---

## 커스텀 Formatter

```csharp
using System;
using MemoryPack;
using UnityEngine;

/// <summary>
/// Unity Color를 위한 커스텀 Formatter
/// </summary>
public class ColorFormatter : MemoryPackFormatter<Color>
{
    public override void Serialize<TBufferWriter>(
        ref MemoryPackWriter<TBufferWriter> writer,
        scoped ref Color value)
    {
        // RGBA를 각각 기록
        writer.WriteUnmanaged(value.r);
        writer.WriteUnmanaged(value.g);
        writer.WriteUnmanaged(value.b);
        writer.WriteUnmanaged(value.a);
    }

    public override void Deserialize(
        ref MemoryPackReader reader,
        scoped ref Color value)
    {
        value = new Color(
            reader.ReadUnmanaged<float>(),
            reader.ReadUnmanaged<float>(),
            reader.ReadUnmanaged<float>(),
            reader.ReadUnmanaged<float>());
    }
}

/// <summary>
/// Color32를 압축된 형태로 저장
/// </summary>
public class Color32CompactFormatter : MemoryPackFormatter<Color32>
{
    public override void Serialize<TBufferWriter>(
        ref MemoryPackWriter<TBufferWriter> writer,
        scoped ref Color32 value)
    {
        // 4바이트로 압축
        uint packed = (uint)value.r << 24 |
                     (uint)value.g << 16 |
                     (uint)value.b << 8 |
                     value.a;
        writer.WriteUnmanaged(packed);
    }

    public override void Deserialize(
        ref MemoryPackReader reader,
        scoped ref Color32 value)
    {
        uint packed = reader.ReadUnmanaged<uint>();
        value = new Color32(
            (byte)(packed >> 24),
            (byte)(packed >> 16),
            (byte)(packed >> 8),
            (byte)packed);
    }
}

/// <summary>
/// Formatter 등록
/// </summary>
public static class CustomFormatterRegistration
{
    [UnityEngine.RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    public static void Register()
    {
        MemoryPackFormatterProvider.Register(new ColorFormatter());
        MemoryPackFormatterProvider.Register(new Color32CompactFormatter());
    }
}
```

---

## IL2CPP 호환성

### Source Generator 설정

```csharp
// MemoryPack은 Source Generator를 사용하므로
// 별도의 link.xml 설정이 거의 필요 없음

// 단, 제네릭 타입 사용 시 명시적 참조 필요
public static class AotTypeHints
{
    [UnityEngine.Scripting.Preserve]
    public static void PreserveTypes()
    {
        // 제네릭 타입 힌트
        _ = typeof(List<PlayerSaveData>);
        _ = typeof(Dictionary<string, int>);
        _ = typeof(Dictionary<int, InventorySlot>);
    }
}
```

### 빌드 설정

```xml
<!-- Assets/link.xml (필요한 경우만) -->
<linker>
    <assembly fullname="MemoryPack.Core" preserve="all"/>
    <assembly fullname="System.Memory" preserve="all"/>
    <assembly fullname="System.Buffers" preserve="all"/>
</linker>
```

---

## 성능 비교

```csharp
using System.Diagnostics;
using MemoryPack;
using UnityEngine;
using Debug = UnityEngine.Debug;

/// <summary>
/// 직렬화 성능 벤치마크
/// </summary>
public class SerializationBenchmark : MonoBehaviour
{
    [MemoryPackable]
    public partial class TestData
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public float[] Values { get; set; }
        public Dictionary<string, int> Map { get; set; }
    }

    private void Start()
    {
        var data = new TestData
        {
            Id = 12345,
            Name = "Test Object",
            Values = new float[100],
            Map = new Dictionary<string, int>
            {
                { "key1", 100 },
                { "key2", 200 },
                { "key3", 300 }
            }
        };

        const int iterations = 10000;

        // MemoryPack 벤치마크
        var sw = Stopwatch.StartNew();
        byte[] mpBytes = null;
        for (int i = 0; i < iterations; i++)
        {
            mpBytes = MemoryPackSerializer.Serialize(data);
        }
        sw.Stop();
        Debug.Log($"MemoryPack 직렬화: {sw.ElapsedMilliseconds}ms, 크기: {mpBytes.Length} bytes");

        sw.Restart();
        for (int i = 0; i < iterations; i++)
        {
            MemoryPackSerializer.Deserialize<TestData>(mpBytes);
        }
        sw.Stop();
        Debug.Log($"MemoryPack 역직렬화: {sw.ElapsedMilliseconds}ms");

        // JsonUtility 비교
        sw.Restart();
        string json = null;
        for (int i = 0; i < iterations; i++)
        {
            json = JsonUtility.ToJson(data);
        }
        sw.Stop();
        Debug.Log($"JsonUtility 직렬화: {sw.ElapsedMilliseconds}ms, 크기: {json.Length} bytes");
    }
}
```

---

## 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────┐
│                   MemoryPack 베스트 프랙티스                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 타입 정의                                                    │
│     ├── partial 키워드 필수                                      │
│     ├── 구조체는 값 복사에 주의                                   │
│     └── Union으로 다형성 처리                                    │
│                                                                  │
│  2. 성능 최적화                                                   │
│     ├── ArrayBufferWriter 재사용                                 │
│     ├── 구조체는 ref로 전달                                       │
│     └── 작은 데이터는 stackalloc 활용                             │
│                                                                  │
│  3. 버전 관리                                                    │
│     ├── Version 필드 포함                                        │
│     ├── 새 필드는 nullable 타입                                   │
│     └── 마이그레이션 로직 구현                                   │
│                                                                  │
│  4. 네트워크 사용                                                 │
│     ├── 길이 프리픽스 포함                                        │
│     ├── Union으로 메시지 타입 구분                                │
│     └── 배치 처리로 오버헤드 감소                                 │
│                                                                  │
│  5. Unity 통합                                                   │
│     ├── Source Generator 지원 확인 (2022.3+)                     │
│     ├── 커스텀 Formatter로 Unity 타입 지원                        │
│     └── IL2CPP 자동 호환                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [MemoryPack GitHub](https://github.com/Cysharp/MemoryPack)
- [MemoryPack Documentation](https://github.com/Cysharp/MemoryPack#readme)
- [Unity Integration Guide](https://github.com/Cysharp/MemoryPack#unity)
- [Performance Comparison](https://github.com/Cysharp/MemoryPack#performance)

---

## 다음 단계

- [Section 32: FlatBuffers](./32-flatbuffers.md) - 제로 파싱 직렬화
- [Section 33: GraphQL](./33-graphql.md) - 유연한 쿼리 API
