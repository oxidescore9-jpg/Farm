local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera
local clickerConnection, patrolConnection, cameraLockConnection = nil, nil, nil
local isPatrolling = false
local currentDirection = Vector3.new(0, 0, 0)
local activeBarriers = {}
local camHeight = 17.5 

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "FarmSystem_Final_V14"
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
mainFrame.Size = UDim2.new(0, 180, 0, 340)
mainFrame.Position = UDim2.new(0.5, -90, 0.5, -170)
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
    b.Size = UDim2.new(0, 160, 0, 35)
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
inputFrame.Size = UDim2.new(0, 160, 0, 28)
inputFrame.BackgroundTransparency = 1
inputFrame.Parent = mainFrame
Instance.new("UIListLayout", inputFrame).FillDirection = Enum.FillDirection.Horizontal
inputFrame.UIListLayout.Padding = UDim.new(0, 4)

local function createInp(val, hint)
    local i = Instance.new("TextBox")
    i.Size = UDim2.new(0, 37, 1, 0)
    i.BackgroundColor3 = Color3.fromRGB(25,25,30)
    i.Text = val
    i.PlaceholderText = hint
    i.TextColor3 = Color3.new(1,1,1)
    i.Font = Enum.Font.GothamBold
    i.TextSize = 9
    i.ClearTextOnFocus = false
    i.Parent = inputFrame
    Instance.new("UICorner", i)
    return i
end

local bInp = createInp("12", "Beds")
local lInp = createInp("480", "Len")
local sInp = createInp("30", "Sec")
local wInp = createInp("10", "Width")

local cStart = btn("START CLICKER", Color3.fromRGB(46, 184, 114))
local cStop  = btn("STOP CLICKER", Color3.fromRGB(231, 76, 60))
local pBtn   = btn("START PATROL", Color3.fromRGB(52, 152, 219))
local bBtn   = btn("SPAWN BEDS", Color3.fromRGB(155, 89, 182))
local dBtn   = btn("DESTROY", Color3.fromRGB(40, 40, 45))

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

local function waitPatrol(duration)
    local elapsed = 0
    while elapsed < duration and isPatrolling do elapsed = elapsed + task.wait() end
end

pBtn.MouseButton1Click:Connect(function()
    if isPatrolling then
        isPatrolling = false
        if cameraLockConnection then cameraLockConnection:Disconnect() end
        if patrolConnection then patrolConnection:Disconnect() end
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
                local currentWalkSec = tonumber(sInp.Text) or 30
                currentDirection = fwd; waitPatrol(currentWalkSec)
                if not isPatrolling then break end
                currentDirection = right; waitPatrol(0.6)
                if not isPatrolling then break end
                currentDirection = back; waitPatrol(currentWalkSec)
                if not isPatrolling then break end
                currentDirection = right; waitPatrol(0.6)
            end
            if cameraLockConnection then cameraLockConnection:Disconnect() end
            if patrolConnection then patrolConnection:Disconnect() end
            camera.CameraType = Enum.CameraType.Custom
            currentDirection = Vector3.new(0,0,0)
        end)
    end
end)

bBtn.MouseButton1Click:Connect(function()
    for _, o in pairs(activeBarriers) do if o then o:Destroy() end end
    activeBarriers = {}
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    local count = tonumber(bInp.Text) or 12
    local len = tonumber(lInp.Text) or 480
    local width = tonumber(wInp.Text) or 10
    
    for i = 0, count do
        local w = Instance.new("Part")
        w.Name = "FarmWall"
        w.Size = Vector3.new(0.5, 30, len)
        w.Material = Enum.Material.ForceField
        w.Color = Color3.fromRGB(0, 255, 100)
        w.Anchored = true
        w.CanCollide = true
        w.Transparency = 0.5
        w.CFrame = hrp.CFrame * CFrame.new(-(width/2) + (i * width), 0, -len/2)
        w.Parent = workspace
        table.insert(activeBarriers, w)
        
        if i < count then
            local roof = Instance.new("Part")
            roof.Name = "FarmRoof"
            roof.Size = Vector3.new(width, 0.5, len)
            roof.Material = Enum.Material.ForceField
            roof.Color = Color3.fromRGB(0, 255, 100)
            roof.Anchored = true
            roof.CanCollide = true
            roof.Transparency = 0.5
            roof.CFrame = hrp.CFrame * CFrame.new(i * width, 4, -len / 2)
            roof.Parent = workspace
            table.insert(activeBarriers, roof)
        end
    end
end)

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
