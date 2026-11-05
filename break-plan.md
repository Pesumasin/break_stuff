# BREAK STUFF — Game Specification & Implementation Plan

Single source of truth for building the game. Implementer: Claude Sonnet via Roblox Studio MCP. Section refs like **§6.3** point inside this file. Decisions marked **[D]** are final unless the user changes them; **[A]** are assumptions (may be revised without asking if playtesting proves them wrong; log the change in §20).

---

## 0. How to use this file

- Follow the session routine in §0.1. Read §0–§3 and the current phase in §17 before each work session. Read other sections when the task touches them.
- Work through §17 phases in order. Do not start a phase before the previous phase's Definition of Done passes.
- Keep §19 (checklist) and §20 (decision log) up to date. They are the only parts of this file the implementer edits routinely. Change the spec only when the user approves, and log it in §20.
- The Studio place is the source of truth for code and assets. This repo holds the plan. **[D]** No Rojo or file sync: all scripts, UI and objects are created and edited in Studio through MCP.

### 0.1 Session handoff (work may continue on another device, in a new session) **[D]**
A new session knows only what is written in this repo and what is in the Studio place. Continuity depends on both being current.
- **Place**: the place is published and **Team Create is on** (set up in P0), so every Studio edit autosaves to the Roblox cloud and any device opens the latest version. Never rely on a local `.rbxl`.
- **Session start**: (1) `git pull`. (2) Read §0–§3, §4.5, §19 "Current state" and the current phase in §17. (3) Confirm Studio has the place open (not a local copy) via MCP state. (4) Inspect the tree and compare it with "Current state". If they disagree, trust the place, fix "Current state", and note it in §20. (5) Continue from "Next step".
- **Session end, and after every completed task** (not only per phase): update §19 "Current state" and the checklist, add §20 entries, then commit and push **break-plan.md only**: `git add break-plan.md && git commit -m "Progress: <task>" && git push`. Never leave the plan with unpushed progress.
- If a task is interrupted halfway, record exactly what is half-done in "Current state" (which scripts and objects exist, what is unwired, what is untested).

### 0.2 Session rollover (save tokens, keep context sharp) **[D]**
Long sessions re-send their whole history on every turn, which costs tokens and dilutes attention. Fresh sessions starting from "Current state" are cheaper and more focused. Sonnet **tells the user to start a new session** when any of these is true:
- A phase's Definition of Done just passed.
- About 3 tasks have been completed in this session.
- The conversation was compacted or summarized, or is clearly long (many tool outputs, long debugging).
- The next task is in a different area (e.g. from destruction code to UI) and nothing in this session's history is needed for it.
- A debugging tangent ended, so its history is now noise.

Never suggest a rollover mid-task or while waiting for the user's asset answer. First finish the current task and the §0.1 session-end routine (state updated, plan pushed). Then end the report with exactly:
```
🔁 Good point to start a new session (saves tokens). Paste this there:
Continue Break Stuff. Follow CLAUDE.md: git pull, run the break-plan.md §0.1 session-start routine, and resume from §19 "Current state" (next: <next step>).
```

---

## 1. Vision (one paragraph)

First-person, colourful, chunky cartoon **demolition simulator**. You are the rookie at **Smash & Sons Demolition Co.**, a slightly incompetent company that takes absurd jobs: knock down Grandma's shed, then houses, shops, castles, banks and skyscrapers. You smash block structures, fill your rubble bag, feed it to a giant **Crusher** for cash, and buy bigger tools. Every structure collapses in stages with a big show. The fun comes from **impact, not clicking**: every hit gives instant feedback, every break throws chunky debris, and every collapse is a spectacle. Co-op squads of up to 4 smash the same structure together.

---

## 2. Locked design decisions (from the user interview)

| # | Decision |
|---|---|
| D1 | Theme: cartoon demolition company. Missions are "contracts" from silly clients. The humour lives in mission names, client blurbs, UI copy and over-the-top effects. |
| D2 | **Phased collapse**: each structure has N phases. Breaking ≥20% of the blocks remaining at the start of a phase collapses that phase's section in a big show. The final phase collapses everything = mission complete (§8). |
| D3 | **Full bag**: blocks still take damage but cannot break (HP stays at a minimum of 1) while the attacker's bag can't fit them. Big "BAG FULL — go sell!" warning plus a guide arrow to the nearest Crusher. No resources are ever lost. |
| D4 | Map flow: one place. **Hub** (HQ, shop, crusher, mission board, gate) plus **mission arena(s)** in the same place. Later, squads get reserved servers of the same place (§10). |
| D5 | Co-op loot: **whoever breaks a block gets it** in their own bag. On each phase collapse and mission completion, every contributor gets the same money bonus. Squads of max 4. Individual inventory and money; structure and mission progress are shared. |
| D6 | **Most blocks, especially early, are one hit.** Progression comes from speed, area, reach and the ability to break tougher materials (hardness tiers), not from grinding HP. |
| D7 | Tools: Hands → Pickaxe → Sledgehammer → Chainsaw → Rocket Launcher → Laser. The MVP ships Hands, Pickaxe and Sledgehammer. Helicopter is far future. |
| D8 | Seller: **Giant Crusher** now. A "Hungry Rock Monster" variant comes later as a reskin or cosmetic. |
| D9 | Replay: completed missions are replayable for money and **star ratings** (time). Rebirth/prestige is not planned now; keep the economy multiplier-friendly so it can be added. |
| D10 | PC first and first-person only. Mobile later (design input/UI so it can be added). Console out of scope. |
| D11 | The user picks all external assets and sounds (§14). |

Visual references: `reference/style-outdoor.webp` (world, terrain, arms, lighting) and `reference/style-indoor-hud.webp` (HUD, cracks, rubble, target outlines). They are the style target for §13. Use them as style guidance only, never as assets to copy.

---

## 3. Core loop & moment-to-moment feel

1. Spawn at HQ (hub). Mission Board → pick a contract → fade → arena.
2. Aim at a block (centre crosshair) → attack → instant local swing, hit spark and sound. The server confirms the break → block pops, debris cubes fly, dust puffs, small screen shake, a "+$" chip floats up and the bag counter ticks.
3. The phase progress bar fills toward **COLLAPSE**. At 100%: 1 s warning rumble → the section collapses (physics show) → banner and phase bonus.
4. Bag full → warning plus arrow → walk to the arena Crusher → step on the pad → rubble flies into the hopper, the jaws chomp, coins burst, money ticks up.
5. Buy tools and upgrades at a shop counter (in hub and arena). Feel the difference immediately.
6. Final phase collapses → results card (time, stars, earnings, unlock) → Replay or Back to HQ.

Readability rules: money, bag (count/cap), equipped tool, and objective (phase k/N plus progress bar) are always visible. In missions, a timer pill (clock icon, `m:ss`) shows elapsed time, since stars depend on it. A target block under the crosshair gets a subtle outline (SelectionBox), turning red if the tool tier is too low. **Area tools preview their full hit area**: every cell the next hit will reach is outlined in orange (§6.5).

---

## 4. Architecture overview

### 4.1 Principles
- **Server authoritative**: damage, breaks, rewards, bag, money, purchases, ownership, mission progress and completion. Client: input, camera, viewmodel, UI, all cosmetic FX, optimistic prediction (§5.6).
- **Data-driven**: materials, tools, upgrades, missions, economy and assets live in config ModuleScripts. Adding content = adding config rows plus (optionally) a blueprint function.
- **Mission instances from day one**: all structure and mission logic runs inside a `MissionInstance` object with a `members` set. Solo = squad of one. This keeps co-op (§10) a small change, not a rewrite.
- **Minimal framework**: plain ModuleScript "services" (server) and "controllers" (client) with `init()`/`start()` called by one bootstrap script each. No third-party frameworks. Exception: none in the MVP. Any third-party module needs user approval (§14).

### 4.2 Studio hierarchy **[D]**
```
ReplicatedStorage
  Shared/                      (ModuleScripts, required by both sides)
    Config/
      Materials, Tools, Upgrades, Economy, Missions, Assets, FX, Theme
    Blueprints/                one ModuleScript per structure (pure functions → cell list)
    Util/
      Grid        cell math (index<->xyz, neighbours, sphere/ray iteration)
      Format      number formatting ($1.2K, 3.4M)
      Packer      buffer pack/unpack for FX batches
      BlueprintKit  shape helpers for blueprints (§5.1)
      ToolStats   effective tool stats from upgrades (§6.2)
  Remotes/                     (Folder; RemoteEvents/RemoteFunctions created in Studio, names in §4.4)
  Assets/                      ToolModels/, CrusherParts/ (placeholder models built from Parts)
ServerScriptService
  Server.server.lua            bootstrap: requires services in order, init() then start()
  Services/
    DataService                load/save/session lock (§11)
    PlayerStateService         money, bag, owned tools, upgrades; replication to client
    ToolService                equip, attack validation, damage dispatch (§6)
    StructureService           grid model, spawning, damage, breaking, FX batching (§5)
    MissionService             MissionInstance lifecycle, phases, collapse, completion (§8)
    EconomyService             selling, purchases (§7)
    SquadService               (P8+) squads, invites (§10)
  Tests/
    RunTests                   ModuleScript: pure-logic self-tests, run via MCP execute_luau
ServerStorage
  (empty in MVP; mission-server-only assets later)
StarterPlayer/StarterPlayerScripts
  Client.client.lua            bootstrap
  Controllers/
    CameraController           first-person lock, shake, FOV kick
    InputController            action mapping (ContextActionService) → abstract actions
    ViewmodelController        arms + tool model on camera, procedural anims
    ToolController             targeting raycast, cooldown UI, sends attacks, prediction
    FXController               debris pool, particles, sounds, collapse show, sell FX
    UIController               screen routing; one module per screen in Controllers/UI/ (HUD, Shop, MissionBoard, Results, Toasts)
StarterGui
  HUD (ScreenGui), Shop, MissionBoard, Results, Toasts   (built in Studio, logic in UIController)
Workspace
  Hub/                         HQ, ShopCounter, Crusher, MissionBoard, MissionGate, spawn
  Arenas/Arena1/               plot, ArenaSpawn, StructureOrigin (Part), Crusher, ShopCounter, ExitPad
  Missions/                    runtime: <instanceId>/Structure/<block parts>
```
Tag (CollectionService) interactables so new ones work without code changes: `Crusher`, `ShopCounter`, `MissionBoard`, `ExitPad`, `ArenaSlot`.

