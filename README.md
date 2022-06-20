## Premise
You have a popular game where players can collect and trade rare pets. How do you keep your trading feature safe from players losing their items or (un)intentionally duplicating them? This explanation is targeted at trading between players in a single game instance, but the principle applies to cross-instance trading too.

General player persistence code will be kept extremely simple and wouldn't be suitable for use in games.

## Feedback
I welcome any comments or criticism, including nitpicking. I want to know what didn't make sense and why. This tutorial has a [GitHub repository](https://github.com/Qualadore/Tutorial-preventing-duplication-and-data-loss-from-trading) if you'd like to file a Pull Request or Issue.

## Target audience
This tutorial is intended for developers who are already experienced with Luau and the Roblox API.

## Naive approach
We'll start with a simplified implementation of player persistence.
```lua
--!strict
local data_store_service = game:GetService("DataStoreService")
local players_service = game:GetService("Players")

type pet = {
	category: "Unicorn" | "Dragon",
}
type persistence = {
	pets: {[string]: pet},
}
local player_to_persistence: {[Player]: persistence} = {}

local persistence_data_store = data_store_service:GetDataStore("player_persistence")

players_service.PlayerAdded:Connect(function(player: Player)
	local persistence: persistence? = nil
	local ok = pcall(function()
		persistence = persistence_data_store:GetAsync(player.UserId)
	end)
	if ok == false then
		return
	end

	if persistence == nil then
		persistence = {
			pets = {},
		}
	end
	assert(persistence :: persistence, "")

	player_to_persistence[player] = persistence
end)

players_service.PlayerRemoving:Connect(function(player: Player)
	local persistence = player_to_persistence[player]
	if persistence == nil then
		return
	end
	player_to_persistence[player] = nil

	pcall(function()
		persistence_data_store:SetAsync(player.UserId, persistence, {
			player.UserId,
		})
	end)
end)
```

Now let's try to add trading.
```lua
-- [...] Copied from above.

local replicated_storage_service = game:GetService("ReplicatedStorage")

local network = replicated_storage_service:FindFirstChild("network")
local trade_pets_event = network:FindFirstChild("trade_pets")

-- We'll let any player trade away anyone's items without asking them for simplicity.
trade_pets_event.OnServerEvent:Connect(function(
	sender: Player,
	receiver: Player,
	giving_identifier_set: {[string]: true},
	receiving_identifier_set: {[string]: true}
)
	if typeof(sender) ~= "Instance" or typeof(receiver) ~= "Instance" or sender == receiver then
		return
	end
	if sender:IsA("Player") == false or receiver:IsA("Player") == false then
		return
	end
	local sender_persistence = player_to_persistence[sender]
	if sender_persistence == nil then
		return
	end
	local receiver_persistence = player_to_persistence[receiver]
	if receiver_persistence == nil then
		return
	end

	if typeof(giving_identifier_set) == "table" then
		for key in giving_identifier_set do
			if type(key) == "string" then
				if sender_persistence.pets[key] == nil then
					return
				end
			else
				return
			end
		end
	else
		return
	end
	
	if typeof(receiving_identifier_set) == "table" then
		for key in receiving_identifier_set do
			if type(key) == "string" then
				if receiver_persistence.pets[key] == nil then
					return
				end
			else
				return
			end
		end
	end

	for giving_identifier in giving_identifier_set do
		local pet = sender_persistence.pets[giving_identifier]
		sender_persistence.pets[giving_identifier] = nil
		receiver_persistence.pets[giving_identifier] = pet
	end

	for receiving_identifier in receiving_identifier_set do
		local pet = receiver_persistence.pets[receiving_identifier]
		receiver_persistence.pets[receiving_identifier] = nil
		sender_persistence.pets[receiving_identifier] = pet
	end

	task.spawn(function()
		pcall(function()
			persistence_data_store:SetAsync(sender.UserId, sender_persistence, {
				sender.UserId,
			})
		end)
	end)
	task.spawn(function()
		pcall(function()
			persistence_data_store:SetAsync(receiver.UserId, receiver_persistence, {
				receiver.UserId,
			})
		end)
	end)
end)
```

