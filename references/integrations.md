# 엔진/언어 통합 & 생태계 (2026-09 기준)

## 릴리스 타임라인 (요점만)

- **5.6.0** (2026-07): GPU 컴퓨트 인터페이스(DX12/Vulkan/Metal), GPU 스트랜드 헤어(Cosserat rod), 새 마찰 모델(~40% perf↑, 메모리 70%↓), RISC-V RVV, 결정론 수정(ARM64↔x64 더블, CharacterVirtual 계단)
- **5.5.0** (2025-12): `JPH_TRACK_SIMULATION_STATS`, `RagdollSettings::CalculateConstraintPriorities`, VS2026
- **5.4.0** (2025-09): Cosserat 로드, `Ragdoll::DriveToPoseUsingMotors`, `BodyInterface::SetIsSensor`
- **5.3.0** (2025-03): 커스텀 컨테이너(UnorderedMap/Set), `SimShapeFilter`/`SimCollideBodyVsBody`, 1.1억 삼각형 메시, Vulkan/Metal 샘플
- **5.2.0** (2024-11): PlaneShape, TaperedCylinderShape, EmptyShape, 삼각형 userData
- **5.1.0** (2024-08): `CharacterVsCharacterCollision`, 차량 중력 오버라이드, WASM 결정론
- **5.0.0** (2024-04): 소프트바디 대개편(스킨/디헤드럴/LRA), SoftBodyContactListener, 휠별 마찰
- **4.0.0** (2023-11): 소프트바디 도입

업그레이드 시 `Docs/APIChanges.md` 필독.

## 채용 (Docs/ProjectsUsingJolt.md)

- AAA: **Horizon Forbidden West**, **Death Stranding 2**, War Thunder(Dagor), X4 Foundations, Blood Line: A Rebel Moon Game
- 엔진: **Godot(4.4+ 기본 3D 물리)**, ezEngine, Wicked, NeoAxis, Dagor, Nazara, LÖVR, GDevelop, luxe
- 툴: VPhysics Jolt(Source/GMod 대체), Qt Quick 3D Jolt Physics, Bonjolt(Maya), Autodesk Synthesis, Geekbench 7
- 플랫폼: HypeHype, Sceneri, The Mirror, Substrata

## 바인딩/통합 표

| 대상 | 것 | 상태 |
|---|---|---|
| **Godot** | 엔진 내장 모듈 (4.4+, 신규 프로젝트 기본). Project Settings → Physics 3D → "Jolt Physics" | 프로덕션. godot-jolt 확장(github.com/godot-jolt/godot-jolt)은 메인터넌스 모드(모듈이 후속) |
| **Unreal** | Yadhu-S/UnrealJolt (2025+, BP API, 결정론/롤백 목적) · jkurzer/Barrage · Artillery Eco(상용) | 전부 초창기. Chaos 완전 대체 아님. ※ TechartApe/JoltPhysicsForUnreal은 삭제됨 — 참조 금지 |
| **Unity** | seep/JoltPhysicsUnity (v5.3, 실험) · MacSpain/JoltPhysicsSharpUnity | 실험적. 성공 사례는 있으나 바인딩 불완전 |
| **C#/.NET** | amerkoleci/JoltPhysicsSharp (NuGet, joltc 기반) | 가장 성숙한 비-C++ 옵션. 활발 |
| **C API** | amerkoleci/joltc — C#/Rust/Zig 바인딩의 공통 기반 | 건강 |
| **Rust** | SecondHalfGames/jolt-rust (joltc-sys + rolt 안전 래퍼) · cohaereo/jolt-rs | 초기. `jolt` crate은 a16z zkVM(관계 없음!) |
| **Web/WASM** | jrouwe/JoltPhysics.js (공식, npm `jolt-physics`) — 7가지 빌드(SIMD/멀티스레드/asm.js 포함) | 성숙. ammo.js의 후계. react-three-jolt(pmndrs), Babylon.js 플러그인, GDevelop 사용 |
| **Java** | jolt-jni(stephengold) · JoltPhysics-JavaFFM | 소규모 |
| **Python** | Culverin | 초기 |
| **Zig** | zig-gamedev/zphysics | joltc 기반 |
| **Source 엔진** | Joshua-Ashton/VPhysics-Jolt (Facepunch/GMod 채용) | 프로덕션 |
| **Qt** | Qt Quick 3D Jolt Physics | 상용 |

## JoltPhysics.js (WASM) 필수 지식

- API는 C++ 미러 — 이 스킬의 C++ 지식이 그대로 적용.
- **GC 없음**: `Jolt.destroy(obj)`로 해제, RefCount는 `AddRef()/Release()`.
- 모든 `new Jolt.Vec3(...)`는 브리지 비용 — 벡터 재사용, 루프 밖 할당.
- 초기화는 비동기: `await Jolt();` 후 팩토리 사용.
- 데모/예제: https://jrouwe.github.io/JoltPhysics.js/

## Godot에서 Jolt 쓸 때 차이점 (공식 문서 "Using Jolt Physics" 요약 — GDScript 작업 시 유용)

- 관절 소프트리밋 파라미터 미지원(Pin/Hinge/Slider/ConeTwist/6DOF에서 기본값 아닌 값 경고)
- 단일 바디 관절은 `node_b` 취급(Godot Physics는 `node_a` 가정) — 월드 토글 방향 주의
- collision margin은 **늘리는 게 아니라 줄이고 라운드** (Collision Margin Fraction × 최소 AABB 축)
- Baumgarte는 position-only(`Baumgarte Stabilization Factor` 0-1, 1은 빠르지만 불안정)
- `face_index`는 기본 -1 (활성화 시 오목 모양 메모리 ~25% 증가)
- 키네마틱-정적 접촉은 opt-in(`Generate All Kinematic Contacts`)
- 접촉 임펄스는 추정치
- 설정 경로: `physics/jolt_physics_3d` (구 `physics/jolt_3d`에서 개명)
- 스레드 세이프, "Run On Separate Thread" 실험 지원
- 확장(godot-jolt) 이슈가 아니라 엔진 이슈로 리포트해야 함

## 학습 자료 (공식 외)

- GDC 2022 "Architecting Jolt Physics for 'Horizon Forbidden West'" — https://gdcvault.com/play/1027891/ (잡 그래프 멀티스레딩, 병렬 유니온-파인드 아일랜드 빌딩)
- Jan Wedekind 실습 가이드 (2024) — https://www.wedesoft.de/simulation/2024/09/26/jolt-physics-engine/ (레이어/차량/캐릭터 코드)
- wirepair CharacterVirtual 도입기 — https://wirepair.org/2024/05/24/the-journey-to-get-jolts-character-class-to-not-fall-to-its-doom/
- jrouwe.nl/articles.php — BVH 레이캐스트 비교 등 논문
- GitHub Discussions가 사실상의 FAQ — jrouwe가 거의 모든 질문에 답변. 검색: https://github.com/jrouwe/JoltPhysics/discussions
- r/gamedev 릴리스 스레드 — https://www.reddit.com/r/gamedev/comments/qr0oj6/
