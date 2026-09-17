local FONT_BOLD = (pcall(function() return Enum.Font.GothamBold end) and Enum.Font.GothamBold) or Enum.Font.Gotham
local FONT_REGULAR = (pcall(function() return Enum.Font.SourceSansBold end) and Enum.Font.SourceSansBold) or Enum.Font.SourceSans

if not LPH_NO_VIRTUALIZE then LPH_NO_VIRTUALIZE = function(fn) return fn end end

local cloneref = cloneref or function(x) return x end

local Players = cloneref(game:GetService("Players"))
local TweenService = cloneref(game:GetService("TweenService"))
local UserInputService = cloneref(game:GetService("UserInputService"))
local RunService = cloneref(game:GetService("RunService"))
local Lighting = cloneref(game:GetService("Lighting"))
local HttpService = cloneref(game:GetService("HttpService"))
local ReplicatedStorage = cloneref(game:GetService("ReplicatedStorage"))
local Workspace = cloneref(game:GetService("Workspace"))
local ContextActionService = cloneref(game:GetService("ContextActionService"))
local StarterGui = cloneref(game:GetService("StarterGui"))
local LogService = cloneref(game:GetService("LogService"))

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local CONFIG_FOLDER = "ZAXMINI"
local CONFIG_PATH   = CONFIG_FOLDER .. "/config.json"

local Config = {
    autoSteal        = false,
    invisOnSteal     = true,
    autoTPOnExecute  = true,
    keybind          = "T",
    targetMode       = "gen",
    invisDepth       = 4.2,
    invisRotation    = 180,
    tpMode           = "grapple",
    grappleTweenSpeed  = 400,
    grappleCloneDelay  = 1.0,
    carpetTweenSpeed   = 400,
    carpetCloneDelay   = 1.0,
    sideApproach     = "auto",
    guiScale         = 1.0,
    panelPosition    = { X=0.5, XOffset=0, Y=0.5, YOffset=0 },
    tpDelay          = 0.1,
}

local function saveConfig()
    pcall(function()
        if not isfolder(CONFIG_FOLDER) then makefolder(CONFIG_FOLDER) end
        writefile(CONFIG_PATH, HttpService:JSONEncode(Config))
    end)
end
local function loadConfig()
    pcall(function()
        if isfile(CONFIG_PATH) then
            local d = HttpService:JSONDecode(readfile(CONFIG_PATH))
            for k,v in pairs(d) do Config[k]=v end
        end
    end)
end
loadConfig()
if not Config.tpMode or Config.tpMode==""         then Config.tpMode="grapple"  end
if not Config.sideApproach or Config.sideApproach=="" then Config.sideApproach="auto" end
if not Config.targetMode or Config.targetMode==""  then Config.targetMode="gen" end
if not Config.guiScale                             then Config.guiScale=1.0      end
if type(Config.panelPosition)~="table"             then Config.panelPosition={X=0.5,XOffset=0,Y=0.5,YOffset=0} end

local Theme = {
    Panel      = Color3.fromRGB(28,25,48),   PanelAlt   = Color3.fromRGB(34,31,56),
    PanelField = Color3.fromRGB(38,35,58),   PanelField2= Color3.fromRGB(46,42,70),
    Border     = Color3.fromRGB(54,49,82),   Outline    = Color3.fromRGB(78,156,255),
    Text       = Color3.fromRGB(235,233,247), TextDim   = Color3.fromRGB(163,159,189),
    TextFaint  = Color3.fromRGB(120,115,148), AccentA   = Color3.fromRGB(94,133,246),
    AccentB    = Color3.fromRGB(130,93,240),  Green     = Color3.fromRGB(56,224,137),
    GreenDark  = Color3.fromRGB(31,189,108),  CardBg    = Color3.fromRGB(32,28,52),
}

local function new(cl,props,parent)
    local i=Instance.new(cl) for k,v in pairs(props or {}) do i[k]=v end
    if parent then i.Parent=parent end return i
