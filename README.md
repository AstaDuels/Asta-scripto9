local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UIS = game:GetService("UserInputService")
local player = Players.LocalPlayer

-- ===== ESTADO DO BYPASS TP =====
local bypassTPEnabled = false
local bypassTPConn = nil
local bypassHitCD = false
local bypassSwingCD = 0.2
local bypassHitDist = 6
local guiLocked = false

-- ===== ESTADO DO ANTI DIE =====
local antiDieEnabled = true
local antiDieHumanoid = nil
local antiDieRoot = nil
local antiDieChar = nil

-- ===== REATIVAÇÃO RÁPIDA DO BYPASS =====
local function reativarBypass()
    if not bypassTPEnabled then return end
    if bypassTPConn then
        bypassTPConn:Disconnect()
        bypassTPConn = nil
    end
    task.wait(0.1)
    startBypassTP()
end

player.CharacterAdded:Connect(function(char)
    task.wait(0.1)
    reativarBypass()
end)

-- ============================================================
--  FUNÇÕES DO BYPASS TP
-- ============================================================
local function findBat()
    local char = player.Character
    if not char then return nil end
    
    local BAT_LIST = {
        "Bat", "Slap", "Iron Slap", "Gold Slap", "Diamond Slap",
        "Emerald Slap", "Ruby Slap", "Dark Matter Slap", "Flame Slap",
        "Nuclear Slap", "Galaxy Slap", "Glitched Slap"
    }
    
    for _, name in ipairs(BAT_LIST) do
        local t = char:FindFirstChild(name)
        if t and t:IsA("Tool") then return t end
    end
    
    local bp = player:FindFirstChildOfClass("Backpack")
    if bp then
        for _, name in ipairs(BAT_LIST) do
            local t = bp:FindFirstChild(name)
            if t and t:IsA("Tool") then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then pcall(function() hum:EquipTool(t) end) end
                return t
            end
        end
    end
    return nil
end

local function trySwing()
    if bypassHitCD then return end
    bypassHitCD = true
    pcall(function()
        local bat = findBat()
        if bat then
            local char = player.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hum and bat.Parent ~= char then
                pcall(function() hum:EquipTool(bat) end)
            end
            pcall(function() bat:Activate() end)
        end
    end)
    task.delay(bypassSwingCD, function() bypassHitCD = false end)
end

local function getClosestTarget()
    local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not root then return nil, math.huge end
    
    local closest, minDist = nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= player and plr.Character then
            local tRoot = plr.Character:FindFirstChild("HumanoidRootPart")
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            if tRoot and hum and hum.Health > 0 then
                local dist = (tRoot.Position - root.Position).Magnitude
                if dist < minDist then
                    minDist = dist
                    closest = tRoot
                end
            end
        end
    end
    return closest, minDist
end

local function stickToTarget(root, target)
    if not root or not target then return end
    
    local targetPos = target.Position
    local currentPos = root.Position
    local heightDiff = targetPos.Y - currentPos.Y
    
    if math.abs(heightDiff) > 1.5 then
        root.CFrame = CFrame.new(targetPos + Vector3.new(0, 1.5, 0))
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
        return
    end
    
    local targetCF = CFrame.new(targetPos + Vector3.new(0, 1.5, 0))
    local tweenInfo = TweenInfo.new(0.1, Enum.EasingStyle.Linear, Enum.EasingDirection.Out)
    local tween = TweenService:Create(root, tweenInfo, { CFrame = targetCF })
    tween:Play()
    root.AssemblyLinearVelocity = Vector3.zero
    root.AssemblyAngularVelocity = Vector3.zero
end

