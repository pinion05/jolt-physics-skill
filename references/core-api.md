# 핵심 API — 초기화, 레이어, 바디, 셰이프, 업데이트 루프, 힘/속도

## 레이어 시스템 (물리 성능의 70%가 여기서 결정)

2단계 필터:

- **ObjectLayer** (16/32비트, `JPH_OBJECT_LAYER_BITS`) — 바디별 얇은 필터. 몇십 개 가능.
- **BroadPhaseLayer** — 브로드페이스 트리 버킷. **소수(2~4개) 유지**. 최소한 "정적/비이동"과 "이동" 트리 분리(이슈 #265에서 jrouwe의 1순위 진단 항목).

`PhysicsSystem::Init`에 넘기는 3개 인터페이스는 **참조로 저장되므로 PhysicsSystem보다 오래 살아야 한다.**

테이블 기반 헬퍼(직접 가상 클래스 구현할 필요 없음):

```cpp
// 1) 오브젝트 레이어 정의
namespace Layers {
    static constexpr ObjectLayer NON_MOVING = 0;
    static constexpr ObjectLayer MOVING = 1;
    static constexpr ObjectLayer NUM_LAYERS = 2;
}

// 2) BroadPhaseLayer 매핑
BPLayerInterfaceImpl : BroadPhaseLayerInterface     // 직접 구현하거나
BroadPhaseLayerInterfaceTable bp_layers(Layers::NUM_LAYERS, /*numBroadPhaseLayers*/ 2);
bp_layers.MapObjectToBroadPhaseLayer(Layers::NON_MOVING, BroadPhaseLayers::NON_MOVING);
bp_layers.MapObjectToBroadPhaseLayer(Layers::MOVING,    BroadPhaseLayers::MOVING);
bp_layers.SetBroadPhaseLayerName(...);  // 디버그 이름

// 3) ObjectLayer 쌍 필터 — 정적vs정적은 반드시 false (가장 큰 낭비 방지)
ObjectLayerPairFilterTable pair_filter(Layers::NUM_LAYERS);   // 기본 전부 충돌 OFF
pair_filter.EnableCollision(Layers::NON_MOVING, Layers::MOVING);
pair_filter.EnableCollision(Layers::MOVING, Layers::MOVING);
// EnableCollision은 대칭으로 양쪽에 적용됨. ShouldCollide(l1,l2)는 대칭이어야 함.

// 4) Object ↔ BroadPhase 필터
ObjectVsBroadPhaseLayerFilterTable obj_vs_bp(bp_layers, ...);
```

마스크 방식(Bullet 스타일 group&mask) 변형도 존재: `ObjectLayerPairFilterMask`, `BroadPhaseLayerInterfaceMask`, `ObjectVsBroadPhaseLayerFilterMask`.

**필터 파이프라인 순서(비싼 순)**: BroadPhaseLayerFilter → ObjectLayerPairFilter → GroupFilter(래그돌용 `GroupFilterTable`) → BodyFilter(쿼리 전용) → ShapeFilter/`SetSimShapeFilter` → `OnContactValidate`(최후, 가장 비쌈).

## PhysicsSystem::Init 파라미터

```cpp
physics_system.Init(inMaxBodies, inNumBodyMutexes, inMaxBodyPairs, inMaxContactConstraints,
                    broad_phase_layer_interface, obj_vs_bp_filter, obj_pair_filter);
```

- `inMaxBodies` ≤ `cMaxBodiesLimit`. 실전 값 ~65536.
- `inNumBodyMutexes`: 2의 거듭제곱 [1,64], **0 = 자동**.
- `inMaxBodyPairs`: 브로드페이스 쌍 수. 접촉 안 하는 가까운 쌍도 포함되므로 예상 접촉수보다 훨씬 크게 (65536 권장).
- `inMaxContactConstraints`: **초과하면 바디가 세계를 뚫고 떨어짐** (10240 권장).

주요 PhysicsSystem API: `SetGravity(Vec3)` / `SetPhysicsSettings` / `SetContactListener` / `SetBodyActivationListener` / `SetSoftBodyContactListener` / `SetCombineFriction`·`SetCombineRestitution`(기본: 마찰 sqrt(f1·f2) 기하평균, 반발 max) / `AddConstraint`·`RemoveConstraint`(배치 `AddConstraints`도) / `AddStepListener`·`RemoveStepListener` / `GetNumBodies` / `GetNumActiveBodies(EBodyType)` / `GetBodies(BodyIDVector&)` / `WereBodiesInContact(id1, id2)` / `OptimizeBroadPhase()` / `SaveState`·`RestoreState`.

## 바디 생성

```cpp
BodyInterface &bi = physics_system.GetBodyInterface();          // 띉킹 (콜백 안에서는 NoLock)

// 방법 A: Settings → Shape 생성 (캐시됨, 에러 확인 가능)
BoxShapeSettings box_settings(Vec3(100, 1, 100));
// box_settings.SetEmbedded();          // 스택에 Settings를 둘 때 (RefTarget 수명)
ShapeSettings::ShapeResult result = box_settings.Create();
if (result.HasError()) { /* result.GetError() */ }
BodyCreationSettings bcs(result.Get(), RVec3(0_r, -1_r, 0_r), Quat::sIdentity(),
                         EMotionType::Static, Layers::NON_MOVING);
BodyID id = bi.CreateAndAddBody(bcs, EActivation::DontActivate);

// 방법 B: Shape 직접 (간단한 경우)
BodyID id2 = bi.CreateAndAddBody(
    BodyCreationSettings(new SphereShape(0.5f), RVec3(0_r, 2_r, 0_r),
                         Quat::sIdentity(), EMotionType::Dynamic, Layers::MOVING),
    EActivation::Activate);

// 수명: CreateBody + AddBody(id, EActivation) / RemoveBody(id) / DestroyBody(id)
// 대량: AddBodiesPrepare + AddBodiesFinalize (또는 AddBodiesAbort), RemoveBodies(배열)
// 소프트바디: CreateSoftBody / CreateAndAddSoftBody
```

`EActivation` (Jolt/Physics/EActivation.h): `Activate` / `DontActivate`(상태 유지).

### BodyCreationSettings 주요 필드 (기본값)

| 필드 | 기본 | 비고 |
|---|---|---|
| `mMotionType` | — | `Static` / `Kinematic`(키 프레임: 속도로 이동, 힘 무시) / `Dynamic` |
| `mMotionQuality` | `Discrete` | **`LinearCast` = CCD**. 빠른 물체 필수 |
| `mObjectLayer` | — | 필터용 |
| `mFriction` | 0.2 | 조합은 sqrt(f1·f2) |
| `mRestitution` | 0.0 | 조합은 max. 1.0에 가까우면 에너지 증가(#1286) |
| `mGravityFactor` | 1.0 | 바디별 중력 배율 (0 = 무중력) |
| `mLinearDamping`/`mAngularDamping` | 0.05 | 지수 감쇠, 보통 0 근처 |
| `mMaxLinearVelocity` | 500 m/s | 고속 시뮬 필요시 상향(#228) |
| `mMaxAngularVelocity` | 0.25·π·60 rad/s | |
| `mIsSensor` | false | 센서(트리거): 콜백만, 반응 없음 |
| `mUseManifoldReduction` | true | 서브셰이프별 센서 접촉 필요시 false(#1395) |
| `mEnhancedInternalEdgeRemoval` | false | 고스트 충돌 제거. **같은 바디 내부 셰이프에만 효과** |
| `mAllowedDOFs` | All | 2D 시뮬레이션 등 축 제한 |
| `mOverrideMassProperties` | `CalculateMassAndInertia` | `CalculateInertia`(질량 유지), `MassAndInertiaProvided`(직접 지정) |
| `mMassPropertiesOverride` / `mInertiaMultiplier` | | 밀도는 ShapeSettings::mDensity 로도 |
| `mAllowSleeping` | true | |
| `mApplyGyroscopicForce` | false | true면 텀블링(드자니베코프) 효과 |
| `mLinearVelocity`/`mAngularVelocity` | | 초기 속도 |
| `mCollisionGroup` | | GroupFilter/GroupID — 래그돌 내부 충돌 제어 |
| `mNumVelocityStepsOverride`/`mNumPositionStepsOverride` | 0(글로벌 사용) | 바디별 solver 반복 |

`mPosition`은 COM이 아니라 **바디 원점** 기준이지만, 셰이프는 생성 시 COM 기준으로 재중심화된다(`GetCenterOfMass`). `OffsetCenterOfMassShape`/`RotatedTranslatedShape`로 COM/원점 조정.

## 셰이프

- 컨벡스: `SphereShape`, `BoxShape`(convexRadius 있음 — 클수록 빠르고 덜 정확), `CapsuleShape`(Y축), `TaperedCapsuleShape`, `CylinderShape`(**"가장 불안정한 셰이프"** — 피할 것), `TaperedCylinderShape`, `ConvexHullShape`, `TriangleShape`
- 컴파운드: `StaticCompoundShape`, `MutableCompoundShape`
- 데코레이터: `RotatedTranslatedShape`(재배치/재회전), `OffsetCenterOfMassShape`, `ScaledShape`
- 메시: `MeshShape`, `HeightFieldShape` — **`MustBeStatic()` true. 다이나믹 금지.** 5.3+ 1억 1천만 삼각형 지원. 삼각형별 userData 지원(5.2+)
- 기타: `PlaneShape`(5.2+), `EmptyShape`, `SoftBodyShape`
- 삼각형 순회: `GetTrianglesStart`/`GetTrianglesNext` (요청당 최소 32개, `cGetTrianglesMinTrianglesRequested`)
- 스케일: `IsValidScale`/`MakeScaleValid`/`ScaleShape` — 구/캡슐은 균일 스케일만. 비균일은 ScaledShapes 샘플 참고
- Settings 클래스는 `Create()` 결과 캐시 — 재생성 시 `ClearCachedResult()`

## 업데이트 루프

```cpp
// 고정 dt accumulator (가변 dt 금지 — 결정론/안정성 붕괴, #959/#968)
float accumulator = frame_time;
while (accumulator >= dt) {
    physics_system.Update(dt, collision_steps, &temp_allocator, &job_system);
    accumulator -= dt;
}
```

- `Update(inDeltaTime, inCollisionSteps, TempAllocator*, JobSystem*)` → `EPhysicsUpdateError` (비트 플래그, **항상 검사**).
- dt > 1/60이면 `inCollisionSteps = ceil(dt / (1/60))` — collision step은 1/60당 1개(반올림 up). 60Hz+1step이 안정 기준. 2 step이면 내부적으로 120Hz 충돌/적분.
- 리턴 시 모든 잡 종료. 콜백(리스너)은 잡 스레드에서 호출됨 — thread-safe 필수.
- `OptimizeBroadPhase()`: 대량 추가 직후 1회. Update는 이후 비용을 분산 처리. 프레임마다 호출 금지, 다른 스레드가 바디 수정 중일 때 호출 금지.

## 힘 / 임펄스 / 속도 (BodyInterface — 정확한 메서드명)

- 속도: `SetLinearVelocity` / `GetLinearVelocity` / `SetAngularVelocity` / `SetLinearAndAngularVelocity` / `AddLinearVelocity` / `AddLinearAndAngularVelocity` / `GetPointVelocity` (선속도는 **COM 기준**)
- 변환: `SetPosition` / `SetRotation` / `SetPositionAndRotation` / `SetPositionRotationAndVelocity` / `SetPositionAndRotationWhenChanged`
- 키네마틱: `MoveKinematic(id, targetPos, targetRot, dt)` — 목표 도달 속도 자동 계산, **Update 전에 호출**. 아직 Add 안 된 바디에 호출 금지(#847)
- 힘: `AddForce(Vec3)`/`AddForce(Vec3, RVec3 position)` / `AddTorque` / `AddForceAndTorque` — 다음 스텝에 적용. 스텝 콜백 중 호출 금지(락 걸림)
- 임펄스: `AddImpulse` / `AddAngularImpulse` (즉시 속도 변화). **`AddLinearAndAngularImpulse`는 존재하지 않는다** — 둘 따로
- 활성화: `ActivateBody`/`ActivateBodies`/`ActivateBodiesInAABox`/`DeactivateBody`/`IsActive`/`ResetSleepTimer`/`ActivateConstraint`
- 부력: `ApplyBuoyancyImpulse`
- 힘/임펄스는 `EActivation` 인자(기본 Activate), 속도 설정류는 자동 활성화
- 깨우기 규칙: `Body::SetLinearVelocity`(바디 직접)는 **안 깨움**, `BodyInterface::SetLinearVelocity`는 깨움

## Body 직접 접근 vs BodyInterface

- 단발 수정/조회: `BodyInterface`(내부 락 처리)가 간편.
- 읽기 다수: `BodyLockRead lock(physics_system.GetBodyLockInterface(), id); if (lock.Succeeded()) { const Body &b = lock.GetBody(); }` / 쓰기는 `BodyLockWrite`.
- 콜백 내부(이미 락 걸림): `GetBodyInterfaceNoLock()`.
