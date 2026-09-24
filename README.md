--[[
    Dragon Ware — built on MentalityUI
    Features: movement, camera, lighting, MM2 tools, shooting,
              wallbang, killaura, kill all, kill sounds, auras,
              kill effects, crosshair, theme presets, anti-fling,
              FPS booster, auto-trade, bomb jump, bundle animation,
              mobile buttons, binds.
    Only works in MM2.
]]

-- ───────────────────────────── Services ─────────────────────────────
local Players           = game:GetService("Players")
local RunService        = game:GetService("RunService")
local UserInputService  = game:GetService("UserInputService")
local Lighting          = game:GetService("Lighting")
local TeleportService   = game:GetService("TeleportService")
local VirtualUser       = game:GetService("VirtualUser")
local Stats             = game:GetService("Stats")
local SoundService      = game:GetService("SoundService")
local Debris            = game:GetService("Debris")
local VIM               = game:GetService("VirtualInputManager")
local CoreGui           = game:GetService("CoreGui")
local RS                = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local Camera      = workspace.CurrentCamera

-- ═══════════════════════ GAME GUARD ═══════════════════════════
local ALLOWED_PLACES = { [142823291] = "Murder Mystery 2" }
do
    if not ALLOWED_PLACES[game.PlaceId] then
        local list = ""
        for _, n in pairs(ALLOWED_PLACES) do list = list .. " • " .. n .. "\n" end
        pcall(function()
            LocalPlayer:Kick("\n[Dragon Ware]\n\nThis script only works in:\n" .. list ..
                "\nYour PlaceId: " .. tostring(game.PlaceId))
        end)
        return
    end
end
-- ═══════════════════════════════════════════════════════════════

-- ───────────────────────────── Central State ────────────────────────
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

    shooting = false, prediction = 0.08, aimPart = "Head",
    wallbangOn = false,

    kauraOn = false, kauraDelay = 0, kauraRadius = 12,
    kauraMethod = "Activate", kauraIgnoreSheriff = true,
    kauraTarget = "Nearest", kauraLastHit = 0,

    killSoundOn = false, killSoundVolume = 1.5, killSoundOnlyMe = false,
    killSoundPack = "Neverlose", killSoundCustomId = "",

    flingPower = 80000, flingTime = 1.8, flingMethod = "Both",

    auraOn = false,
    auraColor = Color3.fromRGB(179, 165, 255),
    auraSize = 4, auraTransparency = 0.5,
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
    autoTradeOn = false, autoTradeInterval = 3, autoTradeLast = 0,

    bombJumpPower = 250, bombJumpHorizontal = 0, bombJumpAuto = false,

    bundleAnimId = "", bundleAnimPlaying = false, bundleAnimTrack = nil,

    buttonSize = 130, onScreen = {}, savedPositions = {},
}

local Connections = DW.Connections
local Cleanups = DW.Cleanups

local function track(conn)
    table.insert(Connections, conn)
    return conn
end

-- ───────────────────────────── Helpers ──────────────────────────────
local LOGO = "97594400820219"

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

-- ───────────────────────────── Library ──────────────────────────────
local LIB_URL = "https://raw.githubusercontent.com/samuraa1/MentalityUI/main/Library.lua"
local Library = loadstring(game:HttpGet(LIB_URL))()

local function notify(title, desc, dur)
    Library:Notification({
        Title = title, Description = desc or "",
        Duration = dur or 3, Icon = LOGO,
    })
end

local fps = 60
do
    local frames, last = 0, os.clock()
    track(RunService.RenderStepped:Connect(function()
        frames += 1
        local now = os.clock()
        if now - last >= 1 then
            fps = frames; frames, last = 0, now
        end
    end))
end

-- ───────────────────────────── Window ───────────────────────────────
local Window = Library:Window({ Name = "Dragon Ware", SubName = "MentalityUI", Logo = LOGO })

Window:Category("Main")
Window:TabDivider()

local PlayerPage = Window:Page({ Name = "Player", Icon = "gamepad-2" })
local WorldPage  = Window:Page({ Name = "World",  Icon = "layers" })
local VisualPage = Window:Page({ Name = "Visual", Icon = "sparkles" })
local MiscPage   = Window:Page({ Name = "Misc",   Icon = "box" })

Window:DashboardPage({
    Name = "Dashboard", Icon = "layout-dashboard",
    WelcomeText = "WELCOME TO", HubName = "DRAGON WARE",
    StatusText = "client-side tools", Badge = "PLAYER",
    GameName = tostring(game.Name ~= "" and game.Name or "This Game"),
    GameDescription = "Movement, MM2 tools, visual FX, bomb jump, binds.",
    Links = {
        { Icon = "copy", Tooltip = "Copy Job ID",
          Callback = function()
              pcall(function() setclipboard(game.JobId) end)
              notify("Copied", "Job ID copied.", 2)
          end },
    },
    Stats = {
        { Name = "FPS", Icon = "activity", GetValue = function() return tostring(fps) end },
        { Name = "PING", Icon = "clock", GetValue = function()
            local ok, v = pcall(function()
                return math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue())
            end)
            return ok and (tostring(v) .. "ms") or "n/a"
        end },
        { Name = "TIME", Icon = "clock", GetValue = function()
            return tostring(math.floor(workspace.DistributedGameTime)) .. "s"
        end },
    },
    Credits = { { Name = "samet", Role = "MentalityUI" } },
    QuickAccess = {},
})

-- ═════════════════════════════ PLAYER PAGE ══════════════════════════
local Movement  = PlayerPage:Section({ Name = "Movement",  Icon = "zap", Side = 1 })
local Abilities = PlayerPage:Section({ Name = "Abilities", Icon = "sliders-horizontal", Side = 2 })

Movement:Toggle({
    Name = "Custom walk speed", Flag = "WalkSpeedOn", Default = false,
    Callback = function(v)
        DW.speedOn = v
        if not v then local h = getHum(); if h then h.WalkSpeed = 16 end end
    end,
})
Movement:Slider({
    Name = "Walk speed", Flag = "WalkSpeedVal",
    Min = 16, Max = 120, Default = 16, Decimals = 1,
    Callback = function(v) DW.speedVal = v end,
})
Movement:Toggle({
    Name = "Custom jump power", Flag = "JumpOn", Default = false,
    Callback = function(v)
        DW.jumpOn = v
        if not v then local h = getHum(); if h then h.UseJumpPower = true; h.JumpPower = 50 end end
    end,
})
Movement:Slider({
    Name = "Jump power", Flag = "JumpVal",
    Min = 50, Max = 250, Default = 50, Decimals = 1,
    Callback = function(v) DW.jumpVal = v end,
})

