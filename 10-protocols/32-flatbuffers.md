# 32. FlatBuffers

## 개요

**FlatBuffers**는 Google이 게임 개발을 위해 만든 **고성능 크로스 플랫폼 직렬화 라이브러리**입니다. 가장 큰 특징은 **Zero-copy 역직렬화**로, 직렬화된 바이너리 데이터를 별도의 파싱/언패킹 과정 없이 직접 접근할 수 있습니다. 이로 인해 메모리 할당이 거의 없고, 접근 속도가 매우 빠르며, Unity 게임의 네트워크 패킷, 설정 파일, 저장 데이터 등에 이상적입니다.

---

## 1. FlatBuffers vs 다른 직렬화 포맷

### 직렬화 포맷 비교

| 특성 | JSON | Protocol Buffers | MessagePack | FlatBuffers |
|------|------|-----------------|-------------|-------------|
| **포맷** | 텍스트 | 바이너리 | 바이너리 | 바이너리 |
| **스키마** | 없음 | 필수 (.proto) | 선택 | 필수 (.fbs) |
| **역직렬화** | 파싱 필요 | 언패킹 필요 | 언패킹 필요 | Zero-copy |
| **메모리 할당** | 많음 | 중간 | 중간 | 거의 없음 |
| **접근 속도** | 느림 | 빠름 | 빠름 | 가장 빠름 |
| **데이터 크기** | 큼 | 작음 | 작음 | 중간 |
| **가독성** | 높음 | 낮음 | 낮음 | 낮음 |
| **수정 가능** | 예 | 예 | 예 | 제한적 |
| **GC 압력** | 높음 | 중간 | 중간 | 매우 낮음 |

### 왜 게임에서 FlatBuffers인가?

```csharp
using UnityEngine;
using System.Diagnostics;

/// <summary>
/// 직렬화 방식별 성능 차이를 보여주는 개념 예제
/// </summary>
public class SerializationComparisonExample : MonoBehaviour
{
    // =============================================
    // JSON 방식: 파싱 + 객체 생성 필요
    // =============================================

    /*
     * JSON 역직렬화 과정:
     * 1. 바이트 배열 → 문자열 변환 (메모리 할당)
     * 2. 문자열 파싱 (CPU 사용)
     * 3. C# 객체 생성 (메모리 할당 + GC 압력)
     * 4. 필드에 값 대입
     *
     * 예: 1000개의 몬스터 데이터 → 1000개의 객체 생성
     */

    // =============================================
    // FlatBuffers 방식: Zero-copy 직접 접근
    // =============================================

    /*
     * FlatBuffers 역직렬화 과정:
     * 1. 바이트 배열을 ByteBuffer로 래핑 (할당 최소)
     * 2. 오프셋 기반으로 데이터 직접 접근 (파싱 없음)
     * 3. 객체 생성 없음 (struct 기반 접근자)
     *
     * 예: 1000개의 몬스터 데이터 → 필요한 것만 직접 읽기
     */

    void Start()
    {
        // 개념적 비교
        UnityEngine.Debug.Log("=== 직렬화 방식 비교 ===");
        UnityEngine.Debug.Log("JSON: 전체 파싱 → 전체 객체 생성 → 필드 접근");
        UnityEngine.Debug.Log("ProtoBuf: 전체 언패킹 → 전체 객체 생성 → 필드 접근");
        UnityEngine.Debug.Log("FlatBuffers: 버퍼 래핑 → 오프셋으로 직접 접근 (Zero-copy)");
    }
}
```

---

## 2. Schema 정의 (.fbs 파일)

### 기본 스키마 구조

```
// game_schema.fbs
// FlatBuffers 스키마 정의 파일

namespace GameData;

// =============================================
// Enum 정의
// =============================================

enum ElementType : byte {
    None = 0,
    Fire,
    Water,
    Earth,
    Wind,
    Lightning
}

enum ItemRarity : byte {
    Common = 0,
    Uncommon,
    Rare,
    Epic,
    Legendary
}

// =============================================
// Struct 정의 (고정 크기, 인라인 저장)
// =============================================

// struct는 모든 필드가 스칼라 타입이어야 함
struct Vec3 {
    x: float;
    y: float;
    z: float;
}

struct Color {
    r: ubyte;
    g: ubyte;
    b: ubyte;
    a: ubyte;
}

// =============================================
// Table 정의 (가변 크기, 오프셋 기반)
// =============================================

table Stat {
    name: string;
    base_value: float;
    modifier: float = 1.0;
}

table Item {
    id: int;
    name: string;
    rarity: ItemRarity = Common;
    stats: [Stat];
    icon_path: string;
}

table Monster {
    id: int;
    name: string;
    level: short;
    hp: int;
    position: Vec3;
    element: ElementType = None;
    skills: [string];
    drop_items: [Item];
    is_boss: bool = false;
}

// =============================================
// Root Table (최상위 컨테이너)
// =============================================

table MonsterDatabase {
    version: int;
    monsters: [Monster];
    last_updated: string;
}

root_type MonsterDatabase;
```

### 스키마 타입 시스템

```
// FlatBuffers 지원 타입:
//
// 스칼라 타입:
//   bool, byte, ubyte, short, ushort,
//   int, uint, long, ulong, float, double
//
// 복합 타입:
//   string     - UTF-8 문자열
//   [type]     - 벡터(배열)
//   table      - 가변 크기 구조체 (필드 추가/제거 가능)
//   struct     - 고정 크기 구조체 (스칼라만 포함, 더 빠름)
//   enum       - 열거형
//   union      - 여러 table 중 하나를 선택
//
// struct vs table:
//   struct: 인라인 저장, 오프셋 없음, 더 빠르고 작음
//   table:  오프셋 기반, 기본값 지원, 스키마 진화 가능

// =============================================
// Union 정의 (다형성 지원)
// =============================================

// network_schema.fbs

namespace NetworkData;

table LoginRequest {
    username: string;
    password_hash: string;
}

table MoveRequest {
    position: Vec3;
    velocity: Vec3;
    timestamp: long;
}

table AttackRequest {
    target_id: int;
    skill_id: int;
}

table ChatMessage {
    channel: string;
    message: string;
    sender_id: int;
}

union PacketBody {
    LoginRequest,
    MoveRequest,
    AttackRequest,
    ChatMessage
}

table NetworkPacket {
    sequence: uint;
    timestamp: long;
    body: PacketBody;
}

root_type NetworkPacket;
```

---

## 3. flatc 컴파일러로 C# 코드 생성

### 설치 및 코드 생성

```bash
# flatc 컴파일러 설치 (macOS)
# brew install flatbuffers

# flatc 컴파일러 설치 (Linux)
# apt-get install flatbuffers-compiler

# Windows: GitHub Releases에서 다운로드
# https://github.com/google/flatbuffers/releases

# C# 코드 생성
# flatc --csharp -o Assets/Generated/ game_schema.fbs

# 네임스페이스 지정
# flatc --csharp --gen-onefile -o Assets/Generated/ game_schema.fbs

# 여러 스키마 한번에 컴파일
# flatc --csharp -o Assets/Generated/ game_schema.fbs network_schema.fbs
```

