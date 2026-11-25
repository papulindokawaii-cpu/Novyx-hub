-- ServerScriptService/NovyxServer.lua
-- Servidor autoritativo para la zona de entrenamiento de Novyx Hub | v1

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")

local RemotesFolder = ReplicatedStorage:WaitForChild("NovyxRemotes")
local NovyxAction = RemotesFolder:WaitForChild("NovyxAction") :: RemoteEvent
local NovyxQuery = RemotesFolder:WaitForChild("NovyxQuery") :: RemoteFunction

-- Carpeta exclusiva para objetivos y elementos de práctica (no toca jugadores reales)
local TrainingFolder = Workspace:FindFirstChild("NovyxTraining")
if not TrainingFolder then
	TrainingFolder = Instance.new("Folder")
	TrainingFolder.Name = "NovyxTraining"
	TrainingFolder.Parent = Workspace
end

local TargetsFolder = TrainingFolder:FindFirstChild("Dummies")
if not TargetsFolder then
	TargetsFolder = Instance.new("Folder")
	TargetsFolder.Name = "Dummies"
	TargetsFolder.Parent = TrainingFolder
end

local PickupsFolder = TrainingFolder:FindFirstChild("PracticeItems")
if not PickupsFolder then
	PickupsFolder = Instance.new("Folder")
	PickupsFolder.Name = "PracticeItems"
	PickupsFolder.Parent = TrainingFolder
end

-- Estado de racha de práctica por jugador
local PlayerStats: { [Player]: { Kills: number, Streak: number, BestStreak: number } } = {}

local function getStats(player: Player)
	if not PlayerStats[player] then
		PlayerStats[player] = { Kills = 0, Streak = 0, BestStreak = 0 }
	end
	return PlayerStats[player]
end

local function pushStats(player: Player)
	local stats = getStats(player)
	NovyxAction:FireClient(player, "StatsUpdate", {
		Kills = stats.Kills,
		Streak = stats.Streak,
		BestStreak = stats.BestStreak,
	})
end

-- Crea un muñeco NPC ficticio de entrenamiento en una posición dada
local function spawnDummy(index: number, position: Vector3, style: string?)
	local model = Instance.new("Model")
	model.Name = string.format("PracticeDummy_%02d", index)
	model:SetAttribute("IsPracticeDummy", true)
	model:SetAttribute("TargetStyle", style or "Estandar")

	local root = Instance.new("Part")
	root.Name = "HumanoidRootPart"
	root.Size = Vector3.new(2, 2, 1)
	root.Position = position
	root.Anchored = false
	root.CanCollide = true
	root.Material = Enum.Material.SmoothPlastic
	root.Color = Color3.fromRGB(115, 65, 220)
	root.Parent = model

	local head = Instance.new("Part")
	head.Name = "Head"
	head.Size = Vector3.new(1.5, 1.5, 1.5)
	head.Position = position + Vector3.new(0, 2.25, 0)
	head.Anchored = false
	head.CanCollide = true
	head.Material = Enum.Material.Neon
	head.Color = Color3.fromRGB(65, 165, 255)
	head.Parent = model

	local torso = Instance.new("Part")
	torso.Name = "Torso"
	torso.Size = Vector3.new(2, 2, 1)
	torso.Position = position + Vector3.new(0, 1, 0)
	torso.Anchored = false
	torso.CanCollide = true
	torso.Material = Enum.Material.SmoothPlastic
	torso.Color = Color3.fromRGB(40, 42, 60)
	torso.Parent = model

	local weldTorso = Instance.new("WeldConstraint")
	weldTorso.Part0 = root
	weldTorso.Part1 = torso
	weldTorso.Parent = root

	local weldHead = Instance.new("WeldConstraint")
	weldHead.Part0 = torso
	weldHead.Part1 = head
	weldHead.Parent = torso

	local humanoid = Instance.new("Humanoid")
	humanoid.MaxHealth = 100
	humanoid.Health = 100
	humanoid.WalkSpeed = if style == "Movil" then 10 else 0
	humanoid.Parent = model

	model.PrimaryPart = root
	model.Parent = TargetsFolder

	-- Si es estilo "Movil", oscila suavemente dentro del área de práctica
	if style == "Movil" then
		task.spawn(function()
			local offset = index * 1.1
			while model.Parent and humanoid.Health > 0 do
				local t = os.clock() + offset
				if root.Parent then
					root.AssemblyLinearVelocity = Vector3.new(math.sin(t) * 8, root.AssemblyLinearVelocity.Y, math.cos(t) * 8)
				end
				task.wait(0.1)
			end
		end)
	end

	return model
end

local function ensureDummies(count: number, style: string?)
	for _, child in ipairs(TargetsFolder:GetChildren()) do
		child:Destroy()
	end
	local clamped = math.clamp(count, 1, 15)
	for i = 1, clamped do
		local angle = (i / clamped) * math.pi * 2
		local radius = 14 + (i % 3) * 4
		local pos = Vector3.new(math.cos(angle) * radius, 4, math.sin(angle) * radius)
		spawnDummy(i, pos, style)
	end
