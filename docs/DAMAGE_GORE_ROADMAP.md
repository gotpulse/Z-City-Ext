# Z-City: Damage & Gore Extension Roadmap

An evaluation of the current damage/organism/gore code and a ranked master list of features that extend it.
All paths are relative to the repo root.

---

## 1. How the system works today (short version)

| Layer | Where | What it does |
|---|---|---|
| Damage entry | `lua/homigrad/organism/tier_1/sv_input.lua:426` (`EntityTakeDamage "homigrad-damage"`) | Single choke point. Reads `inf.bullet` / weapon fields (`Penetration`, `Diameter`, `BleedMultiplier`, `PainMultiplier`, `ShockMultiplier`, …), traces the shot through organ boxes, applies wounds/pain/shock, handles exit-hole re-fire. |
| Organ hitboxes | `lua/homigrad/organism/tier_0/sh_hitboxorgans_manual.lua` | Oriented boxes on bones: `{name, boneResist, localPos, ang, size, color}`. Each name maps to a handler in `hg.organism.input_list`. **Adding an organ = one box + one handler.** |
| Organ handlers | `modules_input/sv_organs.lua`, `modules_input/sv_bone.lua`, `lua/homigrad/sv_equipment.lua`, `equipment_system/sh_equipment.lua` | heart, liver, stomach, intestines, brain, lungs, trachea (disabled), 5 arteries (spine artery disabled), skull, jaw, chest, pelvis, spine1-3, 4 limbs (up/down), armor plates. |
| Physiology | `organism/tier_1/modules/sv_blood.lua`, `sv_pain.lua`, `sv_lungs.lua`, `sv_pulse.lua`, … | Blood volume/type, wound bleed + coagulation, arterial pumping, internal bleed, pneumothorax, pain/shock/consciousness, adrenaline, analgesia. |
| Damage-type mapping | `sv_input.lua:1176` `hg.organism.DamageTypeAffliction` | Maps DMG_* → bleed / hurt / instant pain / immobilization. |
| Dismemberment | `sv_input.lua:163` `hg.organism.AmputateLimb`, `:940-1040` damage stack | Forearm/calf only. Triggered by per-hitgroup damage stack >100, blasts, heavy crush. Rendered by bone-scaling + a flesh "nugget" stump (`cl_main.lua:1004` `hg.GoreCalc`). |
| Head gib | `lua/homigrad/headgib/init_sv.lua` | Bone-scale head to 0, stump prop (`headboom.mdl`), 8-10 re-textured watermelon chunks, neck fountain. Only fires on already-dead bodies. |
| Blood FX | `organism/tier_1/modules/particles/*`, `cl_main.lua:688+` | Custom particle sim (streaks, clouds, underwater), escalating decals (`Normal/Arterial.Blood21-25`), wound drips & arterial jets from networked `wounds` / `arterialwounds`. |
| Ragdoll ("fake") | `lua/homigrad/fake/*` | Death/knockdown ragdolls keep the organism (keep bleeding). Collision → bone/skull/spine damage (`sv_input.lua:1368` `velocityDamage`). Neck-break constraint swap. Death spasms (`organism/sv_brainfuck.lua`). |
| Screen FX | `lua/homigrad/cl_screeneffects.lua`, `cl_main.lua:356` | Vignette, grain, blur, desaturation by blood, chromatic aberration by pain, tinnitus, unconscious dreams. |

### Extension points already available
- Hooks: `PreTraceOrganBulletDamage` (mutate/deny per-organ damage), `PreHomigradDamage`, `PreHomigradDamageBulletBleedAdd`, `HomigradDamage`, `OnAmputateLimb`, `HG_BloodParticleStartedDropping`, `HG_OrganAvalible`, `Org Clear`, `Org Think`, `HG_OnOtrub`, `EntityFireBullets` / `PostEntityFireBullets`.
- Tables: `hg.organism.input_list`, `hg.ammotypes` (`BulletSettings` / `BulletFunctions`), `hg.armor`, `hg.amputeetable`, `hg.bonetohitgroup`.
- Module pattern: `hg.organism.module.X = {[1]=init(org), [2]=think(owner, org, dt)}`.

### What is conspicuously missing
- **No gore ConVars at all**: no global toggle, no dismemberment/headgib switch, no gore level for clients.
- **No wound visuals on bodies**: wounds have bone-local position+angle data and are networked, but nothing draws a hole/decal. Only drips come out of them.
- **No burn visuals**: `org.burns` is counted and never used; no charring.
- **Gibs are placeholders**: watermelon props with a flesh texture (the code comment at `headgib/init_sv.lua:170` literally says "swap the models").
- **Organism only covers 3 NPC classes** (`sv_npcstuff.lua:5`), so zombies/VJ/nextbots get no gore.
- **Medical gaps**: no splint, no sutures/surgery, no stump care, no dislocation reduction item.