Consider a trade where the sender is giving a Legendary Dragon for the receiver's Common Unicorn. What if the receiver and sender are cheaters, and the sender very quickly joins another instance of the game after trading, before there is time for their data to save? The new game instance fetches the sender's old persistence from before the trade was completed, when the sender still had a Legendary Dragon. But the receiver still received the Legendary Dragon—it has been duplicated! What's the solution?

## Session locking
First we need to implement *session locking*. Session locking involves placing a *lock* on a player's persistence so no other game instances can access it, and we won't unlock it until their play *session* is over because they've left the game. We have to consider that an instance could crash without a chance to unlock a player's session.

```lua
--!strict
local data_store_service = game:GetService("DataStoreService")
local memory_store_service = game:GetService("MemoryStoreService")
local players_service = game:GetService("Players")

type pet = {
	category: "Unicorn" | "Dragon",
}
type persistence = {
	pets: {[string]: pet},
}
local player_to_persistence: {[Player]: persistence} = {}
local loading_player_set: {[Player]: true} = {}

local persistence_data_store = data_store_service:GetDataStore("player_persistence")
local session_lock_sorted_map = memory_store_service:GetSortedMap("player_persistence_session_lock")
local lock_duration = 120

players_service.PlayerAdded:Connect(function(player: Player)
	-- Ignore players if they rejoin this instance too quickly.
	-- This is to prevent two `PlayerAdded` listeners making requests for the same player.
	if loading_player_set[player] == nil then
		loading_player_set[player] = true
	else
		return
	end

	local is_locked = true
	pcall(function()
		-- We use `MemoryStoreSortedMap:UpdateAsync()` instead of getting and setting the lock in two requests.
		-- This makes the operation atomic. Otherwise, two game instances could read that persistence was unlocked and both think they could lock it.
		session_lock_sorted_map:UpdateAsync(player.UserId, function(old_key: boolean?)
			if old_key == nil then
				is_locked = false
				return true
			else
				-- Cancel the update, session is already locked.
				return nil
			end
		end, lock_duration)
	end)

	local persistence: persistence? = nil
	local ok = pcall(function()
		persistence = persistence_data_store:GetAsync(player.UserId)
	end)
	if ok == false then
		return
	end

	if persistence == nil then
		persistence = {
			pets = {},
		}
	end
	assert(persistence :: persistence, "")

	player_to_persistence[player] = persistence
	loading_player_set[player] = nil

	task.spawn(function()
		-- Repeatedly renew our lock so it doesn't expire.
		while true do
			if player_to_persistence[player] ~= nil then
				return
			end

			pcall(function()
				memory_store_sorted_map:SetAsync(player.UserId, true, lock_duration)
			end)

			task.wait(15)
		end
	end)
end)

players_service.PlayerRemoving:Connect(function(player: Player)
	local persistence = player_to_persistence[player]
	if persistence == nil then
		return
	end
	player_to_persistence[player] = nil

	pcall(function()
		persistence_data_store:SetAsync(player.UserId, persistence, {
			player.UserId,
		})
	end)
	-- Only unlock persistence after an attempt to save it has been made.
	pcall(function()
		session_lock_sorted_map:RemoveAsync(player.UserId)
	end)
end)
```

We try to read from the `MemoryStoreSortedMap` whether or not a player's persistence is locked. If it is already locked, we don't try to load the player's persistence. If it isn't, we activate the lock with a duration argument. Once that duration elapses, the key will be automatically removed from the `MemoryStoreSortedMap` and it will unlock the persistence. We repeatedly renew the lock to prevent it from expiring while the player is still in the game, and when a player leaves we finally unlock their persistence.

