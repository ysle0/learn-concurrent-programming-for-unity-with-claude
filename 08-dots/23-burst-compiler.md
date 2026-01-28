# 23. Burst Compiler

## 개요

**Burst Compiler**는 Unity의 **고성능 LLVM 기반 컴파일러**입니다. C# Job 코드를 최적화된 네이티브 코드로 변환하여 **C++ 수준의 성능**을 달성합니다.

---

## 설치

### Unity Package Manager

```
com.unity.burst
```

---

## 기본 사용법

### [BurstCompile] 속성

```csharp
using Unity.Jobs;
using Unity.Collections;
using Unity.Burst;
using Unity.Mathematics;

[BurstCompile] // Burst 컴파일 활성화
public struct BurstJob : IJobParallelFor
{
    [ReadOnly]
    public NativeArray<float> Input;

    [WriteOnly]
    public NativeArray<float> Output;

    public void Execute(int index)
    {
        // Burst로 최적화되는 코드
        Output[index] = math.sin(Input[index]) * math.cos(Input[index]);
    }
}
```

### 성능 비교

```csharp
// 100만 개 요소 처리

// Mono (Burst 없음): ~150ms
// IL2CPP (Burst 없음): ~50ms
// Burst: ~5ms (30배 향상!)
```

---

## Burst 옵션

### 컴파일 옵션

```csharp
[BurstCompile(
    CompileSynchronously = false,    // 비동기 컴파일 (기본)
    FloatMode = FloatMode.Fast,      // 부동소수점 모드
    FloatPrecision = FloatPrecision.Standard,
    DisableSafetyChecks = false      // 안전성 검사
)]
public struct OptimizedJob : IJob
{
    public NativeArray<float> Data;

    public void Execute()
    {
        for (int i = 0; i < Data.Length; i++)
        {
            Data[i] *= 2f;
        }
    }
}
```

### FloatMode 옵션

| 모드 | 설명 | 성능 |
|------|------|------|
| **Default** | 표준 IEEE 754 | 기본 |
| **Strict** | 정확한 IEEE 754 | 느림 |
| **Deterministic** | 결정적 결과 | 중간 |
| **Fast** | 빠른 근사치 | 가장 빠름 |

```csharp
// Fast 모드: NaN/Inf 처리 무시, 연산 순서 변경 허용
[BurstCompile(FloatMode = FloatMode.Fast)]
public struct FastMathJob : IJob { ... }
```

### DisableSafetyChecks

```csharp
// 프로덕션 빌드에서 안전성 검사 비활성화
[BurstCompile(DisableSafetyChecks = true)]
public struct ProductionJob : IJob
{
    public NativeArray<int> Data;

    public void Execute()
    {
        // 경계 검사 비활성화로 성능 향상
        for (int i = 0; i < Data.Length; i++)
        {
            Data[i] *= 2;
        }
    }
}
```

---

## Unity.Mathematics

### 기본 타입

```csharp
using Unity.Mathematics;

public class MathematicsTypes : MonoBehaviour
{
    void Example()
    {
        // 벡터 타입
        float2 v2 = new float2(1f, 2f);
        float3 v3 = new float3(1f, 2f, 3f);
        float4 v4 = new float4(1f, 2f, 3f, 4f);

        // 정수 벡터
        int3 i3 = new int3(1, 2, 3);

        // 행렬
        float4x4 matrix = float4x4.identity;
        float3x3 rotation = float3x3.identity;

        // 쿼터니언
        quaternion q = quaternion.identity;

        // Swizzle (HLSL 스타일)
        float3 xyz = v4.xyz;
        float2 xy = v3.xy;
    }
}
```

### math 함수

```csharp
[BurstCompile]
public struct MathFunctionsJob : IJobParallelFor
{
    public NativeArray<float3> Positions;
    public float3 Target;

    public void Execute(int index)
    {
        float3 pos = Positions[index];

        // 삼각함수
        float s = math.sin(pos.x);
        float c = math.cos(pos.y);

        // 거리
        float dist = math.distance(pos, Target);

        // 정규화
        float3 dir = math.normalize(Target - pos);

        // 내적/외적
        float dot = math.dot(dir, math.up());
        float3 cross = math.cross(dir, math.up());

        // 보간
        float3 lerped = math.lerp(pos, Target, 0.5f);

        // 클램프
        float clamped = math.clamp(dist, 0f, 100f);

        // 최소/최대
        float minVal = math.min(pos.x, pos.y);
        float maxVal = math.max(pos.y, pos.z);

        // 절대값
        float3 absPos = math.abs(pos);

        Positions[index] = lerped;
    }
}
```

### 랜덤

```csharp
using Unity.Mathematics;

[BurstCompile]
public struct RandomJob : IJobParallelFor
{
    public NativeArray<float3> Positions;
    public uint Seed;

    public void Execute(int index)
    {
        // Unity.Mathematics.Random (Burst 호환)
        var random = new Unity.Mathematics.Random(Seed + (uint)index);

        // 랜덤 값 생성
        float randomFloat = random.NextFloat();
        float rangeFloat = random.NextFloat(-10f, 10f);
        float3 randomDir = random.NextFloat3Direction();

        Positions[index] = randomDir * rangeFloat;
    }
}
```

