local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera
local clickerConnection, patrolConnection, cameraLockConnection = nil, nil, nil
local isPatrolling = false 
local isTimedPatrolling = false 
local isClicking = false
local currentDirection = Vector3.new(0, 0, 0)
local activeBarriers = {}
local waypoints = {}
local currentWpIndex = 1
local camHeight = 17.5 

local color1 = Color3.fromRGB(180, 100, 255) 
local color2 = Color3.fromRGB(40, 10, 80)  
local currentGradiantColor = color1

task.spawn(function()
    local t = 0
    RunService.Heartbeat:Connect(function(dt)
        t = t + dt * 0.6
        local ratio = (math.sin(t) + 1) / 2
        currentGradiantColor = color1:Lerp(color2, ratio)
    end)
end)

local function applyGradiant(obj, prop)
    RunService.Heartbeat:Connect(function()
        if obj and obj.Parent then
            obj[prop] = currentGradiantColor
        end
    end)
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "FarmSystem_Clean_Slang"
ScreenGui.ResetOnSpawn = false
local success, coreGui = pcall(function() return game:GetService("CoreGui") end)
ScreenGui.Parent = success and coreGui or player:WaitForChild("PlayerGui")

local btnHide = Instance.new("ImageButton", ScreenGui)
btnHide.Size = UDim2.new(0, 45, 0, 45)
btnHide.Position = UDim2.new(0, 10, 0.5, -135)
btnHide.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
btnHide.Image = "rbxassetid://100142668268808"
Instance.new("UICorner", btnHide).CornerRadius = UDim.new(0, 12)
local hStroke = Instance.new("UIStroke", btnHide)
hStroke.Thickness = 2
applyGradiant(hStroke, "Color")

local MasterFrame = Instance.new("Frame", ScreenGui)
MasterFrame.Size = UDim2.new(0, 180, 0, 310) 
MasterFrame.Position = UDim2.new(0, 70, 0.5, -155)
MasterFrame.BackgroundTransparency = 1
MasterFrame.Active = true
MasterFrame.Draggable = true

local MainMenu = Instance.new("ImageLabel", MasterFrame)
MainMenu.Size = UDim2.new(1, 0, 1, 0)
MainMenu.BackgroundTransparency = 1
MainMenu.Image = "rbxassetid://136216611932487"
MainMenu.ScaleType = Enum.ScaleType.Crop
Instance.new("UICorner", MainMenu).CornerRadius = UDim.new(0, 15)
local mStroke = Instance.new("UIStroke", MainMenu)
mStroke.Thickness = 3
applyGradiant(mStroke, "Color")

local listLayout = Instance.new("UIListLayout", MainMenu)
listLayout.Padding = UDim.new(0, 6)
listLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
listLayout.VerticalAlignment = Enum.VerticalAlignment.Center
Instance.new("UIPadding", MainMenu).PaddingTop = UDim.new(0, 8)

local InputFrame = Instance.new("Frame", MainMenu)
InputFrame.Size = UDim2.new(0.9, 0, 0, 28)
InputFrame.BackgroundTransparency = 1
local ilist = Instance.new("UIListLayout", InputFrame)
ilist.FillDirection = Enum.FillDirection.Horizontal
ilist.Padding = UDim.new(0, 4)
ilist.HorizontalAlignment = Enum.HorizontalAlignment.Center

local function createInp(val, hint)
    local i = Instance.new("TextBox", InputFrame)
    i.Size = UDim2.new(0, 36, 1, 0)
    i.BackgroundColor3 = Color3.fromRGB(15, 10, 20)
    i.BackgroundTransparency = 0.2
    i.Text = val
    i.PlaceholderText = hint
    i.TextColor3 = Color3.fromRGB(255, 255, 255)
    i.Font = Enum.Font.GothamBold
    i.TextSize = 10
    i.ClearTextOnFocus = false
    Instance.new("UICorner", i).CornerRadius = UDim.new(0, 4)
    local stroke = Instance.new("UIStroke", i)
    stroke.Thickness = 1
    applyGradiant(stroke, "Color")
    return i
