local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")

-- Upgrades
local Upgrades = {
	click = { id="click", name="Click", baseCost=12, basePower=1, costMult=1.18, maxLevel=100, type="increment" },
	autoclicker = { id="autoclicker", name="Auto", baseCost=90, basePower=1, costMult=1.22, maxLevel=200, type="persecond" },
	crit = { id="crit", name="Crit", baseCost=240, basePower=0.02, costMult=1.5, maxLevel=50, type="chance" },
	multiplier = { id="multiplier", name="Mult", baseCost=900, basePower=0.05, costMult=1.6, maxLevel=40, type="multiplier" }
}

local function calcCost(def, currentLevel)
	local nextLevel = (currentLevel or 0) + 1
	return math.floor(def.baseCost * (def.costMult ^ (nextLevel-1)) + 0.5)
end

local function calcPower(def, level)
	level = level or 0
	if def.type == "increment" then
		return def.basePower * level
	elseif def.type == "persecond" then
		return def.basePower * level
	elseif def.type == "chance" then
		return def.basePower * level
	elseif def.type == "multiplier" then
		return 1 + (def.basePower * level)
	end
	return 0
end

-- Remotes
local remotes = ReplicatedStorage:FindFirstChild("SingleClickerRemotes")
if not remotes then
	remotes = Instance.new("Folder")
	remotes.Name = "SingleClickerRemotes"
	remotes.Parent = ReplicatedStorage
end

local RemoteClick = remotes:FindFirstChild("Click") or Instance.new("RemoteEvent")
RemoteClick.Name = "Click"
RemoteClick.Parent = remotes

local RemoteRequest = remotes:FindFirstChild("RequestState") or Instance.new("RemoteFunction")
RemoteRequest.Name = "RequestState"
RemoteRequest.Parent = remotes

local RemotePurchase = remotes:FindFirstChild("PurchaseUpgrade") or Instance.new("RemoteEvent")
RemotePurchase.Name = "PurchaseUpgrade"
RemotePurchase.Parent = remotes

local RemoteUpdate = remotes:FindFirstChild("UpdateUI") or Instance.new("RemoteEvent")
RemoteUpdate.Name = "UpdateUI"
RemoteUpdate.Parent = remotes

-- Player Data
local PlayerData = {} -- [userId] = {coins, upgrades, dirty}

local function createLeaderstats(player)
	local folder = Instance.new("Folder")
	folder.Name = "leaderstats"
	folder.Parent = player
	local coins = Instance.new("IntValue")
	coins.Name = "Coins"
	coins.Value = 0
	coins.Parent = folder
end

local function defaultUpgrades()
	local t = {}
	for k,v in pairs(Upgrades) do t[k] = 0 end
	return t
end

local function initPlayer(player)
	local uid = player.UserId
	PlayerData[uid] = { coins = 0, upgrades = defaultUpgrades(), dirty = false }
	createLeaderstats(player)
	if RunService:IsStudio() then
		PlayerData[uid].coins = PlayerData[uid].coins + 500
	end
	RemoteUpdate:FireClient(player, {coins = PlayerData[uid].coins, upgrades = PlayerData[uid].upgrades})
end

local function cleanupPlayer(player)
	PlayerData[player.UserId] = nil
end

Players.PlayerAdded:Connect(initPlayer)
Players.PlayerRemoving:Connect(cleanupPlayer)

-- Server Logic
local function getPlayerData(player)
	return PlayerData[player.UserId]
end

local function awardCoins(player, amount)
	if amount <= 0 then return end
	local pdata = getPlayerData(player)
	if not pdata then return end
	local mult = calcPower(Upgrades.multiplier, pdata.upgrades.multiplier)
	local total = math.floor(amount * mult + 0.5)
	pdata.coins = pdata.coins + total
	pdata.dirty = true
	local stats = player:FindFirstChild("leaderstats")
	if stats and stats:FindFirstChild("Coins") then
		stats.Coins.Value = pdata.coins
	end
	RemoteUpdate:FireClient(player, {coins = pdata.coins, upgrades = pdata.upgrades})
