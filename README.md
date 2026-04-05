--[[
    SCRIPT COMPLETO - LocalScript em StarterPlayerScripts ou StarterGui
    Sistema profissional de AIMASSIST + ESP com cores diferenciadas (aliado/inimigo)
    Linhas dinâmicas, barras de vida, verificação de paredes e suporte pós-morte
--]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Teams = game:GetService("Teams")

local player = Players.LocalPlayer
local mouse = player:GetMouse()
local camera = workspace.CurrentCamera

-- ======================== CONFIGURAÇÕES ========================
local CONFIG = {
    VALID_KEY = "free2026",
    FOV_RADIUS = 120,
    AIM_SMOOTHNESS = 0.25,
    ESP_LINE_WIDTH = 2,
    RAINBOW_SPEED = 2,
    HEALTH_BAR_HEIGHT = 4
}

-- ======================== ESTADOS DO SISTEMA ========================
local System = {
    keyValidated = false,
    aimAssistActive = false,
    espActive = false,
    currentTarget = nil,
    uiReferences = {},
    dragging = false,
    isMinimized = false,
    espData = {}, -- player -> {highlight, healthBar, lineFrame, lineBillboard, rainbowConnection}
    rainbowHue = 0,
    lastPlayerList = {}
}

-- ======================== FUNÇÕES AUXILIARES ========================
local function CreateTween(object, properties, duration, style, direction)
    style = style or Enum.EasingStyle.Quad
    direction = direction or Enum.EasingDirection.Out
    local tween = TweenService:Create(object, TweenInfo.new(duration, style, direction), properties)
    return tween
end

local function AnimateButton(button)
    button.MouseButton1Down:Connect(function()
        CreateTween(button, {BackgroundTransparency = 0.2}, 0.05):Play()
    end)
    button.MouseButton1Up:Connect(function()
        CreateTween(button, {BackgroundTransparency = 0}, 0.05):Play()
    end)
end

local function MakeDraggable(frame, dragHandle)
    local dragObj = dragHandle or frame
    local dragging = false
    local dragStart, startPos = nil, nil
    dragObj.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement) then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

local function ShowMessage(parent, text, isError)
    local msg = Instance.new("TextLabel")
    msg.Text = text
    msg.TextColor3 = isError and Color3.fromRGB(255, 80, 80) or Color3.fromRGB(80, 255, 120)
    msg.TextSize = 14
    msg.Font = Enum.Font.GothamSemibold
    msg.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
    msg.BackgroundTransparency = 0.05
    msg.BorderSizePixel = 0
    msg.Size = UDim2.fromOffset(0, 30)
    msg.Position = UDim2.fromScale(0.5, 0.8)
    msg.AnchorPoint = Vector2.new(0.5, 0.5)
    msg.Parent = parent
    local bounds = game:GetService("TextService"):GetTextSize(text, 14, Enum.Font.GothamSemibold, Vector2.new(500, 30))
    msg.Size = UDim2.fromOffset(bounds.X + 40, 30)
    msg.BackgroundTransparency = 1
    msg.TextTransparency = 1
    CreateTween(msg, {BackgroundTransparency = 0.05}, 0.2):Play()
    CreateTween(msg, {TextTransparency = 0}, 0.2):Play()
    task.wait(2)
    CreateTween(msg, {BackgroundTransparency = 1}, 0.2):Play()
    CreateTween(msg, {TextTransparency = 1}, 0.2):Play()
    task.wait(0.3)
    msg:Destroy()
end

-- ======================== VERIFICAÇÕES ========================
local function IsEnemy(targetPlayer)
    if #Teams:GetTeams() == 0 then return true end
    local pTeam = player.Team
    local tTeam = targetPlayer.Team
    if pTeam and tTeam then return pTeam ~= tTeam end
    return true
end

local function IsPlayerVisible(targetPlayer)
    if not targetPlayer or not targetPlayer.Character then return false end
    local head = targetPlayer.Character:FindFirstChild("Head")
    if not head then return false end
    local origin = camera.CFrame.Position
    local direction = (head.Position - origin).Unit
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Blacklist
    params.FilterDescendantsInstances = {player.Character, targetPlayer.Character}
    local result = workspace:Raycast(origin, direction * (origin - head.Position).Magnitude, params)
    return result == nil
end

local function GetRainbowColor()
    System.rainbowHue = (System.rainbowHue + CONFIG.RAINBOW_SPEED * RunService.RenderStepped:Wait()) % 360
    return Color3.fromHSV(System.rainbowHue / 360, 1, 1)
