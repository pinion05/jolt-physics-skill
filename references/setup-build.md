# Jolt 빌드 & 연동 (CMake, defines, 링크 에러)

2026-09 기준 최신: **v5.6.0** (2026-07-11). 플랫폼: Windows/Linux/FreeBSD/macOS/iOS/Android/MSYS2/WASM.
컴파일러: VS2022+(VS2026 지원), Clang 16+, GCC 12+. C++17. x86 SSE2 기본(SSE4.1/4.2/AVX/AVX2/AVX512 옵션), ARM64 NEON, RISC-V RVV.

## FetchContent로 연동 (공식 JoltPhysicsHelloWorld 예제에서 검증된 패턴)

**핵심: CMakeLists.txt가 저장소 루트가 아니라 `Build/` 아래에 있다.** `SOURCE_SUBDIR "Build"` 없으면 FetchContent가 실패한다.

```cmake
cmake_minimum_required(VERSION 3.20 FATAL_ERROR)
project(MyGame CXX)
include(FetchContent)

set(DOUBLE_PRECISION OFF)              # 큰 세계(>5km) 필요시 ON, ~5-10% 느림
set(CROSS_PLATFORM_DETERMINISTIC OFF)  # 플랫폼 간 결정론 필요시 ON, ~8% 느림
set(INTERPROCEDURAL_OPTIMIZATION ON)   # LTO — 내 프로젝트에서도 켜야 함(아래 링크 에러 참고)
set(CPP_EXCEPTIONS_ENABLED OFF)        # 기본값
set(CPP_RTTI_ENABLED OFF)              # 기본값
set(OBJECT_LAYER_BITS 16)              # 레이어 32개 필요시 32
set(USE_SSE4_1 ON)  set(USE_AVX2 ON)   # 타깃 CPU에 맞게. USE_AVX512 기본 OFF

FetchContent_Declare(JoltPhysics
    GIT_REPOSITORY "https://github.com/jrouwe/JoltPhysics"
    GIT_TAG "v5.6.0"
    SOURCE_SUBDIR "Build")
FetchContent_MakeAvailable(JoltPhysics)

add_executable(MyGame src/main.cpp)
target_include_directories(MyGame PUBLIC ${JoltPhysics_SOURCE_DIR}/..)
target_link_libraries(MyGame Jolt)     # 타깃명 Jolt, 익스포트명 Jolt::Jolt
```

체크아웃을 `add_subdirectory`로 쓸 때: `add_subdirectory(JoltPhysics/Build)`.
vcpkg/xmake/conan 패키지도 존재(Build/README.md 링크).

## 주요 CMake 옵션 (기본값)

- `USE_ASSERTS` OFF — 디버깅 빌드에서 켜라 (`JPH_ENABLE_ASSERTS`)
- `DOUBLE_PRECISION` OFF — `JPH_DOUBLE_PRECISION`
- `CROSS_PLATFORM_DETERMINISTIC` OFF — `JPH_CROSS_PLATFORM_DETERMINISTIC` (FMADD 무시, `/fp:precise`·`-ffp-contract=off` 강제)
- `FLOATING_POINT_EXCEPTIONS_ENABLED` ON (Windows 전용, `JPH_FLOATING_POINT_EXCEPTIONS_ENABLED`)
- `CPP_EXCEPTIONS_ENABLED` / `CPP_RTTI_ENABLED` OFF — **내 프로젝트와 일치해야 함**
- `DEBUG_RENDERER_IN_DEBUG_AND_RELEASE` ON / `DEBUG_RENDERER_IN_DISTRIBUTION` OFF (`JPH_DEBUG_RENDERER`)
- `PROFILER_IN_DEBUG_AND_RELEASE` ON (`JPH_PROFILE_ENABLED`)
- `ENABLE_OBJECT_STREAM` ON (`JPH_OBJECT_STREAM`)
- `OBJECT_LAYER_BITS` 16
- `INTERPROCEDURAL_OPTIMIZATION` ON (LTO)
- 통계: `TRACK_BROADPHASE_STATS` / `TRACK_NARROWPHASE_STATS` / `JPH_TRACK_SIMULATION_STATS` (5.5+, 바디별 비용 추적)
- 타깅 앱: `TARGET_UNIT_TESTS`, `TARGET_HELLO_WORLD`, `TARGET_PERFORMANCE_TEST`, `TARGET_SAMPLES`, `TARGET_VIEWER` (전부 ON)
- SIMD: `USE_SSE4_1/USE_SSE4_2/USE_AVX/USE_AVX2/USE_LZCNT/USE_TZCNT/USE_F16C/USE_FMADD` ON, `USE_AVX512` OFF

빌드 구성 3종: **Debug**(assert), **Release**(프로파일링+디버그 드로잉 포함), **Distribution**(모든 디버깅 지원 제거 — 출하용).

## 링크/크래시 에러 → 원인 (Build/README.md)