end

local function ensurePracticePickups()
	if #PickupsFolder:GetChildren() > 0 then
		return
	end
	local items = {
		{ Name = "Orbe de Impulso", Color = Color3.fromRGB(0, 210, 255), Pos = Vector3.new(8, 2.5, 8) },
		{ Name = "Cristal de Racha", Color = Color3.fromRGB(165, 70, 255), Pos = Vector3.new(-8, 2.5, 8) },
		{ Name = "Bateria de Salto", Color = Color3.fromRGB(110, 255, 160), Pos = Vector3.new(0, 2.5, -10) },
	}
	for _, info in ipairs(items) do
		local orb = Instance.new("Part")
		orb.Name = info.Name
		orb.Shape = Enum.PartType.Ball
		orb.Size = Vector3.new(2, 2, 2)
		orb.Position = info.Pos
		orb.Anchored = true
		orb.CanCollide = false
		orb.Material = Enum.Material.Neon
		orb.Color = info.Color
		orb:SetAttribute("PracticeItem", true)
		orb.Parent = PickupsFolder
	end
end

-- Inicialización del campo de práctica
ensureDummies(6, "Estandar")
ensurePracticePickups()

-- Entrega segura de objetos autorizados de prueba al jugador
local AllowedItems: { [string]: () -> Tool } = {
	["Bastón de Práctica"] = function()
		local tool = Instance.new("Tool")
		tool.Name = "Bastón de Práctica"
		tool.RequiresHandle = true

		local handle = Instance.new("Part")
		handle.Name = "Handle"
		handle.Size = Vector3.new(0.4, 4, 0.4)
		handle.Material = Enum.Material.Neon
		handle.Color = Color3.fromRGB(145, 75, 255)
		handle.Parent = tool

		tool.Activated:Connect(function()
			local char = tool.Parent
			local root = char and char:FindFirstChild("HumanoidRootPart") :: BasePart?
			if not root then
				return
			end
			-- Afecta únicamente a NPC dummies de prueba dentro de 10 studs
			for _, dummy in ipairs(TargetsFolder:GetChildren()) do
				if dummy:GetAttribute("IsPracticeDummy") then
					local dRoot = dummy:FindFirstChild("HumanoidRootPart") :: BasePart?
					local dHum = dummy:FindFirstChildOfClass("Humanoid")
					if dRoot and dHum and dHum.Health > 0 and (dRoot.Position - root.Position).Magnitude <= 10 then
						dHum:TakeDamage(100)
					end
				end
			end
		end)

		return tool
	end,

	["Radar de Campo"] = function()
		local tool = Instance.new("Tool")
		tool.Name = "Radar de Campo"
		tool.RequiresHandle = true

		local handle = Instance.new("Part")
		handle.Name = "Handle"
		handle.Size = Vector3.new(1.2, 0.4, 1.2)
		handle.Shape = Enum.PartType.Cylinder
		handle.Material = Enum.Material.Metal
		handle.Color = Color3.fromRGB(50, 175, 255)
		handle.Parent = tool

		return tool
	end,
}

NovyxAction.OnServerEvent:Connect(function(player: Player, action: string, payload: any)
	if typeof(action) ~= "string" then
		return
	end
	local stats = getStats(player)

	if action == "SpawnPracticeField" then
		local count = (typeof(payload) == "table" and typeof(payload.Count) == "number") and payload.Count or 6
		local style = (typeof(payload) == "table" and typeof(payload.Style) == "string") and payload.Style or "Estandar"
		ensureDummies(count, style)
		ensurePracticePickups()

		-- Teletransporta suavemente al jugador al borde del campo de práctica si tiene personaje
		local char = player.Character
		local root = char and char:FindFirstChild("HumanoidRootPart") :: BasePart?
		if root then
			root.CFrame = CFrame.new(0, 6, 22)
		end
		NovyxAction:FireClient(player, "Notify", "Campo de práctica listo con " .. tostring(math.clamp(count, 1, 15)) .. " NPCs.")

	elseif action == "KillPracticeDummies" then
		-- Elimina exclusivamente NPCs ficticios de prueba (nunca jugadores reales)
		local eliminated = 0
		for _, dummy in ipairs(TargetsFolder:GetChildren()) do
			if dummy:IsA("Model") and dummy:GetAttribute("IsPracticeDummy") then
				local hum = dummy:FindFirstChildOfClass("Humanoid")
				if hum and hum.Health > 0 then
					hum.Health = 0
					eliminated += 1
				end
				dummy:Destroy()
			end
		end

		if eliminated > 0 then
			stats.Kills += eliminated
			stats.Streak += eliminated
			if stats.Streak > stats.BestStreak then
				stats.BestStreak = stats.Streak
			end
		end

		-- Respawn automático para mantener el entrenamiento continuo
		task.delay(1.2, function()
			if #TargetsFolder:GetChildren() == 0 then
				ensureDummies(6, "Estandar")
			end
		end)

		pushStats(player)
		NovyxAction:FireClient(player, "Notify", "Eliminados " .. tostring(eliminated) .. " NPCs de prueba. Racha: " .. tostring(stats.Streak))

	elseif action == "ResetStreak" then
		stats.Streak = 0
		pushStats(player)
		NovyxAction:FireClient(player, "Notify", "Racha de entrenamiento reiniciada.")

	elseif action == "GivePracticeItem" then
		if typeof(payload) ~= "string" then
			return
		end
		local builder = AllowedItems[payload]
		local backpack = player:FindFirstChildOfClass("Backpack")
		if builder and backpack then
			local existing = backpack:FindFirstChild(payload) or (player.Character and player.Character:FindFirstChild(payload))
			if not existing then
				local tool = builder()
				tool.Parent = backpack
				NovyxAction:FireClient(player, "Notify", "Objeto equipado en mochila: " .. payload)
			else
				NovyxAction:FireClient(player, "Notify", "Ya tienes en tu inventario: " .. payload)
			end
		end
	end
end)

