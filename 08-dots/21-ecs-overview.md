# 21. ECS 개요 (Entity Component System)

## 개요

**ECS**(Entity Component System)는 Unity DOTS(Data-Oriented Technology Stack)의 핵심 아키텍처입니다. 기존 GameObject/MonoBehaviour 대신 **데이터 지향 설계**를 통해 대규모 오브젝트를 효율적으로 처리합니다.

---

## 설치

### Unity Package Manager

```
// Unity 2022.2+ 권장
com.unity.entities
com.unity.entities.graphics
```

### 패키지 종속성

```
Entities 패키지 설치 시 자동 포함:
- com.unity.collections (NativeContainer)
- com.unity.burst (Burst Compiler)
- com.unity.jobs (Job System)
- com.unity.mathematics (고성능 수학 라이브러리)
```

---

## 기본 개념

### GameObject vs ECS

| 항목 | GameObject | ECS |
|------|------------|-----|
| **데이터** | MonoBehaviour 클래스 | IComponentData 구조체 |
| **로직** | Update() 메서드 | ISystem |
| **식별** | GameObject 인스턴스 | Entity (int ID) |
| **메모리** | 분산 (힙) | 연속 (청크) |
| **성능** | 느림 | 매우 빠름 |
| **SIMD** | 어려움 | Burst로 자동 |

### ECS 삼요소

```
┌─────────────────────────────────────────────────────────┐
│                         World                            │
│  ┌──────────────────────────────────────────────────┐   │
│  │                     System                        │   │
│  │  - 로직 담당                                      │   │
│  │  - Entity 조회 및 처리                            │   │
│  └──────────────────────────────────────────────────┘   │
│                           ↓                              │
│  ┌──────────────────────────────────────────────────┐   │
│  │                     Entity                        │   │
│  │  - 고유 ID (int)                                 │   │
│  │  - Component들의 컨테이너                         │   │
│  └──────────────────────────────────────────────────┘   │
│                           ↓                              │
│  ┌──────────────────────────────────────────────────┐   │
│  │                    Component                      │   │
│  │  - 순수 데이터 (IComponentData)                   │   │
│  │  - 로직 없음                                      │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Component 정의

### IComponentData

```csharp
using Unity.Entities;
using Unity.Mathematics;

// 기본 컴포넌트 - 데이터만 포함
public struct Position : IComponentData
{
    public float3 Value;
}

public struct Velocity : IComponentData
{
    public float3 Value;
}

public struct Health : IComponentData
{
    public int Current;
    public int Max;
}

// 태그 컴포넌트 - 데이터 없이 식별용
public struct PlayerTag : IComponentData { }
public struct EnemyTag : IComponentData { }

// 활성화 컴포넌트
public struct Disabled : IComponentData, IEnableableComponent { }
```

### ISharedComponentData

```csharp
// 여러 Entity가 공유하는 데이터
public struct RenderMesh : ISharedComponentData
{
    public Mesh Mesh;
    public Material Material;
}

// 같은 SharedComponent 값을 가진 Entity들은 같은 청크에 저장
```

### IBufferElementData

```csharp
// 가변 길이 배열을 위한 동적 버퍼
[InternalBufferCapacity(8)] // 청크 내 초기 용량
public struct DamageBufferElement : IBufferElementData
{
    public int Value;
    public float Timestamp;
}

// 사용 예
// var buffer = EntityManager.GetBuffer<DamageBufferElement>(entity);
// buffer.Add(new DamageBufferElement { Value = 10, Timestamp = 0 });
```

---

## Entity 생성

### EntityManager 사용

```csharp
using Unity.Entities;

public partial class EntitySpawnSystem : SystemBase
{
    protected override void OnCreate()
    {
        // EntityManager로 Entity 생성
        var entity = EntityManager.CreateEntity(
            typeof(Position),
            typeof(Velocity),
            typeof(Health));

        // 컴포넌트 값 설정
        EntityManager.SetComponentData(entity, new Position { Value = float3.zero });
        EntityManager.SetComponentData(entity, new Velocity { Value = new float3(1, 0, 0) });
        EntityManager.SetComponentData(entity, new Health { Current = 100, Max = 100 });
    }

