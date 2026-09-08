--[[
    ROBLOX UI SCANNER V5.1 HYBRID
    UI Scanner + Real-time Remote Spy + UI-to-Lua Generator
    Executor-compatible client-side tool

    Features:
      - Scan PlayerGui + relevant CoreGui + Workspace BillboardGui/SurfaceGui
      - Real-time Remote Spy (hookmetamethod on FireServer & InvokeServer)
      - Debounced UI added/removed discovery (Watch Mode)
      - Batched traversal to prevent freezes & crashes
      - Search, Explorer tree & Property inspector
      - Copy selected JSON / Full Export for VS Code & AI Training
      - UI-to-Lua generator + Captured Remotes Code Generator

    Scope:
      Client-visible UI metadata + Outgoing network requests.
      Does NOT touch server code.
]]

local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
if not LocalPlayer then return end

--==================================================
-- Executor helpers
--==================================================

local function copyToClipboard(text)
    for _, name in ipairs({"setclipboard", "toclipboard", "set_clipboard", "writeclipboard"}) do
        local fn = getfenv and getfenv()[name] or _G[name]
        if type(fn) == "function" then
            if pcall(fn, text) then return true end
        end
    end
    return false
end

local function fsAvailable()
    return type(makefolder) == "function" and type(writefile) == "function"
end

local function ensureFolder(path)
    if type(isfolder) == "function" then
        local ok, exists = pcall(isfolder, path)
        if ok and exists then return true end
    end
    return pcall(makefolder, path)
end

local function writeTextFile(path, text)
    if type(writefile) ~= "function" then return false end
    return pcall(writefile, path, text)
end

--==================================================
-- Cleanup old UI
--==================================================

pcall(function()
    local old = game:GetService("CoreGui"):FindFirstChild("DeltaUIScanner")
    if old then old:Destroy() end
end)

--==================================================
-- Theme
--==================================================

local BG = Color3.fromRGB(14, 16, 20)
local PANEL = Color3.fromRGB(20, 23, 29)
local PANEL2 = Color3.fromRGB(25, 29, 36)
local BORDER = Color3.fromRGB(48, 55, 66)
local TEXT = Color3.fromRGB(235, 239, 245)
local MUTED = Color3.fromRGB(145, 154, 168)
local ACCENT = Color3.fromRGB(0, 220, 145)
local RED = Color3.fromRGB(255, 90, 100)
local YELLOW = Color3.fromRGB(255, 190, 70)

local function new(className, props, parent)
    local obj = Instance.new(className)
    for k, v in pairs(props or {}) do obj[k] = v end
    if parent then obj.Parent = parent end
    return obj
end

--==================================================
-- GUI
--==================================================

local gui = new("ScreenGui", {
    Name = "DeltaUIScanner",
    ResetOnSpawn = false,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling
})

local parentOK = pcall(function()
    gui.Parent = game:GetService("CoreGui")
end)
if not parentOK then
    gui.Parent = LocalPlayer:WaitForChild("PlayerGui")
end

local main = new("Frame", {
    Size = UDim2.fromOffset(900, 590),
    Position = UDim2.new(0.5, -450, 0.5, -295),
    BackgroundColor3 = BG,
    BorderSizePixel = 0
}, gui)
new("UICorner", {CornerRadius = UDim.new(0, 10)}, main)
new("UIStroke", {Color = BORDER, Thickness = 1}, main)

--==================================================
-- Header
--==================================================

local top = new("Frame", {
    Size = UDim2.new(1, 0, 0, 54),
    BackgroundColor3 = PANEL,
    BorderSizePixel = 0
}, main)
new("UICorner", {CornerRadius = UDim.new(0, 10)}, top)

new("TextLabel", {
    BackgroundTransparency = 1,
    Position = UDim2.fromOffset(18, 7),
    Size = UDim2.fromOffset(350, 24),
    Font = Enum.Font.GothamBold,
    Text = "ROBLOX UI SCANNER V5",
    TextColor3 = ACCENT,
    TextSize = 17,
    TextXAlignment = Enum.TextXAlignment.Left
}, top)

new("TextLabel", {
    BackgroundTransparency = 1,
    Position = UDim2.fromOffset(19, 29),
    Size = UDim2.fromOffset(430, 18),
    Font = Enum.Font.Gotham,
    Text = "Live UI discovery + Remote Spy Integrated",
    TextColor3 = MUTED,
    TextSize = 11,
    TextXAlignment = Enum.TextXAlignment.Left
}, top)

local scanButton = new("TextButton", {
    Position = UDim2.new(1, -356, 0, 10),
    Size = UDim2.fromOffset(105, 34),
    BackgroundColor3 = ACCENT,
    BorderSizePixel = 0,
    Font = Enum.Font.GothamBold,
    Text = "SCAN UI",
    TextColor3 = Color3.fromRGB(8, 12, 12),
    TextSize = 12
}, top)
new("UICorner", {CornerRadius = UDim.new(0, 7)}, scanButton)

local watchButton = new("TextButton", {
    Position = UDim2.new(1, -242, 0, 10),
    Size = UDim2.fromOffset(105, 34),
    BackgroundColor3 = PANEL2,
    BorderSizePixel = 0,
    Font = Enum.Font.GothamBold,
    Text = "WATCH: OFF",
    TextColor3 = TEXT,
    TextSize = 11
}, top)
new("UICorner", {CornerRadius = UDim.new(0, 7)}, watchButton)

local exportButton = new("TextButton", {
    Position = UDim2.new(1, -128, 0, 10),
    Size = UDim2.fromOffset(105, 34),
    BackgroundColor3 = PANEL2,
    BorderSizePixel = 0,
    Font = Enum.Font.GothamBold,
    Text = "EXPORT",
    TextColor3 = TEXT,
    TextSize = 11
}, top)
new("UICorner", {CornerRadius = UDim.new(0, 7)}, exportButton)

