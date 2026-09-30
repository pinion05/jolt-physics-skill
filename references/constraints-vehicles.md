# 컨스트레인트, 모터, 차량, 래그돌, 소프트바디

## 공통 패턴 (TwoBodyConstraint)

```cpp
HingeConstraintSettings settings;
settings.mPoint1 = ...; settings.mPoint2 = ...;        // EConstraintSpace 따라 로컬/월드
// ... 축 등 설정 ...
TwoBodyConstraint *constraint = settings.Create(body1, body2);   // body는 Body& (BodyLockWrite로 획득)
physics_system.AddConstraint(constraint);
// 제거: physics_system.RemoveConstraint(constraint)
// 온오프: constraint->SetEnabled(bool)
```

종류: `FixedConstraint`, `PointConstraint`, `DistanceConstraint`(스프링), `HingeConstraint`, `SliderConstraint`, `ConeConstraint`, `SwingTwistConstraint`(어깨 등), `SixDOFConstraint`, `PathConstraint`(스플라인), `PulleyConstraint`, `GearConstraint`, `RackAndPinionConstraint`.

`ConstraintSettings` 공통 필드: `mEnabled`, `mDrawConstraintSize`, `mNumVelocityStepsOverride`/`mNumPositionStepsOverride`(0=PhysicsSettings 값), `mConstraintPriority`(높을수록 우선 — 고유값 주면 결정론적 순서), `mUserData`.

## HingeConstraintSettings 대표 필드

- `mSpace`: `EConstraintSpace::WorldSpace`(기본, 생성 시 월드 좌표로 지정) / `LocalSpace`. 다른 모든 좌표는 이 공간 기준
- `mPoint1`/`mPoint2`, `mHingeAxis1`/`mHingeAxis2`(기본 `Vec3::sAxisY()`), `mNormalAxis1`/`mNormalAxis2`(기본 X, 정렬 시 각도 0)
- 축 ⟂ 노멀축 필수
- `mLimitsMin`/`mLimitsMax`(라디안, 힌지 기본 [-pi, 0]), `mLimitsSpringSettings`(소프트 리밋 — 별도 enable 플래그 없음, 스프링 설정 넣으면 소프트)
- `mMaxFrictionTorque`, `mMotorSettings`

런타임: `SetMotorState(EMotorState::Off/Velocity/Position/PositionAndVelocity)`, `SetTargetAngularVelocity`(rad/s), `SetTargetAngle`(rad), `GetCurrentAngle()`, `SetMaxFrictionTorque()`.

## SixDOFConstraint 특수 규칙 (외우기 쉽지 않음)

`mLimitMin`/`mLimitMax`는 `float[EAxis::Num]` (Translation X/Y/Z = 미터, Rotation X/Y/Z = 라디안):

- **자유축**: min `-FLT_MAX`, max `+FLT_MAX`
- **고정축**: min `+FLT_MAX`, max `-FLT_MAX` (min > max → 0으로 강제)
- 헬퍼: `MakeFreeAxis(axis)`, `MakeFixedAxis(axis)`, `SetLimitedAxis(axis, min, max)`, `IsFreeAxis/IsFixedAxis`
- `mAxisX1/2`, `mAxisY1/2`로 프레임 지정(Z = X×Y)
- `mMaxFriction[EAxis::Num]`, `mMotorSettings[EAxis::Num]`
- `mLimitsSpringSettings[EAxis::NumTranslation]` — 회전축 소프트 리밋 미지원

## MotorSettings / SpringSettings

