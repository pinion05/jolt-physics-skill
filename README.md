# jolt-physics — AI agent skill for Jolt Physics

A knowledge-pack skill for AI coding agents (ZCode, Claude Code, and anything that reads
the `SKILL.md` convention) covering **[Jolt Physics](https://github.com/jrouwe/JoltPhysics)** —
the C++ rigid body physics engine behind *Horizon Forbidden West*, *Death Stranding 2*,
and Godot 4.4+'s default 3D physics.

Researched on 2026-09-30 against Jolt **v5.6.0**: the official Doxygen docs (20+ pages
fetched and cross-checked), `HelloWorld.cpp` / `Build/README.md` sources, release notes,
and 30+ GitHub issues/discussions answered by Jolt's author (jrouwe).

No dedicated Jolt skill existed anywhere (SkillsMP, awesome-claude-skills,
awesome-cursorrules all checked) — this is the first.

## What it gives your agent

- **Verified API names** — the exact method/enum/field names that models tend to
  hallucinate (`ActivateBody` not `Activate`, `NotSupported` not `NotOnGround`,
  `PhysicsSystem` has no `SaveBinaryState`, …)
- **A copy-paste minimal simulation loop** distilled from the official HelloWorld
- **Layer/broadphase setup patterns** with the table-driven helper classes
- **A symptom→fix troubleshooting table** for tunneling, stack jitter, ghost collisions,
  sleeping, scale/unit problems — each backed by a specific GitHub issue number
- **PhysicsSettings cheat sheet** with defaults and when *not* to touch them
- **CharacterVirtual / vehicles / ragdolls / soft bodies** playbooks with the
  author-recommended settings
- **Build integration** — including the FetchContent gotcha (`SOURCE_SUBDIR "Build"`),
  CMake options, define mismatches, and link-error triage
- **Determinism rules** (same-binary vs `CROSS_PLATFORM_DETERMINISTIC`) and
  serialization pitfalls (no version compat between releases)
- **Ecosystem map** — Godot/Unreal/Unity/WASM/Rust/C# bindings with maturity notes

## Install

```bash
# ZCode / any agent reading ~/.agents/skills
git clone https://github.com/pinion05/jolt-physics-skill ~/.agents/skills/jolt-physics

# Claude Code style
git clone https://github.com/pinion05/jolt-physics-skill ~/.claude/skills/jolt-physics
```

Then just mention Jolt in a prompt — the description triggers automatically. A minimal
smoke test:

> "Jolt로 공이 바닥에 떨어져 튀는 최소 C++ 예제랑 CMakeLists 만들어줘"

## Structure

```
jolt-physics/
├── SKILL.md                             # routing + minimal loop + top-10 gotchas
└── references/
    ├── setup-build.md                   # CMake/FetchContent/defines/link errors
    ├── core-api.md                      # init, layers, bodies, shapes, forces
    ├── queries-contacts.md              # ray/shape casts, ContactListener, sensors
    ├── constraints-vehicles.md          # constraints, motors, vehicles, ragdolls, soft bodies
    ├── characters.md                    # CharacterVirtual deep dive
    ├── tuning-troubleshooting.md        # PhysicsSettings, symptom→fix, determinism
    └── integrations.md                  # engine/language bindings, releases
```

## Accuracy notes

- Facts carry their source (GitHub issue/discussion numbers, doc pages) inside the
  reference files, so claims stay auditable.
- Jolt's API changes between releases — check [`Docs/APIChanges.md`](https://github.com/jrouwe/JoltPhysics/blob/master/Docs/APIChanges.md)
  when targeting a version other than 5.6.0.
- The docs-site class-page URL pattern is baked into the skill
  (`https://jrouwe.github.io/JoltPhysics/class_<snake_case>.html`) so the agent can
  verify any symbol against the live docs.

## License

MIT — see [LICENSE](LICENSE). Jolt Physics itself is MIT, by Jorrit Rouwe;
this skill is an independent knowledge pack and not affiliated with the project.