local scopeButton = new("TextButton", {
    Position = UDim2.new(1, -470, 0, 10),
    Size = UDim2.fromOffset(105, 34),
    BackgroundColor3 = PANEL2,
    BorderSizePixel = 0,
    Font = Enum.Font.GothamBold,
    Text = "SCOPE: ALL",
    TextColor3 = TEXT,
    TextSize = 10
}, top)
new("UICorner", {CornerRadius = UDim.new(0, 7)}, scopeButton)

--==================================================
-- Search
--==================================================

local search = new("TextBox", {
    Position = UDim2.fromOffset(14, 66),
    Size = UDim2.new(1, -28, 0, 38),
    BackgroundColor3 = PANEL,
    BorderSizePixel = 0,
    ClearTextOnFocus = false,
    Font = Enum.Font.Gotham,
    PlaceholderText = "Search name, class or path...",
    PlaceholderColor3 = MUTED,
    Text = "",
    TextColor3 = TEXT,
    TextSize = 12,
    TextXAlignment = Enum.TextXAlignment.Left
}, main)
new("UICorner", {CornerRadius = UDim.new(0, 7)}, search)
new("UIPadding", {PaddingLeft = UDim.new(0, 12), PaddingRight = UDim.new(0, 12)}, search)

--==================================================
-- Explorer panel
--==================================================

local explorer = new("Frame", {
    Position = UDim2.fromOffset(14, 114),
    Size = UDim2.new(0.54, -20, 1, -128),
    BackgroundColor3 = PANEL,
    BorderSizePixel = 0
}, main)
new("UICorner", {CornerRadius = UDim.new(0, 8)}, explorer)
new("UIStroke", {Color = BORDER, Thickness = 1}, explorer)

new("TextLabel", {
    Position = UDim2.fromOffset(12, 8),
    Size = UDim2.fromOffset(150, 22),
    BackgroundTransparency = 1,
    Font = Enum.Font.GothamBold,
    Text = "UI SOURCES",
    TextColor3 = TEXT,
    TextSize = 12,
    TextXAlignment = Enum.TextXAlignment.Left
}, explorer)

local countLabel = new("TextLabel", {
    Position = UDim2.new(1, -190, 0, 8),
    Size = UDim2.fromOffset(178, 22),
    BackgroundTransparency = 1,
    Font = Enum.Font.Gotham,
    Text = "0 objects",
    TextColor3 = MUTED,
    TextSize = 11,
    TextXAlignment = Enum.TextXAlignment.Right
}, explorer)

local treeScroll = new("ScrollingFrame", {
    Position = UDim2.fromOffset(7, 38),
    Size = UDim2.new(1, -14, 1, -88),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    ScrollBarThickness = 5,
    ScrollBarImageColor3 = BORDER,
    CanvasSize = UDim2.new(),
    AutomaticCanvasSize = Enum.AutomaticSize.Y
}, explorer)

local treeList = new("UIListLayout", {
    Padding = UDim.new(0, 2),
    SortOrder = Enum.SortOrder.LayoutOrder
}, treeScroll)

--==================================================
-- Properties panel
--==================================================

local properties = new("Frame", {
    Position = UDim2.new(0.54, 2, 0, 114),
    Size = UDim2.new(0.46, -16, 1, -128),
    BackgroundColor3 = PANEL,
    BorderSizePixel = 0
}, main)
new("UICorner", {CornerRadius = UDim.new(0, 8)}, properties)
new("UIStroke", {Color = BORDER, Thickness = 1}, properties)

new("TextLabel", {
    Position = UDim2.fromOffset(12, 9),
    Size = UDim2.new(1, -24, 0, 24),
    BackgroundTransparency = 1,
    Font = Enum.Font.GothamBold,
    Text = "PROPERTIES",
    TextColor3 = TEXT,
    TextSize = 12,
    TextXAlignment = Enum.TextXAlignment.Left
}, properties)

local selectedLabel = new("TextLabel", {
    Position = UDim2.fromOffset(12, 34),
    Size = UDim2.new(1, -24, 0, 38),
    BackgroundTransparency = 1,
    Font = Enum.Font.Gotham,
    Text = "Nothing selected",
    TextColor3 = MUTED,
    TextSize = 11,
    TextWrapped = true,
    TextXAlignment = Enum.TextXAlignment.Left,
    TextYAlignment = Enum.TextYAlignment.Top
}, properties)

local statusLabel = new("TextLabel", {
    Position = UDim2.fromOffset(12, 70),
    Size = UDim2.new(1, -24, 0, 22),
    BackgroundTransparency = 1,
    Font = Enum.Font.Gotham,
    Text = "Ready.",
    TextColor3 = MUTED,
    TextSize = 10,
    TextXAlignment = Enum.TextXAlignment.Left
}, properties)

local propScroll = new("ScrollingFrame", {
    Position = UDim2.fromOffset(8, 95),
    Size = UDim2.new(1, -16, 1, -143),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    ScrollBarThickness = 5,
    ScrollBarImageColor3 = BORDER,
    CanvasSize = UDim2.new(),
    AutomaticCanvasSize = Enum.AutomaticSize.Y
}, properties)

local propList = new("UIListLayout", {
    Padding = UDim.new(0, 4),
    SortOrder = Enum.SortOrder.LayoutOrder
}, propScroll)

local copySelected = new("TextButton", {
    Position = UDim2.new(0, 10, 1, -43),
    Size = UDim2.new(1, -20, 0, 34),
    BackgroundColor3 = PANEL2,
    BorderSizePixel = 0,
    Font = Enum.Font.GothamBold,
    Text = "COPY SELECTED JSON",
    TextColor3 = TEXT,
    TextSize = 11
}, properties)
new("UICorner", {CornerRadius = UDim.new(0, 7)}, copySelected)