NovyxQuery.OnServerInvoke = function(player: Player, queryType: string)
	if queryType == "GetStats" then
		return getStats(player)
	end
	return nil
end

Players.PlayerAdded:Connect(function(player: Player)
	getStats(player)
	task.wait(1)
	pushStats(player)
end)

Players.PlayerRemoving:Connect(function(player: Player)
	PlayerStats[player] = nil
end)
-- StarterPlayer/StarterPlayerScripts/NovyxClientUI.lua
-- Interfaz y funcionalidades del cliente para Novyx Hub | v1 (Móvil y PC)

local Players = game:GetService("Players")
local Lighting = game:GetService("Lighting")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local SoundService = game:GetService("SoundService")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local camera = Workspace.CurrentCamera

local RemotesFolder = ReplicatedStorage:WaitForChild("NovyxRemotes")
local NovyxAction = RemotesFolder:WaitForChild("NovyxAction") :: RemoteEvent
local NovyxQuery = RemotesFolder:WaitForChild("NovyxQuery") :: RemoteFunction

-- Paleta de colores: Negro profundo, morado, azul neón y blanco
local Theme = {
	BgMain = Color3.fromRGB(11, 12, 18),
	BgPanel = Color3.fromRGB(18, 20, 30),
	BgCard = Color3.fromRGB(25, 28, 42),
	BgHover = Color3.fromRGB(34, 38, 58),
	AccentPurple = Color3.fromRGB(138, 66, 255),
	AccentBlue = Color3.fromRGB(48, 162, 255),
	TextWhite = Color3.fromRGB(245, 247, 255),
	TextMuted = Color3.fromRGB(155, 163, 190),
	Success = Color3.fromRGB(55, 215, 140),
	Danger = Color3.fromRGB(255, 80, 110),
	Stroke = Color3.fromRGB(68, 52, 120),
}

-- Estado centralizado del HUB
local State = {
	CustomDisplayName = player.DisplayName,
	CustomBio = "Entrenamiento y pruebas en Novyx Hub | v1",
	AccentColor = Theme.AccentPurple,
	GraphicsPreset = "Original",
	LightingEnabled = true,

	FloatingButtonsVisible = false,
	AutoSelectButtons = false,
	VisibleFloatingButtons = {
		["Vuelo"] = true,
		["Kill NPCs"] = true,
		["Invisible"] = true,
		["Emote"] = true,
	},
	SelectedEmote = "Saludo",
	EquippedEmotes = { "Saludo", "Baile", "Victoria" },
	EmoteSpamActive = false,

	TrainingDummyStyle = "Estandar",
	TrainingDummyCount = 6,
	AimIndicatorEnabled = false,
	AimSensitivity = 0.35,
	FovIndicatorEnabled = false,
	FovRadius = 140,

	EspEnabled = false,
	ObjectIndicatorsEnabled = false,
	VisualInfoEnabled = true,
	EspColor = Color3.fromRGB(64, 185, 255),
	MaxEspDistance = 150,
	ConfigurableMarkers = true,

	InvisibleEnabled = false,

	WalkSpeedValue = 16,
	JumpPowerValue = 50,
	FlyEnabled = false,
	FlySpeed = 45,

	Stats = { Kills = 0, Streak = 0, BestStreak = 0 },

	ParticlesEnabled = false,
	AuraEnabled = false,
	EffectColor = Color3.fromRGB(145, 70, 255),

	SelectedAnimation = "Salto Energico",
	CurrentTrack = nil :: AnimationTrack?,

	SoundEffectsEnabled = true,
	SoundVolume = 0.6,

	MusicIndex = 1,
	MusicPlaying = false,
	MusicVolume = 0.45,

	SelectedItemName = "Bastón de Práctica",
	SelectedExtraScript = "Contador de FPS y Latencia",
}

