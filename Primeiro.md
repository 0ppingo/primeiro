-- Compact Aimbot FOV (Luau) - in-game (KRNL compatible) - NO NOTIFICATIONS / NO PRINTS
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

-- wait for local player and camera (in-game)
local LocalPlayer = Players.LocalPlayer
local tries = 0
while not LocalPlayer and tries < 50 do tries = tries + 1; task.wait(0.1); LocalPlayer = Players.LocalPlayer end
if not LocalPlayer then return end

local Camera = Workspace.CurrentCamera
tries = 0
while (not Camera or not Camera:IsA("Camera")) and tries < 60 do tries = tries + 1; task.wait(0.1); Camera = Workspace.CurrentCamera end
if not Camera then return end

-- CONFIG (minimal)
local C = {
    Aimbot = false,
    TeamCheck = true,
    FOVVisible = true,
    FOVRadius = 100, -- px
    HeadOffset = 0.9, -- studs above head
    LeadMS = 80, -- ms
    MaxPrediction = 6.0, -- studs
    Smooth = 0.4
}

-- try to parent GUI to CoreGui, fallback PlayerGui
local function getGuiParent()
    local ok = pcall(function()
        local g = Instance.new("ScreenGui")
        g.Parent = game:GetService("CoreGui")
        g:Destroy()
    end)
    if ok then return game:GetService("CoreGui") end
    return LocalPlayer:WaitForChild("PlayerGui")
end

local parent = getGuiParent()

-- build UI (compact)
local sg = Instance.new("ScreenGui")
sg.Name = "AimbotUI_Compact"
sg.ResetOnSpawn = false
sg.IgnoreGuiInset = true
sg.Parent = parent

local tog = Instance.new("TextButton", sg)
tog.Size = UDim2.new(0, 40, 0, 40)
tog.Position = UDim2.new(0.8, 0, 0.85, 0)
tog.Text = "⚙"
tog.Font = Enum.Font.GothamBold
tog.TextSize = 20
tog.BackgroundColor3 = Color3.fromRGB(30,30,30)
Instance.new("UICorner", tog).CornerRadius = UDim.new(1,0)

local main = Instance.new("Frame", sg)
main.Size = UDim2.new(0, 200, 0, 220)
main.Position = UDim2.new(0.5, -100, 0.5, -110)
main.BackgroundColor3 = Color3.fromRGB(24,24,24)
main.Visible = false
Instance.new("UICorner", main).CornerRadius = UDim.new(0,10)

local function addLabel(txt, posY)
    local l = Instance.new("TextLabel", main)
    l.Size = UDim2.new(0.6,0,0,24)
    l.Position = UDim2.new(0.02,0,posY,0)
    l.BackgroundTransparency = 1
    l.Text = txt
    l.TextColor3 = Color3.new(1,1,1)
    l.Font = Enum.Font.Gotham
    l.TextSize = 13
    l.TextXAlignment = Enum.TextXAlignment.Left
    return l
end

local function addBtn(txt, posY)
    local b = Instance.new("TextButton", main)
    b.Size = UDim2.new(0.96,0,0,28)
    b.Position = UDim2.new(0.02,0,posY,0)
    b.Text = txt
    b.Font = Enum.Font.Gotham
    b.TextSize = 13
    b.BackgroundColor3 = Color3.fromRGB(60,60,60)
    b.TextColor3 = Color3.new(1,1,1)
    Instance.new("UICorner", b).CornerRadius = UDim.new(0,6)
    return b
end

-- elements
local aimBtn = addBtn("Aimbot: OFF", 0.02)
local teamBtn = addBtn("TeamCheck: ON", 0.12)
local fovBtn = addBtn("FOV: ON", 0.22)
addLabel("FOV Radius: "..C.FOVRadius, 0.32)
local fovLbl = main:FindFirstChildOfClass("TextLabel") -- last created
local decF = Instance.new("TextButton", main); decF.Size=UDim2.new(0.18,0,0,24); decF.Position=UDim2.new(0.64,0,0.36,0); decF.Text="-" ; decF.Font=Enum.Font.GothamBold; decF.BackgroundColor3=Color3.fromRGB(70,70,70); Instance.new("UICorner", decF).CornerRadius=UDim.new(0,6)
local incF = decF:Clone(); incF.Parent = main; incF.Position=UDim2.new(0.84,0,0.36,0); incF.Text="+"

