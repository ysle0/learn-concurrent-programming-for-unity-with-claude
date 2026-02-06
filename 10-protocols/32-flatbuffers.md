# Section 32: FlatBuffers

## 개요

FlatBuffers는 Google이 개발한 제로 파싱(Zero-Copy) 직렬화 라이브러리입니다. 역직렬화 없이 직렬화된 데이터에 직접 접근할 수 있어, 게임과 같은 성능 중심 애플리케이션에 적합합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    FlatBuffers Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   기존 직렬화 (Parsing Required):                                 │
│   ┌─────────┐   Parse    ┌─────────┐   Use    ┌─────────┐      │
│   │ Binary  │ ─────────▶ │ Object  │ ───────▶ │ Access  │      │
│   │  Data   │  (느림)    │ (할당)  │          │  Data   │      │
│   └─────────┘            └─────────┘          └─────────┘      │
│                                                                  │
│   FlatBuffers (Zero-Copy):                                       │
│   ┌─────────┐   Direct Access   ┌─────────┐                     │
│   │ Binary  │ ────────────────▶ │ Access  │                     │
│   │  Data   │   (즉시, 무할당)   │  Data   │                     │
│   └─────────┘                   └─────────┘                     │
│                                                                  │
│   장점:                                                          │
│   ┌────────────────────────────────────────────────────┐        │
│   │ • 역직렬화 비용 없음 (Zero-Copy)                     │        │
│   │ • 메모리 할당 없음 (GC 없음)                         │        │
│   │ • 스키마 진화 지원 (하위 호환성)                     │        │
│   │ • 랜덤 액세스 가능                                  │        │
│   │ • 작은 메모리 풋프린트                              │        │
│   └────────────────────────────────────────────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## FlatBuffers vs 다른 포맷

| 특성 | FlatBuffers | Protocol Buffers | JSON | MessagePack |
|------|------------|------------------|------|-------------|
| **파싱 필요** | 없음 | 필요 | 필요 | 필요 |
| **메모리 할당** | 없음 | 있음 | 있음 | 있음 |
| **랜덤 액세스** | 가능 | 불가능 | 불가능 | 불가능 |
| **스키마** | 필수 (.fbs) | 필수 (.proto) | 선택 | 선택 |
| **크기** | 중간 | 작음 | 큼 | 작음 |
| **생성 속도** | 중간 | 빠름 | 느림 | 빠름 |
| **읽기 속도** | 매우 빠름 | 빠름 | 느림 | 빠름 |

---

## 설치 및 설정

### FlatBuffers 컴파일러 설치

```bash
# macOS
brew install flatbuffers

# Windows (Chocolatey)
choco install flatbuffers

# 또는 GitHub Releases에서 다운로드
# https://github.com/google/flatbuffers/releases
```

### Unity 패키지 추가

```
1. FlatBuffers C# 라이브러리 다운로드
   - GitHub: https://github.com/google/flatbuffers
   - Runtime: flatbuffers/net/FlatBuffers

2. Unity Plugins 폴더에 복사
   - Assets/Plugins/FlatBuffers/
```

---

## 스키마 정의 (.fbs)

### 기본 스키마