-- Guardar iluminación original para restaurar sin errores
local OriginalLighting = {
	Ambient = Lighting.Ambient,
	OutdoorAmbient = Lighting.OutdoorAmbient,
	Brightness = Lighting.Brightness,
	ClockTime = Lighting.ClockTime,
	FogEnd = Lighting.FogEnd,
	FogColor = Lighting.FogColor,
}

-- Catálogo de canciones (IDs de sonido de Roblox)
local Playlist = {
	{ Title = "Synth Pulse (Test)", Artist = "Novyx Audio", SoundId = "rbxassetid://1837849285" },
	{ Title = "Night Drift (Test)", Artist = "Novyx Audio", SoundId = "rbxassetid://1843404009" },
	{ Title = "Cyber Horizon (Test)", Artist = "Novyx Audio", SoundId = "rbxassetid://9043887091" },
}

local MusicSound = Instance.new("Sound")
MusicSound.Name = "NovyxMusicPlayer"
MusicSound.Looped = true
MusicSound.Volume = State.MusicVolume
MusicSound.Parent = SoundService

local SfxSound = Instance.new("Sound")
SfxSound.Name = "NovyxSfxPlayer"
SfxSound.SoundId = "rbxassetid://6895079853"
SfxSound.Volume = State.SoundVolume
SfxSound.Parent = SoundService

local function playClickSfx()
	if State.SoundEffectsEnabled then
		SfxSound.Volume = State.SoundVolume
		SfxSound:Play()
	end
end

-- Utilidad para redondear esquinas y bordes
local function addCorner(inst: GuiObject, radius: number)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, radius)
	c.Parent = inst
	return c
end

local function addStroke(inst: GuiObject, color: Color3?, thickness: number?)
	local s = Instance.new("UIStroke")
	s.Color = color or Theme.Stroke
	s.Thickness = thickness or 1.2
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Parent = inst
	return s
end

-- Función para hacer cualquier Frame arrastrable en móvil (Touch) y PC (Mouse)
local function makeDraggable(dragHandle: GuiObject, targetFrame: GuiObject, onDragEnd: (() -> ())?)
	local dragging = false
	local dragInput: InputObject? = nil
	local startInputPos = Vector2.zero
	local startFramePos = UDim2.new()

	dragHandle.InputBegan:Connect(function(input: InputObject)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			dragInput = input
			startInputPos = Vector2.new(input.Position.X, input.Position.Y)
			startFramePos = targetFrame.Position

			input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					dragging = false
					if onDragEnd then
						onDragEnd()
					end
				end
			end)
		end
	end)

	dragHandle.InputChanged:Connect(function(input: InputObject)
		if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
			dragInput = input
		end
	end)

	UserInputService.InputChanged:Connect(function(input: InputObject)
		if dragging and input == dragInput then
			local delta = Vector2.new(input.Position.X, input.Position.Y) - startInputPos
			targetFrame.Position = UDim2.new(
				startFramePos.X.Scale,
				startFramePos.X.Offset + delta.X,
				startFramePos.Y.Scale,
				startFramePos.Y.Offset + delta.Y
			)
		end
	end)
end

-- ScreenGui principal
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "NovyxHubGui"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.IgnoreGuiInset = true
ScreenGui.Parent = playerGui

-- Toast de notificación superior
local ToastLabel = Instance.new("TextLabel")
ToastLabel.Name = "Toast"
ToastLabel.Size = UDim2.new(0, 320, 0, 38)
ToastLabel.Position = UDim2.new(0.5, -160, 0, -50)
ToastLabel.BackgroundColor3 = Theme.BgCard
ToastLabel.TextColor3 = Theme.TextWhite
ToastLabel.Font = Enum.Font.GothamBold
ToastLabel.TextSize = 12
ToastLabel.Text = "Novyx Hub | v1 cargado"
ToastLabel.Visible = false
ToastLabel.Parent = ScreenGui
addCorner(ToastLabel, 10)
addStroke(ToastLabel, Theme.AccentBlue, 1.4)

local function notify(msg: string)
	ToastLabel.Text = msg
	ToastLabel.Visible = true
	TweenService:Create(ToastLabel, TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		Position = UDim2.new(0.5, -160, 0, 18),
	}):Play()
	task.delay(2.4, function()
		if ToastLabel.Visible then
			local tweenOut = TweenService:Create(ToastLabel, TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
				Position = UDim2.new(0.5, -160, 0, -50),
			})
			tweenOut:Play()
			tweenOut.Completed:Wait()
			ToastLabel.Visible = false
		end
	end)
end

