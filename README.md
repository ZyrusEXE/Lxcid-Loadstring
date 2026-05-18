-- // Load UI from external repo (already defined)
loadstring(game:HttpGet("https://raw.githubusercontent.com/ZyrusEXE/Lxcid/main/lxcid%20lua"))()

-- Wait for UI to fully load
task.wait(0.5)
local Options = Library.Options
local Toggles = Library.Toggles

-- ======================== FOV CONTROLLER ========================
local RunService = game:GetService("RunService")
local WorkspaceCamera = workspace.CurrentCamera
RunService.RenderStepped:Connect(function()
    if Options.FieldOfView then
        local fov = Options.FieldOfView.Value
        if WorkspaceCamera and WorkspaceCamera.FieldOfView ~= fov then
            WorkspaceCamera.FieldOfView = fov
        end
    end
end)

-- ======================== AIMBOT LOGIC ========================
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local Stats = game:GetService("Stats")
local Camera = Workspace.CurrentCamera or Workspace:WaitForChild("Camera")
local LocalPlayer = Players.LocalPlayer
if not LocalPlayer.Character then
    LocalPlayer.CharacterAdded:Wait()
end
task.wait(0.2)
local Mouse = LocalPlayer:GetMouse()

local AIM_RADIUS = 90
local BOX_COLOR = Color3.fromRGB(255, 255, 255)
local STICKY_RADIUS = 100
local STICKY_SMOOTHNESS = 0.15

local predictionTable = {
    {0, 0.1332}, {10, 0.1234555}, {20, 0.12435}, {30, 0.124123}, {40, 0.12766},
    {50, 0.128643}, {60, 0.1264236}, {70, 0.12533}, {80, 0.1321042}, {90, 0.1421951},
    {100, 0.134143}, {105, 0.141199}, {110, 0.142199}, {125, 0.15465}, {130, 0.12399},
    {135, 0.1659921}, {140, 0.1659921}, {145, 0.129934}, {150, 0.1652131}, {155, 0.125333},
    {160, 0.1223333}, {165, 0.1652131}, {170, 0.16863}, {175, 0.16312}, {180, 0.1632},
    {185, 0.16823}, {190, 0.18659}, {205, 0.17782}, {215, 0.16937}, {225, 0.176332},
}

-- ======================== HELPERS ========================
local function IsBehindWall(targetPart)
    if not LocalPlayer.Character or not targetPart or not targetPart.Parent then return false end
    local origin = Camera.CFrame.Position
    local direction = targetPart.Position - origin
    if direction.Magnitude < 0.001 then return false end

    local dir = direction.Unit * 500

    local success, result = pcall(function() return Workspace:Raycast(origin, dir) end)
    if success then
        if result then
            local hit = result.Instance
            if targetPart.Parent and hit:IsDescendantOf(targetPart.Parent) then return false
            elseif hit:IsDescendantOf(LocalPlayer.Character) then return false
            else return true end
        end
        return false
    end

    local success2, hit = pcall(function()
        local ray = Ray.new(origin, dir)
        local ignore = {LocalPlayer.Character, targetPart.Parent}
        return Workspace:FindPartOnRay(ray, ignore, false, true)
    end)
    if success2 and hit then
        if hit:IsDescendantOf(targetPart.Parent) or hit:IsDescendantOf(LocalPlayer.Character) then return false
        else return true end
    end
    return false
end

local wallCache = { pos = Vector3.zero, result = false, time = 0 }
local function IsBehindWallCached(part)
    local now = tick()
    if (part.Position - wallCache.pos).Magnitude < 2 and (now - wallCache.time) < 0.15 then
        return wallCache.result
    end
    wallCache.result = IsBehindWall(part)
    wallCache.pos = part.Position
    wallCache.time = now
    return wallCache.result
end

local function IsKnocked(character)
    local root = character:FindFirstChild("HumanoidRootPart")
    if root and root.Anchored then return true end
    for _, obj in ipairs(character:GetDescendants()) do
        if obj:IsA("Motor6D") and not obj.Enabled then return true end
    end
    local hum = character:FindFirstChildWhichIsA("Humanoid")
    if hum and hum.Health <= 0 then return true end
    return false
end

