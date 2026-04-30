--[[
    LIQUID BOUNCE BLINK v20.0 — JJS EDITION
    Complete rewrite with full settings GUI

    Default keybinds:
    [M] = Time Stop (toggle)
    [N] = Fake Lag (hold = freeze, release = rubberband)
    [B] = Force sync

    GUI: Drag the window anywhere | Customize everything
]]

local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
local Camera      = workspace.CurrentCamera

-- ================================================
-- CONFIG (editable via GUI)
-- ================================================
local Config = {
    ToggleKey       = Enum.KeyCode.M,
    FakeLagKey      = Enum.KeyCode.N,
    SyncKey         = Enum.KeyCode.B,
    MoveSpeed       = 24,
    FloatSpeed      = 16,
    ShowEffects     = true,
    -- Freeze method: "hook" or "loop"
    FreezeMethod    = "hook",
    EffectColor     = Color3.fromRGB(0,   150, 255),
    LagColor        = Color3.fromRGB(255,  80,   0),
    SyncColor       = Color3.fromRGB(0,   255, 100),
    CloneColorV     = Color3.fromRGB(100, 180, 255),
    CloneColorB     = Color3.fromRGB(255, 140,  60),
}

-- ================================================
-- STATE
-- ================================================
local State = {
    Character    = nil,
    Humanoid     = nil,
    RootPart     = nil,
    Active       = false,
    Mode         = nil,
    StartTime    = 0,
    LockedCFrame = nil,
    VClone       = nil,
    BClone       = nil,
    MoveConn     = nil,
    NoclipConn   = nil,
    TimerThread  = nil,
}

local function Log(msg) print("✓ [Blink] " .. msg) end
local function GetElapsed() return math.floor((tick() - State.StartTime) * 10) / 10 end

-- ================================================
-- EFFECTS
-- ================================================
local function SpawnEffect(pos, color)
    if not Config.ShowEffects then return end
    task.spawn(function()
        pcall(function()
            local p = Instance.new("Part")
            p.Anchored = true; p.CanCollide = false
            p.Shape = Enum.PartType.Ball; p.Size = Vector3.new(1,1,1)
            p.Color = color; p.Material = Enum.Material.Neon
            p.Transparency = 0; p.CFrame = CFrame.new(pos)
            p.TopSurface = Enum.SurfaceType.Smooth
            p.BottomSurface = Enum.SurfaceType.Smooth
            p.Parent = workspace
            local t = 0
            local c; c = RunService.Heartbeat:Connect(function(dt)
                t += dt
                if t >= 1.2 then c:Disconnect(); pcall(function() p:Destroy() end) return end
                p.Transparency = t/1.2
                local s = 1+5*(t/1.2); p.Size = Vector3.new(s,s,s)
            end)
        end)
    end)
end

-- ================================================
-- CLONE
-- ================================================
local function BuildCloneAt(cf, color)
    local m = Instance.new("Model"); m.Name = "LBClone"
    local function P(sz, off)
        local p = Instance.new("Part")
        p.Anchored = true; p.CanCollide = false
        p.Size = sz; p.Color = color
        p.Material = Enum.Material.Neon; p.Transparency = 0.15
        p.TopSurface = Enum.SurfaceType.Smooth
        p.BottomSurface = Enum.SurfaceType.Smooth
        p.CFrame = cf * CFrame.new(off); p.Parent = m
    end
    P(Vector3.new(2,   2,   1  ), Vector3.new( 0,    0,   0))
    P(Vector3.new(1.2, 1.2, 1.2), Vector3.new( 0,    1.6, 0))
    P(Vector3.new(0.8, 2,   0.8), Vector3.new(-1.4,  0,   0))
    P(Vector3.new(0.8, 2,   0.8), Vector3.new( 1.4,  0,   0))
    P(Vector3.new(0.8, 2,   0.8), Vector3.new(-0.6, -2,   0))
    P(Vector3.new(0.8, 2,   0.8), Vector3.new( 0.6, -2,   0))
    m.Parent = workspace
    return m