end
local function corner(p,r)  return new("UICorner",{CornerRadius=UDim.new(0,r or 10)},p) end
local function strokeOn(p,c,t,tr) return new("UIStroke",{Color=c or Theme.Border,Thickness=t or 1,Transparency=tr or 0},p) end
local function grad(p,cs,rot)    return new("UIGradient",{Color=cs,Rotation=rot or 0},p) end
local function accentGrad(p,r)   return grad(p,ColorSequence.new(Theme.AccentA,Theme.AccentB),r or 0) end
local function bgDepthGrad(p,r)  return grad(p,ColorSequence.new(Theme.PanelAlt,Theme.Panel),r or 105) end
local function titleUnderline(p,w)
    local b=new("Frame",{Size=UDim2.new(0,w or 34,0,2),Position=UDim2.new(0,0,1,3),BackgroundColor3=Theme.AccentA,BorderSizePixel=0},p)
    corner(b,2); accentGrad(b,0); return b
end
local function tw(i,pr,t,s,d)
    local t2=TweenService:Create(i,TweenInfo.new(t or 0.2,s or Enum.EasingStyle.Quad,d or Enum.EasingDirection.Out),pr) t2:Play(); return t2
end

do
    local function getSynchronizer()
        local ok, mod = pcall(require, ReplicatedStorage.Packages.Synchronizer)
        if ok then return mod end
        return nil
    end

    local function getTable(name)
        local sync = getSynchronizer()
        if not sync then return nil end
        local ok, data = pcall(sync.GetTableFromChannel, sync, name)
        if ok then return data end
        return nil
    end

    _G.XenSyncAll = function()
        return {}
    end

    _G.XenSyncGet = function(name)
        if type(name) ~= "string" then return nil end
        return getTable(name)
    end

    _G.sProp = function(ch, key)
        if type(ch) ~= "table" or key == nil then return nil end
        return ch[key]
    end

    local _AD, _MD, _TD
    local function _data()
        if _AD then return true end
        local ok = pcall(function()
            local d = ReplicatedStorage:WaitForChild("Datas")
            _AD = require(d:WaitForChild("Animals"))
            _MD = require(d:WaitForChild("Mutations"))
            _TD = require(d:WaitForChild("Traits"))
        end)
        return ok and _AD ~= nil
    end
    _G._xenGen = function(index, mutation, traits)
        if not _data() then return 0 end
        local info = _AD[index]
        if not info or not info.Generation then return 0 end
        local mult = 1
        if mutation and mutation ~= "None" and mutation ~= "" then
            local m = _MD[mutation]
            if m and m.Modifier then mult = mult + m.Modifier end
        end
        if type(traits) == "table" then
            for _, tr in ipairs(traits) do
                local t = _TD[tr]
                if t and t.MultiplierModifier then mult = mult + t.MultiplierModifier end
            end
        end
        return info.Generation * mult
    end
    _G._xenAnimShim = setmetatable({
        GetGeneration = function(_, index, mutation, traits)
            return _G._xenGen(index, mutation, traits)
        end
    }, {
        __index = function(_, k)
            local ok, real = pcall(function()
                return require(ReplicatedStorage:WaitForChild("Shared"):WaitForChild("Animals"))
            end)
            if ok and type(real) == "table" then return rawget(real, k) end
            return nil
        end
    })
end

task.spawn(function()
    local plots
    local t0 = os.clock()
    repeat
        plots = Workspace:FindFirstChild("Plots")
        if not plots then task.wait(0.05) end
    until plots or (os.clock() - t0) > 25
    if not plots then return end
    local done, warmT0 = {}, os.clock()
    while (os.clock() - warmT0) < 15 do
        local pending = false
        for _, plot in ipairs(plots:GetChildren()) do
            if not done[plot.Name] then
                local ch = _G.XenSyncGet and _G.XenSyncGet(plot.Name)
                if ch then
                    pcall(function()
                        if ch.Get then ch:Get("AnimalList"); ch:Get("Owner") end
                    end)
                    if _G.sProp(ch, "AnimalList") ~= nil then
                        done[plot.Name] = true
                    else
                        pending = true
                    end
                else
                    pending = true
                end
            end
        end
        if not pending and #plots:GetChildren() > 0 then break end
        task.wait(0.05)
    end
end)