-- Indicador visual FOV de entrenamiento en el centro de la pantalla
local FovCircle = Instance.new("Frame")
FovCircle.Name = "TrainingFovIndicator"
FovCircle.AnchorPoint = Vector2.new(0.5, 0.5)
FovCircle.Position = UDim2.new(0.5, 0, 0.5, 0)
FovCircle.Size = UDim2.fromOffset(State.FovRadius * 2, State.FovRadius * 2)
FovCircle.BackgroundTransparency = 1
FovCircle.Visible = false
FovCircle.Parent = ScreenGui
addCorner(FovCircle, 999)
local FovStroke = addStroke(FovCircle, Theme.AccentBlue, 1.8)

-- Botón "Open HUB" flotante y arrastrable
local OpenHubButton = Instance.new("TextButton")
OpenHubButton.Name = "OpenHubButton"
OpenHubButton.Size = UDim2.new(0, 126, 0, 40)
OpenHubButton.Position = UDim2.new(0, 16, 0.5, -20)
OpenHubButton.BackgroundColor3 = Theme.BgCard
OpenHubButton.TextColor3 = Theme.TextWhite
OpenHubButton.Font = Enum.Font.GothamBold
OpenHubButton.TextSize = 13
OpenHubButton.Text = "⚡ Open HUB"
OpenHubButton.Parent = ScreenGui
addCorner(OpenHubButton, 12)
local OpenStroke = addStroke(OpenHubButton, Theme.AccentPurple, 1.8)

local OpenGrad = Instance.new("UIGradient")
OpenGrad.Color = ColorSequence.new({
	ColorSequenceKeypoint.new(0, Color3.fromRGB(95, 38, 195)),
	ColorSequenceKeypoint.new(1, Color3.fromRGB(25, 110, 210)),
})
OpenGrad.Rotation = 35
OpenGrad.Parent = OpenHubButton

-- Ventana principal (adaptable a cualquier resolución y compacta para Android)
local MainWindow = Instance.new("Frame")
MainWindow.Name = "NovyxMainWindow"
MainWindow.Size = UDim2.new(0.84, 0, 0.78, 0)
MainWindow.Position = UDim2.new(0.08, 0, 0.11, 0)
MainWindow.BackgroundColor3 = Theme.BgMain
MainWindow.BorderSizePixel = 0
MainWindow.Visible = false
MainWindow.ClipsDescendants = true
MainWindow.Parent = ScreenGui
addCorner(MainWindow, 14)
local MainStroke = addStroke(MainWindow, Theme.AccentPurple, 1.8)

local SizeConstraint = Instance.new("UISizeConstraint")
SizeConstraint.MinSize = Vector2.new(330, 250)
SizeConstraint.MaxSize = Vector2.new(780, 480)
SizeConstraint.Parent = MainWindow

-- Barra superior (Header)
local TopBar = Instance.new("Frame")
TopBar.Name = "TopBar"
TopBar.Size = UDim2.new(1, 0, 0, 44)
TopBar.BackgroundColor3 = Theme.BgPanel
TopBar.BorderSizePixel = 0
TopBar.Parent = MainWindow
addCorner(TopBar, 14)

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(0.55, -12, 1, 0)
TitleLabel.Position = UDim2.new(0, 14, 0, 0)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "Novyx Hub | v1"
TitleLabel.TextColor3 = Theme.TextWhite
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.TextSize = 15
TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
TitleLabel.Parent = TopBar

local SubBrandLabel = Instance.new("TextLabel")
SubBrandLabel.Size = UDim2.new(0.25, 0, 1, 0)
SubBrandLabel.Position = UDim2.new(0.52, 0, 0, 0)
SubBrandLabel.BackgroundTransparency = 1
SubBrandLabel.Text = "NEXVYR HUB"
SubBrandLabel.TextColor3 = Theme.AccentBlue
SubBrandLabel.Font = Enum.Font.GothamBold
SubBrandLabel.TextSize = 11
SubBrandLabel.TextXAlignment = Enum.TextXAlignment.Right
SubBrandLabel.Parent = TopBar

-- Botones Minimizar y Cerrar
local MinButton = Instance.new("TextButton")
MinButton.Size = UDim2.new(0, 30, 0, 28)
MinButton.Position = UDim2.new(1, -70, 0.5, -14)
MinButton.BackgroundColor3 = Theme.BgCard
MinButton.TextColor3 = Theme.AccentBlue
MinButton.Font = Enum.Font.GothamBold
MinButton.TextSize = 14
MinButton.Text = "—"
MinButton.Parent = TopBar
addCorner(MinButton, 7)

local CloseButton = Instance.new("TextButton")
CloseButton.Size = UDim2.new(0, 30, 0, 28)
CloseButton.Position = UDim2.new(1, -35, 0.5, -14)
CloseButton.BackgroundColor3 = Theme.BgCard
CloseButton.TextColor3 = Theme.Danger
CloseButton.Font = Enum.Font.GothamBold
CloseButton.TextSize = 14
CloseButton.Text = "✕"
CloseButton.Parent = TopBar
addCorner(CloseButton, 7)