-- Export copy buttons (works even when executor has no filesystem API)
local exportBar = new("Frame", {
    Position = UDim2.new(0, 8, 1, -86),
    Size = UDim2.new(1, -16, 0, 38),
    BackgroundTransparency = 1,
    BorderSizePixel = 0
}, explorer)

local copyUIButton = new("TextButton", {
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.new(0.25, -3, 1, 0),
    BackgroundColor3 = PANEL2,
    BorderSizePixel = 0,
    Font = Enum.Font.GothamBold,
    Text = "COPY UI JSON",
    TextColor3 = TEXT,
    TextSize = 9
}, exportBar)
new("UICorner", {CornerRadius = UDim.new(0, 6)}, copyUIButton)

local copyRemoteButton = new("TextButton", {
    Position = UDim2.new(0.25, 1, 0, 0),
    Size = UDim2.new(0.25, -3, 1, 0),
    BackgroundColor3 = PANEL2,
    BorderSizePixel = 0,
    Font = Enum.Font.GothamBold,
    Text = "COPY REMOTES",
    TextColor3 = TEXT,
    TextSize = 9
}, exportBar)
new("UICorner", {CornerRadius = UDim.new(0, 6)}, copyRemoteButton)

local copyLuaButton = new("TextButton", {
    Position = UDim2.new(0.50, 2, 0, 0),
    Size = UDim2.new(0.25, -3, 1, 0),
    BackgroundColor3 = PANEL2,
    BorderSizePixel = 0,
    Font = Enum.Font.GothamBold,
    Text = "COPY LUA",
    TextColor3 = TEXT,
    TextSize = 9
}, exportBar)
new("UICorner", {CornerRadius = UDim.new(0, 6)}, copyLuaButton)

local copyAIButton = new("TextButton", {
    Position = UDim2.new(0.75, 3, 0, 0),
    Size = UDim2.new(0.25, -3, 1, 0),
    BackgroundColor3 = ACCENT,
    BorderSizePixel = 0,
    Font = Enum.Font.GothamBold,
    Text = "COPY AI REPORT",
    TextColor3 = Color3.fromRGB(8, 12, 12),
    TextSize = 9
}, exportBar)
new("UICorner", {CornerRadius = UDim.new(0, 6)}, copyAIButton)

--==================================================
-- State
--==================================================

local recordsByInstance = {}
local allRecords = {}
local selectedRecord = nil
local watchEnabled = false
local connections = {}
local discoveryLog = {}
local scanNumber = 0
local scanScope = "all" -- all / playergui
local watchGeneration = 0
local watchQueued = false
local WATCH_DEBOUNCE = 0.25
local SCAN_YIELD_EVERY = 150
local TREE_YIELD_EVERY = 100
local EXPORT_YIELD_EVERY = 50
local MAX_SCAN_OBJECTS = 50000
local TREE_REBUILD_DEBOUNCE = 0.08

local scanBusy = false
local treeBuildGeneration = 0
local treeRebuildQueued = false

local discoveredByKey = {}
local discoveredOrder = {}

-- Remote Spy State
local remoteLogs = {}
local remoteCount = 0
local IGNORED_REMOTES = {
    ["CharacterSoundEvent"] = true,
    ["Ping"] = true,
    ["UpdatePosition"] = true,
    ["MovementUpdate"] = true
}

--==================================================
-- Serialization helpers
--==================================================

local function safeString(v)
    local ok, result = pcall(tostring, v)
    return ok and result or "<unavailable>"
end

local function vec2(v)
    return {
        x = math.round(v.X * 1000) / 1000,
        y = math.round(v.Y * 1000) / 1000
    }
end

local function udim2(v)
    return {
        xScale = math.round(v.X.Scale * 1000) / 1000,
        xOffset = v.X.Offset,
        yScale = math.round(v.Y.Scale * 1000) / 1000,
        yOffset = v.Y.Offset
    }
end

local function color(v)
    return {
        r = math.round(v.R * 255),
        g = math.round(v.G * 255),
        b = math.round(v.B * 255)
    }
end

local function serialize(obj, depth)
    local d = {
        name = obj.Name,
        className = obj.ClassName,
        fullName = safeString(obj:GetFullName()),
        depth = depth or 0
    }

    pcall(function() d.visible = obj.Visible end)
    pcall(function() d.enabled = obj.Enabled end)
    pcall(function() d.position = udim2(obj.Position) end)
    pcall(function() d.size = udim2(obj.Size) end)
    pcall(function() d.absolutePosition = vec2(obj.AbsolutePosition) end)
    pcall(function() d.absoluteSize = vec2(obj.AbsoluteSize) end)
    pcall(function() d.zIndex = obj.ZIndex end)
    pcall(function() d.text = obj.Text end)
    pcall(function() d.placeholderText = obj.PlaceholderText end)
    pcall(function() d.image = obj.Image end)
    pcall(function() d.backgroundColor3 = color(obj.BackgroundColor3) end)
    pcall(function() d.textColor3 = color(obj.TextColor3) end)
    pcall(function() d.backgroundTransparency = obj.BackgroundTransparency end)
    pcall(function() d.textTransparency = obj.TextTransparency end)
    pcall(function() d.layoutOrder = obj.LayoutOrder end)
    pcall(function() d.anchorPoint = vec2(obj.AnchorPoint) end)
    pcall(function() d.rotation = obj.Rotation end)
    pcall(function() d.active = obj.Active end)

    return d
end