### 4.3 Naming
Services/controllers/modules: PascalCase. Locals and functions: camelCase. Config ids: lowercase strings (`"pickaxe"`, `"brick"`, `"m01_shed"`). Remotes and attributes: PascalCase. Constants: UPPER_SNAKE. One responsibility per module. Keep modules under ~400 lines and split if they grow.

### 4.4 Remotes **[D]**
| Name | Type | Dir | Payload | Server validation |
|---|---|---|---|---|
| `EquipTool` | Event | C→S | toolId: string | owned, exists |
| `UseTool` | Event | C→S | target: BasePart?, hitPos: Vector3, aimDir: Vector3 | §6.3 |
| `Purchase` | Function | C→S | kind: "tool"\|"upgrade"\|"capacity", id: string, track: string? | §7.3 |
| `StartMission` | Function | C→S | missionId | unlocked, not already in mission, arena slot free |
| `LeaveMission` | Event | C→S | — | is member |
| `MissionResultChoice` | Event | C→S | "replay"\|"hub" | squad leader for replay |
| `StateSync` | Event | S→C | full or partial state table (owned, upgrades, missions) | — |
| `BlockFX` | Event | S→C | buffer of break/hit records (§5.5) | — |
| `CollapseFX` | Event | S→C | phase index, warn/do flag, buffer of collapsing cells | — |
| `ProjectileFX` | Event | S→C | toolId, origin, dir, speed, serverTime (P7) | — |
| `MissionState` | Event | S→C | {missionId, phase, phases, progress 0–1, elapsed, members} | — |
| `SellFX` | Event | S→C | playerUserId, count, value, crusherId | — |
| `Notify` | Event | S→C | key, args (toasts: "BagFull", "TooTough", "Purchased"…) | — |
| (P8) `Squad*` | Events | both | §10.2 | — |

Hot per-player values are replicated as **Player attributes** set by the server: `Money`, `BagCount`, `BagCap`, `Equipped`, `MissionId`. The client listens with `GetAttributeChangedSignal`. Structured data (owned tools, upgrade levels, mission records) goes through `StateSync` on change.

Every C→S handler: `typeof` check all args, reject NaN/inf vectors, per-player rate limit (token bucket: `UseTool` 25/s burst 30, others 5/s), silently drop invalid requests (log a counter, never error).

---

### 4.5 Code style **[D]** (keeps every session's output indistinguishable)
- Before writing a new module, read 1–2 existing modules of the same kind and match them. Once P0 exists, existing code is the reference style.
- Module template:
  ```lua
  -- StructureService: grid-backed destructible structures (§5).
  local ReplicatedStorage = game:GetService("ReplicatedStorage")
  local Config = require(ReplicatedStorage.Shared.Config.Materials)

  local StructureService = {}

  function StructureService.init() end      -- create state, connect remotes
  function StructureService.start() end     -- begin loops (after all init)

  return StructureService
  ```
- First line: a one-line purpose comment plus the § reference. Other comments explain only *why*, not *what*. No commented-out code, no TODOs without a §20 entry.
- Services first (`game:GetService`), then requires, then constants, then local state, then functions. Small local functions, early returns, no deep nesting.
- Config values only in `Shared/Config`. No magic numbers in services (except trivial 0/1).
- Client↔server communication only through §4.4 remotes. Add new remotes to the §4.4 table in the same task.
- `print` only behind `Config.Economy.DEBUG`. `warn` for recoverable problems. No silent `pcall` except around DataStore/MemoryStore/Teleport calls, which must log failures.
- UI: build instances in Studio and keep logic in UIController modules (one module per screen under `Controllers/UI/`). **Every colour, font, corner radius and stroke comes from `Shared/Config/Theme`** (§13). A small `Theme.apply(guiObject, styleName)` helper styles elements. No hard-coded colours or fonts in UI code or in world builders.

## 5. Destruction system

### 5.1 Grid model **[D]**
- Block size **1×1×1 stud** (user change 2026-11-05: 4 → 2 → 1). Blueprints are drawn at human scale: 1 cell = 1 stud, a character is ~5 studs tall (storey ≈ 9 studs, door 3×7, window 3×4). Structures are axis-aligned on a grid anchored at an `origin` CFrame (position only, no rotation in the MVP).
- A structure is a server-side table:
  ```lua
  Structure = {
    size = Vector3int16 (cells), origin = Vector3,
    cells = { [index] = {mat=matId, hp=number, section=sectionId} } -- only non-empty cells
    parts = { [index] = BasePart },  partToIndex = { [BasePart] = index },
    sectionCounts = { [sectionId] = aliveCount }, alive = number,
  }
  index = x + y*SX + z*SX*SY   (0-based cells; Util.Grid owns the math)
  ```
- **The grid is authoritative; Parts are views.** All damage, area and ray queries iterate the grid (no `GetPartBoundsInRadius`/physics queries), which gives deterministic, cheap and dedupe-free AoE.
- Blueprints (`Shared/Blueprints/<id>`) are pure functions returning `{size, sections={ids in collapse order}, cells={ {x,y,z,mat,section} ... }}`. They are built from helpers in a `BlueprintKit` util: `box`, `hollowBox(wallThickness)`, `floor`, `gableRoof`, `windowRow`, `door`, `fillMaterial(predicate, mat)`. Hand-typed cell lists are forbidden: compose helpers. **[D]** Structures are made from data, never hand-placed parts, so they can be rebuilt, replayed and loaded in mission servers.

### 5.2 Block parts
Part properties: `Anchored=true, CanCollide=true, CanTouch=false, CanQuery=true, CastShadow=` config `FX.blockShadows` (default true, set false for structures >3000 blocks), `Material=SmoothPlastic` (always, §13), `Color = material.color` with deterministic brightness variation = **patterned noise** `math.noise(x*0.18, y*0.18, z*0.18 + seed)` → ±8% (darker/lighter blotches spanning neighbouring blocks, like stains on a wall) plus ±2% per-block hash jitter (voxel richness at zero cost, no textures), `Name="B"`, attribute `M` = matId (uint) for client FX and prediction. Parent: `Workspace.Missions.<instanceId>.Structure`.

Spawning: batched at ≤300 parts per Heartbeat, **bottom layer first**, so construction reads as the structure "rising" (with a client dust puff per layer row; optional). Never `Instance.new` thousands of parts in one frame.

### 5.3 Materials (Config/Materials) **[A] values are placeholders for balancing**
| id | name | hp | value $ | minTier | color | notes | phase |
|---|---|---|---|---|---|---|---|
| `plank` | Planks | 1 | 1 | 0 | `#C8823C` | splinter debris | MVP |
| `roof` | Roof Tile | 1 | 1 | 0 | `#D9534F` | | MVP |
| `glass` | Glass | 1 | 2 | 0 | `#9FE3FF`, Transparency 0.35 | shatter: shards, no cubes | MVP |
| `brick` | Brick | 2 | 3 | 0 | `#B5452E` | hands 2 hits, pickaxe 1 | MVP |
| `concrete` | Concrete | 4 | 6 | 1 | `#B8B8B0` | | P7 |
| `steel` | Steel Beam | 10 | 15 | 3 | `#6E7F8E` | sparks, clang | P7 |
| `gold` | Vault Gold | 20 | 100 | 3 | `#FFD23F` | coin burst, sparkle particles | P7 |
| `tnt` | TNT Crate | 1 | 0 | 0 | `#E8322E` | special: explodes, radius 2 cells, dmg 6 | P7 |

Schema: `{id, name, hp, value, minTier, color, transparency?, debris="cubes"|"shards"|"sparks", soundSet, onBreak=nil|"explode"}`. `onBreak` hooks are dispatched by a small table in StructureService. New special blocks = new hook functions, no core changes.

**Hardness tiers**: if `tool.tier < material.minTier`, the hit does 0 damage and plays a "TOO TOUGH" clank. The outline turns red so players know what to buy next.

### 5.4 Damage & break pipeline (server, StructureService)
```
applyDamage(instance, attacker, hits: {index -> damage}, cause)
  for index, dmg in hits:                       -- caller builds a set: dedupe guaranteed
    cell = cells[index]; if not cell or cell.locked then continue
    if toolTier < Materials[cell.mat].minTier then queue TooTough; continue
    cell.hp -= dmg
    if cell.hp <= 0:
      if not PlayerState.bagCanFit(attacker, 1) then
        cell.hp = 1; notify BagFull (throttled 1 per 2 s); queue hit FX     -- D3
      else breakCell(index, attacker)
    else queue hit FX (index, hp/maxHp)
breakCell: remove from cells/parts maps, part:Destroy(), sectionCounts--, alive--,
           PlayerState.addToBag(attacker, mat, 1), MissionService.onBlockBroken(instance, attacker),
           run material.onBreak hook, queue BlockFX record
```
`onBreak` hooks (e.g. TNT) call `applyDamage` with `attacker` = the breaker and a recursion guard (depth ≤ 3).

Breaking under collapse lock: a section in its 1 s collapse warning is `locked` (no damage).

