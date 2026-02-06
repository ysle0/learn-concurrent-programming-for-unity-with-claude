# Unity 동시성 프로그래밍 완벽 가이드

Unity와 C#을 활용한 비동기/병렬 프로그래밍 교육 레포지토리입니다.

## 학습 목표

- Unity 환경에서의 동시성 프로그래밍 패턴 이해
- async/await, Coroutine, UniTask 등 다양한 비동기 기법 마스터
- 네트워크 통신 및 프로토콜 처리 방법 습득
- 플랫폼별 최적화 및 주의사항 파악

---

## 목차

### Part 1: 기초 개념 (Foundation)

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 01 | [동시성 프로그래밍 기초 개념](./01-fundamentals/01-concurrency-basics.md) | ⭐⭐⭐⭐⭐ | 동기/비동기, 병렬/동시성, Blocking/Non-blocking, CPU-bound vs I/O-bound |
| 02 | [유니티 엔진 라이프 사이클과 동시성](./01-fundamentals/02-unity-lifecycle.md) | ⭐⭐⭐⭐⭐ | Main Thread, PlayerLoop, Update 순서, 프레임 기반 실행 모델 |
| 03 | [SynchronizationContext](./01-fundamentals/03-synchronization-context.md) | ⭐⭐⭐⭐ | UnitySynchronizationContext, 스레드 마샬링, 컨텍스트 캡처 |

### Part 2: Thread 기초 (Threading Fundamentals)

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 04 | [Thread & ThreadPool](./02-threading/04-thread-and-threadpool.md) | ⭐⭐⭐⭐⭐ | Thread 생성, ThreadPool, Unity에서의 Thread 제약사항 |
| 05 | [동기화 기법](./02-threading/05-synchronization-primitives.md) | ⭐⭐⭐⭐ | lock, Monitor, Mutex, Semaphore, Interlocked, SpinLock |

### Part 3: 전통적인 비동기 패턴 (Legacy Patterns)

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 06 | [APM (Asynchronous Programming Model)](./03-legacy-patterns/06-apm.md) | ⭐⭐⭐ | BeginXxx/EndXxx 패턴, IAsyncResult (레거시 이해용) |
| 07 | [Coroutine](./03-legacy-patterns/07-coroutine.md) | ⭐⭐⭐⭐⭐ | IEnumerator, yield return, StartCoroutine, 생명주기 |

### Part 4: 현대적 비동기 패턴 - async/await (Modern Async Patterns)

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 08 | [async/await 기초](./04-async-await/08-async-await-basics.md) | ⭐⭐⭐⭐⭐ | 문법, 상태머신, 실행 흐름, 반환 타입 |
| 09 | [TAP (Task Asynchronous Programming)](./04-async-await/09-tap.md) | ⭐⭐⭐⭐⭐ | Task, Task<T>, Task.Run, Task.WhenAll, Task.WhenAny |
| 10 | [ConfigureAwait & 컨텍스트 제어](./04-async-await/10-configure-await.md) | ⭐⭐⭐⭐⭐ | ConfigureAwait(false), 데드락 방지, 컨텍스트 전환 |
| 11 | [ValueTask](./04-async-await/11-valuetask.md) | ⭐⭐⭐⭐ | Task vs ValueTask, allocation-free 비동기, IValueTaskSource |
| 12 | [Awaitable (Unity 2023+)](./04-async-await/12-awaitable.md) | ⭐⭐⭐⭐⭐ | Unity 네이티브 async/await, AwaitableCompletionSource |
| 13 | [UniTask](./04-async-await/13-unitask.md) | ⭐⭐⭐⭐⭐ | Zero-allocation, PlayerLoop 통합, UniTask vs Task |

