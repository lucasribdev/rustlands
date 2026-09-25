# Rustlands rounds

## Startup

From the repository root, use the dedicated profile:

```sh
dotnet run --project Content.Server -- --config-file Resources/ConfigPresets/Rustlands.toml
```

For an installation with its own configuration, add `presets = "Rustlands"` to the
server configuration's `[config]` section. Presets supply defaults: explicit values
in the host configuration and command line take precedence. Remove old conflicting
map, mode, event, and shuttle overrides. The generic `runserver.*` scripts remain
generic; running those scripts alone does not select Rustlands.
The old Wasteland override was moved from the example `server_config.toml` to
`Rustlands.toml`. No global C# defaults or generic game modes were changed.

## Findings and decisions

| Area | Inherited behavior | Rustlands baseline |
| --- | --- | --- |
| Mode | Without an explicit mode, the default is `Secret`; fallback allows `Traitor,Extended`. | `Rustlands` preset, `rules: []`, no fallback, and no mode-change vote. |
| Events | The example enabled events. `Extended` still includes meteors, station events, and space traffic. `Greenshift` still applies `BasicRoundstartVariation`. | No schedulers; `events.enabled = false`. No automatic variations of wires, lights, trash, contraband, or solar panels. |
| Station | `TestStation` already lacked arrivals, CentComm, and evacuation, but included station alerts. | `RustlandsStation` reuses only `BaseStation`, `BaseStationJobsSpawning`, and `BaseStationRecords`. No alerts, cargo, CentComm, expeditions, or event eligibility. |
| Jobs | The map used `Passenger` and its station loadout. | `RustlandsSurvivor` (Survivor), with `[-1, -1]` slots, a dedicated kit, and no access permissions. Reuses characters, inventory, and records. No automatic antagonist selection. |
| Objectives | Antagonist rules can generate station-related missions and victory conditions. | No mode-assigned objectives, mandatory victory, or deadline. Exploring, surviving, and interacting are player choices. |
| Arrival | A `SpawnPointLatejoin` exists on the grid associated with `Wasteland`. | `SpawnPointRustlandsSurvivor` uses the same location for round start, avoiding invalid spawn fallback; arrivals disabled. Lobby and late joining enabled. |
| Evacuation | `RoundEndSystem` checks automatic calls independently of the rule list; the example uses 90 minutes. | `shuttle.emergency = false`, initial call and extension set to zero, and no evacuation component on the station. |
| Round ending | Specific rules and evacuation can end a round; voting can also restart it directly. | No time-limit, inactivity, or victory rule. Restart voting and administrative commands remain available. |

Selecting only the Rustlands mode does not apply the server profile's CVars.
Load the full profile to prevent automatic shuttle calls and mode changes.
Admins can still deliberately change CVars, add rules, or end the session;
this configuration does not block administrative tools.

## What remains active

The new preset adds no `GameRule` entities. GameTicker, the lobby, characters,
job slots, spawning, records, and normal object and creature systems remain active:
damage, death, hunger, thirst, inventory, interaction, and construction.
This is not `Sandbox` mode and does not grant players creation tools.
The map retains its configured breathable atmosphere, gravity, and lighting.

The initial identity is **Survivor** (`RustlandsSurvivor`). The Rustlands category
makes it selectable through existing preferences; it does not represent a hierarchy.
The `JobRustlandsSurvivor` tracker satisfies the standard job contract, with no
unlocks, skills, classes, or progression. There are no playtime requirements,
access permissions, supervisors, PDAs, radios, or ID cards. The Passenger loadout
is not applied. Other maps and the Passenger job remain unchanged.

### Starting kit

| Slot/content | Prototype | Compatibility and quantity |
| --- | --- | --- |
| Clothing | `ClothingUniformJumpsuitColorGrey` | Simple clothing, `jumpsuit` slot (`innerclothing`), no armor protection. |
| Footwear | `ClothingShoesColorBlack` | `shoes` slot (`FEET`), no armor protection. |
| Backpack | `ClothingBackpack` | `back` slot, storage grid of 7 × 4 cells. |
| Water in backpack | `RustlandsWaterBottle` | Inherits `DrinkWaterBottleFull`; retains a 60u container, starts with only 20u of `Water`, Small size. |
| Food in backpack | `FoodSnackRaisins` | One portion, 10u of `Nutriment`, Tiny size. |
| Dressing in backpack | `Gauze1` | One gauze application for bleeding, cuts, and punctures, Small size. |

All three supplies fit in the backpack simultaneously (default shapes of 2, 1, and
2 cells), do not match its blacklist, and use the normal drinking, eating, and
healing systems. The partially filled bottle is exclusive to Rustlands; the generic
bottle remains full. The clothing retains its original components, including
sensors, but spawning does not depend on a station network. There are no weapons,
armor, tools, emergency boxes, or advanced medicines: the kit supports an initial
search for resources without sustaining indefinite exploration.