---

## SIMD 최적화

### 자동 벡터화

```csharp
[BurstCompile]
public struct SIMDJob : IJobParallelFor
{
    [ReadOnly]
    public NativeArray<float4> A;

    [ReadOnly]
    public NativeArray<float4> B;

    [WriteOnly]
    public NativeArray<float4> Result;

    public void Execute(int index)
    {
        // Burst가 자동으로 SIMD 명령어로 변환
        // SSE/AVX (x86) 또는 NEON (ARM)
        Result[index] = A[index] + B[index];
    }
}
```

### 명시적 SIMD

```csharp
using Unity.Burst.Intrinsics;

[BurstCompile]
public struct ExplicitSIMDJob : IJob
{
    public NativeArray<float> Data;

    public void Execute()
    {
        // 명시적 SIMD 사용 (고급)
        int simdWidth = 4; // SSE: 4 floats
        int i = 0;

        for (; i <= Data.Length - simdWidth; i += simdWidth)
        {
            // 4개 float를 동시에 처리
            v128 vec = new v128(
                Data[i],
                Data[i + 1],
                Data[i + 2],
                Data[i + 3]);

            v128 result = X86.Sse.mul_ps(vec, new v128(2f));

            Data[i] = result.Float0;
            Data[i + 1] = result.Float1;
            Data[i + 2] = result.Float2;
            Data[i + 3] = result.Float3;
        }

        // 나머지 처리
        for (; i < Data.Length; i++)
        {
            Data[i] *= 2f;
        }
    }
}
```

---

## Burst 제약사항

### 허용되지 않는 기능

```csharp
[BurstCompile]
public struct RestrictedJob : IJob
{
    public void Execute()
    {
        // ❌ 허용 안 됨
        // string str = "Hello";           // 관리 타입
        // object obj = new object();      // 박싱
        // try { } catch { }              // 예외 처리 (일부 제한)
        // Debug.Log("test");              // 관리 메서드 호출
        // throw new Exception();          // 예외 던지기

        // ✅ 허용됨
        int x = 10;
        float y = math.sin(x);
        // NativeArray 접근
        // 값 타입 연산
        // math 함수
    }
}
```

### SharedStatic

```csharp
// 정적 필드 대안
public struct JobSharedData
{
    public static readonly SharedStatic<int> Counter =
        SharedStatic<int>.GetOrCreate<JobSharedData>();
}

[BurstCompile]
public struct SharedStaticJob : IJob
{
    public void Execute()
    {
        // SharedStatic 읽기/쓰기 (Burst 호환)
        JobSharedData.Counter.Data++;
    }
}
```

### FunctionPointer

```csharp
// 델리게이트 대안
[BurstCompile]
public struct FunctionPointerExample
{
    public delegate float MathOperation(float a, float b);

    [BurstCompile]
    public static float Add(float a, float b) => a + b;

    [BurstCompile]
    public static float Multiply(float a, float b) => a * b;
}

public class FunctionPointerUsage : MonoBehaviour
{
    void Start()
    {
        // Burst 컴파일된 함수 포인터
        var addPtr = BurstCompiler.CompileFunctionPointer<FunctionPointerExample.MathOperation>(
            FunctionPointerExample.Add);

        float result = addPtr.Invoke(5f, 3f); // 8
    }
}
```

---

## 디버깅

### Burst Inspector

```
Window > Burst > Burst Inspector

기능:
- 생성된 어셈블리 코드 확인
- SIMD 명령어 확인
- 컴파일 오류 확인
- 성능 힌트
```

### 조건부 컴파일

```csharp
[BurstCompile]
public struct DebugJob : IJob
{
    public NativeArray<float> Data;

    public void Execute()
    {
        for (int i = 0; i < Data.Length; i++)
        {
            Data[i] *= 2f;

            // Burst에서도 디버그 출력 가능
            #if ENABLE_UNITY_COLLECTIONS_CHECKS
            Unity.Debug.Log($"Processed index {i}");
            #endif
        }
    }
}
```

### 어설션

```csharp
using Unity.Assertions;

[BurstCompile]
public struct AssertJob : IJob
{
    public NativeArray<float> Data;

    public void Execute()
    {
        // Burst 호환 어설션
        Assert.IsTrue(Data.Length > 0);
        Assert.AreEqual(Data.Length, 100);

        for (int i = 0; i < Data.Length; i++)
        {
            Assert.IsTrue(Data[i] >= 0);
            Data[i] = math.sqrt(Data[i]);
        }
    }
}
```

---

## 실전 패턴

### 패턴 1: 노이즈 생성