local function updateBypass()
    if not bypassTPEnabled then return end
    
    local char = player.Character
    if not char then return end
    
    local root = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not root or not hum then return end
    
    if not char:FindFirstChildOfClass("Tool") then
        local bat = findBat()
        if bat then pcall(function() hum:EquipTool(bat) end) end
    end
    
    local target, targetDist = getClosestTarget()
    if not target then return end
    
    if targetDist > 3 then
        stickToTarget(root, target)
    else
        local targetPos = target.Position + Vector3.new(0, 1.5, 0)
        local currentPos = root.Position
        local diff = (targetPos - currentPos)
        if diff.Magnitude > 0.5 then
            local moveSpeed = 60
            local moveDir = diff.Unit
            root.AssemblyLinearVelocity = Vector3.new(
                moveDir.X * moveSpeed,
                diff.Y * 15,
                moveDir.Z * moveSpeed
            )
        else
            root.AssemblyLinearVelocity = Vector3.zero
        end
    end
    
    if sethiddenproperty then
        pcall(function() sethiddenproperty(root, "PhysicsRepRootPart", target) end)
    end
    
    local cam = workspace.CurrentCamera
    if cam then
        cam.CFrame = CFrame.new(cam.CFrame.Position, target.Position)
    end
    
    local myPos = root.Position
    local targetPos = target.Position
    local toTarget = targetPos - myPos
    if toTarget.Magnitude > 0.1 then
        local goalCF = CFrame.lookAt(myPos, targetPos)
        local diffCF = root.CFrame:Inverse() * goalCF
        local rx, ry, rz = diffCF:ToEulerAnglesXYZ()
        rx = math.clamp(rx, -2.5, 2.5)
        ry = math.clamp(ry, -2.5, 2.5)
        rz = math.clamp(rz, -2.5, 2.5)
        root.AssemblyAngularVelocity = root.CFrame:VectorToWorldSpace(Vector3.new(rx * 42, ry * 42, rz * 42))
    end
    
    if targetDist <= bypassHitDist then trySwing() end
end

function startBypassTP()
    if bypassTPConn then
        bypassTPConn:Disconnect()
        bypassTPConn = nil
    end
    
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    if hum then hum.AutoRotate = false end
    
    bypassTPConn = RunService.Heartbeat:Connect(updateBypass)
end

function stopBypassTP()
    if bypassTPConn then
        bypassTPConn:Disconnect()
        bypassTPConn = nil
    end
    
    local char = player.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
    end
    
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if hum then
        hum.AutoRotate = true
        pcall(function() hum:ChangeState(Enum.HumanoidStateType.GettingUp) end)
    end
    
    bypassHitCD = false
end

function toggleBypassTP()
    bypassTPEnabled = not bypassTPEnabled
    if bypassTPEnabled then
        startBypassTP()
    else
        stopBypassTP()
    end
    return bypassTPEnabled
end

-- ============================================================
--  ANTI DIE - FUNCIONALIDADE COMPLETA
-- ============================================================
local function healPlayer()
    if not antiDieEnabled or not antiDieHumanoid then return end
    
    antiDieHumanoid.Health = antiDieHumanoid.MaxHealth
    
    if antiDieHumanoid.Health <= 0 then
        antiDieHumanoid:BreakJoints()
        task.wait(0.05)
        antiDieHumanoid.Health = antiDieHumanoid.MaxHealth
    end
end

local function onHealthChanged(newHealth)
    if not antiDieEnabled then return end
    if newHealth <= 5 and antiDieHumanoid and antiDieHumanoid.Health > 0 then
        antiDieHumanoid.Health = antiDieHumanoid.MaxHealth
        return
    end
    if newHealth <= 0 then
        healPlayer()
    end
end

local function setupAntiDie(char)
    if not char then return end
    antiDieChar = char
    antiDieHumanoid = char:FindFirstChildOfClass("Humanoid")
    antiDieRoot = char:FindFirstChild("HumanoidRootPart")
    if not antiDieHumanoid then return end
    
    antiDieHumanoid.HealthChanged:Connect(onHealthChanged)
    antiDieHumanoid.Died:Connect(function()
        if antiDieEnabled then healPlayer() end
    end)
end

player.CharacterAdded:Connect(function(char)
    task.wait(0.1)
    setupAntiDie(char)
end)

