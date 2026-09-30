---
name: jolt-physics
version: 1.0.0
description: Jolt Physics(jrouwe/JoltPhysics, C++ 리지드바디 물리엔진 — Horizon Forbidden West/Death Stranding 2/Godot 4 기본 3D 물리)로 시뮬레이션·캐릭터 컨트롤러·차량·래그돌·소프트바디·충돌 쿼리 코드를 작성·디버깅·튜닝할 때 쓴다. 2026-09 기준 v5.6.0 공식 문서 + jrouwe가 직접 답한 GitHub 이슈들에서 검증된 API 정확한 이름, 초기화 시퀀스, PhysicsSettings 튜닝값, 터널링/지터/슬리핑/scale 함정 해법이 내장되어 있다. 트리거 — "Jolt", "JoltPhysics", "jolt physics", CharacterVirtual, BodyInterface, PhysicsSystem, JoltPhysics.js(jolt-physics npm), Godot Jolt, VPhysics-Jolt, "물리엔진 Jolt", C++ 물리엔진 연동/통합, 래그돌·차량·컨베이어 물리 구현, 터널링/지터/쌓임 불안정 튜닝.
---

# Jolt Physics (C++)

버전 1.0.0 — 2026-09-30 리서치 기준, Jolt v5.6.0 (2026-07-11). MIT 라이선스, C++17,
RTTI/예외 기본 OFF, 의존성 없음(STL만). Windows/Linux/macOS/iOS/Android/WASM.

**정확한 API 이름이 중요하다.** 아래 요약 + references/ 의 검증된 이름을 그대로 쓰고,
의심나면 공식 문서에서 확인한다 — 클래스 페이지 URL 패턴:
`https://jrouwe.github.io/JoltPhysics/class_<snake_case>.html`
(예: `class_character_virtual.html`, `class_physics_system.html`).
FAQ 페이지는 존재하지 않는다(404). 단위/한계는 Architecture 문서 "Conventions and Limits" 절에 있다.

## 핵심 팩트 (매번 맞춰야 하는 것)

| 항목 | 값 |
|---|---|
| 단위 | SI — 미터/킬로그램/초, 라디안. 오른손 좌표계, **Y-up**, column-major |
| 권장 스케일 | 다이나믹 오브젝트 [0.1, 10] m, 정적 [0.1, 2000] m, 속도 ≤ 500 m/s, 중력 [0, 10] m/s² |
| 기본 스텝 | 60 Hz + collision step 1 이 안정 기준. **고정 dt accumulator 필수**, 가변 dt 금지 |
| 좌표 기준 | 바디 position은 질량중심(COM)이 아니라 원점 기준이나, 대부분 API/속도는 **COM 기준** |
| 결정론 | 같은 바이너리 + 같은 API 호출 순서 → 결정론적. 크로스플랫폼은 `CROSS_PLATFORM_DETERMINISTIC=ON` (~8% 느림) |
| 최신 | v5.6.0: GPU compute(DX12/VK/Metal), GPU 헤어, 새 마찰 모델(~40% perf↑) |

## 작업별 레퍼런스 라우팅

코드 작성 전 해당 파일을 먼저 읽는다:

| 작업 | 읽을 파일 |
|---|---|
| 빌드 연동 (CMake/FetchContent/defines/링크 에러) | [references/setup-build.md](references/setup-build.md) |
| 초기화, 레이어, 바디/셰이프 생성, 메인 루프, 힘/속도 | [references/core-api.md](references/core-api.md) |
| 레이캐스트/샤이프캐스트/충돌 쿼리, ContactListener, 센서/트리거 | [references/queries-contacts.md](references/queries-contacts.md) |
| 컨스트레인트(힌지/슬라이더/6DOF…), 모터, 차량, 래그돌, 소프트바디 | [references/constraints-vehicles.md](references/constraints-vehicles.md) |
| 플레이어 캐릭터 컨트롤러 (CharacterVirtual) | [references/characters.md](references/characters.md) |
| 튜닝(PhysicsSettings), 터널링/지터/슬리핑/scale 문제, 결정론, 직렬화 | [references/tuning-troubleshooting.md](references/tuning-troubleshooting.md) |
| Godot/Unreal/Unity/WASM/Rust/C# 바인딩, 도입 사례 | [references/integrations.md](references/integrations.md) |

## 최소 시뮬레이션 루프 (HelloWorld 골격, 그대로 복사해도 되는 패턴)