end

local function Kill(obj) if obj then pcall(function() obj:Destroy() end) end end

-- ================================================
-- FLOOR
-- ================================================
local function GetFloorY(pos)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Blacklist
    params.FilterDescendantsInstances = {State.Character, State.VClone, State.BClone}
    local r = workspace:Raycast(pos, Vector3.new(0,-100,0), params)
    return r and (r.Position.Y + 3) or nil
end

-- ================================================
-- HOOKMETAMETHOD FREEZE
-- ================================================
local frozenRoot   = nil
local ourWrite     = false
local origNewIndex = nil
local hookInstalled = false

local function InstallHook()
    if hookInstalled then return true end
    if not hookmetamethod then
        Log("⚠️  hookmetamethod not available — using loop mode only")
        return false
    end
    local ok = pcall(function()
        origNewIndex = hookmetamethod(game, "__newindex", function(self, key, value)
            if frozenRoot and self == frozenRoot and not ourWrite then
                if key == "CFrame" or key == "Position" then return end
                if key == "Anchored" and value == false   then return end
            end
            return origNewIndex(self, key, value)
        end)
    end)
    hookInstalled = ok
    if ok then Log("✅ hookmetamethod installed") end
    return ok
end

local function SetRootCFrame(root, cf)
    ourWrite = true
    pcall(function() root.CFrame = cf end)
    ourWrite = false
end

-- ================================================
-- NOCLIP
-- ================================================
local function StartNoclip()
    if State.NoclipConn then State.NoclipConn:Disconnect() end
    State.NoclipConn = RunService.Stepped:Connect(function()
        local char = State.Character; if not char then return end
        pcall(function()
            for _, p in pairs(char:GetDescendants()) do
                if p:IsA("BasePart") then p.CanCollide = false end
            end
        end)
    end)
end

local function StopNoclip()
    if State.NoclipConn then State.NoclipConn:Disconnect(); State.NoclipConn = nil end
    pcall(function()
        local char = State.Character; if not char then return end
        for _, p in pairs(char:GetDescendants()) do
            if p:IsA("BasePart") then p.CanCollide = true end
        end
    end)
end

-- ================================================
-- MOVEMENT LOOP
-- ================================================
local function StartMoveLoop()
    if State.MoveConn then State.MoveConn:Disconnect() end
    State.MoveConn = RunService.RenderStepped:Connect(function(dt)
        if not State.Active then return end
        local root = State.RootPart; if not root then return end

        local pos   = root.CFrame.Position
        local camCF = Camera.CFrame
        local f = Vector3.new(camCF.LookVector.X,  0, camCF.LookVector.Z)
        local r = Vector3.new(camCF.RightVector.X, 0, camCF.RightVector.Z)
        if f.Magnitude > 0 then f = f.Unit end
        if r.Magnitude > 0 then r = r.Unit end

        local move = Vector3.zero
        if UserInputService:IsKeyDown(Enum.KeyCode.W) then move += f end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then move -= f end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then move -= r end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then move += r end
        if move.Magnitude > 0 then pos += move.Unit * Config.MoveSpeed * dt end

        if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
            pos += Vector3.new(0, Config.FloatSpeed * dt, 0)
        elseif UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
            pos -= Vector3.new(0, Config.FloatSpeed * dt, 0)
        else
            local fy = GetFloorY(pos)
            if fy and pos.Y > fy then
                local fall = math.min((pos.Y-fy)*5, 60)
                pos -= Vector3.new(0, fall*dt, 0)
                if pos.Y < fy then pos = Vector3.new(pos.X, fy, pos.Z) end
            end
        end

        local look = pos + Vector3.new(camCF.LookVector.X, 0, camCF.LookVector.Z)
        State.LockedCFrame = CFrame.new(pos, look)
        frozenRoot = root

        if Config.FreezeMethod == "hook" then
            SetRootCFrame(root, State.LockedCFrame)
        else
            pcall(function() root.CFrame = State.LockedCFrame end)
        end

        pcall(function()
            root.AssemblyLinearVelocity  = Vector3.zero
            root.AssemblyAngularVelocity = Vector3.zero
        end)
    end)