### Unity 프로젝트 설정

```csharp
using UnityEngine;

/// <summary>
/// FlatBuffers Unity 프로젝트 설정 가이드
/// </summary>
public class FlatBuffersSetupGuide : MonoBehaviour
{
    /*
     * Unity 프로젝트에 FlatBuffers 추가하기:
     *
     * 1. NuGet에서 Google.FlatBuffers 패키지 설치
     *    또는 FlatBuffers C# 런타임 소스를 직접 프로젝트에 포함
     *
     * 2. 프로젝트 구조:
     *    Assets/
     *    ├── Plugins/
     *    │   └── FlatBuffers/           ← FlatBuffers 런타임 라이브러리
     *    │       ├── FlatBufferBuilder.cs
     *    │       ├── ByteBuffer.cs
     *    │       ├── Table.cs
     *    │       └── Struct.cs
     *    ├── Schemas/
     *    │   ├── game_schema.fbs        ← 스키마 파일
     *    │   └── network_schema.fbs
     *    ├── Generated/
     *    │   └── GameData/              ← flatc가 생성한 코드
     *    │       ├── Monster.cs
     *    │       ├── Item.cs
     *    │       └── MonsterDatabase.cs
     *    └── Scripts/
     *        └── ...                    ← 게임 로직
     *
     * 3. Editor 스크립트로 자동 컴파일 설정 가능
     */

    void Start()
    {
        Debug.Log("FlatBuffers 설정 완료");
    }
}
```

### 자동 빌드 스크립트 (Editor)

```csharp
#if UNITY_EDITOR
using UnityEngine;
using UnityEditor;
using System.Diagnostics;
using System.IO;

/// <summary>
/// .fbs 파일 변경 시 자동으로 C# 코드를 재생성하는 에디터 유틸리티
/// </summary>
public class FlatBuffersCompiler : EditorWindow
{
    private static string flatcPath = "flatc"; // flatc 실행 파일 경로
    private static string schemaDir = "Assets/Schemas";
    private static string outputDir = "Assets/Generated";

    [MenuItem("Tools/FlatBuffers/Compile All Schemas")]
    public static void CompileAllSchemas()
    {
        if (!Directory.Exists(schemaDir))
        {
            UnityEngine.Debug.LogError($"스키마 디렉터리가 없습니다: {schemaDir}");
            return;
        }

        if (!Directory.Exists(outputDir))
        {
            Directory.CreateDirectory(outputDir);
        }

        string[] schemaFiles = Directory.GetFiles(schemaDir, "*.fbs");

        foreach (string schemaFile in schemaFiles)
        {
            CompileSchema(schemaFile);
        }

        AssetDatabase.Refresh();
        UnityEngine.Debug.Log($"FlatBuffers: {schemaFiles.Length}개 스키마 컴파일 완료");
    }

    private static void CompileSchema(string schemaPath)
    {
        ProcessStartInfo startInfo = new ProcessStartInfo
        {
            FileName = flatcPath,
            Arguments = $"--csharp -o \"{outputDir}\" \"{schemaPath}\"",
            UseShellExecute = false,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            CreateNoWindow = true
        };

        using (Process process = Process.Start(startInfo))
        {
            string output = process.StandardOutput.ReadToEnd();
            string error = process.StandardError.ReadToEnd();
            process.WaitForExit();

            if (process.ExitCode != 0)
            {
                UnityEngine.Debug.LogError($"flatc 에러 ({schemaPath}): {error}");
            }
            else
            {
                UnityEngine.Debug.Log($"컴파일 성공: {Path.GetFileName(schemaPath)}");
            }
        }
    }
}
#endif
```

---

## 4. FlatBufferBuilder 사용법 (직렬화)

### 기본 데이터 직렬화

```csharp
using UnityEngine;
using FlatBuffers;
using GameData;

/// <summary>
/// FlatBufferBuilder를 사용한 데이터 직렬화 예제
/// </summary>
public class FlatBufferSerializeExample : MonoBehaviour
{
    // =============================================
    // 기본 직렬화 흐름
    // =============================================

    public byte[] SerializeMonster()
    {
        // 1. FlatBufferBuilder 생성 (초기 버퍼 크기 지정)
        var builder = new FlatBufferBuilder(1024);

        // -----------------------------------------------
        // 주의: FlatBuffers는 역순으로 데이터를 구성합니다.
        // 먼저 하위 객체(문자열, 벡터 등)를 생성한 후
        // 상위 테이블에서 참조합니다.
        // -----------------------------------------------

        // 2. 문자열은 미리 생성
        var nameOffset = builder.CreateString("고블린");
        var skill1 = builder.CreateString("할퀴기");
        var skill2 = builder.CreateString("돌던지기");

        // 3. 벡터(배열)도 미리 생성
        // StartVector → Add 요소들 → EndVector 패턴
        Monster.StartSkillsVector(builder, 2);
        builder.AddOffset(skill2.Value); // 역순으로 추가
        builder.AddOffset(skill1.Value);
        var skillsOffset = builder.EndVector();

        // 4. 테이블 생성: Start → 필드 추가 → End
        Monster.StartMonster(builder);
        Monster.AddId(builder, 1001);
        Monster.AddName(builder, nameOffset);
        Monster.AddLevel(builder, 5);
        Monster.AddHp(builder, 150);
        Monster.AddPosition(builder, Vec3.CreateVec3(builder, 10.5f, 0f, -3.2f));
        Monster.AddElement(builder, ElementType.Fire);
        Monster.AddSkills(builder, skillsOffset);
        Monster.AddIsBoss(builder, false);
        var monsterOffset = builder.EndTable();

        // 5. 루트 테이블 지정 및 완료
        builder.Finish(monsterOffset.Value);

        // 6. 바이트 배열 추출
        byte[] buffer = builder.SizedByteArray();

        Debug.Log($"직렬화 완료: {buffer.Length} bytes");
        return buffer;
    }

    // =============================================
    // 복합 구조 직렬화 (MonsterDatabase)
    // =============================================

    public byte[] SerializeMonsterDatabase()
    {
        var builder = new FlatBufferBuilder(4096);

        // 몬스터 여러 개 생성
        var monsterOffsets = new Offset<Monster>[3];

        // 몬스터 1: 슬라임
        var slimeName = builder.CreateString("슬라임");
        Monster.StartMonster(builder);
        Monster.AddId(builder, 1);
        Monster.AddName(builder, slimeName);
        Monster.AddLevel(builder, 1);
        Monster.AddHp(builder, 50);
        Monster.AddPosition(builder, Vec3.CreateVec3(builder, 0f, 0f, 0f));
        Monster.AddElement(builder, ElementType.Water);
        monsterOffsets[0] = Monster.EndMonster(builder);

        // 몬스터 2: 고블린
        var goblinName = builder.CreateString("고블린");
        var goblinSkill = builder.CreateString("할퀴기");
        Monster.StartSkillsVector(builder, 1);
        builder.AddOffset(goblinSkill.Value);
        var goblinSkills = builder.EndVector();

        Monster.StartMonster(builder);
        Monster.AddId(builder, 2);
        Monster.AddName(builder, goblinName);
        Monster.AddLevel(builder, 5);
        Monster.AddHp(builder, 150);
        Monster.AddPosition(builder, Vec3.CreateVec3(builder, 10f, 0f, 5f));
        Monster.AddElement(builder, ElementType.Earth);
        Monster.AddSkills(builder, goblinSkills);
        monsterOffsets[1] = Monster.EndMonster(builder);

        // 몬스터 3: 드래곤 (보스)
        var dragonName = builder.CreateString("고대 드래곤");
        Monster.StartMonster(builder);
        Monster.AddId(builder, 3);
        Monster.AddName(builder, dragonName);
        Monster.AddLevel(builder, 50);
        Monster.AddHp(builder, 99999);
        Monster.AddPosition(builder, Vec3.CreateVec3(builder, 100f, 50f, 100f));
        Monster.AddElement(builder, ElementType.Fire);
        Monster.AddIsBoss(builder, true);
        monsterOffsets[2] = Monster.EndMonster(builder);

        // 몬스터 벡터 생성
        var monstersVector = MonsterDatabase.CreateMonstersVector(builder, monsterOffsets);
        var versionStr = builder.CreateString("2025-01-01");

        // MonsterDatabase 테이블 생성
        MonsterDatabase.StartMonsterDatabase(builder);
        MonsterDatabase.AddVersion(builder, 1);
        MonsterDatabase.AddMonsters(builder, monstersVector);
        MonsterDatabase.AddLastUpdated(builder, versionStr);
        var dbOffset = MonsterDatabase.EndMonsterDatabase(builder);

        builder.Finish(dbOffset.Value);

        return builder.SizedByteArray();
    }
}
```

