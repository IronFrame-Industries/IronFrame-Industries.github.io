# Horde Survival Template

A production-ready, data-driven foundation for **Vampire Survivors / Brotato-style** horde survival games in Unity.
Built for mass-entity performance: thousands of pooled enemies, projectiles and damage numbers with zero
per-frame garbage.

- Unity **2022.3 LTS or newer**. Verified on **Unity 6000.6** (compiles with zero warnings; demo scene play-tested).
- 2D top-down (Physics2D). No art assets required; works with placeholder sprites.
- Dependencies: **TextMeshPro** (ships with Unity's UI package). Optional: Input System package.

## Quick start (60 seconds)

1. Import the template. Import TMP resources if prompted (`Window ▸ TextMeshPro ▸ Import TMP Essential Resources`).
2. Menu **Horde Survival ▸ Create Demo Scene**.
3. Press **Play**, click **START**. WASD / arrows to move, Esc to pause.

The builder creates layers, sample weapons / passives / enemies, a difficulty profile, prefabs and a full UI under
`Assets/HordeSurvivalTemplate/Demo`. Everything it makes is ordinary data you can inspect, tweak or delete, and
`Scripts/Editor/DemoSceneBuilder.cs` doubles as a worked example of wiring every system together.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Setup & Installation](#2-setup--installation)
3. [Folder & Script Map](#3-folder--script-map)
4. [Tutorial: Add a Custom Item (Weapon / Passive)](#4-tutorial-add-a-custom-item)
5. [Tutorial: Add a Custom Enemy](#5-tutorial-add-a-custom-enemy)
6. [Tutorial: Create a Custom AI State](#6-tutorial-create-a-custom-ai-state)
7. [Tutorial: Create a Custom Weapon Behaviour](#7-tutorial-create-a-custom-weapon-behaviour)
8. [Save / Load](#8-save--load)
9. [Performance Notes](#9-performance-notes)
10. [FAQ](#10-faq)

---

## 1. Architecture Overview

The template is built around three rules:

1. **Data lives in ScriptableObjects.** Items, weapons, enemies, player stats, difficulty pacing and the
   upgrade database are all assets you create from `Assets ▸ Create ▸ Horde Survival ▸ …`.
2. **Systems talk through events, never references.** `GameEvents` (static, typed C# events) is the only
   thing most systems know about. UI subscribes; it never polls.
3. **Nothing is instantiated or destroyed during gameplay.** Enemies, projectiles, XP gems, popups and VFX
   all go through `PoolManager`.

### How the systems talk

```
                       ┌────────────────────┐
     UI buttons ──────▶│  GameManagerBase   │── OnGameStateChanged ──▶ UIManager ──▶ UIViews
                       │ (state + timeScale)│
                       └─────────▲──────────┘
                                 │ EnterLevelUp / ResumeGameplay / EndRun
                                 │
┌──────────────┐  OnPlayerLevelUp  ┌──────────────────────┐  OnLevelUpChoicesReady  ┌────────────┐
│PlayerCharacter│────────────────▶│ LevelUpChoiceSystem  │───────────────────────▶│ LevelUpView│
│ Stats/XP/Inv │◀────────────────│ (rolls from database)│◀───────────────────────│  (cards)   │
└──────▲───────┘ Inventory.Add    └──────────────────────┘    OnUpgradeChosen      └────────────┘
       │
       │ reads StatSheet + Inventory
┌──────┴────────┐  fires   ┌────────────────┐  spawns   ┌─────────────┐  hits  ┌──────────────┐
│WeaponController│────────▶│ WeaponBehavior │──────────▶│ PoolManager │───────▶│EnemyController│
│ (cooldowns)   │          │ (SO strategy)  │           │             │        │ (FSM + Health)│
└───────────────┘          └────────────────┘           └──────▲──────┘        └──────┬───────┘
                                                               │                      │ OnEnemyKilled
┌──────────────┐  curves   ┌────────────────┐  Spawn()         │                      ▼
│DifficultyData │─────────▶│  EnemySpawner  │──────────────────┘              HordeGameManager (kills)
└──────────────┘           │ (reads GameTimer)│                                 HUDView (kill counter)
                           └────────────────┘
```

### Key classes

| Layer | Class | Responsibility |
|---|---|---|
| Core | `GameManagerBase` / `HordeGameManager` | Abstract state machine (`MainMenu, Gameplay, Paused, LevelUp, GameOver, Victory`), time scale, run stats, scene loading. |
| Core | `GameEvents` | Static typed event bus. Allocation-free. |
| Core | `GameEvent` / `GameEventListener` | ScriptableObject event channels for designer-wired reactions (audio, VFX). |
| Pooling | `PoolManager`, `ObjectPool`, `PooledObject`, `IPoolable` | Prefab pools keyed by prefab; prewarm; O(1) get/release; reset hooks. |
| Data | `ItemData` → `WeaponData`, `PassiveItemData` | Static item definitions with per-tier values. |
| Data | `WeaponBehavior` (+ `ProjectileWeaponBehavior`, `AreaWeaponBehavior`) | Strategy assets describing *how* a weapon attacks. |
| Data | `EnemyData`, `PlayerStatsData`, `DifficultyData`, `UpgradeDatabase` | Enemies, character base stats, pacing curves, level-up catalogue + rarity weights. |
| Stats | `StatSheet`, `StatModifier` | Modifier pipeline: `(base + Σflat) × (1 + Σ%add) × Π(1 + %mult)`, cached per stat. |
| Inventory | `PlayerInventory`, `ItemInstance` | Runtime items, tiers, duplicate handling, slot limits, keeps `StatSheet` in sync. |
| Combat | `Health`, `IDamageable`, `DamageInfo`, `Projectile`, `DamagePopup` | Damage pipeline, piercing projectiles, pooled popups. |
| Player | `PlayerCharacter`, `PlayerController`, `WeaponController`, `ExperienceGem`, `InputReader` | The player entity. |
| Enemies | `EnemyController`, `EnemyRegistry`, `EnemySpawner`, `GameTimer` | Pooled FSM enemies, fast nearest/radius queries, time-based spawning and victory. |
| AI | `StateMachine`, `BaseState`, `AIController`, `IdleState`, `ChaseState`, `AttackState` | Reusable lightweight FSM. |
| Systems | `LevelUpChoiceSystem` | Level-up → pause → weighted roll → UI → apply → resume. Queues multiple level-ups. |
| Save | `SaveManager`, `ISaveable`, `MetaProgression` | JSON + obfuscation, atomic writes, backup and corruption recovery. |
| UI | `UIManager`, `UIView`, `HUDView`, `MainMenuView`, `PauseMenuView`, `GameOverView`, `LevelUpView` | State-driven screens; all updates are event-driven. |

### Game flow in one run

1. `MainMenuView` ▸ Start → `GameManagerBase.StartRun()` → state `Gameplay`, `OnRunStarted`.
2. `PlayerCharacter` rebuilds its `StatSheet` + `PlayerInventory` from `PlayerStatsData` (starting items included).
3. `EnemySpawner` samples `DifficultyData` curves against `GameTimer` and spawns pooled enemies off-screen.
4. `WeaponController` ticks weapon cooldowns and calls each `WeaponBehavior.Fire()` with a `WeaponFireContext`
   (weapon tier × player stats).
5. Enemies die → `OnEnemyKilled` → XP gem spawned → `PlayerCharacter.AddExperience` → `OnPlayerLevelUp`.
6. `LevelUpChoiceSystem` enters state `LevelUp` (time frozen), rolls choices from `UpgradeDatabase`, broadcasts
   `OnLevelUpChoicesReady`. `LevelUpView` shows cards; a click raises `OnUpgradeChosen`; the system applies it
   and resumes.
7. Player death → `OnPlayerDied` → `HordeGameManager.EndRun()` → `GameOver`. Surviving `VictoryMinutes` → `Victory`.
8. `MetaProgression` records the result and auto-saves through `SaveManager`.

---

## 2. Setup & Installation

### 2.1 Import

1. Copy `Assets/HordeSurvivalTemplate` into your project (or import the `.unitypackage`).
2. Make sure **TextMeshPro** essentials are imported (`Window ▸ TextMeshPro ▸ Import TMP Essential Resources`).

### 2.2 Input

The template reads input through one static class, `Player/InputReader.cs`, and supports both systems with no setup:

| Project Setting (`Edit ▸ Project Settings ▸ Player ▸ Active Input Handling`) | Behaviour |
|---|---|
| **Input Manager (Old)** | Uses `Input.GetAxisRaw("Horizontal"/"Vertical")` and `Escape`. |
| **Input System Package (New)** or **Both** | Uses `Keyboard.current` (WASD / arrows / Esc) and `Gamepad.current` (left stick / Start). |

To use an Input Actions asset with rebinding, replace the two method bodies in `InputReader` with reads from your
generated actions class. Nothing else in the template touches input.

### 2.3 Layers & Physics2D

1. Create layers **Player**, **Enemy**, **Projectile**.
2. In `Project Settings ▸ Physics 2D ▸ Layer Collision Matrix`: Projectile ↔ Enemy **on**; Projectile ↔ Projectile and
   Projectile ↔ Player **off**. Enemy ↔ Enemy on if you want them to push each other apart (recommended for a
   natural crowd), off for maximum performance.

### 2.4 Scene checklist

Create an empty scene and add these objects (one of each):

| GameObject | Components | Notes |
|---|---|---|
| `GameManager` | `HordeGameManager` | Set *Initial State* = MainMenu. Leave scene names empty for a single-scene setup. |
| `PoolManager` | `PoolManager` | Add prewarm entries for your enemy, projectile, gem and popup prefabs. |
| `SaveManager` | `SaveManager` | Optional. Persists across scenes. |
| `MetaProgression` | `MetaProgression` | Optional example saveable. |
| `GameTimer` | `GameTimer` | |
| `EnemySpawner` | `EnemySpawner` | Assign a `DifficultyData` asset and the default XP gem prefab. |
| `LevelUpSystem` | `LevelUpChoiceSystem` | Assign an `UpgradeDatabase` asset. |
| `DamagePopups` | `DamagePopupSpawner` | Assign the popup prefab. |
| `Player` | `Rigidbody2D` (Dynamic, Gravity 0, Freeze Z), `Collider2D`, `Health` (untick *Show Damage Numbers*), `PlayerCharacter`, `PlayerController`, `WeaponController` | Layer **Player**. Assign a `PlayerStatsData`. Set *Hit Mask* on `WeaponController` to **Enemy**. |
| `Canvas` | `UIManager` + child views | See below. |

**Prefabs**

- **Enemy**: `Rigidbody2D` (Dynamic, Gravity 0, Freeze Z), `Collider2D`, `Health`, `EnemyController`. Layer **Enemy**.
- **Projectile**: `Rigidbody2D` (Kinematic), `Collider2D` (*Is Trigger* ✔), `Projectile`. Layer **Projectile**.
- **XP Gem**: `ExperienceGem` + a sprite. No collider needed.
- **Damage Popup**: a world-space `TextMeshPro` + `DamagePopup`.

**UI**: under the Canvas create one panel per screen, each with a `CanvasGroup` and the matching view component.
Tick *Visible In States* on each view:

| View | Visible in |
|---|---|
| `MainMenuView` | MainMenu |
| `HUDView` | Gameplay, Paused, LevelUp |
| `PauseMenuView` | Paused |
| `LevelUpView` | LevelUp |
| `GameOverView` | GameOver, Victory |

Drag all views into `UIManager ▸ Views`. In `LevelUpView`, place 3+ `UpgradeOptionButton` cards.

### 2.5 Minimum data assets

Create via `Assets ▸ Create ▸ Horde Survival ▸ …`:

1. **Player Stats** – base stats, XP curve, starting items.
2. **Weapon Behaviors ▸ Projectile** – assign your projectile prefab.
3. **Items ▸ Weapon** – assign the behaviour; fill tiers.
4. **Enemy** – assign the enemy prefab.
5. **Difficulty Profile** – add the enemy to *Spawn Entries* with Weight 1, Group Size 1.
6. **Upgrade Database** – add every item that may appear on level-up.

Press Play, click Start.

---

## 3. Folder & Script Map

```
Assets/HordeSurvivalTemplate/Scripts/
├── Core/        GameState, GameEvents, GameManagerBase, HordeGameManager, Events/GameEvent(+Listener)
├── Pooling/     IPoolable, PooledObject, ObjectPool, PoolManager
├── Stats/       StatTypes (StatType, ModifierType, StatModifier), StatSheet
├── Data/        PlayerStatsData, ItemData, PassiveItemData, WeaponData, EnemyData, DifficultyData, UpgradeDatabase
├── Inventory/   ItemInstance, PlayerInventory
├── Combat/      DamageInfo(+IDamageable), Health, Projectile, DamagePopup, DamagePopupSpawner,
│                WeaponBehavior(+WeaponFireContext), WeaponController, TimedDespawn,
│                Behaviors/ProjectileWeaponBehavior, Behaviors/AreaWeaponBehavior
├── Player/      InputReader, PlayerController, PlayerCharacter, ExperienceGem
├── Enemies/     EnemyRegistry, EnemyController, EnemySpawner, GameTimer
├── AI/          BaseState, StateMachine, AIController, States/IdleState, ChaseState, AttackState
├── Systems/     LevelUpChoiceSystem
├── Save/        ISaveable, SaveManager, MetaProgression
└── UI/          UIView, UIManager, RarityColors, Views/MainMenuView, HUDView, PauseMenuView,
                 GameOverView, LevelUpView, UpgradeOptionButton
```

---

## 4. Tutorial: Add a Custom Item

### 4.1 A passive item ("Iron Skin": +Armor and +Max Health)

1. `Assets ▸ Create ▸ Horde Survival ▸ Items ▸ Passive Item`. Name it `IronSkin`.
2. Fill **Id** (`iron_skin`), **Display Name**, **Description**, **Rarity** = Rare, optional **Icon**.
3. Under **Per-Tier Modifiers**, add 3 entries. Each tier lists the *total* bonus at that tier:

   | Tier | Modifiers |
   |---|---|
   | 1 | Armor, Flat, 1 · MaxHealth, PercentAdditive, 0.10 |
   | 2 | Armor, Flat, 2 · MaxHealth, PercentAdditive, 0.20 |
   | 3 | Armor, Flat, 3 · MaxHealth, PercentAdditive, 0.30 |

   `MaxTier` is set automatically from the number of tiers.
4. Add the asset to your **Upgrade Database ▸ Items**. Done – it will now appear as a level-up card and apply
   through the `StatSheet` pipeline automatically.

### 4.2 A weapon ("Fan of Knives")

1. Create a projectile prefab (see §2.4) and a `Weapon Behaviors ▸ Projectile` asset:
   *Projectile Prefab* = your prefab, *Targeting* = AimDirection, *Spread Angle* = 40.
2. `Assets ▸ Create ▸ Horde Survival ▸ Items ▸ Weapon`. Assign **Behavior** = the asset above.
3. Fill **Per-Tier Stats**:

   | Tier | Damage | Cooldown | Projectiles | Pierce | Area | Speed | Duration |
   |---|---|---|---|---|---|---|---|
   | 1 | 8 | 1.0 | 3 | 0 | 1 | 14 | 1.5 |
   | 2 | 8 | 1.0 | 5 | 1 | 1 | 14 | 1.5 |
   | 3 | 12 | 0.8 | 7 | 2 | 1.2 | 16 | 1.5 |

4. Add it to the **Upgrade Database** (and/or to a character's **Starting Items**).

**How player stats combine with tiers** (see `WeaponController.FireWeapon`):

```
damage       = tier.Damage      × Stat(Damage)
cooldown     = tier.Cooldown    × Stat(Cooldown)          // 0.8 = 20% faster
projectiles  = tier.Projectiles + Stat(ProjectileCount)   // flat
pierce       = tier.Pierce      + Stat(Pierce)            // flat
area / speed / duration = tier value × matching stat multiplier
```

### 4.3 Adding a brand-new item category

Subclass `ItemData` (add a value to `ItemKind` if it needs its own slot type), override `Kind` and
`GetTierDescription`, and handle it where it applies – e.g. a `ConsumableItemData` consumed by a new
`ConsumableController`. The inventory, database and UI already understand any `ItemData`.

---

## 5. Tutorial: Add a Custom Enemy

1. Build a prefab: sprite + `Rigidbody2D` (Dynamic, Gravity Scale 0, Freeze Rotation Z) + `CircleCollider2D` +
   `Health` + `EnemyController`. Set the layer to **Enemy**.
2. `Assets ▸ Create ▸ Horde Survival ▸ Enemy`. Assign the **Prefab**, set **Max Health**, **Move Speed**,
   **Contact Damage**, **Attack Range**, **Attack Cooldown**, **XP Reward**, and the per-minute scaling.
3. Open your **Difficulty Profile** and add a **Spawn Entry**: Enemy = the asset, Start Minute = 3, End Minute = 0,
   Weight = 2, Group Size = 1. From minute 3 it is rolled against other active entries by weight.
4. Optional: add the prefab to `PoolManager ▸ Prewarm Pools` with a count of ~100 to avoid first-spawn hitches.

Enemy health and damage scale automatically with `HealthScalingPerMinute` / `DamageScalingPerMinute`.
Spawn rate, burst size and the alive cap are the three curves on `DifficultyData`.

To give an enemy *different behaviour* (ranged, exploding, fleeing), subclass `EnemyController` and override
`CreateStates()` and/or `PerformAttack()` – see the next section.

---

## 6. Tutorial: Create a Custom AI State

The FSM is three small pieces:

- `BaseState` – abstract state with `Enter`, `Exit`, `Tick(dt)`, `FixedTick(dt)`.
- `StateMachine` – holds the current state; `ChangeState()` calls Exit/Enter.
- `AIController` – MonoBehaviour that owns a `StateMachine`, the `Target`, movement helpers
  (`MoveTowards`, `Stop`, `DirectionToTarget`, `SqrDistanceToTarget`) and the numbers states read
  (`MoveSpeed`, `AggroRange`, `AttackRange`, `AttackCooldown`).

States are plain C# objects created **once** per controller and reused across pool recycles, so reset your
fields in `Enter()`.

### Example: a `FleeState` that runs away when health is low

**Step 1 – write the state** (`Scripts/AI/States/FleeState.cs`):

```csharp
using HordeSurvival.AI;
using HordeSurvival.Combat;

public class FleeState : BaseState
{
    private readonly Health _health;
    private readonly float _recoverFraction;
    public BaseState ChaseState { get; set; }

    public FleeState(AIController controller, Health health, float recoverFraction = 0.5f) : base(controller)
    {
        _health = health;
        _recoverFraction = recoverFraction;
    }

    public override void Tick(float deltaTime)
    {
        // Leave once we have recovered or lost the target.
        if (!Controller.HasTarget || _health.Fraction >= _recoverFraction)
        {
            Controller.Machine.ChangeState(ChaseState);
        }
    }

    public override void FixedTick(float fixedDeltaTime)
    {
        // Run directly away from the target, slightly faster than normal.
        Controller.MoveInDirection(-Controller.DirectionToTarget(), Controller.MoveSpeed * 1.3f);
    }

    public override void Exit() => Controller.Stop();
}
```

**Step 2 – wire it into an enemy** (subclass `EnemyController`):

```csharp
using HordeSurvival.Enemies;

public class CowardEnemy : EnemyController
{
    private FleeState _flee;

    protected override void CreateStates()
    {
        base.CreateStates();                       // builds Idle / Chase / Attack
        _flee = new FleeState(this, Health, 0.6f) { ChaseState = Chase };
    }

    protected override void Update()
    {
        base.Update();
        // Global transition: any state → Flee when hurt. Keep checks like this cheap.
        if (Health.IsAlive && Health.Fraction < 0.3f && !Machine.IsIn<FleeState>())
        {
            Machine.ChangeState(_flee);
        }
    }
}
```

Put `CowardEnemy` on the prefab instead of `EnemyController`; everything else (pooling, registry, XP drops)
is inherited.

### Example: a ranged attack

Override `PerformAttack()` only – the Idle/Chase/Attack graph stays the same:

```csharp
public class ArcherEnemy : EnemyController
{
    [SerializeField] private Projectile _arrowPrefab;

    public override void PerformAttack()
    {
        var arrow = PoolManager.Instance.Spawn(_arrowPrefab, transform.position, Quaternion.identity);
        arrow.Launch(DirectionToTarget(), 8f, Data.ContactDamage, 0, 3f, 0f, false, 1f,
                     LayerMask.GetMask("Player"), gameObject);
    }
}
```

Set a larger **Attack Range** on the archer's `EnemyData` so it stops at a distance.

---

## 7. Tutorial: Create a Custom Weapon Behaviour

Subclass `WeaponBehavior` and implement `Fire(in WeaponFireContext ctx)`. The context already contains the
final damage, counts, area, speed, duration, crit values and the owning `ItemInstance`.

```csharp
using HordeSurvival.Combat;
using HordeSurvival.Enemies;
using HordeSurvival.Pooling;
using UnityEngine;

[CreateAssetMenu(menuName = "Horde Survival/Weapon Behaviors/Chain Lightning")]
public class ChainLightningBehavior : WeaponBehavior
{
    [SerializeField] private float _jumpRadius = 3f;

    public override void Fire(in WeaponFireContext ctx)
    {
        if (!EnemyRegistry.TryFindNearest(ctx.Origin, out EnemyController current)) return;

        int jumps = ctx.ProjectileCount;          // reuse "projectiles" as number of chain jumps
        for (int i = 0; i < jumps && current != null; i++)
        {
            float dmg = ctx.RollDamage(out bool crit);
            current.Health.TakeDamage(new DamageInfo(dmg, Vector2.zero, 0f, crit, ctx.Owner.gameObject));
            EnemyRegistry.TryFindNearest(current.Position + Vector2.one * 0.01f, out current);
        }
    }
}
```

Rules for behaviours:

- They are **shared assets** – never store per-player state in fields. Use `ctx.Instance` for per-weapon data.
- Spawn everything through `PoolManager`.
- Prefer `EnemyRegistry` queries over `Physics2D` for targeting and AoE (faster at scale, no layer setup).

---

## 8. Save / Load

```csharp
public class Wallet : MonoBehaviour, ISaveable
{
    [System.Serializable] private class Data { public int Gold; }
    private Data _data = new Data();

    public string SaveKey => "Wallet";
    public string CaptureState() => JsonUtility.ToJson(_data);
    public void RestoreState(string json) => _data = JsonUtility.FromJson<Data>(json) ?? new Data();

    void Start()  => SaveManager.Instance.Register(this);
    void OnDestroy() => SaveManager.Instance?.Unregister(this);
}
```

- `SaveManager.Instance.Save(slot)` / `Load(slot)` / `HasSave(slot)` / `Delete(slot)`.
- Files live in `Application.persistentDataPath` (`save0.dat`, `save0.dat.bak`).
- Writes are atomic (temp file + move) and the previous file becomes the backup.
- A corrupt file is copied to `*.corrupt`, the backup is tried, and if both fail defaults are kept – the game
  never throws on a bad save.
- Obfuscation (XOR + Base64 with a header) deters casual editing. It is **not** encryption; use a real
  cryptographic library if you need tamper-proofing.
- To use Newtonsoft JSON, replace the two `JsonUtility` calls in `SaveManager` – the `ISaveable` contract is
  string-based and does not care.

---

## 9. Performance Notes

- **Pools**: every runtime object (enemy, projectile, gem, popup, VFX) is pooled. `Get`/`Release` are O(1) and
  allocation-free. Set **Prewarm** counts on `PoolManager` to the peak you expect (e.g. 500 enemies,
  300 projectiles, 100 popups). Use **Max Idle** to cap memory after huge waves.
- **No per-frame allocations**: no LINQ, no closures, no `FindObjectOfType`, no string concatenation in hot paths.
  TMP text uses `SetText(format, float)`.
- **EnemyRegistry**: a flat list with O(1) swap-remove. `TryFindNearest` and `CollectInRadius` are linear
  scans; at 1,000 enemies that is ~1,000 float ops per weapon shot – far cheaper than physics queries. For
  5,000+ enemies, add a spatial hash here; it is the single integration point.
- **Stats** are cached and only recomputed when a modifier changes.
- **Damage popups** are rate-limited per frame (`Max Per Frame`) so an AoE hitting 400 enemies does not spawn
  400 text meshes in one frame.
- **Despawn distance** on `EnemyController` recycles enemies the player has outrun.
- Profile with the Unity Profiler → *GC Alloc* column should read 0 B during steady-state gameplay.

---

## 10. FAQ

**Can I use this in 3D?** Yes. Replace `Rigidbody2D`/`Collider2D` with their 3D counterparts in
`AIController`, `PlayerController`, `Projectile` and `EnemyController`, and change `Vector2` positions to
`Vector3` where relevant. All data, pooling, stats, inventory, save and UI code is dimension-agnostic.

**How do I add a new game state (e.g. Shop)?** Add it to the `GameState` enum, add a case to
`GameManagerBase.ApplyTimeScale` if it should freeze time, create a `UIView` for it and tick the state on that
view. Call `GameManagerBase.Instance.ChangeState(GameState.Shop)` from wherever it should open.

**How do I play a sound on level-up without code?** Create a `Horde Survival ▸ Events ▸ Game Event` asset,
raise it from a tiny bridge script that subscribes to `GameEvents.OnPlayerLevelUp`, and add a
`GameEventListener` with an AudioSource.Play response.

**Does it support multiple players?** `PlayerCharacter.Instance` is a single-player convenience used by XP
gems and enemy targeting. For co-op, replace those two call sites with a player registry (mirroring
`EnemyRegistry`).

**Unity 6 says `Rigidbody2D.velocity` is obsolete.** The template uses `linearVelocity` under
`UNITY_6000_0_OR_NEWER` and `velocity` on 2022 LTS, so no warnings on either.
