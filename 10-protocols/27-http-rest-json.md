# 27. HTTP/REST & JSON

## 개요

REST(Representational State Transfer)는 웹 기반 API의 사실상 표준 아키텍처 스타일이며, JSON(JavaScript Object Notation)은 REST API에서 가장 널리 사용되는 데이터 교환 형식입니다. Unity/C# 환경에서는 `JsonUtility`, `Newtonsoft.Json`, `System.Text.Json` 등 다양한 JSON 라이브러리를 활용하여 REST API와 통신할 수 있습니다. 이 섹션에서는 REST API의 기초 개념부터 Unity에서의 실전 구현 패턴까지 다룹니다.

---

## 1. REST API 기초 개념

### HTTP 메서드 (CRUD 매핑)

```csharp
using UnityEngine;

/// <summary>
/// REST API의 핵심 HTTP 메서드와 CRUD 연산 매핑
/// </summary>
public class RestMethodReference : MonoBehaviour
{
    // =============================================
    // HTTP 메서드 정리
    // =============================================
    //
    // | 메서드  | CRUD   | 설명                    | 멱등성 | 안전성 |
    // |---------|--------|-------------------------|--------|--------|
    // | GET     | Read   | 리소스 조회              | O      | O      |
    // | POST    | Create | 리소스 생성              | X      | X      |
    // | PUT     | Update | 리소스 전체 교체          | O      | X      |
    // | PATCH   | Update | 리소스 부분 수정          | X      | X      |
    // | DELETE  | Delete | 리소스 삭제              | O      | X      |
    // | HEAD    | -      | 헤더만 조회 (본문 없음)   | O      | O      |
    // | OPTIONS | -      | 지원 메서드 확인 (CORS)   | O      | O      |

    // 멱등성(Idempotent): 동일 요청을 여러 번 보내도 결과가 같음
    // 안전성(Safe): 서버 상태를 변경하지 않음
}
```

### HTTP 상태 코드

```csharp
using System.Collections.Generic;

/// <summary>
/// REST API에서 자주 사용하는 HTTP 상태 코드 분류
/// </summary>
public static class HttpStatusReference
{
    // =============================================
    // 2xx: 성공
    // =============================================
    // 200 OK              - 요청 성공 (일반적인 성공)
    // 201 Created         - 리소스 생성 성공 (POST 응답)
    // 204 No Content      - 성공했으나 본문 없음 (DELETE 응답)

    // =============================================
    // 3xx: 리다이렉션
    // =============================================
    // 301 Moved Permanently  - 영구 이동
    // 304 Not Modified       - 캐시 유효 (ETag/Last-Modified)

    // =============================================
    // 4xx: 클라이언트 에러
    // =============================================
    // 400 Bad Request        - 잘못된 요청 형식
    // 401 Unauthorized       - 인증 필요
    // 403 Forbidden          - 권한 없음
    // 404 Not Found          - 리소스 없음
    // 409 Conflict           - 리소스 충돌
    // 422 Unprocessable      - 유효성 검증 실패
    // 429 Too Many Requests  - 요청 빈도 초과 (Rate Limiting)

    // =============================================
    // 5xx: 서버 에러
    // =============================================
    // 500 Internal Server Error  - 서버 내부 오류
    // 502 Bad Gateway            - 게이트웨이 오류
    // 503 Service Unavailable    - 서비스 일시 중단

    public static bool IsSuccess(long statusCode) => statusCode >= 200 && statusCode < 300;
    public static bool IsClientError(long statusCode) => statusCode >= 400 && statusCode < 500;
    public static bool IsServerError(long statusCode) => statusCode >= 500 && statusCode < 600;
    public static bool IsRetryable(long statusCode) => statusCode == 429 || statusCode >= 500;
}
```

### 핵심 HTTP 헤더

```csharp
using UnityEngine.Networking;

/// <summary>
/// REST API에서 자주 사용하는 HTTP 헤더 설정
/// </summary>
public static class HttpHeaderHelper
{
    // =============================================
    // 요청 헤더 설정 유틸리티
    // =============================================

    /// <summary>JSON 요청/응답 헤더 설정</summary>
    public static void SetJsonHeaders(UnityWebRequest request)
    {
        request.SetRequestHeader("Content-Type", "application/json; charset=utf-8");
        request.SetRequestHeader("Accept", "application/json");
    }

    /// <summary>Bearer 토큰 인증 헤더</summary>
    public static void SetBearerAuth(UnityWebRequest request, string token)
    {
        request.SetRequestHeader("Authorization", $"Bearer {token}");
    }

    /// <summary>캐시 제어 헤더</summary>
    public static void SetCacheControl(UnityWebRequest request, string directive = "no-cache")
    {
        request.SetRequestHeader("Cache-Control", directive);
    }

    /// <summary>API 버전 헤더</summary>
    public static void SetApiVersion(UnityWebRequest request, string version)
    {
        request.SetRequestHeader("Accept", $"application/vnd.myapi.v{version}+json");
    }

    // 주요 헤더 정리:
    // Content-Type     : 요청 본문의 미디어 타입
    // Accept           : 클라이언트가 원하는 응답 형식
    // Authorization    : 인증 정보 (Bearer, Basic 등)
    // Cache-Control    : 캐시 정책
    // ETag / If-None-Match : 조건부 요청 (캐시 검증)
    // X-Request-ID     : 요청 추적용 고유 ID
}
```

---

## 2. JsonUtility (Unity 내장 JSON)

Unity에 기본 내장된 JSON 직렬화 도구로, 별도의 패키지 설치 없이 사용할 수 있습니다. 성능이 뛰어나지만 기능이 제한적입니다.

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;

public class JsonUtilityExample : MonoBehaviour
{
    // =============================================
    // 기본 데이터 클래스 (반드시 [Serializable] 필요)
    // =============================================

    [Serializable]
    public class PlayerData
    {
        public string playerName;
        public int level;
        public float health;
        public bool isActive;
        public Vector3 position;       // Unity 타입 지원
        public Quaternion rotation;    // Unity 타입 지원
        public Color skinColor;        // Unity 타입 지원
    }

    // =============================================
    // 직렬화 (객체 → JSON)
    // =============================================

    private void SerializeExample()
    {
        PlayerData player = new PlayerData
        {
            playerName = "전사",
            level = 42,
            health = 95.5f,
            isActive = true,
            position = new Vector3(1.0f, 2.0f, 3.0f),
            rotation = Quaternion.identity,
            skinColor = Color.red
        };

        // 기본 직렬화
        string json = JsonUtility.ToJson(player);
        Debug.Log($"JSON: {json}");

        // prettyPrint 옵션으로 보기 좋게 출력
        string prettyJson = JsonUtility.ToJson(player, true);
        Debug.Log($"Pretty JSON:\n{prettyJson}");
    }

    // =============================================
    // 역직렬화 (JSON → 객체)
    // =============================================

    private void DeserializeExample()
    {
        string json = "{\"playerName\":\"전사\",\"level\":42,\"health\":95.5}";

        // 새 객체 생성
        PlayerData player = JsonUtility.FromJson<PlayerData>(json);
        Debug.Log($"이름: {player.playerName}, 레벨: {player.level}");

        // 기존 객체에 덮어쓰기 (GC 할당 감소)
        PlayerData existingPlayer = new PlayerData();
        JsonUtility.FromJsonOverwrite(json, existingPlayer);
        Debug.Log($"덮어쓴 이름: {existingPlayer.playerName}");
    }

    // =============================================
    // 중첩 객체
    // =============================================

    [Serializable]
    public class Inventory
    {
        public string ownerName;
        public InventorySlot[] slots;  // 배열 지원
    }

    [Serializable]
    public class InventorySlot
    {
        public string itemName;
        public int quantity;
    }