-- Cuerpo inferior (Barra lateral izquierda + Contenido derecho)
local BodyFrame = Instance.new("Frame")
BodyFrame.Name = "BodyFrame"
BodyFrame.Size = UDim2.new(1, 0, 1, -44)
BodyFrame.Position = UDim2.new(0, 0, 0, 44)
BodyFrame.BackgroundTransparency = 1
BodyFrame.Parent = MainWindow

local SidebarFrame = Instance.new("Frame")
SidebarFrame.Name = "Sidebar"
SidebarFrame.Size = UDim2.new(0.32, 0, 1, 0)
SidebarFrame.BackgroundColor3 = Theme.BgPanel
SidebarFrame.BorderSizePixel = 0
SidebarFrame.Parent = BodyFrame

local SidebarConstraint = Instance.new("UISizeConstraint")
SidebarConstraint.MinSize = Vector2.new(112, 0)
SidebarConstraint.MaxSize = Vector2.new(210, 9999)
SidebarConstraint.Parent = SidebarFrame

-- Buscador de opciones
local SearchBox = Instance.new("TextBox")
SearchBox.Name = "SearchBox"
SearchBox.Size = UDim2.new(1, -12, 0, 32)
SearchBox.Position = UDim2.new(0, 6, 0, 8)
SearchBox.BackgroundColor3 = Theme.BgCard
SearchBox.TextColor3 = Theme.TextWhite
SearchBox.PlaceholderColor3 = Theme.TextMuted
SearchBox.PlaceholderText = "🔍 Buscar..."
SearchBox.Font = Enum.Font.Gotham
SearchBox.TextSize = 12
SearchBox.ClearTextOnFocus = false
SearchBox.Parent = SidebarFrame
addCorner(SearchBox, 8)
addStroke(SearchBox, Theme.Stroke, 1)

-- Lista desplazable de categorías
local CategoryList = Instance.new("ScrollingFrame")
CategoryList.Name = "CategoryList"
CategoryList.Size = UDim2.new(1, -8, 1, -48)
CategoryList.Position = UDim2.new(0, 4, 0, 44)
CategoryList.BackgroundTransparency = 1
CategoryList.ScrollBarThickness = 3
CategoryList.ScrollBarImageColor3 = Theme.AccentPurple
CategoryList.CanvasSize = UDim2.new(0, 0, 0, 0)
CategoryList.AutomaticCanvasSize = Enum.AutomaticSize.Y
CategoryList.Parent = SidebarFrame

local CategoryLayout = Instance.new("UIListLayout")
CategoryLayout.Padding = UDim.new(0, 4)
CategoryLayout.SortOrder = Enum.SortOrder.LayoutOrder
CategoryLayout.Parent = CategoryList

-- Contenedor de páginas de cada categoría
local ContentArea = Instance.new("Frame")
ContentArea.Name = "ContentArea"
ContentArea.Size = UDim2.new(0.68, 0, 1, 0)
ContentArea.Position = UDim2.new(0.32, 0, 0, 0)
ContentArea.BackgroundTransparency = 1
ContentArea.Parent = BodyFrame

local ContentHeader = Instance.new("TextLabel")
ContentHeader.Size = UDim2.new(1, -16, 0, 32)
ContentHeader.Position = UDim2.new(0, 12, 0, 6)
ContentHeader.BackgroundTransparency = 1
ContentHeader.Text = "INICIO"
ContentHeader.TextColor3 = Theme.TextWhite
ContentHeader.Font = Enum.Font.GothamBold
ContentHeader.TextSize = 14
ContentHeader.TextXAlignment = Enum.TextXAlignment.Left
ContentHeader.Parent = ContentArea

local PagesHolder = Instance.new("Frame")
PagesHolder.Size = UDim2.new(1, -16, 1, -42)
PagesHolder.Position = UDim2.new(0, 8, 0, 38)
PagesHolder.BackgroundTransparency = 1
PagesHolder.Parent = ContentArea

-- Abrir / Minimizar / Cerrar con animaciones suaves
local minimized = false
local isOpen = false

local function setHubOpen(open: boolean)
	playClickSfx()
	isOpen = open
	if open then
		minimized = false
		MainWindow.Visible = true
		MainWindow.Size = UDim2.new(0.72, 0, 0.62, 0)
		TweenService:Create(MainWindow, TweenInfo.new(0.22, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
			Size = UDim2.new(0.84, 0, 0.78, 0),
		}):Play()
	else
		local t = TweenService:Create(MainWindow, TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
			Size = UDim2.new(0.72, 0, 0.62, 0),
		})
		t:Play()
		t.Completed:Wait()
		if not isOpen then
			MainWindow.Visible = false
		end
	end
