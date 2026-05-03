# AlifePlus Features

**Last updated:** 2026-05-03

> **WARNING: NOT SYNCED WITH CODE.** This document is a snapshot of cause and consequence inventory at the date above. The code may have moved on (renames, new causes, dropped causes, changed RULES/SCAN). Treat this as a reference, not a contract. Re-verify against `gamedata/scripts/ap_ext_cause_*.script` / `ap_ext_causes_*.script` and `ap_ext_consequences_*.script` before relying on any specific row. When the inventory changes, update this file and bump the date.

For the abstract pipeline rules and category model, see `architecture.md`. This file holds only the per-cause and per-consequence inventory tables.

---

## Causes

| Category | Cause | Description | Xray event | RULES | SCAN | ACTION |
|---|---|---|---|---|---|---|
| Reactions | massacre | Deaths pile up at a smart that is not a base. | squad_on_npc_death | alignment_human victim; kill_count >= threshold; at smart; not is_base | — | publish cause:massacre |
| Reactions | squadkill | A squad's last member dies away from base. | squad_on_npc_death | alignment_human victim; last member dead; not is_base | — | publish cause:squadkill |
| Reactions | basekill | Deaths pile up at a faction base. | squad_on_npc_death | alignment_human victim; kill_count >= threshold; at is_base | — | publish cause:basekill |
| Reactions | alpha | A mutant accumulates enough kills to level up as an alpha. | squad_on_npc_death | killer is mutant; killer not protected; projected kills cross new level | — | publish cause:alpha |
| Reactions | alphakill | An alpha mutant dies. | squad_on_npc_death | victim is alpha; killer not protected; cooldown clear | — | publish cause:alphakill |
| Reactions | wounded | An NPC or player uses a healing item. | WOUNDED_CALLBACKS | subject not protected; not is_base | — | publish cause:wounded |
| Reactions | harvest | An NPC or player picks up an artefact. | HARVEST_CALLBACKS | IsArtefact(item); NPC taker not protected | — | publish cause:harvest |
| Opportunities | stash_fill | Squad finds an empty stash and stocks it. | RADIANT_CALLBACKS | alignment_human; not at base; MVT(stash_fill); personality(greed, relation) | find_stashes in eye range (empty); find_smart for destination | publish cause:stash_fill |
| Opportunities | stash_loot | Squad finds a non-empty stash and loots it. | RADIANT_CALLBACKS | alignment_loot; not at base; MVT(stash_loot); personality(greed, perception) | find_stashes in eye range (non-empty); find_smart for destination | publish cause:stash_loot |
| Opportunities | stash_ambush | Outlaw squad finds a heavily-stocked stash and stakes it out. | RADIANT_CALLBACKS | alignment_outlaw; not at base; items_count >= ambush_min_items; MVT(stash_ambush); personality(greed, aggression) | find_stashes in eye range (non-empty); find_smart for destination | publish cause:stash_ambush |
| Opportunities | area_conquer | Stalker squad claims an empty smart with additive shared spawn. | RADIANT_CALLBACKS | alignment_conquer_human; MVT(area_conquer); personality(territory, aggression) | find_smart in eye range; filter is_smart_empty, not_base | publish cause:area_conquer |
| Opportunities | area_swarm | Mutant squad claims an empty smart with additive shared spawn. | RADIANT_CALLBACKS | alignment_conquer_mutant; MVT(area_swarm); personality(territory, aggression) | find_smart in eye range; filter is_smart_empty, not_base | publish cause:area_swarm |
| Opportunities | area_infest | Mutant claims a lair or surge shelter with exclusive spawn replacement. | RADIANT_CALLBACKS | alignment_mutant by species; squad has alpha; per-level cap; MVT(area_infest); personality(territory, survival) | find_smart in scent range; filter by species (lair, lair_or_surge, or surge), not infested | publish cause:area_infest |
| Needs | hunger_campfire | Hunger drive overdue; squad heads to a campfire. | RADIANT_CALLBACKS | alignment_human; Hull(hunger); personality(survival) | find_smart in radio range; filter has_campfire, exclude_enemy | publish cause:hunger_campfire |
| Needs | sleep_campfire | Sleep drive overdue; squad heads to a campfire for the night. | RADIANT_CALLBACKS | alignment_human; Hull(sleep); personality(survival) | find_smart in radio range; filter has_campfire, exclude_enemy | publish cause:sleep_campfire |
| Needs | rest_campfire | Rest drive overdue; squad heads to a campfire to consume rest item. | RADIANT_CALLBACKS | alignment_human; Hull(rest); personality(survival) | find_smart in radio range; filter has_campfire, exclude_enemy | publish cause:rest_campfire |
| Needs | heal_shelter | Heal drive overdue; squad heads to a surge shelter to consume medkit. | RADIANT_CALLBACKS | alignment_human; Hull(heal); personality(territory, survival) | find_smart in radio range; filter has_surge_shelter, exclude_enemy | publish cause:heal_shelter |
| Needs | shelter_indoor | Shelter drive overdue; squad heads to a surge shelter. | RADIANT_CALLBACKS | alignment_human; Hull(shelter); personality(territory, discipline) | find_smart in radio range; filter has_surge_shelter, exclude_enemy | publish cause:shelter_indoor |
| Needs | shelter_outdoor | Shelter drive overdue; squad heads to a campfire as outdoor shelter. | RADIANT_CALLBACKS | alignment_human; Hull(shelter); personality(perception) | find_smart in radio range; filter has_campfire, exclude_enemy | publish cause:shelter_outdoor |
| Needs | supply_trader | Supply drive overdue; squad heads to a trader. | RADIANT_CALLBACKS | alignment_human; Hull(supply); personality(greed, relation) | find_smart in radio range; filter has_trader_job | publish cause:supply_trader |
| Needs | money_harvest | Money drive overdue; selfserving stalker heads to anomaly field. | RADIANT_CALLBACKS | alignment_selfserving; Hull(money); personality(greed, perception) | find_smart in radio range; filter has_anomaly | publish cause:money_harvest |
| Needs | money_hunt | Money drive overdue; naturalist stalker heads to mutant lair. | RADIANT_CALLBACKS | alignment_naturalist; Hull(money); personality(greed, perception) | find_smart in radio range; filter is_lair | publish cause:money_hunt |
| Needs | job_outpost | Job drive overdue; principled stalker heads to non-base smart to guard. | RADIANT_CALLBACKS | alignment_principled; Hull(job); personality(territory, discipline) | find_smart in radio range; filter not_base, not_lair, exclude_enemy | publish cause:job_outpost |
| Needs | job_explore | Job drive overdue; selfserving stalker heads to unclaimed smart. | RADIANT_CALLBACKS | alignment_selfserving; Hull(job); personality(perception, territory) | find_smart in radio range; filter is_unclaimed | publish cause:job_explore |
| Needs | job_research | Job drive overdue; ecolog stalker heads to anomaly field. | RADIANT_CALLBACKS | alignment_ecolog; Hull(job); personality(perception, discipline) | find_smart in radio range; filter has_anomaly | publish cause:job_research |
| Needs | social_campfire | Social drive overdue; squad heads to campfire to share stories. | RADIANT_CALLBACKS | alignment_human; Hull(social); personality(relation) | find_smart in radio range; filter has_campfire, exclude_enemy | publish cause:social_campfire |
| Needs | social_base | Social drive overdue; principled stalker heads to faction base. | RADIANT_CALLBACKS | alignment_principled; Hull(social); personality(relation, territory) | find_smart in radio range; filter is_base, exclude_enemy | publish cause:social_base |
| Instincts | scatter | Lower-tier mutant detects a higher-tier predator in line of sight; flees. | RADIANT_CALLBACKS | alignment_mutant tier 0-2; threats present in eye range; Hull(scatter); personality(inv_aggression, survival) | find_squads in eye range (detect threats); find_smart in eye range; filter not_base, no threats | publish cause:scatter |
| Instincts | feed | Feed drive overdue; mutant heads to open territory to hunt or scavenge. | RADIANT_CALLBACKS | alignment_mutant; Hull(feed); personality(survival, perception) | find_smart in scent range; filter is_territory, not_base | publish cause:feed |
| Instincts | slumber_field | Slumber drive overdue; cowardly mutant beds down in open territory. | RADIANT_CALLBACKS | alignment_mutant_cowardly; Hull(slumber); personality(survival) | find_smart in scent range; filter is_territory, not_base | publish cause:slumber_field |
| Instincts | slumber_lair | Slumber drive overdue; feral or predator returns to lair. | RADIANT_CALLBACKS | alignment_mutant_feral or _predator; Hull(slumber); personality(survival) | find_smart in scent range; filter is_lair, not_base | publish cause:slumber_lair |
| Instincts | slumber_surge | Slumber drive overdue; aberrant or predator takes to surge shelter. | RADIANT_CALLBACKS | alignment_mutant_aberrant or _predator; Hull(slumber); personality(survival) | find_smart in scent range; filter has_surge_shelter, not_base | publish cause:slumber_surge |
| Instincts | roam | Roam drive overdue; mutant wanders to nearby territory or lair. | RADIANT_CALLBACKS | alignment_mutant cowardly + feral + predator; Hull(roam); personality(perception, territory) | find_smart in eye-to-scent range; filter is_territory or is_lair, not_base, exclude current | publish cause:roam |
| Instincts | pack | Pack drive overdue; feral or predator moves toward kin. | RADIANT_CALLBACKS | alignment_mutant_feral or _predator; Hull(pack); personality(relation) | find_smart in scent range; filter has same-faction kin, not_base | publish cause:pack |

