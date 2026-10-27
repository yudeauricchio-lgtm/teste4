-- ============================================================
-- BLACKHOLE PROJECT - VERSAO XENO COMPATIVEL (FINAL v3)
-- FOV redondo via ImageLabel + AimColor com fallback de mouse
-- ============================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local VirtualInputManager = game:GetService("VirtualInputManager")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera
local CoreGui = game:GetService("CoreGui")

-- ====================== HELPERS XENO ======================
local function safeWait()
    RunService.Heartbeat:Wait()
end

local heartbeatTasks = {}
local function safeSpawn(fn)
    table.insert(heartbeatTasks, fn)
end

RunService.Heartbeat:Connect(function()
    if #heartbeatTasks == 0 then return end
    local queue = heartbeatTasks
    heartbeatTasks = {}
    for _, fn in ipairs(queue) do
        local ok, err = pcall(fn)
        if not ok then warn("[Blackhole]", err) end
    end
end)

-- ====================== VARIAVEIS ======================
local espEnabled, espNicksEnabled, espDistanceEnabled, espLifeEnabled, espGlowEnabled, espHitboxEnabled = false, false, false, false, false, false
local teamCheck, wallCheck, aimbotEnabled, aimColorEnabled = false, false, false, false
local showFov = true
local showAimColorFov = true
local targetPart, fovRadius, aimColorFovRadius = "Head", 150, 100
local aimColorPrediction, aimColorStrength = 0.08, 0.85
local aimColorDisfarceEnabled, smoothingFactor, minimized, showFps, tpEnabled = false, 0.7, false, false, false
local toggleKey, switchTargetKey = Enum.KeyCode.Insert, Enum.KeyCode.T
local currentTitleColor = Color3.fromRGB(255, 140, 0)
local savedPosition, panelLoaded, anonimatoEnabled = nil, false, false
local clickCount, lastClickTime, nextFireTime = 0, 0, 0
local mouse1Down = false
local ignoreMouse1Release = false
local frameCount, lastTime, currentFps, fpsLabel, espUpdateTimer = 0, tick(), 0, nil, 0

local hitboxExpanderEnabled = false
local hitboxSize = 5
local hitboxTransparency = 0.4
local hitboxOriginalSizes = {}
local espHitboxObjects = {}

local visibilityCache = {}
local VISIBILITY_CACHE_TIME = 0.2
local espBillboardObjects = {}
local glowObjects = {}
local lifeObjects = {}
local selectorButtons = {}
local sliderFills = {}

local ScreenGui, MainFrame, MinimizedFrame, ColorMenu, KeyMenu, AimbotKeyMenu = nil, nil, nil, nil, nil, nil
local ChangeColorButton, ChangeKeyButton, ChangeAimbotKeyButton = nil, nil, nil
local CurrentKeyButton, CurrentAimbotKeyButton, CurrentColorButton = nil, nil, nil
local AnonimatoButton, AnonimatoIndicator = nil, nil
local targetPartsList = {"Head", "HumanoidRootPart"}
local currentTargetIndex = 1
local aimbotTargetFrame = nil
local aimbotTargetLabel = nil

local fovFrame = nil
local aimColorFovFrame = nil
local usingRgb = false
local rgbHue = 0
local rgbConnection = nil

-- ====================== DISFARCE PARTS (R6/R15 DINAMICO) ======================
local function getDisfarceParts()
    local parts = {"Head", "HumanoidRootPart"}
    local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
    if hum and hum.RigType == Enum.HumanoidRigType.R6 then
        table.insert(parts, "Torso")
        table.insert(parts, "Left Arm")
        table.insert(parts, "Right Arm")
        table.insert(parts, "Left Leg")
        table.insert(parts, "Right Leg")
    else
        table.insert(parts, "UpperTorso")
        table.insert(parts, "LowerTorso")
        table.insert(parts, "LeftUpperArm")
        table.insert(parts, "LeftLowerArm")
        table.insert(parts, "LeftHand")
        table.insert(parts, "RightUpperArm")
        table.insert(parts, "RightLowerArm")
        table.insert(parts, "RightHand")
        table.insert(parts, "LeftUpperLeg")
        table.insert(parts, "LeftLowerLeg")
        table.insert(parts, "LeftFoot")
        table.insert(parts, "RightUpperLeg")
        table.insert(parts, "RightLowerLeg")
        table.insert(parts, "RightFoot")
    end
    return parts
end

-- ====================== MOUSE MOVE (FALLBACK ROBUSTO) ======================
local mouseMoveFunc = nil

if mousemoverel then
    mouseMoveFunc = function(dx, dy)
        pcall(function() mousemoverel(dx, dy) end)
    end
elseif mouse_move then
    mouseMoveFunc = function(dx, dy)
        pcall(function() mouse_move(dx, dy) end)
    end
elseif setmousepos then
    mouseMoveFunc = function(dx, dy)
        pcall(function()
            local cur = UserInputService:GetMouseLocation()
            setmousepos(cur.X + dx, cur.Y + dy)
        end)
    end
elseif VirtualInputManager then
    mouseMoveFunc = function(dx, dy)
        pcall(function()
            local cur = UserInputService:GetMouseLocation()
            VirtualInputManager:SendMouseMoveEvent(cur.X + dx, cur.Y + dy)
        end)
    end
end

if not mouseMoveFunc then
    warn("[Blackhole] Nenhuma funcao de mover mouse encontrada - AimColor nao vai puxar.")
end

local mouseClickFunc = nil
if mouse1click then
    mouseClickFunc = function() mouse1click() end
elseif mouse1press and mouse1release then
    mouseClickFunc = function()
        mouse1press()
        RunService.Heartbeat:Wait()
        mouse1release()
    end
end

-- ====================== FUNCOES AUXILIARES ======================
local function getTeamColor(player)
    if player.Team then
        if player.Team.TeamColor then return player.Team.TeamColor.Color end
        if player.Team.BrickColor then return player.Team.BrickColor.Color end
    end
    return Color3.fromRGB(180, 180, 180)
end

local function getLifeFontSizeByDistance(distance)
    local d = math.clamp(distance, 5, 200)
    return math.max(8, math.min(18, 18 - (d - 5) * (10 / 195)))
end

local function getFontSizeByDistance(distance)
    local d = math.clamp(distance, 5, 150)
    return math.max(8, math.min(16, 16 - (d - 5) * (8 / 145)))
end

local function isPlayerAlive(player)
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    return hum and hum.Health > 0 or false
end

local function passesFilters(player)
    return true
end

local function toggleMinimize()
    if not MainFrame or not MinimizedFrame then return end
    minimized = not minimized
    if minimized then
        savedPosition = MainFrame.Position
        MainFrame.Visible = false
        MinimizedFrame.Visible = not anonimatoEnabled
        MinimizedFrame.Position = savedPosition
        if ColorMenu then ColorMenu.Visible = false end
        if KeyMenu then KeyMenu.Visible = false end
        if AimbotKeyMenu then AimbotKeyMenu.Visible = false end
    else
        MainFrame.Visible = true
        MinimizedFrame.Visible = false
        if savedPosition then MainFrame.Position = savedPosition end
    end
end

local function isMouseOverPanel()
    if not MainFrame or not MainFrame.Visible then return false end
    local mp = UserInputService:GetMouseLocation()
    local ap = MainFrame.AbsolutePosition
    local as = MainFrame.AbsoluteSize
    return mp.X >= ap.X and mp.X <= ap.X + as.X and mp.Y >= ap.Y and mp.Y <= ap.Y + as.Y
end

-- ====================== CORES ======================
local COLORS = {
    PRIMARY = Color3.fromRGB(0, 0, 0),
    SECONDARY = Color3.fromRGB(35, 35, 35),
    ENABLED = Color3.fromRGB(0, 200, 0),
    DISABLED = Color3.fromRGB(200, 0, 0),
    TEXT = Color3.fromRGB(255, 255, 255),
    HEART = Color3.fromRGB(255, 80, 80)
}

local TITLE_COLORS = {
    {Nome = "Vermelho", Cor = Color3.fromRGB(220, 20, 60)},
    {Nome = "Azul", Cor = Color3.fromRGB(0, 100, 200)},
    {Nome = "Preto", Cor = Color3.fromRGB(0, 0, 0)},
    {Nome = "Roxo", Cor = Color3.fromRGB(128, 0, 128)},
    {Nome = "Verde", Cor = Color3.fromRGB(34, 139, 34)},
    {Nome = "Amarelo", Cor = Color3.fromRGB(218, 165, 32)},
    {Nome = "Laranja", Cor = Color3.fromRGB(255, 140, 0)},
    {Nome = "Rosa", Cor = Color3.fromRGB(255, 105, 180)},
    {Nome = "Ciano", Cor = Color3.fromRGB(0, 206, 209)},
    {Nome = "Branco", Cor = Color3.fromRGB(255, 255, 255)},
    {Nome = "Cinza", Cor = Color3.fromRGB(128, 128, 128)},
    {Nome = "RGB", Cor = Color3.fromRGB(255, 0, 0)}
}