end

OpenHubButton.MouseButton1Click:Connect(function()
	setHubOpen(not isOpen)
end)

CloseButton.MouseButton1Click:Connect(function()
	setHubOpen(false)
end)

MinButton.MouseButton1Click:Connect(function()
	playClickSfx()
	minimized = not minimized
	BodyFrame.Visible = not minimized
	TweenService:Create(MainWindow, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		Size = minimized and UDim2.new(0.55, 0, 0, 44) or UDim2.new(0.84, 0, 0.78, 0),
	}):Play()
end)

makeDraggable(OpenHubButton, OpenHubButton)
makeDraggable(TopBar, MainWindow)

-- Fábrica de elementos UI reutilizables dentro de cada página
local Pages: { [string]: ScrollingFrame } = {}
local SidebarButtons: { [string]: TextButton } = {}

local function createPage(categoryName: string): ScrollingFrame
	local page = Instance.new("ScrollingFrame")
	page.Name = "Page_" .. categoryName
	page.Size = UDim2.new(1, 0, 1, 0)
	page.BackgroundTransparency = 1
	page.ScrollBarThickness = 3
	page.ScrollBarImageColor3 = Theme.AccentBlue
	page.CanvasSize = UDim2.new(0, 0, 0, 0)
	page.AutomaticCanvasSize = Enum.AutomaticSize.Y
	page.Visible = false
	page.Parent = PagesHolder

	local layout = Instance.new("UIListLayout")
	layout.Padding = UDim.new(0, 6)
	layout.SortOrder = Enum.SortOrder.LayoutOrder
	layout.Parent = page

	local pad = Instance.new("UIPadding")
	pad.PaddingBottom = UDim.new(0, 10)
	pad.Parent = page

	Pages[categoryName] = page
	return page
end

local function selectCategory(categoryName: string)
	playClickSfx()
	ContentHeader.Text = categoryName
	for name, page in pairs(Pages) do
		page.Visible = (name == categoryName)
	end
	for name, btn in pairs(SidebarButtons) do
		local active = (name == categoryName)
		btn.BackgroundColor3 = active and Theme.AccentPurple or Theme.BgCard
		btn.TextColor3 = active and Theme.TextWhite or Theme.TextMuted
	end
end

local function addInfoCard(parent: GuiObject, titleText: string, bodyText: string): TextLabel
	local card = Instance.new("Frame")
	card.Size = UDim2.new(1, -4, 0, 0)
	card.AutomaticSize = Enum.AutomaticSize.Y
	card.BackgroundColor3 = Theme.BgCard
	card.Parent = parent
	addCorner(card, 9)
	addStroke(card, Theme.Stroke, 1)

	local pad = Instance.new("UIPadding")
	pad.PaddingTop = UDim.new(0, 8)
	pad.PaddingBottom = UDim.new(0, 8)
	pad.PaddingLeft = UDim.new(0, 10)
	pad.PaddingRight = UDim.new(0, 10)
	pad.Parent = card

	local list = Instance.new("UIListLayout")
	list.Padding = UDim.new(0, 3)
	list.Parent = card

	local title = Instance.new("TextLabel")
	title.Size = UDim2.new(1, 0, 0, 18)
	title.BackgroundTransparency = 1
	title.Text = titleText
	title.TextColor3 = Theme.AccentBlue
	title.Font = Enum.Font.GothamBold
	title.TextSize = 12
	title.TextXAlignment = Enum.TextXAlignment.Left
	title.Parent = card

	local body = Instance.new("TextLabel")
	body.Size = UDim2.new(1, 0, 0, 0)
	body.AutomaticSize = Enum.AutomaticSize.Y
	body.BackgroundTransparency = 1
	body.Text = bodyText
	body.TextColor3 = Theme.TextWhite
	body.Font = Enum.Font.Gotham
	body.TextSize = 11
	body.TextWrapped = true
	body.TextXAlignment = Enum.TextXAlignment.Left
	body.Parent = card

	return body
end

local function addActionButton(parent: GuiObject, labelText: string, callback: () -> ())
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(1, -4, 0, 34)
	btn.BackgroundColor3 = Theme.BgCard
	btn.TextColor3 = Theme.TextWhite
	btn.Font = Enum.Font.GothamBold
	btn.TextSize = 12
	btn.Text = labelText
	btn.Parent = parent
	addCorner(btn, 8)
	addStroke(btn, Theme.AccentBlue, 1)

	btn.MouseButton1Click:Connect(function()
		playClickSfx()
		callback()
	end)
	return btn
end