task.spawn(function()
    if not Workspace.StreamingEnabled then return end
    local plots
    local t0 = os.clock()
    repeat
        plots = Workspace:FindFirstChild("Plots")
        if not plots then task.wait(0.1) end
    until plots or (os.clock() - t0) > 25
    if not plots then return end
    for _, plot in ipairs(plots:GetChildren()) do
        local pos
        pcall(function()
            pos = plot:GetPivot().Position
        end)
        if pos then
            task.spawn(function()
                pcall(function() player:RequestStreamAroundAsync(pos) end)
            end)
        end
    end
end)

do
    local RS = ReplicatedStorage
    local function resolveRemote(name)
        local pkgs = RS:FindFirstChild("Packages")
        local Net = pkgs and pkgs:FindFirstChild("Net")
        if not Net then return nil end
        local slots, byLeaf = {}, {}
        for i, c in ipairs(Net:GetChildren()) do
            local p, rest = string.match(c.Name, "^(%a+)/(.+)$")
            if p and #rest == 64 and string.match(rest, "^%x+$") then
                slots[i] = c
                byLeaf[rest] = byLeaf[rest] or {}
                table.insert(byLeaf[rest], { slot = i, class = c.ClassName })
            end
        end
        if name then
            for leaf, list in pairs(byLeaf) do
                if leaf == name then
                    for _, e in ipairs(list) do
                        if e.class == "RemoteEvent" then
                            return slots[e.slot]
                        end
                    end
                end
            end
            return nil
        end
        local anchor, dual = nil, 0
        for _, list in pairs(byLeaf) do
            if #list > 1 then
                dual = dual + 1
                for _, e in ipairs(list) do
                    if e.class == "RemoteEvent" then anchor = e.slot end
                end
            end
        end
        if dual ~= 1 or not anchor then return nil end
        local t = slots[anchor - 1]
        if t and t.ClassName == "RemoteEvent" and t.Parent == Net then return t end
        return nil
    end

    local remoteCache = {}
    local function getRemote(name)
        local key = name or "__UseItem"
        if remoteCache[key] and remoteCache[key].Parent then return remoteCache[key] end
        local r = resolveRemote(name)
        if r then remoteCache[key] = r end
        return r
    end

    _G.Net = {
        RemoteEvent = function(_, name) return getRemote(name) end,
        RemoteFunction = function(_, name) return nil end,
        UnreliableRemoteEvent = function(_, name) return nil end,
    }

    _G.sabcomFireGrapple2 = function(pos)
        local r = getRemote()
        if not r then return false end
        local char = player.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then return false end
        pos = pos or (hrp.Position + workspace.CurrentCamera.CFrame.LookVector * 100)
        local dist = (pos - hrp.Position).Magnitude
        local a = math.clamp(dist / 120, 0, 2)
        return pcall(function()
            r:FireServer(a, pos)
        end)
    end
    _G.sabcomGrappleReady = function()
        return getRemote() ~= nil
    end
end

_G.sabcomInstantClone = function()
    local c = player.Character
    if not c then return end
    local h = c:FindFirstChildOfClass("Humanoid")
    if not h then return end
    local cl = player.Backpack:FindFirstChild("Quantum Cloner") or c:FindFirstChild("Quantum Cloner")
    if not cl then return end
    pcall(function() h:UnequipTools() end)
    task.wait()
    if cl.Parent ~= c then h:EquipTool(cl); task.wait() end
    local tf = playerGui:FindFirstChild("ToolsFrames")
    local qc = tf and tf:FindFirstChild("QuantumCloner")
    local tb = qc and qc:FindFirstChild("TeleportToClone")
    if not tb then return end

    _G.isCloning = true
    cl:Activate()
    task.wait(0.05)
    tb.Visible = true

    pcall(function() firesignal(tb.MouseButton1Click) end)
    pcall(function() firesignal(tb.MouseButton1Up) end)
    pcall(function() firesignal(tb.Activated) end)

    task.delay(0.55, function() _G.isCloning = false end)
end

local function disableHRPAntiCheat()
    local char = player.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    if not getconnections then return end
    for _, sigName in ipairs({"CFrame", "Position"}) do
        local sig = hrp:GetPropertyChangedSignal(sigName)
        if sig then
            for _, conn in ipairs(getconnections(sig)) do
                if conn.Enabled and conn.Function then
                    local ok, info = pcall(getinfo, conn.Function)
                    if ok and info and tostring(info.source):find("ReplicatedFirst") and tostring(info.source):find("test") then
                        pcall(function() conn:Disable() end)
                    end
                end
            end
        end
    end
