-- [[ CONFIGURATION - MATRIX X ]] --
getgenv().Matrix = {
    ["Main"] = {
        ["Prediction"] = 0.1,
        ["HorizontalPrediction"] = 0.165,
        ["VerticalPrediction"] = 0.00,
        ["AimPart"] = "Head",
        ["ToggleKey"] = "q",
        ["IgnoreKnocked"] = true
    },
    ["DistanceSettings"] = {
        ["Enabled"] = false,
        ["DistanceThreshold"] = 30,
        ["FarAimPart"] = "Head",
        ["FarHitboxSize"] = 7
    },
    ["Misc"] = {
        ["AutoAir"] = true,
        ["AirDelay"] = 0.15
    },
    ["Offsets"] = {
        ["Jump"] = 0.08,
        ["Fall"] = -0.5
    },
    ["Smoothness"] = 0.68
}

-- [[ GLOBAL SETTINGS ]] --
_G.HitboxEnabled = true 
_G.HitboxSize = 2      
_G.HitboxTransparency = 1 
_G.AutoReload = true

-- [[ SERVICES & CORE VARIABLES ]] --
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LocalPlayer = game.Players.LocalPlayer
local mouse = LocalPlayer:GetMouse()
local cam = workspace.CurrentCamera
local Players = game:GetService("Players")

-- [[ SCRIPT VARIABLES ]] --
local CamlockState = false
local AutoAirState = getgenv().Matrix.Misc.AutoAir
local speedEnabled = false
local internalActive = false 
local espEnabled = false
local enemy = nil
local lastMoveTime = tick() 

local speedValue = 175 
local UAnimation = Instance.new("Animation")
local activeScriptTrack = nil -- Biến lưu trữ animation duy nhất của script

game:GetService("TextChatService").ChatWindowConfiguration.Enabled = true

-- [[ HÀM XỬ LÝ ANIMATION - ĐÃ CẬP NHẬT ]] --
local function stopScriptAnimation()
    if activeScriptTrack then
        activeScriptTrack:Stop()
        activeScriptTrack = nil
    end
end

local function loadAnimation(id)
    local humanoid = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
    if humanoid then
        stopScriptAnimation() -- Chỉ dừng animation cũ của script này
        UAnimation.AnimationId = id
        activeScriptTrack = humanoid:LoadAnimation(UAnimation)
        activeScriptTrack:Play()
        return activeScriptTrack
    end
end

-- [[ HÀM TRANG BỊ WALLET ]] --
local function equipWallet()
    local char = LocalPlayer.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    local backpack = LocalPlayer:FindFirstChild("Backpack")
    
    if char and hum and backpack then
        local wallet = backpack:FindFirstChild("[Wallet]") or char:FindFirstChild("[Wallet]")
        if wallet then
            hum:EquipTool(wallet)
            task.wait(0.15)
            wallet.Parent = backpack
        end
    end
end

-- [[ HÀM TẮT SPEED - KHÔNG LÀM MẤT ANIMATION GAME ]] --
local function autoDisableSpeed()
    speedEnabled = false
    internalActive = false
    
    -- Chỉ dừng Animation của script (Crouch), không đụng vào Animation hệ thống (Carry/Súng)
    stopScriptAnimation()
    
    -- Reset WalkSpeed về mặc định
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
        LocalPlayer.Character:FindFirstChildOfClass("Humanoid").WalkSpeed = 16
    end

    local screenGui = LocalPlayer.PlayerGui:FindFirstChild("DaHoodPro")
    if screenGui and screenGui:FindFirstChild("Frame") and screenGui.Frame:FindFirstChild("TextButton") then
        local btn = screenGui.Frame.TextButton
        btn.Text = "Speed"
        btn.TextColor3 = Color3.fromRGB(20, 110, 255)
    end
end