-- ====================== GUI ======================
ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "BlackholeProject"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.IgnoreGuiInset = true
ScreenGui.DisplayOrder = 999999
local okParent = pcall(function() ScreenGui.Parent = CoreGui end)
if not okParent or not ScreenGui.Parent then
    pcall(function() ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui") end)
end

MainFrame = Instance.new("Frame", ScreenGui)
MainFrame.Size = UDim2.new(0, 560, 0, 500)
MainFrame.Position = UDim2.new(0.5, -280, 0.5, -250)
MainFrame.BackgroundColor3 = COLORS.PRIMARY
MainFrame.BackgroundTransparency = 0.6
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Visible = true

local MainFrameCorner = Instance.new("UICorner", MainFrame)
MainFrameCorner.CornerRadius = UDim.new(0, 12)

local TitleBar = Instance.new("Frame", MainFrame)
TitleBar.Size = UDim2.new(1, 0, 0, 45)
TitleBar.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
TitleBar.BackgroundTransparency = 0.5
TitleBar.BorderSizePixel = 0
TitleBar.Active = true

local TitleBarCorner = Instance.new("UICorner", TitleBar)
TitleBarCorner.CornerRadius = UDim.new(0, 12)

local Title = Instance.new("TextLabel", TitleBar)
Title.Text = "Blackhole Project"
Title.Size = UDim2.new(0.5, 0, 1, 0)
Title.Position = UDim2.new(0, 15, 0, 0)
Title.BackgroundTransparency = 1
Title.TextColor3 = currentTitleColor
Title.Font = Enum.Font.GothamBlack
Title.TextSize = 16
Title.TextXAlignment = Enum.TextXAlignment.Left

local CreditText = Instance.new("TextLabel", TitleBar)
CreditText.Text = "by: machadinho900, zks_frozen"
CreditText.Size = UDim2.new(0.25, 0, 1, 0)
CreditText.Position = UDim2.new(0.35, 0, 0, 0)
CreditText.BackgroundTransparency = 1
CreditText.TextColor3 = currentTitleColor
CreditText.Font = Enum.Font.GothamBlack
CreditText.TextSize = 12
CreditText.TextXAlignment = Enum.TextXAlignment.Left

local MinimizeButton = Instance.new("TextButton", TitleBar)
MinimizeButton.Size = UDim2.new(0, 30, 0, 30)
MinimizeButton.Position = UDim2.new(1, -40, 0.5, -15)
MinimizeButton.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
MinimizeButton.BorderSizePixel = 0
MinimizeButton.Text = ""
local MinimizeCorner = Instance.new("UICorner", MinimizeButton)
MinimizeCorner.CornerRadius = UDim.new(0, 6)

local MinimizeIcon = Instance.new("TextLabel", MinimizeButton)
MinimizeIcon.Text = "+"
MinimizeIcon.Size = UDim2.new(1, 0, 1, 0)
MinimizeIcon.BackgroundTransparency = 1
MinimizeIcon.TextColor3 = currentTitleColor
MinimizeIcon.Font = Enum.Font.GothamBlack
MinimizeIcon.TextSize = 24
MinimizeIcon.TextScaled = true

AnonimatoButton = Instance.new("TextButton", TitleBar)
AnonimatoButton.Size = UDim2.new(0, 100, 0, 30)
AnonimatoButton.Position = UDim2.new(1, -140, 0.5, -15)
AnonimatoButton.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
AnonimatoButton.BackgroundTransparency = 0.5
AnonimatoButton.BorderSizePixel = 0
AnonimatoButton.Text = "Anonimato"
AnonimatoButton.TextColor3 = currentTitleColor
AnonimatoButton.Font = Enum.Font.GothamBlack
AnonimatoButton.TextSize = 12
AnonimatoButton.AutoButtonColor = false
local AnonimatoCorner = Instance.new("UICorner", AnonimatoButton)
AnonimatoCorner.CornerRadius = UDim.new(0, 6)

AnonimatoIndicator = Instance.new("Frame", AnonimatoButton)
AnonimatoIndicator.Size = UDim2.new(0, 14, 0, 14)
AnonimatoIndicator.Position = UDim2.new(1, -16, 0.5, -7)
AnonimatoIndicator.BorderSizePixel = 0
AnonimatoIndicator.BackgroundColor3 = COLORS.DISABLED
local AnonimatoIndicatorCorner = Instance.new("UICorner", AnonimatoIndicator)
AnonimatoIndicatorCorner.CornerRadius = UDim.new(0, 7)

local function updateAnonimatoVisual()
    AnonimatoIndicator.BackgroundColor3 = anonimatoEnabled and COLORS.ENABLED or COLORS.DISABLED
    MinimizeButton.Visible = not anonimatoEnabled
end

AnonimatoButton.MouseButton1Click:Connect(function()
    anonimatoEnabled = not anonimatoEnabled
    clickCount = 0
    updateAnonimatoVisual()
end)

local draggingPanel = false
local dragStartPos, panelStartPos

TitleBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        draggingPanel = true
        dragStartPos = input.Position
        panelStartPos = MainFrame.Position
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if draggingPanel and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStartPos
        MainFrame.Position = UDim2.new(panelStartPos.X.Scale, panelStartPos.X.Offset + delta.X,
                                        panelStartPos.Y.Scale, panelStartPos.Y.Offset + delta.Y)
        savedPosition = MainFrame.Position
        if ColorMenu and ColorMenu.Visible and ChangeColorButton then
            local bp = ChangeColorButton.AbsolutePosition
            ColorMenu.Position = UDim2.new(0, bp.X - 150, 0, bp.Y + 35)
        end
        if KeyMenu and KeyMenu.Visible and ChangeKeyButton then
            local bp = ChangeKeyButton.AbsolutePosition
            KeyMenu.Position = UDim2.new(0, bp.X - 150, 0, bp.Y + 35)
        end
        if AimbotKeyMenu and AimbotKeyMenu.Visible and ChangeAimbotKeyButton then
            local bp = ChangeAimbotKeyButton.AbsolutePosition
            AimbotKeyMenu.Position = UDim2.new(0, bp.X - 150, 0, bp.Y + 35)
        end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        draggingPanel = false
    end
end)

MinimizedFrame = Instance.new("Frame", ScreenGui)
MinimizedFrame.Size = UDim2.new(0, 50, 0, 50)
MinimizedFrame.Position = UDim2.new(0.5, -25, 0.5, -25)
MinimizedFrame.BackgroundColor3 = currentTitleColor
MinimizedFrame.BorderSizePixel = 0
MinimizedFrame.Visible = false
MinimizedFrame.Active = true
local MinimizedCorner = Instance.new("UICorner", MinimizedFrame)
MinimizedCorner.CornerRadius = UDim.new(0, 25)

local MinimizedGhost = Instance.new("TextLabel", MinimizedFrame)
MinimizedGhost.Text = "🪐"
MinimizedGhost.Size = UDim2.new(1, 0, 1, 0)
MinimizedGhost.BackgroundTransparency = 1
MinimizedGhost.TextColor3 = currentTitleColor
MinimizedGhost.Font = Enum.Font.GothamBlack
MinimizedGhost.TextSize = 30
MinimizedGhost.TextScaled = true

MinimizeButton.MouseButton1Click:Connect(toggleMinimize)

local draggingMin = false
local dragStartMin, panelStartMin
MinimizedFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        draggingMin = true
        dragStartMin = input.Position
        panelStartMin = MinimizedFrame.Position
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if draggingMin and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStartMin
        MinimizedFrame.Position = UDim2.new(panelStartMin.X.Scale, panelStartMin.X.Offset + delta.X,
                                            panelStartMin.Y.Scale, panelStartMin.Y.Offset + delta.Y)
        savedPosition = MinimizedFrame.Position
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        if draggingMin then
            local delta = input.Position - dragStartMin
            if math.abs(delta.X) < 5 and math.abs(delta.Y) < 5 then toggleMinimize() end
        end
        draggingMin = false
    end
end)

-- ====================== ABAS ======================
local TabContent = Instance.new("Frame", MainFrame)
TabContent.Size = UDim2.new(1, -20, 0, 300)
TabContent.Position = UDim2.new(0, 10, 0, 80)
TabContent.BackgroundTransparency = 1
TabContent.BorderSizePixel = 0

local ESPTabFrame = Instance.new("Frame", TabContent)
ESPTabFrame.Size = UDim2.new(1, 0, 1, 0)
ESPTabFrame.BackgroundTransparency = 1
ESPTabFrame.Visible = true

local ExtraTabFrame = Instance.new("Frame", TabContent)
ExtraTabFrame.Size = UDim2.new(1, 0, 1, 0)
ExtraTabFrame.BackgroundTransparency = 1
ExtraTabFrame.Visible = false

local AimbotTabFrame = Instance.new("Frame", TabContent)
AimbotTabFrame.Size = UDim2.new(1, 0, 1, 0)
AimbotTabFrame.BackgroundTransparency = 1
AimbotTabFrame.Visible = false

local AimColorTabFrame = Instance.new("Frame", TabContent)
AimColorTabFrame.Size = UDim2.new(1, 0, 1, 0)
AimColorTabFrame.BackgroundTransparency = 1
AimColorTabFrame.Visible = false

local HitboxTabFrame = Instance.new("Frame", TabContent)
HitboxTabFrame.Size = UDim2.new(1, 0, 1, 0)
HitboxTabFrame.BackgroundTransparency = 1
HitboxTabFrame.Visible = false

local function addListLayout(parent)
    local l = Instance.new("UIListLayout", parent)
    l.Padding = UDim.new(0, 6)
    l.FillDirection = Enum.FillDirection.Vertical
    l.HorizontalAlignment = Enum.HorizontalAlignment.Center
    l.VerticalAlignment = Enum.VerticalAlignment.Top
    return l
end
addListLayout(ESPTabFrame)
addListLayout(ExtraTabFrame)
addListLayout(HitboxTabFrame)

local AimbotLeftFrame = Instance.new("Frame", AimbotTabFrame)
AimbotLeftFrame.Size = UDim2.new(0.5, -5, 0, 220)
AimbotLeftFrame.BackgroundTransparency = 1
addListLayout(AimbotLeftFrame)

local AimbotRightFrame = Instance.new("Frame", AimbotTabFrame)
AimbotRightFrame.Size = UDim2.new(0.5, -5, 0, 220)
AimbotRightFrame.Position = UDim2.new(0.5, 5, 0, 0)
AimbotRightFrame.BackgroundTransparency = 1
addListLayout(AimbotRightFrame)

