--[[
    Dragon Ware — Facility UI Edition
    MM2 only.
    Ауры через MeshPart-torus. Silent Aim встроен в "Shoot Murderer".
    + Fun Tab: tracers, dodge, char fx, fun actions, bullet tracers, DW users
    + Меню по LeftControl
]]

-- ─────────────── Services ───────────────
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Lighting         = game:GetService("Lighting")
local TeleportService  = game:GetService("TeleportService")
local VirtualUser      = game:GetService("VirtualUser")
local SoundService     = game:GetService("SoundService")
local Debris           = game:GetService("Debris")
local VIM              = game:GetService("VirtualInputManager")
local CoreGui          = game:GetService("CoreGui")
local RS               = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local Camera      = workspace.CurrentCamera
local Mouse       = LocalPlayer:GetMouse()

-- ─────────────── Game Guard ───────────────
local ALLOWED_PLACES = { [142823291] = "Murder Mystery 2" }
if not ALLOWED_PLACES[game.PlaceId] then
    local list = ""
    for _, n in pairs(ALLOWED_PLACES) do list = list .. " • " .. n .. "\n" end
    pcall(function()
        LocalPlayer:Kick("\n[Dragon Ware]\n\nOnly works in:\n" .. list ..
            "\nPlaceId: " .. tostring(game.PlaceId))
    end)
    return
end

