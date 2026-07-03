Full Deobfuscated Script || By iblameaabis

local playerService = game:GetService("Players")
local localPlayer = playerService.LocalPlayer
local replicatedStorage = game:GetService("ReplicatedStorage")
local runService = game:GetService("RunService")
local virtualInput = game:GetService("VirtualInputManager")
local lighting = game:GetService("Lighting")

local whitelistUrl = "https://raw.githubusercontent.com/reaperXKSGIV/ml/refs/heads/main/Moon%20Hub/whitelist.lua"
local changeSpeedSizeRemote = replicatedStorage.rEvents.changeSpeedSizeRemote

local waterParts = {}
local waterPartSize = 2048
local waterGridRange = 50000
local waterBasePosition = Vector3.new(-2, -9.5, -2)

local trackedPlayer = nil
local lockPositionEnabled = false
local lockPositionCoroutine = nil
local lockedRootPart = nil
local lockedPosition = nil
local spectateConnection = nil
local spectateTarget = nil
local spectateEnabled = false
local currentCamera = workspace.CurrentCamera
local infiniteJumpConnection = nil

local statIcons = {
    Time = utf8.char(128347),
    Stats = utf8.char(128202),
    Strength = utf8.char(128170),
    Rebirths = utf8.char(128260),
    Durability = utf8.char(128737),
    Kills = utf8.char(128128),
    Agility = utf8.char(127939),
    ["Evil Karma"] = utf8.char(128520),
    ["Good Karma"] = utf8.char(128519),
    Brawls = utf8.char(129354),
}

local statDefinitions = {
    { name = "Strength", statName = "Strength" },
    { name = "Rebirths", statName = "Rebirths" },
    { name = "Durability", statName = "Durability" },
    { name = "Agility", statName = "Agility" },
    { name = "Kills", statName = "Kills" },
    { name = "Evil Karma", statName = "evilKarma" },
    { name = "Good Karma", statName = "goodKarma" },
    { name = "Brawls", statName = "Brawls" },
}

local function checkWhitelist(playerName)
    local success, response = pcall(function()
        return game:HttpGet(whitelistUrl)
    end)
    if success and response then
        for line in string.gmatch(response, "[^\r\n]+") do
            if string.lower(line:match("^%s*(.-)%s*$")) == string.lower(playerName) then
                return true
            end
        end
    end
    return false
end

local function formatNumberShort(value)
    if value >= 1e15 then return string.format("%.1fqa", value / 1e15) end
    if value >= 1e12 then return string.format("%.1ft", value / 1e12) end
    if value >= 1e9 then return string.format("%.1fb", value / 1e9) end
    if value >= 1e6 then return string.format("%.1fm", value / 1e6) end
    if value >= 1e3 then return string.format("%.1fk", value / 1e3) end
    return tostring(value)
end

local function formatNumberComma(value)
    local formatted = tostring(floorFn(value))
    while true do
        local result, replacements = formatted:gsub("^(-?%d+)(%d%d%d)", "%1,%2")
        formatted = result
        if replacements == 0 then break end
    end
    return formatted
end

local function getPlayerDisplayEntry(player)
    return player.DisplayName .. " | " .. player.Name
end

local function getAllPlayers()
    return playerService:GetPlayers()
end

local function getLocalCharacter()
    local character = localPlayer.Character
    if not character then
        task.wait()
        character = localPlayer.Character
    end
    return character
end

local function doPunch()
    for _, tool in pairs(localPlayer.Backpack:GetChildren()) do
        if tool.Name == "Punch" then
            local character = localPlayer.Character
            if character and character:FindFirstChild("Humanoid") then
                character.Humanoid:EquipTool(tool)
            end
        end
    end
    localPlayer.muscleEvent:FireServer("punch", "leftHand")
    localPlayer.muscleEvent:FireServer("punch", "rightHand")
end

local function isPlayerValid(player)
    if player and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
        return true
    end
    return false
end

local function attackPlayer(target)
    if not isPlayerValid(target) then return end
    local myCharacter = getLocalCharacter()
    if myCharacter and myCharacter:FindFirstChild("LeftHand") then
        pcall(function()
            firetouchinterest(target.Character.HumanoidRootPart, myCharacter.LeftHand, 0)
            firetouchinterest(target.Character.HumanoidRootPart, myCharacter.LeftHand, 1)
            doPunch()
        end)
    end
end

local function getEnemyHealth(player)
    if not player then return 0 end
    local durability = player:FindFirstChild("Durability")
    local leaderstats = player:FindFirstChild("leaderstats")
    local baseHealth = 0

    if durability then
        baseHealth = durability.Value
    elseif leaderstats then
        local durStat = leaderstats:FindFirstChild("Durability")
        if durStat then baseHealth = durStat.Value end
    end

    local healthMultiplier = 1
    local ultimatesFolder = player:FindFirstChild("ultimatesFolder")
    if ultimatesFolder then
        local infernalHealth = ultimatesFolder:FindFirstChild("Infernal Health")
        if infernalHealth then
            healthMultiplier = 1 + 0.15 * (infernalHealth.Value or 0)
        end
    end

    local petMultiplier = 1
    local equippedPets = player:FindFirstChild("equippedPets")
    if equippedPets then
        local petBonus = 0
        for _, pet in ipairs(equippedPets:GetChildren()) do
            if pet:IsA("ObjectValue") and pet.Value then
                local petName = string.lower(pet.Value.Name)
                if petName:match("mighty") and petName:match("monster") then
                    petBonus = petBonus + 0.5
                end
            end
        end
        petMultiplier = 1 + petBonus
    end

    return baseHealth * healthMultiplier * petMultiplier
end

local function getPlayerDamage()
    local leaderstats = localPlayer:FindFirstChild("leaderstats")
    if not leaderstats then return 0 end
    local strengthStat = leaderstats:FindFirstChild("Strength")
    if not strengthStat then return 0 end

    local damageMultiplier = 1
    local ultimatesFolder = localPlayer:FindFirstChild("ultimatesFolder")
    if ultimatesFolder then
        local demonDamage = ultimatesFolder:FindFirstChild("Demon Damage")
        if demonDamage then
            damageMultiplier = 1 + 0.1 * (demonDamage.Value or 0)
        end
    end

    local petMultiplier = 1
    local equippedPets = localPlayer:FindFirstChild("equippedPets")
    if equippedPets then
        local petBonus = 0
        for _, pet in ipairs(equippedPets:GetChildren()) do
            if pet:IsA("ObjectValue") and pet.Value then
                local petName = string.lower(pet.Value.Name)
                if petName:match("wild") and petName:match("wizard") then
                    petBonus = petBonus + 0.5
                end
            end
        end
        petMultiplier = 1 + petBonus
    end

    return strengthStat.Value * 0.0667 * damageMultiplier * petMultiplier