local function serializeArgs(args)
    local parts = {}
    for _, arg in ipairs(args) do
        local t = typeof(arg)
        if t == "string" then
            table.insert(parts, string.format("%q", arg))
        elseif t == "number" or t == "boolean" then
            table.insert(parts, tostring(arg))
        elseif t == "Instance" then
            table.insert(parts, arg:GetFullName())
        elseif t == "Vector3" then
            table.insert(parts, string.format("Vector3.new(%.3f, %.3f, %.3f)", arg.X, arg.Y, arg.Z))
        elseif t == "CFrame" then
            table.insert(parts, string.format("CFrame.new(%s)", tostring(arg)))
        elseif t == "table" then
            local ok, encoded = pcall(HttpService.JSONEncode, HttpService, arg)
            table.insert(parts, ok and encoded or "{...}")
        else
            table.insert(parts, "nil --[[" .. t .. "]]")
        end
    end
    return table.concat(parts, ", ")
end

--==================================================
-- Remote Spy Engine (hookmetamethod)
--==================================================

local function startRemoteSpy()
    local hook = hookmetamethod or hookfunction
    if not hook then
        warn("[UI Scanner V5] Executor does not support hookmetamethod. Remote Spy disabled.")
        return
    end

    local oldNamecall
    oldNamecall = hookmetamethod(game, "__namecall", newcclosure(function(self, ...)
        local method = getnamecallmethod()
        local args = {...}

        if (method == "FireServer" or method == "fireServer") and self:IsA("RemoteEvent") then
            if not IGNORED_REMOTES[self.Name] then
                task.spawn(function()
                    remoteCount += 1
                    local fullPath = safeString(self:GetFullName())
                    local callString = fullPath .. ":FireServer(" .. serializeArgs(args) .. ")"
                    table.insert(remoteLogs, {
                        time = os.time(),
                        type = "RemoteEvent",
                        remote = fullPath,
                        args = args,
                        code = callString
                    })
                end)
            end
        elseif (method == "InvokeServer" or method == "invokeServer") and self:IsA("RemoteFunction") then
            if not IGNORED_REMOTES[self.Name] then
                task.spawn(function()
                    remoteCount += 1
                    local fullPath = safeString(self:GetFullName())
                    local callString = fullPath .. ":InvokeServer(" .. serializeArgs(args) .. ")"
                    table.insert(remoteLogs, {
                        time = os.time(),
                        type = "RemoteFunction",
                        remote = fullPath,
                        args = args,
                        code = callString
                    })
                end)
            end
        end

        return oldNamecall(self, ...)
    end))
end

pcall(startRemoteSpy)

--==================================================
-- Discovery & Persistence
--==================================================

local function discoveryKey(obj)
    return safeString(obj:GetFullName()) .. "|" .. obj.ClassName
end

local function rememberSnapshot(snapshot)
    for _, record in ipairs(snapshot) do
        local obj = record.instance
        if obj then
            local key = discoveryKey(obj)
            local data = serialize(obj, record.depth)
            pcall(function()
                if obj.Parent then
                    data.parentFullName = safeString(obj.Parent:GetFullName())
                    data.parentClassName = obj.Parent.ClassName
                end
            end)
            data.discoveryKey = key
            data.lastSeenAt = os.time()
            if not discoveredByKey[key] then
                table.insert(discoveredOrder, key)
            end
            discoveredByKey[key] = data
        end
    end
end

local function getDiscoveredList()
    local list = {}
    for _, key in ipairs(discoveredOrder) do
        local data = discoveredByKey[key]
        if data then table.insert(list, data) end
    end
    table.sort(list, function(a, b)
        if (a.depth or 0) ~= (b.depth or 0) then
            return (a.depth or 0) < (b.depth or 0)
        end
        return safeString(a.fullName) < safeString(b.fullName)
    end)
    return list
end

local function luaString(v)
    return string.format("%q", safeString(v))
end

local function luaNumber(v)
    return type(v) == "number" and string.format("%.6g", v) or "0"
end

local function luaValue(v)
    local t = type(v)
    if t == "string" then return luaString(v) end
    if t == "number" then return luaNumber(v) end
    if t == "boolean" then return tostring(v) end
    if t == "table" then
        if v.xScale ~= nil then
            return string.format("UDim2.new(%s, %s, %s, %s)", luaNumber(v.xScale), luaNumber(v.xOffset), luaNumber(v.yScale), luaNumber(v.yOffset))
        end
        if v.r ~= nil then
            return string.format("Color3.fromRGB(%s, %s, %s)", luaNumber(v.r), luaNumber(v.g), luaNumber(v.b))
        end
        if v.x ~= nil and v.y ~= nil then
            return string.format("Vector2.new(%s, %s)", luaNumber(v.x), luaNumber(v.y))
        end
    end
end

local GENERATOR_FIELDS = {
    "visible","enabled","position","size","zIndex","text","placeholderText",
    "image","backgroundColor3","textColor3","backgroundTransparency",
    "textTransparency","layoutOrder","anchorPoint","rotation","active"
}

local function generatedParentExpression(data, vars)
    local parent = data.parentFullName
    if parent and vars[parent] then return vars[parent] end
    if parent and string.find(parent, "PlayerGui", 1, true) then
        return "game:GetService(\"Players\").LocalPlayer:WaitForChild(\"PlayerGui\")"
    end
    if parent and string.find(parent, "CoreGui", 1, true) then
        return "game:GetService(\"CoreGui\")"
    end
    if parent and string.find(parent, "Workspace", 1, true) then
        return "workspace"
    end
    return "game:GetService(\"Players\").LocalPlayer:WaitForChild(\"PlayerGui\")"
end