-- ─────────────── Facility UI ───────────────
local BASE = "https://raw.githubusercontent.com/FacilityHUB/UI-Facility/refs/heads/main/"
local src = game:HttpGet(BASE .. "main.luau")
assert(#src > 1000 and not string.find(src, "404", 1, true), "Facility main.luau not found")

local chunk, err = loadstring(src)
assert(chunk, "Facility compile error: " .. tostring(err))

local Library = chunk()
assert(type(Library) == "table", "Facility returned no table")
Library.BaseUrl = BASE

local SaveManager      = Library.SaveManager
local InterfaceManager = Library.InterfaceManager
local ThemeManager     = Library.ThemeManager

-- ─────────────── State ───────────────
local DW = {
    Connections = {}, Cleanups = {},
    flying = false, flinging = false, noclip = false, mm2Running = true,

    speedOn = false, speedVal = 16,
    jumpOn = false, jumpVal = 50,
    glitchOn = false, glitchSpeed = 60, glitchMode = "Jump", glitchMethod = "Velocity",
    infJump = false,
    defaultGravity = workspace.Gravity,

    spinOn = false, spinSpeed = 2200, spinAxis = "Y",
    flySpeed = 60, flyBV = nil, flyBG = nil,

    defaultFOV = workspace.CurrentCamera.FieldOfView,
    fovOn = false, fovVal = 70,
    fullbright = false,
    origLighting = {
        Brightness = Lighting.Brightness, ClockTime = Lighting.ClockTime,
        FogEnd = Lighting.FogEnd, GlobalShadows = Lighting.GlobalShadows,
        Ambient = Lighting.Ambient,
    },

    roleData = {}, fetching = false,
    roleOn = { Murderer = false, Sheriff = false, Innocent = false },
    roleColor = {
        Murderer = Color3.fromRGB(255, 60, 60),
        Sheriff  = Color3.fromRGB(70, 140, 255),
        Innocent = Color3.fromRGB(70, 230, 120),
    },
    showNames = true,
    espObjs = {}, watchedHums = {},
    gunDrop = nil, grabbing = false, autoGrab = false,

    shooting = false,
    prediction = 0.08,
    aimPart = "Head",
    wallbangOn = false,

    -- SILENT AIM
    silentAimOn = false,
    silentAimMethod = "All",
    bulletSpeed = 2500,
    silentPrediction = 0.08,
    cachedTargetPos = nil,
    cachedTargetPart = nil,
    oldNamecall = nil,
    oldIndex = nil,

    kauraOn = false, kauraDelay = 0, kauraRadius = 12,
    kauraMethod = "Activate", kauraIgnoreSheriff = true,
    kauraTarget = "Nearest", kauraLastHit = 0,

    killSoundOn = false, killSoundVolume = 1.5, killSoundOnlyMe = false,
    killSoundPack = "Neverlose", killSoundCustomId = "",

    flingPower = 80000, flingTime = 1.8, flingMethod = "Both",

    auraOn = false,
    auraColor = Color3.fromRGB(179, 165, 255),
    auraSize = 4, auraThickness = 0.6, auraTransparency = 0.5,
    auraRainbow = false, auraSpin = true, auraSpeed = 3,
    auraParticles = true, auraLight = true, auraObjects = {},

    killFxOn = false, killFxType = "Explosion",
    killFxColor = Color3.fromRGB(255, 80, 80), killFxSize = 8, killFxRainbow = false,

    crossOn = false, crossStyle = "Cross",
    crossColor = Color3.fromRGB(255, 255, 255),
    crossSize = 12, crossThickness = 2, crossGap = 4, crossOutline = true,

    antiflingOn = false, antiflingThreshold = 100,
    antiflingSnap = true, antiflingLastPos = nil, antiflingLastTime = 0,

    fpsBoostOn = false, fpsBoostLevel = "Medium", fpsBoostClean = {},

    tradeRemotes = {}, selectedPlayer = nil, whitelist = {},

    bombJumpPower = 250, bombJumpHorizontal = 0, bombJumpAuto = false,

    bundleAnimId = "", bundleAnimPlaying = false, bundleAnimTrack = nil,

    buttonSize = 130, onScreen = {}, savedPositions = {},

    killAllOn = false, killAllInterval = 0.5, killAllLast = 0, killAllRange = 500,

    antiAfk = true,

    -- FUN: tracers (общие ESP-трассы)
    tracersOn = false, tracerMurderer = true, tracerSheriff = false,
    tracerColorM = Color3.fromRGB(255, 60, 60),
    tracerColorS = Color3.fromRGB(70, 140, 255),
    tracerThickness = 2, tracerObjs = {},

    dodgeOn = false, dodgeRadius = 30, dodgePower = 60,

    charFireOn = false, charRainbowOn = false,
    charSparklesOn = false, charLightOn = false, rainbowHue = 0,

    -- BULLET TRACERS
    bulletTracersOn    = false,
    bulletTracerColor  = Color3.fromRGB(255, 230, 100),
    bulletTracerThick  = 0.15,
    bulletTracerLife   = 0.35,
    tracerFolder       = nil,

    -- DRAGON WARE USERS
    dwUsersOn    = false,
    dwKorbloxOn  = true,
    dwHeadlessOn = true,
    dwUsers      = {},
    dwVisuals    = {},
}

local Connections = DW.Connections
local Cleanups = DW.Cleanups

local function track(conn)
    table.insert(Connections, conn)
    return conn
end

-- ─────────────── Helpers ───────────────
local function getChar() return LocalPlayer.Character end
local function getHum()
    local c = getChar()
    return c and c:FindFirstChildOfClass("Humanoid")
end
local function getRoot()
    local c = getChar()
    return c and c:FindFirstChild("HumanoidRootPart")
end
local function guiParent()
    local ok, h = pcall(function() return gethui() end)
    return (ok and h) or CoreGui
end
local function notify(title, desc, dur, style)
    pcall(function()
        Library:Notify({
            Title = title,
            Text = desc or "",
            Duration = dur or 3,
            Style = style or "accent",
        })
    end)
end

-- ─────────────── Role / Tools ───────────────
local function hasTool(plr, name)
    local char = plr.Character
    local pack = plr:FindFirstChildOfClass("Backpack")
    return (char and char:FindFirstChild(name)) or (pack and pack:FindFirstChild(name)) or nil
end

local function getRole(plr)
    local d = DW.roleData[plr.Name]
    if type(d) == "table" and type(d.Role) == "string" then
        local r = d.Role
        if r == "Hero" then r = "Sheriff" end
        if r == "Murderer" or r == "Sheriff" then return r end
        return "Innocent"
    end
    if hasTool(plr, "Knife") then return "Murderer" end
    if hasTool(plr, "Gun") or hasTool(plr, "Revolver") then return "Sheriff" end
    return "Innocent"
end

local function getMurderer()
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and getRole(plr) == "Murderer" then return plr end
    end
end
local function getSheriff()
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and getRole(plr) == "Sheriff" then return plr end
    end
end

local function refreshRoles()
    if DW.fetching then return end
    DW.fetching = true
    task.spawn(function()
        local ok, data = pcall(function()
            return RS.Remotes.Extras.GetPlayerData:InvokeServer()
        end)
        if ok and type(data) == "table" then DW.roleData = data end
        DW.fetching = false
    end)
end

-- ══════════════════════════════════════════════════════════════
-- ═══ BULLET TRACERS + DRAGON WARE USER DETECTION ═══
-- ══════════════════════════════════════════════════════════════

local tracerFolder = Instance.new("Folder")
tracerFolder.Name = "DragonWareBulletTracers"
tracerFolder.Parent = workspace
DW.tracerFolder = tracerFolder
table.insert(Cleanups, function() tracerFolder:Destroy() end)

local function drawBulletTracer(from, to, color)
    if not DW.bulletTracersOn then return end
    if not from or not to then return end
    local delta = to - from
    local dist = delta.Magnitude
    if dist < 0.5 then return end

    local part = Instance.new("Part")
    part.Name = "DWTracer"
    part.Anchored = true
    part.CanCollide = false
    part.CanQuery = false
    part.CanTouch = false
    part.CastShadow = false
    part.Material = Enum.Material.Neon
    part.Color = color or DW.bulletTracerColor
    part.Size = Vector3.new(DW.bulletTracerThick, DW.bulletTracerThick, dist)
    part.CFrame = CFrame.lookAt((from + to) / 2, to)
    part.Transparency = 0
    part.Parent = tracerFolder

    task.spawn(function()
        local life = DW.bulletTracerLife
        local steps = 10
        for i = 1, steps do
            if not part.Parent then return end
            part.Transparency = i / steps
            task.wait(life / steps)
        end
        pcall(function() part:Destroy() end)
    end)
end
DW.drawBulletTracer = drawBulletTracer

-- ---- Маркер DW-юзера на себе ----
local function markSelfAsDwUser()
    local char = getChar()
    if not char then return end
    pcall(function() char:SetAttribute("DragonWareUser", true) end)
    if not char:FindFirstChild("DragonWareMarker") then
        local marker = Instance.new("BoolValue")
        marker.Name = "DragonWareMarker"
        marker.Value = true
        marker.Parent = char
    end
end
markSelfAsDwUser()
track(LocalPlayer.CharacterAdded:Connect(function()
    task.wait(0.3); markSelfAsDwUser()
end))

local function isDwUser(plr)
    if plr == LocalPlayer then return true end
    local char = plr.Character
    if not char then return false end
    if char:GetAttribute("DragonWareUser") then return true end
    if char:FindFirstChild("DragonWareMarker") then return true end
    return false
end

local KORBLOX_MESH = "rbxassetid://1394481756"

local function removeDwVisuals(plr)
    local v = DW.dwVisuals[plr]
    if not v then return end
    if v.korblox then pcall(function() v.korblox:Destroy() end) end
    if v.headRefs then
        for _, ref in ipairs(v.headRefs) do
            pcall(function() ref.obj.Transparency = ref.old end)
        end
    end
    if v.legRefs then
        for _, ref in ipairs(v.legRefs) do
            pcall(function() ref.obj.Transparency = ref.old end)
        end
    end
    DW.dwVisuals[plr] = nil
end

local function applyDwVisuals(plr)
    local char = plr.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end

    local v = DW.dwVisuals[plr]
    if v and v.char == char then return end
    removeDwVisuals(plr)
    v = { char = char, headRefs = {}, legRefs = {} }

    if DW.dwHeadlessOn then
        local head = char:FindFirstChild("Head")
        if head and head:IsA("BasePart") then
            table.insert(v.headRefs, { obj = head, old = head.Transparency })
            head.Transparency = 1
            for _, h in ipairs(head:GetChildren()) do
                if h:IsA("Decal") or h:IsA("SpecialMesh") then
                    if h:IsA("Decal") then
                        table.insert(v.headRefs, { obj = h, old = h.Transparency })
                        h.Transparency = 1
                    end
                end
            end
        end
        for _, acc in ipairs(char:GetChildren()) do
            if acc:IsA("Accessory") then
                local handle = acc:FindFirstChild("Handle")
                if handle and handle:IsA("BasePart") then
                    table.insert(v.headRefs, { obj = handle, old = handle.Transparency })
                    handle.Transparency = 1
                end
            end
        end
    end

    if DW.dwKorbloxOn then
        for _, name in ipairs({"RightUpperLeg", "RightLowerLeg", "RightFoot"}) do
            local p = char:FindFirstChild(name)
            if p and p:IsA("BasePart") then
                table.insert(v.legRefs, { obj = p, old = p.Transparency })
                p.Transparency = 1
            end
        end

        local upperLeg = char:FindFirstChild("RightUpperLeg")
        if upperLeg then
            local bone = Instance.new("MeshPart")
            bone.Name = "DW_KorbloxBone"
            bone.MeshId = KORBLOX_MESH
            bone.TextureId = ""
            bone.Anchored = true
            bone.CanCollide = false
            bone.CanQuery = false
            bone.CanTouch = false
            bone.CastShadow = false
            bone.Massless = true
            bone.Material = Enum.Material.SmoothPlastic
            bone.Color = Color3.fromRGB(20, 20, 20)
            bone.Size = Vector3.new(1, 2.5, 1)
            bone.CFrame = upperLeg.CFrame * CFrame.new(0, -1.2, 0)
            bone.Parent = tracerFolder
            v.korblox = bone
        end
    end

    DW.dwVisuals[plr] = v
end

track(RunService.RenderStepped:Connect(function()
    for plr, v in pairs(DW.dwVisuals) do
        if v.korblox and v.char then
            local leg = v.char:FindFirstChild("RightUpperLeg")
            if leg and leg.Parent and v.korblox.Parent then
                v.korblox.CFrame = leg.CFrame * CFrame.new(0, -1.2, 0)
            end
        end
    end
end))

task.spawn(function()
    while DW.mm2Running do
        if DW.dwUsersOn then
            for _, plr in ipairs(Players:GetPlayers()) do
                if plr == LocalPlayer then continue end
                if isDwUser(plr) then
                    DW.dwUsers[plr] = true
                    if plr.Character then
                        applyDwVisuals(plr)
                    end
                else
                    if DW.dwUsers[plr] then
                        DW.dwUsers[plr] = nil
                        removeDwVisuals(plr)
                    end
                end
            end
        else
            if next(DW.dwVisuals) then
                for plr in pairs(DW.dwVisuals) do removeDwVisuals(plr) end
            end
            table.clear(DW.dwUsers)
        end
        task.wait(0.4)
    end
end)

track(Players.PlayerRemoving:Connect(function(plr)
    DW.dwUsers[plr] = nil
    removeDwVisuals(plr)
end))

track(Players.PlayerAdded:Connect(function(plr)
    track(plr.CharacterAdded:Connect(function()
        task.wait(0.5)
        if DW.dwUsersOn and isDwUser(plr) then
            applyDwVisuals(plr)
        end
    end))
end))

-- ══════════════════════════════════════════════════════════════
-- ═══ SILENT AIM core ═══
-- ══════════════════════════════════════════════════════════════
local GRAVITY = Vector3.new(0, -workspace.Gravity, 0)

track(RunService.Heartbeat:Connect(function()
    if not DW.silentAimOn then
        DW.cachedTargetPos = nil
        DW.cachedTargetPart = nil
        return
    end
    local murd = getMurderer()
    if murd and murd.Character then
        local tHrp = murd.Character:FindFirstChild("HumanoidRootPart")
        local myRoot = getRoot()
        if tHrp and myRoot then
            local origin = myRoot.Position
            local targetPos = tHrp.Position
            local targetVel = tHrp.AssemblyLinearVelocity
            local dist = (targetPos - origin).Magnitude
            local t = dist / math.max(DW.bulletSpeed, 1)
            local predicted = targetPos + targetVel * t

            local hum = murd.Character:FindFirstChildOfClass("Humanoid")
            if hum then
                local state = hum:GetState()
                if state == Enum.HumanoidStateType.Freefall
                    or state == Enum.HumanoidStateType.Jumping
                    or math.abs(targetVel.Y) > 0.1 then
                    predicted = predicted + 0.5 * GRAVITY * (t * t)
                end
            end
            DW.cachedTargetPos = predicted
            DW.cachedTargetPart = tHrp
        else
            DW.cachedTargetPos = nil
            DW.cachedTargetPart = nil
        end
    else
        DW.cachedTargetPos = nil
        DW.cachedTargetPart = nil
    end
end))

if hookmetamethod and newcclosure and checkcaller then
    local oldNamecall
    oldNamecall = hookmetamethod(game, "__namecall", newcclosure(function(self, ...)
        if DW.silentAimOn and DW.cachedTargetPos and not checkcaller() then
            if DW.silentAimMethod == "Hook" or DW.silentAimMethod == "All" then
                local method = getnamecallmethod()
                local args = { ... }
                if method == "Raycast" and typeof(args[1]) == "Vector3" then
                    args[2] = (DW.cachedTargetPos - args[1]).Unit * 1000
                    return oldNamecall(self, table.unpack(args))
                elseif (method == "FindPartOnRayWithIgnoreList" or method == "FindPartOnRay")
                    and typeof(args[1]) == "Ray" then
                    args[1] = Ray.new(args[1].Origin,
                        (DW.cachedTargetPos - args[1].Origin).Unit * 1000)
                    return oldNamecall(self, table.unpack(args))
                end
            end
        end
        return oldNamecall(self, ...)
    end))
    DW.oldNamecall = oldNamecall

    local oldIndex
    oldIndex = hookmetamethod(game, "__index", newcclosure(function(self, key)
        if DW.silentAimOn and DW.cachedTargetPos and not checkcaller() and self == Mouse then
            if key == "Hit" and (DW.silentAimMethod == "CFrame" or DW.silentAimMethod == "All") then
                return CFrame.new(DW.cachedTargetPos)
            elseif key == "Target" and (DW.silentAimMethod == "Index" or DW.silentAimMethod == "All") then
                return DW.cachedTargetPart
            end
        end
        return oldIndex(self, key)
    end))
    DW.oldIndex = oldIndex
end

-- ─────────────── Window ───────────────
local Window = Library:Window({
    Title = "Dragon Ware",
    Suffix = ".mm2",
    Width = 720,
    Height = 640,
    TabStyle = "side",
    TabWidth = 170,
    MobileButton = true,
})

-- ─────────────── PLAYER TAB ───────────────
local PlayerTab = Window:Tab({ Title = "Player", Icon = "user" })

local MoveSec = PlayerTab:Section("Movement", 1)

MoveSec:Toggle({
    Text = "Custom walk speed", Flag = "dw_speedOn", Default = false,
    Callback = function(v)
        DW.speedOn = v
        if not v then local h = getHum(); if h then h.WalkSpeed = 16 end end
    end,
})
MoveSec:Slider({
    Text = "Walk speed", Flag = "dw_speedVal", Min = 16, Max = 120, Default = 16,
    Callback = function(v) DW.speedVal = v end,
})
MoveSec:Toggle({
    Text = "Custom jump power", Flag = "dw_jumpOn", Default = false,
    Callback = function(v)
        DW.jumpOn = v
        if not v then local h = getHum(); if h then h.UseJumpPower = true; h.JumpPower = 50 end end
    end,
})
MoveSec:Slider({
    Text = "Jump power", Flag = "dw_jumpVal", Min = 50, Max = 250, Default = 50,
    Callback = function(v) DW.jumpVal = v end,
})
MoveSec:Slider({
    Text = "Gravity", Flag = "dw_gravity", Min = 20, Max = 300, Default = math.floor(DW.defaultGravity),
    Callback = function(v) workspace.Gravity = v end,
})
MoveSec:Button({
    Text = "Reset gravity",
    Callback = function()
        workspace.Gravity = DW.defaultGravity
        notify("Gravity", "Restored.", 2, "success")
    end,
})

track(RunService.Heartbeat:Connect(function()
    local h = getHum(); if not h then return end
    if DW.speedOn then h.WalkSpeed = DW.speedVal end
    if DW.jumpOn then h.UseJumpPower = true; h.JumpPower = DW.jumpVal end
end))

local BombSec = PlayerTab:Section("Bomb Jump", 1)
BombSec:Slider({ Text = "Bomb jump power", Flag = "dw_bjPower", Min = 50, Max = 600, Default = 250,
    Callback = function(v) DW.bombJumpPower = v end })
BombSec:Slider({ Text = "Horizontal boost", Flag = "dw_bjHor", Min = 0, Max = 300, Default = 0,
    Callback = function(v) DW.bombJumpHorizontal = v end })

local function bombJump()
    local root, hum = getRoot(), getHum()
    if not root or not hum then return end
    local forward = Vector3.zero
    if DW.bombJumpHorizontal > 0 then
        local cam = workspace.CurrentCamera
        forward = cam.CFrame.LookVector * DW.bombJumpHorizontal
    end
    root.AssemblyLinearVelocity = Vector3.new(forward.X, DW.bombJumpPower, forward.Z)
end

BombSec:Button({ Text = "Launch now", Callback = bombJump })
BombSec:Toggle({ Text = "Auto bomb jump (Space)", Flag = "dw_bjAuto", Default = false,
    Callback = function(v) DW.bombJumpAuto = v end })

track(UserInputService.JumpRequest:Connect(function()
    if DW.bombJumpAuto then bombJump() end
end))

local GlitchSec = PlayerTab:Section("Speed Glitch", 2)
GlitchSec:Toggle({ Text = "Enable", Flag = "dw_glitchOn", Default = false,
    Callback = function(v) DW.glitchOn = v end })
GlitchSec:Slider({ Text = "Jump speed", Flag = "dw_glitchSpeed", Min = 16, Max = 300, Default = 60,
    Callback = function(v) DW.glitchSpeed = v end })
GlitchSec:Dropdown({ Text = "Mode", Flag = "dw_glitchMode", Options = { "Jump", "Always" }, Default = "Jump",
    Callback = function(v) DW.glitchMode = v end })
GlitchSec:Dropdown({ Text = "Method", Flag = "dw_glitchMethod", Options = { "Velocity", "CFrame" }, Default = "Velocity",
    Callback = function(v) DW.glitchMethod = v end })

track(RunService.Heartbeat:Connect(function(dt)
    if not DW.glitchOn or DW.flying or DW.flinging then return end
    local hum, root = getHum(), getRoot()
    if not hum or not root then return end
    local inAir = hum.FloorMaterial == Enum.Material.Air
    if DW.glitchMode == "Jump" and not inAir then return end
    local md = hum.MoveDirection
    if md.Magnitude < 0.1 then return end
    md = md.Unit
    if DW.glitchMethod == "Velocity" then
        local v = root.AssemblyLinearVelocity
        root.AssemblyLinearVelocity = Vector3.new(md.X * DW.glitchSpeed, v.Y, md.Z * DW.glitchSpeed)
    else
        root.CFrame = root.CFrame + md * (DW.glitchSpeed * dt)
    end
end))

local AbilSec = PlayerTab:Section("Abilities", 2)

AbilSec:Toggle({ Text = "Infinite jump", Flag = "dw_infJump", Default = false,
    Callback = function(v) DW.infJump = v end })

track(UserInputService.JumpRequest:Connect(function()
    if DW.infJump then
        local h = getHum()
        if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end
    end
end))

AbilSec:Toggle({ Text = "Noclip", Flag = "dw_noclip", Default = false,
    Callback = function(v) DW.noclip = v end })

track(RunService.Stepped:Connect(function()
    if not DW.noclip then return end
    local c = getChar(); if not c then return end
    for _, p in ipairs(c:GetDescendants()) do
        if p:IsA("BasePart") and p.CanCollide then p.CanCollide = false end
    end
end))

AbilSec:Toggle({ Text = "Spin bot", Flag = "dw_spinOn", Default = false,
    Callback = function(v) DW.spinOn = v end })
AbilSec:Slider({ Text = "Spin speed", Flag = "dw_spinSpeed", Min = 200, Max = 8000, Default = 2200,
    Callback = function(v) DW.spinSpeed = v end })
AbilSec:Dropdown({ Text = "Spin axis", Flag = "dw_spinAxis", Options = { "Y", "XY", "XYZ" }, Default = "Y",
    Callback = function(v) DW.spinAxis = v end })

track(RunService.Heartbeat:Connect(function(dt)
    if not DW.spinOn or DW.flying or DW.flinging then return end
    local root = getRoot(); if not root then return end
    local axis = DW.spinAxis == "Y" and Vector3.new(0, 1, 0)
        or DW.spinAxis == "XY" and Vector3.new(1, 1, 0)
        or Vector3.new(1, 1, 1)
    root.CFrame = root.CFrame * CFrame.fromAxisAngle(axis.Unit, math.rad(DW.spinSpeed) * dt)
end))

local function stopFly()
    DW.flying = false
    if DW.flyBV then DW.flyBV:Destroy(); DW.flyBV = nil end
    if DW.flyBG then DW.flyBG:Destroy(); DW.flyBG = nil end
    local h = getHum(); if h then h.PlatformStand = false end
end

local function startFly()
    local root, hum = getRoot(), getHum()
    if not root or not hum then return end
    DW.flying = true
    hum.PlatformStand = true
    DW.flyBV = Instance.new("BodyVelocity")
    DW.flyBV.MaxForce = Vector3.new(1e9, 1e9, 1e9)
    DW.flyBV.Velocity = Vector3.zero
    DW.flyBV.Parent = root
    DW.flyBG = Instance.new("BodyGyro")
    DW.flyBG.MaxTorque = Vector3.new(1e9, 1e9, 1e9)
    DW.flyBG.P = 9e4
    DW.flyBG.CFrame = root.CFrame
    DW.flyBG.Parent = root
end

AbilSec:Toggle({ Text = "Fly", Flag = "dw_fly", Default = false,
    Callback = function(v) if v then startFly() else stopFly() end end })
AbilSec:Slider({ Text = "Fly speed", Flag = "dw_flySpeed", Min = 10, Max = 250, Default = 60,
    Callback = function(v) DW.flySpeed = v end })

track(RunService.RenderStepped:Connect(function()
    if not DW.flying or not DW.flyBV or not DW.flyBG then return end
    local root = getRoot(); if not root then return end
    local cam = workspace.CurrentCamera
    local dir = Vector3.zero
    if UserInputService:IsKeyDown(Enum.KeyCode.W) then dir += cam.CFrame.LookVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) then dir -= cam.CFrame.LookVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) then dir += cam.CFrame.RightVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) then dir -= cam.CFrame.RightVector end
    if UserInputService:IsKeyDown(Enum.KeyCode.Space) then dir += Vector3.yAxis end
    if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then dir -= Vector3.yAxis end
    DW.flyBV.Velocity = dir.Magnitude > 0 and dir.Unit * DW.flySpeed or Vector3.zero
    DW.flyBG.CFrame = cam.CFrame