### 5.5 Replication & FX batching
- Part destruction replicates automatically. **Do not** also replicate per-block state via attributes.
- `BlockFX` is flushed once per Heartbeat per instance, `FireClient` to each **member** (non-members get nothing). Buffer records, 8 bytes each: `x:u8 y:u8 z:u8 (cell coords)`, `mat:u8`, `kind:u8 (0=hit,1=break,2=tooTough)`, `hpFrac:u8`, `attackerSlot:u8 (member index 1–4)`, `pad:u8`, plus a header with the structure origin. `Util.Packer` owns the format plus a self-test.
- **Cracks (standard damage look)**: one-hit blocks never crack. Hits on multi-hit blocks show crack stages 1–4 (user rule 2026-11-05): stage = `ceil((1-hpFrac)*4) + 1 + hashBit(cell)`, clamped 1–4, so the last hit before a break shows heavy cracks (2-hit blocks show 3 or 4). Each stage puts a client-only dark crack `Texture` on all 6 faces (one shared image per stage, random tile offset per face for variety, removed on break) and darkens the colour by `stage*0.06`.
- Budget: ≤ 2 KB/s per client in normal play; rocket bursts may spike to ~1 KB per event. Profile in P6.

### 5.6 Client feel & prediction
- On attack input the client immediately plays: swing anim, hit spark at `hitPos`, hit sound. It does not wait for the server.
- **Optimistic break** (P6): if `material.hp` (or the last known hp from BlockFX) ≤ predicted damage, AND the tier is OK, AND the local `BagCount < BagCap`, then hide the part locally (`LocalTransparencyModifier=1`, set `CanQuery=false` locally) and play the full break FX right away. Record it as pending. If no server break record arrives within 0.6 s, restore the part (rare). Server break records for already-predicted cells skip FX.
- Debris (FXController): **client-only** pooled cube Parts (pool of 200, size 0.8–1.6, material colour, `CollisionGroup="Debris"`, which collides with world/structure and other debris, not players). Per break: 4–6 cubes with outward random impulse, spin. They **settle and linger as rubble for ~3 s** (random 2.5–3.5 s), then shrink over 0.3 s and return to the pool. If the pool is exhausted, recycle the oldest pieces first, then fall back to particles only. Dust: one pooled `ParticleEmitter` per material colour at a reused attachment, `:Emit(n)` with built-in texture `rbxasset://textures/particles/smoke_main.dds`.
- Camera: break shake 0.15 amplitude/0.12 s; collapse shake 1.0/1.2 s; FOV kick +3° on heavy hits. Shakes are additive with a cap. Respect a settings toggle (P6).
- Sounds: pooled `Sound` objects per sound key (max 6 concurrent per key, random pitch 0.92–1.08). Missing asset ids = silent (§14).

### 5.7 Performance plan **[D]**
| Risk | Mitigation |
|---|---|
| Part count | 1-stud blocks; hollow blueprints (walls 1 thick, floors), MVP ≤ 1,500 blocks/structure, later ≤ 6,000 spawned. For solid/large structures (P6/P7 if profiling demands): **surface-only spawning**. A cell only gets a Part when at least one of its 6 neighbours is empty. When a block breaks, spawn Parts for newly exposed neighbours. The grid already holds every cell, so this is a local change in StructureService. |
| Spawn hitches | ≤300 parts/frame batching (§5.2). |
| Network | Per-frame batched buffers, members only, no attributes per block, no physics parts replicated for FX. |
| Physics | Intact blocks anchored. All debris and collapse chunks client-only, pooled and capped. Server simulates nothing physical for destruction. Projectiles are kinematic raycasts (§6.4). |
| Collapse | Chunk collapse cells into 4×4×4-cell clumps (4-stud cubes, `FX.collapse.clump`). Cap 150 physics bodies per collapse; overflow becomes dust bursts plus a quick shrink. Bodies fade out by 4 s. |
| Particles/sounds | Pools, `Emit()` not continuous emitters, concurrency caps. |
| UI | Event-driven updates via attribute signals. No per-frame text updates except active tweens and the objective bar while it animates. |
| Server CPU | Grid ops are O(cells affected). Target: StructureService + MissionService < 1.5 ms/frame under a rocket spam test (MicroProfiler / `debug.profilebegin` labels). |
| DataStore | §11 budgets. |

---

## 6. Tools & combat

### 6.1 Tool identities (Config/Tools) **[A] numbers**
| id | tier | price | input | dmg | cooldown s | range studs | area | identity / special | phase |
|---|---|---|---|---|---|---|---|---|---|
| `hands` | 0 | free | click | 1 | 0.35 | 10 | single | quick jabs, alternating L/R punches | MVP |
| `pickaxe` | 1 | 100 | click/hold | 2 | 0.28 | 12 | single | fast, one-shots bricks, unlocks concrete | MVP |
| `sledge` | 2 | 750 | click | 2 | 0.85 | 12 | sphere r=1.5 cells around target | heavy wind-up, big shake, smashes chunks | MVP |
| `chainsaw` | 3 | 5,000 | hold | 1.2/tick | 0.08 | 9 | target + 1 cell behind (cut) | continuous shredding, best DPS vs hard blocks, unlocks steel | P7 |
| `rocket` | 4 | 25,000 | click | 8 centre | 2.5 | 300 | sphere r=3 cells, linear falloff to 35% | kinematic projectile, huge blast, knock-back FX | P7 |
| `laser` | 5 | 150,000 | hold | 3/tick | 0.1 | 120 | ray pierces up to 8 blocks | heat-up 0.5 s, beam cuts straight tunnels | P10 |

Schema: `{id, name, tier, price, unlockMission?, input="click"|"hold", damage, cooldown, range, area={kind="single"|"sphere"|"cut"|"pierce", radius?, depth?, falloffMin?}, projectile?={speed, gravity=0}, viewmodel="ToolModels/<id>", upgradeTracks={...}, soundSet}`.

Purchase requirement: `price` plus optional `unlockMission` (e.g. chainsaw requires `m03_store` done). This gates content pacing without hard walls on materials.

### 6.2 Upgrades (Config/Upgrades) **[A]**
Each tool has 2 tracks, max level 5. Cost for level L = `base * growth^(L-1)` (growth 1.8). Identity-driven:
| tool | track A (effect per level) | track B |
|---|---|---|
| hands | Speed: cooldown ×0.9 | Reach: +1 stud |
| pickaxe | Speed ×0.9 | Power: dmg +0.5 |
| sledge | Radius +0.25 cells | Speed ×0.9 |
| chainsaw | Power +0.3/tick | Cut depth +1 (max 3) |
| rocket | Blast radius +0.4 cells | Reload ×0.88 |
| laser | Pierce +2 | Power +0.6/tick |

Track base costs ≈ 30% of the tool price (hands: $25).

**Bag capacity** (global upgrade): `[20, 35, 55, 80, 120, 175, 250, 350, 500, 750, 1000]`, costs `[_, 50, 150, 400, 900, 2000, 4500, 9000, 18000, 35000, 70000]`.

Effective stats are computed by one pure function `ToolStats.get(toolId, upgradeLevels)` used by server (truth) and client (UI/prediction). Self-test it.

### 6.3 Attack validation (server, ToolService.onUseTool)
1. Rate-limit token bucket passes. Player alive, has character, is a member of a mission instance (hub attacks ignored).
2. `equipped` (server-side value) is owned. Cooldown: `now - lastUse >= stats.cooldown * 0.85` (tolerance for jitter). Else drop.
3. Melee/hold tools: `target` is a BasePart in `partToIndex` of **the player's instance** (prevents cross-squad damage). Distance from head to the target's closest point ≤ `stats.range + 3`. LOS: server raycast head → hitPos ignoring characters and debris; the first hit must be the target or within 1 cell of it.
4. Build the hit set by `area.kind` using Util.Grid (sphere → cells within radius with falloff; cut → target + cells along `aimDir` axis-snapped; pierce → 3D DDA traversal along aimDir up to depth/range). Dedupe is inherent (table keyed by index).
5. `StructureService.applyDamage(instance, player, hits, toolId)`.

The client never sends damage, area, rewards or cell lists.

### 6.4 Projectiles (rocket, P7)
Server: on a valid fire, compute the path by **kinematic stepping raycasts** (speed 160 studs/s, no gravity, step per Heartbeat, max range 300) against the instance's structure + world. On impact → sphere hits → applyDamage. Fire `ProjectileFX` to members at fire time. Clients render a local rocket model plus trail along the same deterministic path (start time from `workspace:GetServerTimeNow()`). Visual impact comes from the server's BlockFX (no separate client impact authority). No physics Parts, no network ownership issues.

### 6.5 Viewmodel & animation **[D]**
- No Roblox `Tool` objects. Custom hotbar (keys 1–6, scroll wheel) avoids Backpack quirks and maps cleanly to mobile buttons later.
- Client viewmodel: a Model (blocky arms + tool built from Parts, `Assets/ToolModels/<id>`) parented to the camera and positioned each RenderStepped with **procedural spring animation**: idle bob, walk bob, mouse sway, swing curves per tool (wind-up / strike / recover = 0.06/0.10/0.18 s for light tools, 0.25/0.12/0.3 for sledge), recoil for rocket, vibration for chainsaw/laser. No animation assets are required.
- Other players (P8): server welds a copy of the tool model to the character's right hand on `EquipTool`. Swings are shown with a simple client-side Motor6D C0 tween driven by the `attackerSlot` in BlockFX. Real animation assets are optional polish (§14).
- Crosshair: small centre dot plus a circular cooldown ring.
- **AoE preview** (ToolController, client-only): for `sphere`/`cut`/`pierce` tools, each RenderStepped (throttled to 20 Hz) compute the hit set with the same `Util.Grid` function the server uses, and outline those cells with pooled `SelectionBox`es (orange `#FF8A00`, thickness 0.05, max 40; the centre cell gets a thicker box). Rocket previews the blast at the crosshair ray's impact point. Cells the tool can't damage (tier too low) get red boxes.
- **Mouse lock / unlock**: first-person locks the mouse by default. Opening any modal (shop, mission board, results, squad) unlocks it automatically, and closing restores the lock. **V** toggles unlock by hand (via InputController action `ToggleMouse`). While unlocked, attacks are disabled and the camera doesn't turn. A HUD hint pill bottom-left shows `[V] Unlock mouse` / `[V] Lock mouse`.