### Part 5: 비동기 스트림 & 채널 (Async Streams & Channels)

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 14 | [IAsyncEnumerable](./05-async-streams/14-async-enumerable.md) | ⭐⭐⭐ | 비동기 스트림, await foreach, 점진적 데이터 처리 |
| 15 | [Channel & Pipeline](./05-async-streams/15-channels.md) | ⭐⭐⭐ | System.Threading.Channels, 생산자-소비자 패턴 |

### Part 6: Reactive Programming

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 16 | [UniRx](./06-reactive/16-unirx.md) | ⭐⭐⭐⭐ | Observable, Subject, Operators, Unity 이벤트 통합 |
| 17 | [R3](./06-reactive/17-r3.md) | ⭐⭐⭐⭐ | 차세대 Rx (neuecc), UniRx와의 차이점, 마이그레이션 |

### Part 7: Unity-Compatible C# 병렬 프로그래밍 도구

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 18 | [Parallel 클래스](./07-parallel-tools/18-parallel-class.md) | ⭐⭐⭐⭐ | Parallel.For, Parallel.ForEach, Parallel.Invoke |
| 19 | [PLINQ](./07-parallel-tools/19-plinq.md) | ⭐⭐⭐ | AsParallel(), 병렬 쿼리, Unity에서의 활용 |
| 20 | [Concurrent Collections](./07-parallel-tools/20-concurrent-collections.md) | ⭐⭐⭐⭐ | ConcurrentDictionary, ConcurrentQueue, ConcurrentBag, BlockingCollection |

### Part 8: Unity DOTS & Job System

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 21 | [ECS 개요](./08-dots/21-ecs-overview.md) | ⭐⭐⭐⭐ | Entities 패키지, World, Entity, Component, System |
| 22 | [Job System](./08-dots/22-job-system.md) | ⭐⭐⭐⭐ | IJob, IJobParallelFor, JobHandle, 의존성 관리 |
| 23 | [Burst Compiler](./08-dots/23-burst-compiler.md) | ⭐⭐⭐⭐ | SIMD, 네이티브 코드 최적화, 제약사항 |
| 24 | [NativeContainer](./08-dots/24-native-container.md) | ⭐⭐⭐⭐ | NativeArray, NativeList, NativeHashMap, 메모리 안전성 |

### Part 9: 네트워크 통신 (Networking)

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 25 | [UnityWebRequest](./09-networking/25-unity-web-request.md) | ⭐⭐⭐⭐⭐ | GET/POST, 다운로드/업로드, 인증서 처리 |
| 26 | [HttpClient](./09-networking/26-httpclient.md) | ⭐⭐⭐⭐ | HttpClient 재사용, Unity에서의 설정, 타임아웃 |

### Part 10: 프로토콜 & 직렬화 (Protocols & Serialization)

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 27 | [HTTP/REST & JSON](./10-protocols/27-http-rest-json.md) | ⭐⭐⭐⭐⭐ | REST API, JsonUtility, Newtonsoft.Json, System.Text.Json |
| 28 | [WebSocket](./10-protocols/28-websocket.md) | ⭐⭐⭐⭐ | ClientWebSocket, 실시간 양방향 통신, 재연결 전략 |
| 29 | [gRPC](./10-protocols/29-grpc.md) | ⭐⭐⭐⭐ | gRPC-Web, Unary/Streaming, Protocol Buffers |
| 30 | [MessagePack](./10-protocols/30-messagepack.md) | ⭐⭐⭐⭐ | MessagePack-CSharp, 고성능 바이너리 직렬화 |
| 31 | [MemoryPack](./10-protocols/31-memorypack.md) | ⭐⭐⭐⭐ | Zero-encoding, 초고속 직렬화, Source Generator |
| 32 | [FlatBuffers](./10-protocols/32-flatbuffers.md) | ⭐⭐⭐ | Zero-copy 접근, 게임 데이터에 적합 |
| 33 | [GraphQL](./10-protocols/33-graphql.md) | ⭐⭐⭐ | 클라이언트 쿼리, Unity GraphQL 라이브러리 |