end

local function getHitsToKill(health, damage)
    if damage <= 0 then return "âˆž" end
    local hits = math.ceil(health / damage)
    if hits > 100 then return "âˆž" end
    if hits < 1 then return 1 end
    return hits
end

local function isWhitelistedPlayer(player)
    if not _G.whitelistedPlayers then return false end
    for _, name in ipairs(_G.whitelistedPlayers) do
        if name:lower() == player.Name:lower() then
            return true
        end
    end
    return false
end

local function isBlacklistedPlayer(player)
    if not _G.blacklistedPlayers then return false end
    for _, name in ipairs(_G.blacklistedPlayers) do
        if name:lower() == player.Name:lower() then
            return true
        end
    end
    return false
end

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "SCPPaidLoad"
screenGui.ResetOnSpawn = false
screenGui.Parent = localPlayer:WaitForChild("PlayerGui")

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 380, 0, 170)
mainFrame.Position = UDim2.new(0.5, -190, 0.5, -85)
mainFrame.BackgroundColor3 = Color3.fromRGB(12, 12, 12)
mainFrame.BorderSizePixel = 0
mainFrame.Parent = screenGui
Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 14)

local headerFrame = Instance.new("Frame")
headerFrame.Size = UDim2.new(1, 0, 0, 50)
headerFrame.BackgroundColor3 = Color3.fromRGB(139, 0, 0)
headerFrame.BorderSizePixel = 0
headerFrame.Parent = mainFrame
Instance.new("UICorner", headerFrame).CornerRadius = UDim.new(0, 14)

local headerLabel = Instance.new("TextLabel")
headerLabel.Size = UDim2.new(1, 0, 1, 0)
headerLabel.BackgroundTransparency = 1
headerLabel.Text = "Moon Hub | Private Script"
headerLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
headerLabel.TextSize = 20
headerLabel.Font = Enum.Font.GothamBold
headerLabel.Parent = headerFrame

local statusLabel = Instance.new("TextLabel")
statusLabel.Size = UDim2.new(1, 0, 0, 40)
statusLabel.Position = UDim2.new(0, 0, 0, 60)
statusLabel.BackgroundTransparency = 1
statusLabel.Text = "Checking whitelist..."
statusLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
statusLabel.TextSize = 15
statusLabel.Font = Enum.Font.Gotham
statusLabel.Parent = mainFrame

local footerLabel = Instance.new("TextLabel")
footerLabel.Size = UDim2.new(1, 0, 0, 30)
footerLabel.Position = UDim2.new(0, 0, 0, 128)
footerLabel.BackgroundTransparency = 1
footerLabel.Text = "Private Can't Buy"
footerLabel.TextColor3 = Color3.fromRGB(139, 0, 0)
footerLabel.TextSize = 14
footerLabel.Font = Enum.Font.GothamBold
footerLabel.Parent = mainFrame

task.wait()