    protected override void OnUpdate() { }
}
```

### Archetype 사용

```csharp
public partial class ArchetypeSpawnSystem : SystemBase
{
    private EntityArchetype _enemyArchetype;

    protected override void OnCreate()
    {
        // Archetype 정의 (컴포넌트 조합)
        _enemyArchetype = EntityManager.CreateArchetype(
            typeof(Position),
            typeof(Velocity),
            typeof(Health),
            typeof(EnemyTag));
    }

    protected override void OnUpdate()
    {
        // Archetype으로 빠른 Entity 생성
        var enemy = EntityManager.CreateEntity(_enemyArchetype);

        EntityManager.SetComponentData(enemy, new Position { Value = new float3(0, 0, 0) });
        EntityManager.SetComponentData(enemy, new Health { Current = 50, Max = 50 });
    }
}
```

### EntityCommandBuffer 사용

```csharp
public partial class BufferedSpawnSystem : SystemBase
{
    private EndSimulationEntityCommandBufferSystem _ecbSystem;

    protected override void OnCreate()
    {
        _ecbSystem = World.GetOrCreateSystemManaged<EndSimulationEntityCommandBufferSystem>();
    }

    protected override void OnUpdate()
    {
        var ecb = _ecbSystem.CreateCommandBuffer();

        // Job 내에서 Entity 생성 예약
        Entities
            .WithAll<SpawnerTag>()
            .ForEach((Entity entity, in SpawnRequest request) =>
            {
                // 실제 생성은 프레임 끝에 실행
                var newEntity = ecb.CreateEntity();
                ecb.AddComponent(newEntity, new Position { Value = request.Position });
                ecb.AddComponent(newEntity, new Health { Current = 100, Max = 100 });
            })
            .Schedule();

        _ecbSystem.AddJobHandleForProducer(Dependency);
    }
}

public struct SpawnerTag : IComponentData { }
public struct SpawnRequest : IComponentData
{
    public float3 Position;
}
```

---

## System 정의

### SystemBase (Managed)

```csharp
using Unity.Entities;
using Unity.Transforms;

public partial class MovementSystem : SystemBase
{
    protected override void OnUpdate()
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        // Entities.ForEach - 람다 기반
        Entities
            .ForEach((ref LocalTransform transform, in Velocity velocity) =>
            {
                transform.Position += velocity.Value * deltaTime;
            })
            .ScheduleParallel(); // 병렬 실행
    }
}
```

### ISystem (Unmanaged, 권장)

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Transforms;

[BurstCompile]
public partial struct MovementSystemISystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // 초기화
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        // SystemAPI.Query 사용
        foreach (var (transform, velocity) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * deltaTime;
        }
    }

    [BurstCompile]
    public void OnDestroy(ref SystemState state)
    {
        // 정리
    }
}
```

### IJobEntity (Job 기반)

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Transforms;

[BurstCompile]
public partial struct MovementJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref LocalTransform transform, in Velocity velocity)
    {
        transform.Position += velocity.Value * DeltaTime;
    }
}

[BurstCompile]
public partial struct MovementJobSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var job = new MovementJob
        {
            DeltaTime = SystemAPI.Time.DeltaTime
        };
        job.ScheduleParallel();
    }
}
```

---

## Entity 쿼리

### EntityQuery

```csharp
public partial class QueryExampleSystem : SystemBase
{
    private EntityQuery _enemyQuery;

    protected override void OnCreate()
    {
        // 쿼리 정의
        _enemyQuery = GetEntityQuery(
            ComponentType.ReadOnly<EnemyTag>(),
            ComponentType.ReadWrite<Health>(),
            ComponentType.Exclude<Disabled>()
        );
    }

