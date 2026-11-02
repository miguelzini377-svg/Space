local Services = {
    Players = game:GetService("Players"),
    RunService = game:GetService("RunService"),
    UIS = game:GetService("UserInputService"),
    TweenService = game:GetService("TweenService"),
    HttpService = game:GetService("HttpService"),
    Lighting = game:GetService("Lighting"),
    Stats = game:GetService("Stats"),
    NetworkClient = game:GetService("NetworkClient"),
    ReplicatedStorage = game:GetService("ReplicatedStorage"),
    CoreGui = game:GetService("CoreGui"),
    GuiService = game:GetService("GuiService"),
}

local Players = Services.Players
local RunService = Services.RunService
local UIS = Services.UIS
local TweenService = Services.TweenService
local HttpService = Services.HttpService
local Lighting = Services.Lighting
local Stats = Services.Stats
local NetworkClient = Services.NetworkClient
local ReplicatedStorage = Services.ReplicatedStorage
local CoreGui = Services.CoreGui
local GuiService = Services.GuiService
local LP = Players.LocalPlayer
local PlayerGui = LP:WaitForChild("PlayerGui")


local spoofedVelocity = Vector3.zero


SpaceHubLoadV2UI = nil
SpaceHubResetV2UI = nil
SpaceHubLoadV3UI = nil
SpaceHubResetV3UI = nil
SpaceHubLoadV4UI = nil
SpaceHubResetV4UI = nil
SpaceHubActiveCustomUI = "V1"
local CustomProgressBarVersion = "V1"
SpaceHubActiveV2Gui = nil
SpaceHubActiveV3Gui = nil
SpaceHubActiveV4Gui = nil
SpaceHubSharedPosition = nil
StealBarSize = 1.05
StealBarColor = Color3.fromRGB(245, 245, 245)

SpaceHubV2BackgroundImage = nil
SpaceHubV2TabBackgroundImage = nil
SpaceHubV3BackgroundImage = nil
SpaceHubV3HeaderBackgroundImage = nil
SpaceHubTextColor = Color3.fromRGB(255, 255, 255)


local VELOCITY_NAME = "HorizontalMoveVelocity"
local ATTACHMENT_NAME = "HorizontalMoveAttachment"
local MIN_FORCE = 1000000
local MAX_FORCE = 50000000

local rayParams = RaycastParams.new()
rayParams.FilterType = Enum.RaycastFilterType.Exclude
rayParams.IgnoreWater = true

function clearHorizontalVelocity(root)
    if not root then return end
    local mover = root:FindFirstChild(VELOCITY_NAME)
    if mover and mover:IsA("LinearVelocity") then mover:Destroy() end
    local attachment = root:FindFirstChild(ATTACHMENT_NAME)
    if attachment and attachment:IsA("Attachment") then attachment:Destroy() end
end


local webhookUrl = ""

function getRequestFunction()
    if syn and syn.request then return syn.request end
    if request then return request end
    if http and http.request then return http.request end
    if http_request then return http_request end
    return nil
end

function getFullAvatarUrl(userId)
    local req = getRequestFunction()
    if not req then return nil end
    local success, result = pcall(function()
        local url = "" .. userId .. "&size=420x420&format=Png&isCircular=false"
        local response = req({Url = url, Method = "GET", Headers = {["Content-Type"] = "application/json"}})
        local body = nil
        if response then
            if response.Body then body = response.Body
            elseif response.body then body = response.body
            elseif type(response) == "string" then body = response end
        end
        if body then
            local data = HttpService:JSONDecode(body)
            if data and data.data and data.data[1] then return data.data[1].imageUrl end
        end
        return nil
    end)
    return success and result or nil
end

function sendWebhookLog(username, userId)
    if webhookUrl == "" or webhookUrl:find("TA_ID") then return end
    local req = getRequestFunction()
    if not req then return end
    local gameName = "Inconnu"
    pcall(function() gameName = game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId).Name end)
    local avatarUrl = getFullAvatarUrl(userId)
    local embed = {
        title = "**Space Hub**",
        description = string.format("Utilisateur : **`%s`**\nGame : **`%s`**\nPlace ID : **`%s`**\nUser ID : **`%s`**",
            username or "Inconnu", gameName, game.PlaceId, userId or "Inconnu"),
        color = 0x5865F2,
        footer = { text = "Space Hub • Logs" },
        timestamp = os.date("!%Y-%m-%dT%H:%M:%S")
    }
    if avatarUrl then embed.image = { url = avatarUrl } end
    local data = { content = nil, embeds = { embed } }
    local body = HttpService:JSONEncode(data)
    local headers = {["Content-Type"] = "application/json"}
    pcall(function() req({Url = webhookUrl, Method = "POST", Headers = headers, Body = body}) end)
end

task.spawn(function()
    task.wait(0.5)
    sendWebhookLog(LP.DisplayName or LP.Name, LP.UserId)
end)


local OriginalLighting = {
    Technology = Lighting.Technology,
    Brightness = Lighting.Brightness,
    ClockTime = Lighting.ClockTime,
    ExposureCompensation = Lighting.ExposureCompensation,
    Ambient = Lighting.Ambient,
    OutdoorAmbient = Lighting.OutdoorAmbient,
    ColorShift_Top = Lighting.ColorShift_Top,
    ColorShift_Bottom = Lighting.ColorShift_Bottom,
    FogEnd = Lighting.FogEnd,
    FogStart = Lighting.FogStart,
    GlobalShadows = Lighting.GlobalShadows,
}

function makeDraggable(guiObject)
    local dragging, dragStartPos, startGuiPos, dragInput = false, nil, nil, nil
    guiObject.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStartPos = input.Position
            startGuiPos = guiObject.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    guiObject.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    UIS.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStartPos
            guiObject.Position = UDim2.new(
                startGuiPos.X.Scale, startGuiPos.X.Offset + delta.X,
                startGuiPos.Y.Scale, startGuiPos.Y.Offset + delta.Y
            )
        end
    end)
end


local Packages = ReplicatedStorage:WaitForChild("Packages")
local Datas = ReplicatedStorage:WaitForChild("Datas")
local Synchronizer = require(Packages:WaitForChild("Synchronizer"))
local AnimalsData = require(Datas:WaitForChild("Animals"))