local function IsAirborne(character)
    local hum = character:FindFirstChildWhichIsA("Humanoid")
    if hum and hum:GetState() == Enum.HumanoidStateType.Freefall then return true end
    local root = character:FindFirstChild("HumanoidRootPart")
    if root then return root.Velocity.Y > 2 or root.Velocity.Y < -2 end
    return false
end

local function GetFirstTargetInRadius()
    if not Camera or not Mouse then return nil end
    local mousePos = Vector2.new(Mouse.X, Mouse.Y + 36)
    local bestDist = AIM_RADIUS + 1
    local bestChar = nil
    for _, player in ipairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        local char = player.Character
        if not char then continue end
        local hum = char:FindFirstChildWhichIsA("Humanoid")
        if not hum or hum.Health <= 0 then continue end

        if Toggles.KnockOut.Value and IsKnocked(char) then continue end
        if Toggles.Friend.Value and player:IsFriendsWith(LocalPlayer.UserId) then continue end
        if Toggles.Team.Value and player.Team == LocalPlayer.Team then continue end

        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hrp then continue end
        if Toggles.Wall.Value and IsBehindWall(hrp) then continue end

        local screenPos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
        if onScreen then
            local dist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
            if dist <= AIM_RADIUS and dist < bestDist then
                bestDist = dist
                bestChar = char
            end
        end
    end
    return bestChar
end

local function GetAimPart(character)
    local aimPartName = IsAirborne(character)
        and Options.AirAimPart.Value
        or Options.AimPart.Value
    return character:FindFirstChild(aimPartName)
        or character:FindFirstChild("HumanoidRootPart")
        or character:FindFirstChild("Head")
end

local function GetPredictedPosition(part, useResolver)
    local vel
    if useResolver and Toggles.Resolver.Value then
        local hum = part.Parent:FindFirstChildWhichIsA("Humanoid")
        vel = hum and (hum.MoveDirection * hum.WalkSpeed) or part.Velocity
    else
        vel = part.Velocity
    end
    local hPred = Options.HorizontalPrediction.Value
    local vPred = Options.VerticalPrediction.Value
    return part.Position + Vector3.new(vel.X * hPred, vel.Y * vPred, vel.Z * hPred)
end

-- ======================== LOCK STATE ========================
local AimbotActive = false
local LockedTarget, LockedPlayer = nil, nil

local aimStart = 0
local humanizerOffset = Vector3.zero
local lastOffsetChange = 0

local function ToggleAimbot()
    AimbotActive = not AimbotActive
    if AimbotActive then
        local target = GetFirstTargetInRadius()
        if target then
            LockedTarget = target
            LockedPlayer = Players:GetPlayerFromCharacter(target)
            aimStart = tick()
            humanizerOffset = Vector3.zero
            lastOffsetChange = 0
        else
            LockedTarget, LockedPlayer = nil, nil
        end
    else
        LockedTarget, LockedPlayer = nil, nil
        aimStart = 0
        humanizerOffset = Vector3.zero
    end
end

local lockKeybind = Options.LockKeybind
if lockKeybind and lockKeybind.OnClick then lockKeybind:OnClick(ToggleAimbot) end
UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.UserInputType == Enum.UserInputType.Gamepad1 and input.KeyCode == Enum.KeyCode.DPadDown then ToggleAimbot()
    elseif input.KeyCode == Enum.KeyCode.T then ToggleAimbot() end
end)

