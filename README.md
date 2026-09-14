--// GREEN PORTAL GUN
--// One LocalScript
--// Put inside StarterPlayer > StarterPlayerScripts
--// For your own Roblox Studio experience

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

local RANGE = 1200
local PORTAL_COLOR = Color3.fromRGB(0, 255, 70)
local MENU_COLOR = Color3.fromRGB(8, 15, 12)

local character
local humanoid
local root
local gun

local portalA
local portalB
local portalMode = "A"

local portalsEnabled = true
local aimAssistEnabled = false
local shiftLockEnabled = false
local gunEquipped = false
local lastTeleport = 0

local gui
local menu
local openButton
local gunButton
local aimButton
local shiftButton
local portalButton
local closeButton
local minimizeButton
local statusLabel

local portalFolder = Instance.new("Folder")
portalFolder.Name = "LocalPortalEffects"
portalFolder.Parent = workspace

local portalConnections = {}

local function disconnectPortalConnections()
	for _, connection in ipairs(portalConnections) do
		if connection then
			connection:Disconnect()
		end
	end

	table.clear(portalConnections)
end

local function setupCharacter(char)
	character = char
	humanoid = char:WaitForChild("Humanoid")
	root = char:WaitForChild("HumanoidRootPart")
end

setupCharacter(player.Character or player.CharacterAdded:Wait())

local function playSound(id, parent, volume, speed)
	local sound = Instance.new("Sound")
	sound.SoundId = id
	sound.Volume = volume or 0.7
	sound.PlaybackSpeed = speed or 1
	sound.Parent = parent or workspace
	sound:Play()
	Debris:AddItem(sound, 4)
end

local function updateStatus(text)
	if statusLabel then
		statusLabel.Text = text
	end
end

local function updateToggle(button, enabled, label)
	if enabled then
		button.Text = label .. ": ON  ✓"
		button.BackgroundColor3 = Color3.fromRGB(0, 190, 60)
		button.TextColor3 = Color3.fromRGB(255, 255, 255)
	else
		button.Text = label .. ": OFF  +"
		button.BackgroundColor3 = Color3.fromRGB(18, 35, 24)
		button.TextColor3 = Color3.fromRGB(180, 255, 190)
	end
end

local function clearPortalReferences()
	disconnectPortalConnections()

	if portalFolder then
		portalFolder:ClearAllChildren()
	end

	portalA = nil
	portalB = nil
	portalMode = "A"
end

local function closeAllPortals()
	clearPortalReferences()
	updateStatus("PORTALS CLEARED")
	playSound("rbxassetid://6026984224", root, 0.8, 1.2)
end

local function createPortal(position, normal)
	local center = position + normal * 0.08

	local up = normal.Unit
	local right = up:Cross(Vector3.new(0, 1, 0))

	if right.Magnitude < 0.05 then
		right = up:Cross(Vector3.new(1, 0, 0))
	end

	right = right.Unit
	local back = right:Cross(up).Unit
	local portalCFrame = CFrame.fromMatrix(center, right, up, back)

	local model = Instance.new("Model")
	model.Name = "Portal"
	model.Parent = portalFolder

	local ring = Instance.new("Part")
	ring.Name = "GreenRoundPortal"
	ring.Shape = Enum.PartType.Cylinder
	ring.Size = Vector3.new(0.25, 8, 8)
	ring.CFrame = portalCFrame
	ring.Anchored = true
	ring.CanCollide = false
	ring.CanTouch = false
	ring.CanQuery = false
	ring.Material = Enum.Material.Neon
	ring.Color = PORTAL_COLOR
	ring.Transparency = 0.05
	ring.Parent = model

	local core = Instance.new("Part")
	core.Name = "PortalEnergy"
	core.Shape = Enum.PartType.Cylinder
	core.Size = Vector3.new(0.12, 6.8, 6.8)
	core.CFrame = portalCFrame * CFrame.new(-0.15, 0, 0)
	core.Anchored = true
	core.CanCollide = false
	core.CanTouch = false
	core.CanQuery = false
	core.Material = Enum.Material.Neon
	core.Color = Color3.fromRGB(30, 255, 90)
	core.Transparency = 0.35
	core.Parent = model

	local light = Instance.new("PointLight")
	light.Color = PORTAL_COLOR
	light.Brightness = 5
	light.Range = 18
	light.Parent = ring

	local particles = Instance.new("ParticleEmitter")
	particles.Color = ColorSequence.new(PORTAL_COLOR)
	particles.LightEmission = 1
	particles.Rate = 55
	particles.Lifetime = NumberRange.new(0.3, 0.8)
	particles.Speed = NumberRange.new(1, 4)
	particles.SpreadAngle = Vector2.new(360, 360)
	particles.Size = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0.15),
		NumberSequenceKeypoint.new(1, 0)
	})
	particles.Parent = ring

	local rotationConnection
	rotationConnection = RunService.RenderStepped:Connect(function()
		if not model.Parent then
			rotationConnection:Disconnect()
			return
		end

		local rotation = CFrame.Angles(0, math.rad(1.5), 0)
		ring.CFrame = ring.CFrame * rotation
		core.CFrame = core.CFrame * rotation
	end)

	table.insert(portalConnections, rotationConnection)

	playSound("rbxassetid://6026984224", ring, 1, 0.8)

	return model