```fbs
// game_schema.fbs

namespace Game.FlatBuffers;

// 열거형
enum ItemRarity : byte {
    Common = 0,
    Uncommon = 1,
    Rare = 2,
    Epic = 3,
    Legendary = 4
}

enum EntityType : byte {
    Player = 0,
    NPC = 1,
    Monster = 2,
    Projectile = 3
}

// 구조체 (인라인, 고정 크기)
struct Vec3 {
    x: float;
    y: float;
    z: float;
}

struct Quaternion {
    x: float;
    y: float;
    z: float;
    w: float;
}

struct Color {
    r: byte;
    g: byte;
    b: byte;
    a: byte;
}

// 테이블 (가변 크기, 필드 추가/제거 가능)
table Item {
    id: int;
    name: string;
    rarity: ItemRarity = Common;
    quantity: int = 1;
    durability: float = 100.0;
    custom_data: [byte];
}

table PlayerStats {
    health: float = 100.0;
    max_health: float = 100.0;
    mana: float = 50.0;
    max_mana: float = 50.0;
    attack: int = 10;
    defense: int = 5;
    speed: float = 5.0;
}

table PlayerData {
    id: string (required);
    name: string;
    level: int = 1;
    experience: long;
    gold: int;
    position: Vec3;
    rotation: Quaternion;
    stats: PlayerStats;
    inventory: [Item];
    equipped_items: [int];
}

// 게임 상태
table EntityState {
    entity_id: int;
    entity_type: EntityType;
    position: Vec3;
    rotation: Quaternion;
    velocity: Vec3;
    health: float;
    animation_state: byte;
    flags: ushort;
}

table WorldState {
    tick: long;
    timestamp: long;
    entities: [EntityState];
}

// 루트 타입 지정
root_type WorldState;
```

### 스키마 컴파일

```bash
# C# 코드 생성
flatc --csharp -o Assets/Scripts/Generated game_schema.fbs

# 생성되는 파일들:
# - Vec3.cs
# - Quaternion.cs
# - Color.cs
# - Item.cs
# - PlayerStats.cs
# - PlayerData.cs
# - EntityState.cs
# - WorldState.cs
```

---

## 기본 사용법

### 데이터 생성 (직렬화)

```csharp
using FlatBuffers;
using Game.FlatBuffers;
using UnityEngine;

/// <summary>
/// FlatBuffers 데이터 생성 예제
/// </summary>
public class FlatBufferBuilder_Example : MonoBehaviour
{
    private void Start()
    {
        // FlatBufferBuilder 생성 (버퍼 크기 지정)
        var builder = new FlatBufferBuilder(1024);

        // 문자열은 먼저 생성해야 함 (오프셋 반환)
        var playerId = builder.CreateString("player_001");
        var playerName = builder.CreateString("Hero");

        // 아이템 생성
        var itemName1 = builder.CreateString("Sword of Fire");
        var item1 = Item.CreateItem(
            builder,
            id: 1,
            nameOffset: itemName1,
            rarity: ItemRarity.Rare,
            quantity: 1,
            durability: 95.5f);

        var itemName2 = builder.CreateString("Health Potion");
        var item2 = Item.CreateItem(
            builder,
            id: 2,
            nameOffset: itemName2,
            rarity: ItemRarity.Common,
            quantity: 10);

        // 아이템 배열 생성
        var inventoryOffset = PlayerData.CreateInventoryVector(builder, new[] { item1, item2 });

        // 장착 아이템 배열
        var equippedOffset = PlayerData.CreateEquippedItemsVector(builder, new[] { 1, 0, 0, 0 });

        // PlayerStats 테이블 생성
        PlayerStats.StartPlayerStats(builder);
        PlayerStats.AddHealth(builder, 85.5f);
        PlayerStats.AddMaxHealth(builder, 100f);
        PlayerStats.AddMana(builder, 40f);
        PlayerStats.AddMaxMana(builder, 50f);
        PlayerStats.AddAttack(builder, 25);
        PlayerStats.AddDefense(builder, 15);
        PlayerStats.AddSpeed(builder, 6.5f);
        var statsOffset = PlayerStats.EndPlayerStats(builder);

        // PlayerData 테이블 생성
        PlayerData.StartPlayerData(builder);
        PlayerData.AddId(builder, playerId);
        PlayerData.AddName(builder, playerName);
        PlayerData.AddLevel(builder, 42);
        PlayerData.AddExperience(builder, 1234567L);
        PlayerData.AddGold(builder, 9999);
        PlayerData.AddPosition(builder, Vec3.CreateVec3(builder, 10f, 0f, 20f));
        PlayerData.AddRotation(builder, Quaternion.CreateQuaternion(builder, 0f, 0.7f, 0f, 0.7f));
        PlayerData.AddStats(builder, statsOffset);
        PlayerData.AddInventory(builder, inventoryOffset);
        PlayerData.AddEquippedItems(builder, equippedOffset);
        var playerOffset = PlayerData.EndPlayerData(builder);

        // 루트로 마무리
        builder.Finish(playerOffset.Value);

        // 바이트 배열 얻기
        byte[] bytes = builder.SizedByteArray();
        Debug.Log($"직렬화 크기: {bytes.Length} bytes");

        // 저장 또는 네트워크 전송
        SaveOrSend(bytes);
    }

    private void SaveOrSend(byte[] data)
    {
        // 네트워크 전송 또는 파일 저장
    }
}
```

