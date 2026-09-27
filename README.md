
--[[
LUR UI v30 - Refined interface, UI preview edition.
The original pages, controls and demo-only behavior are preserved.
This revision focuses on responsive UX, accessibility, visual hierarchy and defensive UI state handling.
All gameplay-related controls change UI state only. No game actions or network requests.
Run as a LocalScript in StarterPlayerScripts. Profiles last for this UI session.

v30.6 changes:
- Fixed OffscreenRecovery (the small "bring the window back" handle) rendering
  with the wrong vertical position whenever the window was scrolled off the
  TOP or BOTTOM edge. It uses AnchorPoint (0, 0.5), so its Y coordinate must
  be the button's vertical CENTER, not its top edge - the top/bottom branches
  were missing that +/-23px (half of 46) correction, which is what made the
  handle look clipped/misaligned only in the vertical direction while
  left/right stayed fine.
- Fixed the handle not responding to a plain tap/click at all: it only ever
  listened for a drag, so pressing it without moving the mouse/finger did
  nothing. A tap now snaps the window fully back into view immediately;
  dragging it to a specific spot still works too.
- Fixed poor visibility/"looks buggy" complaints: the direction glyphs were
  special Unicode arrow characters ("›" "‹" "⌄" "⌃") that Roblox's built-in
  fonts don't reliably render, so they could show up blank. Swapped for
  plain ASCII arrows (">" "<" "v" "^"), bigger and bolder. Background now
  gets a subtle accent tint (via the theme system) instead of flat near-black,
  the border ring is thicker and fully opaque, and it now thickens further
  on hover for clearer "this is clickable" feedback.
- Added a subtle pulse animation to the recovery handle so it's easier to
  notice the moment the window goes off-screen (respects ReducedMotionEnabled;
  pauses automatically while being dragged).
]]

local VERSION = "30.6"

-- Services
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInput = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Stats = game:GetService("Stats")
local Http = game:GetService("HttpService")
local SoundService = game:GetService("SoundService")

assert(RunService:IsClient(), "[LurUI] Run this file as a LocalScript on the client")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer
assert(LocalPlayer, "[LurUI] LocalPlayer is unavailable; use a LocalScript")
local ParentGui = LocalPlayer:FindFirstChildOfClass("PlayerGui") or LocalPlayer:WaitForChild("PlayerGui", 10)
assert(ParentGui, "[LurUI] PlayerGui was not available after 10 seconds")

for _, name in ipairs({ "LurUI", "LurBubble" }) do
	local old = ParentGui:FindFirstChild(name)
	if old then old:Destroy() end
end

local IS_TOUCH = UserInput.TouchEnabled
local theme -- Forward-declared so early focus handlers can use the active accent safely.

local function IsMove(i)
	return i.UserInputType == Enum.UserInputType.MouseMovement
		or i.UserInputType == Enum.UserInputType.Touch
end

local function IsRelease(i)
	return i.UserInputType == Enum.UserInputType.MouseButton1
		or i.UserInputType == Enum.UserInputType.Touch
end

local function GetInputPos(inp)
	if inp and inp.Position then
		return Vector2.new(inp.Position.X, inp.Position.Y)
	end
	return UserInput:GetMouseLocation()
end

local C = {
	-- Neutral, slightly cool surfaces give the interface more depth without looking neon.
	Window = Color3.fromRGB(14, 16, 21),
	Sidebar = Color3.fromRGB(17, 20, 26),
	Title = Color3.fromRGB(18, 21, 28),
	Card = Color3.fromRGB(22, 25, 32),
	CardSub = Color3.fromRGB(18, 21, 27),
	SurfaceHover = Color3.fromRGB(31, 35, 44),
	Border = Color3.fromRGB(42, 47, 58),
	BorderLight = Color3.fromRGB(68, 75, 90),
	Search = Color3.fromRGB(27, 31, 39),
	White = Color3.fromRGB(245, 247, 250),
	Gray = Color3.fromRGB(158, 166, 181),
	GrayDk = Color3.fromRGB(91, 99, 115),
	Off = Color3.fromRGB(49, 55, 67),
	Success = Color3.fromRGB(34, 197, 94),
	Warning = Color3.fromRGB(234, 179, 8),
	Danger = Color3.fromRGB(239, 68, 68),
}

-- Color utilities used by controls. Keep hue stable and only adjust luminance.
local function MixColor(a, b, t)
	t = math.clamp(tonumber(t) or 0, 0, 1)
	return Color3.new(
		a.R + (b.R - a.R) * t,
		a.G + (b.G - a.G) * t,
		a.B + (b.B - a.B) * t
	)
end

local function TintColor(color, amount)
	return MixColor(color, Color3.new(1, 1, 1), amount)
end

local function ShadeColor(color, amount)
	return MixColor(color, Color3.new(0, 0, 0), amount)
end

local EASE = TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
local SPRING = TweenInfo.new(0.14, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
local ROW_H = 64
local HEADER_H = 28
local GAP = 14
local SIDEBAR_W = IS_TOUCH and 205 or 250

-- Centralize long-lived UI state, cleanup, responsive metadata and interaction helpers.
local Refine = {
	alive = true,
	connections = {},
	rowDefaults = setmetatable({}, {__mode = "k"}),
	rowSearch = setmetatable({}, {__mode = "k"}),
}
Refine.FooterHeight = 28
Refine.guiService = game:GetService("GuiService")
Refine.tweens = {}
Refine.tweenOwners = setmetatable({}, {__mode="k"})
Refine.navigation = false
Refine.actions = {}
function Refine.Connect(signal, callback)
	local connection = signal:Connect(callback)
	table.insert(Refine.connections, connection)
	return connection
end

function Refine.SafeCall(context, callback, ...)
	if type(callback) ~= "function" then return true end
	local ok, result = pcall(callback, ...)
	if not ok then
		warn(string.format("[LurUI] %s failed: %s", tostring(context or "callback"), tostring(result)))
		if Refine.ReportError then Refine.ReportError(result) end
	end
	return ok, result
end

function Refine.ClosePopup()
	local close=Refine.popup
	Refine.popup=nil
	if close then close() end
end
function Refine.StopTween(object)
	local previous = Refine.tweens[object]
	if previous then previous.tween:Cancel(); Refine.tweens[object] = nil end
end
function Refine.Tween(object, info, properties)
	Refine.StopTween(object)
	local tween = TweenService:Create(object, Refine.reducedMotion and TweenInfo.new(0) or info, properties)
	local entry = {tween=tween, properties=properties}
	Refine.tweens[object] = entry
	tween.Completed:Connect(function()
		if Refine.tweens[object] == entry then Refine.tweens[object] = nil end
	end)
	if not Refine.tweenOwners[object] then
		Refine.tweenOwners[object] = true
		object.Destroying:Connect(function()
			local active = Refine.tweens[object]
			if active then active.tween:Cancel(); Refine.tweens[object] = nil end
		end)
	end
	return tween
end
pcall(function()
	Refine.reducedMotion = Refine.guiService.ReducedMotionEnabled == true
	Refine.Connect(Refine.guiService:GetPropertyChangedSignal("ReducedMotionEnabled"), function()
		Refine.reducedMotion = Refine.guiService.ReducedMotionEnabled == true
		if Refine.reducedMotion then
			for object,entry in pairs(Refine.tweens) do
				entry.tween:Cancel()
				for property,value in pairs(entry.properties) do object[property] = value end
			end
			table.clear(Refine.tweens)
		end
	end)
end)
function Refine.IsVisible(object)
	if not object or not object.Parent then return false end
	local ancestor = object
	while ancestor do
		if ancestor == Refine.sidebar and not Refine.sidebarVisible then return false end
		if ancestor:IsA("GuiObject") and (not ancestor.Visible or ancestor.AbsoluteSize.X <= 0 or ancestor.AbsoluteSize.Y <= 0) then return false end
		if ancestor:IsA("ScreenGui") then return ancestor.Enabled end
		ancestor = ancestor.Parent
	end
	return false
end
function Refine.FocusFirst(container)
	if not Refine.navigation or not container then return end
	local fallback
	for _,object in ipairs(container:GetDescendants()) do
		if object:IsA("GuiObject") and object.Selectable and Refine.IsVisible(object) then
			fallback = fallback or object
			local inView, ancestor = true, object.Parent
			while ancestor do
				if ancestor:IsA("ScrollingFrame") then
					local position, size = object.AbsolutePosition, object.AbsoluteSize
					local origin, view = ancestor.AbsolutePosition, ancestor.AbsoluteWindowSize
					if position.Y + size.Y <= origin.Y or position.Y >= origin.Y + view.Y then inView = false; break end
				end
				ancestor = ancestor.Parent
			end
			if inView then Refine.guiService.SelectedObject = object; return object end
		end
	end
	if fallback then Refine.guiService.SelectedObject = fallback end
	return fallback
end
function Refine.RestoreFocus(opener)
	if Refine.navigation then
		local target = opener or Refine.popupOpener or Refine.focusEntry
		if target and Refine.IsVisible(target) then Refine.guiService.SelectedObject = target end
	end
	Refine.popupOpener = nil
end
function Refine.FocusPopup(container, opener)
	Refine.popupOpener = opener
	container.SelectionGroup = true
	container.SelectionBehaviorUp = Enum.SelectionBehavior.Stop
	container.SelectionBehaviorDown = Enum.SelectionBehavior.Stop
	container.SelectionBehaviorLeft = Enum.SelectionBehavior.Stop
	container.SelectionBehaviorRight = Enum.SelectionBehavior.Stop
	Refine.FocusFirst(container)
end
function Refine.New(cls, props, parent)
	local i = Instance.new(cls)
	if i:IsA("GuiObject") then i.BorderSizePixel = 0 end
	for k, v in pairs(props or {}) do
		i[k] = v
	end
	if parent then
		i.Parent = parent
	end
	return i
end

function Refine.Round(p, r)
	Refine.New("UICorner", { CornerRadius = UDim.new(0, r) }, p)
end

function Refine.Shade(obj)
	Refine.New("UIGradient", {
		Color = ColorSequence.new(Color3.new(1, 1, 1), Color3.new(0.86, 0.86, 0.86)),
		Rotation = 90,
	}, obj)
end

function Refine.Chevron(parent, pos)
	local w = Refine.New("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = pos,
		Size = UDim2.new(0, 12, 0, 8),
		BackgroundTransparency = 1,
		ZIndex = 3,
	}, parent)

	Refine.New("Frame", {
		AnchorPoint = Vector2.new(0, 0.5),
		Position = UDim2.new(0, 1, 0, 4),
		Size = UDim2.new(0, 7, 0, 1.6),
		Rotation = 45,
		BackgroundColor3 = C.Gray,
		BorderSizePixel = 0,
		ZIndex = 3,
	}, w)

	Refine.New("Frame", {
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(0, 11, 0, 4),
		Size = UDim2.new(0, 7, 0, 1.6),
		Rotation = -45,
		BackgroundColor3 = C.Gray,
		BorderSizePixel = 0,
		ZIndex = 3,
	}, w)

	return w
end

local ICONS = {
	bolt = { {8,1,4,9,20}, {6,5,5,3,20}, {4,7,4,9,20} },
	quest = { {3,1,10,14,nil,2,true}, {5,5,6,1.5}, {5,8,6,1.5}, {5,11,4,1.5} },
	user = { {5,2,6,6,nil,"f"}, {3,9,10,5,0,2} },
	layers = { {2,3,12,2.5,0,1}, {3,7,10,2.5,0,1}, {4,11,8,2.5,0,1} },
	chart = { {3,9,2.5,6,0,1}, {7,5,2.5,10,0,1}, {11,7,2.5,8,0,1} },
	book = { {2,3,12,10,nil,2,true}, {7.5,3,1,10} },
	spark = { {5,5,6,6,45,1}, {11,2,2.5,2.5,0,"f"}, {2,11,2.5,2.5,0,"f"} },
	target = { {3,3,10,10,nil,"f",true}, {6.5,6.5,3,3,0,"f"} },
	wave = { {2,5,12,2,10,1}, {2,9,12,2,-10,1} },
	skull = { {4,2,8,9,nil,"f"}, {5.5,5,2,2,0,"f",nil,true}, {8.5,5,2,2,0,"f",nil,true}, {5,12,2,2,0,1}, {9,12,2,2,0,1} },
	bell = { {4,3,8,8,0,4}, {7,12,2,2,0,"f"} },
	console = { {2,3,12,10,nil,2,true}, {4.5,6,3,1.5,45}, {4.5,8,3,1.5,-45}, {8,9,3,1.5} },
	gear = { {5,5,6,6,nil,"f",true}, {7,1,2,3,0,1}, {7,12,2,3,0,1}, {1,7,3,2,0,1}, {12,7,3,2,0,1} },
	dot = { {6,6,4,4,0,"f"} },
}

function Refine.DrawIcon(kind, parent, x)
	local spec = ICONS[kind] or ICONS.dot
	local box = Refine.New("Frame", {
		Position = UDim2.new(0, x, 0.5, -8),
		Size = UDim2.new(0, 16, 0, 16),
		BackgroundTransparency = 1,
		ZIndex = 2,
	}, parent)

	local updaters = {}

	for _, p in ipairs(spec) do
		local f = Refine.New("Frame", {
			Position = UDim2.new(0, p[1], 0, p[2]),
			Size = UDim2.new(0, p[3], 0, p[4]),
			BorderSizePixel = 0,
			ZIndex = 2,
		}, box)

		if p[5] then f.Rotation = p[5] end

		if p[6] == "f" then
			Refine.Round(f, math.min(p[3], p[4]) / 2)
		elseif p[6] then
			Refine.Round(f, p[6])
		end

		if p[8] then
			f.BackgroundColor3 = C.Sidebar
		elseif p[7] then
			f.BackgroundTransparency = 1
			local s = Refine.New("UIStroke", { Thickness = 1.2 }, f)
			table.insert(updaters, function(c) s.Color = c end)
		else
			table.insert(updaters, function(c) f.BackgroundColor3 = c end)
		end
	end

	return function(c)
		for _, u in ipairs(updaters) do
			u(c)
		end
	end
end

local isPanning = false
local isSliderActive = false
local bindCancelTs = 0
local soundEnabled = true
local clickSound = nil

pcall(function()
	clickSound = Instance.new("Sound")
	clickSound.SoundId = "rbxasset://sounds/electronicpingshort.wav"
	clickSound.Volume = 0.2
	clickSound.PlaybackSpeed = 1.15
	clickSound.Parent = SoundService
end)

function Refine.Click(obj, fn)
	local lastClick = 0
	local touchInput, touchStart
	obj.InputBegan:Connect(function(input)
		if input.UserInputType==Enum.UserInputType.Touch and not touchInput then
			touchInput, touchStart = input, input.Position
		end
	end)
	obj.InputEnded:Connect(function(input)
		if input == touchInput then
			task.defer(function() if touchInput == input then touchInput, touchStart = nil, nil end end)
		end
	end)
	local function fire(input)
		if not Refine.alive or Refine.systemMenuOpen then return end
		if input and input.UserInputType==Enum.UserInputType.Touch and touchInput and input~=touchInput then return end
		if input and touchStart and input.UserInputType==Enum.UserInputType.Touch and (input.Position-touchStart).Magnitude>10 then return end
		if os.clock() - lastClick < 0.04 then return end
		if os.clock() - bindCancelTs < 0.15 then return end
		if isPanning or isSliderActive then return end
		lastClick = os.clock()

		if soundEnabled and clickSound then
			pcall(function()
				clickSound:Play()
			end)
		end

		local ok, message = pcall(fn)
		if not ok then
			warn("[LurUI] Control callback failed: " .. tostring(message))
			if Refine.ReportError then Refine.ReportError(message) end
		end
	end

	obj.Selectable=true
	obj.Activated:Connect(fire)
	Refine.actions[obj] = fire
	obj.Destroying:Connect(function() Refine.actions[obj] = nil end)
end

function Refine.AddTactileFeedback(button)
	local scale = button:FindFirstChildOfClass("UIScale") or Refine.New("UIScale", { Scale = 1 }, button)

	button.InputBegan:Connect(function(inp)
		if inp.UserInputType == Enum.UserInputType.MouseButton1
			or inp.UserInputType == Enum.UserInputType.Touch then
			scale.Scale = Refine.reducedMotion and 1 or 0.97
		end
	end)
	button.MouseLeave:Connect(function() scale.Scale = 1 end)
	button.SelectionLost:Connect(function() scale.Scale = 1 end)

	button.InputEnded:Connect(function(inp)
		if IsRelease(inp) then
			scale.Scale = 1
		end
	end)
end

function Refine.EnableTouchPan(sf)
	sf.Active=true
	sf.ScrollingEnabled=true
	sf.ScrollingDirection=Enum.ScrollingDirection.Y
	sf.ElasticBehavior=Enum.ElasticBehavior.WhenScrollable
end

local activeSlider = nil
local activeSliderCommit = nil
function Refine.EndSlider()
	if activeSliderCommit then pcall(activeSliderCommit) end
	activeSlider,activeSliderCommit,Refine.sliderInput=nil,nil,nil
	isSliderActive=false
end

Refine.Connect(UserInput.InputChanged, function(inp)
	local source = Refine.sliderInput
	if not activeSlider or not source then return end
	if source.UserInputType == Enum.UserInputType.MouseButton1 and inp.UserInputType == Enum.UserInputType.MouseMovement then
		activeSlider(GetInputPos(inp).X)
	elseif inp == source then
		activeSlider(GetInputPos(inp).X)
	end
end)

-- Touch dragging is handled explicitly because TouchMoved/TouchEnded is more reliable
-- for mobile sliders than relying on a TextButton's InputChanged alone.
Refine.Connect(UserInput.TouchMoved, function(inp)
	if activeSlider and Refine.sliderInput == inp then
		activeSlider(inp.Position.X)
	end
end)

Refine.Connect(UserInput.TouchEnded, function(inp)
	if inp == Refine.sliderInput then Refine.EndSlider() end
end)

Refine.Connect(UserInput.InputEnded, function(inp)
	if inp == Refine.sliderInput and inp.UserInputType ~= Enum.UserInputType.Touch then
		Refine.EndSlider()
	end
end)

local Gui = Refine.New("ScreenGui", {
	Name = "LurUI",
	ResetOnSpawn = false,
	IgnoreGuiInset = true,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	DisplayOrder = 999,
	Parent = ParentGui,
})

Gui.Destroying:Connect(function()
	Refine.EndSlider()
	Refine.alive=false
	for _,entry in pairs(Refine.tweens) do entry.tween:Cancel() end
	table.clear(Refine.tweens)
	table.clear(Refine.actions)
	local selected = Refine.guiService.SelectedObject
	if selected and selected:IsDescendantOf(Gui) then Refine.guiService.SelectedObject = nil end
	for _,connection in ipairs(Refine.connections) do connection:Disconnect() end
	table.clear(Refine.connections)
	if clickSound then clickSound:Destroy() end
end)
Gui:GetPropertyChangedSignal("Enabled"):Connect(function()
	if not Gui.Enabled then Refine.EndSlider(); Refine.ClosePopup() end
end)
local vp = Vector2.new(1280, 720)
if Camera and Camera.ViewportSize.X>=240 and Camera.ViewportSize.Y>=180 then
	vp = Camera.ViewportSize
end

local TOP_GAP = IS_TOUCH and 70 or 36
function Refine.DefaultBounds()
	if not IS_TOUCH then return math.min(vp.X - 12, 980), math.min(vp.Y - TOP_GAP - 16, 664) end
	local landscape = vp.X > vp.Y
	local availableHeight = math.max(100, vp.Y - TOP_GAP - 24)
	local width = landscape and math.max(500, math.min(580, vp.X * 0.72)) or 420
	return math.min(vp.X - 24, width), math.min(availableHeight, landscape and math.min(340, availableHeight * 0.92) or 420)
end
function Refine.HeaderHeight(width)
	if width < (IS_TOUCH and 500 or 720) then return IS_TOUCH and 88 or 100 end
	return IS_TOUCH and 52 or 56
end
local curW, curH = Refine.DefaultBounds()
Refine.touchLandscape = vp.X > vp.Y
local NARROW = curW < (IS_TOUCH and 500 or 720)
local TITLE_H = Refine.HeaderHeight(curW)

local shadow1 = Refine.New("Frame", {
	ZIndex = 0,
	BackgroundColor3 = Color3.fromRGB(0, 0, 0),
	BackgroundTransparency = 0.55,
	Parent = Gui,
})
Refine.Round(shadow1, 17)

local Window = Refine.New("Frame", {
	Name = "LurWindow",
	SelectionGroup = true,
	SelectionBehaviorUp = Enum.SelectionBehavior.Stop,
	SelectionBehaviorDown = Enum.SelectionBehavior.Stop,
	SelectionBehaviorLeft = Enum.SelectionBehavior.Stop,
	SelectionBehaviorRight = Enum.SelectionBehavior.Stop,
	ZIndex = 1,
	AnchorPoint = Vector2.new(0, 0),
	Position = UDim2.fromOffset((vp.X - curW) / 2, TOP_GAP + (vp.Y - TOP_GAP - curH) / 2),
	Size = UDim2.new(0, curW, 0, curH),
	BackgroundColor3 = C.Window,
	ClipsDescendants = true,
	Parent = Gui,
})
Refine.Round(Window, 14)
Refine.New("UIStroke", { Color = C.Border, Thickness = 1.2 }, Window)

local intro = Refine.New("UIScale", { Scale = 0.96 }, Window)
Refine.Tween(intro, SPRING, { Scale = 1 }):Play()

local function SyncShadow()
	local a = Window.Position.X.Offset
	local b = Window.Position.Y.Offset
	local w = Window.Size.X.Offset
	local h = Window.Size.Y.Offset

	shadow1.Position = UDim2.fromOffset(a - 4, b - 3)
	shadow1.Size = UDim2.new(0, w + 8, 0, h + 7)
end

Window:GetPropertyChangedSignal("Position"):Connect(SyncShadow)
Window:GetPropertyChangedSignal("Size"):Connect(SyncShadow)
SyncShadow()

local TitleBar = Refine.New("Frame", {
	Name = "LurTitleBar",
	Size = UDim2.new(1, 0, 0, TITLE_H),
	BackgroundColor3 = C.Title,
	Parent = Window,
})
Refine.Round(TitleBar, 14)
Refine.New("Frame", {
	Name = "TitleCornerFill", Position = UDim2.fromOffset(0, 14), Size = UDim2.new(1, 0, 1, -14),
	BackgroundColor3 = C.Title,
}, TitleBar)

Refine.New("Frame", {
	Position = UDim2.new(0, 0, 1, 0),
	Size = UDim2.new(1, 0, 0, 1),
	BackgroundColor3 = C.Border,
	BorderSizePixel = 0,
}, TitleBar)

local minimized, maximized, savedPos = false, false, nil

local Body = Refine.New("Frame", {
	Position = UDim2.new(0, 0, 0, TITLE_H),
	Size = UDim2.new(1, 0, 1, -TITLE_H),
	BackgroundTransparency = 1,
	Parent = Window,
})

local GS = IS_TOUCH and 28 or 18
local ResizeGrip = Refine.New("TextButton", {
	AnchorPoint = Vector2.new(1, 1),
	Position = UDim2.new(1, -6, 1, -6),
	Size = UDim2.new(0, GS, 0, GS),
	BackgroundTransparency = 1,
	AutoButtonColor = false,
	Text = "",
	ZIndex = 4,
}, Window)