local AimColorLeftFrame = Instance.new("Frame", AimColorTabFrame)
AimColorLeftFrame.Size = UDim2.new(0.5, -5, 0, 220)
AimColorLeftFrame.BackgroundTransparency = 1
addListLayout(AimColorLeftFrame)

local AimColorRightFrame = Instance.new("Frame", AimColorTabFrame)
AimColorRightFrame.Size = UDim2.new(0.5, -5, 0, 220)
AimColorRightFrame.Position = UDim2.new(0.5, 5, 0, 0)
AimColorRightFrame.BackgroundTransparency = 1
addListLayout(AimColorRightFrame)

-- ====================== PAINEIS INFERIORES ======================
local KeyPanel = Instance.new("Frame", MainFrame)
KeyPanel.Size = UDim2.new(1, -20, 0, 45)
KeyPanel.Position = UDim2.new(0, 10, 1, -50)
KeyPanel.BackgroundColor3 = COLORS.SECONDARY
KeyPanel.BackgroundTransparency = 0.5
KeyPanel.BorderSizePixel = 0
local KeyPanelCorner = Instance.new("UICorner", KeyPanel)
KeyPanelCorner.CornerRadius = UDim.new(0, 10)

local KeyLabel = Instance.new("TextLabel", KeyPanel)
KeyLabel.Text = "Tecla para minimizar:"
KeyLabel.Size = UDim2.new(0, 150, 1, 0)
KeyLabel.Position = UDim2.new(0, 10, 0, 0)
KeyLabel.BackgroundTransparency = 1
KeyLabel.TextColor3 = COLORS.TEXT
KeyLabel.Font = Enum.Font.GothamBlack
KeyLabel.TextSize = 11
KeyLabel.TextXAlignment = Enum.TextXAlignment.Left

CurrentKeyButton = Instance.new("TextButton", KeyPanel)
CurrentKeyButton.Size = UDim2.new(0, 80, 0, 28)
CurrentKeyButton.Position = UDim2.new(0, 165, 0.5, -14)
CurrentKeyButton.BackgroundColor3 = currentTitleColor
CurrentKeyButton.BorderSizePixel = 0
CurrentKeyButton.Text = "Insert"
CurrentKeyButton.TextColor3 = Color3.new(1, 1, 1)
CurrentKeyButton.Font = Enum.Font.GothamBlack
CurrentKeyButton.TextSize = 11
local CurrentKeyCorner = Instance.new("UICorner", CurrentKeyButton)
CurrentKeyCorner.CornerRadius = UDim.new(0, 6)

ChangeKeyButton = Instance.new("TextButton", KeyPanel)
ChangeKeyButton.Size = UDim2.new(0, 60, 0, 28)
ChangeKeyButton.Position = UDim2.new(0, 255, 0.5, -14)
ChangeKeyButton.BackgroundColor3 = currentTitleColor
ChangeKeyButton.BorderSizePixel = 0
ChangeKeyButton.Text = "Mudar"
ChangeKeyButton.TextColor3 = Color3.new(1, 1, 1)
ChangeKeyButton.Font = Enum.Font.GothamBlack
ChangeKeyButton.TextSize = 11
local ChangeKeyCorner = Instance.new("UICorner", ChangeKeyButton)
ChangeKeyCorner.CornerRadius = UDim.new(0, 6)

local AimbotKeyPanel = Instance.new("Frame", MainFrame)
AimbotKeyPanel.Size = UDim2.new(1, -20, 0, 45)
AimbotKeyPanel.Position = UDim2.new(0, 10, 1, -50)
AimbotKeyPanel.BackgroundColor3 = COLORS.SECONDARY
AimbotKeyPanel.BackgroundTransparency = 0.5
AimbotKeyPanel.BorderSizePixel = 0
local AimbotKeyPanelCorner = Instance.new("UICorner", AimbotKeyPanel)
AimbotKeyPanelCorner.CornerRadius = UDim.new(0, 10)

local AimbotKeyLabel = Instance.new("TextLabel", AimbotKeyPanel)
AimbotKeyLabel.Text = "Tecla trocar alvo:"
AimbotKeyLabel.Size = UDim2.new(0, 150, 1, 0)
AimbotKeyLabel.Position = UDim2.new(0, 10, 0, 0)
AimbotKeyLabel.BackgroundTransparency = 1
AimbotKeyLabel.TextColor3 = COLORS.TEXT
AimbotKeyLabel.Font = Enum.Font.GothamBlack
AimbotKeyLabel.TextSize = 11
AimbotKeyLabel.TextXAlignment = Enum.TextXAlignment.Left

CurrentAimbotKeyButton = Instance.new("TextButton", AimbotKeyPanel)
CurrentAimbotKeyButton.Size = UDim2.new(0, 80, 0, 28)
CurrentAimbotKeyButton.Position = UDim2.new(0, 165, 0.5, -14)
CurrentAimbotKeyButton.BackgroundColor3 = currentTitleColor
CurrentAimbotKeyButton.BorderSizePixel = 0
CurrentAimbotKeyButton.Text = "T"
CurrentAimbotKeyButton.TextColor3 = Color3.new(1, 1, 1)
CurrentAimbotKeyButton.Font = Enum.Font.GothamBlack
CurrentAimbotKeyButton.TextSize = 11
local CurrentAimbotKeyCorner = Instance.new("UICorner", CurrentAimbotKeyButton)
CurrentAimbotKeyCorner.CornerRadius = UDim.new(0, 6)

ChangeAimbotKeyButton = Instance.new("TextButton", AimbotKeyPanel)
ChangeAimbotKeyButton.Size = UDim2.new(0, 60, 0, 28)
ChangeAimbotKeyButton.Position = UDim2.new(0, 255, 0.5, -14)
ChangeAimbotKeyButton.BackgroundColor3 = currentTitleColor
ChangeAimbotKeyButton.BorderSizePixel = 0
ChangeAimbotKeyButton.Text = "Mudar"
ChangeAimbotKeyButton.TextColor3 = Color3.new(1, 1, 1)
ChangeAimbotKeyButton.Font = Enum.Font.GothamBlack
ChangeAimbotKeyButton.TextSize = 11
local ChangeAimbotKeyCorner = Instance.new("UICorner", ChangeAimbotKeyButton)
ChangeAimbotKeyCorner.CornerRadius = UDim.new(0, 6)

local ColorPanel = Instance.new("Frame", MainFrame)
ColorPanel.Size = UDim2.new(1, -20, 0, 45)
ColorPanel.Position = UDim2.new(0, 10, 1, -100)
ColorPanel.BackgroundColor3 = COLORS.SECONDARY
ColorPanel.BackgroundTransparency = 0.5
ColorPanel.BorderSizePixel = 0
local ColorPanelCorner = Instance.new("UICorner", ColorPanel)
ColorPanelCorner.CornerRadius = UDim.new(0, 10)

local ColorLabel = Instance.new("TextLabel", ColorPanel)
ColorLabel.Text = "Cor do Topo:"
ColorLabel.Size = UDim2.new(0, 80, 1, 0)
ColorLabel.Position = UDim2.new(0, 10, 0, 0)
ColorLabel.BackgroundTransparency = 1
ColorLabel.TextColor3 = COLORS.TEXT
ColorLabel.Font = Enum.Font.GothamBlack
ColorLabel.TextSize = 11
ColorLabel.TextXAlignment = Enum.TextXAlignment.Left

CurrentColorButton = Instance.new("TextButton", ColorPanel)
CurrentColorButton.Size = UDim2.new(0, 50, 0, 28)
CurrentColorButton.Position = UDim2.new(0, 95, 0.5, -14)
CurrentColorButton.BackgroundColor3 = currentTitleColor
CurrentColorButton.BorderSizePixel = 0
CurrentColorButton.Text = ""
CurrentColorButton.AutoButtonColor = false
local CurrentColorCorner = Instance.new("UICorner", CurrentColorButton)
CurrentColorCorner.CornerRadius = UDim.new(0, 6)

ChangeColorButton = Instance.new("TextButton", ColorPanel)
ChangeColorButton.Size = UDim2.new(0, 60, 0, 28)
ChangeColorButton.Position = UDim2.new(0, 155, 0.5, -14)
ChangeColorButton.BackgroundColor3 = currentTitleColor
ChangeColorButton.BorderSizePixel = 0
ChangeColorButton.Text = "Mudar"
ChangeColorButton.TextColor3 = Color3.new(1, 1, 1)
ChangeColorButton.Font = Enum.Font.GothamBlack
ChangeColorButton.TextSize = 11
local ChangeColorCorner = Instance.new("UICorner", ChangeColorButton)
ChangeColorCorner.CornerRadius = UDim.new(0, 6)

local function updatePanelsVisibility(tabName)
    if tabName == "Aimbot" or tabName == "AimColor" then
        KeyPanel.Visible = false
        ColorPanel.Visible = false
        AimbotKeyPanel.Visible = true
    else
        KeyPanel.Visible = true
        ColorPanel.Visible = true
        AimbotKeyPanel.Visible = false
    end
end

local function showTab(tabName)
    ESPTabFrame.Visible = (tabName == "ESP")
    ExtraTabFrame.Visible = (tabName == "Extra")
    AimbotTabFrame.Visible = (tabName == "Aimbot")
    AimColorTabFrame.Visible = (tabName == "AimColor")
    HitboxTabFrame.Visible = (tabName == "Hitbox")
    ESPTabButton.BackgroundColor3 = (tabName == "ESP") and currentTitleColor or Color3.fromRGB(20, 20, 20)
    ExtraTabButton.BackgroundColor3 = (tabName == "Extra") and currentTitleColor or Color3.fromRGB(20, 20, 20)
    AimbotTabButton.BackgroundColor3 = (tabName == "Aimbot") and currentTitleColor or Color3.fromRGB(20, 20, 20)
    AimColorTabButton.BackgroundColor3 = (tabName == "AimColor") and currentTitleColor or Color3.fromRGB(20, 20, 20)
    HitboxTabButton.BackgroundColor3 = (tabName == "Hitbox") and currentTitleColor or Color3.fromRGB(20, 20, 20)
    updatePanelsVisibility(tabName)