    private void NestedObjectExample()
    {
        Inventory inventory = new Inventory
        {
            ownerName = "전사",
            slots = new InventorySlot[]
            {
                new InventorySlot { itemName = "체력 포션", quantity = 5 },
                new InventorySlot { itemName = "마나 포션", quantity = 3 }
            }
        };

        string json = JsonUtility.ToJson(inventory, true);
        Debug.Log(json);

        Inventory restored = JsonUtility.FromJson<Inventory>(json);
        Debug.Log($"슬롯 수: {restored.slots.Length}");
    }

    // =============================================
    // ⚠️ JsonUtility의 제한 사항
    // =============================================

    [Serializable]
    public class LimitationDemo
    {
        // ✅ 지원: 기본 타입, 배열, Unity 타입
        public int score;
        public string[] tags;
        public Vector3 position;

        // ❌ 미지원: Dictionary
        // public Dictionary<string, int> stats;  // 직렬화 불가!

        // ❌ 미지원: 프로퍼티
        // public int Score { get; set; }          // 무시됨!

        // ❌ 미지원: 다형성 (상속된 타입)
        // ❌ 미지원: null 값 (기본값으로 대체됨)
        // ❌ 미지원: 최상위 배열 직렬화
    }

    // =============================================
    // Dictionary 직렬화 우회 방법
    // =============================================

    [Serializable]
    public class SerializableDictionary
    {
        public List<string> keys = new List<string>();
        public List<int> values = new List<int>();

        public void Add(string key, int value)
        {
            keys.Add(key);
            values.Add(value);
        }

        public Dictionary<string, int> ToDictionary()
        {
            var dict = new Dictionary<string, int>();
            for (int i = 0; i < keys.Count; i++)
            {
                dict[keys[i]] = values[i];
            }
            return dict;
        }
    }

    // =============================================
    // 최상위 배열 직렬화 우회
    // =============================================

    [Serializable]
    public class ArrayWrapper<T>
    {
        public T[] items;
    }

    private void TopLevelArrayExample()
    {
        // ❌ 직접 배열 직렬화는 불가
        // string json = JsonUtility.ToJson(new int[] {1, 2, 3});

        // ✅ 래퍼 클래스 사용
        var wrapper = new ArrayWrapper<InventorySlot>
        {
            items = new InventorySlot[]
            {
                new InventorySlot { itemName = "검", quantity = 1 },
                new InventorySlot { itemName = "방패", quantity = 1 }
            }
        };

        string json = JsonUtility.ToJson(wrapper);
        Debug.Log(json);

        // 서버에서 배열 JSON 수신 시: {"items": [...]} 래핑 필요
        string serverJson = "[{\"itemName\":\"검\"},{\"itemName\":\"방패\"}]";
        string wrappedJson = $"{{\"items\":{serverJson}}}";
        var result = JsonUtility.FromJson<ArrayWrapper<InventorySlot>>(wrappedJson);
    }
}
```

---

## 3. Newtonsoft.Json (Json.NET) - Unity에서 사용법

Unity 환경에서 가장 기능이 풍부한 JSON 라이브러리입니다. Unity 2020+에서는 패키지 매니저를 통해 공식 지원됩니다.

```csharp
// 설치 방법:
// 1. Unity Package Manager → "com.unity.nuget.newtonsoft-json" 검색
// 2. 또는 manifest.json에 직접 추가:
//    "com.unity.nuget.newtonsoft-json": "3.2.1"

using UnityEngine;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;
using Newtonsoft.Json.Converters;
using Newtonsoft.Json.Serialization;
using System;
using System.Collections.Generic;

public class NewtonsoftJsonExample : MonoBehaviour
{
    // =============================================
    // 기본 직렬화/역직렬화
    // =============================================

    public class GameConfig
    {
        // 프로퍼티 지원
        [JsonProperty("server_url")]            // JSON 키 이름 커스터마이즈
        public string ServerUrl { get; set; }

        [JsonProperty("max_players")]
        public int MaxPlayers { get; set; }

        [JsonProperty("features")]
        public Dictionary<string, bool> Features { get; set; }  // Dictionary 지원!

        [JsonIgnore]                             // 직렬화에서 제외
        public string InternalId { get; set; }

        [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
        public string OptionalField { get; set; }  // null이면 JSON에 포함 안 함
    }

    private void BasicExample()
    {
        var config = new GameConfig
        {
            ServerUrl = "https://api.game.com",
            MaxPlayers = 100,
            Features = new Dictionary<string, bool>
            {
                ["pvp"] = true,
                ["trading"] = false,
                ["guilds"] = true
            },
            InternalId = "secret-123"     // JsonIgnore로 직렬화 제외
        };

        // 직렬화
        string json = JsonConvert.SerializeObject(config, Formatting.Indented);
        Debug.Log(json);
        // 출력:
        // {
        //   "server_url": "https://api.game.com",
        //   "max_players": 100,
        //   "features": { "pvp": true, "trading": false, "guilds": true }
        // }

        // 역직렬화
        GameConfig restored = JsonConvert.DeserializeObject<GameConfig>(json);
        Debug.Log($"서버: {restored.ServerUrl}, 최대 플레이어: {restored.MaxPlayers}");
    }

    // =============================================
    // 다형성 (상속) 처리
    // =============================================

    [JsonConverter(typeof(StringEnumConverter))]  // enum을 문자열로
    public enum ItemType { Weapon, Armor, Consumable }

    public abstract class GameItem
    {
        public string Name { get; set; }
        public ItemType Type { get; set; }
    }

    public class WeaponItem : GameItem
    {
        public int Damage { get; set; }
        public float AttackSpeed { get; set; }
    }

    public class ArmorItem : GameItem
    {
        public int Defense { get; set; }
        public float Weight { get; set; }
    }

    private void PolymorphismExample()
    {
        List<GameItem> items = new List<GameItem>
        {
            new WeaponItem { Name = "불꽃 검", Type = ItemType.Weapon, Damage = 50, AttackSpeed = 1.2f },
            new ArmorItem { Name = "강철 갑옷", Type = ItemType.Armor, Defense = 30, Weight = 15.0f }
        };

        // TypeNameHandling으로 타입 정보 포함
        var settings = new JsonSerializerSettings
        {
            TypeNameHandling = TypeNameHandling.Auto,
            Formatting = Formatting.Indented
        };

        string json = JsonConvert.SerializeObject(items, settings);
        Debug.Log(json);

        // 역직렬화 시 올바른 하위 타입으로 복원
        List<GameItem> restored = JsonConvert.DeserializeObject<List<GameItem>>(json, settings);
        foreach (var item in restored)
        {
            if (item is WeaponItem weapon)
                Debug.Log($"무기: {weapon.Name}, 공격력: {weapon.Damage}");
            else if (item is ArmorItem armor)
                Debug.Log($"방어구: {armor.Name}, 방어력: {armor.Defense}");
        }
    }

    // =============================================
    // LINQ to JSON (동적 JSON 파싱)
    // =============================================

    private void LinqToJsonExample()
    {
        // 스키마를 모르는 JSON 동적 파싱
        string unknownJson = @"{
            ""player"": {
                ""name"": ""전사"",
                ""stats"": { ""hp"": 100, ""mp"": 50 },
                ""skills"": [""검격"", ""방어"", ""돌진""]
            },
            ""server_time"": ""2024-01-15T10:30:00Z""
        }";

        JObject root = JObject.Parse(unknownJson);

        // 경로로 접근
        string name = (string)root["player"]["name"];
        int hp = (int)root["player"]["stats"]["hp"];
        Debug.Log($"이름: {name}, HP: {hp}");

        // SelectToken (JSONPath 유사)
        JToken mpToken = root.SelectToken("player.stats.mp");
        Debug.Log($"MP: {mpToken}");