for i = 1, 3 do
	Refine.New("Frame", {
		Position = UDim2.new(0, 2 + i * 3, 0, 16 - i * 3),
		Size = UDim2.new(0, 16 - i * 4, 0, 1.6),
		Rotation = -45,
		BackgroundColor3 = C.GrayDk,
		BorderSizePixel = 0,
		ZIndex = 4,
	}, ResizeGrip)
end

local function SetMinimized(on)
	Refine.EndSlider(); Refine.ClosePopup()
	if on then maximized=false end
	minimized = on
	Body.Visible = not on
	ResizeGrip.Visible = not on
	if on then Refine.RestoreFocus() end
	TITLE_H = Refine.HeaderHeight(curW)
	Refine.Tween(Window, EASE, {
		Size = UDim2.new(0, curW, 0, on and TITLE_H or curH),
	}):Play()
end

local function SetMaximized(on)
	Refine.EndSlider(); Refine.ClosePopup()
	maximized = on

	if minimized then
		SetMinimized(false)
	end

	if on then
		savedPos = Window.Position
		Refine.Tween(Window, EASE, {
			Position = UDim2.fromOffset(6, TOP_GAP + 6),
			Size = UDim2.new(0, vp.X - 12, 0, vp.Y - TOP_GAP - 12),
		}):Play()
	else
		local restore = savedPos or UDim2.fromOffset((vp.X - curW) / 2, TOP_GAP + (vp.Y - TOP_GAP - curH) / 2)
		Refine.Tween(Window, EASE, {
			Position = UDim2.fromOffset(
				math.clamp(restore.X.Offset, 6, math.max(6, vp.X - curW - 6)),
				math.clamp(restore.Y.Offset, TOP_GAP + 6, math.max(TOP_GAP + 6, vp.Y - curH - 6))
			),
			Size = UDim2.new(0, curW, 0, curH),
		}):Play()
	end
end

local TS = IS_TOUCH and 16 or 12
for i, d in ipairs({
	{ Color3.fromRGB(255, 95, 87), "x" },
	{ Color3.fromRGB(254, 188, 46), "-" },
	{ Color3.fromRGB(40, 200, 64), "+" },
}) do
	local dot = Refine.New("TextButton", {
		Size = UDim2.new(0, TS, 0, TS),
		Position = UDim2.new(0, IS_TOUCH and (12 + (i - 1) * 22) or (16 + (i - 1) * 20), 0, 28 - TS / 2),
		BackgroundColor3 = d[1],
		BorderSizePixel = 0,
		AutoButtonColor = false,
		Text = "",
	}, TitleBar)
	Refine.Round(dot, TS / 2)

	local sym = Refine.New("TextLabel", {
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundTransparency = 1,
		Text = d[2],
		TextColor3 = Color3.fromRGB(60, 30, 10),
		Font = Enum.Font.GothamBold,
		TextSize = 9,
		TextTransparency = IS_TOUCH and 0.15 or 1,
	}, dot)

	dot.MouseEnter:Connect(function() sym.TextTransparency = 0.15 end)
	dot.MouseLeave:Connect(function() sym.TextTransparency = IS_TOUCH and 0.15 or 1 end)

	Refine.Click(dot, function()
		if i == 1 then
			Gui.Enabled = not Gui.Enabled
		elseif i == 2 then
			SetMinimized(not minimized)
		else
			SetMaximized(not maximized)
		end
	end)
	if IS_TOUCH then
		local touchHit = Refine.New("TextButton", {
			Name = "TitleTouchTarget", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
			Size = UDim2.fromOffset(22, 40), BackgroundTransparency = 1, Text = "", AutoButtonColor = false,
		}, dot)
		dot.Selectable = false
		Refine.Click(touchHit, function() Refine.actions[dot]() end)
	end
end

local panelBtn = Refine.New("TextButton", {
	Name = "LurNavigationToggle",
	Position = UDim2.new(0, 76, 0, IS_TOUCH and 8 or 20),
	Size = UDim2.new(0, 22, 0, IS_TOUCH and 40 or 18),
	BackgroundTransparency = 1,
	AutoButtonColor = false,
	Text = "",
	ZIndex = 3,
}, TitleBar)
Refine.focusEntry = panelBtn

local panelBox = Refine.New("Frame", {
	Position = UDim2.new(0, 1, 0, IS_TOUCH and 13 or 1),
	Size = UDim2.new(0, 18, 0, 15),
	BackgroundTransparency = 1,
	ZIndex = 3,
}, panelBtn)
Refine.Round(panelBox, 3)

Refine.New("UIStroke", { Color = C.Gray, Thickness = 1.2 }, panelBox)
Refine.New("Frame", {
	Size = UDim2.new(0, 5, 1, 0),
	BackgroundColor3 = C.Gray,
	BorderSizePixel = 0,
	ZIndex = 3,
}, panelBox)

Refine.New("TextLabel", {
	Position = UDim2.new(0, 110, 0, 11),
	Size = UDim2.new(0, 180, 0, 20),
	BackgroundTransparency = 1,
	Text = "Lur UI",
	TextColor3 = C.White,
	Font = Enum.Font.GothamBold,
	TextSize = 17,
	TextXAlignment = Enum.TextXAlignment.Left,
}, TitleBar)

if not NARROW then
	Refine.New("TextLabel", {
		Name = "VersionLabel",
		Position = UDim2.new(0, 110, 0, 31),
		Size = UDim2.new(0, 180, 0, 16),
		BackgroundTransparency = 1,
		Text = "Roblox Edition v" .. VERSION,
		TextColor3 = C.Gray,
		Font = Enum.Font.Gotham,
		TextSize = 11,
		TextXAlignment = Enum.TextXAlignment.Left,
	}, TitleBar)
end

local SearchFrame
if NARROW then
	SearchFrame = Refine.New("Frame", {
		Position = UDim2.new(0, 76, 0, 58),
		Size = UDim2.new(1, -88, 0, 30),
		BackgroundColor3 = C.Search,
	}, TitleBar)
else
	SearchFrame = Refine.New("Frame", {
		Position = UDim2.new(1, -240, 0.5, -17),
		Size = UDim2.new(0, 224, 0, 34),
		BackgroundColor3 = C.Search,
	}, TitleBar)
end
Refine.Round(SearchFrame, 8)

local searchStroke = Refine.New("UIStroke", { Color = C.Border, Thickness = 1 }, SearchFrame)

local lens = Refine.New("Frame", {
	AnchorPoint = Vector2.new(0, 0.5),
	Position = UDim2.new(0, 12, 0.5, -1),
	Size = UDim2.new(0, 11, 0, 11),
	BackgroundTransparency = 1,
}, SearchFrame)
Refine.Round(lens, 5)

Refine.New("UIStroke", { Color = C.Gray, Thickness = 1.4 }, lens)
Refine.New("Frame", {
	Position = UDim2.new(0, 21, 0.5, 3),
	Size = UDim2.new(0, 1.6, 0, 6),
	Rotation = -45,
	BackgroundColor3 = C.Gray,
	BorderSizePixel = 0,
}, SearchFrame)

local SearchBox = Refine.New("TextBox", {
	Position = UDim2.new(0, 32, 0, 0),
	Size = UDim2.new(1, -60, 1, 0),
	BackgroundTransparency = 1,
	Text = "",
	PlaceholderText = "Search controls...",
	PlaceholderColor3 = C.Gray,
	TextColor3 = C.White,
	Font = Enum.Font.Gotham,
	TextSize = 13,
	TextXAlignment = Enum.TextXAlignment.Left,
	ClearTextOnFocus = false,
}, SearchFrame)

local searchClearBtn = Refine.New("TextButton", {
	AnchorPoint = Vector2.new(1, 0.5),
	Position = UDim2.new(1, -8, 0.5, 0),
	Size = UDim2.new(0, 18, 0, 18),
	BackgroundTransparency = 1,
	Text = "✕",
	TextColor3 = C.Gray,
	Font = Enum.Font.GothamBold,
	TextSize = 10,
	Visible = false,
}, SearchFrame)

Refine.Click(searchClearBtn, function()
	SearchBox.Text = ""
end)

SearchBox.Focused:Connect(function()
	searchStroke.Color = theme and theme.Accent or C.BorderLight
end)

SearchBox.FocusLost:Connect(function()
	searchStroke.Color = C.Border
end)

local DragRegion = Refine.New("TextButton", {
	Position = UDim2.new(0, 100, 0, 0),
	Size = UDim2.new(0, 120, 0, 56),
	BackgroundTransparency = 1,
	AutoButtonColor = false,
	Text = "",
	ZIndex = 2,
}, TitleBar)

local dragging, grabD = false, nil
local titleTapStart = nil
local lastTitleTap = 0
local titleTapPending = false

local function UpdateDragRegion()
	local reserved = NARROW and 210 or 430
	local w = math.max(70, Window.AbsoluteSize.X - reserved - 100)
	DragRegion.Size = UDim2.new(0, w, 0, math.min(56, TITLE_H))
end

-- Free window positioning: the user may move the window substantially beyond
-- any viewport edge. A small recovery strip remains reachable so the window
-- can always be brought back. This behavior is intentional on desktop too,
-- matching the movable-window behavior requested by the mobile layout.
local RECOVERY_X = 56
local RECOVERY_TOP = 52
local RECOVERY_BOTTOM = 52

local function GetDragBounds()
	local width = math.max(1, Window.AbsoluteSize.X)
	local height = math.max(1, Window.AbsoluteSize.Y)
	local visibleTop = math.max(28, math.min(RECOVERY_TOP, math.max(28, TITLE_H - 6)))
	return
		-width + RECOVERY_X,
		vp.X - RECOVERY_X,
		-height + visibleTop,
		vp.Y - RECOVERY_BOTTOM
end

local function SetWindowPositionClamped(x, y)
	local minX, maxX, minY, maxY = GetDragBounds()
	x = tonumber(x) or Window.Position.X.Offset
	y = tonumber(y) or Window.Position.Y.Offset
	Window.Position = UDim2.fromOffset(
		math.clamp(x, minX, maxX),
		math.clamp(y, minY, maxY)
	)
end

Window:GetPropertyChangedSignal("AbsoluteSize"):Connect(UpdateDragRegion)
UpdateDragRegion()

DragRegion.InputBegan:Connect(function(inp)
	if inp.UserInputType == Enum.UserInputType.MouseButton1
		or inp.UserInputType == Enum.UserInputType.Touch then
		if dragging then return end
		if not maximized then Refine.StopTween(Window) end
		dragging = inp
		titleTapStart = GetInputPos(inp)
		grabD = GetInputPos(inp) - Vector2.new(
			Window.Position.X.Offset,
			Window.Position.Y.Offset
		)
	end
end)

DragRegion.InputEnded:Connect(function(inp)
	if inp~=dragging or not titleTapStart then return end

	local moved = (GetInputPos(inp) - titleTapStart).Magnitude > 6
	titleTapStart = nil

	if moved then
		titleTapPending = false
		return
	end

	local now = os.clock()

	if titleTapPending and now - lastTitleTap < 0.35 then
		titleTapPending = false
		SetMaximized(not maximized)
	else
		titleTapPending = true
	end

	lastTitleTap = now
end)

DragRegion.MouseButton2Click:Connect(function()
	SetMaximized(not maximized)
end)

Refine.Connect(UserInput.InputEnded, function(inp)
	if inp==dragging then
		task.defer(function() if dragging==inp then dragging=false end end)
	end
end)

Refine.Connect(UserInput.InputChanged, function(inp)
	if dragging and not maximized and (inp==dragging or (dragging.UserInputType==Enum.UserInputType.MouseButton1 and inp.UserInputType==Enum.UserInputType.MouseMovement)) then
		local m = GetInputPos(inp)
		SetWindowPositionClamped(m.X - grabD.X, m.Y - grabD.Y)
	end
end)

-- Touch has a dedicated movement signal; use it so off-screen dragging remains
-- reliable even when the original title-bar hit target leaves the viewport.
Refine.Connect(UserInput.TouchMoved, function(touch, _processed)
	if dragging == touch and not maximized then
		local m = GetInputPos(touch)
		SetWindowPositionClamped(m.X - grabD.X, m.Y - grabD.Y)
	end
end)

Refine.Connect(UserInput.TouchEnded, function(touch, _processed)
	if dragging == touch then
		dragging = false
		titleTapStart = nil
	end
end)

local resizing, rStart, rStartSize = false, nil, nil

ResizeGrip.InputBegan:Connect(function(inp)
	if inp.UserInputType == Enum.UserInputType.MouseButton1
		or inp.UserInputType == Enum.UserInputType.Touch then
		if resizing then return end
		resizing = inp
		Refine.userSized = true
		Refine.StopTween(Window)
		maximized = false
		rStart = GetInputPos(inp)
		rStartSize = Window.AbsoluteSize
	end
end)

Refine.Connect(UserInput.InputEnded, function(inp)
	if inp==resizing then
		resizing = false
	end
end)

Refine.Connect(UserInput.InputChanged, function(inp)
	if resizing and (inp==resizing or (resizing.UserInputType==Enum.UserInputType.MouseButton1 and inp.UserInputType==Enum.UserInputType.MouseMovement)) then
		local d = GetInputPos(inp) - rStart
		local maxWidth, maxHeight
		if IS_TOUCH then
			-- When the window is off-screen, do not artificially shrink it just
			-- because its origin is negative or beyond the viewport.
			maxWidth = math.max(320, vp.X * 1.75)
			maxHeight = math.max(240, vp.Y * 1.75)
		else
			maxWidth = math.max(1, vp.X - Window.Position.X.Offset - 6)
			maxHeight = math.max(1, vp.Y - Window.Position.Y.Offset - 6)
		end
		curW = math.clamp(rStartSize.X+d.X, math.min(320,maxWidth), maxWidth)
		curH = math.clamp(rStartSize.Y+d.Y, math.min(240,maxHeight), maxHeight)
		Window.Size = UDim2.new(0, curW, 0, curH)
	end
end)

function Refine.Header()
	if not Refine.alive then return end
	NARROW=Window.AbsoluteSize.X<(IS_TOUCH and 500 or 720)
	TITLE_H=Refine.HeaderHeight(Window.AbsoluteSize.X)
	TitleBar.Size=UDim2.new(1,0,0,TITLE_H)
	Body.Position=UDim2.fromOffset(0,TITLE_H)
	Body.Size=UDim2.new(1,0,1,-TITLE_H)
	if NARROW then
		SearchFrame.Position=UDim2.fromOffset(12,IS_TOUCH and 50 or 58)
		SearchFrame.Size=UDim2.new(1,-24,0,IS_TOUCH and 30 or 34)
	else
		SearchFrame.Position=UDim2.new(1,IS_TOUCH and -192 or -240,0,IS_TOUCH and 9 or 11)
		SearchFrame.Size=UDim2.fromOffset(IS_TOUCH and 176 or 224,34)
	end
	if TitleBar:FindFirstChild("VersionLabel") then TitleBar.VersionLabel.Visible=not NARROW end
	UpdateDragRegion()
	if Refine.QueueRows then Refine.QueueRows() end
end
function Refine.FitWindow()
	if not Refine.alive then return end
	Refine.StopTween(Window)
	if Camera and Camera.ViewportSize.X>=240 and Camera.ViewportSize.Y>=180 then vp=Camera.ViewportSize end
	if IS_TOUCH and Refine.touchLandscape ~= (vp.X > vp.Y) then
		Refine.touchLandscape = vp.X > vp.Y
		if not Refine.userSized then curW, curH = Refine.DefaultBounds() end
	end
	curW=math.max(1,math.min(curW,vp.X-12))
	curH=math.max(1,math.min(curH,vp.Y-TOP_GAP-12))
	local width=maximized and vp.X-12 or curW
	TITLE_H=Refine.HeaderHeight(width)
	local height=minimized and TITLE_H or (maximized and vp.Y-TOP_GAP-12 or curH)
	Window.Size=UDim2.fromOffset(width,height)
	local px, py = Window.Position.X.Offset, Window.Position.Y.Offset
	-- Preserve the user's mobile off-screen placement across rotation/resizes,
	-- while still guaranteeing a small reachable recovery strip.
	SetWindowPositionClamped(px, py)
	Refine.Header(); Refine.ClosePopup()
	if Refine.BubbleClamp then Refine.BubbleClamp() end
end
do
	local cameraConnection
	local function observe()
		if cameraConnection then cameraConnection:Disconnect() end
		Camera=workspace.CurrentCamera
		if Camera then cameraConnection=Refine.Connect(Camera:GetPropertyChangedSignal("ViewportSize"),Refine.FitWindow) end
		Refine.FitWindow()
	end
	Refine.Connect(workspace:GetPropertyChangedSignal("CurrentCamera"),observe)
	observe()
end
Window:GetPropertyChangedSignal("AbsoluteSize"):Connect(Refine.Header)
Window:GetPropertyChangedSignal("Position"):Connect(Refine.ClosePopup)
Refine.Connect(UserInput.WindowFocusReleased, function()
	dragging,resizing=false,false
	Refine.EndSlider()
end)

local uiToggleKey = Enum.KeyCode.RightControl
local pickingHotkey = false

Refine.Connect(UserInput.InputBegan, function(inp, gpe)
	if Refine.systemMenuOpen or gpe or pickingHotkey or Refine.capturingKey or UserInput:GetFocusedTextBox() then return end
	if inp.KeyCode == uiToggleKey then
		Gui.Enabled = not Gui.Enabled
	end
end)

local OffscreenRecovery = Refine.New("TextButton", {
	Name = "LurOffscreenRecovery",
	Size = UDim2.fromOffset(46, 46),
	AnchorPoint = Vector2.new(0, 0.5),
	Position = UDim2.fromOffset(6, vp.Y * 0.5),
	BackgroundColor3 = C.Title,
	BackgroundTransparency = 0,
	Text = "",
	AutoButtonColor = false,
	Visible = false,
	ZIndex = 90,
	Parent = Gui,
})
Refine.Round(OffscreenRecovery, 12)
local recoveryStroke = Refine.New("UIStroke", { Color = C.BorderLight, Thickness = 2, Transparency = 0 }, OffscreenRecovery)
-- Text glyphs like "›"/"⌄" aren't guaranteed to exist in Roblox's built-in fonts and can
-- render as nothing, which is part of why this looked broken/invisible. Plain ASCII
-- arrows always render, so use those, bigger and bolder for contrast against the dark fill.
local recoveryLabel = Refine.New("TextLabel", {
	Size = UDim2.fromScale(1, 1),
	BackgroundTransparency = 1,
	Text = "LUR",
	TextColor3 = C.White,
	Font = Enum.Font.GothamBlack,
	TextSize = 20,
	ZIndex = 91,
}, OffscreenRecovery)

local recoveryMode = nil
local recoveryDrag = false
local recoveryStart = nil
local recoveryOrigin = nil

-- Gentle attention pulse so the handle is easy to spot the moment the window
-- scrolls off-screen, instead of silently sitting there looking inert.
local recoveryScale = Refine.New("UIScale", { Scale = 1 }, OffscreenRecovery)
local recoveryPulseTween = nil
local function SetRecoveryPulse(on)
	if on then
		if recoveryPulseTween or Refine.reducedMotion then return end
		recoveryPulseTween = TweenService:Create(recoveryScale, TweenInfo.new(0.7, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), { Scale = 1.08 })
		recoveryPulseTween:Play()
	elseif recoveryPulseTween then
		recoveryPulseTween:Cancel()
		recoveryPulseTween = nil
		recoveryScale.Scale = 1
	end
end
OffscreenRecovery.Destroying:Connect(function() SetRecoveryPulse(false) end)
Refine.Connect(Refine.guiService:GetPropertyChangedSignal("ReducedMotionEnabled"), function()
	if Refine.guiService.ReducedMotionEnabled then SetRecoveryPulse(false) end
end)

-- A plain tap used to do nothing at all (the handle only responded to an actual
-- drag). Tapping it now snaps the window fully back into view immediately -
-- dragging it to a precise spot still works too.
local RECOVER_MARGIN = 6
local function RecoverWindowFully()
	if not Refine.alive then return end
	local w, h = math.max(1, Window.AbsoluteSize.X), math.max(1, Window.AbsoluteSize.Y)
	local minX, maxX = RECOVER_MARGIN, math.max(RECOVER_MARGIN, vp.X - w - RECOVER_MARGIN)
	local minY, maxY = TOP_GAP + RECOVER_MARGIN, math.max(TOP_GAP + RECOVER_MARGIN, vp.Y - h - RECOVER_MARGIN)
	local targetX = math.clamp(Window.Position.X.Offset, minX, maxX)
	local targetY = math.clamp(Window.Position.Y.Offset, minY, maxY)
	Refine.StopTween(Window)
	Refine.Tween(Window, EASE, { Position = UDim2.fromOffset(targetX, targetY) }):Play()
end

local function UpdateRecoveryHandle()
	if not Refine.alive or maximized or not Window.Parent then
		OffscreenRecovery.Visible = false
		SetRecoveryPulse(false)
		return
	end
	local x, y = Window.Position.X.Offset, Window.Position.Y.Offset
	local w, h = Window.AbsoluteSize.X, Window.AbsoluteSize.Y
	local offLeft = x < 0
	local offRight = x + w > vp.X
	local offTop = y < TOP_GAP
	local offBottom = y + h > vp.Y
	if not (offLeft or offRight or offTop or offBottom) then
		OffscreenRecovery.Visible = false
		SetRecoveryPulse(false)
		return
	end

	OffscreenRecovery.Visible = true
	SetRecoveryPulse(true)
	-- OffscreenRecovery uses AnchorPoint (0, 0.5): X is a left-edge offset, but
	-- Y is the button's vertical CENTER. The top/bottom branches below must add
	-- back half the button's height (23 = 46 / 2) or the handle ends up shifted
	-- up by a whole half-height, clipping into the top inset / looking cut off.
	local RECOVERY_HALF = 23
	local px, py = 8, math.clamp(y + 32, 8, math.max(8, vp.Y - 54))
	if offLeft then
		recoveryMode = "left"
		px = 6
	elseif offRight then
		recoveryMode = "right"
		px = math.max(6, vp.X - 52)
	elseif offTop then
		recoveryMode = "top"
		px = math.clamp(x + w * 0.5 - 23, 6, math.max(6, vp.X - 52))
		py = TOP_GAP + 6 + RECOVERY_HALF
	else
		recoveryMode = "bottom"
		px = math.clamp(x + w * 0.5 - 23, 6, math.max(6, vp.X - 52))
		py = math.max(TOP_GAP + 6 + RECOVERY_HALF, vp.Y - 6 - RECOVERY_HALF)
	end
	OffscreenRecovery.Position = UDim2.fromOffset(px, py)
	recoveryLabel.Text = recoveryMode == "left" and ">" or recoveryMode == "right" and "<" or recoveryMode == "top" and "v" or "^"
	recoveryStroke.Color = theme and theme.Accent or C.BorderLight
end

OffscreenRecovery.MouseEnter:Connect(function() recoveryStroke.Thickness = 3 end)
OffscreenRecovery.MouseLeave:Connect(function() recoveryStroke.Thickness = 2 end)

OffscreenRecovery.InputBegan:Connect(function(inp)
	if inp.UserInputType ~= Enum.UserInputType.MouseButton1 and inp.UserInputType ~= Enum.UserInputType.Touch then return end
	recoveryDrag = true
	recoveryStart = GetInputPos(inp)
	recoveryOrigin = Vector2.new(Window.Position.X.Offset, Window.Position.Y.Offset)
	SetRecoveryPulse(false)
	Refine.StopTween(Window)
end)

Refine.Connect(UserInput.InputChanged, function(inp)
	if not recoveryDrag then return end
	if inp.UserInputType == Enum.UserInputType.MouseMovement or inp.UserInputType == Enum.UserInputType.Touch then
		local m = GetInputPos(inp)
		local d = m - recoveryStart
		SetWindowPositionClamped(recoveryOrigin.X + d.X, recoveryOrigin.Y + d.Y)
		UpdateRecoveryHandle()
	end
end)