end

local TabBar = Instance.new("Frame", MainFrame)
TabBar.Size = UDim2.new(1, 0, 0, 30)
TabBar.Position = UDim2.new(0, 0, 0, 45)
TabBar.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
TabBar.BackgroundTransparency = 0.5
TabBar.BorderSizePixel = 0
TabBar.ZIndex = 100

local function createTabButton(parent, text, position, onClick)
    local b = Instance.new("TextButton", parent)
    b.Size = UDim2.new(0, 100, 1, 0)
    b.Position = position
    b.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    b.BackgroundTransparency = 0.3
    b.BorderSizePixel = 0
    b.Text = text
    b.TextColor3 = COLORS.TEXT
    b.Font = Enum.Font.GothamBlack
    b.TextSize = 12
    b.AutoButtonColor = false
    b.ZIndex = 101
    local c = Instance.new("UICorner", b)
    c.CornerRadius = UDim.new(0, 8)
    b.MouseButton1Click:Connect(onClick)
    return b
end

ESPTabButton = createTabButton(TabBar, "ESP", UDim2.new(0, 10, 0, 0), function() showTab("ESP") end)
ExtraTabButton = createTabButton(TabBar, "Extra", UDim2.new(0, 120, 0, 0), function() showTab("Extra") end)
AimColorTabButton = createTabButton(TabBar, "AimColor", UDim2.new(0, 230, 0, 0), function() showTab("AimColor") end)
AimbotTabButton = createTabButton(TabBar, "Aimbot", UDim2.new(0, 340, 0, 0), function() showTab("Aimbot") end)
HitboxTabButton = createTabButton(TabBar, "Hitbox", UDim2.new(0, 450, 0, 0), function() showTab("Hitbox") end)

showTab("ESP")

-- ====================== MENUS DE TECLA ======================
KeyMenu = Instance.new("Frame", ScreenGui)
KeyMenu.Size = UDim2.new(0, 250, 0, 120)
KeyMenu.BackgroundColor3 = COLORS.PRIMARYKeyMenu.BackgroundTransparency = 0.5
KeyMenu.BorderSizePixel = 0
KeyMenu.Visible = false
KeyMenu.ZIndex = 10
local KeyMenuCorner = Instance.new("UICorner", KeyMenu)
KeyMenuCorner.CornerRadius = UDim.new(0, 8)

local KeyMenuTitle = Instance.new("TextLabel", KeyMenu)
KeyMenuTitle.Text = "Pressione uma tecla..."
KeyMenuTitle.Size = UDim2.new(1, 0, 0, 40)
KeyMenuTitle.BackgroundColor3 = currentTitleColor
KeyMenuTitle.TextColor3 = Color3.new(1, 1, 1)
KeyMenuTitle.Font = Enum.Font.GothamBlack
KeyMenuTitle.TextSize = 14
local KeyMenuCorner2 = Instance.new("UICorner", KeyMenuTitle)
KeyMenuCorner2.CornerRadius = UDim.new(0, 8)

AimbotKeyMenu = Instance.new("Frame", ScreenGui)
AimbotKeyMenu.Size = UDim2.new(0, 250, 0, 120)
AimbotKeyMenu.BackgroundColor3 = COLORS.PRIMARY
AimbotKeyMenu.BackgroundTransparency = 0.5
AimbotKeyMenu.BorderSizePixel = 0
AimbotKeyMenu.Visible = false
AimbotKeyMenu.ZIndex = 10
local AimbotKeyMenuCorner = Instance.new("UICorner", AimbotKeyMenu)
AimbotKeyMenuCorner.CornerRadius = UDim.new(0, 8)

local AimbotKeyMenuTitle = Instance.new("TextLabel", AimbotKeyMenu)
AimbotKeyMenuTitle.Text = "Pressione uma tecla..."
AimbotKeyMenuTitle.Size = UDim2.new(1, 0, 0, 40)
AimbotKeyMenuTitle.BackgroundColor3 = currentTitleColor
AimbotKeyMenuTitle.TextColor3 = Color3.new(1, 1, 1)
AimbotKeyMenuTitle.Font = Enum.Font.GothamBlack
AimbotKeyMenuTitle.TextSize = 14
local AimbotKeyMenuCorner2 = Instance.new("UICorner", AimbotKeyMenuTitle)
AimbotKeyMenuCorner2.CornerRadius = UDim.new(0, 8)

local waitingForKey = false
local keyConnection = nil

local function closeKeyMenu()
    KeyMenu.Visible = false
    if keyConnection then keyConnection:Disconnect(); keyConnection = nil end
    waitingForKey = false
end

ChangeKeyButton.MouseButton1Click:Connect(function()
    if KeyMenu.Visible then
        closeKeyMenu()
    else
        KeyMenu.Visible = true
        if keyConnection then keyConnection:Disconnect() end
        waitingForKey = true
        KeyMenuTitle.Text = "Pressione uma tecla..."
        KeyMenuTitle.BackgroundColor3 = currentTitleColor
        keyConnection = UserInputService.InputBegan:Connect(function(input, gp)
            if gp then return end
            if waitingForKey and input.KeyCode ~= Enum.KeyCode.Unknown then
                toggleKey = input.KeyCode
                CurrentKeyButton.Text = tostring(input.KeyCode):gsub("Enum.KeyCode.", "")
                KeyMenuTitle.Text = "Tecla selecionada!"
                waitingForKey = false
                safeSpawn(function() safeWait(); closeKeyMenu() end)
            end
        end)
    end
end)

local waitingForAimbotKey = false
local aimbotKeyConnection = nil

local function closeAimbotKeyMenu()
    AimbotKeyMenu.Visible = false
    if aimbotKeyConnection then aimbotKeyConnection:Disconnect(); aimbotKeyConnection = nil end
    waitingForAimbotKey = false
end

ChangeAimbotKeyButton.MouseButton1Click:Connect(function()
    if AimbotKeyMenu.Visible then
        closeAimbotKeyMenu()
    else
        AimbotKeyMenu.Visible = true
        if aimbotKeyConnection then aimbotKeyConnection:Disconnect() end
        waitingForAimbotKey = true
        AimbotKeyMenuTitle.Text = "Pressione uma tecla..."
        AimbotKeyMenuTitle.BackgroundColor3 = currentTitleColor
        aimbotKeyConnection = UserInputService.InputBegan:Connect(function(input, gp)
            if gp then return end
            if waitingForAimbotKey and input.KeyCode ~= Enum.KeyCode.Unknown then
                switchTargetKey = input.KeyCode
                CurrentAimbotKeyButton.Text = tostring(input.KeyCode):gsub("Enum.KeyCode.", "")
                AimbotKeyMenuTitle.Text = "Tecla selecionada!"
                waitingForAimbotKey = false
                safeSpawn(function() safeWait(); closeAimbotKeyMenu() end)
            end
        end)
    end
end)

-- ====================== RASTREAMENTO DO MOUSE (REFORCADO) ======================
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        mouse1Down = true
    end
    if gameProcessed then return end
    if input.KeyCode == toggleKey then
        if anonimatoEnabled then
            local t = tick()
            if t - lastClickTime > 1 then clickCount = 0 end
            clickCount = clickCount + 1
            lastClickTime = t
            if clickCount >= 3 then toggleMinimize(); clickCount = 0 end
        else
            toggleMinimize()
        end
    end
    if input.KeyCode == switchTargetKey then
        currentTargetIndex = currentTargetIndex % #targetPartsList + 1
        targetPart = targetPartsList[currentTargetIndex]
    end
    if tpEnabled and input.UserInputType == Enum.UserInputType.MouseButton1 then
        local ctrl = UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) or UserInputService:IsKeyDown(Enum.KeyCode.RightControl)
        if ctrl and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
            local m = LocalPlayer:GetMouse()
            local p = m.Hit.p
            LocalPlayer.Character.HumanoidRootPart.CFrame = CFrame.new(p.X, p.Y + 3, p.Z)
        end
    end
end)

UserInputService.InputChanged:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        mouse1Down = true
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton1 and not ignoreMouse1Release then
        mouse1Down = false
    end
end)

-- ====================== FOV FRAMES (CIRCULO REDONDO - IMAGELABEL) ======================
local CIRCLE_IMAGE = "rbxassetid://4965945816"

local function createCircleFrame(name, radius, initialPos)
    local holder = Instance.new("ImageLabel")
    holder.Name = name
    holder.BackgroundTransparency = 1
    holder.Image = CIRCLE_IMAGE
    holder.ImageColor3 = currentTitleColor
    holder.ImageTransparency = 0
    holder.ScaleType = Enum.ScaleType.Fit
    holder.Size = UDim2.new(0, radius * 2, 0, radius * 2)
    holder.AnchorPoint = Vector2.new(0.5, 0.5)
    holder.Position = initialPos
    holder.ZIndex = 5
    holder.Visible = false
    return holder
end

local function setCircleColor(circle, color)
    if not circle then return end
    if circle:IsA("ImageLabel") then
        circle.ImageColor3 = color
    elseif circle:IsA("Frame") then
        circle.BackgroundColor3 = color
    end
end

fovFrame = createCircleFrame("FOV_Aimbot", fovRadius, UDim2.new(0.5, 0, 0.5, 0))
aimColorFovFrame = createCircleFrame("FOV_AimColor", aimColorFovRadius, UDim2.new(0, 0, 0, 0))