---

## 2. Bugs & dead wiring found during the review

All fixed on this branch.

| # | Issue | Location | Fix |
|---|---|---|---|
| B1 | `hg.organism.CoughBlood` used undefined `ent`, `bon`, `mat` (Lua error on the blood-spit branch, reachable from `sv_phrases.lua:480`) | `organism/tier_1/modules/sv_blood.lua` | Resolve the character, head bone and matrix locally; bail out if missing. |
| B2 | `insolid` read before its `local` declaration, so blood decals were placed even from inside solids | `particles/cl_blood.lua` | Declare `insolid` before the decal check. |
| B3 | `PreHomigradDamage` passed `hitgroup`, `hitBoxs`, `inputHole` as undefined globals | `organism/tier_1/sv_input.lua` | Still fires before the organ trace (hooks scale damage there), now passes the hitgroup from the pre-trace and empty tables. |
| B4 | `RubberBullets` read but never set; would also have errored on a nil `Penetration` | `modules_input/sv_bone.lua`, `sh_ammostuff.lua` | New `BulletSettings.RubberBullets` flag on 12/70 beanbag, .45 Rubber, 18x45mm Traumatic; resolved from the ammo table (weapon flag still honoured) with a safe penetration fallback. |
| B5 | Stray undefined `penmul` argument passed to `callbackBullet` (which only takes 6 params, so it was ignored) | `weapons/homigrad_base/sh_bullet.lua` | Removed. |
| B6 | `BulletSettings.Mass` unused for dismemberment | `organism/tier_1/sv_input.lua` | Heavy rounds build the limb/head gib stack faster: `clamp(sqrt(Mass/10), 1, 1.5)`, bullets only, rubber excluded, never weaker than before. Toggle: `hg_dmgstack_mass` (default 1). |
| B7 | Legacy armor never degraded, and its health multiplier couldn't change stop/penetrate outcomes | `lua/homigrad/sv_equipment.lua`, `lua/entities/armor_base/init.lua` | Protection is now `protection * health - penetration` (same as the new equipment system; identical for fresh armor). Each bullet wears it by `pen / protection * 0.05` (divided by pellet count for shotguns). Wear follows the item on drop, pickup, death and looting. |

---|---|---|---|
| B1 | `hg.organism.CoughBlood` uses undefined `ent`, `bon`, `mat` | `organism/tier_1/modules/sv_blood.lua:266-285` (called from `sv_phrases.lua:480`) | Lua error on the 1-in-5 blood-spit branch whenever a player with vomit in the throat tries to talk. |
| B2 | `insolid` read before its `local` declaration | `particles/cl_blood.lua:229` vs `:235` | Condition is always true (global nil); decals get placed even from inside solids. |
| B3 | `PreHomigradDamage` receives `hitgroup`, `hitBoxs`, `inputHole` before they exist | `sv_input.lua:610` | Hook consumers always get `nil` for those args — hurts every extension. |
| B4 | `RubberBullets` read but never set | `modules_input/sv_bone.lua:3,19`; spec'd in `weapons/homigrad_base/shared.lua:28` | Less-lethal bone-crush path is dead code. |
| B5 | `penmul` passed but undefined | `weapons/homigrad_base/sh_bullet.lua:362` | Always nil → world-penetration multiplier silently ignored. |
| B6 | `BulletSettings.Mass` unused for damage | `sv_input.lua:938` (commented) | Heavy vs light rounds feel identical for dismemberment stacking. |
| B7 | Legacy armor never degrades (only `protovisor`) | `lua/homigrad/sv_equipment.lua:225-269` | Inconsistent with the new equipment system's durability. |

---

## 3. Master feature list

Scores: **Impact** 1-5 (how much it changes feel/gameplay), **Effort** S/M/L/XL, **Leverage** = how much existing code does the heavy lifting.

### A. Foundations
| ID | Feature | Builds on | Impact | Effort |
|---|---|---|---|---|
| A1 | **Gore settings suite**: `hg_gore` (server master), `hg_dismemberment`, `hg_headgib`, `hg_gib_lifetime`, `hg_gore_scale`; client `hg_gore_level` (0 = no gibs/sprays, 1 = reduced, 2 = full) and add them to `cl_menu_options.lua` "Blood" category | `AmputateLimb`, `Gib_Input`, `SpawnMeatGore`, blood particle receivers | 4 | S |
| A2 | ~~Fix bugs B1-B7~~ (done) | — | 3 | S |
| A3 | **Gore API**: `hg.gore.RegisterWoundType`, `hg.gore.RegisterGib`, `hg.gore.Sever(ent, bone, opts)`, uniform `OnSever`/`OnGib` hooks so gamemodes and addons extend instead of patching `sv_input.lua` | Existing hooks | 3 | M |

