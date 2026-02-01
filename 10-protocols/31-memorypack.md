# 31. MemoryPack

## 개요

**MemoryPack**은 neuecc(Yoshifumi Kawai)가 개발한 **Zero-encoding 초고속 바이너리 직렬화 라이브러리**입니다. C# 메모리 레이아웃을 그대로 복사하는 방식으로, 기존 직렬화 라이브러리 대비 압도적인 성능을 제공합니다. Source Generator 기반으로 동작하여 런타임 리플렉션이 불필요하며, Unity IL2CPP/AOT 환경에서도 안전하게 사용할 수 있습니다.

neuecc는 MessagePack for C#, Utf8Json, UniRx, UniTask, R3 등을 만든 Cysharp의 CTO이며, MemoryPack은 그의 4번째 직렬화 라이브러리입니다.

---

## 1. 설치

### NuGet (.NET 프로젝트)

```
dotnet add package MemoryPack
```

### Unity 설치 (NuGetForUnity 사용)

```
// 1. NuGetForUnity 설치
//    Unity Package Manager > Add package from git URL:
//    https://github.com/GlitchEnzo/NuGetForUnity.git?path=/src/NuGetForUnity

// 2. NuGet 메뉴에서 MemoryPack 검색 후 설치
//    Window > NuGet > Manage NuGet Packages
//    "MemoryPack" 검색 > Install
```

### Unity 설치 (Git URL 방식)

```
// Unity Package Manager > Add package from git URL:
https://github.com/Cysharp/MemoryPack.git?path=src/MemoryPack.Unity/Assets/MemoryPack
```

### 최소 요구 사항

| 환경 | 버전 |
|------|------|
| **.NET** | .NET Standard 2.1 이상 (.NET 5, 6, 7, 8) |
| **.NET 최적 성능** | .NET 7 이상 |
| **Unity** | 2021.3 이상 |
| **C#** | 9.0 이상 |

---

## 2. 기본 사용법

### [MemoryPackable] 어트리뷰트

```csharp
using MemoryPack;
using UnityEngine;

// =============================================
// MemoryPackable 기본 사용법
// 클래스에 [MemoryPackable] + partial 키워드 적용
// =============================================

[MemoryPackable]
public partial class PlayerData
{
    public string PlayerName { get; set; }
    public int Level { get; set; }
    public float Health { get; set; }
    public long Score { get; set; }
}

// 구조체도 지원
[MemoryPackable]
public partial struct Vector3Data
{
    public float X { get; set; }
    public float Y { get; set; }
    public float Z { get; set; }
}

// record 타입 지원
[MemoryPackable]
public partial record GameSettings(
    int ResolutionWidth,
    int ResolutionHeight,
    float Volume,
    bool Fullscreen
);

public class MemoryPackBasicExample : MonoBehaviour
{
    private void Start()
    {
        // =============================================
        // Serialize (직렬화)
        // =============================================
        var player = new PlayerData
        {
            PlayerName = "Hero",
            Level = 50,
            Health = 100.0f,
            Score = 999999L
        };

        byte[] serialized = MemoryPackSerializer.Serialize(player);
        Debug.Log($"직렬화 크기: {serialized.Length} bytes");

        // =============================================
        // Deserialize (역직렬화)
        // =============================================
        var deserialized = MemoryPackSerializer.Deserialize<PlayerData>(serialized);
        Debug.Log($"이름: {deserialized.PlayerName}, 레벨: {deserialized.Level}");
    }
}
```

### 핵심 규칙

```csharp
using MemoryPack;

// ✅ 올바른 사용: partial 키워드 필수
[MemoryPackable]
public partial class CorrectClass
{
    public int Value { get; set; }
}

// ❌ 잘못된 사용: partial 누락 - 컴파일 에러 발생
// [MemoryPackable]
// public class WrongClass
// {
//     public int Value { get; set; }
// }

// ✅ 멤버 제어: [MemoryPackIgnore]로 직렬화에서 제외
[MemoryPackable]
public partial class SelectiveSerialization
{
    public int ImportantData { get; set; }        // 직렬화됨

    [MemoryPackIgnore]
    public int CachedValue { get; set; }          // 직렬화에서 제외

    [MemoryPackInclude]
    private int _internalState;                    // private이지만 직렬화에 포함
}
```

---

## 3. 지원 타입

### 기본 지원 타입 목록

```csharp
using MemoryPack;
using System;
using System.Collections.Generic;
using System.Numerics;

// =============================================
// MemoryPack이 지원하는 주요 타입
// =============================================

[MemoryPackable]
public partial class SupportedTypesExample
{
    // --- Primitive 타입 ---
    public bool BoolValue { get; set; }
    public byte ByteValue { get; set; }
    public sbyte SByteValue { get; set; }
    public short ShortValue { get; set; }
    public int IntValue { get; set; }
    public long LongValue { get; set; }
    public float FloatValue { get; set; }
    public double DoubleValue { get; set; }
    public char CharValue { get; set; }
    public decimal DecimalValue { get; set; }

    // --- String ---
    public string StringValue { get; set; }

    // --- Nullable ---
    public int? NullableInt { get; set; }
    public float? NullableFloat { get; set; }
    public DateTime? NullableDateTime { get; set; }

    // --- DateTime 관련 ---
    public DateTime DateTimeValue { get; set; }
    public DateTimeOffset DateTimeOffsetValue { get; set; }
    public TimeSpan TimeSpanValue { get; set; }

    // --- Guid ---
    public Guid GuidValue { get; set; }

    // --- Collection 타입 ---
    public int[] IntArray { get; set; }
    public List<string> StringList { get; set; }
    public Dictionary<string, int> StringIntDict { get; set; }
    public HashSet<int> IntHashSet { get; set; }
    public Queue<string> StringQueue { get; set; }
    public Stack<int> IntStack { get; set; }

    // --- Tuple ---
    public (int, string) TupleValue { get; set; }
    public (int Id, string Name, float Score) NamedTuple { get; set; }

    // --- Enum ---
    public DayOfWeek EnumValue { get; set; }

    // --- 바이트 배열 (효율적 직렬화) ---
    public byte[] RawData { get; set; }
}

// Enum 직렬화
public enum GameState
{
    MainMenu,
    Playing,
    Paused,
    GameOver
}

[MemoryPackable]
public partial class GameStateData
{
    public GameState CurrentState { get; set; }
    public GameState PreviousState { get; set; }
}
```

### Unity 특화 타입 (Unity 환경)