end

local bInp = createInp("12", "B")
local lInp = createInp("480", "L")
local sInp = createInp("30", "S") 
local wInp = createInp("10", "W")

local function createStyledBtn(text, order)
    local btn = Instance.new("TextButton", MainMenu)
    btn.Size = UDim2.new(0.9, 0, 0, 32)
    btn.BackgroundColor3 = Color3.fromRGB(10, 10, 15)
    btn.BackgroundTransparency = 0.25 
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 11
    btn.TextColor3 = Color3.fromRGB(255, 255, 255) 
    btn.Text = text
    btn.LayoutOrder = order
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 8)
    local stroke = Instance.new("UIStroke", btn)
    stroke.Thickness = 1.2
    stroke.Transparency = 0.5
    applyGradiant(stroke, "Color") 
    return btn
end

local btnSpawn = createStyledBtn("🏁 СТЕНЫ", 1)
local btnPatrol = createStyledBtn("♟️ ФАРМ КОТЛОВ", 2)
local btnTimerPatrol = createStyledBtn("🍀 ПЛЕНТ КОТЛОВ", 3)
local btnClicker = createStyledBtn("🎮 КЛИК: 🔳", 4)
local btnDestroy = createStyledBtn("✖️ СНЕСТИ", 5)

local function stopAllPatrols()
    isPatrolling = false
    isTimedPatrolling = false
    if patrolConnection then patrolConnection:Disconnect() end
    camera.CameraType = Enum.CameraType.Custom
    if cameraLockConnection then cameraLockConnection:Disconnect() end
    btnPatrol.Text = "♟️ ФАРМ КОТЛОВ "
    btnTimerPatrol.Text = "🍀 ПЛЕНТ КОТЛОВ "
    currentDirection = Vector3.new(0,0,0)
end

local function waitPatrol(duration)
    local elapsed = 0
    while elapsed < duration and isTimedPatrolling do elapsed = elapsed + task.wait() end
end

btnSpawn.MouseButton1Click:Connect(function()
    for _, o in pairs(activeBarriers) do if o then o:Destroy() end end
    activeBarriers = {}
    waypoints = {} 
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    local count = tonumber(bInp.Text) or 12
    local len = tonumber(lInp.Text) or 480
    local width = tonumber(wInp.Text) or 10
    local baseCF = hrp.CFrame

    for i = 0, count - 1 do
        for wallIdx = 0, 1 do
            local w = Instance.new("Part", workspace)
            w.Size = Vector3.new(0.5, 30, len)
            w.Material = Enum.Material.ForceField
            w.Anchored = true; w.Transparency = 0.5
            w.CFrame = baseCF * CFrame.new(-(width/2) + ((i + wallIdx) * width), 0, -len/2)
            table.insert(activeBarriers, w)
            applyGradiant(w, "Color")
        end
        
        local roof = Instance.new("Part", workspace)
        roof.Size = Vector3.new(width, 0.5, len)
        roof.Material = Enum.Material.ForceField
        roof.Anchored = true; roof.Transparency = 0.5
        roof.CFrame = baseCF * CFrame.new(i * width, 4, -len / 2)
        table.insert(activeBarriers, roof)
        applyGradiant(roof, "Color")
        
        local ptStartPos = (baseCF * CFrame.new(i * width, 0, 5)).Position
        local ptEndPos = (baseCF * CFrame.new(i * width, 0, -len - 5)).Position
        
        if i % 2 == 0 then
            table.insert(waypoints, ptStartPos)
            table.insert(waypoints, ptEndPos)
        else
            table.insert(waypoints, ptEndPos)
            table.insert(waypoints, ptStartPos)
        end
    end
end)