end

-- ======================== SISTEMA DE AIM (LOCK COM VERIFICAÇÃO) ========================
local function GetClosestTargetInFOV()
    local center = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
    local best = nil
    local bestDist = CONFIG.FOV_RADIUS
    for _, other in ipairs(Players:GetPlayers()) do
        if other ~= player and IsEnemy(other) and other.Character and other.Character:FindFirstChild("Head") then
            if IsPlayerVisible(other) then
                local headPos, onScreen = camera:WorldToViewportPoint(other.Character.Head.Position)
                if onScreen then
                    local screenPos = Vector2.new(headPos.X, headPos.Y)
                    local dist = (screenPos - center).Magnitude
                    if dist <= CONFIG.FOV_RADIUS and dist < bestDist then
                        bestDist = dist
                        best = other
                    end
                end
            end
        end
    end
    return best
end

local function LockToTarget(target)
    if not target or not target.Character or not target.Character:FindFirstChild("Head") then return false end
    if not IsPlayerVisible(target) then return false end
    local headPos = target.Character.Head.Position
    local currentCFrame = camera.CFrame
    local targetCFrame = CFrame.new(currentCFrame.Position, headPos)
    camera.CFrame = currentCFrame:Lerp(targetCFrame, CONFIG.AIM_SMOOTHNESS)
    return true
end

-- ======================== ESP: HIGHLIGHT, HEALTH BAR, RAINBOW LINE ========================
local function CreateHealthBar(character)
    local barBg = Instance.new("BillboardGui")
    barBg.Name = "HealthBarBG"
    barBg.Adornee = character
    barBg.Size = UDim2.fromOffset(100, 8)
    barBg.StudsOffset = Vector3.new(0, 2.5, 0)
    barBg.AlwaysOnTop = true
    barBg.Parent = character
    
    local bgFrame = Instance.new("Frame")
    bgFrame.Size = UDim2.fromScale(1, 1)
    bgFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    bgFrame.BackgroundTransparency = 0.5
    bgFrame.BorderSizePixel = 0
    bgFrame.Parent = barBg
    
    local healthFill = Instance.new("Frame")
    healthFill.Name = "HealthFill"
    healthFill.Size = UDim2.fromScale(1, 1)
    healthFill.BackgroundColor3 = Color3.fromRGB(80, 255, 80)
    healthFill.BorderSizePixel = 0
    healthFill.Parent = bgFrame
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 2)
    corner.Parent = bgFrame
    
    return barBg, healthFill
end

local function UpdateHealthBar(character, healthFill)
    local humanoid = character:FindFirstChild("Humanoid")
    if humanoid then
        local healthPercent = humanoid.Health / humanoid.MaxHealth
        healthFill.Size = UDim2.fromScale(healthPercent, 1)
        healthFill.BackgroundColor3 = Color3.fromRGB(80 + (1-healthPercent)*175, 255*healthPercent, 80)
    end
end

local function CreateLineToPlayer(targetPlayer)
    -- Criamos um ScreenGui para desenhar a linha 2D
    local lineGui = Instance.new("ScreenGui")
    lineGui.Name = "LineTo_" .. targetPlayer.Name
    lineGui.ResetOnSpawn = false
    lineGui.Parent = player.PlayerGui
    
    local lineFrame = Instance.new("Frame")
    lineFrame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    lineFrame.BorderSizePixel = 0
    lineFrame.Size = UDim2.fromOffset(0, CONFIG.ESP_LINE_WIDTH)
    lineFrame.Parent = lineGui
    
    return lineGui, lineFrame
end

local function UpdateLinePosition(lineGui, lineFrame, targetPlayer)
    if not targetPlayer.Character or not targetPlayer.Character:FindFirstChild("Head") then
        lineGui.Enabled = false
        return
    end
    local head = targetPlayer.Character.Head
    local headPos, onScreen = camera:WorldToViewportPoint(head.Position)
    if not onScreen then
        lineGui.Enabled = false
        return
    end
    local origin = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
    local targetPos = Vector2.new(headPos.X, headPos.Y)
    local delta = targetPos - origin
    local distance = delta.Magnitude
    local angle = math.atan2(delta.Y, delta.X)
    
    lineFrame.Size = UDim2.fromOffset(distance, CONFIG.ESP_LINE_WIDTH)
    lineFrame.Position = UDim2.fromOffset(origin.X, origin.Y)
    lineFrame.Rotation = math.deg(angle)
    lineGui.Enabled = true
    -- Cor arco-íris
    lineFrame.BackgroundColor3 = GetRainbowColor()