### 데이터 읽기 (제로 카피)

```csharp
using FlatBuffers;
using Game.FlatBuffers;
using UnityEngine;

/// <summary>
/// FlatBuffers 데이터 읽기 예제
/// </summary>
public class FlatBufferReader_Example : MonoBehaviour
{
    public void ReadPlayerData(byte[] data)
    {
        // ByteBuffer 래핑 (복사 없음!)
        var buffer = new ByteBuffer(data);

        // 루트 객체 접근 (파싱 없음!)
        var player = PlayerData.GetRootAsPlayerData(buffer);

        // 기본 필드 접근
        Debug.Log($"ID: {player.Id}");
        Debug.Log($"이름: {player.Name}");
        Debug.Log($"레벨: {player.Level}");
        Debug.Log($"경험치: {player.Experience}");
        Debug.Log($"골드: {player.Gold}");

        // 구조체 접근 (인라인)
        var pos = player.Position;
        if (pos.HasValue)
        {
            Debug.Log($"위치: ({pos.Value.X}, {pos.Value.Y}, {pos.Value.Z})");
        }

        var rot = player.Rotation;
        if (rot.HasValue)
        {
            var unityRot = new UnityEngine.Quaternion(
                rot.Value.X, rot.Value.Y, rot.Value.Z, rot.Value.W);
            Debug.Log($"회전: {unityRot.eulerAngles}");
        }

        // 중첩 테이블 접근
        var stats = player.Stats;
        if (stats.HasValue)
        {
            Debug.Log($"HP: {stats.Value.Health}/{stats.Value.MaxHealth}");
            Debug.Log($"MP: {stats.Value.Mana}/{stats.Value.MaxMana}");
        }

        // 배열 접근
        Debug.Log($"인벤토리 ({player.InventoryLength}개 아이템):");
        for (int i = 0; i < player.InventoryLength; i++)
        {
            var item = player.Inventory(i);
            if (item.HasValue)
            {
                Debug.Log($"  - {item.Value.Name} x{item.Value.Quantity} ({item.Value.Rarity})");
            }
        }

        // 장착 아이템
        Debug.Log($"장착 슬롯: {player.EquippedItemsLength}개");
        for (int i = 0; i < player.EquippedItemsLength; i++)
        {
            Debug.Log($"  슬롯 {i}: 아이템 ID {player.EquippedItems(i)}");
        }
    }
}
```

---

## 네트워크 프로토콜

### 네트워크 메시지 스키마