### B. Dismemberment & gibbing
| ID | Feature | Builds on | Impact | Effort |
|---|---|---|---|---|
| B-1 | **Severed limb props**: spawn the actual detached limb (clone ragdoll, scale everything *except* the severed chain to 0, or use gib models) instead of generic chunks. Pickup/throw/loot (cuffs, weapon in hand drops with it) | `AmputateLimb`, `Gib_RemoveBone`, `OnAmputateLimb` | 5 | M |
| B-2 | **Real gib models** replacing the watermelon chunks (skull fragments, brain matter, meat, bone shards) with per-region sets | `SpawnMeatGore` (`headgib/init_sv.lua:86`) | 4 | M (art) |
| B-3 | **Full-limb amputation (shoulder / hip)**: enable the commented `UpperArm`/`Thigh` rows, two-stage severing (distal first, then proximal), femoral/brachial bleed is lethal in ~1 min | `hg.amputeetable`, `GoreCalc`, arteries | 4 | M |
| B-4 | **Directional blast dismemberment**: re-enable the commented blast block with distance + line-of-sight + facing, point-blank full-body gib (reuse `explode()` particles) | `sv_input.lua:1042-1056`, `hg.ExplosionTrace` | 4 | M |
| B-5 | **Melee decapitation / chop**: heavy-slash weapons (axe, machete, sword) can sever head/limbs on downed or dead targets via a `SeverChance` weapon field | `weapon_melee.lua`, `hg.ExplodeHead`, damage stack | 4 | S-M |
| B-6 | **Mangled limb state** (between broken and amputated): limb hangs, bone exposed, can't be used, heavy bleed; tourniquet or amputate to stabilise | `org.dmgstack` 50-100 range, `sv_bone.lua` | 3 | M |
| B-7 | **Evisceration**: large slash/blast to the abdomen exposes intestines (prop/bodygroup), heavy bleed, forces crawling | `intestines` organ, `DMG_SLASH` path | 3 | L |
| B-8 | **Jaw / face destruction**: jaw hit above threshold removes jaw (bone scale + stump), muffled speech, can't eat/drink | `jaw` organ, `jawdislocation`, voice-change mask code | 3 | M |
| B-9 | **Headgib on living targets for high-energy rounds** (.50, 12ga slug at contact range); currently only fires once already dead | `sv_input.lua:994` alive check | 3 | S |

### C. Wound & body visuals
| ID | Feature | Builds on | Impact | Effort |
|---|---|---|---|---|
| C-1 | **Visible wound decals on bodies** (entry/exit holes, cuts, stab marks) drawn at the already-networked bone-local wound positions; persist on death ragdolls | `org.wounds` `{size, lpos, lang, bone, time}` NetVar, `sh_render.lua` | 5 | M |
| C-2 | **Progressive blood-soaking of clothes/skin** using the render-target `$detail` overlay already used for bloody weapons | `lua/homigrad/sh_bloody_decals.lua` `AddDecalToEnt` | 4 | M |
| C-3 | **Blood pools** that grow under bleeding bodies (ragdolls keep bleeding post-death) instead of only stacked decals | `HG_BloodParticleStartedDropping`, `hg.bloodpositions` | 4 | M |
| C-4 | **Exit-wound back-spatter** onto walls behind the victim (exit `hg_bloodimpact` is commented out) | `sv_input.lua:664-679`, `decalBlood` | 4 | S |
| C-5 | **Blood trails** when dragged or crawling while bleeding | `org.bleed`, fake ragdoll control | 3 | S |
| C-6 | **Burn visuals**: charring material blend scaled by `org.burns`, smouldering particles, fully charred corpses | `org.burns` (unused), vFire | 4 | M |
| C-7 | **Compound fracture visuals**: bent limb (`ManipulateBoneAngles`) and bone-shard model on broken limbs | `org.lleg == 1` etc. | 3 | M |
| C-8 | **Pallor / cyanosis**: skin tint shifts with blood volume and O2 (pale → grey → blue lips) | `org.blood`, `org.o2` | 2 | S |
| C-9 | **Blood on screen** for the victim (close hits) and bystanders (splatter at close range) | `cl_screeneffects.lua` | 2 | S |