```cpp
#include <Jolt/Jolt.h>        // 반드시 첫 헤더 (PCH로 쓰기에도 최적)
#include <Jolt/RegisterTypes.h>
#include <Jolt/Core/Factory.h>
#include <Jolt/Core/TempAllocator.h>
#include <Jolt/Core/JobSystemThreadPool.h>
#include <Jolt/Physics/PhysicsSystem.h>
#include <Jolt/Physics/Body/BodyCreationSettings.h>
#include <Jolt/Physics/Collision/Shape/BoxShape.h>

using namespace JPH;
using namespace JPH::literals;   // 0.0_r 리터럴 → JPH_DOUBLE_PRECISION에 따라 float/double

// 1) 원격 준비 (다른 Jolt 함수보다 먼저)
RegisterDefaultAllocator();
Trace = TraceImpl;               // 필수는 아니지만 권장
Factory::sInstance = new Factory();   // 직렬화/역직렬화에 필요
RegisterTypes();

// 2) 지원 객체
TempAllocatorImpl temp_allocator(10 * 1024 * 1024);   // 스텝 중 할당 방지용 10MB
JobSystemThreadPool job_system(cMaxPhysicsJobs, cMaxPhysicsBarriers,
                               std::thread::hardware_concurrency() - 1);

// 3) 레이어 (인터페이스들은 PhysicsSystem보다 오래 살아있어야 한다 — 참조로 저장됨)
//    ObjectLayer(다수, 얇은 필터) → BroadPhaseLayer(소수, 브로드페이스 트리 버킷)
//    헬퍼: ObjectLayerPairFilterTable / BroadPhaseLayerInterfaceTable /
//          ObjectVsBroadPhaseLayerFilterTable  (자세한 코드는 core-api.md)

// 4) PhysicsSystem
PhysicsSystem physics_system;
physics_system.Init(/*maxBodies*/ 65536, /*numBodyMutexes*/ 0,
                    /*maxBodyPairs*/ 65536, /*maxContactConstraints*/ 10240,
                    broad_phase_layer_interface, object_vs_broadphase_layer_filter,
                    object_layer_pair_filter);
// maxBodyPairs/maxContactConstraints 초과분은 "떨어져서 세계 밖으로 빠짐" — 넉넉히

// 5) 바디
BodyInterface &bi = physics_system.GetBodyInterface();   // 락킹 버전 (NoLock 버전은 리스너 콜백 안에서만)
BodyID sphere = bi.CreateAndAddBody(
    BodyCreationSettings(new SphereShape(0.5f), RVec3(0_r, 2_r, 0_r), Quat::sIdentity(),
                         EMotionType::Dynamic, Layers::MOVING),
    EActivation::Activate);        // DontActivate로 넣으면 절대 시뮬레이션 안 함

// 6) 대량 추가 후 1회 (프레임마다 호출 금지 — 비쌈)
physics_system.OptimizeBroadPhase();

// 7) 고정 스텝 루프
const float dt = 1.0f / 60.0f;
// accumulator로 실제 경과시간을 쪼개서 Update 호출 (가변 dt 금지)
physics_system.Update(dt, /*collisionSteps*/ 1, &temp_allocator, &job_system);

// 8) 종료: RemoveBody/DestroyBody 개별 호출 → UnregisterTypes(); delete Factory::sInstance
```

## Top 함정 (실수 방지 — 전체 목록은 tuning-troubleshooting.md)

