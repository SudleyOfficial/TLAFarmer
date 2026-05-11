local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local UserInputService = game:GetService("UserInputService")

local localPlayer = Players.LocalPlayer
local playerGui = localPlayer:WaitForChild("PlayerGui")
local Live = workspace:WaitForChild("Live")

local STUDS_UNDER = -1
local TWEEN_TIME = 0.01
local KEY_SPAM_DELAY = 0.1
local FOLLOW_DELAY = 0.01

local enabled = false
local keyLoopRunning = false
local followLoopRunning = false
local sessionStartTime = 0
local currentTargetPlayer = nil
local whitelistLookup = {}
local whitelistOrder = {}
local playerButtons = {}
local whitelistButtons = {}
local amountBox

local THEME_BLACK = Color3.fromRGB(10, 10, 12)
local THEME_PANEL = Color3.fromRGB(18, 18, 22)
local THEME_PANEL_ALT = Color3.fromRGB(26, 26, 31)
local THEME_RED = Color3.fromRGB(210, 34, 42)
local THEME_RED_DARK = Color3.fromRGB(120, 18, 24)
local THEME_WHITE = Color3.fromRGB(245, 245, 245)
local THEME_MUTED = Color3.fromRGB(170, 170, 176)
local THEME_OUTLINE = Color3.fromRGB(95, 20, 26)
local SIDE_DROPDOWN_WIDTH = 180

local dragging = false
local dragStart
local startPos

local function formatSessionTime(totalSeconds)
	local seconds = math.max(0, math.floor(totalSeconds))
	local minutes = math.floor(seconds / 60)
	local remainingSeconds = seconds % 60

	if minutes > 0 then
		return string.format("%dm %ds", minutes, remainingSeconds)
	end

	return string.format("%ds", remainingSeconds)
end

local function getCurrentKaiju()
	local leaderstats = localPlayer:FindFirstChild("leaderstats")
	local kaijuValue = leaderstats and leaderstats:FindFirstChild("Kaiju")
	return kaijuValue and kaijuValue.Value or nil
end

local function getAttackKeycodes()
	local keycodes = {
		Enum.KeyCode.One,
		Enum.KeyCode.Two,
		Enum.KeyCode.Three,
	}

	local currentKaiju = getCurrentKaiju()
	if currentKaiju == "Biollante" then
		local studsUnder = tonumber(amountBox.Text) or STUDS_UNDER
		if studsUnder >= 15 and studsUnder <= 25 then
			return {
				Enum.KeyCode.Two,
				Enum.KeyCode.Three,
			}
		end
	end

	if currentKaiju == "Kong 2017" then
		table.insert(keycodes, Enum.KeyCode.Four)
		table.insert(keycodes, Enum.KeyCode.Five)
		return keycodes
	end

	if currentKaiju == "Kong 2021" then
		return {
			Enum.KeyCode.One,
			Enum.KeyCode.Two,
			Enum.KeyCode.Four,
			Enum.KeyCode.Five,
		}
	end

	if currentKaiju == "Godzilla Minus One" then
		local character = localPlayer.Character
		local humanoid = character and character:FindFirstChildOfClass("Humanoid")
		if humanoid and humanoid.MaxHealth > 0 and (humanoid.Health / humanoid.MaxHealth) <= 0.3 then
			table.insert(keycodes, Enum.KeyCode.Four)
			table.insert(keycodes, Enum.KeyCode.Four)
			table.insert(keycodes, Enum.KeyCode.Four)
		end
	end

	if currentKaiju == "Shin Godzilla" then
		table.insert(keycodes, Enum.KeyCode.Five)
	end

	return keycodes
end

local gui = Instance.new("ScreenGui")
gui.Name = "AltFarmerGui"
gui.ResetOnSpawn = false
gui.Parent = playerGui

local shadow = Instance.new("Frame")
shadow.Size = UDim2.new(0, 320, 0, 460)
shadow.Position = UDim2.new(0, 25, 0, 25)
shadow.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
shadow.BackgroundTransparency = 0.35
shadow.BorderSizePixel = 0
shadow.Parent = gui

local shadowCorner = Instance.new("UICorner")
shadowCorner.CornerRadius = UDim.new(0, 22)
shadowCorner.Parent = shadow

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 320, 0, 460)
frame.Position = UDim2.new(0, 20, 0, 20)
frame.BackgroundColor3 = THEME_BLACK
frame.BorderSizePixel = 0
frame.Parent = gui

local frameCorner = Instance.new("UICorner")
frameCorner.CornerRadius = UDim.new(0, 20)
frameCorner.Parent = frame