---

## 7. Inventory, selling, shop, economy

### 7.1 Bag **[D]**
- Server state: `bag = {[matId]=count}`, `bagCount`, `bagCap` (from capacity level). Capacity unit = **blocks** (UI: "Bag 37 / 55"). Value is per material, so better structures fill the bag with more valuable rubble.
- `bagCanFit(p, n)`, `addToBag(p, mat, n)`, `clearBag(p) -> valueTotal`. Bag persists in save data.
- At ≥90%: bag bar pulses orange. At 100%: red "BAG FULL — go sell!" toast (throttled), plus a 3D guide arrow (Beam from character to the nearest `Crusher`). Blocks clamp at 1 HP (D3).

### 7.2 Selling (Crusher) **[D]**
- Server-side `Touched` on the Crusher's `SellPad` (tagged model). Per-player `selling` debounce (1.5 s). If `bagCount > 0`: `value = Σ count × material.value × sellMultiplier(=1)`, **clear the bag first**, then `money += value`, then `SellFX` to that player plus nearby players.
- Client FX: up to 20 rubble cubes arc from the player into the hopper (0.6 s), jaws chomp ×3 (TweenService on jaw parts), coin burst out the front, money label counts up, `+$123` big floater.
- Crusher placeholder model: big yellow/black-striped box body, two grey jaw blocks, hopper funnel, conveyor pad, smoke stack. Built from Parts, with `Jaw1`/`Jaw2`/`Hopper`/`SellPad`/`CoinSpout` named parts so a later reskin (Rock Monster) keeps the same contract.

### 7.3 Purchases
`Purchase` RemoteFunction → EconomyService: validate the kind/id exists, requirements met (unlockMission, previous capacity level, not maxed, not owned), player is within 20 studs of a `ShopCounter`, `money >= price`. Then `money -= price`, grant, mark data dirty, **request a priority save** (§11), return `{ok=true}` or `{ok=false, reason}`. Server-side purchase lock per player (no concurrent purchases).

### 7.4 Economy pacing targets **[A]**
| milestone | target playtime |
|---|---|
| First capacity upgrade | ~40 s |
| Pickaxe | ~1.5–2 min |
| Sledgehammer | ~8–10 min |
| Mission 1 complete | ~10–12 min |
| Chainsaw | ~45–60 min |
| Rocket | ~2–3 h |
| Laser | ~6–8 h |

All values live in `Config/Economy` and `Config/Tools`. Phase bonuses and completion rewards are in `Config/Missions`. Money is an integer (`math.floor`) with no hard cap. Display via `Util.Format`. P6 includes a balancing pass that logs `$ per minute` in Studio playtests (debug print every 60 s, behind a `DEBUG` flag).

---

## 8. Missions, phases, campaign

### 8.1 Phased collapse — exact rules **[D]**
- A blueprint lists `sections` in collapse order (usually top → bottom). Every cell belongs to exactly one section. Phase k (1..N) owns section k. The final phase owns the remaining section(s).
- At phase start: `phaseBase = structure.alive`, `phaseBroken = 0`.
- Every break **anywhere** in the structure: `phaseBroken += 1`. `progress = phaseBroken / (phaseBase * threshold)` clamped 0–1 (`threshold` default 0.20, per-phase override allowed).
- At progress = 1:
  1. **Warn (t=0)**: lock section k (no damage), `CollapseFX{warn}` → rumble sound, cracks flash, strong shake ramp, banner "IT'S GOING!".
  2. **Collapse (t=1.0 s)**: server removes all alive cells of section k (no bag rewards: collapsed rubble is paid out as the bonus), sends `CollapseFX{do, buffer of cells+mat}`, then destroys the parts in batches. Clients build ≤150 clumped debris bodies with outward/downward impulses, big dust, heavy shake. Bodies fade by t+4 s.
  3. **Reward**: every contributor (≥1 break during this phase, member at the time) gets `phaseBonus[k]` money. Banner "PHASE k DEMOLISHED! +$X".
  4. If k < N: phase k+1 starts (re-base). If k = N: **mission complete** (§8.2).
- Floating leftover blocks are allowed (cartoon physics). Blocks in later sections stay breakable at all times and count toward progress.
- Edge case: if a phase's section is already fully destroyed when the threshold is hit, still play warn plus a dust-only "collapse" and pay the bonus.

### 8.2 Completion & replay
- Results card: time, blocks broken (per player), bag earnings this run, phase bonuses, completion reward, stars. **Stars**: 1 = complete, 2 = under `parTime`, 3 = under `goldTime` (per mission config).
- Rewards: `completionReward` on first clear, `replayReward = 40%` of it on repeats. Record `{done, bestTime, stars}`. Unlock the next mission (`unlocks` field).
- The squad leader picks **Replay** (rebuild with the rising-construction animation after 3 s, timer restarts) or **Back to HQ** (instance destroyed). Others can leave anytime via ExitPad or the Leave button.

### 8.3 Mission config schema
```lua
{ id="m01_shed", order=1, name="Grandma's Shed", client="Grandma Edna",
  blurb="It's haunted. Probably. Knock it down, dear.", blueprint="Shed",
  requires=nil, unlocks="m02_house", threshold=0.20,
  phaseBonus={40,80}, completionReward=150, parTime=600, goldTime=360,
  arena="standard", materialsHint={"plank","roof","glass","brick"} }
```

### 8.4 Campaign roadmap **[A]**
| # | Mission | Blocks | Phases | Highlights | Phase |
|---|---|---|---|---|---|
| 1 | Grandma's Shed | ~1,400 | 2 (roof → rest) | planks, roof, glass, brick chimney | MVP |
| 2 | Suburban House | ~900 | 3 | brick walls, garage, pool fence | P7 |
| 3 | Corner Store | ~1,500 | 3 | concrete, big shop windows, TNT crate stash | P7 |
| 4 | Medieval Castle | ~3,000 | 4 | concrete "stone", towers collapse separately | P7 |
| 5 | City Bank | ~3,500 | 3 | steel frame; final phase reveals the gold vault | P7 |
| 6 | Office Tower | ~5,000 | 5 | tall, floor-by-floor collapses, surface-only spawning | P10 |
| 7 | Mega Mall (event) | 3 structures | 3 each | multi-structure (§8.5) | P10 |
| 8 | Skyscraper | ~6,000 | 6 | laser showcase | P10 |

### 8.5 Multi-structure missions (P10)
A mission may list several `{blueprint, originOffset}`. Each structure has its own phases. The mission completes when all structures are complete. The objective UI shows one row per structure. MissionInstance holds a list of structures from the start (MVP list length 1) so this is additive.

---

## 9. World layout

### 9.1 Hub (HQ) — at origin
Circular plaza (~160 studs) of chunky grass/dirt plateau parts. Elements: **Smash & Sons HQ** (cute blocky office with a big sign), **Shop counter** (tagged, ProximityPrompt "Open Shop"), **Crusher** (also sells in the hub), **Mission Board** (ProximityPrompt "Contracts"), **Mission Gate** (decorative arch; Mission Board opens the same UI), spawn pad, a few blocky trees/fences/props. Invisible boundary walls. Later: squad lounge (P8/P9).

### 9.2 Arena — `Workspace.Arenas.Arena1` at (0, 0, 3000)
Flat fenced plot 220×220 studs. Structure pad at the centre with `StructureOrigin` part. **Crusher 45–60 studs from the structure** (the walk is part of the loop). ShopCounter next to the Crusher. ExitPad back to HQ. Spectator-friendly sightlines. In P8, add `ArenaSlot` copies at (±3000, 0, 3000) etc. All arena geometry is anchored and decorative only.

Travel: `StartMission` → server creates the instance → screen fade (client, 0.4 s) → `PivotTo(ArenaSpawn)` → structure builds (rising) while the player watches. Exit = reverse.

---

## 10. Multiplayer architecture

### 10.1 Evaluation **[D]**
| Option | Isolation | Studio-testable | Complexity | Verdict |
|---|---|---|---|---|
| Many squads share a public server, arenas far apart | Weak (visible/reachable) | Yes | Low | Stepping stone only (P8) |
| **Reserved servers of the same place per squad** (public server = hub, reserved = mission) | Full: only teleported members can join | Partly (logic yes, teleports need a published game) | Medium | **Final (P9)** |
| Separate hub place and mission place | Full | Worse (two places to keep in sync) | Higher | Rejected |

**Role detection** (single place): `isMissionServer = game.PrivateServerId ~= "" and game.PrivateServerOwnerId == 0`. In Studio, force a role with `Config/Economy.DEV_ROLE = "hub"|"mission"|nil`.

### 10.2 Squads (P8: in-server; P9: across servers)
- A solo player is a squad of 1 (MVP already uses `members`).
- Create: Squad panel in the hub → "Create squad" (you lead). Invite players **in the same hub server** from a list → invitee gets a toast (Accept/Decline, 20 s expiry). Max 4. Friends join via Roblox "Join" into the hub, then get invited.
- Leader picks a mission they have unlocked. All members get first-clear credit/unlocks on completion **[A]** (co-op friendly: carrying friends is fine).
- Leader leaves → leadership passes to the longest-standing member. Squad dissolves when empty.
- Server checks for every squad op: sender is the leader (start/kick/invite), target exists and is not in another squad, size limit, invite exists and has not expired.

### 10.3 Shared vs individual **[D]**
Shared: structure grid, phase/progress, mission timer, results. Individual: bag, money, tools, upgrades, campaign records, stats. Contribution = breaks during a phase (D5). AFK protection: a member with 0 breaks in the phase gets no phase bonus.