### 재사용 가능한 Builder 패턴

```csharp
using UnityEngine;
using FlatBuffers;
using GameData;
using System;

/// <summary>
/// FlatBufferBuilder를 재사용하여 GC를 줄이는 패턴
/// </summary>
public class ReusableFlatBufferBuilderExample : MonoBehaviour
{
    // Builder를 재사용하여 매 프레임 할당 방지
    private FlatBufferBuilder _reusableBuilder;

    // 직렬화 결과를 담을 재사용 버퍼
    private byte[] _sendBuffer;

    void Awake()
    {
        // 초기 버퍼 크기를 넉넉히 잡으면 재할당 줄임
        _reusableBuilder = new FlatBufferBuilder(2048);
        _sendBuffer = new byte[2048];
    }

    // =============================================
    // 매 프레임 위치 데이터를 직렬화하는 경우
    // =============================================

    public ArraySegment<byte> SerializePosition(Vector3 position, Vector3 velocity)
    {
        // Clear()로 Builder를 재사용 (내부 버퍼는 유지)
        _reusableBuilder.Clear();

        // MoveRequest 직렬화
        var posVec3 = Vec3.CreateVec3(_reusableBuilder,
            position.x, position.y, position.z);
        var velVec3 = Vec3.CreateVec3(_reusableBuilder,
            velocity.x, velocity.y, velocity.z);

        // MoveRequest 생성 (스키마에 정의된 테이블 사용)
        MoveRequest.StartMoveRequest(_reusableBuilder);
        MoveRequest.AddPosition(_reusableBuilder, posVec3);
        MoveRequest.AddVelocity(_reusableBuilder, velVec3);
        MoveRequest.AddTimestamp(_reusableBuilder, DateTimeOffset.UtcNow.ToUnixTimeMilliseconds());
        var moveOffset = MoveRequest.EndMoveRequest(_reusableBuilder);

        _reusableBuilder.Finish(moveOffset.Value);

        // DataBuffer에서 직접 참조 (복사 없이)
        var buf = _reusableBuilder.DataBuffer;
        return new ArraySegment<byte>(
            buf.ToSizedArray(),
            buf.Position,
            buf.Length - buf.Position
        );
    }
}
```

---

## 5. Zero-copy 읽기 (역직렬화)

### 기본 데이터 읽기

```csharp
using UnityEngine;
using FlatBuffers;
using GameData;

/// <summary>
/// FlatBuffers의 핵심: Zero-copy 데이터 접근
/// 바이트 배열을 파싱하지 않고 직접 데이터에 접근합니다.
/// </summary>
public class FlatBufferDeserializeExample : MonoBehaviour
{
    // =============================================
    // 기본 읽기: 객체 생성 없이 바로 접근
    // =============================================

    public void ReadMonster(byte[] buffer)
    {
        // ByteBuffer로 래핑 (파싱 없음, 오프셋만 계산)
        var byteBuffer = new ByteBuffer(buffer);

        // 루트 테이블 접근 (객체 생성 아님, 오프셋 참조)
        var monster = Monster.GetRootAsMonster(byteBuffer);

        // 필드에 직접 접근 (필요한 필드만 읽기)
        Debug.Log($"ID: {monster.Id}");           // int 직접 읽기
        Debug.Log($"이름: {monster.Name}");        // string 접근
        Debug.Log($"레벨: {monster.Level}");       // short 직접 읽기
        Debug.Log($"HP: {monster.Hp}");            // int 직접 읽기
        Debug.Log($"보스 여부: {monster.IsBoss}"); // bool 직접 읽기

        // Struct 접근 (인라인 데이터, 추가 할당 없음)
        var pos = monster.Position;
        if (pos.HasValue)
        {
            Vec3 position = pos.Value;
            Debug.Log($"위치: ({position.X}, {position.Y}, {position.Z})");
        }

        // Enum 접근
        Debug.Log($"속성: {monster.Element}"); // ElementType.Fire 등

        // 벡터(배열) 접근
        int skillCount = monster.SkillsLength;
        for (int i = 0; i < skillCount; i++)
        {
            Debug.Log($"스킬 {i}: {monster.Skills(i)}");
        }
    }

    // =============================================
    // MonsterDatabase 읽기
    // =============================================

    public void ReadMonsterDatabase(byte[] buffer)
    {
        var byteBuffer = new ByteBuffer(buffer);
        var db = MonsterDatabase.GetRootAsMonsterDatabase(byteBuffer);

        Debug.Log($"버전: {db.Version}");
        Debug.Log($"마지막 업데이트: {db.LastUpdated}");
        Debug.Log($"몬스터 수: {db.MonstersLength}");

        // 필요한 몬스터만 접근 (전체를 파싱하지 않음!)
        for (int i = 0; i < db.MonstersLength; i++)
        {
            var monster = db.Monsters(i);
            if (monster.HasValue)
            {
                Monster m = monster.Value;
                Debug.Log($"[{m.Id}] {m.Name} Lv.{m.Level} HP:{m.Hp}");
            }
        }
    }

    // =============================================
    // 핵심: 부분 접근의 이점
    // =============================================

    /// <summary>
    /// 1000개의 몬스터 중 보스만 찾기
    /// JSON: 1000개 전부 파싱 후 필터링
    /// FlatBuffers: 각 몬스터의 is_boss 필드만 직접 읽기
    /// </summary>
    public void FindBossMonsters(byte[] buffer)
    {
        var byteBuffer = new ByteBuffer(buffer);
        var db = MonsterDatabase.GetRootAsMonsterDatabase(byteBuffer);

        for (int i = 0; i < db.MonstersLength; i++)
        {
            var monster = db.Monsters(i);
            if (monster.HasValue && monster.Value.IsBoss)
            {
                // 보스인 경우에만 상세 데이터 접근
                Monster boss = monster.Value;
                Debug.Log($"보스 발견: {boss.Name} (Lv.{boss.Level})");
            }
            // 보스가 아닌 몬스터의 Name, Skills 등은 읽지 않음 → 비용 0
        }
    }
}
```