You may have seen a similar trick involving `DataStore:UpdateAsync()` and a timestamp check. If so, [spoiler]note that our approach is used instead because there are [time accuracy issues](https://twitter.com/ScriptOff41968/status/1525929536956813318) between instances that are avoided. That means we can set shorter lock durations, which is good in case an instance crashes.[/spoiler]

## Transactions
Great! We've mostly solved the problem of players quickly rejoining and duplicating pets that way. Are we done?

Not quite. What happens if the receiver's persistence saves, but the sender's fails to save? Duplication of the Legendary Dragon again! This could happen because of platform issues, network errors, unintentionally triggered request limits, or cheaters intentionally triggering request limits. What do we do?

We want to *only* complete the trade if both the sender and receiver can successfully save their persistences. How do we do this?

First, we want to generate a Globally Unique Identifier (GUID) for the trade. A GUID is a long string identifier that in our case will look something like `43F15C96-C55D-4F3A-968C-ED522BF46AC6`. There's so many possibilities that generating the same one as any other ever generated—even by any other software outside Roblox—is incredibly mathematically unlikely, so we can easily generate a unique identifier for every trade without storing a registry of them to check if they've been used!

Then, we want a separate `DataStore` where the keys are the GUIDs of trades (GUIDs are comfortably shorter than the maximum `DataStore` key name length). The value will be `true` if the trade has been completed, otherwise `nil`.

Then for all players involved in the trade, we'll modify their persistence tables. We'll remove their pets from the `pets` subtable, and create a table for the pending trade. The new `pending_trades` key of persistences will map from a trade's GUID to information about what is being given and what is being received by the player. We're assuming that each pet also has its own GUID.

```lua
pending_trades = {
	["43F15C96-C55D-4F3A-968C-ED522BF46AC6"] = {
		giving = {
			["1EFCB6A6-CB4A-4973-8978-0935EEB0D9DD"] = {
				category = "Dragon",
			},
			["21244568-F1AB-4BFA-9F7A-18AED5AE6B73"] = {
				category = "Dragon",
			}
		},
		receiving = {
			["9503B73D-2667-4A8D-BAC8-1D1B2629734B"] = {
				category = "Unicorn",
			},
		},
	},
}
```

The general idea is that we'll first try to save all players' persistences with these pending trade subtables added. If any one player's saving request fails, we consider the trade cancelled and return players their pets. If all saving requests succeed, we mark the trade as completed within the trade `DataStore`, then give each player their pets.

What happens if a player rejoins with pending trades? We look up the trade GUID within the trade `DataStore` and check if it has been marked as completed. If it has, we give the player the pets they were meant to receive. If it hasn't, we can safely assume that the trade has failed and return the player their pets.

When could a player rejoin with pending trades? Say all players' persistences have been saved with the trade added to `pending_trades`, and either the request to mark the trade as completed fails, or succeeds but subsequent requests to save players' persistences fail.

This is our finished code
```lua
--!strict
local data_store_service = game:GetService("DataStoreService")
local HTTP_service = game:GetService("HttpService")
local memory_store_service = game:GetService("MemoryStoreService")
local players_service = game:GetService("Players")
local replicated_storage_service = game:GetService("ReplicatedStorage")

type pet = {
	category: "Unicorn" | "Dragon",
}
type pending_trade = {
	giving: {[string]: pet},
	receiving: {[string]: pet},
}
type persistence = {
	pets: {[string]: pet},
	pending_trades: {[string]: pending_trade}
}
local player_to_persistence: {[Player]: persistence} = {}
local loading_player_set: {[Player]: true} = {}
local trading_player_set: {[Player]: true} = {}

local trade_data_store = data_store_service:GetDataStore("pet_trade")
local persistence_data_store = data_store_service:GetDataStore("player_persistence")
local session_lock_sorted_map = memory_store_service:GetSortedMap("player_persistence_session_lock")
local lock_duration = 120
local player_to_lock_renewal_time: {[Player]: number} = {}
local trading_maximum_time_since_lock_renewal = 20

players_service.PlayerAdded:Connect(function(player: Player)
	-- Ignore players if they rejoin this instance too quickly.
	-- This is to prevent two `PlayerAdded` listeners making requests for the same player.
	if loading_player_set[player] == nil then
		loading_player_set[player] = true
	else
		return
	end

	local is_locked = true
	pcall(function()
		-- We use `MemoryStoreSortedMap:UpdateAsync()` instead of getting and setting the lock in two requests.
		-- This makes the operation atomic. Otherwise, two game instances could read that persistence was unlocked and both think they could lock it.
		session_lock_sorted_map:UpdateAsync(player.UserId, function(old_key: boolean?): boolean?
			if old_key == nil then
				is_locked = false
				return true
			else
				-- Cancel the update, session is already locked.
				return nil
			end
		end, lock_duration)
	end)

	if is_locked == true then
		return
	end
	
	player_to_lock_renewal_time[player] = os.time()

	local persistence: persistence? = nil
	local ok = pcall(function()
		persistence = persistence_data_store:GetAsync(player.UserId)
	end)
	if ok == false then
		return
	end

	if persistence == nil then
		persistence = {
			pets = {},
			pending_trades = {},
		}
	end
	assert(persistence :: persistence, "")

	player_to_persistence[player] = persistence
	loading_player_set[player] = nil

	for GUID, pending_trade in persistence.pending_trades do
		task.spawn(function()
			pcall(function()
				local result = trade_data_store:GetAsync(GUID)
				if result == true then
					for identifier, receiving_pet in pending_trade.receiving do
						persistence.pets[identifier] = receiving_pet
					end
				else
					-- Since the session is unlocked and the trade is still pending, we can assume it's cancelled.
					for identifier, giving_pet in pending_trade.giving do
						persistence.pets[identifier] = giving_pet
					end
				end
				persistence.pending_trades[GUID] = nil
			end)
		end)
	end

	task.spawn(function()
		-- Repeatedly renew our lock so it doesn't expire.
		while true do
			-- Keep renewing lock while player is trading even if they've left.
			if trading_player_set[player] == nil and player_to_persistence[player] ~= nil then
				return
			end

			local renewal_ok = pcall(function()
				session_lock_sorted_map:SetAsync(player.UserId, true, lock_duration)
			end)
			if player_to_persistence[player] ~= nil and renewal_ok == true then
				player_to_lock_renewal_time[player] = os.time()
			end

			task.wait(15)
		end
	end)
end)

players_service.PlayerRemoving:Connect(function(player: Player)
	local persistence = player_to_persistence[player]
	if persistence == nil then
		return
	end
	player_to_persistence[player] = nil
	player_to_lock_renewal_time[player] = nil

	pcall(function()
		persistence_data_store:SetAsync(player.UserId, persistence, {
			player.UserId,
		})
	end)
	-- If a player has a trade pending, don't unlock their persistence yet.
	while trading_player_set[player] == true do
		task.wait()
	end

	-- Only unlock persistence after an attempt to save it has been made.
	pcall(function()
		session_lock_sorted_map:RemoveAsync(player.UserId)
	end)
end)

local network = replicated_storage_service:FindFirstChild("network")
local trade_pets_event = network:FindFirstChild("trade_pets")

-- We'll let any player trade away anyone's items without asking them for simplicity.
trade_pets_event.OnServerEvent:Connect(function(
	sender: Player,
	receiver: Player,
	giving_identifier_set: {[string]: true},
	receiving_identifier_set: {[string]: true}
)
	if typeof(sender) ~= "Instance" or typeof(receiver) ~= "Instance" or sender == receiver then
		return
	end
	if sender:IsA("Player") == false or receiver:IsA("Player") == false then
		return
	end
	local sender_persistence = player_to_persistence[sender]
	if sender_persistence == nil then
		return
	end
	local receiver_persistence = player_to_persistence[receiver]
	if receiver_persistence == nil then
		return
	end

	if typeof(giving_identifier_set) == "table" then
		for key in giving_identifier_set do
			if type(key) == "string" then
				if sender_persistence.pets[key] == nil then
					return
				end
			else
				return
			end
		end
	else
		return
	end
	
	if typeof(receiving_identifier_set) == "table" then
		for key in receiving_identifier_set do
			if type(key) == "string" then
				if receiver_persistence.pets[key] == nil then
					return
				end
			else
				return
			end
		end
	end

	if trading_player_set[sender] == true or trading_player_set[receiver] == true then
		return
	end

	-- If either player's session lock hasn't been successfully renewed recently enough, cancel the trade.
	if player_to_lock_renewal_time[sender] < trading_maximum_time_since_lock_renewal then
		return
	end
	if player_to_lock_renewal_time[receiver] < trading_maximum_time_since_lock_renewal then
		return
	end

	trading_player_set[sender] = true
	trading_player_set[receiver] = true

	local GUID = HTTP_service:GenerateGUID(false)

	local sender_pending_trade: pending_trade = {
		giving = {},
		receiving = {},
	}
	local receiver_pending_trade: pending_trade = {
		giving = {},
		receiving = {},
	}
	for identifier in giving_identifier_set do
		local pet = sender_persistence.pets[identifier]
		sender_persistence.pets[identifier] = nil
		sender_pending_trade.giving[identifier] = pet
		receiver_pending_trade.receiving[identifier] = pet
	end
	for identifier in receiving_identifier_set do
		local pet = receiver_persistence.pets[identifier]
		receiver_persistence.pets[identifier] = nil
		receiver_pending_trade.giving[identifier] = pet
		sender_pending_trade.receiving[identifier] = pet
	end
	sender_persistence.pending_trades[GUID] = sender_pending_trade
	receiver_persistence.pending_trades[GUID] = receiver_pending_trade

	local sender_result: boolean? = nil
	task.spawn(function()
		sender_result = pcall(function()
			persistence_data_store:SetAsync(sender.UserId, sender_persistence, {
				sender.UserId,
			})
		end)
	end)
	local receiver_result: boolean? = nil
	task.spawn(function()
		receiver_result = pcall(function()
			persistence_data_store:SetAsync(receiver.UserId, receiver_persistence, {
				receiver.UserId,
			})
		end)
	end)

	task.spawn(function()
		while true do
			if sender_result ~= nil and receiver_result ~= nil then
				if sender_result == true and receiver_result == true then
					local trade_ok = true
					-- If either player has left, cancel the trade.
					if player_to_persistence[sender] == nil or player_to_persistence[receiver] == nil then
						trade_ok = false
					end

					if trade_ok == true then
						trade_ok = pcall(function()
							trade_data_store:SetAsync(GUID, true)
						end)
					end

					if trade_ok == true then
						for identifier, pet in sender_pending_trade.giving do
							receiver_persistence.pets[identifier] = pet
						end
						for identifier, pet in sender_pending_trade.receiving do
							sender_persistence.pets[identifier] = pet
						end
					else
						for identifier, pet in sender_pending_trade.giving do
							sender_persistence.pets[identifier] = pet
						end
						for identifier, pet in sender_pending_trade.receiving do
							receiver_persistence.pets[identifier] = pet
						end
					end

					sender_persistence.pending_trades[GUID] = nil
					receiver_persistence.pending_trades[GUID] = nil
				end

				break
			end

			task.wait()
		end

		trading_player_set[sender] = nil
		trading_player_set[receiver] = nil
	end)
end)
```

In a real game, it would be wasteful not to remove the trade from the trade `DataStore` once all players know the trade's result.

While this code applies to trades of two players, this approach scales to an arbitrary number, so long as all players are within the same instance.

## Is it possible for this to fail?
Technically yes, if the session lock expired before trading was able to be completed, duplication would be possible. This is why a relatively high value of 120 seconds was chosen for the lock duration. If either the sender's or receiver's lock wasn't renewed recently enough, the trade will be cancelled.