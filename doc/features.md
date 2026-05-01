# AlifePlus Features

*Inventory of causes and consequences. Framework rules, phase semantics, naming conventions, and uniformity scope live in `architecture.md`.*

## Causes

### Reactions

| Cause | Description | Xray Event |
|---|---|---|
| massacre | Deaths pile up at a smart terrain that isn't a base. | squad_on_npc_death |
| squad_kill | A squad's last member dies away from base. | squad_on_npc_death |
| base_kill | Deaths pile up at a faction base. | squad_on_npc_death |
| wounded | An NPC or player uses a healing item. | x_npc_medkit_use, actor_on_item_use |
| harvest | An NPC or player picks up an artefact. | actor_on_item_take, npc_on_item_take |
| alpha | A mutant accumulates enough kills to level up as an alpha. | squad_on_npc_death |
| alpha_kill | An alpha mutant dies. | squad_on_npc_death |

### Opportunities

| Cause | Description | Xray Event |
|---|---|---|
| stash_empty | Squad discovers an empty stash. | RADIANT_CALLBACKS |
| stash_full | Squad discovers a non-empty stash. | RADIANT_CALLBACKS |
| stash_trap | Squad discovers a heavily-stocked stash. | RADIANT_CALLBACKS |
| area_empty | Squad discovers an unclaimed smart terrain. | RADIANT_CALLBACKS |
| area_lair | Squad discovers a mutant lair. | RADIANT_CALLBACKS |
| area_base | Squad discovers a friendly base of its own faction. | RADIANT_CALLBACKS |
| overpowered_human | Stalker squad detects an overwhelming enemy force in line of sight. | RADIANT_CALLBACKS |
| overpowered_mutant | Mutant pack detects a higher-tier predator in line of sight. | RADIANT_CALLBACKS |

### Needs

| Cause | Description | Xray Event |
|---|---|---|
| hunger | Hunger is the strongest unmet need. | RADIANT_CALLBACKS |
| sleep | Sleep is the strongest unmet need. | RADIANT_CALLBACKS |
| rest | Rest is the strongest unmet need. | RADIANT_CALLBACKS |
| heal | Healing is the strongest unmet need. | RADIANT_CALLBACKS |
| shelter | Shelter is the strongest unmet need. | RADIANT_CALLBACKS |
| money | Money is the strongest unmet need. | RADIANT_CALLBACKS |
| supply | Supplies are the strongest unmet need. | RADIANT_CALLBACKS |
| job | Work duty is the strongest unmet need. | RADIANT_CALLBACKS |
| social | Social contact is the strongest unmet need. | RADIANT_CALLBACKS |

### Instincts

| Cause | Description | Xray Event |
|---|---|---|
| feed | Hunger is the strongest drive. | RADIANT_CALLBACKS |
| hibernate | Sleep is the strongest drive. | RADIANT_CALLBACKS |
| explore | Exploration is the strongest drive. | RADIANT_CALLBACKS |
| herd | Pack contact is the strongest drive. | RADIANT_CALLBACKS |

## Consequences

| Cause | Consequence | Description |
|---|---|---|
| massacre | massacre_investigate | Victim's faction sends squads to investigate the kill site. |
| massacre | massacre_scavenge | Cowardly mutants converge to feed on corpses. |
| massacre | massacre_flee | Non-military human witnesses flee to a far campfire. |
| squad_kill | squad_revenge | Same-faction squads pursue the killer. |
| base_kill | base_support | Friendly squads rush to reinforce the base. |
| base_kill | base_flee | Squads at the attacked base evacuate to the nearest faction base. |
| wounded | wounded_hunt | Predator and aberrant mutants converge on the wounded. |
| wounded | wounded_help | Same-faction squads rush to help the wounded. |
| harvest | harvest_rob | Outlaws pursue the artefact carrier. |
| harvest | harvest_haunt | Aberrant mutants converge on the artefact pickup site. |
| alpha | alpha_promote | Mutant gains buffs and carries valuable loot. |
| alpha_kill | alpha_revenge | Same-species mutants on the same level pursue the alpha killer. |
| stash_empty | stash_fill | Stalker squad walks to the empty stash and hides supplies. |
| stash_full | stash_loot | Stalker squad walks to the stash and loots its contents. |
| stash_trap | stash_ambush | Outlaw squad camps at the stash to ambush looters. |
| area_empty | area_conquer | Human faction claims an empty smart terrain. |
| area_empty | area_overrun | Mutant species claims an empty smart terrain. |
| area_lair | area_infest | Mutants claim a lair as their nest. |
| area_lair | area_drift | Deprived mutant pack walks to a lair to recover. |
| area_base | area_desert | Deprived stalker squad walks home to recover. |
| overpowered_human | overpowered_flee | Stalker squad falls back to a friendly base. |
| overpowered_mutant | overpowered_scatter | Mutant pack scatters to a distant safe smart terrain. |
| hunger | hunger_campfire | Squad walks to a campfire to eat. |
| sleep | sleep_campfire | Squad walks to a campfire to sleep. |
| rest | rest_campfire | Squad walks to a campfire to relax. |
| heal | heal_shelter | Squad walks to shelter for medical attention. |
| shelter | shelter_indoor | Squad walks to a surge-sheltered indoor location. |
| shelter | shelter_outdoor | Squad falls back to a campfire when no indoor shelter is nearby. |
| money | money_harvest | Naturalist squad walks to an anomaly zone to harvest artefacts. |
| money | money_hunt | Squad walks to a mutant lair to hunt for trophies and pelts. |
| supply | supply_trader | Squad walks to a trader smart to exchange goods. |
| job | job_outpost | Squad walks to a friendly outpost for guard duty. |
| job | job_explore | Squad walks to a distant smart terrain to explore. |
| job | job_research | Squad walks to an anomaly zone to study anomalous phenomena. |
| social | social_campfire | Squad joins another at a campfire to socialize. |
| social | social_base | Squad walks home to its faction base for companionship. |
| feed | feed_hunt | Mutant pack moves to open territory to hunt. |
| hibernate | hibernate_lair | Mutant pack returns to a rest location. |
| explore | explore_roam | Mutant pack roams to nearby territory or lair. |
| herd | herd_join | Pack animals join same-faction packs. |

## Files

### One cause per file (`ap_ext_cause_<name>.script`)

- massacre
- squad_kill
- base_kill
- wounded
- harvest
- alpha
- alpha_kill
- stash_empty
- stash_full
- stash_trap
- area_empty
- area_lair
- area_base
- overpowered_human
- overpowered_mutant

### Bundled cause files (Hull predicate)

| File | Causes |
|---|---|
| ap_ext_cause_needs.script | hunger, sleep, rest, heal, shelter, money, supply, job, social |
| ap_ext_cause_instincts.script | feed, hibernate, explore, herd |

### One consequence per file (`ap_ext_consequence_<name>.script`)

- squad_revenge
- alpha_promote
- alpha_revenge

### Hand-written multi-consequence sets (`ap_ext_consequence_<family>_set.script`)

| Set | Consequences |
|---|---|
| massacre_set | 3 |
| base_kill_set | 2 |
| wounded_set | 2 |
| harvest_set | 2 |
| stash_set | 3 |
| area_set | 5 |
| fear_set | 2 |

### CONFIGS factory consequence files

| File | Consequences |
|---|---|
| ap_ext_consequence_needs.script | 14 |
| ap_ext_consequence_instincts.script | 4 |