    protected override void OnUpdate()
    {
        // 쿼리 결과 사용
        int enemyCount = _enemyQuery.CalculateEntityCount();
        Debug.Log($"Active enemies: {enemyCount}");

        // 청크 순회
        var chunks = _enemyQuery.ToArchetypeChunkArray(Allocator.Temp);
        foreach (var chunk in chunks)
        {
            var healthArray = chunk.GetNativeArray(
                ref GetComponentTypeHandle<Health>());
            // 처리...
        }
        chunks.Dispose();
    }
}
```

### SystemAPI.Query

```csharp
[BurstCompile]
public partial struct QuerySystemAPISystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // 기본 쿼리
        foreach (var (transform, health) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Health>>())
        {
            // transform과 health 모두 있는 Entity 처리
        }

        // 필터링
        foreach (var (transform, health, entity) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Health>>()
                .WithAll<EnemyTag>()        // EnemyTag 필수
                .WithNone<Disabled>()        // Disabled 제외
                .WithEntityAccess())         // Entity 접근
        {
            // 적만 처리
        }
    }
}
```

---

## 메모리 레이아웃

### Archetype과 Chunk

```
Archetype: Position + Velocity + Health 조합

Chunk 0 (16KB):
┌─────────────────────────────────────────────────────────┐
│ Position[] │ Velocity[] │ Health[] │ Entity[] │ 여유공간 │
│ [0..127]   │ [0..127]   │ [0..127] │ [0..127] │          │
└─────────────────────────────────────────────────────────┘

Chunk 1 (16KB):
┌─────────────────────────────────────────────────────────┐
│ Position[] │ Velocity[] │ Health[] │ Entity[] │ 여유공간 │
│ [128..255] │ [128..255] │ [128..255]│[128..255]│          │
└─────────────────────────────────────────────────────────┘
```

### 캐시 효율성

```csharp
// ECS: 캐시 친화적
// Position 데이터가 연속 배열로 저장
// CPU 캐시 히트율 매우 높음

// GameObject: 캐시 비효율적
// 각 GameObject의 Transform이 힙에 분산
// 캐시 미스 빈번
```

---

## 변환 (Baking)

### Baker

```csharp
using Unity.Entities;
using UnityEngine;

// MonoBehaviour Authoring 컴포넌트
public class EnemyAuthoring : MonoBehaviour
{
    public int maxHealth = 100;
    public float speed = 5f;
}

// Baker: MonoBehaviour → ECS Component 변환
public class EnemyBaker : Baker<EnemyAuthoring>
{
    public override void Bake(EnemyAuthoring authoring)
    {
        var entity = GetEntity(TransformUsageFlags.Dynamic);

        AddComponent(entity, new Health
        {
            Current = authoring.maxHealth,
            Max = authoring.maxHealth
        });

        AddComponent(entity, new Velocity
        {
            Value = new float3(0, 0, authoring.speed)
        });

        AddComponent<EnemyTag>(entity);
    }
}
```

### SubScene

```csharp
// SubScene에 배치된 GameObject들이 자동으로 Entity로 변환됨
// Edit Time: GameObject로 편집
// Runtime: Entity로 실행
```

---

## 실전 예제

### 총알 시스템

```csharp
// Components
public struct Bullet : IComponentData
{
    public float Speed;
    public float Lifetime;
    public int Damage;
}

// System
[BurstCompile]
public partial struct BulletSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;
        var ecb = SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>()
            .CreateCommandBuffer(state.WorldUnmanaged);

        foreach (var (transform, bullet, entity) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRW<Bullet>>()
                .WithEntityAccess())
        {
            // 이동
            transform.ValueRW.Position += new float3(0, 0, bullet.ValueRO.Speed * deltaTime);

            // 수명 감소
            bullet.ValueRW.Lifetime -= deltaTime;

            // 수명 종료 시 제거
            if (bullet.ValueRO.Lifetime <= 0)
            {
                ecb.DestroyEntity(entity);
            }
        }
    }
}
```

### 데미지 시스템

```csharp
// Components
public struct DamageEvent : IComponentData
{
    public Entity Target;
    public int Amount;
}