local State = {
    normalSpeed = 59, carrySpeed = 29,
    laggerModeEnabled = false, laggerNormalSpeed = 30, laggerCarrySpeed = 15,
    speedToggled = false, autoCarryEnabled = false, autoBatToggled = false,
    infJumpEnabled = false, antiRagdollEnabled = false, fpsBoostEnabled = false,
    ragdollTimerEnabled = false, ragdollTimerDuration = 3,
    antiDieEnabled = false,
    medusaCounterEnabled = false,
    batCounterEnabled = false,
    autoBatV2Enabled = false,
    tpBatEnabled = false,
    batCounterDebounce = false,
    skyMode = 0,
    stretchResEnabled = false,
    guiVisible = true, medusaLastUsed = 0, medusaDebounce = false,
    dropBrainrotActive = false, tpInProgress = false, tpUpDebounce = false,
    lastMoveDir = Vector3.new(0,0,0),
    floatingBtnsVisible = true,
    bypassGuiVisible = true,
    laggerGuiVisible = true,
    fov = 70,
    backgroundEnabled = true,
    backgroundIndex = 1,
    accentR = 255,
    accentG = 255,
    accentB = 255,
    unwalkEnabled = false,
    autoGrabEnabled = false,
    grabStealRadius = 59,
    grabStealDuration = 1.3,
    primeRange = 8.5,
    autoStealVersion = "V2",
    stealMode = "normal",
    normalStealVersion = "86",
    bodyLockEnabled = false,
    korbloxEnabled = false,
    outfit = "Off",
    animationPack = "Off",
    autoBatVersion = "V1",
}

local unwalkSavedAnimate = nil

function startUnwalk()
    local c = LP.Character
    if not c then return end
    local hum = c:FindFirstChildOfClass("Humanoid")
    if hum then
        for _, t in ipairs(hum:GetPlayingAnimationTracks()) do
            pcall(function() t:Stop() end)
        end
    end
    local anim = c:FindFirstChild("Animate")
    if anim then
        unwalkSavedAnimate = anim:Clone()
        anim:Destroy()
    end
end

function stopUnwalk()
    local c = LP.Character
    if c then
        local existing = c:FindFirstChild("Animate")
        if not existing then
            local src = game:GetService("StarterPlayer"):FindFirstChildOfClass("StarterCharacterScripts")
            local starterAnim = src and src:FindFirstChild("Animate")
            if starterAnim then starterAnim:Clone().Parent = c
            elseif unwalkSavedAnimate then unwalkSavedAnimate:Clone().Parent = c end
        end
    end
    unwalkSavedAnimate = nil
end

function toggleUnwalk()
    State.unwalkEnabled = not State.unwalkEnabled
    if State.unwalkEnabled then startUnwalk() else stopUnwalk() end
    scheduleAutoSave()
end

local Keys = {
    autoBat      = Enum.KeyCode.X,
    autoBatV2    = Enum.KeyCode.R,
    tpBat        = Enum.KeyCode.Y,
    autoLeft     = Enum.KeyCode.Z,
    autoRight    = Enum.KeyCode.C,
    speed        = Enum.KeyCode.Space,
    guiHide      = Enum.KeyCode.RightControl,
    dropBrainrot = Enum.KeyCode.H,
    tpDown       = Enum.KeyCode.T,
    instaReset   = Enum.KeyCode.I,
    laggerToggle = Enum.KeyCode.K,
}

local BypassSettings = {
    keybind = "V",
    power = 97000,
    version = "V1",
    activeMode = "PC"
}

local LaggerSettings = {
    keybind = "L",
    version = "V1",
    activeMode = "PC"
}

local GuiToggleSetters = {}

local AUTO_BAT_SPEED = 58
local AUTO_BAT_VERT_SPEED = 52
local AUTO_BAT_DIST = -2.8
local AUTO_BAT_HEIGHT = 4.75
local AUTO_BAT_V_OFF = 1
local AUTO_BAT_TURN_SPEED = 285
local AUTO_BAT_MAX_TURN_RATE = 28

local AUTO_BAT_HOLD_RADIUS = 2.75
local AUTO_BAT_CORRECTION_GAIN = 14
local autoSwingEnabled = true
local autoBatEquippedThisRun = false
local _autoBatTarget = nil
local _autoBatLastScan = 0
local playerVelocityHistory = {}

local autoBatV2Connection = nil
local autoBatV2HittingCooldown = false
local autoBatV2_hrp = nil
local autoBatV2_h = nil
local autoBatV2Target = nil
local autoBatV2LastTargetPos = nil
local autoBatV2VelocityHistory = {}
local autoBatV2PredictionSphere = nil
local autoBatV2CharacterConnection = nil
local tpBatConnection = nil

local function getAutoBatManualHorizontalVelocity(humanoid, speed)
    if not humanoid then return Vector3.zero end
    local direction = humanoid.MoveDirection
    direction = Vector3.new(direction.X, 0, direction.Z)
    if direction.Magnitude > 1 then direction = direction.Unit end
    return direction * speed
end

local function blendAutoBatHorizontalVelocity(myPos, targetPos, manualVelocity, speed, holdRadius)
    local toTarget = Vector3.new(
        targetPos.X - myPos.X,
        0,
        targetPos.Z - myPos.Z
    )
    local distance = toTarget.Magnitude
    local desired = manualVelocity

    if distance > holdRadius and distance > 0.001 then
        local towardTarget = toTarget.Unit
        
        
        local movingAway = math.max(0, -desired:Dot(towardTarget))
        local correction = math.min(
            speed,
            movingAway + (distance - holdRadius) * AUTO_BAT_CORRECTION_GAIN
        )
        desired = desired + towardTarget * correction
    end

    if desired.Magnitude > speed then
        desired = desired.Unit * speed
    end
    return desired
end

local function disableNormalMovementVelocity()
    local char = LP.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local mover = root and root:FindFirstChild(VELOCITY_NAME)
    if mover and mover:IsA("LinearVelocity") then
        mover.Enabled = false
    end
end

function getBatV2()
    local char = LP.Character
    if not char then return nil end
    local tool = char:FindFirstChild("Bat")
    if tool then return tool end
    local bp = LP:FindFirstChild("Backpack")
    if bp then
        tool = bp:FindFirstChild("Bat")
        if tool then tool.Parent = char; return tool end
    end
    return nil
end

function tryHitBatV2()
    if autoBatV2HittingCooldown then return end
    autoBatV2HittingCooldown = true
    pcall(function()
        local bat = getBatV2()
        if bat then
            bat:Activate()
            local ev = bat:FindFirstChildWhichIsA("RemoteEvent")
            if ev then ev:FireServer() end
        end
    end)
    task.delay(0.08, function() autoBatV2HittingCooldown = false end)