```fbs
// network_messages.fbs

namespace Game.Network;

// 메시지 타입 열거형
enum MessageType : byte {
    Ping = 0,
    Pong = 1,
    Login = 2,
    LoginResponse = 3,
    PlayerInput = 4,
    WorldState = 5,
    GameEvent = 6
}

// 공통 헤더
table MessageHeader {
    type: MessageType;
    sequence: uint;
    timestamp: long;
}

// Ping/Pong
table Ping {
    client_time: long;
}

table Pong {
    client_time: long;
    server_time: long;
}

// 로그인
table LoginRequest {
    username: string;
    token: string;
    device_id: string;
    version: string;
}

table LoginResponse {
    success: bool;
    player_id: string;
    session_token: string;
    error_code: int;
    error_message: string;
}

// 플레이어 입력
struct InputState {
    move_x: float;
    move_y: float;
    aim_x: float;
    aim_y: float;
    buttons: ushort;  // 비트 플래그
}

table PlayerInput {
    sequence: int;
    input: InputState;
}

// 엔티티 상태
struct EntitySnapshot {
    entity_id: int;
    entity_type: byte;
    pos_x: float;
    pos_y: float;
    pos_z: float;
    rot_y: float;
    vel_x: float;
    vel_z: float;
    health: float;
    anim_state: byte;
    flags: ushort;
}

table WorldStateUpdate {
    tick: long;
    last_processed_input: int;
    entities: [EntitySnapshot];
}

// 게임 이벤트
table GameEvent {
    event_type: int;
    source_id: int;
    target_id: int;
    position: Vec3;
    value: float;
    data: [byte];
}

// Union을 사용한 다형성 메시지
union MessagePayload {
    Ping,
    Pong,
    LoginRequest,
    LoginResponse,
    PlayerInput,
    WorldStateUpdate,
    GameEvent
}

// 래퍼 메시지
table NetworkMessage {
    header: MessageHeader;
    payload: MessagePayload;
}

root_type NetworkMessage;
```

### 네트워크 직렬화 서비스

```csharp
using System;
using System.Buffers;
using FlatBuffers;
using Game.Network;
using UnityEngine;

/// <summary>
/// FlatBuffers 네트워크 직렬화 서비스
/// </summary>
public class FlatBufferNetworkService
{
    // 버퍼 풀링
    private readonly FlatBufferBuilder _builder;
    private uint _sequenceNumber;

    public FlatBufferNetworkService(int initialBufferSize = 1024)
    {
        _builder = new FlatBufferBuilder(initialBufferSize);
    }

    /// <summary>
    /// 빌더 초기화 (재사용)
    /// </summary>
    private void ResetBuilder()
    {
        _builder.Clear();
    }

    /// <summary>
    /// Ping 메시지 생성
    /// </summary>
    public byte[] CreatePing()
    {
        ResetBuilder();

        var ping = Ping.CreatePing(_builder,
            clientTime: DateTimeOffset.UtcNow.ToUnixTimeMilliseconds());

        return FinishMessage(MessageType.Ping, MessagePayload.Ping, ping.Value);
    }

    /// <summary>
    /// 플레이어 입력 메시지 생성
    /// </summary>
    public byte[] CreatePlayerInput(int sequence, float moveX, float moveY,
        float aimX, float aimY, ushort buttons)
    {
        ResetBuilder();

        var input = new InputState();
        // 구조체 필드는 직접 설정할 수 없으므로 CreateInputState 사용

        PlayerInput.StartPlayerInput(_builder);
        PlayerInput.AddSequence(_builder, sequence);
        PlayerInput.AddInput(_builder, InputState.CreateInputState(
            _builder, moveX, moveY, aimX, aimY, buttons));
        var inputOffset = PlayerInput.EndPlayerInput(_builder);

        return FinishMessage(MessageType.PlayerInput, MessagePayload.PlayerInput, inputOffset.Value);
    }

    /// <summary>
    /// 월드 상태 업데이트 생성 (서버용)
    /// </summary>
    public byte[] CreateWorldState(long tick, int lastInput, EntitySnapshot[] entities)
    {
        ResetBuilder();

        // 엔티티 배열 생성
        WorldStateUpdate.StartEntitiesVector(_builder, entities.Length);
        for (int i = entities.Length - 1; i >= 0; i--)
        {
            var e = entities[i];
            EntitySnapshot.CreateEntitySnapshot(_builder,
                e.EntityId, e.EntityType,
                e.PosX, e.PosY, e.PosZ, e.RotY,
                e.VelX, e.VelZ,
                e.Health, e.AnimState, e.Flags);
        }
        var entitiesOffset = _builder.EndVector();

        var stateOffset = WorldStateUpdate.CreateWorldStateUpdate(
            _builder, tick, lastInput, entitiesOffset);

        return FinishMessage(MessageType.WorldState, MessagePayload.WorldStateUpdate, stateOffset.Value);
    }

    /// <summary>
    /// 메시지 완성
    /// </summary>
    private byte[] FinishMessage(MessageType type, MessagePayload payloadType, int payloadOffset)
    {
        // 헤더 생성
        var header = MessageHeader.CreateMessageHeader(
            _builder,
            type,
            ++_sequenceNumber,
            DateTimeOffset.UtcNow.ToUnixTimeMilliseconds());

        // 메시지 생성
        var message = NetworkMessage.CreateNetworkMessage(
            _builder,
            header,
            payloadType,
            payloadOffset);

        _builder.Finish(message.Value);
        return _builder.SizedByteArray();
    }

    /// <summary>
    /// 메시지 파싱
    /// </summary>
    public (MessageType Type, object Payload) ParseMessage(byte[] data)
    {
        var buffer = new ByteBuffer(data);
        var message = NetworkMessage.GetRootAsNetworkMessage(buffer);

        var header = message.Header;
        var type = header.Value.Type;

        object payload = type switch
        {
            MessageType.Ping => message.Payload<Ping>(),
            MessageType.Pong => message.Payload<Pong>(),
            MessageType.LoginRequest => message.Payload<LoginRequest>(),
            MessageType.LoginResponse => message.Payload<LoginResponse>(),
            MessageType.PlayerInput => message.Payload<PlayerInput>(),
            MessageType.WorldState => message.Payload<WorldStateUpdate>(),
            MessageType.GameEvent => message.Payload<GameEvent>(),
            _ => null
        };

        return (type, payload);
    }
}
```