local frameStroke = Instance.new("UIStroke")
frameStroke.Color = THEME_OUTLINE
frameStroke.Thickness = 2
frameStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
frameStroke.Parent = frame

local header = Instance.new("Frame")
header.Size = UDim2.new(1, -24, 0, 58)
header.Position = UDim2.new(0, 12, 0, 10)
header.BackgroundColor3 = THEME_PANEL
header.BorderSizePixel = 0
header.Parent = frame

local headerCorner = Instance.new("UICorner")
headerCorner.CornerRadius = UDim.new(0, 16)
headerCorner.Parent = header

local headerStroke = Instance.new("UIStroke")
headerStroke.Color = THEME_OUTLINE
headerStroke.Thickness = 1.5
headerStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
headerStroke.Parent = header

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -18, 0, 26)
title.Position = UDim2.new(0, 10, 0, 7)
title.BackgroundTransparency = 1
title.Font = Enum.Font.GothamBold
title.TextSize = 20
title.TextXAlignment = Enum.TextXAlignment.Left
title.Text = "Kaiju Alpha"
title.TextColor3 = THEME_WHITE
title.Parent = header

local creditLabel = Instance.new("TextLabel")
creditLabel.Size = UDim2.new(1, -18, 0, 18)
creditLabel.Position = UDim2.new(0, 10, 0, 32)
creditLabel.BackgroundTransparency = 1
creditLabel.Font = Enum.Font.GothamSemibold
creditLabel.TextSize = 11
creditLabel.TextXAlignment = Enum.TextXAlignment.Left
creditLabel.TextColor3 = THEME_MUTED
creditLabel.Text = "ALT FARMER"
creditLabel.Parent = header

local page = Instance.new("Frame")
page.Size = UDim2.new(1, -24, 1, -86)
page.Position = UDim2.new(0, 12, 0, 78)
page.BackgroundTransparency = 1
page.Parent = frame

local toggleButton = Instance.new("TextButton")
toggleButton.Size = UDim2.new(1, 0, 0, 40)
toggleButton.Position = UDim2.new(0, 0, 0, 0)
toggleButton.BackgroundColor3 = THEME_RED_DARK
toggleButton.TextColor3 = THEME_WHITE
toggleButton.Font = Enum.Font.GothamBold
toggleButton.TextSize = 16
toggleButton.Text = "OFF"
toggleButton.Parent = page

local toggleCorner = Instance.new("UICorner")
toggleCorner.CornerRadius = UDim.new(0, 12)
toggleCorner.Parent = toggleButton

local toggleStroke = Instance.new("UIStroke")
toggleStroke.Color = THEME_RED
toggleStroke.Thickness = 1.5
toggleStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
toggleStroke.Parent = toggleButton

amountBox = Instance.new("TextBox")
amountBox.Size = UDim2.new(1, 0, 0, 36)
amountBox.Position = UDim2.new(0, 0, 0, 48)
amountBox.BackgroundColor3 = THEME_PANEL
amountBox.TextColor3 = THEME_WHITE
amountBox.PlaceholderColor3 = THEME_MUTED
amountBox.Font = Enum.Font.GothamSemibold
amountBox.TextSize = 15
amountBox.PlaceholderText = "Studs under target"
amountBox.Text = tostring(STUDS_UNDER)
amountBox.Parent = page

local amountCorner = Instance.new("UICorner")
amountCorner.CornerRadius = UDim.new(0, 12)
amountCorner.Parent = amountBox

local amountStroke = Instance.new("UIStroke")
amountStroke.Color = THEME_OUTLINE
amountStroke.Thickness = 1.5
amountStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
amountStroke.Parent = amountBox

local dropdownButton = Instance.new("TextButton")
dropdownButton.Size = UDim2.new(1, 0, 0, 36)
dropdownButton.Position = UDim2.new(0, 0, 0, 92)
dropdownButton.BackgroundColor3 = THEME_PANEL_ALT
dropdownButton.TextColor3 = THEME_WHITE
dropdownButton.Font = Enum.Font.GothamSemibold
dropdownButton.TextSize = 15
dropdownButton.Text = "Add Whitelist Player"
dropdownButton.Parent = page

local dropdownCorner = Instance.new("UICorner")
dropdownCorner.CornerRadius = UDim.new(0, 12)
dropdownCorner.Parent = dropdownButton

local dropdownStroke = Instance.new("UIStroke")
dropdownStroke.Color = THEME_OUTLINE
dropdownStroke.Thickness = 1.5
dropdownStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
dropdownStroke.Parent = dropdownButton