local function parentFovFrames()
    if ScreenGui and ScreenGui.Parent then
        if fovFrame and not fovFrame.Parent then
            fovFrame.Parent = ScreenGui
        end
        if aimColorFovFrame and not aimColorFovFrame.Parent then
            aimColorFovFrame.Parent = ScreenGui
        end
    end
end

parentFovFrames()
safeSpawn(function() safeWait(); parentFovFrames() end)
safeSpawn(function() safeWait(); safeWait(); parentFovFrames() end)

aimbotTargetFrame = Instance.new("Frame", ScreenGui)
aimbotTargetFrame.Size = UDim2.new(0, 200, 0, 40)
aimbotTargetFrame.Position = UDim2.new(0.5, 0, 0, 40)
aimbotTargetFrame.AnchorPoint = Vector2.new(0.5, 0.5)
aimbotTargetFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
aimbotTargetFrame.BackgroundTransparency = 0.5
aimbotTargetFrame.BorderSizePixel = 0
aimbotTargetFrame.Visible = false
aimbotTargetFrame.ZIndex = 50
local atc = Instance.new("UICorner", aimbotTargetFrame)
atc.CornerRadius = UDim.new(0, 12)
aimbotTargetLabel = Instance.new("TextLabel", aimbotTargetFrame)
aimbotTargetLabel.Size = UDim2.new(1, 0, 1, 0)
aimbotTargetLabel.BackgroundTransparency = 1
aimbotTargetLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
aimbotTargetLabel.Font = Enum.Font.GothamBlack
aimbotTargetLabel.TextSize = 18
aimbotTargetLabel.Text = ""

local function createFPSDisplay()
    if fpsLabel then pcall(function() fpsLabel:Destroy() end); fpsLabel = nil end
    if not showFps then return end
    local lbl = Instance.new("TextLabel", ScreenGui)
    lbl.Size = UDim2.new(0, 100, 0, 20)
    lbl.Position = UDim2.new(1, -110, 0, 10)
    lbl.BackgroundTransparency = 1
    lbl.TextColor3 = currentTitleColor
    lbl.Font = Enum.Font.GothamBlack
    lbl.TextSize = 16
    lbl.TextXAlignment = Enum.TextXAlignment.Right
    lbl.Text = "FPS: 0"
    lbl.ZIndex = 100
    fpsLabel = lbl
end

-- ====================== COLOR MENU ======================
local colorMenuTitle = nil

local function updateAllDynamicColors(color)
    TitleBar.BackgroundColor3 = color
    MinimizedFrame.BackgroundColor3 = color
    CurrentColorButton.BackgroundColor3 = color
    ChangeColorButton.BackgroundColor3 = color
    CurrentKeyButton.BackgroundColor3 = color
    ChangeKeyButton.BackgroundColor3 = color
    CurrentAimbotKeyButton.BackgroundColor3 = color
    ChangeAimbotKeyButton.BackgroundColor3 = color
    if colorMenuTitle then colorMenuTitle.BackgroundColor3 = color end
    if KeyMenuTitle then KeyMenuTitle.BackgroundColor3 = color end
    if AimbotKeyMenuTitle then AimbotKeyMenuTitle.BackgroundColor3 = color end
    for _, b in pairs(selectorButtons) do pcall(function() b.BackgroundColor3 = color end) end
    for _, s in pairs(sliderFills) do pcall(function() s.BackgroundColor3 = color end) end
    if fovFrame then setCircleColor(fovFrame, color) end
    if aimColorFovFrame then setCircleColor(aimColorFovFrame, color) end
end

local function startRgbMode()
    usingRgb = true
    if rgbConnection then rgbConnection:Disconnect() end
    rgbConnection = RunService.RenderStepped:Connect(function()
        rgbHue = (rgbHue + 0.002) % 1
        local c = Color3.fromHSV(rgbHue, 1, 1)
        updateAllDynamicColors(c)
    end)
end

local function stopRgbMode()
    if rgbConnection then rgbConnection:Disconnect(); rgbConnection = nil end
    usingRgb = false
end

ColorMenu = Instance.new("Frame", ScreenGui)
ColorMenu.Size = UDim2.new(0, 220, 0, 400)
ColorMenu.BackgroundColor3 = COLORS.PRIMARY
ColorMenu.BackgroundTransparency = 0.5
ColorMenu.BorderSizePixel = 0
ColorMenu.Visible = false
ColorMenu.ZIndex = 10
local ColorMenuCorner = Instance.new("UICorner", ColorMenu)
ColorMenuCorner.CornerRadius = UDim.new(0, 8)

colorMenuTitle = Instance.new("TextLabel", ColorMenu)
colorMenuTitle.Text = "Selecione uma cor"
colorMenuTitle.Size = UDim2.new(1, 0, 0, 30)
colorMenuTitle.BackgroundColor3 = currentTitleColor
colorMenuTitle.TextColor3 = Color3.new(1, 1, 1)
colorMenuTitle.Font = Enum.Font.GothamBlack
colorMenuTitle.TextSize = 14
local ColorMenuCorner2 = Instance.new("UICorner", colorMenuTitle)
ColorMenuCorner2.CornerRadius = UDim.new(0, 8)

local ColorList = Instance.new("ScrollingFrame", ColorMenu)
ColorList.Size = UDim2.new(1, -10, 1, -40)
ColorList.Position = UDim2.new(0, 5, 0, 35)
ColorList.BackgroundTransparency = 1
ColorList.BorderSizePixel = 0
ColorList.ScrollBarThickness = 4
local ColorListLayout = Instance.new("UIListLayout", ColorList)
ColorListLayout.Padding = UDim.new(0, 5)

for _, cd in ipairs(TITLE_COLORS) do
    local cb = Instance.new("TextButton", ColorList)
    cb.Size = UDim2.new(1, -10, 0, 35)
    cb.BackgroundColor3 = cd.Cor
    cb.BorderSizePixel = 0
    cb.Text = cd.Nome
    cb.TextColor3 = (cd.Nome == "Preto" or cd.Nome == "Roxo") and Color3.fromRGB(255,255,255) or Color3.fromRGB(0,0,0)
    cb.Font = Enum.Font.GothamBlack
    cb.TextSize = 12
    local c = Instance.new("UICorner", cb)
    c.CornerRadius = UDim.new(0, 4)
    cb.MouseButton1Click:Connect(function()
        if cd.Nome == "RGB" then startRgbMode() else stopRgbMode(); updateAllDynamicColors(cd.Cor) end
        ColorMenu.Visible = false
    end)
end

safeSpawn(function()
    safeWait()
    ColorList.CanvasSize = UDim2.new(0, 0, 0, ColorListLayout.AbsoluteContentSize.Y + 10)
end)

ChangeColorButton.MouseButton1Click:Connect(function()
    ColorMenu.Visible = not ColorMenu.Visible
    if ColorMenu.Visible then
        local bp = ChangeColorButton.AbsolutePosition
        ColorMenu.Position = UDim2.new(0, bp.X - 150, 0, bp.Y - 200)
    end
end)

-- ====================== TOGGLE E SLIDER ======================
local function createToggleButton(parent, name, getState, onToggle)
    local f = Instance.new("Frame", parent)
    f.Size = UDim2.new(1, 0, 0, 32)
    f.BackgroundColor3 = COLORS.SECONDARY
    f.BackgroundTransparency = 0.5
    f.BorderSizePixel = 0
    local c = Instance.new("UICorner", f)
    c.CornerRadius = UDim.new(0, 6)
    local si = Instance.new("Frame", f)
    si.Size = UDim2.new(0, 14, 0, 14)
    si.Position = UDim2.new(0, 8, 0.5, -7)
    si.BorderSizePixel = 0
    local sic = Instance.new("UICorner", si)
    sic.CornerRadius = UDim.new(0, 7)
    local lbl = Instance.new("TextLabel", f)
    lbl.Text = name
    lbl.Size = UDim2.new(1, -40, 1, 0)
    lbl.Position = UDim2.new(0, 30, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    lbl.Font = Enum.Font.GothamBlack
    lbl.TextSize = 13
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    local hb = Instance.new("TextButton", f)
    hb.Size = UDim2.new(1, 0, 1, 0)
    hb.BackgroundTransparency = 1
    hb.Text = ""
    local function upd()
        si.BackgroundColor3 = getState() and COLORS.ENABLED or COLORS.DISABLED
    end
    hb.MouseButton1Click:Connect(function() onToggle(); upd() end)
    upd()
    return f
end

local function createSlider(parent, name, min, max, default, onChanged)
    local sc = Instance.new("Frame", parent)
    sc.Size = UDim2.new(1, 0, 0, 48)
    sc.BackgroundColor3 = COLORS.SECONDARY
    sc.BackgroundTransparency = 0.5
    sc.BorderSizePixel = 0
    local c = Instance.new("UICorner", sc)
    c.CornerRadius = UDim.new(0, 6)
    local lbl = Instance.new("TextLabel", sc)
    lbl.Text = name .. ": " .. default
    lbl.Size = UDim2.new(1, -20, 0, 18)
    lbl.Position = UDim2.new(0, 10, 0, 5)
    lbl.BackgroundTransparency = 1
    lbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    lbl.Font = Enum.Font.GothamBlack
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    local sbg = Instance.new("Frame", sc)
    sbg.Size = UDim2.new(1, -20, 0, 5)
    sbg.Position = UDim2.new(0, 10, 0, 28)
    sbg.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    sbg.BorderSizePixel = 0
    local sbgc = Instance.new("UICorner", sbg)
    sbgc.CornerRadius = UDim.new(0, 3)
    local sf = Instance.new("Frame", sbg)
    sf.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)
    sf.BackgroundColor3 = currentTitleColor
    sf.BorderSizePixel = 0
    table.insert(sliderFills, sf)
    local sfc = Instance.new("UICorner", sf)
    sfc.CornerRadius = UDim.new(0, 3)
    local mouseDownSlider = false
    local inputConn = nil
    local endConn = nil

    local function upd(input)
        local pos = math.clamp((input.Position.X - sbg.AbsolutePosition.X) / sbg.AbsoluteSize.X, 0, 1)
        local v = math.floor(min + (max - min) * pos)
        sf.Size = UDim2.new(pos, 0, 1, 0)
        lbl.Text = name .. ": " .. v
        onChanged(v)
    end

    sbg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            mouseDownSlider = true
            upd(input)
            if inputConn then inputConn:Disconnect() end
            if endConn then endConn:Disconnect() end
            inputConn = UserInputService.InputChanged:Connect(function(i)
                if mouseDownSlider and i.UserInputType == Enum.UserInputType.MouseMovement then upd(i) end
            end)
            endConn = UserInputService.InputEnded:Connect(function(i)
                if i.UserInputType == Enum.UserInputType.MouseButton1 then
                    mouseDownSlider = false
                    if inputConn then inputConn:Disconnect(); inputConn = nil end
                    if endConn then endConn:Disconnect(); endConn = nil end
                end
            end)
        end
    end)
    return sc
