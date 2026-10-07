-- LpMods Hub — Fluent UI
-- Aimbot (Drawing FOV) + Silent Aim + ESP + Hitbox + Spinbot + Movement

local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()

local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace        = game:GetService("Workspace")
local CoreGui          = game:GetService("CoreGui")

local LP     = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

local Window = Fluent:CreateWindow({
    Title = "LpMods",
    SubTitle = "by LpMods",
    TabWidth = 140,
    Size = UDim2.fromOffset(530, 380),
    Theme = "Dark",
    MinimizeKey = Enum.KeyCode.LeftControl
})

local Tabs = {
    Combat   = Window:AddTab({ Title = "Combat",   Icon = "crosshair" }),
    Visuals  = Window:AddTab({ Title = "Visuals",  Icon = "eye" }),
    Hitbox   = Window:AddTab({ Title = "Hitbox",   Icon = "box" }),
    Movement = Window:AddTab({ Title = "Movement", Icon = "gauge" }),
    Discord  = Window:AddTab({ Title = "Discord",  Icon = "disc" }),
}

local Settings = {
    Aimbot      = false,
    IgnoreDead  = false,
    FOVSize     = 100,
    ShowFOV     = false,

    SilentAim         = false,
    SilentAimFOV      = 120,
    SilentAimBodyPart = "Head",
    SilentAimTeam     = true,
    SilentAimWall     = false,
    SilentAimMaxDist  = 500,

    ESP_Master = false,
    ESP_Player = false,
    ESP_Boxes  = false,
    ESP_Lines  = false,
    ESP_Names  = false,

    HitboxEnabled = false,
    HitboxSize    = 10,
    HitboxPart    = "Todos",
    Fullbody      = false,

    Spinbot = false,

    EnableSpeed  = false,
    Speed        = 16,
    JumpEnabled  = false,
    JumpValue    = 1,
    FlyEnabled   = false,
    ThirdPerson  = false,
}

local ACCENT = Color3.fromRGB(150, 90, 240)

local function isAlly(p)
    if not p or not p.Team then return false end
    return p.Team == LP.Team
end
local function shouldIgnore(p)
    if not p or p == LP then return true end
    if isAlly(p) then return true end
    return false
end

local FOVCircle = Drawing.new("Circle")
FOVCircle.Color = ACCENT
FOVCircle.Thickness = 1.5
FOVCircle.Filled = false
FOVCircle.Transparency = 0.8
FOVCircle.Visible = false

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "LpModsFloatGui"
ScreenGui.ResetOnSpawn = false
if syn and syn.protect_gui then
    syn.protect_gui(ScreenGui)
    ScreenGui.Parent = CoreGui
elseif gethui then
    ScreenGui.Parent = gethui()
else
    ScreenGui.Parent = CoreGui
end

local FloatButton = Instance.new("ImageButton")
FloatButton.Name = "LpModsFloatBtn"
FloatButton.Parent = ScreenGui
FloatButton.Size = UDim2.new(0, 55, 0, 55)
FloatButton.Position = UDim2.new(0.05, 0, 0.2, 0)
FloatButton.Image = "rbxassetid://99262930883927"
FloatButton.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
FloatButton.BackgroundTransparency = 0.2
FloatButton.BorderSizePixel = 0
FloatButton.Active = true
FloatButton.Draggable = true

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 10)
UICorner.Parent = FloatButton

FloatButton.MouseButton1Click:Connect(function()
    if Window then Window:Minimize() end
end)