end))

track(LocalPlayer.CharacterAdded:Connect(function()
    if DW.flying then
        DW.flyBV, DW.flyBG = nil, nil
        task.wait(0.5); startFly()
    end
end))

local AnimSec = PlayerTab:Section("Bundle Animation", 1)

local animPresets = {
    None          = "",
    GangnamStyle  = "rbxassetid://11778112981",
    Floss         = "rbxassetid://5915693811",
    OrangeJustice = "rbxassetid://3814800961",
    Robot         = "rbxassetid://6495505137",
    DefaultIdle   = "rbxassetid://507766666",
}

local function stopBundleAnim()
    if DW.bundleAnimTrack then
        pcall(function() DW.bundleAnimTrack:Stop() end)
        DW.bundleAnimTrack = nil
    end
    DW.bundleAnimPlaying = false
end

local function playBundleAnim(id)
    stopBundleAnim()
    if not id or id == "" then return end
    local char = getChar(); if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid"); if not hum then return end
    local animator = hum:FindFirstChildOfClass("Animator")
    if not animator then animator = Instance.new("Animator"); animator.Parent = hum end
    local anim = Instance.new("Animation")
    anim.AnimationId = id
    local ok, tr = pcall(function() return animator:LoadAnimation(anim) end)
    if ok and tr then
        tr.Looped = true
        tr.Priority = Enum.AnimationPriority.Action
        tr:Play()
        DW.bundleAnimTrack = tr
        DW.bundleAnimPlaying = true
    end
end

AnimSec:Dropdown({
    Text = "Preset", Flag = "dw_animPreset",
    Options = { "None", "GangnamStyle", "Floss", "OrangeJustice", "Robot", "DefaultIdle" },
    Default = "None",
    Callback = function(v)
        local id = animPresets[v]
        DW.bundleAnimId = id
        if id == "" then stopBundleAnim() else playBundleAnim(id) end
    end,
})
AnimSec:Input({
    Text = "Custom animation ID", Flag = "dw_animId", Default = "",
    Placeholder = "rbxassetid://...",
    Callback = function(s)
        if s == "" then return end
        if not s:match("^rbxassetid://") then s = "rbxassetid://" .. s:gsub("%D", "") end
        DW.bundleAnimId = s
        playBundleAnim(s)
    end,
})
AnimSec:Button({ Text = "Play animation",
    Callback = function()
        if DW.bundleAnimId == "" then notify("Bundle Animation", "ID пустой.", 2, "warning"); return end
        playBundleAnim(DW.bundleAnimId)
    end,
})
AnimSec:Button({ Text = "Stop animation", Callback = stopBundleAnim })

track(LocalPlayer.CharacterAdded:Connect(function()
    stopBundleAnim()
    if DW.bundleAnimId ~= "" then task.wait(0.5); playBundleAnim(DW.bundleAnimId) end
end))

-- ─────────────── WORLD TAB ───────────────
local WorldTab = Window:Tab({ Title = "World", Icon = "globe" })

local CamSec = WorldTab:Section("Camera", 1)
CamSec:Toggle({ Text = "Custom FOV", Flag = "dw_fovOn", Default = false,
    Callback = function(v)
        DW.fovOn = v
        if not v then workspace.CurrentCamera.FieldOfView = DW.defaultFOV end
    end })
CamSec:Slider({ Text = "Field of view", Flag = "dw_fovVal", Min = 30, Max = 120, Default = 70,
    Callback = function(v) DW.fovVal = v end })

track(RunService.RenderStepped:Connect(function()
    if DW.fovOn then workspace.CurrentCamera.FieldOfView = DW.fovVal end
end))

CamSec:Dropdown({ Text = "Camera mode", Flag = "dw_camMode",
    Options = { "Classic", "LockFirstPerson" }, Default = "Classic",
    Callback = function(v) pcall(function() LocalPlayer.CameraMode = Enum.CameraMode[v] end) end })

local LightSec = WorldTab:Section("Lighting", 2)
LightSec:Toggle({ Text = "Fullbright", Flag = "dw_fullbright", Default = false,
    Callback = function(v)
        DW.fullbright = v
        if not v then
            for k, val in pairs(DW.origLighting) do pcall(function() Lighting[k] = val end) end
        end
    end })

track(RunService.RenderStepped:Connect(function()
    if not DW.fullbright then return end
    Lighting.Brightness = 2; Lighting.ClockTime = 14
    Lighting.FogEnd = 1e6; Lighting.GlobalShadows = false
    Lighting.Ambient = Color3.fromRGB(178, 178, 178)
end))

LightSec:Slider({ Text = "Time of day", Flag = "dw_timeofday", Min = 0, Max = 24, Default = math.floor(Lighting.ClockTime),
    Callback = function(v) if not DW.fullbright then Lighting.ClockTime = v end end })
LightSec:ColorPicker({ Text = "Ambient tint", Flag = "dw_ambient", Default = Lighting.Ambient,
    Callback = function(c) if not DW.fullbright then Lighting.Ambient = c end end })

-- ─────────────── VISUAL TAB ───────────────
local VisualTab = Window:Tab({ Title = "Visual", Icon = "eye" })

local AuraSec = VisualTab:Section("Auras", 1)

local auraFolder = Instance.new("Folder")
auraFolder.Name = "DragonWareAuras"
auraFolder.Parent = workspace
table.insert(Cleanups, function() auraFolder:Destroy() end)

local function cleanupAura(plr)
    local o = DW.auraObjects[plr]
    if not o then return end
    pcall(function() o.ring:Destroy() end)
    pcall(function() o.light:Destroy() end)
    pcall(function() o.particles:Destroy() end)
    DW.auraObjects[plr] = nil
end

local function makeAura(plr, root)
    cleanupAura(plr)
    local ring = Instance.new("MeshPart")
    ring.Name = "AuraRing"
    ring.Anchored = true
    ring.CanCollide = false
    ring.CanQuery = false
    ring.CanTouch = false
    ring.CastShadow = false
    ring.MeshId = "rbxassetid://12221759"
    ring.TextureId = ""
    ring.Material = Enum.Material.Neon
    ring.Color = DW.auraColor
    ring.Transparency = DW.auraTransparency
    ring.Size = Vector3.new(DW.auraSize, DW.auraThickness, DW.auraSize)
    ring.CFrame = CFrame.new(root.Position)
    ring.Parent = auraFolder

    local light = Instance.new("PointLight")
    light.Color = DW.auraColor
    light.Brightness = DW.auraLight and 3 or 0
    light.Range = DW.auraSize * 2
    light.Parent = ring

    local particles = Instance.new("ParticleEmitter")
    particles.Texture = "rbxassetid://241876428"
    particles.Color = ColorSequence.new(DW.auraColor)
    particles.Size = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0),
        NumberSequenceKeypoint.new(0.5, 0.6),
        NumberSequenceKeypoint.new(1, 0),
    })
    particles.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 1),
        NumberSequenceKeypoint.new(0.3, 0.2),
        NumberSequenceKeypoint.new(1, 1),
    })
    particles.Lifetime = NumberRange.new(1, 1.5)
    particles.Rate = 40
    particles.Speed = NumberRange.new(0.5, 2)
    particles.SpreadAngle = Vector2.new(180, 180)
    particles.Acceleration = Vector3.new(0, 5, 0)
    particles.LightEmission = 1
    particles.LightInfluence = 0
    particles.Enabled = DW.auraParticles
    particles.Parent = ring

    DW.auraObjects[plr] = { ring = ring, light = light, particles = particles }
end

task.spawn(function()
    while true do
        if DW.auraOn then
            for _, plr in ipairs(Players:GetPlayers()) do
                local char = plr.Character
                local hum  = char and char:FindFirstChildOfClass("Humanoid")
                local root = char and char:FindFirstChild("HumanoidRootPart")
                if not char or not hum or not root or hum.Health <= 0 then
                    cleanupAura(plr); continue
                end
                local o = DW.auraObjects[plr]
                if not o or not o.ring.Parent then
                    makeAura(plr, root); o = DW.auraObjects[plr]
                end
                if not o then continue end

                local yRot = DW.auraSpin and math.rad(tick() * 60 * DW.auraSpeed) or 0
                o.ring.Size = Vector3.new(DW.auraSize, DW.auraThickness, DW.auraSize)
                o.ring.CFrame = CFrame.new(root.Position) * CFrame.Angles(0, yRot, 0)
                o.ring.Transparency = DW.auraTransparency
                o.light.Brightness = DW.auraLight and 3 or 0
                o.light.Range = DW.auraSize * 2
                if o.particles then o.particles.Enabled = DW.auraParticles end

                if DW.auraRainbow then
                    local col = Color3.fromHSV((tick() * 0.2) % 1, 0.8, 1)
                    o.ring.Color = col
                    o.light.Color = col
                    if o.particles then o.particles.Color = ColorSequence.new(col) end
                else
                    o.ring.Color = DW.auraColor
                    o.light.Color = DW.auraColor
                    if o.particles then o.particles.Color = ColorSequence.new(DW.auraColor) end
                end
            end
        else
            for plr in pairs(DW.auraObjects) do cleanupAura(plr) end
        end
        task.wait(0.05)
    end
end)

track(Players.PlayerRemoving:Connect(cleanupAura))

AuraSec:Toggle({ Text = "Enable auras", Flag = "dw_auraOn", Default = false,
    Callback = function(v) DW.auraOn = v end })

local auraPresets = {
    Default = { color = Color3.fromRGB(179,165,255), size = 4, speed = 3, transparency = 0.5, rainbow = false, thickness = 0.6 },
    Demon   = { color = Color3.fromRGB(255,40,40),   size = 5, speed = 5, transparency = 0.3, rainbow = false, thickness = 0.5 },
    Angel   = { color = Color3.fromRGB(255,255,255), size = 6, speed = 2, transparency = 0.4, rainbow = false, thickness = 0.4 },
    Galaxy  = { color = Color3.fromRGB(150,100,255), size = 7, speed = 4, transparency = 0.5, rainbow = true,  thickness = 0.7 },
    Fire    = { color = Color3.fromRGB(255,120,0),   size = 5, speed = 6, transparency = 0.3, rainbow = false, thickness = 0.8 },
    Toxic   = { color = Color3.fromRGB(120,255,0),   size = 4, speed = 4, transparency = 0.4, rainbow = false, thickness = 0.6 },
    Ice     = { color = Color3.fromRGB(100,220,255), size = 5, speed = 2, transparency = 0.4, rainbow = false, thickness = 0.5 },
}

AuraSec:Dropdown({ Text = "Aura preset", Flag = "dw_auraPreset",
    Options = { "Default", "Demon", "Angel", "Galaxy", "Fire", "Toxic", "Ice" },
    Default = "Default",
    Callback = function(v)
        local p = auraPresets[v]; if not p then return end
        DW.auraColor = p.color
        DW.auraSize = p.size
        DW.auraSpeed = p.speed
        DW.auraTransparency = p.transparency
        DW.auraRainbow = p.rainbow
        DW.auraThickness = p.thickness or 0.6
    end })

AuraSec:ColorPicker({ Text = "Aura color", Flag = "dw_auraColor", Default = DW.auraColor,
    Callback = function(c) DW.auraColor = c end })
AuraSec:Slider({ Text = "Aura size", Flag = "dw_auraSize", Min = 2, Max = 20, Default = 4,
    Callback = function(v) DW.auraSize = v end })
AuraSec:Slider({ Text = "Толщина кольца", Flag = "dw_auraThick", Min = 0.2, Max = 3, Decimals = 2, Default = 0.6,
    Callback = function(v) DW.auraThickness = v end })
