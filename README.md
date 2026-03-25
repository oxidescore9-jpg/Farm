-- SERVICES
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

-- VARIABLES
local player = Players.LocalPlayer
local camera = workspace.CurrentCamera
local clickerConnection, patrolConnection, cameraLockConnection = nil, nil, nil
local isPatrolling = false
local currentDirection = Vector3.new(0, 0, 0)
local activeBarriers = {}
local camHeight = 17.5 

-- ==========================================
-- 1. ИНТЕРФЕЙС
-- ==========================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "FarmFix_Final_V11"
screenGui.ResetOnSpawn = false
local success, coreGui = pcall(function() return game:GetService("CoreGui") end)
screenGui.Parent = success and coreGui or player:WaitForChild("PlayerGui")

local toggleBtn = Instance.new("TextButton")
toggleBtn.Size = UDim2.new(0, 40, 0, 40)
toggleBtn.Position = UDim2.new(0, 15, 0, 15)
toggleBtn.BackgroundColor3 = Color3.fromRGB(46, 184, 114)
toggleBtn.Text = "F"
toggleBtn.TextColor3 = Color3.new(1,1,1)
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.ZIndex = 10
toggleBtn.Parent = screenGui
Instance.new("UICorner", toggleBtn).CornerRadius = UDim.new(0, 10)

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 170, 0, 340)
mainFrame.Position = UDim2.new(0.5, -85, 0.5, -170)
mainFrame.BackgroundColor3 = Color3.fromRGB(12, 12, 15)
mainFrame.BorderSizePixel = 0
mainFrame.Parent = screenGui
Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 12)

local uiScale = Instance.new("UIScale", mainFrame)
local list = Instance.new("UIListLayout", mainFrame)
list.Padding = UDim.new(0, 5)
list.HorizontalAlignment = Enum.HorizontalAlignment.Center
list.VerticalAlignment = Enum.VerticalAlignment.Center

local function btn(txt, clr)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0, 150, 0, 35)
    b.BackgroundColor3 = clr
    b.Text = txt
    b.TextColor3 = Color3.new(1,1,1)
    b.Font = Enum.Font.GothamBold
    b.TextSize = 11
    b.Parent = mainFrame
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)
    return b
end

local inputFrame = Instance.new("Frame")
inputFrame.Size = UDim2.new(0, 150, 0, 28)
inputFrame.BackgroundTransparency = 1
inputFrame.Parent = mainFrame
Instance.new("UIListLayout", inputFrame).FillDirection = Enum.FillDirection.Horizontal
inputFrame.UIListLayout.Padding = UDim.new(0, 4)

local function createInp(val, hint)
    local i = Instance.new("TextBox")
    i.Size = UDim2.new(0, 47, 1, 0)
    i.BackgroundColor3 = Color3.fromRGB(25,25,30)
    i.Text = val
    i.PlaceholderText = hint
    i.TextColor3 = Color3.new(1,1,1)
    i.Font = Enum.Font.GothamBold
    i.TextSize = 10
    i.ClearTextOnFocus = false -- Чтобы текст не удалялся сам
    i.Parent = inputFrame
    Instance.new("UICorner", i)
    return i
end

local bInp = createInp("12", "Beds")
local lInp = createInp("480", "Len")
local sInp = createInp("30", "Sec")

local cStart = btn("START CLICKER", Color3.fromRGB(46, 184, 114))
local cStop  = btn("STOP CLICKER", Color3.fromRGB(231, 76, 60))
local pBtn   = btn("START PATROL", Color3.fromRGB(52, 152, 219))
local bBtn   = btn("SPAWN BEDS", Color3.fromRGB(155, 89, 182))
local dBtn   = btn("DESTROY", Color3.fromRGB(40, 40, 45))

-- ==========================================
-- 2. СКРЫТИЕ И ДРАГ
-- ==========================================
local isHidden = false
toggleBtn.MouseButton1Click:Connect(function()
    isHidden = not isHidden
    local targetScale = isHidden and 0 or 1
    TweenService:Create(uiScale, TweenInfo.new(0.3, Enum.EasingStyle.Quart), {Scale = targetScale}):Play()
    mainFrame.Visible = true 
    task.delay(isHidden and 0.3 or 0, function()
        if isHidden then mainFrame.Visible = false end
    end)
end)

local d, s, p
mainFrame.InputBegan:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then d = true s = i.Position p = mainFrame.Position end end)
UserInputService.InputChanged:Connect(function(i) if d and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then 
    local del = i.Position - s mainFrame.Position = UDim2.new(p.X.Scale, p.X.Offset + del.X, p.Y.Scale, p.Y.Offset + del.Y) 
end end)
UserInputService.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then d = false end end)