The default human breathes Wasteland's atmosphere without additional equipment.
The dedicated roleLoadout preserves only conditional life support for Vox:
`GroupTankHarness`, `RustlandsSurvivorBreathing`, and `GroupSpeciesBreathTool`.
These reuse `LoadoutTankHarness`, `LoadoutSpeciesVoxNitrogen`, and
`LoadoutSpeciesBreathTool`, all restricted by `EffectSpeciesVox`. Vox therefore
receive a tank harness, nitrogen tank, and mask; humans receive no extras.
The `Survival` group and its station boxes are not included. Existing species
physiology, diets, and healing restrictions remain in effect.
No new respawn or persistence system was created. Death does not automatically
end the round or guarantee respawning.

Restart voting retains the generic restrictions (including population/ghosts,
quorum, and admin presence). `restartround` ends the current round and schedules
the next through the existing flow; `restartroundnow` restarts immediately.
A restart does not preserve the world. Do not use evacuation calls as the normal
way to end a round.

## Validation and in-game test procedure

Static validation should check TOML, existing CVars, YAML references, localization,
station inheritance, and the spawn point's association with the Wasteland grid.
Run the full linter with:

```sh
dotnet run --project Content.YAMLLinter
```

To check item insertion through the engine, also run the existing test
(covers all `startingGear` prototypes, including `RustlandsSurvivorGear`):

```sh
dotnet test Content.IntegrationTests --filter FullyQualifiedName~StartingGearPrototypeStorageTest
```

This test checks individual insertion. The procedure below checks the entire kit
simultaneously and actual spawning.

In-game testing is still required:

1. Start the server using the startup command above. Check the Wasteland map,
   Rustlands mode, lobby, and absence of automatic rules using administrative tools.
2. Create a human with a custom name and appearance. Under Rustlands, select
   Survivor at high priority and ready up in the lobby. Start the round:
   the character should appear on the Wasteland grid at `0.5,-1.5` through
   `SpawnPointRustlandsSurvivor`, retaining their name and appearance. Check for
   no antagonist objectives and no spawn/equipment errors in the log.
3. Join with a second human after the round starts and select Survivor:
   the character should use `SpawnPointLatejoin` on the same grid and location,
   without arrivals. Repeat with more players: slots should remain available
   for both entry types. Also test unavailable job preferences with the option
   to join as overflow: the assigned job should be Survivor, never Passenger.
4. For both entry types, check every item and quantity in the table, supplies
   inside the backpack, and no items dropped by failed equipment insertion.
   Check empty ID/ear slots and the absence of access permissions, PDAs, headsets,
   weapons, emergency boxes, and the old loadout, including a profile previously
   configured for Passenger. Open the bottle, drink, eat, and apply gauze to an
   appropriate injury; confirm finite consumption and reduced bleeding.
   Test removing and reinserting items. Check that the supervisor message does
   not refer to station command. Repeat both entry types with Vox: the mask,
   harness, and tank should be equipped, internals functional, and the backpack
   should contain the same basic kit. Humans should not receive this support.
   Check eating behavior according to the species' diet.
5. Explore and test damage, death, hunger/thirst, and interaction. Check that
   players do not receive Sandbox tools and the round does not automatically
   end when everyone dies.
6. Continue beyond 90 minutes: no automatic evacuation call, meteors, or space
   traffic. Also check the profile's effective CVar values.
7. Check that mode/map voting is unavailable. Test restart voting under the
   permitted generic conditions and `restartround`. The next round should retain
   Wasteland/Rustlands and allow new entries.
8. Start separately without the profile and check that generic modes and maps
   remain available. Do not reuse Rustlands overrides for this test.

### Implementation validation results

- Static Survivor validation covered job, tracker, category, and localization
  references; item inheritance, slots, sizes, quantities, and blacklist;
  unlimited round-start/late-join slots and markers on the same grid/location.
- Breathing support: referenced groups/loadouts exist, apply `EffectSpeciesVox`,
  and use existing entities; no `Survival` group.
- `git diff --check`: no errors (only CRLF/LF normalization warnings).
- The attempt to run `dotnet bin/Content.YAMLLinter/Content.YAMLLinter.dll` was
  stopped after approximately seven minutes without output or a result. This
  does not confirm engine validation of the prototypes. The environment already
  had a previous failure recorded in `ResourceCache.PreloadRsis`/ImageSharp;
  this attempt produced no diagnostic linking the delay to the same cause.
  Rerun the linter in a working environment, preferably building from source
  with the command above.
- No game clients or engine storage test were run; the round-start/late-join,
  equipment, and consumption test procedure remains pending.