### 10.4 Reserved-server flow (P9)
1. Hub: leader presses Start → server `TeleportService:ReserveServer(placeId)` → gets `accessCode`, `privateServerId`.
2. Write the manifest to **MemoryStoreService** HashMap `SquadManifest[accessCode] = {missionId, leaderId, members={ids}, created}` (TTL 2 h).
3. `TeleportAsync(placeId, players, TeleportOptions{ReservedServerAccessCode})`. Handle failure with retry ×3 and a "Teleport failed" toast.
4. Mission server boot: read the manifest (retry up to 10 s), create the MissionInstance, and accept only listed userIds (kick others with a message). `TeleportData` is **not trusted**.
5. Write `ActiveMission[userId] = {accessCode, expires}` per member (MemoryStore, TTL 30 min, refreshed every 60 s while the server lives).
6. Disconnect → the mission continues for the others. Their bag/money is saved on leave. Hub shows "Rejoin your squad" if `ActiveMission` exists and the server heartbeat is fresh → teleport with the access code.
7. Mission end → "Back to HQ" teleports everyone to the public place. Empty mission server → shuts down (mission progress is not persisted mid-mission **[A]**: missions are 5–20 min).
8. Data handoff: save + release the session lock on leave **before** the teleport completes. The destination load retries for a lock up to 15 s (§11).
9. Testing: logic is tested in Studio with `DEV_ROLE` and multi-client Local Server. Real teleports require the user to publish and test in the live game (Sonnet asks the user).

### 10.5 Security in co-op
All of §6.3, plus a membership check on every structure op. A player can only target parts in their own instance. No PvP: characters have no damage paths. Set `CanCollide` between players off via the collision group `Players` to prevent body-blocking griefs **[A]**.

---

## 11. Persistence (DataService) **[D]**

- Custom minimal DataService (no third-party module; ProfileStore is acceptable later only with user approval). DataStore `"PlayerData_v1"`, key `"u_"..userId`. In Studio: `"PlayerData_DEV_v1"`.
- **Load** with `UpdateAsync`: if `lock` exists, its `jobId ~= game.JobId`, and `lock.time > now-1800` → retry every 3 s for up to 15 s, then take over (stale-lock steal). Write `lock={jobId, time}`. Sanitize/migrate (`v` field). On total failure (DataStore down), kick with "Couldn't load your data, please rejoin" (never play on default data that would overwrite real data).
- **Save** (`UpdateAsync`, verifies we still own the lock): autosave every 120 s ±15 s jitter if dirty. Priority save after purchases, mission completion and capacity upgrades (debounced 10 s). On `PlayerRemoving` save + release the lock. `game:BindToClose`: save all in parallel (`task.spawn`), wait until done or 25 s.
- Retries: 3 attempts, backoff 1/2/4 s. Respect `GetRequestBudgetForRequestType`. Log failures, keep dirty.
- Never save per hit/break. Bag changes mark dirty; the autosave covers them.
- Schema v1:
  ```lua
  { v=1, money=0, bag={}, capLevel=1,
    tools={ hands={owned=true, up={0,0}} }, equipped="hands",
    missions={ m01_shed={done=false, bestTime=nil, stars=0} },
    stats={ blocks=0, earned=0, playTime=0 }, tutorialStep=0, badges={},
    settings={ music=0.5, sfx=0.8, shake=true, debris="normal", aoePreview=true },
    receipts={} }      -- later, monetization idempotency
  ```
- Sanitize on load: clamp numbers to ≥0 and finite, drop unknown tool/mission ids, ensure `hands` is owned, `equipped` is owned, `capLevel` is in range, bag counts are integers ≤ cap.
- Studio needs **Game Settings → Security → Enable Studio Access to API Services**, which requires a published place. If unavailable: in-memory mode with a loud warning. Sonnet asks the user to publish/enable this at P3.

---

## 12. Security checklist (exploit → defence)
| Exploit | Defence |
|---|---|
| Spoofed damage/area/rewards | The client sends only target/aim; the server computes everything (§6.3). |
| Attack spam / fast-fire | Token bucket plus server cooldown per tool. |
| Reach / through-wall hacks | Range plus server LOS raycast. |
| Damaging another squad's blocks | Instance membership plus `partToIndex` of own instance. |
| Sell dupes | Debounce, bag cleared before money is added, server-side pad trigger. |
| Purchase exploits | Prices from server config, lock, owned/requirement checks, proximity. |
| Fake mission completion | Completion only from server phase logic. |
| Teleport data forgery | MemoryStore manifest is the authority. |
| Remote argument fuzzing | `typeof` checks, NaN/inf rejection, pcall-free handlers that early-return. |
| Data corruption | Sanitize on load, never save default data after a failed load, lock ownership on save. |

---

## 13. Design bible

**Pillars**: chunky, bold, readable, playful, juicy. Everything looks like it's made of the same toy blocks.

**References**: `reference/*.webp` (§2). Look: flat colours, **no textures, no shiny materials**. Every Part in the game uses `SmoothPlastic` (no Metal/Glass/Neon/realistic materials). Materials differ only by colour, transparency and FX.

**Palette** (single source: `Shared/Config/Theme`, used by UI, world builders, blocks and FX):
- World: grass `#6BC04B`, grass dark `#4E9A36`, dirt `#8B5A2B`, sand `#E8C98A`, sky `#8FD3FF`, rock `#9AA3AE`.
- Materials: §5.3 colours. Each material must contrast with the arena ground.
- UI: primary yellow `#FFC93C`, action orange `#FF8A00`, money green `#3DDC84`, danger red `#FF4D4D`, info blue `#3BA7FF`, panel cream `#FFF6E0`, outline/ink `#2B2B3A`.
- Brand stripes (machines/HQ): yellow `#FFC93C` / ink `#2B2B3A` hazard stripes.

**Environment**: parts-based chunky terrain (no smooth Terrain), plateau steps of 4 studs. **Ground blocks are two-tone**: a dirt-coloured body plus a thin (0.8-stud) grass-green cap part on top (static decoration only; destructible blocks stay one Part each), blocky trees (brown trunk plus 2–3 green cubes), fences, cones, barrels. Lighting: `Lighting.Technology=ShadowMap`, bright midday, Atmosphere density 0.25 with a light blue haze, ColorCorrection saturation +0.15 / contrast +0.05, subtle Bloom (intensity 0.4, threshold 0.95). No realism, no grime.

**Structures/blocks**: 1-stud cube grid, SmoothPlastic, patterned noise colour blotches plus slight per-block jitter (§5.2), dark crack overlays on damaged blocks, rubble cubes that settle briefly, readable silhouettes (gable roofs, big windows, chimneys, flags). Details are made of blocks too (a door = 3×7 cells of a different colour). Every structure has at least 3 materials for visual and economic variety.

**Equipment**: oversized, toy-like proportions (head 1.5× realistic), 2–3 colours each, bold primary accent (pickaxe: wood handle, grey head with yellow band; sledge: red head; chainsaw: orange body; rocket: green tube with yellow stripes; laser: white/cyan sci-fi). Built from Parts first. Meshes only via §14.

**Machines**: Crusher = big, hazard-striped, exaggerated moving parts. All machines show their function through motion (jaws, pistons, conveyors).

**UI**: Font `Enum.Font.FredokaOne` (headings/numbers), `GothamBold` (small text). Rounded panels (UICorner 12), thick ink UIStroke (3 px), drop-shadow offset frame, bright header bands. Sizes in Scale plus `UIAspectRatioConstraint`. A `UIScale` per ScreenGui (mobile later). Buttons: 3D-pressed look (darker bottom band), hover scale 1.05, click scale 0.95. Icons: flat, chunky, ink-outlined (placeholders = coloured squares with text until the user supplies icons). Hierarchy: money top-left (big), objective top-centre, timer pill top-right (missions only), bag pill bottom-centre above the hotbar, mouse-lock hint bottom-left, toasts mid-top, shop/board as centre modals.

**Animation timing**: snappy anticipation → strike → overshoot. UI pops `Back` easing 0.18 s in / 0.12 s out. Number tickers 0.4 s. World FX hits < 0.1 s response, breaks 0.2 s, collapse 1 s warning + 3 s show.

**Particles/FX**: square debris cubes in material colours (signature look), soft round dust puffs, yellow star sparks for hits, coin bursts (gold cubes/discs) for money. No realistic smoke, fire sprites or blood. Max on-screen budget per §5.7.

**Sound direction**: punchy, cartoon-exaggerated, slightly comedic (bonks, crunches, glass tinkles, huge rumbles), short and with bright transients. Music: upbeat, light, loopable, not distracting.

**Copy/humour**: short and silly. Client blurbs ≤ 2 lines. Toasts in caps ("BAG FULL!", "TOO TOUGH! GET A BIGGER TOOL"). No meanness.

**Rule for new content**: reuse the palette, grid, fonts and FX vocabulary. If something can't fit the style, adapt it (blockify, recolour) or reject it. Ask the user before mixing styles.

---

## 14. Asset & sound workflow — MANDATORY

The user personally chooses every external asset (models, meshes, textures, decals, icons, animations, sounds, music, fonts, third-party code). Sonnet **never** picks final external assets.

1. While planning a task, list the external assets it needs.
2. **Blocker?** (the choice changes architecture or implementation, e.g. a rigged model that dictates animation setup) → ask the user **before** building the dependent part.
3. **Otherwise** build the feature fully with Parts, built-in effects, built-in `rbxasset://` textures, or silence. Register every asset slot in `Shared/Config/Assets` with an empty id:
   ```lua
   SFX = { hit_plank = {id="", volume=0.6, purpose="hands/pickaxe hit on planks"} , ... }
   ```
   Code plays nothing if `id == ""`. Placeholders are never final assets.
