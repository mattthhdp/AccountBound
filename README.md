# AccountBound

AccountBound is an AzerothCore module for WotLK 3.3.5a that shares selected
collections and progression systems between eligible characters on the same
game account.

It combines achievements, mounts, companion pets, titles, reputations,
professions and friends into one module with one configuration file. Every
category can be enabled, disabled and filtered independently.

## Features

- Account-wide achievements with faction conversion support.
- Account-wide mounts with riding, class and faction requirement checks.
- Account-wide vanity companion pets.
- Account-wide titles with faction conversion support.
- Account-wide reputations that only copy higher standings.
- Account-wide profession ranks, skill progress and recipes.
- Configurable primary-profession limit up to all 11 WotLK professions.
- Account-wide friend lists and friend notes.
- Per-category startup backfill for existing realms.
- Silent SQL synchronization without learned-spell or collection spam.
- Character creation seeding before the character's first login.
- Per-category ID allow lists and block lists.
- One master switch and one module configuration file.
- Uses AzerothCore's existing character tables.
- No custom SQL tables are required.

## Compatibility

- AzerothCore WotLK 3.3.5a
- AzerothCore `master`
- `mod-playerbots/azerothcore-wotlk`
- Built as a normal source module under `modules/`
- Tested with a Release worldserver build on Windows

The module uses hooks currently available in AzerothCore. Stock AzerothCore
does not require a core patch.

## Silent Database Synchronization

AccountBound writes shared data directly to AzerothCore's existing character
tables. It does not call `learnSpell()` or replay achievements, titles,
reputation gains or friend notifications during login.

The synchronization lifecycle is:

1. `StartupBackfill` migrates characters that already exist before players can
   log in.
2. `SyncOnCreate` seeds a new character after its initial database save and
   before its first login.
3. Normal gameplay hooks persist newly earned data to the other characters on
   the account.
4. The target character loads the already-persisted rows during its next normal
   login.

This prevents login spam. If a requirement changes while a character is
already online, such as gaining a higher riding rank, the newly eligible rows
are stored immediately and become visible after one relog.

## Installation

Clone the repository into the AzerothCore modules directory:

```bash
cd azerothcore-wotlk/modules
git clone https://github.com/AlsoNotMehh/AccountBound.git
```

Re-run CMake and rebuild `worldserver`.

Copy:

```text
modules/AccountBound/conf/AccountBound.conf.dist
```

to:

```text
configs/modules/AccountBound.conf
```

Restart `worldserver`.

The startup log should contain:

```text
Using modules configuration:
> AccountBound.conf
```

## Quick Config

The master switch controls the entire module:

```ini
AccountBound.Enable = 1
```

Each system has an independent switch:

```ini
AccountBound.Achievements.Enable = 1
AccountBound.Mounts.Enable = 1
AccountBound.Pets.Enable = 1
AccountBound.Titles.Enable = 1
AccountBound.Reputations.Enable = 1
AccountBound.Professions.Enable = 1
AccountBound.Friends.Enable = 1
```

Set any category to `0` to keep it character-specific.

## Shared Data

| Category | Shared data |
| --- | --- |
| Achievements | Completed achievements and completion date |
| Mounts | Learned mount spells |
| Pets | Vanity companion and minipet spells |
| Titles | Known character titles |
| Reputations | Highest eligible standing and reputation flags |
| Professions | Ranks, current/max skill and learned recipes |
| Friends | Friend list entries and notes |

The module intentionally does not share quests, inventory, gold, currencies,
honor, arena rating, instance lockouts, cooldowns, talents, glyphs, action bars,
mail, auctions or hunter stable pets.

Those systems contain character-specific progression, ownership or class data
and should not be merged automatically.

## ID Filters

Achievements, mounts, pets, titles and reputations support:

```ini
AccountBound.CATEGORY.AllowList = all
AccountBound.CATEGORY.BlockList = ""
```

Replace `CATEGORY` with:

```text
Achievements
Mounts
Pets
Titles
Reputations
```

`AllowList = all` permits every valid entry.

`AllowList = none` permits no entries.

A comma-separated allow list shares only those IDs:

```ini
AccountBound.Mounts.AllowList = 458,470,6648
```

The block list always wins:

```ini
AccountBound.Mounts.AllowList = all
AccountBound.Mounts.BlockList = 48778,54729
```