Refine.Connect(UserInput.InputEnded, function(inp)
	if inp.UserInputType == Enum.UserInputType.MouseButton1 or inp.UserInputType == Enum.UserInputType.Touch then
		if recoveryDrag then
			local moved = recoveryStart and (GetInputPos(inp) - recoveryStart).Magnitude > 6
			recoveryDrag = false
			if not moved then
				RecoverWindowFully()
			else
				UpdateRecoveryHandle()
			end
		end
	end
end)

Window:GetPropertyChangedSignal("Position"):Connect(UpdateRecoveryHandle)
Window:GetPropertyChangedSignal("AbsoluteSize"):Connect(UpdateRecoveryHandle)

local ToastHolder = Refine.New("Frame", {
	Name = "LurNotifications",
	AnchorPoint = Vector2.new(1, 0),
	Position = UDim2.new(1, -16, 0, TOP_GAP + 12),
	Size = UDim2.new(0, 270, 1, -90),
	BackgroundTransparency = 1,
	ClipsDescendants = true,
	ZIndex = 70,
	Parent = Gui,
})

Refine.New("UIListLayout", {
	Padding = UDim.new(0, 8),
	SortOrder = Enum.SortOrder.LayoutOrder,
}, ToastHolder)

theme = {
	Accent = Color3.fromRGB(16, 185, 129),
	On = Color3.fromRGB(7, 122, 88),
}

local currentThemeName = "Emerald"
local themeHooks = {}
local liveDefaults = {}

-- Recovery handle theme binding is registered after HookTheme exists.
local function HookTheme(fn, owner)
	themeHooks[fn] = true
	if owner then owner.Destroying:Connect(function() themeHooks[fn] = nil end) end
	fn(theme)
end

HookTheme(function(t)
	if recoveryStroke and recoveryStroke.Parent then recoveryStroke.Color = t.Accent end
	if OffscreenRecovery and OffscreenRecovery.Parent then OffscreenRecovery.BackgroundColor3 = MixColor(C.Title, t.Accent, 0.22) end
end, OffscreenRecovery)

UpdateRecoveryHandle()

if ParentGui then
	local BubbleGui = Refine.New("ScreenGui", {
		Name = "LurBubble",
		ResetOnSpawn = false,
		IgnoreGuiInset = true,
		ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
		DisplayOrder = 998,
		Parent = ParentGui,
	})

	Refine.bubbleGui = BubbleGui
	BubbleGui.Enabled=not Refine.systemMenuOpen and (IS_TOUCH or not Gui.Enabled)
	Gui.Destroying:Connect(function() BubbleGui:Destroy() end)
	Gui:GetPropertyChangedSignal("Enabled"):Connect(function() BubbleGui.Enabled=not Refine.systemMenuOpen and (IS_TOUCH or not Gui.Enabled) end)
	local bubbleBtn = Refine.New("TextButton", {
		Name = "LurLauncher",
		Size = UDim2.new(0, 44, 0, 44),
		Position = UDim2.new(1, -60, 1, -120),
		BackgroundColor3 = Color3.fromRGB(25, 28, 34),
		Text = "",
		AutoButtonColor = false,
		ZIndex = 50,
	}, BubbleGui)

	Refine.Round(bubbleBtn, 12)
	Refine.Shade(bubbleBtn)
	Refine.AddTactileFeedback(bubbleBtn)

	local launcherStroke = Refine.New("UIStroke", {
		Color = theme.Accent,
		Thickness = 1.4,
		Transparency = 0.12,
	}, bubbleBtn)

	Refine.New("TextLabel", {
		Name = "LauncherWordmark", Size = UDim2.new(1, 0, 0, 20), Position = UDim2.fromOffset(0, 11),
		BackgroundTransparency = 1, Text = "LUR", Font = Enum.Font.GothamBlack,
		TextSize = 13, TextColor3 = C.White, ZIndex = 51,
	}, bubbleBtn)
	local launcherLine = Refine.New("Frame", {
		AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 34), Size = UDim2.fromOffset(16, 2),
		BackgroundColor3 = theme.Accent, BorderSizePixel = 0, ZIndex = 51,
	}, bubbleBtn)
	Refine.Round(launcherLine, 1)
	local launcherCorner = Refine.New("Frame", {
		Position = UDim2.fromOffset(7, 7), Size = UDim2.fromOffset(4, 2), Rotation = -35,
		BackgroundColor3 = theme.Accent, BorderSizePixel = 0, ZIndex = 51,
	}, bubbleBtn)
	Gui:GetPropertyChangedSignal("Enabled"):Connect(function()
		launcherLine.BackgroundTransparency = Gui.Enabled and 0 or 0.55
	end)

	HookTheme(function(t)
		launcherStroke.Color = t.Accent
		launcherLine.BackgroundColor3 = t.Accent
		launcherCorner.BackgroundColor3 = t.Accent
	end, bubbleBtn)

	Refine.BubbleClamp=function()
		local pos=bubbleBtn.AbsolutePosition
		bubbleBtn.Position=UDim2.fromOffset(math.clamp(pos.X,8,math.max(8,vp.X-52)),math.clamp(pos.Y,TOP_GAP+8,math.max(TOP_GAP+8,vp.Y-52)))
	end
	local bDrag, bStart, bOrigin, bMoved = false, nil, nil, false
	Refine.StopBubbleDrag = function() bDrag, bMoved = false, false end
	Refine.Connect(UserInput.WindowFocusReleased, Refine.StopBubbleDrag)

	bubbleBtn.InputBegan:Connect(function(inp)
		if inp.UserInputType == Enum.UserInputType.Touch
			or inp.UserInputType == Enum.UserInputType.MouseButton1 then
			if bDrag then return end
			bDrag = inp
			bMoved = false
			bStart = GetInputPos(inp)
			bOrigin = bubbleBtn.AbsolutePosition
		end
	end)

	Refine.Connect(UserInput.InputChanged, function(inp)
		if bDrag and (inp==bDrag or (bDrag.UserInputType==Enum.UserInputType.MouseButton1 and inp.UserInputType==Enum.UserInputType.MouseMovement)) then
			local d = GetInputPos(inp) - bStart

			if d.Magnitude > 8 then
				bMoved = true
			end

			bubbleBtn.Position = UDim2.fromOffset(
				math.clamp(bOrigin.X + d.X, 8, math.max(8, vp.X - 52)),
				math.clamp(bOrigin.Y + d.Y, TOP_GAP + 8, math.max(TOP_GAP + 8, vp.Y - 52))
			)
		end
	end)

	Refine.Connect(UserInput.InputEnded, function(inp)
		if inp==bDrag then
			bDrag = false

		end
	end)
	Refine.Click(bubbleBtn, function()
		if Refine.navigation or not bMoved then Gui.Enabled = not Gui.Enabled end
	end)
	Refine.bubbleButton = bubbleBtn
end

local toastOrder = 0

local function Notify(title, desc, duration, level)
	if not Refine.alive then return end
	title, desc = tostring(title or "Lur UI"), tostring(desc or "")
	duration = math.clamp(tonumber(duration) or 3.2, 0.25, 20)

	local barColor = theme.Accent

	if level == "success" then
		barColor = C.Success
	elseif level == "warning" then
		barColor = C.Warning
	elseif level == "error" then
		barColor = C.Danger
	end

	local existing = {}
	for _, k in ipairs(ToastHolder:GetChildren()) do
		if k:IsA("Frame") then
			table.insert(existing, k)
		end
	end

	if #existing >= 4 then
		table.sort(existing, function(a, b)
			return (a.LayoutOrder or 0) < (b.LayoutOrder or 0)
		end)
		existing[1]:Destroy()
	end

	toastOrder = toastOrder + 1

	local t = Refine.New("Frame", {
		Name = "LurNotification",
		Size = UDim2.new(0, 0, 0, 72),
		BackgroundColor3 = C.Card,
		ClipsDescendants = true,
		ZIndex = 71,
		LayoutOrder = toastOrder,
	}, ToastHolder)
	Refine.Round(t, 10)
	Refine.New("UIStroke", { Color = C.Border, Thickness = 1 }, t)

	local bar = Refine.New("Frame", {
		Size = UDim2.new(0, 4, 1, 0),
		BackgroundColor3 = barColor,
		BorderSizePixel = 0,
		ZIndex = 72,
	}, t)
	Refine.Shade(bar)

	Refine.New("TextLabel", {
		Position = UDim2.new(0, 14, 0, 8),
		Size = UDim2.new(1, -46, 0, 18),
		BackgroundTransparency = 1,
		Text = title,
		TextColor3 = C.White,
		Font = Enum.Font.GothamSemibold,
		TextSize = 13,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextTruncate = Enum.TextTruncate.AtEnd,
		ZIndex = 72,
	}, t)

	Refine.New("TextLabel", {
		Position = UDim2.new(0, 14, 0, 28),
		Size = UDim2.new(1, -30, 0, 32),
		BackgroundTransparency = 1,
		Text = desc,
		TextColor3 = C.Gray,
		Font = Enum.Font.Gotham,
		TextSize = 11,
		TextWrapped = true,
		TextYAlignment = Enum.TextYAlignment.Top,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextTruncate = Enum.TextTruncate.AtEnd,
		ZIndex = 72,
	}, t)

	local prog = Refine.New("Frame", {
		Position = UDim2.new(0, 0, 1, -2),
		Size = UDim2.new(1, 0, 0, 2),
		BackgroundColor3 = barColor,
		BorderSizePixel = 0,
		ZIndex = 73,
	}, t)

	local dismiss = Refine.New("TextButton", {
		Name = "Dismiss", AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, 0, 0, 0),
		Size = UDim2.fromOffset(32, 32), BackgroundTransparency = 1, Text = "×",
		Font = Enum.Font.Gotham, TextSize = 17, TextColor3 = C.Gray, AutoButtonColor = false, ZIndex = 74,
	}, t)
	Refine.Click(dismiss, function() t:Destroy() end)
	Refine.Tween(t, EASE, { Size = UDim2.new(1, 0, 0, 72) }):Play()
	Refine.Tween(
		prog,
		TweenInfo.new(duration, Enum.EasingStyle.Linear),
		{ Size = UDim2.new(0, 0, 0, 2) }
	):Play()

	local timeout
	timeout = task.delay(duration, function()
		timeout = nil
		if t and t.Parent then
			Refine.Tween(t, EASE, { Size = UDim2.new(0, 0, 0, 72) }):Play()
			task.wait(0.2)
			t:Destroy()
		end
	end)
	t.Destroying:Connect(function()
		if timeout and coroutine.status(timeout) ~= "dead" then task.cancel(timeout) end
		timeout = nil
	end)
end

Refine.ReportError = function(message)
	Notify("Action unavailable", "This UI action could not finish. Details are in the Roblox Output.", 4, "error")
end

do
	local function fitNotifications()
		ToastHolder.Size = UDim2.new(0, math.min(270, math.max(180, vp.X - 32)), 1, -TOP_GAP - 28)
	end
	Window:GetPropertyChangedSignal("AbsoluteSize"):Connect(fitNotifications)
	fitNotifications()
end

local logBuffer, consoleFrames = {}, {}

local LOG_COLORS = {
	INFO = "#6E6E73",
	OK = "#22C55E",
	WARN = "#EAB308",
	ERR = "#EF4444",
}

function Refine.Safe(s)
	return tostring(s)
		:gsub("&", "&amp;")
		:gsub("<", "&lt;")
		:gsub(">", "&gt;")
end

local function Log(msg, level)
	local ts = os.date("%H:%M:%S")
	local tag = level or "INFO"
	local color = LOG_COLORS[tag] or LOG_COLORS.INFO

	local formatted = string.format(
		'<font color="%s">[%s] [%s]</font> %s',
		color,
		ts,
		tag,
		Refine.Safe(msg)
	)

	table.insert(logBuffer, { ts, formatted })
	if #logBuffer > 150 then
		table.remove(logBuffer, 1)
	end

	for i = #consoleFrames, 1, -1 do
		local cf = consoleFrames[i]

		if cf.sc and cf.sc.Parent then
			pcall(cf.append, formatted)
		else
			table.remove(consoleFrames, i)
		end
	end
end

local THEMES = {
	Crimson = { Accent = Color3.fromRGB(214, 51, 82), On = Color3.fromRGB(190, 35, 65) },
	Ocean = { Accent = Color3.fromRGB(14, 165, 233), On = Color3.fromRGB(3, 105, 161) },
	Violet = { Accent = Color3.fromRGB(139, 92, 246), On = Color3.fromRGB(109, 40, 217) },
	Emerald = { Accent = Color3.fromRGB(16, 185, 129), On = Color3.fromRGB(7, 122, 88) },
	Sunset = { Accent = Color3.fromRGB(249, 115, 22), On = Color3.fromRGB(154, 69, 12) },
	Midnight = { Accent = Color3.fromRGB(99, 102, 241), On = Color3.fromRGB(67, 56, 202) },
	Cyberpunk = { Accent = Color3.fromRGB(244, 63, 94), On = Color3.fromRGB(190, 18, 60) },
	Sakura = { Accent = Color3.fromRGB(244, 114, 182), On = Color3.fromRGB(157, 23, 77) },
}

local currentActiveMenu = "Lobby"
local hoveredMenu = nil
local menuRegistry = {}
local sectionRegistry = {}

function Refine.UpdateMenuVisuals()
	for _, e in ipairs(menuRegistry) do
		local isActive = (e.name == currentActiveMenu)
		local isHovered = (e.name == hoveredMenu and not isActive)

		if isActive then
			e.sel.BackgroundTransparency = 0.82
			e.hover.BackgroundTransparency = 1

			if e.label then
				e.label.TextColor3 = C.White
				e.label.Font = Enum.Font.GothamBold
			end

			if e.paint then e.paint(C.White) end

			if e.glow then
				e.glow.BackgroundColor3 = theme.Accent
				e.glow.BackgroundTransparency = 0
				e.glow.Size = UDim2.new(0, 3, 0, 20)
			end
		elseif isHovered then
			e.sel.BackgroundTransparency = 1
			e.hover.BackgroundTransparency = 0.93

			if e.label then
				e.label.TextColor3 = C.White
				e.label.Font = Enum.Font.GothamMedium
			end

			if e.paint then e.paint(C.White) end

			if e.glow then
				e.glow.BackgroundTransparency = 1
				e.glow.Size = UDim2.new(0, 3, 0, 0)
			end
		else
			e.sel.BackgroundTransparency = 1
			e.hover.BackgroundTransparency = 1

			if e.label then
				e.label.TextColor3 = C.Gray
				e.label.Font = Enum.Font.GothamMedium
			end

			if e.paint then e.paint(C.Gray) end

			if e.glow then
				e.glow.BackgroundTransparency = 1
				e.glow.Size = UDim2.new(0, 3, 0, 0)
			end
		end
	end
end

local function ApplyTheme(name, silent)
	if not THEMES[name] then return false end
	theme = THEMES[name]
	currentThemeName = name
	liveDefaults["Theme Preset"] = name

	for fn in pairs(themeHooks) do
		pcall(fn, theme)
	end

	Refine.UpdateMenuVisuals()

	if not silent then
		Notify("Theme", "Applied: " .. name)
		Log("Theme changed -> " .. name, "OK")
	end
end

local Content = Refine.New("ScrollingFrame", {
	Name = "LurContent",
	Position = IS_TOUCH and UDim2.new(0, 0, 0, 0) or UDim2.new(0, SIDEBAR_W, 0, 0),
	Size = IS_TOUCH and UDim2.new(1, 0, 1, -Refine.FooterHeight) or UDim2.new(1, -SIDEBAR_W, 1, -Refine.FooterHeight),
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	ScrollBarThickness = IS_TOUCH and 6 or 5,
	ScrollBarImageColor3 = C.Gray,
	ScrollBarImageTransparency = 0.15,
	VerticalScrollBarInset = Enum.ScrollBarInset.Always,
	AutomaticCanvasSize = Enum.AutomaticSize.None,
	ClipsDescendants = true,
	CanvasSize = UDim2.new(0, 0, 0, 600),
}, Body)

Refine.contentPadding = Refine.New("UIPadding", {
	PaddingTop = UDim.new(0, IS_TOUCH and 12 or 16),
	PaddingBottom = UDim.new(0, IS_TOUCH and 24 or 30),
	PaddingLeft = UDim.new(0, 18),
	PaddingRight = UDim.new(0, 18),
}, Content)

Refine.EnableTouchPan(Content)

Refine.emptyState = Refine.New("Frame", {
	Name = "LurSearchEmpty", Position = UDim2.fromOffset(18, 24), Size = UDim2.new(1, -36, 0, 130),
	BackgroundColor3 = C.Card, Visible = false,
}, Content)
Refine.Round(Refine.emptyState, 10)
Refine.New("UIStroke", {Color = C.Border, Thickness = 1}, Refine.emptyState)
Refine.New("TextLabel", {
	Position = UDim2.fromOffset(18, 20), Size = UDim2.new(1, -36, 0, 24), BackgroundTransparency = 1,
	Text = "No matching controls", TextColor3 = C.White, Font = Enum.Font.GothamSemibold, TextSize = 16,
	TextXAlignment = Enum.TextXAlignment.Left,
}, Refine.emptyState)
Refine.New("TextLabel", {
	Position = UDim2.fromOffset(18, 49), Size = UDim2.new(1, -36, 0, 32), BackgroundTransparency = 1,
	Text = "Try another keyword or choose a menu with results.", TextColor3 = C.Gray,
	Font = Enum.Font.Gotham, TextSize = 12, TextWrapped = true, TextXAlignment = Enum.TextXAlignment.Left,
}, Refine.emptyState)
do
	local clear = Refine.New("TextButton", {
		Position = UDim2.fromOffset(18, 86), Size = UDim2.fromOffset(110, 30), BackgroundColor3 = C.Search,
		Text = "Clear search", TextColor3 = C.White, Font = Enum.Font.GothamMedium, TextSize = 12,
		AutoButtonColor = false,
	}, Refine.emptyState)
	Refine.Round(clear, 6)
	Refine.Click(clear, function() SearchBox.Text = "" end)
end

local PageCache = Refine.New("Folder", { Name = "PageCache" }, Gui)

local pages, allRows = {}, {}
local currentPage = nil
local BuildPage, ShowPage, Recompute
function Refine.SyncCanvas(page)
	if not Refine.alive or not page or page ~= currentPage or page.frame.Parent ~= Content then return end
	local contentHeight = page.contentHeight or page.frame.Size.Y.Offset
	local scale = math.max(0.01, intro.Scale)
	if page.layout then contentHeight = math.max(contentHeight, page.layout.AbsoluteContentSize.Y / scale) end
	local origin = page.frame.AbsolutePosition.Y
	for _,row in ipairs(allRows) do
		if row.Parent and row:IsDescendantOf(page.frame) and Refine.IsVisible(row) then
			contentHeight = math.max(contentHeight, (row.AbsolutePosition.Y + row.AbsoluteSize.Y - origin) / scale)
		end
	end
	local total = math.ceil(contentHeight + Refine.contentPadding.PaddingTop.Offset + Refine.contentPadding.PaddingBottom.Offset)
	if Refine.emptyState.Visible then total = 170 end
	page.cachedCanvas = total
	Content.CanvasSize = UDim2.fromOffset(0, total)
end
function Refine.QueueCanvas()
	if Refine.canvasPending then return end
	Refine.canvasPending = true
	task.defer(function()
		Refine.canvasPending = false
		Refine.SyncCanvas(currentPage)
	end)
end
intro:GetPropertyChangedSignal("Scale"):Connect(Refine.QueueCanvas)
function Refine.ApplyRows()
	if not Refine.alive then return end
	for index = #allRows, 1, -1 do
		if not allRows[index].Parent then table.remove(allRows, index) end
	end
	local stacked=Content.AbsoluteSize.X<(IS_TOUCH and 460 or 500)
	ROW_H=stacked and 112 or (IS_TOUCH and vp.X > vp.Y and vp.Y <= 600 and 56 or 64)
	for _,row in ipairs(allRows) do
		if row.Parent then
			row.Size=UDim2.new(1,0,0,ROW_H)
			for _,child in ipairs(row:GetChildren()) do
				if child:IsA("GuiObject") then
					local original=Refine.rowDefaults[child]
					if not original then original={position=child.Position,size=child.Size}; Refine.rowDefaults[child]=original end
					if child.Name=="Title" or child.Name=="Desc" then
						local inset=original.position.X.Offset
						child.TextTruncate=Enum.TextTruncate.AtEnd
						child.TextWrapped=stacked and child.Name=="Desc"
						child.Size=UDim2.new(1,stacked and -inset-20 or -inset-204,0,stacked and child.Name=="Desc" and 28 or original.size.Y.Offset)
					elseif original.position.Y.Scale==0.5 then
						child.Position=stacked and UDim2.new(original.position.X.Scale,original.position.X.Offset,1,original.position.Y.Offset-26) or original.position
					end
				end
			end
		end
	end
	if Recompute then for _,page in pairs(pages) do page.dirty=true; Recompute(page) end end
end
function Refine.QueueRows()
	if Refine.rowsPending then return end
	Refine.rowsPending=true
	task.defer(function() Refine.rowsPending=false; Refine.ApplyRows() end)
end
Content:GetPropertyChangedSignal("AbsoluteSize"):Connect(Refine.QueueRows)
Content:GetPropertyChangedSignal("AbsoluteWindowSize"):Connect(Refine.QueueCanvas)
Content:GetPropertyChangedSignal("CanvasPosition"):Connect(Refine.ClosePopup)

local Backdrop = Refine.New("TextButton", {
	Size = UDim2.new(1, 0, 1, 0),
	BackgroundColor3 = Color3.fromRGB(0, 0, 0),
	BackgroundTransparency = 1,
	AutoButtonColor = false,
	Text = "",
	Visible = false,
	ZIndex = 5,
}, Body)

-- Mobile always uses an overlay drawer so the navigation never steals width from page content.
-- Desktop keeps the split sidebar only when the window is wide enough.
function Refine.UseSidebarOverlay()
	return IS_TOUCH or Window.AbsoluteSize.X < 760
end

Refine.overlay=Refine.UseSidebarOverlay()
local sidebarOpen = not Refine.overlay

local Sidebar = Refine.New("Frame", {
	Position = IS_TOUCH and UDim2.new(0, -SIDEBAR_W, 0, 0) or UDim2.new(0, 0, 0, 0),
	Size = UDim2.new(0, SIDEBAR_W, 1, -Refine.FooterHeight),
	BackgroundColor3 = C.Sidebar,
	ZIndex = 6,
	ClipsDescendants = true,
}, Body)
Refine.sidebar = Sidebar
Refine.sidebarVisible = sidebarOpen

Refine.New("Frame", {
	Position = UDim2.new(1, 0, 0, 0),
	Size = UDim2.new(0, 1, 1, 0),
	BackgroundColor3 = C.Border,
	BorderSizePixel = 0,
}, Sidebar)

local SideScroll = Refine.New("ScrollingFrame", {
	Size = UDim2.new(1, 0, 1, 0),
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	ScrollBarThickness = IS_TOUCH and 3 or 4,
	ScrollBarImageColor3 = C.GrayDk,
	CanvasSize = UDim2.new(0, 0, 0, 0),
	AutomaticCanvasSize = Enum.AutomaticSize.Y,
}, Sidebar)

Refine.EnableTouchPan(SideScroll)