```csharp
using MemoryPack;
using UnityEngine;

// =============================================
// Unity 환경에서 추가로 지원되는 타입
// Unmanaged 타입과 일부 클래스 직렬화 가능
// =============================================

[MemoryPackable]
public partial class UnityTypesData
{
    // --- Unmanaged 구조체 (메모리 직접 복사) ---
    public Vector2 Position2D { get; set; }
    public Vector3 Position3D { get; set; }
    public Vector4 Position4D { get; set; }
    public Quaternion Rotation { get; set; }
    public Color ColorValue { get; set; }
    public Color32 Color32Value { get; set; }
    public Rect RectValue { get; set; }
    public Bounds BoundsValue { get; set; }
    public Vector2Int GridPosition { get; set; }
    public Vector3Int VoxelPosition { get; set; }
    public Matrix4x4 TransformMatrix { get; set; }

    // --- Unity 클래스 타입 ---
    public AnimationCurve Curve { get; set; }
    public Gradient GradientValue { get; set; }
    public RectOffset Padding { get; set; }
}

public class UnityMemoryPackExample : MonoBehaviour
{
    private void Start()
    {
        var data = new UnityTypesData
        {
            Position3D = transform.position,
            Rotation = transform.rotation,
            ColorValue = Color.red,
            Curve = AnimationCurve.EaseInOut(0f, 0f, 1f, 1f)
        };

        byte[] bytes = MemoryPackSerializer.Serialize(data);
        Debug.Log($"Unity 데이터 직렬화 크기: {bytes.Length} bytes");

        var restored = MemoryPackSerializer.Deserialize<UnityTypesData>(bytes);
        Debug.Log($"복원된 위치: {restored.Position3D}");
    }
}
```

---

## 4. Serialize / Deserialize API

### 다양한 직렬화/역직렬화 오버로드

```csharp
using MemoryPack;
using System;
using System.Buffers;
using System.IO;
using System.Threading.Tasks;
using UnityEngine;

public class SerializeApiExample : MonoBehaviour
{
    // =============================================
    // 1. 기본 byte[] 직렬화/역직렬화
    // =============================================
    private void BasicSerialize()
    {
        var data = new PlayerData { PlayerName = "Alice", Level = 10 };

        // Serialize -> byte[]
        byte[] bytes = MemoryPackSerializer.Serialize(data);

        // Deserialize <- byte[]
        PlayerData result = MemoryPackSerializer.Deserialize<PlayerData>(bytes);

        // ReadOnlySpan<byte>에서 역직렬화
        ReadOnlySpan<byte> span = bytes.AsSpan();
        PlayerData fromSpan = MemoryPackSerializer.Deserialize<PlayerData>(span);
    }

    // =============================================
    // 2. IBufferWriter<byte> 직렬화 (Zero-copy 지향)
    // =============================================
    private void BufferWriterSerialize()
    {
        var data = new PlayerData { PlayerName = "Bob", Level = 20 };

        // ArrayBufferWriter를 사용한 직렬화
        var bufferWriter = new ArrayBufferWriter<byte>();
        MemoryPackSerializer.Serialize(bufferWriter, data);

        // 결과 읽기
        ReadOnlySpan<byte> written = bufferWriter.WrittenSpan;
        Debug.Log($"버퍼에 기록된 크기: {bufferWriter.WrittenCount} bytes");
    }

    // =============================================
    // 3. Stream 기반 비동기 직렬화/역직렬화
    // =============================================
    private async Task StreamSerializeAsync()
    {
        var data = new PlayerData { PlayerName = "Charlie", Level = 30 };

        // Stream에 직렬화
        using var stream = new MemoryStream();
        await MemoryPackSerializer.SerializeAsync(stream, data);

        // Stream에서 역직렬화
        stream.Position = 0;
        PlayerData result = await MemoryPackSerializer.DeserializeAsync<PlayerData>(stream);

        Debug.Log($"스트림 역직렬화: {result.PlayerName}");
    }

    // =============================================
    // 4. 기존 객체에 덮어쓰기 (Overwrite Deserialize)
    // =============================================
    private void OverwriteDeserialize()
    {
        var data = new PlayerData { PlayerName = "Dave", Level = 40 };
        byte[] bytes = MemoryPackSerializer.Serialize(data);

        // 기존 인스턴스를 재활용하여 역직렬화 (GC 감소)
        var existing = new PlayerData();
        MemoryPackSerializer.Deserialize(bytes, ref existing);

        Debug.Log($"덮어쓰기 역직렬화: {existing.PlayerName}, Lv.{existing.Level}");
    }

    // =============================================
    // 5. 파일 저장/로드 실전 예제
    // =============================================
    public async Task SaveToFileAsync(PlayerData data, string filePath)
    {
        using var fileStream = new FileStream(
            filePath, FileMode.Create, FileAccess.Write, FileShare.None,
            bufferSize: 4096, useAsync: true);

        await MemoryPackSerializer.SerializeAsync(fileStream, data);
        Debug.Log($"저장 완료: {filePath}");
    }

    public async Task<PlayerData> LoadFromFileAsync(string filePath)
    {
        using var fileStream = new FileStream(
            filePath, FileMode.Open, FileAccess.Read, FileShare.Read,
            bufferSize: 4096, useAsync: true);

        var data = await MemoryPackSerializer.DeserializeAsync<PlayerData>(fileStream);
        Debug.Log($"로드 완료: {data.PlayerName}");
        return data;
    }
}
```

### 컬렉션 직렬화

```csharp
using MemoryPack;
using System.Collections.Generic;
using UnityEngine;

public class CollectionSerializeExample : MonoBehaviour
{
    [MemoryPackable]
    public partial class Inventory
    {
        public List<ItemData> Items { get; set; } = new();
        public Dictionary<string, int> ResourceCounts { get; set; } = new();
    }

    [MemoryPackable]
    public partial class ItemData
    {
        public string ItemId { get; set; }
        public string Name { get; set; }
        public int Quantity { get; set; }
        public float Weight { get; set; }
    }

    private void Start()
    {
        var inventory = new Inventory
        {
            Items = new List<ItemData>
            {
                new() { ItemId = "sword_01", Name = "불꽃 검", Quantity = 1, Weight = 3.5f },
                new() { ItemId = "potion_hp", Name = "체력 포션", Quantity = 99, Weight = 0.1f },
                new() { ItemId = "shield_02", Name = "강철 방패", Quantity = 1, Weight = 8.0f }
            },
            ResourceCounts = new Dictionary<string, int>
            {
                ["gold"] = 15000,
                ["gem"] = 250,
                ["wood"] = 500
            }
        };

        // 직렬화
        byte[] bytes = MemoryPackSerializer.Serialize(inventory);
        Debug.Log($"인벤토리 직렬화: {bytes.Length} bytes");

        // 역직렬화
        var restored = MemoryPackSerializer.Deserialize<Inventory>(bytes);
        Debug.Log($"아이템 수: {restored.Items.Count}");
        Debug.Log($"골드: {restored.ResourceCounts["gold"]}");
    }
}
```

---

## 5. Versioning (버전 관리)

### 기본 버전 관리 (GenerateType.Object)

```csharp
using MemoryPack;

// =============================================
// 기본 모드: GenerateType.Object (기본값)
// - 멤버를 끝에 추가하는 것만 허용
// - 멤버 삭제, 순서 변경 불가
// - 가장 빠른 직렬화 성능
// =============================================

// Version 1
[MemoryPackable]
public partial class SaveDataV1
{
    public string PlayerName { get; set; }
    public int Level { get; set; }
}

// Version 2: 끝에 멤버 추가 (OK)
[MemoryPackable]
public partial class SaveDataV2
{
    public string PlayerName { get; set; }    // 기존 멤버 유지
    public int Level { get; set; }            // 기존 멤버 유지
    public float PlayTime { get; set; }       // ✅ 끝에 추가 가능
}

// ❌ 잘못된 버전 관리: 중간에 멤버 삽입
// [MemoryPackable]
// public partial class SaveDataWrong
// {
//     public string PlayerName { get; set; }
//     public float NewField { get; set; }    // ❌ 중간에 삽입 불가
//     public int Level { get; set; }
// }
```