Converted Horde or Alliance equivalents must also pass the configured filter.

## Achievements

Recommended defaults:

```ini
AccountBound.Achievements.Enable = 1
AccountBound.Achievements.SyncOnCreate = 1
AccountBound.Achievements.SyncRealmFirst = 0
AccountBound.Achievements.SyncHidden = 0
AccountBound.Achievements.ConvertFactionSpecific = 1
```

Realm First achievements are excluded by default because they represent a
specific character or realm race.

Faction-specific achievements are converted with AzerothCore's faction-change
mapping when `ConvertFactionSpecific = 1`.

Achievements are inserted into `character_achievement`. They are never replayed
through the live achievement reward path, so the module does not resend
announcements, mail, items or title notifications.

## Mounts

Recommended defaults:

```ini
AccountBound.Mounts.Enable = 1
AccountBound.Mounts.SyncOnCreate = 1
AccountBound.Mounts.RespectFactionRestrictions = 1
AccountBound.Mounts.ConvertFactionSpecific = 1
AccountBound.Mounts.RequireRiding = 1
AccountBound.Mounts.RequireClass = 1
```

The module discovers mount spells from spell data and item templates.

It can enforce:

- Riding skill requirements.
- Class restrictions for class mounts.
- Horde and Alliance restrictions.
- Same-faction restrictions for class mounts.

Invalid shared mount rows can be cleaned at startup:

```ini
AccountBound.Mounts.CleanupInvalid = 1
```

## Companion Pets

The pets category shares vanity companions only:

```ini
AccountBound.Pets.SyncOnCreate = 1
AccountBound.Pets.IncludeCompanionSkillLine = 1
AccountBound.Pets.IncludeMinipetSummons = 1
```

It detects WotLK companion skill-line spells and custom summons using
`SUMMON_TYPE_MINIPET`.

Hunter stable pets, permanent combat pets and pet talent data are not shared.

## Titles

Recommended defaults:

```ini
AccountBound.Titles.Enable = 1
AccountBound.Titles.SyncOnCreate = 1
AccountBound.Titles.SyncOnSave = 1
AccountBound.Titles.SyncRealmFirst = 0
AccountBound.Titles.ConvertFactionSpecific = 1
```

Realm First reward titles are excluded by default. Faction-specific titles are
converted using AzerothCore's faction-change title mappings.

## Reputations

Recommended public-realm defaults:

```ini
AccountBound.Reputations.Enable = 1
AccountBound.Reputations.SyncOnCreate = 1
AccountBound.Reputations.SyncOnChange = 1
AccountBound.Reputations.SameFactionOnly = 1
AccountBound.Reputations.ConvertFactionSpecific = 1
```

Reputation sync is monotonic: it copies a standing only when it improves the
target character. It never lowers another character's standing.

Base reputation differences for race and class are respected. Cross-faction
sharing can convert supported faction pairs.

## Professions

The default configuration allows all 11 WotLK primary professions:

```ini
AccountBound.Professions.MaxPrimaryProfessions = 11
AccountBound.Professions.Primary.AllowList = all
AccountBound.Professions.Primary.BlockList = ""
```

Accepted English profession names:

```text
alchemy, blacksmithing, enchanting, engineering, herbalism,
inscription, jewelcrafting, leatherworking, mining, skinning, tailoring
```

Accepted Spanish aliases:

```text
alquimia, herreria, encantamiento, ingenieria, herboristeria,
inscripcion, joyeria, peleteria, mineria, desuello, sastreria
```

Skill IDs are also accepted:

```text
171,164,333,202,182,773,755,165,186,393,197
```

Example:

```ini
AccountBound.Professions.Primary.AllowList = alchemy, enchanting, tailoring
AccountBound.Professions.MaxPrimaryProfessions = 3
```

Strict allow-list enforcement is disabled by default:

```ini
AccountBound.Professions.Primary.EnforceAllowedList = 0
```

Warning: enabling it can remove blocked professions from existing characters.

Account-wide profession settings:

```ini
AccountBound.Professions.SyncOnCreate = 1
AccountBound.Professions.SyncSkillProgress = 1
AccountBound.Professions.SyncProfessionRanks = 1
AccountBound.Professions.SyncRecipes = 1
AccountBound.Professions.SyncSkillGrantedSpells = 1
AccountBound.Professions.IncludeSecondaryProfessions = 1
AccountBound.Professions.RequireRecipeSkill = 1
```

