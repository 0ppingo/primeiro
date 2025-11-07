-- Aimbot FOV simplificado (Luau)
local Players=game:GetService("Players")
local RunService=game:GetService("RunService")
local Camera=workspace.CurrentCamera
local LocalPlayer=Players.LocalPlayer
local UserInputService=game:GetService("UserInputService")

-- Config
local Config={AimbotEnabled=false,TeamCheck=true,FOVVisible=true,FOVRadius=100,HeadOffset=0.9,Smooth=0.4,LeadMS=80,MaxPrediction=6.0}

-- GUI
local gui=Instance.new("ScreenGui",game:GetService("CoreGui"))
gui.IgnoreGuiInset=true

local toggle=Instance.new("TextButton",gui)
toggle.Size=UDim2.new(0,40,0,40)
toggle.Position=UDim2.new(0.8,0,0.85,0)
toggle.Text="⚙"
toggle.BackgroundColor3=Color3.fromRGB(30,30,30)
toggle.TextColor3=Color3.fromRGB(255,255,255)
Instance.new("UICorner",toggle).CornerRadius=UDim.new(1,0)

local main=Instance.new("Frame",gui)
main.Size=UDim2.new(0,200,0,220)
main.Position=UDim2.new(0.5,-100,0.5,-110)
main.BackgroundColor3=Color3.fromRGB(24,24,24)
main.Visible=false
Instance.new("UICorner",main).CornerRadius=UDim.new(0,12)

-- Buttons
local function makeBtn(text,y)
 local b=Instance.new("TextButton",main)
 b.Size=UDim2.new(0.96,0,0,28)
 b.Position=UDim2.new(0.02,0,y,0)
 b.Text=text;b.Font=Enum.Font.Gotham;b.TextSize=13;b.BackgroundColor3=Color3.fromRGB(60,60,60);b.TextColor3=Color3.new(1,1,1)
 Instance.new("UICorner",b).CornerRadius=UDim.new(0,8)
 return b
end

local aimBtn=makeBtn("Aimbot: OFF",0.02)
aimBtn.MouseButton1Click:Connect(function()
 Config.AimbotEnabled=not Config.AimbotEnabled
 aimBtn.Text="Aimbot: "..(Config.AimbotEnabled and "ON" or "OFF")
end)

local teamBtn=makeBtn("TeamCheck: ON",0.12)
teamBtn.MouseButton1Click:Connect(function()
 Config.TeamCheck=not Config.TeamCheck
 teamBtn.Text="TeamCheck: "..(Config.TeamCheck and "ON" or "OFF")
end)

local fovToggle=makeBtn("FOV: ON",0.22)
fovToggle.MouseButton1Click:Connect(function()
 Config.FOVVisible=not Config.FOVVisible
 fovToggle.Text="FOV: "..(Config.FOVVisible and "ON" or "OFF")
end)

local fovScaleLbl=Instance.new("TextLabel",main)
fovScaleLbl.Size=UDim2.new(0.6,0,0,24)
fovScaleLbl.Position=UDim2.new(0.02,0,0.32,0)
fovScaleLbl.BackgroundTransparency=1
fovScaleLbl.Text="FOV Radius: "..Config.FOVRadius
fovScaleLbl.TextColor3=Color3.new(1,1,1)
fovScaleLbl.Font=Enum.Font.Gotham
fovScaleLbl.TextSize=13
fovScaleLbl.TextXAlignment=Enum.TextXAlignment.Left

local fovDec=makeBtn("-",0.36)
fovDec.Size=UDim2.new(0.18,0,0,24)
fovDec.MouseButton1Click:Connect(function() Config.FOVRadius=math.clamp(Config.FOVRadius-10,50,300); fovScaleLbl.Text="FOV Radius: "..Config.FOVRadius end)

local fovInc=fovDec:Clone()
fovInc.Parent=main
fovInc.Position=UDim2.new(0.84,0,0.36,0)
fovInc.Text="+"
fovInc.MouseButton1Click:Connect(function() Config.FOVRadius=math.clamp(Config.FOVRadius+10,50,300); fovScaleLbl.Text="FOV Radius: "..Config.FOVRadius end)

toggle.MouseButton1Click:Connect(function()
 main.Visible=not main.Visible
 if main.Visible then main.Position=UDim2.new(0.5,-main.Size.X.Offset/2,0.5,-main.Size.Y.Offset/2) end
end)