        // 배열 순회
        JArray skills = (JArray)root["player"]["skills"];
        foreach (JToken skill in skills)
        {
            Debug.Log($"스킬: {skill}");
        }

        // 동적 JSON 수정
        root["player"]["stats"]["hp"] = 200;
        root["player"]["stats"]["sp"] = 75;   // 새 필드 추가

        Debug.Log(root.ToString(Formatting.Indented));
    }

    // =============================================
    // 커스텀 JsonConverter
    // =============================================

    /// <summary>
    /// Unity Vector3를 간결한 배열 형태로 직렬화하는 컨버터
    /// </summary>
    public class Vector3Converter : JsonConverter<Vector3>
    {
        public override void WriteJson(JsonWriter writer, Vector3 value, JsonSerializer serializer)
        {
            writer.WriteStartArray();
            writer.WriteValue(value.x);
            writer.WriteValue(value.y);
            writer.WriteValue(value.z);
            writer.WriteEndArray();
        }

        public override Vector3 ReadJson(JsonReader reader, Type objectType,
            Vector3 existingValue, bool hasExistingValue, JsonSerializer serializer)
        {
            JArray array = JArray.Load(reader);
            return new Vector3(
                (float)array[0],
                (float)array[1],
                (float)array[2]
            );
        }
    }

    public class TransformData
    {
        [JsonConverter(typeof(Vector3Converter))]
        public Vector3 Position { get; set; }

        [JsonConverter(typeof(Vector3Converter))]
        public Vector3 Scale { get; set; }
    }

    private void CustomConverterExample()
    {
        var data = new TransformData
        {
            Position = new Vector3(1, 2, 3),
            Scale = new Vector3(1, 1, 1)
        };

        string json = JsonConvert.SerializeObject(data, Formatting.Indented);
        Debug.Log(json);
        // 출력:
        // {
        //   "Position": [1.0, 2.0, 3.0],
        //   "Scale": [1.0, 1.0, 1.0]
        // }
    }

    // =============================================
    // 전역 설정 (글로벌 기본값)
    // =============================================

    private void ConfigureGlobalSettings()
    {
        JsonConvert.DefaultSettings = () => new JsonSerializerSettings
        {
            // camelCase 네이밍
            ContractResolver = new CamelCasePropertyNamesContractResolver(),
            // null 무시
            NullValueHandling = NullValueHandling.Ignore,
            // 날짜 형식
            DateFormatString = "yyyy-MM-ddTHH:mm:ssZ",
            // enum을 문자열로
            Converters = new List<JsonConverter>
            {
                new StringEnumConverter(),
                new Vector3Converter()
            },
            // 참조 루프 무시
            ReferenceLoopHandling = ReferenceLoopHandling.Ignore,
            // 포맷팅
            Formatting = Formatting.None
        };
    }
}
```

---

## 4. System.Text.Json (.NET 표준)

.NET Core 3.0+에서 도입된 표준 라이브러리로, Unity 2021+ (.NET Standard 2.1)에서 사용할 수 있습니다. 고성능이 특징입니다.

```csharp
// ⚠️ Unity에서 System.Text.Json 사용 시:
// - Player Settings → Api Compatibility Level → .NET Standard 2.1 또는 .NET Framework
// - Unity 2021.2+ 권장
// - NuGet 패키지 설치 필요할 수 있음

#if NET_STANDARD_2_1 || NET_UNITY_4_8
using System;
using System.Text.Json;
using System.Text.Json.Serialization;
using System.Collections.Generic;
using UnityEngine;

public class SystemTextJsonExample : MonoBehaviour
{
    // =============================================
    // 기본 사용법
    // =============================================

    public class ServerResponse
    {
        [JsonPropertyName("status_code")]       // JSON 키 이름 지정
        public int StatusCode { get; set; }

        [JsonPropertyName("message")]
        public string Message { get; set; }

        [JsonPropertyName("data")]
        public Dictionary<string, object> Data { get; set; }

        [JsonIgnore]                             // 직렬화 제외
        public DateTime ReceivedAt { get; set; }
    }

    private void BasicExample()
    {
        var options = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,   // camelCase
            WriteIndented = true,                                 // 들여쓰기
            DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull  // null 무시
        };

        var response = new ServerResponse
        {
            StatusCode = 200,
            Message = "성공",
            Data = new Dictionary<string, object>
            {
                ["userId"] = 42,
                ["userName"] = "전사"
            }
        };

        // 직렬화
        string json = JsonSerializer.Serialize(response, options);
        Debug.Log(json);

        // 역직렬화
        ServerResponse restored = JsonSerializer.Deserialize<ServerResponse>(json, options);
        Debug.Log($"상태: {restored.StatusCode}, 메시지: {restored.Message}");
    }

    // =============================================
    // Source Generator (AOT/IL2CPP 호환)
    // =============================================

    // .NET 6+ / Unity 2022+에서 Source Generator 사용 가능
    // 리플렉션 없이 직렬화하여 IL2CPP/AOT 환경에 적합

    [JsonSerializable(typeof(PlayerInfo))]
    [JsonSerializable(typeof(List<PlayerInfo>))]
    public partial class PlayerInfoContext : JsonSerializerContext { }

    public class PlayerInfo
    {
        [JsonPropertyName("name")]
        public string Name { get; set; }

        [JsonPropertyName("score")]
        public int Score { get; set; }
    }

    private void SourceGeneratorExample()
    {
        var player = new PlayerInfo { Name = "전사", Score = 9999 };

        // Source Generator 사용 직렬화 (리플렉션 없음)
        string json = JsonSerializer.Serialize(player, PlayerInfoContext.Default.PlayerInfo);
        Debug.Log(json);

        // Source Generator 사용 역직렬화
        PlayerInfo restored = JsonSerializer.Deserialize<PlayerInfo>(
            json, PlayerInfoContext.Default.PlayerInfo);
        Debug.Log($"이름: {restored.Name}");
    }

    // =============================================
    // Utf8JsonReader/Utf8JsonWriter (저수준 고성능)
    // =============================================

    private void LowLevelReaderExample()
    {
        string json = "{\"name\":\"전사\",\"hp\":100,\"skills\":[\"검격\",\"방어\"]}";
        byte[] utf8Bytes = System.Text.Encoding.UTF8.GetBytes(json);

        var reader = new Utf8JsonReader(utf8Bytes);
        while (reader.Read())
        {
            switch (reader.TokenType)
            {
                case JsonTokenType.PropertyName:
                    Debug.Log($"속성: {reader.GetString()}");
                    break;
                case JsonTokenType.String:
                    Debug.Log($"문자열: {reader.GetString()}");
                    break;
                case JsonTokenType.Number:
                    Debug.Log($"숫자: {reader.GetInt32()}");
                    break;
            }
        }
    }
}
#endif
```

---

## 5. JSON 직렬화/역직렬화 성능 비교

```csharp
using UnityEngine;
using System;
using System.Diagnostics;
using Debug = UnityEngine.Debug;

/// <summary>
/// Unity 환경에서의 JSON 라이브러리 성능 비교
/// </summary>
public class JsonPerformanceBenchmark : MonoBehaviour
{
    // =============================================
    // 성능 비교 결과 (참고용 - 실제 환경에 따라 다름)
    // =============================================
    //
    // | 라이브러리           | 직렬화 (ms) | 역직렬화 (ms) | GC 할당  | IL2CPP |
    // |---------------------|------------|--------------|---------|--------|
    // | JsonUtility         | ~0.8       | ~1.0         | 최소     | ✅     |
    // | Newtonsoft.Json      | ~3.5       | ~5.0         | 중간     | ✅*    |
    // | System.Text.Json     | ~1.5       | ~2.0         | 적음     | ⚠️     |
    // | System.Text.Json+SG  | ~1.2       | ~1.5         | 최소     | ✅     |
    //
    // * Newtonsoft.Json은 Unity용 패키지(com.unity.nuget.newtonsoft-json) 사용 시 IL2CPP 호환
    // * SG = Source Generator
    // * 10,000회 반복 기준 평균, 단순 객체(필드 5개)

