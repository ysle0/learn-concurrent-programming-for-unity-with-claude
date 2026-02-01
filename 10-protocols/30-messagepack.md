# 30. MessagePack

## 개요

**MessagePack**는 효율적인 바이너리 직렬화 포맷입니다. JSON과 유사한 구조를 가지면서도 바이너리 인코딩을 사용하여 더 작은 크기와 더 빠른 처리 속도를 제공합니다. Unity 게임 개발에서 네트워크 통신, 세이브 시스템, 캐싱 등에 널리 사용됩니다.

### MessagePack vs JSON 비교

| 항목 | JSON | MessagePack |
|------|------|-------------|
| **포맷** | 텍스트 (UTF-8) | 바이너리 |
| **가독성** | 사람이 읽기 쉬움 | 사람이 읽기 어려움 |
| **크기** | 상대적으로 큼 | JSON 대비 50~80% 수준 |
| **직렬화 속도** | 느림 | 빠름 (2~5배) |
| **역직렬화 속도** | 느림 | 빠름 (2~5배) |
| **타입 지원** | 문자열/숫자/불리언/null/배열/객체 | +바이너리(byte[]), 확장 타입 |
| **Unity 지원** | JsonUtility, Newtonsoft.Json | MessagePack-CSharp |
| **디버깅** | 쉬움 (텍스트 확인) | 별도 도구 필요 |

### MessagePack 바이너리 포맷 구조

```
JSON:    {"compact":true,"schema":0}       (27 bytes)
MsgPack: 82 a7 compact c3 a6 schema 00    (18 bytes, ~67%)
```

```
포맷 바이트 구조:
┌──────────────────────────────────────────┐
│ 0x00-0x7f : positive fixint (0~127)      │
│ 0x80-0x8f : fixmap (0~15 요소)           │
│ 0x90-0x9f : fixarray (0~15 요소)         │
│ 0xa0-0xbf : fixstr (0~31 bytes)          │
│ 0xc0      : nil                          │
│ 0xc2-0xc3 : false / true                 │
│ 0xca-0xcb : float32 / float64            │
│ 0xcc-0xd3 : uint/int (8~64bit)           │
│ 0xe0-0xff : negative fixint (-32~-1)     │
└──────────────────────────────────────────┘
```

---

## 1. MessagePack-CSharp 라이브러리 설치

**MessagePack-CSharp**는 neuecc(Yoshifumi Kawai)가 개발한 C#용 고성능 MessagePack 구현체입니다. Unity 환경에 최적화되어 있으며, Zero-allocation API를 제공합니다.

### NuGet 패키지 (일반 .NET)

```
dotnet add package MessagePack
dotnet add package MessagePack.Annotations
```

### Unity 설치 (NuGetForUnity)

```
// NuGetForUnity를 통해 설치
// Window > NuGet Package Manager > MessagePack 검색

// 또는 UPM Git URL
// https://github.com/MessagePack-CSharp/MessagePack-CSharp.git?path=src/MessagePack.UnityClient/Assets/Scripts/MessagePack
```

### Unity Package 직접 설치

```
// 1. GitHub Release에서 .unitypackage 다운로드
//    https://github.com/MessagePack-CSharp/MessagePack-CSharp/releases

// 2. Assets > Import Package > Custom Package
//    MessagePack.x.x.x.unitypackage 선택

// 3. 필요한 어셈블리 확인
//    - MessagePack.dll
//    - MessagePack.Annotations.dll
//    - Microsoft.NET.StringTools.dll (의존성)
```

---

## 2. 기본 사용법 (Serialize / Deserialize)

### 기본 직렬화와 역직렬화

```csharp
using UnityEngine;
using MessagePack;

// =============================================
// MessagePackObject 어트리뷰트로 직렬화 대상 지정
// =============================================
[MessagePackObject]
public class PlayerData
{
    [Key(0)]
    public string PlayerName { get; set; }

    [Key(1)]
    public int Level { get; set; }

    [Key(2)]
    public float Experience { get; set; }

    [Key(3)]
    public Vector3Serializable Position { get; set; }
}

// Unity Vector3는 직접 직렬화 불가 -> 래퍼 사용
[MessagePackObject]
public struct Vector3Serializable
{
    [Key(0)]
    public float X { get; set; }

    [Key(1)]
    public float Y { get; set; }

    [Key(2)]
    public float Z { get; set; }

    public Vector3Serializable(Vector3 v)
    {
        X = v.x;
        Y = v.y;
        Z = v.z;
    }

    public Vector3 ToVector3() => new Vector3(X, Y, Z);
}

public class MessagePackBasicExample : MonoBehaviour
{
    private void Start()
    {
        // =============================================
        // 직렬화 (객체 -> byte[])
        // =============================================
        var player = new PlayerData
        {
            PlayerName = "홍길동",
            Level = 42,
            Experience = 1234.56f,
            Position = new Vector3Serializable(new Vector3(1.5f, 2.0f, 3.5f))
        };

        byte[] bytes = MessagePackSerializer.Serialize(player);
        Debug.Log($"직렬화 크기: {bytes.Length} bytes");

        // =============================================
        // 역직렬화 (byte[] -> 객체)
        // =============================================
        PlayerData deserialized = MessagePackSerializer.Deserialize<PlayerData>(bytes);
        Debug.Log($"이름: {deserialized.PlayerName}");
        Debug.Log($"레벨: {deserialized.Level}");
        Debug.Log($"경험치: {deserialized.Experience}");
        Debug.Log($"위치: {deserialized.Position.ToVector3()}");

        // =============================================
        // JSON 형태로 디버깅 출력
        // =============================================
        string json = MessagePackSerializer.ConvertToJson(bytes);
        Debug.Log($"JSON 표현: {json}");
    }
}
```

### ReadOnlyMemory / Stream 기반 직렬화

```csharp
using UnityEngine;
using MessagePack;
using System;
using System.IO;
using System.Threading;
using Cysharp.Threading.Tasks;

public class MessagePackStreamExample : MonoBehaviour
{
    // =============================================
    // ReadOnlyMemory<byte> 기반 역직렬화
    // =============================================
    public void DeserializeFromMemory(ReadOnlyMemory<byte> buffer)
    {
        var data = MessagePackSerializer.Deserialize<PlayerData>(buffer);
        Debug.Log($"메모리에서 역직렬화: {data.PlayerName}");
    }

    // =============================================
    // Stream 기반 비동기 직렬화/역직렬화
    // =============================================
    public async UniTaskVoid SaveToFileAsync(PlayerData data, string filePath,
        CancellationToken ct)
    {
        using var stream = new FileStream(filePath, FileMode.Create,
            FileAccess.Write, FileShare.None, 4096, useAsync: true);

        await MessagePackSerializer.SerializeAsync(stream, data,
            cancellationToken: ct);

        Debug.Log($"파일 저장 완료: {filePath}");
    }

    public async UniTask<PlayerData> LoadFromFileAsync(string filePath,
        CancellationToken ct)
    {
        using var stream = new FileStream(filePath, FileMode.Open,
            FileAccess.Read, FileShare.Read, 4096, useAsync: true);

        var data = await MessagePackSerializer.DeserializeAsync<PlayerData>(
            stream, cancellationToken: ct);

        Debug.Log($"파일 로드 완료: {data.PlayerName}");
        return data;
    }
}
```

---

## 3. 어트리뷰트 시스템 ([MessagePackObject], [Key])

### 정수 키 (Array 포맷) - 권장

```csharp
using MessagePack;

// =============================================
// 정수 Key: 배열 형태로 직렬화 (더 컴팩트)
// =============================================
[MessagePackObject]
public class GameState
{
    [Key(0)]
    public int CurrentStage { get; set; }

    [Key(1)]
    public int Score { get; set; }

    [Key(2)]
    public float PlayTime { get; set; }

    [Key(3)]
    public bool IsCompleted { get; set; }

    // Key를 건너뛸 수 있음 (빈 슬롯은 null/default)
    // Key(4)는 건너뜀

    [Key(5)]
    public string[] UnlockedItems { get; set; }

    // IgnoreMember: 직렬화에서 제외
    [IgnoreMember]
    public float CachedValue { get; set; }
}

// 직렬화 결과: [1, 1000, 123.45, true, null, ["sword","shield"]]
//              ^ 배열 형태 (Key 인덱스 기반)
```