| 증상 | 원인 | 조치 |
|---|---|---|
| "file format not recognized" | LTO(IPC) 불일치 | 내 프로젝트도 `CMAKE_INTERPROCEDURAL_OPTIMIZATION` ON |
| unresolved `GetSubmergedVolume` | `JPH_DEBUG_RENDERER` 불일치 | 양쪽 같은 값으로 |
| typeinfo undefined | RTTI 불일치 | `CPP_RTTI_ENABLED` 양쪽 일치 |
| "Illegal instruction" | SIMD 세트 초과 | Jolt를 타깃 CPU에 맞게 낮춰 재빌드 |
| 공유 라이브러리 혼합 시 이상 동작(#2139) | defines/`NDEBUG` 불일치 | `JPH_DOUBLE_PRECISION`, `JPH_OBJECT_STREAM`, `NDEBUG` 전부 동일하게, `Trace` 콜백으로 경로 노출 |

## 소스 디렉터리 지도 (코드 찾을 때)

```
Jolt/Jolt.h              — 첫 헤더/PCH
Jolt/RegisterTypes.h     — RegisterTypes()/UnregisterTypes()
Jolt/Core/               — allocator, TempAllocator, JobSystem(ThreadPool/SingleThreaded), Mutex, Factory, RTTI, Stream, Array/UnorderedMap(5.3+ 커스텀), Profiler, Reference(Ref/RefConst)
Jolt/Math/               — Vec3/4, Quat, Mat44, DMat44, UVec4, DeterministicLog
Jolt/Physics/            — PhysicsSystem, PhysicsSettings, PhysicsStepListener, StateSave/StateRestore
Jolt/Physics/Body/       — Body, BodyInterface, BodyCreationSettings, BodyLockRead/Write, MotionProperties, MassProperties, BodyActivationListener
Jolt/Physics/Collision/  — NarrowPhaseQuery, GroupFilter(Table), ObjectLayer.h, ContactListener.h, collectors, InternalEdgeRemovingCollector, SimShapeFilter
Jolt/Physics/Collision/BroadPhase/  — BroadPhaseQuadTree, (ObjectLayerPair|BroadPhaseLayer|ObjectVsBroadPhaseLayer)Filter Table/Mask 헬퍼들
Jolt/Physics/Collision/Shape/       — 전체 셰이프 + Settings 클래스, SubShapeID
Jolt/Physics/Character/  — Character, CharacterVirtual, CharacterBase, ContactListener들
Jolt/Physics/Constraints/ — SixDOF, Hinge, Point, Distance, Slider, Cone, SwingTwist, Path, Gear, RackAndPinion, Pulley, Fixed, VehicleConstraint, MotorSettings
Jolt/Physics/Ragdoll/    — Ragdoll, RagdollSettings
Jolt/Physics/Vehicle/    — Wheeled/Tracked/MotorcycleController, Engine, Transmission, Wheel, AntiRollBar, Differential
Jolt/Physics/SoftBody/   — SoftBodyCreationSettings, SoftBodySharedSettings, SoftBodyShape, SoftBodyContactListener
Jolt/Physics/Hair/       — GPU 스트랜드 헤어 (5.6)
Jolt/Renderer/, Jolt/Shaders/ — DebugRenderer 추상화
Jolt/Compute/            — DX12/Vulkan/Metal 컴퓨트 인터페이스 (5.6)
Jolt/Skeleton/           — Skeleton, SkeletonPose, SkeletalAnimation, SkeletonMapper
Jolt/ObjectStream/       — RTTI 기반 텍스트/바이너리 직렬화 (선택)
HelloWorld/              — 최소 예제 (CC0 — 복붙 자유)
Samples/                 — 학습용 1순위. Tests/ 아래 카테고리별
UnitTests/               — doctest. PhysicsTestContext, LoggingContactListener 등 테스트 지지대
PerformanceTest/         — 벤치 (Docs/PerformanceTest.md)
JoltViewer/              — .jolt 녹화 스트림 재생
Docs/                    — Architecture.md(문서 사이트 원본), APIChanges.md, ReleaseNotes.md, Samples.md, ProjectsUsingJolt.md
```

## 스레딩

- `JobSystemThreadPool(cMaxPhysicsJobs, cMaxPhysicsBarriers, hardware_concurrency() - 1)` — 물리 스레드 = 코어수-1, 메인 스레드가 대기 중 잡을 함께 실행하므로 실제 동시성 = +1.
- `Update()`는 잡 그래프를 돌리고 **리턴 시 잡이 전부 끝난다.** 바디/컨스트레인트 조작은 Update 호출 사이에만.
- 커스텀 엔진 스케줄러에 얹으려면 `JobSystem` 인터페이스 직접 구현.
- TempAllocator는 Update 1회分 스크래치 메모리. Update 끝나면 전부 해제됨. `TempAllocatorImpl`(고정 풀), `TempAllocatorMalloc`(malloc), `TempAllocatorTracker`(누수 추적) 제공.

## 코딩 컨벤션 (PR/기여 시)

`.clang-format` + `.editorconfig` 강제 (탭, Allman). 헤더 관례: `JPH_NAMESPACE_BEGIN`, 파라미터 접두 `in`/`io`/`out`, static 함수 `s` 접두(`Quat::sIdentity()`), Settings 클래스 + `Create() → Result` 패턴, `Ref`/`RefConst` 스마트 포인터, `RVec3`(float/double 전환형).
PR 전 `ContributorAgreement.md`(FLA 2.0) 서명 필요.

## 참고 링크

- 빌드 문서: https://jrouwe.github.io/JoltPhysics/md__build_2_r_e_a_d_m_e.html
- FetchContent 예제: https://github.com/jrouwe/JoltPhysicsHelloWorld
- 릴리스: https://github.com/jrouwe/JoltPhysics/releases
- GDC 2022 "Architecting Jolt Physics for Horizon Forbidden West": https://gdcvault.com/play/1027891/