    [Serializable]
    public class BenchmarkData
    {
        public string name;
        public int score;
        public float health;
        public bool active;
        public string[] tags;
    }

    private void Start()
    {
        RunBenchmark();
    }

    private void RunBenchmark()
    {
        const int iterations = 10000;

        var data = new BenchmarkData
        {
            name = "TestPlayer",
            score = 99999,
            health = 95.5f,
            active = true,
            tags = new string[] { "warrior", "leader", "veteran" }
        };

        // =============================================
        // JsonUtility 벤치마크
        // =============================================

        var sw = Stopwatch.StartNew();
        string jsonUtilityResult = "";
        for (int i = 0; i < iterations; i++)
        {
            jsonUtilityResult = JsonUtility.ToJson(data);
        }
        sw.Stop();
        Debug.Log($"[JsonUtility] 직렬화 {iterations}회: {sw.ElapsedMilliseconds}ms");

        sw.Restart();
        for (int i = 0; i < iterations; i++)
        {
            JsonUtility.FromJson<BenchmarkData>(jsonUtilityResult);
        }
        sw.Stop();
        Debug.Log($"[JsonUtility] 역직렬화 {iterations}회: {sw.ElapsedMilliseconds}ms");

        // =============================================
        // FromJsonOverwrite (GC 최소화)
        // =============================================

        var reusable = new BenchmarkData();
        sw.Restart();
        for (int i = 0; i < iterations; i++)
        {
            JsonUtility.FromJsonOverwrite(jsonUtilityResult, reusable);
        }
        sw.Stop();
        Debug.Log($"[JsonUtility] Overwrite 역직렬화 {iterations}회: {sw.ElapsedMilliseconds}ms");
    }

    // =============================================
    // 라이브러리 선택 가이드
    // =============================================
    //
    // ✅ JsonUtility 선택:
    //   - 최고 성능이 필요할 때
    //   - 단순한 데이터 구조
    //   - GC 할당 최소화가 중요할 때 (모바일)
    //   - IL2CPP 빌드 안정성이 중요할 때
    //
    // ✅ Newtonsoft.Json 선택:
    //   - Dictionary, 다형성, LINQ to JSON 필요
    //   - 복잡한 JSON 구조 처리
    //   - 커스텀 직렬화 로직이 많을 때
    //   - 서드파티 API 연동 (유연한 파싱)
    //
    // ✅ System.Text.Json 선택:
    //   - .NET 표준 호환성이 중요할 때
    //   - 서버/클라이언트 코드 공유
    //   - Source Generator로 AOT 최적화 가능
    //   - 고성능 + 기능 균형
}
```

---

## 6. REST API 클라이언트 패턴

### 기본 REST 클라이언트

```csharp
using UnityEngine;
using UnityEngine.Networking;
using Newtonsoft.Json;
using System;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

#if UNITASK_AVAILABLE
using Cysharp.Threading.Tasks;
#endif

/// <summary>
/// Unity 환경에 최적화된 REST API 클라이언트
/// </summary>
public class RestApiClient
{
    private readonly string baseUrl;
    private readonly JsonSerializerSettings jsonSettings;
    private string authToken;
    private int timeoutSeconds = 30;

    public RestApiClient(string baseUrl)
    {
        this.baseUrl = baseUrl.TrimEnd('/');
        this.jsonSettings = new JsonSerializerSettings
        {
            NullValueHandling = NullValueHandling.Ignore,
            DateFormatString = "yyyy-MM-ddTHH:mm:ssZ"
        };
    }

    public void SetAuthToken(string token) => authToken = token;
    public void SetTimeout(int seconds) => timeoutSeconds = seconds;

    // =============================================
    // GET 요청
    // =============================================

    public async Task<ApiResponse<T>> GetAsync<T>(string endpoint,
        CancellationToken cancellationToken = default)
    {
        string url = $"{baseUrl}/{endpoint.TrimStart('/')}";

        using (var request = UnityWebRequest.Get(url))
        {
            ConfigureRequest(request);
            await SendRequestAsync(request, cancellationToken);
            return ParseResponse<T>(request);
        }
    }

    // =============================================
    // POST 요청
    // =============================================

    public async Task<ApiResponse<TResponse>> PostAsync<TRequest, TResponse>(
        string endpoint, TRequest body,
        CancellationToken cancellationToken = default)
    {
        string url = $"{baseUrl}/{endpoint.TrimStart('/')}";
        string json = JsonConvert.SerializeObject(body, jsonSettings);
        byte[] bodyRaw = Encoding.UTF8.GetBytes(json);

        using (var request = new UnityWebRequest(url, "POST"))
        {
            request.uploadHandler = new UploadHandlerRaw(bodyRaw);
            request.downloadHandler = new DownloadHandlerBuffer();
            ConfigureRequest(request);
            request.SetRequestHeader("Content-Type", "application/json");

            await SendRequestAsync(request, cancellationToken);
            return ParseResponse<TResponse>(request);
        }
    }

    // =============================================
    // PUT 요청
    // =============================================

    public async Task<ApiResponse<TResponse>> PutAsync<TRequest, TResponse>(
        string endpoint, TRequest body,
        CancellationToken cancellationToken = default)
    {
        string url = $"{baseUrl}/{endpoint.TrimStart('/')}";
        string json = JsonConvert.SerializeObject(body, jsonSettings);
        byte[] bodyRaw = Encoding.UTF8.GetBytes(json);

        using (var request = new UnityWebRequest(url, "PUT"))
        {
            request.uploadHandler = new UploadHandlerRaw(bodyRaw);
            request.downloadHandler = new DownloadHandlerBuffer();
            ConfigureRequest(request);
            request.SetRequestHeader("Content-Type", "application/json");

            await SendRequestAsync(request, cancellationToken);
            return ParseResponse<TResponse>(request);
        }
    }

    // =============================================
    // DELETE 요청
    // =============================================

    public async Task<ApiResponse<object>> DeleteAsync(string endpoint,
        CancellationToken cancellationToken = default)
    {
        string url = $"{baseUrl}/{endpoint.TrimStart('/')}";

        using (var request = UnityWebRequest.Delete(url))
        {
            request.downloadHandler = new DownloadHandlerBuffer();
            ConfigureRequest(request);

            await SendRequestAsync(request, cancellationToken);
            return ParseResponse<object>(request);
        }
    }

    // =============================================
    // 내부 헬퍼 메서드
    // =============================================

    private void ConfigureRequest(UnityWebRequest request)
    {
        request.timeout = timeoutSeconds;
        request.SetRequestHeader("Accept", "application/json");

        if (!string.IsNullOrEmpty(authToken))
        {
            request.SetRequestHeader("Authorization", $"Bearer {authToken}");
        }
    }

    private async Task SendRequestAsync(UnityWebRequest request,
        CancellationToken cancellationToken)
    {
        var operation = request.SendWebRequest();

        while (!operation.isDone)
        {
            cancellationToken.ThrowIfCancellationRequested();
            await Task.Yield();
        }
    }

    private ApiResponse<T> ParseResponse<T>(UnityWebRequest request)
    {
        var response = new ApiResponse<T>
        {
            StatusCode = request.responseCode,
            IsSuccess = request.result == UnityWebRequest.Result.Success,
            RawBody = request.downloadHandler?.text
        };

        if (response.IsSuccess && !string.IsNullOrEmpty(response.RawBody))
        {
            try
            {
                response.Data = JsonConvert.DeserializeObject<T>(
                    response.RawBody, jsonSettings);
            }
            catch (JsonException ex)
            {
                response.IsSuccess = false;
                response.Error = $"JSON 파싱 에러: {ex.Message}";
            }
        }
        else if (!response.IsSuccess)
        {
            response.Error = request.error;

            // 서버 에러 응답 본문 파싱 시도
            if (!string.IsNullOrEmpty(response.RawBody))
            {
                try
                {
                    response.ErrorDetail = JsonConvert.DeserializeObject<ApiError>(
                        response.RawBody, jsonSettings);
                }
                catch { /* 에러 본문 파싱 실패 시 무시 */ }
            }
        }

        return response;
    }
}

