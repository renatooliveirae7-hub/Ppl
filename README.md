--[[
    ══════════════════════════════════════════════════════════════
      EGG STEAL  |  Steal An Egg
      Roubo filtrado (>10B) + Server Browser (8+) + Antihit
    ══════════════════════════════════════════════════════════════
]]

if getgenv().EGG_STEAL then pcall(function() getgenv().EGG_STEAL:Destroy() end) end

local Players           = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TeleportService   = game:GetService("TeleportService")
local HttpService       = game:GetService("HttpService")
local VirtualUser       = game:GetService("VirtualUser")
local TweenService      = game:GetService("TweenService")
local UserInputService  = game:GetService("UserInputService")

local LP = Players.LocalPlayer

local CFG = {
    Enabled   = false,
    MinValue  = 10e9,
    AntiHit   = true,
    AutoJoin  = true,
    MinPool   = 8,      -- servidores mantidos na fila
    Delay     = 0.30,
    GrabCD    = 0.55,
    EmptyWait = 25,
}

local SUFFIX = { k = 1e3, m = 1e6, b = 1e9, t = 1e12, q = 1e15 }
local C_BG, C_PANEL, C_EL = Color3.fromRGB(14, 14, 17), Color3.fromRGB(20, 20, 25), Color3.fromRGB(30, 30, 38)
local C_ACCENT, C_TEXT, C_DIM = Color3.fromRGB(228, 58, 42), Color3.fromRGB(232, 232, 236), Color3.fromRGB(128, 128, 142)

local cache, grabTimes, conns = {}, {}, {}
local POOL, visited = {}, {}
local lastGood, running = os.clock(), true

-- ───────────────────── helpers ─────────────────────
local function parseValue(raw)
    if type(raw) == "number" then return raw end
    if type(raw) ~= "string" then return 0 end
    local s = raw:lower():gsub("%$", ""):gsub("%s", "")
    local n, suf = s:match("^([%d%.,]+)([kmbtq]?)$")
    if not n then n, suf = s:match("([%d%.,]+)([kmbtq]?)") end
    if not n or n == "" then return 0 end
    if n:find("%.") then n = n:gsub(",", "") else n = n:gsub(",", ".") end
    return (tonumber(n) or 0) * (SUFFIX[suf] or 1)
end

local function fmt(v)
    if v >= 1e12 then return ("%.1fT"):format(v / 1e12) end
    if v >= 1e9  then return ("%.1fB"):format(v / 1e9)  end
    if v >= 1e6  then return ("%.1fM"):format(v / 1e6)  end
    return tostring(math.floor(v))
end

local function char()
    local c = LP.Character
    local hrp = c and c:FindFirstChild("HumanoidRootPart")
    local hum = c and c:FindFirstChildOfClass("Humanoid")
    if hrp and hum and hum.Health > 0 then return c, hrp end
end

local function mk(class, props, parent)
    local i = Instance.new(class)
    for k, v in pairs(props) do i[k] = v end
    if parent then i.Parent = parent end
    return i
end

local function round(p, r) mk("UICorner", { CornerRadius = UDim.new(0, r or 7) }, p) end

-- ───────────────────── valor do ovo ─────────────────────
local function eggValue(egg)
    local c = cache[egg]
    if c and (os.clock() - c.t) < 3 then return c.v end
    local v = 0
    for _, a in ipairs({ "Value", "Worth", "Price", "Cost", "EggValue" }) do
        local ok, got = pcall(egg.GetAttribute, egg, a)
        if ok and got then v = parseValue(got); if v > 0 then break end end
    end
    if v == 0 then
        for _, d in ipairs(egg:GetDescendants()) do
            if d:IsA("NumberValue") or d:IsA("IntValue") or d:IsA("StringValue") then
                v = parseValue(d.Value); if v > 0 then break end
            end
        end
    end
    if v == 0 then
        for _, d in ipairs(egg:GetDescendants()) do
            if d:IsA("TextLabel") then v = parseValue(d.Text); if v > 0 then break end end
        end
    end
    if v == 0 then v = parseValue(egg.Name) end
    cache[egg] = { v = v, t = os.clock() }
    return v
end

-- ───────────────────── roubo ─────────────────────
local remotes
local function getRemotes()
    if remotes then return remotes end
    remotes = {}
    local ok, list = pcall(function() return ReplicatedStorage:GetDescendants() end)
    if ok then
        for _, o in ipairs(list) do
            if o:IsA("RemoteEvent") or o:IsA("RemoteFunction") then
                if o.Name:lower():match("steal|grab|pick|collect") then table.insert(remotes, o) end
            end
        end
    end
    return remotes
end