if localPlayer and localPlayer.Character then
    localPlayer:WaitForChild("leaderstats", 15)
    task.wait(1)

    if not checkWhitelist(localPlayer.Name) then
        statusLabel.Text = "NOT WHITELISTED!"
        statusLabel.TextColor3 = Color3.fromRGB(255, 60, 60)
        task.wait(3)
        screenGui:Destroy()
        localPlayer:Kick("NOT WHITELISTED!")
        return
    end

    statusLabel.Text = "Whitelisted! Loading hub..."
    statusLabel.TextColor3 = Color3.fromRGB(0, 220, 100)
    task.wait(1.5)
    screenGui:Destroy()

    local muscleEvent = localPlayer:WaitForChild("muscleEvent")
    local leaderstats = localPlayer:WaitForChild("leaderstats")
    local rebirthsStat = leaderstats:WaitForChild("Rebirths")

    local library = loadstring(game:HttpGet("https://raw.githubusercontent.com/imhenne187/SilenceElerium/refs/heads/main/src/SilenceEleriumLibrary.luau", true))()
    local mainWindow = library:AddWindow("                   MOON HUB |                    PRIVATE SCRIPT |                    WELCOME    " .. localPlayer.DisplayName, {
        title_bar = {
            Color3.fromRGB(0, 0, 0),
            Color3.fromRGB(115, 0, 0),
            Color3.fromRGB(50, 0, 0),
        },
        title_bar_transparency = 0.2,
        background = {
            Color3.fromRGB(0, 0, 0),
            Color3.fromRGB(34, 0, 0),
            Color3.fromRGB(68, 0, 0),
        },
        background_transparency = 0.1,
        main_color = Color3.fromRGB(0, 0, 0),
        min_size = Vector2.new(650, 650),
        can_resize = true,
    })

    local separator = string.rep("â•", 38)
    local dashSeparator = string.rep("â€”", 28)

    local infoTab = mainWindow:AddTab("InfoTab")
    infoTab:AddLabel(separator)

    local welcomeLabel = infoTab:AddLabel("WELCOME TO MOON HUB PRIVATE")
    welcomeLabel.TextSize = 28
    task.spawn(function()
        while true do
            welcomeLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    infoTab:AddLabel(separator)

    local madeByLabel = infoTab:AddLabel("Made by: REAPER ||")
    task.spawn(function()
        while true do
            madeByLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    local versionLabel = infoTab:AddLabel("Version: PRIVATE V1")
    task.spawn(function()
        while true do
            versionLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    infoTab:AddLabel(separator)

    local friendsHeaderLabel = infoTab:AddLabel("                                                  My Friends:                                                 ")
    task.spawn(function()
        while true do
            friendsHeaderLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    local friend1Label = infoTab:AddLabel("Lil Skills Aka KingSkillsGaming")
    task.spawn(function()
        while true do
            friend1Label.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    local friend2Label = infoTab:AddLabel("Epic Dev Aka Zolo")
    task.spawn(function()
        while true do
            friend2Label.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    infoTab:Show()

    local mainTab = mainWindow:AddTab("Main")

    local playerOptionsLabel = mainTab:AddLabel("âš™ï¸Player Options âš™ï¸")
    task.spawn(function()
        while true do
            playerOptionsLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    local targetSize = 2
    local autoSizeEnabled = false

    mainTab:AddTextBox("Size", function(input)
        local cleaned = input:gsub("%s+", "")
        local num = tonumber(cleaned)
        if num and num > 0 then
            targetSize = num
        end
    end)

    local sizeSwitch = mainTab:AddSwitch("Set Size", function(enabled)
        autoSizeEnabled = enabled
    end)
    sizeSwitch:Set(false)

    task.spawn(function()
        while true do
            if autoSizeEnabled then
                local character = localPlayer.Character
                if character and character:FindFirstChildOfClass("Humanoid") then
                    changeSpeedSizeRemote:InvokeServer("changeSize", targetSize)
                end
            end
            task.wait(0.15)
        end
    end)

    local targetSpeed = 120
    local autoSpeedEnabled = false

    mainTab:AddTextBox("Speed", function(input)
        local cleaned = input:gsub("%s+", "")
        local num = tonumber(cleaned)
        if num and num > 0 then
            targetSpeed = num
        end
    end)

    local speedSwitch = mainTab:AddSwitch("Set Speed", function(enabled)
        autoSpeedEnabled = enabled
    end)
    speedSwitch:Set(false)

    task.spawn(function()
        while true do
            if autoSpeedEnabled then
                local character = localPlayer.Character
                if character and character:FindFirstChildOfClass("Humanoid") then
                    changeSpeedSizeRemote:InvokeServer("changeSpeed", targetSpeed)
                end
            end
            task.wait(0.15)
        end
    end)

    local protectionLabel = mainTab:AddLabel("ðŸ›¡ï¸Player Protection and Visuals:")
    task.spawn(function()
        while true do
            protectionLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    local antiFlingSwitch = mainTab:AddSwitch("Anti Fling", function(enabled)
        local character = workspace:FindFirstChild(localPlayer.Name)
        if enabled then
            if character then
                local rootPart = character:FindFirstChild("HumanoidRootPart")
                if rootPart then
                    local bodyVelocity = Instance.new("BodyVelocity")
                    bodyVelocity.MaxForce = Vector3.new(100000, 0, 100000)
                    bodyVelocity.Velocity = Vector3.new(0, 0, 0)
                    bodyVelocity.P = 1250
                    bodyVelocity.Parent = rootPart
                end
            end
        else
            if character then
                local rootPart = character:FindFirstChild("HumanoidRootPart")
                if rootPart then
                    local bodyVelocity = rootPart:FindFirstChild("BodyVelocity")
                    if bodyVelocity and bodyVelocity.MaxForce == Vector3.new(100000, 0, 100000) then
                        bodyVelocity:Destroy()
                    end
                end
            end
        end
    end)
    antiFlingSwitch:Set(true)

    local lockPosSwitch = mainTab:AddSwitch("Lock Position", function(enabled)
        lockPositionEnabled = enabled
        if enabled then
            local character = localPlayer.Character
            if not character then
                character = localPlayer.CharacterAdded:Wait()
            end
            lockedRootPart = character:WaitForChild("HumanoidRootPart")
            lockedPosition = lockedRootPart.Position
            lockPositionCoroutine = coroutine.create(function()
                while lockPositionEnabled do
                    lockedRootPart.Velocity = Vector3.new(0, 0, 0)
                    lockedRootPart.RotVelocity = Vector3.new(0, 0, 0)
                    lockedRootPart.CFrame = CFrame.new(lockedPosition)
                    wait(0.05)
                end
            end)
            coroutine.resume(lockPositionCoroutine)
        end
    end)
    lockPosSwitch:Set(false)

    local showPetsSwitch = mainTab:AddSwitch("Show Pets", function(enabled)
        local player = playerService.LocalPlayer
        if player:FindFirstChild("hidePets") then
            player.hidePets.Value = enabled
        end
    end)
    showPetsSwitch:Set(false)

    local showOtherPetsSwitch = mainTab:AddSwitch("Show Other Pets", function(enabled)
        local player = playerService.LocalPlayer
        if player:FindFirstChild("showOtherPetsOn") then
            player.showOtherPetsOn.Value = enabled
        end
    end)
    showOtherPetsSwitch:Set(false)

    local statFrameNames = {
        "strengthFrame",
        "durabilityFrame",
        "agilityFrame",
        "evilKarmaFrame",
        "goodKarmaFrame",
    }

    local hideStatFramesSwitch = mainTab:AddSwitch("Hide Stat Frames", function(enabled)
        if enabled then
            for _, frameName in ipairs(statFrameNames) do
                local frame = replicatedStorage:FindFirstChild(frameName)
                if frame and frame:IsA("GuiObject") then
                    frame.Visible = false
                end
            end
            replicatedStorage.ChildAdded:Connect(function(child)
                if table.find(statFrameNames, child.Name) and child:IsA("GuiObject") then
                    child.Visible = false
                end
            end)
        end
    end)
    hideStatFramesSwitch:Set(false)

    task.spawn(function()
        local gridCount = math.ceil(waterGridRange / waterPartSize)
        local function createWaterPart(position, name)
            local part = Instance.new("Part")
            part.Size = Vector3.new(waterPartSize, 1, waterPartSize)
            part.Position = position
            part.Anchored = true
            part.Transparency = 1
            part.CanCollide = true
            part.Name = name
            part.Parent = workspace
            return part
        end
        for x = 1, gridCount do
            for z = 1, gridCount do
                table.insert(waterParts, createWaterPart(waterBasePosition + Vector3.new(x * waterPartSize, 0, z * waterPartSize), "Part_Side_" .. x .. "_" .. z))
                table.insert(waterParts, createWaterPart(waterBasePosition + Vector3.new(-x * waterPartSize, 0, z * waterPartSize), "Part_LeftRight_" .. x .. "_" .. z))
                table.insert(waterParts, createWaterPart(waterBasePosition + Vector3.new(-x * waterPartSize, 0, -z * waterPartSize), "Part_UpLeft_" .. x .. "_" .. z))
                table.insert(waterParts, createWaterPart(waterBasePosition + Vector3.new(x * waterPartSize, 0, -z * waterPartSize), "Part_UpRight_" .. x .. "_" .. z))
            end
        end
    end)

    local walkOnWaterSwitch = mainTab:AddSwitch("Walk on Water", function(enabled)
        for _, part in ipairs(waterParts) do
            if part and part.Parent then
                part.CanCollide = enabled
            end
        end
    end)
    walkOnWaterSwitch:Set(true)

    local miscLabel = mainTab:AddLabel("âš™ï¸ Misc Options")
    task.spawn(function()
        while true do
            miscLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    mainTab:AddSwitch("Infinite Jump", function(enabled)
        _G.InfiniteJump = enabled
        if enabled then
            infiniteJumpConnection = game:GetService("UserInputService").JumpRequest:Connect(function()
                if _G.InfiniteJump then
                    local character = playerService.LocalPlayer.Character
                    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
                    if humanoid then
                        humanoid:ChangeState("Jumping")
                    end
                else
                    if infiniteJumpConnection then
                        infiniteJumpConnection:Disconnect()
                    end
                end
            end)
        end
    end)

    mainTab:AddSwitch("Auto Spin Fortune Wheel", function(enabled)
        _G.AutoSpinWheel = enabled
        if enabled then
            spawn(function()
                while _G.AutoSpinWheel do
                    pcall(function()
                        replicatedStorage.rEvents.openFortuneWheelRemote:InvokeServer(
                            "openFortuneWheel",
                            replicatedStorage.fortuneWheelChances["Fortune Wheel"]
                        )
                    end)
                    wait(1)
                end
            end)
        end
    end)

    local timeDropdown = mainTab:AddDropdown("Change Time", function(selected)
        if selected == "Night" then
            lighting.ClockTime = 0
        elseif selected == "Day" then
            lighting.ClockTime = 12
        elseif selected == "Midnight" then
            lighting.ClockTime = 6
        end
    end)
    timeDropdown:Add("Night")
    timeDropdown:Add("Day")
    timeDropdown:Add("Midnight")

    local statsTab = mainWindow:AddTab("Stats")

    local statsHeaderLabel = statsTab:AddLabel("ðŸ“ŠPlayer Stats:")
    task.spawn(function()
        while true do
            statsHeaderLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    local playerDropdown = statsTab:AddDropdown("Choose Player", function(selected)
        for _, player in ipairs(getAllPlayers()) do
            if selected == player.DisplayName .. " | " .. player.Name then
                trackedPlayer = player
                break
            end
        end
    end)

    for _, player in ipairs(getAllPlayers()) do
        playerDropdown:Add(player.DisplayName .. " | " .. player.Name)
    end

    playerService.PlayerAdded:Connect(function(player)
        playerDropdown:Add(player.DisplayName .. " | " .. player.Name)
    end)

    playerService.PlayerRemoving:Connect(function(player)
        playerDropdown:Clear()
        for _, plr in ipairs(getAllPlayers()) do
            playerDropdown:Add(plr.DisplayName .. " | " .. plr.Name)
        end
    end)

    local nameLabel = statsTab:AddLabel("Name: N/A")
    nameLabel.TextSize = 20
    local usernameLabel = statsTab:AddLabel("Username: N/A")
    usernameLabel.TextSize = 20

    local statLabels = {}
    for _, def in ipairs(statDefinitions) do
        statLabels[def.name] = statsTab:AddLabel((statIcons[def.name] or "") .. " " .. def.name .. ": N/A")
        statLabels[def.name].TextSize = 20
    end

    local function updateStatLabels(player)
        if not player then return end
        nameLabel.Text = "Name: " .. player.DisplayName
        usernameLabel.Text = "Username: " .. player.Name

        local playerLeaderstats = player:FindFirstChild("leaderstats")
        if not playerLeaderstats then return end

        for _, def in ipairs(statDefinitions) do
            local statObj = playerLeaderstats:FindFirstChild(def.statName) or player:FindFirstChild(def.statName)
            if statObj then
                local value = statObj.Value
                statLabels[def.name].Text = string.format(
                    "%s %s: %s (%s)",
                    statIcons[def.name] or "",
                    def.name,
                    formatNumberShort(value),
                    formatNumberComma(value)
                )
            else
                statLabels[def.name].Text = (statIcons[def.name] or "") .. " " .. def.name .. ": 0 (0)"
            end
        end
    end

    task.spawn(function()
        while true do
            if trackedPlayer then
                updateStatLabels(trackedPlayer)
            end
            task.wait(0.2)
        end
    end)

    statsTab:AddLabel(dashSeparator)
    local advancedStatsLabel = statsTab:AddLabel("Advanced Stats:")
    advancedStatsLabel.TextSize = 24

    local enemyHealthLabel = statsTab:AddLabel("Enemy Health: N/A")
    enemyHealthLabel.TextSize = 20
    enemyHealthLabel.TextColor3 = Color3.fromRGB(0, 140, 255)

    local playerDamageLabel = statsTab:AddLabel("Your Damage: N/A")
    playerDamageLabel.TextSize = 20
    playerDamageLabel.TextColor3 = Color3.fromRGB(255, 0, 0)

    local hitsToKillLabel = statsTab:AddLabel("Hits to Kill: N/A")
    hitsToKillLabel.TextSize = 20
    hitsToKillLabel.TextColor3 = Color3.fromRGB(255, 0, 0)

    local function updateCombatInfo(player)
        if not player then
            enemyHealthLabel.Text = "Enemy Health: N/A"
            playerDamageLabel.Text = "Your Damage: N/A"
            hitsToKillLabel.Text = "Hits to Kill: N/A"
            return
        end
        local health = getEnemyHealth(player)
        local damage = getPlayerDamage()
        enemyHealthLabel.Text = string.format("Enemy Health: %s (%s)", formatNumberShort(health), formatNumberComma(health))
        playerDamageLabel.Text = string.format("Your Damage: %s (%s)", formatNumberShort(damage), formatNumberComma(damage))
        hitsToKillLabel.Text = string.format("Hits to Kill: %s", tostring(getHitsToKill(health, damage)))
    end

    task.spawn(function()
        while true do
            if trackedPlayer then
                updateCombatInfo(trackedPlayer)
            else
                updateCombatInfo(nil)
            end
            task.wait(0.1)
        end
    end)

    local combatTab = mainWindow:AddTab("Combat")

    local combatHeaderLabel = combatTab:AddLabel("âš”ï¸ Combat Options")
    task.spawn(function()
        while true do
            combatHeaderLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    local petDropdown = combatTab:AddDropdown("Equip Unique Pet", function(petName)
        pcall(function()
            local petsFolder = localPlayer.petsFolder
            for _, folder in pairs(petsFolder:GetChildren()) do
                if folder:IsA("Folder") then
                    for _, pet in pairs(folder:GetChildren()) do
                        replicatedStorage.rEvents.equipPetEvent:FireServer("unequipPet", pet)
                    end
                end
            end
        end)
        task.wait(0.2)

        local matchingPets = {}
        pcall(function()
            local uniqueFolder = localPlayer.petsFolder.Unique
            for _, pet in pairs(uniqueFolder:GetChildren()) do
                if pet.Name == petName then
                    table.insert(matchingPets, pet)
                end
            end
        end)

        for i = 1, math.min(8, #matchingPets) do
            replicatedStorage.rEvents.equipPetEvent:FireServer("equipPet", matchingPets[i])
            task.wait(0.1)
        end
    end)
    petDropdown:Add("Swift Samurai")
    petDropdown:Add("Tribal Overlord")

    local punchAnimIds = {
        ["rbxassetid://3638729053"] = true,
        ["rbxassetid://3638767427"] = true,
    }

    local function stopPunchAnimations()
        local character = localPlayer.Character
        if not character or not character:FindFirstChild("Humanoid") then return end
        local humanoid = character:FindFirstChild("Humanoid")

        for _, track in pairs(humanoid:GetPlayingAnimationTracks()) do
            if track.Animation then
                local trackName = track.Name:lower()
                if punchAnimIds[track.Animation.AnimationId] or trackName:match("punch") or trackName:match("attack") or trackName:match("right") then
                    track:Stop()
                end
            end
        end

        _G.AnimBlockConnection = humanoid.AnimationPlayed:Connect(function(track)
            if track.Animation then
                local trackName = track.Name:lower()
                if punchAnimIds[track.Animation.AnimationId] or trackName:match("punch") or trackName:match("attack") or trackName:match("right") then
                    track:Stop()
                end
            end
        end)
    end

    local function setupToolAnimBlock(tool)
        if not tool then return end
        if tool:GetAttribute("ActivatedOverride") then return end
        tool:SetAttribute("ActivatedOverride", true)

        _G.ToolConnections = _G.ToolConnections or {}
        _G.ToolConnections[tool] = tool.Activated:Connect(function()
            task.wait(0.05)
            local character = localPlayer.Character
            if character and character:FindFirstChild("Humanoid") then
                for _, track in pairs(character.Humanoid:GetPlayingAnimationTracks()) do
                    if track.Animation then
                        local trackName = track.Name:lower()
                        if punchAnimIds[track.Animation.AnimationId] or trackName:match("punch") or trackName:match("attack") or trackName:match("right") then
                            track:Stop()
                        end
                    end
                end
            end
        end)
    end

    local function setupAllToolAnimBlocks()
        for _, tool in pairs(localPlayer.Backpack:GetChildren()) do
            setupToolAnimBlock(tool)
        end
        local character = localPlayer.Character
        if character then
            for _, child in pairs(character:GetChildren()) do
                if child:IsA("Tool") then
                    setupToolAnimBlock(child)
                end
            end
        end
        _G.BackpackAddedConnection = localPlayer.Backpack.ChildAdded:Connect(function(child)
            if child:IsA("Tool") then
                task.wait(0.1)
                setupToolAnimBlock(child)
            end
        end)
        if character then
            _G.CharacterToolAddedConnection = character.ChildAdded:Connect(function(child)
                if child:IsA("Tool") then
                    task.wait(0.1)
                    setupToolAnimBlock(child)
                end
            end)
        end
    end

    local animBlockSwitch = combatTab:AddSwitch("Block Punch Animations", function(enabled)
        if enabled then
            _G.AnimMonitorConnection = runService.Heartbeat:Connect(function()
                if tick() % 0.5 < 0.01 then
                    local character = localPlayer.Character
                    if character and character:FindFirstChild("Humanoid") then
                        for _, track in pairs(character.Humanoid:GetPlayingAnimationTracks()) do
                            if track.Animation then
                                local trackName = track.Name:lower()
                                if punchAnimIds[track.Animation.AnimationId] or trackName:match("punch") or trackName:match("attack") or trackName:match("right") then
                                    track:Stop()
                                end
                            end
                        end
                    end
                end
            end)

            _G.CharacterAddedConnection = localPlayer.CharacterAdded:Connect(function()
                task.wait(1)
                stopPunchAnimations()
                setupAllToolAnimBlocks()
                if _G.CharacterToolAddedConnection then
                    _G.CharacterToolAddedConnection:Disconnect()
                end
                local character = localPlayer.Character
                if character then
                    _G.CharacterToolAddedConnection = character.ChildAdded:Connect(function(child)
                        if child:IsA("Tool") then
                            task.wait(0.1)
                            setupToolAnimBlock(child)
                        end
                    end)
                end
            end)

            stopPunchAnimations()
            setupAllToolAnimBlocks()
        else
            if _G.AnimBlockConnection then
                _G.AnimBlockConnection:Disconnect()
                _G.AnimBlockConnection = nil
            end
            if _G.AnimMonitorConnection then
                _G.AnimMonitorConnection:Disconnect()
                _G.AnimMonitorConnection = nil
            end
            if _G.CharacterAddedConnection then
                _G.CharacterAddedConnection:Disconnect()
                _G.CharacterAddedConnection = nil
            end
            if _G.BackpackAddedConnection then
                _G.BackpackAddedConnection:Disconnect()
                _G.BackpackAddedConnection = nil
            end
            if _G.CharacterToolAddedConnection then
                _G.CharacterToolAddedConnection:Disconnect()
                _G.CharacterToolAddedConnection = nil
            end
            if _G.ToolConnections then
                for tool, connection in pairs(_G.ToolConnections) do
                    if connection then connection:Disconnect() end
                    if tool and tool:GetAttribute("ActivatedOverride") then
                        tool:SetAttribute("ActivatedOverride", nil)
                    end
                end
                _G.ToolConnections = nil
            end
        end
    end)
    animBlockSwitch:Set(false)

    local proteinEggEnabled = false
    local proteinEggCoroutine = nil
    local proteinCharConnection = nil

    local function manageProteinEgg()
        if not proteinEggEnabled or not localPlayer.Character then return end
        local character = localPlayer.Character
        local eggCount = 0

        for _, child in ipairs(character:GetChildren()) do
            if child.Name == "Protein Egg" then
                eggCount = eggCount + 1
                if eggCount > 1 then
                    child.Parent = localPlayer.Backpack
                end
            end
        end

        if eggCount == 0 then
            local backpackEgg = localPlayer.Backpack:FindFirstChild("Protein Egg")
            if backpackEgg then
                backpackEgg.Parent = localPlayer.Character
            end
        end
    end

    local proteinEggSwitch = combatTab:AddSwitch("Auto Protein Egg", function(enabled)
        proteinEggEnabled = enabled
        if enabled then
            changeSpeedSizeRemote:InvokeServer("changeSize", 0 / 0)
            proteinEggCoroutine = task.spawn(function()
                while proteinEggEnabled do
                    manageProteinEgg()
                    task.wait(0.2)
                end
            end)
            proteinCharConnection = localPlayer.CharacterAdded:Connect(function()
                task.wait(0.5)
                manageProteinEgg()
            end)
            manageProteinEgg()
        else
            if proteinEggCoroutine then
                task.cancel(proteinEggCoroutine)
            end
            if proteinCharConnection then
                proteinCharConnection:Disconnect()
            end
        end
    end)
    proteinEggSwitch:Set(false)

    combatTab:AddButton("Load External Script", function()
        loadstring(game:HttpGet("https://raw.githubusercontent.com/244ihssp/IlIIS/refs/heads/main/1"))()
    end)

    local externalScriptLabel = combatTab:AddLabel("External Script Status")
    task.spawn(function()
        while true do
            externalScriptLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    _G.whitelistedPlayers = _G.whitelistedPlayers or {}
    _G.blacklistedPlayers = _G.blacklistedPlayers or {}

    local whitelistDropdown = combatTab:AddDropdown("Add to Whitelist", function(selected)
        local name = selected:match("| (.+)$")
        if name then
            name = name:gsub("^%s*(.-)%s*$", "%1")
            for _, existing in ipairs(_G.whitelistedPlayers) do
                if existing:lower() == name:lower() then return end
            end
            table.insert(_G.whitelistedPlayers, name)
        end
    end)

    local killAllSwitch = combatTab:AddSwitch("Kill All (Except Whitelist)", function(enabled)
        _G.killAll = enabled
        if enabled then
            if not _G.killAllConnection then
                _G.killAllConnection = runService.Heartbeat:Connect(function()
                    if _G.killAll then
                        for _, player in ipairs(playerService:GetPlayers()) do
                            if player ~= localPlayer and not isWhitelistedPlayer(player) then
                                attackPlayer(player)
                            end
                        end
                    end
                end)
            end
        else
            if _G.killAllConnection then
                _G.killAllConnection:Disconnect()
                _G.killAllConnection = nil
            end
        end
    end)
    killAllSwitch:Set(false)

    local whitelistFriendsSwitch = combatTab:AddSwitch("Whitelist Friends", function(enabled)
        _G.whitelistFriends = enabled
        if enabled then
            for _, player in pairs(playerService:GetPlayers()) do
                if player ~= localPlayer and player:IsFriendsWith(localPlayer.UserId) then
                    local playerName = player.Name
                    local alreadyExists = false
                    for _, existing in ipairs(_G.whitelistedPlayers) do
                        if existing:lower() == playerName:lower() then
                            alreadyExists = true
                            break
                        end
                    end
                    if not alreadyExists then
                        table.insert(_G.whitelistedPlayers, playerName)
                    end
                end
            end
            playerService.PlayerAdded:Connect(function(player)
                if _G.whitelistFriends and player:IsFriendsWith(localPlayer.UserId) then
                    local playerName = player.Name
                    local alreadyExists = false
                    for _, existing in ipairs(_G.whitelistedPlayers) do
                        if existing:lower() == playerName:lower() then
                            alreadyExists = true
                            break
                        end
                    end
                    if not alreadyExists then
                        table.insert(_G.whitelistedPlayers, playerName)
                    end
                end
            end)
        end
    end)
    whitelistFriendsSwitch:Set(false)

    local combatInfoLabel = combatTab:AddLabel("Combat Target Info")
    task.spawn(function()
        while true do
            combatInfoLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)

    local blacklistDropdown = combatTab:AddDropdown("Add to Blacklist", function(selected)
        local name = selected:match("| (.+)$")
        if name then
            name = name:gsub("^%s*(.-)%s*$", "%1")
            for _, existing in ipairs(_G.blacklistedPlayers) do
                if existing:lower() == name:lower() then return end
            end
            table.insert(_G.blacklistedPlayers, name)
        end
    end)

    for _, player in ipairs(getAllPlayers()) do
        if player ~= localPlayer then
            local entry = getPlayerDisplayEntry(player)
            whitelistDropdown:Add(entry)
            blacklistDropdown:Add(entry)
        end
    end

    playerService.PlayerAdded:Connect(function(player)
        if player ~= localPlayer then
            local entry = getPlayerDisplayEntry(player)
            whitelistDropdown:Add(entry)
            blacklistDropdown:Add(entry)
        end
    end)

    local killBlacklistSwitch = combatTab:AddSwitch("Kill Blacklisted Only", function(enabled)
        _G.killBlacklistedOnly = enabled
        if enabled then
            if not _G.blacklistKillConnection then
                _G.blacklistKillConnection = runService.Heartbeat:Connect(function()
                    if _G.killBlacklistedOnly then
                        for _, player in ipairs(playerService:GetPlayers()) do
                            if player ~= localPlayer and isBlacklistedPlayer(player) then
                                attackPlayer(player)
                            end
                        end
                    end
                end)
            end
        else
            if _G.blacklistKillConnection then
                _G.blacklistKillConnection:Disconnect()
                _G.blacklistKillConnection = nil
            end
        end
    end)

    local function setSpectateTarget(player)
        if spectateConnection then
            spectateConnection:Disconnect()
        end
        if player and player.Character then
            local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
            if humanoid then
                currentCamera.CameraSubject = player.Character
                spectateConnection = player.CharacterAdded:Connect(function(newChar)
                    task.wait(0.2)
                    local newHumanoid = newChar:FindFirstChildOfClass("Humanoid")
                    if newHumanoid then
                        currentCamera.CameraSubject = newChar
                    end
                end)
            end
        end
    end

    local spectateDropdown = combatTab:AddDropdown("Spectate Player", function(selected)
        for _, player in ipairs(getAllPlayers()) do
            if selected == player.DisplayName .. " | " .. player.Name then
                spectateTarget = player
                if spectateEnabled then
                    setSpectateTarget(player)
                end
                break
            end
        end
    end)

    local spectateSwitch = combatTab:AddSwitch("Spectate", function(enabled)
        spectateEnabled = enabled
        if enabled and spectateTarget then
            setSpectateTarget(spectateTarget)
        else
            if spectateConnection then
                spectateConnection:Disconnect()
                spectateConnection = nil
            end
            local character = localPlayer.Character
            if character then
                local humanoid = character:FindFirstChildOfClass("Humanoid")
                if humanoid then
                    currentCamera.CameraSubject = humanoid
                end
            end
        end
    end)
    
    for _, player in ipairs(getAllPlayers()) do
        local entry = player.DisplayName .. " | " .. player.Name
        spectateDropdown:Add(entry)
    end
    
    playerService.PlayerAdded:Connect(function(player)
        spectateDropdown:Add(player.DisplayName .. " | " .. player.Name)
    end)
    
    playerService.PlayerRemoving:Connect(function(player)
        if spectateTarget and spectateTarget == player then
            spectateTarget = nil
            if spectateEnabled then
                spectateSwitch:Set(false)
            end
        end
    end)
    
    local combatInfoLabel2 = combatTab:AddLabel("âš”ï¸ Combat Info")
    combatInfoLabel2.TextSize = 22
    task.spawn(function()
        while true do
            combatInfoLabel2.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)
    
    local deathRingPart = nil
    local deathRingColor = Color3.fromRGB(0, 0, 0)
    local deathRingTransparency = 0.6
    _G.showDeathRing = false
    _G.deathRingRange = 20
    
    local function updateDeathRingSize()
        if not deathRingPart then return end
        local diameter = (_G.deathRingRange or 20) * 2
        deathRingPart.Size = Vector3.new(0.2, diameter, diameter)
    end
    
    combatTab:AddTextBox("Death Ring Range", function(input)
        local num = tonumber(input)
        if num then
            _G.deathRingRange = math.clamp(num, 1, 140)
            updateDeathRingSize()
        end
    end)
    
    local function createOrDestroyDeathRing()
        if _G.showDeathRing then
            deathRingPart = Instance.new("Part")
            deathRingPart.Shape = Enum.PartType.Cylinder
            deathRingPart.Material = Enum.Material.Neon
            deathRingPart.Color = deathRingColor
            deathRingPart.Transparency = deathRingTransparency
            deathRingPart.Anchored = true
            deathRingPart.CanCollide = false
            deathRingPart.CastShadow = false
            updateDeathRingSize()
            deathRingPart.Parent = workspace
        else
            if deathRingPart then
                deathRingPart:Destroy()
                deathRingPart = nil
            end
        end
    end
    
    local function updateDeathRingPosition()
        if not deathRingPart then return end
        local character = getLocalCharacter()
        local rootPart = character and character:FindFirstChild("HumanoidRootPart")
        if rootPart then
            deathRingPart.CFrame = rootPart.CFrame * CFrame.Angles(0, 0, math.rad(90))
        end
    end
    
    local deathRingSwitch = combatTab:AddSwitch("Death Ring Kill", function(enabled)
        _G.deathRingEnabled = enabled
        if enabled then
            if not _G.deathRingConnection then
                _G.deathRingConnection = runService.Heartbeat:Connect(function()
                    updateDeathRingPosition()
                    local myCharacter = getLocalCharacter()
                    if not (myCharacter and myCharacter:FindFirstChild("HumanoidRootPart")) then return end
                    local myPosition = myCharacter.HumanoidRootPart.Position
    
                    for _, player in ipairs(playerService:GetPlayers()) do
                        if player ~= localPlayer and not isWhitelistedPlayer(player) then
                            pcall(function()
                                if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
                                    local distance = (myPosition - player.Character.HumanoidRootPart.Position).Magnitude
                                    if distance <= (_G.deathRingRange or 20) then
                                        attackPlayer(player)
                                    end
                                end
                            end)
                        end
                    end
                end)
            end
        else
            if _G.deathRingConnection then
                _G.deathRingConnection:Disconnect()
                _G.deathRingConnection = nil
            end
        end
    end)
    
    local showDeathRingSwitch = combatTab:AddSwitch("Show Death Ring", function(enabled)
        _G.showDeathRing = enabled
        createOrDestroyDeathRing()
    end)
    
    deathRingSwitch:Set(false)
    showDeathRingSwitch:Set(false)
    
    local whitelistInfoLabel = combatTab:AddLabel("Whitelist: None")
    whitelistInfoLabel.TextColor3 = Color3.fromRGB(26, 122, 212)
    whitelistInfoLabel.TextSize = 17
    
    combatTab:AddButton("Clear Whitelist", function()
        _G.whitelistedPlayers = {}
    end)
    
    local blacklistInfoLabel = combatTab:AddLabel("Killlist: None")
    blacklistInfoLabel.TextColor3 = Color3.fromRGB(191, 58, 25)
    blacklistInfoLabel.TextSize = 17
    
    combatTab:AddButton("Clear Blacklist", function()
        _G.blacklistedPlayers = {}
    end)
    
    local function updateWhitelistDisplay()
        if #_G.whitelistedPlayers == 0 then
            whitelistInfoLabel.Text = "Whitelist: None"
        else
            whitelistInfoLabel.Text = "Whitelist: " .. table.concat(_G.whitelistedPlayers, ", ")
        end
    end
    
    local function updateBlacklistDisplay()
        if #_G.blacklistedPlayers == 0 then
            blacklistInfoLabel.Text = "Killlist: None"
        else
            blacklistInfoLabel.Text = "Killlist: " .. table.concat(_G.blacklistedPlayers, ", ")
        end
    end
    
    task.spawn(function()
        while true do
            updateWhitelistDisplay()
            updateBlacklistDisplay()
            task.wait(0.2)
        end
    end)
    
    local farmTab = mainWindow:AddTab("Farm")
    
    local farmHeaderLabel = farmTab:AddLabel("ðŸŒ¾ Auto Farm")
    farmHeaderLabel.TextSize = 22
    task.spawn(function()
        while true do
            farmHeaderLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)
    
    local fakeGamepassSwitch = farmTab:AddSwitch("Fake Gamepasses", function(enabled)
        if enabled then
            pcall(function()
                local gamepassIds = replicatedStorage.gamepassIds
                for _, gp in pairs(gamepassIds:GetChildren()) do
                    local clone = Instance.new("IntValue")
                    clone.Name = gp.Name
                    clone.Value = gp.Value
                    clone.Parent = localPlayer.ownedGamepasses
                end
            end)
        else
            pcall(function()
                local ownedGamepasses = localPlayer.ownedGamepasses
                local gamepassIds = replicatedStorage.gamepassIds
                for _, gp in pairs(gamepassIds:GetChildren()) do
                    local existing = ownedGamepasses:FindFirstChild(gp.Name)
                    if existing and existing.Value == gp.Value then
                        existing:Destroy()
                    end
                end
            end)
        end
    end)
    
    local farmStatusLabel = farmTab:AddLabel("Farm Status")
    task.spawn(function()
        while true do
            farmStatusLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)
    
    local rebirthsCurrentLabel = farmTab:AddLabel("Rebirths: 0")
    rebirthsCurrentLabel.TextSize = 22
    task.spawn(function()
        while true do
            rebirthsCurrentLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)
    
    local rebirthsTargetLabel = farmTab:AddLabel("Target Rebirths: 0")
    rebirthsTargetLabel.TextSize = 17
    
    local rebirthStatValue = leaderstats:WaitForChild("Rebirths")
    local targetRebirths = 0
    local autoRebirthRunning = false
    local autoRebirthSwitch = nil
    
    local function formatWithCommas(num)
        local formatted = tostring(floorFn(num))
        local result = ""
        local count = 0
        for i = #formatted, 1, -1 do
            count = count + 1
            result = formatted:sub(i, i) .. result
            if count % 3 == 0 and i > 1 then
                result = "," .. result
            end
        end
        return result
    end
    
    local function autoRebirthLoop()
        autoRebirthRunning = true
        while autoRebirthRunning do
            if rebirthStatValue.Value >= targetRebirths then
                autoRebirthRunning = false
                if autoRebirthSwitch then
                    autoRebirthSwitch:Set(false)
                end
                break
            end
            replicatedStorage.rEvents.rebirthRemote:InvokeServer("rebirthRequest")
            task.wait(0.05)
        end
    end
    
    farmTab:AddTextBox("Target Rebirths", function(input)
        local num = tonumber(input)
        if num and num >= 0 then
            targetRebirths = num
            print("Target set to: " .. targetRebirths)
        else
            print("Invalid.")
        end
    end)
    
    autoRebirthSwitch = farmTab:AddSwitch("Auto Rebirth", function(enabled)
        if enabled then
            if targetRebirths > 0 and rebirthStatValue.Value < targetRebirths then
                task.spawn(autoRebirthLoop)
            else
                autoRebirthSwitch:Set(false)
            end
        else
            autoRebirthRunning = false
        end
    end)
    
    task.spawn(function()
        while true do
            rebirthsCurrentLabel.Text = "Rebirths: " .. formatWithCommas(rebirthStatValue.Value)
            rebirthsTargetLabel.Text = "Target Rebirths: " .. formatWithCommas(targetRebirths)
            task.wait(0.2)
        end
    end)
    
    local autoSize1Enabled = false
    farmTab:AddSwitch("Auto Size 1", function(enabled)
        autoSize1Enabled = enabled
    end)
    task.spawn(function()
        while true do
            if autoSize1Enabled then
                local character = localPlayer.Character
                if character and character:FindFirstChildOfClass("Humanoid") then
                    changeSpeedSizeRemote:InvokeServer("changeSize", 1)
                end
            end
            task.wait(0.05)
        end
    end)
    
    local kingCFrame = CFrame.new(-8665.4, 17.21, -5792.9)
    local autoKingEnabled = false
    
    farmTab:AddSwitch("Auto King", function(enabled)
        autoKingEnabled = enabled
    end)
    
    task.spawn(function()
        local character = localPlayer.Character
        if not character then
            character = localPlayer.CharacterAdded:Wait()
        end
        character:WaitForChild("HumanoidRootPart")
        while true do
            if autoKingEnabled then
                local rootPart = character and character:FindFirstChild("HumanoidRootPart")
                if rootPart and (rootPart.Position - kingCFrame.Position).Magnitude > 5 then
                    rootPart.CFrame = kingCFrame
                end
            end
            task.wait(0.05)
        end
    end)
    
    local kingInfoLabel = farmTab:AddLabel("ðŸ‘‘ King Position Lock")
    kingInfoLabel.TextSize = 22
    task.spawn(function()
        while true do
            kingInfoLabel.TextColor3 = Color3.fromHSV(tick() * 0.2 % 1, 0.9, 1)
            task.wait(0.02)
        end
    end)
    
    local selectedExercise = nil
    local autoExerciseEnabled = false
    local machineDropdown = nil
    
    farmTab:AddDropdown("Select Exercise", function(selected)
        selectedExercise = selected
    end)
    
    local exerciseDropdown = farmTab:AddDropdown("Select Exercise", function(selected)
        selectedExercise = selected
    end)
    exerciseDropdown:Add("Weight")
    exerciseDropdown:Add("Pushups")
    exerciseDropdown:Add("Situps")
    exerciseDropdown:Add("Handstands")
    exerciseDropdown:Add("Fast Punch")
    exerciseDropdown:Add("Stomp")
    exerciseDropdown:Add("Ground Slam")
    
    farmTab:AddSwitch("Auto Exercise", function(enabled)
        autoExerciseEnabled = enabled
        if enabled then
            task.spawn(function()
                while autoExerciseEnabled do
                    local player = localPlayer
                    local character = player.Character
    
                    if selectedExercise == "Weight" then
                        if character and not character:FindFirstChild("Weight") then
                            local tool = player.Backpack:FindFirstChild("Weight")
                            if tool then
                                character.Humanoid:EquipTool(tool)
                            end
                        end
                        player.muscleEvent:FireServer("rep")
    
                    elseif selectedExercise == "Pushups" then
                        if character and not character:FindFirstChild("Pushups") then
                            local tool = player.Backpack:FindFirstChild("Pushups")
                            if tool then
                                character.Humanoid:EquipTool(tool)
                            end
                        end
                        player.muscleEvent:FireServer("rep")
    
                    elseif selectedExercise == "Situps" then
                        if character and not character:FindFirstChild("Situps") then
                            local tool = player.Backpack:FindFirstChild("Situps")
                            if tool then
                                character.Humanoid:EquipTool(tool)
                            end
                        end
                        player.muscleEvent:FireServer("rep")
    
                    elseif selectedExercise == "Handstands" then
                        if character and not character:FindFirstChild("Handstands") then
                            local tool = player.Backpack:FindFirstChild("Handstands")
                            if tool then