track(RunService.Heartbeat:Connect(function()
    local h = getHum()
    if not h then return end
    if DW.speedOn then h.WalkSpeed = DW.speedVal end
    if DW.jumpOn then h.UseJumpPower = true; h.JumpPower = DW.jumpVal end
end))

Movement:Divider("Gravity")
Movement:Slider({
    Name = "Gravity", Flag = "GravityVal",
    Min = 20, Max = 300, Default = math.floor(DW.defaultGravity), Decimals = 1,
    Callback = function(v) workspace.Gravity = v end,
})
Movement:Button({
    Name = "Reset gravity", Icon = "bell",
    Callback = function()
        workspace.Gravity = DW.defaultGravity
        notify("Gravity", "Restored to " .. math.floor(DW.defaultGravity) .. ".", 2)
    end,
})

-- ── Bomb Jump ──
Movement:Divider("Bomb Jump")
Movement:Slider({
    Name = "Bomb jump power", Flag = "BombJumpPower",
    Min = 50, Max = 600, Default = 250, Decimals = 1,
    Callback = function(v) DW.bombJumpPower = v end,
})
Movement:Slider({
    Name = "Bomb jump horizontal", Flag = "BombJumpHoriz",
    Min = 0, Max = 300, Default = 0, Decimals = 1, Suffix = " studs",
    Callback = function(v) DW.bombJumpHorizontal = v end,
})

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

Movement:Button({
    Name = "Launch now (bomb jump)", Icon = "rocket",
    Callback = bombJump,
})
Movement:Toggle({
    Name = "Auto bomb jump (on Space)", Flag = "BombJumpAuto", Default = false,
    Callback = function(v) DW.bombJumpAuto = v end,
})
track(UserInputService.JumpRequest:Connect(function()
    if not DW.bombJumpAuto then return end
    bombJump()
end))

-- ── Speed glitch ──
Movement:Toggle({
    Name = "Speed glitch", Flag = "SpeedGlitch", Default = false,
    Callback = function(v) DW.glitchOn = v end,
})
Movement:Slider({
    Name = "Jump speed", Flag = "GlitchSpeed",
    Min = 16, Max = 300, Default = 60, Decimals = 1,
    Callback = function(v) DW.glitchSpeed = v end,
})
Movement:Dropdown({
    Name = "Glitch mode", Flag = "GlitchMode",
    Items = { "Jump", "Always" }, Default = "Jump",
    Callback = function(v) DW.glitchMode = v end,
})
Movement:Dropdown({
    Name = "Glitch method", Flag = "GlitchMethod",
    Items = { "Velocity", "CFrame" }, Default = "Velocity",
    Callback = function(v) DW.glitchMethod = v end,
})

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

Abilities:Toggle({
    Name = "Infinite jump", Flag = "InfJump", Default = false,
    Callback = function(v) DW.infJump = v end,
})
track(UserInputService.JumpRequest:Connect(function()
    if not DW.infJump then return end
    local h = getHum()
    if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end
end))

Abilities:Toggle({
    Name = "Noclip", Flag = "Noclip", Default = false,
    Callback = function(v) DW.noclip = v end,
})
track(RunService.Stepped:Connect(function()
    if not DW.noclip then return end
    local c = getChar()
    if not c then return end
    for _, p in ipairs(c:GetDescendants()) do
        if p:IsA("BasePart") and p.CanCollide then p.CanCollide = false end
    end
end))

Abilities:Toggle({
    Name = "Spin bot", Flag = "SpinBot", Default = false,
    Callback = function(v) DW.spinOn = v end,
})
Abilities:Slider({
    Name = "Spin speed", Flag = "SpinSpeed",
    Min = 200, Max = 8000, Default = 2200, Decimals = 1, Suffix = "°/s",
    Callback = function(v) DW.spinSpeed = v end,
})
Abilities:Dropdown({
    Name = "Spin axis", Flag = "SpinAxis",
    Items = { "Y", "XY", "XYZ" }, Default = "Y",
    Callback = function(v) DW.spinAxis = v end,
})

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

Abilities:Toggle({
    Name = "Fly", Flag = "FlyOn", Default = false,
    Callback = function(v) if v then startFly() else stopFly() end end,
})
Abilities:Slider({
    Name = "Fly speed", Flag = "FlySpeed",
    Min = 10, Max = 250, Default = 60, Decimals = 1,
    Callback = function(v) DW.flySpeed = v end,
})

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

-- ── Bundle Animation ──
local AnimSec = PlayerPage:Section({ Name = "Bundle Animation", Icon = "person-standing", Side = 2 })

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
    if not animator then
        animator = Instance.new("Animator")
        animator.Parent = hum
    end
    local anim = Instance.new("Animation")
    anim.AnimationId = id
    local ok, track = pcall(function() return animator:LoadAnimation(anim) end)
    if ok and track then
        track.Looped = true
        track.Priority = Enum.AnimationPriority.Action
        track:Play()
        DW.bundleAnimTrack = track
        DW.bundleAnimPlaying = true
    end
end

AnimSec:Dropdown({
    Name = "Preset", Flag = "BundleAnimPreset",
    Items = { "None", "GangnamStyle", "Floss", "OrangeJustice", "Robot", "DefaultIdle" },
    Default = "None",
    Callback = function(v)
        local id = animPresets[v]
        DW.bundleAnimId = id
        if id == "" then stopBundleAnim() else playBundleAnim(id) end
    end,
})
AnimSec:Textbox({
    Name = "Custom Animation ID", Flag = "BundleAnimCustom",
    Placeholder = "rbxassetid://...", Finished = true,
    Callback = function(s)
        if s == "" then return end
        if not s:match("^rbxassetid://") then
            s = "rbxassetid://" .. s:gsub("%D", "")
        end
        DW.bundleAnimId = s
        playBundleAnim(s)
    end,
})
AnimSec:Button({ Name = "Play animation", Icon = "play",
    Callback = function()
        if DW.bundleAnimId == "" then notify("Bundle Animation", "ID пустой.", 2); return end
        playBundleAnim(DW.bundleAnimId)
    end })