-- [[ ESP LOGIC ]] --
local function CreateESP(p)
    if p == LocalPlayer then return end
    local function setup(char)
        local head = char:WaitForChild("Head", 10)
        local hum = char:WaitForChild("Humanoid", 10)
        if not head then return end
        
        local bgui = Instance.new("BillboardGui", head)
        bgui.Name = "ESP_Gui"
        bgui.Size = UDim2.new(0, 200, 0, 50)
        bgui.StudsOffset = Vector3.new(0, 3, 0)
        bgui.AlwaysOnTop = true 

        local tl = Instance.new("TextLabel", bgui)
        tl.BackgroundTransparency = 1
        tl.Size = UDim2.new(1, 0, 1, 0)
        tl.TextColor3 = Color3.new(1, 1, 1)
        tl.TextStrokeTransparency = 0
        tl.Font = Enum.Font.SourceSansBold
        tl.TextSize = 14

        local conn
        conn = RunService.RenderStepped:Connect(function()
            if not espEnabled or not char.Parent or not head.Parent then
                if bgui then bgui:Destroy() end
                conn:Disconnect()
                return
            end
            local myRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
            local targetRoot = char:FindFirstChild("HumanoidRootPart")
            if myRoot and targetRoot then
                local dist = (myRoot.Position - targetRoot.Position).Magnitude
                tl.Text = p.DisplayName .. "\n" .. math.floor(hum.Health) .. " HP [" .. math.floor(dist) .. "m]"
            end
        end)
    end
    p.CharacterAdded:Connect(setup)
    if p.Character then setup(p.Character) end
end

-- [[ SILENT AIM HOOK ]] --
local function IsValidTarget(Player)
    if not (Player and Player.Character) then return false end
    local char = Player.Character
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not (hum and hum.Health > 0) then return false end

    if getgenv().Matrix.Main.IgnoreKnocked then
        local be = char:FindFirstChild("BodyEffects")
        if be then
            local ko = be:FindFirstChild("K.O") or be:FindFirstChild("KO")
            if ko and ko.Value == true then return false end
        end
    end
    return true
end

local function FindNearestEnemy()
    local bestTarget, minScreenDist = nil, math.huge
    local center = Vector2.new(cam.ViewportSize.X / 2, cam.ViewportSize.Y / 2)
    local configPart = getgenv().Matrix.Main.AimPart

    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and IsValidTarget(p) then
            local part = p.Character:FindFirstChild(configPart)
            if part then
                local pos, onScreen = cam:WorldToViewportPoint(part.Position)
                if onScreen then
                    local dist = (Vector2.new(pos.X, pos.Y) - center).Magnitude
                    if dist < minScreenDist then
                        minScreenDist = dist
                        bestTarget = part
                    end
                end
            end
        end
    end
    return bestTarget
end

local handler = require(ReplicatedStorage.Modules.GunHandler)
local oldFunc = handler.getAim

handler.getAim = function(origin, maxDist)
    local currentEnemy = (CamlockState and IsValidTarget(Players:GetPlayerFromCharacter(enemy and enemy.Parent)) and enemy) or nil
    if not currentEnemy then
        local mousePos = Vector2.new(mouse.X, mouse.Y)
        local best, dist = nil, 1e9
        for _, v in pairs(Players:GetPlayers()) do
            if v ~= LocalPlayer and IsValidTarget(v) then
                local p = v.Character:FindFirstChild(getgenv().Matrix.Main.AimPart)
                if p then
                    local sPos, on = cam:WorldToScreenPoint(p.Position)
                    if on then
                        local d = (Vector2.new(sPos.X, sPos.Y) - mousePos).Magnitude
                        if d < dist then dist = d; best = p end
                    end
                end
            end
        end
        currentEnemy = best
    end
    if currentEnemy then
        local dir = (currentEnemy.Position - origin).Unit
        return dir, math.min((currentEnemy.Position - origin).Magnitude, maxDist or 200)
    end
    return oldFunc(origin, maxDist)
end