end

local function CreateESPForPlayer(targetPlayer)
    if System.espData[targetPlayer] then return end
    if not targetPlayer.Character then return end
    
    local character = targetPlayer.Character
    local isEnemy = IsEnemy(targetPlayer)
    local highlightColor = isEnemy and Color3.fromRGB(255, 80, 80) or Color3.fromRGB(80, 255, 80)
    
    -- Highlight
    local highlight = Instance.new("Highlight")
    highlight.FillTransparency = 0.6
    highlight.OutlineTransparency = 0.2
    highlight.FillColor = highlightColor
    highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
    highlight.Adornee = character
    highlight.Parent = character
    
    -- Barra de vida
    local healthBar, healthFill = CreateHealthBar(character)
    
    -- Linha colorida (apenas para inimigos)
    local lineGui, lineFrame = nil, nil
    if isEnemy then
        lineGui, lineFrame = CreateLineToPlayer(targetPlayer)
    end
    
    System.espData[targetPlayer] = {
        highlight = highlight,
        healthBar = healthBar,
        healthFill = healthFill,
        lineGui = lineGui,
        lineFrame = lineFrame,
        isEnemy = isEnemy
    }
    
    -- Atualiza vida periodicamente
    task.spawn(function()
        while System.espData[targetPlayer] and targetPlayer.Character and targetPlayer.Character.Parent do
            UpdateHealthBar(targetPlayer.Character, healthFill)
            if isEnemy and lineGui and lineFrame then
                UpdateLinePosition(lineGui, lineFrame, targetPlayer)
            end
            task.wait(0.1)
        end
    end)
end

local function RemoveESPForPlayer(targetPlayer)
    local data = System.espData[targetPlayer]
    if data then
        if data.highlight then data.highlight:Destroy() end
        if data.healthBar then data.healthBar:Destroy() end
        if data.lineGui then data.lineGui:Destroy() end
        System.espData[targetPlayer] = nil
    end
end

local function RefreshAllESP()
    if not System.espActive then
        for p in pairs(System.espData) do
            RemoveESPForPlayer(p)
        end
        return
    end
    for _, other in ipairs(Players:GetPlayers()) do
        if other ~= player then
            if other.Character and other.Character.Parent then
                if System.espData[other] then
                    -- Se mudou de time, recria
                    local oldIsEnemy = System.espData[other].isEnemy
                    if oldIsEnemy ~= IsEnemy(other) then
                        RemoveESPForPlayer(other)
                        CreateESPForPlayer(other)
                    end
                else
                    CreateESPForPlayer(other)
                end
            else
                RemoveESPForPlayer(other)
            end
        end
    end
end

-- Monitorar novos jogadores, respawn e troca de time
local function StartMonitoring()
    -- Lista inicial
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= player then
            p.CharacterAdded:Connect(function()
                task.wait(0.2)
                if System.espActive then
                    RemoveESPForPlayer(p)
                    CreateESPForPlayer(p)
                end
            end)
            p:GetPropertyChangedSignal("Team"):Connect(function()
                if System.espActive then
                    RemoveESPForPlayer(p)
                    CreateESPForPlayer(p)
                end
            end)
        end
    end
    
    Players.PlayerAdded:Connect(function(newPlayer)
        if newPlayer ~= player then
            newPlayer.CharacterAdded:Connect(function()
                task.wait(0.2)
                if System.espActive then
                    RemoveESPForPlayer(newPlayer)
                    CreateESPForPlayer(newPlayer)
                end
            end)
            newPlayer:GetPropertyChangedSignal("Team"):Connect(function()
                if System.espActive then
                    RemoveESPForPlayer(newPlayer)
                    CreateESPForPlayer(newPlayer)
                end
            end)
        end
    end)
    
    -- Loop periódico para garantir sincronia
    task.spawn(function()
        while true do
            if System.keyValidated then
                RefreshAllESP()
            end
            task.wait(0.5)
        end
    end)
end

-- ======================== INTERFACE MODERNA (BOTÃO FLUTUANTE E JANELA) ========================
local floatingGui, floatingButton