AnimSec:Button({ Name = "Stop animation", Icon = "square", Callback = stopBundleAnim })
track(LocalPlayer.CharacterAdded:Connect(function()
    stopBundleAnim()
    if DW.bundleAnimId ~= "" then
        task.wait(0.5); playBundleAnim(DW.bundleAnimId)
    end
end))

-- ═════════════════════════════ WORLD PAGE ═══════════════════════════
local CamSec   = WorldPage:Section({ Name = "Camera", Icon = "sliders-horizontal", Side = 1 })
local LightSec = WorldPage:Section({ Name = "Lighting", Icon = "layers", Side = 2 })

CamSec:Toggle({
    Name = "Custom FOV", Flag = "FovOn", Default = false,
    Callback = function(v)
        DW.fovOn = v
        if not v then workspace.CurrentCamera.FieldOfView = DW.defaultFOV end
    end,
})
CamSec:Slider({
    Name = "Field of view", Flag = "FovVal",
    Min = 30, Max = 120, Default = 70, Decimals = 1,
    Callback = function(v) DW.fovVal = v end,
})
track(RunService.RenderStepped:Connect(function()
    if DW.fovOn then workspace.CurrentCamera.FieldOfView = DW.fovVal end
end))
CamSec:Dropdown({
    Name = "Camera mode", Flag = "CamMode",
    Items = { "Classic", "LockFirstPerson" }, Default = "Classic",
    Callback = function(v) pcall(function() LocalPlayer.CameraMode = Enum.CameraMode[v] end) end,
})

LightSec:Toggle({
    Name = "Fullbright", Flag = "Fullbright", Default = false,
    Callback = function(v)
        DW.fullbright = v
        if not v then
            for k, val in pairs(DW.origLighting) do
                pcall(function() Lighting[k] = val end)
            end
        end
    end,
})
track(RunService.RenderStepped:Connect(function()
    if not DW.fullbright then return end
    Lighting.Brightness = 2; Lighting.ClockTime = 14
    Lighting.FogEnd = 1e6; Lighting.GlobalShadows = false
    Lighting.Ambient = Color3.fromRGB(178, 178, 178)
end))
LightSec:Slider({
    Name = "Time of day", Flag = "TimeOfDay",
    Min = 0, Max = 24, Default = math.floor(Lighting.ClockTime), Decimals = 1, Suffix = "h",
    Callback = function(v) if not DW.fullbright then Lighting.ClockTime = v end end,
})
LightSec:Label("Ambient tint"):Colorpicker({
    Name = "Ambient tint", Flag = "AmbientTint", Default = Lighting.Ambient,
    Callback = function(c) if not DW.fullbright then Lighting.Ambient = c end end,
})

-- ═════════════════════════════ VISUAL PAGE ══════════════════════════
local AuraSec = VisualPage:Section({ Name = "Auras", Icon = "sparkles", Side = 1 })

local auraFolder = Instance.new("Folder")
auraFolder.Name = "DragonWareAuras"
auraFolder.Parent = guiParent()
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
    local ring = Instance.new("Part")
    ring.Anchored = true; ring.CanCollide = false; ring.CanQuery = false; ring.CanTouch = false
    ring.Material = Enum.Material.Neon
    ring.Color = DW.auraColor; ring.Transparency = DW.auraTransparency
    ring.Size = Vector3.new(DW.auraSize, 0.08, DW.auraSize)
    ring.Shape = Enum.PartType.Cylinder
    ring.CFrame = CFrame.new(root.Position) * CFrame.Angles(0, 0, math.rad(90))
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
    particles.Lifetime = NumberRange.new(1, 1.5); particles.Rate = 40
    particles.Speed = NumberRange.new(0.5, 2)
    particles.SpreadAngle = Vector2.new(180, 180)
    particles.Acceleration = Vector3.new(0, 5, 0)
    particles.LightEmission = 1; particles.LightInfluence = 0
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
                o.ring.Size = Vector3.new(DW.auraSize, 0.08, DW.auraSize)
                o.ring.CFrame = CFrame.new(root.Position)
                    * CFrame.Angles(0, yRot, 0) * CFrame.Angles(0, 0, math.rad(90))
                o.ring.Transparency = DW.auraTransparency
                o.light.Brightness = DW.auraLight and 3 or 0
                o.light.Range = DW.auraSize * 2
                if o.particles then o.particles.Enabled = DW.auraParticles end
                if DW.auraRainbow then
                    local col = Color3.fromHSV((tick() * 0.2) % 1, 0.8, 1)
                    o.ring.Color = col; o.light.Color = col
                    if o.particles then o.particles.Color = ColorSequence.new(col) end
                else
                    o.ring.Color = DW.auraColor; o.light.Color = DW.auraColor
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

AuraSec:Toggle({ Name = "Enable auras", Flag = "AuraOn", Default = false,
    Callback = function(v) DW.auraOn = v end })

local auraPresets = {
    Default = { color = Color3.fromRGB(179, 165, 255), size = 4, speed = 3, transparency = 0.5, rainbow = false },
    Demon   = { color = Color3.fromRGB(255, 40, 40),   size = 5, speed = 5, transparency = 0.3, rainbow = false },
    Angel   = { color = Color3.fromRGB(255, 255, 255), size = 6, speed = 2, transparency = 0.4, rainbow = false },
    Galaxy  = { color = Color3.fromRGB(150, 100, 255), size = 7, speed = 4, transparency = 0.5, rainbow = true  },
    Fire    = { color = Color3.fromRGB(255, 120, 0),   size = 5, speed = 6, transparency = 0.3, rainbow = false },
    Toxic   = { color = Color3.fromRGB(120, 255, 0),   size = 4, speed = 4, transparency = 0.4, rainbow = false },
    Ice     = { color = Color3.fromRGB(100, 220, 255), size = 5, speed = 2, transparency = 0.4, rainbow = false },
}
AuraSec:Dropdown({
    Name = "Aura preset", Flag = "AuraPreset",
    Items = { "Default", "Demon", "Angel", "Galaxy", "Fire", "Toxic", "Ice" },
    Default = "Default",
    Callback = function(v)
        local p = auraPresets[v]; if not p then return end
        DW.auraColor = p.color; DW.auraSize = p.size
        DW.auraSpeed = p.speed; DW.auraTransparency = p.transparency
        DW.auraRainbow = p.rainbow
    end,
})
AuraSec:Label("Aura color"):Colorpicker({
    Name = "Aura color", Flag = "AuraColor", Default = DW.auraColor,
    Callback = function(c) DW.auraColor = c end })