### 문자열 키 (Map 포맷) - 호환성 우선

```csharp
using MessagePack;

// =============================================
// 문자열 Key: 맵 형태로 직렬화 (JSON과 유사)
// =============================================
[MessagePackObject(keyAsPropertyName: true)]
public class GameConfig
{
    // 프로퍼티 이름이 자동으로 키가 됨
    public int MaxPlayers { get; set; }
    public float TickRate { get; set; }
    public string ServerRegion { get; set; }
    public bool EnableChat { get; set; }

    [IgnoreMember]
    public bool IsDirty { get; set; }
}

// 직렬화 결과: {"MaxPlayers":4,"TickRate":60.0,"ServerRegion":"Asia","EnableChat":true}
//              ^ 맵 형태 (프로퍼티 이름 기반)
```

### Key 명시 문자열 키

```csharp
using MessagePack;

// =============================================
// 명시적 문자열 Key (서버 호환용)
// =============================================
[MessagePackObject]
public class ApiResponse
{
    [Key("status_code")]
    public int StatusCode { get; set; }

    [Key("message")]
    public string Message { get; set; }

    [Key("data")]
    public byte[] Data { get; set; }

    [Key("timestamp")]
    public long Timestamp { get; set; }
}
```

### 정수 키 vs 문자열 키 비교

```
정수 키 (Array 포맷):
┌───────────────────────────────────────┐
│ 장점: 크기가 작고 직렬화/역직렬화 빠름 │
│ 단점: 키 번호 관리 필요, 순서 중요      │
│ 용도: 게임 내부 통신, 세이브 데이터     │
└───────────────────────────────────────┘

문자열 키 (Map 포맷):
┌───────────────────────────────────────┐
│ 장점: 가독성 좋음, JSON 호환 용이      │
│ 단점: 키 이름으로 인해 크기 증가        │
│ 용도: REST API 통신, 외부 시스템 연동   │
└───────────────────────────────────────┘
```

---

## 4. 복합 타입과 컬렉션 직렬화

```csharp
using UnityEngine;
using MessagePack;
using System;
using System.Collections.Generic;

// =============================================
// 중첩 객체 직렬화
// =============================================
[MessagePackObject]
public class GameSaveData
{
    [Key(0)]
    public PlayerInfo Player { get; set; }

    [Key(1)]
    public List<InventoryItem> Inventory { get; set; }

    [Key(2)]
    public Dictionary<string, int> Achievements { get; set; }

    [Key(3)]
    public QuestProgress[] ActiveQuests { get; set; }

    [Key(4)]
    public DateTime SavedAt { get; set; }
}

[MessagePackObject]
public class PlayerInfo
{
    [Key(0)]
    public string Name { get; set; }

    [Key(1)]
    public int Level { get; set; }

    [Key(2)]
    public int Hp { get; set; }

    [Key(3)]
    public int MaxHp { get; set; }

    [Key(4)]
    public Dictionary<string, float> Stats { get; set; }
}

[MessagePackObject]
public class InventoryItem
{
    [Key(0)]
    public int ItemId { get; set; }

    [Key(1)]
    public string ItemName { get; set; }

    [Key(2)]
    public int Quantity { get; set; }

    [Key(3)]
    public ItemRarity Rarity { get; set; }
}

public enum ItemRarity
{
    Common,
    Uncommon,
    Rare,
    Epic,
    Legendary
}

[MessagePackObject]
public class QuestProgress
{
    [Key(0)]
    public int QuestId { get; set; }

    [Key(1)]
    public string QuestName { get; set; }

    [Key(2)]
    public float Progress { get; set; }  // 0.0 ~ 1.0
}

public class SaveSystemExample : MonoBehaviour
{
    public void SaveGame()
    {
        var saveData = new GameSaveData
        {
            Player = new PlayerInfo
            {
                Name = "전사",
                Level = 25,
                Hp = 850,
                MaxHp = 1000,
                Stats = new Dictionary<string, float>
                {
                    { "STR", 45.5f },
                    { "DEX", 30.0f },
                    { "INT", 20.0f }
                }
            },
            Inventory = new List<InventoryItem>
            {
                new InventoryItem
                {
                    ItemId = 1001, ItemName = "불꽃의 검",
                    Quantity = 1, Rarity = ItemRarity.Epic
                },
                new InventoryItem
                {
                    ItemId = 2001, ItemName = "체력 물약",
                    Quantity = 50, Rarity = ItemRarity.Common
                }
            },
            Achievements = new Dictionary<string, int>
            {
                { "first_kill", 1 },
                { "dungeon_clear", 3 },
                { "boss_defeat", 1 }
            },
            ActiveQuests = new[]
            {
                new QuestProgress { QuestId = 101, QuestName = "마을 구출", Progress = 0.75f }
            },
            SavedAt = DateTime.UtcNow
        };

        // 직렬화
        byte[] bytes = MessagePackSerializer.Serialize(saveData);
        Debug.Log($"세이브 데이터 크기: {bytes.Length} bytes");

        // 비교: JSON 크기
        string json = MessagePackSerializer.ConvertToJson(bytes);
        Debug.Log($"JSON 표현 크기: {System.Text.Encoding.UTF8.GetByteCount(json)} bytes");

        // 역직렬화
        var loaded = MessagePackSerializer.Deserialize<GameSaveData>(bytes);
        Debug.Log($"로드된 플레이어: {loaded.Player.Name} (Lv.{loaded.Player.Level})");
        Debug.Log($"인벤토리 아이템 수: {loaded.Inventory.Count}");
    }
}
```

---

## 5. Union (다형성 지원)

`Union` 어트리뷰트를 사용하면 인터페이스나 추상 클래스의 다형성 직렬화가 가능합니다.

### 인터페이스 기반 Union

```csharp
using UnityEngine;
using MessagePack;
using System.Collections.Generic;

// =============================================
// Union: 다형성 직렬화를 위한 태그 지정
// =============================================
[Union(0, typeof(MoveCommand))]
[Union(1, typeof(AttackCommand))]
[Union(2, typeof(UseItemCommand))]
[Union(3, typeof(ChatCommand))]
public interface IGameCommand
{
    long Timestamp { get; set; }
    int PlayerId { get; set; }
}

[MessagePackObject]
public class MoveCommand : IGameCommand
{
    [Key(0)]
    public long Timestamp { get; set; }

    [Key(1)]
    public int PlayerId { get; set; }

    [Key(2)]
    public float TargetX { get; set; }

    [Key(3)]
    public float TargetY { get; set; }

    [Key(4)]
    public float TargetZ { get; set; }
}

[MessagePackObject]
public class AttackCommand : IGameCommand
{
    [Key(0)]
    public long Timestamp { get; set; }

    [Key(1)]
    public int PlayerId { get; set; }

    [Key(2)]
    public int TargetId { get; set; }

    [Key(3)]
    public int SkillId { get; set; }
}

[MessagePackObject]
public class UseItemCommand : IGameCommand
{
    [Key(0)]
    public long Timestamp { get; set; }

    [Key(1)]
    public int PlayerId { get; set; }

    [Key(2)]
    public int ItemId { get; set; }

    [Key(3)]
    public int SlotIndex { get; set; }
}

[MessagePackObject]
public class ChatCommand : IGameCommand
{
    [Key(0)]
    public long Timestamp { get; set; }

    [Key(1)]
    public int PlayerId { get; set; }

    [Key(2)]
    public string Message { get; set; }

    [Key(3)]
    public int Channel { get; set; }  // 0: 전체, 1: 팀, 2: 귓속말
}

public class UnionExample : MonoBehaviour
{
    private void Start()
    {
        // 다양한 커맨드를 하나의 인터페이스로 직렬화
        var commands = new List<IGameCommand>
        {
            new MoveCommand
            {
                Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
                PlayerId = 1,
                TargetX = 10.5f, TargetY = 0f, TargetZ = 20.3f
            },
            new AttackCommand
            {
                Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
                PlayerId = 1,
                TargetId = 42,
                SkillId = 3
            },
            new ChatCommand
            {
                Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
                PlayerId = 1,
                Message = "파티 참가 가능한가요?",
                Channel = 0
            }
        };

        // 직렬화 - Union 태그가 자동으로 포함됨
        byte[] bytes = MessagePackSerializer.Serialize(commands);
        Debug.Log($"커맨드 패킷 크기: {bytes.Length} bytes");

        // 역직렬화 - 실제 타입이 복원됨
        var deserialized = MessagePackSerializer.Deserialize<List<IGameCommand>>(bytes);

        foreach (var cmd in deserialized)
        {
            switch (cmd)
            {
                case MoveCommand move:
                    Debug.Log($"[이동] Player {move.PlayerId} -> ({move.TargetX}, {move.TargetY}, {move.TargetZ})");
                    break;
                case AttackCommand attack:
                    Debug.Log($"[공격] Player {attack.PlayerId} -> Target {attack.TargetId}, Skill {attack.SkillId}");
                    break;
                case ChatCommand chat:
                    Debug.Log($"[채팅] Player {chat.PlayerId}: {chat.Message}");
                    break;
            }
        }
    }
}
```

