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

	Experiments = { },  

	Tempo = 0.5,
	Rainbow = false,
}

-- ══════════════════════════════════════
--           Tempo Controller
-- ══════════════════════════════════════
local function SetTempo(seconds)
	if typeof(seconds) ~= "number" then return end
	if seconds <= 0 then return end
	Options.Tempo = seconds
end

local Tempos = {
	Rapido = 0.5, -- 100 JJs em 50s
	Normal = 1,   -- 1 msg/s
	Seguro = 2    -- 0.5 msg/s
}

-- Exemplo (remova se quiser):
-- SetTempo(Tempos.Rapido)

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
local Request = require("Modules/Request")

-- ══════════════════════════════════════
--  	        Constants				
-- ══════════════════════════════════════
local Char = Character.new(LP)
local UIElements = UI.UIElements
local Connections = {}

local Threading
local FinishedThread = false
local Toggled = false

local Settings = {
	Keybind = Options.Keybind or "Home",

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
		RemoteChat:Send(string.format("%s%s", Message, Prefix))
	end,

	["Lowercase"] = function(Message, Prefix)
		if Settings.Jump then Char:Jump() end
		RemoteChat:Send(string.format("%s%s", string.lower(Message), Prefix))
	end,

	["HJ"] = function(Message, Prefix)
		for i = 1, #Message do
			if Settings.Jump then Char:Jump() end
			RemoteChat:Send(string.format("%s%s", string.sub(Message, i, i), Prefix))
			task.wait(Options.Tempo)
		end

		if Settings.Jump then Char:Jump() end
		RemoteChat:Send(string.format("%s%s", Message, Prefix))
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
			Settings.Config[Name] = Element.Text
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

local function DoJJ(Name, Number, Prefix)
	local Success, String = Extenso:Convert(Number)
	Prefix = Prefix or ""

	local Method = Methods[Name]
	if not Method then
		Notification:Notify(12)
		return
	end

	if Success then
		Method(String, Prefix)
	end
end

local function StartThread()
	local Config = Settings.Config
	if not Config.Start or not Config.End then return end
	if Threading then EndThread(false) return end

	local Method =
		table.find(Options.Experiments, "hell_jacks_2024_02-dev") and "HJ" or
		table.find(Options.Experiments, "lowercase_jjs_2024_12") and "Lowercase" or
		"Normal"

	Notification:Notify(5)

	Threading = task.spawn(function()
		for Amount = Config.Start, Config.End do
			DoJJ(Method, Amount, Config.Prefix)

			if Amount ~= tonumber(Config.End) then
				task.wait(Options.Tempo)
			end
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

for Name, Element in pairs(UIElements.Box) do
	task.spawn(Listen, Name, Element)
end

if Notification then
	Notification:SetupJJs()
end