end
task.spawn(function()
    while true do
        task.wait(2)
        pcall(disableHRPAntiCheat)
    end
end)

pcall(function()
    local function silenceFTUE()
        if not (getconnections and getinfo) then return end
        local RS = game:GetService("RunService")
        for _, sig in ipairs({ RS.RenderStepped, RS.Heartbeat, RS.Stepped }) do
            pcall(function()
                for _, c in ipairs(getconnections(sig)) do
                    local f = c.Function
                    if f and c.Enabled then
                        local ok, info = pcall(getinfo, f)
                        if ok and info and tostring(info.source):find("FTUEController") then
                            pcall(function() c:Disable() end)
                        end
                    end
                end
            end)
        end
    end
    LogService.MessageOut:Connect(function(msg, msgType)
        if msgType == Enum.MessageType.MessageError and tostring(msg):find("TutorialArrow", 1, true) then
            task.defer(silenceFTUE)
        end
    end)
end)

local function instantReset()
    pcall(_G.sabcomFireGrapple2)
end
local resetEvent = Instance.new("BindableEvent")
resetEvent.Event:Connect(instantReset)
task.spawn(function()
    for _ = 1, 12 do
        pcall(function()
            StarterGui:SetCore("ResetButtonCallback", resetEvent)
        end)
        task.wait(1)
    end
end)

local function godMode(plr)
    plr.CharacterAdded:Connect(function(character)
        local humanoid = character:WaitForChild("Humanoid")
        humanoid:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
        humanoid:GetPropertyChangedSignal("Health"):Connect(function()
            if humanoid.Health < humanoid.MaxHealth then
                humanoid.Health = humanoid.MaxHealth
            end
        end)
        local deathConnection
        deathConnection = humanoid.Died:Connect(function()
            deathConnection:Disconnect()
            task.wait(0.05)
            if humanoid and humanoid.Parent then
                humanoid.Health = humanoid.MaxHealth
                humanoid:ChangeState(Enum.HumanoidStateType.Running)
                deathConnection = humanoid.Died:Connect(function()
                    task.wait(0.05)
                    if humanoid and humanoid.Parent then
                        humanoid.Health = humanoid.MaxHealth
                        humanoid:ChangeState(Enum.HumanoidStateType.Running)
                    end
                end)
            end
        end)
    end)
end

for _, plr in ipairs(Players:GetPlayers()) do
    godMode(plr)
end
Players.PlayerAdded:Connect(godMode)

local screenGui = new("ScreenGui",{Name="ZAXHUBMINI",ResetOnSpawn=false,ZIndexBehavior=Enum.ZIndexBehavior.Sibling,IgnoreGuiInset=true},playerGui)
local blur      = new("BlurEffect",{Name="ZAXHUBMINIBlur",Size=0},Lighting)
local openModalCount = 0
local function pushBlur() openModalCount=openModalCount+1; tw(blur,{Size=18},0.25) end
local function popBlur()  openModalCount=math.max(0,openModalCount-1); if openModalCount==0 then tw(blur,{Size=0},0.2) end end

local mainPanel = new("Frame",{
    Name="MainPanel", AnchorPoint=Vector2.new(0.5,0.5),
    Position=UDim2.new(0.5,0,0.5,0),
    Size=UDim2.new(0,192,0,0), AutomaticSize=Enum.AutomaticSize.Y,
    BackgroundColor3=Theme.Panel, BorderSizePixel=0, ZIndex=2,
},screenGui)
corner(mainPanel,14); strokeOn(mainPanel,Theme.Border,1); bgDepthGrad(mainPanel)
new("UIPadding",{PaddingTop=UDim.new(0,10),PaddingBottom=UDim.new(0,10),PaddingLeft=UDim.new(0,10),PaddingRight=UDim.new(0,10)},mainPanel)
new("UIListLayout",{SortOrder=Enum.SortOrder.LayoutOrder,Padding=UDim.new(0,6)},mainPanel)
local guiScaleObj = new("UIScale",{Scale=Config.guiScale},mainPanel)

local function applyPanelPosition()
    local pp=Config.panelPosition
    mainPanel.Position=UDim2.new(pp.X,pp.XOffset,pp.Y,pp.YOffset)