Refine.New("UIListLayout", {
	Padding = UDim.new(0, 4),
	SortOrder = Enum.SortOrder.LayoutOrder,
	HorizontalAlignment = Enum.HorizontalAlignment.Center,
}, SideScroll)

Refine.New("UIPadding", {
	PaddingTop = UDim.new(0, 14),
	PaddingBottom = UDim.new(0, 14),
}, SideScroll)

local function SetSidebar(open, preserveMinimized)
	if minimized and not preserveMinimized then
		SetMinimized(false)
	end

	sidebarOpen = open
	Refine.sidebarVisible = open

	if Refine.overlay then
		Refine.StopTween(Content)
		Sidebar.Size=UDim2.new(0,SIDEBAR_W,1,-Refine.FooterHeight)
		Content.Position=UDim2.fromOffset(0,0)
		Content.Size=UDim2.new(1,0,1,-Refine.FooterHeight)
		Backdrop.Visible = open
		Refine.Tween(Backdrop, EASE, { BackgroundTransparency = open and 0.5 or 1 }):Play()
		Refine.Tween(Sidebar, EASE, {
			Position = open and UDim2.new(0, 0, 0, 0) or UDim2.new(0, -SIDEBAR_W, 0, 0),
		}):Play()
	else
		Backdrop.Visible=false
		Sidebar.Position=UDim2.fromOffset(0,0)
		Refine.Tween(Sidebar, EASE, {
			Size = open and UDim2.new(0, SIDEBAR_W, 1, -Refine.FooterHeight) or UDim2.new(0, 0, 1, -Refine.FooterHeight),
		}):Play()

		Refine.Tween(Content, EASE, {
			Position = open and UDim2.new(0, SIDEBAR_W, 0, 0) or UDim2.new(0, 0, 0, 0),
			Size = open and UDim2.new(1, -SIDEBAR_W, 1, -Refine.FooterHeight) or UDim2.new(1, 0, 1, -Refine.FooterHeight),
		}):Play()
	end
	if Refine.navigation and not preserveMinimized then
		task.delay(0.15, function()
			if not Refine.alive or not Gui.Enabled or sidebarOpen ~= open or Refine.popup then return end
			if open then Refine.FocusFirst(Sidebar) else Refine.FocusFirst(Content) end
		end)
	end
end

Refine.Click(panelBtn, function()
	SetSidebar(not sidebarOpen)
end)

Refine.Click(Backdrop, function()
	if Refine.overlay and sidebarOpen then
		SetSidebar(false)
	end
end)

Window:GetPropertyChangedSignal("AbsoluteSize"):Connect(function()
	local overlay=Refine.UseSidebarOverlay()
	if overlay~=Refine.overlay then
		Refine.overlay=overlay
		SetSidebar(not overlay, true)
	elseif overlay then
		-- A phone/tablet may rotate or maximize without changing overlay mode.
		-- Keep the content at full width and let the sidebar float above it.
		Content.Position=UDim2.fromOffset(0,0)
		Content.Size=UDim2.new(1,0,1,-Refine.FooterHeight)
	end
end)
SetSidebar(sidebarOpen)

local Footer = Refine.New("Frame", {
	Name = "LurFooter",
	Position = UDim2.new(0, 0, 1, -Refine.FooterHeight),
	Size = UDim2.new(1, 0, 0, Refine.FooterHeight),
	BackgroundColor3 = C.Title,
	Parent = Body,
})
Refine.Round(Footer, 14)
Refine.New("Frame", {
	Name = "FooterCornerFill", Size = UDim2.new(1, 0, 1, -14), BackgroundColor3 = C.Title,
}, Footer)

Refine.New("Frame", {
	Size = UDim2.new(1, 0, 0, 1),
	BackgroundColor3 = C.Border,
	BorderSizePixel = 0,
}, Footer)

local fpsLab = Refine.New("TextLabel", {
	Position = UDim2.new(0, 12, 0, 0),
	Size = UDim2.new(0, 90, 1, 0),
	BackgroundTransparency = 1,
	Text = "FPS --",
	TextColor3 = C.Success,
	Font = Enum.Font.GothamBold,
	TextSize = 11,
	TextXAlignment = Enum.TextXAlignment.Left,
}, Footer)

local pingLab = Refine.New("TextLabel", {
	Position = UDim2.new(0, 110, 0, 0),
	Size = UDim2.new(0, 110, 1, 0),
	BackgroundTransparency = 1,
	Text = "Ping --",
	TextColor3 = C.Success,
	Font = Enum.Font.GothamBold,
	TextSize = 11,
	TextXAlignment = Enum.TextXAlignment.Left,
}, Footer)

if not NARROW then
	Refine.New("TextLabel", {
		Name = "FooterVersion",
		AnchorPoint = Vector2.new(0.5, 0),
		Position = UDim2.new(0.5, 0, 0, 0),
		Size = UDim2.new(0, 140, 1, 0),
		BackgroundTransparency = 1,
		Text = "Lur UI v" .. VERSION,
		TextColor3 = C.GrayDk,
		Font = Enum.Font.Gotham,
		TextSize = 11,
		TextXAlignment = Enum.TextXAlignment.Center,
	}, Footer)
end

local plLab = Refine.New("TextLabel", {
	AnchorPoint = Vector2.new(1, 0),
	Position = UDim2.new(1, -12, 0, 0),
	Size = UDim2.new(0, 110, 1, 0),
	BackgroundTransparency = 1,
	Text = "Players --",
	TextColor3 = C.Gray,
	Font = Enum.Font.Gotham,
	TextSize = 11,
	TextXAlignment = Enum.TextXAlignment.Right,
}, Footer)

do
	local function fitFooter()
		local width = Window.AbsoluteSize.X
		local version = Footer:FindFirstChild("FooterVersion")
		if version then version.Visible = width >= 660 end
		pingLab.Visible = width >= 360
		plLab.Size = UDim2.fromOffset(math.min(110, width * 0.31), Refine.FooterHeight)
	end
	Window:GetPropertyChangedSignal("AbsoluteSize"):Connect(fitFooter)
	fitFooter()
end

local frames = 0
local fpsWindowStart = os.clock()
Refine.Connect(RunService.RenderStepped, function()
	frames += 1
end)

task.spawn(function()
	while Refine.alive do
		task.wait(1)
		if not Refine.alive then break end

		local now = os.clock()
		local elapsed = math.max(0.001, now - fpsWindowStart)
		local fps = math.floor(frames / elapsed + 0.5)
		fpsWindowStart = now
		frames = 0
		fpsLab.Text = "FPS " .. fps
		fpsLab.TextColor3 = (fps >= 50 and C.Success) or (fps >= 30 and C.Warning) or C.Danger

		local pingOk, pingVal = pcall(function()
			return math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue() + 0.5)
		end)
		if pingOk then
			pingLab.Text = "Ping " .. pingVal .. "ms"
			pingLab.TextColor3 = (pingVal <= 80 and C.Success) or (pingVal <= 160 and C.Warning) or C.Danger
		else
			pingLab.Text = "Ping --"
			pingLab.TextColor3 = C.GrayDk
		end

		plLab.Text = "Players " .. #Players:GetPlayers()
	end
end)

local configRegistry, binds, bindLabels, toggleByTitle, hooks = {}, {}, {}, {}, {}
local pendingValues = {}
local savedBounds = nil

local configSession = { profiles = {}, exported = "", esp = false, spectating = nil }

local activeProfile = "Default"
local availableProfiles = { "Default", "AFK_Farming", "PVP_Mode", "Boss_Raid" }
local autoSaveConfig = true

function Refine.SanitizeProfileName(s)
	s = tostring(s or "")
	s = s:gsub("[%c/\\:*?\"<>|]", "")
	s = s:gsub("^%s+", "")
	s = s:gsub("%s+$", "")
	s = s:gsub("%s+", "_")
	if s == "" then return nil end
	local length = utf8.len(s)
	if not length then return nil end
	if length > 32 then s = s:sub(1, utf8.offset(s, 33) - 1) end
	return s
end

function Refine.FireHook(t, v)
	if hooks[t] then
		Refine.SafeCall("Hook:" .. tostring(t), hooks[t], v)
	end
end

Refine.configSchema = {}

function Refine.IndexConfigSchema(pagesData)
	local function indexRows(rows)
		for _, spec in ipairs(rows or {}) do
			if spec.kind == "accordion" then
				indexRows(spec.rows)
			elseif spec.kind == "toggle" or spec.kind == "slider" or spec.kind == "input"
				or spec.kind == "drop" or spec.kind == "multidrop" then
				Refine.configSchema[spec.title] = { kind = spec.kind, spec = spec }
			end
		end
	end
	for _, blocks in pairs(pagesData) do
		for _, block in ipairs(blocks) do indexRows(block.rows) end
	end
end

function Refine.ConfigValue(kind, value, spec)
	if kind == "toggle" then
		if type(value) == "boolean" then return value, true end
	elseif kind == "slider" then
		if type(value) == "number" and value == value and math.abs(value) < math.huge then
			return math.clamp(math.floor(value + 0.5), spec.min or 0, spec.max or 100), true
		end
	elseif kind == "input" then
		if type(value) == "string" and #value <= 4096 then return value, true end
	elseif kind == "drop" then
		if type(value) == "string" and table.find(spec.options or {}, value) then return value, true end
	elseif kind == "multidrop" and type(value) == "table" then
		local selected = {}
		for key, choice in pairs(value) do
			local option = type(key) == "number" and choice or key
			if type(option) ~= "string" or (type(key) ~= "number" and choice ~= true)
				or not table.find(spec.options or {}, option) then return nil, false end
			selected[option] = true
		end
		return selected, true
	end
	return nil, false
end

function Refine.ConfigKey(name)
	if type(name) ~= "string" or #name > 32 or name == "Unknown"
		or name:match("^Button") or name:match("^DPad") or name:match("^Thumbstick") then return nil end
	local ok, key = pcall(function() return Enum.KeyCode[name] end)
	if ok and key and key ~= Enum.KeyCode.Unknown then return key end
	return nil
end

function Refine.RegConfig(key, kind, get, set, def, spec)
	spec = spec or {}
	Refine.configSchema[key] = { kind = kind, spec = spec }
	local restore = pendingValues[key]
	for i = #configRegistry, 1, -1 do
		if configRegistry[i].key == key then
			-- Pending imports are consumed once. Rebuilds use the latest live value.
			local ok, value = pcall(configRegistry[i].get)
			if ok then restore = value end
			table.remove(configRegistry, i)
			break
		end
	end
	if key == "Active Profile" then restore = activeProfile end
	if key == "Theme Preset" then restore = currentThemeName end
	pendingValues[key] = nil
	table.insert(configRegistry, { key = key, kind = kind, get = get, set = set, default = def })
	if restore ~= nil then
		local value, valid = Refine.ConfigValue(kind, restore, spec)
		if valid then pcall(set, value) end
	end
end

function Refine.SerializeConfigData()
	local data = {
		profile = activeProfile,
		placeId = game.PlaceId,
		theme = currentThemeName,
		binds = table.clone(binds),
		hotkey = uiToggleKey.Name,
		values = {},
		savedAt = os.date("%Y-%m-%d %H:%M:%S"),
		window = {
			x = (maximized and savedPos or Window.Position).X.Offset,
			y = (maximized and savedPos or Window.Position).Y.Offset,
			w = curW,
			h = curH,
		},
	}

	for key, value in pairs(pendingValues) do
		data.values[key] = type(value) == "table" and table.clone(value) or value
	end
	for _, e in ipairs(configRegistry) do
		local ok, value = pcall(e.get)
		if ok then
			data.values[e.key] = type(value) == "table" and table.clone(value) or value
		else
		warn("[LurUI] Failed to read setting: " .. tostring(e.key))
		end
	end
	data.values["Active Profile"] = activeProfile
	data.values["Theme Preset"] = currentThemeName
	-- Draft fields are not settings; exporting pasted JSON must not recursively embed it.
	data.values["Paste JSON String"] = nil
	data.values["New Profile Name"] = nil

	return data
end

function Refine.ValidateConfigData(data)
	if type(data) ~= "table" then return nil, "Configuration must be an object" end
	if data.values == nil and data.theme == nil and data.binds == nil and data.hotkey == nil and data.window == nil then
		return nil, "No configuration settings found"
	end
	local clean = {}
	if data.profile ~= nil then
		if type(data.profile) ~= "string" then return nil, "Invalid profile name" end
		clean.profile = Refine.SanitizeProfileName(data.profile)
		if not clean.profile then return nil, "Invalid profile name" end
	end
	if data.theme ~= nil then
		if type(data.theme) ~= "string" or not THEMES[data.theme] then return nil, "Unknown theme" end
		clean.theme = data.theme
	end
	if data.values ~= nil then
		if type(data.values) ~= "table" then return nil, "Settings must be an object" end
		clean.values = {}
		local count = 0
		for key, value in pairs(data.values) do
			count += 1
			if count > 128 or type(key) ~= "string" or #key > 128 then return nil, "Invalid settings list" end
			if key ~= "Active Profile" and key ~= "New Profile Name" and key ~= "Paste JSON String" then
				local schema = Refine.configSchema[key]
				if not schema then return nil, "Unknown setting: " .. key end
				local checked, valid = Refine.ConfigValue(schema.kind, value, schema.spec)
				if not valid then return nil, "Invalid value for " .. key end
				clean.values[key] = checked
			end
		end
	end
	if not clean.theme and clean.values and clean.values["Theme Preset"] then
		if not THEMES[clean.values["Theme Preset"]] then return nil, "Unknown theme" end
		clean.theme = clean.values["Theme Preset"]
	end
	if data.binds ~= nil then
		if type(data.binds) ~= "table" then return nil, "Keybinds must be an object" end
		clean.binds = {}
		local used, count = {}, 0
		for title, name in pairs(data.binds) do
			count += 1
			if count > 128 or type(title) ~= "string" or #title > 128 or not Refine.ConfigKey(name)
				or used[name] then return nil, "Invalid or duplicate keybind" end
			local schema = Refine.configSchema[title]
			if not schema or schema.kind ~= "toggle" then return nil, "Keybind requires a known toggle" end
			used[name] = true
			clean.binds[title] = name
		end
	end
	if data.hotkey ~= nil then
		if not Refine.ConfigKey(data.hotkey) then return nil, "Invalid UI hotkey" end
		if clean.binds then
			for title, keyName in pairs(clean.binds) do
				if keyName == data.hotkey then
					return nil, "UI hotkey conflicts with keybind: " .. title
				end
			end
		end
		clean.hotkey = data.hotkey
	end
	if data.window ~= nil then
		if type(data.window) ~= "table" then return nil, "Invalid window bounds" end
		clean.window = {}
		for _, key in ipairs({ "x", "y", "w", "h" }) do
			local value = data.window[key]
			if type(value) ~= "number" or value ~= value or math.abs(value) == math.huge
				or (key == "w" or key == "h") and value <= 0 then return nil, "Invalid window bounds" end
			clean.window[key] = value
		end
	end
	return clean
end

function Refine.ApplyData(data, silent)
	local clean, reason = Refine.ValidateConfigData(data)
	if not clean then
		if not silent then Notify("Config", reason, nil, "error") end
		return false, reason
	end
	data = clean
	Refine.EndSlider()
	Refine.ClosePopup()
	if Refine.CancelKeyCapture then Refine.CancelKeyCapture() end

	if type(data.profile) == "string" then
		local pname = Refine.SanitizeProfileName(data.profile)
		if pname then
			activeProfile = pname
			liveDefaults["Active Profile"] = pname
			if not table.find(availableProfiles, activeProfile) then
				table.insert(availableProfiles, activeProfile)
			end
		end
	end

	if type(data.values) == "table" then
		pendingValues = table.clone(data.values)

		for _, e in ipairs(configRegistry) do
			local v = data.values[e.key]
			if v ~= nil then
				pcall(e.set, v)
				pendingValues[e.key] = nil
			end
		end
	end

	if type(data.binds) == "table" then
		binds = table.clone(data.binds)
		for title, update in pairs(bindLabels) do
			update(binds[title] or "")
		end
	end

	if type(data.hotkey) == "string" then
		uiToggleKey = Refine.ConfigKey(data.hotkey)
		if Refine.SyncHotkey then Refine.SyncHotkey() end
	end

	if type(data.window) == "table" then
		savedBounds = table.clone(data.window)
		Refine.userSized = true
		minimized, maximized = false, false
		Body.Visible, ResizeGrip.Visible = true, true
		curW = math.clamp(savedBounds.w, math.min(320, vp.X - 12), vp.X - 12)
		curH = math.clamp(savedBounds.h, math.min(240, vp.Y - TOP_GAP - 12), vp.Y - TOP_GAP - 12)
		Window.Position = UDim2.fromOffset(savedBounds.x, savedBounds.y)
		Refine.FitWindow()
		savedPos = Window.Position
	end

	if type(data.theme) == "string" then
		ApplyTheme(data.theme, true)
	end
	for _, entry in ipairs(configRegistry) do
		if entry.key == "Active Profile" then entry.set(activeProfile)
		elseif entry.key == "Theme Preset" then entry.set(currentThemeName) end
	end

	if not silent then
		Notify("Profile Loaded", "Profile: " .. (data.profile or activeProfile))
		Log("Loaded profile -> " .. (data.profile or activeProfile), "OK")
	end
	return true
end

local saveDebounce = nil

local function SaveConfig(silent)
	local data = Refine.SerializeConfigData()
	local ok, encoded = pcall(function() return Http:JSONEncode(data) end)
	if not ok then
		Notify("Config", "Could not encode configuration", nil, "error")
		return false
	end
	configSession.profiles[activeProfile] = encoded
	if not silent then
		Notify("Config Saved", "Session profile: " .. activeProfile, nil, "success")
		Log("Saved session profile -> " .. activeProfile, "OK")
	end
	return true
end

local function TriggerAutoSave()
	if not autoSaveConfig then return end

	if saveDebounce and coroutine.status(saveDebounce)~="dead" then
		task.cancel(saveDebounce)
	end

	saveDebounce = task.delay(0.5, function()
		saveDebounce=nil
		if Refine.alive then SaveConfig(true) end
	end)
end

local function LoadConfig(profileName, silent)
	profileName = profileName or activeProfile
	local encoded = configSession.profiles[profileName]
	if not encoded then
		if not silent then Notify("Config", "No saved session profile: " .. profileName, nil, "warning") end
		return false
	end
	local ok, data = pcall(function() return Http:JSONDecode(encoded) end)
	if ok and type(data) == "table" then
		return Refine.ApplyData(data, silent)
	end
	if not silent then
		Notify("Config", "Saved profile is invalid or unreadable", nil, "error")
	end
	return false
end

local function ExportConfigToClipboard()
	Refine.ClosePopup()
	local ok, encoded = pcall(function() return Http:JSONEncode(Refine.SerializeConfigData()) end)
	if not ok then
		Notify("Export", "Could not encode configuration", nil, "error")
		return nil
	end
	configSession.exported = encoded
	local old = Gui:FindFirstChild("LurConfigExport")
	if old then old:Destroy() end
	local overlay = Refine.New("TextButton", {
		Name = "LurConfigExport", Size = UDim2.fromScale(1, 1), Text = "",
		BackgroundColor3 = Color3.new(0, 0, 0), BackgroundTransparency = 0.35,
		AutoButtonColor = false, ZIndex = 50,
	}, Gui)
	local panel = Refine.New("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
		Size = UDim2.new(0.88, 0, 0.65, 0), BackgroundColor3 = C.Window, Active = true, ZIndex = 51,
	}, overlay)
	Refine.Round(panel, 10)
	Refine.New("UISizeConstraint", { MaxSize = Vector2.new(660, 430) }, panel)
	Refine.New("UIStroke", { Color = C.Border, Thickness = 1.2 }, panel)
	Refine.New("TextLabel", {
		Position = UDim2.fromOffset(18, 14), Size = UDim2.new(1, -112, 0, 26),
		BackgroundTransparency = 1, Text = "Configuration JSON", TextXAlignment = Enum.TextXAlignment.Left,
		Font = Enum.Font.GothamSemibold, TextSize = 16, TextColor3 = C.White, ZIndex = 52,
	}, panel)
	local close = Refine.New("TextButton", {
		AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -14, 0, 12), Size = UDim2.fromOffset(72, 32),
		BackgroundColor3 = theme.On, Text = "Close", Font = Enum.Font.GothamSemibold,
		TextSize = 12, TextColor3 = C.White, AutoButtonColor = false, ZIndex = 52,
	}, panel)
	Refine.Round(close, 6)
	local json = Refine.New("TextBox", {
		Position = UDim2.fromOffset(18, 56), Size = UDim2.new(1, -36, 1, -106),
		BackgroundColor3 = C.Search, Text = encoded, TextColor3 = C.White,
		Font = Enum.Font.Code, TextSize = 13, TextWrapped = true, MultiLine = true,
		TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top,
		ClearTextOnFocus = false, ClipsDescendants = true, ZIndex = 52,
	}, panel)
	Refine.Round(json, 6)
	Refine.New("UIPadding", { PaddingLeft = UDim.new(0, 10), PaddingRight = UDim.new(0, 10), PaddingTop = UDim.new(0, 10) }, json)
	Refine.New("TextLabel", {
		Position = UDim2.new(0, 18, 1, -40), Size = UDim2.new(1, -36, 0, 28),
		BackgroundTransparency = 1, Text = "Select and copy this text. Profiles are stored for this UI session.",
		TextWrapped = true, Font = Enum.Font.Gotham, TextSize = 11, TextColor3 = C.Gray, ZIndex = 52,
	}, panel)
	local function closeExport()
		if Refine.popup==closeExport then Refine.popup=nil end
		json:ReleaseFocus()
		overlay:Destroy()
	end
	Refine.popup=closeExport
	Refine.Click(close,closeExport)
	Refine.Click(overlay,closeExport)
	json:CaptureFocus()
	json.SelectionStart = 1
	json.CursorPosition = #encoded + 1
	Log("Configuration JSON opened for manual copy", "OK")
	return encoded
end

local function ImportConfigFromString(jsonString)
	if type(jsonString) ~= "string" or jsonString == "" or #jsonString > 131072 then
		Notify("Import Failed", "Paste configuration JSON up to 128 KB", nil, "error")
		return false
	end

	local ok, data = pcall(function()
		return Http:JSONDecode(jsonString)
	end)

	local applied, reason = false, "Invalid JSON format"
	if ok and type(data) == "table" then
		applied, reason = Refine.ApplyData(data, true)
	end
	if applied then
		SaveConfig(true)
		Notify("Import Success", "Loaded config from JSON!", nil, "success")
		Log("Imported config JSON", "OK")
	else
		Notify("Import Error", reason or "Invalid configuration", nil, "error")
		Log("JSON Import failed: " .. (reason or "Invalid configuration"), "ERR")
	end
	return applied
end

local function ResetToDefault()
	Refine.EndSlider()
	Refine.ClosePopup()
	if Refine.CancelKeyCapture then Refine.CancelKeyCapture() end
	pendingValues = {}

	for _, e in ipairs(configRegistry) do
		pcall(e.set, e.default)
	end

	binds = {}
	uiToggleKey = Enum.KeyCode.RightControl
	if Refine.SyncHotkey then Refine.SyncHotkey() end
	for t, _ in pairs(bindLabels) do
		bindLabels[t]("")
	end

	ApplyTheme("Emerald", true)
	liveDefaults["Theme Preset"] = "Emerald"
	liveDefaults["Active Profile"] = activeProfile
	for _, entry in ipairs(configRegistry) do
		if entry.key == "Active Profile" then entry.set(activeProfile) end
	end

	Notify("Reset Complete", "All settings reverted to default")
	Log("Reset all settings to default", "WARN")
	TriggerAutoSave()