AuraSec:Slider({ Name = "Aura size", Flag = "AuraSize",
    Min = 2, Max = 20, Default = 4, Decimals = 1, Suffix = " studs",
    Callback = function(v) DW.auraSize = v end })
AuraSec:Slider({ Name = "Transparency", Flag = "AuraTransparency",
    Min = 0, Max = 1, Default = 0.5, Decimals = 1,
    Callback = function(v) DW.auraTransparency = v end })
AuraSec:Toggle({ Name = "Rainbow aura", Flag = "AuraRainbow", Default = false,
    Callback = function(v) DW.auraRainbow = v end })
AuraSec:Toggle({ Name = "Spin ring", Flag = "AuraSpin", Default = true,
    Callback = function(v) DW.auraSpin = v end })
AuraSec:Slider({ Name = "Spin speed", Flag = "AuraSpeed",
    Min = 1, Max = 20, Default = 3, Decimals = 1, Suffix = "x",
    Callback = function(v) DW.auraSpeed = v end })
AuraSec:Toggle({ Name = "Particles", Flag = "AuraParticles", Default = true,
    Callback = function(v) DW.auraParticles = v end })
AuraSec:Toggle({ Name = "Point light", Flag = "AuraLight", Default = true,
    Callback = function(v) DW.auraLight = v end })

-- Kill Effect
local KillFxSec = VisualPage:Section({ Name = "Kill Effect", Icon = "bomb", Side = 2 })
local killFxFolder = Instance.new("Folder")
killFxFolder.Name = "DragonWareKillFX"
killFxFolder.Parent = guiParent()
table.insert(Cleanups, function() killFxFolder:Destroy() end)

local function spawnKillEffect(position)
    if not DW.killFxOn or not position then return end
    local color = DW.killFxRainbow and Color3.fromHSV((tick() * 0.3) % 1, 0.85, 1) or DW.killFxColor
    if DW.killFxType == "Explosion" or DW.killFxType == "Sparkles" then
        local part = Instance.new("Part")
        part.Anchored = true; part.CanCollide = false; part.CanQuery = false; part.CanTouch = false
        part.Transparency = 1; part.Size = Vector3.new(0.1, 0.1, 0.1)
        part.CFrame = CFrame.new(position); part.Parent = killFxFolder
        local emitter = Instance.new("ParticleEmitter")
        emitter.Texture = "rbxassetid://241876428"
        emitter.Color = ColorSequence.new(color)
        emitter.Size = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0),
            NumberSequenceKeypoint.new(0.3, DW.killFxSize),
            NumberSequenceKeypoint.new(1, 0),
        })
        emitter.Lifetime = NumberRange.new(0.8, 1.4)
        emitter.Speed = NumberRange.new(20, 40)
        emitter.SpreadAngle = Vector2.new(180, 180)
        emitter.LightEmission = 1; emitter.Rate = 0
        emitter:Emit(DW.killFxType == "Explosion" and 80 or 60)
        emitter.Parent = part
        Debris:AddItem(part, 3)
    elseif DW.killFxType == "Fire" then
        local firePart = Instance.new("Part")
        firePart.Anchored = true; firePart.CanCollide = false; firePart.CanQuery = false; firePart.CanTouch = false
        firePart.Transparency = 1; firePart.Size = Vector3.new(0.1, 0.1, 0.1)
        firePart.CFrame = CFrame.new(position); firePart.Parent = killFxFolder
        local fire = Instance.new("Fire")
        fire.Color = color; fire.SecondaryColor = color
        fire.Size = DW.killFxSize; fire.Heat = 5
        fire.Parent = firePart
        Debris:AddItem(firePart, 2)
    end
end

KillFxSec:Toggle({ Name = "Enable kill effect", Flag = "KillFxOn", Default = false,
    Callback = function(v) DW.killFxOn = v end })
KillFxSec:Dropdown({ Name = "Effect type", Flag = "KillFxType",
    Items = { "Explosion", "Sparkles", "Fire" }, Default = "Explosion",
    Callback = function(v) DW.killFxType = v end })
KillFxSec:Label("Effect color"):Colorpicker({
    Name = "Effect color", Flag = "KillFxColor", Default = DW.killFxColor,
    Callback = function(c) DW.killFxColor = c end })
KillFxSec:Slider({ Name = "Effect size", Flag = "KillFxSize",
    Min = 2, Max = 20, Default = 8, Decimals = 1, Suffix = " studs",
    Callback = function(v) DW.killFxSize = v end })
KillFxSec:Toggle({ Name = "Rainbow effect", Flag = "KillFxRainbow", Default = false,
    Callback = function(v) DW.killFxRainbow = v end })

-- Crosshair
local CrossSec = VisualPage:Section({ Name = "Crosshair", Icon = "crosshair", Side = 2 })
local crossGui = Instance.new("ScreenGui")
crossGui.Name = "DragonWareCrosshair"; crossGui.ResetOnSpawn = false
crossGui.IgnoreGuiInset = true; crossGui.DisplayOrder = 95
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
        f.BackgroundColor3 = c; f.BorderSizePixel = 0
        f.Position = UDim2.new(0.5, x, 0.5, y)
        f.Size = UDim2.fromOffset(w, h)
        f.AnchorPoint = Vector2.new(0.5, 0.5)
        f.ZIndex = 3; f.Parent = crossHolder
    end
    if DW.crossStyle == "Dot" then line(0, 0, t + 1, t + 1)
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
        circle.ZIndex = 3; circle.Parent = crossHolder
        Instance.new("UICorner", circle).CornerRadius = UDim.new(1, 0)
        local stroke = Instance.new("UIStroke", circle)
        stroke.Color = c; stroke.Thickness = t
    end
end

CrossSec:Toggle({ Name = "Enable crosshair", Flag = "CrossOn", Default = false,
    Callback = function(v) DW.crossOn = v; rebuildCrosshair() end })
CrossSec:Dropdown({ Name = "Style", Flag = "CrossStyle",
    Items = { "Dot", "Cross", "Circle", "Cross+Dot" }, Default = "Cross",
    Callback = function(v) DW.crossStyle = v; rebuildCrosshair() end })
CrossSec:Label("Crosshair color"):Colorpicker({
    Name = "Crosshair color", Flag = "CrossColor", Default = DW.crossColor,
    Callback = function(c) DW.crossColor = c; rebuildCrosshair() end })