end

local function StopMoveLoop()
    if State.MoveConn then State.MoveConn:Disconnect(); State.MoveConn = nil end
end

-- ================================================
-- TIMER
-- ================================================
local function StartTimer()
    if State.TimerThread then pcall(task.cancel, State.TimerThread) end
    State.TimerThread = task.spawn(function()
        while State.Active do
            task.wait(1)
            if State.Active then
                local root = State.RootPart
                Log(string.format("⏱  %s | %.0fs | %.1f %.1f %.1f",
                    State.Mode == "desync" and "TIME STOP" or "FAKE LAG",
                    GetElapsed(),
                    root and root.Position.X or 0,
                    root and root.Position.Y or 0,
                    root and root.Position.Z or 0))
            end
        end
    end)
end

local function StopTimer()
    if State.TimerThread then pcall(task.cancel, State.TimerThread); State.TimerThread = nil end
end

-- ================================================
-- ACTIVATE / DEACTIVATE
-- ================================================
local function Activate(mode)
    if State.Active then return end
    local root = State.RootPart; if not root then return end

    State.Active       = true
    State.Mode         = mode
    State.StartTime    = tick()
    State.LockedCFrame = root.CFrame
    frozenRoot         = root

    pcall(function()
        local h = State.Humanoid; if not h then return end
        h.PlatformStand = true; h.WalkSpeed = 0; h.JumpPower = 0
        h:ChangeState(Enum.HumanoidStateType.Physics)
    end)

    StartMoveLoop(); StartNoclip(); StartTimer()

    if mode == "desync" then
        State.VClone = BuildCloneAt(root.CFrame, Config.CloneColorV)
        SpawnEffect(root.Position, Config.EffectColor)
        Log("⏸  TIME STOP ON [" .. Config.ToggleKey.Name .. "] | Method: " .. Config.FreezeMethod)
    else
        State.BClone = BuildCloneAt(root.CFrame, Config.CloneColorB)
        SpawnEffect(root.Position, Config.LagColor)
        Log("🟠 FAKE LAG ON [" .. Config.FakeLagKey.Name .. " hold] | Method: " .. Config.FreezeMethod)
    end
end

local function Deactivate()
    if not State.Active then return end
    local elapsed = GetElapsed(); local mode = State.Mode
    State.Active = false; State.Mode = nil; frozenRoot = nil

    StopMoveLoop(); StopNoclip(); StopTimer()

    pcall(function()
        local h = State.Humanoid; if not h then return end
        h.PlatformStand = false; h.AutoRotate = true
        h.WalkSpeed = 16; h.JumpPower = 50
        h:ChangeState(Enum.HumanoidStateType.GettingUp)
    end)

    local root = State.RootPart
    if root then
        SpawnEffect(root.Position, Config.SyncColor)
        Log(string.format("▶  %s OFF | %.1fs | %.1f %.1f %.1f",
            mode == "desync" and "TIME STOP" or "FAKE LAG", elapsed,
            root.Position.X, root.Position.Y, root.Position.Z))
    end

    Kill(State.VClone); State.VClone = nil
    Kill(State.BClone); State.BClone = nil
    State.LockedCFrame = nil
end

local function ForceSync()
    if not State.Active then return end
    Deactivate(); Log("🔄 Force synced")
end