local function addToggle(parent: GuiObject, labelText: string, initial: boolean, callback: (boolean) -> ())
	local active = initial
	local row = Instance.new("Frame")
	row.Size = UDim2.new(1, -4, 0, 36)
	row.BackgroundColor3 = Theme.BgCard
	row.Parent = parent
	addCorner(row, 8)
	addStroke(row, Theme.Stroke, 1)

	local lbl = Instance.new("TextLabel")
	lbl.Size = UDim2.new(0.68, -10, 1, 0)
	lbl.Position = UDim2.new(0, 10, 0, 0)
	lbl.BackgroundTransparency = 1
	lbl.Text = labelText
	lbl.TextColor3 = Theme.TextWhite
	lbl.Font = Enum.Font.Gotham
	lbl.TextSize = 12
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.TextTruncate = Enum.TextTruncate.AtEnd
	lbl.Parent = row

	local pill = Instance.new("TextButton")
	pill.Size = UDim2.new(0, 68, 0, 24)
	pill.Position = UDim2.new(1, -76, 0.5, -12)
	pill.Font = Enum.Font.GothamBold
	pill.TextSize = 11
	pill.TextColor3 = Theme.TextWhite
	pill.Parent = row
	addCorner(pill, 6)

	local function refreshVisual()
		pill.Text = active and "ON" or "OFF"
		pill.BackgroundColor3 = active and Theme.Success or Color3.fromRGB(52, 55, 75)
	end
	refreshVisual()

	pill.MouseButton1Click:Connect(function()
		playClickSfx()
		active = not active
		refreshVisual()
		callback(active)
	end)

	return {
		Set = function(val: boolean)
			active = val
			refreshVisual()
		end,
	}
end

local function addSlider(parent: GuiObject, labelText: string, minVal: number, maxVal: number, defaultVal: number, callback: (number) -> ())
	local current = math.clamp(defaultVal, minVal, maxVal)

	local box = Instance.new("Frame")
	box.Size = UDim2.new(1, -4, 0, 48)
	box.BackgroundColor3 = Theme.BgCard
	box.Parent = parent
	addCorner(box, 8)
	addStroke(box, Theme.Stroke, 1)

	local top = Instance.new("TextLabel")
	top.Size = UDim2.new(1, -16, 0, 18)
	top.Position = UDim2.new(0, 8, 0, 4)
		top.BackgroundTransparency = 1
	top.Text = string.format("%s: %s", labelText, tostring(math.floor(current * 100) / 100))
	top.TextColor3 = Theme.TextWhite
	top.Font = Enum.Font.Gotham
	top.TextSize = 11
	top.TextXAlignment = Enum.TextXAlignment.Left
	top.Parent = box

	local track = Instance.new("Frame")
	track.Size = UDim2.new(1, -16, 0, 8)
	track.Position = UDim2.new(0, 8, 0, 28)
	track.BackgroundColor3 = Color3.fromRGB(38, 42, 62)
	track.Parent = box
	addCorner(track, 4)

	local ratio = (current - minVal) / math.max(0.0001, (maxVal - minVal))
	local fill = Instance.new("Frame")
	fill.Size = UDim2.new(ratio, 0, 1, 0)
	fill.BackgroundColor3 = Theme.AccentBlue
	fill.Parent = track
	addCorner(fill, 4)

	local sliding = false
	local function updateFromX(xPos: number)
		local absX = track.AbsolutePosition.X
		local absW = math.max(1, track.AbsoluteSize.X)
		local r = math.clamp((xPos - absX) / absW, 0, 1)
		current = minVal + (maxVal - minVal) * r
		local rounded = math.floor(current * 100) / 100
		fill.Size = UDim2.new(r, 0, 1, 0)
		top.Text = string.format("%s: %s", labelText, tostring(rounded))
		callback(rounded)
	end

	track.InputBegan:Connect(function(input: InputObject)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			sliding = true
			updateFromX(input.Position.X)
		end
	end)

	UserInputService.InputChanged:Connect(function(input: InputObject)
		if sliding and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			updateFromX(input.Position.X)
		end
	end)

	UserInputService.InputEnded:Connect(function(input: InputObject)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			sliding = false
		end
	end)
end

local function addDropdown(parent: GuiObject, labelText: string, options: { string }, defaultOption: string, callback: (string) -> ())
	local index = 1
	for i, opt in ipairs(options) do
		if opt == defaultOption then
			index = i
			break
		end
	end

	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(1, -4, 0, 34)
	btn.BackgroundColor3 = Theme.BgCard
	btn.TextColor3 = Theme.TextWhite
	btn.Font = Enum.Font.GothamBold
	btn.TextSize = 11
	btn.Text = string.format("▾ %s: %s", labelText, options[index])
	btn.Parent = parent
	addCorner(btn, 8)
	addStroke(btn, Theme.AccentPurple, 1)

	btn.MouseButton1Click:Connect(function()
		playClickSfx()
		index = (index % #options) + 1
		local chosen = options[index]
		btn.Text = string.format("▾ %s: %s", labelText, chosen)
		callback(chosen)
	end)
end
