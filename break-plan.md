# BREAK STUFF — Game Specification & Implementation Plan

Single source of truth for building the game. Implementer: Claude Sonnet via Roblox Studio MCP. Section refs like **§6.3** point inside this file. Decisions marked **[D]** are final unless the user changes them; **[A]** are assumptions (may be revised without asking if playtesting proves them wrong; log the change in §20).

---

## 0. How to use this file

- Read §0–§3 and the current phase in §17 before each work session. Read other sections when the task touches them.
- Work through §17 phases in order. Do not start a phase before the previous phase's Definition of Done passes.
- Keep §19 (checklist) and §20 (decision log) up to date. They are the only parts of this file the implementer edits routinely. Change the spec only when the user approves, and log it in §20.
- The Studio place is the source of truth for code and assets. This repo holds the plan. **[D]** No Rojo or file sync: all scripts, UI and objects are created and edited in Studio through MCP.

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

No visual reference image exists in the repo. **[A]** The design bible (§13) is derived from the brief's description: colourful, chunky, low-poly/voxel simulator style.

---

## 3. Core loop & moment-to-moment feel

1. Spawn at HQ (hub). Mission Board → pick a contract → fade → arena.
2. Aim at a block (centre crosshair) → attack → instant local swing, hit spark and sound. The server confirms the break → block pops, debris cubes fly, dust puffs, small screen shake, a "+$" chip floats up and the bag counter ticks.
3. The phase progress bar fills toward **COLLAPSE**. At 100%: 1 s warning rumble → the section collapses (physics show) → banner and phase bonus.
4. Bag full → warning plus arrow → walk to the arena Crusher → step on the pad → rubble flies into the hopper, the jaws chomp, coins burst, money ticks up.
5. Buy tools and upgrades at a shop counter (in hub and arena). Feel the difference immediately.
6. Final phase collapses → results card (time, stars, earnings, unlock) → Replay or Back to HQ.

Readability rules: money, bag (count/cap), equipped tool, and objective (phase k/N plus progress bar) are always visible. A target block under the crosshair gets a subtle outline (SelectionBox), turning red if the tool tier is too low.

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
      Materials, Tools, Upgrades, Economy, Missions, Assets, FX
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
    UIController               HUD, shop, mission board, results, toasts
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

## 5. Destruction system

### 5.1 Grid model **[D]**
- Block size **4×4×4 studs** (chunky, fewer parts). Structures are axis-aligned on a grid anchored at an `origin` CFrame (position only, no rotation in the MVP).
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
Part properties: `Anchored=true, CanCollide=true, CanTouch=false, CanQuery=true, CastShadow=` config `FX.blockShadows` (default true, set false for structures >3000 blocks), `Material=SmoothPlastic` (or per-material override), `Color = material.color` with deterministic ±5% brightness jitter from a hash of the cell index (voxel richness at zero cost), `Name="B"`, attribute `M` = matId (uint) for client FX and prediction. Parent: `Workspace.Missions.<instanceId>.Structure`.

Spawning: batched at ≤300 parts per Heartbeat, **bottom layer first**, so construction reads as the structure "rising" (with a client dust puff per layer row; optional). Never `Instance.new` thousands of parts in one frame.

### 5.3 Materials (Config/Materials) **[A] values are placeholders for balancing**
| id | name | hp | value $ | minTier | color | notes | phase |
|---|---|---|---|---|---|---|---|
| `plank` | Planks | 1 | 1 | 0 | `#C8823C` | splinter debris | MVP |
| `roof` | Roof Tile | 1 | 1 | 0 | `#D9534F` | | MVP |
| `glass` | Glass | 1 | 2 | 0 | `#9FE3FF`, Transparency 0.35, Material Glass | shatter: shards, no cubes | MVP |
| `brick` | Brick | 2 | 3 | 0 | `#B5452E` | hands 2 hits, pickaxe 1 | MVP |
| `concrete` | Concrete | 4 | 6 | 1 | `#B8B8B0` | | P7 |
| `steel` | Steel Beam | 10 | 15 | 3 | `#6E7F8E`, Material Metal | sparks, clang | P7 |
| `gold` | Vault Gold | 20 | 100 | 3 | `#FFD23F`, Material Neon | coin burst | P7 |
| `tnt` | TNT Crate | 1 | 0 | 0 | `#E8322E` | special: explodes, radius 2 cells, dmg 6 | P7 |

Schema: `{id, name, hp, value, minTier, color, material?, transparency?, debris="cubes"|"shards"|"sparks", soundSet, onBreak=nil|"explode"}`. `onBreak` hooks are dispatched by a small table in StructureService. New special blocks = new hook functions, no core changes.

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
- Hits on multi-hit blocks: the client darkens the block colour by `(1-hpFrac)*0.35` and adds a crack Texture once the crack asset is approved (placeholder: darkening only).
- Budget: ≤ 2 KB/s per client in normal play; rocket bursts may spike to ~1 KB per event. Profile in P6.