-- Loop agressivo do Anti Die
task.spawn(function()
    while true do
        task.wait(0.1)
        if antiDieEnabled and antiDieHumanoid and antiDieHumanoid.Health > 0 then
            if antiDieHumanoid.Health < (antiDieHumanoid.MaxHealth * 0.5) then
                antiDieHumanoid.Health = antiDieHumanoid.MaxHealth
            end
        end
    end
end)

-- Ativa Anti Die no personagem atual
if player.Character then
    task.wait(0.2)
    setupAntiDie(player.Character)
end

function toggleAntiDie()
    antiDieEnabled = not antiDieEnabled
    return antiDieEnabled
end

-- ============================================================
--  GUI MINIMALISTA COM ANTI DIE TOGGLE
-- ============================================================
local function createGUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "BypassTPGUI"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = game:GetService("CoreGui")
    
    -- Frame (aumentado para caber o toggle)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 150, 0, 185)
    frame.Position = UDim2.new(0.5, -75, 0.5, -92)
    frame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    frame.BorderSizePixel = 1
    frame.BorderColor3 = Color3.fromRGB(200, 200, 200)
    frame.Active = true
    frame.Parent = screenGui
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 8)
    
    -- Título
    local title = Instance.new("TextLabel", frame)
    title.Size = UDim2.new(1, 0, 0, 22)
    title.Position = UDim2.new(0, 0, 0, 0)
    title.BackgroundTransparency = 1
    title.Text = "Tp Bypass Free v2"
    title.TextColor3 = Color3.fromRGB(0, 0, 0)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 11
    title.TextYAlignment = Enum.TextYAlignment.Center
    
    -- Botão Unlock
    local lockBtn = Instance.new("TextButton", frame)
    lockBtn.Size = UDim2.new(0, 40, 0, 18)
    lockBtn.Position = UDim2.new(1, -44, 0, 2)
    lockBtn.BackgroundColor3 = Color3.fromRGB(240, 240, 240)
    lockBtn.BorderSizePixel = 1
    lockBtn.BorderColor3 = Color3.fromRGB(200, 200, 200)
    lockBtn.Text = "Unlock"
    lockBtn.TextColor3 = Color3.fromRGB(0, 0, 0)
    lockBtn.Font = Enum.Font.Gotham
    lockBtn.TextSize = 9
    lockBtn.AutoButtonColor = false
    Instance.new("UICorner", lockBtn).CornerRadius = UDim.new(0, 3)
    
    -- Divisória
    local divider = Instance.new("Frame", frame)
    divider.Size = UDim2.new(1, -10, 0, 1)
    divider.Position = UDim2.new(0, 5, 0, 24)
    divider.BackgroundColor3 = Color3.fromRGB(220, 220, 220)
    divider.BorderSizePixel = 0
    
    -- Status do Bypass
    local statusLabel = Instance.new("TextLabel", frame)
    statusLabel.Size = UDim2.new(1, 0, 0, 18)
    statusLabel.Position = UDim2.new(0, 0, 0, 28)
    statusLabel.BackgroundTransparency = 1
    statusLabel.Text = "BYPASS: OFF"
    statusLabel.TextColor3 = Color3.fromRGB(200, 50, 50)
    statusLabel.Font = Enum.Font.GothamBold
    statusLabel.TextSize = 10
    statusLabel.TextYAlignment = Enum.TextYAlignment.Center
    
    -- Descrição
    local descLabel = Instance.new("TextLabel", frame)
    descLabel.Size = UDim2.new(1, 0, 0, 14)
    descLabel.Position = UDim2.new(0, 0, 0, 47)
    descLabel.BackgroundTransparency = 1
    descLabel.Text = "use anti die pra não ficar tomando reset"
    descLabel.TextColor3 = Color3.fromRGB(100, 100, 100)
    descLabel.Font = Enum.Font.Gotham
    descLabel.TextSize = 8
    descLabel.TextWrapped = true
    descLabel.TextYAlignment = Enum.TextYAlignment.Top
    descLabel.TextXAlignment = Enum.TextXAlignment.Center
    
    -- Botão do Bypass
    local btn = Instance.new("TextButton", frame)
    btn.Size = UDim2.new(0, 85, 0, 22)
    btn.Position = UDim2.new(0.5, -42, 0, 68)
    btn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    btn.BorderSizePixel = 0
    btn.Text = "ATIVAR"
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 10
    btn.AutoButtonColor = false
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
    
    -- Status do Anti Die
    local antiDieStatus = Instance.new("TextLabel", frame)
    antiDieStatus.Size = UDim2.new(1, 0, 0, 16)
    antiDieStatus.Position = UDim2.new(0, 0, 0, 98)
    antiDieStatus.BackgroundTransparency = 1
    antiDieStatus.Text = "ANTI DIE: ON"
    antiDieStatus.TextColor3 = Color3.fromRGB(50, 200, 50)
    antiDieStatus.Font = Enum.Font.GothamBold
    antiDieStatus.TextSize = 10
    antiDieStatus.TextYAlignment = Enum.TextYAlignment.Center
    
    -- Botão do Anti Die (toggle)
    local antiDieBtn = Instance.new("TextButton", frame)
    antiDieBtn.Size = UDim2.new(0, 85, 0, 22)
    antiDieBtn.Position = UDim2.new(0.5, -42, 0, 118)
    antiDieBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
    antiDieBtn.BorderSizePixel = 0
    antiDieBtn.Text = "ANTI DIE: ON"
    antiDieBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    antiDieBtn.Font = Enum.Font.GothamBold
    antiDieBtn.TextSize = 10
    antiDieBtn.AutoButtonColor = false
    Instance.new("UICorner", antiDieBtn).CornerRadius = UDim.new(0, 4)
    
    -- ===== LÓGICA DA GUI =====
    local function updateLock()
        if guiLocked then
            lockBtn.Text = "Lock"
            frame.Active = false
            frame.Draggable = false
        else
            lockBtn.Text = "Unlock"
            frame.Active = true
            frame.Draggable = true
        end
    end
    
    lockBtn.Activated:Connect(function()
        guiLocked = not guiLocked
        updateLock()
    end)
    updateLock()
    
    -- Atualiza a UI do Bypass
    local function updateBypassUI()
        if bypassTPEnabled then
            statusLabel.Text = "BYPASS: ON"
            statusLabel.TextColor3 = Color3.fromRGB(50, 200, 50)
            btn.Text = "DESATIVAR"
        else
            statusLabel.Text = "BYPASS: OFF"
            statusLabel.TextColor3 = Color3.fromRGB(200, 50, 50)
            btn.Text = "ATIVAR"
        end
    end
    
    btn.Activated:Connect(function()
        toggleBypassTP()
        updateBypassUI()
    end)
    updateBypassUI()
    
    -- Atualiza a UI do Anti Die
    local function updateAntiDieUI()
        if antiDieEnabled then
            antiDieStatus.Text = "ANTI DIE: ON"
            antiDieStatus.TextColor3 = Color3.fromRGB(50, 200, 50)
            antiDieBtn.Text = "ANTI DIE: ON"
            antiDieBtn.BackgroundColor3 = Color3.fromRGB(50, 200, 50)
        else
            antiDieStatus.Text = "ANTI DIE: OFF"
            antiDieStatus.TextColor3 = Color3.fromRGB(200, 50, 50)
            antiDieBtn.Text = "ANTI DIE: OFF"
            antiDieBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
        end
    end
    
    antiDieBtn.Activated:Connect(function()
        toggleAntiDie()
        updateAntiDieUI()
    end)
    updateAntiDieUI()
    
    -- Drag
    local dragging = false
    local dragStart, startPos
    
    frame.InputBegan:Connect(function(input)
        if guiLocked then return end
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    
    UIS.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X,
                                       startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

createGUI()
