--[[
    ══════════════════════════════════════════════════════════════
      EGG STEAL  |  Steal An Egg
      Menu com botao de acesso + roubo verificado + browser + antihit
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
local RunService        = game:GetService("RunService")

local LP = Players.LocalPlayer

local CFG = {
    Enabled     = false,
    MinValue    = 10e9,
    AntiHit     = true,
    AutoJoin    = true,
    AutoDeposit = true,
    MinPool     = 8,
    Delay       = 0.25,
    GrabCD      = 0.45,
    EmptyWait   = 25,
    Retries     = 3,
}

local SUFFIX = { k = 1e3, m = 1e6, b = 1e9, t = 1e12, q = 1e15 }
local C_BG, C_PANEL, C_EL = Color3.fromRGB(14, 14, 17), Color3.fromRGB(20, 20, 25), Color3.fromRGB(30, 30, 38)
local C_ACCENT, C_TEXT, C_DIM = Color3.fromRGB(228, 58, 42), Color3.fromRGB(232, 232, 236), Color3.fromRGB(128, 128, 142)

local cache, grabTimes, conns, failed = {}, {}, {}, {}
local POOL, visited = {}, {}
local lastGood, running = os.clock(), true
local grabStyle = {}

-- ───────────────────── helpers ─────────────────────
local gui

local function mk(class, props, parent)
    local i = Instance.new(class)
    for k, v in pairs(props) do i[k] = v end
    if parent then i.Parent = parent end
    return i
end

local function round(p, r) mk("UICorner", { CornerRadius = UDim.new(0, r or 7) }, p) end

local function notify(t, txt)
    if not gui then return end
    local n = mk("Frame", { Size = UDim2.fromOffset(200, 42), Position = UDim2.new(1, -210, 0, 0), BackgroundColor3 = C_PANEL, BorderSizePixel = 0, Parent = gui })
    round(n, 6)
    mk("Frame", { Size = UDim2.fromOffset(3, 42), BackgroundColor3 = C_ACCENT, BorderSizePixel = 0, Parent = n })
    mk("TextLabel", { Size = UDim2.new(1, -14, 0, 17), Position = UDim2.fromOffset(10, 4), BackgroundTransparency = 1, Font = Enum.Font.GothamBold, TextSize = 11, TextColor3 = C_ACCENT, TextXAlignment = Enum.TextXAlignment.Left, Text = t, Parent = n })
    mk("TextLabel", { Size = UDim2.new(1, -14, 0, 17), Position = UDim2.fromOffset(10, 20), BackgroundTransparency = 1, Font = Enum.Font.Gotham, TextSize = 10, TextColor3 = C_DIM, TextXAlignment = Enum.TextXAlignment.Left, Text = txt, Parent = n })
    task.delay(3, function() n:Destroy() end)
end

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

-- ───────────────────── valor ─────────────────────
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

-- ───────────────────── base e carga ─────────────────────
local function isMine(inst)
    local o = inst:GetAttribute("Owner") or inst:GetAttribute("Player") or inst:GetAttribute("UserId")
    if o == LP.Name or o == LP.UserId or o == tostring(LP.UserId) then return true end
    for _, d in ipairs(inst:GetDescendants()) do
        if d:IsA("TextLabel") and d.Text:find(LP.DisplayName, 1, true) then return true end
    end