### 완전한 버전 관리 (GenerateType.VersionTolerant)

```csharp
using MemoryPack;

// =============================================
// VersionTolerant 모드:
// - 멤버 추가/삭제 가능
// - 순서 번호 재사용 불가
// - [MemoryPackOrder] 필수
// - 기본 모드보다 약간 느리고 페이로드가 큼
// =============================================

// Version 1
[MemoryPackable(GenerateType.VersionTolerant)]
public partial class ProfileV1
{
    [MemoryPackOrder(0)]
    public string UserName { get; set; }

    [MemoryPackOrder(1)]
    public int Age { get; set; }

    [MemoryPackOrder(2)]
    public string Email { get; set; }
}

// Version 2: Age 삭제, Nickname 추가
[MemoryPackable(GenerateType.VersionTolerant)]
public partial class ProfileV2
{
    [MemoryPackOrder(0)]
    public string UserName { get; set; }      // 유지

    // Order(1) - Age 삭제됨 (번호 건너뜀)

    [MemoryPackOrder(2)]
    public string Email { get; set; }          // 유지

    [MemoryPackOrder(3)]
    public string Nickname { get; set; }       // ✅ 새 번호로 추가
}

// ❌ 잘못된 VersionTolerant 사용
// [MemoryPackable(GenerateType.VersionTolerant)]
// public partial class ProfileWrong
// {
//     [MemoryPackOrder(0)]
//     public string UserName { get; set; }
//
//     [MemoryPackOrder(1)]                    // ❌ 삭제된 번호 재사용 불가
//     public string Nickname { get; set; }
// }
```

### 실전 버전 관리 예제

```csharp
using MemoryPack;
using UnityEngine;

// =============================================
// 게임 세이브 데이터 버전 관리 실전 패턴
// =============================================

[MemoryPackable(GenerateType.VersionTolerant)]
public partial class GameSaveData
{
    // 초기 버전 (v1.0) 멤버들
    [MemoryPackOrder(0)]
    public int SaveVersion { get; set; } = 3;

    [MemoryPackOrder(1)]
    public string PlayerName { get; set; }

    [MemoryPackOrder(2)]
    public int Level { get; set; }

    [MemoryPackOrder(3)]
    public float PlayTimeHours { get; set; }

    // v1.1에서 추가
    [MemoryPackOrder(4)]
    public int AchievementCount { get; set; }

    // v1.2에서 추가
    [MemoryPackOrder(5)]
    public string LastSaveLocation { get; set; }

    // v1.3에서 추가
    [MemoryPackOrder(6)]
    public long LastSaveTimestamp { get; set; }
}

public class SaveManager : MonoBehaviour
{
    private const string SaveFileName = "gamesave.bin";

    public void SaveGame(GameSaveData data)
    {
        string path = System.IO.Path.Combine(
            Application.persistentDataPath, SaveFileName);

        byte[] bytes = MemoryPackSerializer.Serialize(data);
        System.IO.File.WriteAllBytes(path, bytes);

        Debug.Log($"게임 저장 완료 (v{data.SaveVersion}): {bytes.Length} bytes");
    }

    public GameSaveData LoadGame()
    {
        string path = System.IO.Path.Combine(
            Application.persistentDataPath, SaveFileName);

        if (!System.IO.File.Exists(path))
        {
            Debug.Log("세이브 파일 없음, 새 데이터 생성");
            return new GameSaveData();
        }

        byte[] bytes = System.IO.File.ReadAllBytes(path);
        var data = MemoryPackSerializer.Deserialize<GameSaveData>(bytes);

        // 이전 버전 데이터의 누락 필드는 default 값으로 설정됨
        Debug.Log($"게임 로드 완료 (v{data.SaveVersion})");
        return data;
    }
}
```

---

## 6. Union (다형성 직렬화)

### 인터페이스 기반 Union

```csharp
using MemoryPack;
using UnityEngine;

// =============================================
// Union: 인터페이스/추상 클래스를 통한 다형성 직렬화
// 태그 번호(0~65535)로 구체 타입을 식별
// =============================================

[MemoryPackable]
[MemoryPackUnion(0, typeof(MoveCommand))]
[MemoryPackUnion(1, typeof(AttackCommand))]
[MemoryPackUnion(2, typeof(SkillCommand))]
[MemoryPackUnion(3, typeof(ItemCommand))]
public partial interface IGameCommand
{
    float Timestamp { get; set; }
}

[MemoryPackable]
public partial class MoveCommand : IGameCommand
{
    public float Timestamp { get; set; }
    public float TargetX { get; set; }
    public float TargetY { get; set; }
    public float TargetZ { get; set; }
}

[MemoryPackable]
public partial class AttackCommand : IGameCommand
{
    public float Timestamp { get; set; }
    public int TargetEntityId { get; set; }
    public int SkillSlot { get; set; }
}

[MemoryPackable]
public partial class SkillCommand : IGameCommand
{
    public float Timestamp { get; set; }
    public int SkillId { get; set; }
    public float DirectionX { get; set; }
    public float DirectionY { get; set; }
}

[MemoryPackable]
public partial class ItemCommand : IGameCommand
{
    public float Timestamp { get; set; }
    public string ItemId { get; set; }
    public int TargetSlot { get; set; }
}
```

### Union 직렬화/역직렬화

```csharp
using MemoryPack;
using System.Collections.Generic;
using UnityEngine;

public class UnionExample : MonoBehaviour
{
    private void Start()
    {
        // 다양한 커맨드를 인터페이스 타입으로 직렬화
        IGameCommand moveCmd = new MoveCommand
        {
            Timestamp = Time.time,
            TargetX = 10f,
            TargetY = 0f,
            TargetZ = 5f
        };

        IGameCommand attackCmd = new AttackCommand
        {
            Timestamp = Time.time,
            TargetEntityId = 42,
            SkillSlot = 1
        };

        // ✅ 인터페이스 타입으로 직렬화
        byte[] moveBytes = MemoryPackSerializer.Serialize<IGameCommand>(moveCmd);
        byte[] attackBytes = MemoryPackSerializer.Serialize<IGameCommand>(attackCmd);

        // ✅ 인터페이스 타입으로 역직렬화 -> 구체 타입 복원
        IGameCommand restoredMove = MemoryPackSerializer.Deserialize<IGameCommand>(moveBytes);
        IGameCommand restoredAttack = MemoryPackSerializer.Deserialize<IGameCommand>(attackBytes);

        // 패턴 매칭으로 타입 확인
        switch (restoredMove)
        {
            case MoveCommand move:
                Debug.Log($"이동 명령: ({move.TargetX}, {move.TargetY}, {move.TargetZ})");
                break;
            case AttackCommand attack:
                Debug.Log($"공격 명령: 대상 {attack.TargetEntityId}");
                break;
            case SkillCommand skill:
                Debug.Log($"스킬 명령: ID {skill.SkillId}");
                break;
            case ItemCommand item:
                Debug.Log($"아이템 명령: {item.ItemId}");
                break;
        }
    }
}
```