// =============================================
// API 응답 모델
// =============================================

public class ApiResponse<T>
{
    public long StatusCode { get; set; }
    public bool IsSuccess { get; set; }
    public T Data { get; set; }
    public string RawBody { get; set; }
    public string Error { get; set; }
    public ApiError ErrorDetail { get; set; }
}

public class ApiError
{
    [JsonProperty("error")]
    public string ErrorCode { get; set; }

    [JsonProperty("message")]
    public string Message { get; set; }

    [JsonProperty("details")]
    public object Details { get; set; }
}
```

### 사용 예시

```csharp
using UnityEngine;
using Newtonsoft.Json;
using System;
using System.Threading;

public class RestClientUsageExample : MonoBehaviour
{
    private RestApiClient apiClient;
    private CancellationTokenSource cts;

    // =============================================
    // DTO (Data Transfer Objects)
    // =============================================

    public class UserDto
    {
        [JsonProperty("id")]
        public int Id { get; set; }

        [JsonProperty("username")]
        public string Username { get; set; }

        [JsonProperty("email")]
        public string Email { get; set; }

        [JsonProperty("created_at")]
        public DateTime CreatedAt { get; set; }
    }

    public class CreateUserRequest
    {
        [JsonProperty("username")]
        public string Username { get; set; }

        [JsonProperty("email")]
        public string Email { get; set; }

        [JsonProperty("password")]
        public string Password { get; set; }
    }

    // =============================================
    // 초기화 및 사용
    // =============================================

    private void Awake()
    {
        apiClient = new RestApiClient("https://api.mygame.com/v1");
        apiClient.SetAuthToken("your-jwt-token-here");
        apiClient.SetTimeout(15);
        cts = new CancellationTokenSource();
    }

    private async void Start()
    {
        // GET - 사용자 조회
        var getUserResponse = await apiClient.GetAsync<UserDto>(
            "/users/42", cts.Token);

        if (getUserResponse.IsSuccess)
        {
            Debug.Log($"사용자: {getUserResponse.Data.Username}");
        }
        else
        {
            Debug.LogError($"에러 [{getUserResponse.StatusCode}]: {getUserResponse.Error}");
        }

        // POST - 사용자 생성
        var createRequest = new CreateUserRequest
        {
            Username = "newplayer",
            Email = "player@game.com",
            Password = "securepass123"
        };

        var createResponse = await apiClient.PostAsync<CreateUserRequest, UserDto>(
            "/users", createRequest, cts.Token);

        if (createResponse.IsSuccess)
        {
            Debug.Log($"생성된 사용자 ID: {createResponse.Data.Id}");
        }

        // DELETE - 사용자 삭제
        var deleteResponse = await apiClient.DeleteAsync("/users/42", cts.Token);
        Debug.Log($"삭제 결과: {deleteResponse.StatusCode}");
    }

    private void OnDestroy()
    {
        cts?.Cancel();
        cts?.Dispose();
    }
}
```

---

## 7. 페이지네이션, 인증 헤더, API 버전 관리

### 페이지네이션 (Pagination)

```csharp
using UnityEngine;
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// REST API 페이지네이션 처리
/// </summary>
public class PaginationExample : MonoBehaviour
{
    private RestApiClient apiClient;

    // =============================================
    // 오프셋 기반 페이지네이션 응답 모델
    // =============================================

    public class PaginatedResponse<T>
    {
        [JsonProperty("data")]
        public List<T> Data { get; set; }

        [JsonProperty("total")]
        public int Total { get; set; }

        [JsonProperty("page")]
        public int Page { get; set; }

        [JsonProperty("per_page")]
        public int PerPage { get; set; }

        [JsonProperty("total_pages")]
        public int TotalPages { get; set; }

        public bool HasNextPage => Page < TotalPages;
        public bool HasPrevPage => Page > 1;
    }

    // =============================================
    // 커서 기반 페이지네이션 응답 모델
    // =============================================

    public class CursorPaginatedResponse<T>
    {
        [JsonProperty("data")]
        public List<T> Data { get; set; }

        [JsonProperty("next_cursor")]
        public string NextCursor { get; set; }

        [JsonProperty("prev_cursor")]
        public string PrevCursor { get; set; }

        [JsonProperty("has_more")]
        public bool HasMore { get; set; }
    }

    public class LeaderboardEntry
    {
        [JsonProperty("rank")]
        public int Rank { get; set; }

        [JsonProperty("player_name")]
        public string PlayerName { get; set; }

        [JsonProperty("score")]
        public long Score { get; set; }
    }

    // =============================================
    // 오프셋 기반 페이지네이션 사용
    // =============================================

    private async Task<PaginatedResponse<LeaderboardEntry>> GetLeaderboardPage(
        int page, int perPage = 20, CancellationToken token = default)
    {
        string endpoint = $"/leaderboard?page={page}&per_page={perPage}";
        var response = await apiClient.GetAsync<PaginatedResponse<LeaderboardEntry>>(
            endpoint, token);

        if (response.IsSuccess)
            return response.Data;

        Debug.LogError($"리더보드 조회 실패: {response.Error}");
        return null;
    }

    /// <summary>
    /// 전체 리더보드 데이터를 페이지별로 가져오기
    /// </summary>
    private async Task<List<LeaderboardEntry>> GetAllLeaderboard(
        CancellationToken token = default)
    {
        var allEntries = new List<LeaderboardEntry>();
        int currentPage = 1;

        while (true)
        {
            token.ThrowIfCancellationRequested();

            var page = await GetLeaderboardPage(currentPage, 50, token);
            if (page == null || page.Data.Count == 0) break;

            allEntries.AddRange(page.Data);
            Debug.Log($"페이지 {currentPage}/{page.TotalPages} 로드 완료 " +
                      $"({allEntries.Count}/{page.Total})");

            if (!page.HasNextPage) break;
            currentPage++;
        }

        return allEntries;
    }

    // =============================================
    // 커서 기반 페이지네이션 사용
    // =============================================

    /// <summary>
    /// 커서 기반으로 다음 페이지 가져오기
    /// - 대규모 데이터셋에서 오프셋보다 효율적
    /// - 실시간 데이터 변경에 강건
    /// </summary>
    private async Task<CursorPaginatedResponse<LeaderboardEntry>> GetLeaderboardCursor(
        string cursor = null, int limit = 20, CancellationToken token = default)
    {
        string endpoint = $"/leaderboard?limit={limit}";
        if (!string.IsNullOrEmpty(cursor))
        {
            endpoint += $"&cursor={cursor}";
        }

        var response = await apiClient.GetAsync<CursorPaginatedResponse<LeaderboardEntry>>(
            endpoint, token);

        return response.IsSuccess ? response.Data : null;
    }

    private async Task LoadAllWithCursor(CancellationToken token)
    {
        string cursor = null;
        int totalLoaded = 0;

        do
        {
            var page = await GetLeaderboardCursor(cursor, 50, token);
            if (page == null) break;

            totalLoaded += page.Data.Count;
            cursor = page.NextCursor;

            Debug.Log($"로드: {totalLoaded}개 (다음 커서: {cursor})");

        } while (!string.IsNullOrEmpty(cursor));
    }
}
```

### 인증 헤더 관리

```csharp
using UnityEngine;
using Newtonsoft.Json;
using System;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// JWT 기반 인증 토큰 관리자
/// </summary>
public class AuthTokenManager
{
    private string accessToken;
    private string refreshToken;
    private DateTime tokenExpiry;
    private readonly RestApiClient apiClient;
    private readonly SemaphoreSlim refreshLock = new SemaphoreSlim(1, 1);

