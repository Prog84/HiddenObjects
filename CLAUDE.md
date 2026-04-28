# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**Hidden Objects** — 2D hidden object finder game built in Unity 2022.3.62f3.

- Language: C#
- No custom game scripts yet — only third-party asset packs (Epic Toon FX, Freeui ZOSMA UI kit, DOTween)
- Main scene: `Assets/Scenes/SampleScene.unity`
- Game assets: 43 item sprites (`Assets/Sprites/Items/`), 26 level backgrounds (`Assets/Sprites/Level Background/`)

## Build & Run

This is a Unity project — no CLI build commands. Open in Unity 2022.3.62f3, open `SampleScene.unity`, press Play.

## Key Dependencies

- **DOTween 1.22.3** — animation tweening (use for all animations)
- **Addressables 1.22.3** — asset loading
- **TextMesh Pro 3.0.7** — text rendering
- **com.coplaydev.unity-mcp** — AI/MCP integration
- **Freeui ZOSMA** — UI kit (Main, Set, Stage, Start screens)
- **Epic Toon FX** — VFX particles

## Architecture: Composition Over Inheritance

All game logic must follow the component composition pattern documented in `AI/UNITY_COMPOSITION_GUIDE.md`. Key rules:

- One `MonoBehaviour` = one responsibility
- Compose GameObjects from small, reusable `*Component` scripts
- Wire dependencies via `[SerializeField]`, not `GetComponent` in Update
- Components communicate through C# events (`event Action`), subscribe in `OnEnable`, unsubscribe in `OnDisable`
- Use `AndCondition` pattern for extensible preconditions on component actions
- Orchestrator scripts (e.g. `Character`) only hold references, set up conditions in `Awake()`, and wire events — minimal own logic
- Use interfaces (`IDamageTaker`, etc.) for cross-component interaction, not concrete types
- Inheritance is acceptable only for Template Method pattern or when 80%+ of logic is shared

## Code Conventions

- Components: suffix with `Component` (e.g. `LifeComponent`, `MoveComponent`)
- Controllers: suffix with `Controller`
- Private fields: `_camelCase` with `[SerializeField]`
- No public fields — use `[SerializeField] private` + read-only property if needed
- Group serialized fields with `[Header("...")]`
- Files under 200 lines
- New scripts go in `Assets/Scripts/` organized by system (`Components/`, `Controllers/`, `Objects/`)

## Developer Preferences (from AI/CLAUDE.md1)

- Be terse, code-focused — no high-level fluff
- Fewer lines of code is better
- Proceed like a senior developer — assume expertise
- Give the answer immediately, explain after if needed
- Think in Russian, code in English
- Minimal changes when fixing bugs — consider multiple causes before deciding
- Test after every meaningful change
- Never break existing functionality