end
applyPanelPosition()

local header = new("Frame",{Size=UDim2.new(1,0,0,18),BackgroundTransparency=1,LayoutOrder=1},mainPanel)
local title  = new("TextLabel",{Text="ZAXHUB mini",Font=FONT_BOLD,TextSize=13,TextColor3=Color3.new(1,1,1),TextXAlignment=Enum.TextXAlignment.Left,BackgroundTransparency=1,Size=UDim2.new(0.7,0,1,0)},header)
accentGrad(title,0)
local gearBtn = new("TextButton",{Text="⚙",Font=FONT_BOLD,TextSize=14,TextColor3=Theme.TextDim,AutoButtonColor=false,BackgroundTransparency=1,Size=UDim2.new(0,20,0,20),Position=UDim2.new(1,-20,0.5,-10),ZIndex=3},header)
gearBtn.MouseEnter:Connect(function() tw(gearBtn,{TextColor3=Theme.Text},0.15) end)
gearBtn.MouseLeave:Connect(function() tw(gearBtn,{TextColor3=Theme.TextDim},0.15) end)

do
    local dragHandle = new("Frame",{Size=UDim2.new(1,-24,1,0),BackgroundTransparency=1,Active=true,ZIndex=2},header)
    local dragging,startInput,startPos
    local function isPtr(i) return i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch end
    dragHandle.InputBegan:Connect(function(i) if not isPtr(i) then return end dragging=true; startInput=i.Position; startPos=mainPanel.Position end)
    UserInputService.InputChanged:Connect(function(i)
        if not dragging then return end
        if i.UserInputType~=Enum.UserInputType.MouseMovement and i.UserInputType~=Enum.UserInputType.Touch then return end
        local d=i.Position-startInput
        mainPanel.Position=UDim2.new(startPos.X.Scale,startPos.X.Offset+d.X,startPos.Y.Scale,startPos.Y.Offset+d.Y)
    end)
    UserInputService.InputEnded:Connect(function(i)
        if isPtr(i) and dragging then
            dragging=false
            local pos=mainPanel.Position
            Config.panelPosition={X=pos.X.Scale,XOffset=pos.X.Offset,Y=pos.Y.Scale,YOffset=pos.Y.Offset}
            saveConfig()
        end
    end)
end

local manualRow = new("Frame",{Size=UDim2.new(1,0,0,28),BackgroundColor3=Theme.PanelField,LayoutOrder=2},mainPanel)
corner(manualRow,8)
local manualTPBtn = new("TextButton",{Text="Manual TP",Font=FONT_BOLD,TextSize=12,TextColor3=Theme.Text,AutoButtonColor=false,BackgroundTransparency=1,Position=UDim2.new(0,10,0,0),Size=UDim2.new(1,-70,1,0)},manualRow)
manualTPBtn.MouseButton1Down:Connect(function() tw(manualTPBtn,{TextColor3=Theme.AccentB},0.08) end)
manualTPBtn.MouseButton1Up:Connect(function()   tw(manualTPBtn,{TextColor3=Theme.Text},0.1)    end)

local keybindBtn = new("TextButton",{Text=Config.keybind,Font=FONT_BOLD,TextSize=10,TextColor3=Theme.Text,AutoButtonColor=false,BackgroundColor3=Theme.PanelField2,Size=UDim2.new(0,40,0,20),Position=UDim2.new(1,-48,0.5,-10)},manualRow)
corner(keybindBtn,6); local keybindStroke = strokeOn(keybindBtn,Theme.Border,1)
local keybindListening = false
local function setKeybindListening(on)
    keybindListening=on
    if on then keybindBtn.Text="..."; tw(keybindBtn,{BackgroundColor3=Theme.PanelField},0.15); keybindStroke.Color=Theme.AccentB; keybindStroke.Thickness=2
    else       tw(keybindBtn,{BackgroundColor3=Theme.PanelField2},0.15); keybindStroke.Color=Theme.Border; keybindStroke.Thickness=1 end
end
keybindBtn.MouseButton1Click:Connect(function() if not keybindListening then setKeybindListening(true) end end)
UserInputService.InputBegan:Connect(function(input,_)
 
