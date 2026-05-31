-- ═══════════════════════════════════════════════════════════
--  Fish It! Plaza Booth Sniper v2.0 + Auto Server Hop
--  Scan booth → bandingkan RAP/ALP → kirim snipe ke Discord
--  → Auto rehop ke server lain setelah scan selesai
-- ═══════════════════════════════════════════════════════════

local Players           = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local HttpService       = game:GetService("HttpService")
local TeleportService   = game:GetService("TeleportService")
local Workspace         = game:GetService("Workspace")

local player  = Players.LocalPlayer
local placeId = game.PlaceId
local jobId   = game.JobId

-- ═══════════ CONFIG ═══════════
local WEBHOOK_URL    = "https://discord.com/api/webhooks/1510604266094727350/KaePnBfD09tfO6TnzwgteSJq-pqOJEFSDoz9cgOm07v8Se-_z3civS06Z6lQRZibZBuf"
local MAX_PRICE      = 1500       -- harga MAKSIMAL (tokens) untuk di-list
-- ITEM FILTER
local TRACKED_ITEMS = {
    "megalodon",              -- ALL megalodon (Galaxy, Frozen, Bermutasi, Ghost, dll)
}

-- SERVER HOP CONFIG
local REHOP_ENABLED  = true       -- auto pindah server setelah scan
local REHOP_DELAY    = 5          -- delay (detik) sebelum hop
local SCAN_WAIT      = 8          -- tunggu data load sebelum scan (detik)
local MIN_PLAYERS    = 10         -- minimal player di server target
local RETRY_DELAY    = 3          -- delay antar retry join (detik)
-- ═══════════════════════════════

-- Cek apakah item name cocok dengan tracked items
local function isTrackedItem(itemName)
    if not itemName then return false end
    local lower = itemName:lower()
    for _, keyword in ipairs(TRACKED_ITEMS) do
        if lower:find(keyword, 1, true) then
            return true
        end
    end
    return false
end

local httpFunc = (syn and syn.request) or (http and http.request)
    or http_request or (fluxus and fluxus.request) or request
if not httpFunc then warn("❌ HTTP tidak support!"); return end

-- ═══════════════════════════════
-- REPLION: Cari remote
-- ═══════════════════════════════
local function findReplionRemote(name)
    local pkg = ReplicatedStorage:FindFirstChild("Packages")
    if not pkg then return nil end
    local idx = pkg:FindFirstChild("_Index")
    if not idx then return nil end
    for _, f in ipairs(idx:GetChildren()) do
        if f.Name:lower():find("replion") then
            local function s(p, d)
                if d > 6 then return nil end
                for _, o in ipairs(p:GetChildren()) do
                    if o.Name == name and o:IsA("RemoteEvent") then return o end
                    local r = s(o, d+1); if r then return r end
                end; return nil
            end
            local r = s(f, 0); if r then return r end
        end
    end; return nil
end

-- ═══════════════════════════════
-- DATA STORAGE
-- ═══════════════════════════════
local boothData    = {}   -- [sellerName] = { items... }
local rapCache     = {}   -- [itemId] = rap value
local alpCache     = {}   -- [itemId] = {avg, count}
local allRemoteData= {}
local snipeLog     = {}   -- sudah dikirim, anti spam

-- ═══════════════════════════════
-- UTILITY
-- ═══════════════════════════════
local function fmt(n)
    local s = tostring(math.floor(tonumber(n) or 0))
    return s:reverse():gsub("(%d%d%d)","%1,"):reverse():gsub("^,","")
end

local function sendWebhook(payload)
    pcall(function()
        httpFunc({
            Url = WEBHOOK_URL, Method = "POST",
            Headers = {["Content-Type"] = "application/json"},
            Body = HttpService:JSONEncode(payload)
        })
    end)
end

-- ═══════════════════════════════
-- DEEP MERGE
-- ═══════════════════════════════
local function mergeDeep(dst, src, d)
    if d > 8 then return end
    for k, v in pairs(src) do
        if type(v)=="table" and type(dst[k])=="table" then
            mergeDeep(dst[k], v, d+1)
        else dst[k] = v end
    end
end

-- ═══════════════════════════════
-- HOOK REPLION REMOTES
-- ═══════════════════════════════
local hooked = {}

