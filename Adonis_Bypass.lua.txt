--[[
    Syphon — Adonis Anti-Cheat Bypass (standalone, for review)
    ---------------------------------------------------------
    Single-chokepoint neutralization of the Adonis client anticheat.

    HOW IT WORKS
    ------------
    Every Adonis detector (movement, metamethod, anti-kick, remote-spy,
    string-scan, etc.) funnels through ONE client-side closure: `Detected`
    (Adonis/MainModule/Client/Core/Anti.luau:81). That closure fires an
    encrypted remote at the server ("Detected") and then locally
    kicks/kills/crashes the player. If `Detected` is a no-op, ALL reporting
    and ALL local punishment stops.

    Adonis actively self-defends `Detected` via two checks:
      1. Signature self-check (Anti.luau:417-429): compares
         debug.info(Detected,"slanf") against a golden copy captured at
         startup; requires arity==3, non-vararg, closure identity intact.
         A mismatch -> `while true do end` hang.
      2. TableCheck thread (Anti.luau:661-672): independently re-reads
         client.Anti.Detected and identity-checks it -> crash.

    THE KEY INSIGHT (why this doesn't freeze the game):
      On a modern executor `hookfunction` patches the closure IN PLACE.
      The closure reference (`closure == Detected`) and its debug.info
      signature both stay identical automatically — which is exactly what
      both checks require. So we need NO global debug.info hook.
      (An earlier version hooked getrenv().debug.info globally with a Lua
       trampoline; debug.info is on every hot path — Adonis's loops, its
       metamethod detectors, and Roblox internals — so that hook multiplied
       into a render-thread freeze. This version removes it entirely.)

    We verify transparency up front and only fall back to a tiny,
    non-recursive spoof if the executor is NOT transparent (rare).
]]

-- Services
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

local function log(...) warn("[Bypass]", ...) end

-- ===== The bypass =====
local bypassArmed = false

local function armAdonisBypass()
    local getType = typeof or type

    -- Capability detection (defensive — every path still works if absent).
    local hasHookfunction  = getType(hookfunction) == "function"
    local hasNewcclosure   = getType(newcclosure) == "function"
    local hasIshooked      = getType(isfunctionhooked) == "function"
    local hasGetgc         = getType(getgc) == "function"
    local hasClonefunction= getType(clonefunction) == "function"
    local hasGetrenv       = getType(getrenv) == "function"
    local debug_info       = debug.info or (debug.getinfo and function(f, what)
        local di = debug.getinfo(f, what)
        if not di then return nil end
        return di.source, di.currentline, di.name, di.nparams
    end)

    -- 1) Locate the client `Detected` closure via GC.
    --    The Adonis Anti table is a ReadOnly proxy whose __index returns a
    --    table carrying Detected/Detectors/RLocked/AddDetector. We look for
    --    any table exposing those keys; the function is the chokepoint.
    local ANTI_KEYS = { "Detected", "Detectors", "RLocked", "AddDetector" }
    local function rawHas(t, keys)
        for _, x in ipairs(keys) do
            if rawget(t, x) == nil then return false end
        end
        return true
    end

    local function findDetected()
        if not hasGetgc then return nil end
        local gc = getgc(true)
        for i = 1, #gc do
            local ok, v = pcall(function() return gc[i] end)
            if ok and getType(v) == "table" and rawHas(v, ANTI_KEYS) then
                local d = rawget(v, "Detected")
                if getType(d) == "function" then return d end
            end
            if i % 1024 == 0 then task.wait() end  -- breathe; never stall render
        end
        return nil
    end

    -- Retry until Adonis is loaded and Detected is found.
    local detected, attempt = nil, 0
    while not detected do
        attempt += 1
        local ok, d = pcall(findDetected)
        if ok and d then
            detected = d
        else
            if attempt == 1 or attempt % 10 == 0 then
                log("Detected closure not found, attempt", attempt)
            end
            task.wait(1)
        end
    end

    -- 2) Capture the GOLDEN debug.info signature BEFORE hooking.
    --    Adonis caches ("s","l","a","n","f") on startup and re-reads them
    --    every ~5s; they must remain identical after the hook.
    local g_source, g_line, g_argN, g_name, g_ref
    pcall(function()
        g_source, g_line, g_argN, g_name = debug_info(detected, "slan")
        g_ref = debug_info(detected, "f")
    end)

    -- 3) Neutralize `Detected` in place.
    --    Arity 3, non-vararg, returns true (Adonis' liveness probe at
    --    Anti.luau:425 requires the "_" action to return true).
    --    hookfunction keeps closure identity + signature intact on every
    --    modern executor, so the self-check and TableCheck both pass.
    local hookedRef = detected
    if hasHookfunction and hasNewcclosure then
        local noop = newcclosure(function(action, info, nocrash)
            return true  -- swallow every report; never kick/kill/crash
        end)
        if not (hasIshooked and isfunctionhooked(detected)) then
            local ok = pcall(hookfunction, detected, noop)
            if not ok then log("hookfunction(Detected) failed") end
        end
    else
        log("hookfunction/newcclosure unavailable — cannot neutralize Detected")
    end

    -- 4) Verify transparency. If the signature survived, we are done and
    --    need NO global hook (the freeze-free path).
    local transparent = true
    pcall(function()
        if g_source ~= nil then
            local s2, l2, a2, n2 = debug_info(detected, "slan")
            local r2 = debug_info(detected, "f")
            transparent = (s2 == g_source and l2 == g_line and a2 == g_argN
                and n2 == g_name and r2 == hookedRef)
        end
    end)

    -- 5) Fallback ONLY if non-transparent: install a MINIMAL debug.info
    --    spoof. Key differences from the old freeze-causing hook:
    --      * clone the ORIGINAL debug.info first -> C-closure speed, zero
    --        Lua-side recursion (forwarding through the clone never
    --        re-enters our own hook).
    --      * fast-path everything except the exact `detected` reference.
    --      * no per-char string parsing in the hot path.
    if not transparent and hasGetrenv then
        local renv = getrenv()
        local origInfo = renv.debug.info
        local cloned = hasClonefunction and clonefunction(origInfo) or origInfo

        local spoof = function(target, what, ...)
            if target == hookedRef then
                local out = {}
                local len = #what
                for i = 1, len do
                    local c = what:byte(i)
                    if     c == 0x73 then out[i] = g_source   -- 's'
                    elseif c == 0x6C then out[i] = g_line     -- 'l'
                    elseif c == 0x61 then out[i] = g_argN     -- 'a'
                    elseif c == 0x6E then out[i] = g_name     -- 'n'
                    elseif c == 0x66 then out[i] = hookedRef  -- 'f'
                    elseif c == 0x75 then out[i] = 10         -- 'u'
                    end
                end
                return table.unpack(out, 1, len)
            end
            return cloned(target, what, ...)
        end

        if hasHookfunction and hasNewcclosure
           and not (hasIshooked and isfunctionhooked(origInfo)) then
            pcall(hookfunction, origInfo, newcclosure(spoof))
            log("debug.info spoof installed (non-transparent executor)")
        end
    else
        log("hookfunction is signature-transparent — no debug.info hook needed")
    end

    bypassArmed = true
    log("Adonis bypass armed after", attempt, "attempt(s) | transparent=", transparent)
    return true
end

-- Run it (off the render thread).
task.spawn(armAdonisBypass)

-- Expose the armed state so other code can gate on it.
return function() return bypassArmed end