local function grab(egg, part)
    local _, hrp = char()
    if not hrp then return end
    local prompt = egg:FindFirstChildWhichIsA("ProximityPrompt", true)
    if prompt then pcall(fireproximityprompt, prompt) end
    if part then
        pcall(firetouchinterest, hrp, part, 0)
        task.wait(0.03)
        pcall(firetouchinterest, hrp, part, 1)
    end
    for _, r in ipairs(getRemotes()) do
        pcall(function()
            if r:IsA("RemoteFunction") then r:InvokeServer(egg.Name) else r:FireServer(egg.Name) end
        end)
    end
    grabTimes[egg] = os.clock()
end

local function scanBest()
    local best, bv = nil, 0
    for _, egg in ipairs(workspace:GetDescendants()) do
        if (egg:IsA("Model") or egg:IsA("BasePart")) and egg.Name:lower():find("egg") then
            local last = grabTimes[egg]
            if not last or (os.clock() - last) > CFG.GrabCD then
                local v = eggValue(egg)
                if v >= CFG.MinValue and v > bv then best, bv = egg, v end
            end
        end
    end
    return best, bv
end

-- ───────────────────── server browser ─────────────────────
local function fetchServers()
    local ok, res = pcall(function()
        local fn = request or http_request or syn.request
        return fn({
            Url = ("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=100"):format(game.PlaceId),
            Method = "GET",
        })
    end)
    if not ok or not res then return false end
    local ok2, data = pcall(function() return HttpService:JSONDecode(res.Body or res) end)
    if not ok2 or not data or not data.data then return false end

    local free, full = {}, {}
    for _, s in ipairs(data.data) do
        if s.id ~= game.JobId then
            if s.playing < s.maxPlayers then table.insert(free, s) else table.insert(full, s) end
        end
    end

    local out = {}
    for _, s in ipairs(free) do table.insert(out, s) end
    for _, s in ipairs(full) do
        if #out >= CFG.MinPool * 2 then break end
        table.insert(out, s)
    end

    POOL = out
    visited = {}
    return #POOL > 0
end