end

local function getAimResult()
	local viewport = camera.ViewportSize

	local ray = camera:ViewportPointToRay(
		viewport.X / 2,
		viewport.Y / 2
	)

	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.IgnoreWater = true
	params.FilterDescendantsInstances = {
		character,
		gun,
		portalFolder
	}

	local result = workspace:Raycast(
		ray.Origin,
		ray.Direction * RANGE,
		params
	)

	if result then
		return result.Position, result.Normal
	end

	return ray.Origin + ray.Direction * RANGE, -ray.Direction
end

local function firePortal()
	if not portalsEnabled then
		updateStatus("PORTALS ARE OFF")
		return
	end

	if not gunEquipped or not gun then
		updateStatus("EQUIP THE PORTAL GUN")
		return
	end

	local position, normal = getAimResult()

	if not position or not normal then
		return
	end

	local newPortal = createPortal(position, normal)

	if portalMode == "A" then
		if portalA then
			portalA:Destroy()
		end

		portalA = newPortal
		portalMode = "B"
		updateStatus("PORTAL A SET — PORTAL B NEXT")
	else
		if portalB then
			portalB:Destroy()
		end

		portalB = newPortal
		portalMode = "A"
		updateStatus("PORTAL A + B LINKED")
	end
end

local function getPortalPosition(portal)
	if not portal or not portal.Parent then
		return nil
	end

	local part = portal:FindFirstChild("GreenRoundPortal")
	if part then
		return part.Position
	end

	return nil
end

local function getPortalCFrame(portal)
	if not portal or not portal.Parent then
		return nil
	end

	local part = portal:FindFirstChild("GreenRoundPortal")
	if part then
		return part.CFrame
	end

	return nil
end

local function teleportTo(targetPortal)
	if not root or not targetPortal then
		return
	end

	if os.clock() - lastTeleport < 1 then
		return
	end

	local targetCFrame = getPortalCFrame(targetPortal)

	if not targetCFrame then
		return
	end

	lastTeleport = os.clock()

	root.CFrame =
		targetCFrame
		* CFrame.new(0, 0, -5)
		* CFrame.Angles(0, math.rad(180), 0)

	playSound("rbxassetid://6026984224", root, 1, 1.8)
end

local function checkTeleport()
	if not root or not portalA or not portalB then
		return
	end

	local positionA = getPortalPosition(portalA)
	local positionB = getPortalPosition(portalB)

	if not positionA or not positionB then
		return
	end

	if (root.Position - positionA).Magnitude < 5 then
		teleportTo(portalB)
	elseif (root.Position - positionB).Magnitude < 5 then
		teleportTo(portalA)
	end
end

local function removeOldGuns()
	local backpack = player:FindFirstChildOfClass("Backpack")

	if character then
		for _, item in ipairs(character:GetChildren()) do
			if item:IsA("Tool") and item.Name == "Green Portal Gun" then
				item:Destroy()
			end
		end
	end

	if backpack then
		for _, item in ipairs(backpack:GetChildren()) do
			if item:IsA("Tool") and item.Name == "Green Portal Gun" then
				item:Destroy()
			end
		end
	end
end