### D. Damage-model depth
| ID | Feature | Builds on | Impact | Effort |
|---|---|---|---|---|
| D-1 | **Ammo behaviours**: hollow-point (bigger wound, less pen), FMJ, AP (armor-favoured), tumbling / fragmenting (5.56 yaw & fragment inside body), slugs, all as `BulletSettings` flags | `PreTraceOrganBulletDamage`, `hg.ammotypes`, exit-hole re-fire | 5 | M |
| D-2 | **Temporary cavity / hydrostatic damage**: high-velocity rounds damage organs adjacent to the wound channel (use `Speed` & `Mass`) | `hg.organism.Trace` `size` param, B6 | 4 | M |
| D-3 | **New organs**: kidneys, spleen (internal bleed), eyes (partial blindness - `org.blindness` exists), femoral artery on thigh, bladder, re-enable trachea & spine artery | Hitbox box format + `input_list` | 4 | S-M |
| D-4 | **Explicit behind-armor blunt trauma**: stopped rounds break ribs, knock the wind out (O2 drop), bruise organs based on plate material | Equipment `protec`, `ProtectionDamageMul` | 4 | S |
| D-5 | **Rib fractures / flail chest**: breathing pain, stamina cap, chance to cause pneumothorax on exertion | `org.chest`, `sv_lungs.lua` | 3 | S |
| D-6 | **Lodged bullets & shrapnel** as persistent foreign bodies (infection source, pain on movement, need extraction) | `org.LodgedEntities` (arrows only today) | 3 | M |
| D-7 | **Burn depth model**: 1st/2nd/3rd degree per body region, fluid loss, burn shock, airway burns from fire inhalation | `DMG_BURN` path (`sv_input.lua:840`) | 3 | M |
| D-8 | **Wound infection / sepsis** over long rounds: dirty/untreated wounds → fever (`org.temperature` exists) → sepsis | `org.wounds`, `sv_virus.lua` stage pattern | 3 | M |
| D-9 | **Concussion / TBI** from blunt head hits: delayed symptoms, vomiting, confusion | `org.skull`, `org.brain`, `hg.organism.Vomit` | 3 | S |
| D-10 | **Organ-level ricochet / glancing skull shots** (commented out) | `sv_hitboxorgans.lua:84-103` | 2 | S |
| D-11 | **Legacy armor durability** + visible damage on armor models | B7, new equipment system | 2 | S |

### E. Medical counterplay (keeps extra gore survivable and interesting)
| ID | Feature | Builds on | Impact | Effort |
|---|---|---|---|---|
| E-1 | **Apply pressure**: hold a key (or on another player) to slow the largest wound's bleed; hands occupied, bloody hands | `org.wounds`, `weapon_hands_sh` | 4 | S |
| E-2 | **Splint item** (dedicated, instead of bandage fallback) | `weapon_bandage_sh.lua:558-597` | 3 | S |
| E-3 | **Suture / surgery kit**: permanently close wounds, extract lodged bullets (pairs with D-6) | `weapon_hg_medicine_base.lua` | 3 | M |
| E-4 | **Hemostatic gauze / chest seal**: packing for arterial & junctional wounds; seal for open chest wounds | Bandage base, `org.pneumothorax` | 3 | S |
| E-5 | **Stump care**: tourniquet or cauterise (lighter/fire) an amputation stump | `AmputateLimb` arterial wound, `weapon_tourniquet` | 3 | S |
| E-6 | **Reduce dislocation** action (fields exist, no treatment other than class reset) | `org.*dislocation` | 2 | S |

### F. Feedback, NPCs & modes
| ID | Feature | Builds on | Impact | Effort |
|---|---|---|---|---|
| F-1 | **Organism & gore for all NPCs** (zombies, VJ, nextbots) behind a ConVar, with per-class organ templates | `sv_npcstuff.lua:5` whitelist, `npcDmg` table in `sv_input.lua` | 5 | M |
| F-2 | **Forensics / body examination UI**: turn the chat-print exam (`weapon_hands_sh.lua:972+`) into an inspection panel showing wound locations, types, calibre of lodged rounds, time of death (Homicide mode gold) | `bulletwounds/stabwounds/…` counters, `org.wounds` | 4 | M |
| F-3 | **Pain vocalisation**: re-enable per-hitgroup pain voice lines with cooldown; screams on amputation/burns | `paintable` (`sv_input.lua:1130-1174`, commented) | 3 | S |
| F-4 | **Wounded animations**: limp on leg injury, clutch wound, one-arm handling on broken arm | `dynamic_anims_util`, `sh_anims.lua` | 4 | M |
| F-5 | **More death behaviour**: use the unused `extend`/`flexion` spasms, agonal breathing, dying gurgle when `arteria` hit | `sv_brainfuck.lua:9` | 2 | S |
| F-6 | **Attacker hit feedback**: distinct audio for headshot / bone break / armor stop | `HomigradDamage` hook | 2 | S |
| F-7 | **Per-gamemode gore profiles** (e.g. TDM arcade-lite, Homicide full realism) | A1 ConVars, mode tables | 2 | S |