### 추상 클래스 기반 Union

```csharp
using MessagePack;

// =============================================
// 추상 클래스 기반 Union (공통 로직 포함 가능)
// =============================================
[Union(0, typeof(DamageEffect))]
[Union(1, typeof(HealEffect))]
[Union(2, typeof(BuffEffect))]
[MessagePackObject]
public abstract class GameEffect
{
    [Key(0)]
    public int TargetEntityId { get; set; }

    [Key(1)]
    public float Duration { get; set; }

    [IgnoreMember]
    public abstract string EffectType { get; }
}

[MessagePackObject]
public class DamageEffect : GameEffect
{
    [Key(2)]
    public int DamageAmount { get; set; }

    [Key(3)]
    public string DamageType { get; set; }  // "physical", "magic", "true"

    [IgnoreMember]
    public override string EffectType => "Damage";
}

[MessagePackObject]
public class HealEffect : GameEffect
{
    [Key(2)]
    public int HealAmount { get; set; }

    [Key(3)]
    public bool IsOverTime { get; set; }  // HoT (Heal over Time)

    [IgnoreMember]
    public override string EffectType => "Heal";
}

[MessagePackObject]
public class BuffEffect : GameEffect
{
    [Key(2)]
    public string StatName { get; set; }

    [Key(3)]
    public float Multiplier { get; set; }

    [IgnoreMember]
    public override string EffectType => "Buff";
}
```

---

## 6. 커스텀 포매터 (Custom Formatter)

기본 포매터가 지원하지 않는 타입에 대해 커스텀 포매터를 작성할 수 있습니다.

### Unity Vector3 포매터

```csharp
using UnityEngine;
using MessagePack;
using MessagePack.Formatters;

// =============================================
// Unity Vector3를 위한 커스텀 포매터
// =============================================
public class Vector3Formatter : IMessagePackFormatter<Vector3>
{
    public static readonly Vector3Formatter Instance = new Vector3Formatter();

    public void Serialize(ref MessagePackWriter writer, Vector3 value,
        MessagePackSerializerOptions options)
    {
        // 3개의 float를 배열로 기록
        writer.WriteArrayHeader(3);
        writer.Write(value.x);
        writer.Write(value.y);
        writer.Write(value.z);
    }

    public Vector3 Deserialize(ref MessagePackReader reader,
        MessagePackSerializerOptions options)
    {
        var count = reader.ReadArrayHeader();
        if (count != 3)
            throw new MessagePackSerializationException(
                $"Vector3는 3개의 요소가 필요합니다. 실제: {count}");

        float x = reader.ReadSingle();
        float y = reader.ReadSingle();
        float z = reader.ReadSingle();

        return new Vector3(x, y, z);
    }
}

// =============================================
// Quaternion 포매터
// =============================================
public class QuaternionFormatter : IMessagePackFormatter<Quaternion>
{
    public static readonly QuaternionFormatter Instance = new QuaternionFormatter();

    public void Serialize(ref MessagePackWriter writer, Quaternion value,
        MessagePackSerializerOptions options)
    {
        writer.WriteArrayHeader(4);
        writer.Write(value.x);
        writer.Write(value.y);
        writer.Write(value.z);
        writer.Write(value.w);
    }

    public Quaternion Deserialize(ref MessagePackReader reader,
        MessagePackSerializerOptions options)
    {
        var count = reader.ReadArrayHeader();
        if (count != 4)
            throw new MessagePackSerializationException(
                $"Quaternion은 4개의 요소가 필요합니다. 실제: {count}");

        float x = reader.ReadSingle();
        float y = reader.ReadSingle();
        float z = reader.ReadSingle();
        float w = reader.ReadSingle();

        return new Quaternion(x, y, z, w);
    }
}

// =============================================
// Color 포매터
// =============================================
public class ColorFormatter : IMessagePackFormatter<Color>
{
    public static readonly ColorFormatter Instance = new ColorFormatter();

    public void Serialize(ref MessagePackWriter writer, Color value,
        MessagePackSerializerOptions options)
    {
        writer.WriteArrayHeader(4);
        writer.Write(value.r);
        writer.Write(value.g);
        writer.Write(value.b);
        writer.Write(value.a);
    }

    public Color Deserialize(ref MessagePackReader reader,
        MessagePackSerializerOptions options)
    {
        var count = reader.ReadArrayHeader();
        if (count != 4)
            throw new MessagePackSerializationException(
                $"Color는 4개의 요소가 필요합니다. 실제: {count}");

        return new Color(
            reader.ReadSingle(),
            reader.ReadSingle(),
            reader.ReadSingle(),
            reader.ReadSingle()
        );
    }
}
```

### 커스텀 Resolver 등록