addLabel("HeadOffset: "..string.format("%.1f", C.HeadOffset), 0.46)
local headLbl = main:FindFirstChildOfClass("TextLabel", true) -- we'll keep ref below
-- simple inc/dec for headoffset
local decH = Instance.new("TextButton", main); decH.Size=UDim2.new(0.18,0,0,24); decH.Position=UDim2.new(0.64,0,0.5,0); decH.Text="-" ; decH.Font=Enum.Font.GothamBold; decH.BackgroundColor3=Color3.fromRGB(70,70,70); Instance.new("UICorner", decH).CornerRadius=UDim.new(0,6)
local incH = decH:Clone(); incH.Parent = main; incH.Position=UDim2.new(0.84,0,0.5,0); incH.Text="+"
-- find the labels more robustly
local labels = {}
for _,v in ipairs(main:GetChildren()) do if v:IsA("TextLabel") then table.insert(labels, v) end end
fovLbl = labels[1]; headLbl = labels[2]

-- button actions
aimBtn.MouseButton1Click:Connect(function()
    C.Aimbot = not C.Aimbot
    aimBtn.Text = "Aimbot: "..(C.Aimbot and "ON" or "OFF")
end)
teamBtn.MouseButton1Click:Connect(function()
    C.TeamCheck = not C.TeamCheck
    teamBtn.Text = "TeamCheck: "..(C.TeamCheck and "ON" or "OFF")
end)
fovBtn.MouseButton1Click:Connect(function()
    C.FOVVisible = not C.FOVVisible
    fovBtn.Text = "FOV: "..(C.FOVVisible and "ON" or "OFF")
end)
decF.MouseButton1Click:Connect(function() C.FOVRadius = math.clamp(C.FOVRadius - 10, 50, 300); fovLbl.Text = "FOV Radius: "..C.FOVRadius end)
incF.MouseButton1Click:Connect(function() C.FOVRadius = math.clamp(C.FOVRadius + 10, 50, 300); fovLbl.Text = "FOV Radius: "..C.FOVRadius end)
decH.MouseButton1Click:Connect(function() C.HeadOffset = math.clamp(C.HeadOffset - 0.1, 0.4, 1.4); headLbl.Text = "HeadOffset: "..string.format("%.1f", C.HeadOffset) end)
incH.MouseButton1Click:Connect(function() C.HeadOffset = math.clamp(C.HeadOffset + 0.1, 0.4, 1.4); headLbl.Text = "HeadOffset: "..string.format("%.1f", C.HeadOffset) end)

tog.MouseButton1Click:Connect(function()
    main.Visible = not main.Visible
    if main.Visible then main.Position = UDim2.new(0.5, -main.Size.X.Offset/2, 0.5, -main.Size.Y.Offset/2) end
end)

-- simple draggable (works for touch & mouse)
local function makeDraggable(obj)
    local dragging, dragInput, dragStart, startPos
    obj.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = obj.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    obj.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then dragInput = input end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            obj.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end
makeDraggable(main); makeDraggable(tog)

-- FOV visual
local FOVFrame = Instance.new("Frame", sg)
FOVFrame.AnchorPoint = Vector2.new(0.5, 0.5)
FOVFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
FOVFrame.Size = UDim2.new(0, C.FOVRadius*2, 0, C.FOVRadius*2)
FOVFrame.BackgroundTransparency = 1
FOVFrame.BorderSizePixel = 0
Instance.new("UICorner", FOVFrame).CornerRadius = UDim.new(1,0)
local stroke = Instance.new("UIStroke", FOVFrame); stroke.Thickness = 2; stroke.Color = Color3.fromRGB(150,255,150)
FOVFrame.Visible = C.FOVVisible