    public AuthTokenManager(RestApiClient apiClient)
    {
        this.apiClient = apiClient;
    }

    // =============================================
    // 로그인
    // =============================================

    public class LoginRequest
    {
        [JsonProperty("email")]
        public string Email { get; set; }

        [JsonProperty("password")]
        public string Password { get; set; }
    }

    public class AuthTokenResponse
    {
        [JsonProperty("access_token")]
        public string AccessToken { get; set; }

        [JsonProperty("refresh_token")]
        public string RefreshToken { get; set; }

        [JsonProperty("expires_in")]
        public int ExpiresInSeconds { get; set; }
    }

    public async Task<bool> LoginAsync(string email, string password,
        CancellationToken token = default)
    {
        var request = new LoginRequest { Email = email, Password = password };
        var response = await apiClient.PostAsync<LoginRequest, AuthTokenResponse>(
            "/auth/login", request, token);

        if (response.IsSuccess)
        {
            SetTokens(response.Data);
            return true;
        }

        Debug.LogError($"로그인 실패: {response.Error}");
        return false;
    }

    // =============================================
    // 토큰 갱신 (자동)
    // =============================================

    /// <summary>
    /// 유효한 액세스 토큰을 반환. 만료 시 자동으로 갱신합니다.
    /// </summary>
    public async Task<string> GetValidTokenAsync(CancellationToken token = default)
    {
        // 토큰이 아직 유효한 경우
        if (!string.IsNullOrEmpty(accessToken) && DateTime.UtcNow < tokenExpiry.AddMinutes(-1))
        {
            return accessToken;
        }

        // 동시 갱신 방지 (하나의 요청만 갱신 수행)
        await refreshLock.WaitAsync(token);
        try
        {
            // Double-check: 다른 스레드가 이미 갱신했을 수 있음
            if (!string.IsNullOrEmpty(accessToken) && DateTime.UtcNow < tokenExpiry.AddMinutes(-1))
            {
                return accessToken;
            }

            return await RefreshTokenAsync(token);
        }
        finally
        {
            refreshLock.Release();
        }
    }

    private async Task<string> RefreshTokenAsync(CancellationToken token)
    {
        if (string.IsNullOrEmpty(refreshToken))
        {
            throw new InvalidOperationException("리프레시 토큰이 없습니다. 다시 로그인하세요.");
        }

        var request = new { refresh_token = refreshToken };
        var response = await apiClient.PostAsync<object, AuthTokenResponse>(
            "/auth/refresh", request, token);

        if (response.IsSuccess)
        {
            SetTokens(response.Data);
            Debug.Log("토큰 갱신 성공");
            return accessToken;
        }

        // 리프레시 토큰도 만료된 경우
        accessToken = null;
        refreshToken = null;
        throw new UnauthorizedAccessException("토큰 갱신 실패. 다시 로그인이 필요합니다.");
    }

    private void SetTokens(AuthTokenResponse authResponse)
    {
        accessToken = authResponse.AccessToken;
        refreshToken = authResponse.RefreshToken;
        tokenExpiry = DateTime.UtcNow.AddSeconds(authResponse.ExpiresInSeconds);

        apiClient.SetAuthToken(accessToken);
    }
}
```

### API 버전 관리

```csharp
using UnityEngine;
using UnityEngine.Networking;

/// <summary>
/// REST API 버전 관리 전략
/// </summary>
public class ApiVersioningExample : MonoBehaviour
{
    // =============================================
    // 전략 1: URL 경로 기반 (가장 일반적)
    // =============================================
    // ✅ 명확하고 직관적
    // ✅ 캐싱이 쉬움
    // ❌ URL이 길어질 수 있음

    private string GetUrlPathVersioned(string version, string endpoint)
    {
        // 예: https://api.game.com/v1/users
        //     https://api.game.com/v2/users
        return $"https://api.game.com/{version}/{endpoint}";
    }

    // =============================================
    // 전략 2: 헤더 기반
    // =============================================
    // ✅ URL이 깔끔
    // ❌ 테스트/디버깅이 어려움

    private void SetHeaderVersion(UnityWebRequest request, string version)
    {
        // 커스텀 헤더
        request.SetRequestHeader("X-API-Version", version);

        // 또는 Accept 헤더에 포함
        request.SetRequestHeader("Accept", $"application/vnd.mygame.v{version}+json");
    }

    // =============================================
    // 전략 3: 쿼리 파라미터 기반
    // =============================================
    // ✅ 구현이 간단
    // ❌ 캐싱 키가 복잡해짐

    private string GetQueryVersioned(string endpoint, string version)
    {
        // 예: https://api.game.com/users?api_version=2
        return $"https://api.game.com/{endpoint}?api_version={version}";
    }

    // =============================================
    // 통합 API 버전 관리 클래스
    // =============================================

    public enum ApiVersionStrategy { UrlPath, Header, QueryParameter }

    public class VersionedApiClient
    {
        private readonly string baseUrl;
        private readonly string version;
        private readonly ApiVersionStrategy strategy;

        public VersionedApiClient(string baseUrl, string version,
            ApiVersionStrategy strategy = ApiVersionStrategy.UrlPath)
        {
            this.baseUrl = baseUrl.TrimEnd('/');
            this.version = version;
            this.strategy = strategy;
        }

        public string BuildUrl(string endpoint)
        {
            endpoint = endpoint.TrimStart('/');

            return strategy switch
            {
                ApiVersionStrategy.UrlPath =>
                    $"{baseUrl}/v{version}/{endpoint}",
                ApiVersionStrategy.QueryParameter =>
                    $"{baseUrl}/{endpoint}?api_version={version}",
                _ =>
                    $"{baseUrl}/{endpoint}"
            };
        }

        public void ApplyVersionHeaders(UnityWebRequest request)
        {
            if (strategy == ApiVersionStrategy.Header)
            {
                request.SetRequestHeader("X-API-Version", version);
            }
        }
    }
}
```

---

## 8. DTO 패턴과 JSON Schema 활용

### DTO (Data Transfer Object) 패턴

```csharp
using UnityEngine;
using Newtonsoft.Json;
using System;
using System.Collections.Generic;

/// <summary>
/// DTO 패턴: API 통신과 게임 로직의 데이터 모델을 분리
/// </summary>
public class DtoPatternExample : MonoBehaviour
{
    // =============================================
    // API 응답 DTO (서버와의 계약)
    // =============================================

    /// <summary>
    /// 서버에서 받는 플레이어 데이터 형식.
    /// API 스펙에 맞춰 필드명/타입을 정의합니다.
    /// </summary>
    public class PlayerResponseDto
    {
        [JsonProperty("player_id")]
        public string PlayerId { get; set; }

        [JsonProperty("display_name")]
        public string DisplayName { get; set; }

        [JsonProperty("experience_points")]
        public long ExperiencePoints { get; set; }

        [JsonProperty("current_level")]
        public int CurrentLevel { get; set; }

        [JsonProperty("position_x")]
        public float PositionX { get; set; }

        [JsonProperty("position_y")]
        public float PositionY { get; set; }

        [JsonProperty("position_z")]
        public float PositionZ { get; set; }

        [JsonProperty("inventory_items")]
        public List<ItemResponseDto> InventoryItems { get; set; }

        [JsonProperty("last_login")]
        public string LastLogin { get; set; }
    }

    public class ItemResponseDto
    {
        [JsonProperty("item_id")]
        public string ItemId { get; set; }

        [JsonProperty("item_name")]
        public string ItemName { get; set; }