AuraSec:Slider({ Text = "Transparency", Flag = "dw_auraTrans", Min = 0, Max = 1, Decimals = 2, Default = 0.5,
    Callback = function(v) DW.auraTransparency = v end })
AuraSec:Toggle({ Text = "Rainbow aura", Flag = "dw_auraRainbow", Default = false,
    Callback = function(v) DW.auraRainbow = v end })
AuraSec:Toggle({ Text = "Spin ring", Flag = "dw_auraSpin", Default = true,
    Callback = function(v) DW.auraSpin = v end })
AuraSec:Slider({ Text = "Spin speed", Flag = "dw_auraSpeed", Min = 1, Max = 20, Default = 3,
    Callback = function(v) DW.auraSpeed = v end })
AuraSec:Toggle({ Text = "Particles", Flag = "dw_auraParts", Default = true,
    Callback = function(v) DW.auraParticles = v end })
AuraSec:Toggle({ Text = "Point light", Flag = "dw_auraLight", Default = true,
    Callback = function(v) DW.auraLight = v end })

local KillFxSec = VisualTab:Section("Kill Effect", 1)

local killFxFolder = Instance.new("Folder")
killFxFolder.Name = "DragonWareKillFX"
killFxFolder.Parent = workspace
table.insert(Cleanups, function() killFxFolder:Destroy() end)

local function spawnKillEffect(position)
    if not DW.killFxOn or not position then return end
    local color = DW.killFxRainbow and Color3.fromHSV((tick() * 0.3) % 1, 0.85, 1) or DW.killFxColor

    if DW.killFxType == "Explosion" or DW.killFxType == "Sparkles" then
        local part = Instance.new("Part")
        part.Anchored = true
        part.CanCollide = false
        part.CanQuery = false
        part.CanTouch = false
        part.CastShadow = false
        part.Transparency = 1
        part.Size = Vector3.new(0.1, 0.1, 0.1)
        part.CFrame = CFrame.new(position)
        part.Parent = killFxFolder

        local emitter = Instance.new("ParticleEmitter")
        emitter.Texture = "rbxassetid://241876428"
        emitter.Color = ColorSequence.new(color)
        emitter.Size = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0),
            NumberSequenceKeypoint.new(0.3, DW.killFxSize),
            NumberSequenceKeypoint.new(1, 0),
        })
        emitter.Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0.2),
            NumberSequenceKeypoint.new(1, 1),
        })
        emitter.Lifetime = NumberRange.new(0.8, 1.4)
        emitter.Speed = NumberRange.new(20, 40)
        emitter.SpreadAngle = Vector2.new(180, 180)
        emitter.LightEmission = 1
        emitter.LightInfluence = 0
        emitter.Rate = 0
        emitter:Emit(DW.killFxType == "Explosion" and 80 or 60)
        emitter.Parent = part

        Debris:AddItem(part, 3)
    elseif DW.killFxType == "Fire" then
        local firePart = Instance.new("Part")
        firePart.Anchored = true
        firePart.CanCollide = false
        firePart.CanQuery = false
        firePart.CanTouch = false
        firePart.CastShadow = false
        firePart.Transparency = 1
        firePart.Size = Vector3.new(0.1, 0.1, 0.1)
        firePart.CFrame = CFrame.new(position)
        firePart.Parent = killFxFolder

        local fire = Instance.new("Fire")
        fire.Color = color
        fire.SecondaryColor = color
        fire.Size = DW.killFxSize
        fire.Heat = 5
        fire.Parent = firePart

        Debris:AddItem(firePart, 2)
    end
end

KillFxSec:Toggle({ Text = "Enable kill effect", Flag = "dw_killFxOn", Default = false,
    Callback = function(v) DW.killFxOn = v end })
KillFxSec:Dropdown({ Text = "Effect type", Flag = "dw_killFxType",
    Options = { "Explosion", "Sparkles", "Fire" }, Default = "Explosion",
    Callback = function(v) DW.killFxType = v end })
KillFxSec:ColorPicker({ Text = "Effect color", Flag = "dw_killFxColor", Default = DW.killFxColor,
    Callback = function(c) DW.killFxColor = c end })
KillFxSec:Slider({ Text = "Effect size", Flag = "dw_killFxSize", Min = 2, Max = 20, Default = 8,
    Callback = function(v) DW.killFxSize = v end })
KillFxSec:Toggle({ Text = "Rainbow effect", Flag = "dw_killFxRainbow", Default = false,
    Callback = function(v) DW.killFxRainbow = v end })

KillFxSec:Button({ Text = "Тест (на себе)",
    Callback = function()
        local r = getRoot()
        if r then
            local prev = DW.killFxOn
            DW.killFxOn = true
            spawnKillEffect(r.Position + Vector3.new(0, 2, 0))
            DW.killFxOn = prev
            notify("Kill FX", "Тест-эффект создан.", 2, "accent")
        end
    end })

local CrossSec = VisualTab:Section("Crosshair", 2)

local crossGui = Instance.new("ScreenGui")
crossGui.Name = "DragonWareCrosshair"
crossGui.ResetOnSpawn = false
crossGui.IgnoreGuiInset = true
crossGui.DisplayOrder = 95
crossGui.Parent = guiParent()
table.insert(Cleanups, function() crossGui:Destroy() end)

local crossHolder = Instance.new("Frame")
crossHolder.AnchorPoint = Vector2.new(0.5, 0.5)
crossHolder.Position = UDim2.new(0.5, 0, 0.5, 0)
crossHolder.BackgroundTransparency = 1
crossHolder.Size = UDim2.fromOffset(60, 60)
crossHolder.Visible = false
crossHolder.Parent = crossGui

local function rebuildCrosshair()
    for _, c in ipairs(crossHolder:GetChildren()) do c:Destroy() end
    if not DW.crossOn then crossHolder.Visible = false; return end
    crossHolder.Visible = true
    local c, t, gap, len = DW.crossColor, DW.crossThickness, DW.crossGap, DW.crossSize
    local function line(x, y, w, h)
        if DW.crossOutline then
            local outer = Instance.new("Frame")
            outer.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
            outer.BorderSizePixel = 0
            outer.Position = UDim2.new(0.5, x, 0.5, y)
            outer.Size = UDim2.fromOffset(w + t + 2, h + t + 2)
            outer.AnchorPoint = Vector2.new(0.5, 0.5)
            outer.ZIndex = 2
            outer.Parent = crossHolder
        end
        local f = Instance.new("Frame")
        f.BackgroundColor3 = c
        f.BorderSizePixel = 0
        f.Position = UDim2.new(0.5, x, 0.5, y)
        f.Size = UDim2.fromOffset(w, h)
        f.AnchorPoint = Vector2.new(0.5, 0.5)
        f.ZIndex = 3
        f.Parent = crossHolder
    end
    if DW.crossStyle == "Dot" then
        line(0, 0, t + 1, t + 1)
    elseif DW.crossStyle == "Cross" then
        line(0, -gap - len/2, t, len); line(0, gap + len/2, t, len)
        line(-gap - len/2, 0, len, t); line(gap + len/2, 0, len, t)
    elseif DW.crossStyle == "Cross+Dot" then
        line(0, -gap - len/2, t, len); line(0, gap + len/2, t, len)
        line(-gap - len/2, 0, len, t); line(gap + len/2, 0, len, t)
        line(0, 0, t + 1, t + 1)
    elseif DW.crossStyle == "Circle" then
        local circle = Instance.new("Frame")
        circle.BackgroundTransparency = 1
        circle.AnchorPoint = Vector2.new(0.5, 0.5)
        circle.Position = UDim2.new(0.5, 0, 0.5, 0)
        circle.Size = UDim2.fromOffset(DW.crossSize * 2, DW.crossSize * 2)
        circle.ZIndex = 3
        circle.Parent = crossHolder
        Instance.new("UICorner", circle).CornerRadius = UDim.new(1, 0)
        local stroke = Instance.new("UIStroke", circle)
        stroke.Color = c
        stroke.Thickness = t
    end
end

CrossSec:Toggle({ Text = "Enable crosshair", Flag = "dw_crossOn", Default = false,
    Callback = function(v) DW.crossOn = v; rebuildCrosshair() end })
CrossSec:Dropdown({ Text = "Style", Flag = "dw_crossStyle",
    Options = { "Dot", "Cross", "Circle", "Cross+Dot" }, Default = "Cross",
    Callback = function(v) DW.crossStyle = v; rebuildCrosshair() end })
CrossSec:ColorPicker({ Text = "Crosshair color", Flag = "dw_crossColor", Default = DW.crossColor,
    Callback = function(c) DW.crossColor = c; rebuildCrosshair() end })
CrossSec:Slider({ Text = "Size", Flag = "dw_crossSize", Min = 4, Max = 40, Default = 12,
    Callback = function(v) DW.crossSize = v; rebuildCrosshair() end })
CrossSec:Slider({ Text = "Thickness", Flag = "dw_crossThick", Min = 1, Max = 6, Default = 2,
    Callback = function(v) DW.crossThickness = v; rebuildCrosshair() end })
CrossSec:Slider({ Text = "Gap", Flag = "dw_crossGap", Min = 0, Max = 20, Default = 4,
    Callback = function(v) DW.crossGap = v; rebuildCrosshair() end })
CrossSec:Toggle({ Text = "Outline", Flag = "dw_crossOutline", Default = true,
    Callback = function(v) DW.crossOutline = v; rebuildCrosshair() end })

-- ─────────────── MM2 TAB ───────────────
local MM2Tab = Window:Tab({ Title = "MM2", Icon = "sword" })

local ESPSec = MM2Tab:Section("ESP Roles", 1)

local espFolder = Instance.new("Folder")
espFolder.Name = "DragonWareESP"
espFolder.Parent = guiParent()
table.insert(Cleanups, function()
    for plr in pairs(DW.espObjs) do
        local o = DW.espObjs[plr]
        if o then
            pcall(function() o.hl:Destroy() end)
            pcall(function() o.bb:Destroy() end)
        end
    end
    espFolder:Destroy()
end)

local function clearESP(plr)
    local o = DW.espObjs[plr]
    if o then
        pcall(function() o.hl:Destroy() end)
        pcall(function() o.bb:Destroy() end)
        DW.espObjs[plr] = nil
    end
end
track(Players.PlayerRemoving:Connect(clearESP))

local function ensureESP(plr, char)
    local o = DW.espObjs[plr]
    if o and o.hl.Parent and o.char == char then return o end
    clearESP(plr)
    local hl = Instance.new("Highlight")
    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    hl.FillTransparency = 0.55
    hl.OutlineTransparency = 0
    hl.Adornee = char
    hl.Parent = espFolder
    local bb = Instance.new("BillboardGui")
    bb.AlwaysOnTop = true
    bb.Size = UDim2.fromOffset(160, 20)
    bb.StudsOffset = Vector3.new(0, 3, 0)
    bb.Adornee = char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
    bb.Parent = espFolder
    local lbl = Instance.new("TextLabel")
    lbl.BackgroundTransparency = 1
    lbl.Size = UDim2.fromScale(1, 1)
    lbl.Font = Enum.Font.GothamMedium
    lbl.TextSize = 13
    lbl.TextStrokeTransparency = 0.4
    lbl.Parent = bb
    o = { hl = hl, bb = bb, lbl = lbl, char = char }
    DW.espObjs[plr] = o
    return o
end

task.spawn(function()
    local lastFetch = 0
    while DW.mm2Running do
        local anyOn = DW.roleOn.Murderer or DW.roleOn.Sheriff or DW.roleOn.Innocent
        if anyOn and os.clock() - lastFetch > 1 then
            lastFetch = os.clock(); refreshRoles()
        end
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer then
                local char = plr.Character
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                if anyOn and char and hum and hum.Health > 0 then
                    local role = getRole(plr)
                    if DW.roleOn[role] then
                        local o = ensureESP(plr, char)
                        local c = DW.roleColor[role]
                        o.hl.FillColor = c
                        o.hl.OutlineColor = c
                        o.lbl.TextColor3 = c
                        o.lbl.Visible = DW.showNames
                        o.lbl.Text = plr.DisplayName .. " [" .. role .. "]"
                    else clearESP(plr) end
                else clearESP(plr) end
            end
        end
        task.wait(0.3)
    end
end)

ESPSec:Toggle({ Text = "Murderer ESP", Flag = "dw_espMurderer", Default = false,
    Callback = function(v) DW.roleOn.Murderer = v end })
ESPSec:ColorPicker({ Text = "Murderer color", Flag = "dw_colorMurderer", Default = DW.roleColor.Murderer,
    Callback = function(c) DW.roleColor.Murderer = c end })
ESPSec:Toggle({ Text = "Sheriff ESP", Flag = "dw_espSheriff", Default = false,
    Callback = function(v) DW.roleOn.Sheriff = v end })
ESPSec:ColorPicker({ Text = "Sheriff color", Flag = "dw_colorSheriff", Default = DW.roleColor.Sheriff,
    Callback = function(c) DW.roleColor.Sheriff = c end })