-- velocity history for prediction
local VelH = {}
local MAX_SAMPLES = 6
local function updVel()
    for _,pl in ipairs(Players:GetPlayers()) do
        if pl.Character and pl.Character:FindFirstChild("HumanoidRootPart") then
            local hrp = pl.Character.HumanoidRootPart
            local hist = VelH[pl] or {}
            table.insert(hist, 1, hrp.Velocity)
            if #hist > MAX_SAMPLES then table.remove(hist, #hist) end
            VelH[pl] = hist
        end
    end
end
local function avgVel(pl)
    local h = VelH[pl]; if not h or #h == 0 then return Vector3.new(0,0,0) end
    local s = Vector3.new(0,0,0)
    for _,v in ipairs(h) do s = s + v end
    return s / #h
end

local function computePred(pl, pos, camPos)
    local predSec = (C.LeadMS or 80) / 1000
    local a = avgVel(pl)
    -- simple clamp based on distance
    local pred = a * predSec
    local dist = (pos - camPos).Magnitude
    local maxP = math.clamp(dist * 0.12, 1, C.MaxPrediction or 6)
    if pred.Magnitude > maxP then pred = pred.Unit * maxP end
    return pred
end

local function canSee(camPos, char)
    if not char then return false end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return false end
    local rp = RaycastParams.new()
    rp.FilterDescendantsInstances = {LocalPlayer.Character}
    rp.FilterType = Enum.RaycastFilterType.Blacklist
    local res = Workspace:Raycast(camPos, (hrp.Position - camPos).Unit * ((hrp.Position - camPos).Magnitude + 2), rp)
    return res and res.Instance and res.Instance:IsDescendantOf(char)
end

-- Get closest target inside FOV
local function GetClosestTarget()
    local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    local camPos = Camera.CFrame.Position
    local closest = nil; local shortest = C.FOVRadius
    for _,pl in ipairs(Players:GetPlayers()) do
        if pl ~= LocalPlayer and pl.Character and pl.Character:FindFirstChild("HumanoidRootPart") then
            if C.TeamCheck and pl.Team == LocalPlayer.Team then goto cont end
            local hrp = pl.Character.HumanoidRootPart
            local headPos = pl.Character:FindFirstChild("Head") and (pl.Character.Head.Position + Vector3.new(0, C.HeadOffset, 0)) or (hrp.Position + Vector3.new(0,1.6 + C.HeadOffset,0))
            local pred = computePred(pl, headPos, camPos)
            local predicted = headPos + pred
            local sPos, onScreen = Camera:WorldToViewportPoint(predicted)
            if onScreen then
                local d = (Vector2.new(sPos.X, sPos.Y) - center).Magnitude
                if d <= shortest then
                    if canSee(camPos, pl.Character) then closest = pl; shortest = d end
                end
            end
        end
        ::cont::
    end
    return closest
end

-- main loop
RunService.RenderStepped:Connect(function()
    -- visuals
    FOVFrame.Size = UDim2.new(0, C.FOVRadius*2, 0, C.FOVRadius*2)
    FOVFrame.Visible = C.FOVVisible

    -- update velocity
    updVel()

    if C.Aimbot then
        local t = GetClosestTarget()
        if t and t.Character and t.Character:FindFirstChild("HumanoidRootPart") then
            local hrp = t.Character.HumanoidRootPart
            local headPos = t.Character:FindFirstChild("Head") and (t.Character.Head.Position + Vector3.new(0, C.HeadOffset, 0)) or (hrp.Position + Vector3.new(0,1.6 + C.HeadOffset, 0))
            local aim = headPos + computePred(t, headPos, Camera.CFrame.Position)
            -- check still inside FOV
            local sPos, onScreen = Camera:WorldToViewportPoint(aim)
            local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
            local d = (Vector2.new(sPos.X, sPos.Y) - center).Magnitude
            if onScreen and d <= C.FOVRadius then
                local newCF = CFrame.new(Camera.CFrame.Position, aim)
                Camera.CFrame = Camera.CFrame:Lerp(newCF, math.clamp(C.Smooth, 0.01, 0.99))
            end
        end
    end
end)