local function CreateFloatingButton()
    floatingGui = Instance.new("ScreenGui")
    floatingGui.Name = "FloatingUI"
    floatingGui.Parent = player.PlayerGui
    floatingGui.Enabled = false
    
    local btn = Instance.new("Frame")
    btn.Size = UDim2.fromOffset(55, 55)
    btn.Position = UDim2.fromScale(0.92, 0.85)
    btn.AnchorPoint = Vector2.new(0.5, 0.5)
    btn.BackgroundColor3 = Color3.fromRGB(255, 70, 120)
    btn.BorderSizePixel = 0
    btn.Parent = floatingGui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(1, 0)
    corner.Parent = btn
    
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(255, 255, 255)
    stroke.Thickness = 2
    stroke.Transparency = 0.4
    stroke.Parent = btn
    
    local icon = Instance.new("TextLabel")
    icon.Text = "🎯"
    icon.TextSize = 28
    icon.TextColor3 = Color3.fromRGB(255, 255, 255)
    icon.BackgroundTransparency = 1
    icon.Size = UDim2.fromScale(1, 1)
    icon.Parent = btn
    
    MakeDraggable(btn, btn)
    
    local lastClick = 0
    btn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            local now = tick()
            if now - lastClick < 0.3 then
                -- duplo clique
                System.isMinimized = false
                floatingGui.Enabled = false
                if System.uiReferences.mainGui then
                    System.uiReferences.mainGui.Enabled = true
                    local mf = System.uiReferences.mainFrame
                    local sh = System.uiReferences.shadow
                    mf.Visible = true
                    sh.Visible = true
                    CreateTween(mf, {BackgroundTransparency = 0.05}, 0.3):Play()
                    CreateTween(sh, {ImageTransparency = 0.4}, 0.3):Play()
                end
            end
            lastClick = now
        end
    end)
    
    floatingButton = btn
end

