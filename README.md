local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")
local HttpService = game:GetService("HttpService")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera
local IsMobile = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled

local Config = {
    ESP_Enabled = true,
    ESP_Box = true,
    ESP_Name = true,
    ESP_Distance = true,
    ESP_Health = true,
    ESP_TeamCheck = true,
    ESP_MaxDistance = 1000,
    ESP_BoxColor = Color3.fromRGB(220, 60, 60),
    ESP_FriendlyColor = Color3.fromRGB(100, 180, 255),
    ESP_Snaplines = false,
    ESP_Grenades = false,
    ESP_Chams = false,
    ESP_Weapon = false,
    
    AIM_Enabled = false,
    AIM_FOV = IsMobile and 120 or 150,
    AIM_Smooth = IsMobile and 35 or 15,
    AIM_RageMode = false,
    AIM_TeamCheck = true,
    AIM_TargetPart = "Head",
    AIM_ShowFOV = true,
    AIM_VisCheck = true,
    AIM_Prediction = true,
    AIM_PredictStrength = 100,
    AIM_StickyAim = false,
    AIM_AimAssist = false,
    AIM_SnapRadius = 5,
    AIM_AimKey = "RMB",
    
    TRIGGER_Enabled = false,
    TRIGGER_Delay = 150,
    TRIGGER_FOV = 20,
    TRIGGER_TeamCheck = true,
    TRIGGER_VisCheck = true,
    TRIGGER_HeadOnly = false,
    
    VISUAL_Crosshair = true,
    VISUAL_AutoSpot = false,
    VISUAL_NoSmoke = false,
    VISUAL_NoRain = false,
    VISUAL_NoCamShake = false,
    
    VISUAL_BlinkTP = false,
    VISUAL_BlinkDist = 1.9,
    VISUAL_BlinkDelay = 0.09,
    VISUAL_BlinkSync = true,
    VISUAL_BlinkSyncInt = 0.6,
    
    MISC_Rearview = false,
    MISC_RearviewSize = IsMobile and 140 or 200,
    
    AIM_MobileToggle = true,
    AIM_ShowRageBtn = false,
    
    RADAR_Enabled = false,
    RADAR_Size = 120,
    RADAR_Range = 100,
    
    MENU_Key = "RightShift",
    MenuOpen = true,
    
    PERF_LiteMode = false
}

local State = { Unloaded = false, Aiming = false, MobileAiming = false }
local MobileRageBtn = nil
local Cache = { Targets = {}, ESP = {}, Visibility = {}, Chams = {}, GrenadeESP = {}, Grenades = {} }
local Connections = {}
local RearviewClones = {}
local RearviewEnvClones = {}
local RearviewGround = nil

local VirtualInputManager = nil
pcall(function() VirtualInputManager = game:GetService("VirtualInputManager") end)

local MouseMoveRel, Mouse1Click
do
    local mmr = mousemoverel or (Input and Input.MouseMoveRelative)
    if mmr then MouseMoveRel = function(x, y) pcall(mmr, x, y) end
    elseif VirtualInputManager then MouseMoveRel = function(x, y) pcall(function() VirtualInputManager:SendMouseMoveEvent(x, y, workspace.CurrentCamera.ViewportSize.X/2, workspace.CurrentCamera.ViewportSize.Y/2) end) end
    else MouseMoveRel = function() end end
    local m1c = mouse1click or click
    if m1c then Mouse1Click = function() pcall(m1c) end
    elseif VirtualInputManager then Mouse1Click = function() pcall(function() local p = UserInputService:GetMouseLocation() VirtualInputManager:SendMouseButtonEvent(p.X,p.Y,0,true,game,1) task.wait(0.016) VirtualInputManager:SendMouseButtonEvent(p.X,p.Y,0,false,game,1) end) end
    else Mouse1Click = function() end end
end

local DrawingAvailable = false
pcall(function() local t = Drawing.new("Line") if t then t:Remove() DrawingAvailable = true end end)

local Theme = {
    Bar = Color3.fromRGB(58, 58, 58),
    Panel = Color3.fromRGB(45, 45, 48),
    PanelLight = Color3.fromRGB(55, 55, 58),
    Border = Color3.fromRGB(35, 35, 38),
    TabActive = Color3.fromRGB(200, 120, 60),
    TabInactive = Color3.fromRGB(70, 70, 75),
    Text = Color3.fromRGB(220, 220, 220),
    TextDim = Color3.fromRGB(150, 150, 150),
    Accent = Color3.fromRGB(200, 120, 60),
    Input = Color3.fromRGB(35, 35, 38),
    Check = Color3.fromRGB(200, 120, 60),
    Green = Color3.fromRGB(100, 200, 100),
    Red = Color3.fromRGB(200, 100, 100)
}

local function Create(class, props, children)
    local inst = Instance.new(class)
    for k, v in pairs(props) do if k ~= "Parent" then inst[k] = v end end
    if children then for _, c in ipairs(children) do c.Parent = inst end end
    if props.Parent then inst.Parent = props.Parent end
    return inst
end

local ScreenGui = Create("ScreenGui", {
    Name = "EntrenchedGUI",
    ResetOnSpawn = false,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
    IgnoreGuiInset = true,
    DisplayOrder = 999999999,
    Parent = (gethui and gethui()) or (syn and syn.protect_gui and syn.protect_gui(Instance.new("ScreenGui")) and game:GetService("CoreGui")) or LocalPlayer:WaitForChild("PlayerGui")
})

local BarHeight = IsMobile and 44 or 28
local ContentHeight = IsMobile and 240 or 180

local BottomBar = Create("Frame", {
    Name = "BottomBar",
    BackgroundColor3 = Theme.Bar,
    BorderSizePixel = 0,
    Position = UDim2.new(0, 0, 1, -BarHeight),
    Size = UDim2.new(1, 0, 0, 