local function hookRemote(rem)
    if hooked[rem] then return end
    hooked[rem] = true
    pcall(function()
        rem.OnClientEvent:Connect(function(...)
            local args = {...}
            for _, a in ipairs(args) do
                if type(a) == "table" then
                    mergeDeep(allRemoteData, a, 0)
                end
            end
        end)
    end)
end

local function hookAll(p, d)
    if d > 10 then return end
    for _, c in ipairs(p:GetChildren()) do
        if c:IsA("RemoteEvent") then hookRemote(c) end
        pcall(hookAll, c, d+1)
    end
end
hookAll(game, 0)
game.DescendantAdded:Connect(function(o)
    if o:IsA("RemoteEvent") then hookRemote(o) end
end)

-- ═══════════════════════════════
-- SCAN BOOTH DI WORKSPACE (GUI-based)
-- Data booth ada di GUI elements pada model booth
-- ═══════════════════════════════

-- Cari semua booth model di workspace
local function findAllBooths()
    local booths = {}
    local seen = {}
    for _, desc in ipairs(Workspace:GetDescendants()) do
        if desc:IsA("Model") and not seen[desc] then
            local n = desc.Name:lower()
            local path = desc:GetFullName():lower()
            -- Skip menu/shop stands (Rod Stand, Bobber Stand, dll)
            local isMenu = path:find("menu") or path:find("worldsetup") or path:find("shop")
            if not isMenu and (n:find("booth") or n:find("stall") or n:find("stand")) then
                seen[desc] = true
                table.insert(booths, desc)
            end
        end
    end
    return booths
end

-- Ambil semua TextLabel/TextButton text dari sebuah GUI
local function collectTexts(guiInstance)
    local texts = {}
    for _, desc in ipairs(guiInstance:GetDescendants()) do
        if (desc:IsA("TextLabel") or desc:IsA("TextButton")) and desc.Text ~= "" then
            table.insert(texts, {
                name = desc.Name,
                text = desc.Text,
                obj  = desc,
            })
        end
    end
    return texts
end

-- Parse angka dari text (support K, M suffix dan comma)
local function parseNumber(str)
    if not str or type(str) ~= "string" then return 0 end
    str = tostring(str):gsub(",", ""):gsub("%s+", "")
    if str == "" then return 0 end
    -- Handle K/M suffix
    local num, suffix = str:match("([%d%.]+)([KkMm]?)")
    if not num then return 0 end
    num = tonumber(num) or 0
    suffix = suffix or ""
    if suffix:lower() == "k" then num = num * 1000
    elseif suffix:lower() == "m" then num = num * 1000000 end
    return num
end