1. **`EActivation::DontActivate`로 추가한 바디는 영원히 안 움직인다.** 활성화하려면 `bi.ActivateBody(id)`. (`Activate()`라는 메서드는 BodyInterface에 없다 — `ActivateBody`/`ActivateBodies`.)
2. **ContactListener 콜백은 바디가 락 걸린 채 잡 스레드에서 호출된다.** 콜백 안에서 `GetBodyInterface()`(띠킹)를 부르면 데드락 — `GetBodyInterfaceNoLock()`을 쓰고, 리스너는 thread-safe여야 한다.
3. **`VehicleConstraint`는 `physics_system.AddConstraint()`에 더해 `AddStepListener(&vehicle_constraint)`까지 해야 한다.** 까먹으면 차가 안 굴러간다.
4. **`Update` 리턴값 확인** — `EPhysicsUpdateError` 비트 플래그. maxBodyPairs/maxContactConstraints 초과로 바디가 세계를 뚫고 떨어지는 게 이것 때문인 경우가 많다.
5. **빠른 물체 터널링** → `mMotionQuality = EMotionQuality::LinearCast` (기본은 Discrete). 단 센서는 LinearCast 적용이 안 돼 트리거를 건너뛸 수 있음(#1142). 총알류는 시뮬레이션 말고 레이캐스트 권장.
6. **scale이 최다 원인.** 0.1 m 미만 물체는 기본 `mPenetrationSlop`(0.02 m = 2 cm)보다 얇어서 뚫린다. 세계를 키우거나 slop을 낮춰라. 중력 196, 오브젝트 0.2 단위 같은 세팅은 "지원되지 않는 스케일"(#1740).
7. **슬리핑**: 바디 제거는 이웃을 깨우지 않는다(의도된 것) — `ActivateBodiesInAABox`로 직접 깨울 것. `Body::SetLinearVelocity`는 깨우지 않지만 `BodyInterface::SetLinearVelocity`는 깨운다. 잠든 바디의 센서 접촉은 삭제됨(OnContactRemoved 발생) — 엔터/익시트 추적은 직접 맵 관리(#1795).
8. **`PhysicsSystem`에는 `SaveBinaryState`가 없다.** 세계 저장은 `SaveState(StateRecorder&)`, 설정류 저장은 `ConstraintSettings::SaveBinaryState` 등 각 클래스에. 바이너리 포맷은 버전 간 호환 없음 — 업그레이드 시 리쿡.
9. **Mesh/HeightField 셰이프는 `MustBeStatic()`** — 다이나믹으로 못 쓰고, 질량속성도 정의 안 되니 다이나믹이 필요하면 convex decomposition.
10. **`OnContactValidate`는 가장 비싼 필터 지점.** 충돌 거부는 레이어/그룹에서 최대한 일찍. 필터 순서: BroadPhaseLayer → ObjectLayerPair → GroupFilter → (쿼리) BodyFilter → ShapeFilter → ContactListener.

## 쿼리 한눈에

```cpp
// 모든 쿼리는 NarrowPhaseQuery (정밀) 또는 GetBroadPhaseQuery() (대략적)
const NarrowPhaseQuery &q = physics_system.GetNarrowPhaseQuery();
RRayCast ray; ray.mOrigin = RVec3(0, 10, 0); ray.mDirection = Vec3(0, -20, 0);
RayCastResult hit;
if (q.CastRay(ray, hit))   // bool. 정밀 제어는 RayCastSettings + RayCastCollector 오버로드
    Vec3 normal = ... ;    // body.GetWorldSpaceSurfaceNormal(hit.mSubShapeID2, ray.mOrigin + ... )
// CastShape / CollideShape / CollidePoint / CastRay 모두 동일한 4-필터 체인:
// BroadPhaseLayerFilter, ObjectLayerFilter, BodyFilter, ShapeFilter
```

## 디버깅 워크플로

- 재현은 Samples 앱 스냅샷(`SamplesApp::TakeSnapshot`) + 정밀도/버전/커밋 해시 — jrouwe가 가장 빨리 받아주는 형식.
- `USE_ASSERTS=ON`(`JPH_ENABLE_ASSERTS`) 빌드로 돌려보라 — 정규화 안 된 quat 같은 입력을 잡아준다(#1860).
- 지터 원인 찾기: ContactListener로 "뭐가 충돌 중인지" 부터 확인. position step을 올리는 건 거의 답이 아니다(#265).
- 렌더: `JPH_DEBUG_RENDERER` + `DebugRenderer` 구현(`DrawLine/DrawTriangle`), 프레임마다 `NextFrame()` 호출 필수.
- 검색 순서: GitHub Issues/Discussions (jrouwe가 거의 모든 질문에 답함) → Docs/Architecture.md → Samples 소스. Stack Overflow 태그는 거의 비어 있다.

## 업그레이드

`Docs/APIChanges.md`(릴리스별 브레이킹 체인지)와 `Docs/ReleaseNotes.md`를 필독. 주요 최근 변경: 5.0 소프트바디 대개편, 5.1 `CharacterVsCharacterCollision`, 5.3 `SimShapeFilter`/커스텀 컨테이너, 5.4 `SetIsSensor` API/코서트 로드, 5.6 새 마찰 모델 + GPU compute/헤어.
