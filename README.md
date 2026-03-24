-- SERVICES
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local UserInputService = game:GetService("UserInputService")

-- VARIABLES
local player = Players.LocalPlayer
local camera = workspace.CurrentCamera
local clickerConnection = nil
local patrolConnection = nil
local isPatrolling = false
local currentDirection = Vector3.new(0, 0, 0)
local forceSpeedMultiplier = 0.4 
local activeBarriers = {} 

-- ==========================================
-- 1. СОЗДАНИЕ ИНТЕРФЕЙСА (GUI)
-- ==========================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "MobileOptimizer_Pro"
screenGui.ResetOnSpawn = false
local success, coreGui = pcall(function() return game:GetService("CoreGui") end)
screenGui.Parent = success and coreGui or player:WaitForChild("PlayerGui")

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 260, 0, 420) 
mainFrame.Position = UDim2.new(0.5, -130, 0.5, -210)
mainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
mainFrame.BorderSizePixel = 0
mainFrame.Active = true
mainFrame.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 15)
corner.Parent = mainFrame

local listLayout = Instance.new("UIListLayout")
listLayout.Parent = mainFrame
listLayout.SortOrder = Enum.SortOrder.LayoutOrder
listLayout.Padding = UDim.new(0, 10)
listLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
listLayout.VerticalAlignment = Enum.VerticalAlignment.Center

local function createButton(text, bgColor)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 220, 0, 55)
    btn.BackgroundColor3 = bgColor
    btn.Text = text
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 16
    btn.Parent = mainFrame
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 10)
    btnCorner.Parent = btn
    return btn
end

local btnStartClicker = createButton("START CLICKER", Color3.fromRGB(46, 184, 114))
local btnStopClicker  = createButton("STOP CLICKER", Color3.fromRGB(231, 76, 60))
local btnPatrol       = createButton("START PATROL", Color3.fromRGB(52, 152, 219))
local btnBoundaries   = createButton("SPAWN 12 BEDS", Color3.fromRGB(155, 89, 182))
local btnDestroy      = createButton("DESTROY SCRIPT", Color3.fromRGB(40, 40, 45))

-- Перетаскивание
local dragging, dragInput, dragStart, startPos
mainFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true; dragStart = input.Position; startPos = mainFrame.Position
        input.Changed:Connect(function() if input.UserInputState == Enum.UserInputState.End then dragging = false end end)
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        mainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

-- ==========================================
-- 2. ФУНКЦИЯ БАРЬЕРОВ (ТЕПЕРЬ 12 ГРЯДОК ВПРАВО)
-- ==========================================
btnBoundaries.MouseButton1Click:Connect(function()
    for _, obj in pairs(activeBarriers) do if obj then obj:Destroy() end end
    activeBarriers = {}

    local char = player.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    local function createWall(offsetSide)
        local wall = Instance.new("Part")
        wall.Name = "FarmBarrier"
        wall.Size = Vector3.new(0.5, 30, 480) -- Высота 30 (в 2 раза больше оригинала)
        wall.Material = Enum.Material.ForceField 
        wall.Color = Color3.fromRGB(0, 255, 100) 
        wall.Transparency = 0.5
        wall.Anchored = true
        wall.CanCollide = true 
        
        local relativePos = hrp.CFrame * CFrame.new(offsetSide, 0, -240)
        wall.CFrame = relativePos
        wall.Parent = workspace
        table.insert(activeBarriers, wall)
    end

    -- Создаем 13 стен (чтобы получить 12 дорожек по 10 студов шириной)
    for i = 0, 12 do
        createWall(-5 + (i * 10))
    end
end)

-- ==========================================
-- 3. АВТОКЛИКЕР
-- ==========================================
btnStartClicker.MouseButton1Click:Connect(function()
    if clickerConnection then return end
    clickerConnection = RunService.RenderStepped:Connect(function()
        local targetX = (camera.ViewportSize.X / 2) - 50
        local targetY = camera.ViewportSize.Y / 2
        VirtualInputManager:SendMouseButtonEvent(targetX, targetY, 0, true, game, 1)
        VirtualInputManager:SendMouseButtonEvent(targetX, targetY, 0, false, game, 1)
    end)
    btnStartClicker.Text = "CLICKER ACTIVE"
end)

btnStopClicker.MouseButton1Click:Connect(function()
    if clickerConnection then clickerConnection:Disconnect() clickerConnection = nil end
    btnStartClicker.Text = "START CLICKER"
end)

-- ==========================================
-- 4. ПАТРУЛЬ (ОРИГИНАЛЬНАЯ ЛОГИКА)
-- ==========================================
local function waitPatrol(duration)
    local elapsed = 0
    while elapsed < duration and isPatrolling do elapsed += task.wait() end
end

local function patrolCycle()
    while isPatrolling do
        local char = player.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then task.wait(1) continue end
        local fwd = hrp.CFrame.LookVector
        local right = hrp.CFrame.RightVector
        local back = -hrp.CFrame.LookVector
        
        currentDirection = fwd; waitPatrol(30)
        if not isPatrolling then break end
        currentDirection = right; waitPatrol(0.5)
        if not isPatrolling then break end
        currentDirection = back; waitPatrol(30)
        if not isPatrolling then break end
        currentDirection = right; waitPatrol(0.5)
        if not isPatrolling then break end
    end
    currentDirection = Vector3.new(0, 0, 0)
end

btnPatrol.MouseButton1Click:Connect(function()
    if isPatrolling then
        isPatrolling = false; btnPatrol.Text = "START PATROL"
        btnPatrol.BackgroundColor3 = Color3.fromRGB(52, 152, 219)
        if patrolConnection then patrolConnection:Disconnect() end
    else
        isPatrolling = true; btnPatrol.Text = "STOP PATROL"
        btnPatrol.BackgroundColor3 = Color3.fromRGB(230, 126, 34)
        patrolConnection = RunService.RenderStepped:Connect(function()
            local hum = player.Character and player.Character:FindFirstChild("Humanoid")
            if hum and isPatrolling then hum:Move(currentDirection * forceSpeedMultiplier, false) end
        end)
        task.spawn(patrolCycle)
    end
end)

-- ==========================================
-- 5. УДАЛЕНИЕ (DESTROY)
-- ==========================================
btnDestroy.MouseButton1Click:Connect(function()
    isPatrolling = false
    if clickerConnection then clickerConnection:Disconnect() end
    if patrolConnection then patrolConnection:Disconnect() end
    for _, obj in pairs(activeBarriers) do if obj then obj:Destroy() end end
    task.wait(0.1)
    if player.Character and player.Character:FindFirstChild("Humanoid") then
        player.Character.Humanoid:Move(Vector3.new(0,0,0), false)
    end
    screenGui:Destroy()
end)
