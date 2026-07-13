# LootCollector — Agent Instructions

## Project Overview

LootCollector is a **World of Warcraft 3.3.5a (WotLK) Ace3 addon** for the Project Ascension private server. It crowd-sources Worldforged item and Mystic Scroll locations, sharing them in real-time via a hidden chat channel.

- **Language:** Lua 5.1 (WoW embedded)
- **Framework:** Ace3 (AceAddon, AceEvent, AceComm, AceDB, AceSerializer, AceConsole)
- **Target client:** 3.3.5a (Interface: 30300)
- **SavedVariables:** `LootCollectorDB_Asc`

## Build & Run

No build system or test framework exists. Deployment is manual copy to `Interface/AddOns/`. The `.toc` file defines load order — respect it when adding new files.

## Architecture

Load order: Libraries → `LootCollector.lua` → Constants → Zone data → Core logic → Communication → UI modules.

```
LootCollector.lua          -- Main addon object, global alias, profiler
Modules/
  Constants.lua            -- Protocol constants, enums, thresholds
  Scanner.lua              -- Item tooltip scanning
  ZoneList.lua             -- Zone ID/name mappings
  ZoneRelationships.lua    -- Parent/child zone resolution
  Core.lua                 -- Central DB logic (add/remove/migrate discoveries)
  Detect.lua               -- Loot event detection
  Reinforce.lua            -- Periodic confirmation broadcasts
  Decay.lua                -- Stale entry cleanup
  Comm.lua                 -- Network serialization, compression, rate-limiting
  Moderation.lua           -- Sender validation, blacklisting
  DBSync.lua               -- Bulk database sharing (whisper/party/guild)
  ProximityList.lua        -- Nearby discovery UI list
  Map.lua                  -- WorldMap/Minimap pin rendering
  Arrow.lua                -- TomTom navigation integration
  Settings.lua             -- Options panel
  Toast.lua                -- Notification popups
  ImportExport.lua         -- Text-based DB import/export
  Viewer.lua               -- Item browser UI
  MinimapButton.lua        -- Minimap icon
  Tooltip.lua              -- Enhanced tooltip display
```

## Module Pattern

Every module follows this structure:

```lua
local L = LootCollector                                  -- always first line
local MyModule = L:NewModule("MyModule", "AceEvent-3.0") -- register with mixins
local OtherMod = L:GetModule("OtherMod", true)           -- true = silent if absent

function MyModule:OnInitialize()
    if L.LEGACY_MODE_ACTIVE then return end
    -- one-time setup
end

function MyModule:OnEnable()
    if L.LEGACY_MODE_ACTIVE then return end
    -- register events, start timers
end
```

## Key Conventions

| Convention | Detail |
|---|---|
| Addon alias | `local L = LootCollector` — always the first line of every module |
| Legacy guard | Check `if L.LEGACY_MODE_ACTIVE then return end` in all lifecycle hooks |
| Performance upvalues | Localize globals: `local floor = math.floor; local pairs = pairs` |
| Profiler wrapping | `local pTime = L:ProfileStart()` ... `L:ProfileStop("FuncName", pTime)` |
| Debug logging | `L._debug("ModuleName", msg)` — gated by `db.profile.debugMode` |
| Coordinate precision | Always 4 decimal places via `L:Round4(v)` — GUID matching depends on this |
| GUID format | `"continent-zone-innerzone-itemID-x.xxxx-y.xxxx"` via `L:GenerateGUID()` |
| Deferred scheduling | `ScheduleAfter(seconds, func)` — C_Timer.After with OnUpdate fallback |
| Frame-budget loops | Long iterations must check `debugprofilestop()` vs `SCAN_BUDGET_MS` per tick |
| Silent module access | Always pass `true` to `L:GetModule("X", true)` to avoid errors from load-order |

## Communication

- **Internal events** (AceEvent): `L:SendMessage("LootCollector_DiscoveriesUpdated", action, guid, data)` with actions `"add"`, `"remove"`, `"update"`, `"bulk"`, `"clear"`
- **Network protocol**: `LC[1-5]:OP:payload` over hidden channel, prefix `"BBLC25AM"`, channel `"BBLC25C"`
- **Operations**: `DISC` (discovery), `CONF` (confirmation), `ACK` (acknowledgement)
- **Rate limiting**: Token bucket + cooldown queue + deduplication (`ingressSeen`)

## Database

```
L.db (AceDB-3.0)
├── global.realms[realmKey]
│   ├── discoveries {}       -- L:GetDiscoveriesDB()
│   └── blackmarketVendors {}
├── profile                  -- settings, filters, viewer prefs
└── char                     -- per-character: looted{}, hidden{}, pause
```

## Critical Rules

1. **Never modify the DB schema** without a migration script — see [CONTRIBUTING.md](LootCollector/CONTRIBUTING.md)
2. **Always call `L:ActivateRealmBucket()`** before accessing discovery data (or use `L:GetDiscoveriesDB()` which handles it)
3. **Forked versions must change `Constants.ADDON_PREFIX_DEFAULT` and `Constants.CHANNEL_NAME_DEFAULT`** to avoid protocol conflicts
4. **New files must be added to `LootCollector.toc`** in the correct load-order section
5. **No external dependencies** beyond what's bundled in `libs/` — the addon must be self-contained