### 추상 클래스 기반 Union

```csharp
using MemoryPack;
using UnityEngine;

// =============================================
// 추상 클래스에도 Union 적용 가능
// =============================================

[MemoryPackable]
[MemoryPackUnion(0, typeof(DamageEffect))]
[MemoryPackUnion(1, typeof(HealEffect))]
[MemoryPackUnion(2, typeof(BuffEffect))]
public abstract partial class GameEffect
{
    public int TargetId { get; set; }
    public float Duration { get; set; }
}

[MemoryPackable]
public partial class DamageEffect : GameEffect
{
    public int DamageAmount { get; set; }
    public string DamageType { get; set; }   // "physical", "magical", "true"
}

[MemoryPackable]
public partial class HealEffect : GameEffect
{
    public int HealAmount { get; set; }
    public bool IsOverTime { get; set; }
}

[MemoryPackable]
public partial class BuffEffect : GameEffect
{
    public string StatName { get; set; }
    public float Multiplier { get; set; }
}

public class EffectSystem : MonoBehaviour
{
    public void ApplyEffect(GameEffect effect)
    {
        // 직렬화 (네트워크 전송 등)
        byte[] bytes = MemoryPackSerializer.Serialize(effect);

        // 역직렬화
        GameEffect restored = MemoryPackSerializer.Deserialize<GameEffect>(bytes);

        switch (restored)
        {
            case DamageEffect dmg:
                Debug.Log($"피해 {dmg.DamageAmount} ({dmg.DamageType})");
                break;
            case HealEffect heal:
                Debug.Log($"회복 {heal.HealAmount}");
                break;
            case BuffEffect buff:
                Debug.Log($"버프 {buff.StatName} x{buff.Multiplier}");
                break;
        }
    }
}
```

---

## 7. 직렬화 콜백과 커스텀 로직

### 직렬화 콜백

```csharp
using MemoryPack;
using System;
using UnityEngine;

// =============================================
// 직렬화/역직렬화 전후에 커스텀 로직 실행
// =============================================

[MemoryPackable]
public partial class CharacterState
{
    public string Name { get; set; }
    public int CurrentHp { get; set; }
    public int MaxHp { get; set; }

    [MemoryPackIgnore]
    public float HpPercentage { get; private set; }

    [MemoryPackIgnore]
    public DateTime LastLoadTime { get; private set; }

    // 직렬화 전 호출
    [MemoryPackOnSerializing]
    static partial void OnSerializing(ref CharacterState state)
    {
        // 직렬화 전에 데이터 정합성 보장
        if (state.CurrentHp > state.MaxHp)
            state.CurrentHp = state.MaxHp;
    }

    // 역직렬화 후 호출
    [MemoryPackOnDeserialized]
    static partial void OnDeserialized(ref CharacterState state)
    {
        // 역직렬화 후 계산 필드 복원
        state.HpPercentage = state.MaxHp > 0
            ? (float)state.CurrentHp / state.MaxHp
            : 0f;
        state.LastLoadTime = DateTime.Now;
    }
}
```

### 커스텀 생성자

```csharp
using MemoryPack;

// =============================================
// 여러 생성자가 있을 때 [MemoryPackConstructor] 지정
// 매개변수 이름은 프로퍼티 이름과 일치해야 함 (대소문자 무시)
// =============================================

[MemoryPackable]
public partial class EnemyData
{
    public string EnemyId { get; set; }
    public string EnemyType { get; set; }
    public int Health { get; set; }
    public float Speed { get; set; }

    // 기본 생성자
    public EnemyData() { }

    // 편의 생성자
    public EnemyData(string enemyType, int health)
    {
        EnemyId = Guid.NewGuid().ToString();
        EnemyType = enemyType;
        Health = health;
        Speed = 1.0f;
    }

    // MemoryPack이 사용할 생성자
    [MemoryPackConstructor]
    public EnemyData(string enemyId, string enemyType, int health, float speed)
    {
        EnemyId = enemyId;
        EnemyType = enemyType;
        Health = health;
        Speed = speed;
    }
}
```

---

## 8. Source Generator 동작 원리

### 왜 Source Generator인가

```csharp
// =============================================
// Source Generator vs 런타임 리플렉션 비교
// =============================================

// ❌ 전통적 직렬화 (런타임 리플렉션)
// - 실행 시점에 타입 정보를 분석
// - IL.Emit 또는 Expression Tree로 동적 코드 생성
// - IL2CPP/AOT에서 동적 코드 생성 불가 -> 크래시
// - 리플렉션 오버헤드 발생

// ✅ MemoryPack (Source Generator)
// - 컴파일 시점에 직렬화 코드 자동 생성
// - 런타임 리플렉션 완전 불필요
// - IL2CPP/AOT에서 100% 안전
// - 제로 오버헤드 (직접 작성한 코드와 동일 성능)
```

### 생성되는 코드 구조

```csharp
// [MemoryPackable]을 적용하면 컴파일 시점에
// 아래와 같은 코드가 자동으로 생성됩니다 (개념적 예시)

// 원본 코드
[MemoryPackable]
public partial class PlayerData
{
    public string PlayerName { get; set; }
    public int Level { get; set; }
}

// Source Generator가 생성하는 코드 (개념적 예시)
// 실제 생성 코드는 더 복잡하며, MemoryPack 내부 API를 사용
partial class PlayerData : IMemoryPackable<PlayerData>
{
    static void IMemoryPackable<PlayerData>.Serialize<TBufferWriter>(
        ref MemoryPackWriter<TBufferWriter> writer,
        scoped ref PlayerData? value)
    {
        if (value == null)
        {
            writer.WriteNullObjectHeader();
            return;
        }
        writer.WriteObjectHeader(2);           // 멤버 수
        writer.WriteString(value.PlayerName);   // 멤버 직접 기록
        writer.WriteUnmanaged(value.Level);     // unmanaged 타입은 메모리 복사
    }

    static void IMemoryPackable<PlayerData>.Deserialize(
        ref MemoryPackReader reader,
        scoped ref PlayerData? value)
    {
        // ... 역직렬화 코드 자동 생성
    }
}
```

---

## 9. 성능 비교

### 벤치마크 개요

| 직렬화 라이브러리 | 표준 객체 성능 | struct 배열 성능 | 인코딩 방식 |
|-------------------|---------------|-----------------|-------------|
| **MemoryPack** | **기준 (1x)** | **기준 (1x)** | Zero-encoding |
| MessagePack for C# | ~2~5x 느림 | ~50x 느림 | MessagePack |
| System.Text.Json | ~10x 느림 | ~100x+ 느림 | JSON |
| protobuf-net | ~3~5x 느림 | ~50x+ 느림 | Protocol Buffers |

### 성능 차이의 원리