local function GetClosestPlayer()
    local Target, MaxDistance = nil, Settings.FOVSize
    local Center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LP and player.Character then
            local char = player.Character
            local head = char:FindFirstChild("Head")
            local hum  = char:FindFirstChildOfClass("Humanoid")
            local root = char:FindFirstChild("HumanoidRootPart")
            if head and hum and root then
                if Settings.IgnoreDead then
                    if hum.Health <= 0 or hum:GetState() == Enum.HumanoidStateType.Dead
                    or not char:IsDescendantOf(Workspace) then
                        continue
                    end
                else
                    if hum.Health <= 0 or hum:GetState() == Enum.HumanoidStateType.Dead then
                        continue
                    end
                end
                local headPos, onScreen = Camera:WorldToViewportPoint(head.Position)
                if onScreen then
                    local dist = (Vector2.new(headPos.X, headPos.Y) - Center).Magnitude
                    if dist <= MaxDistance then
                        MaxDistance = dist
                        Target = player
                    end
                end
            end
        end
    end
    return Target
end

RunService.RenderStepped:Connect(function()
    local Center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    FOVCircle.Position = Center
    FOVCircle.Radius = Settings.FOVSize
    FOVCircle.Visible = Settings.ShowFOV
    FOVCircle.Color = ACCENT

    if Settings.Aimbot then
        local target = GetClosestPlayer()
        if target and target.Character then
            local head = target.Character:FindFirstChild("Head")
            local hum  = target.Character:FindFirstChildOfClass("Humanoid")
            if head and hum and hum.Health > 0 then
                Camera.CFrame = CFrame.new(Camera.CFrame.Position, head.Position)
            end
        end
    end
end)

local silentTarget = nil

local function getPart(p, name)
    local c = p.Character
    return c and c:FindFirstChild(name)
end

local function isAlive(p)
    local c = p.Character
    if not c then return false end
    local h = c:FindFirstChildOfClass("Humanoid")
    return h and h.Health > 0
end

RunService.Heartbeat:Connect(function()
    if not Settings.SilentAim then silentTarget = nil; return end
    local myChar = LP.Character
    local myHRP  = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myHRP then return end

    local best, bestDist = nil, Settings.SilentAimFOV
    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LP and isAlive(p) then
            if not (Settings.SilentAimTeam and shouldIgnore(p)) then
                local part = getPart(p, Settings.SilentAimBodyPart)
                if part then
                    local d = (part.Position - myHRP.Position).Magnitude
                    if d <= Settings.SilentAimMaxDist then
                        local sp, on = Camera:WorldToViewportPoint(part.Position)
                        if on then
                            local sd = (Vector2.new(sp.X, sp.Y) - center).Magnitude
                            if sd <= bestDist then
                                bestDist = sd
                                best = { Player = p, Part = part }
                            end
                        end
                    end
                end
            end
        end
    end
    silentTarget = best
end)

_G.LpMods = _G.LpMods or {}
_G.LpMods.SilentAim = {
    getTarget = function() return silentTarget and silentTarget.Player or nil end,
    getAimPosition = function() return silentTarget and silentTarget.Part.Position or nil end,
}

local ESPDrawings = {}

local function CreateESP(player)
    if ESPDrawings[player] then return end
    local d = {
        Box  = Drawing.new("Square"),
        Line = Drawing.new("Line"),
        Name = Drawing.new("Text"),
    }
    d.Box.Color = ACCENT; d.Box.Thickness = 1.5; d.Box.Filled = false
    d.Line.Color = ACCENT; d.Line.Thickness = 1
    d.Name.Color = ACCENT; d.Name.Size = 13; d.Name.Center = true; d.Name.Outline = true
    ESPDrawings[player] = d
end

local function RemoveESP(player)
    local d = ESPDrawings[player]
    if d then
        for _, v in pairs(d) do pcall(function() v:Remove() end) end
        ESPDrawings[player] = nil
    end
    local c = player.Character
    if c and c:FindFirstChild("LpChams") then c.LpChams:Destroy() end
end

Players.PlayerRemoving:Connect(RemoveESP)