### 클라이언트 사용 예시

```csharp
using Game.Network;
using UnityEngine;

/// <summary>
/// FlatBuffers 네트워크 클라이언트
/// </summary>
public class FlatBufferNetworkClient : MonoBehaviour
{
    private FlatBufferNetworkService _serializer;
    private int _inputSequence;

    private void Awake()
    {
        _serializer = new FlatBufferNetworkService();
    }

    private void Update()
    {
        // 입력 전송
        SendInput();
    }

    private void SendInput()
    {
        float moveX = Input.GetAxis("Horizontal");
        float moveY = Input.GetAxis("Vertical");
        Vector2 aim = GetAimDirection();
        ushort buttons = GetButtonFlags();

        byte[] data = _serializer.CreatePlayerInput(
            ++_inputSequence,
            moveX, moveY,
            aim.x, aim.y,
            buttons);

        SendToServer(data);
    }

    private ushort GetButtonFlags()
    {
        ushort flags = 0;
        if (Input.GetButton("Jump")) flags |= 0x01;
        if (Input.GetButton("Fire1")) flags |= 0x02;
        if (Input.GetButton("Fire2")) flags |= 0x04;
        if (Input.GetKey(KeyCode.LeftShift)) flags |= 0x08;
        return flags;
    }

    private Vector2 GetAimDirection()
    {
        return Camera.main.ScreenToViewportPoint(Input.mousePosition);
    }

    public void OnDataReceived(byte[] data)
    {
        var (type, payload) = _serializer.ParseMessage(data);

        switch (type)
        {
            case MessageType.Pong:
                HandlePong((Pong?)payload);
                break;

            case MessageType.WorldState:
                HandleWorldState((WorldStateUpdate?)payload);
                break;

            case MessageType.GameEvent:
                HandleGameEvent((GameEvent?)payload);
                break;
        }
    }

    private void HandlePong(Pong? pong)
    {
        if (!pong.HasValue) return;

        long rtt = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds() - pong.Value.ClientTime;
        Debug.Log($"RTT: {rtt}ms");
    }

    private void HandleWorldState(WorldStateUpdate? state)
    {
        if (!state.HasValue) return;

        // 엔티티 상태 적용
        for (int i = 0; i < state.Value.EntitiesLength; i++)
        {
            var entity = state.Value.Entities(i);
            if (entity.HasValue)
            {
                ApplyEntityState(entity.Value);
            }
        }
    }

    private void ApplyEntityState(EntitySnapshot entity)
    {
        // 엔티티 위치/상태 적용
        Debug.Log($"Entity {entity.EntityId}: ({entity.PosX}, {entity.PosY}, {entity.PosZ})");
    }

    private void HandleGameEvent(GameEvent? evt)
    {
        if (!evt.HasValue) return;

        Debug.Log($"Game Event: Type={evt.Value.EventType}, " +
                 $"Source={evt.Value.SourceId}, Target={evt.Value.TargetId}");
    }

    private void SendToServer(byte[] data)
    {
        // WebSocket 또는 UDP로 전송
    }
}
```