btnPatrol.MouseButton1Click:Connect(function()
    if isPatrolling then stopAllPatrols() else
        stopAllPatrols()
        if #waypoints == 0 then return end 
        local char = player.Character
        local humanoid = char:FindFirstChild("Humanoid")
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not (humanoid and hrp) then return end

        isPatrolling = true
        btnPatrol.Text = "◼️ СТОП"
        
        local startRotation = hrp.CFrame.Rotation
        camera.CameraType = Enum.CameraType.Scriptable
        cameraLockConnection = RunService.RenderStepped:Connect(function()
            if hrp and isPatrolling then
                camera.CFrame = CFrame.new(hrp.Position) * startRotation * CFrame.new(0, camHeight, 0) * CFrame.Angles(math.rad(-90), 0, 0)
            end
        end)

        currentWpIndex = 1
        patrolConnection = RunService.Heartbeat:Connect(function()
            if not isPatrolling then return end
            if currentWpIndex <= #waypoints then
                local targetPos = waypoints[currentWpIndex]
                humanoid:MoveTo(targetPos)
                if (Vector3.new(hrp.Position.X, 0, hrp.Position.Z) - Vector3.new(targetPos.X, 0, targetPos.Z)).Magnitude < 1.2 then 
                    currentWpIndex = currentWpIndex + 1
                end
            else
                currentWpIndex = 1 
            end
        end)
    end
end)

btnTimerPatrol.MouseButton1Click:Connect(function()
    if isTimedPatrolling then stopAllPatrols() else
        stopAllPatrols()
        local char = player.Character
        local hrp = char:FindFirstChild("HumanoidRootPart")
        local humanoid = char:FindFirstChild("Humanoid")
        if not (hrp and humanoid) then return end

        isTimedPatrolling = true
        btnTimerPatrol.Text = "◼️ СТОП"

        local fwd = hrp.CFrame.LookVector
        local back = -fwd
        local right = hrp.CFrame.RightVector
        
        camera.CameraType = Enum.CameraType.Scriptable
        local startRotation = hrp.CFrame.Rotation
        cameraLockConnection = RunService.RenderStepped:Connect(function()
            if hrp and isTimedPatrolling then
                camera.CFrame = CFrame.new(hrp.Position) * startRotation * CFrame.new(0, camHeight, 0) * CFrame.Angles(math.rad(-90), 0, 0)
            end
        end)

        patrolConnection = RunService.RenderStepped:Connect(function()
            if humanoid and isTimedPatrolling then humanoid:Move(currentDirection * 0.4, false) end
        end)

        task.spawn(function()
            while isTimedPatrolling do
                local walkTime = tonumber(sInp.Text) or 30
                currentDirection = fwd; waitPatrol(walkTime)
                if not isTimedPatrolling then break end
                
                currentDirection = right; waitPatrol(0.65)
                if not isTimedPatrolling then break end
                
                currentDirection = back; waitPatrol(walkTime)
                if not isTimedPatrolling then break end
                
                currentDirection = right; waitPatrol(0.65)
            end
            stopAllPatrols()
        end)
    end
end)

btnClicker.MouseButton1Click:Connect(function()
    isClicking = not isClicking
    btnClicker.Text = isClicking and "🖱️ КЛИК: 🔳" or "🖱️ КЛИК: 🔲"
    if isClicking then
        clickerConnection = RunService.RenderStepped:Connect(function()
            VirtualInputManager:SendMouseButtonEvent(camera.ViewportSize.X/2 - 50, camera.ViewportSize.Y/2, 0, true, game, 1)
            VirtualInputManager:SendMouseButtonEvent(camera.ViewportSize.X/2 - 50, camera.ViewportSize.Y/2, 0, false, game, 1)
        end)
    elseif clickerConnection then
        clickerConnection:Disconnect(); clickerConnection = nil
    end
end)

btnDestroy.MouseButton1Click:Connect(function()
    stopAllPatrols()
    isClicking = false
    if clickerConnection then clickerConnection:Disconnect() end
    for _, o in pairs(activeBarriers) do if o then o:Destroy() end end
    ScreenGui:Destroy()
end)

btnHide.MouseButton1Click:Connect(function() MasterFrame.Visible = not MasterFrame.Visible end)