```csharp
using MemoryPack;
using System;
using UnityEngine;
using System.Diagnostics;

// =============================================
// MemoryPack이 빠른 이유: Zero-encoding
//
// 전통적 직렬화: 객체 -> 인코딩 -> 바이트 (변환 비용)
// MemoryPack:     객체 -> 메모리 직접 복사 (변환 비용 거의 없음)
//
// 특히 unmanaged struct 배열은 메모리 블록을 통째로 복사
// =============================================

[MemoryPackable]
public partial struct TransformSnapshot
{
    public float PosX, PosY, PosZ;
    public float RotX, RotY, RotZ, RotW;
    public float ScaleX, ScaleY, ScaleZ;
}

public class PerformanceComparisonExample : MonoBehaviour
{
    private void Start()
    {
        // struct 배열 벤치마크
        const int count = 10000;
        var snapshots = new TransformSnapshot[count];
        for (int i = 0; i < count; i++)
        {
            snapshots[i] = new TransformSnapshot
            {
                PosX = i * 0.1f, PosY = i * 0.2f, PosZ = i * 0.3f,
                RotX = 0, RotY = 0, RotZ = 0, RotW = 1,
                ScaleX = 1, ScaleY = 1, ScaleZ = 1
            };
        }

        // MemoryPack: unmanaged struct 배열은 메모리 블록 복사
        var sw = Stopwatch.StartNew();
        byte[] packed = MemoryPackSerializer.Serialize(snapshots);
        sw.Stop();

        UnityEngine.Debug.Log($"MemoryPack 직렬화: {sw.Elapsed.TotalMilliseconds:F3}ms");
        UnityEngine.Debug.Log($"크기: {packed.Length} bytes ({packed.Length / 1024}KB)");

        // 역직렬화
        sw.Restart();
        var restored = MemoryPackSerializer.Deserialize<TransformSnapshot[]>(packed);
        sw.Stop();

        UnityEngine.Debug.Log($"MemoryPack 역직렬화: {sw.Elapsed.TotalMilliseconds:F3}ms");
        UnityEngine.Debug.Log($"복원된 요소 수: {restored.Length}");
    }
}
```

### Unity에서 JsonUtility와 비교

```csharp
using MemoryPack;
using System.Diagnostics;
using UnityEngine;

// =============================================
// Unity 환경: MemoryPack vs JsonUtility
// MemoryPack은 JsonUtility 대비 약 3~10배 빠름
// =============================================

[System.Serializable]
[MemoryPackable]
public partial class BenchmarkData
{
    public string Name;
    public int Value;
    public float Score;
    public bool Active;
}

public class UnityBenchmarkExample : MonoBehaviour
{
    private void Start()
    {
        var data = new BenchmarkData
        {
            Name = "TestPlayer",
            Value = 42,
            Score = 99.5f,
            Active = true
        };

        const int iterations = 10000;

        // --- JsonUtility 벤치마크 ---
        var sw = Stopwatch.StartNew();
        string json = "";
        for (int i = 0; i < iterations; i++)
        {
            json = JsonUtility.ToJson(data);
        }
        sw.Stop();
        var jsonSerTime = sw.Elapsed.TotalMilliseconds;

        sw.Restart();
        for (int i = 0; i < iterations; i++)
        {
            JsonUtility.FromJson<BenchmarkData>(json);
        }
        sw.Stop();
        var jsonDeserTime = sw.Elapsed.TotalMilliseconds;

        // --- MemoryPack 벤치마크 ---
        sw.Restart();
        byte[] bytes = null;
        for (int i = 0; i < iterations; i++)
        {
            bytes = MemoryPackSerializer.Serialize(data);
        }
        sw.Stop();
        var mpSerTime = sw.Elapsed.TotalMilliseconds;

        sw.Restart();
        for (int i = 0; i < iterations; i++)
        {
            MemoryPackSerializer.Deserialize<BenchmarkData>(bytes);
        }
        sw.Stop();
        var mpDeserTime = sw.Elapsed.TotalMilliseconds;

        Debug.Log($"=== {iterations}회 반복 벤치마크 ===");
        Debug.Log($"JsonUtility - 직렬화: {jsonSerTime:F2}ms, 역직렬화: {jsonDeserTime:F2}ms");
        Debug.Log($"MemoryPack  - 직렬화: {mpSerTime:F2}ms, 역직렬화: {mpDeserTime:F2}ms");
        Debug.Log($"JSON 크기: {System.Text.Encoding.UTF8.GetByteCount(json)} bytes");
        Debug.Log($"MemoryPack 크기: {bytes.Length} bytes");
    }
}
```

---

## 10. Unity 통합 실전 패턴

### 세이브/로드 시스템

```csharp
using MemoryPack;
using System;
using System.Collections.Generic;
using System.IO;
using System.Threading.Tasks;
using UnityEngine;

// =============================================
// MemoryPack 기반 Unity 세이브 시스템
// =============================================

[MemoryPackable(GenerateType.VersionTolerant)]
public partial class CompleteGameSave
{
    [MemoryPackOrder(0)]
    public int Version { get; set; } = 1;

    [MemoryPackOrder(1)]
    public PlayerSaveData Player { get; set; }

    [MemoryPackOrder(2)]
    public WorldSaveData World { get; set; }

    [MemoryPackOrder(3)]
    public SettingsSaveData Settings { get; set; }

    [MemoryPackOrder(4)]
    public long SaveTimestamp { get; set; }
}

[MemoryPackable]
public partial class PlayerSaveData
{
    public string Name { get; set; }
    public int Level { get; set; }
    public int Experience { get; set; }
    public float PositionX { get; set; }
    public float PositionY { get; set; }
    public float PositionZ { get; set; }
    public List<string> CompletedQuests { get; set; } = new();
    public Dictionary<string, int> Inventory { get; set; } = new();
}

[MemoryPackable]
public partial class WorldSaveData
{
    public string CurrentScene { get; set; }
    public List<string> UnlockedAreas { get; set; } = new();
    public Dictionary<string, bool> Flags { get; set; } = new();
}

[MemoryPackable]
public partial class SettingsSaveData
{
    public float MasterVolume { get; set; } = 1.0f;
    public float MusicVolume { get; set; } = 0.8f;
    public float SfxVolume { get; set; } = 1.0f;
    public int QualityLevel { get; set; } = 2;
    public bool VSync { get; set; } = true;
}

public class MemoryPackSaveSystem : MonoBehaviour
{
    private string SaveDirectory => Path.Combine(
        Application.persistentDataPath, "saves");

    private void Awake()
    {
        if (!Directory.Exists(SaveDirectory))
            Directory.CreateDirectory(SaveDirectory);
    }

    // ✅ 동기 저장 (작은 데이터)
    public void Save(CompleteGameSave data, string slotName = "slot1")
    {
        data.SaveTimestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds();

        byte[] bytes = MemoryPackSerializer.Serialize(data);
        string path = Path.Combine(SaveDirectory, $"{slotName}.sav");
        File.WriteAllBytes(path, bytes);

        Debug.Log($"저장 완료: {path} ({bytes.Length} bytes)");
    }

    // ✅ 비동기 저장 (큰 데이터)
    public async Task SaveAsync(CompleteGameSave data, string slotName = "slot1")
    {
        data.SaveTimestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds();

        string path = Path.Combine(SaveDirectory, $"{slotName}.sav");
        using var stream = new FileStream(
            path, FileMode.Create, FileAccess.Write, FileShare.None,
            bufferSize: 4096, useAsync: true);

        await MemoryPackSerializer.SerializeAsync(stream, data);
        Debug.Log($"비동기 저장 완료: {path}");
    }

    // ✅ 동기 로드
    public CompleteGameSave Load(string slotName = "slot1")
    {
        string path = Path.Combine(SaveDirectory, $"{slotName}.sav");
        if (!File.Exists(path))
        {
            Debug.LogWarning("세이브 파일이 없습니다");
            return null;
        }

        byte[] bytes = File.ReadAllBytes(path);
        var data = MemoryPackSerializer.Deserialize<CompleteGameSave>(bytes);

        Debug.Log($"로드 완료: 플레이어 {data.Player?.Name}, Lv.{data.Player?.Level}");
        return data;
    }

    // ✅ 비동기 로드
    public async Task<CompleteGameSave> LoadAsync(string slotName = "slot1")
    {
        string path = Path.Combine(SaveDirectory, $"{slotName}.sav");
        if (!File.Exists(path)) return null;

        using var stream = new FileStream(
            path, FileMode.Open, FileAccess.Read, FileShare.Read,
            bufferSize: 4096, useAsync: true);

        return await MemoryPackSerializer.DeserializeAsync<CompleteGameSave>(stream);
    }

    // 세이브 슬롯 목록 조회
    public string[] GetSaveSlots()
    {
        if (!Directory.Exists(SaveDirectory))
            return Array.Empty<string>();

        string[] files = Directory.GetFiles(SaveDirectory, "*.sav");
        string[] slots = new string[files.Length];
        for (int i = 0; i < files.Length; i++)
            slots[i] = Path.GetFileNameWithoutExtension(files[i]);
        return slots;
    }
}
```