local function UpdateAutoPrediction()
    if not Toggles.AutoPred.Value then return end
    local ok, ping = pcall(function() return Stats.Network.ServerStatsItem["Data Ping"]:GetValue() end)
    if not ok or not ping then return end
    local best = predictionTable[#predictionTable][2]
    for i = 1, #predictionTable do
        if ping < predictionTable[i][1] then best = predictionTable[i][2]; break end
    end
    pcall(function() Options.HorizontalPrediction:SetValue(best); Options.VerticalPrediction:SetValue(best) end)
end

-- ======================== SILENT AIM + RAPID FIRE ========================
local remote = nil
for _, obj in ipairs(game:GetService("ReplicatedStorage"):GetDescendants()) do
    if obj.Name == "MAINEVENT" and obj:IsA("RemoteEvent") then remote = obj break end
end

local hookedTools = {}

if remote then
    local mouseButtonDown = false
    local rapidFireLoop = nil

    UserInputService.InputBegan:Connect(function(input, gp)
        if gp then return end
        if input.UserInputType == Enum.UserInputType.MouseButton1 then mouseButtonDown = true end
    end)
    UserInputService.InputEnded:Connect(function(input, gp)
        if gp then return end
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            mouseButtonDown = false
            if rapidFireLoop then rapidFireLoop = nil end
        end
    end)

    local function onToolActivated()
        if Toggles.SilentAim.Value and AimbotActive and LockedTarget and LockedTarget.Parent then
            if Toggles.KnockOut.Value and IsKnocked(LockedTarget) then return end

            local aimPart = GetAimPart(LockedTarget)
            if aimPart and not (Toggles.Wall.Value and IsBehindWallCached(aimPart)) then
                local aimPos = GetPredictedPosition(aimPart, Toggles.Resolver.Value)
                local chance = Options.HitChance.Value or 100
                if chance >= 100 or math.random(100) <= chance then
                    local antiCurveDeg = Options.AntiCurve.Value or 0
                    local allow = true
                    if antiCurveDeg > 0 then
                        local cameraPos = Camera.CFrame.Position
                        local origDir = Camera.CFrame.LookVector
                        local newDir = (aimPos - cameraPos).Unit
                        local angle = math.acos(math.clamp(origDir:Dot(newDir), -1, 1))
                        if math.deg(angle) > antiCurveDeg then allow = false end
                    end
                    if allow then remote:FireServer("MOUSE", aimPos) end
                end
            end
        end
    end

    local function hookTool(tool)
        if not tool:IsA("Tool") or hookedTools[tool] then return end
        hookedTools[tool] = true
        tool.Activated:Connect(function()
            onToolActivated()
            if Toggles.RapidFire.Value and mouseButtonDown then
                if not rapidFireLoop then
                    rapidFireLoop = task.spawn(function()
                        while mouseButtonDown do
                            task.wait(Options.RapidFireDelay.Value or 0.04)
                            if tool and tool:IsDescendantOf(LocalPlayer.Character or LocalPlayer.Backpack) then
                                tool:Activate()
                            else break end
                        end
                        rapidFireLoop = nil
                    end)
                end
            end
        end)
    end

    if LocalPlayer.Backpack then
        for _, t in ipairs(LocalPlayer.Backpack:GetChildren()) do hookTool(t) end
        LocalPlayer.Backpack.ChildAdded:Connect(hookTool)
    end
    LocalPlayer.CharacterAdded:Connect(function(character)
        for _, t in ipairs(character:GetChildren()) do hookTool(t) end
        character.ChildAdded:Connect(function(child) hookTool(child) end)
    end)
    print("Silent Aim + Rapid Fire installed.")
else
    warn("MAINEVENT not found – silent aim disabled.")
end

-- ======================== CAMLOCK (HUMANIZER SPEED × SLIDER) ========================
RunService.RenderStepped:Connect(function(dt)
    UpdateAutoPrediction()
    if Toggles.StickyAim.Value then return end
    if not AimbotActive or not Toggles.Enabled.Value then return end

    if LockedTarget and LockedPlayer then
        local hum = LockedTarget:FindFirstChildWhichIsA("Humanoid")
        if not hum or hum.Health <= 0 or (Toggles.KnockOut.Value and IsKnocked(LockedTarget)) then
            LockedTarget = nil; LockedPlayer = nil; AimbotActive = false; return
        end
        if LockedPlayer.Parent == nil or LockedPlayer.Character ~= LockedTarget then
            LockedTarget = nil; LockedPlayer = nil; AimbotActive = false; return
        end
    else
        LockedTarget = nil; LockedPlayer = nil; AimbotActive = false; return
    end

    local part = GetAimPart(LockedTarget)
    if not part then return end
    if Toggles.Wall.Value and IsBehindWallCached(part) then return end

    local aimPos = GetPredictedPosition(part, Toggles.Resolver.Value)

    local smooth
    if Toggles.Humanizer.Value then
        local minSpeed = Options.HumanizerMinSpeed.Value or 0.06
        local maxSpeed = Options.HumanizerMaxSpeed.Value or 0.09
        local acceleration = Options.HumanizerAcceleration.Value or 0.013
        local elapsed = tick() - aimStart
        local humanizerSpeed = math.min(minSpeed + elapsed * acceleration, maxSpeed)

        local slider = Options.CameraSmoothness.Value or 0.9
        smooth = humanizerSpeed * math.clamp(slider, 0, 1)

        local hRange = Options.HumanizerHOffset.Value or 0.2
        local vRange = Options.HumanizerVOffset.Value or 0.15
        local changeInterval = Options.HumanizerChangeInterval.Value or 0.3
        local now = tick()
        if now - lastOffsetChange > changeInterval then
            humanizerOffset = Vector3.new(
                (math.random()*2-1) * hRange,
                (math.random()*2-1) * vRange,
                (math.random()*2-1) * hRange
            )
            lastOffsetChange = now
        end
        aimPos = aimPos + humanizerOffset
    else
        smooth = Options.CameraSmoothness.Value or 0.9
    end

    smooth = math.clamp(smooth, 0, 1)
    if smooth >= 0.99 then smooth = 1 end
    Camera.CFrame = Camera.CFrame:Lerp(CFrame.new(Camera.CFrame.Position, aimPos), smooth)
end)

-- ======================== STICKY AIM ========================
RunService.RenderStepped:Connect(function(dt)
    if not Toggles.StickyAim.Value or not Toggles.Enabled.Value or not LockedTarget then return end
    if Toggles.KnockOut.Value and IsKnocked(LockedTarget) then return end

    local part = GetAimPart(LockedTarget)
    if not part then return end
    if Toggles.Wall.Value and IsBehindWallCached(part) then return end

    local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
    if not onScreen then return end
    local mousePos = Vector2.new(Mouse.X, Mouse.Y + 36)
    local distToTarget = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
    if distToTarget <= STICKY_RADIUS then
        local aimPos = GetPredictedPosition(part, Toggles.Resolver.Value)
        local smoothness = math.clamp(STICKY_SMOOTHNESS, 0, 1)
        Camera.CFrame = Camera.CFrame:Lerp(CFrame.new(Camera.CFrame.Position, aimPos), smoothness)
    end
end)

-- ======================== LOOK AT ========================
RunService.RenderStepped:Connect(function()
    if not AimbotActive or not Toggles.Enabled.Value or not Toggles.LookAt.Value then
        local char = LocalPlayer.Character
        if char and char:FindFirstChildOfClass("Humanoid") then char.Humanoid.AutoRotate = true end
        return
    end
    if not LockedTarget or not LockedPlayer then
        local char = LocalPlayer.Character
        if char and char:FindFirstChildOfClass("Humanoid") then char.Humanoid.AutoRotate = true end
        return
    end
    local localChar = LocalPlayer.Character
    if not localChar then return end
    local root = localChar:FindFirstChild("HumanoidRootPart")
    local hum = localChar:FindFirstChildWhichIsA("Humanoid")
    if not root or not hum then return end
    if LockedPlayer.Character and LockedPlayer.Character:FindFirstChild("HumanoidRootPart") then
        local targetRoot = LockedPlayer.Character.HumanoidRootPart
        local dir = Vector3.new(targetRoot.Position.X-root.Position.X, 0, targetRoot.Position.Z-root.Position.Z).Unit
        root.CFrame = CFrame.new(root.Position, root.Position+dir)
        hum.AutoRotate = false
    else
        hum.AutoRotate = true
    end
end)

-- ======================== AUTO AIR ========================
local lastAir = nil
task.spawn(function()
    while true do
        if Toggles.AutoAir.Value and AimbotActive and LockedTarget and LockedPlayer then
            if Toggles.KnockOut.Value and IsKnocked(LockedTarget) then task.wait(0.1); continue end
            local char = LockedTarget
            if char and char:FindFirstChild("HumanoidRootPart") then
                local hrp = char.HumanoidRootPart
                local hum = char:FindFirstChildWhichIsA("Humanoid")
                if hum and hrp and (hum:GetState()==Enum.HumanoidStateType.Freefall or hrp.Velocity.Y>2 or hrp.Velocity.Y<-2) then
                    local now = tick()
                    local delay = Options.AutoAirDelay.Value or 0.22
                    if not lastAir or now-lastAir >= delay then
                        local tool = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildWhichIsA("Tool")
                        if tool then tool:Activate() end
                        lastAir = now
                    end
                else lastAir = nil end
            end
        end
        task.wait(0.1)
    end
end)

-- ======================== P1000 DESYNC (SAFE HOOK) ========================
local DesyncStore = {}

UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == Enum.KeyCode.P then
        Toggles.P1000Desync:SetValue(not Toggles.P1000Desync.Value)
    end
    if input.KeyCode == Enum.KeyCode.Equals then
        game:GetService("TeleportService"):Teleport(game.PlaceId, LocalPlayer)
    end
end)