        [JsonProperty("quantity")]
        public int Quantity { get; set; }

        [JsonProperty("rarity")]
        public string Rarity { get; set; }
    }

    // =============================================
    // API 요청 DTO
    // =============================================

    public class UpdatePlayerRequestDto
    {
        [JsonProperty("display_name")]
        public string DisplayName { get; set; }

        [JsonProperty("position_x")]
        public float PositionX { get; set; }

        [JsonProperty("position_y")]
        public float PositionY { get; set; }

        [JsonProperty("position_z")]
        public float PositionZ { get; set; }
    }

    // =============================================
    // 게임 내부 도메인 모델 (게임 로직용)
    // =============================================

    public class PlayerModel
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public long Exp { get; set; }
        public int Level { get; set; }
        public Vector3 Position { get; set; }
        public List<ItemModel> Inventory { get; set; }
        public DateTime LastLogin { get; set; }
    }

    public class ItemModel
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public int Count { get; set; }
        public ItemRarity Rarity { get; set; }
    }

    public enum ItemRarity { Common, Uncommon, Rare, Epic, Legendary }

    // =============================================
    // DTO ↔ 도메인 모델 매퍼
    // =============================================

    /// <summary>
    /// DTO와 도메인 모델 간 변환을 담당합니다.
    /// API 스펙 변경 시 매퍼만 수정하면 됩니다.
    /// </summary>
    public static class PlayerMapper
    {
        // ✅ DTO → 도메인 모델 변환
        public static PlayerModel ToDomain(PlayerResponseDto dto)
        {
            return new PlayerModel
            {
                Id = dto.PlayerId,
                Name = dto.DisplayName,
                Exp = dto.ExperiencePoints,
                Level = dto.CurrentLevel,
                Position = new Vector3(dto.PositionX, dto.PositionY, dto.PositionZ),
                Inventory = dto.InventoryItems?.ConvertAll(ItemMapper.ToDomain)
                            ?? new List<ItemModel>(),
                LastLogin = DateTime.TryParse(dto.LastLogin, out var date)
                            ? date : DateTime.MinValue
            };
        }

        // ✅ 도메인 모델 → 요청 DTO 변환
        public static UpdatePlayerRequestDto ToUpdateDto(PlayerModel model)
        {
            return new UpdatePlayerRequestDto
            {
                DisplayName = model.Name,
                PositionX = model.Position.x,
                PositionY = model.Position.y,
                PositionZ = model.Position.z
            };
        }
    }

    public static class ItemMapper
    {
        public static ItemModel ToDomain(ItemResponseDto dto)
        {
            return new ItemModel
            {
                Id = dto.ItemId,
                Name = dto.ItemName,
                Count = dto.Quantity,
                Rarity = ParseRarity(dto.Rarity)
            };
        }

        private static ItemRarity ParseRarity(string rarity)
        {
            return rarity?.ToLower() switch
            {
                "common" => ItemRarity.Common,
                "uncommon" => ItemRarity.Uncommon,
                "rare" => ItemRarity.Rare,
                "epic" => ItemRarity.Epic,
                "legendary" => ItemRarity.Legendary,
                _ => ItemRarity.Common
            };
        }
    }

    // =============================================
    // 사용 예시
    // =============================================

    private async void Start()
    {
        var apiClient = new RestApiClient("https://api.mygame.com/v1");

        // 서버에서 DTO로 수신
        var response = await apiClient.GetAsync<PlayerResponseDto>("/players/me");

        if (response.IsSuccess)
        {
            // DTO → 도메인 모델 변환
            PlayerModel player = PlayerMapper.ToDomain(response.Data);

            // 게임 로직에서는 도메인 모델 사용
            Debug.Log($"플레이어: {player.Name}, 위치: {player.Position}");
            Debug.Log($"인벤토리: {player.Inventory.Count}개 아이템");

            // 업데이트 시: 도메인 모델 → 요청 DTO 변환
            player.Position = new Vector3(10, 0, 20);
            var updateDto = PlayerMapper.ToUpdateDto(player);
            await apiClient.PutAsync<UpdatePlayerRequestDto, object>(
                "/players/me", updateDto);
        }
    }
}
```

### JSON Schema 유효성 검증

```csharp
using UnityEngine;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;
using Newtonsoft.Json.Schema;  // NuGet: Newtonsoft.Json.Schema
using System.Collections.Generic;

/// <summary>
/// JSON Schema를 활용한 API 응답 유효성 검증
///
/// ⚠️ Newtonsoft.Json.Schema는 별도 패키지입니다.
/// 간단한 검증은 직접 구현할 수 있습니다.
/// </summary>
public class JsonValidationExample : MonoBehaviour
{
    // =============================================
    // 수동 JSON 유효성 검증 (라이브러리 없이)
    // =============================================

    /// <summary>
    /// 필수 필드 존재 여부와 타입을 검증합니다.
    /// 외부 패키지 없이도 기본적인 검증이 가능합니다.
    /// </summary>
    public static class JsonValidator
    {
        public class ValidationResult
        {
            public bool IsValid { get; set; } = true;
            public List<string> Errors { get; set; } = new List<string>();

            public void AddError(string error)
            {
                IsValid = false;
                Errors.Add(error);
            }
        }

        /// <summary>JSON 응답에서 필수 필드 검증</summary>
        public static ValidationResult ValidatePlayerResponse(string json)
        {
            var result = new ValidationResult();

            JObject obj;
            try
            {
                obj = JObject.Parse(json);
            }
            catch (JsonReaderException ex)
            {
                result.AddError($"잘못된 JSON 형식: {ex.Message}");
                return result;
            }

            // 필수 필드 존재 확인
            RequireField(obj, "player_id", JTokenType.String, result);
            RequireField(obj, "display_name", JTokenType.String, result);
            RequireField(obj, "current_level", JTokenType.Integer, result);

            // 범위 검증
            if (obj.TryGetValue("current_level", out JToken levelToken))
            {
                int level = levelToken.Value<int>();
                if (level < 1 || level > 999)
                {
                    result.AddError($"current_level은 1~999 범위여야 합니다. 실제: {level}");
                }
            }

            // 배열 필드 검증
            if (obj.TryGetValue("inventory_items", out JToken itemsToken))
            {
                if (itemsToken.Type != JTokenType.Array)
                {
                    result.AddError("inventory_items는 배열이어야 합니다.");
                }
                else
                {
                    var items = (JArray)itemsToken;
                    for (int i = 0; i < items.Count; i++)
                    {
                        if (items[i].Type != JTokenType.Object)
                        {
                            result.AddError($"inventory_items[{i}]는 객체여야 합니다.");
                        }
                    }
                }
            }

            return result;
        }

        private static void RequireField(JObject obj, string fieldName,
            JTokenType expectedType, ValidationResult result)
        {
            if (!obj.TryGetValue(fieldName, out JToken token))
            {
                result.AddError($"필수 필드 누락: {fieldName}");
                return;
            }

            if (token.Type != expectedType && token.Type != JTokenType.Null)
            {
                result.AddError(
                    $"필드 '{fieldName}' 타입 불일치. " +
                    $"기대: {expectedType}, 실제: {token.Type}");
            }
        }
    }

    // =============================================
    // 사용 예시
    // =============================================

    private void ValidateResponse()
    {
        string serverJson = @"{
            ""player_id"": ""p-12345"",
            ""display_name"": ""전사"",
            ""current_level"": 42,
            ""inventory_items"": [
                {""item_id"": ""i-001"", ""item_name"": ""검"", ""quantity"": 1}
            ]
        }";

        var result = JsonValidator.ValidatePlayerResponse(serverJson);

