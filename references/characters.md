# 캐릭터 컨트롤러 — Character vs CharacterVirtual

## 선택

- **`CharacterVirtual`** — 플레이어용 권장. 강체 없이 충돌 검사만으로 이동("virtual"). PhysicsSystem에 등록되지 않아 다른 바디에게 보이지 않음(→ `mInnerBodyShape` 설정으로 해결 가능). 상태 저장/복원은 직접(#1655).
- **`Character`** — 실제 강체 1개(보통 캡슐) 기반 키 프레임 캐릭터. NPC/단순 용도.
- 대량 군중(수백+) — CharacterVirtual은 무겁다. jrouwe의 DIY 대안(#2090): 수평 이동 → 바닥 레이캐스트 → 블로킹 셰이프캐스트, 모양을 바닥에서 살짝 뜨게 유지.

## CharacterVirtual 기본 흐름

```cpp
CharacterVirtualSettings settings;
settings.mShape = new CapsuleShape(0.5f, 0.4f);   // 캡슐 권장 — "sliding이 훨씬 쉽다"(#88). 박스는 노멀에 민감
// ... 아래 필드표 ...
CharacterVirtual character(&settings, spawn_pos, Quat::sIdentity(), 0, &physics_system);
character.SetListener(&contact_listener);          // CharacterContactListener

// 프레임 루프 (Update 전):
character.UpdateGroundVelocity();                  // 바닥 속도 갱신(저렴, 충돌 검사 없음)
character.SetLinearVelocity(desired_velocity);     // 중력/이동을 직접 계산해 반영
// physics_system.Update(...)
// Update 후:
character.PostSimulation(max_separation_distance); // 접촉 갱신 — "every Update 후 호출 필요"
```

**편리 통합 API**: `character.ExtendedUpdate(dt, gravity, extended_settings, bp_filter, obj_filter, body_filter, shape_filter, temp_allocator, job_system)` — Update + StickToFloor + WalkStairs를 한 번에 (`ExtendedUpdateSettings`로 세부 조정). 이걸 쓰면 위의 Pre/Post 흐름 대신 Update 다음에 호출.

### 핵심: 중력은 자기 책임

`CharacterVirtual`은 강체가 아니므로 **중력이 자동 적용되지 않는다.** `Update(inDeltaTime, inGravity, ...)`의 `inGravity` 인자는 캐릭터 자체를 끌어내리는 게 아니라 **캐릭터가 서 있는 바디를 누르는 힘**이다. 캐릭터 자체 하강은 매 프레임 `SetLinearVelocity(v + gravity * dt)` 식으로 직접. 대신 임의 up 벡터/행성 중력도 자유롭게 구현 가능(#88, #2046).

## CharacterVirtualSettings 필드 (기본값)

| 필드 | 기본 | 비고 |
|---|---|---|
| `mMass` | 70 kg | 서 있는 바디를 누르는 질량 |
| `mMaxStrength` | 100 N | 밀어내는 힘 상한 |
| `mUp` | `Vec3::sAxisY()` | |
| `mSupportingVolume` | Plane | 이 부피 안의 접촉을 "지지"로 인정 |
| `mMaxSlopeAngle` | 50° | 이상은 미끄러짐 |
| `mPredictiveContactDistance` | 0.1 | |
| `mCharacterPadding` | 0.02 | 인공 컨벡스 반경 — 계단 파라미터보다 작게 유지 금지(아래) |
| `mPenetrationRecoverySpeed` | 1.0 | |
| `mMaxCollisionIterations` / `mMaxConstraintIterations` | 5 / 15 | |
| `mMaxNumHits` | 256 | |
| `mHitReductionCosMaxAngle` | 0.999 | |
| `mEnhancedInternalEdgeRemoval` | false | 메시 에지 걸림 방지(5.1+ 캐릭터 지원) |
| `mBackFaceMode` | CollideWithBackFaces | |
| `mInnerBodyShape`/`mInnerBodyLayer`/`mInnerBodyIDOverride` | | 다른 바디가 캐릭터를 감지하게 하는 내부 강체 |

계단 오르기(`CharacterSettings` 계열): `mWalkStairsMinStepForward`, `mWalkStairsStepForwardTest` — FPS 독립적 계단 처리용으로 만들어짐. **`mWalkStairsMinStepForward`를 패딩보다 크게** — 아니면 아래쪽 스윕이 수직 충돌을 놓침(#2036 경험). 슬로프 미끄럼 방지 헬퍼: `CancelVelocityTowardsSteepSlopes()`.

디버그 static 플래그: `CharacterVirtual::sDrawConstraints`, `sDrawWalkStairs`, `sDrawStickToFloor`, `sDrawSupportingVolume`.

## 지면 판정

- `GetGroundState()` → `EGroundState::OnGround / OnSteepGround / NotSupported / InAir` (**"NotOnGround"라는 값은 없다**)
- `IsSupported()`, `GetGroundBodyID()`(이미 제거됐을 수 있음 — 반드시 유효성 체크), `GetGroundPosition/Normal/Velocity/Material/UserData`
- 주의: `IsSupported()`는 가파른 경사에서도 true → 입력 게이팅에 쓰면 속도가 누적된다(#363). `OnGround`로 판정할 것.

## CharacterContactListener (CharacterVirtual.h 안에 있음)

```cpp
class MyListener : public CharacterContactListener {
    bool OnContactValidate(CharacterVirtual*, const Body&, RVec3Arg) override;   // 충돌 무시 시 false
    void OnContactAdded/Persisted(CharacterVirtual*, const Body&, ContactID, CharacterContact&, CharacterContactSettings&) override;
    void OnContactRemoved(CharacterVirtual*, const Body&, ContactID, SubShapeID) override;
    // 슬라이딩/바운스 커스터마이즈 — 가장 자주 쓰는 훅 (#211 이후 추가)
    void OnContactSolve(CharacterVirtual*, const Body&, ContactID, RVec3Arg contactPosition,
                        Vec3Arg contactNormal, Vec3Arg contactVelocity,
                        const PhysicsMaterial*, Vec3Arg characterVelocity,
                        Vec3 &ioNewCharacterVelocity) override;
    // 컨베이어 벨트 등 — 바디 쪽 속도 조정
    void OnAdjustBodyVelocity(CharacterVirtual*, const Body&, Vec3 &ioLinearVelocity, Vec3 &ioAngularVelocity) override;
    // 캐릭터끼리: OnCharacterContactValidate/Added/Persisted/Removed/Solve (CharacterVsCharacterCollision 설정 시, 5.1+)
};
```

## 알려진 이슈 클러스터 (해결된 것/열린 것)

- 둔각 코너 진동(slide 코드 경로) — #1044 논의
- 다이나믹 바디 위에서 지터 — #1924 (open)
- 사선 이동 플랫폼 위 지터 — #1514 (open)
- 경사진 계단 버그 — #2036 (수정 이력)
- 계단/막힘 관련 결정론 수정 — 5.6.0에서 다수 반영

이런 증상을 만나면 최신 버전인지 먼저 확인하고 Issues 검색(#88이 캐릭터 총론 스레드).

## Guerrilla(Horizon)의 실전 패턴 (#88)

플레이어 이동은 게임 업데이트 중 CollideShape/CastShape "평면" 시리즈로 처리하고,
다이나믹 오브젝트를 밀어내기 위해서만 강체 `Character`(중력 없이 SetLinearVelocity로 이동)를 별도로 운용.