local function RandomNumberRange(a)
    return math.random(-a * 100, a * 100) / 100
end

local XDDDDDD = nil
pcall(function()
    XDDDDDD = hookmetamethod(game, "__index", newcclosure(function(self, key)
        if Toggles.EnableAntiLock.Value and Toggles.P1000Desync.Value then
            if not checkcaller() then
                if key == "CFrame" and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") and
                   LocalPlayer.Character:FindFirstChild("Humanoid") and LocalPlayer.Character.Humanoid.Health > 0 then
                    if self == LocalPlayer.Character.HumanoidRootPart then
                        return DesyncStore[1] or CFrame.new()
                    elseif self == LocalPlayer.Character.Head then
                        return DesyncStore[1] and DesyncStore[1] + Vector3.new(0, LocalPlayer.Character.HumanoidRootPart.Size / 2 + 0.5, 0) or CFrame.new()
                    end
                end
            end
        end
        return XDDDDDD(self, key)
    end))
end)
if not XDDDDDD then
    Toggles.P1000Desync:SetValue(false)
    Toggles.EnableAntiLock:SetValue(false)
    warn("P1000 Desync unavailable – executor missing hookmetamethod/newcclosure/checkcaller.")
end

RunService.Heartbeat:Connect(function()
    if Toggles.EnableAntiLock.Value and Toggles.P1000Desync.Value and XDDDDDD then
        local char = LocalPlayer.Character
        if char and char:FindFirstChild("HumanoidRootPart") and char:FindFirstChild("Humanoid") and char.Humanoid.Health > 0 then
            local hrp = char.HumanoidRootPart
            DesyncStore[1] = hrp.CFrame
            DesyncStore[2] = hrp.AssemblyLinearVelocity

            local spoofCF = hrp.CFrame
            spoofCF = spoofCF * CFrame.new(Vector3.new(0, 0, 0))
            spoofCF = spoofCF * CFrame.Angles(math.rad(RandomNumberRange(180)), math.rad(RandomNumberRange(180)), math.rad(RandomNumberRange(180)))
            hrp.CFrame = spoofCF
            hrp.AssemblyLinearVelocity = Vector3.new(1, 1, 1) * 16384

            RunService.RenderStepped:Wait()

            hrp.CFrame = DesyncStore[1]
            hrp.AssemblyLinearVelocity = DesyncStore[2]
        end
    end
end)