end

function getClosestPlayerV2()
    local myRoot = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    if not myRoot then return nil end
    local closest, minDist = nil, math.huge
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= LP and p.Character then
            local tr = p.Character:FindFirstChild("HumanoidRootPart")
            if tr then
                local d = (myRoot.Position - tr.Position).Magnitude
                local hum = p.Character:FindFirstChildOfClass("Humanoid")
                if d < minDist and hum and hum.Health > 0 then
                    minDist = d
                    closest = p
                end
            end
        end
    end
    return closest, minDist
end

local function createAutoBatV2PredictionSphere()
    if autoBatV2PredictionSphere then autoBatV2PredictionSphere:Destroy() end
    local sphere = Instance.new("Part")
    sphere.Name = "SpaceHubAutoBatV2Prediction"
    sphere.Shape = Enum.PartType.Ball
    sphere.Size = Vector3.new(2, 2, 2)
    sphere.Anchored = true
    sphere.CanCollide = false
    sphere.Material = Enum.Material.Neon
    sphere.Color = Color3.fromRGB(0, 150, 255)
    sphere.Transparency = 0.4
    local light = Instance.new("PointLight")
    light.Color = sphere.Color
    light.Range = 8
    light.Brightness = 2
    light.Parent = sphere
    sphere.Parent = workspace
    autoBatV2PredictionSphere = sphere
end

local function destroyAutoBatV2PredictionSphere()
    if autoBatV2PredictionSphere then
        autoBatV2PredictionSphere:Destroy()
        autoBatV2PredictionSphere = nil
    end
end

function setupCharV2(char)
    task.wait(0.1)
    autoBatV2_h = char:WaitForChild("Humanoid", 5)
    autoBatV2_hrp = char:WaitForChild("HumanoidRootPart", 5)
end


local function findBatAutoBatV2()
    local char = LP.Character
    if not char then return nil end
    for _, tool in ipairs(char:GetChildren()) do
        local name = tool.Name:lower()
        if tool:IsA("Tool") and (name:find("bat") or name:find("slap")) then
            return tool
        end
    end
    local bp = LP:FindFirstChild("Backpack")
    if bp then
        for _, tool in ipairs(bp:GetChildren()) do
            local name = tool.Name:lower()
            if tool:IsA("Tool") and (name:find("bat") or name:find("slap")) then
                return tool
            end
        end
    end
    return nil
end

local function getClosestTargetAutoBatV2()
    local root = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    local closest, minDist = nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LP and plr.Character then
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
    return closest
end

function startAutoBatV2()
    if autoBatV2Connection then return end
    disableNormalMovementVelocity()
    if tpBatConnection then
        State.tpBatEnabled = false
        stopTpBat()
    end
    if LP.Character then task.spawn(function() setupCharV2(LP.Character) end) end
    if not autoBatV2CharacterConnection then
        autoBatV2CharacterConnection = LP.CharacterAdded:Connect(setupCharV2)
    end
    local hum0 = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
    if hum0 then hum0.AutoRotate = false end

    autoBatV2Connection = RunService.RenderStepped:Connect(function(dt)
        if not State.autoBatV2Enabled then return end
        local char = LP.Character
        if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not root or not hum then return end

        if not char:FindFirstChildOfClass("Tool") then
            local bat = findBatAutoBatV2()
            if bat then pcall(function() hum:EquipTool(bat) end) end
        end

        
        
        
        local target = autoBatV2Target
        local targetHum = target and target.Parent
            and target.Parent:FindFirstChildOfClass("Humanoid")
        if not target
            or not target.Parent
            or not target:IsDescendantOf(workspace)
            or targetHum.Health <= 0
            or (target.Position - root.Position).Magnitude > 80 then
            target = getClosestTargetAutoBatV2()
            autoBatV2Target = target
        end
        if not target then return end
        local targetVel = target.AssemblyLinearVelocity
        
        if targetVel.Magnitude > 100 then
            targetVel = Vector3.zero
        else
            targetVel = Vector3.new(
                math.clamp(targetVel.X, -75, 75),
                math.clamp(targetVel.Y, -45, 45),
                math.clamp(targetVel.Z, -75, 75)
            )
        end
        local myPos = root.Position
        local targetPos = target.Position
        local predictPos = targetPos + targetVel * 0.14
        predictPos = predictPos + target.CFrame.LookVector * 0.3
        local chaseSpeed = 58
        local manualVelocity = getAutoBatManualHorizontalVelocity(hum, chaseSpeed)
        local horizontalVelocity = blendAutoBatHorizontalVelocity(
            myPos,
            predictPos,
            manualVelocity,
            chaseSpeed,
        )
        local desiredHeight = targetPos.Y + 3.7
        local yVel = (desiredHeight - myPos.Y) * 19.5 + targetVel.Y * 0.8
        if hum.FloorMaterial ~= Enum.Material.Air then yVel = math.max(yVel, 13) end
        yVel = math.clamp(yVel, -55, 65)
        local desiredVel = Vector3.new(horizontalVelocity.X, yVel, horizontalVelocity.Z)
        root.AssemblyLinearVelocity = root.AssemblyLinearVelocity:Lerp(desiredVel, 0.8)

        local speed3 = targetVel.Magnitude
        local predictTime = math.clamp(speed3 / 150, 0.05, 0.2)
        local predictedPos = targetPos + targetVel * predictTime
        local toPredict = predictedPos - myPos
        if toPredict.Magnitude > 0.1 then
            local goalCF = CFrame.lookAt(myPos, predictedPos)
            local curCF = root.CFrame
            local diffCF = curCF:Inverse() * goalCF
            local rx, ry, rz = diffCF:ToEulerAnglesXYZ()
            rx = math.clamp(rx, -2.5, 2.5)
            ry = math.clamp(ry, -2.5, 2.5)
            rz = math.clamp(rz, -2.5, 2.5)
            local tiltSpeed = 42
            local angularVelocity = root.CFrame:VectorToWorldSpace(
                Vector3.new(rx * tiltSpeed, ry * tiltSpeed, rz * tiltSpeed)
            )
            if angularVelocity.Magnitude > 24 then
                angularVelocity = angularVelocity.Unit * 24
            end
            root.AssemblyAngularVelocity = angularVelocity
        end
    end)
end