```csharp
using MessagePack;
using MessagePack.Formatters;
using MessagePack.Resolvers;
using UnityEngine;

// =============================================
// Unity 타입 전용 Resolver
// =============================================
public class UnityResolver : IFormatterResolver
{
    public static readonly UnityResolver Instance = new UnityResolver();

    private UnityResolver() { }

    public IMessagePackFormatter<T> GetFormatter<T>()
    {
        return FormatterCache<T>.Formatter;
    }

    private static class FormatterCache<T>
    {
        internal static readonly IMessagePackFormatter<T> Formatter;

        static FormatterCache()
        {
            Formatter = (IMessagePackFormatter<T>)UnityResolverGetFormatterHelper
                .GetFormatter(typeof(T));
        }
    }
}

internal static class UnityResolverGetFormatterHelper
{
    internal static object GetFormatter(System.Type t)
    {
        if (t == typeof(Vector2))
            return new Vector2Formatter();
        if (t == typeof(Vector3))
            return Vector3Formatter.Instance;
        if (t == typeof(Vector4))
            return new Vector4Formatter();
        if (t == typeof(Quaternion))
            return QuaternionFormatter.Instance;
        if (t == typeof(Color))
            return ColorFormatter.Instance;
        if (t == typeof(Color32))
            return new Color32Formatter();
        if (t == typeof(Rect))
            return new RectFormatter();

        return null;
    }
}

// Vector2 포매터 (추가 예시)
public class Vector2Formatter : IMessagePackFormatter<Vector2>
{
    public void Serialize(ref MessagePackWriter writer, Vector2 value,
        MessagePackSerializerOptions options)
    {
        writer.WriteArrayHeader(2);
        writer.Write(value.x);
        writer.Write(value.y);
    }

    public Vector2 Deserialize(ref MessagePackReader reader,
        MessagePackSerializerOptions options)
    {
        reader.ReadArrayHeader();
        return new Vector2(reader.ReadSingle(), reader.ReadSingle());
    }
}

// Vector4 포매터
public class Vector4Formatter : IMessagePackFormatter<Vector4>
{
    public void Serialize(ref MessagePackWriter writer, Vector4 value,
        MessagePackSerializerOptions options)
    {
        writer.WriteArrayHeader(4);
        writer.Write(value.x);
        writer.Write(value.y);
        writer.Write(value.z);
        writer.Write(value.w);
    }

    public Vector4 Deserialize(ref MessagePackReader reader,
        MessagePackSerializerOptions options)
    {
        reader.ReadArrayHeader();
        return new Vector4(
            reader.ReadSingle(), reader.ReadSingle(),
            reader.ReadSingle(), reader.ReadSingle());
    }
}

// Color32 포매터
public class Color32Formatter : IMessagePackFormatter<Color32>
{
    public void Serialize(ref MessagePackWriter writer, Color32 value,
        MessagePackSerializerOptions options)
    {
        writer.WriteArrayHeader(4);
        writer.Write(value.r);
        writer.Write(value.g);
        writer.Write(value.b);
        writer.Write(value.a);
    }

    public Color32 Deserialize(ref MessagePackReader reader,
        MessagePackSerializerOptions options)
    {
        reader.ReadArrayHeader();
        return new Color32(
            reader.ReadByte(), reader.ReadByte(),
            reader.ReadByte(), reader.ReadByte());
    }
}

// Rect 포매터
public class RectFormatter : IMessagePackFormatter<Rect>
{
    public void Serialize(ref MessagePackWriter writer, Rect value,
        MessagePackSerializerOptions options)
    {
        writer.WriteArrayHeader(4);
        writer.Write(value.x);
        writer.Write(value.y);
        writer.Write(value.width);
        writer.Write(value.height);
    }

    public Rect Deserialize(ref MessagePackReader reader,
        MessagePackSerializerOptions options)
    {
        reader.ReadArrayHeader();
        return new Rect(
            reader.ReadSingle(), reader.ReadSingle(),
            reader.ReadSingle(), reader.ReadSingle());
    }
}
```

---

## 7. LZ4 압축 모드

MessagePack-CSharp는 LZ4 압축을 내장 지원하여 데이터를 추가로 압축할 수 있습니다.

### LZ4 압축 사용법

```csharp
using UnityEngine;
using MessagePack;
using System.Diagnostics;
using Debug = UnityEngine.Debug;

public class Lz4CompressionExample : MonoBehaviour
{
    private void Start()
    {
        CompareCompressionModes();
    }

    private void CompareCompressionModes()
    {
        // 테스트용 대규모 데이터 생성
        var largeData = CreateLargeTestData();

        // =============================================
        // 1. 기본 모드 (압축 없음)
        // =============================================
        var defaultOptions = MessagePackSerializerOptions.Standard;

        var sw = Stopwatch.StartNew();
        byte[] normalBytes = MessagePackSerializer.Serialize(largeData, defaultOptions);
        sw.Stop();
        long normalSerializeMs = sw.ElapsedMilliseconds;

        sw.Restart();
        MessagePackSerializer.Deserialize<LargeGameData>(normalBytes, defaultOptions);
        sw.Stop();
        long normalDeserializeMs = sw.ElapsedMilliseconds;

        // =============================================
        // 2. LZ4 블록 압축 (일반적인 경우 권장)
        // =============================================
        var lz4BlockOptions = MessagePackSerializerOptions.Standard
            .WithCompression(MessagePackCompression.Lz4Block);

        sw.Restart();
        byte[] lz4BlockBytes = MessagePackSerializer.Serialize(largeData, lz4BlockOptions);
        sw.Stop();
        long lz4BlockSerializeMs = sw.ElapsedMilliseconds;

        sw.Restart();
        MessagePackSerializer.Deserialize<LargeGameData>(lz4BlockBytes, lz4BlockOptions);
        sw.Stop();
        long lz4BlockDeserializeMs = sw.ElapsedMilliseconds;

        // =============================================
        // 3. LZ4 블록 배열 (대규모 데이터 권장)
        // =============================================
        var lz4ArrayOptions = MessagePackSerializerOptions.Standard
            .WithCompression(MessagePackCompression.Lz4BlockArray);

        sw.Restart();
        byte[] lz4ArrayBytes = MessagePackSerializer.Serialize(largeData, lz4ArrayOptions);
        sw.Stop();
        long lz4ArraySerializeMs = sw.ElapsedMilliseconds;

        sw.Restart();
        MessagePackSerializer.Deserialize<LargeGameData>(lz4ArrayBytes, lz4ArrayOptions);
        sw.Stop();
        long lz4ArrayDeserializeMs = sw.ElapsedMilliseconds;

        // =============================================
        // 결과 비교
        // =============================================
        Debug.Log("=== 압축 모드 비교 ===");
        Debug.Log($"기본:          {normalBytes.Length,8} bytes | " +
                  $"직렬화: {normalSerializeMs}ms | 역직렬화: {normalDeserializeMs}ms");
        Debug.Log($"LZ4 Block:     {lz4BlockBytes.Length,8} bytes | " +
                  $"직렬화: {lz4BlockSerializeMs}ms | 역직렬화: {lz4BlockDeserializeMs}ms");
        Debug.Log($"LZ4 BlockArray:{lz4ArrayBytes.Length,8} bytes | " +
                  $"직렬화: {lz4ArraySerializeMs}ms | 역직렬화: {lz4ArrayDeserializeMs}ms");
        Debug.Log($"압축률: {(1f - (float)lz4BlockBytes.Length / normalBytes.Length) * 100:F1}% 절감");
    }

    private LargeGameData CreateLargeTestData()
    {
        var data = new LargeGameData
        {
            MapTiles = new int[100 * 100],
            EntityPositions = new float[1000 * 3],
            EventLog = new string[500]
        };

        for (int i = 0; i < data.MapTiles.Length; i++)
            data.MapTiles[i] = Random.Range(0, 10);  // 반복적인 데이터 (압축 효과 큼)

        for (int i = 0; i < data.EntityPositions.Length; i++)
            data.EntityPositions[i] = Random.Range(-100f, 100f);

        for (int i = 0; i < data.EventLog.Length; i++)
            data.EventLog[i] = $"이벤트 #{i}: 플레이어 행동 기록";

        return data;
    }
}

[MessagePackObject]
public class LargeGameData
{
    [Key(0)]
    public int[] MapTiles { get; set; }

    [Key(1)]
    public float[] EntityPositions { get; set; }

    [Key(2)]
    public string[] EventLog { get; set; }
}
```

### LZ4 압축 모드 선택 가이드

```
LZ4Block:
┌──────────────────────────────────────────────┐
│ - 전체 데이터를 하나의 블록으로 압축           │
│ - 소~중규모 데이터에 적합 (수 KB ~ 수 MB)     │
│ - 역직렬화 시 전체를 메모리에 해제             │
│ - 일반적인 사용 사례에 권장                    │
└──────────────────────────────────────────────┘

LZ4BlockArray:
┌──────────────────────────────────────────────┐
│ - 데이터를 여러 블록으로 나누어 압축           │
│ - 대규모 데이터에 적합 (수십 MB 이상)          │
│ - 스트리밍 역직렬화 가능                       │
│ - 메모리 피크가 낮음                          │
└──────────────────────────────────────────────┘

None (기본):
┌──────────────────────────────────────────────┐
│ - 압축 오버헤드 없음                          │
│ - 실시간 패킷 (매 프레임 전송)에 적합          │
│ - 데이터가 이미 작거나 압축 불가한 경우         │
└──────────────────────────────────────────────┘
```

---

## 8. AOT Code Generation (IL2CPP 호환)

Unity IL2CPP 빌드(iOS, WebGL 등)에서는 런타임 코드 생성이 불가능하므로, 사전에 직렬화 코드를 생성해야 합니다.

### mpc (MessagePack Compiler) 도구 사용