-- ═══════════════════════════════
-- EXTRACT BOOTH ITEMS DARI WORKSPACE GUI
-- Scan BillboardGui / SurfaceGui di booth models
-- ═══════════════════════════════
local function extractBoothItems()
    local items = {}
    local booths = findAllBooths()

    for _, booth in ipairs(booths) do
        -- Cari seller name dari booth name (contoh: "Riyy's Booth" → "Riyy")
        local seller = booth.Name:gsub("'s Booth", ""):gsub("'s booth", "")
            :gsub(" Booth", ""):gsub(" booth", "")
        if seller == "" then seller = "Unknown" end

        -- Scan semua GUI di booth
        for _, desc in ipairs(booth:GetDescendants()) do
            if desc:IsA("BillboardGui") or desc:IsA("SurfaceGui") or desc:IsA("Frame") then
                local texts = collectTexts(desc)

                -- Coba extract item data dari text labels
                local itemName, itemPrice, itemRap, itemWeight, itemVariant = nil, nil, nil, nil, nil

                for _, t in ipairs(texts) do
                    local txt = t.text
                    local tName = t.name:lower()

                    -- Detect price (angka dengan token icon atau "Tokens")
                    if tName:find("price") or tName:find("cost") or tName:find("token")
                        or txt:match("^%d[%d,%.]*$") or txt:match("^%d[%d,%.]*[KkMm]?$") then
                        local p = parseNumber(txt)
                        if p > 0 and p < 10000000 then
                            itemPrice = itemPrice or p
                        end
                    end

                    -- Detect RAP
                    if tName:find("rap") or txt:lower():find("rap") then
                        local rapNum = parseNumber(txt:gsub("[Rr][Aa][Pp]:?", ""))
                        if rapNum > 0 then itemRap = rapNum end
                    end

                    -- Detect weight
                    if txt:match("%d+%.?%d*kg") then
                        local w = tonumber(txt:match("(%d+%.?%d*)kg")) or 0
                        if w > 0 then itemWeight = w end
                    end

                    -- Detect item name (text yang bukan angka, bukan berat, bukan RAP)
                    if not txt:match("^%d") and not txt:lower():find("rap")
                        and not txt:match("kg$") and not txt:lower():find("booth")
                        and not txt:lower():find("sold") and not txt:lower():find("buy")
                        and #txt > 2 and #txt < 60 then
                        itemName = itemName or txt
                    end

                    -- Variant
                    if tName:find("variant") or tName:find("type") then
                        itemVariant = txt
                    end
                end

                -- Jika kita punya item dengan harga, simpan
                if itemName and itemPrice and itemPrice > 0 then
                    table.insert(items, {
                        uid      = booth.Name.."_"..tostring(#items+1),
                        name     = itemName,
                        itemId   = 0,
                        price    = itemPrice,
                        seller   = seller,
                        variant  = itemVariant or "",
                        rarity   = "",
                        weight   = itemWeight or 0,
                        rap      = itemRap or 0,
                        alp      = 0,
                        alpCount = 0,
                        itemType = "Fish",
                        path     = booth:GetFullName(),
                    })
                end
            end
        end
    end

    -- JUGA scan remote data (backup)
    local function deepScan(tbl, path, depth)
        if type(tbl) ~= "table" or depth > 12 then return end
        for k, v in pairs(tbl) do
            if type(v) == "table" then
                local price = v.Price or v.price or v.Cost or v.cost
                    or v.TokenPrice or v.tokenPrice or v.Tokens or v.tokens
                local name = v.Name or v.name or v.DisplayName or v.displayName
                local seller = v.Seller or v.seller or v.Owner or v.owner
                    or v.SellerName or v.PlayerName
                local uid = v.UUID or v.uuid or v.UID or v.uid or tostring(k)
                local itemId = v.Id or v.id or v.ItemId or v.itemId or 0

                if price and tonumber(price) and tonumber(price) > 0 then
                    local rap = tonumber(v.RAP or v.rap or v.RecentAveragePrice
                        or v.AveragePrice or v.Value or v.value or 0) or 0
                    local variant = v.VariantId or v.variantId or v.Variant or ""
                    local weight = 0
                    if type(v.Metadata) == "table" then
                        weight = tonumber(v.Metadata.Weight or 0) or 0
                    end

                    table.insert(items, {
                        uid      = uid,
                        name     = tostring(name or "Item#"..tostring(itemId)),
                        itemId   = tonumber(itemId) or 0,
                        price    = tonumber(price),
                        seller   = tostring(seller or "Unknown"),
                        variant  = tostring(variant),
                        rarity   = tostring(v.Rarity or v.rarity or ""),
                        weight   = weight,
                        rap      = rap,
                        alp      = tonumber(v.ALP or v.alp or 0) or 0,
                        alpCount = tonumber(v.ALPCount or v.ListingCount or 0) or 0,
                        itemType = tostring(v.ItemType or v.itemType or ""),
                        path     = path.."/"..tostring(k),
                    })
                end
                deepScan(v, path.."/"..tostring(k), depth+1)
            end
        end
    end
    deepScan(allRemoteData, "remote", 0)

    return items
end

-- ═══════════════════════════════
-- DEBUG: DUMP BOOTH GUI STRUCTURE
-- Untuk melihat struktur GUI booth yang sebenarnya
-- ═══════════════════════════════
local function dumpBoothStructure()
    local booths = findAllBooths()
    print("\n🏪 ═══ BOOTH STRUCTURE DUMP ═══")
    print("Total booth: "..#booths)

    for i, booth in ipairs(booths) do
        if i > 5 then print("[..."..#booths-5 .." more booths]"); break end
        print("\n🏪 Booth #"..i..": "..booth.Name.." ("..booth:GetFullName()..")")

        local n = 0
        for _, desc in ipairs(booth:GetDescendants()) do
            n += 1
            if n > 100 then print("  [...truncated]"); break end

            local indent = "  "
            if desc:IsA("BillboardGui") or desc:IsA("SurfaceGui") then
                print(indent.."🖼️ "..desc.ClassName.." ["..desc.Name.."]")
            elseif desc:IsA("Frame") then
                print(indent.."📦 Frame ["..desc.Name.."]")
            elseif desc:IsA("TextLabel") or desc:IsA("TextButton") then
                local txt = desc.Text:sub(1, 80)
                print(indent.."📝 "..desc.ClassName.." ["..desc.Name.."] = \""..txt.."\"")
            elseif desc:IsA("ImageLabel") or desc:IsA("ImageButton") then
                print(indent.."🖼️ "..desc.ClassName.." ["..desc.Name.."] img="..tostring(desc.Image):sub(1,40))
            end
        end
    end
    print("═══ END BOOTH DUMP ═══\n")
end

-- ═══════════════════════════════
-- RESOLVE ITEM NAME
-- ═══════════════════════════════
local function resolveItemName(item)
    local base = item.name or "Unknown"
    if base == "" or base:find("^Item#") then
        base = "Fish#"..item.itemId
    end
    if item.variant ~= "" and item.variant ~= "None" and item.variant ~= "Default" then
        base = base .. " - " .. item.variant
    end
    if item.rarity ~= "" and item.rarity ~= "Common" then
        base = base .. " (" .. item.rarity .. ")"
    end
    return base
end

-- ═══════════════════════════════
-- FILTER: Harga <= MAX_PRICE
-- ═══════════════════════════════
local function findSnipes(items)
    local snipes = {}
    for _, item in ipairs(items) do
        -- Skip item milik sendiri
        if item.seller == player.Name then continue end

        local price = item.price
        local rap = item.rap

        -- Hanya list item dengan harga <= MAX_PRICE
        if price <= MAX_PRICE then
            local rapPct = rap > 0 and math.floor((price / rap) * 100) or 0
            local save = rap > 0 and (rap - price) or 0
            local alpPct = 0
            local alpStr = "N/A"
            if item.alp > 0 then
                alpPct = math.floor((price / item.alp) * 100)
                alpStr = fmt(math.floor(item.alp)).." ("..alpPct.."%)"
                if item.alpCount > 0 then
                    alpStr = alpStr .. " · "..item.alpCount.." listings"
                end
            end

            table.insert(snipes, {
                name     = resolveItemName(item),
                price    = price,
                seller   = item.seller,
                rap      = rap,
                rapPct   = rapPct,
                save     = save,
                alp      = item.alp,
                alpPct   = alpPct,
                alpStr   = alpStr,
                alpCount = item.alpCount,
                uid      = item.uid,
            })
        end
    end

    -- Sort by price ascending (termurah dulu)
    table.sort(snipes, function(a, b) return a.price < b.price end)
    return snipes
end

-- ═══════════════════════════════
-- KIRIM KE DISCORD
-- ═══════════════════════════════
local function sendSnipesToDiscord(snipes)
    if #snipes == 0 then return end

    -- Anti spam: skip jika sudah dikirim
    local newSnipes = {}
    for _, s in ipairs(snipes) do
        local key = s.uid .. "_" .. s.price
        if not snipeLog[key] then
            snipeLog[key] = os.clock()
            table.insert(newSnipes, s)
        end
    end
    if #newSnipes == 0 then return end

    -- Max 25 items
    local maxItems = math.min(25, #newSnipes)

    -- Build teleport script
    local teleportScript = string.format(
        'game:GetService("TeleportService"):TeleportToPlaceInstance(%d, "%s", game.Players.LocalPlayer)',
        placeId, jobId
    )

    -- Header description
    local joinLink = string.format(
        "https://www.roblox.com/games/start?placeId=%d&gameInstanceId=%s",
        placeId, jobId
    )
    local desc = string.format(
        "**Server:** `%s`\n"..
        "**Scanner:** @%s\n"..
        "**Total:** %d item (max %d tokens)\n\n"..
        "🔗 **[Join Server](%s)**\n\n"..
        "**Join Script:**\n```\n%s\n```",
        jobId:sub(1,12), player.Name, maxItems, MAX_PRICE, joinLink, teleportScript
    )

    -- Build fields: 1 item per field, inline (3 kolom per baris)
    local fields = {}

    for i = 1, maxItems do
        local s = newSnipes[i]
        local rapStr = s.rap > 0 and fmt(s.rap) or "N/A"
        local rapPct = s.rapPct or 0
        local saveStr = s.save ~= 0 and fmt(s.save) or "0"
        local alpStr = s.alpStr or "N/A"

        local fieldValue = string.format(
            "Nama: **%s**\n"..
            "Harga: **%s Tokens**\n"..
            "Seller: %s\n"..
            "RAP: **%s** (%d%%)\n"..
            "Save: **%s Tokens**\n"..
            "ALP: **%s**",
            s.name,
            fmt(s.price),
            s.seller,
            rapStr, rapPct,
            saveStr,
            alpStr
        )

        table.insert(fields, {
            name = "🔥 #" .. i .. " SNIPE",
            value = fieldValue,
            inline = true
        })
    end

    local embed = {
        title       = "🐟 Megalodon ≤ "..MAX_PRICE.." Tokens!",
        description = desc,
        color       = 3066993,
        fields      = fields,
        footer      = {
            text = "Fish It Sniper | @"..player.Name
                .." | "..os.date("%d/%m/%Y %H:%M")
        }
    }

    sendWebhook({
        username   = "🐟 Fish It Booth Sniper",
        avatar_url = "https://tr.rbxcdn.com/180DAY-"..player.UserId,
        embeds     = { embed }
    })

    print("✅ Dikirim "..maxItems.." item ke Discord!")
end

-- ═══════════════════════════════
-- SCAN FUNCTION
-- ═══════════════════════════════
local scanCount = 0

local function doScan()
    scanCount += 1
    print("\n🔍 ═══ SCAN #"..scanCount.." ═══")

    local booths = findAllBooths()
    print("🏪 Ditemukan "..#booths.." booth model di workspace")

    local allItems = extractBoothItems()
    print("📦 Ditemukan "..#allItems.." listing total di booth")

    -- Filter: hanya item yang di-track
    local items = {}
    for _, item in ipairs(allItems) do
        local name = resolveItemName(item)
        if isTrackedItem(name) or isTrackedItem(item.name) then
            table.insert(items, item)
        end
    end

    -- Sort: megalodon paling atas, lalu by harga termurah
    table.sort(items, function(a, b)
        local aName = resolveItemName(a):lower()
        local bName = resolveItemName(b):lower()
        local aMega = aName:find("megalodon") and true or false
        local bMega = bName:find("megalodon") and true or false
        if aMega ~= bMega then return aMega end
        return a.price < b.price
    end)

    print("🎯 Tracked items: "..#items.." dari "..#allItems.." total")

    if #allItems == 0 then
        print("⚠️ Tidak ada listing terdeteksi!")
        print("   Menjalankan booth structure dump untuk debug...")
        dumpBoothStructure()
        return
    end

    if #items == 0 then
        print("❌ Tidak ada item yang di-track di server ini")
        return
    end

    -- Print semua listing
    print("\n📋 Listing ("..#items.." item):")
    print("  ┌─────────────────────────────────────────────────────────┐")
    for i, item in ipairs(items) do
        local name = resolveItemName(item)
        local rapStr = item.rap > 0 and fmt(item.rap) or "N/A"
        print(string.format("  │ #%d  🎣 %s", i, name))
        print(string.format("  │     💰 Harga: %s Tokens  |  📊 RAP: %s", fmt(item.price), rapStr))
        print(string.format("  │     👤 Seller: %s  |  📍 %s", item.seller, item.path:sub(1,40)))
        if item.rap > 0 and item.price < item.rap then
            local save = item.rap - item.price
            local pct = math.floor((item.price / item.rap) * 100)
            print(string.format("  │     🔥 DEAL! %s%% of RAP (Save %s tokens)", pct, fmt(save)))
        end
        print("  │")
    end
    print("  └─────────────────────────────────────────────────────────┘")

    -- Cari snipe (harga <= MAX_PRICE)
    local snipes = findSnipes(items)
    if #snipes > 0 then
        print("\n🔥 ITEMS ≤ "..MAX_PRICE.." TOKENS: "..#snipes)
        for i, s in ipairs(snipes) do
            local rapStr = s.rap > 0 and fmt(s.rap) or "N/A"
            print(string.format("  🔥 #%d %s → %s Tokens | RAP: %s | Seller: %s",
                i, s.name, fmt(s.price), rapStr, s.seller))
        end

        -- Kirim ke Discord
        if WEBHOOK_URL ~= "GANTI_WEBHOOK_DISCORD_KAMU" then
            sendSnipesToDiscord(snipes)
        else
            print("⚠️ Webhook belum diset! Ganti WEBHOOK_URL di config.")
        end
    else
        print("❌ Tidak ada item dengan harga ≤ "..MAX_PRICE.." tokens")
    end
end

-- ═══════════════════════════════
-- DEBUG: DUMP SEMUA DATA
-- ═══════════════════════════════
local function dumpAll()
    print("\n🔍 ═══ FULL REMOTE DATA DUMP ═══")
    local n = 0
    local function dump(t, pre, d)
        if type(t) ~= "table" or d > 5 then return end
        for k, v in pairs(t) do
            n += 1; if n > 500 then print("[truncated]"); return end
            if type(v) == "table" then
                print("  📁 "..pre..tostring(k))
                dump(v, pre.."  ", d+1)
            else
                print("  "..pre..tostring(k).." = "..tostring(v))
            end
        end
    end
    dump(allRemoteData, "", 0)
    print("═══ END ("..n.." entries) ═══")

    print("\n🏪 ═══ BOOTH GUI DUMP ═══")
    dumpBoothStructure()
end

-- ═══════════════════════════════
-- CLEANUP SPAM LOG (tiap 5 menit)
-- ═══════════════════════════════
task.spawn(function()
    while true do
        task.wait(300)
        local now = os.clock()
        for k, t in pairs(snipeLog) do
            if now - t > 600 then snipeLog[k] = nil end
        end
    end
end)

-- ═══════════════════════════════
-- SERVER HOP: Fetch server list & teleport
-- ═══════════════════════════════
local serverHopCount = 0

local function getServerList()
    local servers = {}
    local cursor = ""
    local pages = 0

    repeat
        pages += 1
        local url = string.format(
            "https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Desc&limit=100&cursor=%s",
            placeId, cursor
        )
        local ok, res = pcall(function()
            return httpFunc({ Url = url, Method = "GET" })
        end)

        if not ok or not res then break end

        local body = res.Body or res.body or ""
        local success, data = pcall(HttpService.JSONDecode, HttpService, body)
        if not success or not data or not data.data then break end

        for _, srv in ipairs(data.data) do
            -- Ambil SEMUA server (bukan hanya yang belum dikunjungi)
            -- Filter: bukan server saat ini + minimal MIN_PLAYERS
            local playerCount = srv.playing or 0
            if srv.id and srv.id ~= jobId and playerCount >= MIN_PLAYERS then
                table.insert(servers, {
                    id      = srv.id,
                    players = playerCount,
                    maxPlr  = srv.maxPlayers or 0,
                    ping    = srv.ping or 0,
                    isFull  = playerCount >= (srv.maxPlayers or 999),
                })
            end
        end

        cursor = data.nextPageCursor or ""
    until cursor == "" or #servers >= 100 or pages >= 5

    -- Sort: server dengan lebih banyak player dulu (lebih banyak booth)
    table.sort(servers, function(a, b) return a.players > b.players end)
    return servers
end

local function tryJoinServer(target)
    local attempt = 0

    while true do
        attempt += 1
        print(string.format("  🔌 Attempt %d → %s (%d/%d players)",
            attempt, target.id:sub(1,12), target.players, target.maxPlr))

        -- Setup listener untuk detect ASYNC teleport failure (GameFull, etc)
        local teleportFailed = false
        local failReason = ""
        local failConn = nil

        pcall(function()
            failConn = TeleportService.TeleportInitFailed:Connect(function(plr, result, errorMessage)
                if plr == player then
                    teleportFailed = true
                    failReason = tostring(errorMessage or result or "Unknown")
                    print("  ❌ Teleport GAGAL (async): "..failReason)
                end
            end)
        end)

        -- Fire teleport
        local ok, err = pcall(function()
            TeleportService:TeleportToPlaceInstance(placeId, target.id, player)
        end)

        if not ok then
            -- Immediate error (rare)
            print("  ⚠️ Teleport error langsung: "..tostring(err))
            if failConn then pcall(function() failConn:Disconnect() end) end
            task.wait(RETRY_DELAY)
            -- Continue to next attempt
        else
            -- Teleport fired OK, tunggu apakah berhasil atau async fail
            print("  ⏳ Menunggu respons teleport...")

            -- Tunggu max 15 detik untuk menentukan hasil
            local waitTime = 0
            local checkInterval = 0.5
            while waitTime < 15 and not teleportFailed do
                task.wait(checkInterval)
                waitTime += checkInterval
            end

            -- Disconnect listener
            if failConn then pcall(function() failConn:Disconnect() end) end

            if teleportFailed then
                -- Teleport gagal async (GameFull, etc)
                local isGameFull = failReason:lower():find("full")
                    or failReason:lower():find("gamefull")
                    or failReason:lower():find("capacity")

                if isGameFull then
                    print(string.format("  🔄 Server PENUH! Retry #%d dalam %ds...", attempt, RETRY_DELAY))
                else
                    print(string.format("  🔄 Gagal: %s | Retry #%d dalam %ds...", failReason, attempt, RETRY_DELAY))
                end

                -- Dismiss dialog "Teleport Failed" jika muncul
                pcall(function()
                    local coreGui = game:GetService("CoreGui")
                    for _, gui in ipairs(coreGui:GetDescendants()) do
                        if gui:IsA("TextButton") and (gui.Text == "OK" or gui.Text == "Retry") then
                            pcall(function()
                                if typeof(fireclick) == "function" then fireclick(gui) end
                            end)
                            pcall(function() gui.Activated:Fire() end)
                            pcall(function()
                                if typeof(getconnections) == "function" then
                                    for _, conn in ipairs(getconnections(gui.Activated)) do
                                        conn:Fire()
                                    end
                                end
                            end)
                        end
                    end
                end)

                task.wait(RETRY_DELAY)
                -- Continue loop → retry
            else
                -- 15 detik tanpa fail = kemungkinan berhasil teleport
                print("  ✅ Teleport berhasil! Loading server baru...")
                task.wait(30)
                return true
            end
        end
    end
end

local function hopToNextServer()
    serverHopCount += 1
    print("\n🔄 ═══ SERVER HOP #"..serverHopCount.." ═══")
    print("📡 Mengambil daftar server (min "..MIN_PLAYERS.." players)...")

    local servers = getServerList()
    print("📋 Ditemukan "..#servers.." server dengan "..MIN_PLAYERS.."+ players")

    if #servers == 0 then
        print("⚠️ Tidak ada server dengan "..MIN_PLAYERS.."+ players!")
        print("🔄 Retry dalam 30 detik...")
        task.wait(30)
        return hopToNextServer()
    end

    -- Tampilkan daftar server
    print("\n📋 Server List (top 10):")
    for i = 1, math.min(10, #servers) do
        local s = servers[i]
        local fullTag = s.isFull and " [FULL]" or ""
        print(string.format("  %d. %s | %d/%d players%s",
            i, s.id:sub(1,12), s.players, s.maxPlr, fullTag))
    end

    -- Coba join server satu per satu sampai berhasil
    for i, target in ipairs(servers) do
        print(string.format("\n🎯 Mencoba server #%d: %s (%d/%d players)",
            i, target.id:sub(1,12), target.players, target.maxPlr))

        -- Kirim notif hop ke Discord
        if WEBHOOK_URL ~= "GANTI_WEBHOOK_DISCORD_KAMU" then
            sendWebhook({
                username = "🐟 Fish It Booth Sniper",
                embeds = {{
                    title = "🔄 Server Hop #"..serverHopCount,
                    description = string.format(
                        "Pindah ke server lain\n"
                        .."**From:** `%s`\n"
                        .."**To:** `%s` (%d/%d players)\n"
                        .."**Total Hop:** %d",
                        jobId:sub(1,12), target.id:sub(1,12),
                        target.players, target.maxPlr, serverHopCount
                    ),
                    color = 15844367,
                    footer = { text = "Scanner: @"..player.Name.." • "..os.date("%H:%M:%S") }
                }}
            })
        end

        task.wait(REHOP_DELAY)

        -- Coba join dengan retry jika penuh
        local joined = tryJoinServer(target)
        if joined then return end

        print("🔄 Coba server berikutnya...")
    end

    -- Semua server gagal, retry dari awal
    print("\n❌ Semua server gagal! Retry dalam 30 detik...")
    task.wait(30)
    return hopToNextServer()
end

-- ═══════════════════════════════
-- QUEUE ON TELEPORT (auto re-execute)
-- ═══════════════════════════════
local function setupAutoExecute()
    -- Coba queue script untuk dijalankan ulang setelah teleport
    local scriptSource = nil
    pcall(function()
        -- Baca source script ini
        if getgenv and getgenv().FISH_SNIPER_SOURCE then
            scriptSource = getgenv().FISH_SNIPER_SOURCE
        end
    end)

    -- queue_on_teleport tersedia di kebanyakan executor
    if queue_on_teleport then
        local queueScript = scriptSource or
            '-- Auto re-execute Fish It Sniper after teleport\n'
            ..'print("🔄 Re-executing Fish It Sniper...")\n'
            ..'task.wait(3)\n'
            ..'-- Script akan dijalankan ulang otomatis oleh executor'
        pcall(queue_on_teleport, queueScript)
        print("✅ queue_on_teleport terpasang!")
    elseif syn and syn.queue_on_teleport then
        pcall(syn.queue_on_teleport, "-- auto rehop")
        print("✅ syn.queue_on_teleport terpasang!")
    else
        print("⚠️ queue_on_teleport tidak tersedia!")
        print("   Script TIDAK akan auto-execute setelah teleport.")
        print("   Kamu perlu jalankan ulang script manual setelah hop.")
    end
end

-- ═══════════════════════════════
-- MAIN — AUTO START
-- ═══════════════════════════════
print("🐟 ═══════════════════════════════════")
print("🐟  Fish It! Plaza Booth Sniper v3.0")
print("🐟  Scanner: " .. player.Name)
print("🐟  Server:  " .. jobId:sub(1,12))
print("🐟  Max Price: " .. MAX_PRICE .. " Tokens")
print("🐟  Min Players: " .. MIN_PLAYERS)
print("🐟  Rehop: " .. (REHOP_ENABLED and "✅ ON" or "❌ OFF"))
print("🐟  Filter: megalodon saja")
print("🐟 ═══════════════════════════════════")

-- Setup auto execute setelah teleport
setupAutoExecute()

-- AUTO START: langsung scan tanpa perlu trigger manual
print("\n⏳ Auto Start! Menunggu "..SCAN_WAIT.."s agar data load...")
task.wait(SCAN_WAIT)

print("🔍 Auto Scan dimulai...")
doScan()

-- Setelah scan → langsung hop ke server lain
if REHOP_ENABLED then
    print("\n🔄 AUTO REHOP AKTIF!")
    print("   Flow: Scan → Hop → Scan → Hop → ...")
    print("   Target: server dengan "..MIN_PLAYERS.."+ players")
    print("   Jika server penuh: retry terus sampai masuk")
    task.spawn(function()
        task.wait(2)
        hopToNextServer()
    end)
end

-- Expose global untuk manual
getgenv().FishSniper = {
    scan = doScan,
    dump = dumpAll,
    hop  = hopToNextServer,
    setWebhook = function(url)
        WEBHOOK_URL = url
        print("✅ Webhook diset!")
    end,
    setMaxPrice = function(n)
        MAX_PRICE = n
        print("✅ Max price: "..n.." tokens")
    end,
    setRehop = function(on)
        REHOP_ENABLED = on
        print("✅ Rehop: "..(on and "ON" or "OFF"))
    end,
    setMinPlayers = function(n)
        MIN_PLAYERS = n
        print("✅ Min players: "..n)
    end,
}

print("\n💡 Command manual (F9 console):")
print("   getgenv().FishSniper.scan()            → scan manual")
print("   getgenv().FishSniper.hop()             → hop manual")
print("   getgenv().FishSniper.dump()            → dump data")
print('   getgenv().FishSniper.setWebhook("URL")  → set webhook')
print("   getgenv().FishSniper.setMaxPrice(200)    → ubah harga max")
print("   getgenv().FishSniper.setMinPlayers(15)   → ubah min players")
print("   getgenv().FishSniper.setRehop(false)     → matikan rehop")