### Zero-copy의 메모리 이점

```csharp
using UnityEngine;
using FlatBuffers;
using GameData;
using System;

/// <summary>
/// FlatBuffers Zero-copy와 전통적 역직렬화의 메모리 비교
/// </summary>
public class ZeroCopyBenefitExample : MonoBehaviour
{
    // =============================================
    // ❌ 전통적 방식: JSON 역직렬화
    // =============================================

    // JSON 역직렬화 시 발생하는 메모리 할당:
    //
    // 1. string 파싱을 위한 임시 버퍼
    // 2. 각 필드값을 위한 박싱/변환
    // 3. 최종 C# 객체 생성
    // 4. 중첩 객체/배열 각각에 대한 할당
    //
    // 몬스터 1000마리 데이터:
    // - 약 1000개의 MonsterData 객체 생성
    // - 수천 개의 string 할당
    // - 배열/리스트 할당
    // - 총 수 MB의 관리 힙 메모리 사용
    // - GC 스파이크 발생 가능

    [Serializable]
    public class MonsterDataJson
    {
        public int id;
        public string name;
        public int level;
        public int hp;
        public float posX, posY, posZ;
        public string[] skills;    // 배열 할당
        public ItemData[] items;   // 배열 + 객체 할당
    }

    [Serializable]
    public class ItemData
    {
        public int id;
        public string name;
        public string rarity;
    }

    // =============================================
    // ✅ FlatBuffers 방식: Zero-copy
    // =============================================

    // FlatBuffers 접근 시 메모리 할당:
    //
    // 1. ByteBuffer 래핑: ~48 bytes (1회)
    // 2. 데이터 접근: 추가 할당 없음 (오프셋 계산만)
    // 3. string 접근 시에만 string 객체 생성
    //
    // 몬스터 1000마리 데이터:
    // - ByteBuffer 1개만 생성
    // - 개별 몬스터 접근 시 추가 할당 없음
    // - Name 등 string 접근 시에만 할당
    // - GC 압력 최소

    public void DemonstrateZeroCopy(byte[] buffer)
    {
        // 이 한 줄이 "역직렬화"의 전부
        var byteBuffer = new ByteBuffer(buffer);
        var db = MonsterDatabase.GetRootAsMonsterDatabase(byteBuffer);

        // 숫자형 필드 접근: 메모리 할당 0
        int monsterCount = db.MonstersLength;

        for (int i = 0; i < monsterCount; i++)
        {
            var monster = db.Monsters(i);
            if (!monster.HasValue) continue;

            // int, float, bool 등 스칼라 접근: 할당 0
            int id = monster.Value.Id;
            int hp = monster.Value.Hp;
            short level = monster.Value.Level;
            bool isBoss = monster.Value.IsBoss;

            // Vec3 struct 접근: 할당 0 (인라인 데이터)
            var pos = monster.Value.Position;
            if (pos.HasValue)
            {
                float x = pos.Value.X;
                float y = pos.Value.Y;
                float z = pos.Value.Z;
            }

            // string 접근: string 객체 할당 발생
            // 필요한 경우에만 접근하는 것이 좋음
            if (isBoss)
            {
                string name = monster.Value.Name; // 여기서만 할당
                Debug.Log($"보스: {name}");
            }
        }
    }
}
```

---

## 6. 네트워크 패킷에서의 활용

### 게임 네트워크 패킷 처리

