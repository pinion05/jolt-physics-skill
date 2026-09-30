# 튜닝 & 트러블슈팅 — PhysicsSettings, 증상별 처방, 결정론, 직렬화

## PhysicsSettings 치트시트 (기본값 + 언제 손대나)

`physics_system.GetPhysicsSettings()` / `SetPhysicsSettings`.

| 설정 | 기본 | 노트 |
|---|---|---|
| `mNumVelocitySteps` | 10 | ≥2 필수(마찰 깨짐 방지). 라바 위 차량·정밀 기어 늘릴 때 상향 |
| `mNumPositionSteps` | 2 | **올리는 건 거의 답이 아니다** — position solve가 비싸면 뭔가 이상한 게 충돌 중이라는 신호(#265). ContactListener로 원인 확인 |
| `mBaumgarte` | 0.2 | 위치 오류 수정 비율. 1.0 = 빠르지만 불안정 |
| `mMaxPenetrationDistance` | 0.2 m | position 반복당 최대 보정 |
| `mSpeculativeContactDistance` | 0.02 m | speculative 선행거리. 크면 고스트 충돌, 정밀 기계는 줄여라(#1199) |
| `mPenetrationSlop` | 0.02 m | 허용 침투. 소형 물체(≤2cm)는 이걸 줄여야(예: 0.002, #1155/#1936). 0은 LinearCast 진행 불가 — 금지 |
| `mLinearCastThreshold`/`mLinearCastMaxPenetration` | 0.75 / 0.25 | CCD 트리거 비율(내부 반경 대비) |
| `mMinVelocityForRestitution` | 1.0 m/s | **이 속도 미만은 반발 무시 — 느린 물체는 안 튕긴다**(놀라기 쉬운 정상 동작) |
| `mTimeBeforeSleep` / `mPointVelocitySleepThreshold` | 0.5 s / 0.03 m/s | 섬 슬립 판정 |
| `mUseBodyPairContactCache` | true | 접촉 캐시. 슬립 비활성 시 구체가 뭉개지며 크리핑하면 false(#1686) |
| `mUseManifoldReduction` | true | 매니폴드 병합. 서브셰이프 센서 접촉 필요시 false(#1395) |
| `mConstraintWarmStart` | true | |
| `mMaxInFlightBodyPairs` | 16384 | 메모리 vs 속도 |

바디별 오버라이드: `BodyCreationSettings::mNumVelocityStepsOverride` 등.

## 증상 → 처방 표

### 터널링 (빠른 물체가 벽 통과)
1. `mMotionQuality = EMotionQuality::LinearCast` (기본 Discrete는 관통). LinearCast는 셰이프를 캐스트 후 시간을 "훔쳐" 접촉 위치에 멈춤 — 살짝 느려 보일 수 있음.
2. 급회전하는 긴 물체는 여전히 회전 관통 가능(회전이 캐스트 후 적용).
3. `maxContactConstraints`/`maxBodyPairs` 초과 확인(Update 에러 플래그) — 초과분은 그냥 세계 밖으로 떨어짐.
4. 정적 지형이 너무 얇으면 두껍게 하거나 `mPenetrationSlop` 축소.
5. 총알/레이저류: 시뮬레이션 하지 말고 **이전→현재 위치 레이캐스트** (커뮤니티 정설 + 빠름).
6. CCD 바디 vs 센서는 LinearCast 미적용(#1142) — 센서 확대 또는 쿼리 판정.
7. 혼합 주의: 이동 프레임(우주선) 위의 Discrete/LinearCast 혼용은 순서 문제로 플레이어가 함체에 밀릴 수 있음 — 둘 다 같은 quality로(#1524).

### 지터 / 쌓임 불안정
1. 기본: 60Hz + collision step 1. dt가 크면 collision steps 상향(1/60당 1개).
2. 반발 0, 마찰 적정(0.2~0.8). **restitution ≈ 1.0은 튕일 때마다 에너지가 증가**하는 solver 특성(#1286) — 정확한 물리가 아님.
3. 질량비 축소(극단적 무거움+가벼움 회피). 래그돌은 루트→말단 감소(#554).
4. 슬립 활성 유지(섬은 자야 안정). 슬립 끄고 구체가 크리핑하면 `mUseBodyPairContactCache = false`(#1686).
5. CylinderShape 회피 — "가장 불안정한 셰이프".
6. 메시 위 활공/튕김(고스트 충돌): 아래 "고스트 충돌" 절.
7. 스케일 확인 — 0.1m 미만 물체는 근본적으로 지원 밖(#1936/#1740).

### 고스트 충돌 (메시 에지에 걸림/튕김)
1. Active edges: `MeshShapeSettings::mActiveEdgeCosThresholdAngle`(기본 5°) — jrouwe: "50°까지도 아티팩트 없이 올릴 수 있다".
2. `BodyCreationSettings::mEnhancedInternalEdgeRemoval = true` — 단, **같은 바디 내부의 셰이프에만** 효과(하나의 MeshShape 또는 StaticCompoundShape 안). 서로 다른 바디 간 에지는 절대 용접 안 됨. 매니폴드 메시(비매니폴드/T-edge 금지, ~0.1mm 매칭 공차).
3. 쿼리에서는 `CollideShapeSettings::mActiveEdgeMovementDirection`로 이동 반대 노멀 무시.
4. speculative 접촉도 active edge에 걸릴 수 있음(#630).

### 슬리핑 관련
- 바디 제거는 이웃을 **안 깨움**(의도) → 제거 전 AABB 쿼리 + `ActivateBodiesInAABox`.
- `Body::SetLinearVelocity`는 안 깨움 / `BodyInterface::` 버전은 깨움.
- 잠든 섬의 접촉은 제거됨(OnContactRemoved 발생) — Enter/Exit 추적 직접 구현(#1795 참고).
- "래그돌 잠들었나" 확인: BodyActivationListener로 카운트 vs `GetBodyIDs()`(#521).

### 스케일/단위 문제 (최다 원인)
- SI 미터 기준 권장: 다이나믹 [0.1, 10]m, 정적 [0.1, 2000]m, 속도 ≤500 m/s, 중력 [0,10]m/s².
- cm/mm 단위 엔진(Godot cm 등)에서 옮겨온 씬은 스케일 변환 필수. "중력 196, 오브젝트 0.2" = 사실상 0.01m 오브젝트 → 미지원(#1740).
- 상한은 소프트(100m 오브젝트도 아마 동작), **하한은 빨리 깨짐**(#1389). 내부 허용치: `mPenetrationSlop`, `CollideSettingsBase::mCollisionTolerance` 등 일부만 노출.
- 원점에서 ~5km 이상 떨어지면 float 정밀도 저하 → `DOUBLE_PRECISION=ON`(5-10% 느림) + 쿼리에 `inBaseOffset` 활용. 10,000km 부터 브로드페이스 해상도 ~1m로 저하(성능만).

### 키네마틱
- 이동은 `MoveKinematic(id, pos, rot, dt)` (Update 직전) — 다이나믹이 올바른 속도를 받음. SetPosition 계열은 속도 정보 없음.
- 키네마틱/정적 쌍의 접촉은 기본 생성 안 됨(Godot-Jolt: opt-in).

### 원인 모를 이상 동작 공통 체크리스트
1. Update 리턴값(`EPhysicsUpdateError`) 확인.
2. `USE_ASSERTS=ON` 빌드로 재현 (#1860: 비정규화 quat가 대표적).
3. 단위/스케일 점검.
4. defines 불일치(더블/RTTI/디버그렌더러/NDEBUG) 점검 — 혼합 빌드는 묘하게 죽는다(#2139).
5. ContactListener로 실제 접촉 대상 확인.
6. 재현을 Samples 스냅샷으로 만들어 이슈 등록 — jrouwe 응답 빠름.

## 결정론

- 조건: (1) 시뮬레이션 수정 API가 **항상 같은 순서**로 호출, (2) **같은 바이너리**. 같은 OS에선 CPU 벤더 무관.
- 크로스 플랫폼(컴파일러/OS/아키텍처 x86·ARM·RISC-V·WASM): `CROSS_PLATFORM_DETERMINISTIC=ON` (+`JPH_CROSS_PLATFORM_DETERMINISTIC`), ~8% 느림, FMADD 비활성 + 정밀 FP 강제.
- **앱 쪽도 맞춰야 함**: `/fp:precise`·`-ffp-contract=off`, std::sin/cos/정렬/해시 대신 Jolt의 `Sin`/`Cos`/`QuickSort`/`Hash` 사용.
- **브로드페이스 쿼리는 비결정론적**(트리가 다중 스레드에서 수정됨) — 쿼리 순서에 의존하는 로직 금지. 리스너 콜백 순서도 비결정론적.
- 디버그: `JPH_ENABLE_DETERMINISM_LOG`. 스텝 롤백: `SaveState`/`RestoreState`(5.2+ 부분 지원).
- 슬립은 결정론적(같은 입력 → 같은 시점 슬립).

## 직렬화 주의사항

- 바이너리(`SaveBinaryState`류)는 **헤더/버전 검사 없음, 릴리스 간 호환 0** — 업그레이드할 때마다 리쿡(#1709). 정밀도 모드 혼용 금지.
- Restore 전 `Factory::sInstance` + `RegisterTypes()` 필수.
- HeightField 저장/복원 느림(#1080), 제한 DOF에서 NaN 저장 사례(#915) 등 과거 버그 — 최신 버전 확인.
- 캐릭터 상태는 미지원(#1655).

## 성능 튜닝 순서 (커뮤니티+jrouwe 종합)

1. 브로드페이스 레이어: 최소 2트리(정적/이동). 정적-정적 충돌 OFF.
2. 대량 추가는 배치 API + `OptimizeBroadPhase()` 1회.
3. 스레드 = 코어수-1 (`JobSystemThreadPool`).
4. `maxBodyPairs`/`maxContactConstraints` 넉넉히(초과 = 떨어짐 + 에러).
5. 컨벡스는 convexRadius 유지(0에 가깝게 하지 않기). ConvexHull 단순하게.
6. 정적 지형은 Mesh/HeightField 1개로(여러 바디 분할보다 내부 에지 제거도 됨).
7. 슬립 활성(섬이 자야 CPU 0).
8. `JPH_TRACK_SIMULATION_STATS`(5.5+)로 바디별 비용 추적, PerformanceTest로 회귀 측정.
9. LOD: 원거리 차량 `SetNumStepsBetweenCollisionTestInactive`, 센서는 `SimCollideBodyVsBody`.
10. 5.6.0의 새 마찰 모델 자체가 ~40% perf / 70% 메모리 개선 — 구버전이면 업그레이드.

## 참조 이슈 인덱스 (GitHub jrouwe/JoltPhysics)

터널링 #1142 #633 #753 · 지터/스택 #1286 #983 #1686 #265 · 스케일 #1936 #1740 #1389 · 슬립 #267 #1795 #2100 #521 · 고스트 #630 #717 #2005 · 키네마틱 #177 #847 · 캐릭터 #88 #211 #363 #1044 #1514 #1924 #2036 #2090 · 차량 #297 #300 #455 #472 · 소프트바디 #789 · 직렬화 #1709 #915 #1080 #1655 · 빌드 #2139 · dt 관련 discussions #959 #968 · 래그돌 discussion #554 · 기어 #1199.