```bash
# MessagePack Compiler 설치
dotnet tool install --global MessagePack.Generator

# 코드 생성 실행
# -i: 입력 프로젝트 경로
# -o: 출력 파일 경로
mpc -i ./Assets/Scripts/Assembly-CSharp.csproj \
    -o ./Assets/Scripts/Generated/MessagePackGenerated.cs \
    -r GeneratedResolver
```

### Source Generator 방식 (v3.x+)

```csharp
using MessagePack;

// =============================================
// Source Generator 기반 AOT 코드 생성
// MessagePack v3.x 이상에서 사용 가능
// =============================================

// Partial 클래스로 Source Generator 적용
[MessagePackObject]
public partial class NetworkPacket
{
    [Key(0)]
    public int PacketId { get; set; }

    [Key(1)]
    public byte[] Payload { get; set; }

    [Key(2)]
    public long SequenceNumber { get; set; }
}
```

### Resolver 초기화 (v2.x)

```csharp
using UnityEngine;
using MessagePack;
using MessagePack.Resolvers;

// =============================================
// 앱 시작 시 Resolver 초기화 (RuntimeInitializeOnLoadMethod)
// IL2CPP 환경에서 필수
// =============================================
public static class MessagePackInitializer
{
    private static bool _isInitialized = false;

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    public static void Initialize()
    {
        if (_isInitialized) return;

        // Composite Resolver: 여러 Resolver를 결합
        var resolver = CompositeResolver.Create(
            // 생성된 Resolver (AOT)
            GeneratedResolver.Instance,

            // Unity 타입 Resolver (커스텀)
            UnityResolver.Instance,

            // 기본 내장 Resolver
            StandardResolver.Instance
        );

        var options = MessagePackSerializerOptions.Standard
            .WithResolver(resolver)
            .WithCompression(MessagePackCompression.Lz4Block);

        // 글로벌 기본 옵션으로 설정
        MessagePackSerializer.DefaultOptions = options;

        _isInitialized = true;
        Debug.Log("MessagePack 초기화 완료 (AOT 모드)");
    }
}
```

### AOT 코드 생성 확인

```csharp
using UnityEngine;
using MessagePack;

public class AotVerificationExample : MonoBehaviour
{
    // =============================================
    // AOT 환경에서 직렬화 가능 여부 테스트
    // =============================================
    private void Start()
    {
        #if ENABLE_IL2CPP
        Debug.Log("IL2CPP 모드: AOT 코드 생성 확인 중...");

        try
        {
            // 각 타입별 직렬화 테스트
            TestSerialization(new PlayerData
            {
                PlayerName = "테스트",
                Level = 1
            });

            TestSerialization(new GameState
            {
                CurrentStage = 1,
                Score = 0
            });

            Debug.Log("모든 타입 직렬화 성공!");
        }
        catch (MessagePackSerializationException ex)
        {
            Debug.LogError($"AOT 직렬화 실패: {ex.Message}");
            Debug.LogError("mpc 도구로 코드를 재생성해주세요.");
        }
        #endif
    }

    private void TestSerialization<T>(T obj)
    {
        byte[] bytes = MessagePackSerializer.Serialize(obj);
        T result = MessagePackSerializer.Deserialize<T>(bytes);
        Debug.Log($"{typeof(T).Name}: 직렬화/역직렬화 성공 ({bytes.Length} bytes)");
    }
}
```

---

## 9. 성능 벤치마크

### MessagePack vs JSON vs MemoryPack 비교

```csharp
using UnityEngine;
using MessagePack;
using System.Diagnostics;
using Debug = UnityEngine.Debug;

public class SerializationBenchmark : MonoBehaviour
{
    private const int Iterations = 10000;

    private void Start()
    {
        RunBenchmark();
    }

    private void RunBenchmark()
    {
        var testData = new PlayerData
        {
            PlayerName = "BenchmarkPlayer",
            Level = 99,
            Experience = 99999.99f,
            Position = new Vector3Serializable(
                new Vector3(123.456f, 789.012f, 345.678f))
        };

        Debug.Log($"=== 직렬화 벤치마크 ({Iterations}회 반복) ===");

        // =============================================
        // MessagePack 벤치마크
        // =============================================
        var sw = Stopwatch.StartNew();
        byte[] msgpackBytes = null;
        for (int i = 0; i < Iterations; i++)
        {
            msgpackBytes = MessagePackSerializer.Serialize(testData);
        }
        sw.Stop();
        long msgpackSerMs = sw.ElapsedMilliseconds;

        sw.Restart();
        for (int i = 0; i < Iterations; i++)
        {
            MessagePackSerializer.Deserialize<PlayerData>(msgpackBytes);
        }
        sw.Stop();
        long msgpackDeMs = sw.ElapsedMilliseconds;

        // =============================================
        // MessagePack + LZ4 벤치마크
        // =============================================
        var lz4Options = MessagePackSerializerOptions.Standard
            .WithCompression(MessagePackCompression.Lz4Block);

        sw.Restart();
        byte[] lz4Bytes = null;
        for (int i = 0; i < Iterations; i++)
        {
            lz4Bytes = MessagePackSerializer.Serialize(testData, lz4Options);
        }
        sw.Stop();
        long lz4SerMs = sw.ElapsedMilliseconds;

        sw.Restart();
        for (int i = 0; i < Iterations; i++)
        {
            MessagePackSerializer.Deserialize<PlayerData>(lz4Bytes, lz4Options);
        }
        sw.Stop();
        long lz4DeMs = sw.ElapsedMilliseconds;

        // =============================================
        // JsonUtility 벤치마크 (Unity 내장)
        // =============================================
        // 참고: JsonUtility는 MessagePackObject를 직접 지원하지 않으므로
        //       별도의 JSON 호환 클래스 필요
        var jsonData = new PlayerDataJson
        {
            playerName = "BenchmarkPlayer",
            level = 99,
            experience = 99999.99f,
            posX = 123.456f, posY = 789.012f, posZ = 345.678f
        };

        sw.Restart();
        string jsonStr = null;
        for (int i = 0; i < Iterations; i++)
        {
            jsonStr = JsonUtility.ToJson(jsonData);
        }
        sw.Stop();
        long jsonSerMs = sw.ElapsedMilliseconds;

        sw.Restart();
        for (int i = 0; i < Iterations; i++)
        {
            JsonUtility.FromJson<PlayerDataJson>(jsonStr);
        }
        sw.Stop();
        long jsonDeMs = sw.ElapsedMilliseconds;

        byte[] jsonBytes = System.Text.Encoding.UTF8.GetBytes(jsonStr);

        // =============================================
        // 결과 출력
        // =============================================
        Debug.Log("--- 크기 비교 ---");
        Debug.Log($"MessagePack:       {msgpackBytes.Length,6} bytes");
        Debug.Log($"MessagePack+LZ4:   {lz4Bytes.Length,6} bytes");
        Debug.Log($"JSON (UTF-8):      {jsonBytes.Length,6} bytes");

        Debug.Log("--- 직렬화 속도 (ms) ---");
        Debug.Log($"MessagePack:       {msgpackSerMs,6} ms");
        Debug.Log($"MessagePack+LZ4:   {lz4SerMs,6} ms");
        Debug.Log($"JSON:              {jsonSerMs,6} ms");

        Debug.Log("--- 역직렬화 속도 (ms) ---");
        Debug.Log($"MessagePack:       {msgpackDeMs,6} ms");
        Debug.Log($"MessagePack+LZ4:   {lz4DeMs,6} ms");
        Debug.Log($"JSON:              {jsonDeMs,6} ms");
    }
}

// JsonUtility 호환 클래스
[System.Serializable]
public class PlayerDataJson
{
    public string playerName;
    public int level;
    public float experience;
    public float posX, posY, posZ;
}
```

### 일반적인 벤치마크 결과 (참고값)