end

-- ====================== ESP / GLOW / LIFE / HITBOX ======================
local function cleanupLifeESP(player)
    local d = lifeObjects[player]
    if d then
        if d.billboard then pcall(function() d.billboard:Destroy() end) end
        if d.healthConnection then pcall(function() d.healthConnection:Disconnect() end) end
        if d.deathConnection then pcall(function() d.deathConnection:Disconnect() end) end
        if d.characterConn then pcall(function() d.characterConn:Disconnect() end) end
        lifeObjects[player] = nil
    end
end

local function updateLifeVisual(player)
    local d = lifeObjects[player]
    if not d or not d.billboard or not d.humanoid or not d.rootPart then return end
    if not d.humanoid.Parent or not d.rootPart.Parent then return end
    local pct = math.max(0, math.min(1, d.humanoid.Health / d.humanoid.MaxHealth))
    local dist = (Camera.CFrame.Position - d.rootPart.Position).Magnitude
    local fs = getLifeFontSizeByDistance(dist)
    local sc = fs / 18
    d.billboard.Size = UDim2.new(0, 60 * sc, 0, 22 * sc)
    d.healthText.TextSize = fs
    d.healthText.Text = string.format("%.0f", d.humanoid.Health)
    if pct > 0.6 then d.healthFill.BackgroundColor3 = Color3.fromRGB(0,255,0)
    elseif pct > 0.3 then d.healthFill.BackgroundColor3 = Color3.fromRGB(255,200,0)
    else d.healthFill.BackgroundColor3 = Color3.fromRGB(255,0,0) end
    d.healthFill.Size = UDim2.new(pct, 0, 1, 0)
end

local function updateLifeESP(player)
    cleanupLifeESP(player)
    if not espEnabled or not espLifeEnabled or player == LocalPlayer then return end
    local char = player.Character
    if not char then
        local conn = player.CharacterAdded:Connect(function()
            conn:Disconnect()
            safeSpawn(function() safeWait(); updateLifeESP(player) end)
        end)
        lifeObjects[player] = {waitingConn = conn}
        return
    end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart")
    if not hum or not root then return end

    local bb = Instance.new("BillboardGui")
    bb.Size = UDim2.new(0, 60, 0, 22)
    bb.StudsOffset = Vector3.new(0, -3.5, 0)
    bb.AlwaysOnTop = true
    bb.Parent = root

    local mf = Instance.new("Frame", bb)
    mf.Size = UDim2.new(1, 0, 1, 0)
    mf.BackgroundTransparency = 0.85
    mf.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    local mc = Instance.new("UICorner", mf)
    mc.CornerRadius = UDim.new(1, 0)

    local ht = Instance.new("TextLabel", mf)
    ht.Size = UDim2.new(1, 0, 0.8, 0)
    ht.Position = UDim2.new(0, 0, 0.1, 0)
    ht.BackgroundTransparency = 1
    ht.TextColor3 = Color3.fromRGB(0, 255, 0)
    ht.TextSize = 16
    ht.Font = Enum.Font.GothamBlack
    ht.TextXAlignment = Enum.TextXAlignment.Center

    local hb = Instance.new("Frame", mf)
    hb.Size = UDim2.new(1, -8, 0, 2)
    hb.Position = UDim2.new(0, 4, 1, -4)
    hb.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    hb.BorderSizePixel = 0
    local hbc = Instance.new("UICorner", hb)
    hbc.CornerRadius = UDim.new(1, 0)

    local hf = Instance.new("Frame", hb)
    hf.Size = UDim2.new(1, 0, 1, 0)
    hf.BackgroundColor3 = COLORS.HEART
    hf.BorderSizePixel = 0
    local hfc = Instance.new("UICorner", hf)
    hfc.CornerRadius = UDim.new(1, 0)

    local hconn = hum.HealthChanged:Connect(function() updateLifeVisual(player) end)
    local dconn = hum.Died:Connect(function()
        cleanupLifeESP(player)
        local rc = player.CharacterAdded:Connect(function()
            rc:Disconnect()
            safeSpawn(function() safeWait(); updateLifeESP(player) end)
        end)
    end)
    local cconn = player.CharacterAdded:Connect(function()
        safeSpawn(function() safeWait(); updateLifeESP(player) end)
    end)

    lifeObjects[player] = {
        billboard = bb, healthText = ht, healthFill = hf,
        humanoid = hum, rootPart = root,
        healthConnection = hconn, deathConnection = dconn, characterConn = cconn
    }
    local show = passesFilters(player) and (not teamCheck or player.Team ~= LocalPlayer.Team)
    bb.Enabled = show
    updateLifeVisual(player)
end

local function updateAllLifeESP()
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer then updateLifeESP(p) end
    end
end

local function clearAllLifeESP()
    for p in pairs(lifeObjects) do cleanupLifeESP(p) end
    lifeObjects = {}
end

local function updateGlowForPlayer(player)
    if not espGlowEnabled or not espEnabled then
        if glowObjects[player] then
            if glowObjects[player].highlight then pcall(function() glowObjects[player].highlight:Destroy() end) end
            glowObjects[player] = nil
        end
        return
    end
    if player == LocalPlayer then return end
    local char = player.Character
    if not char then
        if glowObjects[player] then
            if glowObjects[player].highlight then pcall(function() glowObjects[player].highlight:Destroy() end) end
            glowObjects[player] = nil
        end
        return
    end
    if glowObjects[player] and glowObjects[player].character ~= char then
        if glowObjects[player].highlight then pcall(function() glowObjects[player].highlight:Destroy() end) end
        glowObjects[player] = nil
    end
    if not glowObjects[player] then
        local tc = getTeamColor(player)
        local hl = Instance.new("Highlight")
        hl.FillColor = tc
        hl.OutlineColor = tc
        hl.FillTransparency = 0.5
        hl.OutlineTransparency = 0.3
        hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
        hl.Adornee = char
        hl.Parent = char
        glowObjects[player] = {highlight = hl, character = char}
    end
    local tc = getTeamColor(player)
    if glowObjects[player].highlight.FillColor ~= tc then
        glowObjects[player].highlight.FillColor = tc
        glowObjects[player].highlight.OutlineColor = tc
    end
    local show = passesFilters(player) and (not teamCheck or player.Team ~= LocalPlayer.Team) and isPlayerAlive(player)
    glowObjects[player].highlight.Enabled = show
end

local function clearAllGlow()
    for p, d in pairs(glowObjects) do
        if d.highlight then pcall(function() d.highlight:Destroy() end) end
    end
    glowObjects = {}
end

-- ====================== ESP HITBOX ======================
local function createESPHitbox(player)
    if player == LocalPlayer then return end
    if espHitboxObjects[player] then
        if espHitboxObjects[player].folder then
            pcall(function() espHitboxObjects[player].folder:Destroy() end)
        end
        espHitboxObjects[player] = nil
    end

    local character = player.Character
    if not character then return end

    local folder = Instance.new("Folder")
    folder.Name = "ESPHitbox_" .. player.Name
    folder.Parent = character

    local teamColor = getTeamColor(player)

    for _, part in ipairs(character:GetChildren()) do
        if part:IsA("BasePart") then
            local adornment = Instance.new("BoxHandleAdornment")
            adornment.Name = "Adorn_" .. part.Name
            adornment.Adornee = part
            adornment.AlwaysOnTop = true
            adornment.ZIndex = 10
            adornment.Size = part.Size
            adornment.Transparency = 0.4
            adornment.Color3 = teamColor
            adornment.Parent = folder
        end
    end

    espHitboxObjects[player] = {folder = folder, character = character}
end

local function updateAllESPHitbox()
    if not espHitboxEnabled or not espEnabled then return end
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local isAlly = teamCheck and player.Team == LocalPlayer.Team
            local data = espHitboxObjects[player]
            if not data or data.character ~= player.Character then
                createESPHitbox(player)
                data = espHitboxObjects[player]
            end
            if data and data.folder then
                if isAlly then
                    data.folder.Parent = nil
                else
                    if data.folder.Parent ~= player.Character then
                        data.folder.Parent = player.Character
                    end
                    local teamColor = getTeamColor(player)
                    for _, adorn in ipairs(data.folder:GetChildren()) do
                        if adorn:IsA("BoxHandleAdornment") then
                            adorn.Color3 = teamColor
                        end
                    end
                end
            end
        end
    end
    for player, data in pairs(espHitboxObjects) do
        if not player.Character or not player.Character.Parent then
            if data.folder then pcall(function() data.folder:Destroy() end) end
            espHitboxObjects[player] = nil
        end
    end