### 5.6 Client feel & prediction
- On attack input the client immediately plays: swing anim, hit spark at `hitPos`, hit sound. It does not wait for the server.
- **Optimistic break** (P6): if `material.hp` (or the last known hp from BlockFX) ≤ predicted damage, AND the tier is OK, AND the local `BagCount < BagCap`, then hide the part locally (`LocalTransparencyModifier=1`, set `CanQuery=false` locally) and play the full break FX right away. Record it as pending. If no server break record arrives within 0.6 s, restore the part (rare). Server break records for already-predicted cells skip FX.
- Debris (FXController): **client-only** pooled cube Parts (pool of 150, size 0.8–1.6, material colour, `CollisionGroup="Debris"`, which collides with world/structure only, not players or other debris). Per break: 4–6 cubes with outward random impulse, spin; lifetime 1.2–2.0 s; shrink tween then return to pool. If the pool is exhausted: particles only. Dust: one pooled `ParticleEmitter` per material colour at a reused attachment, `:Emit(n)` with built-in texture `rbxasset://textures/particles/smoke_main.dds`.
- Camera: break shake 0.15 amplitude/0.12 s; collapse shake 1.0/1.2 s; FOV kick +3° on heavy hits. Shakes are additive with a cap. Respect a settings toggle (P6).
- Sounds: pooled `Sound` objects per sound key (max 6 concurrent per key, random pitch 0.92–1.08). Missing asset ids = silent (§14).

### 5.7 Performance plan **[D]**
| Risk | Mitigation |
|---|---|
| Part count | 4-stud blocks; hollow blueprints (walls 1 thick, floors), MVP ≤ 1,500 blocks/structure, later ≤ 6,000 spawned. For solid/large structures (P6/P7 if profiling demands): **surface-only spawning**. A cell only gets a Part when at least one of its 6 neighbours is empty. When a block breaks, spawn Parts for newly exposed neighbours. The grid already holds every cell, so this is a local change in StructureService. |
| Spawn hitches | ≤300 parts/frame batching (§5.2). |
| Network | Per-frame batched buffers, members only, no attributes per block, no physics parts replicated for FX. |
| Physics | Intact blocks anchored. All debris and collapse chunks client-only, pooled and capped. Server simulates nothing physical for destruction. Projectiles are kinematic raycasts (§6.4). |
| Collapse | Chunk collapse cells into 2×2×2-cell clumps (8-stud cubes). Cap 150 physics bodies per collapse; overflow becomes dust bursts plus a quick shrink. Bodies fade out by 4 s. |
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
| 1 | Grandma's Shed | ~350 | 2 (roof → rest) | planks, roof, glass, brick chimney | MVP |
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
    stats={ blocks=0, earned=0, playTime=0 }, settings={ shake=true },
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

**Palette** (Config/FX + UI):
- World: grass `#6BC04B`, grass dark `#4E9A36`, dirt `#8B5A2B`, sand `#E8C98A`, sky `#8FD3FF`, rock `#9AA3AE`.
- Materials: §5.3 colours. Each material must contrast with the arena ground.
- UI: primary yellow `#FFC93C`, action orange `#FF8A00`, money green `#3DDC84`, danger red `#FF4D4D`, info blue `#3BA7FF`, panel cream `#FFF6E0`, outline/ink `#2B2B3A`.
- Brand stripes (machines/HQ): yellow `#FFC93C` / ink `#2B2B3A` hazard stripes.

**Environment**: parts-based chunky terrain (no smooth Terrain), plateau steps of 4 studs, blocky trees (brown trunk plus 2–3 green cubes), fences, cones, barrels. Lighting: `Lighting.Technology=ShadowMap`, bright midday, Atmosphere density 0.25 with a light blue haze, ColorCorrection saturation +0.15 / contrast +0.05, subtle Bloom (intensity 0.4, threshold 0.95). No realism, no grime.

**Structures/blocks**: 4-stud cube grid, SmoothPlastic, ±5% brightness jitter, readable silhouettes (gable roofs, big windows, chimneys, flags). Details are made of blocks too (a door = 1×2 cells of a different colour). Every structure has at least 3 materials for visual and economic variety.

**Equipment**: oversized, toy-like proportions (head 1.5× realistic), 2–3 colours each, bold primary accent (pickaxe: wood handle, grey head with yellow band; sledge: red head; chainsaw: orange body; rocket: green tube with yellow stripes; laser: white/cyan sci-fi). Built from Parts first. Meshes only via §14.

