--> Overdrive H On Top.

local shared = odh_shared_plugins

local Players          = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService       = game:GetService("RunService")
local TweenService     = game:GetService("TweenService")
local player           = Players.LocalPlayer
local Camera           = workspace.CurrentCamera
local playerGui        = player:WaitForChild("PlayerGui")

local CONFIG = {
    FovGuiName = "AimLockPrime_FOV",
}

local tab = shared.CreateTab("Aim-LockPrime", "/n12217915-source/BATATA-CENTRAL-1/refs/heads/main/icon")

local SaveConfig = (function()
    local FOLDER    = "Batata Central"
    local SUBFOLDER = "Arquivos Secretos"
    local FILENAME  = "AimLockPrime_Config.json"
    local FILE_PATH = FOLDER .. "/" .. SUBFOLDER .. "/" .. FILENAME

    local data, defaults = {}, {}
    local saveQueued = false
    local FS_OK     = (type(writefile) == "function" and type(readfile) == "function" and type(isfile) == "function")
    local FOLDER_OK = (type(makefolder) == "function")

    if FS_OK and FOLDER_OK then
        pcall(function()
            pcall(makefolder, FOLDER)
            pcall(makefolder, FOLDER .. "/" .. SUBFOLDER)
        end)
    end

    local function jsonEncode(t)
        local function esc(s)
            return s:gsub('[%c"\\]', function(c)
                if c == '"' then return '\\"' end
                if c == '\\' then return '\\\\' end
                if c == '\n' then return '\\n' end
                if c == '\r' then return '\\r' end
                if c == '\t' then return '\\t' end
                return string.format('\\u%04x', c:byte())
            end)
        end
        local function enc(v)
            local ty = type(v)
            if ty == "nil" then return "null"
            elseif ty == "boolean" then return tostring(v)
            elseif ty == "number" then return tostring(v)
            elseif ty == "string" then return '"'..esc(v)..'"'
            elseif ty == "table" then
                local isArray = #v > 0
                local parts = {}
                if isArray then
                    for _, x in ipairs(v) do parts[#parts+1] = enc(x) end
                    return "["..table.concat(parts, ",").."]"
                else
                    for k, x in pairs(v) do
                        parts[#parts+1] = '"'..esc(tostring(k))..'":'..enc(x)
                    end
                    return "{"..table.concat(parts, ",").."}"
                end
            end
            return "null"
        end
        return enc(t)
    end

    local function jsonDecode(s)
        local pos = 1
        local function skip() while pos <= #s and s:sub(pos,pos):match("%s") do pos = pos + 1 end end
        local function parse()
            skip()
            local c = s:sub(pos,pos)
            if c == '"' then
                pos = pos + 1
                local out = {}
                while pos <= #s do
                    local ch = s:sub(pos,pos)
                    if ch == '\\' then
                        local nxt = s:sub(pos+1,pos+1)
                        if nxt == 'n' then out[#out+1] = '\n'
                        elseif nxt == 't' then out[#out+1] = '\t'
                        elseif nxt == 'r' then out[#out+1] = '\r'
                        else out[#out+1] = nxt end
                        pos = pos + 2
                    elseif ch == '"' then pos = pos + 1 break
                    else out[#out+1] = ch pos = pos + 1 end
                end
                return table.concat(out)
            elseif c == '{' then
                pos = pos + 1
                local obj = {}
                skip()
                if s:sub(pos,pos) == '}' then pos = pos + 1 return obj end
                while true do
                    skip()
                    local k = parse()
                    skip()
                    pos = pos + 1
                    obj[k] = parse()
                    skip()
                    local n = s:sub(pos,pos)
                    pos = pos + 1
                    if n == '}' then break end
                end
                return obj
            elseif c == '[' then
                pos = pos + 1
                local arr = {}
                skip()
                if s:sub(pos,pos) == ']' then pos = pos + 1 return arr end
                while true do
                    arr[#arr+1] = parse()
                    skip()
                    local n = s:sub(pos,pos)
                    pos = pos + 1
                    if n == ']' then break end
                end
                return arr
            elseif c == 't' then pos = pos + 4 return true
            elseif c == 'f' then pos = pos + 5 return false
            elseif c == 'n' then pos = pos + 4 return nil
            else
                local num = s:match("^-?%d+%.?%d*[eE]?[-+]?%d*", pos)
                if not num then return nil end
                pos = pos + #num
                return tonumber(num)
            end
        end
        local ok, res = pcall(parse)
        return ok and res or {}
    end

    local function load()
        if not FS_OK then return end
        local exists = false
        pcall(function() exists = isfile(FILE_PATH) end)
        if not exists then return end
        local ok, content = pcall(readfile, FILE_PATH)
        if not ok or not content or content == "" then return end
        local ok2, parsed = pcall(jsonDecode, content)
        if ok2 and type(parsed) == "table" then
            for k, v in pairs(parsed) do data[k] = v end
        end
    end

    local function save()
        if not FS_OK then return end
        local ok = pcall(writefile, FILE_PATH, jsonEncode(data))
        if not ok and FOLDER_OK then
            pcall(makefolder, FOLDER)
            pcall(makefolder, FOLDER .. "/" .. SUBFOLDER)
            pcall(writefile, FILE_PATH, jsonEncode(data))
        end
    end

    local function queueSave()
        if saveQueued then return end
        saveQueued = true
        task.delay(0.6, function()
            saveQueued = false
            save()
        end)
    end

    load()

    return {
        register = function(key, default)
            defaults[key] = default
            if data[key] == nil then data[key] = default end
            return data[key]
        end,
        get = function(key) return data[key] end,
        set = function(key, value)
            data[key] = value
            queueSave()
        end,
        saveNow = save,
        reset = function()
            data = {}
            for k, v in pairs(defaults) do data[k] = v end
            save()
        end,
        FS_OK = FS_OK,
        PATH  = FILE_PATH,
    }
end)()

local aimState = {
    Enabled     = SaveConfig.register("aim.enabled",    false),
    AutoLock    = SaveConfig.register("aim.autolock",   true),
    FOV         = SaveConfig.register("aim.fov",        120),
    Smooth      = SaveConfig.register("aim.smooth",     0.25),
    AimPart     = SaveConfig.register("aim.part",       "Head"),
    Prediction  = SaveConfig.register("aim.prediction", 0.15),
    ShowFOV     = SaveConfig.register("aim.showfov",    true),
    Highlight   = SaveConfig.register("aim.highlight",  true),
    TeamCheck   = SaveConfig.register("aim.teamcheck",  true),
    DragOutDist = SaveConfig.register("aim.dragout",    40),
    KnifeOnly   = SaveConfig.register("aim.knifeonly",  true),
    HoldKey     = Enum.KeyCode.E,
}

local stickyTarget, highlightObj = nil, nil
local fovFrameRef = nil

local function aimIsAlive(plr)
    local c = plr.Character
    if not c then return false end
    local h = c:FindFirstChildOfClass("Humanoid")
    return h and h.Health > 0
end

local function hasKnifeTool(plr)
    local c = plr.Character
    if not c then return false end
    for _, inst in ipairs(c:GetChildren()) do
        if inst:IsA("Tool") and inst.Name:lower():find("knife", 1, true) then
            return true
        end
    end
    local bp = plr:FindFirstChild("Backpack")
    if bp then
        for _, inst in ipairs(bp:GetChildren()) do
            if inst:IsA("Tool") and inst.Name:lower():find("knife", 1, true) then
                return true
            end
        end
    end
    return false
end

local function aimIsEnemy(plr)
    if plr == player then return false end
    if not aimIsAlive(plr) then return false end
    if aimState.KnifeOnly and not hasKnifeTool(plr) then return false end
    if not aimState.TeamCheck then return true end
    return plr.Team ~= player.Team
end

local function aimGetPart(plr)
    local c = plr.Character
    if not c then return nil end
    if aimState.AimPart == "Head" then return c:FindFirstChild("Head")
    elseif aimState.AimPart == "Torso" then return c:FindFirstChild("Torso") or c:FindFirstChild("UpperTorso")
    else return c:FindFirstChild("HumanoidRootPart") end
end

local function aimWorldToScreen(pos)
    local sp, on = Camera:WorldToViewportPoint(pos)
    return Vector2.new(sp.X, sp.Y), on
end

local function aimCenter()
    local vp = Camera.ViewportSize
    return Vector2.new(vp.X/2, vp.Y/2)
end

local function aimIsVisible(part)
    local origin = Camera.CFrame.Position
    local dir = part.Position - origin
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = { player.Character, part.Parent, Camera }
    params.IgnoreWater = true
    local r = workspace:Raycast(origin, dir, params)
    return r == nil or r.Instance:IsDescendantOf(part.Parent)
end

local function aimClosest()
    local center = aimCenter()
    local best, bestDist = nil, aimState.FOV
    for _, plr in ipairs(Players:GetPlayers()) do
        if aimIsEnemy(plr) then
            local part = aimGetPart(plr)
            if part then
                local sp, on = aimWorldToScreen(part.Position)
                if on then
                    local d = (sp - center).Magnitude
                    if d < bestDist and aimIsVisible(part) then
                        bestDist = d; best = plr
                    end
                end
            end
        end
    end
    return best
end

local function aimPredict(part)
    if aimState.Prediction <= 0 then return part.Position end
    return part.Position + part.AssemblyLinearVelocity * aimState.Prediction
end

local function aimClearHighlight()
    if highlightObj then highlightObj:Destroy(); highlightObj = nil end
end

local function aimApplyHighlight(plr)
    aimClearHighlight()
    if not aimState.Highlight or not plr or not plr.Character then return end
    local h = Instance.new("Highlight")
    h.FillColor = Color3.fromRGB(255,60,60)
    h.OutlineColor = Color3.new(1,1,1)
    h.FillTransparency = 0.6
    h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    h.Adornee = plr.Character
    h.Parent = plr.Character
    highlightObj = h
end

local function aimShouldRelease(plr)
    if not plr or not plr.Character then return true end
    if not aimIsAlive(plr) then return true end
    if aimState.KnifeOnly and not hasKnifeTool(plr) then return true end
    local part = aimGetPart(plr)
    if not part then return true end
    local sp, on = aimWorldToScreen(part.Position)
    if not on then return true end
    return (UserInputService:GetMouseLocation() - sp).Magnitude > aimState.DragOutDist
end

local fovGui = Instance.new("ScreenGui")
fovGui.Name = CONFIG.FovGuiName
fovGui.ResetOnSpawn = false
fovGui.IgnoreGuiInset = true
fovGui.Parent = playerGui

local fovFrame = Instance.new("Frame")
fovFrame.AnchorPoint = Vector2.new(0.5, 0.5)
fovFrame.Position = UDim2.fromScale(0.5, 0.5)
fovFrame.BackgroundTransparency = 1
fovFrame.Size = UDim2.fromOffset(aimState.FOV * 2, aimState.FOV * 2)
fovFrame.Visible = aimState.ShowFOV
fovFrame.Parent = fovGui

local fovStroke = Instance.new("UIStroke", fovFrame)
fovStroke.Thickness = 1
fovStroke.Color = Color3.new(1, 1, 1)
fovStroke.Transparency = 0.5

Instance.new("UICorner", fovFrame).CornerRadius = UDim.new(1, 0)
fovFrameRef = fovFrame

local HOLD_TO_DRAG = 0.3
local BTN_SIZE = 64

local mobileGui = Instance.new("ScreenGui")
mobileGui.Name = "AimLockPrimeMobile"
mobileGui.ResetOnSpawn = false
mobileGui.IgnoreGuiInset = true
mobileGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
mobileGui.Parent = playerGui

local savedBtnX = SaveConfig.register("mobile.btnX", 20)
local savedBtnY = SaveConfig.register("mobile.btnY", 200)

local mobileBtn = Instance.new("TextButton")
mobileBtn.Name = "AimToggle"
mobileBtn.Size = UDim2.fromOffset(BTN_SIZE, BTN_SIZE)
mobileBtn.Position = UDim2.fromOffset(savedBtnX, savedBtnY)
mobileBtn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
mobileBtn.BackgroundTransparency = 0.8
mobileBtn.Text = "AIM\nOFF"
mobileBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
mobileBtn.Font = Enum.Font.GothamBold
mobileBtn.TextSize = 11
mobileBtn.BorderSizePixel = 0
mobileBtn.AutoButtonColor = false
mobileBtn.ZIndex = 100
mobileBtn.Parent = mobileGui

Instance.new("UICorner", mobileBtn).CornerRadius = UDim.new(1, 0)

local mobileStroke = Instance.new("UIStroke", mobileBtn)
mobileStroke.Color = Color3.fromRGB(255, 40, 40)
mobileStroke.Thickness = 2
mobileStroke.Transparency = 0

task.spawn(function()
    while mobileBtn.Parent do
        TweenService:Create(mobileStroke, TweenInfo.new(1.2, Enum.EasingStyle.Sine), {Transparency = 0.4}):Play()
        task.wait(1.2)
        TweenService:Create(mobileStroke, TweenInfo.new(1.2, Enum.EasingStyle.Sine), {Transparency = 0}):Play()
        task.wait(1.2)
    end
end)

local function setAimEnabled(v)
    aimState.Enabled = v
    SaveConfig.set("aim.enabled", v)
    mobileBtn.Text = v and "AIM\nON" or "AIM\nOFF"
    mobileBtn.TextColor3 = v and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
    mobileStroke.Color = v and Color3.fromRGB(255, 40, 40) or Color3.fromRGB(120, 40, 40)
end

local holdingBtn, draggingBtn = false, false
local holdStart, dragStartPos, btnStartPos = 0, nil, nil

mobileBtn.InputBegan:Connect(function(input)
    if input.UserInputType ~= Enum.UserInputType.MouseButton1
    and input.UserInputType ~= Enum.UserInputType.Touch then return end
    holdingBtn = true
    draggingBtn = false
    holdStart = os.clock()
    dragStartPos = Vector2.new(input.Position.X, input.Position.Y)
    btnStartPos  = Vector2.new(mobileBtn.AbsolutePosition.X, mobileBtn.AbsolutePosition.Y)
end)

UserInputService.InputChanged:Connect(function(input)
    if not holdingBtn then return end
    if input.UserInputType ~= Enum.UserInputType.MouseMovement
    and input.UserInputType ~= Enum.UserInputType.Touch then return end

    if not draggingBtn and (os.clock() - holdStart) >= HOLD_TO_DRAG then
        draggingBtn = true
    end

    if draggingBtn and dragStartPos and btnStartPos then
        local current = Vector2.new(input.Position.X, input.Position.Y)
        local delta = current - dragStartPos
        local raw = btnStartPos + delta
        local vp = Camera.ViewportSize
        local clamped = Vector2.new(
            math.clamp(raw.X, 0, vp.X - BTN_SIZE),
            math.clamp(raw.Y, 0, vp.Y - BTN_SIZE)
        )
        mobileBtn.Position = UDim2.fromOffset(clamped.X, clamped.Y)
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType ~= Enum.UserInputType.MouseButton1
    and input.UserInputType ~= Enum.UserInputType.Touch then return end
    if not holdingBtn then return end

    local heldFor = os.clock() - holdStart
    holdingBtn = false

    if draggingBtn then
        draggingBtn = false
        SaveConfig.set("mobile.btnX", mobileBtn.Position.X.Offset)
        SaveConfig.set("mobile.btnY", mobileBtn.Position.Y.Offset)
    else
        if heldFor < HOLD_TO_DRAG then
            setAimEnabled(not aimState.Enabled)
        end
    end

    dragStartPos, btnStartPos = nil, nil
end)

local aimSection = tab:AddSection("AIM LOCK", "Mira com FOV, prediction e drag-out")

setAimEnabled(aimState.Enabled)

local toggleAimRef
toggleAimRef = aimSection:AddToggle("Aim Lock", function(v)
    setAimEnabled(v)
end)

aimSection:AddToggle("Auto Lock", function(v)
    aimState.AutoLock = v
    SaveConfig.set("aim.autolock", v)
end)

local knifeToggleRef
knifeToggleRef = aimSection:AddToggle("Only Knife Holders", function(v)
    aimState.KnifeOnly = v
    SaveConfig.set("aim.knifeonly", v)
end)

aimSection:AddToggle("Highlight", function(v)
    aimState.Highlight = v
    SaveConfig.set("aim.highlight", v)
    if not v then aimClearHighlight() end
end)

aimSection:AddToggle("Checar Time", function(v)
    aimState.TeamCheck = v
    SaveConfig.set("aim.teamcheck", v)
end)

aimSection:AddToggle("Mostrar FOV", function(v)
    aimState.ShowFOV = v
    SaveConfig.set("aim.showfov", v)
    if fovFrameRef then fovFrameRef.Visible = v end
end)

aimSection:AddSlider("FOV", 20, 500, aimState.FOV, function(v)
    aimState.FOV = v
    SaveConfig.set("aim.fov", v)
    if fovFrameRef then fovFrameRef.Size = UDim2.fromOffset(v * 2, v * 2) end
end)

aimSection:AddSlider("Smooth x100", 0, 95, math.floor(aimState.Smooth * 100), function(v)
    aimState.Smooth = v / 100
    SaveConfig.set("aim.smooth", aimState.Smooth)
end)

aimSection:AddSlider("Prediction x100", 0, 80, math.floor(aimState.Prediction * 100), function(v)
    aimState.Prediction = v / 100
    SaveConfig.set("aim.prediction", aimState.Prediction)
end)

aimSection:AddSlider("DragOut px", 5, 150, aimState.DragOutDist, function(v)
    aimState.DragOutDist = v
    SaveConfig.set("aim.dragout", v)
end)

local aimPartDropdown = aimSection:AddDropdown("Aim Part", {"Head", "HumanoidRootPart", "Torso"}, function(sel)
    aimState.AimPart = sel
    SaveConfig.set("aim.part", sel)
end)

RunService.RenderStepped:Connect(function()
    if not aimState.Enabled then
        stickyTarget = nil; aimClearHighlight(); return
    end
    local wantsLock = aimState.AutoLock or UserInputService:IsKeyDown(aimState.HoldKey)
    if not wantsLock then
        stickyTarget = nil
    else
        if stickyTarget and (not aimIsAlive(stickyTarget) or aimShouldRelease(stickyTarget)) then
            stickyTarget = nil
        end
        if not stickyTarget then stickyTarget = aimClosest() end
    end
    if not stickyTarget or not aimIsAlive(stickyTarget) then aimClearHighlight(); return end
    if aimState.KnifeOnly and not hasKnifeTool(stickyTarget) then
        stickyTarget = nil; aimClearHighlight(); return
    end
    local part = aimGetPart(stickyTarget)
    if not part or not aimIsVisible(part) then
        stickyTarget = nil; aimClearHighlight(); return
    end
    aimApplyHighlight(stickyTarget)
    local targetPos = aimPredict(part)
    local desired = CFrame.new(Camera.CFrame.Position, targetPos)
    Camera.CFrame = Camera.CFrame:Lerp(desired, math.clamp(1 - aimState.Smooth, 0, 1))
end)

task.defer(function()
    task.wait(0.4)
    if aimState.Enabled and toggleAimRef then
        pcall(function() toggleAimRef() end)
    end
    if aimState.KnifeOnly and knifeToggleRef then
        pcall(function() knifeToggleRef() end)
    end
    if aimPartDropdown and aimPartDropdown.Select then
        pcall(function() aimPartDropdown:Select(aimState.AimPart) end)
    end
    setAimEnabled(SaveConfig.get("aim.enabled") == true)
end)

shared.Notify("Aim-LockPrime carregado!", 3)
print("[Aim-LockPrime] Plugin carregado.")