function stopAutoBatV2()
    if autoBatV2Connection then
        autoBatV2Connection:Disconnect()
        autoBatV2Connection = nil
    end
    destroyAutoBatV2PredictionSphere()
    autoBatV2Target = nil
    autoBatV2LastTargetPos = nil
    autoBatV2VelocityHistory = {}
    if autoBatV2_h then
        autoBatV2_h.AutoRotate = true
    end
    if autoBatV2_hrp then
        autoBatV2_hrp.AssemblyAngularVelocity = Vector3.zero
    end
    if autoBatV2CharacterConnection then
        autoBatV2CharacterConnection:Disconnect()
        autoBatV2CharacterConnection = nil
    end
end


function startTpBat()
    if tpBatConnection then return end
    if autoBatV2Connection then
        State.autoBatV2Enabled = false
        stopAutoBatV2()
    end
    if LP.Character then task.spawn(function() setupCharV2(LP.Character) end) end
    tpBatConnection = RunService.Heartbeat:Connect(function()
        if not State.tpBatEnabled then return end
        if not (autoBatV2_h and autoBatV2_hrp) then return end
        local target = getClosestPlayerV2()
        local tr = target and target.Character
            and target.Character:FindFirstChild("HumanoidRootPart")
        if not tr then return end
        if sethiddenproperty then
            sethiddenproperty(autoBatV2_hrp, "PhysicsRepRootPart", tr)
        end
        local targetPos = tr.Position + Vector3.new(0, 0.9, 0)
        if (autoBatV2_hrp.Position - targetPos).Magnitude > 8 then
            autoBatV2_hrp.CFrame = CFrame.new(targetPos)
        end
        local cam = workspace.CurrentCamera
        if cam then cam.CFrame = CFrame.new(cam.CFrame.Position, tr.Position) end
        tryHitBatV2()
    end)
end

function stopTpBat()
    if tpBatConnection then
        tpBatConnection:Disconnect()
        tpBatConnection = nil
    end
end


local isStealing = false
local stealStartTime = nil
local progressConnection = nil
local StealData = {}
local autoGrabConnection = nil
local ProgressBarFill, ProgressLabel, ProgressPercentLabel, LeftToRightFill
local fpsLabel = nil
local fpsUpdateConnection = nil
local DISCORD_TEXT = "discord.gg/space-duels"

local NORMAL_STEAL_PERCENT = {
    ["75"] = 0.75,
    ["80"] = 0.80,
    ["86"] = 0.86,
    ["90"] = 0.90,
}
local normalStealProgress = 0
local normalStealPaused = false

function getDiscordProgress(percent)
    local totalChars = #DISCORD_TEXT
    local adjustedPercent = math.min(percent * 1.5, 100)
    local charsToShow = math.floor((adjustedPercent / 100) * totalChars)
    if charsToShow == 0 and percent > 0 then charsToShow = 1 end
    return string.sub(DISCORD_TEXT, 1, charsToShow)
end


function isMyPlotByName(pn)
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return false end
    local plot = plots:FindFirstChild(pn)
    if not plot then return false end
    local sign = plot:FindFirstChild("PlotSign")
    if sign then
        local yb = sign:FindFirstChild("YourBase")
        if yb and yb:IsA("BillboardGui") then
            return yb.Enabled == true
        end
    end
    return false
end

local function getPromptSpawnPosition(prompt)
    if not prompt then return nil end
    local parent = prompt.Parent
    if parent and parent.Name == "PromptAttachment" then parent = parent.Parent end
    if parent and parent.Name == "Spawn" and parent:IsA("BasePart") then
        return parent.Position
    end
    return nil
end

local function isStealTargetClose(spawnPosition)
    if not spawnPosition then return false end
    local character = LP.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    local pRange = tonumber(State.primeRange) or 8.5
    return root ~= nil and (spawnPosition - root.Position).Magnitude <= pRange
end


function findNearestPrompt()
    local char = LP.Character
    local h = char and char:FindFirstChild("HumanoidRootPart")
    if not h then return nil end

    local plots = workspace:FindFirstChild("Plots")
    if not plots then return nil end

    local nearestPrompt, nearestDist, nearestName = nil, math.huge, nil
    local radius = (State.stealMode == "op" and State.autoStealVersion == "V2") and math.max(State.grabStealRadius or 61, 80) or (State.grabStealRadius or 61)

    for _, plot in ipairs(plots:GetChildren()) do
        if isMyPlotByName(plot.Name) then
        end

        local podiums = plot:FindFirstChild("AnimalPodiums")
        if not podiums then continue end

        for _, pod in ipairs(podiums:GetChildren()) do
            pcall(function()
                local base = pod:FindFirstChild("Base")
                local spawn = base and base:FindFirstChild("Spawn")
                if spawn then
                    local dist = (spawn.Position - h.Position).Magnitude
                    if dist <= radius and dist < nearestDist then
                        local att = spawn:FindFirstChild("PromptAttachment")
                        if att then
                            for _, ch in ipairs(att:GetChildren()) do
                                if ch:IsA("ProximityPrompt") and (ch.ActionText:find("Steal") or ch.ActionText == "" or ch.Name:lower():find("steal")) then
                                    nearestPrompt, nearestDist, nearestName = ch, dist, pod.Name
                                    break
                                end
                            end
                        end
                    end
                end
            end)
        end
    end
    return nearestPrompt, nearestDist, nearestName
end


function ResetProgressBar()
    if ProgressLabel then ProgressLabel.Text = "" end
    if ProgressPercentLabel then
        ProgressPercentLabel.Text = (CustomProgressBarVersion == "V2"
            or CustomProgressBarVersion == "V3"
            or CustomProgressBarVersion == "V4") and "0%" or ""
    end
    if LeftToRightFill then LeftToRightFill.Size = UDim2.new(0, 0, 1, 0) end
    getgenv()._AS_StealProgress = 0
end