RunService.RenderStepped:Connect(function()
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LP then
            if not ESPDrawings[player] then CreateESP(player) end
            local d = ESPDrawings[player]
            local char = player.Character
            local hum  = char and char:FindFirstChildOfClass("Humanoid")

            if Settings.ESP_Master and char and hum and hum.Health > 0 then
                local cframe, size = char:GetBoundingBox()
                local screenPos, onScreen = Camera:WorldToViewportPoint(cframe.Position)
                if onScreen then
                    local topWorld    = cframe.Position + Vector3.new(0, size.Y / 2, 0)
                    local bottomWorld = cframe.Position - Vector3.new(0, size.Y / 2, 0)
                    local topScreen    = Camera:WorldToViewportPoint(topWorld)
                    local bottomScreen = Camera:WorldToViewportPoint(bottomWorld)
                    local height = math.abs(topScreen.Y - bottomScreen.Y)
                    local width  = height / 1.6

                    if Settings.ESP_Boxes then
                        d.Box.Size = Vector2.new(width, height)
                        d.Box.Position = Vector2.new(screenPos.X - width / 2, topScreen.Y)
                        d.Box.Color = ACCENT
                        d.Box.Visible = true
                    else d.Box.Visible = false end

                    if Settings.ESP_Lines then
                        d.Line.From = Vector2.new(Camera.ViewportSize.X / 2, 0)
                        d.Line.To   = Vector2.new(screenPos.X, topScreen.Y)
                        d.Line.Color = ACCENT
                        d.Line.Visible = true
                    else d.Line.Visible = false end

                    if Settings.ESP_Names then
                        d.Name.Text = player.Name
                        d.Name.Position = Vector2.new(screenPos.X, topScreen.Y - 16)
                        d.Name.Color = ACCENT
                        d.Name.Visible = true
                    else d.Name.Visible = false end

                    if Settings.ESP_Player then
                        local hl = char:FindFirstChild("LpChams")
                        if not hl then
                            hl = Instance.new("Highlight")
                            hl.Name = "LpChams"
                            hl.FillColor = ACCENT
                            hl.OutlineColor = ACCENT
                            hl.FillTransparency = 0.5
                            hl.OutlineTransparency = 0
                            hl.Parent = char
                        end
                        hl.Enabled = true
                    else
                        local hl = char:FindFirstChild("LpChams")
                        if hl then hl.Enabled = false end
                    end
                else
                    d.Box.Visible = false; d.Line.Visible = false; d.Name.Visible = false
                    local hl = char:FindFirstChild("LpChams")
                    if hl then hl.Enabled = false end
                end
            else
                d.Box.Visible = false; d.Line.Visible = false; d.Name.Visible = false
                local hl = char and char:FindFirstChild("LpChams")
                if hl then hl.Enabled = false end
            end
        end
    end
end)

local hitboxOriginals = {}

local function isEnemy(p)
    if p == LP then return false end
    if isAlly(p) then return false end
    return true
end

local function enableHitbox()
    for _, plr in ipairs(Players:GetPlayers()) do
        if isEnemy(plr) then
            local c = plr.Character
            if c then
                for _, part in ipairs(c:GetChildren()) do
                    if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                        if not hitboxOriginals[part] then
                            hitboxOriginals[part] = part.Size
                        end
                        local apply = (Settings.HitboxPart == "Todos") or (part.Name == Settings.HitboxPart)
                        if apply then
                            part.Size = hitboxOriginals[part] * (Settings.HitboxSize / 5)
                        end
                    end
                end
            end
        end
    end
end

local function disableHitbox()
    for part, size in pairs(hitboxOriginals) do
        if part and part.Parent then part.Size = size end
    end
    hitboxOriginals = {}
end

local function refreshHitbox()
    if Settings.HitboxEnabled then
        disableHitbox()
        task.wait(0.05)
        enableHitbox()
    end
end

local fullbodyData = {}
local function enableFullbody()
    for _, plr in ipairs(Players:GetPlayers()) do
        if isEnemy(plr) then
            local c = plr.Character
            if c then
                local hrp = c:FindFirstChild("HumanoidRootPart")
                if hrp then
                    if not fullbodyData[hrp] then fullbodyData[hrp] = hrp.Size end
                    hrp.Size = Vector3.new(6, 6, 6)
                end
            end
        end
    end