local dropdownFrame = Instance.new("ScrollingFrame")
dropdownFrame.Size = UDim2.new(0, 0, 0, 120)
dropdownFrame.Position = UDim2.new(0, -176, 0, 92)
dropdownFrame.BackgroundColor3 = THEME_PANEL
dropdownFrame.BorderSizePixel = 0
dropdownFrame.ScrollBarThickness = 4
dropdownFrame.ScrollBarImageColor3 = THEME_RED
dropdownFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
dropdownFrame.Visible = false
dropdownFrame.Parent = page

local dropdownFrameCorner = Instance.new("UICorner")
dropdownFrameCorner.CornerRadius = UDim.new(0, 12)
dropdownFrameCorner.Parent = dropdownFrame

local dropdownFrameStroke = Instance.new("UIStroke")
dropdownFrameStroke.Color = THEME_OUTLINE
dropdownFrameStroke.Thickness = 1.5
dropdownFrameStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
dropdownFrameStroke.Parent = dropdownFrame

local listLayout = Instance.new("UIListLayout")
listLayout.Padding = UDim.new(0, 4)
listLayout.Parent = dropdownFrame

local whitelistFrame = Instance.new("ScrollingFrame")
whitelistFrame.Size = UDim2.new(1, 0, 0, 118)
whitelistFrame.Position = UDim2.new(0, 0, 0, 136)
whitelistFrame.BackgroundColor3 = THEME_PANEL
whitelistFrame.BorderSizePixel = 0
whitelistFrame.ScrollBarThickness = 4
whitelistFrame.ScrollBarImageColor3 = THEME_RED
whitelistFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
whitelistFrame.Parent = page

local whitelistFrameCorner = Instance.new("UICorner")
whitelistFrameCorner.CornerRadius = UDim.new(0, 12)
whitelistFrameCorner.Parent = whitelistFrame

local whitelistFrameStroke = Instance.new("UIStroke")
whitelistFrameStroke.Color = THEME_OUTLINE
whitelistFrameStroke.Thickness = 1.5
whitelistFrameStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
whitelistFrameStroke.Parent = whitelistFrame

local whitelistListLayout = Instance.new("UIListLayout")
whitelistListLayout.Padding = UDim.new(0, 4)
whitelistListLayout.Parent = whitelistFrame

local whitelistLabel = Instance.new("TextLabel")
whitelistLabel.Size = UDim2.new(1, 0, 0, 18)
whitelistLabel.Position = UDim2.new(0, 0, 0, 258)
whitelistLabel.BackgroundTransparency = 1
whitelistLabel.TextColor3 = THEME_MUTED
whitelistLabel.Font = Enum.Font.GothamSemibold
whitelistLabel.TextSize = 13
whitelistLabel.TextXAlignment = Enum.TextXAlignment.Left
whitelistLabel.Text = "Whitelist: 0"
whitelistLabel.Parent = page

local sessionTimeLabel = Instance.new("TextLabel")
sessionTimeLabel.Size = UDim2.new(1, 0, 0, 20)
sessionTimeLabel.Position = UDim2.new(0, 0, 1, -42)
sessionTimeLabel.BackgroundTransparency = 1
sessionTimeLabel.TextColor3 = THEME_MUTED
sessionTimeLabel.Font = Enum.Font.GothamMedium
sessionTimeLabel.TextSize = 14
sessionTimeLabel.TextXAlignment = Enum.TextXAlignment.Left
sessionTimeLabel.Text = "Session Time: 0s"
sessionTimeLabel.Parent = page

local statusLabel = Instance.new("TextLabel")
statusLabel.Size = UDim2.new(1, 0, 0, 20)
statusLabel.Position = UDim2.new(0, 0, 1, -20)
statusLabel.BackgroundTransparency = 1
statusLabel.TextColor3 = THEME_RED
statusLabel.Font = Enum.Font.GothamSemibold
statusLabel.TextSize = 14
statusLabel.TextXAlignment = Enum.TextXAlignment.Left
statusLabel.Text = "Target: none"
statusLabel.Parent = page

local function updateSessionTimeLabel()
	if enabled and sessionStartTime > 0 then
		sessionTimeLabel.Text = "Session Time: " .. formatSessionTime(os.clock() - sessionStartTime)
	else
		sessionTimeLabel.Text = "Session Time: 0s"
	end
end

local function getLevelLabelFromLiveModel(model)
	local overhead = model and model:FindFirstChild("Overhead")
	local outerFrame = overhead and overhead:FindFirstChildOfClass("Frame")
	local levelFrame = outerFrame and outerFrame:FindFirstChild("Level")
	return levelFrame and levelFrame:FindFirstChildOfClass("TextLabel") or nil