---

## 세이브 데이터 시스템

```csharp
using System.IO;
using FlatBuffers;
using Game.FlatBuffers;
using UnityEngine;

/// <summary>
/// FlatBuffers 기반 세이브 시스템
/// </summary>
public class FlatBufferSaveSystem : MonoBehaviour
{
    private const string SaveFileName = "save.fbs";
    private string SavePath => Path.Combine(Application.persistentDataPath, SaveFileName);

    /// <summary>
    /// 게임 저장
    /// </summary>
    public void Save(PlayerData_Game data)
    {
        var builder = new FlatBufferBuilder(2048);

        // 플레이어 데이터 생성
        var playerId = builder.CreateString(data.PlayerId);
        var playerName = builder.CreateString(data.PlayerName);

        // 인벤토리 생성
        var itemOffsets = new Offset<Item>[data.Inventory.Length];
        for (int i = 0; i < data.Inventory.Length; i++)
        {
            var item = data.Inventory[i];
            var itemName = builder.CreateString(item.Name);

            itemOffsets[i] = Item.CreateItem(builder,
                item.Id, itemName, (ItemRarity)item.Rarity,
                item.Quantity, item.Durability);
        }
        var inventoryOffset = PlayerData.CreateInventoryVector(builder, itemOffsets);
        var equippedOffset = PlayerData.CreateEquippedItemsVector(builder, data.EquippedItems);

        // Stats 생성
        var statsOffset = PlayerStats.CreatePlayerStats(builder,
            data.Stats.Health, data.Stats.MaxHealth,
            data.Stats.Mana, data.Stats.MaxMana,
            data.Stats.Attack, data.Stats.Defense, data.Stats.Speed);

        // PlayerData 생성
        PlayerData.StartPlayerData(builder);
        PlayerData.AddId(builder, playerId);
        PlayerData.AddName(builder, playerName);
        PlayerData.AddLevel(builder, data.Level);
        PlayerData.AddExperience(builder, data.Experience);
        PlayerData.AddGold(builder, data.Gold);
        PlayerData.AddPosition(builder, Vec3.CreateVec3(builder,
            data.Position.x, data.Position.y, data.Position.z));
        PlayerData.AddRotation(builder, Quaternion.CreateQuaternion(builder,
            data.Rotation.x, data.Rotation.y, data.Rotation.z, data.Rotation.w));
        PlayerData.AddStats(builder, statsOffset);
        PlayerData.AddInventory(builder, inventoryOffset);
        PlayerData.AddEquippedItems(builder, equippedOffset);
        var playerOffset = PlayerData.EndPlayerData(builder);

        builder.Finish(playerOffset.Value);

        // 파일 저장
        File.WriteAllBytes(SavePath, builder.SizedByteArray());
        Debug.Log($"게임 저장 완료: {SavePath}");
    }

    /// <summary>
    /// 게임 로드
    /// </summary>
    public PlayerData_Game Load()
    {
        if (!File.Exists(SavePath))
        {
            return CreateNewGame();
        }

        byte[] data = File.ReadAllBytes(SavePath);
        var buffer = new ByteBuffer(data);
        var fb = PlayerData.GetRootAsPlayerData(buffer);

        var result = new PlayerData_Game
        {
            PlayerId = fb.Id,
            PlayerName = fb.Name,
            Level = fb.Level,
            Experience = fb.Experience,
            Gold = fb.Gold
        };

        // Position
        if (fb.Position.HasValue)
        {
            var pos = fb.Position.Value;
            result.Position = new Vector3(pos.X, pos.Y, pos.Z);
        }

        // Rotation
        if (fb.Rotation.HasValue)
        {
            var rot = fb.Rotation.Value;
            result.Rotation = new UnityEngine.Quaternion(rot.X, rot.Y, rot.Z, rot.W);
        }

        // Stats
        if (fb.Stats.HasValue)
        {
            var s = fb.Stats.Value;
            result.Stats = new PlayerStats_Game
            {
                Health = s.Health,
                MaxHealth = s.MaxHealth,
                Mana = s.Mana,
                MaxMana = s.MaxMana,
                Attack = s.Attack,
                Defense = s.Defense,
                Speed = s.Speed
            };
        }

        // Inventory
        result.Inventory = new Item_Game[fb.InventoryLength];
        for (int i = 0; i < fb.InventoryLength; i++)
        {
            var item = fb.Inventory(i);
            if (item.HasValue)
            {
                result.Inventory[i] = new Item_Game
                {
                    Id = item.Value.Id,
                    Name = item.Value.Name,
                    Rarity = (int)item.Value.Rarity,
                    Quantity = item.Value.Quantity,
                    Durability = item.Value.Durability
                };
            }
        }

        // Equipped Items
        result.EquippedItems = new int[fb.EquippedItemsLength];
        for (int i = 0; i < fb.EquippedItemsLength; i++)
        {
            result.EquippedItems[i] = fb.EquippedItems(i);
        }

        Debug.Log("게임 로드 완료");
        return result;
    }

    private PlayerData_Game CreateNewGame()
    {
        return new PlayerData_Game
        {
            PlayerId = System.Guid.NewGuid().ToString(),
            PlayerName = "Hero",
            Level = 1,
            Experience = 0,
            Gold = 100,
            Position = Vector3.zero,
            Rotation = UnityEngine.Quaternion.identity,
            Stats = new PlayerStats_Game
            {
                Health = 100,
                MaxHealth = 100,
                Mana = 50,
                MaxMana = 50,
                Attack = 10,
                Defense = 5,
                Speed = 5
            },
            Inventory = new Item_Game[0],
            EquippedItems = new int[10]
        };
    }
}

// 게임 내부 데이터 클래스
public class PlayerData_Game
{
    public string PlayerId;
    public string PlayerName;
    public int Level;
    public long Experience;
    public int Gold;
    public Vector3 Position;
    public UnityEngine.Quaternion Rotation;
    public PlayerStats_Game Stats;
    public Item_Game[] Inventory;
    public int[] EquippedItems;
}

public class PlayerStats_Game
{
    public float Health, MaxHealth, Mana, MaxMana;
    public int Attack, Defense;
    public float Speed;
}

public class Item_Game
{
    public int Id;
    public string Name;
    public int Rarity;
    public int Quantity;
    public float Durability;
}
```