end
local function disableFullbody()
    for part, size in pairs(fullbodyData) do
        if part and part.Parent then part.Size = size end
    end
    fullbodyData = {}
end

RunService.RenderStepped:Connect(function()
    if not Settings.Spinbot then return end
    local c = LP.Character
    local hrp = c and c:FindFirstChild("HumanoidRootPart")
    if hrp then
        hrp.CFrame = hrp.CFrame * CFrame.Angles(0, math.rad(150), 0)
    end
end)

local function applySpeed()
    local c = LP.Character
    if not c then return end
    local h = c:FindFirstChildOfClass("Humanoid")
    if not h then return end
    h.WalkSpeed = Settings.EnableSpeed and Settings.Speed or 16
end

RunService.Stepped:Connect(function()
    if not Settings.EnableSpeed then return end
    local c = LP.Character
    if not c then return end
    local h = c:FindFirstChildOfClass("Humanoid")
    if h and h.WalkSpeed ~= Settings.Speed then h.WalkSpeed = Settings.Speed end
end)

RunService.Heartbeat:Connect(function()
    if not Settings.JumpEnabled then return end
    local c = LP.Character
    local h = c and c:FindFirstChildOfClass("Humanoid")
    if h then h.JumpPower = 50 * Settings.JumpValue; h.UseJumpPower = true end
end)

local flyBV, flyBG
local function enableFly()
    local c = LP.Character
    local hrp = c and c:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    flyBV = Instance.new("BodyVelocity")
    flyBV.MaxForce = Vector3.new(1e5, 1e5, 1e5)
    flyBV.Velocity = Vector3.zero
    flyBV.Parent = hrp
    flyBG = Instance.new("BodyGyro")
    flyBG.MaxTorque = Vector3.new(1e5, 1e5, 1e5)
    flyBG.P = 1000
    flyBG.Parent = hrp
end
local function disableFly()
    if flyBV then flyBV:Destroy(); flyBV = nil end
    if flyBG then flyBG:Destroy(); flyBG = nil end
end

UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if not Settings.FlyEnabled then return end
    if input.KeyCode == Enum.KeyCode.Space then
        if flyBV then flyBV.Velocity = Vector3.new(0, 50, 0) end
    end
end)

local thirdPersonConn
local function enableThirdPerson()
    if thirdPersonConn then thirdPersonConn:Disconnect() end
    Camera.CameraType = Enum.CameraType.Scriptable
    thirdPersonConn = RunService.RenderStepped:Connect(function()
        if not Settings.ThirdPerson then return end
        local c = LP.Character
        local hrp = c and c:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        local offset = hrp.CFrame.LookVector * -12 + Vector3.new(0, 5, 0)
        Camera.CFrame = CFrame.new(hrp.Position + offset, hrp.Position)
    end)
end
local function disableThirdPerson()
    if thirdPersonConn then thirdPersonConn:Disconnect(); thirdPersonConn = nil end
    Camera.CameraType = Enum.CameraType.Custom
    Camera.CameraSubject = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
end

Tabs.Combat:AddToggle("Aimbot", {
    Title = "Aimbot Gruda na Cabeça",
    Default = false,
    Callback = function(v) Settings.Aimbot = v end
})
Tabs.Combat:AddToggle("IgnoreDead", {
    Title = "Ignorar Mortos no Chão",
    Default = false,
    Callback = function(v) Settings.IgnoreDead = v end
})
Tabs.Combat:AddToggle("ShowFOV", {
    Title = "Mostrar FOV",
    Default = false,
    Callback = function(v) Settings.ShowFOV = v end
})
Tabs.Combat:AddSlider("FOVSize", {
    Title = "Tamanho do FOV",
    Min = 30, Max = 500, Default = 100, Rounding = 0,
    Callback = function(v) Settings.FOVSize = v end
})

