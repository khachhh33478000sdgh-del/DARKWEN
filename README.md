-- Darkwen | Fling Things and People | Delta Mobile | Custom UI Đen/Trắng
-- Mật khẩu: ff9012

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local VirtualUser = game:GetService("VirtualUser")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")
local Workspace = game:GetService("Workspace")

local LP = Players.LocalPlayer
local Camera = Workspace.CurrentCamera
local PASSWORD = "ff9012"

local httpRequest = (syn and syn.request) or (http and http.request) or http_request or request
local hasHook = (hookmetamethod and getnamecallmethod and newcclosure and checkcaller) and true or false

local BLACK = Color3.fromRGB(0, 0, 0)
local PANEL = Color3.fromRGB(12, 12, 12)
local ITEM = Color3.fromRGB(24, 24, 24)
local OFF = Color3.fromRGB(55, 55, 55)
local WHITE = Color3.fromRGB(255, 255, 255)
local GREY = Color3.fromRGB(150, 150, 150)

local State = {
	ESP = false, Name = true, Distance = true, Health = true, Tracer = true, Box = true, Chams = true,
	Speed = false, SpeedVal = 50,
	Jump = false, JumpVal = 120,
	Fly = false, FlySpeed = 80,
	Noclip = false, InfJump = false,
	Fling = false, Kill = false, FlingRadius = 15,
	Hitbox = false, HitboxSize = 10,
	Pickup = false, PickupRadius = 20, ItemESP = false,
	AntiAFK = false, RemoteSpy = false,
}

local Conns, ESPObjects, Hitboxes, ItemTags, SpyLog = {}, {}, {}, {}, {}
local Toggles, Sliders, Handlers, Pages, TabBtns = {}, {}, {}, {}, {}
local SpyDirty, Unloaded = false, false
local Gui, Main, FloatBar, MenuBtn, LockBtn, PlayerLabel, SpyLabel
local Locked = false
local jobTarget = ""
local flyWasOn = false
local flingOrigin, flingChar
local TABS = {"Player", "Combat", "Items", "Server", "Misc"}