### 네트워크 메시지 시스템

```csharp
using MemoryPack;
using System;
using System.Collections.Generic;
using UnityEngine;

// =============================================
// MemoryPack 기반 네트워크 메시지 프로토콜
// =============================================

[MemoryPackable]
[MemoryPackUnion(0, typeof(PlayerMoveMessage))]
[MemoryPackUnion(1, typeof(PlayerActionMessage))]
[MemoryPackUnion(2, typeof(ChatMessage))]
[MemoryPackUnion(3, typeof(SyncStateMessage))]
public partial interface INetworkMessage
{
    int SenderId { get; set; }
    long Timestamp { get; set; }
}

[MemoryPackable]
public partial class PlayerMoveMessage : INetworkMessage
{
    public int SenderId { get; set; }
    public long Timestamp { get; set; }
    public float X { get; set; }
    public float Y { get; set; }
    public float Z { get; set; }
    public float Yaw { get; set; }
}

[MemoryPackable]
public partial class PlayerActionMessage : INetworkMessage
{
    public int SenderId { get; set; }
    public long Timestamp { get; set; }
    public int ActionId { get; set; }
    public int TargetId { get; set; }
}

[MemoryPackable]
public partial class ChatMessage : INetworkMessage
{
    public int SenderId { get; set; }
    public long Timestamp { get; set; }
    public string Content { get; set; }
    public int Channel { get; set; }
}

[MemoryPackable]
public partial class SyncStateMessage : INetworkMessage
{
    public int SenderId { get; set; }
    public long Timestamp { get; set; }
    public Dictionary<int, float> EntityHealthMap { get; set; }
    public List<int> ActiveEffectIds { get; set; }
}

public class NetworkProtocol : MonoBehaviour
{
    // 메시지 직렬화 (전송용)
    public byte[] PackMessage(INetworkMessage message)
    {
        return MemoryPackSerializer.Serialize<INetworkMessage>(message);
    }

    // 메시지 역직렬화 (수신용)
    public INetworkMessage UnpackMessage(byte[] data)
    {
        return MemoryPackSerializer.Deserialize<INetworkMessage>(data);
    }

    // 사용 예시
    private void Example()
    {
        // 전송
        var moveMsg = new PlayerMoveMessage
        {
            SenderId = 1,
            Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
            X = 10f, Y = 0f, Z = 5f, Yaw = 90f
        };

        byte[] packet = PackMessage(moveMsg);
        Debug.Log($"패킷 크기: {packet.Length} bytes");  // 매우 작은 크기

        // 수신
        INetworkMessage received = UnpackMessage(packet);
        if (received is PlayerMoveMessage move)
        {
            Debug.Log($"플레이어 {move.SenderId} 이동: ({move.X}, {move.Y}, {move.Z})");
        }
    }
}
```

---

## 11. IL2CPP / AOT 호환성

### Source Generator 기반 AOT 안전성

```csharp
// =============================================
// MemoryPack의 IL2CPP/AOT 호환성
//
// Source Generator가 컴파일 시점에 모든 직렬화 코드를 생성하므로:
// - IL.Emit (동적 코드 생성) 불필요
// - 런타임 리플렉션 불필요
// - 제네릭 가상 메서드 문제 없음 (.NET 8+에서 수정됨)
//
// 결과: iOS, Android, WebGL, 콘솔 등 모든 AOT 플랫폼에서 안전
// =============================================

// ✅ AOT-Safe: Source Generator가 코드 생성
[MemoryPackable]
public partial class AotSafeData
{
    public int Id { get; set; }
    public string Name { get; set; }
    public List<int> Scores { get; set; }
}

// ✅ AOT-Safe: Union도 Source Generator가 처리
[MemoryPackable]
[MemoryPackUnion(0, typeof(ConcreteA))]
[MemoryPackUnion(1, typeof(ConcreteB))]
public partial interface IAotSafeInterface { }

[MemoryPackable]
public partial class ConcreteA : IAotSafeInterface
{
    public int Value { get; set; }
}

[MemoryPackable]
public partial class ConcreteB : IAotSafeInterface
{
    public string Text { get; set; }
}
```

### IL2CPP 주의사항

```csharp
using MemoryPack;
using System.Collections.Generic;
using UnityEngine;

// =============================================
// IL2CPP 환경에서의 주의사항
// =============================================

public class IL2CPPNotes : MonoBehaviour
{
    // ✅ 정상 작동: 기본 타입과 컬렉션
    [MemoryPackable]
    public partial class BasicIL2CPPData
    {
        public int Value { get; set; }
        public string Text { get; set; }
        public List<int> Numbers { get; set; }
        public Dictionary<string, int> Map { get; set; }
    }

    // ✅ Unity 버전에서는 CustomFormatter 미지원
    //    대신 [MemoryPackable]로 직접 타입을 감싸기

    // ❌ Unity 환경에서 미지원
    //    - ImmutableCollections
    //    - CustomFormatter

    // ✅ Union의 ModuleInitializer는 Unity에서 미지원
    //    -> 크로스 어셈블리 Union은 수동 등록 필요

    private void Start()
    {
        // 기본 사용은 모든 플랫폼에서 동일
        var data = new BasicIL2CPPData
        {
            Value = 100,
            Text = "IL2CPP Safe",
            Numbers = new List<int> { 1, 2, 3 },
            Map = new Dictionary<string, int> { ["key"] = 42 }
        };

        byte[] bytes = MemoryPackSerializer.Serialize(data);
        var restored = MemoryPackSerializer.Deserialize<BasicIL2CPPData>(bytes);
        Debug.Log($"IL2CPP 테스트: {restored.Text}");
    }
}
```