ESPSec:Toggle({ Text = "Innocent ESP", Flag = "dw_espInnocent", Default = false,
    Callback = function(v) DW.roleOn.Innocent = v end })
ESPSec:ColorPicker({ Text = "Innocent color", Flag = "dw_colorInnocent", Default = DW.roleColor.Innocent,
    Callback = function(c) DW.roleColor.Innocent = c end })
ESPSec:Toggle({ Text = "Show names", Flag = "dw_showNames", Default = true,
    Callback = function(v) DW.showNames = v end })

local GunSec = MM2Tab:Section("Gun", 1)

DW.gunDrop = workspace:FindFirstChild("GunDrop", true)
track(workspace.DescendantAdded:Connect(function(d)
    if d.Name == "GunDrop" then DW.gunDrop = d end
end))

local function getGunPart()
    if not (DW.gunDrop and DW.gunDrop.Parent) then
        DW.gunDrop = workspace:FindFirstChild("GunDrop", true)
    end
    if not DW.gunDrop then return nil end
    if DW.gunDrop:IsA("BasePart") then return DW.gunDrop end
    return DW.gunDrop:FindFirstChildWhichIsA("BasePart", true)
end

local function grabGun(silent)
    if DW.grabbing then return end
    local root, part = getRoot(), getGunPart()
    if not root or not part then
        if not silent then notify("Grab gun", "No dropped gun on the map.", 2, "warning") end
        return
    end
    if hasTool(LocalPlayer, "Gun") or hasTool(LocalPlayer, "Knife") then return end
    DW.grabbing = true
    if firetouchinterest then
        pcall(firetouchinterest, root, part, 0); task.wait(0.05)
        pcall(firetouchinterest, root, part, 1); task.wait(0.25)
    end
    if part.Parent and not hasTool(LocalPlayer, "Gun") then
        local r = getRoot()
        if r then
            local back = r.CFrame
            r.CFrame = part.CFrame + Vector3.new(0, 2, 0)
            task.wait(0.3)
            r = getRoot(); if r then r.CFrame = back end
        end
    end
    if not silent then
        notify("Grab gun", hasTool(LocalPlayer, "Gun") and "Got the gun." or "Tried.", 2, "success")
    end
    DW.grabbing = false
end

GunSec:Toggle({ Text = "Auto grab gun", Flag = "dw_autoGrab", Default = false,
    Callback = function(v) DW.autoGrab = v end })
GunSec:Button({ Text = "Grab gun now", Callback = function() grabGun(false) end })

task.spawn(function()
    while DW.mm2Running do
        if DW.autoGrab then pcall(grabGun, true) end
        task.wait(0.4)
    end
end)

-- ═══════════════════ SHOOT MURDERER + SILENT AIM ═══════════════════
local ShootSec = MM2Tab:Section("Shoot Murderer", 2)

local function shootMurderer()
    if DW.shooting then return end
    local gun = hasTool(LocalPlayer, "Gun")
    if not gun then notify("Shoot murderer", "You don't have the gun.", 2, "warning"); return end
    refreshRoles()
    local murd = getMurderer()
    local mChar = murd and murd.Character
    local mRoot = mChar and mChar:FindFirstChild("HumanoidRootPart")
    local mHum  = mChar and mChar:FindFirstChildOfClass("Humanoid")
    if not mRoot or not mHum or mHum.Health <= 0 then
        notify("Shoot murderer", "Murderer not found.", 2, "warning"); return
    end

    if DW.wallbangOn then
        local root, hum = getRoot(), getHum()
        if not root or not hum then return end
        local gun2 = hasTool(LocalPlayer, "Gun")
        if not gun2 then return end
        if gun2.Parent ~= getChar() then hum:EquipTool(gun2); task.wait(0.1) end
        local origin = root.CFrame
        local prevNoclip = DW.noclip; DW.noclip = true
        root.CFrame = mRoot.CFrame
        RunService.RenderStepped:Wait()
        pcall(function() gun2:Activate() end)
        task.wait(0.05)
        root.CFrame = origin
        DW.noclip = prevNoclip
        notify("Shoot murderer", "Wallbang at " .. murd.DisplayName, 2, "success")
        return
    end

    DW.shooting = true

    local hum = getHum()
    if hum and gun.Parent ~= getChar() then
        hum:EquipTool(gun); task.wait(0.08)
    end

    local function fireTracer(targetPos)
        if not DW.bulletTracersOn then return end
        local handle = gun:FindFirstChild("Handle") or gun:FindFirstChildWhichIsA("BasePart")
        local from = handle and handle.Position
            or (getRoot() and getRoot().Position + Vector3.new(0, 1.5, 0))
            or (workspace.CurrentCamera.CFrame.Position)
        drawBulletTracer(from, targetPos, DW.bulletTracerColor)
    end

    if DW.silentAimOn then
        local tPart = mChar:FindFirstChild(DW.aimPart) or mRoot
        fireTracer(tPart.Position)
        pcall(function() gun:Activate() end)
        task.wait(0.35)
        DW.shooting = false
        return
    end

    local tPart = mChar:FindFirstChild(DW.aimPart) or mRoot
    local aimPos = tPart.Position + tPart.AssemblyLinearVelocity * DW.prediction
    local cam = workspace.CurrentCamera
    cam.CFrame = CFrame.lookAt(cam.CFrame.Position, aimPos)
    RunService.RenderStepped:Wait()
    local sp = cam:WorldToScreenPoint(aimPos)
    pcall(function()
        VIM:SendMouseMoveEvent(sp.X, sp.Y, game); task.wait(0.03)
        VIM:SendMouseButtonEvent(sp.X, sp.Y, 0, true, game, 0); task.wait(0.05)
        VIM:SendMouseButtonEvent(sp.X, sp.Y, 0, false, game, 0)
    end)
    fireTracer(aimPos)
    pcall(function() gun:Activate() end)
    task.wait(0.4)
    DW.shooting = false
end

ShootSec:Button({ Text = "Shoot murderer", Callback = shootMurderer })

ShootSec:Toggle({
    Text = "Silent Aim",
    Flag = "dw_silentAim",
    Default = false,
    Hint = "Подмена направления пули на сервере. Камера не двигается.",
    Callback = function(v)
        DW.silentAimOn = v
        if v and not DW.oldNamecall then
            notify("Silent Aim", "Hook недоступен — будет обычный аим.", 4, "warning")
        end
    end,
})

ShootSec:Dropdown({
    Text = "Silent method",
    Flag = "dw_silentMethod",
    Options = { "All", "Hook", "Index", "CFrame" },
    Default = "All",
    Callback = function(v) DW.silentAimMethod = v end,
})

ShootSec:Slider({
    Text = "Bullet speed",
    Flag = "dw_bulletSpeed",
    Min = 500, Max = 6000, Default = 2500,
    Callback = function(v) DW.bulletSpeed = v end,
})

ShootSec:Slider({ Text = "Aim part (1=Head 2=HRP 3=Torso)", Flag = "dw_aimPart",
    Min = 1, Max = 3, Default = 1,
    Callback = function(v)
        local i = math.floor(v + 0.5)
        DW.aimPart = i == 1 and "Head" or i == 2 and "HumanoidRootPart" or "UpperTorso"
    end })

ShootSec:Slider({ Text = "Lead prediction", Flag = "dw_prediction",
    Min = 0, Max = 0.4, Decimals = 2, Default = 0.08,
    Callback = function(v) DW.prediction = v end })

ShootSec:Toggle({ Text = "Wallbang (teleport)", Flag = "dw_wallbang", Default = false,
    Callback = function(v) DW.wallbangOn = v end })

ShootSec:Button({ Text = "Who is the murderer?",
    Callback = function()
        refreshRoles(); task.wait(0.3)
        local m = getMurderer()
        notify("Murderer", m and m.DisplayName or "Not detected.", 3, "accent")
    end })

-- ─────────────── Kill All ───────────────
local KillAllSec = MM2Tab:Section("Kill All", 1)

local function killAll()
    local knife = hasTool(LocalPlayer, "Knife")
    local gun = hasTool(LocalPlayer, "Gun")
    if not knife and not gun then
        notify("Kill All", "У тебя нет оружия.", 2, "warning"); return
    end
    refreshRoles()
    local myRoot = getRoot(); if not myRoot then return end
    local hum = getHum()

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        local char = plr.Character
        local hum2 = char and char:FindFirstChildOfClass("Humanoid")
        local pRoot = char and char:FindFirstChild("HumanoidRootPart")
        if not hum2 or not pRoot or hum2.Health <= 0 then continue end
        if (pRoot.Position - myRoot.Position).Magnitude > DW.killAllRange then continue end

        if knife then
            if knife.Parent ~= getChar() and hum then hum:EquipTool(knife); task.wait(0.05) end
            local origin = myRoot.CFrame
            local prevNoclip = DW.noclip; DW.noclip = true
            myRoot.CFrame = pRoot.CFrame + Vector3.new(0, 1, 0)
            RunService.RenderStepped:Wait()
            pcall(function() knife:Activate() end)
            task.wait(0.04)
            myRoot.CFrame = origin
            DW.noclip = prevNoclip
        elseif gun then
            if gun.Parent ~= getChar() and hum then hum:EquipTool(gun); task.wait(0.05) end
            local origin = myRoot.CFrame
            local prevNoclip = DW.noclip; DW.noclip = true
            myRoot.CFrame = pRoot.CFrame + Vector3.new(0, 2, -3)
            RunService.RenderStepped:Wait()
            pcall(function() gun:Activate() end)
            if DW.bulletTracersOn then
                drawBulletTracer(myRoot.Position + Vector3.new(0, 1.5, 0), pRoot.Position, DW.bulletTracerColor)
            end
            task.wait(0.05)
            myRoot.CFrame = origin
            DW.noclip = prevNoclip
        end
        task.wait(0.03)
    end
end

KillAllSec:Button({ Text = "Kill all now", Callback = killAll })
KillAllSec:Slider({ Text = "Range", Flag = "dw_kaRange", Min = 50, Max = 2000, Default = 500,
    Callback = function(v) DW.killAllRange = v end })
KillAllSec:Slider({ Text = "Interval", Flag = "dw_kaInterval", Min = 0.1, Max = 5, Decimals = 1, Default = 0.5,
    Callback = function(v) DW.killAllInterval = v end })
KillAllSec:Toggle({ Text = "Auto kill all", Flag = "dw_kaAuto", Default = false,
    Callback = function(v) DW.killAllOn = v end })

task.spawn(function()
    while DW.mm2Running do
        if DW.killAllOn and os.clock() - DW.killAllLast >= DW.killAllInterval then
            DW.killAllLast = os.clock()
            pcall(killAll)
        end
        task.wait(0.1)
    end
end)

local KauraSec = MM2Tab:Section("Killaura", 2)

local function findKauraTarget()
    local root = getRoot(); if not root then return nil end
    local best, bestDist, bestHP = nil, DW.kauraRadius + 1, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        local char = plr.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        local pRoot = char and char:FindFirstChild("HumanoidRootPart")
        if not hum or not pRoot or hum.Health <= 0 then continue end
        if DW.kauraIgnoreSheriff and getRole(plr) == "Sheriff" then continue end
        local dist = (pRoot.Position - root.Position).Magnitude
        if dist > DW.kauraRadius then continue end
        if DW.kauraTarget == "Nearest" then
            if dist < bestDist then bestDist = dist; best = plr end
        else
            if hum.Health < bestHP then bestHP = hum.Health; best = plr end
        end
    end
    return best
end

local function kauraHit(plr, knife)
    if not plr or not plr.Character then return end
    local hum = getHum(); if not hum then return end
    if knife.Parent ~= getChar() then hum:EquipTool(knife); task.wait(0.08) end
    if DW.kauraMethod == "Teleport" then
        local root = getRoot()
        local tRoot = plr.Character:FindFirstChild("HumanoidRootPart")
        if not root or not tRoot then return end
        local origin = root.CFrame
        local prevNoclip = DW.noclip; DW.noclip = true
        root.CFrame = tRoot.CFrame + Vector3.new(0, 1, 0)
        RunService.RenderStepped:Wait()
        pcall(function() knife:Activate() end)
        task.wait(0.05)
        root.CFrame = origin
        DW.noclip = prevNoclip
    else
        pcall(function() knife:Activate() end)
    end
end

task.spawn(function()
    while DW.mm2Running do
        if DW.kauraOn and not DW.flying and not DW.flinging then
            local knife = hasTool(LocalPlayer, "Knife")
            if not knife then task.wait(0.5); continue end
            if os.clock() - DW.kauraLastHit >= DW.kauraDelay then
                local target = findKauraTarget()
                if target then
                    DW.kauraLastHit = os.clock()
                    kauraHit(target, knife)
                end
            end
        end
        task.wait(0.08)
    end
end)