-- FOV circle
local FOVCircle=Instance.new("Frame",gui)
FOVCircle.AnchorPoint=Vector2.new(0.5,0.5)
FOVCircle.Position=UDim2.new(0.5,0,0.5,0)
FOVCircle.Size=UDim2.new(0,Config.FOVRadius*2,0,Config.FOVRadius*2)
FOVCircle.BackgroundTransparency=1
FOVCircle.BorderSizePixel=0
FOVCircle.Visible=Config.FOVVisible
Instance.new("UICorner",FOVCircle).CornerRadius=UDim.new(1,0)
local fovStroke=Instance.new("UIStroke",FOVCircle)
fovStroke.Thickness=2;fovStroke.Color=Color3.fromRGB(150,255,150)

-- Prediction
local VelHistory={}
local MAX_SAMPLES=6
local function updateVelHistory()
 for _,pl in ipairs(Players:GetPlayers()) do
  if pl.Character and pl.Character:FindFirstChild("HumanoidRootPart") then
   local hrp=pl.Character.HumanoidRootPart
   local h=VelHistory[pl]or{}
   table.insert(h,1,hrp.Velocity)
   if #h>MAX_SAMPLES then table.remove(h,#h) end
   VelHistory[pl]=h
  end
 end
end

local function avgVel(pl)
 local h=VelHistory[pl]
 if not h or #h==0 then return Vector3.new() end
 local s=Vector3.new()
 for _,v in ipairs(h) do s=s+v end
 return s/#h
end

local function computePrediction(pl,pos,cam)
 local predSec=Config.LeadMS/1000
 local pred=avgVel(pl)*predSec
 local dist=(pos-cam).Magnitude
 local maxP=math.clamp(dist*0.12,1,Config.MaxPrediction)
 if pred.Magnitude>maxP then pred=pred.Unit*maxP end
 return pred
end

local function canSeeTarget(cam,char)
 local hrp=char:FindFirstChild("HumanoidRootPart")
 if not hrp then return false end
 local rp=RaycastParams.new()
 rp.FilterDescendantsInstances={LocalPlayer.Character}
 rp.FilterType=Enum.RaycastFilterType.Blacklist
 local res=workspace:Raycast(cam,(hrp.Position-cam).Unit*(hrp.Position-cam).Magnitude+2,rp)
 return res and res.Instance and res.Instance:IsDescendantOf(char)
end

-- Closest target
local function GetClosestTarget()
 local closest=nil;local short=Config.FOVRadius
 local center=Vector2.new(Camera.ViewportSize.X/2,Camera.ViewportSize.Y/2)
 local camPos=Camera.CFrame.Position
 for _,pl in ipairs(Players:GetPlayers()) do
  if pl~=LocalPlayer and pl.Character and pl.Character:FindFirstChild("HumanoidRootPart") then
   if Config.TeamCheck and pl.Team==LocalPlayer.Team then goto cont end
   local hrp=pl.Character.HumanoidRootPart
   local headPos=pl.Character:FindFirstChild("Head") and (pl.Character.Head.Position+Vector3.new(0,Config.HeadOffset,0)) or (hrp.Position+Vector3.new(0,1.6+Config.HeadOffset,0))
   local pred=computePrediction(pl,headPos,camPos)
   local sPos,onScreen=Camera:WorldToViewportPoint(headPos+pred)
   if onScreen then
    local d=(Vector2.new(sPos.X,sPos.Y)-center).Magnitude
    if d<=short and canSeeTarget(camPos,pl.Character) then closest=pl;short=d end
   end
  end
  ::cont::
 end
 return closest
end

-- Render loop
RunService.RenderStepped:Connect(function()
 FOVCircle.Size=UDim2.new(0,Config.FOVRadius*2,0,Config.FOVRadius*2)
 FOVCircle.Visible=Config.FOVVisible
 updateVelHistory()
 if Config.AimbotEnabled then
  local target=GetClosestTarget()
  if target and target.Character and target.Character:FindFirstChild("HumanoidRootPart") then
   local hrp=target.Character.HumanoidRootPart
   local headPos=target.Character:FindFirstChild("Head") and (target.Character.Head.Position+Vector3.new(0,Config.HeadOffset,0)) or (hrp.Position+Vector3.new(0,1.6+Config.HeadOffset,0))
   local aimPos=headPos+computePrediction(target,headPos,Camera.CFrame.Position)
   local newCF=CFrame.new(Camera.CFrame.Position,aimPos)
   Camera.CFrame=Camera.CFrame:Lerp(newCF,math.clamp(Config.Smooth,0.01,0.99))
  end
 end
end)