-- ================================================
-- INPUT
-- ================================================
UserInputService.InputBegan:Connect(function(input, _)
    if input.KeyCode == Config.ToggleKey then
        if State.Active and State.Mode == "desync" then Deactivate()
        elseif not State.Active then Activate("desync") end
    elseif input.KeyCode == Config.FakeLagKey then
        if not State.Active then Activate("fakelag") end
    elseif input.KeyCode == Config.SyncKey then
        ForceSync()
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.KeyCode == Config.FakeLagKey then
        if State.Active and State.Mode == "fakelag" then Deactivate() end
    end
end)

-- ================================================
-- CHARACTER
-- ================================================
local function OnCharacterAdded(char)
    if State.Active then
        State.Active = false; frozenRoot = nil
        StopMoveLoop(); StopNoclip(); StopTimer()
    end
    Kill(State.VClone); State.VClone = nil
    Kill(State.BClone); State.BClone = nil
    State.LockedCFrame = nil
    State.Character = char
    State.RootPart  = char:WaitForChild("HumanoidRootPart", 10)
    State.Humanoid  = char:WaitForChild("Humanoid", 10)
    Log("✅ Ready | [" .. Config.ToggleKey.Name .. "] Time Stop | [" .. Config.FakeLagKey.Name .. " hold] Fake Lag | [" .. Config.SyncKey.Name .. "] Sync")
    if State.Humanoid then
        State.Humanoid.Died:Connect(function()
            State.Active = false; frozenRoot = nil
            StopMoveLoop(); StopNoclip(); StopTimer()
            Kill(State.VClone); State.VClone = nil
            Kill(State.BClone); State.BClone = nil
        end)
    end
end

LocalPlayer.CharacterAdded:Connect(OnCharacterAdded)
if LocalPlayer.Character then task.spawn(OnCharacterAdded, LocalPlayer.Character) end