end

local Lur = {
	Notify = Notify,
	Log = Log,
	On = function(t, fn) hooks[t] = fn end,
	Set = function(t, v)
		if toggleByTitle[t] then
			toggleByTitle[t](v)
		end
	end,
	Save = SaveConfig,
	Load = LoadConfig,
	Export = ExportConfigToClipboard,
	Import = ImportConfigFromString,
	Reset = ResetToDefault,
	Theme = function(n) ApplyTheme(n) end,
}

local listening, listeningTitle = nil, nil
local BIND_W, BIND_H = IS_TOUCH and 42 or 30, IS_TOUCH and 26 or 18

function Refine.AddToggle(row, spec)
	local title = row.Title.Text
	local state = spec.on == true
	local TRACK_W, TRACK_H = IS_TOUCH and 54 or 50, IS_TOUCH and 30 or 27
	local KNOB = IS_TOUCH and 22 or 20
	local PAD = 4
	local LEFT_SCALE = 0.25
	local RIGHT_SCALE = 0.75

	local track = Refine.New("Frame", {
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -16, 0.5, 0),
		Size = UDim2.fromOffset(TRACK_W, TRACK_H),
		BackgroundColor3 = state and theme.On or C.Off,
		BorderSizePixel = 0,
		ZIndex = 2,
	}, row)
	Refine.Round(track, TRACK_H / 2)

	local trackGradient = Refine.New("UIGradient", {
		Rotation = 90,
		Color = ColorSequence.new({
			ColorSequenceKeypoint.new(0, C.Off),
			ColorSequenceKeypoint.new(1, C.Off),
		}),
	}, track)

	local trackStroke = Refine.New("UIStroke", {
		Thickness = 1,
		Color = C.BorderLight,
		Transparency = state and 0.06 or 0.30,
	}, track)

	local knob = Refine.New("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.new(state and RIGHT_SCALE or LEFT_SCALE, 0, 0.5, 0),
		Size = UDim2.fromOffset(KNOB, KNOB),
		BackgroundColor3 = C.White,
		BorderSizePixel = 0,
		ZIndex = 4,
	}, track)
	Refine.Round(knob, KNOB / 2)

	local knobStroke = Refine.New("UIStroke", {
		Thickness = 1,
		Color = C.BorderLight,
		Transparency = 0.10,
	}, knob)

	local shine = Refine.New("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.new(0.30, 0, 0.25, 0),
		Size = UDim2.fromOffset(math.max(4, KNOB * 0.30), 2),
		BackgroundColor3 = Color3.new(1, 1, 1),
		BackgroundTransparency = 0.18,
		Rotation = -10,
		ZIndex = 5,
	}, knob)
	Refine.Round(shine, 1)

	local hit = Refine.New("TextButton", {
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.fromScale(0.5, 0.5),
		Size = UDim2.new(1, 14, 1, 14),
		BackgroundTransparency = 1,
		AutoButtonColor = false,
		Text = "",
		ZIndex = 6,
	}, track)

	local bindBtn = Refine.New("TextButton", {
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -74, 0.5, 0),
		Size = UDim2.new(0, BIND_W, 0, BIND_H),
		BackgroundTransparency = 1,
		Text = binds[title] or "...",
		TextColor3 = C.GrayDk,
		Font = Enum.Font.Gotham,
		TextSize = 10,
		AutoButtonColor = false,
		ZIndex = 2,
	}, row)

	bindLabels[title] = function(k)
		bindBtn.Text = (k and k ~= "") and k or "..."
	end

	Refine.Click(bindBtn, function()
		listening = true
		Refine.capturingKey = true
		listeningTitle = title
		bindBtn.Text = "..."
		Notify("Keybind", "Press a key for " .. title)
	end)

	local function paint(animated)
		local on = state
		local accent = theme.Accent
		local base = on and theme.On or C.Off
		local top = on and ShadeColor(accent, 0.02) or TintColor(C.Off, 0.08)
		local bottom = on and ShadeColor(theme.On, 0.16) or ShadeColor(C.Off, 0.04)

		track.BackgroundColor3 = base
		trackGradient.Color = ColorSequence.new({
			ColorSequenceKeypoint.new(0, top),
			ColorSequenceKeypoint.new(1, bottom),
		})
		trackStroke.Color = on and TintColor(accent, 0.10) or C.BorderLight
		trackStroke.Transparency = on and 0.04 or 0.28
		knobStroke.Color = on and TintColor(accent, 0.02) or C.BorderLight
		shine.BackgroundTransparency = on and 0.14 or 0.20

		local target = UDim2.new(on and RIGHT_SCALE or LEFT_SCALE, 0, 0.5, 0)
		if animated and not Refine.reducedMotion then
			Refine.Tween(knob, TweenInfo.new(0.16, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), { Position = target }):Play()
		else
			knob.Position = target
		end
	end

	local function SetState(v, silent)
		local nextState = v == true
		if nextState == state and not silent then return end
		state = nextState
		paint(true)

		if spec.onChange then Refine.SafeCall("Toggle:" .. title, spec.onChange, state) end
		Refine.FireHook(title, state)

		if not silent then
			Log(title .. " -> " .. (state and "ON" or "OFF"), state and "OK" or "INFO")
			Notify(title, state and "Enabled" or "Disabled")
			TriggerAutoSave()
		end
	end

	HookTheme(function() paint(false) end, row)

	Refine.Click(hit, function()
		SetState(not state)
	end)

	hit.MouseEnter:Connect(function()
		Refine.Tween(trackStroke, EASE, { Transparency = state and 0.0 or 0.14 }):Play()
	end)
	hit.MouseLeave:Connect(function()
		Refine.Tween(trackStroke, EASE, { Transparency = state and 0.04 or 0.28 }):Play()
	end)

	local rowHit = Refine.New("TextButton", {
		Size = UDim2.new(1, -80, 1, 0),
		BackgroundTransparency = 1,
		AutoButtonColor = false,
		Text = "",
		ZIndex = 1,
	}, row)
	Refine.Click(rowHit, function() SetState(not state) end)

	Refine.RegConfig(title, "toggle",
		function() return state end,
		function(v) SetState(v == true, true) end,
		spec.on == true, spec
	)
	toggleByTitle[title] = function(v)
		SetState(v == nil and not state or v == true, true)
	end

	paint(false)
end

function Refine.AddSlider(row, spec)
	local min = tonumber(spec.min) or 0
	local max = tonumber(spec.max) or 100
	if max < min then min, max = max, min end
	local val = math.clamp(tonumber(spec.value) or min, min, max)
	local defaultValue = val
	local titleLab = row.Title
	local title = titleLab.Text
	local W = IS_TOUCH and 138 or 168
	local TRACK_H = IS_TOUCH and 8 or 6
	local KNOB = IS_TOUCH and 24 or 18
	local EDGE = KNOB / 2

	local track = Refine.New("Frame", {
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -16, 0.5, -1),
		Size = UDim2.fromOffset(W, TRACK_H),
		BackgroundColor3 = Color3.fromRGB(58, 61, 70),
		BorderSizePixel = 0,
		ZIndex = 2,
	}, row)
	Refine.Round(track, TRACK_H / 2)

	local baseGradient = Refine.New("UIGradient", {
		Rotation = 90,
		Color = ColorSequence.new({
			ColorSequenceKeypoint.new(0, Color3.fromRGB(68, 72, 82)),
			ColorSequenceKeypoint.new(1, Color3.fromRGB(47, 50, 58)),
		}),
	}, track)

	local trackStroke = Refine.New("UIStroke", { Thickness = 1, Transparency = 0.12, Color = Color3.fromRGB(74, 78, 88) }, track)

	local fill = Refine.New("Frame", {
		AnchorPoint = Vector2.new(0, 0.5),
		Position = UDim2.new(0, 0, 0.5, 0),
		Size = UDim2.new(0, 0, 1, 0),
		BackgroundColor3 = theme.Accent,
		BorderSizePixel = 0,
		ZIndex = 3,
	}, track)
	Refine.Round(fill, TRACK_H / 2)
	local fillGradient = Refine.New("UIGradient", {
		Rotation = 90,
		Color = ColorSequence.new({
			ColorSequenceKeypoint.new(0, TintColor(theme.Accent, 0.12)),
			ColorSequenceKeypoint.new(1, ShadeColor(theme.Accent, 0.05)),
		}),
	}, fill)
	local fillStroke = Refine.New("UIStroke", { Thickness = 1, Transparency = 0.18, Color = TintColor(theme.Accent, 0.02) }, fill)

	local knob = Refine.New("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.new(0, EDGE, 0.5, 0),
		Size = UDim2.fromOffset(KNOB, KNOB),
		BackgroundColor3 = C.White,
		BorderSizePixel = 0,
		ZIndex = 6,
	}, track)
	Refine.Round(knob, KNOB / 2)
	local knobStroke = Refine.New("UIStroke", { Thickness = 1.15, Transparency = 0.05, Color = TintColor(theme.Accent, 0.02) }, knob)
	local knobShadow = Refine.New("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.fromOffset(0, 2),
		Size = UDim2.fromOffset(KNOB - 2, KNOB - 2),
		BackgroundColor3 = Color3.new(0, 0, 0),
		BackgroundTransparency = 0.82,
		ZIndex = 5,
	}, knob)
	Refine.Round(knobShadow, (KNOB - 2) / 2)
	local shine = Refine.New("Frame", {
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.new(0.28, 0, 0.24, 0),
		Size = UDim2.fromOffset(math.max(5, KNOB * 0.28), 2),
		BackgroundColor3 = Color3.new(1, 1, 1),
		BackgroundTransparency = 0.16,
		Rotation = -12,
		ZIndex = 7,
	}, knob)
	Refine.Round(shine, 1)

	-- The hitbox is intentionally wider than the visible track for mobile accuracy.
	local hit = Refine.New("TextButton", {
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -16, 0.5, 0),
		Size = UDim2.fromOffset(W + 36, IS_TOUCH and 52 or 40),
		BackgroundTransparency = 1,
		AutoButtonColor = false,
		Text = "",
		ZIndex = 8,
	}, row)

	local function Ratio(v)
		if max == min then return 0 end
		return math.clamp((v - min) / (max - min), 0, 1)
	end

	local function Apply(animated)
		local ratio = Ratio(val)
		local travel = math.max(1, W - KNOB)
		local targetFill = UDim2.new(0, EDGE + ratio * travel, 1, 0)
		local targetKnob = UDim2.new(0, EDGE + ratio * travel, 0.5, 0)

		fill.BackgroundColor3 = theme.Accent
		fillGradient.Color = ColorSequence.new({
			ColorSequenceKeypoint.new(0, TintColor(theme.Accent, 0.12)),
			ColorSequenceKeypoint.new(1, ShadeColor(theme.Accent, 0.05)),
		})
		fillStroke.Color = TintColor(theme.Accent, 0.02)
		knobStroke.Color = TintColor(theme.Accent, 0.02)

		if animated and not Refine.reducedMotion then
			Refine.Tween(fill, TweenInfo.new(0.10, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), { Size = targetFill }):Play()
			Refine.Tween(knob, TweenInfo.new(0.12, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), { Position = targetKnob }):Play()
		else
			fill.Size = targetFill
			knob.Position = targetKnob
		end

		titleLab.Text = string.format("%s ( %d )", title, math.floor(val + 0.5))
	end

	local dirty = false
	local function SetVal(v, silent)
		v = math.clamp(math.floor((tonumber(v) or min) + 0.5), min, max)
		if v == val then return end
		val = v
		dirty = true
		Apply(true)
		if spec.onChanged then Refine.SafeCall("Slider:" .. title, spec.onChanged, val) end
		if not silent then
			Log(title .. " -> " .. val)
			TriggerAutoSave()
		end
	end

	local function CommitDrag()
		if not dirty then return end
		dirty = false
		Log(title .. " -> " .. val)
		TriggerAutoSave()
	end

	local function SetFromX(x)
		local left = track.AbsolutePosition.X
		local width = math.max(track.AbsoluteSize.X, W)
		local usable = math.max(1, width - KNOB)
		local ratio = math.clamp((x - left - EDGE) / usable, 0, 1)
		SetVal(min + ratio * (max - min), true)
	end

	local function beginDrag(inp)
		if inp.UserInputType ~= Enum.UserInputType.MouseButton1 and inp.UserInputType ~= Enum.UserInputType.Touch then return end
		if activeSlider and activeSlider ~= SetFromX then return end

		Refine.EndSlider()
		local scrolling = {}
		local ancestor = hit.Parent
		while ancestor do
			if ancestor:IsA("ScrollingFrame") then
				table.insert(scrolling, { ancestor, ancestor.ScrollingEnabled })
				ancestor.ScrollingEnabled = false
			end
			ancestor = ancestor.Parent
		end

		Refine.sliderInput = inp
		activeSlider = SetFromX
		isSliderActive = true
		dirty = false
		activeSliderCommit = function()
			for _, entry in ipairs(scrolling) do
				if entry[1].Parent then entry[1].ScrollingEnabled = entry[2] end
			end
			CommitDrag()
		end

		SetFromX(GetInputPos(inp).X)
		if not Refine.reducedMotion then
			Refine.Tween(knob, EASE, { Size = UDim2.fromOffset(KNOB + 3, KNOB + 3) }):Play()
		end
	end

	hit.InputBegan:Connect(beginDrag)
	hit.InputEnded:Connect(function(inp)
		if inp == Refine.sliderInput and inp.UserInputType == Enum.UserInputType.MouseButton1 then
			Refine.EndSlider()
		end
	end)

	hit.MouseEnter:Connect(function()
		Refine.Tween(trackStroke, EASE, { Transparency = 0.08 }):Play()
	end)
	hit.MouseLeave:Connect(function()
		if not isSliderActive then Refine.Tween(trackStroke, EASE, { Transparency = 0.12 }):Play() end
	end)

	hit.Selectable = true
	hit.NextSelectionLeft = hit
	hit.NextSelectionRight = hit
	hit.InputBegan:Connect(function(inp)
		if inp.KeyCode == Enum.KeyCode.DPadLeft or inp.KeyCode == Enum.KeyCode.Left then
			SetVal(val - 1, false)
		elseif inp.KeyCode == Enum.KeyCode.DPadRight or inp.KeyCode == Enum.KeyCode.Right then
			SetVal(val + 1, false)
		end
	end)

	HookTheme(function() Apply(false) end, row)
	Apply(false)

	Refine.RegConfig(title, "slider",
		function() return val end,
		function(v) SetVal(v, true) end,
		defaultValue, spec
	)
end

function Refine.AddButton(row, spec)
	local btn = Refine.New("TextButton", {
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -16, 0.5, 0),
		Size = UDim2.new(0, 72, 0, IS_TOUCH and 40 or 26),
		BackgroundColor3 = theme.On,
		Text = spec.buttonText or "Go",
		TextColor3 = C.White,
		Font = Enum.Font.GothamSemibold,
		TextSize = 12,
		AutoButtonColor = false,
		ZIndex = 2,
	}, row)
	Refine.Round(btn, 6)
	Refine.Shade(btn)
	Refine.AddTactileFeedback(btn)

	HookTheme(function(t)
		btn.BackgroundColor3 = t.On
	end, row)

	Refine.Click(btn, function()
		Log((spec.logName or row.Title.Text) .. " pressed")
		if spec.onPress then
			Refine.SafeCall("Button:" .. (spec.logName or row.Title.Text), spec.onPress,
				function(t)
					row.Title.Text = t
				end,
				function(d)
					local dl = row:FindFirstChild("Desc")
					if dl then
						dl.Text = d
					end
				end
			)
		end
	end)
end

function Refine.AddInput(row, spec)
	local box = Refine.New("TextBox", {
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -16, 0.5, 0),
		Size = UDim2.new(0, IS_TOUCH and 140 or 170, 0, 28),
		BackgroundColor3 = C.Search,
		Text = spec.value or "",
		PlaceholderText = spec.placeholder or "Value...",
		PlaceholderColor3 = C.Gray,
		TextColor3 = C.White,
		Font = Enum.Font.Gotham,
		TextSize = 12,
		ClearTextOnFocus = false,
		ZIndex = 2,
	}, row)
	Refine.Round(box, 6)

	local boxStroke = Refine.New("UIStroke", { Color = C.Border, Thickness = 1 }, box)
	Refine.New("UIPadding", { PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 8) }, box)

	box.Focused:Connect(function()
		boxStroke.Color = theme.Accent
	end)

	box.FocusLost:Connect(function()
		boxStroke.Color = C.Border
		if spec.onChanged then
			Refine.SafeCall("Input:" .. row.Title.Text, spec.onChanged, box.Text)
		end
		TriggerAutoSave()
	end)

	Refine.RegConfig(row.Title.Text, "input",
		function()
			return box.Text
		end,
		function(v)
			box.Text = tostring(v or "")
			if spec.onChanged then Refine.SafeCall("Input:" .. row.Title.Text, spec.onChanged, box.Text) end
		end,
		spec.value or "", spec
	)
end

function Refine.AddDropdown(row, spec)
	local opts = spec.options or {}
	local title = row.Title.Text
	local selectedVal = liveDefaults[title] or spec.default or (opts[1] or "")

	local valueLab = Refine.New("TextLabel", {
		Name = "DropdownValue",
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -32, 0.5, 0),
		Size = UDim2.new(0, IS_TOUCH and 110 or 150, 0, 20),
		BackgroundTransparency = 1,
		Text = selectedVal,
		TextColor3 = C.White,
		Font = Enum.Font.GothamMedium,
		TextSize = 13,
		TextXAlignment = Enum.TextXAlignment.Right,
		TextTruncate = Enum.TextTruncate.AtEnd,
		ZIndex = 2,
	}, row)

	local rowChev = Refine.Chevron(row, UDim2.new(1, -18, 0.5, 0))
	local openList, clickOutside = nil, nil
	local hit

	local function CloseDropdown()
		if Refine.popup==CloseDropdown then Refine.popup=nil end
		Refine.Tween(rowChev, EASE, { Rotation = 0 }):Play()

		if openList then
			openList:Destroy()
			openList = nil
		end

		if clickOutside then
			clickOutside:Destroy()
			clickOutside = nil
		end
		if Refine.RestoreFocus then Refine.RestoreFocus(hit) end
	end

	local function SetOpt(opt, silent)
		if title == "Active Profile" and not silent and opt ~= activeProfile and autoSaveConfig then
			SaveConfig(true)
		end
		selectedVal = opt
		valueLab.Text = opt

		if spec.onSelect and not silent then
			Refine.SafeCall("Dropdown:" .. title, spec.onSelect, opt)
		end

		if not silent then
			Log(title .. " -> " .. opt)
			TriggerAutoSave()
		end
	end

	hit = Refine.New("TextButton", {
		Name = "DropdownOpener",
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundTransparency = 1,
		AutoButtonColor = false,
		Text = "",
		ZIndex = 3,
	}, row)

	Refine.Click(hit, function()
		if openList then
			CloseDropdown()
			return
		end

		Refine.ClosePopup()
		Refine.popup=CloseDropdown

		Refine.Tween(rowChev, EASE, { Rotation = 180 }):Play()

		clickOutside = Refine.New("TextButton", {
			Name = "LurModalBackdrop",
			Size = UDim2.new(1, 0, 1, 0),
			BackgroundTransparency = 1,
			Text = "",
			ZIndex = 1999,
			Parent = Gui,
		})
		clickOutside.Activated:Connect(CloseDropdown)

		local OH = IS_TOUCH and 44 or 28
		local optionHeight = #opts * OH + math.max(0, #opts - 1) * 2
		local listW = math.min(220, math.max(1, vp.X - 20))
		local listH = math.min(40 + math.max(OH, optionHeight), 220, math.max(1, vp.Y - 20))
		local absPos = row.AbsolutePosition
		local absSize = row.AbsoluteSize
		local listY = absPos.Y + absSize.Y

		if listY + listH > vp.Y - 10 then
			listY = math.max(10, absPos.Y - listH)
		end

		local list = Refine.New("Frame", {
			Name = "LurDropdownList",
			BackgroundColor3 = C.Card,
			BorderSizePixel = 0,
			ZIndex = 2000,
			Size = UDim2.new(0, listW, 0, listH),
			Position = UDim2.fromOffset(
				math.clamp(absPos.X + absSize.X - listW - 10, 10, math.max(10, vp.X - listW - 10)),
				math.clamp(listY, 10, math.max(10, vp.Y - listH - 10))
			),
			Parent = Gui,
		})
		Refine.Round(list, 8)

		Refine.New("UIStroke", { Color = C.Border, Thickness = 1.2 }, list)
		Refine.New("UIPadding", {
			PaddingTop = UDim.new(0, 4),
			PaddingBottom = UDim.new(0, 4),
			PaddingLeft = UDim.new(0, 4),
			PaddingRight = UDim.new(0, 4),
		}, list)

		local dSearch = Refine.New("TextBox", {
			Name = "DropdownSearch",
			Size = UDim2.new(1, 0, 0, 28),
			BackgroundColor3 = C.Search,
			Text = "",
			PlaceholderText = "Search...",
			PlaceholderColor3 = C.Gray,
			TextColor3 = C.White,
			Font = Enum.Font.Gotham,
			TextSize = 12,
			TextXAlignment = Enum.TextXAlignment.Left,
			ClearTextOnFocus = false,
			ZIndex = 2001,
		}, list)
		Refine.Round(dSearch, 6)
		Refine.New("UIPadding", { PaddingLeft = UDim.new(0, 8) }, dSearch)

		local scOpts = Refine.New("ScrollingFrame", {
			Name = "DropdownOptions",
			Position = UDim2.new(0, 0, 0, 32),
			Size = UDim2.new(1, 0, 1, -32),
			BackgroundTransparency = 1,
			BorderSizePixel = 0,
			ScrollBarThickness = 3,
			CanvasSize = UDim2.new(0, 0, 0, optionHeight),
			ZIndex = 2001,
			Parent = list,
		})
		Refine.EnableTouchPan(scOpts)
		Refine.New("UIListLayout", {
			Padding = UDim.new(0, 2),
			SortOrder = Enum.SortOrder.LayoutOrder,
		}, scOpts)

		local optBtns = {}
		local emptyLabel = Refine.New("TextLabel", {
			Name = "DropdownEmpty",
			Position = UDim2.fromOffset(4, 36),
			Size = UDim2.new(1, -8, 0, OH),
			BackgroundTransparency = 1,
			Text = "No matching options",
			TextColor3 = C.Gray,
			Font = Enum.Font.Gotham,
			TextSize = 12,
			TextWrapped = true,
			Visible = #opts == 0,
			ZIndex = 2002,
		}, list)

		for _, opt in ipairs(opts) do
			local ob = Refine.New("TextButton", {
				Name = "DropdownOption",
				Size = UDim2.new(1, 0, 0, OH),
				BackgroundTransparency = 1,
				Text = opt,
				TextColor3 = (opt == selectedVal) and C.White or C.Gray,
				Font = Enum.Font.GothamMedium,
				TextSize = 13,
				TextXAlignment = Enum.TextXAlignment.Left,
				TextTruncate = Enum.TextTruncate.AtEnd,
				AutoButtonColor = false,
				ZIndex = 2002,
				Parent = scOpts,
			}, scOpts)

			ob.MouseEnter:Connect(function() ob.TextColor3 = C.White end)
			ob.MouseLeave:Connect(function()
				if opt ~= selectedVal then
					ob.TextColor3 = C.Gray
				end
			end)

			Refine.Click(ob, function()
				SetOpt(opt)
				CloseDropdown()
			end)

			table.insert(optBtns, ob)
		end

		dSearch:GetPropertyChangedSignal("Text"):Connect(function()
			local q = dSearch.Text:lower()
			local vis = 0

			for _, ob in ipairs(optBtns) do
				ob.Visible = q == "" or ob.Text:lower():find(q, 1, true) ~= nil
				if ob.Visible then
					vis = vis + 1
				end
			end

			scOpts.CanvasSize = UDim2.new(0, 0, 0, vis * OH + math.max(0, vis - 1) * 2)
			scOpts.CanvasPosition = Vector2.new(0, 0)
			emptyLabel.Visible = vis == 0
		end)

		openList = list
		if Refine.FocusPopup then Refine.FocusPopup(list, hit) end
	end)

	Refine.RegConfig(title, "drop",
		function()
			if title == "Active Profile" then return activeProfile end
			if title == "Theme Preset" then return currentThemeName end
			return valueLab.Text
		end,
		function(v)
			SetOpt(v, true)
		end,
		spec.default or (opts[1] or ""), spec
	)