-- ==========================================
-- 3. ЛОГИКА ПАТРУЛЯ (ИСПРАВЛЕНО ЧТЕНИЕ ТЕКСТА)
-- ==========================================
local function waitPatrol(duration)
    local elapsed = 0
    while elapsed < duration and isPatrolling do elapsed = elapsed + task.wait() end
end

pBtn.MouseButton1Click:Connect(function()
    if isPatrolling then
        isPatrolling = false
        if cameraLockConnection then cameraLockConnection:Disconnect() end
        camera.CameraType = Enum.CameraType.Custom
        pBtn.Text = "START PATROL"
        pBtn.BackgroundColor3 = Color3.fromRGB(52, 152, 219)
    else
        local char = player.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end

        isPatrolling = true
        pBtn.Text = "STOP PATROL"
        pBtn.BackgroundColor3 = Color3.fromRGB(230, 126, 34)

        -- Запоминаем данные прямо сейчас
        local walkSec = tonumber(sInp.Text) or 30 
        local startRotation = hrp.CFrame.Rotation

        camera.CameraType = Enum.CameraType.Scriptable
        cameraLockConnection = RunService.RenderStepped:Connect(function()
            if hrp and isPatrolling then
                camera.CFrame = CFrame.new(hrp.Position) * startRotation * CFrame.new(0, camHeight, 0) * CFrame.Angles(math.rad(-90), 0, 0)
            end
        end)

        local fwd = hrp.CFrame.LookVector
        local back = -fwd
        local right = hrp.CFrame.RightVector

        patrolConnection = RunService.RenderStepped:Connect(function()
            local h = char:FindFirstChild("Humanoid")
            if h and isPatrolling then h:Move(currentDirection * 0.4, false) end
        end)

        task.spawn(function()
            while isPatrolling do
                -- Каждый раз проверяем, вдруг ты поменял время во время патруля (для следующего круга)
                local currentWalkSec = tonumber(sInp.Text) or 30
                
                currentDirection = fwd; waitPatrol(currentWalkSec)
                if not isPatrolling then break end
                
                currentDirection = right; waitPatrol(0.55)
                if not isPatrolling then break end
                
                currentDirection = back; waitPatrol(currentWalkSec)
                if not isPatrolling then break end
                
                currentDirection = right; waitPatrol(0.55)
            end
            if cameraLockConnection then cameraLockConnection:Disconnect() end
            if patrolConnection then patrolConnection:Disconnect() end
            camera.CameraType = Enum.CameraType.Custom
            currentDirection = Vector3.new(0,0,0)
        end)
    end
end)

-- ==========================================
-- 4. СПАВН СТЕН (ИСПРАВЛЕНО ЧТЕНИЕ ТЕКСТА)
-- ==========================================
bBtn.MouseButton1Click:Connect(function()
    for _, o in pairs(activeBarriers) do if o then o:Destroy() end end
    activeBarriers = {}
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    -- Читаем значения ПРЯМО ЗДЕСЬ
    local count = tonumber(bInp.Text) or 12
    local len = tonumber(lInp.Text) or 480
    
    for i = 0, count do
        local w = Instance.new("Part")
        w.Size = Vector3.new(0.5, 30, len)
        w.Material = Enum.Material.ForceField
        w.Color = Color3.fromRGB(0, 255, 100)
        w.Anchored = true
        w.CanCollide = true
        w.CFrame = hrp.CFrame * CFrame.new(-5 + (i * 10), 0, -len/2)
        w.Parent = workspace
        table.insert(activeBarriers, w)
    end
end)

-- ==========================================
-- 5. КЛИКЕР И УДАЛЕНИЕ
-- ==========================================
cStart.MouseButton1Click:Connect(function()
    if clickerConnection then return end
    clickerConnection = RunService.RenderStepped:Connect(function()
        VirtualInputManager:SendMouseButtonEvent(camera.ViewportSize.X/2 - 50, camera.ViewportSize.Y/2, 0, true, game, 1)
        VirtualInputManager:SendMouseButtonEvent(camera.ViewportSize.X/2 - 50, camera.ViewportSize.Y/2, 0, false, game, 1)
    end)
    cStart.Text = "ON"
end)

cStop.MouseButton1Click:Connect(function()
    if clickerConnection then clickerConnection:Disconnect() clickerConnection = nil end
    cStart.Text = "START CLICKER"
end)

dBtn.MouseButton1Click:Connect(function()
    isPatrolling = false
    camera.CameraType = Enum.CameraType.Custom
    if clickerConnection then clickerConnection:Disconnect() end
    if cameraLockConnection then cameraLockConnection:Disconnect() end
    for _, o in pairs(activeBarriers) do if o then o:Destroy() end end
    screenGui:Destroy()
end)
