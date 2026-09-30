# 쿼리(레이/셰이프 캐스트) & 접촉(ContactListener, 센서)

## 쿼리 진입점

- `physics_system.GetNarrowPhaseQuery()` → `NarrowPhaseQuery` — 정밀 쿼리 (기본)
- `physics_system.GetBroadPhaseQuery()` — 대략적(AABB 수준), 빠름

### 모든 쿼리 공통: 4-필터 체인

```cpp
DefaultBroadPhaseLayerFilter bp_filter;      // 또는 커스텀 ShouldCollide(layer)
DefaultObjectLayerFilter obj_filter;
DefaultBodyFilter body_filter;               // BodyID/바디 단위 제외
DefaultShapeFilter shape_filter;             // SubShapeID 단위 제외
```

### 레이캐스트

```cpp
RRayCast ray;                       // mOrigin(RVec3), mDirection(Vec3, 길이=최대 거리)
RayCastResult hit;
if (query.CastRay(ray, hit, bp_filter, obj_filter, body_filter, shape_filter)) {
    BodyLockRead lock(..., hit.mBodyID);  // hit.mFraction, hit.mSubShapeID2, hit.mShape2
    Vec3 normal = lock.GetBody().GetWorldSpaceSurfaceNormal(hit.mSubShapeID2, ray.mOrigin + ray.mDirection * hit.mFraction);
}

// 정밀 제어版
RayCastSettings rs;   // mBackFaceModeTriangles / mBackFaceModeConvex (별개 필드! 기본 IgnoreBackFaces)
                      // mTreatConvexAsSolid = true (내부에서 시작 시 fraction 0 히트)
ClosestHitCollisionCollector<RayCastCollector> collector;
query.CastRay(ray, rs, collector, RVec3Arg inBaseOffset /*정밀도용 원점*/, filters...);
```

### 셰이프캐스트 / 충돌 검사 / 포인트

```cpp
// 셰이프 캐스트 (CCD 직접 구현, 총알 등)
RShapeCast shape_cast = RShapeCast::sFromWorldTransform(shape, scale, center_of_mass_transform, displacement);
query.CastShape(shape_cast, ShapeCastSettings(), inBaseOffset, collector, filters...);

// 오버랩/정적 검사
query.CollideShape(shape, scale, com_transform, CollideShapeSettings(), inBaseOffset, collector, filters...);
// 내부 에지(고스트 충돌) 제거版
query.CollideShapeWithInternalEdgeRemoval(...);   // InternalEdgeRemovingCollector

query.CollidePoint(RVec3, CollidePointCollector, filters...);
query.CollectTransformedShapes(AABox, TransformedShapeCollector, filters...);
```

콜렉터: `AnyHitCollisionCollector<C>`(첫 히트), `ClosestHitCollisionCollector<C>`, `AllHitCollisionCollector<C>`(전부).

`CollideShapeSettings` 요점: `mBackFaceMode`, `mActiveEdgeMovementDirection`(움직임 방향 반대 노멀 무시 — 메시 에지 걸림 방지, #630), `mMaxSeparationDistance`, `mCollectFacesMode`.

원점에서 멀리 떨어진 세계에서는 `inBaseOffset`(로컬 원점)을 넘겨 정밀도 유지 — 더블 프리시전 필수 코스.

## ContactListener (물리 시뮬 중 접촉 이벤트)

`physics_system.SetContactListener(&listener)` — 리스너는 1개, **잡 스레드에서 다중 호출, thread-safe 필수, 콜백 중 바디 락 이미 걸림 → `GetBodyInterfaceNoLock()`만 사용(데드락 방지)**. 쌍별 직렬화는 보장.

```cpp
class MyListener : public ContactListener {
    // 1) 가장 이른/비싼 검증 지점. 접촉이 만들어지기 전. ValidateResult::RejectAllContacts 등 반환
    ValidateResult OnContactValidate(const Body& b1, const Body& b2, RVec3Arg baseOffset,
                                     const CollideShapeResult& colResult) override;

    // 2) 접촉 추가/지속 — solver 실행 전 (충돌 전 속도. 충격량 추정에 적합)
    void OnContactAdded(const Body& b1, const Body& b2, const ContactManifold& manifold,
                        ContactSettings& settings) override;
    void OnContactPersisted(...) override;    // 컴파운드에서 SubShapeID 재사용 시 위 added가 안 뜰 수 있음

    // 3) 접촉 제거 — 바디가 이미 사라졌을 수 있음. 여기서 heavy work 금지(스텝 지연)
    void OnContactRemoved(const SubShapeIDPair& pair) override;
};
```

- 섬이 잠들면 `OnContactRemoved`가 발생함 — "여전히 닿아 있는가" 판단에 부적합. Update 후 `physics_system.WereBodiesInContact(id1, id2)` 또는 직접 맵 관리(#1795 참조 구현: sleepingContacts 맵 + 바디 제거/텔레포트 시 직접 정리).
- 충격량 추정: `b1.EstimateCollisionResponse(...)` 계열.

### ContactSettings (OnContactAdded에서 수정)

- `mCombinedFriction` / `mCombinedRestitution`
- `mInvMassScale1/2`, `mInvInertiaScale1/2` — 기본 1. **0 = 무한 질량, 2 = 질량 절반**. 시간에 걸쳐 안정적이어야 함
- `mIsSensor` — 접촉을 반응 없이 센서화
- `mRelativeLinearSurfaceVelocity` / `mRelativeAngularSurfaceVelocity` — **컨베이어 벨트** 구현

## 센서(트리거)

```cpp
BodyCreationSettings bcs(shape, pos, rot, EMotionType::Static, Layers::SENSOR);
bcs.mIsSensor = true;                     // 5.4+: BodyInterface::SetIsSensor 로 런타임 전환도 가능
```

- 센서는 **활성(active) 상태의 바디와의 접촉만 감지** — 잠든 바디의 접촉은 사라짐.
- 루트 바디 쌍 단위 추적: `OnContactAdded`(Enter) / `OnContactRemoved`(Exit) — SubShapeIDPair 기반 키 관리.
- 센서 쌍이 브로드페이스에서 재활성화되면 added가 재발행될 수 있으니 멱등하게.
- **주의: LinearCast(CCD) 바디는 센서 상대로 Discrete로 취급** — 고속 CCD 바디가 트리거를 건너뛸 수 있음(#1142). 완화책: 센서를 크게, 또는 OnContactValidate에서 거르거나 쿼리 기반 판정.
- 서브셰이프 단위 접촉이 필요하면 `mUseManifoldReduction = false`(#1395).
- 쿼리만으로 "닿음" 판정이 싸다면 `SimCollideBodyVsBody`(5.3+)도 고려.

## 센서 Enter/Exit 추적 기본형

```cpp
// thread-safe 자료구조 사용. mKey = SubShapeIDPair, 값은 충돌한 BodyID 쌍
void OnContactAdded(...) override {
    // pair 기준으로 map에 insert → 이번 프레임에 새로 들어온 것 표시
}
void OnContactRemoved(const SubShapeIDPair& inPair) override {
    // map에서 erase → Exit 이벤트
}
// 물리 스레드와 게임 스레드 사이 더블버퍼/락으로 전달
```