### Part 11: 취소 & 에러 핸들링 (Cancellation & Error Handling)

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 34 | [CancellationToken](./11-cancellation-error/34-cancellation-token.md) | ⭐⭐⭐⭐⭐ | CancellationTokenSource, 연결된 토큰, 타임아웃 |
| 35 | [예외 처리 패턴](./11-cancellation-error/35-exception-handling.md) | ⭐⭐⭐⭐⭐ | AggregateException, 비동기 예외 전파, Try 패턴 |
| 36 | [생명주기 관리](./11-cancellation-error/36-lifecycle-management.md) | ⭐⭐⭐⭐⭐ | OnDestroy 취소, Scene 전환, DisposableBag |

### Part 12: 플랫폼별 고려사항 (Platform Considerations)

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 37 | [OS별 주의사항](./12-platform/37-os-considerations.md) | ⭐⭐⭐⭐ | iOS/Android 백그라운드 제한, WebGL 제약 |
| 38 | [IL2CPP & AOT](./12-platform/38-il2cpp-aot.md) | ⭐⭐⭐⭐ | 코드 스트리핑, 리플렉션 제한, link.xml |

### Part 13: 최적화 & 디버깅 (Optimization & Debugging)

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 39 | [메모리 관리 & GC](./13-optimization/39-memory-gc.md) | ⭐⭐⭐⭐ | allocation 줄이기, Object Pooling, Span<T> |
| 40 | [프로파일링 & 디버깅](./13-optimization/40-profiling-debugging.md) | ⭐⭐⭐⭐ | Unity Profiler, async 스택 트레이스, 데드락 진단 |
| 41 | [테스트 작성](./13-optimization/41-testing.md) | ⭐⭐⭐ | 비동기 유닛 테스트, EditMode/PlayMode 테스트 |

### Part 14: Source Generator & 고급 기법

| # | 섹션 | 중요도 | 설명 |
|---|------|--------|------|
| 42 | [Source Generator 활용](./14-advanced/42-source-generator.md) | ⭐⭐⭐ | 보일러플레이트 자동 생성, MemoryPack/R3 활용 |

---

## 진행 상황

- [x] Part 1: 기초 개념 (3/3) ✅
- [x] Part 2: Thread 기초 (2/2) ✅
- [x] Part 3: 전통적인 비동기 패턴 (2/2) ✅
- [x] Part 4: async/await (6/6) ✅
- [x] Part 5: 비동기 스트림 & 채널 (2/2) ✅
- [x] Part 6: Reactive Programming (2/2) ✅
- [x] Part 7: C# 병렬 도구 (3/3) ✅
- [x] Part 8: Unity DOTS (4/4) ✅
- [x] Part 9: 네트워크 통신 (2/2) ✅
- [ ] Part 10: 프로토콜 & 직렬화 (0/7)
- [x] Part 11: 취소 & 에러 핸들링 (3/3) ✅
- [ ] Part 12: 플랫폼별 고려사항 (0/2)
- [ ] Part 13: 최적화 & 디버깅 (0/3)
- [ ] Part 14: Source Generator (0/1)

**총 42개 섹션 중 29개 완료 (69%)**

---

## 사용 방법

각 섹션은 다음 구조로 구성되어 있습니다:

1. **개념 설명** - 이론적 배경과 핵심 개념
2. **Unity 예제 코드** - 실제 사용 가능한 C# 코드
3. **주의사항** - 흔한 실수와 함정
4. **베스트 프랙티스** - 권장 패턴과 사용법
5. **참고 자료** - 추가 학습 링크

---

## 환경 요구사항

- Unity 2021.3 LTS 이상 (일부 섹션은 Unity 2023+ 필요)
- .NET Standard 2.1 / .NET 6+
- IDE: Visual Studio 2022 / Rider 추천

---

## 라이선스

이 교육 자료는 학습 목적으로 자유롭게 사용할 수 있습니다.