KauraSec:Slider({ Text = "Killaura speed (0 = off)", Flag = "dw_kauraSpeed",
    Min = 0, Max = 3, Decimals = 1, Default = 0,
    Callback = function(v)
        if v <= 0 then DW.kauraOn = false; DW.kauraDelay = 0
        else DW.kauraOn = true; DW.kauraDelay = v end
    end })
KauraSec:Slider({ Text = "Radius", Flag = "dw_kauraRadius", Min = 3, Max = 50, Default = 12,
    Callback = function(v) DW.kauraRadius = v end })
KauraSec:Dropdown({ Text = "Method", Flag = "dw_kauraMethod",
    Options = { "Activate", "Teleport" }, Default = "Activate",
    Callback = function(v) DW.kauraMethod = v end })
KauraSec:Dropdown({ Text = "Target priority", Flag = "dw_kauraTarget",
    Options = { "Nearest", "LowestHP" }, Default = "Nearest",
    Callback = function(v) DW.kauraTarget = v end })
KauraSec:Toggle({ Text = "Ignore Sheriff", Flag = "dw_kauraIgnoreSheriff", Default = true,
    Callback = function(v) DW.kauraIgnoreSheriff = v end })

local FlingSec = MM2Tab:Section("Fling", 1)
FlingSec:Slider({ Text = "Fling power", Flag = "dw_flingPower",
    Min = 20000, Max = 200000, Default = 80000,
    Callback = function(v) DW.flingPower = v end })
FlingSec:Slider({ Text = "Fling duration", Flag = "dw_flingTime",
    Min = 0.5, Max = 5, Decimals = 1, Default = 1.8,
    Callback = function(v) DW.flingTime = v end })
FlingSec:Dropdown({ Text = "Fling method", Flag = "dw_flingMethod",
    Options = { "Velocity", "Angular", "Both" }, Default = "Both",
    Callback = function(v) DW.flingMethod = v end })

local function flingPlayer(plr, label)
    if DW.flinging then return end
    if DW.flying then notify("Fling", "Turn Fly off first.", 2, "warning"); return end
    local root, hum = getRoot(), getHum()
    local tChar = plr and plr.Character
    local tRoot = tChar and tChar:FindFirstChild("HumanoidRootPart")
    local tHum = tChar and tChar:FindFirstChildOfClass("Humanoid")
    if not root or not hum then notify("Fling", "No character.", 2, "warning"); return end
    if not tRoot or not tHum or tHum.Health <= 0 then
        notify("Fling", (label or "Target") .. " not found.", 2, "warning"); return
    end
    DW.flinging = true
    local origin = root.CFrame
    local prevNoclip = DW.noclip; DW.noclip = true
    local t0 = os.clock()
    while os.clock() - t0 < DW.flingTime do
        local r = getRoot()
        local tr = tChar:FindFirstChild("HumanoidRootPart")
        if not r or not tr or not tr.Parent then break end
        r.CFrame = CFrame.new(tr.Position)
        if DW.flingMethod == "Velocity" or DW.flingMethod == "Both" then
            r.AssemblyLinearVelocity = Vector3.new(DW.flingPower, DW.flingPower, DW.flingPower)
        end
        if DW.flingMethod == "Angular" or DW.flingMethod == "Both" then
            r.AssemblyAngularVelocity = Vector3.new(DW.flingPower, DW.flingPower, DW.flingPower)
        end
        RunService.Heartbeat:Wait()
    end
    for _ = 1, 12 do
        local r = getRoot(); if not r then break end
        r.AssemblyLinearVelocity = Vector3.zero
        r.AssemblyAngularVelocity = Vector3.zero
        r.CFrame = origin
        RunService.Heartbeat:Wait()
    end
    DW.noclip = prevNoclip
    DW.flinging = false
    notify("Fling", "Flung " .. plr.DisplayName .. ".", 2, "success")
end

FlingSec:Button({ Text = "Fling murderer",
    Callback = function() refreshRoles(); task.wait(0.2); flingPlayer(getMurderer(), "Murderer") end })
FlingSec:Button({ Text = "Fling sheriff",
    Callback = function() refreshRoles(); task.wait(0.2); flingPlayer(getSheriff(), "Sheriff") end })

-- ─────────────── MISC TAB ───────────────
local MiscTab = Window:Tab({ Title = "Misc", Icon = "settings" })

local UtilSec = MiscTab:Section("Utility", 1)
UtilSec:Toggle({ Text = "Anti-AFK", Flag = "dw_antiAfk", Default = true,
    Callback = function(v) DW.antiAfk = v end })
track(LocalPlayer.Idled:Connect(function()
    if not DW.antiAfk then return end
    pcall(function() VirtualUser:CaptureController(); VirtualUser:ClickButton2(Vector2.new()) end)
end))
UtilSec:Button({ Text = "Reset character",
    Callback = function() local h = getHum(); if h then h.Health = 0 end end })
UtilSec:Input({ Text = "Quick note", Flag = "dw_note", Default = "", Placeholder = "Type something...",
    Callback = function(s) notify("Note", s ~= "" and s or "(empty)", 2, "accent") end })

local ServerSec = MiscTab:Section("Server", 1)
ServerSec:Button({ Text = "Copy Job ID",
    Callback = function()
        pcall(function() setclipboard(game.JobId) end)
        notify("Copied", "Job ID copied.", 2, "success")
    end })
ServerSec:Button({ Text = "Rejoin server",
    Callback = function()
        notify("Rejoining", "Teleporting back…", 2, "accent"); task.wait(0.5)
        if #Players:GetPlayers() <= 1 then
            LocalPlayer:Kick("\nRejoining…"); task.wait()
            TeleportService:Teleport(game.PlaceId, LocalPlayer)
        else
            TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, LocalPlayer)
        end
    end })

local function unloadAll()
    stopFly(); stopBundleAnim()
    for _, fn in ipairs(Cleanups) do pcall(fn) end
    table.clear(Cleanups)
    workspace.Gravity = DW.defaultGravity
    workspace.CurrentCamera.FieldOfView = DW.defaultFOV
    for k, val in pairs(DW.origLighting) do pcall(function() Lighting[k] = val end) end
    local h = getHum(); if h then h.WalkSpeed = 16; h.JumpPower = 50 end
    for _, c in ipairs(Connections) do pcall(function() c:Disconnect() end) end
    table.clear(Connections)
    pcall(function() Window:Destroy() end)
    pcall(function() Library:Destroy() end)
end
ServerSec:Button({ Text = "Unload Dragon Ware", Callback = unloadAll })

local AntiFlingSec = MiscTab:Section("Anti-Fling", 2)
track(RunService.Heartbeat:Connect(function()
    if not DW.antiflingOn then return end
    if DW.flying or DW.flinging then return end
    local root = getRoot(); if not root then return end
    local now = tick()
    local pos = root.Position
    if DW.antiflingLastPos and now - DW.antiflingLastTime < 0.15 then
        local jump = (pos - DW.antiflingLastPos).Magnitude
        if jump > DW.antiflingThreshold and DW.antiflingSnap then
            root.CFrame = CFrame.new(DW.antiflingLastPos)
            root.AssemblyLinearVelocity = Vector3.zero
            root.AssemblyAngularVelocity = Vector3.zero
        end
    end
    DW.antiflingLastPos = pos
    DW.antiflingLastTime = now
end))
AntiFlingSec:Toggle({ Text = "Anti-Fling", Flag = "dw_antifling", Default = false,
    Callback = function(v) DW.antiflingOn = v; if v then DW.antiflingLastPos = nil end end })
AntiFlingSec:Slider({ Text = "Threshold", Flag = "dw_afThreshold", Min = 20, Max = 500, Default = 100,
    Callback = function(v) DW.antiflingThreshold = v end })
AntiFlingSec:Toggle({ Text = "Snap back", Flag = "dw_afSnap", Default = true,
    Callback = function(v) DW.antiflingSnap = v end })

local FpsSec = MiscTab:Section("FPS Booster", 2)

local function applyFpsBoost()
    for _, item in ipairs(DW.fpsBoostClean) do
        pcall(function() item.obj.Enabled = item.old end)
    end
    DW.fpsBoostClean = {}
    if not DW.fpsBoostOn then return end
    local isHigh = DW.fpsBoostLevel == "High"
    local isMedium = DW.fpsBoostLevel == "Medium" or isHigh
    for _, obj in ipairs(workspace:GetDescendants()) do
        local disable = false
        if obj:IsA("ParticleEmitter") then disable = true end
        if isMedium and (obj:IsA("Trail") or obj:IsA("Beam") or obj:IsA("Fire") or obj:IsA("Smoke")) then disable = true end
        if isHigh and (obj:IsA("PointLight") or obj:IsA("SpotLight") or obj:IsA("SurfaceLight") or obj:IsA("Sparkles")) then disable = true end
        if disable then
            table.insert(DW.fpsBoostClean, { obj = obj, old = obj.Enabled })
            obj.Enabled = false
        end
    end
    for _, fx in ipairs(Lighting:GetChildren()) do
        if fx:IsA("PostEffect") then
            table.insert(DW.fpsBoostClean, { obj = fx, old = fx.Enabled })
            fx.Enabled = false
        end
    end
    notify("FPS Booster", "Applied (" .. DW.fpsBoostLevel .. ")", 2, "success")
end

FpsSec:Toggle({ Text = "FPS Booster", Flag = "dw_fpsBoost", Default = false,
    Callback = function(v) DW.fpsBoostOn = v; applyFpsBoost() end })
FpsSec:Dropdown({ Text = "Level", Flag = "dw_fpsLevel",
    Options = { "Low", "Medium", "High" }, Default = "Medium",
    Callback = function(v) DW.fpsBoostLevel = v; if DW.fpsBoostOn then applyFpsBoost() end end })
FpsSec:Button({ Text = "Apply now", Callback = applyFpsBoost })

local KillSoundSec = MiscTab:Section("Kill Sound", 1)
local killPacks = {
    Neverlose = "rbxassetid://8679627751", Neverlose2 = "rbxassetid://6895079853",
    Nixware = "rbxassetid://6042053626", Gamesense = "rbxassetid://4814280505",
    Fatality = "rbxassetid://158012252", Skeet = "rbxassetid://18865835568",
    CSGO = "rbxassetid://17724151829", Headshot = "rbxassetid://17724154662",
    Hitmarker = "rbxassetid://104183530656718",
}
local function getKillSoundId()
    if DW.killSoundPack == "Custom" then
        local id = DW.killSoundCustomId
        if not id or id == "" then return nil end
        if not id:match("^rbxassetid://") then id = "rbxassetid://" .. id:gsub("%D", "") end
        return id
    end
    return killPacks[DW.killSoundPack]
end
local function playKillSound()
    local id = getKillSoundId(); if not id or id == "" then return end
    local s = Instance.new("Sound")
    s.SoundId = id
    s.Volume = DW.killSoundVolume
    s.Parent = SoundService
    s:Play()
    Debris:AddItem(s, 6)
end
local function watchHumanoid(plr, hum)
    if DW.watchedHums[hum] then return end
    DW.watchedHums[hum] = true
    track(hum.Died:Connect(function()
        DW.watchedHums[hum] = nil
        if not DW.killSoundOn then return end
        if plr == LocalPlayer then return end
        if getRole(plr) ~= "Murderer" then return end
        if DW.killSoundOnlyMe and getRole(LocalPlayer) ~= "Sheriff" then return end
        task.spawn(playKillSound)
        local root = plr.Character and plr.Character:FindFirstChild("HumanoidRootPart")
        if root and DW.killFxOn then spawnKillEffect(root.Position) end
    end))
end
local function watchKillerPlayer(plr)
    if plr == LocalPlayer then return end
    track(plr.CharacterAdded:Connect(function(char)
        local hum = char:WaitForChild("Humanoid", 5)
        if hum then watchHumanoid(plr, hum) end
    end))
    if plr.Character then
        local hum = plr.Character:FindFirstChildOfClass("Humanoid")
        if hum then watchHumanoid(plr, hum) end
    end
end
for _, plr in ipairs(Players:GetPlayers()) do watchKillerPlayer(plr) end
track(Players.PlayerAdded:Connect(watchKillerPlayer))

KillSoundSec:Toggle({ Text = "Kill sound (murderer death)", Flag = "dw_killSound", Default = false,
    Callback = function(v) DW.killSoundOn = v end })
KillSoundSec:Dropdown({ Text = "Sound pack", Flag = "dw_killPack",
    Options = { "Neverlose", "Neverlose2", "Nixware", "Gamesense", "Fatality", "Skeet", "CSGO", "Headshot", "Hitmarker", "Custom" },
    Default = "Neverlose",
    Callback = function(v) DW.killSoundPack = v end })
KillSoundSec:Input({ Text = "Custom sound ID", Flag = "dw_killCustomId", Default = "",
    Placeholder = "rbxassetid://...",
    Callback = function(s) DW.killSoundCustomId = s end })
KillSoundSec:Slider({ Text = "Volume", Flag = "dw_killVol", Min = 0.1, Max = 5, Decimals = 1, Default = 1.5,
    Callback = function(v) DW.killSoundVolume = v end })