local function track(c) Conns[#Conns + 1] = c return c end
local function getChar(p) p = p or LP return p and p.Character end
local function getRoot(c) return c and c:FindFirstChild("HumanoidRootPart") end
local function getHum(c) return c and c:FindFirstChildOfClass("Humanoid") end

local function new(class, props, kids)
	local o = Instance.new(class)
	for k, v in pairs(props or {}) do o[k] = v end
	for _, c in ipairs(kids or {}) do c.Parent = o end
	return o
end
local function corner(r) return new("UICorner", {CornerRadius = UDim.new(0, r or 8)}) end
local function stroke(t) return new("UIStroke", {Color = WHITE, Thickness = 1, Transparency = t or 0.7}) end
local function tween(o, props, time)
	TweenService:Create(o, TweenInfo.new(time or 0.15, Enum.EasingStyle.Quad), props):Play()
end

local function notify(text)
	if not Gui then return end
	local t = new("TextLabel", {
		Size = UDim2.fromOffset(280, 30), Position = UDim2.new(0.5, -140, 0, 12),
		BackgroundColor3 = BLACK, BackgroundTransparency = 0.05, TextColor3 = WHITE,
		Font = Enum.Font.GothamMedium, TextSize = 13, Text = text, ZIndex = 50, Parent = Gui,
	}, {corner(8), stroke(0.4)})
	task.delay(2.2, function()
		if t.Parent then
			tween(t, {BackgroundTransparency = 1, TextTransparency = 1}, 0.4)
			task.wait(0.45)
			t:Destroy()
		end
	end)
end

local function setCham(char, on)
	if not char then return end
	local h = char:FindFirstChild("DW_Cham")
	if on and not h then
		h = Instance.new("Highlight")
		h.Name = "DW_Cham"
		h.FillColor = Color3.fromRGB(255, 255, 255)
		h.FillTransparency = 0.6
		h.OutlineColor = Color3.fromRGB(255, 255, 255)
		h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
		h.Parent = char
	elseif not on and h then
		h:Destroy()
	end
end

local function isCharacterPart(inst)
	local cur = inst
	while cur and cur ~= Workspace do
		if cur:IsA("Model") and cur:FindFirstChildOfClass("Humanoid") then return true end
		cur = cur.Parent
	end
	return false
end

local function promptPart(pp)
	local part = pp.Parent
	if part and part:IsA("Attachment") then part = part.Parent end
	if part and part:IsA("BasePart") then return part end
	return nil
end

---------------------------------------------------------------------
-- ESP
---------------------------------------------------------------------
local function newDrawing(class, props)
	local d = Drawing.new(class)
	for k, v in pairs(props) do d[k] = v end
	return d
end

local function makeESP(p)
	if p == LP or ESPObjects[p] then return end
	ESPObjects[p] = {
		Name = newDrawing("Text", {Size = 14, Center = true, Outline = true, Color = WHITE, Visible = false}),
		Tracer = newDrawing("Line", {Thickness = 1, Color = WHITE, Visible = false}),
		Box = newDrawing("Square", {Thickness = 1, Color = WHITE, Filled = false, Visible = false}),
		HPBack = newDrawing("Square", {Filled = true, Color = BLACK, Visible = false}),
		HPFill = newDrawing("Square", {Filled = true, Color = WHITE, Visible = false}),
	}
end

local function removeESP(p)
	local d = ESPObjects[p]
	if not d then return end
	for _, o in pairs(d) do o:Remove() end
	ESPObjects[p] = nil
end

local function hideESP(d)
	for _, o in pairs(d) do o.Visible = false end
end

for _, p in ipairs(Players:GetPlayers()) do makeESP(p) end
track(Players.PlayerAdded:Connect(makeESP))
track(Players.PlayerRemoving:Connect(removeESP))

track(RunService.RenderStepped:Connect(function()
	if Unloaded then return end
	local vs = Camera.ViewportSize
	local myRoot = getRoot(getChar())
	local textSize = math.clamp(vs.X / 80, 11, 18)

	for p, d in pairs(ESPObjects) do
		local char = getChar(p)
		local root, hum = getRoot(char), getHum(char)

		if not State.ESP or not root or not hum or hum.Health <= 0 then
			hideESP(d)
			setCham(char, false)
		else
			setCham(char, State.Chams)
			local pos = Camera:WorldToViewportPoint(root.Position)
			local top = Camera:WorldToViewportPoint(root.Position + Vector3.new(0, 3, 0))
			local bot = Camera:WorldToViewportPoint(root.Position - Vector3.new(0, 3.5, 0))

			if pos.Z <= 0 then
				hideESP(d)
			else
				local h = math.abs(bot.Y - top.Y)
				local w = h * 0.55
				local x = pos.X - w / 2
				local y = top.Y
				local dist = myRoot and (myRoot.Position - root.Position).Magnitude or 0

				d.Box.Visible = State.Box
				d.Box.Position = Vector2.new(x, y)
				d.Box.Size = Vector2.new(w, h)

				d.Tracer.Visible = State.Tracer
				d.Tracer.From = Vector2.new(vs.X / 2, vs.Y)
				d.Tracer.To = Vector2.new(pos.X, pos.Y)

				local parts = {}
				if State.Name then parts[#parts + 1] = p.DisplayName end
				if State.Distance then parts[#parts + 1] = string.format("[%dm]", math.floor(dist)) end
				if State.Health then parts[#parts + 1] = string.format("HP %d", math.floor(hum.Health)) end
				d.Name.Visible = #parts > 0
				d.Name.Text = table.concat(parts, " ")
				d.Name.Size = textSize
				d.Name.Position = Vector2.new(pos.X, y - textSize - 2)

				local ratio = math.clamp(hum.Health / math.max(hum.MaxHealth, 1), 0, 1)
				d.HPBack.Visible = State.Health
				d.HPBack.Position = Vector2.new(x - 5, y)
				d.HPBack.Size = Vector2.new(3, h)
				d.HPFill.Visible = State.Health
				d.HPFill.Position = Vector2.new(x - 5, y + h * (1 - ratio))
				d.HPFill.Size = Vector2.new(3, h * ratio)
			end
		end
	end
end))

---------------------------------------------------------------------
-- Movement
---------------------------------------------------------------------
track(RunService.Heartbeat:Connect(function()
	if Unloaded then return end
	local char = getChar()
	local hum = getHum(char)
	local root = getRoot(char)

	if hum then
		if State.Speed then hum.WalkSpeed = State.SpeedVal end
		if State.Jump then
			hum.UseJumpPower = true
			hum.JumpPower = State.JumpVal
		end
	end

	if State.Fly and hum and root then
		flyWasOn = true
		hum.PlatformStand = true
		local md = hum.MoveDirection
		local look = Camera.CFrame.LookVector
		local vel = Vector3.zero
		if md.Magnitude > 0 then
			vel = Vector3.new(md.X, look.Y * md.Magnitude, md.Z) * State.FlySpeed
		end
		if hum.Jump then
			vel = vel + Vector3.new(0, State.FlySpeed, 0)
		end
		root.AssemblyLinearVelocity = vel
	elseif flyWasOn then
		flyWasOn = false
		if hum then hum.PlatformStand = false end
	end
end))

track(RunService.Stepped:Connect(function()
	if Unloaded or not State.Noclip then return end
	local char = getChar()
	if not char then return end
	for _, part in ipairs(char:GetDescendants()) do
		if part:IsA("BasePart") and part.CanCollide then
			part.CanCollide = false
		end
	end
end))

track(UIS.JumpRequest:Connect(function()
	if Unloaded or not State.InfJump then return end
	local hum = getHum(getChar())
	if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
end))

---------------------------------------------------------------------
-- Combat: Fling / Kill aura + Hitbox
---------------------------------------------------------------------
track(RunService.Heartbeat:Connect(function()
	if Unloaded then return end
	local char = getChar()
	local root = getRoot(char)
	local hum = getHum(char)
	local active = (State.Fling or State.Kill) and root and hum and hum.Health > 0

	local target, best = nil, State.FlingRadius
	if active then
		for _, p in ipairs(Players:GetPlayers()) do
			if p ~= LP then
				local tr = getRoot(getChar(p))
				if tr then
					local d = (tr.Position - root.Position).Magnitude
					if d < best then
						best = d
						target = tr
					end
				end
			end
		end
	end

	if target then
		if not flingOrigin or flingChar ~= char then
			flingOrigin = root.CFrame
			flingChar = char
		end
		root.CFrame = target.CFrame
		root.AssemblyAngularVelocity = Vector3.new(0, 5000, 0)
		root.AssemblyLinearVelocity = State.Kill and Vector3.new(0, -300, 0) or Vector3.zero
	elseif flingOrigin and root and flingChar == char then
		root.AssemblyAngularVelocity = Vector3.zero
		root.AssemblyLinearVelocity = Vector3.zero
		root.CFrame = flingOrigin
		flingOrigin = nil
	end

	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= LP then
			local r = getRoot(getChar(p))
			if r then
				if State.Hitbox then
					if not Hitboxes[r] then
						Hitboxes[r] = {Size = r.Size, CanCollide = r.CanCollide}
					end
					local s = State.HitboxSize
					r.Size = Vector3.new(s, s, s)
					r.CanCollide = false
				elseif Hitboxes[r] then
					r.Size = Hitboxes[r].Size
					r.CanCollide = Hitboxes[r].CanCollide
					Hitboxes[r] = nil
				end
			end
		end
	end
end))

---------------------------------------------------------------------
-- Items: auto pickup + item ESP
---------------------------------------------------------------------
local function ensureItemTag(pp, part)
	if ItemTags[pp] then return end
	local bb = Instance.new("BillboardGui")
	bb.Name = "DW_ItemTag"
	bb.AlwaysOnTop = true
	bb.Size = UDim2.fromOffset(160, 36)
	bb.StudsOffset = Vector3.new(0, 2, 0)
	bb.Adornee = part
	bb.Parent = part
	local lbl = Instance.new("TextLabel")
	lbl.Name = "Label"
	lbl.BackgroundTransparency = 1
	lbl.Size = UDim2.fromScale(1, 1)
	lbl.TextColor3 = WHITE
	lbl.TextStrokeTransparency = 0
	lbl.Font = Enum.Font.GothamBold
	lbl.TextScaled = true
	lbl.Parent = bb
	ItemTags[pp] = bb
end

local function clearItemTags()
	for pp, bb in pairs(ItemTags) do
		pcall(function() bb:Destroy() end)
		ItemTags[pp] = nil
	end
end

task.spawn(function()
	while not Unloaded do
		task.wait(0.25)
		if State.Pickup and fireproximityprompt then
			local root = getRoot(getChar())
			if root then
				for _, pp in ipairs(Workspace:GetDescendants()) do
					if pp:IsA("ProximityPrompt") and pp.Enabled and not isCharacterPart(pp) then
						local part = promptPart(pp)
						if part and (part.Position - root.Position).Magnitude <= State.PickupRadius then
							pcall(fireproximityprompt, pp)
						end
					end
				end
			end
		end
	end
end)

task.spawn(function()
	while not Unloaded do
		task.wait(1)
		if State.ItemESP then
			local root = getRoot(getChar())
			for _, pp in ipairs(Workspace:GetDescendants()) do
				if pp:IsA("ProximityPrompt") and not isCharacterPart(pp) then
					local part = promptPart(pp)
					if part then
						ensureItemTag(pp, part)
						local bb = ItemTags[pp]
						if bb and bb:FindFirstChild("Label") then
							local name = (pp.ObjectText ~= "" and pp.ObjectText) or part.Name
							local dist = root and math.floor((part.Position - root.Position).Magnitude) or 0
							bb.Label.Text = string.format("%s [%dm]", name, dist)
						end
					end
				end
			end
			for pp, bb in pairs(ItemTags) do
				if not pp.Parent or not bb.Parent then
					pcall(function() bb:Destroy() end)
					ItemTags[pp] = nil
				end
			end
		end
	end
end)

---------------------------------------------------------------------
-- Server
---------------------------------------------------------------------
local function fetchServers(cursor)
	if not httpRequest then return nil end
	local url = ("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=100%s"):format(game.PlaceId, cursor or "")
	local ok, res = pcall(httpRequest, {Url = url, Method = "GET"})
	if not ok or not res or not res.Body then return nil end
	local okDecode, data = pcall(HttpService.JSONDecode, HttpService, res.Body)
	return okDecode and data or nil
end

local function serverHop()
	if not httpRequest then
		notify("Executor không hỗ trợ request")
		return
	end
	local cursor = nil
	local candidates = {}
	for _ = 1, 3 do
		local data = fetchServers(cursor)
		if not data or not data.data then break end
		for _, s in ipairs(data.data) do
			if s.id ~= game.JobId and s.playing < s.maxPlayers then
				candidates[#candidates + 1] = s.id
			end
		end
		if not data.nextPageCursor then break end
		cursor = "&cursor=" .. data.nextPageCursor
	end
	if #candidates == 0 then
		notify("Không tìm thấy server phù hợp")
		return
	end
	local id = candidates[math.random(#candidates)]
	pcall(TeleportService.TeleportToPlaceInstance, TeleportService, game.PlaceId, id, LP)
end

local function rejoin()
	if #Players:GetPlayers() <= 1 then
		pcall(TeleportService.Teleport, TeleportService, game.PlaceId, LP)
	else
		pcall(TeleportService.TeleportToPlaceInstance, TeleportService, game.PlaceId, game.JobId, LP)
	end
end

local function teleportJob(id)
	id = id and id:gsub("%s", "") or ""
	if id == "" then
		notify("Chưa nhập Job ID")
		return
	end
	pcall(TeleportService.TeleportToPlaceInstance, TeleportService, game.PlaceId, id, LP)
end

local function refreshPlayers()
	if not PlayerLabel then return end
	local lines = {}
	local myRoot = getRoot(getChar())
	local list = Players:GetPlayers()
	for _, p in ipairs(list) do
		local r = getRoot(getChar(p))
		local distText = "N/A"
		if myRoot and r then
			distText = math.floor((r.Position - myRoot.Position).Magnitude) .. "m"
		end
		lines[#lines + 1] = string.format("%s (@%s) | %s", p.DisplayName, p.Name, distText)
	end
	PlayerLabel.Text = "Người chơi (" .. #list .. ")\n" .. (#lines > 0 and table.concat(lines, "\n") or "Trống")
end

task.spawn(function()
	while not Unloaded do
		task.wait(2)
		if Pages.Server and Pages.Server.Visible then refreshPlayers() end
	end
end)

---------------------------------------------------------------------
-- Misc: Anti-AFK + Remote spy
---------------------------------------------------------------------
track(LP.Idled:Connect(function()
	if State.AntiAFK then
		VirtualUser:CaptureController()
		VirtualUser:ClickButton2(Vector2.new())
	end
end))

local function fmtArgs(...)
	local t = {}
	for i = 1, select("#", ...) do
		local v = select(i, ...)
		t[#t + 1] = typeof(v) .. ":" .. tostring(v)
	end
	return table.concat(t, ", ")
end

local function addSpyLog(method, path, args)
	table.insert(SpyLog, 1, string.format("[%s] %s %s(%s)", os.date("%X"), method, path, args))
	while #SpyLog > 25 do table.remove(SpyLog) end
	SpyDirty = true
end

if hasHook then
	local oldNamecall
	oldNamecall = hookmetamethod(game, "__namecall", newcclosure(function(self, ...)
		local method = getnamecallmethod()
		if State.RemoteSpy and not checkcaller()
			and (method == "FireServer" or method == "InvokeServer")
			and typeof(self) == "Instance"
			and (self:IsA("RemoteEvent") or self:IsA("RemoteFunction")) then
			pcall(addSpyLog, method, self:GetFullName(), fmtArgs(...))
		end
		return oldNamecall(self, ...)
	end))
end

task.spawn(function()
	while not Unloaded do
		task.wait(1)
		if SpyDirty and SpyLabel then
			SpyDirty = false
			SpyLabel.Text = "Remote Spy (" .. #SpyLog .. ")\n" .. (#SpyLog > 0 and table.concat(SpyLog, "\n") or "Chưa có log")
		end
	end
end)

---------------------------------------------------------------------
-- Feature control + handlers
---------------------------------------------------------------------
local function setFeature(key, v)
	State[key] = v
	if Handlers[key] then Handlers[key](v) end
	if Toggles[key] then Toggles[key](v, true) end
end

Handlers.Speed = function(v)
	local hum = getHum(getChar())
	if not v and hum then hum.WalkSpeed = 16 end
end

Handlers.Jump = function(v)
	local hum = getHum(getChar())
	if not v and hum then hum.JumpPower = 50 end
end

Handlers.Noclip = function(v)
	if v then return end
	local char = getChar()
	if not char then return end
	for _, part in ipairs(char:GetDescendants()) do
		if part:IsA("BasePart") then part.CanCollide = true end
	end
end

Handlers.ItemESP = function(v)
	if not v then clearItemTags() end
end

local CmdMap = {
	fly = "Fly", noclip = "Noclip", infjump = "InfJump", fling = "Fling", kill = "Kill",
	esp = "ESP", pickup = "Pickup", itemesp = "ItemESP", afk = "AntiAFK", spy = "RemoteSpy",
	speed = "Speed", jump = "Jump", hitbox = "Hitbox",
}

local function runCommand(text)
	local words = {}
	for w in text:gmatch("%S+") do words[#words + 1] = w end
	if #words == 0 then return end
	local cmd, arg = words[1]:lower(), words[2]
	local num = tonumber(arg)

	if cmd == "speed" and num then
		Sliders.SpeedVal(num)
		setFeature("Speed", true)
	elseif cmd == "jump" and num then
		Sliders.JumpVal(num)
		setFeature("Jump", true)
	elseif cmd == "hitbox" and num then
		Sliders.HitboxSize(num)
		setFeature("Hitbox", true)
	elseif cmd == "hop" then
		serverHop()
	elseif cmd == "rejoin" then
		rejoin()
	elseif cmd == "jobhop" then
		teleportJob(arg or "")
	elseif CmdMap[cmd] then
		setFeature(CmdMap[cmd], not (arg and arg:lower() == "off"))
	else
		notify("Lệnh không hợp lệ: " .. cmd)
	end
end

---------------------------------------------------------------------
-- UI kit
---------------------------------------------------------------------
local function setLock(v)
	Locked = v
	if LockBtn then
		LockBtn.Text = v and "UNLOCK" or "LOCK"
		tween(LockBtn, {
			BackgroundColor3 = v and WHITE or ITEM,
			TextColor3 = v and BLACK or WHITE,
		})
	end
end

local function makeDraggable(handle, target, onTap)
	local dragging, moved, startMouse, startPos = false, 0, nil, nil
	track(handle.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			moved = 0
			startMouse = i.Position
			startPos = target.Position
		end
	end))
	track(UIS.InputChanged:Connect(function(i)
		if dragging and not Locked and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
			local d = i.Position - startMouse
			moved = math.max(moved, d.Magnitude)
			target.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
		end
	end))
	track(UIS.InputEnded:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch) then
			dragging = false
			if onTap and moved < 8 then onTap() end
		end
	end))
end

local function sectionLabel(page, text)
	new("TextLabel", {
		Size = UDim2.new(1, 0, 0, 22), BackgroundTransparency = 1, Text = string.upper(text),
		TextColor3 = GREY, Font = Enum.Font.GothamBold, TextSize = 11,
		TextXAlignment = Enum.TextXAlignment.Left, Parent = page,
	})
end

local function row(page, h)
	return new("Frame", {Size = UDim2.new(1, 0, 0, h), BackgroundColor3 = ITEM, Parent = page}, {corner(8)})
end

local function toggle(page, label, key)
	local r = row(page, 34)
	new("TextLabel", {
		Size = UDim2.new(1, -60, 1, 0), Position = UDim2.fromOffset(10, 0), BackgroundTransparency = 1,
		Text = label, TextColor3 = WHITE, Font = Enum.Font.Gotham, TextSize = 13,
		TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd, Parent = r,
	})
	local pill = new("Frame", {Size = UDim2.fromOffset(36, 18), Position = UDim2.new(1, -46, 0.5, -9), BackgroundColor3 = OFF, Parent = r}, {corner(9)})
	local knob = new("Frame", {Size = UDim2.fromOffset(14, 14), Position = UDim2.fromOffset(2, 2), BackgroundColor3 = WHITE, Parent = pill}, {corner(7)})
	local hit = new("TextButton", {Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "", AutoButtonColor = false, Parent = r})

	local function set(v, silent)
		State[key] = v
		tween(pill, {BackgroundColor3 = v and WHITE or OFF})
		tween(knob, {
			Position = v and UDim2.fromOffset(20, 2) or UDim2.fromOffset(2, 2),
			BackgroundColor3 = v and BLACK or WHITE,
		})
		if not silent and Handlers[key] then Handlers[key](v) end
	end

	Toggles[key] = set
	track(hit.MouseButton1Click:Connect(function() set(not State[key]) end))
	set(State[key], true)
	return set
end

local function slider(page, label, key, min, max, suffix)
	local r = row(page, 48)
	local valLbl = new("TextLabel", {
		Size = UDim2.new(0, 80, 0, 18), Position = UDim2.new(1, -90, 0, 4), BackgroundTransparency = 1,
		Text = "", TextColor3 = WHITE, Font = Enum.Font.GothamBold, TextSize = 12,
		TextXAlignment = Enum.TextXAlignment.Right, Parent = r,
	})
	new("TextLabel", {
		Size = UDim2.new(1, -100, 0, 18), Position = UDim2.fromOffset(10, 4), BackgroundTransparency = 1,
		Text = label, TextColor3 = GREY, Font = Enum.Font.Gotham, TextSize = 12,
		TextXAlignment = Enum.TextXAlignment.Left, Parent = r,
	})
	local bar = new("Frame", {Size = UDim2.new(1, -20, 0, 4), Position = UDim2.fromOffset(10, 33), BackgroundColor3 = OFF, Parent = r}, {corner(2)})
	local fill = new("Frame", {Size = UDim2.fromScale(0, 1), BackgroundColor3 = WHITE, Parent = bar}, {corner(2)})
	local hit = new("TextButton", {Size = UDim2.new(1, 0, 0, 22), Position = UDim2.new(0, 0, 0, -9), BackgroundTransparency = 1, Text = "", AutoButtonColor = false, Parent = bar})

	local function apply(v)
		v = math.clamp(math.floor(v + 0.5), min, max)
		State[key] = v
		fill.Size = UDim2.fromScale((v - min) / (max - min), 1)
		valLbl.Text = v .. (suffix or "")
		return v
	end

	local dragging = false
	local function fromInput(i)
		local rel = math.clamp((i.Position.X - bar.AbsolutePosition.X) / math.max(bar.AbsoluteSize.X, 1), 0, 1)
		apply(min + (max - min) * rel)
	end

	track(hit.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			fromInput(i)
		end
	end))
	track(UIS.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
			fromInput(i)
		end
	end))
	track(UIS.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging = false
		end
	end))

	Sliders[key] = apply
	apply(State[key])
end

local function button(page, text, cb)
	local b = new("TextButton", {
		Size = UDim2.new(1, 0, 0, 32), BackgroundColor3 = ITEM, TextColor3 = WHITE, Text = text,
		Font = Enum.Font.GothamMedium, TextSize = 13, AutoButtonColor = false, Parent = page,
	}, {corner(8)})
	track(b.MouseButton1Click:Connect(function()
		tween(b, {BackgroundColor3 = WHITE, TextColor3 = BLACK}, 0.08)
		task.delay(0.12, function()
			if b.Parent then tween(b, {BackgroundColor3 = ITEM, TextColor3 = WHITE}, 0.2) end
		end)
		if cb then task.spawn(cb) end
	end))
	return b
end

local function input(page, placeholder)
	return new("TextBox", {
		Size = UDim2.new(1, 0, 0, 32), BackgroundColor3 = ITEM, TextColor3 = WHITE,
		PlaceholderText = placeholder, PlaceholderColor3 = GREY, Text = "",
		Font = Enum.Font.Gotham, TextSize = 12, ClearTextOnFocus = false, Parent = page,
	}, {corner(8)})
end

local function block(page, text)
	return new("TextLabel", {
		Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundColor3 = ITEM,
		TextColor3 = WHITE, Text = text, Font = Enum.Font.Code, TextSize = 11, TextWrapped = true,
		TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, Parent = page,
	}, {
		corner(8),
		new("UIPadding", {
			PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 8),
			PaddingTop = UDim.new(0, 6), PaddingBottom = UDim.new(0, 6),
		}),
	})
end

local function newPage(parent)
	return new("ScrollingFrame", {
		Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, BorderSizePixel = 0,
		ScrollBarThickness = 3, ScrollBarImageColor3 = WHITE, CanvasSize = UDim2.new(),
		AutomaticCanvasSize = Enum.AutomaticSize.Y, Visible = false, Parent = parent,
	}, {
		new("UIListLayout", {Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder}),
		new("UIPadding", {
			PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 10),
			PaddingTop = UDim.new(0, 8), PaddingBottom = UDim.new(0, 8),
		}),
	})
end

local function selectTab(name)
	for n, pg in pairs(Pages) do
		pg.Visible = (n == name)
		local b = TabBtns[n]
		if b then
			tween(b, {
				BackgroundColor3 = (n == name) and WHITE or ITEM,
				TextColor3 = (n == name) and BLACK or GREY,
			})
		end
	end
end

local function layoutMain()
	if not Main then return end
	local vs = Camera.ViewportSize
	Main.Size = UDim2.fromOffset(math.min(480, vs.X * 0.92), math.min(320, vs.Y * 0.82))
end

---------------------------------------------------------------------
-- Unload
---------------------------------------------------------------------
local function unload()
	if Unloaded then return end
	Unloaded = true
	State.RemoteSpy = false
	for _, c in ipairs(Conns) do pcall(function() c:Disconnect() end) end
	for p in pairs(ESPObjects) do removeESP(p) end
	for _, p in ipairs(Players:GetPlayers()) do setCham(getChar(p), false) end
	for r, o in pairs(Hitboxes) do
		if r.Parent then
			r.Size = o.Size
			r.CanCollide = o.CanCollide
		end
	end
	table.clear(Hitboxes)
	clearItemTags()
	local hum = getHum(getChar())
	if hum then
		hum.PlatformStand = false
		hum.WalkSpeed = 16
		hum.JumpPower = 50
	end
	if Gui then Gui:Destroy() end
end

---------------------------------------------------------------------
-- Main window
---------------------------------------------------------------------
local function buildFloat()
	FloatBar = new("Frame", {
		Name = "FloatBar", Size = UDim2.fromOffset(128, 34), Position = UDim2.fromOffset(12, 120),
		BackgroundColor3 = BLACK, Active = true, Parent = Gui,
	}, {corner(17), stroke(0.4)})

	MenuBtn = new("TextButton", {
		Size = UDim2.new(0.5, -6, 1, -8), Position = UDim2.fromOffset(4, 4), BackgroundColor3 = WHITE,
		Text = "MENU", TextColor3 = BLACK, Font = Enum.Font.GothamBlack, TextSize = 11,
		AutoButtonColor = false, Parent = FloatBar,
	}, {corner(13)})

	LockBtn = new("TextButton", {
		Size = UDim2.new(0.5, -6, 1, -8), Position = UDim2.new(0.5, 2, 0, 4), BackgroundColor3 = ITEM,
		Text = "LOCK", TextColor3 = WHITE, Font = Enum.Font.GothamBlack, TextSize = 10,
		AutoButtonColor = false, Parent = FloatBar,
	}, {corner(13)})

	makeDraggable(MenuBtn, FloatBar, function()
		if Main then Main.Visible = not Main.Visible end
	end)
	makeDraggable(LockBtn, FloatBar, function()
		setLock(not Locked)
	end)
	setLock(Locked)
end

local function buildMain()
	Main = new("Frame", {
		Name = "Main", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
		BackgroundColor3 = BLACK, ClipsDescendants = true, Active = true, Parent = Gui,
	}, {corner(12), stroke(0.55)})
	layoutMain()
	track(Camera:GetPropertyChangedSignal("ViewportSize"):Connect(layoutMain))

	local header = new("Frame", {Size = UDim2.new(1, 0, 0, 34), BackgroundColor3 = PANEL, Parent = Main})
	new("TextLabel", {
		Size = UDim2.new(1, -50, 1, 0), Position = UDim2.fromOffset(12, 0), BackgroundTransparency = 1,
		Text = "DARKWEN", TextColor3 = WHITE, Font = Enum.Font.GothamBlack, TextSize = 13,
		TextXAlignment = Enum.TextXAlignment.Left, Parent = header,
	})
	local close = new("TextButton", {
		Size = UDim2.fromOffset(26, 26), Position = UDim2.new(1, -32, 0.5, -13), BackgroundColor3 = ITEM,
		Text = "×", TextColor3 = WHITE, Font = Enum.Font.GothamBold, TextSize = 16, AutoButtonColor = false, Parent = header,
	}, {corner(6)})
	track(close.MouseButton1Click:Connect(function() Main.Visible = false end))
	makeDraggable(header, Main)

	local body = new("Frame", {Size = UDim2.new(1, 0, 1, -34), Position = UDim2.fromOffset(0, 34), BackgroundTransparency = 1, Parent = Main})
	local side = new("Frame", {Size = UDim2.new(0, 104, 1, -12), Position = UDim2.fromOffset(6, 6), BackgroundColor3 = PANEL, Parent = body}, {
		corner(8),
		new("UIListLayout", {Padding = UDim.new(0, 4), SortOrder = Enum.SortOrder.LayoutOrder}),
		new("UIPadding", {PaddingTop = UDim.new(0, 6), PaddingLeft = UDim.new(0, 6), PaddingRight = UDim.new(0, 6)}),
	})
	local content = new("Frame", {Size = UDim2.new(1, -122, 1, -12), Position = UDim2.fromOffset(116, 6), BackgroundTransparency = 1, Parent = body})

	for i, name in ipairs(TABS) do
		local b = new("TextButton", {
			Size = UDim2.new(1, 0, 0, 30), BackgroundColor3 = ITEM, Text = name, TextColor3 = GREY,
			Font = Enum.Font.GothamBold, TextSize = 12, AutoButtonColor = false, LayoutOrder = i, Parent = side,
		}, {corner(6)})
		TabBtns[name] = b
		Pages[name] = newPage(content)
		track(b.MouseButton1Click:Connect(function() selectTab(name) end))
	end

	-- Player
	local P = Pages.Player
	sectionLabel(P, "ESP")
	toggle(P, "ESP Master", "ESP")
	toggle(P, "Tên", "Name")
	toggle(P, "Khoảng cách", "Distance")
	toggle(P, "Máu", "Health")
	toggle(P, "Tracer", "Tracer")
	toggle(P, "Box", "Box")
	toggle(P, "Chams", "Chams")
	sectionLabel(P, "Movement")
	toggle(P, "Speed", "Speed")
	slider(P, "Tốc độ đi", "SpeedVal", 16, 300, " spd")
	toggle(P, "Jump Power", "Jump")
	slider(P, "Giá trị nhảy", "JumpVal", 50, 400, " jp")
	toggle(P, "Fly", "Fly")
	slider(P, "Tốc độ bay", "FlySpeed", 10, 300, " spd")
	toggle(P, "Noclip", "Noclip")
	toggle(P, "Infinite Jump", "InfJump")

	-- Combat
	local C = Pages.Combat
	sectionLabel(C, "Fling")
	toggle(C, "Fling Aura", "Fling")
	toggle(C, "Kill Aura (đẩy xuống void)", "Kill")
	slider(C, "Bán kính", "FlingRadius", 5, 60, " st")
	sectionLabel(C, "Hitbox")
	toggle(C, "Hitbox Expander", "Hitbox")
	slider(C, "Kích thước hitbox", "HitboxSize", 2, 40, " st")

	-- Items
	local I = Pages.Items
	sectionLabel(I, "Auto Pickup")
	toggle(I, "Auto Pickup", "Pickup")
	slider(I, "Bán kính nhặt", "PickupRadius", 5, 100, " st")
	sectionLabel(I, "Item ESP")
	toggle(I, "Item ESP", "ItemESP")

	-- Server
	local S = Pages.Server
	sectionLabel(S, "Hop")
	button(S, "Server Hop", serverHop)
	button(S, "Rejoin", rejoin)
	local jobBox = input(S, "Dán Job ID vào đây")
	button(S, "Teleport tới Job ID", function() teleportJob(jobBox.Text) end)
	sectionLabel(S, "Người chơi")
	PlayerLabel = block(S, "Đang tải...")
	button(S, "Làm mới danh sách", refreshPlayers)

	-- Misc
	local M = Pages.Misc
	sectionLabel(M, "Tiện ích")
	toggle(M, "Anti-AFK", "AntiAFK")
	toggle(M, "Remote Spy", "RemoteSpy")
	SpyLabel = block(M, "Remote Spy (0)\nChưa có log")
	button(M, "In log ra console", function()
		for _, l in ipairs(SpyLog) do print(l) end
	end)
	button(M, "Xóa log", function()
		table.clear(SpyLog)
		SpyDirty = true
	end)
	sectionLabel(M, "Command Bar")
	local cmdBox = input(M, "speed 150 | fly on | hop | jobhop <id>")
	track(cmdBox.FocusLost:Connect(function(enter)
		if enter then runCommand(cmdBox.Text) end
	end))
	button(M, "Chạy lệnh", function() runCommand(cmdBox.Text) end)
	sectionLabel(M, "Hệ thống")
	button(M, "Unload Darkwen", unload)

	selectTab("Player")
	buildFloat()
end

---------------------------------------------------------------------
-- Gate (mật khẩu)
---------------------------------------------------------------------
local function buildGate()
	local vs = Camera.ViewportSize
	local gate = new("Frame", {
		Name = "Gate", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
		Size = UDim2.fromOffset(math.min(290, vs.X - 24), 176), BackgroundColor3 = PANEL, Parent = Gui,
	}, {corner(12), stroke(0.5)})

	new("TextLabel", {
		Size = UDim2.new(1, 0, 0, 34), Position = UDim2.fromOffset(0, 14), BackgroundTransparency = 1,
		Text = "DARKWEN", TextColor3 = WHITE, Font = Enum.Font.GothamBlack, TextSize = 18, Parent = gate,
	})
	new("TextLabel", {
		Size = UDim2.new(1, 0, 0, 16), Position = UDim2.fromOffset(0, 42), BackgroundTransparency = 1,
		Text = "Nhập mật khẩu để tiếp tục", TextColor3 = GREY, Font = Enum.Font.Gotham, TextSize = 12, Parent = gate,
	})
	local box = new("TextBox", {
		Size = UDim2.new(1, -40, 0, 34), Position = UDim2.fromOffset(20, 70), BackgroundColor3 = ITEM,
		TextColor3 = WHITE, PlaceholderText = "Mật khẩu", PlaceholderColor3 = GREY, Text = "",
		Font = Enum.Font.Gotham, TextSize = 14, ClearTextOnFocus = false, Parent = gate,
	}, {corner(8), stroke(0.6)})
	local status = new("TextLabel", {
		Size = UDim2.new(1, 0, 0, 14), Position = UDim2.fromOffset(0, 108), BackgroundTransparency = 1,
		Text = "", TextColor3 = GREY, Font = Enum.Font.Gotham, TextSize = 11, Parent = gate,
	})
	local btn = new("TextButton", {
		Size = UDim2.new(1, -40, 0, 34), Position = UDim2.fromOffset(20, 130), BackgroundColor3 = WHITE,
		TextColor3 = BLACK, Text = "XÁC NHẬN", Font = Enum.Font.GothamBold, TextSize = 13,
		AutoButtonColor = false, Parent = gate,
	}, {corner(8)})

	local unlocked = false
	local function submit()
		if unlocked then return end
		if box.Text == PASSWORD then
			unlocked = true
			gate:Destroy()
			buildMain()
		else
			status.Text = "Sai mật khẩu"
			box.Text = ""
		end
	end

	track(btn.MouseButton1Click:Connect(submit))
	track(box.FocusLost:Connect(function(enter)
		if enter then submit() end
	end))
end

---------------------------------------------------------------------
-- Boot
---------------------------------------------------------------------
local guiParent = LP:WaitForChild("PlayerGui")
if gethui then
	guiParent = gethui()
else
	local ok, cg = pcall(function() return CoreGui end)
	if ok and cg then guiParent = cg end
end

local oldGui = guiParent:FindFirstChild("DarkwenUI")
if oldGui then oldGui:Destroy() end

Gui = new("ScreenGui", {
	Name = "DarkwenUI", ResetOnSpawn = false, IgnoreGuiInset = true, DisplayOrder = 999,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling, Parent = guiParent,
})

buildGate()