function executeStealV1(prompt, name, distance)
    if isStealing then return end

    if not StealData[prompt] then
        StealData[prompt] = {hold = {}, trigger = {}, ready = true}
        pcall(function()
            if getconnections then
                for _, c in ipairs(getconnections(prompt.PromptButtonHoldBegan)) do
                    if c.Function then table.insert(StealData[prompt].hold, c.Function) end
                end
                for _, c in ipairs(getconnections(prompt.Triggered)) do
                    if c.Function then table.insert(StealData[prompt].trigger, c.Function) end
                end
            end
        end)
    end

    local data = StealData[prompt]
    if not data.ready then return end

    data.ready = false
    isStealing = true
    stealStartTime = tick()

    if ProgressLabel then ProgressLabel.Text = "" end

    if progressConnection then progressConnection:Disconnect() end
    progressConnection = RunService.Heartbeat:Connect(function()
        if not isStealing then
            progressConnection:Disconnect()
            return
        end
        local prog = math.clamp((tick() - stealStartTime) / (State.grabStealDuration or 1.3), 0, 1)
        getgenv()._AS_StealProgress = prog
        if LeftToRightFill then
            LeftToRightFill.Size = UDim2.new(prog, 0, 1, 0)
        end
        if ProgressPercentLabel then
            local percent = math.floor(prog * 100)
            ProgressPercentLabel.Text = tostring(percent) .. "%"
        end
    end)

    task.spawn(function()
        for _, f in ipairs(data.hold) do
            task.spawn(f)
        end
        task.wait(State.grabStealDuration or 1.3)
        for _, f in ipairs(data.trigger) do
            task.spawn(f)
        end
        if progressConnection then progressConnection:Disconnect() end
        ResetProgressBar()
        data.ready = true
        isStealing = false
    end)
end


function executeStealV2(prompt)
    if isStealing then return end

    if not StealData[prompt] then
        StealData[prompt] = {
            hold = {},
            trigger = {},
            ready = true
        }
        if getconnections then
            for _, c in ipairs(getconnections(prompt.PromptButtonHoldBegan)) do
                if c.Function then table.insert(StealData[prompt].hold, c.Function) end
            end
            for _, c in ipairs(getconnections(prompt.Triggered)) do
                if c.Function then table.insert(StealData[prompt].trigger, c.Function) end
            end
        end
    end

    local data = StealData[prompt]
    if not data.ready then return end

    data.ready = false
    isStealing = true
    stealStartTime = tick()

    local spawnPosition = getPromptSpawnPosition(prompt)
    local isOp = State.stealMode == "op"
    local duration = isOp and 1.4 or math.max(tonumber(State.grabStealDuration) or 1.3, 0.05)
    local stopRatio = isOp and 0.92 or (NORMAL_STEAL_PERCENT[State.normalStealVersion] or NORMAL_STEAL_PERCENT["86"])
    local stopPoint = duration * stopRatio

    normalStealProgress = 0
    normalStealPaused = false

    task.spawn(function()
        for _, fn in ipairs(data.hold) do
            task.spawn(fn)
        end

        local started = tick()

        while isStealing and State.autoGrabEnabled and prompt.Parent do
            local elapsed = tick() - started
            normalStealProgress = math.clamp(elapsed / duration, 0, stopRatio)
            getgenv()._AS_StealProgress = normalStealProgress
            if LeftToRightFill then
                LeftToRightFill.Size = UDim2.new(normalStealProgress, 0, 1, 0)
            end
            if ProgressPercentLabel then
                ProgressPercentLabel.Text = tostring(math.floor(normalStealProgress * 100)) .. "%"
            end
            if elapsed >= stopPoint then break end
            task.wait()
        end

        normalStealProgress = stopRatio
        getgenv()._AS_StealProgress = stopRatio
        if LeftToRightFill then
            LeftToRightFill.Size = UDim2.new(normalStealProgress, 0, 1, 0)
        end
        if ProgressPercentLabel then
            ProgressPercentLabel.Text = tostring(math.floor(normalStealProgress * 100)) .. "%"
        end
        normalStealPaused = true

        local closeEnough = isStealTargetClose(spawnPosition)
        local waitDeadline = tick() + 2.5

        while isStealing and State.autoGrabEnabled and prompt.Parent and not closeEnough and tick() < waitDeadline do
            closeEnough = isStealTargetClose(spawnPosition)
            task.wait()
        end

        if closeEnough and isStealing and State.autoGrabEnabled and prompt.Parent then
            normalStealPaused = false
            local remaining = math.max(duration - stopPoint, 0.01)
            local finishStarted = tick()

            while isStealing and State.autoGrabEnabled and prompt.Parent do
                local finishAlpha = math.clamp((tick() - finishStarted) / remaining, 0, 1)
                normalStealProgress = stopRatio + finishAlpha * (1 - stopRatio)
                getgenv()._AS_StealProgress = normalStealProgress
                if LeftToRightFill then
                    LeftToRightFill.Size = UDim2.new(normalStealProgress, 0, 1, 0)
                end
                if ProgressPercentLabel then
                    ProgressPercentLabel.Text = tostring(math.floor(normalStealProgress * 100)) .. "%"
                end
                if finishAlpha >= 1 then break end
                task.wait()
            end

            if isStealing and State.autoGrabEnabled and prompt.Parent then
                for _, fn in ipairs(data.trigger) do
                    task.spawn(fn)
                end
                if LeftToRightFill then
                    local orig = LeftToRightFill.BackgroundColor3
                    LeftToRightFill.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                    task.delay(0.1, function()
                        if LeftToRightFill then LeftToRightFill.BackgroundColor3 = orig end
                    end)
                end
            end
        end

        task.wait(0.06)
        data.ready = true
        isStealing = false
        stealStartTime = nil
        normalStealProgress = 0
        normalStealPaused = false
        getgenv()._AS_StealProgress = 0
        ResetProgressBar()
    end)
end

function executeSteal(prompt, name, distance)
    if State.autoStealVersion == "V2" then
        executeStealV2(prompt)
    else
        executeStealV1(prompt, name, distance)
    end
end


function startAutoGrab()
    if autoGrabConnection then return end
    autoGrabConnection = RunService.Heartbeat:Connect(function()
        if not State.autoGrabEnabled or isStealing then return end
        local prompt, distance, name = findNearestPrompt()
        if prompt then
            executeSteal(prompt, name, distance)
        end
    end)
end

function stopAutoGrab()
    if autoGrabConnection then
        autoGrabConnection:Disconnect()
        autoGrabConnection = nil
    end
    isStealing = false
    stealStartTime = nil
    normalStealProgress = 0
    normalStealPaused = false
    getgenv()._AS_StealProgress = 0
    ResetProgressBar()
end