```
┌──────────────────────────────────────────────────────────────────┐
│ 직렬화 포맷 비교 (10,000회 반복, 중간 복잡도 객체 기준)           │
├──────────────────┬──────────┬───────────┬───────────┬───────────┤
│ 포맷             │ 크기     │ 직렬화    │ 역직렬화  │ GC Alloc  │
├──────────────────┼──────────┼───────────┼───────────┼───────────┤
│ MessagePack      │  ~65 B   │   ~12 ms  │   ~10 ms  │ 매우 낮음 │
│ MsgPack + LZ4    │  ~58 B   │   ~18 ms  │   ~14 ms  │ 낮음     │
│ JsonUtility      │ ~120 B   │   ~25 ms  │   ~30 ms  │ 중간     │
│ Newtonsoft.Json   │ ~130 B   │   ~85 ms  │  ~110 ms  │ 높음     │
│ MemoryPack       │  ~52 B   │    ~6 ms  │    ~5 ms  │ 매우 낮음 │
│ System.Text.Json │ ~125 B   │   ~40 ms  │   ~50 ms  │ 중간     │
├──────────────────┴──────────┴───────────┴───────────┴───────────┤
│ * 실제 성능은 데이터 구조, 플랫폼, .NET 버전에 따라 다릅니다     │
│ * MemoryPack은 .NET 7+ / Source Generator 필수                  │
│ * MessagePack은 범용성과 성능의 균형이 가장 뛰어남               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. Unity 통합 (Resolver 설정)

### 전역 설정 패턴

```csharp
using UnityEngine;
using MessagePack;
using MessagePack.Resolvers;

// =============================================
// Unity 프로젝트 전역 MessagePack 설정
// =============================================
public static class MessagePackConfig
{
    private static bool _initialized;
    private static MessagePackSerializerOptions _options;

    public static MessagePackSerializerOptions Options
    {
        get
        {
            if (!_initialized) Initialize();
            return _options;
        }
    }

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    public static void Initialize()
    {
        if (_initialized) return;

        // Resolver 구성
        StaticCompositeResolver.Instance.Register(
            // 1순위: Unity 타입 포매터
            UnityResolver.Instance,

            // 2순위: AOT 생성 코드 (IL2CPP 빌드 시)
            #if ENABLE_IL2CPP
            GeneratedResolver.Instance,
            #endif

            // 3순위: 내장 타입 + 동적 리졸버
            StandardResolver.Instance
        );

        _options = MessagePackSerializerOptions.Standard
            .WithResolver(StaticCompositeResolver.Instance)
            .WithCompression(MessagePackCompression.Lz4Block)
            .WithSecurity(MessagePackSecurity.UntrustedData);

        // 글로벌 기본 옵션 설정
        MessagePackSerializer.DefaultOptions = _options;

        _initialized = true;
    }
}
```

### 보안 설정 (UntrustedData)

```csharp
using MessagePack;

// =============================================
// 외부 데이터 수신 시 보안 설정
// =============================================
public static class SecureMessagePack
{
    // 네트워크에서 수신한 데이터는 반드시 UntrustedData 모드 사용
    private static readonly MessagePackSerializerOptions SafeOptions =
        MessagePackSerializerOptions.Standard
            .WithSecurity(MessagePackSecurity.UntrustedData);

    public static T DeserializeFromNetwork<T>(byte[] data)
    {
        // UntrustedData: 해시 충돌 공격 방어, 깊이 제한 등
        return MessagePackSerializer.Deserialize<T>(data, SafeOptions);
    }

    public static byte[] SerializeForNetwork<T>(T obj)
    {
        return MessagePackSerializer.Serialize(obj, SafeOptions);
    }
}
```

### 네트워크 매니저 통합 예시

```csharp
using UnityEngine;
using MessagePack;
using System;
using System.Collections.Generic;
using Cysharp.Threading.Tasks;

// =============================================
// MessagePack 기반 네트워크 패킷 시스템
// =============================================
[Union(0, typeof(PingPacket))]
[Union(1, typeof(PongPacket))]
[Union(2, typeof(GameStatePacket))]
[Union(3, typeof(PlayerInputPacket))]
public interface INetworkPacket
{
    int PacketType { get; }
    long Timestamp { get; set; }
}

[MessagePackObject]
public class PingPacket : INetworkPacket
{
    [IgnoreMember]
    public int PacketType => 0;

    [Key(0)]
    public long Timestamp { get; set; }

    [Key(1)]
    public int SequenceId { get; set; }
}

[MessagePackObject]
public class PongPacket : INetworkPacket
{
    [IgnoreMember]
    public int PacketType => 1;

    [Key(0)]
    public long Timestamp { get; set; }

    [Key(1)]
    public int SequenceId { get; set; }

    [Key(2)]
    public long ServerTime { get; set; }
}

[MessagePackObject]
public class GameStatePacket : INetworkPacket
{
    [IgnoreMember]
    public int PacketType => 2;

    [Key(0)]
    public long Timestamp { get; set; }

    [Key(1)]
    public int Tick { get; set; }

    [Key(2)]
    public List<EntitySnapshot> Entities { get; set; }
}

[MessagePackObject]
public class EntitySnapshot
{
    [Key(0)]
    public int EntityId { get; set; }

    [Key(1)]
    public float X { get; set; }

    [Key(2)]
    public float Y { get; set; }

    [Key(3)]
    public float Z { get; set; }

    [Key(4)]
    public int State { get; set; }

    [Key(5)]
    public int Hp { get; set; }
}

[MessagePackObject]
public class PlayerInputPacket : INetworkPacket
{
    [IgnoreMember]
    public int PacketType => 3;

    [Key(0)]
    public long Timestamp { get; set; }

    [Key(1)]
    public int InputTick { get; set; }

    [Key(2)]
    public float MoveX { get; set; }

    [Key(3)]
    public float MoveY { get; set; }

    [Key(4)]
    public bool Jump { get; set; }

    [Key(5)]
    public bool Attack { get; set; }
}

// =============================================
// 패킷 직렬화/역직렬화 유틸리티
// =============================================
public static class PacketSerializer
{
    private static readonly MessagePackSerializerOptions NetworkOptions =
        MessagePackSerializerOptions.Standard
            .WithSecurity(MessagePackSecurity.UntrustedData);

    /// <summary>
    /// 패킷을 바이트 배열로 직렬화
    /// </summary>
    public static byte[] Serialize(INetworkPacket packet)
    {
        packet.Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        return MessagePackSerializer.Serialize(packet, NetworkOptions);
    }

    /// <summary>
    /// 바이트 배열을 패킷으로 역직렬화
    /// </summary>
    public static INetworkPacket Deserialize(byte[] data)
    {
        return MessagePackSerializer.Deserialize<INetworkPacket>(data, NetworkOptions);
    }

    /// <summary>
    /// 패킷을 JSON 문자열로 변환 (디버그용)
    /// </summary>
    public static string ToJson(byte[] data)
    {
        return MessagePackSerializer.ConvertToJson(data, NetworkOptions);
    }
}

// =============================================
// 사용 예시: 게임 네트워크 매니저
// =============================================
public class GameNetworkManager : MonoBehaviour
{
    private Queue<INetworkPacket> _receiveQueue = new Queue<INetworkPacket>();

    public void SendPacket(INetworkPacket packet)
    {
        byte[] data = PacketSerializer.Serialize(packet);
        Debug.Log($"패킷 전송: Type={packet.PacketType}, Size={data.Length} bytes");

        // 실제로는 WebSocket이나 UDP로 전송
        // _transport.Send(data);

        // 디버그용 JSON 확인
        #if UNITY_EDITOR
        Debug.Log($"패킷 내용: {PacketSerializer.ToJson(data)}");
        #endif
    }

    public void OnDataReceived(byte[] data)
    {
        try
        {
            var packet = PacketSerializer.Deserialize(data);

            switch (packet)
            {
                case PongPacket pong:
                    long rtt = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
                               - pong.Timestamp;
                    Debug.Log($"Pong 수신: RTT={rtt}ms");
                    break;

                case GameStatePacket state:
                    Debug.Log($"게임 상태 수신: Tick={state.Tick}, " +
                              $"엔티티 수={state.Entities?.Count ?? 0}");
                    _receiveQueue.Enqueue(state);
                    break;
            }
        }
        catch (MessagePackSerializationException ex)
        {
            Debug.LogError($"패킷 역직렬화 실패: {ex.Message}");
        }
    }
}
```

---

## 11. Typeless (스키마리스) 모드

스키마 없이 동적으로 직렬화가 필요한 경우 Typeless 모드를 사용할 수 있습니다.

```csharp
using UnityEngine;
using MessagePack;
using System.Collections.Generic;