-- [[ SMOOTH LOCK LOOP ]] --
RunService.Heartbeat:Connect(function(delta)
    if CamlockState then
        if not IsValidTarget(Players:GetPlayerFromCharacter(enemy and enemy.Parent)) then
            enemy = FindNearestEnemy()
        end
        if enemy then
            local mainCfg = getgenv().Matrix.Main
            local distCfg = getgenv().Matrix.DistanceSettings
            local offsetCfg = getgenv().Matrix.Offsets
            
            local targetPart = enemy
            if distCfg.Enabled then
                local myHrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                if myHrp and (myHrp.Position - enemy.Position).Magnitude > distCfg.DistanceThreshold then
                    local farPart = enemy.Parent:FindFirstChild(distCfg.FarAimPart)
                    if farPart then targetPart = farPart end
                end
            end

            local targetPos = targetPart.Position + Vector3.new(mainCfg.HorizontalPrediction, mainCfg.VerticalPrediction, mainCfg.Prediction)
            local velY = enemy.Velocity.Y
            if velY > 1 then targetPos = targetPos + Vector3.new(0, offsetCfg.Jump, 0)
            elseif velY < -1 then targetPos = targetPos + Vector3.new(0, offsetCfg.Fall, 0) end
            local lerpSpeed = math.clamp(getgenv().Matrix.Smoothness * delta * 60, 0, 1)
            cam.CFrame = cam.CFrame:Lerp(CFrame.new(cam.CFrame.Position, targetPos), lerpSpeed)
        end
    end
    if AutoAirState and enemy and IsValidTarget(Players:GetPlayerFromCharacter(enemy.Parent)) then
        if enemy.Velocity.Y > 30 then
            task.wait(getgenv().Matrix.Misc.AirDelay)
            local char = LocalPlayer.Character
            local tool = char and char:FindFirstChildOfClass("Tool")
            if tool then tool:Activate() end
        end
    end
end)

-- [[ MATRIX UI ]] --
local MatrixUI = Instance.new("ScreenGui")
MatrixUI.Name = "MatrixOptimized"
MatrixUI.ResetOnSpawn = false
MatrixUI.Parent = (game:GetService("CoreGui") or LocalPlayer:WaitForChild("PlayerGui"))

local function styleButton(button, position, size, text)
    button.Size = size
    button.Position = position
    button.Text = text
    button.Font = Enum.Font.GothamBold
    button.TextSize = 16
    button.TextColor3 = Color3.fromRGB(255, 255, 255)
    button.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    button.Draggable = true
    button.Active = true
    local corner = Instance.new("UICorner", button)
    corner.CornerRadius = UDim.new(0, 12)
end

local CamlockButton = Instance.new("TextButton", MatrixUI)
styleButton(CamlockButton, UDim2.new(0.1, 0, 0.2, 0), UDim2.new(0, 250, 0, 50), "Toggle Matrix")
CamlockButton.MouseButton1Click:Connect(function()
    CamlockState = not CamlockState
    CamlockButton.Text = CamlockState and "Matrix (ON)" or "Matrix (OFF)"
    enemy = CamlockState and FindNearestEnemy() or nil
end)

-- [[ SPEED UI ]] --
local screenGui = LocalPlayer.PlayerGui:FindFirstChild("DaHoodPro") or Instance.new("ScreenGui", LocalPlayer.PlayerGui)
screenGui.Name = "DaHoodPro"
screenGui.ResetOnSpawn = false

local mainFrame = Instance.new("Frame", screenGui)
mainFrame.Size = UDim2.new(0, 160, 0, 54)
mainFrame.Position = UDim2.new(0, 30, 0, 100)
mainFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
mainFrame.Draggable = true; mainFrame.Active = true

local speedButton = Instance.new("TextButton", mainFrame)
speedButton.Size = UDim2.new(1, 0, 1, 0)
speedButton.Text = "Speed"
speedButton.BackgroundTransparency = 1 
speedButton.TextColor3 = Color3.fromRGB(20, 110, 255)
speedButton.Font = Enum.Font.SourceSansBold
speedButton.TextSize = 28

speedButton.MouseButton1Click:Connect(function()
    if not speedEnabled then
        speedButton.Text = "..."
        loadAnimation("rbxassetid://3189777795")
        task.wait(1.8)
        equipWallet()
        stopScriptAnimation()
        speedEnabled = true
        internalActive = true 
        speedButton.Text = "Tắt"
        speedButton.TextColor3 = Color3.fromRGB(255, 50, 50)
        lastMoveTime = tick() 
        
        task.wait(0.3) 
        if speedEnabled then
            loadAnimation("rbxassetid://3152394906")
        end
    else
        autoDisableSpeed()
    end
end)

-- [[ ESP BUTTON ]] --
local espFrame = Instance.new("Frame", screenGui)
espFrame.Size = UDim2.new(0, 160, 0, 54)
espFrame.AnchorPoint = Vector2.new(1, 0)
espFrame.Position = UDim2.new(1, -20, 0, 20)
espFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
espFrame.Draggable = true; espFrame.Active = true