end

RemoteClick.OnServerEvent:Connect(function(player)
	local pdata = getPlayerData(player)
	if not pdata then return end
	local base = 1
	local clickPower = calcPower(Upgrades.click, pdata.upgrades.click)
	local critChance = calcPower(Upgrades.crit, pdata.upgrades.crit)
	local gained = base + clickPower
	if math.random() < math.clamp(critChance,0,0.95) then gained = gained*2 end
	awardCoins(player, gained)
end)

RemotePurchase.OnServerEvent:Connect(function(player, upgradeId)
	local pdata = getPlayerData(player)
	if not pdata then return end
	local def = Upgrades[upgradeId]
	if not def then
		RemoteUpdate:FireClient(player, {error="bad"})
		return
	end
	local level = pdata.upgrades[upgradeId] or 0
	if def.maxLevel and level >= def.maxLevel then
		RemoteUpdate:FireClient(player, {error="max"})
		return
	end
	local cost = calcCost(def, level)
	if pdata.coins < cost then
		RemoteUpdate:FireClient(player, {error="low", coins=pdata.coins})
		return
	end
	pdata.coins = pdata.coins - cost
	pdata.upgrades[upgradeId] = level+1
	pdata.dirty = true
	local stats = player:FindFirstChild("leaderstats")
	if stats and stats:FindFirstChild("Coins") then stats.Coins.Value = pdata.coins end
	RemoteUpdate:FireClient(player, {coins=pdata.coins, upgrades=pdata.upgrades, purchased=upgradeId})
end)

RemoteRequest.OnServerInvoke = function(player)
	local pdata = getPlayerData(player)
	if not pdata then return {error="no"} end
	local summary = {}
	for id, def in pairs(Upgrades) do
		summary[id] = {id=def.id,name=def.name,baseCost=def.baseCost,maxLevel=def.maxLevel,type=def.type}
	end
	return {coins=pdata.coins, upgrades=pdata.upgrades, summary=summary}
end

-- Autosave
task.spawn(function()
	while true do
		task.wait(15)
		for _,pdata in pairs(PlayerData) do
			if pdata.dirty then pdata.dirty=false end
		end
	end
end)

-- Autoclicker
task.spawn(function()
	while true do
		task.wait(1)
		for _,player in ipairs(Players:GetPlayers()) do
			local pdata = getPlayerData(player)
			if pdata and pdata.upgrades.autoclicker > 0 then
				local perSec = calcPower(Upgrades.autoclicker, pdata.upgrades.autoclicker)
				if perSec>0 then awardCoins(player, perSec) end
			end
		end
	end
end)

-- Demo NPC
if not Workspace:FindFirstChild("SingleFileClickerDemo") then
	local folder = Instance.new("Folder"); folder.Name="Demo"; folder.Parent=Workspace
	local npc = Instance.new("Model"); npc.Name="Vendor"; npc.Parent=folder
	local head = Instance.new("Part"); head.Name="Head"; head.Size=Vector3.new(2,1,1); head.Position=Vector3.new(0,5,0); head.Anchored=true; head.Parent=npc
	local root = Instance.new("Part"); root.Name="Root"; root.Size=Vector3.new(2,2,1); root.Position=Vector3.new(0,4,0); root.Anchored=true; root.Parent=npc
	local click = Instance.new("ClickDetector"); click.MaxActivationDistance=12; click.Parent=head
	click.MouseClick:Connect(function(player)
		local pdata=getPlayerData(player)
		if not pdata then return end
		if pdata.coins<50 then 
			pdata.coins=pdata.coins+100
			pdata.dirty=true
			local stats=player:FindFirstChild("leaderstats")
			if stats and stats:FindFirstChild("Coins") then stats.Coins.Value=pdata.coins end
			RemoteUpdate:FireClient(player,{coins=pdata.coins})
		else
			RemoteUpdate:FireClient(player,{coins=pdata.coins,upgrades=pdata.upgrades,message="go"})
		end
	end)
end