-- ======================== ANTI NETWORK ========================
RunService.Heartbeat:Connect(function()
    if Toggles.EnableAntiLock.Value and Toggles.AntiNetwork.Value then
        local char = LocalPlayer.Character
        if char and char:FindFirstChild("HumanoidRootPart") then
            local hrp = char.HumanoidRootPart
            pcall(function() hrp:SetNetworkOwner(nil) end)
            pcall(function() sethiddenproperty(hrp, "NetworkIsSleeping", true) end)
        end
    end
end)

-- ======================== CFrame SPEED ========================
RunService.Heartbeat:Connect(function()
    if not Toggles.LoadCFrame.Value then return end
    local char = LocalPlayer.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChildWhichIsA("Humanoid")
    if not root or not hum then return end
    local speedPercent = Options.CFrame_SpeedPercent and Options.CFrame_SpeedPercent.Value or 3
    local speed = speedPercent / 100 * 10
    root.CFrame = root.CFrame + hum.MoveDirection * speed
end)

-- ======================== ESP ========================
if pcall(function() return Drawing.new end) then
    local PlayerESP = {}
    local function NewLine(col,thick) local l=Drawing.new("Line") l.Visible=false l.Thickness=thick or 2 l.Transparency=1 l.Color=col or BOX_COLOR return l end
    local function NewText() local t=Drawing.new("Text") t.Visible=false t.Size=14 t.Color=Color3.new(1,1,1) t.Center=true t.Outline=true t.OutlineColor=Color3.new(0,0,0) return t end

    local function SetupESP(player)
        if player == LocalPlayer then return end
        local box = { Top=NewLine(BOX_COLOR,2), Bottom=NewLine(BOX_COLOR,2), Left=NewLine(BOX_COLOR,2), Right=NewLine(BOX_COLOR,2) }
        local hBg = NewLine(Color3.new(0,0,0),4)
        local hFill = NewLine(Color3.new(0,1,0),2)
        local nameTxt = NewText()
        local conn = RunService.RenderStepped:Connect(function()
            local char = player.Character
            if not char then for _,v in pairs(box) do v.Visible=false end hBg.Visible=false hFill.Visible=false nameTxt.Visible=false return end
            local hrp = char:FindFirstChild("HumanoidRootPart")
            local head = char:FindFirstChild("Head")
            if not hrp or not head then for _,v in pairs(box) do v.Visible=false end hBg.Visible=false hFill.Visible=false nameTxt.Visible=false return end
            local hum = char:FindFirstChildWhichIsA("Humanoid")
            if not hum or hum.Health<=0 then for _,v in pairs(box) do v.Visible=false end hBg.Visible=false hFill.Visible=false nameTxt.Visible=false return end
            local topPos = head.Position+Vector3.new(0,0.5,0)
            local botPos = hrp.Position-Vector3.new(0,hrp.Size.Y/2,0)
            local topScr,topVis = Camera:WorldToViewportPoint(topPos)
            local botScr,botVis = Camera:WorldToViewportPoint(botPos)
            local cenScr,cenVis = Camera:WorldToViewportPoint(head.Position)
            if not topVis and not botVis and not cenVis then for _,v in pairs(box) do v.Visible=false end hBg.Visible=false hFill.Visible=false nameTxt.Visible=false return end
            local baseH = math.abs(topScr.Y-botScr.Y)
            local baseW = baseH*0.5
            local wMul = math.max(tonumber(Options.BoxWidth.Value or 1),0.1)
            local hMul = math.max(tonumber(Options.BoxHeight.Value or 1),0.1)
            local boxH = baseH*hMul
            local boxW = baseW*wMul
            local cx = cenScr.X
            local cy = topScr.Y+baseH/2
            local L = cx-boxW/2
            local R = cx+boxW/2
            local T = cy-boxH/2
            local B = cy+boxH/2
            if Toggles.BoxCorners.Value then
                box.Top.From = Vector2.new(L,T); box.Top.To = Vector2.new(R,T)
                box.Bottom.From = Vector2.new(L,B); box.Bottom.To = Vector2.new(R,B)
                box.Left.From = Vector2.new(L,T); box.Left.To = Vector2.new(L,B)
                box.Right.From = Vector2.new(R,T); box.Right.To = Vector2.new(R,B)
                for _,v in pairs(box) do v.Visible=true end
            else for _,v in pairs(box) do v.Visible=false end end
            if Toggles.HealthBarEnabled.Value then
                local bw=4; local bo=5; local bx=L-bo-bw/2
                local maxHp = hum.MaxHealth
                if maxHp <= 0 then maxHp = 1 end
                local hp = hum.Health / maxHp
                local fh = boxH*hp; local ft = B-fh
                hBg.From = Vector2.new(bx,T); hBg.To = Vector2.new(bx,B); hBg.Thickness=bw; hBg.Visible=true
                hFill.From = Vector2.new(bx,ft); hFill.To = Vector2.new(bx,B); hFill.Thickness=bw-2; hFill.Visible=true
                hFill.Color = Color3.fromRGB(math.clamp(2-2*hp,0,1)*255, math.clamp(2*hp,0,1)*255, 0)
            else hBg.Visible=false; hFill.Visible=false end
            if Toggles.NameEnabled.Value then
                nameTxt.Text = player.Name
                nameTxt.Position = Vector2.new(cx, T-14)
                nameTxt.Visible = true
            else nameTxt.Visible=false end
        end)
        Players.PlayerRemoving:Connect(function(p) if p==player then conn:Disconnect() for _,v in pairs(box) do v:Remove() end hBg:Remove() hFill:Remove() nameTxt:Remove() PlayerESP[player]=nil end end)
    end
    for _,p in ipairs(Players:GetPlayers()) do SetupESP(p) end
    Players.PlayerAdded:Connect(SetupESP)
else warn("Drawing library missing.") end

print("Lxcid - Full script loaded. T lock, P desync, = rejoin.")