public class TypelessExample : MonoBehaviour
{
    private void Start()
    {
        // =============================================
        // Typeless: 타입 정보를 포함하여 직렬화
        // =============================================
        var mixedData = new Dictionary<string, object>
        {
            { "name", "플레이어1" },
            { "level", 42 },
            { "score", 99999.5 },
            { "items", new List<object> { "검", "방패", "물약" } },
            { "isOnline", true }
        };

        // Typeless 직렬화
        byte[] bytes = MessagePackSerializer.Typeless.Serialize(mixedData);
        Debug.Log($"Typeless 크기: {bytes.Length} bytes");

        // Typeless 역직렬화
        var result = MessagePackSerializer.Typeless.Deserialize(bytes)
            as Dictionary<string, object>;

        if (result != null)
        {
            Debug.Log($"이름: {result["name"]}");
            Debug.Log($"레벨: {result["level"]}");
        }
    }
}
```

> **주의**: Typeless 모드는 타입 정보를 포함하므로 크기가 증가하고, 보안 취약점이 생길 수 있습니다. 가능하면 명시적 타입 직렬화를 사용하세요.

---

## 12. 세이브/로드 시스템 실전 예제

```csharp
using UnityEngine;
using MessagePack;
using System;
using System.IO;
using System.Threading;
using Cysharp.Threading.Tasks;

// =============================================
// MessagePack 기반 게임 세이브 시스템
// =============================================
public class SaveManager : MonoBehaviour
{
    private static SaveManager _instance;
    public static SaveManager Instance => _instance;

    private readonly MessagePackSerializerOptions _saveOptions =
        MessagePackSerializerOptions.Standard
            .WithCompression(MessagePackCompression.Lz4Block);

    private string SaveDirectory =>
        Path.Combine(Application.persistentDataPath, "saves");

    private void Awake()
    {
        if (_instance != null && _instance != this)
        {
            Destroy(gameObject);
            return;
        }
        _instance = this;
        DontDestroyOnLoad(gameObject);

        // 세이브 디렉토리 생성
        if (!Directory.Exists(SaveDirectory))
            Directory.CreateDirectory(SaveDirectory);
    }

    // =============================================
    // 비동기 저장
    // =============================================
    public async UniTask SaveGameAsync(GameSaveData data,
        string slotName = "autosave",
        CancellationToken ct = default)
    {
        string filePath = GetSavePath(slotName);

        try
        {
            data.SavedAt = DateTime.UtcNow;

            byte[] bytes = MessagePackSerializer.Serialize(data, _saveOptions);

            // 원자적 쓰기: 임시 파일에 먼저 쓴 후 교체
            string tempPath = filePath + ".tmp";

            using (var stream = new FileStream(tempPath, FileMode.Create,
                FileAccess.Write, FileShare.None, 4096, useAsync: true))
            {
                await stream.WriteAsync(bytes, 0, bytes.Length, ct);
                await stream.FlushAsync(ct);
            }

            // 기존 파일 교체
            if (File.Exists(filePath))
                File.Delete(filePath);
            File.Move(tempPath, filePath);

            Debug.Log($"게임 저장 완료: {slotName} ({bytes.Length} bytes)");
        }
        catch (OperationCanceledException)
        {
            Debug.Log("저장이 취소되었습니다.");
            throw;
        }
        catch (Exception ex)
        {
            Debug.LogError($"저장 실패: {ex.Message}");
            throw;
        }
    }

    // =============================================
    // 비동기 로드
    // =============================================
    public async UniTask<GameSaveData> LoadGameAsync(
        string slotName = "autosave",
        CancellationToken ct = default)
    {
        string filePath = GetSavePath(slotName);

        if (!File.Exists(filePath))
        {
            Debug.LogWarning($"세이브 파일 없음: {slotName}");
            return null;
        }

        try
        {
            byte[] bytes;
            using (var stream = new FileStream(filePath, FileMode.Open,
                FileAccess.Read, FileShare.Read, 4096, useAsync: true))
            {
                bytes = new byte[stream.Length];
                await stream.ReadAsync(bytes, 0, bytes.Length, ct);
            }

            var data = MessagePackSerializer.Deserialize<GameSaveData>(
                bytes, _saveOptions);

            Debug.Log($"게임 로드 완료: {slotName} " +
                      $"(저장 시각: {data.SavedAt:yyyy-MM-dd HH:mm:ss})");

            return data;
        }
        catch (MessagePackSerializationException ex)
        {
            Debug.LogError($"세이브 데이터 손상: {ex.Message}");
            return null;
        }
    }

    // =============================================
    // 세이브 슬롯 목록 조회
    // =============================================
    public string[] GetSaveSlots()
    {
        if (!Directory.Exists(SaveDirectory))
            return Array.Empty<string>();

        var files = Directory.GetFiles(SaveDirectory, "*.save");
        var slots = new string[files.Length];

        for (int i = 0; i < files.Length; i++)
        {
            slots[i] = Path.GetFileNameWithoutExtension(files[i]);
        }

        return slots;
    }

    // =============================================
    // 세이브 삭제
    // =============================================
    public bool DeleteSave(string slotName)
    {
        string filePath = GetSavePath(slotName);

        if (File.Exists(filePath))
        {
            File.Delete(filePath);
            Debug.Log($"세이브 삭제: {slotName}");
            return true;
        }

        return false;
    }

    private string GetSavePath(string slotName)
    {
        return Path.Combine(SaveDirectory, $"{slotName}.save");
    }
}

// =============================================
// 사용 예시
// =============================================
public class GameController : MonoBehaviour
{
    private CancellationTokenSource _cts;

    private void OnEnable()
    {
        _cts = new CancellationTokenSource();
    }

    private void OnDisable()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }

    public async UniTaskVoid QuickSave()
    {
        var saveData = CollectGameState();
        await SaveManager.Instance.SaveGameAsync(saveData, "quicksave", _cts.Token);
    }

    public async UniTaskVoid QuickLoad()
    {
        var saveData = await SaveManager.Instance.LoadGameAsync("quicksave", _cts.Token);
        if (saveData != null)
        {
            ApplyGameState(saveData);
        }
    }

    private GameSaveData CollectGameState()
    {
        // 현재 게임 상태를 수집하여 GameSaveData 객체 생성
        return new GameSaveData
        {
            Player = new PlayerInfo { Name = "Hero", Level = 10, Hp = 500, MaxHp = 500 },
            Inventory = new System.Collections.Generic.List<InventoryItem>(),
            Achievements = new System.Collections.Generic.Dictionary<string, int>(),
            ActiveQuests = new QuestProgress[0],
            SavedAt = DateTime.UtcNow
        };
    }

    private void ApplyGameState(GameSaveData data)
    {
        Debug.Log($"게임 상태 복원: {data.Player.Name} Lv.{data.Player.Level}");
    }
}
```

---

## 주의사항

### 1. 자주 발생하는 실수

```csharp
// =============================================
// MessagePack 사용 시 주의사항
// =============================================

// ❌ 잘못된 패턴: MessagePackObject 없이 직렬화 시도
public class BadExample
{
    public int Value { get; set; }
}
// MessagePackSerializer.Serialize(new BadExample()); // 런타임 에러!

// ✅ 올바른 패턴: 어트리뷰트 필수
[MessagePackObject]
public class GoodExample
{
    [Key(0)]
    public int Value { get; set; }
}


// ❌ 잘못된 패턴: Key 번호 중복
[MessagePackObject]
public class DuplicateKeyBad
{
    [Key(0)]
    public int A { get; set; }

    [Key(0)]  // 중복! 컴파일은 되지만 런타임 에러
    public int B { get; set; }
}