end

local function getPlayerLevel(player)
	local liveModel = Live:FindFirstChild(player.Name)
	local levelLabel = getLevelLabelFromLiveModel(liveModel)
	return levelLabel and tonumber(string.match(levelLabel.Text, "%d+")) or nil
end

local function isAlive(player)
	local character = player and player.Character
	local humanoid = character and character:FindFirstChildOfClass("Humanoid")
	local hrp = character and character:FindFirstChild("HumanoidRootPart")
	return humanoid and humanoid.Health > 0 and hrp
end

local function isWhitelisted(player)
	return player and whitelistLookup[player.Name] == true
end

local function getWhitelistedPlayers()
	local whitelistedPlayers = {}

	for _, playerName in ipairs(whitelistOrder) do
		local player = Players:FindFirstChild(playerName)
		if player and player ~= localPlayer and isAlive(player) then
			table.insert(whitelistedPlayers, player)
		end
	end

	return whitelistedPlayers
end

local function getClosestWhitelistedPlayer()
	local myCharacter = localPlayer.Character
	local myHrp = myCharacter and myCharacter:FindFirstChild("HumanoidRootPart")
	if not myHrp then
		return nil
	end

	local closestPlayer = nil
	local closestDistance = math.huge

	for _, player in ipairs(getWhitelistedPlayers()) do
		local targetHrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
		if targetHrp then
			local distance = (myHrp.Position - targetHrp.Position).Magnitude
			if distance < closestDistance then
				closestDistance = distance
				closestPlayer = player
			end
		end
	end

	return closestPlayer
end

local function refreshStatus()
	updateSessionTimeLabel()

	if not isWhitelisted(currentTargetPlayer) or not isAlive(currentTargetPlayer) then
		currentTargetPlayer = getClosestWhitelistedPlayer()
	end

	whitelistLabel.Text = "Whitelist: " .. tostring(#whitelistOrder)

	if currentTargetPlayer then
		local level = getPlayerLevel(currentTargetPlayer)
		if level then
			statusLabel.Text = "Target: " .. currentTargetPlayer.Name .. " | Level: " .. level
		else
			statusLabel.Text = "Target: " .. currentTargetPlayer.Name
		end
	else
		statusLabel.Text = "Target: none"
	end
end

local function rebuildWhitelistFrame()
	for _, button in ipairs(whitelistButtons) do
		button:Destroy()
	end
	table.clear(whitelistButtons)

	local ySize = 0

	for _, playerName in ipairs(whitelistOrder) do
		local button = Instance.new("TextButton")
		button.Size = UDim2.new(1, -8, 0, 28)
		button.BackgroundColor3 = THEME_PANEL_ALT
		button.TextColor3 = THEME_WHITE
		button.Font = Enum.Font.GothamSemibold
		button.TextSize = 14
		button.Text = playerName .. "  [remove]"
		button.Parent = whitelistFrame

		local buttonCorner = Instance.new("UICorner")
		buttonCorner.CornerRadius = UDim.new(0, 10)
		buttonCorner.Parent = button

		local buttonStroke = Instance.new("UIStroke")
		buttonStroke.Color = THEME_OUTLINE
		buttonStroke.Thickness = 1.2
		buttonStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
		buttonStroke.Parent = button

		button.MouseButton1Click:Connect(function()
			whitelistLookup[playerName] = nil
			for index, name in ipairs(whitelistOrder) do
				if name == playerName then
					table.remove(whitelistOrder, index)
					break
				end
			end
			if currentTargetPlayer and currentTargetPlayer.Name == playerName then
				currentTargetPlayer = nil
			end
			rebuildWhitelistFrame()
			refreshStatus()
		end)

		table.insert(whitelistButtons, button)
		ySize += 32
	end

	whitelistFrame.CanvasSize = UDim2.new(0, 0, 0, ySize)
end

local function rebuildDropdown()
	for _, button in ipairs(playerButtons) do
		button:Destroy()
	end
	table.clear(playerButtons)

	local ySize = 0

	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= localPlayer and not whitelistLookup[player.Name] then
			local button = Instance.new("TextButton")
			button.Size = UDim2.new(1, -8, 0, 28)
			button.BackgroundColor3 = THEME_PANEL_ALT
			button.TextColor3 = THEME_WHITE
			button.Font = Enum.Font.GothamSemibold
			button.TextSize = 14
			button.Text = player.Name
			button.Parent = dropdownFrame

			local buttonCorner = Instance.new("UICorner")
			buttonCorner.CornerRadius = UDim.new(0, 10)
			buttonCorner.Parent = button

			local buttonStroke = Instance.new("UIStroke")
			buttonStroke.Color = THEME_OUTLINE
			buttonStroke.Thickness = 1.2
			buttonStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
			buttonStroke.Parent = button

			button.MouseButton1Click:Connect(function()
				whitelistLookup[player.Name] = true
				table.insert(whitelistOrder, player.Name)
				dropdownFrame.Visible = false
				dropdownFrame.Size = UDim2.new(0, 0, 0, 120)
				rebuildDropdown()
				rebuildWhitelistFrame()
				refreshStatus()
			end)

			table.insert(playerButtons, button)
			ySize += 32
		end
	end

	dropdownFrame.CanvasSize = UDim2.new(0, 0, 0, ySize)
	if dropdownFrame.Visible then
		dropdownFrame.Size = UDim2.new(0, SIDE_DROPDOWN_WIDTH, 0, math.min(ySize, 120))
	end
end

local function pressKey(keyCode)
	VirtualInputManager:SendKeyEvent(true, keyCode, false, game)
	task.wait(0.02)
	VirtualInputManager:SendKeyEvent(false, keyCode, false, game)
end

local function startKeyLoop()
	if keyLoopRunning then
		return
	end

	keyLoopRunning = true
	task.spawn(function()
		while enabled do
			local target = currentTargetPlayer
			local character = localPlayer.Character
			local hrp = character and character:FindFirstChild("HumanoidRootPart")
			local targetHrp = target and target.Character and target.Character:FindFirstChild("HumanoidRootPart")

			if hrp and targetHrp then
				for _, keyCode in ipairs(getAttackKeycodes()) do
					pressKey(keyCode)
				end
			end

			task.wait(KEY_SPAM_DELAY)
		end

		keyLoopRunning = false
	end)
end

local function startFollowLoop()
	if followLoopRunning then
		return
	end

	followLoopRunning = true
	task.spawn(function()
		while enabled do
			local target = getClosestWhitelistedPlayer()
			currentTargetPlayer = target

			local character = localPlayer.Character
			local hrp = character and character:FindFirstChild("HumanoidRootPart")
			local targetHrp = target and target.Character and target.Character:FindFirstChild("HumanoidRootPart")

			if hrp and targetHrp then
				local offset = tonumber(amountBox.Text) or STUDS_UNDER
				local goalCFrame = targetHrp.CFrame * CFrame.new(0, -offset, 0)
				local tween = TweenService:Create(
					hrp,
					TweenInfo.new(TWEEN_TIME, Enum.EasingStyle.Linear),
					{CFrame = goalCFrame}
				)
				tween:Play()
			end

			refreshStatus()
			task.wait(FOLLOW_DELAY)
		end

		followLoopRunning = false
	end)
end

header.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = input.Position
		startPos = frame.Position

		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				dragging = false
			end
		end)
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
		local delta = input.Position - dragStart
		frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
		shadow.Position = UDim2.new(frame.Position.X.Scale, frame.Position.X.Offset + 5, frame.Position.Y.Scale, frame.Position.Y.Offset + 5)
	end