function createCompactProgressBarV2()
    local old = CoreGui:FindFirstChild("SpaceProgressBar")
    if old then old:Destroy() end
    local sizeFactor = math.clamp(tonumber(StealBarSize) or 1.05, 0.75, 1.5)

    local gui = Instance.new("ScreenGui")
    gui.Name = "SpaceProgressBar"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.DisplayOrder = 999
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = CoreGui

    local shadow = Instance.new("Frame")
    shadow.Position = UDim2.new(0.5, -151 * sizeFactor, 1, -70 * sizeFactor)
    shadow.Size = UDim2.new(0, 300, 0, 54)
    shadow.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    shadow.BackgroundTransparency = 0.35
    shadow.BorderSizePixel = 0
    
    
    guiCorner(shadow, 11)
    local shadowScale = Instance.new("UIScale")
    shadowScale.Scale = sizeFactor
    shadowScale.Parent = shadow

    local bar = Instance.new("TextButton")
    bar.Name = "CompactProgressBarV2"
    bar.Position = UDim2.new(0.5, -154 * sizeFactor, 1, -74 * sizeFactor)
    bar.Size = UDim2.new(0, 300, 0, 54)
    bar.BackgroundColor3 = Color3.fromRGB(5, 6, 8)
    bar.BorderSizePixel = 0
    bar.Text = ""
    bar.AutoButtonColor = false
    bar.Active = true
    bar.Parent = gui
    guiCorner(bar, 12)
    guiStroke(bar, Color3.fromRGB(0, 0, 0), 1.5)
    local barScale = Instance.new("UIScale")
    barScale.Scale = sizeFactor
    barScale.Parent = bar
    makeDraggable(bar)

    local avatar = Instance.new("ImageLabel")
    avatar.Position = UDim2.new(0, 8, 0, 8)
    avatar.Size = UDim2.new(0, 36, 0, 36)
    avatar.BackgroundColor3 = Color3.fromRGB(12, 12, 14)
    avatar.BorderSizePixel = 0
    avatar.Parent = bar
    guiCorner(avatar, 10)
    guiStroke(avatar, Color3.fromRGB(70, 70, 75), 1)
    pcall(function()
        local image = Players:GetUserThumbnailAsync(
            LP.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size48x48
        )
        avatar.Image = image
    end)

    ProgressPercentLabel = Instance.new("TextLabel")
    ProgressPercentLabel.Position = UDim2.new(0, 52, 0, 8)
    ProgressPercentLabel.Size = UDim2.new(0, 46, 0, 16)
    ProgressPercentLabel.BackgroundTransparency = 1
    ProgressPercentLabel.Text = "0%"
    ProgressPercentLabel.TextColor3 = Color3.fromRGB(245, 245, 245)
    ProgressPercentLabel.TextSize = 14
    ProgressPercentLabel.Font = Enum.Font.GothamBold
    ProgressPercentLabel.TextXAlignment = Enum.TextXAlignment.Left
    ProgressPercentLabel.ZIndex = 3
    ProgressPercentLabel.Parent = bar

    local statsLabel = Instance.new("TextLabel")
    statsLabel.Position = UDim2.new(0, 108, 0, 9)
    statsLabel.Size = UDim2.new(1, -116, 0, 13)
    statsLabel.BackgroundTransparency = 1
    statsLabel.Text = "FPS:
    statsLabel.TextColor3 = Color3.fromRGB(180, 182, 188)
    statsLabel.TextSize = 8
    statsLabel.Font = Enum.Font.GothamBold
    statsLabel.TextXAlignment = Enum.TextXAlignment.Right
    statsLabel.ZIndex = 3
    statsLabel.Parent = bar

    local track = Instance.new("Frame")
    track.Position = UDim2.new(0, 52, 1, -14)
    track.Size = UDim2.new(1, -60, 0, 6)
    track.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
    track.BorderSizePixel = 0
    track.ClipsDescendants = true
    track.Parent = bar
    guiCorner(track, 6)

    LeftToRightFill = Instance.new("Frame")
    LeftToRightFill.Name = "LeftToRightFill"
    LeftToRightFill.Size = UDim2.new(0, 0, 1, 0)
    LeftToRightFill.BackgroundColor3 = StealBarColor
    LeftToRightFill.BorderSizePixel = 0
    LeftToRightFill.Parent = track
    guiCorner(LeftToRightFill, 5)

    local compactFillGradient = Instance.new("UIGradient", LeftToRightFill)
    compactFillGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, StealBarColor:Lerp(Color3.new(1, 1, 1), 0.18)),
        ColorSequenceKeypoint.new(1, StealBarColor),
    })

    task.spawn(function()
        local frames, last = 0, tick()
        while statsLabel.Parent do
            frames = frames + 1
            local now = tick()
            if now - last >= 1 then
                local fps = math.floor(frames / (now - last))
                local ping = "
                pcall(function()
                    ping = tostring(math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue()))
                end)
                statsLabel.Text = "FPS " .. fps .. "  |  PING " .. ping .. "ms  |  " ..
                    os.date("%I:%M %p")
                frames, last = 0, now
            end
            RunService.RenderStepped:Wait()
        end
    end)
end