KillSoundSec:Toggle({ Text = "Only when I'm Sheriff", Flag = "dw_killOnlyMe", Default = false,
    Callback = function(v) DW.killSoundOnlyMe = v end })
KillSoundSec:Button({ Text = "Test sound", Callback = playKillSound })

local TradeSec = MiscTab:Section("Trade", 2)
local function scanTradeRemotes()
    DW.tradeRemotes = {}
    for _, obj in ipairs(RS:GetDescendants()) do
        if (obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction"))
            and obj.Name:lower():find("trade") then
            table.insert(DW.tradeRemotes, obj)
        end
    end
end
scanTradeRemotes()
track(RS.DescendantAdded:Connect(function(obj)
    if (obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction"))
        and obj.Name:lower():find("trade") then
        table.insert(DW.tradeRemotes, obj)
    end
end))

local function playerNames()
    local names = {}
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer then table.insert(names, plr.Name) end
    end
    if #names == 0 then table.insert(names, "(none)") end
    return names
end

local pn = playerNames()
TradeSec:Dropdown({ Text = "Target player", Flag = "dw_tradeTarget",
    Options = pn, Default = pn[1],
    Callback = function(v) DW.selectedPlayer = v end })
TradeSec:Input({ Text = "Whitelist (ники через запятую)", Flag = "dw_tradeWL", Default = "",
    Placeholder = "user1, user2",
    Callback = function(s)
        DW.whitelist = {}
        for name in string.gmatch(s, "[^,%s]+") do
            DW.whitelist[string.lower(name)] = true
        end
    end })

local function sendTradeRequest(targetName)
    if not targetName then notify("Trade", "No target.", 2, "warning"); return end
    local target = Players:FindFirstChild(targetName)
    if not target then notify("Trade", "Player not found.", 2, "warning"); return end
    local sent = false
    for _, r in ipairs(DW.tradeRemotes) do
        if r:IsA("RemoteEvent") then
            pcall(function() r:FireServer(target); sent = true end)
            if sent then break end
        end
    end
    notify("Trade", sent and ("Sent to " .. target.DisplayName) or "No trade remote.",
        2, sent and "success" or "warning")
end

TradeSec:Button({ Text = "Send trade request",
    Callback = function() sendTradeRequest(DW.selectedPlayer) end })
TradeSec:Button({ Text = "Accept pending trade",
    Callback = function()
        for _, r in ipairs(DW.tradeRemotes) do
            if r:IsA("RemoteEvent") then pcall(function() r:FireServer("accept") end) end
        end
        notify("Trade", "Accept sent.", 2, "success")
    end })
TradeSec:Button({ Text = "Decline pending trade",
    Callback = function()
        for _, r in ipairs(DW.tradeRemotes) do
            if r:IsA("RemoteEvent") then pcall(function() r:FireServer("decline") end) end
        end
        notify("Trade", "Decline sent.", 2, "success")
    end })
TradeSec:Button({ Text = "Rescan trade remotes",
    Callback = function()
        scanTradeRemotes()
        notify("Trade", "Found " .. #DW.tradeRemotes .. " remotes.", 3, "accent")
    end })

-- ─────────────── FUN TAB ───────────────
local FunTab = Window:Tab({ Title = "Fun", Icon = "star" })

-- ── Tracers (ESP-линии к murder/sheriff) ──
local TracerSec = FunTab:Section("Tracers", 1)

local tracerGui = Instance.new("ScreenGui")
tracerGui.Name = "DragonWareTracers"
tracerGui.ResetOnSpawn = false
tracerGui.IgnoreGuiInset = true
tracerGui.Parent = guiParent()
table.insert(Cleanups, function() tracerGui:Destroy() end)

local function clearTracers()
    for _, o in pairs(DW.tracerObjs) do pcall(function() o:Destroy() end) end
    DW.tracerObjs = {}
end

track(RunService.RenderStepped:Connect(function()
    if not DW.tracersOn then
        if next(DW.tracerObjs) then clearTracers() end
        return
    end
    local cam = workspace.CurrentCamera
    local bottom = Vector2.new(cam.ViewportSize.X / 2, cam.ViewportSize.Y)
    local keep = {}
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        local role = getRole(plr)
        local col
        if role == "Murderer" and DW.tracerMurderer then col = DW.tracerColorM
        elseif role == "Sheriff" and DW.tracerSheriff then col = DW.tracerColorS
        else continue end
        local char = plr.Character
        local head = char and (char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart"))
        if not head then continue end
        local sp, onScreen = cam:WorldToViewportPoint(head.Position)
        if not onScreen then continue end
        local to = Vector2.new(sp.X, sp.Y)
        local delta = to - bottom
        local length = delta.Magnitude
        if length < 1 then continue end
        local angle = math.deg(math.atan2(delta.Y, delta.X))
        local f = DW.tracerObjs[plr]
        if not f then
            f = Instance.new("Frame")
            f.BorderSizePixel = 0
            f.ZIndex = 5
            f.AnchorPoint = Vector2.new(0, 0.5)
            f.Parent = tracerGui
            DW.tracerObjs[plr] = f
        end
        f.BackgroundColor3 = col
        f.Size = UDim2.fromOffset(length, DW.tracerThickness)
        f.Position = UDim2.fromOffset(bottom.X, bottom.Y)
        f.Rotation = angle
        keep[plr] = true
    end
    for plr, f in pairs(DW.tracerObjs) do
        if not keep[plr] then f:Destroy(); DW.tracerObjs[plr] = nil end
    end
end))

TracerSec:Toggle({ Text = "Enable tracers", Flag = "dw_tracersOn", Default = false,
    Callback = function(v) DW.tracersOn = v end })
TracerSec:Toggle({ Text = "Tracer to Murderer", Flag = "dw_tracerM", Default = true,
    Callback = function(v) DW.tracerMurderer = v end })
TracerSec:ColorPicker({ Text = "Murderer tracer color", Flag = "dw_tracerColorM", Default = DW.tracerColorM,
    Callback = function(c) DW.tracerColorM = c end })
TracerSec:Toggle({ Text = "Tracer to Sheriff", Flag = "dw_tracerS", Default = false,
    Callback = function(v) DW.tracerSheriff = v end })
TracerSec:ColorPicker({ Text = "Sheriff tracer color", Flag = "dw_tracerColorS", Default = DW.tracerColorS,
    Callback = function(c) DW.tracerColorS = c end })
TracerSec:Slider({ Text = "Thickness", Flag = "dw_tracerThick", Min = 1, Max = 6, Default = 2,
    Callback = function(v) DW.tracerThickness = v end })

-- ── Bullet Tracers ──
local BTSec = FunTab:Section("Bullet Tracers", 1)

BTSec:Toggle({ Text = "Enable bullet tracers", Flag = "dw_bulletTracersOn", Default = false,
    Callback = function(v) DW.bulletTracersOn = v end })
BTSec:ColorPicker({ Text = "Tracer color", Flag = "dw_btColor", Default = DW.bulletTracerColor,
    Callback = function(c) DW.bulletTracerColor = c end })
BTSec:Slider({ Text = "Thickness", Flag = "dw_btThick", Min = 0.05, Max = 1, Decimals = 2, Default = 0.15,
    Callback = function(v) DW.bulletTracerThick = v end })
BTSec:Slider({ Text = "Lifetime (сек)", Flag = "dw_btLife", Min = 0.1, Max = 2, Decimals = 2, Default = 0.35,
    Callback = function(v) DW.bulletTracerLife = v end })
BTSec:Button({ Text = "Тест трассера (перед собой)",
    Callback = function()
        local r = getRoot()
        if r and DW.bulletTracersOn then
            local from = r.Position + Vector3.new(0, 1.5, 0)
            local to   = from + r.CFrame.LookVector * 40
            drawBulletTracer(from, to, DW.bulletTracerColor)
            notify("Bullet Tracers", "Тест создан.", 1.5, "accent")
        else
            notify("Bullet Tracers", "Включи тоггл сначала.", 2, "warning")
        end
    end })

-- ── Auto-Dodge ──
local DodgeSec = FunTab:Section("Auto-Dodge", 1)

task.spawn(function()
    while DW.mm2Running do
        if DW.dodgeOn then
            local murd = getMurderer()
            local myRoot = getRoot()
            if murd and murd.Character and myRoot then
                local mRoot = murd.Character:FindFirstChild("HumanoidRootPart")
                if mRoot then
                    local delta = myRoot.Position - mRoot.Position
                    if delta.Magnitude < DW.dodgeRadius and delta.Magnitude > 0.5 then
                        local away = Vector3.new(delta.X, 0, delta.Z).Unit
                        myRoot.AssemblyLinearVelocity = Vector3.new(
                            away.X * DW.dodgePower, DW.dodgePower, away.Z * DW.dodgePower)
                    end
                end
            end
        end
        task.wait(0.08)
    end
end)

DodgeSec:Toggle({ Text = "Auto-Dodge murderer", Flag = "dw_dodgeOn", Default = false,
    Callback = function(v) DW.dodgeOn = v end })
DodgeSec:Slider({ Text = "Trigger radius", Flag = "dw_dodgeRad", Min = 5, Max = 100, Default = 30,
    Callback = function(v) DW.dodgeRadius = v end })
DodgeSec:Slider({ Text = "Dodge power", Flag = "dw_dodgePow", Min = 20, Max = 200, Default = 60,
    Callback = function(v) DW.dodgePower = v end })

-- ── Character Effects ──
local CharFxSec = FunTab:Section("Character Effects", 1)

task.spawn(function()
    while DW.mm2Running do
        local char = getChar()
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            local hrp = char:FindFirstChild("HumanoidRootPart")

            local fire = char:FindFirstChild("DWFire", true)
            if DW.charFireOn and not fire and hum then
                fire = Instance.new("Fire")
                fire.Name = "DWFire"
                fire.Size = 10
                fire.Heat = 10
                fire.Parent = hum
            elseif not DW.charFireOn and fire then
                fire:Destroy()
            end

            local sp = char:FindFirstChild("DWSparkles", true)
            if DW.charSparklesOn and not sp and hum then
                sp = Instance.new("Sparkles")
                sp.Name = "DWSparkles"
                sp.SparkleColor = Color3.fromRGB(255, 220, 100)
                sp.Parent = hum
            elseif not DW.charSparklesOn and sp then
                sp:Destroy()
            end

            local pl = char:FindFirstChild("DWLight", true)
            if DW.charLightOn and not pl and hrp then
                pl = Instance.new("PointLight")
                pl.Name = "DWLight"
                pl.Brightness = 3
                pl.Range = 20
                pl.Color = Color3.fromRGB(200, 180, 255)
                pl.Parent = hrp
            elseif not DW.charLightOn and pl then
                pl:Destroy()
            end

            if DW.charRainbowOn then
                DW.rainbowHue = (DW.rainbowHue + 0.015) % 1
                local col = Color3.fromHSV(DW.rainbowHue, 0.8, 1)
                for _, d in ipairs(char:GetDescendants()) do
                    if d:IsA("BasePart") and not d:FindFirstChildOfClass("SpecialMesh") then
                        d.Color = col
                    end
                end
                local pl2 = char:FindFirstChild("DWLight", true)
                if pl2 and pl2:IsA("PointLight") then pl2.Color = col end
            end
        end
        task.wait(0.1)
    end
end)

CharFxSec:Toggle({ Text = "Fire on character", Flag = "dw_charFire", Default = false,
    Callback = function(v) DW.charFireOn = v end })
CharFxSec:Toggle({ Text = "Sparkles on character", Flag = "dw_charSparkles", Default = false,
    Callback = function(v) DW.charSparklesOn = v end })
CharFxSec:Toggle({ Text = "Rainbow character", Flag = "dw_charRainbow", Default = false,
    Callback = function(v) DW.charRainbowOn = v end })
CharFxSec:Toggle({ Text = "Glow (point light)", Flag = "dw_charLight", Default = false,
    Callback = function(v) DW.charLightOn = v end })

-- ── Dragon Ware Users ──
local DWUSec = FunTab:Section("Dragon Ware Users", 2)

DWUSec:Toggle({ Text = "Show DW users visuals", Flag = "dw_dwUsersOn", Default = false,
    Hint = "Корблокс + хедлесс для других игроков с этим же скриптом.",
    Callback = function(v) DW.dwUsersOn = v end })
DWUSec:Toggle({ Text = "Korblox (правая нога)", Flag = "dw_dwKorblox", Default = true,
    Callback = function(v)
        DW.dwKorbloxOn = v
        for plr in pairs(DW.dwVisuals) do removeDwVisuals(plr) end
    end })
DWUSec:Toggle({ Text = "Headless (голова)", Flag = "dw_dwHeadless", Default = true,
    Callback = function(v)
        DW.dwHeadlessOn = v
        for plr in pairs(DW.dwVisuals) do removeDwVisuals(plr) end
    end })
DWUSec:Button({ Text = "Кто сейчас DW-юзер?",
    Callback = function()
        local names = {}
        for plr in pairs(DW.dwUsers) do
            table.insert(names, plr.DisplayName)
        end
        if #names == 0 then
            notify("DW Users", "Никого не найдено.", 3, "warning")
        else
            notify("DW Users (" .. #names .. ")", table.concat(names, ", "), 5, "success")
        end
    end })

-- ── Fun Actions ──
local FunActSec = FunTab:Section("Fun Actions", 2)

FunActSec:Button({ Text = "Teleport to murderer",
    Callback = function()
        refreshRoles(); task.wait(0.2)
        local m = getMurderer()
        local r = getRoot()
        local mRoot = m and m.Character and m.Character:FindFirstChild("HumanoidRootPart")
        if mRoot and r then
            r.CFrame = mRoot.CFrame + Vector3.new(0, 3, 0)
            notify("Fun", "Teleported to " .. m.DisplayName, 2, "success")
        else
            notify("Fun", "Murderer not found.", 2, "warning")
        end
    end })

FunActSec:Button({ Text = "Teleport murderer to me",
    Callback = function()
        refreshRoles(); task.wait(0.2)
        local m = getMurderer()
        local r = getRoot()
        local mRoot = m and m.Character and m.Character:FindFirstChild("HumanoidRootPart")
        if mRoot and r then
            mRoot.CFrame = r.CFrame + r.CFrame.LookVector * 4
            notify("Fun", "Pulled " .. m.DisplayName, 2, "success")
        else
            notify("Fun", "Murderer not found.", 2, "warning")
        end
    end })

FunActSec:Button({ Text = "Nuclear fling (all players)",
    Callback = function()
        local r = getRoot(); if not r then return end
        local prevNoclip = DW.noclip; DW.noclip = true
        notify("Fun", "Nuclear fling launched!", 2, "accent")
        task.spawn(function()
            for _ = 1, 50 do
                for _, plr in ipairs(Players:GetPlayers()) do
                    if plr == LocalPlayer then continue end
                    local c = plr.Character
                    local hrp = c and c:FindFirstChild("HumanoidRootPart")
                    if hrp then
                        local pr = getRoot(); if not pr then break end
                        pr.CFrame = hrp.CFrame
                        pr.AssemblyLinearVelocity = Vector3.new(5e5, 5e5, 5e5)
                        pr.AssemblyAngularVelocity = Vector3.new(5e5, 5e5, 5e5)
                    end
                end
                RunService.Heartbeat:Wait()
            end
            DW.noclip = prevNoclip
        end)
    end })

FunActSec:Button({ Text = "Fake death (ragdoll)",
    Callback = function()
        local char = getChar()
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum then
            hum:ChangeState(Enum.HumanoidStateType.Physics)
            hum.PlatformStand = true
            task.wait(2.5)
            hum.PlatformStand = false
        end
    end })

FunActSec:Button({ Text = "Drop everything in backpack",
    Callback = function()
        local char = getChar()
        local hum  = char and char:FindFirstChildOfClass("Humanoid")
        local pack = LocalPlayer:FindFirstChildOfClass("Backpack")
        if not pack or not hum then return end
        for _, tool in ipairs(pack:GetChildren()) do
            pcall(function() hum:UnequipTools(); tool.Parent = char end)
        end
        local c = getChar()
        if c then
            for _, tool in ipairs(c:GetChildren()) do
                if tool:IsA("Tool") then
                    pcall(function() tool.Parent = workspace end)
                end
            end
        end
    end })

FunActSec:Button({ Text = "Spam notifications",
    Callback = function()
        for i = 1, 6 do
            task.spawn(function()
                task.wait(i * 0.15)
                notify("Fun " .. i, "Спам-сообщение #" .. i, 1.5, "accent")
            end)
        end
    end })

-- ─────────────── MOBILE TAB ───────────────
local MobileTab = Window:Tab({ Title = "Mobile", Icon = "smartphone" })

local MobileSec = MobileTab:Section("Master", 1)

local btnGui = Instance.new("ScreenGui")
btnGui.Name = "DragonWareButtons"
btnGui.ResetOnSpawn = false
btnGui.DisplayOrder = 90
btnGui.Parent = guiParent()
table.insert(Cleanups, function() btnGui:Destroy() end)

local function makeButton(id, text, yOffset, callback)
    local b = Instance.new("TextButton")
    b.Size = UDim2.fromOffset(DW.buttonSize, 44)
    b.Position = DW.savedPositions[id] or UDim2.new(1, -150, 0.5, yOffset)
    b.BackgroundColor3 = Color3.fromRGB(20, 20, 24)
    b.BackgroundTransparency = 0.15
    b.TextColor3 = Color3.fromRGB(235, 235, 235)
    b.Font = Enum.Font.GothamMedium
    b.TextSize = 14
    b.Text = text
    b.AutoButtonColor = true
    b.Visible = false
    b.Parent = btnGui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = b
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(100, 149, 255)
    stroke.Thickness = 1.2
    stroke.Parent = b

    local dragging, moved, startInput, startPos = false, false, nil, nil
    b.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            dragging, moved = true, false
            startInput, startPos = input.Position, b.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                    if not moved then task.spawn(callback) end
                end
            end)
        end
    end)
    track(UserInputService.InputChanged:Connect(function(input)
        if not dragging then return end
        if input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch then
            local d = input.Position - startInput
            if d.Magnitude > 8 then moved = true end
            if moved then
                b.Position = UDim2.new(
                    startPos.X.Scale, startPos.X.Offset + d.X,
                    startPos.Y.Scale, startPos.Y.Offset + d.Y)
                DW.savedPositions[id] = b.Position
            end
        end
    end))
    DW.onScreen[id] = b
    return b
end

local actions = {
    { id = "Shoot",  text = "Shoot Murderer", cb = shootMurderer },
    { id = "Grab",   text = "Grab Gun",       cb = function() grabGun(false) end },
    { id = "FlingM", text = "Fling Murderer", cb = function() refreshRoles(); task.wait(0.2); flingPlayer(getMurderer(), "Murderer") end },
    { id = "FlingS", text = "Fling Sheriff",  cb = function() refreshRoles(); task.wait(0.2); flingPlayer(getSheriff(), "Sheriff") end },
    { id = "BombJ",  text = "Bomb Jump",      cb = bombJump },
    { id = "KillA",  text = "Kill All",       cb = killAll },
    { id = "KAura",  text = "Killaura",       cb = function()
          if DW.kauraOn then DW.kauraOn = false; DW.kauraDelay = 0
          else DW.kauraOn = true; DW.kauraDelay = 0.6 end
          notify("Killaura", DW.kauraOn and "On (0.6s)" or "Off", 1.5, "accent")
      end },
    { id = "Aura",   text = "Auras",          cb = function() DW.auraOn = not DW.auraOn end },
    { id = "Cross",  text = "Crosshair",      cb = function() DW.crossOn = not DW.crossOn; rebuildCrosshair() end },
    { id = "KSound", text = "Kill Sound",     cb = function() DW.killSoundOn = not DW.killSoundOn end },
    { id = "Anim",   text = "Bundle Anim",    cb = function()
          if DW.bundleAnimPlaying then stopBundleAnim()
          elseif DW.bundleAnimId ~= "" then playBundleAnim(DW.bundleAnimId) end
      end },
}

for i, a in ipairs(actions) do
    makeButton(a.id, a.text, -200 + (i - 1) * 48, a.cb)
end

MobileSec:Toggle({ Text = "Show all buttons", Flag = "dw_showAllBtns", Default = false,
    Callback = function(v)
        for _, b in pairs(DW.onScreen) do b.Visible = v end
    end })
MobileSec:Slider({ Text = "Button size", Flag = "dw_btnSize", Min = 80, Max = 200, Default = 130,
    Callback = function(v)
        DW.buttonSize = v
        for _, b in pairs(DW.onScreen) do b.Size = UDim2.fromOffset(v, 44) end
    end })

local BtnListSec = MobileTab:Section("On-screen buttons", 2)
for _, a in ipairs(actions) do
    BtnListSec:Toggle({ Text = "Show: " .. a.text,
        Callback = function(v)
            local b = DW.onScreen[a.id]
            if b then b.Visible = v end
        end })
end

-- ─────────────── BINDS TAB ───────────────
local BindsTab = Window:Tab({ Title = "Binds", Icon = "keyboard" })
local BindsSec = BindsTab:Section("Actions", 1)

BindsSec:Keybind({ Text = "Shoot murderer", Flag = "dw_kbShoot", Default = "Q", Callback = shootMurderer })
BindsSec:Keybind({ Text = "Grab gun", Flag = "dw_kbGrab", Default = "G",
    Callback = function() grabGun(false) end })
BindsSec:Keybind({ Text = "Fling murderer", Flag = "dw_kbFlingM", Default = "Z",
    Callback = function() refreshRoles(); task.wait(0.2); flingPlayer(getMurderer(), "Murderer") end })
BindsSec:Keybind({ Text = "Fling sheriff", Flag = "dw_kbFlingS", Default = "C",
    Callback = function() refreshRoles(); task.wait(0.2); flingPlayer(getSheriff(), "Sheriff") end })
BindsSec:Keybind({ Text = "Bomb jump", Flag = "dw_kbBomb", Default = "B", Callback = bombJump })
BindsSec:Keybind({ Text = "Kill all", Flag = "dw_kbKillAll", Default = "K", Callback = killAll })
BindsSec:Keybind({ Text = "Killaura toggle", Flag = "dw_kbKaura", Default = "H",
    Callback = function()
        if DW.kauraOn then DW.kauraOn = false; DW.kauraDelay = 0
        else DW.kauraOn = true; DW.kauraDelay = 0.6 end
        notify("Killaura", DW.kauraOn and "On (0.6s)" or "Off", 1.5, "accent")
    end })
BindsSec:Keybind({ Text = "Auras toggle", Flag = "dw_kbAura", Default = "J",
    Callback = function() DW.auraOn = not DW.auraOn end })
BindsSec:Keybind({ Text = "Crosshair toggle", Flag = "dw_kbCross", Default = "N",
    Callback = function() DW.crossOn = not DW.crossOn; rebuildCrosshair() end })
BindsSec:Keybind({ Text = "Kill sound toggle", Flag = "dw_kbKSound", Default = "M",
    Callback = function() DW.killSoundOn = not DW.killSoundOn end })
BindsSec:Keybind({ Text = "Bundle anim toggle", Flag = "dw_kbAnim", Default = "L",
    Callback = function()
        if DW.bundleAnimPlaying then stopBundleAnim()
        elseif DW.bundleAnimId ~= "" then playBundleAnim(DW.bundleAnimId) end
    end })
BindsSec:Keybind({ Text = "Fly toggle", Flag = "dw_kbFly", Default = "F",
    Callback = function() if DW.flying then stopFly() else startFly() end end })
BindsSec:Keybind({ Text = "Noclip toggle", Flag = "dw_kbNoclip", Default = "V",
    Callback = function() DW.noclip = not DW.noclip end })
BindsSec:Keybind({ Text = "Spin bot toggle", Flag = "dw_kbSpin", Default = "X",
    Callback = function() DW.spinOn = not DW.spinOn end })
BindsSec:Keybind({ Text = "Silent aim toggle", Flag = "dw_kbSilent", Default = "P",
    Callback = function()
        DW.silentAimOn = not DW.silentAimOn
        notify("Silent aim", DW.silentAimOn and "On" or "Off", 1.5, "accent")
    end })

-- ─────────────── SETTINGS TAB ───────────────
local SetTab = Window:Tab({ Title = "Settings", Icon = "settings" })

InterfaceManager:SetLibrary(Library)
InterfaceManager:SetWindow(Window)
InterfaceManager:SetFolder("DragonWare")
InterfaceManager:BuildInterfaceSection(SetTab, 1)

ThemeManager:SetLibrary(Library)
ThemeManager:SetFolder("DragonWare")
ThemeManager:BuildThemeSection(SetTab, 1)

SaveManager:SetLibrary(Library)
SaveManager:SetFolder("DragonWare/configs")
SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({})
SaveManager:BuildConfigSection(SetTab, 2)

Library:SetHotkeysVisible(true)

-- ─────────────── Меню по LeftControl ───────────────
do
    local applied = false

    pcall(function()
        local kb = Library.ToggleKeybind
            or Library.ToggleKey
            or (Window and Window.ToggleKeybind)
        if kb and kb.SetValue then
            kb:SetValue(Enum.KeyCode.LeftControl)
            applied = true
        end
    end)

    if not applied then
        local menuVisible = true
        track(UserInputService.InputBegan:Connect(function(input, gp)
            if gp then return end
            if input.KeyCode == Enum.KeyCode.LeftControl then
                menuVisible = not menuVisible
                pcall(function() Window:SetVisible(menuVisible) end)
            end
        end))
    end
end

notify("Dragon Ware", "Loaded. Toggle menu: LeftControl", 4, "success")

pcall(function() SaveManager:LoadAutoloadConfig() end)

return Library