---

## 12. 고급 패턴

### 압축과 함께 사용

```csharp
using MemoryPack;
using System.IO;
using System.IO.Compression;
using UnityEngine;

// =============================================
// MemoryPack + Brotli 압축
// 네트워크 전송이나 대용량 세이브에 유용
// =============================================

public static class CompressedMemoryPack
{
    // 압축 직렬화
    public static byte[] SerializeCompressed<T>(T value)
    {
        byte[] raw = MemoryPackSerializer.Serialize(value);

        using var output = new MemoryStream();
        using (var brotli = new BrotliStream(output, CompressionLevel.Fastest))
        {
            brotli.Write(raw, 0, raw.Length);
        }

        byte[] compressed = output.ToArray();
        Debug.Log($"원본: {raw.Length} bytes -> 압축: {compressed.Length} bytes " +
                  $"({(float)compressed.Length / raw.Length:P1})");
        return compressed;
    }

    // 압축 해제 역직렬화
    public static T DeserializeCompressed<T>(byte[] compressed)
    {
        using var input = new MemoryStream(compressed);
        using var brotli = new BrotliStream(input, CompressionMode.Decompress);
        using var output = new MemoryStream();

        brotli.CopyTo(output);
        return MemoryPackSerializer.Deserialize<T>(output.ToArray());
    }
}

// 사용 예시
public class CompressionExample : MonoBehaviour
{
    private void Start()
    {
        var largeData = new PlayerSaveData
        {
            Name = "TestPlayer",
            Level = 99,
            Experience = 1234567,
            CompletedQuests = new() { "quest_01", "quest_02", "quest_03" },
            Inventory = new()
            {
                ["sword"] = 1, ["potion"] = 50, ["gold"] = 99999
            }
        };

        byte[] compressed = CompressedMemoryPack.SerializeCompressed(largeData);
        var restored = CompressedMemoryPack.DeserializeCompressed<PlayerSaveData>(compressed);

        Debug.Log($"복원: {restored.Name}, Lv.{restored.Level}");
    }
}
```

### 오브젝트 풀링과 함께 사용

```csharp
using MemoryPack;
using System.Buffers;
using UnityEngine;

// =============================================
// ArrayPool을 활용한 제로 할당 직렬화 패턴
// 빈번한 직렬화에서 GC 압박 최소화
// =============================================

public class PooledSerializationExample : MonoBehaviour
{
    // ArrayBufferWriter 재사용으로 할당 최소화
    private readonly ArrayBufferWriter<byte> _bufferWriter = new(256);

    public byte[] SerializePooled<T>(T value)
    {
        _bufferWriter.Clear();
        MemoryPackSerializer.Serialize(_bufferWriter, value);

        // 필요한 만큼만 복사
        return _bufferWriter.WrittenSpan.ToArray();
    }

    // ref 역직렬화로 기존 객체 재활용
    private PlayerData _reusablePlayer = new();

    public PlayerData DeserializeReuse(byte[] data)
    {
        // 기존 인스턴스를 재활용 (새 객체 생성 안 함)
        MemoryPackSerializer.Deserialize(data, ref _reusablePlayer);
        return _reusablePlayer;
    }

    private void Update()
    {
        // 매 프레임 직렬화가 필요한 경우 (예: 네트워크 동기화)
        if (Time.frameCount % 3 == 0) // 3프레임마다
        {
            var moveData = new PlayerMoveMessage
            {
                SenderId = 1,
                Timestamp = System.DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
                X = transform.position.x,
                Y = transform.position.y,
                Z = transform.position.z,
                Yaw = transform.eulerAngles.y
            };

            byte[] packet = SerializePooled(moveData);
            // SendToServer(packet);
        }
    }
}
```

---

## 주의사항

### MemoryPack의 제한사항

| 항목 | 설명 |
|------|------|
| **C# 전용** | 다른 언어(Java, Python, Go 등)에서 읽을 수 없음 |
| **크로스 플랫폼 포맷 아님** | C# 메모리 레이아웃에 의존하므로 엔디안이 다른 환경 간 비호환 가능 |
| **partial 필수** | 모든 직렬화 대상 클래스에 `partial` 키워드 필요 |
| **멤버 순서 중요** | 기본 모드에서 멤버 순서 변경 시 역직렬화 실패 |
| **Unity 제한** | ImmutableCollections, CustomFormatter 미지원 |
| **ModuleInitializer** | Unity에서 미지원, 크로스 어셈블리 Union 수동 등록 필요 |

### 흔한 실수

```csharp
using MemoryPack;
using System.Collections.Generic;

// ❌ 실수 1: partial 키워드 누락
// [MemoryPackable]
// public class BadExample { }    // 컴파일 에러

// ✅ 수정
[MemoryPackable]
public partial class GoodExample
{
    public int Value { get; set; }
}

// ❌ 실수 2: 기본 모드에서 멤버 중간 삽입
// Version 1
// public partial class DataV1 { public int A; public int B; }
// Version 2 (잘못됨)
// public partial class DataV2 { public int A; public int NEW; public int B; }

// ✅ 수정: 끝에 추가하거나 VersionTolerant 사용

// ❌ 실수 3: Union 타입을 구체 타입으로 직렬화
// IGameCommand cmd = new MoveCommand();
// var bytes = MemoryPackSerializer.Serialize(cmd);  // MoveCommand로 직렬화됨!

// ✅ 수정: 제네릭 타입 매개변수에 인터페이스 명시
// var bytes = MemoryPackSerializer.Serialize<IGameCommand>(cmd);

// ❌ 실수 4: VersionTolerant에서 Order 번호 재사용
// [MemoryPackable(GenerateType.VersionTolerant)]
// public partial class BadVersioning
// {
//     [MemoryPackOrder(0)] public int A;
//     [MemoryPackOrder(0)] public int B;  // 중복 번호!
// }

// ❌ 실수 5: 생성자 매개변수 이름 불일치
// [MemoryPackable]
// public partial class BadCtor
// {
//     public int MyValue { get; set; }
//     public BadCtor(int value) { }  // "value" != "myValue"
// }

// ✅ 수정: 매개변수 이름을 프로퍼티 이름과 일치 (대소문자 무시)
[MemoryPackable]
public partial class GoodCtor
{
    public int MyValue { get; set; }
    public GoodCtor(int myValue)    // "myValue" == "MyValue" (대소문자 무시)
    {
        MyValue = myValue;
    }
}
```

### MemoryPack vs 다른 라이브러리 선택 기준

| 요구사항 | 추천 라이브러리 |
|----------|----------------|
| C# 전용, 최고 성능 필요 | **MemoryPack** |
| 다른 언어와 통신 필요 | MessagePack, Protocol Buffers |
| 사람이 읽을 수 있는 포맷 | JSON (System.Text.Json, Newtonsoft) |
| Unity 기본 기능으로 충분 | JsonUtility |
| 스키마 정의 필수 | Protocol Buffers (gRPC) |
| C# 간 통신 + 적당한 성능 | MessagePack for C# |