```csharp
using UnityEngine;
using FlatBuffers;
using NetworkData;
using System;

/// <summary>
/// FlatBuffers를 활용한 게임 네트워크 패킷 처리
/// </summary>
public class NetworkPacketExample : MonoBehaviour
{
    private FlatBufferBuilder _builder;
    private uint _sequenceNumber;

    void Awake()
    {
        _builder = new FlatBufferBuilder(512);
        _sequenceNumber = 0;
    }

    // =============================================
    // 패킷 전송 (직렬화)
    // =============================================

    /// <summary>
    /// 이동 패킷 생성
    /// </summary>
    public byte[] CreateMovePacket(Vector3 position, Vector3 velocity)
    {
        _builder.Clear();

        // MoveRequest 생성
        MoveRequest.StartMoveRequest(_builder);
        MoveRequest.AddPosition(_builder,
            Vec3.CreateVec3(_builder, position.x, position.y, position.z));
        MoveRequest.AddVelocity(_builder,
            Vec3.CreateVec3(_builder, velocity.x, velocity.y, velocity.z));
        MoveRequest.AddTimestamp(_builder,
            DateTimeOffset.UtcNow.ToUnixTimeMilliseconds());
        var moveOffset = MoveRequest.EndMoveRequest(_builder);

        // NetworkPacket으로 래핑 (Union 사용)
        NetworkPacket.StartNetworkPacket(_builder);
        NetworkPacket.AddSequence(_builder, ++_sequenceNumber);
        NetworkPacket.AddTimestamp(_builder,
            DateTimeOffset.UtcNow.ToUnixTimeMilliseconds());
        NetworkPacket.AddBodyType(_builder, PacketBody.MoveRequest);
        NetworkPacket.AddBody(_builder, moveOffset.Value);
        var packetOffset = NetworkPacket.EndNetworkPacket(_builder);

        _builder.Finish(packetOffset.Value);
        return _builder.SizedByteArray();
    }

    /// <summary>
    /// 공격 패킷 생성
    /// </summary>
    public byte[] CreateAttackPacket(int targetId, int skillId)
    {
        _builder.Clear();

        AttackRequest.StartAttackRequest(_builder);
        AttackRequest.AddTargetId(_builder, targetId);
        AttackRequest.AddSkillId(_builder, skillId);
        var attackOffset = AttackRequest.EndAttackRequest(_builder);

        NetworkPacket.StartNetworkPacket(_builder);
        NetworkPacket.AddSequence(_builder, ++_sequenceNumber);
        NetworkPacket.AddTimestamp(_builder,
            DateTimeOffset.UtcNow.ToUnixTimeMilliseconds());
        NetworkPacket.AddBodyType(_builder, PacketBody.AttackRequest);
        NetworkPacket.AddBody(_builder, attackOffset.Value);
        var packetOffset = NetworkPacket.EndNetworkPacket(_builder);

        _builder.Finish(packetOffset.Value);
        return _builder.SizedByteArray();
    }

    // =============================================
    // 패킷 수신 (역직렬화 - Zero-copy)
    // =============================================

    /// <summary>
    /// 수신한 패킷 처리 (Union 분기)
    /// </summary>
    public void HandleReceivedPacket(byte[] data)
    {
        var byteBuffer = new ByteBuffer(data);
        var packet = NetworkPacket.GetRootAsNetworkPacket(byteBuffer);

        Debug.Log($"패킷 수신 - Seq: {packet.Sequence}, Time: {packet.Timestamp}");

        // Union 타입에 따라 분기
        switch (packet.BodyType)
        {
            case PacketBody.LoginRequest:
                HandleLogin(packet);
                break;

            case PacketBody.MoveRequest:
                HandleMove(packet);
                break;

            case PacketBody.AttackRequest:
                HandleAttack(packet);
                break;

            case PacketBody.ChatMessage:
                HandleChat(packet);
                break;

            default:
                Debug.LogWarning($"알 수 없는 패킷 타입: {packet.BodyType}");
                break;
        }
    }

    private void HandleLogin(NetworkPacket packet)
    {
        var login = packet.Body<LoginRequest>();
        if (login.HasValue)
        {
            Debug.Log($"로그인 요청: {login.Value.Username}");
        }
    }

    private void HandleMove(NetworkPacket packet)
    {
        var move = packet.Body<MoveRequest>();
        if (move.HasValue)
        {
            var pos = move.Value.Position;
            if (pos.HasValue)
            {
                Vector3 position = new Vector3(pos.Value.X, pos.Value.Y, pos.Value.Z);
                Debug.Log($"이동 요청: {position}");
                // transform.position = position; 등으로 적용
            }
        }
    }

    private void HandleAttack(NetworkPacket packet)
    {
        var attack = packet.Body<AttackRequest>();
        if (attack.HasValue)
        {
            Debug.Log($"공격 요청: 대상={attack.Value.TargetId}, " +
                      $"스킬={attack.Value.SkillId}");
        }
    }

    private void HandleChat(NetworkPacket packet)
    {
        var chat = packet.Body<ChatMessage>();
        if (chat.HasValue)
        {
            Debug.Log($"[{chat.Value.Channel}] " +
                      $"Player{chat.Value.SenderId}: {chat.Value.Message}");
        }
    }
}
```

---

## 7. 게임 설정 데이터 로드

### ScriptableObject 대체

```csharp
using UnityEngine;
using FlatBuffers;
using GameData;
using System.IO;

/// <summary>
/// FlatBuffers로 게임 설정 데이터를 로드하는 예제
/// JSON/ScriptableObject 대비 빠른 로드와 적은 메모리 사용
/// </summary>
public class GameDataLoaderExample : MonoBehaviour
{
    private MonsterDatabase? _monsterDb;
    private byte[] _rawBuffer; // 원본 버퍼 유지 (Zero-copy 위해)

    // =============================================
    // 데이터 로드
    // =============================================

    void Awake()
    {
        LoadMonsterDatabase();
    }

    private void LoadMonsterDatabase()
    {
        // StreamingAssets에서 바이너리 파일 로드
        string path = Path.Combine(Application.streamingAssetsPath, "data/monsters.bin");

        if (!File.Exists(path))
        {
            Debug.LogError($"데이터 파일 없음: {path}");
            return;
        }

        // 파일 전체를 한 번에 읽기
        _rawBuffer = File.ReadAllBytes(path);

        // ByteBuffer 래핑만으로 "역직렬화" 완료
        var byteBuffer = new ByteBuffer(_rawBuffer);
        _monsterDb = MonsterDatabase.GetRootAsMonsterDatabase(byteBuffer);

        Debug.Log($"몬스터 데이터 로드 완료: {_monsterDb.Value.MonstersLength}개");
    }

    // =============================================
    // 데이터 조회 (Zero-copy 접근)
    // =============================================

    /// <summary>
    /// ID로 몬스터 검색 (선형 탐색)
    /// </summary>
    public Monster? FindMonsterById(int id)
    {
        if (!_monsterDb.HasValue) return null;

        var db = _monsterDb.Value;

        for (int i = 0; i < db.MonstersLength; i++)
        {
            var monster = db.Monsters(i);
            if (monster.HasValue && monster.Value.Id == id)
            {
                return monster.Value;
            }
        }

        return null;
    }

    /// <summary>
    /// 레벨 범위로 몬스터 필터링
    /// </summary>
    public void GetMonstersByLevelRange(int minLevel, int maxLevel)
    {
        if (!_monsterDb.HasValue) return;

        var db = _monsterDb.Value;

        for (int i = 0; i < db.MonstersLength; i++)
        {
            var monster = db.Monsters(i);
            if (!monster.HasValue) continue;

            // 레벨만 읽고 범위 확인 (다른 필드는 접근하지 않음)
            short level = monster.Value.Level;
            if (level >= minLevel && level <= maxLevel)
            {
                Debug.Log($"[Lv.{level}] {monster.Value.Name} HP:{monster.Value.Hp}");
            }
        }
    }

    // =============================================
    // 원본 버퍼 해제
    // =============================================

    void OnDestroy()
    {
        // 중요: _rawBuffer가 GC되면 Zero-copy 접근 불가
        // MonsterDatabase가 참조하는 동안 유지해야 함
        _rawBuffer = null;
    }
}
```

### Addressables와 연동

```csharp
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;
using FlatBuffers;
using GameData;
using System.Collections;

/// <summary>
/// Addressable Assets으로 FlatBuffers 데이터 비동기 로드
/// </summary>
public class AddressableFlatBufferLoaderExample : MonoBehaviour
{
    private byte[] _rawBuffer;

    // =============================================
    // Addressables를 통한 비동기 로드
    // =============================================

    public IEnumerator LoadDataAsync(string address)
    {
        // TextAsset으로 바이너리 파일 로드
        var handle = Addressables.LoadAssetAsync<TextAsset>(address);
        yield return handle;

        if (handle.Status == AsyncOperationStatus.Succeeded)
        {
            _rawBuffer = handle.Result.bytes;

            var byteBuffer = new ByteBuffer(_rawBuffer);
            var db = MonsterDatabase.GetRootAsMonsterDatabase(byteBuffer);

            Debug.Log($"Addressables 로드 완료: {db.MonstersLength}개 몬스터");

            // TextAsset 해제 (bytes는 이미 복사됨)
            Addressables.Release(handle);
        }
        else
        {
            Debug.LogError($"데이터 로드 실패: {address}");
        }
    }
}
```