4. When the feature works and is tested, **pause and ask** for that feature's assets in one small batch, using this template per item:
   ```
   [REQUIRED|OPTIONAL] <slot key>
   Purpose / where: ...
   Type & category: (Audio | Model/Mesh | Decal/Image | Animation | Font) — Creator Store category
   Style: (per §13) ...
   Tech: duration / loop / size px / tri budget / pivot / format (.ogg/.mp3 ≤ 7 min; images PNG, power-of-2, ≤1024)
   Search terms: "...", "...", "..."
   ```
   Separate REQUIRED-to-finalize from OPTIONAL polish. Max ~8 items per batch.
5. Optional asset requests never block unrelated work. Continue with the next task while waiting.
6. When the user gives an id or approves a choice: insert/assign it, verify in a Studio playtest (plays/appears, correct scale/volume/loop, no console errors), mark it in §19.
7. Do not use MCP `search_asset`, `insert_asset` or `generate_*` (mesh/texture/material/scene/model generation) to choose final assets unless the user explicitly asks for that specific item. Licensing: prefer Roblox-uploaded or Creator-Store assets the user confirms are usable.
8. Tasks needing no external assets → do not ask.

Expected batches (ask only when the feature is ready): P1 hit/break SFX per material + crack overlay images (3 stages, required); P2 crusher/coin/UI SFX; P4 tool swing SFX + optional icons; P5 collapse rumble, mission music, victory jingle, optional clock icon; P6 optional meshes for tools/crusher; later per phase.

---