// System
[BurstCompile]
public partial struct DamageSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>()
            .CreateCommandBuffer(state.WorldUnmanaged);

        foreach (var (damageEvent, entity) in
            SystemAPI.Query<RefRO<DamageEvent>>()
                .WithEntityAccess())
        {
            var target = damageEvent.ValueRO.Target;

            if (SystemAPI.HasComponent<Health>(target))
            {
                var health = SystemAPI.GetComponentRW<Health>(target);
                health.ValueRW.Current -= damageEvent.ValueRO.Amount;

                if (health.ValueRO.Current <= 0)
                {
                    ecb.DestroyEntity(target);
                }
            }

            // 이벤트 Entity 제거
            ecb.DestroyEntity(entity);
        }
    }
}
```

---

## 성능 비교

### 10,000 오브젝트 이동

```csharp
// GameObject: ~16ms (60 FPS 불가)
public class GameObjectMovement : MonoBehaviour
{
    public float speed = 1f;
    void Update()
    {
        transform.position += Vector3.forward * speed * Time.deltaTime;
    }
}

// ECS: ~0.2ms (수백만 개 가능)
[BurstCompile]
public partial struct ECSMovement : IJobEntity
{
    public float DeltaTime;
    public float Speed;

    void Execute(ref LocalTransform transform)
    {
        transform.Position += new float3(0, 0, Speed * DeltaTime);
    }
}
```

---

## 주의사항

### 학습 곡선

```
- ECS는 패러다임 전환 필요
- 기존 OOP 사고방식과 다름
- 초기 학습 비용 높음
```

### 사용 시기

| 상황 | 권장 |
|------|------|
| 수천~수백만 오브젝트 | ✅ ECS |
| 복잡한 상호작용 | ⚠️ 하이브리드 |
| 프로토타입/소규모 | ❌ GameObject |
| 성능 크리티컬 | ✅ ECS |

### 하이브리드 접근

```csharp
// ECS와 MonoBehaviour 혼용
public class HybridExample : MonoBehaviour
{
    private World _world;
    private EntityManager _entityManager;

    void Start()
    {
        _world = World.DefaultGameObjectInjectionWorld;
        _entityManager = _world.EntityManager;

        // MonoBehaviour에서 Entity 생성
        var entity = _entityManager.CreateEntity(typeof(Position));
        _entityManager.SetComponentData(entity, new Position { Value = transform.position });
    }

    void Update()
    {
        // UI, 입력 등은 MonoBehaviour에서 처리
        // 대량 처리는 ECS System에서
    }
}
```

---

## 정리

### ECS 요약

| 항목 | 내용 |
|------|------|
| **Entity** | 고유 ID (int) |
| **Component** | 순수 데이터 (IComponentData) |
| **System** | 로직 처리 (ISystem, SystemBase) |
| **Archetype** | 컴포넌트 조합 |
| **Chunk** | 메모리 블록 (16KB) |

### 체크리스트

- [ ] 대량 오브젝트 처리가 필요한가?
- [ ] 성능이 중요한 시스템인가?
- [ ] 팀이 ECS에 익숙한가?
- [ ] Burst/Job과 함께 사용하는가?

---

## 참고 자료

- [Unity DOTS Documentation](https://docs.unity3d.com/Packages/com.unity.entities@latest)
- [ECS Samples](https://github.com/Unity-Technologies/EntityComponentSystemSamples)
- [DOTS Best Practices](https://docs.unity3d.com/Packages/com.unity.entities@latest/manual/ecs-best-practices.html)
- [Baking Overview](https://docs.unity3d.com/Packages/com.unity.entities@latest/manual/baking-overview.html)