CrossSec:Slider({ Name = "Size", Flag = "CrossSize",
    Min = 4, Max = 40, Default = 12, Decimals = 1,
    Callback = function(v) DW.crossSize = v; rebuildCrosshair() end })
CrossSec:Slider({ Name = "Thickness", Flag = "CrossThickness",
    Min = 1, Max = 6, Default = 2, Decimals = 1,
    Callback = function(v) DW.crossThickness = v; rebuildCrosshair() end })
CrossSec:Slider({ Name = "Gap", Flag = "CrossGap",
    Min = 0, Max = 20, Default = 4, Decimals = 1,
    Callback = function(v) DW.crossGap = v; rebuildCrosshair() end })
CrossSec:Toggle({ Name = "Outline", Flag = "CrossOutline", Default = true,
    Callback = function(v) DW.crossOutline = v; rebuildCrosshair() end })

-- Theme Presets
local ThemeSec = VisualPage:Section({ Name = "Theme Presets", Icon = "palette", Side = 1 })
local themes = {
    Purple = Color3.fromRGB(179, 165, 255), RedDragon = Color3.fromRGB(255, 60, 60),
    CyanIce = Color3.fromRGB(80, 210, 255), MatrixGreen = Color3.fromRGB(50, 255, 100),
    BloodMoon = Color3.fromRGB(200, 20, 40), Sunset = Color3.fromRGB(255, 130, 60),
    Void = Color3.fromRGB(90, 30, 130),
}
ThemeSec:Dropdown({
    Name = "Theme preset", Flag = "ThemePreset",
    Items = { "Purple", "RedDragon", "CyanIce", "MatrixGreen", "BloodMoon", "Sunset", "Void" },
    Default = "Purple",
    Callback = function(v)
        local c = themes[v]; if not c then return end
        pcall(function() Library:ChangeTheme({ Accent = c }) end)
        pcall(function() Library.Theme.Accent = c end)
        DW.auraColor = c; DW.crossColor = c; DW.killFxColor = c
    end,
})

-- ═════════════════════════════ MISC PAGE ════════════════════════════
local Utility = MiscPage:Section({ Name = "Utility", Icon = "box", Side = 1 })
local Server  = MiscPage:Section({ Name = "Server",  Icon = "layers", Side = 2 })

Utility:Toggle({ Name = "Anti-AFK", Flag = "AntiAfk", Default = true,
    Callback = function(v) DW.antiAfk = v end })
DW.antiAfk = true
track(LocalPlayer.Idled:Connect(function()
    if not DW.antiAfk then return end
    pcall(function() VirtualUser:CaptureController(); VirtualUser:ClickButton2(Vector2.new()) end)
end))
Utility:Button({ Name = "Reset character", Icon = "bell",
    Callback = function() local h = getHum(); if h then h.Health = 0 end end })
Utility:Textbox({ Flag = "QuickNote", Placeholder = "Quick note…", Finished = true,
    Callback = function(s) notify("Saved note", s ~= "" and s or "(empty)", 2) end })

Server:Label("Job ID: " .. string.sub(game.JobId, 1, 8) .. "…")
Server:Button({ Name = "Copy Job ID", Icon = "copy",
    Callback = function()
        pcall(function() setclipboard(game.JobId) end)
        notify("Copied", "Job ID copied.", 2)
    end })