## 15. MVP definition (end of P5) **[D]**
Included: first-person camera and viewmodel; hub plus 1 arena; mission 1 (Grandma's Shed, ~1,400 blocks, 2 phases with collapse, completion, replay, stars); materials plank/roof/glass/brick; tools hands/pickaxe/sledge with 2 upgrade tracks each; bag with capacity upgrades; one Crusher model (in hub and arena); shop UI; mission board UI; HUD (money, bag, objective, hotbar, toasts); server-validated everything; DataStore persistence; debris/dust/shake FX; asset slots registered (sounds as the user supplies them).

Excluded until later: other tools, materials and missions; co-op; reserved servers; mobile; monetization; vehicles; tutorial and settings menu (P6); badges, leaderboards, analytics (P12); daily jobs; rebirth.

**Scope guard**: if a task is not in the current phase, do not build it. Note ideas in §20 under "Ideas/backlog" instead.

---

## 16. Testing approach (MCP)

- **Before editing**: check Studio state; make sure **play mode is stopped** (edits during play are lost). Inspect the tree before creating to avoid duplicates.
- **Edit**: create Scripts/ModuleScripts/LocalScripts and UI via MCP (execute Luau in edit context or the script-edit tools). Prefer targeted edits over full rewrites. Re-read a script after editing it.
- **Unit self-tests**: `ServerScriptService/Tests/RunTests` requires pure modules (Grid, Packer, ToolStats, economy cost formulas, data sanitize, phase progress math) and asserts. Run via MCP execute Luau in edit mode after changing those modules. All must pass.
- **Playtest**: start play via MCP, read console output (zero errors/warnings from our scripts), take screen captures to check visuals, simulate input (keyboard/mouse) where possible, stop play, iterate.
- **Multi-client** (P8+): Studio "Local Server + N players" may not be drivable through MCP. If not, give the user a short manual test script (steps and expected results) and ask them to run it and report.
- **Perf** (P6+): stress blueprint `StressCube` (20×15×20 hollow = ~3,000 blocks; and a solid 5,000 variant in P6). Check frame time, part count, network receive (Shift+F3 / MicroProfiler); record numbers in §20.
- Team Create autosaves the place to the cloud (§0.1). If Studio ever shows unsaved changes, ask the user to save to Roblox. A phase is done only after a clean full playtest.

---

## 17. Implementation roadmap

Each phase leaves the game playable. Assets: request only when the phase's features work (§14).

### P0 — Foundation & MCP check
- Objective: verified MCP connection, project skeleton, first-person movement in a test area.
- **User action first**: publish the place to Roblox and turn on Team Create (Home → Collaborate / File → Publish). Sonnet gives the steps and waits for confirmation, then records it in §20.
- Features: §4.2 folders/bootstraps; Config modules with MVP data (including `Theme` with the §13 palette and fonts); Remotes created; Util.Grid, Util.Format, Util.Packer + Tests; collision groups (`Debris`, `Players`); first-person lock (`CameraMode=LockFirstPerson`), crosshair, V mouse lock/unlock toggle plus hint pill (§6.5); basic Lighting per §13; a flat test plate.
- Depends: —. Assets: none.
- Test: self-tests pass; playtest shows first-person and crosshair, no console errors.
- DoD: skeleton exists exactly per §4.2; Studio state clean; §19 updated.

### P1 — Destruction core
- Objective: break blocks with hands; it feels good.
- Features: StructureService (grid, batched rising spawn, applyDamage/breakCell, BlockFX batching); `TestHut` blueprint (~150 blocks) plus BlueprintKit; a temporary dev MissionInstance auto-created for the player at the test plate; ToolService validation (§6.3) for hands; ToolController targeting, outline, cooldown ring; viewmodel arms with punch anim; FXController debris pool (3 s rubble), dust, shake; crack stages (darkening placeholder); patterned noise block colours; hit/break SFX and crack-image slots (empty).
- Depends: P0.
- Test: break all blocks; spam-click doesn't exceed the cooldown (server log counter); invalid remote args (fire from command bar) are dropped; ~150 breaks leave no leaked debris parts (pool count stable).
- DoD: breaking feels responsive; no errors; then **ask asset batch: hit/break sounds per material, crack overlay images**.

### P2 — Bag, Crusher, money, HUD
- Features: PlayerStateService (attributes, bag ops); D3 full-bag clamp, toast and guide beam; Crusher placeholder model plus sell logic and SellFX; HUD (money, bag bar, toasts).
- Depends: P1.
- Test: fill bag → can't break + warning; sell → bag 0, money = expected sum; stand on pad repeatedly → no double payout.
- DoD: complete earn loop without persistence. Ask batch: crusher/coin/UI click SFX.

### P3 — Persistence
- Features: DataService §11; load/save of money/bag/cap/tools/missions/stats; BindToClose; priority saves.
- Depends: P2. **User action**: enable Studio Access to API Services (ask).
- Test: earn money, stop, replay → money restored; sanitize tests (corrupt table → valid data); simulate a save failure (temporary forced error) → retries plus kept dirty.
- DoD: data survives sessions; no saves per hit (count saves in log).

### P4 — Tools, shop, upgrades
- Features: Config for hands/pickaxe/sledge plus upgrades/capacity; ToolStats; hotbar (1–3, wheel) plus EquipTool; viewmodels for pickaxe/sledge from Parts with swing anims; sledge sphere AoE plus AoE preview (§6.5); shop opening auto-unlocks the mouse; Shop UI (tools tab, upgrades tab, capacity), ShopCounter prompt; EconomyService purchases.
- Depends: P3.
- Test: buy each item; insufficient funds rejected; cannot buy owned; sledge AoE breaks the expected cells exactly once and rewards are correct; upgrade effects measurable (cooldown/radius); data persists.
- DoD: three distinct-feeling tools. Ask batch: swing SFX per tool, optional tool icons.

### P5 — Hub, mission flow, phased collapse (= MVP)
- Features: hub layout §9.1 and arena §9.2; Mission Board UI; MissionService (instance lifecycle, StartMission/Leave/Exit, fade travel); `Shed` blueprint (2 sections); phase progress, warn, collapse (§8.1), CollapseFX client show; phase bonus; results card, stars, unlock, replay rebuild; objective HUD plus mission timer pill; remove the dev auto-instance from P1.
- Depends: P4.
- Test: full run hub → mission → 2 collapses → results → replay → hub; progress math matches the self-test; leaving mid-mission cleans up the instance and parts; data records stars/bestTime.
- DoD: **MVP complete per §15**, a 15-minute clean playtest by Sonnet plus a user playtest request. Ask batch: collapse rumble, victory jingle, hub/mission music, ambient.

### P6 — Feel, performance, balance
- Features: optimistic break prediction §5.6; FOV kick; **first-time guidance** and **settings menu** (both below); stress blueprints; profiling; surface-only spawning if 5k solid > budget; economy pacing check vs §7.4; DEBUG money/min logging.
- Test: perf numbers recorded; pacing within ±30% of targets or values adjusted and logged.
- **First-time guidance (FTUE)**: runs once for new players (`tutorialStep` in save data, skippable, never repeats after completion). Steps, each with an on-screen hint card, a 3D guide arrow/Beam and a pulsing highlight on the target:
  1. Hub: "Grab a contract!" → Mission Board (Grandma's Shed is auto-selected).
  2. Arena: "PUNCH IT!" → highlight the shed. Advances after 5 breaks.
  3. Bag full: "Bag full! Feed the Crusher" → arrow to the Crusher.
  4. After the first sale: "Save up for a Pickaxe: $X / $100" (keep smashing and selling). Once money ≥ the Pickaxe price: "Buy a Pickaxe!" → arrow to the ShopCounter, Pickaxe tile highlighted.
  5. Pickaxe equipped: "Smash 20% to collapse the roof!" → objective bar pulses. The tutorial ends at the first collapse, with a "You're hired!" toast.
  Guidance is a small TutorialController (client) plus a server-validated step counter (server advances steps on real events only).
- **Settings menu** (gear button + key `O`, unlocks the mouse): music volume, SFX volume (SoundGroups `Music`/`SFX`), screen shake on/off, debris amount (Low/Normal/High → pool 60/200/300, rubble linger 1/3/3 s), show AoE preview on/off. Saved in `settings` (§11); applied live.
- DoD: 60 FPS on the stress test on the dev machine; feels instant at 150 ms simulated latency (Studio network sim if available); a new profile completes the tutorial without dead ends; settings persist. Ask: optional meshes for tools/crusher, optional tutorial arrow/hint icons.

### P7 — Content expansion I
- Features: materials concrete/steel/gold/tnt plus onBreak hooks; tier gating feedback; chainsaw (hold, cut) and rocket (§6.4 projectiles); missions 2–5 with blueprints; `unlockMission` requirements.
- Test: every tool/material combination; rocket path determinism (client visual vs server impact within 1 cell); campaign progression flow.
- DoD: campaign up to the Bank is playable start to finish. Asset batches per tool/material.

### P8 — Co-op (in-server squads)
- Features: SquadService §10.2; squad UI; MissionInstance with multiple members; ArenaSlots (several far-apart arenas, one per squad); contributor bonuses; other players' tool models plus swing; member list on the HUD; leader handover; disconnect handling.
- Test: Local Server 2–4 players (manual script for the user if MCP can't drive it): shared progress, individual bags, correct bonuses, non-member cannot damage, leader leaves → handover.
- DoD: 4-player co-op works in one server.

### P9 — Isolated squad servers
- Features: role detection; reserved server flow §10.4; MemoryStore manifest and rejoin; teleport failure handling; data lock handoff.
- Test: logic via DEV_ROLE in Studio; live test by the user on the published game (provide a checklist).
- DoD: squads never see each other; rejoin works; no data loss across teleports.

### P10 — Content expansion II
Laser, Office Tower, Mega Mall multi-structure event, Skyscraper; surface-only spawning required.

### P11 — Mobile
Touch buttons via ContextActionService (attack/hold, hotbar, interact), auto-aim assist at the crosshair, UI scaling pass, performance presets (fewer debris).

### P12 — Launch prep (= publishable game)
- Objective: everything needed for a public launch apart from the user's creative/store work.
- Features:
  - **Badges** (BadgeService, server-awarded, recorded in `badges` to avoid repeat calls): Hired! (tutorial done), First Collapse, each mission complete, each tool bought, 3-star any mission, 10k/100k blocks smashed. The user creates badges and icons on Creator Hub and gives the ids (§14 batch). Code skips badges with empty ids.
  - **Leaderboards**: in-server `leaderstats` ("Smashed" = blocks broken). Global top-50 boards in the hub for total blocks smashed and total money earned, via OrderedDataStore: write on save (only if changed), read every 5 min, displayed on a hub billboard.
  - **Analytics** (AnalyticsService): onboarding funnel per tutorial step; core funnel joined → first break → first sell → first purchase → mission 1 complete → mission 2 complete; economy events on sell (source) and purchase (sink); custom events for phase collapses. All logged server-side, once per player per step.
  - **Clean update shutdown**: on `BindToClose` in a live public server (not Studio), save everyone, then teleport players to a temporary reserved server that sends them straight back to a fresh public server. Mission servers save and return squads to the hub. All within the 30 s limit.
  - **Hub server size**: max 16 players (Game Settings), several spawn pads spread around the plaza to avoid crowding. Since there's no damage, the "spawn protection" is a **"Stuck? [R]" button** that respawns you at the nearest safe spawn (hub or arena), with a 5 s cooldown.
  - **Low-end check**: test with Studio graphics quality 1 and the Low debris setting. Target 30+ FPS in the MVP mission.
  - **Debug cleanup**: `DEBUG=false`; remove dev auto-instances, test parts and stray prints; `RunTests` stays but is never required at runtime; the console is clean in a full playthrough.
- **User launch checklist** (Sonnet reminds the user at this phase and tracks it in §19): game icon (512×512), 3+ thumbnails or a short trailer, title and description, the Experience Questionnaire (maturity rating), genre, public/private settings, badge icons, final asset choices, at least one playtest session with real players and economy tuning from what they observe.
- Test: badges award once; leaderboards update; analytics events show in the Creator Dashboard (the user checks); an update shutdown moves players without data loss (live test by the user); full playthrough with zero console output.
- DoD: game is publishable once the user checklist is done.

### P13 — Monetization (fair)
Cosmetic tool skins, crusher skins (Rock Monster!), debris colour themes, VIP cosmetic badge; optional dev product "2× money for 15 min" (non-essential). MonetizationService with idempotent `ProcessReceipt` (receipt ids in profile). Never sell power that gates content.

### P14 — Helicopter (far future, post-launch update)
Unanchored vehicle Model driven by constraints: `LinearVelocity` (MaxForce-limited → momentum/gradual acceleration) toward an input-derived target velocity, `AlignOrientation` for attitude: bank = −lateral velocity × k, pitch = forward accel × k, yaw from mouse with a turn-rate limit; Q/E altitude; hover bob from low-frequency noise; altitude ceiling/floor. Network owner = pilot (responsive); server validates weapon fire rate, position plausibility and hits (same ToolService path; rockets via §6.4). Rotors spin client-side via CFrame. Cartoon chunky heli model (asset workflow). Enter/exit via ProximityPrompt at a helipad in later missions.

---

## 18. Risks & mitigations
| Risk | Mitigation |
|---|---|
| MCP can't drive multi-client tests | Manual test checklists for the user (§16). |
| Teleport/reserved servers untestable in Studio | DEV_ROLE simulation plus live test by the user (§10.4). |
| Large structure lag | §5.7, surface-only spawning, stress tests in P6 before big content. |
| Predicted break desync | 0.6 s rollback (§5.6). |
| Scope creep | §15 scope guard, backlog in §20. |
| Missing asset ids | Silent/placeholder safe-by-default (§14). |
| DataStore outage | Kick on failed load, retries, never save over unloaded data. |

---

## 19. Progress checklist (Sonnet updates)
- [x] P0 Foundation & MCP check (user: publish [x] + Team Create [x])
- [x] P1 Destruction core — assets: hit/break SFX + crack images [ ] (requested 2026-11-05)
- [x] P2 Bag, Crusher, money, HUD — assets: crusher/coin/UI SFX [x] (DoD passed 2026-11-05; integrated 2026-11-05)
- [ ] P3 Persistence (user: API access [ ])
- [ ] P4 Tools, shop, upgrades — assets: swing SFX, icons (opt) [ ]
- [ ] P5 Hub, missions, phased collapse → **MVP** — assets: collapse, music, jingle [ ]
- [ ] P6 Feel, performance, balance
- [ ] P7 Content expansion I
- [ ] P8 Co-op in-server
- [ ] P9 Isolated squad servers
- [ ] P10 Content expansion II
- [ ] P11 Mobile
- [ ] P12 Launch prep — user: icon [ ], thumbnails [ ], description [ ], questionnaire [ ], badges [ ], real playtest [ ]
- [ ] P13 Monetization
- [ ] P14 Helicopter

### Current state (overwrite every task; keep under ~15 lines)
- Updated: 2026-11-05 by Sonnet (implementation)
- Phase / task: **P1 DoD passed** (2026-11-05). Done: BlueprintKit, TestHut (1,032 cells at 1-stud blocks, 21×15-stud human-scale cottage, door faces +Z/spawn), StructureService, minimal MissionService (dev auto-instance), ToolService (hands), ToolController, FXController, ViewmodelController.
- Built: Server ORDER = StructureService, MissionService, ToolService. Client ORDER = Input, Camera, Tool, FX, Viewmodel, UI. FXController: BlockFX listener, 200-part debris pool (`workspace.ClientDebris`), dust/spark emitters, crack darkening, pooled sounds (silent, ids empty), `CameraController.shake`. ViewmodelController clones `Assets/ToolModels/<id>` (Tools.viewmodel) to the camera; side Models `Right`/`Left` alternate punches; idle/walk bob, turn sway; reloads on `Equipped` change. `Assets/ToolModels/hands` = 2 sides x (Glove + Sleeve), colours Theme.Viewmodel. Swing timings in Tools rows (`swing`), offsets in FX.viewmodel.
- Verified (DoD playtest): all 682 TestHut blocks broken via real UseTool remote (810 accepted hits); debris pool 200 total, max 34 active, 0 active after drain, 0 stray; spam 60 shots/3.5 s → 12 accepted (= 0.2975 s cooldown), 48 `cooldown`; fuzz 15 bad arg sets → 14 `args` + 1 `notMember` (Terrain); junk on other remotes harmless. Zero console errors.
- P1 sounds integrated (2026-11-05): hit_* (brick+roof share one), break_plank/glass/brick, new `miss` slot (whiff on attacks with no block). All 8 ids load; miss + hit_roof played on real clicks; break_plank/brick/glass played via server BlockFX.
- Crack images 1–4 integrated (2026-11-05): render on hit bricks, stage 3/4 mix, per-face offsets; zero errors.
- Not verified: camera shake / walk bob / sway by eye; hit_plank/glass/brick by real click (same code path as hit_roof).
- P2 task 1 done (2026-11-05, Sonnet): PlayerStateService (Server ORDER now Structure, PlayerState, Mission, Tool). Sets attrs Money/BagCount/BagCap; API getMoney/addMoney/spendMoney/bagCanFit/addToBag/getBagValue/clearBag(→value,count)/setCapLevel; overrides StructureService.hooks.canFit/addToBag. Verified in playtest: 20 adds fill cap 20, 21st rejected, value 40 = Σ count×mat value, clear → 0, spend rejects overdraw. Zero errors. Not wired yet: toast/beam (notifyBagFull exists in StructureService; check it fires Notify), Crusher, HUD. Persistence of bag/money is P3.
- P2 task 2 done (2026-11-05): `Workspace.Crusher` placeholder (tag `Crusher`, attr CrusherId=test_plate, at (40,0.8,-10) facing +Z; parts Body/Jaw1/Jaw2/Hopper/CoinSpout/SellPad etc.) + EconomyService (Server ORDER: Structure, PlayerState, Economy, Mission, Tool): Touched on SellPad, 1.5 s debounce, clearBag then addMoney, fires SellFX(userId,count,value,crusherId) to seller + players within 120 studs. Verified: 12 bricks → $36, bag 0; re-touch with empty bag → no payout; 5 planks → +$5; zero errors. Not verified by eye: Crusher looks; SellFX has no client listener yet.
- P2 task 3 done (2026-11-05): HUD money label (top-left, 0.4 s ticker), bag pill (bottom-centre, fill blue/orange-pulse ≥90%/red full), toasts (Notify keys via `Theme.Toasts`: BagFull, TooTough) built in StarterGui.HUD (MoneyLabel, BagPill{Fill,BagText}, Toasts{ToastTemplate}); new Theme styles MoneyLabel/BagPill/BagFill/BagText/Toast + Theme.Bag/Timing/Toasts. Verified by screenshot with forced attributes + Notify: all render, zero errors. Not verified: BagFull toast from a real full-bag break (notify path in StructureService untested end to end).
- P2 task 4 done (2026-11-05): SellFX client in FXController (FX.sell tuning; new silent SFX slots sell_chomp, sell_coins in Config/Assets). Rubble (pooled debris) arcs pad→hopper, jaws chomp 2–5× (local tween of Jaw1/Jaw2), coin spark burst from CoinSpout, +$ floater (BillboardGui, local player only), small shake. Verified in playtest by firing SellFX (20 bricks, $1,234,567): 13 pieces in flight, jaws moved 1.2 and returned home, floater text correct, zero errors. Not verified by eye (no screenshot): look/timing; coin burst direction (+Z).
- P2 task 5 done + DoD passed (2026-11-05): GuideController (client ORDER ... Viewmodel, Guide, UI; Beam on Terrain attachments, tuning in Theme.Guide) shows a yellow beam player→nearest `Crusher` while BagCount>=BagCap. E2E real-remote test: 22 UseTool hits filled bag 20/20, BagFull Notify fired, beam on, bag stayed 20 on extra hits; sell on pad → $25 for 20 blocks, bag 0, beam off, repeat touches paid nothing; zero errors from our scripts. Not verified by eye: beam look.
- Crusher resized (2026-11-05, user request): model scaled to 0.6 (~9.5 studs tall, hopper rim ~7.1); SellPad resized to 9×6 at z≈-2.9 in front of the jaws; PadStripes removed. Not retested in play; eye height ~6.7 so rim sits slightly above eye (try 0.5 if the top is still hidden).
- P2 assets integrated (2026-11-05): sell_chomp, sell_coins, ui_click (slot only, no buttons yet; wire in P4 shop), new sell_clang (jaws clang, played with each chomp). All 4 pools 6/6 loaded in playtest, SellFX fired, zero errors. Not verified by ear.
- Next step: P3 Persistence (user action: enable Studio Access to API Services).
- break_roof reuses the brick break sound (user). Sound pools are now created and preloaded at FXController.start (fixes the first-hit audio delay the user reported); all 9 pools 6/6 loaded before the first hit.
- 1-stud switch verified (2026-11-05): RunTests 11/11, 1,032 parts spawn, real-click breaks + debris OK, zero errors. The P1 DoD numbers above were measured on the old 682-block 2-stud hut.
- User checked 1-stud blocks: arms look fine (no viewmodel change).
- Waiting on user: confirm the first-hit audio delay is gone.
- Open asset requests: tooTough deferred to P7.

## 20. Decision log & notes (Sonnet appends; keep entries one line)
- 2026-11-03 — Plan created (Opus). Interview decisions D1–D11 recorded in §2.
- 2026-11-04 — Added FTUE + settings menu (P6) and P12 Launch prep (badges, leaderboards, analytics, update shutdown, server size, Stuck button, low-end check, debug cleanup, user launch checklist). Monetization → P13, helicopter → P14.
- 2026-11-04 — References saved in `reference/`; SmoothPlastic only; two-tone ground; `Config/Theme` as the single style source.
- 2026-11-04 — From the user's reference screenshots: mouse unlock (V), AoE preview, 3 s lingering rubble, crack overlays as the standard damage look, patterned noise colours, HUD timer pill.

- 2026-11-04 — P0: place published (placeId 106030102652542). Lighting.Technology no longer exists in current Studio → dropped from §13 requirements.
- 2026-11-04 — Theme lighting tuned: Brightness 2, Atmosphere Haze 0.3 (was washed out); EnvironmentSpecularScale 0, DiffuseScale 0.5 set in Lighting.
- 2026-11-04 — Materials config is `{byId, byUid}`; uid (u8) is the compact id for part attribute `M` and BlockFX. Upgrade tracks live in Config/Upgrades.tools[toolId], not in Tools rows.
- 2026-11-04 — Mouse unlock: CameraController BindToRenderStep at Camera+1 sets MouseBehavior=Default while unlocked; locked = untouched engine LockFirstPerson. Locked cursor hidden via UIS.MouseIcon = Theme.HIDDEN_CURSOR (built-in rbxasset://textures/Blank.png, fully transparent); "" when unlocked. Modal-button and MouseIconEnabled=false approaches dropped (cursor jitter in Studio, V didn't free it). MouseLockChanged BindableEvent drives the hint.
- 2026-11-04 — Edit-mode `require` caches modules: before RunTests, swap each Shared/Tests ModuleScript for a fresh clone (or run the tests in a playtest).

- 2026-11-04 — P0 DoD passed (user verified mouse lock). Studio MCP studio_id changes when the place is reopened: call list_roblox_studios each session.
- 2026-11-04 — P1: BlueprintKit is a builder (`BlueprintKit.new(sx,sy,sz,sections)`, `:box/:floor/:hollowBox/:gableRoof/:windowRow/:door/:fillMaterial/:build`); later calls overwrite; mat=nil clears; build() sorts bottom-first.
- 2026-11-04 — P1: StructureService API works on a structure (instance.structures[1]), not the instance: `build(instance, blueprintId, origin)`, `applyDamage(structure, attacker, hits, toolId)`, `destroy`. Section collapse lock = `structure.lockedSections[section]` (not per-cell `locked`). Cross-service calls via `StructureService.hooks` {canFit, addToBag, onBroken}, overwritten by PlayerStateService (P2) / MissionService (P5).
- 2026-11-05 — User change: block size 4 → 2 studs (Grid.CELL_SIZE = 2). TestHut redrawn at the same 32×24-stud footprint (682 blocks); BlueprintKit:door now takes width/height. 4× more blocks per area → P6 economy pass must rescale material values/bag caps; P7+ block counts in §8.4 are still 4-stud estimates (≈×4); sledge radius in cells now covers half the studs, revisit in P4.
- 2026-11-05 — Team Create confirmed: edit-mode DataModel has a NetworkClient with a replicator (only present in a Team Create session). Check: `game:FindFirstChildOfClass("NetworkClient") ~= nil` in Edit.
- 2026-11-05 — P1: cooldown ring is a ring that shrinks onto the dot and fades (TweenService), not a radial sweep (radial needs half-mask frames; revisit in P6 polish if wanted). Hands attack on click only (no hold-repeat), per §6.1 input="click". ToolService checks cooldown before membership, so spam at a just-broken part logs as `cooldown` then `notMember`.
- 2026-11-04 — MCP execute_luau in a playtest has its own require cache: test live services by parenting a temporary Script to ServerScriptService that prints results, then read the console.

- 2026-11-05 — P1 FXController: shake is a position-only camera offset (default camera rebuilds position each frame, so no drift); break shake fires for every break record incl. other members' (filter by attackerSlot in P8). Pool exhaustion only recycles oldest (no particles-only fallback needed since recycling always succeeds). Hit spark + hit sound play locally on ToolController.Attacked; break/tooTough sounds come from BlockFX. Cracks find the part client-side with GetPartBoundsInBox at the cell centre (cosmetic only). Character collision group set in MissionService (moves to PlayerStateService in P2 if it fits better).

- 2026-11-05 — P1 viewmodel: arms = orange sleeves (Theme action) + yellow gloves (Theme primary), from the reference arms style; "procedural spring" implemented as exponential smoothing for sway plus piecewise swing curves (no spring sim). Punches alternate only when the model has both `Right` and `Left` sides.

- 2026-11-05 — P1 DoD test harness: client fires UseTool while pinning the HRP via Heartbeat CFrame (an Anchored HRP set on the client doesn't replicate → all `range` drops); remaining `range` drops (591) were teleport-replication lag, every block retried and broke. Long tests: task.spawn the loop on the client and poll `_G` stats (MCP calls time out ~60 s).
- 2026-11-05 — P1 assets from user: new SFX slot `miss` (Config/Assets) played by FXController.onAttacked when the attack hits no block (also world parts like the ground). hit_brick and hit_roof share one sound. break_brick clip is 3.1 s (long tail; fine while pooled ×6).
- 2026-11-05 — User: 4 crack stages (FX.crackStages 4, crackDarken 0.08 → 0.06), stage rule in §5.5. User gave Decal ids (asset type 13); Textures need the Image id → resolved via InsertService:LoadAsset(decalId) → Decal.Texture. Always check AssetTypeId when the user gives image ids.
- 2026-11-05 — User change: block size 2 → 1 stud (8 per old block), TestHut smaller and human-scale (21×15 footprint, 9-stud walls, 1 cell thick → 1,032 blocks). Scaled with it: FX.colorNoise.scale 0.18 → 0.09 (same blotch size in studs), debris 0.4–0.8, shard 0.1, dust 0.6–1.5, collapse clump 4 cells. Economy origin (-10.5, 0.8, -43.5). Tool ranges (studs) unchanged; sledge r=1.5 cells is now tiny → set in studs-equivalent in P4. §8.4 block counts are still 4-stud estimates.

### Ideas/backlog (not scheduled)
- Hungry Rock Monster crusher variant (P13 cosmetic).
- Rebirth/prestige, daily contracts, leaderboards (stats already saved).
- 2026-11-05 — P2 assets: added `sell_clang` slot (user request), played at each jaw close alongside sell_chomp; `ui_click` slot added, wiring deferred to first UI buttons (P4).