---

## 스키마 진화 (버전 관리)

```fbs
// 스키마 진화 규칙:
// 1. 새 필드는 테이블 끝에 추가
// 2. 기존 필드 ID 변경 금지
// 3. 필드 삭제 대신 deprecated 표시
// 4. 구조체는 변경 불가 (테이블 사용 권장)

table PlayerData_v2 {
    // 기존 필드 (ID 유지)
    id: string (required, id: 0);
    name: string (id: 1);
    level: int = 1 (id: 2);
    experience: long (id: 3);
    gold: int (id: 4);

    // deprecated 필드 (삭제하지 않고 표시)
    old_field: int (deprecated, id: 5);

    // v2에서 추가된 필드
    gems: int (id: 6);
    vip_level: int (id: 7);
    achievements: [int] (id: 8);
}
```

---

## 성능 최적화

### 객체 풀링

```csharp
using System.Collections.Concurrent;
using FlatBuffers;

/// <summary>
/// FlatBufferBuilder 풀링
/// </summary>
public class FlatBufferBuilderPool
{
    private readonly ConcurrentBag<FlatBufferBuilder> _pool = new();
    private readonly int _initialSize;

    public FlatBufferBuilderPool(int initialSize = 1024)
    {
        _initialSize = initialSize;
    }

    public FlatBufferBuilder Rent()
    {
        if (_pool.TryTake(out var builder))
        {
            builder.Clear();
            return builder;
        }
        return new FlatBufferBuilder(_initialSize);
    }

    public void Return(FlatBufferBuilder builder)
    {
        _pool.Add(builder);
    }
}

/// <summary>
/// 풀링 사용 예시
/// </summary>
public class PooledFlatBufferExample : MonoBehaviour
{
    private FlatBufferBuilderPool _pool;

    private void Awake()
    {
        _pool = new FlatBufferBuilderPool();
    }

    public byte[] CreateMessage()
    {
        var builder = _pool.Rent();
        try
        {
            // 메시지 생성...
            builder.Finish(/* offset */);
            return builder.SizedByteArray();
        }
        finally
        {
            _pool.Return(builder);
        }
    }
}
```