local function generateLua()
    local list = getDiscoveredList()
    local lines = {
        "-- Generated by Roblox UI Scanner V5.1",
        "-- UI metadata only; game-specific logic/events/assets are not generated.",
        "local Players = game:GetService(\"Players\")",
        "local LocalPlayer = Players.LocalPlayer",
        "local vars = {}",
        ""
    }
    local vars = {}
    local used = {}
    for i, data in ipairs(list) do
        local base = safeString(data.name):gsub("[^%w_]", "_")
        if base == "" or base:match("^%d") then base = "UI_" .. base end
        base = base .. "_" .. i
        if used[base] then base = base .. "_x" end
        used[base] = true
        vars[data.fullName] = base
        table.insert(lines, "local " .. base .. " = Instance.new(" .. luaString(data.className) .. ")")
        table.insert(lines, base .. ".Name = " .. luaString(data.name))
        for _, field in ipairs(GENERATOR_FIELDS) do
            local encoded = luaValue(data[field])
            if encoded then
                table.insert(lines, "pcall(function() " .. base .. "." .. field .. " = " .. encoded .. " end)")
            end
        end
        table.insert(lines, base .. ".Parent = " .. generatedParentExpression(data, vars))
        table.insert(lines, "vars[" .. luaString(data.fullName) .. "] = " .. base)
        table.insert(lines, "")
    end
    table.insert(lines, "return vars")
    return table.concat(lines, "\n")
end

local function clearContainer(container)
    for _, child in ipairs(container:GetChildren()) do
        if not child:IsA("UIListLayout") then child:Destroy() end
    end
end

--==================================================
-- UI tree
--==================================================

local function showProperties(record)
    selectedRecord = record
    clearContainer(propScroll)

    if not record or not record.instance or not record.instance.Parent then
        selectedLabel.Text = "Nothing selected"
        return
    end

    local obj = record.instance
    selectedLabel.Text =
        obj.Name .. "\n" ..
        obj.ClassName .. "\n" ..
        safeString(obj:GetFullName())

    local data = serialize(obj, record.depth)

    local keys = {
        "name","className","fullName","depth","text","placeholderText",
        "image","visible","enabled","position","size","absolutePosition",
        "absoluteSize","zIndex","backgroundColor3","textColor3",
        "backgroundTransparency","textTransparency","layoutOrder",
        "anchorPoint","rotation","active"
    }

    for _, key in ipairs(keys) do
        local value = data[key]
        if value ~= nil then
            local textValue = safeString(value)

            if type(value) == "table" then
                local ok, encoded = pcall(HttpService.JSONEncode, HttpService, value)
                if ok then textValue = encoded end
            end

            local row = new("Frame", {
                Size = UDim2.new(1, -4, 0, 44),
                BackgroundColor3 = PANEL2,
                BorderSizePixel = 0
            }, propScroll)
            new("UICorner", {CornerRadius = UDim.new(0, 5)}, row)

            new("TextLabel", {
                Position = UDim2.fromOffset(8, 4),
                Size = UDim2.fromOffset(115, 17),
                BackgroundTransparency = 1,
                Font = Enum.Font.GothamBold,
                Text = key,
                TextColor3 = MUTED,
                TextSize = 10,
                TextXAlignment = Enum.TextXAlignment.Left
            }, row)

            new("TextLabel", {
                Position = UDim2.fromOffset(123, 3),
                Size = UDim2.new(1, -130, 1, -6),
                BackgroundTransparency = 1,
                Font = Enum.Font.Code,
                Text = textValue,
                TextColor3 = TEXT,
                TextSize = 10,
                TextWrapped = true,
                TextXAlignment = Enum.TextXAlignment.Left,
                TextYAlignment = Enum.TextYAlignment.Center
            }, row)
        end
    end
end