local function createGun()
	removeOldGuns()

	local backpack = player:WaitForChild("Backpack")

	gun = Instance.new("Tool")
	gun.Name = "Green Portal Gun"
	gun.RequiresHandle = true
	gun.CanBeDropped = false
	gun.ToolTip = "Tap or click to fire a portal"

	local handle = Instance.new("Part")
	handle.Name = "Handle"
	handle.Size = Vector3.new(0.6, 1.6, 0.6)
	handle.Material = Enum.Material.Metal
	handle.Color = Color3.fromRGB(25, 25, 25)
	handle.CanCollide = false
	handle.Parent = gun

	local barrel = Instance.new("Part")
	barrel.Name = "GreenBarrel"
	barrel.Size = Vector3.new(0.75, 1.4, 0.75)
	barrel.Material = Enum.Material.Neon
	barrel.Color = PORTAL_COLOR
	barrel.CanCollide = false
	barrel.Parent = gun

	local weld = Instance.new("WeldConstraint")
	weld.Part0 = handle
	weld.Part1 = barrel
	weld.Parent = handle

	barrel.CFrame = handle.CFrame * CFrame.new(0, 0.8, 0)

	local light = Instance.new("PointLight")
	light.Color = PORTAL_COLOR
	light.Brightness = 3
	light.Range = 8
	light.Parent = barrel

	gun.Activated:Connect(function()
		if not gunEquipped then
			return
		end

		local oldSize = barrel.Size
		barrel.Size = Vector3.new(1, 1.8, 1)

		task.delay(0.08, function()
			if barrel and barrel.Parent then
				barrel.Size = oldSize
			end
		end)

		firePortal()
	end)

	gun.Equipped:Connect(function()
		gunEquipped = true
		updateStatus("PORTAL GUN READY")
	end)

	gun.Unequipped:Connect(function()
		gunEquipped = false
	end)

	gun.Parent = backpack
	updateStatus("PORTAL GUN ADDED")
end

local function createButton(parent, text, position, size)
	local button = Instance.new("TextButton")
	button.Name = text
	button.Text = text
	button.Position = position
	button.Size = size
	button.BackgroundColor3 = Color3.fromRGB(18, 35, 24)
	button.BorderSizePixel = 0
	button.TextColor3 = Color3.fromRGB(180, 255, 190)
	button.Font = Enum.Font.GothamBold
	button.TextSize = 13
	button.AutoButtonColor = true
	button.Parent = parent

	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 10)
	corner.Parent = button

	local stroke = Instance.new("UIStroke")
	stroke.Color = PORTAL_COLOR
	stroke.Thickness = 1
	stroke.Transparency = 0.2
	stroke.Parent = button

	return button
end

local function animateMenu(open)
	if open then
		menu.Visible = true
		menu.Position = UDim2.new(0, 20, 0, -350)

		TweenService:Create(
			menu,
			TweenInfo.new(0.35, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
			{
				Position = UDim2.new(0, 20, 0, 80)
			}
		):Play()

		openButton.Visible = false
	else
		local tween = TweenService:Create(
			menu,
			TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.In),
			{
				Position = UDim2.new(0, 20, 0, -350)
			}
		)

		tween:Play()

		tween.Completed:Connect(function()
			menu.Visible = false
			openButton.Visible = true
		end)
	end
end