end

function Refine.AddMultiDropdown(row, spec)
	local opts = spec.options or {}
	local title = row.Title.Text
	local selected = {}

	if type(spec.default) == "table" then
		for _, v in ipairs(spec.default) do
			selected[v] = true
		end

		for k, v in pairs(spec.default) do
			if type(k) == "string" and v == true then
				selected[k] = true
			end
		end
	end

	local defaultSnapshot = {}
	for k in pairs(selected) do
		defaultSnapshot[k] = true
	end

	local valueLab = Refine.New("TextLabel", {
		Name = "MultiDropdownValue",
		AnchorPoint = Vector2.new(1, 0.5),
		Position = UDim2.new(1, -32, 0.5, 0),
		Size = UDim2.new(0, IS_TOUCH and 110 or 150, 0, 20),
		BackgroundTransparency = 1,
		Text = "",
		TextColor3 = C.White,
		Font = Enum.Font.GothamMedium,
		TextSize = 13,
		TextXAlignment = Enum.TextXAlignment.Right,
		TextTruncate = Enum.TextTruncate.AtEnd,
		ZIndex = 2,
	}, row)

	local rowChev = Refine.Chevron(row, UDim2.new(1, -18, 0.5, 0))

	local function GetSelectedList()
		local list = {}
		for _, opt in ipairs(opts) do
			if selected[opt] then
				table.insert(list, opt)
			end
		end
		return list
	end

	local function UpdateDisplaySummary()
		local list = GetSelectedList()

		if #list == 0 then
			valueLab.Text = "None"
			valueLab.TextColor3 = C.Gray
		elseif #list <= 2 then
			valueLab.Text = table.concat(list, ", ")
			valueLab.TextColor3 = C.White
		else
			valueLab.Text = string.format("%d Selected", #list)
			valueLab.TextColor3 = theme.Accent
		end
	end

	UpdateDisplaySummary()

	local openList, clickOutside = nil, nil
	local hit

	local function CloseMulti()
		if Refine.popup==CloseMulti then Refine.popup=nil end
		Refine.Tween(rowChev, EASE, { Rotation = 0 }):Play()

		if openList then
			openList:Destroy()
			openList = nil
		end

		if clickOutside then
			clickOutside:Destroy()
			clickOutside = nil
		end
		if Refine.RestoreFocus then Refine.RestoreFocus(hit) end
	end

	hit = Refine.New("TextButton", {
		Name = "MultiDropdownOpener",
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundTransparency = 1,
		AutoButtonColor = false,
		Text = "",
		ZIndex = 3,
	}, row)

	Refine.Click(hit, function()
		if openList then
			CloseMulti()
			return
		end

		Refine.ClosePopup()
		Refine.popup=CloseMulti

		Refine.Tween(rowChev, EASE, { Rotation = 180 }):Play()

		clickOutside = Refine.New("TextButton", {
			Name = "LurModalBackdrop",
			Size = UDim2.new(1, 0, 1, 0),
			BackgroundTransparency = 1,
			Text = "",
			ZIndex = 1999,
			Parent = Gui,
		})
		clickOutside.Activated:Connect(CloseMulti)

		local OH = IS_TOUCH and 44 or 28
		local optionHeight = #opts * OH + math.max(0, #opts - 1) * 2
		local listW = math.min(235, math.max(1, vp.X - 20))
		local listH = math.min(44 + math.max(OH, optionHeight), 250, math.max(1, vp.Y - 20))
		local absPos = row.AbsolutePosition
		local absSize = row.AbsoluteSize
		local listY = absPos.Y + absSize.Y

		if listY + listH > vp.Y - 10 then
			listY = math.max(10, absPos.Y - listH)
		end

		local list = Refine.New("Frame", {
			Name = "LurDropdownList",
			BackgroundColor3 = C.Card,
			BorderSizePixel = 0,
			ZIndex = 2000,
			Size = UDim2.new(0, listW, 0, listH),
			Position = UDim2.fromOffset(
				math.clamp(absPos.X + absSize.X - listW - 10, 10, math.max(10, vp.X - listW - 10)),
				math.clamp(listY, 10, math.max(10, vp.Y - listH - 10))
			),
			Parent = Gui,
		})
		Refine.Round(list, 8)

		Refine.New("UIStroke", { Color = C.Border, Thickness = 1.2 }, list)
		Refine.New("UIPadding", {
			PaddingTop = UDim.new(0, 5),
			PaddingBottom = UDim.new(0, 5),
			PaddingLeft = UDim.new(0, 5),
			PaddingRight = UDim.new(0, 5),
		}, list)

		local topBar = Refine.New("Frame", {
			Size = UDim2.new(1, 0, 0, 28),
			BackgroundTransparency = 1,
			ZIndex = 2001,
		}, list)

		local dSearch = Refine.New("TextBox", {
			Name = "DropdownSearch",
			Size = UDim2.new(1, -90, 1, 0),
			BackgroundColor3 = C.Search,
			Text = "",
			PlaceholderText = "Search...",
			PlaceholderColor3 = C.Gray,
			TextColor3 = C.White,
			Font = Enum.Font.Gotham,
			TextSize = 11,
			TextXAlignment = Enum.TextXAlignment.Left,
			ClearTextOnFocus = false,
			ZIndex = 2002,
		}, topBar)
		Refine.Round(dSearch, 6)
		Refine.New("UIPadding", { PaddingLeft = UDim.new(0, 6) }, dSearch)

		local btnAll = Refine.New("TextButton", {
			Name = "DropdownSelectAll",
			Position = UDim2.new(1, -84, 0, 0),
			Size = UDim2.new(0, 40, 1, 0),
			BackgroundColor3 = theme.On,
			Text = "All",
			TextColor3 = C.White,
			Font = Enum.Font.GothamSemibold,
			TextSize = 10,
			AutoButtonColor = false,
			ZIndex = 2002,
		}, topBar)
		Refine.Round(btnAll, 5)

		local btnClr = Refine.New("TextButton", {
			Name = "DropdownClearAll",
			Position = UDim2.new(1, -40, 0, 0),
			Size = UDim2.new(0, 40, 1, 0),
			BackgroundColor3 = C.Search,
			Text = "Clear",
			TextColor3 = C.Gray,
			Font = Enum.Font.GothamSemibold,
			TextSize = 10,
			AutoButtonColor = false,
			ZIndex = 2002,
		}, topBar)
		Refine.Round(btnClr, 5)

		local scOpts = Refine.New("ScrollingFrame", {
			Name = "DropdownOptions",
			Position = UDim2.new(0, 0, 0, 34),
			Size = UDim2.new(1, 0, 1, -34),
			BackgroundTransparency = 1,
			BorderSizePixel = 0,
			ScrollBarThickness = 3,
			CanvasSize = UDim2.new(0, 0, 0, optionHeight),
			ZIndex = 2001,
			Parent = list,
		})
		Refine.EnableTouchPan(scOpts)
		Refine.New("UIListLayout", {
			Padding = UDim.new(0, 2),
			SortOrder = Enum.SortOrder.LayoutOrder,
		}, scOpts)

		local itemRecords = {}
		local emptyLabel = Refine.New("TextLabel", {
			Name = "DropdownEmpty",
			Position = UDim2.fromOffset(4, 38),
			Size = UDim2.new(1, -8, 0, OH),
			BackgroundTransparency = 1,
			Text = "No matching options",
			TextColor3 = C.Gray,
			Font = Enum.Font.Gotham,
			TextSize = 12,
			TextWrapped = true,
			Visible = #opts == 0,
			ZIndex = 2002,
		}, list)

		for _, opt in ipairs(opts) do
			local ob = Refine.New("TextButton", {
				Name = "MultiDropdownOption",
				Size = UDim2.new(1, 0, 0, OH),
				BackgroundTransparency = 1,
				AutoButtonColor = false,
				Text = "",
				ZIndex = 2002,
				Parent = scOpts,
			}, scOpts)

			local box = Refine.New("Frame", {
				AnchorPoint = Vector2.new(0, 0.5),
				Position = UDim2.new(0, 4, 0.5, 0),
				Size = UDim2.new(0, 16, 0, 16),
				BackgroundColor3 = selected[opt] and theme.Accent or C.Search,
				BorderSizePixel = 0,
				ZIndex = 2003,
			}, ob)
			Refine.Round(box, 4)

			local selectedStroke = Refine.New("UIStroke", {
				Color = selected[opt] and theme.Accent or C.Border,
				Thickness = 1,
			}, box)

			local checkmark = Refine.New("TextLabel", {
				Size = UDim2.new(1, 0, 1, 0),
				BackgroundTransparency = 1,
				Text = selected[opt] and "✓" or "",
				TextColor3 = C.White,
				Font = Enum.Font.GothamBold,
				TextSize = 11,
				ZIndex = 2004,
			}, box)

			local label = Refine.New("TextLabel", {
				Position = UDim2.new(0, 26, 0, 0),
				Size = UDim2.new(1, -30, 1, 0),
				BackgroundTransparency = 1,
				Text = opt,
				TextColor3 = selected[opt] and C.White or C.Gray,
				Font = Enum.Font.GothamMedium,
				TextSize = 12,
				TextXAlignment = Enum.TextXAlignment.Left,
				TextTruncate = Enum.TextTruncate.AtEnd,
				ZIndex = 2003,
			}, ob)

			local function RefreshItemUI()
				local isSel = selected[opt] == true
				box.BackgroundColor3 = isSel and theme.Accent or C.Search
				selectedStroke.Color = isSel and theme.Accent or C.Border
				checkmark.Text = isSel and "✓" or ""
				label.TextColor3 = isSel and C.White or C.Gray
			end

			Refine.Click(ob, function()
				selected[opt] = not selected[opt]
				RefreshItemUI()
				UpdateDisplaySummary()

				if spec.onSelect then
					Refine.SafeCall("MultiDropdown:" .. title, spec.onSelect, selected, GetSelectedList())
				end

				Log(title .. " -> " .. opt .. " (" .. (selected[opt] and "Added" or "Removed") .. ")")
				TriggerAutoSave()
			end)

			table.insert(itemRecords, { Button = ob, Label = label, Refresh = RefreshItemUI })
		end

		Refine.Click(btnAll, function()
			for _, opt in ipairs(opts) do
				selected[opt] = true
			end

			for _, r in ipairs(itemRecords) do
				r.Refresh()
			end

			UpdateDisplaySummary()

			if spec.onSelect then
				Refine.SafeCall("MultiDropdown:" .. title, spec.onSelect, selected, GetSelectedList())
			end

			TriggerAutoSave()
		end)

		Refine.Click(btnClr, function()
			selected = {}

			for _, r in ipairs(itemRecords) do
				r.Refresh()
			end

			UpdateDisplaySummary()

			if spec.onSelect then
				Refine.SafeCall("MultiDropdown:" .. title, spec.onSelect, selected, GetSelectedList())
			end

			TriggerAutoSave()
		end)

		dSearch:GetPropertyChangedSignal("Text"):Connect(function()
			local q = dSearch.Text:lower()
			local vis = 0

			for _, r in ipairs(itemRecords) do
				local match = q == "" or r.Label.Text:lower():find(q, 1, true) ~= nil
				r.Button.Visible = match
				if match then
					vis = vis + 1
				end
			end

			scOpts.CanvasSize = UDim2.new(0, 0, 0, vis * OH + math.max(0, vis - 1) * 2)
			scOpts.CanvasPosition = Vector2.new(0, 0)
			emptyLabel.Visible = vis == 0
		end)

		openList = list
		if Refine.FocusPopup then Refine.FocusPopup(list, hit) end
	end)

	Refine.RegConfig(title, "multidrop",
		function()
			return selected
		end,
		function(v)
			if type(v) == "table" then
				local newSel = {}

				for k, val in pairs(v) do
					if type(k) == "string" and val == true then
						newSel[k] = true
					elseif type(k) == "number" then
						newSel[val] = true
					end
				end

				selected = newSel
				UpdateDisplaySummary()
			end
		end,
		defaultSnapshot, spec
	)
end

function Refine.AddAccordion(card, layoutIdx, spec)
	local title = spec.title or spec.Title or "Group"
	local desc = spec.desc or spec.Desc or ""
	local isOpen = spec.default or false
	local searchOpen = false
	local childRowsData = spec.rows or {}

	local groupContainer = Refine.New("Frame", {
		Size = UDim2.new(1, 0, 0, ROW_H),
		BackgroundTransparency = 1,
		LayoutOrder = layoutIdx,
		ClipsDescendants = true,
		Parent = card,
	})

	local headRow = Refine.New("Frame", {
		Size = UDim2.new(1, 0, 0, ROW_H),
		BackgroundColor3 = C.Card,
		ZIndex = 2,
		Parent = groupContainer,
	})

	Refine.New("Frame", {
		Position = UDim2.new(0, 18, 1, -1),
		Size = UDim2.new(1, -36, 0, 1),
		BackgroundColor3 = C.Border,
		BorderSizePixel = 0,
		ZIndex = 3,
	}, headRow)

	Refine.New("TextLabel", {
		Position = UDim2.new(0, 20, 0, 11),
		Size = UDim2.new(1, -90, 0, 20),
		BackgroundTransparency = 1,
		Text = title,
		TextColor3 = C.White,
		Font = Enum.Font.GothamBold,
		TextSize = IS_TOUCH and 14 or 15,
		TextXAlignment = Enum.TextXAlignment.Left,
		ZIndex = 3,
	}, headRow)

	Refine.New("TextLabel", {
		Position = UDim2.new(0, 20, 0, 33),
		Size = UDim2.new(1, -90, 0, 18),
		BackgroundTransparency = 1,
		Text = (desc ~= "" and desc) or (#childRowsData .. " Options"),
		TextColor3 = C.Gray,
		Font = Enum.Font.Gotham,
		TextSize = IS_TOUCH and 12 or 13,
		TextXAlignment = Enum.TextXAlignment.Left,
		ZIndex = 3,
	}, headRow)

	local chev = Refine.Chevron(headRow, UDim2.new(1, -24, 0.5, 0))
	chev.Rotation = isOpen and 0 or -90

	local subContent = Refine.New("Frame", {
		Name = "AccordionContent",
		Visible = isOpen,
		Position = UDim2.new(0, 0, 0, ROW_H),
		Size = UDim2.new(1, 0, 0, isOpen and (#childRowsData * ROW_H) or 0),
		BackgroundColor3 = C.CardSub,
		BorderSizePixel = 0,
		ClipsDescendants = true,
		Parent = groupContainer,
	})

	Refine.New("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder }, subContent)

	local childRowFrames = {}

	for ci, cd in ipairs(childRowsData) do
		local cr = Refine.New("Frame", {
			Size = UDim2.new(1, 0, 0, ROW_H),
			BackgroundTransparency = 1,
			LayoutOrder = ci,
		}, subContent)

		Refine.New("TextLabel", {
			Name = "Title",
			Position = UDim2.new(0, 34, 0, 11),
			Size = UDim2.new(1, -214, 0, 20),
			BackgroundTransparency = 1,
			Text = cd.title,
			TextColor3 = C.White,
			Font = Enum.Font.GothamSemibold,
			TextSize = IS_TOUCH and 13 or 14,
			TextXAlignment = Enum.TextXAlignment.Left,
		}, cr)

		Refine.New("TextLabel", {
			Name = "Desc",
			Position = UDim2.new(0, 34, 0, 33),
			Size = UDim2.new(1, -214, 0, 18),
			BackgroundTransparency = 1,
			Text = cd.desc or "",
			TextColor3 = C.Gray,
			Font = Enum.Font.Gotham,
			TextSize = IS_TOUCH and 11 or 12,
			TextXAlignment = Enum.TextXAlignment.Left,
		}, cr)

		if ci < #childRowsData then
			Refine.New("Frame", {
				Position = UDim2.new(0, 32, 1, 0),
				Size = UDim2.new(1, -50, 0, 1),
				BackgroundColor3 = C.Border,
				BorderSizePixel = 0,
			}, cr)
		end

		local kind = cd.kind or "toggle"
		if kind == "toggle" then
			Refine.AddToggle(cr, cd)
		elseif kind == "slider" then
			Refine.AddSlider(cr, cd)
		elseif kind == "button" then
			Refine.AddButton(cr, cd)
		elseif kind == "drop" then
			Refine.AddDropdown(cr, cd)
		elseif kind == "multidrop" then
			Refine.AddMultiDropdown(cr, cd)
		elseif kind == "input" then
			Refine.AddInput(cr, cd)
		end

		table.insert(childRowFrames, cr)
		table.insert(allRows, cr)
	end

	local function CountVisibleChildren()
		local n = 0
		for _, cr in ipairs(childRowFrames) do
			if cr.Visible then
				n = n + 1
			end
		end
		return n
	end

	local function UpdateSize()
		headRow.Size=UDim2.new(1,0,0,ROW_H)
		subContent.Position=UDim2.fromOffset(0,ROW_H)
		local vis = CountVisibleChildren()

		if vis == 0 then
			subContent.Visible = false
			groupContainer.Visible = false
			groupContainer.Size = UDim2.new(1, 0, 0, 0)
			subContent.Size = UDim2.new(1, 0, 0, 0)
			return
		end

		groupContainer.Visible = true
		subContent.Visible = isOpen or searchOpen
		chev.Rotation = (isOpen or searchOpen) and 0 or -90
		local targetH = ((isOpen or searchOpen) and vis > 0) and (vis * ROW_H) or 0
		subContent.Size = UDim2.new(1, 0, 0, targetH)
		groupContainer.Size = UDim2.new(1, 0, 0, ROW_H + targetH)
	end

	local function ToggleAccordion()
		isOpen = not isOpen
		Refine.Tween(chev, EASE, { Rotation = (isOpen or searchOpen) and 0 or -90 }):Play()
		UpdateSize()

		if currentPage then
			Recompute(currentPage)
		end
	end

	local headHit = Refine.New("TextButton", {
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundTransparency = 1,
		AutoButtonColor = false,
		Text = "",
		ZIndex = 4,
	}, headRow)

	Refine.Click(headHit, ToggleAccordion)
	UpdateSize()

	return {
		Container = groupContainer,
		Rows = childRowFrames,
		SearchText = (title .. " " .. desc):lower(),
		SetSearchOpen = function(on) searchOpen = on; UpdateSize() end,
		IsAccordion = true,
		UpdateSize = UpdateSize,
		HasVisibleChild = function()
			return CountVisibleChildren() > 0
		end,
		GetHeight = function()
			local vis = CountVisibleChildren()
			if vis == 0 then
				return 0
			end
			return ROW_H + ((isOpen or searchOpen) and (vis * ROW_H) or 0)
		end,
	}
end

Recompute = function(page)
	local h, visCount = 0, 0

	for _, co in ipairs(page.cards) do
		local cardH = 0
		local vis = 0

		for _, r in ipairs(co.rows) do
			if typeof(r) == "table" and r.IsAccordion then
				if r.UpdateSize then
					r.UpdateSize()
				end

				if not r.HasVisibleChild or r.HasVisibleChild() then
					local accH = r.GetHeight and r.GetHeight() or ROW_H
					if accH > 0 then
						cardH = cardH + accH
						vis = vis + 1
					end
				end
			elseif r.Visible then
				cardH = cardH + ROW_H
				vis = vis + 1
			end
		end

		co.card.Visible = vis > 0
		if co.header then
			co.header.Visible = vis > 0
		end

		co.card.Size = UDim2.new(1, 0, 0, cardH)

		if vis > 0 then
			if co.header then
				h = h + HEADER_H
				visCount = visCount + 1
			end

			h = h + cardH
			visCount = visCount + 1
		end
	end

	local extra = page.extraH or 0
	if extra > 0 and visCount > 0 then
		h = h + GAP
	end

	h = h + extra + GAP * math.max(visCount - 1, 0)

	page.frame.Size = UDim2.new(1, 0, 0, h)
	page.contentHeight = h
	page.cachedCanvas = h + Refine.contentPadding.PaddingTop.Offset + Refine.contentPadding.PaddingBottom.Offset
	page.dirty = false

	if page.frame.Parent == Content then
		Content.CanvasSize = UDim2.new(0, 0, 0, page.cachedCanvas)
		Refine.emptyState.Visible = Refine.query ~= nil and Refine.query ~= "" and visCount == 0 and extra == 0
		if Refine.emptyState.Visible then Content.CanvasSize = UDim2.fromOffset(0, 170) end
		Refine.QueueCanvas()
	end
end

function Refine.MakeConsole(frame)
	local box = Refine.New("Frame", {
		Size = UDim2.new(1, 0, 0, 280),
		BackgroundColor3 = C.Card,
	}, frame)
	Refine.Round(box, 10)
	Refine.New("UIStroke", { Color = C.Border, Thickness = 1.2 }, box)

	local sc = Refine.New("ScrollingFrame", {
		Size = UDim2.new(1, 0, 1, 0),
		BackgroundTransparency = 1,
		BorderSizePixel = 0,
		ScrollBarThickness = 4,
		CanvasSize = UDim2.new(0, 0, 0, 8),
	}, box)
	Refine.EnableTouchPan(sc)

	Refine.New("UIListLayout", {
		Padding = UDim.new(0, 2),
		SortOrder = Enum.SortOrder.LayoutOrder,
	}, sc)

	Refine.New("UIPadding", {
		PaddingTop = UDim.new(0, 8),
		PaddingBottom = UDim.new(0, 8),
		PaddingLeft = UDim.new(0, 10),
		PaddingRight = UDim.new(0, 10),
	}, sc)

	local count = 0
	local lines = {}
	local MAX_CONSOLE_LINES = 200

	local function append(msg)
		count = count + 1

		local lab = Refine.New("TextLabel", {
			Size = UDim2.new(1, 0, 0, 18),
			BackgroundTransparency = 1,
			RichText = true,
			Text = msg,
			TextColor3 = C.White,
			Font = Enum.Font.Gotham,
			TextSize = 12,
			TextXAlignment = Enum.TextXAlignment.Left,
			LayoutOrder = count,
		}, sc)

		table.insert(lines, lab)

		if #lines > MAX_CONSOLE_LINES then
			lines[1]:Destroy()
			table.remove(lines, 1)
		end

		sc.CanvasSize = UDim2.new(0, 0, 0, #lines * 20 + 16)
		sc.CanvasPosition = Vector2.new(0, 1e6)
	end

	for _, e in ipairs(logBuffer) do
		append(e[2])
	end

	table.insert(consoleFrames, { append = append, sc = sc })
end

function Refine.MakeRowFrame(card, i, title, desc)
	local row = Refine.New("Frame", {
		Size = UDim2.new(1, 0, 0, ROW_H),
		BackgroundColor3 = C.SurfaceHover,
		BackgroundTransparency = 1,
		LayoutOrder = i,
	}, card)
	if not IS_TOUCH then
		row.MouseEnter:Connect(function()
			if Refine.alive then Refine.Tween(row, EASE, { BackgroundTransparency = 0.72 }):Play() end
		end)
		row.MouseLeave:Connect(function()
			if Refine.alive then Refine.Tween(row, EASE, { BackgroundTransparency = 1 }):Play() end
		end)
	end

	Refine.New("TextLabel", {
		Name = "Title",
		Position = UDim2.new(0, 20, 0, 11),
		Size = UDim2.new(1, -200, 0, 20),
		BackgroundTransparency = 1,
		Text = title,
		TextColor3 = C.White,
		Font = Enum.Font.GothamSemibold,
		TextSize = IS_TOUCH and 14 or 15,
		TextXAlignment = Enum.TextXAlignment.Left,
	}, row)

	Refine.New("TextLabel", {
		Name = "Desc",
		Position = UDim2.new(0, 20, 0, 33),
		Size = UDim2.new(1, -200, 0, 18),
		BackgroundTransparency = 1,
		Text = desc or "",
		TextColor3 = C.Gray,
		Font = Enum.Font.Gotham,
		TextSize = IS_TOUCH and 12 or 13,
		TextXAlignment = Enum.TextXAlignment.Left,
	}, row)

	Refine.rowSearch[row] = string.lower(tostring(title or "") .. " " .. tostring(desc or ""))
	return row
end

function Refine.MakePlayers(frame)
	local card = Refine.New("Frame", {
		Size = UDim2.new(1, 0, 0, ROW_H * 2),
		BackgroundColor3 = C.Card,
		LayoutOrder = 1,
	}, frame)
	Refine.Round(card, 10)
	Refine.New("UIStroke", { Color = C.Border, Thickness = 1.2 }, card)
	Refine.New("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder }, card)

	local rows = {}
	local tracked = {} -- Fictional display rows; no live character references.
	local i = 0

	i = i + 1
	local espRow = Refine.MakeRowFrame(card, i, "Player ESP", "Preview toggle only; no player highlights")
	Refine.AddToggle(espRow, { on = configSession.esp, onChange = function(v) configSession.esp = v end })
	table.insert(rows, espRow)

	i = i + 1
	local refRow = Refine.MakeRowFrame(card, i, "Refresh List", "Reload fictional preview players")
	Refine.AddButton(refRow, {
		buttonText = "Reload",
		logName = "RefreshPlayers",
		onPress = function()
			local old = pages["Players"]
			if old then
				old.frame:Destroy()
			end
			pages["Players"] = nil
			ShowPage("Players")
		end,
	})
	table.insert(rows, refRow)

	local any = false

	for _, pl in ipairs({
		{ Name = "Scout_Aster", Distance = "24m" },
		{ Name = "Scout_River", Distance = "68m" },
		{ Name = "Scout_Nova", Distance = "125m" },
	}) do
		if pl ~= LocalPlayer then
			any = true
			i = i + 1

			local dist = pl.Distance

			local row = Refine.MakeRowFrame(card, i, pl.Name, "Demo distance: " .. dist)
			local BH = IS_TOUCH and 26 or 22

			local b1 = Refine.New("TextButton", {
				AnchorPoint = Vector2.new(1, 0.5),
				Position = UDim2.new(1, -16, 0.5, 0),
				Size = UDim2.new(0, 44, 0, BH),
				BackgroundColor3 = theme.On,
				Text = "SPEC",
				TextColor3 = C.White,
				Font = Enum.Font.GothamSemibold,
				TextSize = 10,
				AutoButtonColor = false,
				ZIndex = 2,
			}, row)
			Refine.Round(b1, 6)
			Refine.Shade(b1)
			Refine.AddTactileFeedback(b1)

			local b2 = Refine.New("TextButton", {
				AnchorPoint = Vector2.new(1, 0.5),
				Position = UDim2.new(1, -66, 0.5, 0),
				Size = UDim2.new(0, 44, 0, BH),
				BackgroundColor3 = C.Search,
				Text = "TP",
				TextColor3 = C.White,
				Font = Enum.Font.GothamSemibold,
				TextSize = 10,
				AutoButtonColor = false,
				ZIndex = 2,
			}, row)
			Refine.Round(b2, 6)
			Refine.Shade(b2)
			Refine.AddTactileFeedback(b2)

			local b3 = Refine.New("TextButton", {
				AnchorPoint = Vector2.new(1, 0.5),
				Position = UDim2.new(1, -116, 0.5, 0),
				Size = UDim2.new(0, 44, 0, BH),
				BackgroundColor3 = C.Search,
				Text = "COPY",
				TextColor3 = C.White,
				Font = Enum.Font.GothamSemibold,
				TextSize = 10,
				AutoButtonColor = false,
				ZIndex = 2,
			}, row)
			Refine.Round(b3, 6)
			Refine.AddTactileFeedback(b3)

			Refine.Click(b1, function()
				configSession.spectating = configSession.spectating ~= pl.Name and pl.Name or nil
				Notify("Spectate Preview", configSession.spectating and ("Selected " .. pl.Name .. " (demo)") or "Preview selection cleared")
				Log("Spectate preview -> " .. tostring(configSession.spectating))
			end)
			Refine.Click(b2, function()
				Notify("Teleport Preview", "Selected " .. pl.Name .. "; UI demo only")
				Log("Teleport preview -> " .. pl.Name)
			end)
			Refine.Click(b3, function()
				Notify("Player Preview", pl.Name .. " (fictional demo player)")
				Log("Player preview name -> " .. pl.Name)
			end)

			table.insert(rows, row)
			table.insert(tracked, { pl = pl, row = row })
		end
	end

	if not any then
		i = i + 1
		table.insert(rows, Refine.MakeRowFrame(card, i, "No other players", "You are alone"))
	end

	card.Size = UDim2.new(1, 0, 0, i * ROW_H)
	return { card = card, rows = rows, tracked = tracked }
end

local profileNameInput = ""
local importJsonInput = ""
local hotkeyDescSetter = nil

function Refine.SyncHotkey()
	for _,row in ipairs(allRows) do
		if row.Parent and row:FindFirstChild("Title") and row.Title.Text == "UI Hotkey" then
			local description = row:FindFirstChild("Desc")
			if description then description.Text = "Current: " .. uiToggleKey.Name end
		end
	end
end
function Refine.CancelKeyCapture()
	if listeningTitle and bindLabels[listeningTitle] then bindLabels[listeningTitle](binds[listeningTitle]) end
	listening, listeningTitle, hotkeyDescSetter = nil, nil, nil
	pickingHotkey, Refine.capturingKey = false, false
	Refine.SyncHotkey()
end

local PAGES_DATA = {
	["Auto Farm"] = {{
		rows = {
			{ kind = "toggle", on = true, title = "Auto Farm", desc = "UI preview only; controls use demo state" },
			{
				kind = "toggle",
				on = false,
				title = "Auto Rejoin",
				desc = "Retry teleport on disconnect",

			},
			{ kind = "slider", min = 0, max = 10, value = 2, title = "Action Delay", desc = "Delay between actions" },
			{
				kind = "accordion",
				title = "Advanced Farming Settings",
				desc = "Targeting, dodging & skills",
				default = false,
				rows = {
					{
						kind = "multidrop",
						title = "Target Enemy Types",
						desc = "Select targets to attack",
						options = { "Normal Titan", "Abnormal Titan", "Armored Titan", "Crawler Titan", "Beast Titan" },
						default = { "Normal Titan", "Abnormal Titan" },
					},
					{
						kind = "drop",
						title = "Attack Style",
						desc = "Weapon combat technique",
						options = { "Slash Strike", "Spinning Whirlwind", "Thunder Spear Barrage" },
					},
					{ kind = "toggle", on = true, title = "Auto Evade Attacks", desc = "Evade incoming swings" },
					{ kind = "slider", min = 1, max = 100, value = 80, title = "Critical Hit Rate", desc = "Target nape threshold" },
				},
			},
		},
	}},

	["Auto Quest"] = {{
		rows = {
			{ kind = "toggle", on = true, title = "Auto Main Quest", desc = "Auto complete story quests" },
			{ kind = "toggle", on = true, title = "Auto Side Quests", desc = "Auto complete side missions" },
		},
	}},

	["Account"] = {{
		header = "Player Profile",
		rows = {
			{
				kind = "button",
				title = "Player: PreviewUser",
				desc = "UserId: 100001 (fictional preview)",
				buttonText = "Copy ID",
				logName = "CopyUserId",
				onPress = function() Notify("UI Preview", "Demo action only; game state unchanged") end,
			},
			{
				kind = "button",
				title = "Reset Character",
				desc = "Instantly respawn character",
				buttonText = "Reset",
				logName = "ResetChar",
				onPress = function() Notify("UI Preview", "Demo action only; game state unchanged") end,
			},
		},
	}},

	["Lobby"] = {
		{
			rows = {
				{ kind = "toggle", on = true, title = "Auto Claim Main Quests", desc = "Claim story rewards" },
				{ kind = "toggle", on = true, title = "Auto Claim Side Quests", desc = "Claim side mission rewards" },
				{ kind = "toggle", on = true, title = "Auto Claim Achievements", desc = "Claim achievement gifts" },
				{ kind = "toggle", on = true, title = "Auto Claim Daily Quests", desc = "Claim daily login rewards" },
			},
		},
		{
			header = "Equipment",
			rows = {
				{ kind = "toggle", on = false, title = "Auto Upgrade Gear ( Blade )", desc = "Auto upgrade weapon level" },
				{ kind = "toggle", on = false, title = "Auto Equip Thunder Spears", desc = "Equip special weapons" },
			},
		},
	},

	["Progression"] = {{
		header = "Progression & Levels",
		rows = {
			{ kind = "toggle", on = true, title = "Auto Rank Up", desc = "Promote rank when requirements met" },
			{ kind = "slider", min = 1, max = 100, value = 1, title = "Target Level", desc = "Level goal limit" },
			{
				kind = "button",
				title = "Prestige / Rebirth",
				desc = "Reset level for permanent buffs",
				buttonText = "Rebirth",
				logName = "Rebirth",
				onPress = function() Notify("UI Preview", "Demo action only; game state unchanged") end,
			},
		},
	}},

	["Skill Trees"] = {{
		header = "Skill Tree Automation",
		rows = {
			{ kind = "toggle", on = true, title = "Auto Unlock Skills", desc = "Unlock available branch skills" },
			{ kind = "slider", min = 1, max = 50, value = 10, title = "Skill Point Reserve", desc = "Keep points saved" },
			{
				kind = "button",
				title = "Reset All Skills",
				desc = "Refund all allocated points",
				buttonText = "Reset",
				logName = "ResetSkills",
				onPress = function() Notify("UI Preview", "Demo action only; game state unchanged") end,
			},
		},
	}},

	["Perks"] = {{
		header = "Perk Customization",
		rows = {
			{
				kind = "multidrop",
				title = "Active Perks",
				desc = "Choose multiple equipped perks",
				options = { "Damage Boost", "Speed Surge", "Titan Slayer", "Iron Defense", "Gas Saver", "Blade Durability", "Critical Strike" },
				default = { "Damage Boost", "Titan Slayer" },
			},
			{
				kind = "accordion",
				title = "Perk Auto-Reroll System",
				desc = "Reroll criteria & filters",
				default = false,
				rows = {
					{ kind = "toggle", on = false, title = "Enable Auto Reroll", desc = "Reroll until target perks found" },
					{
						kind = "drop",
						title = "Target Tier",
						desc = "Minimum rarity required",
						options = { "Common", "Rare", "Epic", "Legendary", "Mythic" },
					},
					{ kind = "slider", min = 1, max = 100, value = 50, title = "Reroll Speed", desc = "Roll delay ms" },
				},
			},
		},
	}},

	["Universal"] = {{
		header = "Universal Utilities (All Games)",
		rows = {
			{
				kind = "toggle",
				on = false,
				title = "Anti-AFK",
				desc = "Prevent idle kick",

			},
			{
				kind = "toggle",
				on = false,
				title = "Infinite Jump",
				desc = "Unlimited mid-air jumps",

			},
			{
				kind = "toggle",
				on = false,
				title = "Noclip",
				desc = "Walk through walls",

			},
			{
				kind = "slider",
				min = 16,
				max = 250,
				value = 16,
				title = "WalkSpeed",
				desc = "Lock movement walkspeed",

			},
			{
				kind = "slider",
				min = 50,
				max = 350,
				value = 50,
				title = "JumpPower",
				desc = "Lock vertical jumppower",

			},
			{
				kind = "button",
				title = "Rejoin",
				desc = "Teleport back to current game",
				buttonText = "Go",
				logName = "Rejoin",
				onPress = function() Notify("UI Preview", "Demo action only; game state unchanged") end,
			},
			{
				kind = "button",
				title = "Server Hop",
				desc = "Hop into a different server",
				buttonText = "Hop",
				logName = "ServerHop",
				onPress = function() Notify("UI Preview", "Demo action only; game state unchanged") end,
			},
			{
				kind = "button",
				title = "Copy Position",
				desc = "Copy your CFrame coordinates",
				buttonText = "Copy",
				logName = "CopyPos",
				onPress = function() Notify("UI Preview", "Demo action only; game state unchanged") end,
			},
		},
	}},

	["Players"] = {{ players = true }},

	["Visuals"] = {{
		header = "Visual Enhancements",
		rows = {
			{ kind = "toggle", on = false, title = "Full Bright", desc = "Max brightness, no shadows" },
			{ kind = "toggle", on = false, title = "No Fog", desc = "Remove atmospheric distance fog" },
			{ kind = "toggle", on = false, title = "FPS Boost", desc = "Unlock FPS and optimize graphics" },
		},
	}},

	["Missions"] = {{
		header = "Mission Configuration",
		rows = {
			{
				kind = "drop",
				title = "Select Mission",
				desc = "Target district",
				default = "Shiganshina",
				options = { "Shiganshina", "Trost District", "Karanes District", "Stohess District" },
			},
			{
				kind = "drop",
				title = "Select Objective",
				desc = "Battle objective",
				default = "Skirmish",
				options = { "Skirmish", "Elimination", "Defense", "Supply Run" },
			},
			{
				kind = "drop",
				title = "Select Difficulty",
				desc = "Mission difficulty",
				options = { "Easy", "Normal", "Hard", "Nightmare" },
			},
			{ kind = "toggle", on = true, title = "Auto Join Mission", desc = "Automatically join selected mission" },
		},
	}},

	["Wave Mode"] = {{
		header = "Wave Defense",
		rows = {
			{ kind = "toggle", on = true, title = "Auto Next Wave", desc = "Auto start incoming waves" },
			{ kind = "slider", min = 1, max = 50, value = 25, title = "Max Wave Limit", desc = "Stop farming at wave" },
			{
				kind = "button",
				title = "Skip Current Wave",
				desc = "Trigger wave clear event",
				buttonText = "Skip",
				logName = "SkipWave",
				onPress = function() Notify("UI Preview", "Demo action only; game state unchanged") end,
			},
		},
	}},

	["Raid Mode"] = {{
		header = "Boss Raid Battles",
		rows = {
			{
				kind = "drop",
				title = "Select Raid Boss",
				desc = "Target boss battle",
				default = "Colossal Titan",
				options = { "Colossal Titan", "Armored Titan", "Beast Titan", "Female Titan", "War Hammer Titan" },
			},
			{ kind = "toggle", on = true, title = "Auto Attack Nape / Weakpoint", desc = "Target boss critical areas" },
			{ kind = "toggle", on = true, title = "Auto Dodge AoE", desc = "Evade boss devastating strikes" },
		},
	}},

	["Webhook"] = {{
		header = "Discord Notifications",
		rows = {
			{
				kind = "input",
				title = "Webhook URL",
				desc = "Demo endpoint field; nothing is transmitted",
				placeholder = "https://discord.com/api/...",
				value = "",
			},
			{
				kind = "button",
				title = "Send Test",
				desc = "Preview notification locally",
				buttonText = "Send",
				logName = "WebhookTest",
				onPress = function() Notify("UI Preview", "Demo action only; game state unchanged") end,
			},
		},
	}},

	["Console"] = {{ console = true }},

	["Settings"] = {
		{
			header = "Appearance & Interface",
			rows = {
				{
					kind = "drop",
					title = "Theme Preset",
					desc = "Select accent color palette",
					default = "Emerald",
					options = { "Emerald", "Crimson", "Ocean", "Violet", "Sunset", "Midnight", "Cyberpunk", "Sakura" },
					onSelect = function(o)
						ApplyTheme(o)
					end,
				},
				{
					kind = "toggle",
					on = true,
					title = "UI Sounds",
					desc = "Play click feedback sound",
					onChange = function(v)
						soundEnabled = v
					end,
				},
				{
					kind = "button",
					title = "UI Hotkey",
					desc = "Click, then press any key",
					buttonText = "Set",
					logName = "SetHotkey",
					onPress = function(_, setDesc)
						hotkeyDescSetter = setDesc
						pickingHotkey = true
						Notify("UI Hotkey", "Press any key to assign")
					end,
				},
				{
					kind = "toggle",
					on = true,
					title = "Auto-Save Settings",
					desc = "Save changes during this UI session",
					onChange = function(v)
						autoSaveConfig = v
					end,
				},
			},
		},
		{
			header = "Profile Management",
			rows = {
				{
					kind = "drop",
					title = "Active Profile",
					desc = "Switch active configuration profile",
					default = "Default",
					options = availableProfiles,
					onSelect = function(selectedProfile)
						activeProfile = selectedProfile
						liveDefaults["Active Profile"] = selectedProfile
						LoadConfig(selectedProfile, false)
					end,
				},
				{
					kind = "button",
					title = "Save Current Profile",
					desc = "Write settings to active profile",
					buttonText = "Save",
					logName = "SaveProfile",
					onPress = function()
						SaveConfig(false)
					end,
				},
				{
					kind = "button",
					title = "Reload Profile",
					desc = "Reload settings from active profile",
					buttonText = "Reload",
					logName = "LoadProfile",
					onPress = function()
						LoadConfig(activeProfile, false)
					end,
				},
				{
					kind = "accordion",
					title = "Create / Delete Profile",
					desc = "Add custom profiles",
					default = false,
					rows = {
						{
							kind = "input",
							title = "New Profile Name",
							desc = "Enter name for new profile",
							placeholder = "e.g. MyCustom_PVP",
							onChanged = function(text)
								profileNameInput = text
							end,
						},
						{
							kind = "button",
							title = "Create Profile",
							desc = "Add and switch to new profile",
							buttonText = "Create",
							logName = "CreateProfile",
							onPress = function()
								local newName = Refine.SanitizeProfileName(profileNameInput)

								if newName and not table.find(availableProfiles, newName) then
									table.insert(availableProfiles, newName)
									activeProfile = newName
									liveDefaults["Active Profile"] = newName
									SaveConfig(false)

									Notify("Profile Created", "Profile: " .. newName, nil, "success")

									local old = pages["Settings"]
									if old then
										old.frame:Destroy()
									end

									pages["Settings"] = nil
									ShowPage("Settings")
								else
									Notify("Profile Error", "Invalid or existing profile name", nil, "error")
								end
							end,
						},
						{
							kind = "button",
							title = "Delete Active Profile",
							desc = "Remove current session profile",
							buttonText = "Delete",
							logName = "DeleteProfile",
							onPress = function()
								if activeProfile ~= "Default" then
									configSession.profiles[activeProfile] = nil

									for idx, p in ipairs(availableProfiles) do
										if p == activeProfile then
											table.remove(availableProfiles, idx)
											break
										end
									end

									activeProfile = "Default"
									liveDefaults["Active Profile"] = "Default"
									LoadConfig("Default", false)

									Notify("Profile Deleted", "Switched to Default profile")

									local old = pages["Settings"]
									if old then
										old.frame:Destroy()
									end

									pages["Settings"] = nil
									ShowPage("Settings")
								else
									Notify("Profile Error", "Cannot delete Default profile", nil, "error")
								end
							end,
						},
					},
				},
			},
		},
		{
			header = "Share & Backup",
			rows = {
				{
					kind = "button",
					title = "Export to Clipboard",
					desc = "Open JSON text for manual copy and sharing",
					buttonText = "Export",
					logName = "ExportJSON",
					onPress = function()
						ExportConfigToClipboard()
					end,
				},
				{
					kind = "accordion",
					title = "Import Configuration JSON",
					desc = "Paste JSON code to load",
					default = false,
					rows = {
						{
							kind = "input",
							title = "Paste JSON String",
							desc = "Enter valid LurUI JSON string",
							placeholder = '{"values":{...}}',
							onChanged = function(text)
								importJsonInput = text
							end,
						},
						{
							kind = "button",
							title = "Apply Imported JSON",
							desc = "Overwrite current settings with JSON",
							buttonText = "Apply",
							logName = "ApplyJSON",
							onPress = function()
								ImportConfigFromString(importJsonInput)
							end,
						},
					},
				},
				{
					kind = "button",
					title = "Factory Reset",
					desc = "Reset all settings to original defaults",
					buttonText = "Reset All",
					logName = "FactoryReset",
					onPress = function()
						ResetToDefault()
					end,
				},
			},
		},
	},
}

Refine.IndexConfigSchema(PAGES_DATA)
local buildLock = {}

BuildPage = function(name)
	local spec = PAGES_DATA[name]
	if not spec or not Refine.alive then return nil end

	if pages[name] then
		return pages[name]
	end

	while buildLock[name] do
		task.wait()
		if not Refine.alive then return nil end
	end

	if pages[name] then
		return pages[name]
	end

	buildLock[name] = true

	local stagingFrame
	local ok, page = pcall(function()
		local frame = Refine.New("Frame", {
			Size = UDim2.new(1, 0, 0, 100),
			BackgroundTransparency = 1,
			Visible = false,
			Parent = PageCache,
		})
		stagingFrame = frame

		local pageLayout = Refine.New("UIListLayout", {
			Name = "PageLayout",
			Padding = UDim.new(0, GAP),
			SortOrder = Enum.SortOrder.LayoutOrder,
		}, frame)

		local pageObj = {
			frame = frame,
			layout = pageLayout,
			cards = {},
			extraH = 0,
			name = name,
			dirty = true,
		}
		pageLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(Refine.QueueCanvas)

		for blockIndex, block in ipairs(spec) do
			if block.console then
				Refine.MakeConsole(frame)
				pageObj.extraH = 280
			elseif block.players then
				local mp = Refine.MakePlayers(frame)
				pageObj.playerRows = mp.tracked
				for _,row in ipairs(mp.rows) do table.insert(allRows, row) end
				table.insert(pageObj.cards, { card = mp.card, header = nil, rows = mp.rows })
			else
				local header

				if block.header then
					header = Refine.New("TextLabel", {
						Size = UDim2.new(1, 0, 0, HEADER_H),
						BackgroundTransparency = 1,
						Text = block.header,
						TextColor3 = C.White,
						Font = Enum.Font.GothamSemibold,
						TextSize = 16,
						TextXAlignment = Enum.TextXAlignment.Left,
						LayoutOrder = blockIndex*2-1,
					}, frame)
				end

				local card = Refine.New("Frame", {
					Size = UDim2.new(1, 0, 0, #block.rows * ROW_H),
					BackgroundColor3 = C.Card,
					LayoutOrder = blockIndex*2,
				}, frame)
				Refine.Round(card, 10)
				Refine.New("UIStroke", { Color = C.Border, Thickness = 1.2 }, card)
				Refine.New("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder }, card)

				local rows = {}

				for i, d in ipairs(block.rows) do
					local kind = d.kind or "toggle"

					if kind == "accordion" then
						local accObj = Refine.AddAccordion(card, i, d)
						table.insert(rows, accObj)
					else
						local row = Refine.MakeRowFrame(card, i, d.title, d.desc)

						if i < #block.rows then
							Refine.New("Frame", {
								Position = UDim2.new(0, 18, 1, 0),
								Size = UDim2.new(1, -36, 0, 1),
								BackgroundColor3 = C.Border,
								BorderSizePixel = 0,
							}, row)
						end

						if kind == "toggle" then
							Refine.AddToggle(row, d)
						elseif kind == "slider" then
							Refine.AddSlider(row, d)
						elseif kind == "button" then
							Refine.AddButton(row, d)
						elseif kind == "drop" then
							Refine.AddDropdown(row, d)
						elseif kind == "multidrop" then
							Refine.AddMultiDropdown(row, d)
						elseif kind == "input" then
							Refine.AddInput(row, d)
						end

						table.insert(rows, row)
						table.insert(allRows, row)
					end

					if i % 3 == 0 then
						task.wait()
						if not Refine.alive then return nil end
					end
				end

				table.insert(pageObj.cards, { card = card, header = header, rows = rows })
			end

			if blockIndex % 2 == 0 then
				task.wait()
				if not Refine.alive then return nil end
			end
		end

		return pageObj
	end)

	buildLock[name] = nil

	if not Refine.alive then return nil end
	if ok and page then
		pages[name] = page
		Refine.ApplyRows()
		if Refine.ApplySearch then Refine.ApplySearch(false) end
		if name == "Settings" then Refine.SyncHotkey() end
		return page
	end

	if stagingFrame and stagingFrame.Parent then stagingFrame:Destroy() end
	warn("[LurUI] BuildPage failed for " .. tostring(name) .. ": " .. tostring(page))
	return nil
end

local activeSwitchToken = 0

function Refine.FinishShowPage(page)
	if not Refine.alive or not page or not page.frame then return end

	Refine.EndSlider(); Refine.ClosePopup()
	if currentPage and currentPage ~= page and currentPage.frame.Parent == Content then
		currentPage.scrollY = Content.CanvasPosition.Y
	end

	for _, p in pairs(pages) do
		if p ~= page and p.frame then
			p.frame.Visible = false
			p.frame.Parent = PageCache
		end
	end

	currentPage = page
	page.frame.Parent = Content
	page.frame.Visible = true
	page.frame.Position = UDim2.fromOffset(28, 0)

	Refine.Tween(page.frame, SPRING, { Position = UDim2.fromOffset(0, 0) }):Play()

	if page.dirty or not page.cachedCanvas then
		Recompute(page)
	else
		Content.CanvasSize = UDim2.new(0, 0, 0, page.cachedCanvas)
	end

	Content.CanvasPosition = Vector2.new(0, math.clamp(page.scrollY or 0, 0, math.max(0, page.cachedCanvas - Content.AbsoluteWindowSize.Y)))
	if Refine.ApplySearch then Refine.ApplySearch(false) end
	Refine.QueueCanvas()
	if Refine.navigation then Refine.FocusFirst(page.frame) end

end

ShowPage = function(name)
	if not Refine.alive or not PAGES_DATA[name] then return end

	currentActiveMenu = name
	hoveredMenu = nil
	Refine.UpdateMenuVisuals()

	activeSwitchToken = activeSwitchToken + 1
	local token = activeSwitchToken

	local page = pages[name]
	if page then
		Refine.FinishShowPage(page)
		return
	end

	task.spawn(function()
		local built = BuildPage(name)

		if not Refine.alive or token ~= activeSwitchToken then
			return
		end

		if built then
			Refine.FinishShowPage(built)
		end
	end)
end

local IW = SIDEBAR_W - 30
local sideOrder = 0

function Refine.Section(title, items)
	sideOrder = sideOrder + 1

	local head = Refine.New("TextButton", {
		Size = UDim2.new(0, IW, 0, 30),
		BackgroundTransparency = 1,
		AutoButtonColor = false,
		Text = "",
		LayoutOrder = sideOrder,
	}, SideScroll)

	local tLab = Refine.New("TextLabel", {
		Position = UDim2.new(0, 8, 0, 0),
		Size = UDim2.new(1, -28, 1, 0),
		BackgroundTransparency = 1,
		Text = title,
		TextColor3 = C.Gray,
		Font = Enum.Font.GothamSemibold,
		TextSize = 13,
		TextXAlignment = Enum.TextXAlignment.Left,
	}, head)

	local chev = Refine.Chevron(head, UDim2.new(0, IW - 14, 0, 15))

	head.MouseEnter:Connect(function() tLab.TextColor3 = C.White end)
	head.MouseLeave:Connect(function() tLab.TextColor3 = C.Gray end)

	local itemBtns = {}
	local sectionState = { head = head, buttons = itemBtns, chev = chev, collapsed = false }
	table.insert(sectionRegistry, sectionState)

	for _, it in ipairs(items) do
		sideOrder = sideOrder + 1

		local btn = Refine.New("TextButton", {
			Name = "Menu_" .. it[2],
			Size = UDim2.new(0, IW, 0, IS_TOUCH and 42 or 38),
			BackgroundTransparency = 1,
			AutoButtonColor = false,
			Text = "",
			LayoutOrder = sideOrder,
		}, SideScroll)

		local hover = Refine.New("Frame", {
			Size = UDim2.new(1, 0, 1, 0),
			BackgroundColor3 = C.White,
			BackgroundTransparency = 1,
			ZIndex = 1,
			Visible = true,
		}, btn)
		Refine.Round(hover, 8)

		local sel = Refine.New("Frame", {
			Size = UDim2.new(1, 0, 1, 0),
			BackgroundColor3 = theme.Accent,
			BackgroundTransparency = 1,
			ZIndex = 1,
			Visible = true,
		}, btn)
		Refine.Round(sel, 8)
		Refine.Shade(sel)

		HookTheme(function(t)
			sel.BackgroundColor3 = t.Accent
		end, btn)

		local glow = Refine.New("Frame", {
			AnchorPoint = Vector2.new(0, 0.5),
			Position = UDim2.new(0, 2, 0.5, 0),
			Size = UDim2.new(0, 3, 0, 0),
			BackgroundColor3 = C.White,
			BackgroundTransparency = 1,
			BorderSizePixel = 0,
			ZIndex = 3,
		}, btn)
		Refine.Round(glow, 2)

		local paint = Refine.DrawIcon(it[1], btn, 14)

		local lab = Refine.New("TextLabel", {
			Position = UDim2.new(0, 38, 0, 0),
			Size = UDim2.new(1, -44, 1, 0),
			BackgroundTransparency = 1,
			Text = it[2],
			TextColor3 = C.Gray,
			Font = Enum.Font.GothamMedium,
			TextSize = 14,
			TextXAlignment = Enum.TextXAlignment.Left,
			ZIndex = 2,
		}, btn)

		btn.MouseEnter:Connect(function()
			if not IS_TOUCH then
				hoveredMenu = it[2]
				Refine.UpdateMenuVisuals()
				if not pages[it[2]] and not buildLock[it[2]] then
					task.defer(function()
						if Refine.alive and not pages[it[2]] then BuildPage(it[2]) end
					end)
				end
			end
		end)

		btn.MouseLeave:Connect(function()
			if not IS_TOUCH then
				if hoveredMenu == it[2] then
					hoveredMenu = nil
				end
				Refine.UpdateMenuVisuals()
			end
		end)

		Refine.Click(btn, function()
			ShowPage(it[2])
			if Refine.overlay then
				SetSidebar(false)
			end
		end)

		table.insert(itemBtns, btn)
		table.insert(menuRegistry, {
			name = it[2],
			button = btn,
			sel = sel,
			hover = hover,
			glow = glow,
			label = lab,
			paint = paint,
			section = sectionState,
		})
	end

	Refine.UpdateMenuVisuals()

	Refine.Click(head, function()
		if SearchBox.Text ~= "" then return end
		sectionState.collapsed = not sectionState.collapsed

		for _, b in ipairs(itemBtns) do
			b.Visible = not sectionState.collapsed
		end

		Refine.Tween(chev, EASE, { Rotation = sectionState.collapsed and -90 or 0 }):Play()
		if Refine.ApplySearch and SearchBox.Text ~= "" then Refine.ApplySearch(false) end
	end)
end

-- Keep the navigation drawer recoverable and visible whenever explicitly requested.
local function EnsureSidebarVisible()
	if not Refine.alive then return end
	if Refine.overlay and not sidebarOpen then
		SetSidebar(true)
	end
end

Refine.EnsureSidebarVisible = EnsureSidebarVisible

Refine.Section("In-Game", { { "bolt", "Auto Farm" }, { "quest", "Auto Quest" } })
Refine.Section("Lobby", {
	{ "user", "Account" },
	{ "layers", "Lobby" },
	{ "chart", "Progression" },
	{ "book", "Skill Trees" },
	{ "spark", "Perks" },
})
Refine.Section("Universal", {
	{ "target", "Universal" },
	{ "user", "Players" },
	{ "spark", "Visuals" },
})
Refine.Section("Game Mode", {
	{ "quest", "Missions" },
	{ "wave", "Wave Mode" },
	{ "skull", "Raid Mode" },
})
Refine.Section("Misc", {
	{ "bell", "Webhook" },
	{ "console", "Console" },
	{ "gear", "Settings" },
})

function Refine.RowMatches(row, q)
	if q == "" then return true end
	local indexed = Refine.rowSearch[row]
	if indexed and indexed:find(q, 1, true) then return true end
	local titleLab = row:FindFirstChild("Title")
	local descLab = row:FindFirstChild("Desc")
	return (titleLab and titleLab.Text:lower():find(q, 1, true) ~= nil)
		or (descLab and descLab.Text:lower():find(q, 1, true) ~= nil)
end

function Refine.CountSpecMatches(name, q)
	if q == "" then return 0 end
	local blocks = PAGES_DATA[name]
	if not blocks then return 0 end
	local pageMatch = name:lower():find(q, 1, true) ~= nil
	local matches = 0
	for _, block in ipairs(blocks) do
		if block.console then
			local special = "console logs logging output events realtime real time"
			if pageMatch or special:find(q, 1, true) then matches += 1 end
		elseif block.players then
			local special = "player players esp refresh list scout spectate teleport copy preview distance"
			if pageMatch or special:find(q, 1, true) then matches += 1 end
		else
			local headerMatch = block.header and block.header:lower():find(q, 1, true) ~= nil
			for _, spec in ipairs(block.rows or {}) do
				if spec.kind == "accordion" then
					local groupText = ((spec.title or "") .. " " .. (spec.desc or "")):lower()
					local groupMatch = pageMatch or headerMatch or groupText:find(q, 1, true) ~= nil
					for _, child in ipairs(spec.rows or {}) do
						local text = ((child.title or "") .. " " .. (child.desc or "")):lower()
						if groupMatch or text:find(q, 1, true) then matches += 1 end
					end
				else
					local text = ((spec.title or "") .. " " .. (spec.desc or "")):lower()
					if pageMatch or headerMatch or text:find(q, 1, true) then matches += 1 end
				end
			end
		end
	end
	return matches
end

function Refine.ApplySearch(resetScroll)
	local q = SearchBox.Text:lower():match("^%s*(.-)%s*$")
	Refine.query = q

	for name, p in pairs(pages) do
		local pageMatch = q ~= "" and name:lower():find(q, 1, true) ~= nil
		p.matches = 0

		for _, card in ipairs(p.cards) do
			local headerText = card.header and card.header.Text:lower() or ""
			local contextMatch = pageMatch or (q ~= "" and headerText:find(q, 1, true) ~= nil)

			for _, entry in ipairs(card.rows) do
				if type(entry) == "table" and entry.IsAccordion then
					local groupMatch = contextMatch or (q ~= "" and entry.SearchText:find(q, 1, true) ~= nil)
					local matches = 0
					for _, row in ipairs(entry.Rows) do
						row.Visible = (q == "") or groupMatch or Refine.RowMatches(row, q)
						if row.Visible then matches += 1 end
					end
					entry.SetSearchOpen(q ~= "" and matches > 0)
					p.matches += matches
				else
					entry.Visible = (q == "") or contextMatch or Refine.RowMatches(entry, q)
					if entry.Visible then p.matches += 1 end
				end
			end
		end

		if p.extraH and p.extraH > 0 and pageMatch then
			p.matches = math.max(1, p.matches)
		end
		p.dirty = true
	end

	for _, entry in ipairs(menuRegistry) do
		local page = pages[entry.name]
		local staticCount = q ~= "" and Refine.CountSpecMatches(entry.name, q) or 0
		local count = q ~= "" and math.max(page and page.matches or 0, staticCount) or 0
		local searchVisible = q == "" or count > 0
		entry.searchVisible = searchVisible
		entry.label.Text = entry.name .. (q ~= "" and (" · " .. count) or "")
		entry.label.TextTruncate = Enum.TextTruncate.AtEnd
		entry.button.Visible = searchVisible and (q ~= "" or not (entry.section and entry.section.collapsed))
	end

	for _, section in ipairs(sectionRegistry) do
		local anyVisible = false
		for _, entry in ipairs(menuRegistry) do
			if entry.section == section and entry.searchVisible then anyVisible = true; break end
		end
		section.head.Visible = q == "" or anyVisible
		if section.chev then section.chev.Rotation = (q ~= "" or not section.collapsed) and 0 or -90 end
	end

	if currentPage then
		Recompute(currentPage)
		if resetScroll then
			currentPage.scrollY = 0
			Content.CanvasPosition = Vector2.new(0, 0)
		end
	end
end

SearchBox.FocusLost:Connect(function(enterPressed)
	if not enterPressed or SearchBox.Text == "" then return end
	local target
	for _, entry in ipairs(menuRegistry) do
		if entry.searchVisible and entry.name == currentActiveMenu then target = entry; break end
	end
	if not target then
		for _, entry in ipairs(menuRegistry) do
			if entry.searchVisible then target = entry; break end
		end
	end
	if target and target.name ~= currentActiveMenu then ShowPage(target.name) end
end)

Refine.searchGeneration = 0
SearchBox:GetPropertyChangedSignal("Text"):Connect(function()
	searchClearBtn.Visible = SearchBox.Text ~= ""
	Refine.searchGeneration += 1
	local generation = Refine.searchGeneration
	task.delay(0.12, function()
		if not Refine.alive or generation ~= Refine.searchGeneration then return end
		Refine.ApplySearch(true)
	end)
end)

Refine.Connect(UserInput.InputBegan, function(inp, gpe)
	if Refine.systemMenuOpen then return end
	if inp.KeyCode == Enum.KeyCode.Escape and (pickingHotkey or listening) then
		Refine.CancelKeyCapture()
		return
	end
	if pickingHotkey then
		if inp.UserInputType == Enum.UserInputType.Keyboard
			and inp.KeyCode ~= Enum.KeyCode.Unknown
			and not gpe then
			local keyName = inp.KeyCode.Name
			local cleared
			for title, key in pairs(binds) do
				if key == keyName then
					binds[title] = nil
					if bindLabels[title] then bindLabels[title]("") end
					cleared = title
				end
			end
			uiToggleKey = inp.KeyCode

			if hotkeyDescSetter then
				hotkeyDescSetter("Current: " .. uiToggleKey.Name)
			end

			local detail = "Bound to " .. uiToggleKey.Name
			if cleared then detail ..= " · cleared conflicting bind: " .. cleared end
			Notify("UI Hotkey", detail, nil, "success")
			Log("UI hotkey -> " .. uiToggleKey.Name, "OK")
			TriggerAutoSave()
		end

		hotkeyDescSetter = nil
		pickingHotkey = false
		return
	end

	if listening then
		if gpe then
			if listeningTitle and bindLabels[listeningTitle] then
				bindLabels[listeningTitle](binds[listeningTitle])
			end

			listening = nil
			listeningTitle = nil
			Refine.capturingKey = false
			return
		end

		if inp.UserInputType == Enum.UserInputType.Keyboard and inp.KeyCode ~= Enum.KeyCode.Unknown then
			local k = inp.KeyCode.Name

			if inp.KeyCode == uiToggleKey then
				if bindLabels[listeningTitle] then bindLabels[listeningTitle](binds[listeningTitle]) end
				Notify("Keybind", k .. " is reserved for the UI hotkey", nil, "warning")
				Log("Rejected conflicting bind " .. listeningTitle .. " -> " .. k, "WARN")
			else
				for t, key in pairs(binds) do
					if key == k then
						binds[t] = nil
						if bindLabels[t] then bindLabels[t]("") end
					end
				end

				binds[listeningTitle] = k
				if bindLabels[listeningTitle] then bindLabels[listeningTitle](k) end

				Notify("Keybind", listeningTitle .. " = " .. k)
				Log("Bind " .. listeningTitle .. " -> " .. k)
				TriggerAutoSave()
			end
		elseif inp.UserInputType == Enum.UserInputType.Touch
			or inp.UserInputType == Enum.UserInputType.MouseButton1 then
			bindCancelTs = os.clock()

			if bindLabels[listeningTitle] then
				bindLabels[listeningTitle](binds[listeningTitle])
			end
		end

		listening = nil
		listeningTitle = nil
		Refine.capturingKey = false
		return
	end

	if gpe or UserInput:GetFocusedTextBox() then return end

	if inp.UserInputType == Enum.UserInputType.Keyboard then
		local k = inp.KeyCode.Name

		for t, key in pairs(binds) do
			if key == k and toggleByTitle[t] then
				toggleByTitle[t](nil)
			end
		end
	end
end)

do
	local function ownsSelection()
		local selected = Refine.guiService.SelectedObject
		return selected and selected:IsDescendantOf(Gui)
	end
	Gui:GetPropertyChangedSignal("Enabled"):Connect(function()
		if not Gui.Enabled then
			Refine.CancelKeyCapture()
			local textBox = UserInput:GetFocusedTextBox()
			if textBox and textBox:IsDescendantOf(Gui) then textBox:ReleaseFocus() end
			if ownsSelection() then Refine.guiService.SelectedObject = nil end
		elseif Refine.navigation then Refine.RestoreFocus(panelBtn) end
	end)
	Refine.Connect(UserInput.LastInputTypeChanged, function(inputType)
		if inputType.Name:match("^Gamepad") then
			Refine.navigation = true
		elseif inputType == Enum.UserInputType.MouseButton1 or inputType == Enum.UserInputType.Touch then
			Refine.navigation = false
			if ownsSelection() then Refine.guiService.SelectedObject = nil end
		end
	end)
	Refine.Connect(UserInput.InputBegan, function(input, processed)
		if Refine.systemMenuOpen then return end
		local key = input.KeyCode
		local textBox = UserInput:GetFocusedTextBox()
		if key == Enum.KeyCode.ButtonSelect and not processed and not textBox then
			Refine.navigation = true
			if Gui.Enabled and ownsSelection() then Gui.Enabled = false
			else Gui.Enabled = true; SetMinimized(false); Refine.RestoreFocus(panelBtn) end
			return
		end
		if not Gui.Enabled then return end
		if key == Enum.KeyCode.Escape or (key == Enum.KeyCode.ButtonB and ownsSelection()) then
			if Refine.popup then Refine.ClosePopup()
			elseif textBox and textBox:IsDescendantOf(Gui) then textBox:ReleaseFocus()
			elseif SearchBox.Text ~= "" then SearchBox.Text = ""
			elseif Refine.overlay and sidebarOpen then SetSidebar(false)
			elseif ownsSelection() then Refine.guiService.SelectedObject = nil end
			return
		end
		if processed or pickingHotkey or listening then return end
		if key == Enum.KeyCode.F and (UserInput:IsKeyDown(Enum.KeyCode.LeftControl) or UserInput:IsKeyDown(Enum.KeyCode.RightControl)) then
			Refine.ClosePopup(); SetMinimized(false); SearchBox:CaptureFocus()
			return
		end
		if textBox then return end
		if key == Enum.KeyCode.Tab then
			Refine.navigation = true
			local root = Gui:FindFirstChild("LurDropdownList") or Gui:FindFirstChild("LurConfigExport") or Window
			local choices = {}
			for _,object in ipairs(root:GetDescendants()) do
				if object:IsA("GuiObject") and object.Selectable and Refine.IsVisible(object) then table.insert(choices, object) end
			end
			if #choices > 0 then
				local index = table.find(choices, Refine.guiService.SelectedObject) or 0
				local backwards = UserInput:IsKeyDown(Enum.KeyCode.LeftShift) or UserInput:IsKeyDown(Enum.KeyCode.RightShift)
				Refine.guiService.SelectedObject = choices[(index - 1 + (backwards and -1 or 1)) % #choices + 1]
			end
		elseif Refine.navigation and (key == Enum.KeyCode.Return or key == Enum.KeyCode.Space) and ownsSelection() then
			local selected = Refine.guiService.SelectedObject
			if selected:IsA("TextBox") then selected:CaptureFocus()
			elseif Refine.actions[selected] then Refine.actions[selected](input) end
		end
	end)
end

do
	local function menuOpened()
		if Refine.systemMenuOpen then return end
		Refine.systemMenuOpen, Refine.menuRestore = true, Gui.Enabled
		dragging, resizing = false, false
		if Refine.StopBubbleDrag then Refine.StopBubbleDrag() end
		Refine.EndSlider(); Refine.ClosePopup()
		Gui.Enabled = false
		if Refine.bubbleGui then Refine.bubbleGui.Enabled = false end
	end
	local function menuClosed()
		if not Refine.systemMenuOpen then return end
		Refine.systemMenuOpen = false
		Gui.Enabled = Refine.menuRestore == true
		if Refine.bubbleGui then Refine.bubbleGui.Enabled = IS_TOUCH or not Gui.Enabled end
	end
	Refine.Connect(Refine.guiService.MenuOpened, menuOpened)
	Refine.Connect(Refine.guiService.MenuClosed, menuClosed)
	if Refine.guiService.MenuIsOpen then menuOpened() end
end

ShowPage("Lobby")

if type(savedBounds) == "table" then
	local bw = tonumber(savedBounds.w)
	local bh = tonumber(savedBounds.h)

	if bw and bh then
		curW = math.clamp(bw,math.min(320,vp.X-12),vp.X-12)
		curH = math.clamp(bh,math.min(240,vp.Y-TOP_GAP-12),vp.Y-TOP_GAP-12)

		local bx = tonumber(savedBounds.x) or (vp.X - curW) / 2
		local by = tonumber(savedBounds.y) or (TOP_GAP + (vp.Y - TOP_GAP - curH) / 2)

		Window.Size = UDim2.new(0, curW, 0, curH)
		local minX, maxX, minY, maxY = GetDragBounds()
		Window.Position = UDim2.fromOffset(
			math.clamp(bx, minX, maxX),
			math.clamp(by, minY, maxY)
		)

		SyncShadow()
		UpdateDragRegion()
	end
end

task.delay(1.5, function()
	if not Refine.alive then return end
	-- Keep startup memory low. Frequently used utility pages are warmed while the rest remain lazy.
	for _, name in ipairs({ "Settings", "Console" }) do
		if not Refine.alive then return end
		if not pages[name] and not buildLock[name] then BuildPage(name) end
		task.wait()
	end
end)

Log("Lur UI v" .. VERSION .. " initialized (refined UI preview; gameplay controls remain demos)", "OK")
print("[LurUI] v" .. VERSION .. " loaded successfully. Gameplay controls remain UI previews.")
Notify("Lur UI v" .. VERSION, "Refined interface ready. Gameplay controls are demonstrations only.", 4, "success")

Refine.FitWindow()
Refine.ApplyRows()
return Lur