- `MotorSettings` 기본: `mSpringSettings = {ESpringMode::FrequencyAndDamping, frequency 2.0, damping 1.0}` — 스프링은 **position 모터에만** 작동(velocity 모터엔 미사용). `mMinForceLimit`/`mMaxForceLimit`(선형, N), `mMinTorqueLimit`/`mMaxTorqueLimit`(회전, N·m).
- 주파수 범위 [0, 시뮬레이션 주파수의 절반]. "단단한 스프링엔 20이 좋은 값"(60Hz 기준, #265).
- 모터 켜기: 각 컨스트레인트의 `SetMotorState(EMotorState)`.

## 차량 (VehicleConstraint)

```cpp
WheeledVehicleControllerSettings controller;
WheelSettingsWV *wheel = new WheelSettingsWV();
wheel->mPosition = ...;                    // 차 로컬 (Y-up)
wheel->mSuspensionMinLength/MaxLength = ...;
wheel->mRadius = 0.3f; wheel->mWidth = 0.1f;
wheel->mMaxSteerAngle = ...;
wheel->mInertia = 0.01f;                   // 바퀴가 무겁게 느껴지면 낮춰라
controller.mWheels.push_back(wheel);
VehicleConstraintSettings vehicle;
vehicle.SetController(&controller);
vehicle.mAntiRollBars.push_back(...);      // VehicleAntiRollBar(1, 2, 0.5f) — 앞뒤 휠 쌍
VehicleConstraint *constraint = new VehicleConstraint(car_body, vehicle);
physics_system.AddConstraint(constraint);
physics_system.AddStepListener(constraint);   // ★ 이것 없으면 차량 안 돌아감 ★
```

- 구동: `static_cast<WheeledVehicleController*>(constraint->GetController())` → `SetDriverInput(forward [-1,1], right [-1,1], brake [0,1], handBrake [0,1])` — 매 프레임 Update 전.
- 엔진/변속: `GetEngine()`, `GetTransmission()` (수동변속 `mMode`, 차동 `mDifferentials`).
- 휠 접지 검사: `VehicleCollisionTesterRay`(저렴, 기본) / `VehicleCollisionTesterCastCylinder`(훨씬 좋음, #472 "car demo를 크게 개선") / `VehicleCollisionTesterCastShape`. `SetVehicleCollisionTester`.
- 콜백: `SetPreStepCallback`, `SetPostCollideCallback`, `SetPostStepCallback` (부스터/점프 등 커스텀 힘).
- `SetMaxPitchRollAngle`, `OverrideGravity`, `GetAntiRollBars`, LOD: `SetNumStepsBetweenCollisionTestActive/Inactive`.
- 휠 마찰: `SetCombineFriction` 함수 교체, 휠별 마찰은 5.0+ `WheelSettings::mFriction`… 대신 `VehicleConstraint::SetCombineFriction`으로 per-wheel 조합 가능.
- 오토 변속기 진동(고토크 시)은 과거 버그 — 최신 버전 사용(#297/#300 수정됨).
- 트랙: `TrackedVehicleController`, 오토바이: `MotorcycleController`(리닝은 내장 아님 — 외부 PID 힘 제어 권장, #455).
- 라바 위 주행 시 차가 흔들리면: `PhysicsSettings::mNumVelocitySteps` 상향, 질량비 축소, 라바-휠 충돌 필터링.

## 래그돌

구성: `Skeleton`(관절 계층) + `RagdollSettings`(파트 배열) → `Ragdoll`(런타임).

- `Skeleton::AddJoint(name, parentName)` — **부모가 항상 자식보다 앞** 순서(`AreJointsCorrectlyOrdered()`), `CalculateParentJointIndices()`.
- `RagdollSettings::Part` = 관절 1개 = 강체 1 + 부모 연결 `TwoBodyConstraint`(루트 제외). `mAdditionalConstraints`로 비부모-자식 관절 추가.
- 래그돌 생성: `Ragdoll *ragdoll = ragdoll_settings->CreateRagdoll(groupID, userData, &physics_system); ragdoll->AddToPhysicsSystem(EActivation::Activate);` / `RemoveFromPhysicsSystem()`.
- 포즈: `SetPose(SkeletonPose&)`, `GetPose`; 구동: `DriveToPoseUsingKinematics(pose, dt)` 또는 `DriveToPoseUsingMotors(pose)`(모터 파워는 Part의 모터 설정).
- 텔레포트 후 `ResetWarmStart()`.
- 접촉 판정: `GetBodyID(i)`/`GetBodyIDs()`로 BodyInterface/쿼리 사용.
- 직렬화: `RagdollSettings::SaveBinaryState(out, saveShapes, saveGroupFilter)` / `sRestoreFromBinaryState`.
- `SkeletonMapper`로 애니메이션 스켈레톤 ↔ 래그돌 스켈레톤 리타게팅 (Rig 샘플).

### jrouwe의 래그돌 베스트 프랙티스 (discussion #554, 요약)

1. 팔다리 강체를 **겹치게** 만들어 사이 틈 제거
2. 부모-자식 충돌 비활성(`DisableParentChildCollisions`) + 정지 상태 내부 충돌 없게
3. 애니메이션이 관절 리밋을 넘지 않게 (모터 vs 리밋 싸움 = 불안정)
4. **질량이 루트에서 말단으로 갈수록 감소** (`Stabilize()`로 보정)

## 소프트바디 (4.0+, "개발 중 + 주의사항 있음")

```cpp
SoftBodySharedSettings shared;
// mVertices(Face::mMaterialIndex → mMaterials), mEdgeConstraints, mDihedralBendConstraints,
// mVolumeConstraints(사면체), mSkinnedConstraints, mLRAConstraints, Cosserat rod(헤어/로드)
shared.CreateConstraints(vertex_attributes, EBendType::None|Distance|Dihedral, angleTolerance);
shared.CalculateEdgeLengths(); shared.CalculateVolumeConstraintVolumes();
shared.CalculateLRALengths(); shared.CalculateBendConstraintConstants();
shared.Optimize();                     // 병렬 실행을 위한 재배열 — 성능에 필수
SoftBodyCreationSettings create(&shared, pos, rot, object_layer);
create.mNumIterations = 5; create.mPressure = 0; create.mVertexRadius = 0; // ...
BodyID id = bi.CreateAndAddSoftBody(create, EActivation::Activate);
```

- 주요 생성 필드: `mNumIterations`(5), `mLinearDamping`(0.1), `mPressure`(부피 유지/튜브), `mVertexRadius`(입자 반경, 표면 분리), `mUpdatePosition`(정적 세계 부착 시 false), `mMakeRotationIdentity`(true).
- 제약: **소프트-소프트/자가 충돌 없음**(#789). `SoftBodyContactListener`로 접촉 커스터마이즈.
- 스케일: 10K 정점 천 + 박스 121개 ≈ 2ms 멀티스레드(jrouwe 실측).
- 입자 상태: `SoftBodyMotionProperties::GetVertex(i)` — 렌더링용 버텍스 위치 갱신.

## 직렬화 3종 (서로 다른 용도)

1. **ObjectStream** (RTTI 텍스트/바이너리, `JPH_OBJECT_STREAM`) — `JPH_DECLARE_SERIALIZABLE_*` 매크로. Factory 필요.
2. **Settings 클래스 바이너리**: `ConstraintSettings::SaveBinaryState(StreamOut&)` + `sRestoreFromBinaryState(StreamIn&)` (Result 반환). `RagdollSettings`/`Skeleton`/`MotorSettings`/`Shape::SaveWithChildren`(자식+머티리얼 중복 제거)/`SoftBodyCreationSettings::SaveWithChildren` 동일 패턴. **버전 간 호환 없음 — 업그레이드 시 리쿡 필수**(#1709). Restore 전 `RegisterTypes()` 필요.
3. **리플레이/네트워크**: `PhysicsSystem::SaveState(StateRecorder&, EStateRecorderState, StateRecorderFilter*)` / `RestoreState` — `StateRecorderImpl` 구현체 제공. 5.2+ 부분 복원.

캐릭터 상태는 자동 직렬화 안 됨(#1655) — 직접 구현.