local function createGui()
	gui = Instance.new("ScreenGui")
	gui.Name = "PortalGunInterface"
	gui.ResetOnSpawn = false
	gui.IgnoreGuiInset = true
	gui.Parent = player:WaitForChild("PlayerGui")

	menu = Instance.new("Frame")
	menu.Name = "PortalMenu"
	menu.Size = UDim2.new(0, 285, 0, 315)
	menu.Position = UDim2.new(0, 20, 0, 80)
	menu.BackgroundColor3 = MENU_COLOR
	menu.BorderSizePixel = 0
	menu.Parent = gui

	local menuCorner = Instance.new("UICorner")
	menuCorner.CornerRadius = UDim.new(0, 16)
	menuCorner.Parent = menu

	local menuStroke = Instance.new("UIStroke")
	menuStroke.Color = PORTAL_COLOR
	menuStroke.Thickness = 2
	menuStroke.Transparency = 0.15
	menuStroke.Parent = menu

	local title = Instance.new("TextLabel")
	title.Size = UDim2.new(1, -150, 0, 35)
	title.Position = UDim2.new(0, 12, 0, 10)
	title.BackgroundTransparency = 1
	title.Text = "RICK PORTAL SYSTEM"
	title.TextColor3 = PORTAL_COLOR
	title.Font = Enum.Font.GothamBlack
	title.TextSize = 15
	title.TextXAlignment = Enum.TextXAlignment.Left
	title.Parent = menu

	statusLabel = Instance.new("TextLabel")
	statusLabel.Size = UDim2.new(1, -24, 0, 25)
	statusLabel.Position = UDim2.new(0, 12, 0, 48)
	statusLabel.BackgroundTransparency = 1
	statusLabel.Text = "SYSTEM READY"
	statusLabel.TextColor3 = Color3.fromRGB(190, 255, 200)
	statusLabel.Font = Enum.Font.Gotham
	statusLabel.TextSize = 12
	statusLabel.Parent = menu

	gunButton = createButton(
		menu,
		"GET PORTAL GUN",
		UDim2.new(0, 15, 0, 85),
		UDim2.new(1, -30, 0, 40)
	)

	aimButton = createButton(
		menu,
		"AIM ASSIST: OFF  +",
		UDim2.new(0, 15, 0, 130),
		UDim2.new(1, -30, 0, 40)
	)

	shiftButton = createButton(
		menu,
		"SHIFTLOCK: OFF  +",
		UDim2.new(0, 15, 0, 175),
		UDim2.new(1, -30, 0, 40)
	)

	portalButton = createButton(
		menu,
		"PORTALS: ON  ✓",
		UDim2.new(0, 15, 0, 220),
		UDim2.new(1, -30, 0, 40)
	)

	closeButton = createButton(
		menu,
		"CLOSE ALL PORTALS",
		UDim2.new(0, 15, 0, 265),
		UDim2.new(1, -30, 0, 35)
	)

	minimizeButton = createButton(
		menu,
		"—",
		UDim2.new(1, -48, 0, 12),
		UDim2.new(0, 30, 0, 24)
	)

	minimizeButton.TextSize = 15
	minimizeButton.TextColor3 = PORTAL_COLOR

	openButton = createButton(
		gui,
		"OPEN PORTAL MENU",
		UDim2.new(0, 20, 0, 80),
		UDim2.new(0, 170, 0, 40)
	)

	openButton.Visible = false

	gunButton.MouseButton1Click:Connect(createGun)

	aimButton.MouseButton1Click:Connect(function()
		aimAssistEnabled = not aimAssistEnabled
		updateToggle(aimButton, aimAssistEnabled, "AIM ASSIST")

		if aimAssistEnabled then
			updateStatus("AIM ASSIST ENABLED")
		else
			updateStatus("AIM ASSIST DISABLED")
		end
	end)

	shiftButton.MouseButton1Click:Connect(function()
		shiftLockEnabled = not shiftLockEnabled
		updateToggle(shiftButton, shiftLockEnabled, "SHIFTLOCK")
	end)

	portalButton.MouseButton1Click:Connect(function()
		portalsEnabled = not portalsEnabled
		updateToggle(portalButton, portalsEnabled, "PORTALS")

		if not portalsEnabled then
			closeAllPortals()
		end
	end)

	closeButton.MouseButton1Click:Connect(closeAllPortals)

	minimizeButton.MouseButton1Click:Connect(function()
		animateMenu(false)
	end)

	openButton.MouseButton1Click:Connect(function()
		animateMenu(true)
	end)

	updateToggle(aimButton, aimAssistEnabled, "AIM ASSIST")
	updateToggle(shiftButton, shiftLockEnabled, "SHIFTLOCK")
	updateToggle(portalButton, portalsEnabled, "PORTALS")
end

createGui()

UserInputService.InputBegan:Connect(function(input, processed)
	if processed then
		return
	end

	if input.KeyCode == Enum.KeyCode.RightShift then
		animateMenu(not menu.Visible)
	end

	if input.KeyCode == Enum.KeyCode.LeftShift then
		shiftLockEnabled = not shiftLockEnabled
		updateToggle(shiftButton, shiftLockEnabled, "SHIFTLOCK")
	end
end)

RunService.RenderStepped:Connect(function()
	if not character or not root or not humanoid then
		return
	end

	if shiftLockEnabled then
		UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter
		humanoid.AutoRotate = false

		local direction = camera.CFrame.LookVector
		local flatDirection = Vector3.new(
			direction.X,
			0,
			direction.Z
		)

		if flatDirection.Magnitude > 0.05 then
			root.CFrame = CFrame.lookAt(
				root.Position,
				root.Position + flatDirection.Unit
			)
		end
	else
		UserInputService.MouseBehavior = Enum.MouseBehavior.Default
		humanoid.AutoRotate = true
	end
end)

RunService.Heartbeat:Connect(checkTeleport)

player.CharacterAdded:Connect(function(char)
	setupCharacter(char)

	closeAllPortals()

	gun = nil
	gunEquipped = false

	task.wait(1)
	createGun()
end)

task.wait(1)
createGun()
