# Alternative Aura Detection Methods for WoW 12.0

When `C_UnitAuras.GetAuraDataByIndex()` returns secret values, these alternatives may bypass restrictions.

## Method 1: Direct Spell ID Lookup (BEST)

**API:** `C_UnitAuras.GetPlayerAuraBySpellID(spellID)`

```lua
-- Check if player has specific buff by spell ID
local function HasBuffBySpellID_Direct(spellID)
    if not C_UnitAuras or not C_UnitAuras.GetPlayerAuraBySpellID then
        return false
    end
    
    local auraData = C_UnitAuras.GetPlayerAuraBySpellID(spellID)
    if not auraData then return false end
    
    -- Check if result is a secret
    if issecretvalue and issecretvalue(auraData.spellId) then
        return nil  -- Can't determine
    end
    
    return true  -- Has the buff
end
```

**Advantages:**
- Direct lookup, no iteration
- Only queries specific spell we care about
- May be exempt from secret restrictions (targeted query vs bulk query)

**Disadvantages:**
- Still might return secrets (needs testing)
- Only works for known spell IDs

## Method 2: Slot-Based Access

**API:** `C_UnitAuras.GetAuraSlots()` + `C_UnitAuras.GetAuraDataBySlot()`

```lua
-- Use slot system instead of index iteration
local function RefreshAuraCache_Slots()
    local cachedAuras = {}
    cachedAuras.byID = {}
    
    if not C_UnitAuras or not C_UnitAuras.GetAuraSlots then
        return cachedAuras
    end
    
    -- Get all aura slots for helpful buffs
    local continuationToken, slots = C_UnitAuras.GetAuraSlots("player", "HELPFUL")
    
    if not slots then
        return cachedAuras
    end
    
    -- Iterate through slots
    for i = 1, #slots do
        local slot = slots[i]
        local auraData = C_UnitAuras.GetAuraDataBySlot("player", slot)
        
        if auraData and auraData.spellId then
            -- Check for secrets
            if issecretvalue and issecretvalue(auraData.spellId) then
                cachedAuras.hasSecrets = true
                break
            end
            
            cachedAuras.byID[auraData.spellId] = true
        end
    end
    
    return cachedAuras
end
```

**Advantages:**
- Different code path than GetAuraDataByIndex
- Might have different secret restrictions

**Disadvantages:**
- Still likely returns secrets
- More complex API

## Method 3: Spell Name Lookup

**API:** `C_UnitAuras.GetAuraDataBySpellName(unit, spellName, filter)`

```lua
-- Check by spell name instead of ID
local function HasBuffByName_Direct(spellName)
    if not C_UnitAuras or not C_UnitAuras.GetAuraDataBySpellName then
        return false
    end
    
    local auraData = C_UnitAuras.GetAuraDataBySpellName("player", spellName, "HELPFUL")
    if not auraData then return false end
    
    -- Check for secrets
    if issecretvalue and (issecretvalue(auraData.name) or issecretvalue(auraData.spellId)) then
        return nil  -- Can't determine
    end
    
    return true
end
```

**Advantages:**
- Bypasses spell ID lookup entirely
- Works with localized names

**Disadvantages:**
- Names are localized (not portable across regions)
- Still might return secrets

## Method 4: Weapon Buff Detection

**API:** `GetWeaponEnchantInfo()` (for imbues/enchants)

```lua
-- Already implemented for Shaman weapon imbues
-- This API appears to NOT return secrets
local function HasActiveWeaponEnchant()
    if not GetWeaponEnchantInfo then return false end
    
    local hasMainHand, mainHandExpiration = GetWeaponEnchantInfo()
    
    return hasMainHand and (not mainHandExpiration or mainHandExpiration > 10000)
end
```

**Advantages:**
- Confirmed NOT secret in 12.0
- Reliable for weapon enchants

**Disadvantages:**
- Only works for weapon enchants
- Limited use case

## Method 5: Spell Overlay Detection (Proc Detection)

**API:** `C_SpellActivationOverlay.IsSpellOverlayed(spellID)`

```lua
-- Check if spell has proc glow (implies buff is active)
local function IsSpellProccedOrActive(spellID)
    if not C_SpellActivationOverlay or not C_SpellActivationOverlay.IsSpellOverlayed then
        return false
    end
    
    local hasProc = C_SpellActivationOverlay.IsSpellOverlayed(spellID)
    
    -- Check for secrets
    if issecretvalue and issecretvalue(hasProc) then
        return nil
    end
    
    return hasProc
end
```

**Advantages:**
- Different system entirely (visual effects, not auras)
- May bypass aura restrictions

**Disadvantages:**
- Only works for procced abilities
- Doesn't tell us about normal buffs

## Method 6: Combat Log Parsing

**Event:** `COMBAT_LOG_EVENT_UNFILTERED`

⚠️ **NOT AVAILABLE IN 12.0** - Combat log access is also restricted by secrets

## Recommended Strategy

**Layered approach** - try methods in order until one works:

```lua
local function HasBuffBySpellID_Smart(spellID, spellName)
    -- Try 1: Direct lookup by spell ID (best if not secret)
    if C_UnitAuras and C_UnitAuras.GetPlayerAuraBySpellID then
        local auraData = C_UnitAuras.GetPlayerAuraBySpellID(spellID)
        if auraData then
            if not (issecretvalue and issecretvalue(auraData.spellId)) then
                return true  -- Has buff, not secret
            end
            -- Fall through if secret
        else
            return false  -- Definitely doesn't have it
        end
    end
    
    -- Try 2: Lookup by spell name (if provided and different API path)
    if spellName and C_UnitAuras and C_UnitAuras.GetAuraDataBySpellName then
        local auraData = C_UnitAuras.GetAuraDataBySpellName("player", spellName, "HELPFUL")
        if auraData then
            if not (issecretvalue and (issecretvalue(auraData.spellId) or issecretvalue(auraData.name))) then
                return true  -- Has buff, not secret
            end
        else
            return false  -- Doesn't have it
        end
    end
    
    -- Try 3: Fallback to index iteration (current method)
    -- This will set hasSecrets flag if blocked
    return HasBuffBySpellID_Index(spellID)
end
```

## Testing Priority

1. **GetPlayerAuraBySpellID** - Most promising, direct lookup by spell ID
2. **GetAuraDataBySpellName** - Alternative using spell names
3. **Slot-based access** - Different iteration method
4. **Hardcoded filtering** - Current fallback (hide common buffs when secrets detected)

## Implementation Notes

- Test GetPlayerAuraBySpellID first - most likely to bypass secrets (targeted query)
- If that fails, try spell name lookup (different code path)
- Slot-based access is unlikely to help (same underlying data)
- Combat log is NOT available (also restricted by secrets)
- Current fallback: whitelist filtering + hardcoded buff lists