---

## 8. Unity에서의 실전 패턴

### 매니저 클래스 패턴

```csharp
using UnityEngine;
using FlatBuffers;
using GameData;
using System.Collections.Generic;
using System.IO;

/// <summary>
/// FlatBuffers 데이터를 관리하는 중앙 매니저
/// </summary>
public class FlatBufferDataManager : MonoBehaviour
{
    public static FlatBufferDataManager Instance { get; private set; }

    // 원본 버퍼들 (Zero-copy를 위해 유지)
    private Dictionary<string, byte[]> _buffers = new Dictionary<string, byte[]>();

    // 캐시된 루트 테이블들
    private MonsterDatabase? _monsterDb;

    void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);
            LoadAllData();
        }
        else
        {
            Destroy(gameObject);
        }
    }

    // =============================================
    // 데이터 일괄 로드
    // =============================================

    private void LoadAllData()
    {
        _monsterDb = LoadFlatBuffer<MonsterDatabase>(
            "data/monsters.bin",
            MonsterDatabase.GetRootAsMonsterDatabase
        );

        Debug.Log("모든 게임 데이터 로드 완료");
    }

    private T? LoadFlatBuffer<T>(string relativePath,
        System.Func<ByteBuffer, T> getRootFunc) where T : struct
    {
        string path = Path.Combine(Application.streamingAssetsPath, relativePath);

        if (!File.Exists(path))
        {
            Debug.LogError($"파일 없음: {path}");
            return null;
        }

        byte[] buffer = File.ReadAllBytes(path);
        _buffers[relativePath] = buffer; // 버퍼 유지

        var byteBuffer = new ByteBuffer(buffer);
        return getRootFunc(byteBuffer);
    }

    // =============================================
    // 공개 API
    // =============================================

    public Monster? GetMonster(int id)
    {
        if (!_monsterDb.HasValue) return null;

        var db = _monsterDb.Value;
        for (int i = 0; i < db.MonstersLength; i++)
        {
            var monster = db.Monsters(i);
            if (monster.HasValue && monster.Value.Id == id)
                return monster.Value;
        }
        return null;
    }

    public int GetMonsterCount()
    {
        return _monsterDb?.MonstersLength ?? 0;
    }

    // =============================================
    // 정리
    // =============================================

    void OnDestroy()
    {
        _buffers.Clear();
        _monsterDb = null;
    }
}

// =============================================
// 사용 예시
// =============================================

public class GameplayUsageExample : MonoBehaviour
{
    void Start()
    {
        // 매니저를 통해 데이터 접근
        var monster = FlatBufferDataManager.Instance.GetMonster(1001);

        if (monster.HasValue)
        {
            Debug.Log($"몬스터: {monster.Value.Name}");
            Debug.Log($"HP: {monster.Value.Hp}");

            var pos = monster.Value.Position;
            if (pos.HasValue)
            {
                transform.position = new Vector3(
                    pos.Value.X, pos.Value.Y, pos.Value.Z
                );
            }
        }
    }
}
```

### 멀티스레드 안전 접근

```csharp
using UnityEngine;
using FlatBuffers;
using GameData;
using System.Threading;

/// <summary>
/// FlatBuffers의 읽기 전용 특성을 활용한 멀티스레드 접근
/// 같은 버퍼를 여러 스레드에서 동시에 읽을 수 있음
/// </summary>
public class MultithreadFlatBufferExample : MonoBehaviour
{
    private byte[] _sharedBuffer;

    void Start()
    {
        // 버퍼를 한 번 로드
        _sharedBuffer = LoadBuffer();

        // 여러 스레드에서 동시에 읽기 가능
        // (각 스레드가 별도의 ByteBuffer 인스턴스 사용)
        for (int i = 0; i < 4; i++)
        {
            int threadIndex = i;
            ThreadPool.QueueUserWorkItem(_ => ReadOnThread(threadIndex));
        }
    }

    private void ReadOnThread(int threadIndex)
    {
        // 각 스레드마다 별도의 ByteBuffer 생성
        // 같은 byte[]를 참조하지만 읽기 위치는 독립적
        var byteBuffer = new ByteBuffer(_sharedBuffer);
        var db = MonsterDatabase.GetRootAsMonsterDatabase(byteBuffer);

        // 스레드별로 다른 범위의 몬스터 처리
        int total = db.MonstersLength;
        int perThread = total / 4;
        int start = threadIndex * perThread;
        int end = (threadIndex == 3) ? total : start + perThread;

        for (int i = start; i < end; i++)
        {
            var monster = db.Monsters(i);
            if (monster.HasValue)
            {
                // 읽기 전용 접근은 스레드 안전
                int hp = monster.Value.Hp;
                short level = monster.Value.Level;
                // 계산 로직...
            }
        }

        Debug.Log($"스레드 {threadIndex}: {start}~{end - 1} 처리 완료");
    }

    private byte[] LoadBuffer()
    {
        // 실제로는 파일에서 로드
        var builder = new FlatBufferBuilder(1024);
        // ... 데이터 구성 ...
        return builder.SizedByteArray();
    }
}
```

---

## 9. 성능 최적화 패턴

### 메모리 풀링과 결합

```csharp
using UnityEngine;
using FlatBuffers;
using System;
using System.Collections.Generic;

/// <summary>
/// FlatBufferBuilder 풀링으로 빈번한 직렬화 최적화
/// </summary>
public class FlatBufferPoolExample : MonoBehaviour
{
    // Builder 풀 (직렬화가 빈번할 때 사용)
    private static readonly Stack<FlatBufferBuilder> _builderPool = new Stack<FlatBufferBuilder>();
    private static readonly object _poolLock = new object();

    public static FlatBufferBuilder RentBuilder(int initialSize = 1024)
    {
        lock (_poolLock)
        {
            if (_builderPool.Count > 0)
            {
                var builder = _builderPool.Pop();
                builder.Clear();
                return builder;
            }
        }

        return new FlatBufferBuilder(initialSize);
    }

    public static void ReturnBuilder(FlatBufferBuilder builder)
    {
        lock (_poolLock)
        {
            if (_builderPool.Count < 10) // 최대 풀 크기
            {
                _builderPool.Push(builder);
            }
        }
    }

    // =============================================
    // 사용 예시
    // =============================================

    public byte[] SerializeWithPool()
    {
        var builder = RentBuilder();

        try
        {
            // 직렬화 작업...
            var nameOffset = builder.CreateString("데이터");
            // ... 더 많은 직렬화 ...

            builder.Finish(nameOffset.Value);
            return builder.SizedByteArray();
        }
        finally
        {
            ReturnBuilder(builder);
        }
    }
}
```