### ByteBuffer 재사용

```csharp
using FlatBuffers;

/// <summary>
/// ByteBuffer 재사용 래퍼
/// </summary>
public class ReusableByteBuffer
{
    private byte[] _buffer;
    private ByteBuffer _byteBuffer;

    public ReusableByteBuffer(int initialSize = 4096)
    {
        _buffer = new byte[initialSize];
        _byteBuffer = new ByteBuffer(_buffer);
    }

    /// <summary>
    /// 데이터로 버퍼 업데이트
    /// </summary>
    public ByteBuffer Update(byte[] data)
    {
        // 버퍼 크기 확장 필요 시
        if (data.Length > _buffer.Length)
        {
            _buffer = new byte[data.Length * 2];
        }

        // 데이터 복사
        System.Array.Copy(data, _buffer, data.Length);

        // ByteBuffer 재생성 (내부적으로 가벼움)
        _byteBuffer = new ByteBuffer(_buffer);

        return _byteBuffer;
    }
}
```

---

## 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────┐
│                  FlatBuffers 베스트 프랙티스                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 스키마 설계                                                   │
│     ├── 자주 접근하는 필드는 테이블 앞쪽에 배치                    │
│     ├── 구조체는 고정 크기 데이터에만 사용                         │
│     └── 필드 ID는 변경하지 않음                                   │
│                                                                  │
│  2. 성능 최적화                                                   │
│     ├── FlatBufferBuilder 풀링/재사용                            │
│     ├── 필요한 필드만 접근                                        │
│     └── 벡터는 한 번에 생성                                       │
│                                                                  │
│  3. 메모리 관리                                                   │
│     ├── ByteBuffer 재사용                                        │
│     ├── 대용량 데이터는 스트리밍 처리                              │
│     └── 불필요한 복사 피하기                                      │
│                                                                  │
│  4. 버전 관리                                                    │
│     ├── 필드 삭제 대신 deprecated 표시                            │
│     ├── 새 필드는 테이블 끝에 추가                                 │
│     └── 기본값 활용                                               │
│                                                                  │
│  5. 디버깅                                                       │
│     ├── flatc --json으로 JSON 변환                               │
│     ├── reflection 기능 활용                                      │
│     └── 스키마 유효성 검사                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [FlatBuffers Official](https://google.github.io/flatbuffers/)
- [FlatBuffers GitHub](https://github.com/google/flatbuffers)
- [C# Usage Guide](https://google.github.io/flatbuffers/flatbuffers_guide_use_c-sharp.html)
- [Schema Guide](https://google.github.io/flatbuffers/flatbuffers_guide_writing_schema.html)

---

## 다음 단계

- [Section 33: GraphQL](./33-graphql.md) - 유연한 쿼리 API
- [Section 37: 플랫폼별 OS 고려사항](../12-platform/37-os-considerations.md)