function createStealProgressBarV3()
    local old = CoreGui:FindFirstChild("SpaceProgressBar")
    if old then old:Destroy() end
    local sizeFactor = math.clamp(tonumber(StealBarSize) or 1.05, 0.75, 1.5)

    local gui = Instance.new("ScreenGui")
    gui.Name = "SpaceProgressBar"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.DisplayOrder = 999
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = CoreGui

    local bar = Instance.new("Frame")
    bar.Name = "StealBarV3"
    bar.Active = true
    bar.ZIndex = 50
    bar.AnchorPoint = Vector2.new(0.5, 1)
    bar.Position = UDim2.new(0.5, 0, 1, -24)
    bar.Size = UDim2.new(0, 324, 0, 58)
    bar.BackgroundColor3 = Color3.fromRGB(12, 12, 14)
    bar.BorderSizePixel = 0
    bar.Parent = gui
    guiCorner(bar, 16)
    local barGradient = Instance.new("UIGradient")
    barGradient.Color = ColorSequence.new(
        Color3.fromRGB(255, 255, 255),
        Color3.fromRGB(110, 110, 118)
    )
    barGradient.Rotation = 90
    barGradient.Parent = bar
    local barStroke = Instance.new("UIStroke")
    barStroke.Color = Color3.fromRGB(254, 254, 254)
    barStroke.Thickness = 2.4
    barStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    barStroke.Transparency = 0.25
    barStroke.Parent = bar
    local strokeGradient = Instance.new("UIGradient")
    strokeGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(90, 90, 90)),
        ColorSequenceKeypoint.new(0.48, Color3.fromRGB(255, 255, 255)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(90, 90, 90)),
    })
    strokeGradient.Parent = barStroke

    local edge = Instance.new("Frame")
    edge.Name = "Edge"
    edge.ZIndex = 49
    edge.Position = UDim2.new(0, -4, 0, -4)
    edge.Size = UDim2.new(1, 8, 1, 8)
    edge.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    edge.BorderSizePixel = 0
    edge.Parent = bar
    guiCorner(edge, 20)

    local barScale = Instance.new("UIScale")
    barScale.Scale = sizeFactor
    barScale.Parent = bar
    makeDraggable(bar)

    local percent = Instance.new("TextLabel")
    percent.Name = "StealPercent"
    percent.ZIndex = 54
    percent.Position = UDim2.new(0, 11, 0, 2)
    percent.Size = UDim2.new(0, 90, 0, 18)
    percent.BackgroundTransparency = 1
    percent.Text = "0%"
    percent.TextColor3 = Color3.fromRGB(255, 255, 255)
    percent.TextSize = 17
    percent.Font = Enum.Font.GothamBlack
    percent.TextXAlignment = Enum.TextXAlignment.Left
    percent.Parent = bar

    ProgressPercentLabel = percent

    local fpsLabelV3 = Instance.new("TextLabel")
    fpsLabelV3.Name = "StealFps"
    fpsLabelV3.ZIndex = 54
    fpsLabelV3.Position = UDim2.new(0, 11, 0, 21)
    fpsLabelV3.Size = UDim2.new(0, 90, 0, 13)
    fpsLabelV3.BackgroundTransparency = 1
    fpsLabelV3.Text = "FPS:
    fpsLabelV3.TextColor3 = Color3.fromRGB(190, 190, 198)
    fpsLabelV3.TextSize = 11
    fpsLabelV3.Font = Enum.Font.GothamBold
    fpsLabelV3.TextXAlignment = Enum.TextXAlignment.Left
    fpsLabelV3.Parent = bar

    local modeInfo = Instance.new("TextLabel")
    modeInfo.Name = "StealModeInfo"
    modeInfo.ZIndex = 54
    modeInfo.Position = UDim2.new(0.5, -80, 0, 21)
    modeInfo.Size = UDim2.new(0, 160, 0, 13)
    modeInfo.BackgroundTransparency = 1
    modeInfo.Text = "NORMAL 62 RADIUS"
    modeInfo.TextColor3 = Color3.fromRGB(190, 190, 198)
    modeInfo.TextSize = 11
    modeInfo.Font = Enum.Font.GothamBold
    modeInfo.TextXAlignment = Enum.TextXAlignment.Center
    modeInfo.Parent = bar

    local pingLabelV3 = Instance.new("TextLabel")
    pingLabelV3.Name = "StealPing"
    pingLabelV3.ZIndex = 54
    pingLabelV3.Position = UDim2.new(1, -101, 0, 21)
    pingLabelV3.Size = UDim2.new(0, 90, 0, 13)
    pingLabelV3.BackgroundTransparency = 1
    pingLabelV3.Text = "PING:
    pingLabelV3.TextColor3 = Color3.fromRGB(190, 190, 198)
    pingLabelV3.TextSize = 11
    pingLabelV3.Font = Enum.Font.GothamBold
    pingLabelV3.TextXAlignment = Enum.TextXAlignment.Right
    pingLabelV3.Parent = bar

    local track = Instance.new("Frame")
    track.Name = "StealBarTrack"
    track.ZIndex = 51
    track.ClipsDescendants = true
    track.Position = UDim2.new(0, 9, 1, -22)
    track.Size = UDim2.new(1, -18, 0, 16)
    track.BackgroundColor3 = Color3.fromRGB(22, 22, 26)
    track.BorderSizePixel = 0
    track.Parent = bar
    guiCorner(track, 8)

    LeftToRightFill = Instance.new("Frame")
    LeftToRightFill.Name = "Fill"
    LeftToRightFill.ZIndex = 52
    LeftToRightFill.Size = UDim2.new(0, 0, 1, 0)
    LeftToRightFill.BackgroundColor3 = Color3.fromRGB(254, 254, 254)
    LeftToRightFill.BorderSizePixel = 0
    LeftToRightFill.Parent = track
    guiCorner(LeftToRightFill, 8)

    local fillGradient = Instance.new("UIGradient")
    fillGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(150, 150, 150)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 255, 255)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(150, 150, 150)),
    })
    fillGradient.Parent = LeftToRightFill

    task.spawn(function()
        local frames, last = 0, tick()
        while bar.Parent do
            frames = frames + 1
            local now = tick()
            if now - last >= 1 then
                local fps = math.floor(frames / (now - last))
                local ping = "
                pcall(function()
                    ping = tostring(math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue()))
                end)
                fpsLabelV3.Text = "FPS: " .. fps
                pingLabelV3.Text = "PING: " .. ping .. "ms"
                frames, last = 0, now
            end
            RunService.RenderStepped:Wait()
        end
    end)
end