Tabs.Combat:AddSection("Silent Aim")
Tabs.Combat:AddToggle("SilentAim", {
    Title = "Silent Aim",
    Default = false,
    Callback = function(v) Settings.SilentAim = v end
})
Tabs.Combat:AddSlider("SilentFOV", {
    Title = "Silent FOV",
    Min = 30, Max = 400, Default = 120, Rounding = 0,
    Callback = function(v) Settings.SilentAimFOV = v end
})
Tabs.Combat:AddDropdown("SilentPart", {
    Title = "Parte do Corpo",
    Values = {"Head", "UpperTorso", "HumanoidRootPart"},
    Default = 1,
    Multi = false,
    Callback = function(v) Settings.SilentAimBodyPart = v end
})
Tabs.Combat:AddToggle("SilentTeam", {
    Title = "Team Check",
    Default = true,
    Callback = function(v) Settings.SilentAimTeam = v end
})
Tabs.Combat:AddToggle("SilentWall", {
    Title = "Wall Check",
    Default = false,
    Callback = function(v) Settings.SilentAimWall = v end
})

Tabs.Visuals:AddToggle("ESPMaster", {
    Title = "Ativar ESP (Master)",
    Default = false,
    Callback = function(v) Settings.ESP_Master = v end
})
Tabs.Visuals:AddToggle("ESPPlayer", {
    Title = "ESP Player (Chams)",
    Default = false,
    Callback = function(v) Settings.ESP_Player = v end
})
Tabs.Visuals:AddToggle("ESPBoxes", {
    Title = "ESP Boxes",
    Default = false,
    Callback = function(v) Settings.ESP_Boxes = v end
})
Tabs.Visuals:AddToggle("ESPLines", {
    Title = "ESP Lines",
    Default = false,
    Callback = function(v) Settings.ESP_Lines = v end
})
Tabs.Visuals:AddToggle("ESPNames", {
    Title = "ESP Name",
    Default = false,
    Callback = function(v) Settings.ESP_Names = v end
})

Tabs.Hitbox:AddToggle("Hitbox", {
    Title = "Hitbox Expand",
    Default = false,
    Callback = function(v)
        Settings.HitboxEnabled = v
        if v then enableHitbox() else disableHitbox() end
    end
})
Tabs.Hitbox:AddSlider("HitboxSize", {
    Title = "Tamanho",
    Min = 1, Max = 30, Default = 10, Rounding = 0,
    Callback = function(v) Settings.HitboxSize = v; refreshHitbox() end
})
Tabs.Hitbox:AddDropdown("HitboxPart", {
    Title = "Parte",
    Values = {"Todos", "Head", "UpperTorso"},
    Default = 1,
    Multi = false,
    Callback = function(v) Settings.HitboxPart = v; refreshHitbox() end
})
Tabs.Hitbox:AddToggle("Fullbody", {
    Title = "HS Total (Fullbody)",
    Default = false,
    Callback = function(v)
        Settings.Fullbody = v
        if v then enableFullbody() else disableFullbody() end
    end
})

Tabs.Movement:AddToggle("Spinbot", {
    Title = "Spinbot",
    Default = false,
    Callback = function(v) Settings.Spinbot = v end
})
Tabs.Movement:AddToggle("EnableSpeed", {
    Title = "Ativar Velocidade",
    Default = false,
    Callback = function(v) Settings.EnableSpeed = v; applySpeed() end
})
Tabs.Movement:AddSlider("Speed", {
    Title = "Aumentar Velocidade",
    Min = 16, Max = 300, Default = 16, Rounding = 0,
    Callback = function(v) Settings.Speed = v; applySpeed() end
})
Tabs.Movement:AddToggle("Jump", {
    Title = "Jump Boost",
    Default = false,
    Callback = function(v) Settings.JumpEnabled = v end
})
Tabs.Movement:AddSlider("JumpValue", {
    Title = "Jump Multi",
    Min = 1, Max = 8, Default = 1, Rounding = 0,
    Callback = function(v) Settings.JumpValue = v end
})
Tabs.Movement:AddToggle("Fly", {
    Title = "Fly",
    Default = false,
    Callback = function(v)
    