### 벤치마크 비교

```csharp
using UnityEngine;
using System.Diagnostics;
using System.Text;

/// <summary>
/// FlatBuffers vs JSON 성능 비교 개념 예제
/// </summary>
public class FlatBufferBenchmarkExample : MonoBehaviour
{
    /*
     * 일반적인 벤치마크 결과 (몬스터 1000마리 데이터):
     *
     * ┌─────────────────────┬───────────┬────────────┬──────────────┐
     * │ 측정 항목            │ JSON      │ ProtoBuf   │ FlatBuffers  │
     * ├─────────────────────┼───────────┼────────────┼──────────────┤
     * │ 직렬화 시간          │ 15ms      │ 3ms        │ 5ms          │
     * │ 역직렬화 시간        │ 25ms      │ 5ms        │ 0.001ms (*)  │
     * │ 직렬화된 크기        │ 850KB     │ 320KB      │ 450KB        │
     * │ 역직렬화 메모리 할당 │ 2.1MB     │ 1.5MB      │ ~0KB (*)     │
     * │ 개별 필드 접근       │ 즉시(**)  │ 즉시(**)   │ 즉시         │
     * │ GC 발생             │ 높음      │ 중간       │ 거의 없음    │
     * └─────────────────────┴───────────┴────────────┴──────────────┘
     *
     * (*) FlatBuffers는 역직렬화가 사실상 없음 (ByteBuffer 래핑만)
     * (**) JSON/ProtoBuf는 전체 역직렬화 후 접근 가능
     *
     * 핵심 차이:
     * - JSON: 전체 파싱 필수 → 일부 데이터만 필요해도 전체 비용 발생
     * - ProtoBuf: 전체 언패킹 필수 → JSON보다 빠르지만 여전히 할당 발생
     * - FlatBuffers: 필요한 필드만 접근 → 접근한 만큼만 비용 발생
     */

    void Start()
    {
        UnityEngine.Debug.Log("=== 직렬화 성능 비교 ===");
        UnityEngine.Debug.Log("FlatBuffers의 핵심 장점:");
        UnityEngine.Debug.Log("1. 역직렬화 시간이 사실상 0");
        UnityEngine.Debug.Log("2. 메모리 할당이 거의 없음");
        UnityEngine.Debug.Log("3. GC 스파이크 방지");
        UnityEngine.Debug.Log("4. 필요한 데이터만 접근 가능 (Lazy Access)");
    }
}
```

---

## 10. 스키마 진화 (Schema Evolution)

### 호환성 유지 규칙

```
// =============================================
// FlatBuffers 스키마 진화 규칙
// =============================================

// game_schema_v1.fbs (초기 버전)
table Monster {
    id: int;
    name: string;
    hp: int;
}

// game_schema_v2.fbs (확장 버전)
// ✅ 안전한 변경:
table Monster {
    id: int;
    name: string;
    hp: int;
    mp: int = 0;          // 새 필드는 끝에 추가 (기본값 지정)
    element: byte = 0;    // 기본값이 있으므로 이전 데이터와 호환
}

// ❌ 위험한 변경 (절대 하지 말 것):
// - 기존 필드 제거
// - 기존 필드 타입 변경
// - 기존 필드 순서 변경
// - 필드 ID 재사용

// =============================================
// 안전한 스키마 변경 가이드
// =============================================

// ✅ 가능한 변경:
// 1. 테이블 끝에 새 필드 추가 (기본값 필수)
// 2. 새 테이블/enum 추가
// 3. 사용하지 않는 필드를 deprecated로 표시
// 4. enum에 새 값 추가

// ❌ 불가능한 변경:
// 1. 기존 필드의 타입 변경
// 2. 기존 필드 제거 (deprecated만 가능)
// 3. 필드 순서/번호 변경
// 4. struct의 필드 변경 (struct는 진화 불가)
// 5. required 필드를 나중에 추가
```

### deprecated 필드 처리

```csharp
using UnityEngine;
using FlatBuffers;

/// <summary>
/// 스키마 진화 시 버전 호환성 처리 예제
/// </summary>
public class SchemaEvolutionExample : MonoBehaviour
{
    /*
     * 스키마 예시:
     *
     * table Player {
     *     id: int;
     *     name: string;
     *     old_score: int (deprecated);  // v1에서 사용, v2에서 deprecated
     *     level: int;                   // v2에서 추가
     *     experience: long = 0;         // v2에서 추가
     * }
     *
     * deprecated 필드는:
     * - 바이너리에서 공간을 차지하지만 접근자가 생성되지 않음
     * - 이전 버전 데이터를 읽을 때 호환성 유지
     * - 필드 번호를 재사용할 수 없으므로 자리를 보존
     */

    void Start()
    {
        Debug.Log("스키마 진화 핵심 원칙:");
        Debug.Log("1. 새 필드는 항상 테이블 끝에 추가");
        Debug.Log("2. 기본값을 항상 지정");
        Debug.Log("3. 기존 필드는 제거하지 말고 deprecated 표시");
        Debug.Log("4. struct는 변경 불가 (table 사용 권장)");
    }
}
```

---

## 주의사항

### ❌ 자주 하는 실수들