---

## Consequences

| Cause | Consequence | Description | RULES | SCAN | ACTION |
|---|---|---|---|---|---|
| massacre | massacre_investigate | Victim's faction sends squads to investigate the kill site. | alignment_human minus renegade; personality(perception, relation) | find_squads in radio range; factions = victim faction; max_squads | per responder: script_squad to massacre site; record_event |
| massacre | massacre_scavenge | Cowardly mutants converge to feed on corpses. | alignment_mutant + alignment_mutant_cowardly species filter; personality(survival, perception) | find_squads in scent range; factions = alignment_mutant; max_squads; species filter applied per responder | per responder: script_squad to massacre site; record_event |
| massacre | massacre_loot | Outlaw stalkers strip corpses for weapons and gear. | alignment_outlaw; personality(greed, perception) | find_squads in radio range; factions = alignment_outlaw; max_squads | per responder: script_squad to massacre site; record_event |
| squadkill | squadkill_revenge | Same-faction squads pursue the killer. | alignment_human; personality(aggression, relation) | find_squads in radio range; factions = victim faction; max_squads, max_chases | per responder: script_actor_target chase killer; record_event |
| squadkill | squadkill_flee | Same-faction squads retreat to nearest faction base. | alignment_human; personality(inv_discipline, inv_territory) | find_smart for evacuation base; find_squads in radio range; factions = victim faction; max_squads | per responder: script_squad to base; record_event |
| basekill | basekill_support | Friendly squads rush to reinforce the base. | alignment_human; personality(territory, relation) | find_squads in radio range; factions = base faction; max_squads | per responder: script_squad to base; record_event |
| basekill | basekill_flee | Squads at the attacked base evacuate to nearest base. | alignment_human; personality(inv_discipline, inv_territory) | find_smart for evacuation base; find_squads at the attacked base; max_squads | per responder: script_squad to evacuation base; record_event |
| alpha | alpha_promote | Mutant becomes alpha; gains hit-power buffs and loot. | killer not protected; not at max_alphas | — | update_alpha; set hit power; record_event |
| alphakill | alphakill_targeted | Same-species mutants on same level pursue the killer. | alignment_mutant by species; personality(aggression, relation) | find_smart near killer; find_squads in scent range; factions = alpha species; max_squads, max_chases | per responder: script_squad or script_actor_target chase; record_event |
| wounded | wounded_hunt | Predator and aberrant mutants converge on the wounded. | alignment_mutant + alignment_mutant_predator + alignment_mutant_aberrant species filter; personality(aggression, perception) | find_smart near subject; find_squads in scent range; factions = alignment_mutant; max_squads; species filter applied per responder | per responder: script_squad to subject; record_event |
| wounded | wounded_help | Same-faction squads rush to help the wounded. | alignment_human; personality(relation, survival) | find_smart near subject; find_squads in radio range; factions = subject faction; max_squads | per responder: script_squad to subject; record_event |
| harvest | harvest_rob | Outlaws pursue the artefact taker. | alignment_outlaw; personality(greed, aggression) | find_smart near taker; find_squads in radio range; factions = outlaw; max_squads, max_chases | per responder: script_actor_target chase taker; record_event |
| harvest | harvest_haunt | Aberrant mutants converge on the artefact pickup site. | alignment_mutant + alignment_mutant_aberrant species filter; personality(perception, territory) | find_smart near pickup; find_squads in scent range; factions = alignment_mutant; max_squads; species filter applied per responder | per responder: script_squad to pickup; record_event |
| stash_fill | stash_fill | Stalker walks to empty stash, hides supplies. | — | — | resolve squad+smart; script_squad; on_arrive: pick items, fill_stash; record_event |
| stash_loot | stash_loot | Stalker walks to non-empty stash, loots. | — | — | resolve squad+smart; script_squad; on_arrive: loot_stash (skip if protected toolkits); record_event |
| stash_ambush | stash_ambush | Stalker walks to non-empty stash, stakes out. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| area_conquer | area_conquer | Stalker claims empty smart with additive shared spawn. | — | — | resolve squad+smart; script_squad; on_arrive: conquer_smart; record_event |
| area_swarm | area_swarm | Mutant claims empty smart with additive shared spawn. | — | — | resolve squad+smart; script_squad; on_arrive: conquer_smart; record_event |
| area_infest | area_infest | Mutant claims lair or surge shelter with exclusive spawn replacement. | — | — | resolve squad+smart; script_squad; on_arrive: infest_smart; record_event |
| hunger_campfire | hunger_campfire | Stalker walks to campfire and consumes food item. | — | — | resolve squad+smart; script_squad; on_arrive: consume HUNGER section; record_event |
| sleep_campfire | sleep_campfire | Stalker walks to campfire for the night. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| rest_campfire | rest_campfire | Stalker walks to campfire and consumes rest item. | — | — | resolve squad+smart; script_squad; on_arrive: consume REST section; record_event |
| heal_shelter | heal_shelter | Stalker walks to surge shelter and consumes medkit. | — | — | resolve squad+smart; script_squad; on_arrive: consume HEAL section; record_event |
| shelter_indoor | shelter_indoor | Stalker walks to surge shelter. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| shelter_outdoor | shelter_outdoor | Stalker walks to campfire as outdoor shelter. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| supply_trader | supply_trader | Stalker walks to trader, exchanges artefact for supplies. | — | — | resolve squad+smart; script_squad; on_arrive: trade artefact; record_event |
| money_harvest | money_harvest | Stalker walks to anomaly field for artefact harvest. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| money_hunt | money_hunt | Stalker walks to mutant lair for hide hunt. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| job_outpost | job_outpost | Stalker walks to non-base smart to guard. | — | — | resolve squad+smart; script_squad; on_arrive: consume GUARD section; record_event |
| job_explore | job_explore | Stalker walks to unclaimed smart to explore. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| job_research | job_research | Stalker walks to anomaly field to research. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| social_campfire | social_campfire | Stalker walks to campfire to share stories. | — | — | resolve squad+smart; script_squad; on_arrive: consume SOCIAL section; record_event |
| social_base | social_base | Principled stalker walks to faction base for downtime. | — | — | resolve squad+smart; script_squad; on_arrive: consume SOCIAL section; record_event |
| feed | feed | Mutant pack moves to open territory to hunt or scavenge. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| slumber_field | slumber_field | Cowardly mutant beds down in open territory. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| slumber_lair | slumber_lair | Feral or predator returns to lair. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| slumber_surge | slumber_surge | Aberrant or predator takes to surge shelter. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| roam | roam | Mutant wanders to nearby territory or lair. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| pack | pack | Feral or predator moves toward smart with same-faction squads. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
| scatter | scatter | Mutant flees from higher-tier predator to safe smart in eye range. | — | — | resolve squad+smart; script_squad; on_arrive: passive (DTO reset); record_event |