```csharp
[BurstCompile]
public struct NoiseJob : IJobParallelFor
{
    public NativeArray<float> Heights;
    public int Width;
    public float Scale;
    public float2 Offset;

    public void Execute(int index)
    {
        int x = index % Width;
        int y = index / Width;

        float2 pos = new float2(x, y) * Scale + Offset;

        // Perlin 노이즈 (Unity.Mathematics 사용)
        float height = noise.cnoise(pos);

        // 옥타브
        height += noise.cnoise(pos * 2) * 0.5f;
        height += noise.cnoise(pos * 4) * 0.25f;

        Heights[index] = height;
    }
}
```

### 패턴 2: 충돌 감지

```csharp
[BurstCompile]
public struct CollisionJob : IJobParallelFor
{
    [ReadOnly]
    public NativeArray<float3> Positions;

    [ReadOnly]
    public NativeArray<float> Radii;

    public NativeArray<int> CollisionCounts;

    public void Execute(int i)
    {
        int count = 0;
        float3 posA = Positions[i];
        float radiusA = Radii[i];

        for (int j = 0; j < Positions.Length; j++)
        {
            if (i == j) continue;

            float3 posB = Positions[j];
            float radiusB = Radii[j];

            float distance = math.distance(posA, posB);
            float combinedRadius = radiusA + radiusB;

            if (distance < combinedRadius)
            {
                count++;
            }
        }

        CollisionCounts[i] = count;
    }
}
```

### 패턴 3: 보이드 시뮬레이션

```csharp
[BurstCompile]
public struct BoidJob : IJobParallelFor
{
    [ReadOnly]
    public NativeArray<float3> Positions;

    [ReadOnly]
    public NativeArray<float3> Velocities;

    public NativeArray<float3> NewVelocities;

    public float SeparationWeight;
    public float AlignmentWeight;
    public float CohesionWeight;
    public float NeighborRadius;
    public float MaxSpeed;

    public void Execute(int index)
    {
        float3 pos = Positions[index];
        float3 vel = Velocities[index];

        float3 separation = float3.zero;
        float3 alignment = float3.zero;
        float3 cohesion = float3.zero;
        int neighborCount = 0;

        for (int i = 0; i < Positions.Length; i++)
        {
            if (i == index) continue;

            float3 otherPos = Positions[i];
            float dist = math.distance(pos, otherPos);

            if (dist < NeighborRadius)
            {
                // 분리
                float3 diff = pos - otherPos;
                separation += math.normalize(diff) / dist;

                // 정렬
                alignment += Velocities[i];

                // 결집
                cohesion += otherPos;

                neighborCount++;
            }
        }

        if (neighborCount > 0)
        {
            alignment /= neighborCount;
            cohesion = (cohesion / neighborCount) - pos;
        }

        float3 newVel = vel
            + separation * SeparationWeight
            + alignment * AlignmentWeight
            + cohesion * CohesionWeight;

        // 속도 제한
        float speed = math.length(newVel);
        if (speed > MaxSpeed)
        {
            newVel = math.normalize(newVel) * MaxSpeed;
        }

        NewVelocities[index] = newVel;
    }
}
```

---

## 성능 팁

### 루프 최적화

```csharp
[BurstCompile]
public struct OptimizedLoopJob : IJob
{
    public NativeArray<float> Data;

    public void Execute()
    {
        int length = Data.Length;

        // ✅ 좋음: 길이를 로컬 변수에 캐시
        for (int i = 0; i < length; i++)
        {
            Data[i] *= 2f;
        }

        // ❌ 피할 것: 매 반복마다 Length 접근
        // for (int i = 0; i < Data.Length; i++)
    }
}
```

### 브랜치 최소화

```csharp
[BurstCompile]
public struct BranchlessJob : IJobParallelFor
{
    public NativeArray<float> Values;

    public void Execute(int index)
    {
        float value = Values[index];

        // ❌ 브랜치 사용
        // if (value < 0) value = 0;
        // if (value > 1) value = 1;

        // ✅ 브랜치 없는 클램프
        value = math.clamp(value, 0f, 1f);

        // ✅ select 사용 (조건부 선택)
        float result = math.select(value, 0f, value < 0);

        Values[index] = result;
    }
}
```

---

## 정리

### Burst 요약

| 항목 | 내용 |
|------|------|
| **용도** | Job System 코드 최적화 |
| **성능** | 10~30배 향상 가능 |
| **핵심** | [BurstCompile] 속성 |
| **수학** | Unity.Mathematics 필수 |

### 체크리스트

- [ ] [BurstCompile] 속성 적용했는가?
- [ ] Unity.Mathematics 사용하는가?
- [ ] 관리 타입 사용 안 하는가?
- [ ] 예외 처리 최소화했는가?
- [ ] Burst Inspector로 검증했는가?

---

## 참고 자료

- [Burst User Guide](https://docs.unity3d.com/Packages/com.unity.burst@latest)
- [Unity.Mathematics](https://docs.unity3d.com/Packages/com.unity.mathematics@latest)
- [Burst Intrinsics](https://docs.unity3d.com/Packages/com.unity.burst@latest/manual/docs/AdvancedUsages.html)