```csharp
using UnityEngine;
using FlatBuffers;
using GameData;

/// <summary>
/// FlatBuffers 사용 시 흔한 실수와 주의사항
/// </summary>
public class FlatBufferPitfallsExample : MonoBehaviour
{
    // =============================================
    // ❌ 실수 1: 원본 버퍼 해제 후 접근
    // =============================================

    /*
    Monster? LoadAndReturn_Wrong()
    {
        byte[] buffer = File.ReadAllBytes("data.bin");
        var bb = new ByteBuffer(buffer);
        var monster = Monster.GetRootAsMonster(bb);

        // buffer가 이 메서드 스코프를 벗어나면 GC 대상
        return monster;
        // monster를 나중에 사용하면 잘못된 데이터 접근!
    }
    */

    // ✅ 올바른 방법: 버퍼를 유지
    private byte[] _persistentBuffer;

    void LoadAndStore_Correct()
    {
        _persistentBuffer = System.IO.File.ReadAllBytes("data.bin");
        var bb = new ByteBuffer(_persistentBuffer);
        // monster가 유효한 동안 _persistentBuffer 유지
    }

    // =============================================
    // ❌ 실수 2: 직렬화 순서 무시
    // =============================================

    /*
    void WrongOrder(FlatBufferBuilder builder)
    {
        // 테이블을 먼저 시작하고 문자열 생성 시도 → 에러!
        Monster.StartMonster(builder);
        var name = builder.CreateString("고블린"); // 에러 발생!
        Monster.AddName(builder, name);
        Monster.EndMonster(builder);
    }
    */

    // ✅ 올바른 방법: 하위 객체를 먼저 생성
    void CorrectOrder(FlatBufferBuilder builder)
    {
        // 1. 문자열, 벡터 등 하위 객체 먼저 생성
        var name = builder.CreateString("고블린");

        // 2. 그 다음 테이블 시작
        Monster.StartMonster(builder);
        Monster.AddName(builder, name);
        Monster.EndMonster(builder);
    }

    // =============================================
    // ❌ 실수 3: FlatBuffer 데이터 수정 시도
    // =============================================

    /*
    void TryModify_Wrong(byte[] buffer)
    {
        var bb = new ByteBuffer(buffer);
        var monster = Monster.GetRootAsMonster(bb);

        // FlatBuffers는 읽기 전용!
        // monster.Hp = 200; // 이런 세터가 없음!
        // 데이터 수정 불가
    }
    */

    // ✅ 수정이 필요하면 새로 직렬화
    byte[] ModifyAndReserialize(byte[] buffer)
    {
        // 1. 기존 데이터 읽기
        var bb = new ByteBuffer(buffer);
        var oldMonster = Monster.GetRootAsMonster(bb);

        // 2. 수정된 값으로 새로 직렬화
        var builder = new FlatBufferBuilder(buffer.Length);
        var name = builder.CreateString(oldMonster.Name);

        Monster.StartMonster(builder);
        Monster.AddId(builder, oldMonster.Id);
        Monster.AddName(builder, name);
        Monster.AddLevel(builder, oldMonster.Level);
        Monster.AddHp(builder, 200); // 수정된 HP
        var newMonster = Monster.EndMonster(builder);

        builder.Finish(newMonster.Value);
        return builder.SizedByteArray();
    }

    // =============================================
    // ❌ 실수 4: 매 프레임 SizedByteArray() 호출
    // =============================================

    /*
    void EveryFrame_Wrong()
    {
        // SizedByteArray()는 매번 새 배열 생성 → GC 압력
        byte[] data = builder.SizedByteArray(); // 매 프레임 할당!
        SendToServer(data);
    }
    */

    // ✅ DataBuffer의 ArraySegment 활용
    void EveryFrame_Correct(FlatBufferBuilder builder)
    {
        var buf = builder.DataBuffer;
        // 복사 없이 기존 버퍼의 유효 영역만 참조
        var segment = new System.ArraySegment<byte>(
            buf.ToSizedArray(),
            buf.Position,
            buf.Length - buf.Position
        );
        // SendToServer(segment);
    }

    // =============================================
    // ❌ 실수 5: Nullable 체크 누락
    // =============================================

    /*
    void NoNullCheck_Wrong(Monster monster)
    {
        // Position이 설정되지 않은 경우 예외 발생 가능
        float x = monster.Position.Value.X;
    }
    */

    // ✅ 항상 HasValue 체크
    void WithNullCheck_Correct(Monster monster)
    {
        var pos = monster.Position;
        if (pos.HasValue)
        {
            float x = pos.Value.X;
            float y = pos.Value.Y;
            float z = pos.Value.Z;
        }
    }

    void SendToServer(byte[] data) { }
}
```

---

## 베스트 프랙티스

### 1. 버퍼 수명 관리

```csharp
// ✅ 원본 byte[]를 FlatBuffer 접근자와 동일한 수명으로 유지
public class DataHolder
{
    private byte[] _buffer;       // 원본 유지
    private MonsterDatabase _db;  // Zero-copy 접근자

    public void Load(byte[] data)
    {
        _buffer = data;
        _db = MonsterDatabase.GetRootAsMonsterDatabase(new ByteBuffer(_buffer));
    }

    public void Unload()
    {
        _buffer = null; // 접근자도 동시에 무효화
    }
}
```

### 2. Builder 재사용

```csharp
// ✅ FlatBufferBuilder를 재사용하여 메모리 할당 최소화
public class NetworkSender
{
    private readonly FlatBufferBuilder _builder = new FlatBufferBuilder(1024);

    public byte[] Serialize()
    {
        _builder.Clear(); // 내부 버퍼 재사용
        // ... 직렬화 로직 ...
        return _builder.SizedByteArray();
    }
}
```

### 3. 적절한 스키마 설계

```
// ✅ 자주 함께 접근하는 데이터는 같은 테이블에
table MonsterCombatInfo {
    hp: int;
    attack: int;
    defense: int;
    speed: float;
}

// ✅ 큰 데이터는 별도 테이블로 분리
table MonsterVisualInfo {
    model_path: string;
    texture_paths: [string];
    animation_data: [ubyte];
}

// ✅ 고정 크기 수치 데이터는 struct 사용 (더 빠름)
struct Transform {
    pos_x: float;
    pos_y: float;
    pos_z: float;
    rot_x: float;
    rot_y: float;
    rot_z: float;
    rot_w: float;
}
```

### 4. 직렬화 순서 규칙

```csharp
// ✅ FlatBuffers 직렬화 순서 체크리스트:
//
// 1단계: 모든 string 생성
// 2단계: 모든 하위 테이블 생성
// 3단계: 모든 벡터 생성
// 4단계: 루트 테이블 생성
// 5단계: Finish() 호출
//
// 절대 테이블 Start~End 사이에서 다른 객체를 생성하지 말 것!
```

### 5. 사용 시나리오별 가이드

```
// ✅ FlatBuffers가 적합한 경우:
// - 네트워크 패킷 (낮은 지연, 적은 메모리)
// - 게임 설정 데이터 (빠른 로드, 부분 접근)
// - 대용량 데이터 (몬스터 DB, 아이템 DB)
// - 실시간 동기화 (위치, 상태)
// - GC가 민감한 환경 (모바일, VR)

// ❌ FlatBuffers가 부적합한 경우:
// - 데이터를 자주 수정해야 하는 경우
// - 사람이 읽을 수 있는 형식이 필요한 경우 (디버깅용)
// - 스키마가 매우 자주 변경되는 프로토타이핑 단계
// - 매우 작은 데이터 (오버헤드가 더 클 수 있음)
// - 동적 구조가 필요한 경우 (스키마 없이 필드 추가)
```

---

## 참고 자료

- [Google FlatBuffers 공식 문서](https://google.github.io/flatbuffers/)
- [FlatBuffers GitHub Repository](https://github.com/google/flatbuffers)
- [FlatBuffers C# Tutorial](https://google.github.io/flatbuffers/flatbuffers_guide_tutorial.html)
- [FlatBuffers Schema (.fbs) 문법](https://google.github.io/flatbuffers/flatbuffers_guide_writing_schema.html)
- [FlatBuffers 벤치마크](https://google.github.io/flatbuffers/flatbuffers_benchmarks.html)
- [Unity에서 FlatBuffers 사용하기](https://github.com/nicloay/FlatBuffersUnity)

---

## 다음 섹션

[33. gRPC](./33-grpc.md)