local function rebuildTree()
    treeBuildGeneration += 1
    local generation = treeBuildGeneration
    clearContainer(treeScroll)

    local query = string.lower(search.Text or "")
    local shown = 0

    for i, record in ipairs(allRecords) do
        if generation ~= treeBuildGeneration then return end
        local obj = record.instance

        if obj and obj.Parent then
            local haystack = string.lower(
                safeString(obj.Name) .. " " ..
                safeString(obj.ClassName) .. " " ..
                safeString(obj:GetFullName())
            )

            if query == "" or string.find(haystack, query, 1, true) then
                shown += 1

                local button = new("TextButton", {
                    Size = UDim2.new(1, -4, 0, 27),
                    BackgroundTransparency = 1,
                    BorderSizePixel = 0,
                    AutoButtonColor = false,
                    Text = "",
                    LayoutOrder = shown
                }, treeScroll)

                local indent = math.min(record.depth * 16, 240)

                new("TextLabel", {
                    Position = UDim2.fromOffset(5 + indent, 0),
                    Size = UDim2.new(1, -indent - 10, 1, 0),
                    BackgroundTransparency = 1,
                    Font = Enum.Font.Code,
                    Text = (record.hasChildren and "▸ " or "  ") ..
                        obj.Name .. " [" .. obj.ClassName .. "]",
                    TextColor3 = TEXT,
                    TextSize = 11,
                    TextXAlignment = Enum.TextXAlignment.Left
                }, button)

                button.MouseEnter:Connect(function()
                    button.BackgroundTransparency = 0
                    button.BackgroundColor3 = Color3.fromRGB(32,37,45)
                end)

                button.MouseLeave:Connect(function()
                    button.BackgroundTransparency = 1
                end)

                button.MouseButton1Click:Connect(function()
                    showProperties(record)
                end)
            end
        end

        if i % TREE_YIELD_EVERY == 0 then
            task.wait()
        end
    end

    if generation == treeBuildGeneration then
        countLabel.Text = tostring(#allRecords) .. " objects" ..
            (query ~= "" and (" | " .. tostring(shown) .. " shown") or "")
    end
end

local function queueTreeRebuild()
    if treeRebuildQueued then return end
    treeRebuildQueued = true
    task.delay(TREE_REBUILD_DEBOUNCE, function()
        treeRebuildQueued = false
        rebuildTree()
    end)
end

--==================================================
-- Discovery Process
--==================================================

local function isUIObject(obj)
    return obj:IsA("GuiObject")
        or obj:IsA("LayerCollector")
        or obj:IsA("UIBase")
end

local function collectUI()
    local result = {}
    local seen = {}
    local processed = 0
    local truncated = false

    local function shouldSkip(obj)
        return obj == gui or obj:IsDescendantOf(gui)
    end

    local function yieldBudget()
        processed += 1
        if processed >= MAX_SCAN_OBJECTS then
            truncated = true
            return true
        end
        if processed % SCAN_YIELD_EVERY == 0 then
            task.wait()
        end
        return false
    end

    local function walk(obj, depth)
        if seen[obj] or shouldSkip(obj) or truncated then return end
        seen[obj] = true

        local children = obj:GetChildren()
        table.insert(result, {
            instance = obj,
            depth = depth,
            hasChildren = #children > 0
        })

        if yieldBudget() then return end

        for _, child in ipairs(children) do
            if isUIObject(child) then
                walk(child, depth + 1)
            end
            if truncated then return end
        end
    end

    local function walkWorkspaceContainer(obj, depth)
        if seen[obj] or shouldSkip(obj) or truncated then return end
        seen[obj] = true

        for _, child in ipairs(obj:GetChildren()) do
            if child:IsA("BillboardGui") or child:IsA("SurfaceGui") then
                walk(child, depth)
            elseif child:IsA("Model") or child:IsA("Folder") or child:IsA("BasePart") or child:IsA("Attachment") then
                walkWorkspaceContainer(child, depth)
            end
            if yieldBudget() or truncated then return end
        end
    end

    local playerGui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
    if playerGui then
        for _, child in ipairs(playerGui:GetChildren()) do
            if isUIObject(child) then
                walk(child, 0)
            end
            if truncated then break end
        end
    end

    if scanScope == "all" and not truncated then
        pcall(function()
            local coreGui = game:GetService("CoreGui")
            for _, child in ipairs(coreGui:GetChildren()) do
                if isUIObject(child) and child ~= gui then
                    walk(child, 0)
                end
                if truncated then break end
            end
        end)

        if not truncated then
            pcall(function()
                walkWorkspaceContainer(workspace, 0)
            end)
        end
    end

    return result, truncated, processed
end

local function logEvent(kind, obj, extra)
    table.insert(discoveryLog, {
        time = os.time(),
        event = kind,
        object = obj and safeString(obj:GetFullName()) or "<unknown>",
        className = obj and obj.ClassName or "<unknown>",
        details = extra
    })
end

local function disconnectWatch()
    for _, c in ipairs(connections) do
        pcall(function() c:Disconnect() end)
    end
    table.clear(connections)
end

local function attachWatch()
    disconnectWatch()
    watchGeneration += 1
    local generation = watchGeneration

    local function refreshWatch()
        if not watchEnabled or generation ~= watchGeneration or watchQueued then return end
        watchQueued = true
        task.delay(WATCH_DEBOUNCE, function()
            watchQueued = false
            if not watchEnabled or generation ~= watchGeneration then return end

            local snapshot, truncated, processed = collectUI()
            rememberSnapshot(snapshot)
            allRecords = snapshot
            recordsByInstance = {}
            for _, r in ipairs(snapshot) do
                recordsByInstance[r.instance] = r
            end
            rebuildTree()
            statusLabel.Text = "Watching | " .. #allRecords .. " UI | " .. remoteCount .. " Remotes captured"
        end)
    end

    local function watchContainer(container)
        if not container then return end

        table.insert(connections, container.DescendantAdded:Connect(function(obj)
            if isUIObject(obj) and not obj:IsDescendantOf(gui) then
                logEvent("added", obj)
                refreshWatch()
            end
        end))

        table.insert(connections, container.DescendantRemoving:Connect(function(obj)
            if isUIObject(obj) and not obj:IsDescendantOf(gui) then
                logEvent("removed", obj)
                refreshWatch()
            end
        end))
    end

    local playerGui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
    if playerGui then watchContainer(playerGui) end

    if scanScope == "all" then
        pcall(function() watchContainer(game:GetService("CoreGui")) end)
        pcall(function() watchContainer(workspace) end)
    end

    statusLabel.Text = "Watching " .. (scanScope == "all" and "All UI sources" or "PlayerGui") .. "..."
end

local function scan()
    if scanBusy then return end
    scanBusy = true
    scanNumber += 1
    scanButton.Text = "SCANNING..."

    local snapshot, truncated, processed = collectUI()
    allRecords = snapshot
    rememberSnapshot(allRecords)
    recordsByInstance = {}

    for _, record in ipairs(allRecords) do
        recordsByInstance[record.instance] = record
    end

    logEvent("scan", LocalPlayer, {
        scanNumber = scanNumber,
        objectCount = #allRecords,
        processed = processed,
        truncated = truncated
    })

    rebuildTree()

    local hidden = 0
    for i, record in ipairs(allRecords) do
        local obj = record.instance
        pcall(function()
            if obj.Visible == false then hidden += 1 end
        end)
        if i % TREE_YIELD_EVERY == 0 then task.wait() end
    end

    statusLabel.Text =
        "Scan #" .. scanNumber ..
        " complete | " .. #allRecords ..
        " objects | " .. hidden .. " hidden | " ..
        "Remotes: " .. remoteCount

    scanButton.Text = "SCAN UI"
    scanBusy = false

    if watchEnabled then attachWatch() end
end

--==================================================
-- Export Functionality
--==================================================

local function buildAllJSON()
    local output = {
        scanner = "Roblox UI Scanner + Live Discovery",
        version = "5.1",
        exportedAt = os.time(),
        player = LocalPlayer.Name,
        objectCount = #allRecords,
        objects = {}
    }

    for _, record in ipairs(allRecords) do
        if record.instance and record.instance.Parent then
            table.insert(output.objects, serialize(record.instance, record.depth))
        end
    end

    return HttpService:JSONEncode(output)
end

local function buildRemotesText()
    local lines = {
        "-- Captured Remote Calls by Roblox UI Scanner V5.1",
        "-- Observation only: these are client-side outgoing calls captured by the spy.",
        "-- They are not guaranteed to be accepted by the server.",
        ""
    }

    local seen = {}
    local unique = 0
    for _, log in ipairs(remoteLogs) do
        local key = safeString(log.type) .. "|" .. safeString(log.remote) .. "|" .. safeString(log.code)
        if not seen[key] then
            seen[key] = true
            unique += 1
            table.insert(lines, string.format("-- [%s] %s -> %s", os.date("%X", log.time), log.type, log.remote))
            table.insert(lines, safeString(log.code))
            table.insert(lines, "")
        end
    end

    table.insert(lines, "-- Total captured: " .. tostring(#remoteLogs))
    table.insert(lines, "-- Unique call signatures: " .. tostring(unique))
    return table.concat(lines, "\n")
end

local function copyOrPrint(text, button, defaultText, label)
    if copyToClipboard(text) then
        button.Text = "COPIED!"
        statusLabel.Text = label .. " copied to clipboard."
        task.delay(1.2, function()
            if button then button.Text = defaultText end
        end)
        return true
    end
    print("[UI Scanner V5.1] " .. label .. "\n" .. text)
    statusLabel.Text = "Clipboard unavailable; " .. label .. " printed to console."
    return false
end

local function buildAIReport()
    local lines = {
        "=== ROBLOX UI SCANNER V5.1 - AI REPORT ===",
        "Scanner: UI Discovery + Remote Spy",
        "Player: " .. safeString(LocalPlayer.Name),
        "UI objects in current snapshot: " .. tostring(#allRecords),
        "Discovered UI history: " .. tostring(#discoveredOrder),
        "Captured remote calls: " .. tostring(#remoteLogs),
        "",
        "=== UI OBJECTS ===",
        ""
    }

    for i, record in ipairs(allRecords) do
        local obj = record.instance
        if obj and obj.Parent then
            local data = serialize(obj, record.depth)
            table.insert(lines, string.format("[%d] %s | %s", i, safeString(data.name), safeString(data.className)))
            table.insert(lines, "Path: " .. safeString(data.fullName))
            if data.text ~= nil and safeString(data.text) ~= "" then
                table.insert(lines, "Text: " .. safeString(data.text))
            end
            if data.placeholderText ~= nil and safeString(data.placeholderText) ~= "" then
                table.insert(lines, "Placeholder: " .. safeString(data.placeholderText))
            end
            if data.image ~= nil and safeString(data.image) ~= "" then
                table.insert(lines, "Image: " .. safeString(data.image))
            end
            if data.visible ~= nil then table.insert(lines, "Visible: " .. safeString(data.visible)) end
            if data.enabled ~= nil then table.insert(lines, "Enabled: " .. safeString(data.enabled)) end
            if data.parentFullName then table.insert(lines, "Parent: " .. safeString(data.parentFullName)) end
            table.insert(lines, "")
        end
        if i % 100 == 0 then task.wait() end
    end

    table.insert(lines, "=== REMOTE OBSERVATIONS ===")
    table.insert(lines, "")
    local seen = {}
    local unique = 0
    for _, log in ipairs(remoteLogs) do
        local key = safeString(log.type) .. "|" .. safeString(log.remote) .. "|" .. safeString(log.code)
        if not seen[key] then
            seen[key] = true
            unique += 1
            table.insert(lines, string.format("[%d] %s", unique, safeString(log.type)))
            table.insert(lines, "Remote: " .. safeString(log.remote))
            table.insert(lines, "Call: " .. safeString(log.code))
            table.insert(lines, "")
        end
    end

    table.insert(lines, "=== NOTES ===")
    table.insert(lines, "This report contains client-visible UI metadata and observed outgoing remote calls.")
    table.insert(lines, "Remote observations are not server source code and do not prove server-side validation rules.")
    return table.concat(lines, "\n")
end

local function exportFolder()
    if not fsAvailable() then
        statusLabel.Text = "Executor filesystem API unavailable."
        warn("[UI Scanner] Missing makefolder + writefile.")
        return
    end

    if #allRecords == 0 then
        statusLabel.Text = "Scan first."
        return
    end

    local root = "UI_Scanner_V5_Export"
    local data = root .. "/data"
    local objects = data .. "/objects"
    local docs = root .. "/docs"
    local generated = root .. "/generated"

    ensureFolder(root)
    ensureFolder(data)
    ensureFolder(objects)
    ensureFolder(docs)
    ensureFolder(generated)

    local manifest = {
        scanner = "Roblox UI Scanner + Remote Spy",
        version = "5.1",
        player = LocalPlayer.Name,
        exportedAt = os.time(),
        objectCount = #allRecords,
        discoveryEventCount = #discoveryLog,
        capturedRemotesCount = #remoteLogs
    }

    writeTextFile(root .. "/manifest.json", HttpService:JSONEncode(manifest))
    writeTextFile(data .. "/ui_scan.json", buildAllJSON())
    writeTextFile(data .. "/discovery_log.json", HttpService:JSONEncode(discoveryLog))
    writeTextFile(data .. "/discovered_ui.json", HttpService:JSONEncode({
        scanner = "Roblox UI Scanner V5.1",
        version = "5.1",
        exportedAt = os.time(),
        objectCount = #discoveredOrder,
        objects = getDiscoveredList()
    }))
    writeTextFile(generated .. "/generated_ui.lua", generateLua())

    -- Xuất Remotes bắt được thành text/Lua để phân tích.
    writeTextFile(data .. "/remotes_captured.lua", buildRemotesText())
    writeTextFile(data .. "/ai_report.txt", buildAIReport())

    local treeLines = {
        "ROBLOX UI DISCOVERY",
        "==============================",
        "Player: " .. LocalPlayer.Name,
        "Objects: " .. tostring(#allRecords),
        "Events: " .. tostring(#discoveryLog),
        "Remotes: " .. tostring(#remoteLogs),
        ""
    }

    for _, record in ipairs(allRecords) do
        if record.instance and record.instance.Parent then
            table.insert(treeLines,
                string.rep("  ", record.depth) ..
                record.instance.Name ..
                " [" .. record.instance.ClassName .. "]"
            )
        end
    end

    writeTextFile(docs .. "/tree.txt", table.concat(treeLines, "\n"))

    local readme = [[# Roblox UI Scanner V5 Export

Folder for AI Script Generation & VS Code Analysis.

## Files:
- `manifest.json`: Overview of the dump.
- `data/remotes_captured.lua`: EXACT server calls captured when you clicked buttons/attacked (CRUCIAL FOR AUTO SCRIPTS).
- `data/ui_scan.json`: Complete current UI snapshot.
- `data/discovery_log.json`: Added/removed/scan events.
- `docs/tree.txt`: Visual hierarchy for finding UI Paths.
- `generated/generated_ui.lua`: Re-constructs game UI in pure Lua.
]]

    writeTextFile(docs .. "/README.md", readme)

    local written = 0

    for i, record in ipairs(allRecords) do
        local obj = record.instance

        if obj and obj.Parent then
            local safeName = tostring(i) .. "_" ..
                safeString(obj.Name):gsub("[^%w%-%._]", "_")

            if #safeName > 100 then
                safeName = safeName:sub(1, 100)
            end

            local ok, json = pcall(function()
                return HttpService:JSONEncode(serialize(obj, record.depth))
            end)

            if ok and writeTextFile(objects .. "/" .. safeName .. ".json", json) then
                written += 1
            end
        end
        if i % EXPORT_YIELD_EVERY == 0 then
            statusLabel.Text = "Exporting files... " .. i .. "/" .. #allRecords
            task.wait()
        end
    end

    exportButton.Text = "EXPORTED!"
    statusLabel.Text = "Done! Exported UI + " .. #remoteLogs .. " Remotes."

    task.delay(1.5, function()
        if exportButton then exportButton.Text = "EXPORT" end
    end)
end

--==================================================
-- Buttons / Scope
--==================================================

scanButton.MouseButton1Click:Connect(scan)

scopeButton.MouseButton1Click:Connect(function()
    scanScope = (scanScope == "all") and "playergui" or "all"
    scopeButton.Text = scanScope == "all" and "SCOPE: ALL" or "SCOPE: GUI"
    scopeButton.TextColor3 = scanScope == "all" and ACCENT or TEXT
    if watchEnabled then
        attachWatch()
    end
    scan()
end)

watchButton.MouseButton1Click:Connect(function()
    watchEnabled = not watchEnabled

    if watchEnabled then
        watchButton.Text = "WATCH: ON"
        watchButton.TextColor3 = ACCENT
        attachWatch()
        statusLabel.Text = "Live watching enabled."
        logEvent("watch_started", LocalPlayer)
    else
        watchButton.Text = "WATCH: OFF"
        watchButton.TextColor3 = TEXT
        disconnectWatch()
        statusLabel.Text = "Live watching disabled."
        logEvent("watch_stopped", LocalPlayer)
    end
end)

exportButton.MouseButton1Click:Connect(exportFolder)

copySelected.MouseButton1Click:Connect(function()
    if not selectedRecord or not selectedRecord.instance then
        statusLabel.Text = "Select an object first."
        return
    end

    local ok, json = pcall(function()
        return HttpService:JSONEncode(
            serialize(selectedRecord.instance, selectedRecord.depth)
        )
    end)

    if ok and copyToClipboard(json) then
        copySelected.Text = "COPIED!"
        task.delay(1.2, function()
            if copySelected then copySelected.Text = "COPY SELECTED JSON" end
        end)
    elseif ok then
        print(json)
        statusLabel.Text = "Clipboard unavailable; JSON printed to console."
    end
end)

copyUIButton.MouseButton1Click:Connect(function()
    if #allRecords == 0 then
        statusLabel.Text = "Scan first."
        return
    end
    copyOrPrint(buildAllJSON(), copyUIButton, "COPY UI JSON", "UI JSON")
end)

copyRemoteButton.MouseButton1Click:Connect(function()
    copyOrPrint(buildRemotesText(), copyRemoteButton, "COPY REMOTES", "Remote report")
end)

copyLuaButton.MouseButton1Click:Connect(function()
    copyOrPrint(generateLua(), copyLuaButton, "COPY LUA", "Generated Lua")
end)

copyAIButton.MouseButton1Click:Connect(function()
    if #allRecords == 0 then
        statusLabel.Text = "Scan first."
        return
    end
    copyOrPrint(buildAIReport(), copyAIButton, "COPY AI REPORT", "AI report")
end)

search:GetPropertyChangedSignal("Text"):Connect(queueTreeRebuild)

--==================================================
-- Dragging Engine
--==================================================

local dragging = false
local dragStart
local startPos

top.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = true
        dragStart = input.Position
        startPos = main.Position

        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if not dragging then return end

    if input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch then

        local delta = input.Position - dragStart

        main.Position = UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset + delta.X,
            startPos.Y.Scale,
            startPos.Y.Offset + delta.Y
        )
    end
end)

--==================================================
-- Auto Start
--==================================================

task.defer(function()
    task.wait(0.5)
    scan()
end)

print("[UI Scanner V5 Hybrid] Loaded with Real-time Remote Spy.")