// ✅ 올바른 패턴: Key 번호 고유하게 유지
[MessagePackObject]
public class DuplicateKeyGood
{
    [Key(0)]
    public int A { get; set; }

    [Key(1)]
    public int B { get; set; }
}


// ❌ 잘못된 패턴: Unity 타입 직접 직렬화 (포매터 미등록)
[MessagePackObject]
public class DirectUnityTypeBad
{
    [Key(0)]
    public Vector3 Position { get; set; }  // 커스텀 포매터 없으면 실패
}

// ✅ 올바른 패턴: 래퍼 타입 또는 커스텀 포매터 사용
[MessagePackObject]
public class DirectUnityTypeGood
{
    [Key(0)]
    public Vector3Serializable Position { get; set; }  // 직렬화 가능한 래퍼
}
```

### 2. 버전 호환성 관리

```csharp
// =============================================
// 정수 Key를 이용한 안전한 스키마 진화
// =============================================

// v1.0 - 초기 버전
[MessagePackObject]
public class PlayerDataV1
{
    [Key(0)]
    public string Name { get; set; }

    [Key(1)]
    public int Level { get; set; }
}

// v2.0 - 필드 추가 (하위 호환)
// ✅ 새 필드는 새로운 Key 번호로 추가
[MessagePackObject]
public class PlayerDataV2
{
    [Key(0)]
    public string Name { get; set; }

    [Key(1)]
    public int Level { get; set; }

    [Key(2)]  // 새로 추가 - v1 데이터 로드 시 default 값
    public int Rank { get; set; }

    [Key(3)]  // 새로 추가
    public float Experience { get; set; }
}

// ❌ 기존 Key 번호를 재배치하면 기존 데이터와 호환 불가
// Key(0)이 Name이었는데 Level로 바꾸면 안 됨!


// ✅ 올바른 필드 제거 방법: Key 번호는 유지하되 IgnoreMember 사용하지 않음
//    (빈 Key 슬롯은 역직렬화 시 무시됨)
[MessagePackObject]
public class PlayerDataV3
{
    [Key(0)]
    public string Name { get; set; }

    [Key(1)]
    public int Level { get; set; }

    // Key(2)는 더 이상 사용하지 않지만 번호는 건너뜀
    // 새로운 필드는 Key(4)부터 사용

    [Key(3)]
    public float Experience { get; set; }

    [Key(4)]  // v3에서 새로 추가
    public string GuildName { get; set; }
}
```

### 3. 스레드 안전성

```csharp
// =============================================
// MessagePackSerializer는 스레드 안전함
// =============================================

// ✅ 여러 스레드에서 동시 직렬화/역직렬화 가능
// MessagePackSerializer의 정적 메서드는 thread-safe

// ✅ 옵션 객체는 불변(immutable)이므로 공유 가능
private static readonly MessagePackSerializerOptions SharedOptions =
    MessagePackSerializerOptions.Standard
        .WithCompression(MessagePackCompression.Lz4Block);

// ❌ 하지만 직렬화 대상 객체 자체는 스레드 안전하지 않음
// 직렬화 중인 객체를 다른 스레드에서 수정하지 말 것
```

---

## 베스트 프랙티스

### 1. 프로젝트 구성

```
Assets/
├── Scripts/
│   ├── Serialization/
│   │   ├── MessagePackConfig.cs        // 전역 설정
│   │   ├── Resolvers/
│   │   │   └── UnityResolver.cs        // Unity 타입 Resolver
│   │   ├── Formatters/
│   │   │   ├── Vector3Formatter.cs     // 커스텀 포매터
│   │   │   ├── QuaternionFormatter.cs
│   │   │   └── ColorFormatter.cs
│   │   └── Generated/
│   │       └── MessagePackGenerated.cs // AOT 코드 (자동 생성)
│   ├── Models/
│   │   ├── PlayerData.cs              // [MessagePackObject]
│   │   ├── GameState.cs
│   │   └── NetworkPackets.cs          // [Union]
│   └── Systems/
│       ├── SaveManager.cs
│       └── NetworkManager.cs
└── Plugins/
    └── MessagePack/                   // DLL 파일
```

### 2. 권장 패턴 모음

```csharp
// =============================================
// ✅ 베스트 프랙티스 모음
// =============================================

// 1. 정수 Key 사용 (성능 우선)
[MessagePackObject]
public class OptimizedData
{
    [Key(0)] public int Id { get; set; }
    [Key(1)] public string Name { get; set; }
}

// 2. 초기화 시점에서 Resolver 한 번만 설정
// [RuntimeInitializeOnLoadMethod] 사용

// 3. LZ4 압축은 세이브/캐시용, 실시간 패킷은 미압축
// 세이브: MessagePackCompression.Lz4Block
// 실시간: MessagePackCompression.None

// 4. 네트워크 수신 데이터는 UntrustedData 모드
// WithSecurity(MessagePackSecurity.UntrustedData)

// 5. IL2CPP 빌드 전 반드시 mpc로 코드 생성

// 6. Key 번호는 절대 재사용하지 않기 (스키마 진화)

// 7. 큰 byte[] 데이터는 ArrayPool 활용
using (var buffer = new System.Buffers.ArrayBufferWriter<byte>())
{
    MessagePackSerializer.Serialize(buffer, data);
    // buffer.WrittenSpan 사용
}

// 8. 디버그 시 ConvertToJson 활용
#if UNITY_EDITOR
byte[] bytes = MessagePackSerializer.Serialize(data);
Debug.Log(MessagePackSerializer.ConvertToJson(bytes));
#endif

// 9. [IgnoreMember]로 런타임 전용 필드 제외
[MessagePackObject]
public class EntityData
{
    [Key(0)] public int Id { get; set; }
    [Key(1)] public string Name { get; set; }

    [IgnoreMember]
    public GameObject CachedObject { get; set; }  // 직렬화 불필요

    [IgnoreMember]
    public bool IsDirty { get; set; }  // 런타임 전용
}

// 10. Nullable 타입 지원
[MessagePackObject]
public class NullableData
{
    [Key(0)] public int? OptionalInt { get; set; }
    [Key(1)] public string OptionalString { get; set; }  // string은 기본 nullable
    [Key(2)] public float? OptionalFloat { get; set; }
}
```

### 3. 성능 최적화 팁

```csharp
// =============================================
// 성능 최적화 체크리스트
// =============================================

// ✅ 정수 Key 사용 (문자열 Key 대비 ~20% 빠름)
// ✅ 불필요한 필드는 [IgnoreMember]로 제외
// ✅ 큰 컬렉션은 배열(T[]) 사용 (List<T> 대비 빠름)
// ✅ 빈번한 직렬화는 byte[] 풀링 고려
// ✅ LZ4 압축은 반복 데이터가 많을 때만 사용
// ✅ struct 타입 활용 (GC 압력 감소)

// ❌ 매 프레임 직렬화/역직렬화는 피할 것
// ❌ Typeless 모드는 성능이 낮으므로 가능하면 사용 자제
// ❌ 지나치게 깊은 중첩 구조는 성능 저하 유발
// ❌ 거대한 단일 객체보다 분할된 작은 객체가 유리
```

---

## 참고 자료

- [MessagePack-CSharp GitHub](https://github.com/MessagePack-CSharp/MessagePack-CSharp) - 공식 리포지토리
- [MessagePack 스펙](https://msgpack.org/) - MessagePack 포맷 공식 사양
- [neuecc 블로그](https://neue.cc/) - 라이브러리 개발자 블로그
- [MessagePack-CSharp Unity 가이드](https://github.com/MessagePack-CSharp/MessagePack-CSharp#unity) - Unity 통합 가이드
- [MessagePack vs JSON 벤치마크](https://github.com/MessagePack-CSharp/MessagePack-CSharp#performance) - 공식 벤치마크
- [MemoryPack](https://github.com/Cysharp/MemoryPack) - 차세대 직렬화 라이브러리 (비교 참고)

---

## 다음 섹션

[31. MemoryPack](./31-memorypack.md)