local function CreateMainUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "MainUI"
    screenGui.Parent = player.PlayerGui
    
    local shadow = Instance.new("ImageLabel")
    shadow.Image = "rbxassetid://1316045217"
    shadow.ImageColor3 = Color3.fromRGB(0,0,0)
    shadow.ImageTransparency = 0.6
    shadow.ScaleType = Enum.ScaleType.Slice
    shadow.SliceCenter = Rect.new(10,10,10,10)
    shadow.Size = UDim2.fromOffset(420, 540)
    shadow.Position = UDim2.fromScale(0.5, 0.5)
    shadow.AnchorPoint = Vector2.new(0.5,0.5)
    shadow.Parent = screenGui
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.fromOffset(420, 540)
    mainFrame.Position = UDim2.fromScale(0.5,0.5)
    mainFrame.AnchorPoint = Vector2.new(0.5,0.5)
    mainFrame.BackgroundColor3 = Color3.fromRGB(20,20,25)
    mainFrame.BackgroundTransparency = 0.05
    mainFrame.BorderSizePixel = 0
    mainFrame.ClipsDescendants = true
    mainFrame.Parent = screenGui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 14)
    corner.Parent = mainFrame
    
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(255,100,150)
    stroke.Thickness = 1.5
    stroke.Transparency = 0.6
    stroke.Parent = mainFrame
    
    -- Barra título
    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.fromOffset(420, 50)
    titleBar.BackgroundColor3 = Color3.fromRGB(35,35,42)
    titleBar.BackgroundTransparency = 0.1
    titleBar.BorderSizePixel = 0
    titleBar.Parent = mainFrame
    
    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 14)
    titleCorner.Parent = titleBar
    
    local titleText = Instance.new("TextLabel")
    titleText.Text = "⚡ AIMASSIST PREMIUM"
    titleText.TextColor3 = Color3.fromRGB(255,255,255)
    titleText.TextSize = 18
    titleText.Font = Enum.Font.GothamBold
    titleText.BackgroundTransparency = 1
    titleText.Size = UDim2.fromOffset(380, 50)
    titleText.Position = UDim2.fromOffset(10, 0)
    titleText.TextXAlignment = Enum.TextXAlignment.Left
    titleText.Parent = titleBar
    
    local minimizeBtn = Instance.new("TextButton")
    minimizeBtn.Text = "−"
    minimizeBtn.TextColor3 = Color3.fromRGB(255,255,255)
    minimizeBtn.TextSize = 28
    minimizeBtn.Font = Enum.Font.GothamBold
    minimizeBtn.BackgroundTransparency = 1
    minimizeBtn.Size = UDim2.fromOffset(40, 40)
    minimizeBtn.Position = UDim2.fromOffset(370, 5)
    minimizeBtn.Parent = titleBar
    
    -- Botões
    local aimBtn = Instance.new("TextButton")
    aimBtn.Text = "AIMASSIST"
    aimBtn.TextSize = 18
    aimBtn.Font = Enum.Font.GothamBold
    aimBtn.TextColor3 = Color3.fromRGB(255,255,255)
    aimBtn.BackgroundColor3 = Color3.fromRGB(255,70,120)
    aimBtn.Size = UDim2.fromOffset(280, 50)
    aimBtn.Position = UDim2.fromScale(0.5, 0.28)
    aimBtn.AnchorPoint = Vector2.new(0.5,0.5)
    aimBtn.Parent = mainFrame
    local aimCorner = Instance.new("UICorner")
    aimCorner.CornerRadius = UDim.new(0, 25)
    aimCorner.Parent = aimBtn
    
    local espBtn = Instance.new("TextButton")
    espBtn.Text = "ESP (LINHAS + HIGHLIGHT)"
    espBtn.TextSize = 16
    espBtn.Font = Enum.Font.GothamBold
    espBtn.TextColor3 = Color3.fromRGB(255,255,255)
    espBtn.BackgroundColor3 = Color3.fromRGB(100,100,200)
    espBtn.Size = UDim2.fromOffset(280, 50)
    espBtn.Position = UDim2.fromScale(0.5, 0.44)
    espBtn.AnchorPoint = Vector2.new(0.5,0.5)
    espBtn.Parent = mainFrame
    local espCorner = Instance.new("UICorner")
    espCorner.CornerRadius = UDim.new(0, 25)
    espCorner.Parent = espBtn
    
    -- Caixa de texto decorativa
    local configBox = Instance.new("TextBox")
    configBox.PlaceholderText = "⚙️ Configurações futuras..."
    configBox.Text = ""
    configBox.TextColor3 = Color3.fromRGB(200,200,200)
    configBox.TextSize = 14
    configBox.Font = Enum.Font.Gotham
    configBox.BackgroundColor3 = Color3.fromRGB(40,40,48)
    configBox.BackgroundTransparency = 0.2
    configBox.Size = UDim2.fromOffset(280, 40)
    configBox.Position = UDim2.fromScale(0.5, 0.6)
    configBox.AnchorPoint = Vector2.new(0.5,0.5)
    configBox.Parent = mainFrame
    local boxCorner = Instance.new("UICorner")
    boxCorner.CornerRadius = UDim.new(0, 8)
    boxCorner.Parent = configBox
    
    local statusText = Instance.new("TextLabel")
    statusText.Text = "● AIM: OFF  |  ESP: OFF"
    statusText.TextColor3 = Color3.fromRGB(255,70,70)
    statusText.TextSize = 12
    statusText.Font = Enum.Font.GothamSemibold
    statusText.BackgroundTransparency = 1
    statusText.Size = UDim2.fromOffset(300, 30)
    statusText.Position = UDim2.fromScale(0.5, 0.75)
    statusText.AnchorPoint = Vector2.new(0.5,0.5)
    statusText.Parent = mainFrame
    
    local footer = Instance.new("TextLabel")
    footer.Text = "© 2024 | Sistema Profissional"
    footer.TextColor3 = Color3.fromRGB(150,150,150)
    footer.TextSize = 10
    footer.Font = Enum.Font.Gotham
    footer.BackgroundTransparency = 1
    footer.Size = UDim2.fromOffset(420, 20)
    footer.Position = UDim2.fromOffset(0, 520)
    footer.Parent = mainFrame
    
    -- FOV circle
    local fovCircle = Instance.new("ImageLabel")
    fovCircle.Image = "rbxassetid://10992217417"
    fovCircle.ImageColor3 = Color3.fromRGB(255,100,150)
    fovCircle.ImageTransparency = 0.6
    fovCircle.Size = UDim2.fromOffset(CONFIG.FOV_RADIUS*2, CONFIG.FOV_RADIUS*2)
    fovCircle.Position = UDim2.fromScale(0.5,0.5)
    fovCircle.AnchorPoint = Vector2.new(0.5,0.5)
    fovCircle.Visible = false
    fovCircle.Parent = screenGui
    
    System.uiReferences = {
        mainGui = screenGui,
        mainFrame = mainFrame,
        shadow = shadow,
        aimBtn = aimBtn,
        espBtn = espBtn,
        statusText = statusText,
        fovCircle = fovCircle
    }
    
    MakeDraggable(mainFrame, titleBar)
    AnimateButton(aimBtn)
    AnimateButton(espBtn)
    AnimateButton(minimizeBtn)
    
    minimizeBtn.MouseButton1Click:Connect(function()
        System.isMinimized = true
        CreateTween(mainFrame, {BackgroundTransparency = 1}, 0.2):Play()
        CreateTween(shadow, {ImageTransparency = 1}, 0.2):Play()
        task.wait(0.2)
        screenGui.Enabled = false
        if floatingGui then floatingGui.Enabled = true end
    end)
    
    -- Animações de entrada
    mainFrame.BackgroundTransparency = 1
    mainFrame.Size = UDim2.fromOffset(0,0)
    CreateTween(mainFrame, {BackgroundTransparency = 0.05}, 0.3):Play()
    CreateTween(mainFrame, {Size = UDim2.fromOffset(420,540)}, 0.4):Play()
    CreateTween(shadow, {ImageTransparency = 0.4}, 0.4):Play()
    
    -- Toggles
    local aimActive = false
    aimBtn.MouseButton1Click:Connect(function()
        aimActive = not aimActive
        System.aimAssistActive = aimActive
        if aimActive then
            aimBtn.Text = "✓ AIMASSIST ON"
            aimBtn.BackgroundColor3 = Color3.fromRGB(80,200,120)
            fovCircle.Visible = true
        else
            aimBtn.Text = "AIMASSIST"
            aimBtn.BackgroundColor3 = Color3.fromRGB(255,70,120)
            fovCircle.Visible = false
            System.currentTarget = nil
        end
        statusText.Text = string.format("● AIM: %s  |  ESP: %s", aimActive and "ON" or "OFF", System.espActive and "ON" or "OFF")
        statusText.TextColor3 = (aimActive or System.espActive) and Color3.fromRGB(80,255,120) or Color3.fromRGB(255,70,70)
    end)
    
    local espActive = false
    espBtn.MouseButton1Click:Connect(function()
        espActive = not espActive
        System.espActive = espActive
        if espActive then
            espBtn.Text = "✓ ESP ON"
            espBtn.BackgroundColor3 = Color3.fromRGB(80,200,120)
            RefreshAllESP()
        else
            espBtn.Text = "ESP (LINHAS + HIGHLIGHT)"
            espBtn.BackgroundColor3 = Color3.fromRGB(100,100,200)
            for p in pairs(System.espData) do RemoveESPForPlayer(p) end
        end
        statusText.Text = string.format("● AIM: %s  |  ESP: %s", System.aimAssistActive and "ON" or "OFF", espActive and "ON" or "OFF")
        statusText.TextColor3 = (System.aimAssistActive or espActive) and Color3.fromRGB(80,255,120) or Color3.fromRGB(255,70,70)
    end)