end

local function clearAllESPHitbox()
    for player, data in pairs(espHitboxObjects) do
        if data.folder then pcall(function() data.folder:Destroy() end) end
    end
    espHitboxObjects = {}
end

-- ====================== HITBOX EXPANDER ======================
local function restoreHitboxPlayer(player)
    local o = hitboxOriginalSizes[player]
    if not o then return end
    local char = player.Character or o.character
    if char then
        local root = char:FindFirstChild("HumanoidRootPart")
        if root and root:IsA("BasePart") then
            root.Size = o.size
            root.Transparency = o.transparency
            root.CanCollide = o.cancollide
        end
    end
    hitboxOriginalSizes[player] = nil
end

local function applyHitboxToPlayer(player)
    if player == LocalPlayer then return end
    if teamCheck and player.Team == LocalPlayer.Team then
        if hitboxOriginalSizes[player] then
            restoreHitboxPlayer(player)
        end
        return
    end
    local character = player.Character
    if not character then return end
    local root = character:FindFirstChild("HumanoidRootPart")
    if not root or not root:IsA("BasePart") then return end
    if not hitboxOriginalSizes[player] then
        hitboxOriginalSizes[player] = {
            size = root.Size,
            transparency = root.Transparency,
            cancollide = root.CanCollide,
            character = character
        }
    end
    root.CanCollide = false
    root.Size = Vector3.new(hitboxSize, hitboxSize, hitboxSize)
    root.Transparency = hitboxTransparency
end

local function applyHitboxToAll()
    if not hitboxExpanderEnabled then return end
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer then applyHitboxToPlayer(p) end
    end
end

local function clearAllHitbox()
    local players = {}
    for p in pairs(hitboxOriginalSizes) do
        table.insert(players, p)
    end
    for _, p in ipairs(players) do
        restoreHitboxPlayer(p)
    end
    hitboxOriginalSizes = {}
end

local function hookHitboxPlayer(player)
    if player == LocalPlayer then return end
    player.CharacterAdded:Connect(function()
        safeSpawn(function()
            safeWait(); safeWait()
            if hitboxExpanderEnabled then applyHitboxToPlayer(player) end
            if espHitboxEnabled and espEnabled then createESPHitbox(player) end
        end)
    end)
end

-- ====================== ESP BILLBOARD ======================
local function createESP(player)
    if player == LocalPlayer then return end
    if espBillboardObjects[player] then
        if espBillboardObjects[player].character ~= player.Character then
            pcall(function() espBillboardObjects[player].gui:Destroy() end)
            espBillboardObjects[player] = nil
        else
            return
        end
    end
    local character = player.Character
    if not character then return end
    local head = character:FindFirstChild("Head")
    if not head then
        safeSpawn(function()
            local okHead = character:WaitForChild("Head", 5)
            if okHead and espEnabled then
                createESP(player)
            end
        end)
        return
    end

    local bb = Instance.new("BillboardGui")
    bb.Name = "ESP"
    bb.Size = UDim2.new(0, 200, 0, 50)
    bb.StudsOffset = Vector3.new(0, 1.5, 0)
    bb.AlwaysOnTop = true
    bb.MaxDistance = 1000
    bb.Parent = head

    local nl = Instance.new("TextLabel", bb)
    nl.Size = UDim2.new(1, 0, 0.5, 0)
    nl.BackgroundTransparency = 1
    nl.Text = player.Name
    nl.TextColor3 = getTeamColor(player)
    nl.Font = Enum.Font.GothamBlack
    nl.TextSize = 14
    nl.TextStrokeTransparency = 0.5
    nl.Visible = espNicksEnabled

    local dl = Instance.new("TextLabel", bb)
    dl.Size = UDim2.new(1, 0, 0.5, 0)
    dl.Position = UDim2.new(0, 0, 0.5, 0)
    dl.BackgroundTransparency = 1
    dl.TextColor3 = Color3.fromRGB(255,255,255)
    dl.Font = Enum.Font.GothamBlack
    dl.TextSize = 12
    dl.TextStrokeTransparency = 0.5

    espBillboardObjects[player] = {gui = bb, nickLabel = nl, distanceLabel = dl, character = character}
end

local function removeESP(player)
    local d = espBillboardObjects[player]
    if d then pcall(function() d.gui:Destroy() end); espBillboardObjects[player] = nil end
end

local function clearAllESP()
    for p, d in pairs(espBillboardObjects) do pcall(function() d.gui:Destroy() end) end
    espBillboardObjects = {}
end

local function updateESPTexts()
    if not espEnabled then return end
    for p, d in pairs(espBillboardObjects) do
        if not p.Character or d.character ~= p.Character then
            removeESP(p)
            if p.Character then createESP(p) end
        else
            local ok = passesFilters(p) and (not teamCheck or p.Team ~= LocalPlayer.Team) and isPlayerAlive(p)
            d.gui.Enabled = ok
            if ok then
                local head = p.Character and p.Character:FindFirstChild("Head")
                if head then
                    local dist = (Camera.CFrame.Position - head.Position).Magnitude
                    d.distanceLabel.Text = espDistanceEnabled and string.format("%.0fm", dist) or ""
                    d.nickLabel.Visible = espNicksEnabled
                    d.distanceLabel.Visible = espDistanceEnabled
                    local fs = getFontSizeByDistance(dist)
                    d.nickLabel.TextSize = fs
                    d.distanceLabel.TextSize = math.max(8, fs - 2)
                    d.nickLabel.TextColor3 = getTeamColor(p)
                end
            end
        end
    end
end

local function updateAllESP()
    if not espEnabled then return end
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer then
            if not espBillboardObjects[p] then createESP(p) end
            if espGlowEnabled then updateGlowForPlayer(p) end
            if espLifeEnabled then
                if not lifeObjects[p] and p.Character then updateLifeESP(p) end
                if lifeObjects[p] and lifeObjects[p].billboard then
                    lifeObjects[p].billboard.Enabled = passesFilters(p) and (not teamCheck or p.Team ~= LocalPlayer.Team) and isPlayerAlive(p)
                    updateLifeVisual(p)
                end
            end
        end
    end
end

local function recreateAllESP()
    clearAllESP(); clearAllGlow(); clearAllLifeESP(); clearAllESPHitbox()
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer then
            createESP(p)
            updateGlowForPlayer(p)
            updateLifeESP(p)
            if espHitboxEnabled then createESPHitbox(p) end
        end
    end
end

-- ====================== VISIBILIDADE E ALVO ======================
local function isVisible(part)
    if not wallCheck then return true end
    local now = tick()
    local c = visibilityCache[part]
    if c and (now - c.time) < VISIBILITY_CACHE_TIME then return c.result end
    local origin = Camera.CFrame.Position
    local dir = (part.Position - origin)
    local rp = RaycastParams.new()
    rp.FilterType = Enum.RaycastFilterType.Blacklist
    rp.FilterDescendantsInstances = {LocalPlayer.Character}
    local r = Workspace:Raycast(origin, dir.Unit * math.min(dir.Magnitude, 1000), rp)
    local result = false
    if r then result = r.Instance:IsDescendantOf(part.Parent) end
    visibilityCache[part] = {time = now, result = result}
    return result
end

local function getClosestPart(radius)
    local cp, cplr = nil, nil
    local shortest = radius or fovRadius
    local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild(targetPart) then
            if not isPlayerAlive(p) or not passesFilters(p) then continue end
            if teamCheck and p.Team == LocalPlayer.Team then continue end
            local part = p.Character[targetPart]
            local pos, on = Camera:WorldToViewportPoint(part.Position)
            local d = (Vector2.new(pos.X, pos.Y) - center).Magnitude
            if on and d < shortest and isVisible(part) then
                shortest = d; cp = part; cplr = p
            end
        end
    end
    return cp, cplr
end

local function smoothLookAt(target)
    if not target then return end
    local camPos = Camera.CFrame.Position
    local targetPos = target.Position
    local dir = (targetPos - camPos)
    if dir.Magnitude < 0.1 then return end
    local targetCF = CFrame.new(camPos, camPos + dir.Unit)
    local alpha = math.clamp(1 - smoothingFactor, 0.05, 1)
    Camera.CFrame = Camera.CFrame:Lerp(targetCF, alpha)
end

-- ====================== CRIAR ELEMENTOS ======================
createToggleButton(ESPTabFrame, "ESP", function() return espEnabled end, function()
    espEnabled = not espEnabled
    if espEnabled then recreateAllESP() else clearAllESP(); clearAllGlow(); clearAllLifeESP(); clearAllESPHitbox() end
end)
createToggleButton(ESPTabFrame, "ESP Nicks", function() return espNicksEnabled end, function() espNicksEnabled = not espNicksEnabled end)
createToggleButton(ESPTabFrame, "ESP Distance", function() return espDistanceEnabled end, function() espDistanceEnabled = not espDistanceEnabled end)
createToggleButton(ESPTabFrame, "ESP Life", function() return espLifeEnabled end, function()
    espLifeEnabled = not espLifeEnabled
    if espLifeEnabled and espEnabled then updateAllLifeESP() else clearAllLifeESP() end
end)
createToggleButton(ESPTabFrame, "ESP Glow", function() return espGlowEnabled end, function()
    espGlowEnabled = not espGlowEnabled
    if espGlowEnabled and espEnabled then
        for _, p in ipairs(Players:GetPlayers()) do if p ~= LocalPlayer then updateGlowForPlayer(p) end end
    else clearAllGlow() end
end)
createToggleButton(ESPTabFrame, "ESP Hitbox", function() return espHitboxEnabled end, function()
    espHitboxEnabled = not espHitboxEnabled
    if espHitboxEnabled and espEnabled then updateAllESPHitbox() else clearAllESPHitbox() end
end)
createToggleButton(ESPTabFrame, "Verificar Time", function() return teamCheck end, function()
    teamCheck = not teamCheck
    if espEnabled then updateAllESP() end
    if hitboxExpanderEnabled then applyHitboxToAll() end
    if espHitboxEnabled and espEnabled then updateAllESPHitbox() end
end)
createToggleButton(ESPTabFrame, "Verificar Parede", function() return wallCheck end, function() wallCheck = not wallCheck end)