**Machines**: Crusher = big, hazard-striped, exaggerated moving parts. All machines show their function through motion (jaws, pistons, conveyors).

**UI**: Font `Enum.Font.FredokaOne` (headings/numbers), `GothamBold` (small text). Rounded panels (UICorner 12), thick ink UIStroke (3 px), drop-shadow offset frame, bright header bands. Sizes in Scale plus `UIAspectRatioConstraint`. A `UIScale` per ScreenGui (mobile later). Buttons: 3D-pressed look (darker bottom band), hover scale 1.05, click scale 0.95. Icons: flat, chunky, ink-outlined (placeholders = coloured squares with text until the user supplies icons). Hierarchy: money top-left (big), objective top-centre, bag bottom-centre above the hotbar, toasts mid-top, shop/board as centre modals.

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

Expected batches (ask only when the feature is ready): P1 hit/break SFX per material; P2 crusher/coin/UI SFX; P4 tool swing SFX + optional icons; P5 collapse rumble, mission music, victory jingle; P6 crack texture, optional meshes for tools/crusher; later per phase.

---

## 15. MVP definition (end of P5) **[D]**
Included: first-person camera and viewmodel; hub plus 1 arena; mission 1 (Grandma's Shed, ~350 blocks, 2 phases with collapse, completion, replay, stars); materials plank/roof/glass/brick; tools hands/pickaxe/sledge with 2 upgrade tracks each; bag with capacity upgrades; one Crusher model (in hub and arena); shop UI; mission board UI; HUD (money, bag, objective, hotbar, toasts); server-validated everything; DataStore persistence; debris/dust/shake FX; asset slots registered (sounds as the user supplies them).

Excluded until later: other tools, materials and missions; co-op; reserved servers; mobile; monetization; vehicles; settings menu beyond the shake toggle; leaderboards; daily jobs; rebirth.

**Scope guard**: if a task is not in the current phase, do not build it. Note ideas in §20 under "Ideas/backlog" instead.

---

## 16. Testing approach (MCP)

- **Before editing**: check Studio state; make sure **play mode is stopped** (edits during play are lost). Inspect the tree before creating to avoid duplicates.
- **Edit**: create Scripts/ModuleScripts/LocalScripts and UI via MCP (execute Luau in edit context or the script-edit tools). Prefer targeted edits over full rewrites. Re-read a script after editing it.
- **Unit self-tests**: `ServerScriptService/Tests/RunTests` requires pure modules (Grid, Packer, ToolStats, economy cost formulas, data sanitize, phase progress math) and asserts. Run via MCP execute Luau in edit mode after changing those modules. All must pass.
- **Playtest**: start play via MCP, read console output (zero errors/warnings from our scripts), take screen captures to check visuals, simulate input (keyboard/mouse) where possible, stop play, iterate.
- **Multi-client** (P8+): Studio "Local Server + N players" may not be drivable through MCP. If not, give the user a short manual test script (steps and expected results) and ask them to run it and report.
- **Perf** (P6+): stress blueprint `StressCube` (20×15×20 hollow = ~3,000 blocks; and a solid 5,000 variant in P6). Check frame time, part count, network receive (Shift+F3 / MicroProfiler); record numbers in §20.
- After each phase: the user saves/publishes the place (Sonnet reminds them). A phase is done only after a clean full playtest.

---

## 17. Implementation roadmap

Each phase leaves the game playable. Assets: request only when the phase's features work (§14).

### P0 — Foundation & MCP check
- Objective: verified MCP connection, project skeleton, first-person movement in a test area.
- Features: §4.2 folders/bootstraps; Config modules with MVP data; Remotes created; Util.Grid, Util.Format, Util.Packer + Tests; collision groups (`Debris`, `Players`); first-person lock (`CameraMode=LockFirstPerson`), crosshair; basic Lighting per §13; a flat test plate.
- Depends: —. Assets: none.
- Test: self-tests pass; playtest shows first-person and crosshair, no console errors.
- DoD: skeleton exists exactly per §4.2; Studio state clean; §19 updated.

### P1 — Destruction core
- Objective: break blocks with hands; it feels good.
- Features: StructureService (grid, batched rising spawn, applyDamage/breakCell, BlockFX batching); `TestHut` blueprint (~150 blocks) plus BlueprintKit; a temporary dev MissionInstance auto-created for the player at the test plate; ToolService validation (§6.3) for hands; ToolController targeting, outline, cooldown ring; viewmodel arms with punch anim; FXController debris pool, dust, shake; hit/break SFX slots (silent).
- Depends: P0.
- Test: break all blocks; spam-click doesn't exceed the cooldown (server log counter); invalid remote args (fire from command bar) are dropped; ~150 breaks leave no leaked debris parts (pool count stable).
- DoD: breaking feels responsive; no errors; then **ask asset batch: hit/break sounds per material**.

### P2 — Bag, Crusher, money, HUD
- Features: PlayerStateService (attributes, bag ops); D3 full-bag clamp, toast and guide beam; Crusher placeholder model plus sell logic and SellFX; HUD (money, bag bar, toasts).
- Depends: P1.
- Test: fill bag → can't break + warning; sell → bag 0, money = expected sum; stand on pad repeatedly → no double payout.
- DoD: complete earn loop without persistence. Ask batch: crusher/coin/UI click SFX.

### P3 — Persistence
- Features: DataService §11; load/save of money/bag/cap/tools/missions/stats; BindToClose; priority saves.
- Depends: P2. **User action**: publish the place plus enable API access (ask).
- Test: earn money, stop, replay → money restored; sanitize tests (corrupt table → valid data); simulate a save failure (temporary forced error) → retries plus kept dirty.
- DoD: data survives sessions; no saves per hit (count saves in log).

### P4 — Tools, shop, upgrades
- Features: Config for hands/pickaxe/sledge plus upgrades/capacity; ToolStats; hotbar (1–3, wheel) plus EquipTool; viewmodels for pickaxe/sledge from Parts with swing anims; sledge sphere AoE; Shop UI (tools tab, upgrades tab, capacity), ShopCounter prompt; EconomyService purchases.
- Depends: P3.
- Test: buy each item; insufficient funds rejected; cannot buy owned; sledge AoE breaks the expected cells exactly once and rewards are correct; upgrade effects measurable (cooldown/radius); data persists.
- DoD: three distinct-feeling tools. Ask batch: swing SFX per tool, optional tool icons.

### P5 — Hub, mission flow, phased collapse (= MVP)
- Features: hub layout §9.1 and arena §9.2; Mission Board UI; MissionService (instance lifecycle, StartMission/Leave/Exit, fade travel); `Shed` blueprint (2 sections); phase progress, warn, collapse (§8.1), CollapseFX client show; phase bonus; results card, stars, unlock, replay rebuild; objective HUD; remove the dev auto-instance from P1.
- Depends: P4.
- Test: full run hub → mission → 2 collapses → results → replay → hub; progress math matches the self-test; leaving mid-mission cleans up the instance and parts; data records stars/bestTime.
- DoD: **MVP complete per §15**, a 15-minute clean playtest by Sonnet plus a user playtest request. Ask batch: collapse rumble, victory jingle, hub/mission music, ambient.

### P6 — Feel, performance, balance
- Features: optimistic break prediction §5.6; crack darkening (+crack texture if supplied); FOV kick; shake setting toggle; stress blueprints; profiling; surface-only spawning if 5k solid > budget; economy pacing check vs §7.4; DEBUG money/min logging.
- Test: perf numbers recorded; pacing within ±30% of targets or values adjusted and logged.
- DoD: 60 FPS on the stress test on the dev machine; feels instant at 150 ms simulated latency (Studio network sim if available). Ask: optional meshes for tools/crusher.

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

### P12 — Monetization (fair)
Cosmetic tool skins, crusher skins (Rock Monster!), debris colour themes, VIP cosmetic badge; optional dev product "2× money for 15 min" (non-essential). MonetizationService with idempotent `ProcessReceipt` (receipt ids in profile). Never sell power that gates content.

### P13 — Helicopter (far future)
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
- [ ] P0 Foundation & MCP check
- [ ] P1 Destruction core — assets: hit/break SFX [ ]
- [ ] P2 Bag, Crusher, money, HUD — assets: crusher/coin/UI SFX [ ]
- [ ] P3 Persistence (user: publish + API access [ ])
- [ ] P4 Tools, shop, upgrades — assets: swing SFX, icons (opt) [ ]
- [ ] P5 Hub, missions, phased collapse → **MVP** — assets: collapse, music, jingle [ ]
- [ ] P6 Feel, performance, balance
- [ ] P7 Content expansion I
- [ ] P8 Co-op in-server
- [ ] P9 Isolated squad servers
- [ ] P10 Content expansion II
- [ ] P11 Mobile
- [ ] P12 Monetization
- [ ] P13 Helicopter

Current phase: **P0**. Next step: verify the MCP connection and inspect the place.

## 20. Decision log & notes (Sonnet appends; keep entries one line)
- 2026-11-03 — Plan created (Opus). Interview decisions D1–D11 recorded in §2.

### Ideas/backlog (not scheduled)
- Hungry Rock Monster crusher variant (P12 cosmetic).
- Rebirth/prestige, daily contracts, leaderboards (stats already saved).