Server:Button({ Name = "Rejoin server", Icon = "bell",
    Callback = function()
        notify("Rejoining", "Teleporting back…", 2); task.wait(0.5)
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
    Library:Unload()
end
Server:Button({ Name = "Unload Dragon Ware", Icon = "bell", Callback = unloadAll })
Server:Keybind({ Name = "Panic unload", Flag = "PanicKey",
    Default = Enum.KeyCode.End, Callback = unloadAll })

-- ═════════════════════════ MM2 FEATURES ═════════════════════════
local ESPSec   = WorldPage:Section({ Name = "ESP (roles)", Icon = "eye", Side = 1 })
local GunSec   = MiscPage:Section({ Name = "Gun", Icon = "zap", Side = 1 })
local ShootSec = MiscPage:Section({ Name = "Shoot murderer", Icon = "target", Side = 2 })

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
    if hasTool(plr, "Gun") then return "Sheriff" end
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

-- ESP
local espFolder = Instance.new("Folder")
espFolder.Name = "DragonWareESP"; espFolder.Parent = guiParent()
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
    hl.FillTransparency = 0.55; hl.OutlineTransparency = 0
    hl.Adornee = char; hl.Parent = espFolder
    local bb = Instance.new("BillboardGui")
    bb.AlwaysOnTop = true; bb.Size = UDim2.fromOffset(160, 20)
    bb.StudsOffset = Vector3.new(0, 3, 0)
    bb.Adornee = char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
    bb.Parent = espFolder
    local lbl = Instance.new("TextLabel")
    lbl.BackgroundTransparency = 1; lbl.Size = UDim2.fromScale(1, 1)
    lbl.Font = Enum.Font.GothamMedium; lbl.TextSize = 13
    lbl.TextStrokeTransparency = 0.4; lbl.Parent = bb
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
                        o.hl.FillColor = c; o.hl.OutlineColor = c
                        o.lbl.TextColor3 = c; o.lbl.Visible = DW.showNames
                        o.lbl.Text = plr.DisplayName .. " [" .. role .. "]"
                    else clearESP(plr) end
                else clearESP(plr) end
            end
        end
        task.wait(0.3)
    end
end)

ESPSec:Toggle({ Name = "Murderer ESP", Flag = "EspMurderer", Default = false,
    Callback = function(v) DW.roleOn.Murderer = v end })
ESPSec:Label("Murderer color"):Colorpicker({
    Name = "Murderer color", Flag = "EspMurdererColor",
    Default = DW.roleColor.Murderer,
    Callback = function(c) DW.roleColor.Murderer = c end })
ESPSec:Toggle({ Name = "Sheriff ESP", Flag = "EspSheriff", Default = false,
    Callback = function(v) DW.roleOn.Sheriff = v end })
ESPSec:Label("Sheriff color"):Colorpicker({
    Name = "Sheriff color", Flag = "EspSheriffColor",
    Default = DW.roleColor.Sheriff,
    Callback = function(c) DW.roleColor.Sheriff = c end })
ESPSec:Toggle({ Name = "Innocent ESP", Flag = "EspInnocent", Default = false,
    Callback = function(v) DW.roleOn.Innocent = v end })
ESPSec:Label("Innocent color"):Colorpicker({
    Name = "Innocent color", Flag = "EspInnocentColor",
    Default = DW.roleColor.Innocent,
    Callback = function(c) DW.roleColor.Innocent = c end })
ESPSec:Toggle({ Name = "Show names", Flag = "EspNames", Default = true,
    Callback = function(v) DW.showNames = v end })

-- Gun
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
        if not silent then notify("Grab gun", "No dropped gun on the map.", 2) end
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
        notify("Grab gun", hasTool(LocalPlayer, "Gun") and "Got the gun." or "Tried.", 2)
    end
    DW.grabbing = false
end

GunSec:Toggle({ Name = "Auto grab gun", Flag = "AutoGrabGun", Default = false,
    Callback = function(v) DW.autoGrab = v end })
GunSec:Button({ Name = "Grab gun now", Icon = "bell",
    Callback = function() grabGun(false) end })
task.spawn(function()
    while DW.mm2Running do
        if DW.autoGrab then pcall(grabGun, true) end
        task.wait(0.4)
    end
end)

-- Shoot murderer
local function shootMurderer()
    if DW.shooting then return end
    local gun = hasTool(LocalPlayer, "Gun")
    if not gun then notify("Shoot murderer", "You don't have the gun.", 2); return end
    refreshRoles()
    local murd = getMurderer()
    local mChar = murd and murd.Character
    local mRoot = mChar and mChar:FindFirstChild("HumanoidRootPart")
    local mHum  = mChar and mChar:FindFirstChildOfClass("Humanoid")
    if not mRoot or not mHum or mHum.Health <= 0 then
        notify("Shoot murderer", "Murderer not found.", 2); return
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
        notify("Shoot murderer", "Wallbang at " .. murd.DisplayName, 2)
        return
    end

    DW.shooting = true
    local hum = getHum()
    if hum and gun.Parent ~= getChar() then
        hum:EquipTool(gun); task.wait(0.15)
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
    pcall(function() gun:Activate() end)
    task.wait(0.4)
    DW.shooting = false
end

ShootSec:Button({ Name = "Shoot murderer", Icon = "bell", Callback = shootMurderer })
ShootSec:Slider({ Name = "Aim part", Flag = "AimPartIdx",
    Min = 1, Max = 3, Default = 1, Decimals = 1,
    Callback = function(v)
        local i = math.floor(v + 0.5)
        DW.aimPart = i == 1 and "Head" or i == 2 and "HumanoidRootPart" or "UpperTorso"
    end })
ShootSec:Slider({ Name = "Lead prediction", Flag = "ShootPrediction",
    Min = 0, Max = 0.4, Default = 0.08, Decimals = 1, Suffix = "s",
    Callback = function(v) DW.prediction = v end })
ShootSec:Toggle({ Name = "Wallbang (teleport)", Flag = "Wallbang", Default = false,
    Callback = function(v) DW.wallbangOn = v end })
ShootSec:Button({ Name = "Who is the murderer?", Icon = "bell",
    Callback = function()
        refreshRoles(); task.wait(0.3)
        local m = getMurderer()
        notify("Murderer", m and m.DisplayName or "Not detected.", 3)
    end })

-- ── Kill All ──
local KillAllSec = MiscPage:Section({ Name = "Kill All", Icon = "skull", Side = 2 })

local killAllOn = false
local killAllInterval = 0.5
local killAllLast = 0
local killAllRange = 500

local function killAll()
    local knife = hasTool(LocalPlayer, "Knife")
    local gun = hasTool(LocalPlayer, "Gun")
    if not knife and not gun then
        notify("Kill All", "У тебя нет оружия.", 2); return
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
        if (pRoot.Position - myRoot.Position).Magnitude > killAllRange then continue end

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
            task.wait(0.05)
            myRoot.CFrame = origin
            DW.noclip = prevNoclip
        end
        task.wait(0.03)
    end
end

KillAllSec:Button({ Name = "Kill all now", Icon = "skull", Callback = killAll })
KillAllSec:Slider({ Name = "Range", Flag = "KillAllRange",
    Min = 50, Max = 2000, Default = 500, Decimals = 1, Suffix = " studs",
    Callback = function(v) killAllRange = v end })
KillAllSec:Slider({ Name = "Interval", Flag = "KillAllInterval",
    Min = 0.1, Max = 5, Default = 0.5, Decimals = 1, Suffix = "s",
    Callback = function(v) killAllInterval = v end })
KillAllSec:Toggle({ Name = "Auto kill all", Flag = "KillAllAuto", Default = false,
    Callback = function(v) killAllOn = v end })

task.spawn(function()
    while DW.mm2Running do
        if killAllOn and os.clock() - killAllLast >= killAllInterval then
            killAllLast = os.clock()
            pcall(killAll)
        end
        task.wait(0.1)
    end
end)

-- Killaura
local KillauraSec = MiscPage:Section({ Name = "Killaura (Murderer)", Icon = "sword", Side = 1 })

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

KillauraSec:Slider({ Name = "Killaura speed (0 = off)", Flag = "KillauraSpeed",
    Min = 0, Max = 3, Default = 0, Decimals = 1, Suffix = "s",
    Callback = function(v)
        if v <= 0 then DW.kauraOn = false; DW.kauraDelay = 0
        else DW.kauraOn = true; DW.kauraDelay = v end
    end })
KillauraSec:Slider({ Name = "Radius", Flag = "KillauraRadius",
    Min = 3, Max = 50, Default = 12, Decimals = 1, Suffix = " studs",
    Callback = function(v) DW.kauraRadius = v end })
KillauraSec:Dropdown({ Name = "Method", Flag = "KillauraMethod",
    Items = { "Activate", "Teleport" }, Default = "Activate",
    Callback = function(v) DW.kauraMethod = v end })
KillauraSec:Dropdown({ Name = "Target priority", Flag = "KillauraTarget",
    Items = { "Nearest", "LowestHP" }, Default = "Nearest",
    Callback = function(v) DW.kauraTarget = v end })
KillauraSec:Toggle({ Name = "Ignore Sheriff", Flag = "KillauraIgnoreSheriff", Default = true,
    Callback = function(v) DW.kauraIgnoreSheriff = v end })

-- Anti-Fling
local AntiFlingSec = MiscPage:Section({ Name = "Anti-Fling", Icon = "shield", Side = 1 })
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
AntiFlingSec:Toggle({ Name = "Anti-Fling", Flag = "AntiFlingOn", Default = false,
    Callback = function(v) DW.antiflingOn = v; if v then DW.antiflingLastPos = nil end end })
AntiFlingSec:Slider({ Name = "Threshold", Flag = "AntiFlingThreshold",
    Min = 20, Max = 500, Default = 100, Decimals = 1, Suffix = " studs",
    Callback = function(v) DW.antiflingThreshold = v end })
AntiFlingSec:Toggle({ Name = "Snap back", Flag = "AntiFlingSnap", Default = true,
    Callback = function(v) DW.antiflingSnap = v end })

-- FPS Booster
local FpsSec = MiscPage:Section({ Name = "FPS Booster", Icon = "activity", Side = 1 })

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
    notify("FPS Booster", "Applied (" .. DW.fpsBoostLevel .. ")", 2)
end

FpsSec:Toggle({ Name = "FPS Booster", Flag = "FpsBoostOn", Default = false,
    Callback = function(v) DW.fpsBoostOn = v; applyFpsBoost() end })
FpsSec:Dropdown({ Name = "Level", Flag = "FpsBoostLevel",
    Items = { "Low", "Medium", "High" }, Default = "Medium",
    Callback = function(v) DW.fpsBoostLevel = v; if DW.fpsBoostOn then applyFpsBoost() end end })
FpsSec:Button({ Name = "Apply now", Icon = "zap", Callback = applyFpsBoost })

-- Kill Sound
local KillSoundSec = MiscPage:Section({ Name = "Kill Sound", Icon = "music", Side = 2 })
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
    s.SoundId = id; s.Volume = DW.killSoundVolume
    s.Parent = SoundService; s:Play()
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

local killSoundToggle = KillSoundSec:Toggle({
    Name = "Kill sound (murderer death)", Flag = "KillSoundOn", Default = false,
    Callback = function(v) DW.killSoundOn = v end })
local ksSub = killSoundToggle:Settings(160)
ksSub:Dropdown({ Name = "Sound pack", Flag = "KillSoundPack",
    Items = { "Neverlose", "Neverlose2", "Nixware", "Gamesense", "Fatality", "Skeet", "CSGO", "Headshot", "Hitmarker", "Custom" },
    Default = "Neverlose",
    Callback = function(v) DW.killSoundPack = v end })
ksSub:Textbox({ Name = "Custom sound ID", Flag = "KillSoundCustom",
    Placeholder = "rbxassetid:// или цифры", Finished = true,
    Callback = function(s) DW.killSoundCustomId = s end })
ksSub:Slider({ Name = "Volume", Flag = "KillSoundVol",
    Min = 0.1, Max = 5, Default = 1.5, Decimals = 1,
    Callback = function(v) DW.killSoundVolume = v end })
ksSub:Toggle({ Name = "Only when I'm Sheriff", Flag = "KillSoundOnlyMe", Default = false,
    Callback = function(v) DW.killSoundOnlyMe = v end })
KillSoundSec:Button({ Name = "Test sound", Icon = "music", Callback = playKillSound })

-- Fling
local FlingSec = MiscPage:Section({ Name = "Fling", Icon = "zap", Side = 2 })
FlingSec:Slider({ Name = "Fling power", Flag = "FlingPower",
    Min = 20000, Max = 200000, Default = 80000, Decimals = 1,
    Callback = function(v) DW.flingPower = v end })
FlingSec:Slider({ Name = "Fling duration", Flag = "FlingDuration",
    Min = 0.5, Max = 5, Default = 1.8, Decimals = 1, Suffix = "s",
    Callback = function(v) DW.flingTime = v end })
FlingSec:Dropdown({ Name = "Fling method", Flag = "FlingMethod",
    Items = { "Velocity", "Angular", "Both" }, Default = "Both",
    Callback = function(v) DW.flingMethod = v end })

local function flingPlayer(plr, label)
    if DW.flinging then return end
    if DW.flying then notify("Fling", "Turn Fly off first.", 2); return end
    local root, hum = getRoot(), getHum()
    local tChar = plr and plr.Character
    local tRoot = tChar and tChar:FindFirstChild("HumanoidRootPart")
    local tHum = tChar and tChar:FindFirstChildOfClass("Humanoid")
    if not root or not hum then notify("Fling", "No character.", 2); return end
    if not tRoot or not tHum or tHum.Health <= 0 then
        notify("Fling", (label or "Target") .. " not found.", 2); return
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
    notify("Fling", "Flung " .. plr.DisplayName .. ".", 2)
end

FlingSec:Button({ Name = "Fling murderer", Icon = "bell",
    Callback = function() refreshRoles(); task.wait(0.2); flingPlayer(getMurderer(), "Murderer") end })
FlingSec:Button({ Name = "Fling sheriff", Icon = "bell",
    Callback = function() refreshRoles(); task.wait(0.2); flingPlayer(getSheriff(), "Sheriff") end })

-- Trade
local TradeSec = MiscPage:Section({ Name = "Trade", Icon = "arrow-left-right", Side = 2 })
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
    return names
end

TradeSec:Dropdown({ Name = "Target player", Flag = "TradeTarget",
    Items = playerNames(), Default = playerNames()[1],
    Callback = function(v) DW.selectedPlayer = v end })
TradeSec:Textbox({ Name = "Whitelist", Flag = "TradeWhitelist",
    Placeholder = "user1, user2", Finished = true,
    Callback = function(s)
        DW.whitelist = {}
        for name in string.gmatch(s, "[^,%s]+") do
            DW.whitelist[string.lower(name)] = true
        end
    end })

local function sendTradeRequest(targetName)
    if not targetName then notify("Trade", "No target.", 2); return end
    local target = Players:FindFirstChild(targetName)
    if not target then notify("Trade", "Player not found.", 2); return end
    local sent = false
    for _, r in ipairs(DW.tradeRemotes) do
        if r:IsA("RemoteEvent") then
            pcall(function() r:FireServer(target); sent = true end)
            if sent then break end
        end
    end
    notify("Trade", sent and ("Sent to " .. target.DisplayName) or "No trade remote.", 2)
end

TradeSec:Button({ Name = "Send trade request", Icon = "send",
    Callback = function() sendTradeRequest(DW.selectedPlayer) end })
TradeSec:Button({ Name = "Accept pending trade", Icon = "check",
    Callback = function()
        for _, r in ipairs(DW.tradeRemotes) do
            if r:IsA("RemoteEvent") then pcall(function() r:FireServer("accept") end) end
        end
        notify("Trade", "Accept sent.", 2)
    end })
TradeSec:Button({ Name = "Decline pending trade", Icon = "x",
    Callback = function()
        for _, r in ipairs(DW.tradeRemotes) do
            if r:IsA("RemoteEvent") then pcall(function() r:FireServer("decline") end) end
        end
        notify("Trade", "Decline sent.", 2)
    end })
TradeSec:Button({ Name = "Rescan trade remotes", Icon = "refresh-cw",
    Callback = function()
        scanTradeRemotes()
        notify("Trade", "Found " .. #DW.tradeRemotes .. " remotes.", 3)
    end })

-- ═════════════════════════════ Binds ════════════════════════════════
local BindSec = MiscPage:Section({ Name = "Binds", Icon = "keyboard", Side = 2 })

BindSec:Keybind({ Name = "Shoot murderer", Flag = "BindShoot",
    Default = Enum.KeyCode.Q, Callback = shootMurderer })
BindSec:Keybind({ Name = "Grab gun", Flag = "BindGrab",
    Default = Enum.KeyCode.G, Callback = function() grabGun(false) end })
BindSec:Keybind({ Name = "Fling murderer", Flag = "BindFlingM",
    Default = Enum.KeyCode.Z,
    Callback = function() refreshRoles(); task.wait(0.2); flingPlayer(getMurderer(), "Murderer") end })
BindSec:Keybind({ Name = "Fling sheriff", Flag = "BindFlingS",
    Default = Enum.KeyCode.C,
    Callback = function() refreshRoles(); task.wait(0.2); flingPlayer(getSheriff(), "Sheriff") end })
BindSec:Keybind({ Name = "Bomb jump", Flag = "BindBomb",
    Default = Enum.KeyCode.B, Callback = bombJump })
BindSec:Keybind({ Name = "Kill all", Flag = "BindKillAll",
    Default = Enum.KeyCode.K, Callback = killAll })
BindSec:Keybind({ Name = "Killaura toggle", Flag = "BindKAura",
    Default = Enum.KeyCode.H,
    Callback = function()
        if DW.kauraOn then DW.kauraOn = false; DW.kauraDelay = 0
        else DW.kauraOn = true; DW.kauraDelay = 0.6 end
        notify("Killaura", DW.kauraOn and "On (0.6s)" or "Off", 1.5)
    end })
BindSec:Keybind({ Name = "Auras toggle", Flag = "BindAura",
    Default = Enum.KeyCode.J,
    Callback = function() DW.auraOn = not DW.auraOn end })
BindSec:Keybind({ Name = "Crosshair toggle", Flag = "BindCross",
    Default = Enum.KeyCode.N,
    Callback = function() DW.crossOn = not DW.crossOn; rebuildCrosshair() end })
BindSec:Keybind({ Name = "Kill sound toggle", Flag = "BindKSound",
    Default = Enum.KeyCode.M,
    Callback = function() pcall(function() killSoundToggle:Set(not DW.killSoundOn) end) end })
BindSec:Keybind({ Name = "Bundle anim toggle", Flag = "BindAnim",
    Default = Enum.KeyCode.L,
    Callback = function()
        if DW.bundleAnimPlaying then stopBundleAnim()
        elseif DW.bundleAnimId ~= "" then playBundleAnim(DW.bundleAnimId) end
    end })
BindSec:Keybind({ Name = "Fly toggle", Flag = "BindFly",
    Default = Enum.KeyCode.F,
    Callback = function() if DW.flying then stopFly() else startFly() end end })
BindSec:Keybind({ Name = "Noclip toggle", Flag = "BindNoclip",
    Default = Enum.KeyCode.V,
    Callback = function() DW.noclip = not DW.noclip end })
BindSec:Keybind({ Name = "Spin bot toggle", Flag = "BindSpin",
    Default = Enum.KeyCode.X,
    Callback = function() DW.spinOn = not DW.spinOn end })

-- ═════════════════════════════ Mobile UI ════════════════════════════
local BtnSec = MiscPage:Section({ Name = "Mobile UI", Icon = "smartphone", Side = 1 })

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
    b.TextSize = 14; b.Text = text
    b.AutoButtonColor = true
    b.Parent = btnGui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8); corner.Parent = b
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(100, 149, 255); stroke.Thickness = 1.2
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

local function setAllVisible(v)
    for _, b in pairs(DW.onScreen) do b.Visible = v end
end

BtnSec:Toggle({
    Name = "Show all mobile buttons", Flag = "ShowAllBtns",
    Default = UserInputService.TouchEnabled,
    Callback = setAllVisible,
})
BtnSec:Slider({
    Name = "Button size", Flag = "BtnSize",
    Min = 80, Max = 200, Default = 130, Decimals = 1,
    Callback = function(v)
        DW.buttonSize = v
        for _, b in pairs(DW.onScreen) do b.Size = UDim2.fromOffset(v, 44) end
    end,
})

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
          notify("Killaura", DW.kauraOn and "On (0.6s)" or "Off", 1.5)
      end },
    { id = "Aura",   text = "Auras",          cb = function() DW.auraOn = not DW.auraOn end },
    { id = "Cross",  text = "Crosshair",      cb = function() DW.crossOn = not DW.crossOn; rebuildCrosshair() end },
    { id = "KSound", text = "Kill Sound",     cb = function() pcall(function() killSoundToggle:Set(not DW.killSoundOn) end) end },
    { id = "Anim",   text = "Bundle Anim",    cb = function()
          if DW.bundleAnimPlaying then stopBundleAnim()
          elseif DW.bundleAnimId ~= "" then playBundleAnim(DW.bundleAnimId) end
      end },
}

local touch = UserInputService.TouchEnabled
for i, a in ipairs(actions) do
    local b = makeButton(a.id, a.text, -200 + (i - 1) * 48, a.cb)
    b.Visible = touch
    BtnSec:Toggle({
        Name = "Show: " .. a.text,
        Flag = "Btn" .. a.id,
        Default = touch,
        Callback = function(v) b.Visible = v end,
    })
end

-- ═════════════════════════════ Init ═════════════════════════════════
Library:CreateSettingsPage(Window)
Window:Init()
notify("Dragon Ware", "Loaded. Press End to panic-unload.", 3)
return nil