createToggleButton(ExtraTabFrame, "Mostrar FPS", function() return showFps end, function()
    showFps = not showFps
    if showFps then createFPSDisplay() else if fpsLabel then pcall(function() fpsLabel:Destroy() end); fpsLabel = nil end end
end)
createToggleButton(ExtraTabFrame, "TP (Ctrl + Click)", function() return tpEnabled end, function() tpEnabled = not tpEnabled end)

createToggleButton(HitboxTabFrame, "Hitbox Expander", function() return hitboxExpanderEnabled end, function()
    hitboxExpanderEnabled = not hitboxExpanderEnabled
    if hitboxExpanderEnabled then applyHitboxToAll() else clearAllHitbox() end
end)
createSlider(HitboxTabFrame, "Tamanho Hitbox", 1, 20, hitboxSize, function(v)
    hitboxSize = v
    if hitboxExpanderEnabled then applyHitboxToAll() end
end)
createSlider(HitboxTabFrame, "Transparencia Hitbox", 0, 100, math.floor(hitboxTransparency * 100), function(v)
    hitboxTransparency = v / 100
    if hitboxExpanderEnabled then applyHitboxToAll() end
end)

createToggleButton(AimbotLeftFrame, "Aimbot", function() return aimbotEnabled end, function() aimbotEnabled = not aimbotEnabled end)
createToggleButton(AimbotLeftFrame, "Exibir FOV", function() return showFov end, function() showFov = not showFov end)
createSlider(AimbotRightFrame, "FOV Aimbot", 50, 500, fovRadius, function(v) fovRadius = v end)
createSlider(AimbotRightFrame, "Suavizacao Aimbot", 0, 10, smoothingFactor * 10, function(v) smoothingFactor = v / 10 end)

createToggleButton(AimColorLeftFrame, "AimColor", function() return aimColorEnabled end, function() aimColorEnabled = not aimColorEnabled end)
createToggleButton(AimColorLeftFrame, "Exibir FOV AimColor", function() return showAimColorFov end, function() showAimColorFov = not showAimColorFov end)
createToggleButton(AimColorLeftFrame, "AimColor Disfarce", function() return aimColorDisfarceEnabled end, function() aimColorDisfarceEnabled = not aimColorDisfarceEnabled end)
createSlider(AimColorRightFrame, "FOV AimColor", 20, 500, aimColorFovRadius, function(v) aimColorFovRadius = v end)
createSlider(AimColorRightFrame, "Forca AimColor", 5, 100, aimColorStrength * 100, function(v) aimColorStrength = v / 100 end)
createSlider(AimColorRightFrame, "Predicao AimColor", 0, 500, aimColorPrediction * 1000, function(v) aimColorPrediction = v / 1000 end)

-- ====================== CONEXOES DE JOGADORES ======================
for _, p in ipairs(Players:GetPlayers()) do
    if p ~= LocalPlayer then
        createESP(p); updateGlowForPlayer(p); updateLifeESP(p); hookHitboxPlayer(p)
    end
end

Players.PlayerAdded:Connect(function(p)
    safeSpawn(function()
        safeWait()
        hookHitboxPlayer(p)
    end)
    p.CharacterAdded:Connect(function(char)
        safeSpawn(function()
            safeWait()
            if espEnabled then
                removeESP(p)
                createESP(p)
                if espLifeEnabled then updateLifeESP(p) end
                if espGlowEnabled then updateGlowForPlayer(p) end
                if espHitboxEnabled then createESPHitbox(p) end
            end
            if hitboxExpanderEnabled then applyHitboxToPlayer(p) end
        end)
    end)
end)

Players.PlayerRemoving:Connect(function(p)
    removeESP(p)
    if glowObjects[p] then
        if glowObjects[p].highlight then pcall(function() glowObjects[p].highlight:Destroy() end) end
        glowObjects[p] = nil
    end
    cleanupLifeESP(p)
    if espHitboxObjects[p] then
        if espHitboxObjects[p].folder then pcall(function() espHitboxObjects[p].folder:Destroy() end) end
        espHitboxObjects[p] = nil
    end
    hitboxOriginalSizes[p] = nil
end)

-- ====================== LOOP PRINCIPAL ======================
RunService.RenderStepped:Connect(function()
    if not panelLoaded then return end
    local mousePos = UserInputService:GetMouseLocation()
    local overPanel = isMouseOverPanel()

    -- ===== FOV AIMBOT (fixo no centro da tela) =====
    if fovFrame then
        if not fovFrame.Parent and ScreenGui and ScreenGui.Parent then
            fovFrame.Parent = ScreenGui
        end
        if fovFrame.Parent then
            fovFrame.Visible = showFov
            if showFov then
                local r = fovRadius
                fovFrame.Size = UDim2.new(0, r * 2, 0, r * 2)
                fovFrame.AnchorPoint = Vector2.new(0.5, 0.5)
                fovFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
                setCircleColor(fovFrame, usingRgb and Color3.fromHSV(rgbHue, 1, 1) or currentTitleColor)
            end
        end
    end

    -- ===== FOV AIMCOLOR (em volta do cursor) =====
    if aimColorFovFrame then
        if not aimColorFovFrame.Parent and ScreenGui and ScreenGui.Parent then
            aimColorFovFrame.Parent = ScreenGui
        end
        if aimColorFovFrame.Parent then
            aimColorFovFrame.Visible = showAimColorFov
            if showAimColorFov then
                local r = aimColorFovRadius
                aimColorFovFrame.Size = UDim2.new(0, r * 2, 0, r * 2)
                aimColorFovFrame.AnchorPoint = Vector2.new(0.5, 0.5)
                aimColorFovFrame.Position = UDim2.new(0, mousePos.X, 0, mousePos.Y)
                setCircleColor(aimColorFovFrame, usingRgb and Color3.fromHSV(rgbHue, 1, 1) or currentTitleColor)
            end
        end
    end

    -- FPS
    if showFps and fpsLabel then
        frameCount = frameCount + 1
        local ct = tick()
        if ct - lastTime >= 1 then
            currentFps = frameCount
            frameCount = 0
            lastTime = ct
            fpsLabel.Text = ("FPS: %d"):format(currentFps)
        end
    end

    -- Update ESP a cada 0.5s
    if tick() - espUpdateTimer >= 0.5 then
        espUpdateTimer = tick()
        updateAllESP()
        updateESPTexts()
        if espHitboxEnabled and espEnabled then updateAllESPHitbox() end
        if hitboxExpanderEnabled then applyHitboxToAll() end
    end

    -- Aimbot (botao direito)
    local rmbDown = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2)
    if aimbotEnabled and rmbDown and not overPanel then
        local target, tp = getClosestPart(fovRadius)
        if target then
            smoothLookAt(target)
            aimbotTargetFrame.Visible = true
            if tp then
                aimbotTargetLabel.Text = "Alvo: " .. tp.Name
            end
        else
            aimbotTargetFrame.Visible = false
        end
    else
        aimbotTargetFrame.Visible = false
    end

    -- ===== AIMCOLOR (segurar LMB puxa o mouse na direcao do player) =====
    local lmbDown = mouse1Down or UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)
    if aimColorEnabled and lmbDown and not overPanel then
        local parts = aimColorDisfarceEnabled and getDisfarceParts() or {targetPart}
        local mp = UserInputService:GetMouseLocation()
        local center = Vector2.new(mp.X, mp.Y)
        local closest, shortest = nil, aimColorFovRadius
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer and p.Character then
                if not teamCheck or p.Team ~= LocalPlayer.Team then
                    for _, pn in ipairs(parts) do
                        local part = p.Character:FindFirstChild(pn)
                        if part and part:IsA("BasePart") then
                            if not wallCheck or isVisible(part) then
                                local pos, on = Camera:WorldToViewportPoint(part.Position)
                                if on then
                                    local d = (Vector2.new(pos.X, pos.Y) - center).Magnitude
                                    if d < shortest then shortest = d; closest = part end
                                end
                            end
                        end
                    end
                end
            end
        end

        if closest and mouseMoveFunc then
            local strength, pred
            if aimColorDisfarceEnabled then
                strength = 0.35
                pred = 0.005
                local head = closest.Parent and closest.Parent:FindFirstChild("Head")
                if head and head:IsA("BasePart") then
                    closest = head
                end
            else
                strength = aimColorStrength
                pred = aimColorPrediction
            end

            local v = closest.AssemblyLinearVelocity
            local pp = closest.Position + v * pred
            local sp, on = Camera:WorldToViewportPoint(pp)
            if on then
                local dx = sp.X - mp.X
                local dy = sp.Y - mp.Y
                local dist = math.sqrt(dx*dx + dy*dy)
                if dist > 1 then
                    mouseMoveFunc(dx * strength, dy * strength)
                end
            end

            if tick() >= nextFireTime then
                safeSpawn(function()
                    local tool = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Tool")
                    if tool then pcall(function() tool:Activate() end)
                    elseif mouseClickFunc then mouseClickFunc() end
                end)
                nextFireTime = tick() + 0.1
            end
        end
    end
end)

updateAnonimatoVisual()
panelLoaded = true

print("[Blackhole Xeno] Painel carregado (FOV redondo + AimColor com fallback de mouse).")