end

-- ======================== TELA DE LOGIN ========================
local function CreateLoginUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "LoginUI"
    screenGui.Parent = player.PlayerGui
    
    local bg = Instance.new("Frame")
    bg.Size = UDim2.fromScale(1,1)
    bg.BackgroundColor3 = Color3.fromRGB(0,0,0)
    bg.BackgroundTransparency = 0.7
    bg.Parent = screenGui
    
    local loginFrame = Instance.new("Frame")
    loginFrame.Size = UDim2.fromOffset(420, 340)
    loginFrame.Position = UDim2.fromScale(0.5,0.5)
    loginFrame.AnchorPoint = Vector2.new(0.5,0.5)
    loginFrame.BackgroundColor3 = Color3.fromRGB(20,20,25)
    loginFrame.BackgroundTransparency = 0.1
    loginFrame.BorderSizePixel = 0
    loginFrame.Parent = screenGui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 16)
    corner.Parent = loginFrame
    
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(255,100,150)
    stroke.Thickness = 1.5
    stroke.Transparency = 0.5
    stroke.Parent = loginFrame
    
    local logo = Instance.new("TextLabel")
    logo.Text = "🔐"
    logo.TextSize = 54
    logo.TextColor3 = Color3.fromRGB(255,100,150)
    logo.BackgroundTransparency = 1
    logo.Size = UDim2.fromOffset(420, 80)
    logo.Position = UDim2.fromOffset(0, 20)
    logo.Parent = loginFrame
    
    local title = Instance.new("TextLabel")
    title.Text = "INSIRA SUA KEY"
    title.TextColor3 = Color3.fromRGB(255,255,255)
    title.TextSize = 22
    title.Font = Enum.Font.GothamBold
    title.BackgroundTransparency = 1
    title.Size = UDim2.fromOffset(420, 30)
    title.Position = UDim2.fromOffset(0, 80)
    title.Parent = loginFrame
    
    local keyBox = Instance.new("TextBox")
    keyBox.PlaceholderText = "Digite a key..."
    keyBox.Text = ""
    keyBox.TextColor3 = Color3.fromRGB(255,255,255)
    keyBox.TextSize = 16
    keyBox.Font = Enum.Font.Gotham
    keyBox.BackgroundColor3 = Color3.fromRGB(40,40,48)
    keyBox.BackgroundTransparency = 0.1
    keyBox.Size = UDim2.fromOffset(280, 45)
    keyBox.Position = UDim2.fromScale(0.5, 0.5)
    keyBox.AnchorPoint = Vector2.new(0.5,0.5)
    keyBox.Parent = loginFrame
    local boxCorner = Instance.new("UICorner")
    boxCorner.CornerRadius = UDim.new(0, 10)
    boxCorner.Parent = keyBox
    
    local confirmBtn = Instance.new("TextButton")
    confirmBtn.Text = "CONFIRMAR"
    confirmBtn.TextSize = 18
    confirmBtn.Font = Enum.Font.GothamBold
    confirmBtn.TextColor3 = Color3.fromRGB(255,255,255)
    confirmBtn.BackgroundColor3 = Color3.fromRGB(255,70,120)
    confirmBtn.Size = UDim2.fromOffset(200, 48)
    confirmBtn.Position = UDim2.fromScale(0.5, 0.75)
    confirmBtn.AnchorPoint = Vector2.new(0.5,0.5)
    confirmBtn.Parent = loginFrame
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 24)
    btnCorner.Parent = confirmBtn
    
    MakeDraggable(loginFrame, loginFrame)
    AnimateButton(confirmBtn)
    
    -- Animação entrada
    loginFrame.BackgroundTransparency = 1
    loginFrame.Size = UDim2.fromOffset(0,0)
    CreateTween(loginFrame, {BackgroundTransparency = 0.05}, 0.3):Play()
    CreateTween(loginFrame, {Size = UDim2.fromOffset(420,340)}, 0.4):Play()
    
    confirmBtn.MouseButton1Click:Connect(function()
        if keyBox.Text == CONFIG.VALID_KEY then
            System.keyValidated = true
            CreateTween(loginFrame, {BackgroundTransparency = 1}, 0.3):Play()
            CreateTween(loginFrame, {Size = UDim2.fromOffset(0,0)}, 0.4):Play()
            CreateTween(bg, {BackgroundTransparency = 1}, 0.3):Play()
            task.wait(0.4)
            screenGui:Destroy()
            CreateFloatingButton()
            CreateMainUI()
            StartMonitoring()
            -- Loop do aim
            RunService.RenderStepped:Connect(function()
                if System.aimAssistActive then
                    local target = GetClosestTargetInFOV()
                    if target then LockToTarget(target) end
                end
            end)
        else
            ShowMessage(loginFrame, "❌ Key inválida! Tente novamente.", true)
            local orig = loginFrame.Position
            for i=1,3 do
                CreateTween(loginFrame, {Position = orig + UDim2.fromOffset(10,0)}, 0.05):Play()
                task.wait(0.05)
                CreateTween(loginFrame, {Position = orig}, 0.05):Play()
                task.wait(0.05)
            end
        end
    end)
    
    keyBox.Focused:Connect(function()
        CreateTween(keyBox, {BackgroundTransparency = 0}, 0.2):Play()
    end)
    keyBox.FocusLost:Connect(function()
        CreateTween(keyBox, {BackgroundTransparency = 0.1}, 0.2):Play()
    end)
end

-- ======================== INICIALIZAÇÃO ========================
CreateLoginUI()
UserInputService.MouseIconEnabled = true

-- Ajuste do FOV circle na tela
RunService.RenderStepped:Connect(function()
    if System.aimAssistActive and System.uiReferences.fovCircle then
        local vp = camera.ViewportSize
        System.uiReferences.fovCircle.Position = UDim2.fromOffset(vp.X/2, vp.Y/2)
    end
end)