        if (result.IsValid)
        {
            Debug.Log("JSON 유효성 검증 통과");
        }
        else
        {
            foreach (var error in result.Errors)
            {
                Debug.LogError($"검증 실패: {error}");
            }
        }
    }

    // =============================================
    // 방어적 역직렬화 패턴
    // =============================================

    /// <summary>
    /// 안전한 역직렬화: 검증 후 변환
    /// </summary>
    public static T SafeDeserialize<T>(string json, out List<string> errors) where T : class
    {
        errors = new List<string>();

        if (string.IsNullOrEmpty(json))
        {
            errors.Add("빈 JSON 응답");
            return null;
        }

        try
        {
            var result = JsonConvert.DeserializeObject<T>(json, new JsonSerializerSettings
            {
                // 알 수 없는 필드 발견 시 에러 핸들링
                MissingMemberHandling = MissingMemberHandling.Ignore,
                // null이면 기본값 사용
                NullValueHandling = NullValueHandling.Include,
                // 에러 발생 시 계속 진행
                Error = (sender, args) =>
                {
                    errors.Add($"역직렬화 경고: {args.ErrorContext.Error.Message}");
                    args.ErrorContext.Handled = true;
                }
            });

            return result;
        }
        catch (JsonException ex)
        {
            errors.Add($"JSON 파싱 실패: {ex.Message}");
            return null;
        }
    }
}
```

---

## 주의사항

1. **JsonUtility 제한 사항 인지**: `Dictionary`, 프로퍼티, 다형성, null을 지원하지 않습니다. 단순 데이터 구조에만 사용하세요.

2. **IL2CPP 빌드 주의**: `System.Text.Json`의 리플렉션 기반 직렬화는 IL2CPP 환경에서 문제가 발생할 수 있습니다. Source Generator를 사용하거나 `link.xml`에 타입을 보존하세요.

3. **TypeNameHandling 보안 위험**: `Newtonsoft.Json`의 `TypeNameHandling.All`은 역직렬화 공격에 취약합니다. 신뢰할 수 없는 JSON에는 절대 사용하지 마세요.

```csharp
// ❌ 위험: 외부 JSON에 TypeNameHandling 사용
var settings = new JsonSerializerSettings
{
    TypeNameHandling = TypeNameHandling.All  // 보안 위험!
};
var data = JsonConvert.DeserializeObject<object>(untrustedJson, settings);

// ✅ 안전: 내부 데이터에만 제한적 사용
var safeSettings = new JsonSerializerSettings
{
    TypeNameHandling = TypeNameHandling.Auto,
    SerializationBinder = new KnownTypesOnlyBinder()  // 허용 타입 제한
};
```

4. **GC 압박 주의**: 매 프레임 JSON 파싱은 GC 부하를 유발합니다. 결과를 캐싱하고, `JsonUtility.FromJsonOverwrite()`로 할당을 줄이세요.

5. **API 버전 호환성**: 서버 API가 변경되면 DTO만 수정하고 매퍼에서 변환 로직을 조정하세요. 도메인 모델은 안정적으로 유지합니다.

6. **토큰 저장**: 인증 토큰을 `PlayerPrefs`에 평문으로 저장하지 마세요. 플랫폼별 보안 저장소(Keychain, Keystore 등)를 사용하세요.

7. **Rate Limiting 대응**: 429 응답을 받으면 `Retry-After` 헤더를 확인하고 지수 백오프를 적용하세요.

---

## 베스트 프랙티스

### DTO와 도메인 모델 분리

```csharp
// ✅ 올바른 패턴: DTO와 도메인 모델 분리
public class PlayerDto { /* API 스펙에 맞춘 필드 */ }
public class PlayerModel { /* 게임 로직에 맞춘 필드 */ }
public static class PlayerMapper
{
    public static PlayerModel ToDomain(PlayerDto dto) { /* 변환 */ }
}

// ❌ 잘못된 패턴: API DTO를 게임 로직에서 직접 사용
// - API 변경 시 게임 로직 전체에 영향
// - 불필요한 필드가 게임 로직에 노출
```

### JSON 라이브러리 사용 전략

```csharp
// ✅ 올바른 선택:
// 단순 세이브 데이터 → JsonUtility (최고 성능)
// 복잡한 API 통신   → Newtonsoft.Json (최고 기능)
// 서버 공유 코드     → System.Text.Json (표준 호환)

// ❌ 잘못된 선택:
// Dictionary 포함 데이터에 JsonUtility 사용
// 매 프레임 파싱에 Newtonsoft.Json 사용 (GC 부하)
```

### API 호출 구조화

```csharp
// ✅ 서비스 계층으로 API 호출 캡슐화
public class PlayerService
{
    private readonly RestApiClient client;
    private readonly AuthTokenManager auth;

    public async Task<PlayerModel> GetCurrentPlayerAsync(CancellationToken token)
    {
        string validToken = await auth.GetValidTokenAsync(token);
        client.SetAuthToken(validToken);

        var response = await client.GetAsync<PlayerResponseDto>("/players/me", token);

        if (!response.IsSuccess)
            throw new ApiException(response.StatusCode, response.Error);

        return PlayerMapper.ToDomain(response.Data);
    }
}

// ❌ MonoBehaviour에서 직접 HTTP 호출 + JSON 파싱 + 게임 로직 혼합
```

### 에러 응답 표준화

```csharp
// ✅ 일관된 에러 응답 처리
public class ApiException : System.Exception
{
    public long StatusCode { get; }
    public ApiError ErrorBody { get; }

    public ApiException(long statusCode, string message, ApiError errorBody = null)
        : base(message)
    {
        StatusCode = statusCode;
        ErrorBody = errorBody;
    }

    public bool IsUnauthorized => StatusCode == 401;
    public bool IsNotFound => StatusCode == 404;
    public bool IsRateLimited => StatusCode == 429;
    public bool IsServerError => StatusCode >= 500;
}
```

### 캐싱 전략

```csharp
using System.Collections.Generic;

// ✅ API 응답 캐싱으로 불필요한 요청 감소
public class ApiResponseCache<T>
{
    private readonly Dictionary<string, CacheEntry<T>> cache = new();
    private readonly float ttlSeconds;

    public ApiResponseCache(float ttlSeconds = 60f)
    {
        this.ttlSeconds = ttlSeconds;
    }

    public bool TryGet(string key, out T value)
    {
        if (cache.TryGetValue(key, out var entry) &&
            (UnityEngine.Time.realtimeSinceStartup - entry.Timestamp) < ttlSeconds)
        {
            value = entry.Data;
            return true;
        }

        value = default;
        return false;
    }

    public void Set(string key, T value)
    {
        cache[key] = new CacheEntry<T>
        {
            Data = value,
            Timestamp = UnityEngine.Time.realtimeSinceStartup
        };
    }

    public void Invalidate(string key) => cache.Remove(key);
    public void Clear() => cache.Clear();

    private class CacheEntry<TData>
    {
        public TData Data;
        public float Timestamp;
    }
}
```

---

## 참고 자료

- [Unity: JsonUtility](https://docs.unity3d.com/ScriptReference/JsonUtility.html)
- [Newtonsoft.Json (Json.NET) 공식 문서](https://www.newtonsoft.com/json/help/html/Introduction.htm)
- [System.Text.Json - Microsoft Docs](https://learn.microsoft.com/ko-kr/dotnet/standard/serialization/system-text-json/overview)
- [REST API 설계 가이드 - Microsoft](https://learn.microsoft.com/ko-kr/azure/architecture/best-practices/api-design)
- [Unity Package: com.unity.nuget.newtonsoft-json](https://docs.unity3d.com/Packages/com.unity.nuget.newtonsoft-json@3.2/manual/index.html)
- [HTTP 상태 코드 - MDN](https://developer.mozilla.org/ko/docs/Web/HTTP/Status)

---

## 다음 섹션

[28. WebSocket & SignalR](./28-websocket-signalr.md)