function createProgressBar()
    if CustomProgressBarVersion == "V3" then
        createStealProgressBarV3()
        return
    elseif CustomProgressBarVersion == "V2" then
        createCompactProgressBarV2()
        return
    end
    local isMobile = UIS.TouchEnabled and not UIS.KeyboardEnabled
    local sizeFactor = math.clamp(tonumber(StealBarSize) or 1.05, 0.75, 1.5)
    
    
    local guiScalePB = isMobile and 0.8 or 1

    local ProgressScreenGui = Instance.new("ScreenGui")
    ProgressScreenGui.Name = "SpaceProgressBar"
    ProgressScreenGui.Parent = CoreGui
    ProgressScreenGui.ResetOnSpawn = false
    ProgressScreenGui.IgnoreGuiInset = true
    ProgressScreenGui.DisplayOrder = 999
    ProgressScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

    
    local StealBarShadow = Instance.new("Frame")
    StealBarShadow.Name = "StealBarShadow"
    StealBarShadow.ZIndex = 9
    StealBarShadow.Position = UDim2.new(0.5, -177 * guiScalePB * sizeFactor, 1, -39 * guiScalePB * sizeFactor)
    StealBarShadow.Size = UDim2.new(0, 352 * guiScalePB, 0, 35 * guiScalePB)
    StealBarShadow.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    StealBarShadow.BackgroundTransparency = 0.35
    StealBarShadow.BorderSizePixel = 0
    
    
    local shadowScale = Instance.new("UIScale")
    shadowScale.Scale = sizeFactor
    shadowScale.Parent = StealBarShadow

    local ShadowCorner = Instance.new("UICorner")
    ShadowCorner.CornerRadius = UDim.new(0, 14)
    ShadowCorner.Parent = StealBarShadow

    
    local StealBar = Instance.new("TextButton")
    StealBar.Name = "StealBar"
    StealBar.ZIndex = 10
    StealBar.ClipsDescendants = true
    StealBar.Position = UDim2.new(0.5, -180 * guiScalePB * sizeFactor, 1, -43 * guiScalePB * sizeFactor)
    StealBar.Size = UDim2.new(0, 352 * guiScalePB, 0, 35 * guiScalePB)
    StealBar.BackgroundColor3 = Color3.fromRGB(8, 8, 9)
    StealBar.BackgroundTransparency = 0
    StealBar.BorderSizePixel = 0
    StealBar.Text = ""
    StealBar.AutoButtonColor = false
    StealBar.Active = true
    StealBar.Draggable = true
    StealBar.Parent = ProgressScreenGui
    local barScale = Instance.new("UIScale")
    barScale.Scale = sizeFactor
    barScale.Parent = StealBar

    makeDraggable(StealBar)
    makeDraggable(StealBarShadow)

    
    local function updateShadowPosition()
        StealBarShadow.Position = UDim2.new(
            StealBar.Position.X.Scale,
            StealBar.Position.X.Offset - 3 * guiScalePB,
            StealBar.Position.Y.Scale,
            StealBar.Position.Y.Offset + 3 * guiScalePB
        )
    end

    StealBar:GetPropertyChangedSignal("Position"):Connect(updateShadowPosition)
    task.wait()
    updateShadowPosition()

    local BarCorner = Instance.new("UICorner")
    BarCorner.CornerRadius = UDim.new(0, 19)
    BarCorner.Parent = StealBar

    local BarStroke = Instance.new("UIStroke")
    BarStroke.Color = Color3.fromRGB(0, 0, 0)
    BarStroke.Thickness = 1.5
    BarStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    BarStroke.Transparency = 0.1
    BarStroke.Parent = StealBar

    local BarGradient = Instance.new("UIGradient")
    BarGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(22, 22, 24)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(3, 3, 4))
    })
    BarGradient.Rotation = 0
    BarGradient.Parent = StealBar

    ProgressTrack = Instance.new("Frame")
    ProgressTrack.Name = "ProgressTrack"
    ProgressTrack.ZIndex = 11
    ProgressTrack.Size = UDim2.new(0, 210 * guiScalePB, 1, 0)
    ProgressTrack.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
    ProgressTrack.BorderSizePixel = 0
    ProgressTrack.ClipsDescendants = true
    ProgressTrack.Parent = StealBar
    local TrackCorner = Instance.new("UICorner")
    TrackCorner.CornerRadius = UDim.new(1, 0)
    TrackCorner.Parent = ProgressTrack
    local TrackStroke = Instance.new("UIStroke")
    TrackStroke.Thickness = 1.5
    
    TrackStroke.Color = Color3.fromRGB(22, 22, 22)
    TrackStroke.Transparency = 1
    TrackStroke.Parent = ProgressTrack

    
    LeftToRightFill = Instance.new("Frame")
    LeftToRightFill.Name = "LeftToRightFill"
    LeftToRightFill.ZIndex = 11
    LeftToRightFill.Size = UDim2.new(0, 0, 1, 0)
    LeftToRightFill.BackgroundColor3 = StealBarColor
    LeftToRightFill.BackgroundTransparency = 0.08
    LeftToRightFill.BorderSizePixel = 0
    LeftToRightFill.Parent = ProgressTrack

    local FillCorner = Instance.new("UICorner")
    FillCorner.CornerRadius = UDim.new(0, 14)
    FillCorner.Parent = LeftToRightFill

    local FillGradient = Instance.new("UIGradient")
    FillGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, StealBarColor:Lerp(Color3.new(1, 1, 1), 0.18)),
        ColorSequenceKeypoint.new(0.5, StealBarColor),
        ColorSequenceKeypoint.new(1, StealBarColor:Lerp(Color3.new(0, 0, 0), 0.16))
    })
    FillGradient.Parent = LeftToRightFill

    
    local DotPoint = Instance.new("Frame")
    DotPoint.Name = "DotPoint"
    DotPoint.ZIndex = 12
    DotPoint.Position = UDim2.new(0, 12 * guiScalePB, 0.5, -4 * guiScalePB)
    DotPoint.Size = UDim2.new(0, 8 * guiScalePB, 0, 8 * guiScalePB)
    DotPoint.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    DotPoint.BackgroundTransparency = 0.3
    DotPoint.BorderSizePixel = 0
    local DotCorner2 = Instance.new("UICorner")
    DotCorner2.CornerRadius = UDim.new(1, 0)
    DotCorner2.Parent = DotPoint
    DotPoint.Parent = StealBar
    DotPoint.Visible = false

    
    
    ProgressPercentLabel = Instance.new("TextLabel")
    ProgressPercentLabel.Name = "Percent"
    ProgressPercentLabel.ZIndex = 12
    ProgressPercentLabel.Position = UDim2.new(0, 0, 0, 0)
    ProgressPercentLabel.Size = UDim2.new(1, -14 * guiScalePB, 1, 0)
    ProgressPercentLabel.BackgroundTransparency = 1
    ProgressPercentLabel.Text = ""
    ProgressPercentLabel.TextColor3 = Color3.fromRGB(0, 0, 0)
    ProgressPercentLabel.TextSize = 10 * guiScalePB
    ProgressPercentLabel.Font = Enum.Font.GothamBold
    ProgressPercentLabel.TextXAlignment = Enum.TextXAlignment.Right
    ProgressPercentLabel.Visible = true
    ProgressPercentLabel.Parent = StealBar

    local StealTitle = Instance.new("TextLabel")
    StealTitle.Name = "StealTitle"
    StealTitle.ZIndex = 13
    StealTitle.Position = UDim2.new(0, 15 * guiScalePB, 0, 0)
    StealTitle.Size = UDim2.new(0, 125 * guiScalePB, 1, 0)
    StealTitle.BackgroundTransparency = 1
    
    StealTitle.Text = "STEAL"
    StealTitle.TextColor3 = Color3.fromRGB(245, 245, 245)
    StealTitle.TextSize = 12 * guiScalePB
    StealTitle.Font = Enum.Font.GothamBold
    StealTitle.TextXAlignment = Enum.TextXAlignment.Left
    StealTitle.Parent = ProgressTrack

    local StatsDivider = Instance.new("Frame")
    StatsDivider.Name =
... (280 KB restante(s))