---

## 4. Ranking

Ranked by **value = impact x leverage ÷ effort**, i.e. how much player-visible improvement you get per hour, given what the code already supports. Dependencies are noted.

| Rank | ID | Feature | Why here |
|---|---|---|---|
| 1 | A2 | ~~Fix bugs B1-B7~~ **Done** | Cheap, removes Lua errors and makes hooks reliable before building on them. |
| 2 | A1 | Gore settings suite | Required so heavier gore doesn't alienate servers/players; gates everything below. |
| 3 | C-1 | Visible wound decals on bodies | Biggest visual gap; position/angle data is **already networked** — only rendering is missing. |
| 4 | D-1 | Ammo behaviours (HP/AP/tumble/frag) | Plugs straight into `PreTraceOrganBulletDamage` + ammo table; deepens every gunfight. |
| 5 | B-1 | Severed limb props | Dismemberment already happens; this makes it read instantly and adds emergent play. |
| 6 | C-4 | Exit-wound back-spatter | Code path exists (commented), strong feedback for tiny effort. |
| 7 | F-1 | Organism/gore for all NPCs | Instantly applies the whole system to Defense/coop zombie content. |
| 8 | E-1 | Apply pressure | Small, high-agency counterplay to the stronger bleeding the gore features add. |
| 9 | D-4 | Behind-armor blunt trauma | Few lines in `protec`; makes armor hits feel consequential. |
| 10 | D-3 | New organs (kidneys, spleen, eyes, femoral) | Data-driven box + handler; more varied outcomes per hit. |
| 11 | B-2 | Real gib models | Replaces placeholder watermelons; mostly art cost. |
| 12 | C-6 | Burn visuals | `burns` counter already tracked; fire is common (molotovs/vFire). |
| 13 | B-5 | Melee decapitation / chop | Short addition to melee base; satisfying for axe/machete. |
| 14 | C-3 | Blood pools | Bodies already keep bleeding; strong scene-dressing. |
| 15 | F-2 | Forensics UI | Counters exist; huge for Homicide/Fear modes. |
| 16 | B-4 | Directional blast dismemberment | Re-enables commented code with better logic. |
| 17 | C-2 | Blood-soaked clothing | Reuses weapon blood RT system. |
| 18 | B-3 | Full-limb amputation | Needs render + physics work for upper bones. |
| 19 | D-2 | Temporary cavity damage | Depends on D-1 and B6. |
| 20 | F-4 | Wounded animations | Great feel, animation work is the cost. |
| 21 | F-3 | Pain vocalisation | Code exists, needs tuning to avoid spam. |
| 22 | E-2 / E-4 / E-5 / E-6 | Splint, hemostatic/chest seal, stump care, reduce dislocation | Small medical items that balance new injuries. |
| 23 | D-5 / D-9 | Rib fractures, concussion | Cheap depth for blunt damage. |
| 24 | C-5 | Blood trails | Cheap flavour. |
| 25 | B-9 | Headgib on living targets (high-energy) | One condition change, but balance-sensitive. |
| 26 | C-7 | Compound fracture visuals | Bone manipulation on ragdoll/player. |
| 27 | B-6 | Mangled limb state | New intermediate state machine. |
| 28 | D-6 + E-3 | Lodged bullets + surgery kit | Pair; bigger RP-oriented feature. |
| 29 | A3 | Gore API | Worth it once 5+ gore features exist. |
| 30 | D-7 / D-8 | Burn depth, infection/sepsis | Long-round/RP depth. |
| 31 | B-8 | Jaw / face destruction | Niche, needs art. |
| 32 | B-7 | Evisceration | High art + animation cost. |
| 33 | C-8 / C-9 / F-5 / F-6 / F-7 / D-10 / D-11 | Polish items | Small wins, do opportunistically. |

### Suggested milestones
1. **Hardening** - A2, A1.
2. **"You can see it"** - C-1, C-4, B-1, B-2, C-3.
3. **"You can feel it"** - D-1, D-4, D-3, E-1, F-1.
4. **Fire & steel** - C-6, B-5, B-4, B-3.
5. **Realism / RP** - F-2, F-4, D-6 + E-3, D-8, remaining medical items.