---

## 베스트 프랙티스

### 1. 직렬화 타입 설계

```csharp
using MemoryPack;
using System.Collections.Generic;

// ✅ 좋은 패턴: 직렬화 전용 DTO 분리
[MemoryPackable]
public partial class PlayerDto
{
    public string Name { get; set; }
    public int Level { get; set; }
    public float Health { get; set; }
    public List<string> Items { get; set; }
}

// 게임 로직 클래스 (직렬화와 분리)
public class Player
{
    public string Name { get; set; }
    public int Level { get; set; }
    public float Health { get; set; }
    public float MaxHealth { get; set; }           // 계산 가능한 값
    public List<string> Items { get; set; }

    // DTO로 변환
    public PlayerDto ToDto() => new()
    {
        Name = Name,
        Level = Level,
        Health = Health,
        Items = Items
    };

    // DTO에서 복원
    public static Player FromDto(PlayerDto dto) => new()
    {
        Name = dto.Name,
        Level = dto.Level,
        Health = dto.Health,
        MaxHealth = dto.Level * 100f,             // 계산값 복원
        Items = dto.Items
    };
}
```

### 2. 버전 관리 전략

```csharp
using MemoryPack;

// ✅ 좋은 패턴: 처음부터 VersionTolerant 사용
// 나중에 Object -> VersionTolerant로 변경하면 기존 데이터와 비호환

[MemoryPackable(GenerateType.VersionTolerant)]
public partial class RobustSaveData
{
    [MemoryPackOrder(0)]
    public int SchemaVersion { get; set; } = 1;   // 버전 번호 포함

    [MemoryPackOrder(1)]
    public string PlayerName { get; set; }

    [MemoryPackOrder(2)]
    public int Level { get; set; }

    // 향후 추가될 필드를 위해 Order 번호에 여유를 둘 필요 없음
    // VersionTolerant는 번호 건너뛰기를 지원
}
```

### 3. 에러 핸들링

```csharp
using MemoryPack;
using System;
using UnityEngine;

public class SafeDeserialization : MonoBehaviour
{
    // ✅ 좋은 패턴: 역직렬화 시 예외 처리
    public T SafeDeserialize<T>(byte[] data, T fallback = default)
    {
        if (data == null || data.Length == 0)
        {
            Debug.LogWarning("데이터가 비어있습니다");
            return fallback;
        }

        try
        {
            return MemoryPackSerializer.Deserialize<T>(data);
        }
        catch (MemoryPackSerializationException ex)
        {
            Debug.LogError($"역직렬화 실패: {ex.Message}");
            return fallback;
        }
        catch (Exception ex)
        {
            Debug.LogError($"예상치 못한 에러: {ex.Message}");
            return fallback;
        }
    }

    // ✅ 좋은 패턴: 데이터 무결성 검증
    public byte[] SerializeWithChecksum<T>(T value)
    {
        byte[] data = MemoryPackSerializer.Serialize(value);

        // 간단한 체크섬 추가
        uint checksum = 0;
        foreach (byte b in data) checksum += b;

        byte[] checksumBytes = BitConverter.GetBytes(checksum);
        byte[] result = new byte[data.Length + 4];
        Buffer.BlockCopy(checksumBytes, 0, result, 0, 4);
        Buffer.BlockCopy(data, 0, result, 4, data.Length);

        return result;
    }

    public T DeserializeWithChecksum<T>(byte[] packet)
    {
        if (packet.Length < 4)
            throw new InvalidOperationException("데이터가 너무 짧습니다");

        uint storedChecksum = BitConverter.ToUInt32(packet, 0);
        byte[] data = new byte[packet.Length - 4];
        Buffer.BlockCopy(packet, 4, data, 0, data.Length);

        uint actualChecksum = 0;
        foreach (byte b in data) actualChecksum += b;

        if (storedChecksum != actualChecksum)
            throw new InvalidOperationException(
                $"체크섬 불일치: 저장={storedChecksum}, 실제={actualChecksum}");

        return MemoryPackSerializer.Deserialize<T>(data);
    }
}
```

### 4. 요약 체크리스트

```
✅ 모든 직렬화 대상 클래스에 [MemoryPackable] + partial 적용
✅ 세이브 데이터는 처음부터 GenerateType.VersionTolerant 사용
✅ Union 사용 시 인터페이스/추상 클래스에 [MemoryPackUnion] 적용
✅ 직렬화 전용 DTO와 게임 로직 클래스 분리
✅ 역직렬화 시 반드시 예외 처리
✅ 네트워크 전송 시 Union으로 다형성 메시지 처리
✅ 대용량 데이터는 비동기 Stream API 사용
✅ 빈번한 직렬화에는 ArrayBufferWriter 재사용

❌ partial 키워드 빠뜨리지 않기
❌ 기본 모드에서 멤버 순서 변경하지 않기
❌ VersionTolerant에서 Order 번호 재사용하지 않기
❌ Union을 구체 타입으로 직렬화하지 않기
❌ 다른 언어와의 통신에 MemoryPack 사용하지 않기
❌ 생성자 매개변수 이름과 프로퍼티 이름 불일치시키지 않기
```

---

## 정리

### MemoryPack 핵심 요약

| 항목 | 내용 |
|------|------|
| **개발자** | neuecc (Yoshifumi Kawai, Cysharp) |
| **핵심 원리** | Zero-encoding (C# 메모리 직접 복사) |
| **코드 생성** | Source Generator (컴파일 타임) |
| **성능** | 표준 객체 x2~10 빠름, struct 배열 x50~200 빠름 |
| **AOT 지원** | IL2CPP/Native AOT 완전 호환 |
| **Unity** | 2021.3+, NuGetForUnity 또는 Git URL |
| **제한** | C# 전용, 크로스 언어 비호환 |

### 언제 MemoryPack을 사용할까

| 시나리오 | 적합성 |
|----------|--------|
| C# 서버 + Unity 클라이언트 간 통신 | **매우 적합** |
| 게임 세이브/로드 시스템 | **매우 적합** |
| 인게임 데이터 캐싱 | **매우 적합** |
| 다른 언어 서버와 통신 | 부적합 (MessagePack/Protobuf 사용) |
| 사람이 읽을 수 있는 설정 파일 | 부적합 (JSON/YAML 사용) |
| 장기 보관 아카이브 | 주의 필요 (스키마 변경 관리) |

---

## 참고 자료

- [MemoryPack GitHub](https://github.com/Cysharp/MemoryPack)
- [MemoryPack NuGet](https://www.nuget.org/packages/MemoryPack)
- [neuecc Blog - MemoryPack 설계](https://neuecc.medium.com/how-to-make-the-fastest-net-serializer-with-net-7-c-11-case-of-memorypack-ad28c0366516)
- [Cysharp 공식 사이트](https://cysharp.co.jp/)
- [MessagePack for C# (비교 참고)](https://github.com/MessagePack-CSharp/MessagePack-CSharp)