local function pickServer()
    local cands = {}
    for i, s in ipairs(POOL) do
        if not visited[s.id] and s.playing < s.maxPlayers then table.insert(cands, i) end
    end
    if #cands == 0 then
        for i, s in ipairs(POOL) do
            if s.playing < s.maxPlayers then table.insert(cands, i) end
        end
    end
    if #cands == 0 then return nil end
    return POOL[cands[math.random(#cands)]]
end

local function joinServer(server)
    if not server then return end
    visited[server.id] = true
    pcall(TeleportService.TeleportToPlaceInstance, TeleportService, game.PlaceId, server.id, LP)
end

local function autoJoin()
    if #POOL < CFG.MinPool then
        if not fetchServers() then return TeleportService:Teleport(game.PlaceId, LP) end
        renderList()
    end
    local s = pickServer()
    if s then joinServer(s) else TeleportService:Teleport(game.PlaceId, LP) end
end

-- ───────────────────── antihit ─────────────────────
local function installAntiHit()
    local old
    old = hookmetamethod(game, "__namecall", function(self, ...)
        if CFG.AntiHit and getnamecallmethod() == "Kick" and self == LP then
            notify("Antihit", "Kick bloqueado")
            return
        end
        return old(self, ...)
    end)

    table.insert(conns, LP.CharacterAdded:Connect(function(c)
        local hrp = c:WaitForChild("HumanoidRootPart")
        table.insert(conns, game:GetService("RunService").Heartbeat:Connect(function()
            if CFG.AntiHit and hrp.Parent and hrp.Position.Y < -120 then
                pcall(function()
                    hrp.CFrame = CFrame.new(0, 60, 0)
                    hrp.AssemblyLinearVelocity = Vector3.zero
                end)
            end
        end))
    end))

    table.insert(conns, LP.Idled:Connect(function()
        if CFG.AntiHit then
            VirtualUser:CaptureController()
            VirtualUser:ClickButton2(Vector2.new())
        end
    end))

    table.insert(conns, TeleportService.TeleportInitFailed:Connect(function()
        task.delay(4, autoJoin)
    end))
end

-- ───────────────────── UI ─────────────────────
local gui = mk("ScreenGui", { Name = "EggSteal", ResetOnSpawn = false })
gui.Parent = (gethui and gethui()) or LP:WaitForChild("PlayerGui")

local win = mk("Frame", {
    Size = UDim2.fromOffset(300, 470), Position = UDim2.fromOffset(60, 80),
    BackgroundColor3 = C_BG, BorderSizePixel = 0, Active = true, Parent = gui,
})
round(win, 10)
mk("UIStroke", { Color = Color3.fromRGB(45, 45, 55), Thickness = 1 }, win)

local top = mk("Frame", { Size = UDim2.new(1, 0, 0, 32), BackgroundColor3 = C_PANEL, BorderSizePixel = 0, Parent = win })
round(top, 10)
mk("Frame", { Size = UDim2.new(1, 0, 0, 10), Position = UDim2.new(0, 0, 1, -10), BackgroundColor3 = C_PANEL, BorderSizePixel = 0, Parent = top })
mk("Frame", { Size = UDim2.fromOffset(4, 13), Position = UDim2.fromOffset(12, 10), BackgroundColor3 = C_ACCENT, BorderSizePixel = 0, Parent = top })
mk("TextLabel", {
    Size = UDim2.new(1, -30, 1, 0), Position = UDim2.fromOffset(24, 0), BackgroundTransparency = 1,
    Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = C_TEXT,
    TextXAlignment = Enum.TextXAlignment.Left, Text = "EGG STEAL  |  >10B", Parent = top,
})

do
    local drag, start, origin
    top.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then
            drag, start, origin = true, i.Position, win.Position
        end
    end)
    UserInputService.InputChanged:Connect(function(i)
        if drag and i.UserInputType == Enum.UserInputType.MouseMovement then
            win.Position = UDim2.fromOffset(origin.X.Offset + (i.Position.X - start.X), origin.Y.Offset + (i.Position.Y - start.Y))
        end
    end)
    UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then drag = false end
    end)
end

local function notify(t, txt)
    local n = mk("Frame", { Size = UDim2.fromOffset(200, 42), Position = UDim2.new(1, -210, 0, 0), BackgroundColor3 = C_PANEL, BorderSizePixel = 0, Parent = gui })
    round(n, 6)
    mk("Frame", { Size = UDim2.fromOffset(3, 42), BackgroundColor3 = C_ACCENT, BorderSizePixel = 0, Parent = n })
    mk("TextLabel", { Size = UDim2.new(1, -14, 0, 17), Position = UDim2.fromOffset(10, 4), BackgroundTransparency = 1, Font = Enum.Font.GothamBold, TextSize = 11, TextColor3 = C_ACCENT, TextXAlignment = Enum.TextXAlignment.Left, Text = t, Parent = n })
    mk("TextLabel", { Size = UDim2.new(1, -14, 0, 17), Position = UDim2.fromOffset(10, 20), BackgroundTransparency = 1, Font = Enum.Font.Gotham, TextSize = 10, TextColor3 = C_DIM, TextXAlignment = Enum.TextXAlignment.Left, Text = txt, Parent = n })
    task.delay(3, function() n:Destroy() end)
end

local function row(name, y, parent)
    local f = mk("Frame", { Size = UDim2.new(1, -20, 0, 26), Position = UDim2.fromOffset(10, y), BackgroundColor3 = C_EL, BorderSizePixel = 0, Parent = parent })
    round(f, 6)
    mk("TextLabel", {
        Size = UDim2.new(1, -50, 1, 0), Position = UDim2.fromOffset(8, 0), BackgroundTransparency = 1,
        Font = Enum.Font.Gotham, TextSize = 11, TextColor3 = C_TEXT,
        TextXAlignment = Enum.TextXAlignment.Left, Text = name, Parent = f,
    })
    return f
end

local function toggle(name, y, default, cb)
    local f = row(name, y, win)
    local on = default
    local dot = mk("Frame", { Size = UDim2.fromOffset(30, 16), Position = UDim2.new(1, -38, 0.5, -8), BackgroundColor3 = on and C_ACCENT or Color3.fromRGB(55, 55, 65), BorderSizePixel = 0, Parent = f })
    round(dot, 8)
    local knob = mk("Frame", { Size = UDim2.fromOffset(12, 12), Position = UDim2.fromOffset(on and 16 or 2, 2), BackgroundColor3 = C_TEXT, BorderSizePixel = 0, Parent = dot })
    round(knob, 6)
    local b = mk("TextButton", { Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "", Parent = f })
    b.MouseButton1Click:Connect(function()
        on = not on
        TweenService:Create(knob, TweenInfo.new(0.12), { Position = UDim2.fromOffset(on and 16 or 2, 2) }):Play()
        dot.BackgroundColor3 = on and C_ACCENT or Color3.fromRGB(55, 55, 65)
        cb(on)
    end)
end

local function textbox(name, y, default, cb)
    local f = row(name, y, win)
    local box = mk("TextBox", {
        Size = UDim2.fromOffset(80, 18), Position = UDim2.new(1, -88, 0.5, -9), BackgroundColor3 = C_PANEL,
        BorderSizePixel = 0, Font = Enum.Font.Gotham, TextSize = 11, TextColor3 = C_ACCENT,
        Text = default, TextXAlignment = Enum.TextXAlignment.Center, Parent = f,
    })
    round(box, 4)
    box.FocusLost:Connect(function()
        CFG.MinValue = math.max(parseValue(box.Text), 1)
        box.Text = fmt(CFG.MinValue)
        notify("Filtro", "Minimo: " .. fmt(CFG.MinValue))
    end)
end

toggle("Auto Steal", 40, false, function(v) CFG.Enabled = v; notify("Auto Steal", v and "ON" or "OFF") end)
textbox("Valor minimo", 72, "10B", function() end)
toggle("Antihit", 104, true, function(v) CFG.AntiHit = v end)
toggle("Join automatico", 136, true, function(v) CFG.AutoJoin = v end)

-- header da lista
local listHead = mk("Frame", { Size = UDim2.new(1, -20, 0, 24), Position = UDim2.fromOffset(10, 172), BackgroundColor3 = C_PANEL, BorderSizePixel = 0, Parent = win })
round(listHead, 6)
local headLbl = mk("TextLabel", {
    Size = UDim2.new(1, -80, 1, 0), Position = UDim2.fromOffset(8, 0), BackgroundTransparency = 1,
    Font = Enum.Font.GothamBold, TextSize = 11, TextColor3 = C_TEXT,
    TextXAlignment = Enum.TextXAlignment.Left, Text = "SERVIDORES", Parent = listHead,
})
local refreshBtn = mk("TextButton", {
    Size = UDim2.fromOffset(60, 18), Position = UDim2.new(1, -66, 0.5, -9), BackgroundColor3 = C_EL,
    BorderSizePixel = 0, Font = Enum.Font.GothamBold, TextSize = 10, TextColor3 = C_ACCENT, Text = "BUSCAR", Parent = listHead,
})
round(refreshBtn, 4)

local list = mk("ScrollingFrame", {
    Size = UDim2.new(1, -20, 0, 210), Position = UDim2.fromOffset(10, 200),
    BackgroundColor3 = C_PANEL, BorderSizePixel = 0, ScrollBarThickness = 3,
    ScrollBarImageColor3 = C_ACCENT, CanvasSize = UDim2.new(), Parent = win,
})
round(list, 6)

local status = mk("TextLabel", {
    Size = UDim2.new(1, -20, 0, 16), Position = UDim2.fromOffset(10, 418),
    BackgroundTransparency = 1, Font = Enum.Font.Gotham, TextSize = 10, TextColor3 = C_DIM,
    TextXAlignment = Enum.TextXAlignment.Left, Text = "Parado", Parent = win,
})
local autoBtn = mk("TextButton", {
    Size = UDim2.new(1, -20, 0, 26), Position = UDim2.fromOffset(10, 438),
    BackgroundColor3 = C_ACCENT, BorderSizePixel = 0, Font = Enum.Font.GothamBold,
    TextSize = 11, TextColor3 = Color3.fromRGB(255, 255, 255), Text = "ENTRAR EM SERVIDOR ALEATORIO", Parent = win,
})
round(autoBtn, 6)

function renderList()
    for _, c in ipairs(list:GetChildren()) do
        if c:IsA("Frame") then c:Destroy() end
    end
    headLbl.Text = ("SERVIDORES  (%d)"):format(#POOL)

    local y = 4
    for i, s in ipairs(POOL) do
        local r = mk("Frame", { Size = UDim2.new(1, -8, 0, 22), Position = UDim2.fromOffset(4, y), BackgroundColor3 = C_EL, BorderSizePixel = 0, Parent = list })
        round(r, 4)
        local lotado = s.playing >= s.maxPlayers
        mk("TextLabel", {
            Size = UDim2.new(1, -70, 1, 0), Position = UDim2.fromOffset(8, 0), BackgroundTransparency = 1,
            Font = Enum.Font.Gotham, TextSize = 10, TextColor3 = lotado and C_DIM or C_TEXT,
            TextXAlignment = Enum.TextXAlignment.Left,
            Text = ("#%d  ·  %d/%d  ·  %s"):format(i, s.playing, s.maxPlayers, s.ping and (tostring(s.ping) .. "ms") or "--"),
            Parent = r,
        })
        local jb = mk("TextButton", {
            Size = UDim2.fromOffset(56, 16), Position = UDim2.new(1, -62, 0.5, -8),
            BackgroundColor3 = lotado and C_PANEL or C_ACCENT, BorderSizePixel = 0,
            Font = Enum.Font.GothamBold, TextSize = 9, TextColor3 = Color3.fromRGB(255, 255# Ppl
