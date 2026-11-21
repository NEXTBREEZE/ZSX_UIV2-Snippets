# ZSX_UIV2 – Stress & Drunk for QBox

Snippets that show **Stress** and **Drunk** in the ZSX_UIV2 HUD on a **QBox** (`qbx_core`) server.

Both values come from the **player state bag**:

- `stress` → `Player(source).state.stress` (0–100, also saved in the QBox metadata)
- `drunk` → `Player(source).state.drunk` (0–100)

> Community snippets, not official. **Back up every file before you change it.** A ZSX_UIV2 update can overwrite your changes.

---

## Table of contents

1. [Files you will edit](#1-files-you-will-edit)
2. [Step 1 – HUD config](#2-step-1--hud-config)
3. [Step 2 – Client: drunk getter](#3-step-2--client-drunk-getter)
4. [Step 3 – Server: stress and drunk](#4-step-3--server-stress-and-drunk)
5. [Step 4 – server.cfg](#5-step-4--servercfg)
6. [Using stress and drunk from other scripts](#6-using-stress-and-drunk-from-other-scripts)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. Files you will edit

All paths are relative to the resource folder `ZSX_UIV2/`.

| File | Action |
| --- | --- |
| `shared/ui_cfg/config_hud.lua` | add `stress` and `drunk` |
| `client/addon/status.lua` | add one getter in the QBox part |
| `server/config/qbox_status.lua` | **create** this new file |

You do **not** need to change `fxmanifest.lua`. It already loads `shared/ui_cfg/*.lua`, `client/addon/*.lua` and `server/config/*.lua`.

---

## 2. Step 1 – HUD config

**File:** `ZSX_UIV2/shared/ui_cfg/config_hud.lua`

### 2.1 Add the names to `Config.Hud.Order`

```lua
Config.Hud.Order = {
    'health', 'armour', 'hunger', 'thirst', 'stress', 'drunk', 'stamina', 'oxygen', 'voicestatus'
}
```

Keep the names that are already in your list. Add only `'stress'` and `'drunk'` if they are missing.

### 2.2 Add the status entries

Add these blocks below the other `Config.Hud.Status[...]` entries. If `stress` or `drunk` already exists in your file, replace that block.

```lua
Config.Hud.Status['stress'] = {
    name = 'stress',
    label = 'Stress',
    display = { mode = 'above', threshold = 0, hysteresis = 0 },  -- only show when stress > 0
    warning = { mode = 'above', threshold = 80, hysteresis = 5 }, -- warning from 80 %
    icon = 'fas fa-brain',
    value = 0,
    isVisible = true,
}

Config.Hud.Status['drunk'] = {
    name = 'drunk',
    label = 'Drunk',
    display = { mode = 'above', threshold = 0, hysteresis = 0 },  -- only show when drunk > 0
    warning = { enabled = false },
    icon = 'fas fa-wine-glass-alt',
    value = 0,
    isVisible = true,
}
```

---

## 3. Step 2 – Client: drunk getter

**File:** `ZSX_UIV2/client/addon/status.lua`

**Stress needs no client change.** The QBox part of `status.lua` already reads the `stress` state bag as soon as `Config.Hud.Status['stress']` exists (Step 1).

**Drunk** needs one getter that reads the `drunk` state bag.

Find the QBox part inside `Citizen.CreateThread`. It starts with:

```lua
elseif FrameworkSelected == 'QBOX' then
```

Paste this directly **below** that line:

```lua
        -- DRUNK (QBox state bag)
        if Config.Hud.Status['drunk'] then
            Config.Hud.Status['drunk'].get = function()
                return LocalPlayer.state.drunk or 0
            end
        end
```

Result:

```lua
    elseif FrameworkSelected == 'QBOX' then

        -- DRUNK (QBox state bag)
        if Config.Hud.Status['drunk'] then
            Config.Hud.Status['drunk'].get = function()
                return LocalPlayer.state.drunk or 0
            end
        end

        Config.Hud.Status['hunger'].get = function()
            return tempVars['hunger']
        end

        -- ... rest of the QBox part stays as it is
```

The QBox part runs after the general drunk getter, so this getter replaces it on QBox only.

---

## 4. Step 3 – Server: stress and drunk

**File:** `ZSX_UIV2/server/config/qbox_status.lua` → **create** this new file.

```lua
--[[
    Stress and drunk for QBox (qbx_core)

    Server scripts:
        exports['ZSX_UIV2']:AddStress(source, 10)    -- +10 % stress
        exports['ZSX_UIV2']:AddStress(source, -10)   -- -10 % stress
        exports['ZSX_UIV2']:SetDrunk(source, 60)     -- drunk 60 %
        exports['ZSX_UIV2']:SetDrunk(source, 0)      -- not drunk

    Client scripts:
        TriggerServerEvent('hud:server:GainStress', 10)
        TriggerServerEvent('hud:server:RelieveStress', 10)
]]

while not FrameworkSelected do
    Wait(100)
end

if GetResourceState('qbx_core') ~= 'started' then return end

-- max stress change per call, because the stress events come from the client
local MAX_STRESS_STEP = 25

local function AddStress(source, amount)
    amount = math.max(-MAX_STRESS_STEP, math.min(MAX_STRESS_STEP, tonumber(amount) or 0))

    local player = exports.qbx_core:GetPlayer(source)
    if not player then return end

    local stress = math.max(0, math.min(100, (player.PlayerData.metadata.stress or 0) + amount))
    player.Functions.SetMetaData('stress', stress)       -- saved with the character
    Player(source).state:set('stress', stress, true)     -- read by the HUD
end

local function SetDrunk(source, value)
    value = math.max(0, math.min(100, tonumber(value) or 0))
    Player(source).state:set('drunk', value, true)       -- read by the HUD
end

exports('AddStress', AddStress)
exports('SetDrunk', SetDrunk)

-- qbx_hud handles the same events. Do not register them twice.
if GetResourceState('qbx_hud') ~= 'started' then
    RegisterNetEvent('hud:server:GainStress', function(amount)
        AddStress(source, math.abs(tonumber(amount) or 0))
    end)

    RegisterNetEvent('hud:server:RelieveStress', function(amount)
        AddStress(source, -math.abs(tonumber(amount) or 0))
    end)
end
```

---

## 5. Step 4 – server.cfg

ZSX_UIV2 replaces `qbx_hud`. Remove `ensure qbx_hud`. Start `ZSX_UIV2` after `qbx_core`:

```cfg
ensure qbx_core
ensure ZSX_UIV2
```

Restart the server.

---

## 6. Using stress and drunk from other scripts

### Stress

```lua
-- server script
exports['ZSX_UIV2']:AddStress(source, 10)    -- +10 %
exports['ZSX_UIV2']:AddStress(source, -10)   -- -10 %

-- client script
TriggerServerEvent('hud:server:GainStress', 10)     -- +10 %
TriggerServerEvent('hud:server:RelieveStress', 10)  -- -10 %
```

Many qbx scripts (police job, shooting, smoking, food) already trigger `hud:server:GainStress` / `hud:server:RelieveStress`. They work without changes.

> **Security:** `hud:server:GainStress` and `hud:server:RelieveStress` come from the client. A cheater can trigger them with any value. `MAX_STRESS_STEP` limits the change per call.

### Drunk

```lua
-- server script
exports['ZSX_UIV2']:SetDrunk(source, 60)   -- drunk 60 %
exports['ZSX_UIV2']:SetDrunk(source, 0)    -- not drunk
```

Example: alcohol item that makes the player drunk for 2 minutes (server script of your item resource):

```lua
exports.qbx_core:CreateUseableItem('beer', function(source, item)
    exports['ZSX_UIV2']:SetDrunk(source, 40)

    SetTimeout(120000, function()
        exports['ZSX_UIV2']:SetDrunk(source, 0)
    end)
end)
```

Read the current value:

```lua
-- server
local drunk = Player(source).state.drunk or 0
local stress = Player(source).state.stress or 0

-- client
local drunk = LocalPlayer.state.drunk or 0
local stress = LocalPlayer.state.stress or 0
```

> If your resource folder is not named `ZSX_UIV2`, use your folder name in `exports['<name>']`.

---

## 7. Troubleshooting

**Stress or drunk icon never appears**

- `'stress'` / `'drunk'` is missing in `Config.Hud.Order` or in `Config.Hud.Status`.
- The value is 0. Set `display = { mode = 'always' }` to test.

**Stress or drunk stays at 0**

- Check the server console on start: `[FRAMEWORK] Framework <name> loaded!`. It must say `QBOX`. If it says `QBCore`, the HUD reads `PlayerData.metadata` instead of the state bag.
- `server/config/qbox_status.lua` is missing or has a typo. Check the server console for errors.
- Drunk: the getter from Step 2 is not inside the `QBOX` part of `status.lua`.

**Stress rises twice**

- `qbx_hud` still runs, or another resource handles `hud:server:GainStress`. Stop that resource.

**`No such export AddStress in resource ZSX_UIV2`**

- `server/config/qbox_status.lua` is missing, or the resource folder has another name. Use your folder name in `exports['<name>']`.