-- ================================================
-- GUI
-- ================================================
local function CreateGUI()
    -- Remove old GUI if exists
    local old = LocalPlayer.PlayerGui:FindFirstChild("LBBlinkGUI")
    if old then old:Destroy() end

    local sg = Instance.new("ScreenGui")
    sg.Name = "LBBlinkGUI"
    sg.ResetOnSpawn = false
    sg.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    sg.Parent = LocalPlayer.PlayerGui

    -- Main window
    local win = Instance.new("Frame")
    win.Name = "Window"
    win.Size = UDim2.new(0, 320, 0, 420)
    win.Position = UDim2.new(0, 20, 0, 20)
    win.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
    win.BorderSizePixel = 0
    win.Parent = sg

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = win

    -- Title bar
    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.new(1, 0, 0, 36)
    titleBar.BackgroundColor3 = Color3.fromRGB(0, 120, 220)
    titleBar.BorderSizePixel = 0
    titleBar.Parent = win

    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 8)
    titleCorner.Parent = titleBar

    local titleFix = Instance.new("Frame")
    titleFix.Size = UDim2.new(1, 0, 0.5, 0)
    titleFix.Position = UDim2.new(0, 0, 0.5, 0)
    titleFix.BackgroundColor3 = Color3.fromRGB(0, 120, 220)
    titleFix.BorderSizePixel = 0
    titleFix.Parent = titleBar

    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1, -40, 1, 0)
    titleLabel.Position = UDim2.new(0, 10, 0, 0)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = "⚡ Liquid Bounce Blink"
    titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    titleLabel.TextSize = 14
    titleLabel.Font = Enum.Font.GothamBold
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.Parent = titleBar

    -- Close/minimize button
    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 28, 0, 28)
    closeBtn.Position = UDim2.new(1, -32, 0, 4)
    closeBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
    closeBtn.Text = "✕"
    closeBtn.TextColor3 = Color3.fromRGB(255,255,255)
    closeBtn.TextSize = 13
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.BorderSizePixel = 0
    closeBtn.Parent = titleBar
    local cc = Instance.new("UICorner"); cc.CornerRadius = UDim.new(0,5); cc.Parent = closeBtn

    -- Status bar
    local statusBar = Instance.new("Frame")
    statusBar.Size = UDim2.new(1, -16, 0, 28)
    statusBar.Position = UDim2.new(0, 8, 0, 42)
    statusBar.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
    statusBar.BorderSizePixel = 0
    statusBar.Parent = win
    local sc = Instance.new("UICorner"); sc.CornerRadius = UDim.new(0,5); sc.Parent = statusBar

    local statusLabel = Instance.new("TextLabel")
    statusLabel.Size = UDim2.new(1, -8, 1, 0)
    statusLabel.Position = UDim2.new(0, 8, 0, 0)
    statusLabel.BackgroundTransparency = 1
    statusLabel.Text = "● IDLE"
    statusLabel.TextColor3 = Color3.fromRGB(150, 150, 150)
    statusLabel.TextSize = 12
    statusLabel.Font = Enum.Font.Gotham
    statusLabel.TextXAlignment = Enum.TextXAlignment.Left
    statusLabel.Parent = statusBar

    -- Scroll content
    local scroll = Instance.new("ScrollingFrame")
    scroll.Size = UDim2.new(1, 0, 1, -76)
    scroll.Position = UDim2.new(0, 0, 0, 76)
    scroll.BackgroundTransparency = 1
    scroll.BorderSizePixel = 0
    scroll.ScrollBarThickness = 3
    scroll.ScrollBarImageColor3 = Color3.fromRGB(0, 120, 220)
    scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
    scroll.Parent = win

    local layout = Instance.new("UIListLayout")
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Padding = UDim.new(0, 4)
    layout.Parent = scroll

    local padding = Instance.new("UIPadding")
    padding.PaddingLeft = UDim.new(0, 8)
    padding.PaddingRight = UDim.new(0, 8)
    padding.PaddingTop = UDim.new(0, 6)
    padding.Parent = scroll

    -- Helper: section label
    local function SectionLabel(text, order)
        local lbl = Instance.new("TextLabel")
        lbl.Size = UDim2.new(1, 0, 0, 22)
        lbl.BackgroundTransparency = 1
        lbl.Text = text
        lbl.TextColor3 = Color3.fromRGB(0, 150, 255)
        lbl.TextSize = 11
        lbl.Font = Enum.Font.GothamBold
        lbl.TextXAlignment = Enum.TextXAlignment.Left
        lbl.LayoutOrder = order
        lbl.Parent = scroll
        return lbl
    end

    -- Helper: row frame
    local function Row(order, h)
        local f = Instance.new("Frame")
        f.Size = UDim2.new(1, 0, 0, h or 32)
        f.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
        f.BorderSizePixel = 0
        f.LayoutOrder = order
        f.Parent = scroll
        local rc = Instance.new("UICorner"); rc.CornerRadius = UDim.new(0,5); rc.Parent = f
        return f
    end

    -- Helper: label in row
    local function RowLabel(parent, text)
        local l = Instance.new("TextLabel")
        l.Size = UDim2.new(0.55, 0, 1, 0)
        l.Position = UDim2.new(0, 8, 0, 0)
        l.BackgroundTransparency = 1
        l.Text = text
        l.TextColor3 = Color3.fromRGB(200, 200, 200)
        l.TextSize = 12
        l.Font = Enum.Font.Gotham
        l.TextXAlignment = Enum.TextXAlignment.Left
        l.Parent = parent
        return l
    end

    -- Helper: keybind button
    local waitingForKey = false
    local function KeybindBtn(parent, currentKey, onChange)
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0.4, 0, 0.7, 0)
        btn.Position = UDim2.new(0.58, 0, 0.15, 0)
        btn.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
        btn.Text = "[" .. currentKey.Name .. "]"
        btn.TextColor3 = Color3.fromRGB(0, 200, 255)
        btn.TextSize = 11
        btn.Font = Enum.Font.GothamBold
        btn.BorderSizePixel = 0
        btn.Parent = parent
        local bc = Instance.new("UICorner"); bc.CornerRadius = UDim.new(0,4); bc.Parent = btn

        btn.MouseButton1Click:Connect(function()
            if waitingForKey then return end
            waitingForKey = true
            btn.Text = "Press key..."
            btn.TextColor3 = Color3.fromRGB(255, 220, 0)
            local conn
            conn = UserInputService.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.Keyboard then
                    conn:Disconnect()
                    waitingForKey = false
                    onChange(input.KeyCode)
                    btn.Text = "[" .. input.KeyCode.Name .. "]"
                    btn.TextColor3 = Color3.fromRGB(0, 200, 255)
                end
            end)
        end)
        return btn
    end

    -- Helper: slider
    local function Slider(parent, minVal, maxVal, currentVal, onChange)
        local track = Instance.new("Frame")
        track.Size = UDim2.new(0.4, 0, 0, 6)
        track.Position = UDim2.new(0.57, 0, 0.5, -3)
        track.BackgroundColor3 = Color3.fromRGB(50, 50, 70)
        track.BorderSizePixel = 0
        track.Parent = parent
        local tc = Instance.new("UICorner"); tc.CornerRadius = UDim.new(0,3); tc.Parent = track

        local fill = Instance.new("Frame")
        fill.Size = UDim2.new((currentVal - minVal)/(maxVal - minVal), 0, 1, 0)
        fill.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
        fill.BorderSizePixel = 0
        fill.Parent = track
        local fc = Instance.new("UICorner"); fc.CornerRadius = UDim.new(0,3); fc.Parent = fill

        local valLabel = Instance.new("TextLabel")
        valLabel.Size = UDim2.new(0, 30, 1, 0)
        valLabel.Position = UDim2.new(1, 4, 0, 0)
        valLabel.BackgroundTransparency = 1
        valLabel.Text = tostring(math.floor(currentVal))
        valLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
        valLabel.TextSize = 11
        valLabel.Font = Enum.Font.Gotham
        valLabel.Parent = track

        local dragging = false
        track.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragging = true
            end
        end)
        track.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragging = false
            end
        end)
        UserInputService.InputChanged:Connect(function(input)
            if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
                local pos = input.Position.X
                local abs = track.AbsolutePosition
                local size = track.AbsoluteSize
                local pct = math.clamp((pos - abs.X) / size.X, 0, 1)
                local val = math.floor(minVal + pct * (maxVal - minVal))
                fill.Size = UDim2.new(pct, 0, 1, 0)
                valLabel.Text = tostring(val)
                onChange(val)
            end
        end)
        return track
    end

    -- Helper: dropdown for method
    local function MethodDropdown(parent, current, options, onChange)
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0.4, 0, 0.7, 0)
        btn.Position = UDim2.new(0.58, 0, 0.15, 0)
        btn.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
        btn.Text = current .. " ▾"
        btn.TextColor3 = Color3.fromRGB(0, 200, 255)
        btn.TextSize = 11
        btn.Font = Enum.Font.GothamBold
        btn.BorderSizePixel = 0
        btn.Parent = parent
        local bc = Instance.new("UICorner"); bc.CornerRadius = UDim.new(0,4); bc.Parent = btn

        local open = false
        local dropFrame = nil

        btn.MouseButton1Click:Connect(function()
            if open then
                if dropFrame then dropFrame:Destroy(); dropFrame = nil end
                open = false
                return
            end
            open = true
            dropFrame = Instance.new("Frame")
            dropFrame.Size = UDim2.new(0, btn.AbsoluteSize.X, 0, #options * 26)
            dropFrame.Position = UDim2.new(0,
                btn.AbsolutePosition.X - win.AbsolutePosition.X,
                0,
                btn.AbsolutePosition.Y - win.AbsolutePosition.Y + btn.AbsoluteSize.Y + 2)
            dropFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 45)
            dropFrame.BorderSizePixel = 0
            dropFrame.ZIndex = 10
            dropFrame.Parent = win
            local dc = Instance.new("UICorner"); dc.CornerRadius = UDim.new(0,5); dc.Parent = dropFrame

            for i, opt in ipairs(options) do
                local ob = Instance.new("TextButton")
                ob.Size = UDim2.new(1, 0, 0, 26)
                ob.Position = UDim2.new(0, 0, 0, (i-1)*26)
                ob.BackgroundTransparency = 1
                ob.Text = opt
                ob.TextColor3 = Color3.fromRGB(200, 200, 200)
                ob.TextSize = 11
                ob.Font = Enum.Font.Gotham
                ob.ZIndex = 11
                ob.Parent = dropFrame
                ob.MouseButton1Click:Connect(function()
                    onChange(opt)
                    btn.Text = opt .. " ▾"
                    if dropFrame then dropFrame:Destroy(); dropFrame = nil end
                    open = false
                end)
            end
        end)
        return btn
    end

    -- Helper: toggle
    local function Toggle(parent, state, onChange)
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0, 44, 0, 22)
        btn.Position = UDim2.new(1, -50, 0.5, -11)
        btn.BackgroundColor3 = state and Color3.fromRGB(0,150,255) or Color3.fromRGB(50,50,70)
        btn.Text = state and "ON" or "OFF"
        btn.TextColor3 = Color3.fromRGB(255,255,255)
        btn.TextSize = 10
        btn.Font = Enum.Font.GothamBold
        btn.BorderSizePixel = 0
        btn.Parent = parent
        local bc = Instance.new("UICorner"); bc.CornerRadius = UDim.new(0,11); bc.Parent = btn
        local on = state
        btn.MouseButton1Click:Connect(function()
            on = not on
            btn.BackgroundColor3 = on and Color3.fromRGB(0,150,255) or Color3.fromRGB(50,50,70)
            btn.Text = on and "ON" or "OFF"
            onChange(on)
        end)
        return btn
    end

    -- ---- BUILD ROWS ----

    SectionLabel("— KEYBINDS —", 1)

    local r1 = Row(2); RowLabel(r1, "Time Stop Key")
    KeybindBtn(r1, Config.ToggleKey, function(k) Config.ToggleKey = k end)

    local r2 = Row(3); RowLabel(r2, "Fake Lag Key")
    KeybindBtn(r2, Config.FakeLagKey, function(k) Config.FakeLagKey = k end)

    local r3 = Row(4); RowLabel(r3, "Sync Key")
    KeybindBtn(r3, Config.SyncKey, function(k) Config.SyncKey = k end)

    SectionLabel("— SPEED —", 5)

    local r4 = Row(6); RowLabel(r4, "Move Speed")
    Slider(r4, 5, 100, Config.MoveSpeed, function(v) Config.MoveSpeed = v end)

    local r5 = Row(7); RowLabel(r5, "Float Speed")
    Slider(r5, 5, 60, Config.FloatSpeed, function(v) Config.FloatSpeed = v end)

    SectionLabel("— FREEZE METHOD —", 8)

    local r6 = Row(9); RowLabel(r6, "Method")
    MethodDropdown(r6, Config.FreezeMethod,
        {"hook", "loop"},
        function(v)
            Config.FreezeMethod = v
            if v == "hook" then InstallHook() end
            Log("Freeze method changed to: " .. v)
        end)

    local r6b = Row(10, 44)
    local mDesc = Instance.new("TextLabel")
    mDesc.Size = UDim2.new(1, -16, 1, 0)
    mDesc.Position = UDim2.new(0, 8, 0, 0)
    mDesc.BackgroundTransparency = 1
    mDesc.Text = "hook = hookmetamethod (strongest)\nloop = RenderStepped force loop"
    mDesc.TextColor3 = Color3.fromRGB(120, 120, 140)
    mDesc.TextSize = 10
    mDesc.Font = Enum.Font.Gotham
    mDesc.TextXAlignment = Enum.TextXAlignment.Left
    mDesc.TextWrapped = true
    mDesc.LayoutOrder = 10
    mDesc.Parent = r6b

    SectionLabel("— OPTIONS —", 11)

    local r7 = Row(12); RowLabel(r7, "Show Effects")
    Toggle(r7, Config.ShowEffects, function(v) Config.ShowEffects = v end)

    local r8 = Row(13); RowLabel(r8, "Noclip")
    Toggle(r8, true, function(v)
        if not v and State.NoclipConn then
            StopNoclip()
        elseif v and State.Active then
            StartNoclip()
        end
    end)

    SectionLabel("— ACTIONS —", 14)

    local actRow = Row(15, 38)
    local function ActionBtn(text, xPct, onClick, col)
        local ab = Instance.new("TextButton")
        ab.Size = UDim2.new(0.3, 0, 0.7, 0)
        ab.Position = UDim2.new(xPct, 0, 0.15, 0)
        ab.BackgroundColor3 = col or Color3.fromRGB(0, 120, 220)
        ab.Text = text
        ab.TextColor3 = Color3.fromRGB(255,255,255)
        ab.TextSize = 11
        ab.Font = Enum.Font.GothamBold
        ab.BorderSizePixel = 0
        ab.Parent = actRow
        local ac = Instance.new("UICorner"); ac.CornerRadius = UDim.new(0,5); ac.Parent = ab
        ab.MouseButton1Click:Connect(onClick)
        return ab
    end

    ActionBtn("▶ START", 0.02, function() Activate("desync") end, Color3.fromRGB(0,150,100))
    ActionBtn("⏹ STOP",  0.35, function() Deactivate() end, Color3.fromRGB(150,80,0))
    ActionBtn("🔄 SYNC",  0.67, function() ForceSync() end, Color3.fromRGB(80,0,150))

    -- Status update loop
    task.spawn(function()
        while sg and sg.Parent do
            task.wait(0.2)
            if State.Active then
                local root = State.RootPart
                if root then
                    statusLabel.Text = string.format("● %s | %.0fs | %.0f %.0f %.0f",
                        State.Mode == "desync" and "TIME STOP" or "FAKE LAG",
                        GetElapsed(),
                        root.Position.X, root.Position.Y, root.Position.Z)
                    statusLabel.TextColor3 = State.Mode == "desync"
                        and Color3.fromRGB(0, 200, 255)
                        or  Color3.fromRGB(255, 140, 0)
                end
            else
                statusLabel.Text = "● IDLE"
                statusLabel.TextColor3 = Color3.fromRGB(120, 120, 120)
            end
        end
    end)

    -- Draggable window
    local dragging = false
    local dragStart, startPos
    titleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos  = win.Position
        end
    end)
    titleBar.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = input.Position - dragStart
            win.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + delta.X,
                startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)

    -- Close button
    local minimized = false
    local origSize = win.Size
    closeBtn.MouseButton1Click:Connect(function()
        minimized = not minimized
        if minimized then
            TweenService:Create(win, TweenInfo.new(0.2), {Size = UDim2.new(0, 320, 0, 36)}):Play()
            closeBtn.Text = "+"
        else
            TweenService:Create(win, TweenInfo.new(0.2), {Size = origSize}):Play()
            closeBtn.Text = "✕"
        end
    end)

    Log("✅ GUI opened — drag title bar to move")
end

-- ================================================
-- STARTUP
-- ================================================
Log("════════════════════════════════════════")
Log("   Liquid Bounce Blink v20.0 — JJS     ")
Log("════════════════════════════════════════")
InstallHook()
Log("[M] Time Stop | [N hold] Fake Lag | [B] Sync")
Log("GUI opened — customize keybinds, speed & method")
CreateGUI()
