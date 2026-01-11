-- ══════════════════════════════════════
--               Core
-- ══════════════════════════════════════
local Find = function(Table)
	for _, Item in pairs(Table or {}) do
		if typeof(Item) == "table" then
			return Item
		end
	end
end

local Options = Find(({...})) or {
	Keybind = "Home",

	Language = {
		UI = "pt-br",
		Words = "pt-br"
	},

	Experiments = {},

	Tempo = 0.5, -- FIXO EM 0.5s
	Rainbow = false,
}

local Version = "2.1"
local Parent = gethui() or game:GetService("CoreGui")

local require = function(Name)
	return loadstring(game:HttpGet(
		string.format("https://raw.githubusercontent.com/Zv-yz/AutoJJs/main/%s.lua", Name)
	))()
end

-- ══════════════════════════════════════
--              Services
-- ══════════════════════════════════════
local TweenService = game:GetService("TweenService")
local Players = game:GetService("Players")
local LP = Players.LocalPlayer

-- ══════════════════════════════════════
--              Modules
-- ══════════════════════════════════════
local UI = require("UI")
local Notification = require("Notification")

local Extenso = require("Modules/Extenso")
local Character = require("Modules/Character")
local RemoteChat = require("Modules/RemoteChat")

-- ══════════════════════════════════════
--        Chat Queue Controller
-- ══════════════════════════════════════
local ChatQueue = {}
local Sending = false
local CHAT_INTERVAL = 0.5

local function ProcessQueue()
	if Sending then return end
	Sending = true

	task.spawn(function()
		while #ChatQueue > 0 do
			local msg = table.remove(ChatQueue, 1)

			pcall(function()
				RemoteChat:Send(msg)
			end)

			local start = os.clock()
			while os.clock() - start < CHAT_INTERVAL do
				task.wait()
			end
		end

		Sending = false
	end)
end

local function SafeSend(message)
	table.insert(ChatQueue, message)
	ProcessQueue()
end

-- ══════════════════════════════════════
--              Constants
-- ══════════════════════════════════════
local Char = Character.new(LP)
local UIElements = UI.UIElements
local Connections = {}

local Threading
local FinishedThread = false
local Toggled = false

local Settings = {
	Started = false,
	Jump = false,

	Config = {
		Start = nil,
		End = nil,
		Prefix = nil,
	}
}

-- ══════════════════════════════════════
--              Methods
-- ══════════════════════════════════════
local Methods = {

	["Normal"] = function(Message, Prefix)
		if Settings.Jump then Char:Jump() end
		SafeSend(string.format("%s%s", Message, Prefix))
	end,

	["Lowercase"] = function(Message, Prefix)
		if Settings.Jump then Char:Jump() end
		SafeSend(string.format("%s%s", string.lower(Message), Prefix))
	end,

	["HJ"] = function(Message, Prefix)
		for i = 1, #Message do
			if Settings.Jump then Char:Jump() end
			SafeSend(string.format("%s%s", string.sub(Message, i, i), Prefix))
		end

		if Settings.Jump then Char:Jump() end
		SafeSend(string.format("%s%s", Message, Prefix))
	end,
}

-- ══════════════════════════════════════
--              Functions
-- ══════════════════════════════════════
local function Listen(Name, Element)
	if Element:GetAttribute("IntBox") then
		table.insert(Connections,
			Element:GetPropertyChangedSignal("Text"):Connect(function()
				Element.Text = string.gsub(Element.Text, "[^%d]", "")
			end)
		)
	end

	table.insert(Connections,
		Element.FocusLost:Connect(function()
			if not Element.Text or string.match(Element.Text, "^%s*$") then return end
			Settings.Config[Name] = tonumber(Element.Text)
		end)
	)
end

local function EndThread(Success)
	if Threading then
		if not FinishedThread then
			task.cancel(Threading)
		end

		Threading = nil
		FinishedThread = false
		Settings.Started = false

		Notification:Notify(Success and 6 or 12)
	end
end

local function DoJJ(MethodName, Number, Prefix)
	local Success, String = Extenso:Convert(Number)
	if not Success then return end

	local Method = Methods[MethodName]
	if not Method then return end

	Method(String, Prefix or "")
end

local function StartThread()
	local Config = Settings.Config
	if not Config.Start or not Config.End then return end
	if Threading then EndThread(false) return end

	local Method =
		table.find(Options.Experiments, "hell_jacks_2024_02-dev") and "HJ" or
		table.find(Options.Experiments, "lowercase_jjs_2024_12") and "Lowercase" or
		"Normal"

	Threading = task.spawn(function()
		for i = Config.Start, Config.End do
			DoJJ(Method, i, Config.Prefix)
		end

		FinishedThread = true
		EndThread(true)
	end)
end

-- ══════════════════════════════════════
--                Main
-- ══════════════════════════════════════
UI:SetVersion(Version)
UI:SetRainbow(Options.Rainbow)
UI:SetParent(Parent)

Notification:SetParent(UI.getUI())

for Name, Element in pairs(UIElements.Box) do
	task.spawn(Listen, Name, Element)
end

table.insert(Connections, UIElements.Play.MouseButton1Up:Connect(function()
	if not Settings.Config.Start or not Settings.Config.End then return end

	if not Settings.Started then
		Settings.Started = true
		StartThread()
	else
		EndThread(false)
	end
end))

if Notification then
	Notification:SetupJJs()
end
