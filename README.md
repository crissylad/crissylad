- 👋 Hlocal Players = game:GetService("Players")
local DataStoreService = game:GetService("DataStoreService")
local HttpService = game:GetService("HttpService")

-- SETTINGS
local MAX_WALKSPEED = 20
local MAX_JUMPPOWER = 60
local BAN_DURATION = 7 * 24 * 60 * 60  -- 7 days in seconds
local DISCORD_WEBHOOK = "https://discord.com/api/webhooks/YOUR_WEBHOOK_HERE"

local BanStore = DataStoreService:GetDataStore("BanData")
local StatsStore = DataStoreService:GetDataStore("PlayerStats")

local joinTimes = {}

-- 🛡 Ban + Log
local function banPlayer(player, reason)
	local data = {
		bannedAt = os.time(),
		reason = reason,
		duration = BAN_DURATION
	}
	local key = tostring(player.UserId)
	pcall(function()
		BanStore:SetAsync(key, data)
	end)
	
	-- Discord log
	local content = {
		username = "RobloxBanTracker",
		content = string.format("🔨 **%s** (UserId %d) was banned.\nReason: %s\nDuration: 7 days.", player.Name, player.UserId, reason)
	}
	local json = HttpService:JSONEncode(content)
	pcall(function()
		HttpService:PostAsync(DISCORD_WEBHOOK, json, Enum.HttpContentType.ApplicationJson)
	end)

	player:Kick("🚫 You’ve been caught using cheats.\nAppeal via our Discord server.")
end

-- 🕵️ Check ban status
local function isBanned(userId)
	local key = tostring(userId)
	local success, ban = pcall(function()
		return BanStore:GetAsync(key)
	end)
	if success and ban then
		if os.time() < ban.bannedAt + ban.duration then
			return true, ban.reason
		else
			pcall(function()
				BanStore:RemoveAsync(key)
			end)
		end
	end
	return false
end

-- 📊 Update player stats (visits/playtime)
local function updateStats(player)
	local keyVisits = "Visits_" .. player.UserId
	local keyTime = "Time_" .. player.UserId
	
	-- Visits
	local visits = 0
	pcall(function()
		visits = StatsStore:GetAsync(keyVisits) or 0
		StatsStore:SetAsync(keyVisits, visits + 1)
	end)

	-- Leaderstats
	local stats = Instance.new("Folder")
	stats.Name = "leaderstats"
	stats.Parent = player

	local visitValue = Instance.new("IntValue")
	visitValue.Name = "Visits"
	visitValue.Value = visits + 1
	visitValue.Parent = stats

	local timeValue = Instance.new("IntValue")
	timeValue.Name = "PlaytimeMins"
	timeValue.Value = 0
	timeValue.Parent = stats
end

-- 🧠 Save playtime
local function savePlaytime(player)
	local startTime = joinTimes[player.UserId]
	if not startTime then return end
	local session = tick() - startTime
	local key = "Time_" .. player.UserId
	local total = 0
	pcall(function()
		total = StatsStore:GetAsync(key) or 0
		StatsStore:SetAsync(key, total + session)
	end)
end

-- 🧍 On player added
Players.PlayerAdded:Connect(function(player)
	local banned, reason = isBanned(player.UserId)
	if banned then
		player:Kick("🚫 Banned for: " .. reason .. "\nAppeal at our Discord.")
		return
	end

	joinTimes[player.UserId] = tick()
	updateStats(player)

	player.CharacterAdded:Connect(function(char)
		local hum = char:FindFirstChildOfClass("Humanoid")
		if not hum then return end

		-- Movement check
		coroutine.wrap(function()
			while player and player.Parent and hum and hum.Parent do
				if hum.WalkSpeed > MAX_WALKSPEED then
					banPlayer(player, "Speed hack (WalkSpeed too high)")
					break
				end
				if hum.JumpPower > MAX_JUMPPOWER then
					banPlayer(player, "Jump hack (JumpPower too high)")
					break
				end
				wait(2)
			end
		end)()

		-- Damage detection
		local lastHealth = hum.Health
		hum.HealthChanged:Connect(function(new)
			if new < lastHealth then
				lastHealth = new
			elseif new >= hum.MaxHealth then
				-- Possibly cheating by avoiding damage
				banPlayer(player, "No damage taken (Godmode)")
			end
		end)
	end)
end)

-- 📴 On player leave
Players.PlayerRemoving:Connect(function(player)
	savePlaytime(player)
end)