end)

dropdownButton.MouseButton1Click:Connect(function()
	dropdownFrame.Visible = not dropdownFrame.Visible
	if dropdownFrame.Visible then
		rebuildDropdown()
		local canvasHeight = dropdownFrame.CanvasSize.Y.Offset
		dropdownFrame.Size = UDim2.new(0, SIDE_DROPDOWN_WIDTH, 0, math.min(canvasHeight, 120))
	else
		dropdownFrame.Size = UDim2.new(0, 0, 0, 120)
	end
end)

toggleButton.MouseButton1Click:Connect(function()
	enabled = not enabled

	if enabled then
		sessionStartTime = os.clock()
		toggleButton.Text = "ON"
		toggleButton.BackgroundColor3 = THEME_RED
		startKeyLoop()
		startFollowLoop()
	else
		sessionStartTime = 0
		currentTargetPlayer = nil
		toggleButton.Text = "OFF"
		toggleButton.BackgroundColor3 = THEME_RED_DARK
	end

	refreshStatus()
end)

Players.PlayerAdded:Connect(function()
	task.wait(0.2)
	rebuildDropdown()
	refreshStatus()
end)

Players.PlayerRemoving:Connect(function(player)
	task.wait(0.2)
	if whitelistLookup[player.Name] then
		rebuildWhitelistFrame()
	end
	if currentTargetPlayer and currentTargetPlayer.Name == player.Name then
		currentTargetPlayer = nil
	end
	rebuildDropdown()
	refreshStatus()
end)

rebuildDropdown()
rebuildWhitelistFrame()
refreshStatus()