local espButton = Instance.new("TextButton", espFrame)
espButton.Size = UDim2.new(1, 0, 1, 0)
espButton.Text = "ESP"
espButton.BackgroundTransparency = 1 
espButton.TextColor3 = Color3.fromRGB(20, 110, 255)
espButton.Font = Enum.Font.SourceSansBold
espButton.TextSize = 28

espButton.MouseButton1Click:Connect(function()
    espEnabled = not espEnabled
    espButton.Text = espEnabled and "Tắt" or "ESP"
    espButton.TextColor3 = espEnabled and Color3.fromRGB(255, 50, 50) or Color3.fromRGB(20, 110, 255)
    if espEnabled then
        for _, v in pairs(Players:GetPlayers()) do CreateESP(v) end
    else
        for _, v in pairs(Players:GetPlayers()) do
            if v.Character and v.Character:FindFirstChild("Head") and v.Character.Head:FindFirstChild("ESP_Gui") then
                v.Character.Head.ESP_Gui:Destroy()
            end
        end
    end
end)

-- [[ JUMP & LAND ANIMATION LOGIC ]] --
local function setupJumpLandListener(char)
    local hum = char:WaitForChild("Humanoid")
    hum.StateChanged:Connect(function(oldState, newState)
        if speedEnabled and internalActive then
            if newState == Enum.HumanoidStateType.Jumping or newState == Enum.HumanoidStateType.Freefall then
                stopScriptAnimation()
            elseif newState == Enum.HumanoidStateType.Landed or (oldState == Enum.HumanoidStateType.Freefall and newState == Enum.HumanoidStateType.Running) then
                task.wait(0.5) 
                if speedEnabled and internalActive then
                    loadAnimation("rbxassetid://3152394906")
                end
            end
        end
    end)
end

if LocalPlayer.Character then setupJumpLandListener(LocalPlayer.Character) end
LocalPlayer.CharacterAdded:Connect(setupJumpLandListener)

-- [[ SPEED & MOVEMENT LOGIC ]] --
RunService.RenderStepped:Connect(function()
    if speedEnabled and internalActive and LocalPlayer.Character then
        local hrp = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        local hum = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        
        if hrp and hum then
            if CamlockState and enemy and enemy.Parent then
                local dist = (hrp.Position - enemy.Position).Magnitude
                if dist < 30 then
                    autoDisableSpeed()
                    return 
                end
            end

            if hum.MoveDirection.Magnitude > 0 then
                lastMoveTime = tick() 
                hum.WalkSpeed = speedValue
            else
                if tick() - lastMoveTime >= 0.1 then
                    autoDisableSpeed()
                end
            end
        end
    end
end)

-- [[ HITBOX & DISTANCE LOGIC ]] --
task.spawn(function()
    while task.wait(1) do
        local myHrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        local distCfg = getgenv().Matrix.DistanceSettings

        for _, p in pairs(Players:GetPlayers()) do
            if p ~= LocalPlayer and p.Character then
                local hrp = p.Character:FindFirstChild("HumanoidRootPart")
                if hrp then
                    if _G.HitboxEnabled then
                        local currentHitboxSize = _G.HitboxSize
                        if distCfg.Enabled and myHrp then
                            local dist = (myHrp.Position - hrp.Position).Magnitude
                            if dist > distCfg.DistanceThreshold then
                                currentHitboxSize = distCfg.FarHitboxSize
                            end
                        end
                        
                        hrp.Size = Vector3.new(currentHitboxSize, currentHitboxSize, currentHitboxSize)
                        hrp.Transparency = _G.HitboxTransparency
                        hrp.BrickColor = BrickColor.new("Really blue")
                        hrp.CanCollide = false
                    else
                        hrp.Size = Vector3.new(2, 2, 1); hrp.Transparency = 1
                    end
                end
            end
        end
    end
end)

task.spawn(function()
    while task.wait(0.2) do
        if _G.AutoReload and LocalPlayer.Character then
            local tool = LocalPlayer.Character:FindFirstChildWhichIsA("Tool")
            if tool and tool:FindFirstChild("Ammo") and tool.Ammo.Value <= 0 then
                ReplicatedStorage.MainEvent:FireServer("Reload", tool)
                task.wait(1)
            end
        end
    end
end)

Players.PlayerAdded:Connect(CreateESP)