Custom recipe spells can be explicitly allowed or blocked:

```ini
AccountBound.Professions.SpellAllowList = 900001,900002
AccountBound.Professions.SpellBlockList = 20219,20222
```

## Friends

The friends category mirrors the normal character friend list across every
character on an account:

```ini
AccountBound.Friends.SyncOnCreate = 1
AccountBound.Friends.SyncOnlineChanges = 1
AccountBound.Friends.SyncIntervalSeconds = 3
```

Friend notes are shared. Characters belonging to the same account are removed
from the friend list to avoid self-friend entries.

Cross-faction friends are allowed by default:

```ini
AccountBound.Friends.SameFactionOnly = 0
```

## Startup Backfill

Every progression category has a `StartupBackfill` setting.

Example:

```ini
AccountBound.Achievements.StartupBackfill = 1
AccountBound.Mounts.StartupBackfill = 1
AccountBound.Pets.StartupBackfill = 1
AccountBound.Titles.StartupBackfill = 1
AccountBound.Reputations.StartupBackfill = 1
AccountBound.Professions.StartupBackfill = 1
AccountBound.Friends.StartupBackfill = 1
```

For an existing realm:

1. Back up the characters database.
2. Enable the categories you want.
3. Enable their startup backfill options.
4. Start `worldserver` once.
5. Wait for every AccountBound startup message.
6. Stop `worldserver`.
7. Set the startup backfill options back to `0`.
8. Start the server normally.

Backfill uses idempotent inserts or monotonic updates, but it can touch many
rows on a large realm.

Version 1.1 replaces the old `SyncOnLogin` options with `SyncOnCreate`.
Existing custom configuration files should be updated to use the new keys.

## Safe Public-Realm Defaults

```ini
AccountBound.Enable = 1

AccountBound.Achievements.SyncRealmFirst = 0
AccountBound.Achievements.SyncHidden = 0

AccountBound.Mounts.RespectFactionRestrictions = 1
AccountBound.Mounts.RequireRiding = 1
AccountBound.Mounts.RequireClass = 1

AccountBound.Titles.SyncRealmFirst = 0

AccountBound.Reputations.SameFactionOnly = 1
AccountBound.Reputations.SyncUnpairedCrossFaction = 0

AccountBound.Professions.Primary.EnforceAllowedList = 0
AccountBound.Professions.RequireRecipeSkill = 1

AccountBound.Friends.StartupBackfill = 1
```

After the initial migration, keep every broad `StartupBackfill` option disabled.

## Database Tables Used

The module uses normal AzerothCore tables:

```text
characters
character_achievement
character_spell
character_skills
character_reputation
character_social
```

No custom tables or SQL installation files are required.

## Troubleshooting

If the module is not loaded:

- Confirm the folder is `modules/AccountBound`.
- Re-run CMake after adding the module.
- Rebuild `worldserver`.
- Confirm `configs/modules/AccountBound.conf` exists.
- Confirm `AccountBound.Enable = 1`.

If a mount is not shared:

- Confirm the spell passes `AllowList` and `BlockList`.
- Check riding, class and faction requirements.
- Check `RequireRiding`, `RequireClass` and `RespectFactionRestrictions`.

If a profession recipe is not shared:

- Confirm the profession is allowed.
- Confirm `SyncRecipes = 1`.
- Confirm the target has sufficient skill when `RequireRecipeSkill = 1`.
- Add custom recipes to `SpellAllowList`.

If data was inserted while the target character was already online:

- Relog once so AzerothCore reloads the updated character tables.

If startup takes longer than normal:

- Disable broad `StartupBackfill` options after the initial migration.

## Files

```text
AccountBound/
  conf/
    AccountBound.conf.dist
  src/
    AccountBound.h
    Achievements.cpp
    Friends.cpp
    Mounts.cpp
    Pets.cpp
    Professions.cpp
    Reputations.cpp
    Titles.cpp
    loader.cpp
  CHANGELOG.md
  LICENSE
  README.md
```

## License

AccountBound is distributed under GPL-2.0 to match the AzerothCore core
license.
