-- ============================================================================
-- NOVYX HUB | v1 (NEXVYR HUB) — ALL-IN-ONE LUAU SCRIPT (DELTA MOBILE & STUDIO)
-- ============================================================================
local Players = game:GetService("Players")
local Lighting = game:GetService("Lighting")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local SoundService = game:GetService("SoundService")
local Workspace = game:GetService("Workspace")
local CoreGui = game:GetService("CoreGui")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

-- Contenedor seguro para GUI (soporta Delta/CoreGui con fallback a PlayerGui)
local function getSafeGuiParent(): Instance
	local ok, result = pcall(function()
		return (gethui and gethui()) or CoreGui
	end)
	if ok and result then
		return result
	end
	return player:WaitForChild("PlayerGui")
end

-- Evitar duplicados si se vuelve a ejecutar el script
local guiParent = getSafeGuiParent()
local existing = guiParent:FindFirstChild("NovyxHubGui")
if existing then
	existing:Destroy()
end

-- ============================================================================
-- PALETA Y METADATOS
-- ============================================================================
local THEME = {
	BgMain = Color3.fromRGB(10, 10, 15),
	BgPanel = Color3.fromRGB(18, 17, 28),
	BgCard = Color3.fromRGB(25, 23, 39),
	BgHover = Color3.fromRGB(36, 32, 58),
	Purple = Color3.fromRGB(138, 63, 252),
	Blue = Color3.fromRGB(45, 156, 255),
	PurpleDark = Color3.fromRGB(76, 34, 150),
	Text = Color3.fromRGB(255, 255, 255),
	TextMuted = Color3.fromRGB(175, 178, 205),
	Success = Color3.fromRGB(56, 224, 155),
	Danger = Color3.fromRGB(255, 78, 108),
	Stroke = Color3.fromRGB(68, 52, 118),
}

local HUB_META = {
	HeaderTitle = "NEXVYR HUB",
	HubName = "Novyx Hub | v1",
	Version = "v1.0.0-allinone",
	ScriptInfo = "Hub unificado para entrenamiento, pruebas visuales y control móvil/PC.",
	UpdatesInfo = "v1.0.0: Archivo único listo para Delta Mobile y Roblox Studio.",
	Credits = "Desarrollado en Luau • Novyx Hub | v1.",
	Community = "Comunidad Novyx — Canal oficial de pruebas.",
}

-- ============================================================================
-- ESTADO GLOBAL DEL HUB
-- ============================================================================
local State = {
	Minimized = false,
	CustomDisplayName = player.DisplayName,
	AccentMode = "Morado + Azul",
	ProfileTag = "Entrenamiento v1",

	-- Gráficos
	LightingPreset = "Original",
	OriginalLighting = {
		Ambient = Lighting.Ambient,
		OutdoorAmbient = Lighting.OutdoorAmbient,
		ColorShift_Top = Lighting.ColorShift_Top,
		FogColor = Lighting.FogColor,
		FogEnd = Lighting.FogEnd,
		Brightness = Lighting.Brightness,
	},

	-- Botones flotantes y Emotes
	ShowFloatingButtons = true,
	FloatingSelection = "Ambos (Open + Emote)",
	AutoSelectButtons = false,
	EmoteSelector = "Wave",
	EquippedEmotes = {"Wave", "Dance", "Cheer", "Point"},
	EmoteSpamActive = false,

	-- Aimbot (exclusivo campo de práctica / dummies)
	PracticeRange = true,
	DummyIndicators = true,
	SelectedDummyMode = "Más cercano",
	Sensitivity = 0.50,
	ShowTrainingFov = false,
	TrainingFovRadius = 130,

	-- Visual (ESP de entorno de prueba)
	VisualEspEnabled = false,
	ObjectIndicators = true,
	VisualInfoEnabled = true,
	VisualColor = Color3.fromRGB(45, 156, 255),
	VisualMaxDistance = 250,
	MarkerStyle = "Resaltado + Etiqueta",

	-- Kill All (NPC ficticios)
	AutoKillDummies = false,

	-- Invisible (personaje local en pruebas)
	InvisibleEnabled = false,
	SavedTransparency = {} :: {[BasePart]: number},

	-- Trampaso (movimiento de prueba)
	SpeedEnabled = false,
	WalkSpeedValue = 16,
	JumpEnabled = false,
	JumpPowerValue = 50,
	FlyEnabled = false,
	FlySpeedValue = 45,

	-- Farmear Racha (estadísticas de sesión)
	Stats = {
		Kills = 0,
		Streak = 0,
		BestStreak = 0,
		ActiveDummies = 0,
		SessionStart = os.clock(),
	},

	-- Efectos
	ParticlesEnabled = false,
	AuraEnabled = false,
	EffectColor = Color3.fromRGB(138, 63, 252),

	-- Animaciones
	SelectedAnim = "Idle Clásico",
	ActiveAnimTrack = nil :: AnimationTrack?,

	-- Sonidos y Música
	SoundEffectsEnabled = true,
	SfxVolume = 0.7,
	MusicTrackIndex = 1,
	MusicPlaying = false,
	MusicVolume = 0.5,

	-- Items
	SelectedItem = "Espada de Práctica",
}

local EMOTE_IDS: {[string]: string} = {
	Wave = "rbxassetid://507770239",
	Dance = "rbxassetid://507771019",
	Cheer = "rbxassetid://507770677",
	Point = "rbxassetid://507770453",
}

local ANIM_IDS: {[string]: string} = {
	["Idle Clásico"] = "rbxassetid://507766666",
	["Salto de Prueba"] = "rbxassetid://507765000",
	["Carrera Ágil"] = "rbxassetid://507767714",
}

local MUSIC_PLAYLIST = {
	{Name = "Novyx Synthwave 01", SoundId = "rbxassetid://1837849285"},
	{Name = "Night Drive Pulse", SoundId = "rbxassetid://1843523270"},
	{Name = "Cyber Arena Theme", SoundId = "rbxassetid://1846458016"},
}

local ITEMS_CATALOG = {
	["Espada de Práctica"] = {Tipo = "Melee Prueba", Daño = "Entrenamiento", Vista = "Hoja azul/morada de baja fricción"},
	["Bastón de Impulso"] = {Tipo = "Utilidad Prueba", Daño = "0 (Movilidad)", Vista = "Núcleo cinético con brillo violeta"},
	["Dron Marcador"] = {Tipo = "Radar Prueba", Daño = "0 (Visual)", Vista = "Emisor óptico para campo de tiro"},
}

-- ============================================================================
-- GESTIÓN SEGURA DE ENTORNO DE PRUEBA (DUMMIES / REMOTO OPCIONAL)
-- ============================================================================
local function getOrCreateTrainingFolder(): Folder
	local folder = Workspace:FindFirstChild("TrainingDummies")
	if folder and folder:IsA("Folder") then
		return folder
	end
	local created = Instance.new("Folder")
	created.Name = "TrainingDummies"
	created.Parent = Workspace
	return created
end

local function spawnSamplePracticeDummy()
	local folder = getOrCreateTrainingFolder()
	local char = player.Character
	local root = char and char:FindFirstChild("HumanoidRootPart") :: BasePart?
	local basePos = root and (root.Position + root.CFrame.LookVector * 14 + Vector3.new(math.random(-8, 8), 0, math.random(-8, 8))) or Vector3.new(0, 5, 0)

	local dummy = Instance.new("Model")
	dummy.Name = "Dummy_" .. tostring(#folder:GetChildren() + 1)

	local hrp = Instance.new("Part")
	hrp.Name = "HumanoidRootPart"
	hrp.Size = Vector3.new(2, 5, 1)
	hrp.Position = basePos
	hrp.Anchored = true
	hrp.Material = Enum.Material.Neon
	hrp.Color = THEME.Purple
	hrp.Parent = dummy

	local hum = Instance.new("Humanoid")
	hum.MaxHealth = 100
	hum.Health = 100
	hum.Parent = dummy

	dummy.PrimaryPart = hrp
	dummy.Parent = folder
end

local function countActiveDummies(): number
	local folder = Workspace:FindFirstChild("TrainingDummies")
	if not folder then return 0 end
	local count = 0
	for _, m in ipairs(folder:GetChildren()) do
		if m:IsA("Model") then
			local hum = m:FindFirstChildOfClass("Humanoid")
			if hum and hum.Health > 0 then
				count += 1
			end
		end
	end
	return count
end

local function eliminatePracticeDummies(targetModel: Model?)
	local remotes = ReplicatedStorage:FindFirstChild("NovyxRemotes")
	local actionRemote = remotes and remotes:FindFirstChild("PracticeAction")
	if actionRemote and actionRemote:IsA("RemoteEvent") then
		if targetModel then
			actionRemote:FireServer("KillTargetDummy", targetModel)
		else
			actionRemote:FireServer("KillAllDummies")
		end
		return
	end

	-- Ejecución local sobre NPCs ficticios en TrainingDummies (jamás jugadores reales)
	local folder = Workspace:FindFirstChild("TrainingDummies")
	if not folder then return end

	local eliminatedNow = 0
	local function handleModel(m: Instance)
		if not m:IsA("Model") then return end
		if Players:GetPlayerFromCharacter(m) then return end
		local hum = m:FindFirstChildOfClass("Humanoid")
		if hum and hum.Health > 0 then
			hum.Health = 0
			eliminatedNow += 1
			task.delay(0.4, function()
				if m and m.Parent then
					m:Destroy()
				end
			end)
		end
	end

	if targetModel and targetModel:IsDescendantOf(folder) then
		handleModel(targetModel)
	else
		for _, child in ipairs(folder:GetChildren()) do
			handleModel(child)
		end
	end

	if eliminatedNow > 0 then
		State.Stats.Kills += eliminatedNow
		State.Stats.Streak += eliminatedNow
		if State.Stats.Streak > State.Stats.BestStreak then
			State.Stats.BestStreak = State.Stats.Streak
		end
	end
	State.Stats.ActiveDummies = countActiveDummies()
end

-- ============================================================================
-- UTILIDADES DE INTERFAZ Y AUDIO
-- ============================================================================
local function round(obj: GuiObject, radius: number)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, radius)
	c.Parent = obj
	return c
end

local function stroke(obj: GuiObject, color: Color3?, thickness: number?)
	local s = Instance.new("UIStroke")
	s.Color = color or THEME.Stroke
	s.Thickness = thickness or 1
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Parent = obj
	return s
end

local function playUiClick()
	if not State.SoundEffectsEnabled then return end
	local s = Instance.new("Sound")
	s.SoundId = "rbxassetid://6895079853"
	s.Volume = State.SfxVolume
	s.Parent = SoundService
	s.Ended:Connect(function()
		s:Destroy()
	end)
	s:Play()
end

local function makeDraggable(frame: GuiObject, handle: GuiObject?)
	local dragHandle = handle or frame
	local dragging = false
	local dragInput: InputObject? = nil
	local dragStart = Vector2.zero
	local startPos = frame.Position

	dragHandle.InputBegan:Connect(function(input: InputObject)
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

	dragHandle.InputChanged:Connect(function(input: InputObject)
		if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
			dragInput = input
		end
	end)

	UserInputService.InputChanged:Connect(function(input: InputObject)
		if dragging and input == dragInput then
			local delta = input.Position - dragStart
			frame.Position = UDim2.new(
				startPos.X.Scale,
				startPos.X.Offset + delta.X,
				startPos.Y.Scale,
				startPos.Y.Offset + delta.Y
			)
		end
	end)
end

-- ============================================================================
-- CONSTRUCCIÓN DE SCREENGUI PRINCIPAL
-- ============================================================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "NovyxHubGui"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.IgnoreGuiInset = true
screenGui.Parent = guiParent

-- Botón flotante Open HUB
local openBtn = Instance.new("TextButton")
openBtn.Name = "OpenHubButton"
openBtn.Size = UDim2.fromOffset(124, 40)
openBtn.Position = UDim2.new(0, 16, 0.5, -20)
openBtn.BackgroundColor3 = THEME.BgPanel
openBtn.Text = "OPEN HUB"
openBtn.TextColor3 = THEME.Text
openBtn.Font = Enum.Font.GothamBold
openBtn.TextSize = 13
openBtn.Active = true
openBtn.Parent = screenGui
round(openBtn, 12)

local openBtnGradient = Instance.new("UIGradient")
openBtnGradient.Color = ColorSequence.new({
	ColorSequenceKeypoint.new(0, THEME.PurpleDark),
	ColorSequenceKeypoint.new(1, Color3.fromRGB(22, 58, 118)),
})
openBtnGradient.Parent = openBtn
stroke(openBtn, THEME.Purple, 1.5)

-- Botón flotante rápido de Emote
local quickEmoteBtn = Instance.new("TextButton")
quickEmoteBtn.Name = "QuickEmoteButton"
quickEmoteBtn.Size = UDim2.fromOffset(116, 34)
quickEmoteBtn.Position = UDim2.new(0, 16, 0.5, 28)
quickEmoteBtn.BackgroundColor3 = THEME.BgCard
quickEmoteBtn.Text = "EMOTE PRUEBA"
quickEmoteBtn.TextColor3 = THEME.Blue
quickEmoteBtn.Font = Enum.Font.GothamBold
quickEmoteBtn.TextSize = 11
quickEmoteBtn.Active = true
quickEmoteBtn.Parent = screenGui
round(quickEmoteBtn, 10)
stroke(quickEmoteBtn, THEME.Blue, 1.2)

-- Círculo visual de FOV de entrenamiento
local fovCircle = Instance.new("Frame")
fovCircle.Name = "TrainingFovCircle"
fovCircle.AnchorPoint = Vector2.new(0.5, 0.5)
fovCircle.Position = UDim2.fromScale(0.5, 0.5)
fovCircle.Size = UDim2.fromOffset(State.TrainingFovRadius * 2, State.TrainingFovRadius * 2)
fovCircle.BackgroundTransparency = 1
fovCircle.Visible = false
fovCircle.Parent = screenGui
round(fovCircle, 999)
stroke(fovCircle, THEME.Blue, 1.5)

-- Ventana principal NEXVYR HUB
local mainWindow = Instance.new("Frame")
mainWindow.Name = "MainWindow"
mainWindow.AnchorPoint = Vector2.new(0.5, 0.5)
mainWindow.Position = UDim2.fromScale(0.5, 0.5)
mainWindow.Size = UDim2.fromOffset(650, 400)
mainWindow.BackgroundColor3 = THEME.BgMain
mainWindow.ClipsDescendants = true
mainWindow.Visible = true
mainWindow.Active = true
mainWindow.Parent = screenGui
round(mainWindow, 16)
stroke(mainWindow, THEME.Purple, 1.5)

local uiScale = Instance.new("UIScale")
uiScale.Parent = mainWindow

local function updateResponsiveScale()
	local vp = camera and camera.ViewportSize or Vector2.new(1280, 720)
	local scaleX = (vp.X * 0.92) / 650
	local scaleY = (vp.Y * 0.88) / 400
	uiScale.Scale = math.clamp(math.min(scaleX, scaleY), 0.55, 1.15)
end
updateResponsiveScale()
if camera then
	camera:GetPropertyChangedSignal("ViewportSize"):Connect(updateResponsiveScale)
end

-- Barra superior (Header)
local topBar = Instance.new("Frame")
topBar.Name = "TopBar"
topBar.Size = UDim2.new(1, 0, 0, 44)
topBar.BackgroundColor3 = THEME.BgPanel
topBar.BorderSizePixel = 0
topBar.Parent = mainWindow

local topGrad = Instance.new("UIGradient")
topGrad.Color = ColorSequence.new({
	ColorSequenceKeypoint.new(0, Color3.fromRGB(36, 18, 68)),
	ColorSequenceKeypoint.new(1, Color3.fromRGB(14, 34, 68)),
})
topGrad.Parent = topBar

local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(0.65, -16, 1, 0)
titleLabel.Position = UDim2.fromOffset(14, 0)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = HUB_META.HeaderTitle .. "  •  " .. HUB_META.HubName
titleLabel.TextColor3 = THEME.Text
titleLabel.TextXAlignment = Enum.TextXAlignment.Left
titleLabel.Font = Enum.Font.GothamBold
titleLabel.TextSize = 14
titleLabel.Parent = topBar

local minBtn = Instance.new("TextButton")
minBtn.Name = "MinimizeBtn"
minBtn.Size = UDim2.fromOffset(32, 28)
minBtn.Position = UDim2.new(1, -74, 0.5, -14)
minBtn.BackgroundColor3 = THEME.BgCard
minBtn.Text = "—"
minBtn.TextColor3 = THEME.Blue
minBtn.Font = Enum.Font.GothamBold
minBtn.TextSize = 14
minBtn.Parent = topBar
round(minBtn, 8)
stroke(minBtn, THEME.Blue, 1)

local closeBtn = Instance.new("TextButton")
closeBtn.Name = "CloseBtn"
closeBtn.Size = UDim2.fromOffset(32, 28)
closeBtn.Position = UDim2.new(1, -38, 0.5, -14)
closeBtn.BackgroundColor3 = THEME.BgCard
closeBtn.Text = "✕"
closeBtn.TextColor3 = THEME.Danger
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 13
closeBtn.Parent = topBar
round(closeBtn, 8)
stroke(closeBtn, THEME.Danger, 1)

-- Cuerpo inferior (Sidebar + Content)
local bodyFrame = Instance.new("Frame")
bodyFrame.Name = "BodyFrame"
bodyFrame.Size = UDim2.new(1, 0, 1, -44)
bodyFrame.Position = UDim2.fromOffset(0, 44)
bodyFrame.BackgroundTransparency = 1
bodyFrame.Parent = mainWindow

local sidebar = Instance.new("Frame")
sidebar.Name = "Sidebar"
sidebar.Size = UDim2.new(0, 174, 1, -12)
sidebar.Position = UDim2.fromOffset(8, 6)
sidebar.BackgroundColor3 = THEME.BgPanel
sidebar.Parent = bodyFrame
round(sidebar, 12)
stroke(sidebar, THEME.Stroke, 1)

local searchBox = Instance.new("TextBox")
searchBox.Name = "SearchBox"
searchBox.Size = UDim2.new(1, -14, 0, 30)
searchBox.Position = UDim2.fromOffset(7, 7)
searchBox.BackgroundColor3 = THEME.BgMain
searchBox.PlaceholderText = "Buscar opción..."
searchBox.PlaceholderColor3 = THEME.TextMuted
searchBox.Text = ""
searchBox.TextColor3 = THEME.Text
searchBox.Font = Enum.Font.Gotham
searchBox.TextSize = 12
searchBox.ClearTextOnFocus = false
searchBox.Parent = sidebar
round(searchBox, 8)
stroke(searchBox, THEME.Blue, 1)

local navScroll = Instance.new("ScrollingFrame")
navScroll.Name = "NavScroll"
navScroll.Size = UDim2.new(1, -10, 1, -46)
navScroll.Position = UDim2.fromOffset(5, 42)
navScroll.BackgroundTransparency = 1
navScroll.ScrollBarThickness = 3
navScroll.ScrollBarImageColor3 = THEME.Purple
navScroll.CanvasSize = UDim2.fromOffset(0, 0)
navScroll.Parent = sidebar

local navLayout = Instance.new("UIListLayout")
navLayout.Padding = UDim.new(0, 4)
navLayout.SortOrder = Enum.SortOrder.LayoutOrder
navLayout.Parent = navScroll

navLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
	navScroll.CanvasSize = UDim2.fromOffset(0, navLayout.AbsoluteContentSize.Y + 8)
end)

local contentArea = Instance.new("ScrollingFrame")
contentArea.Name = "ContentArea"
contentArea.Size = UDim2.new(1, -196, 1, -12)
contentArea.Position = UDim2.fromOffset(188, 6)
contentArea.BackgroundColor3 = THEME.BgPanel
contentArea.ScrollBarThickness = 4
contentArea.ScrollBarImageColor3 = THEME.Blue
contentArea.CanvasSize = UDim2.fromOffset(0, 0)
contentArea.Parent = bodyFrame
round(contentArea, 12)
stroke(contentArea, THEME.Stroke, 1)

local contentLayout = Instance.new("UIListLayout")
contentLayout.Padding = UDim.new(0, 8)
contentLayout.SortOrder = Enum.SortOrder.LayoutOrder
contentLayout.Parent = contentArea

local contentPad = Instance.new("UIPadding")
contentPad.PaddingTop = UDim.new(0, 10)
contentPad.PaddingBottom = UDim.new(0, 12)
contentPad.PaddingLeft = UDim.new(0, 10)
contentPad.PaddingRight = UDim.new(0, 10)
contentPad.Parent = contentArea

contentLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
	contentArea.CanvasSize = UDim2.fromOffset(0, contentLayout.AbsoluteContentSize.Y + 24)
end)

-- ============================================================================
-- COMPONENTES REUTILIZABLES DE INTERFAZ
-- ============================================================================
local categories: {[string]: {Button: TextButton, Builder: (Frame) -> ()}} = {}
local currentCategoryName = "INICIO"

local function clearContent()
	for _, child in ipairs(contentArea:GetChildren()) do
		if child:IsA("GuiObject") then
			child:Destroy()
		end
	end
end

local function addSectionHeader(parent: Instance, text: string)
	local lbl = Instance.new("TextLabel")
	lbl.Size = UDim2.new(1, 0, 0, 26)
	lbl.BackgroundTransparency = 1
	lbl.Text = text
	lbl.TextColor3 = THEME.Blue
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Font = Enum.Font.GothamBold
	lbl.TextSize = 14
	lbl.Parent = parent
end

local function addInfoCard(parent: Instance, title: string, valueText: string): TextLabel
	local card = Instance.new("Frame")
	card.Name = title
	card.Size = UDim2.new(1, 0, 0, 50)
	card.BackgroundColor3 = THEME.BgCard
	card.Parent = parent
	round(card, 10)
	stroke(card, THEME.Stroke, 1)

	local t = Instance.new("TextLabel")
	t.Size = UDim2.new(1, -16, 0, 18)
	t.Position = UDim2.fromOffset(8, 4)
	t.BackgroundTransparency = 1
	t.Text = title
	t.TextColor3 = THEME.Purple
	t.TextXAlignment = Enum.TextXAlignment.Left
	t.Font = Enum.Font.GothamBold
	t.TextSize = 12
	t.Parent = card

	local v = Instance.new("TextLabel")
	v.Size = UDim2.new(1, -16, 0, 22)
	v.Position = UDim2.fromOffset(8, 23)
	v.BackgroundTransparency = 1
	v.Text = valueText
	v.TextColor3 = THEME.Text
	v.TextXAlignment = Enum.TextXAlignment.Left
	v.TextTruncate = Enum.TextTruncate.AtEnd
	v.Font = Enum.Font.Gotham
	v.TextSize = 11
	v.Parent = card
	return v
end

local function addActionButton(parent: Instance, label: string, buttonText: string, onClick: () -> ())
	local card = Instance.new("Frame")
	card.Name = label
	card.Size = UDim2.new(1, 0, 0, 40)
	card.BackgroundColor3 = THEME.BgCard
	card.Parent = parent
	round(card, 10)
	stroke(card, THEME.Stroke, 1)

	local lbl = Instance.new("TextLabel")
	lbl.Size = UDim2.new(0.62, -10, 1, 0)
	lbl.Position = UDim2.fromOffset(10, 0)
	lbl.BackgroundTransparency = 1
	lbl.Text = label
	lbl.TextColor3 = THEME.Text
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Font = Enum.Font.GothamMedium
	lbl.TextSize = 12
	lbl.Parent = card

	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(0.34, -6, 0, 26)
	btn.Position = UDim2.new(0.65, 0, 0.5, -13)
	btn.BackgroundColor3 = THEME.Purple
	btn.Text = buttonText
	btn.TextColor3 = THEME.Text
	btn.Font = Enum.Font.GothamBold
	btn.TextSize = 11
	btn.Parent = card
	round(btn, 8)

	btn.MouseButton1Click:Connect(function()
		playUiClick()
		onClick()
	end)
end

local function addToggle(parent: Instance, label: string, initialValue: boolean, onChanged: (boolean) -> ())
	local value = initialValue
	local card = Instance.new("Frame")
	card.Name = label
	card.Size = UDim2.new(1, 0, 0, 40)
	card.BackgroundColor3 = THEME.BgCard
	card.Parent = parent
	round(card, 10)
	stroke(card, THEME.Stroke, 1)

	local lbl = Instance.new("TextLabel")
	lbl.Size = UDim2.new(0.72, -10, 1, 0)
	lbl.Position = UDim2.fromOffset(10, 0)
	lbl.BackgroundTransparency = 1
	lbl.Text = label
	lbl.TextColor3 = THEME.Text
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Font = Enum.Font.GothamMedium
	lbl.TextSize = 12
	lbl.Parent = card

	local pill = Instance.new("TextButton")
	pill.Size = UDim2.fromOffset(70, 26)
	pill.Position = UDim2.new(1, -80, 0.5, -13)
	pill.Font = Enum.Font.GothamBold
	pill.TextSize = 11
	pill.TextColor3 = THEME.Text
	pill.Parent = card
	round(pill, 13)

	local function refreshVisual()
		pill.Text = value and "ON" or "OFF"
		local targetColor = value and THEME.Blue or Color3.fromRGB(48, 46, 68)
		TweenService:Create(pill, TweenInfo.new(0.15), {BackgroundColor3 = targetColor}):Play()
	end
	refreshVisual()

	pill.MouseButton1Click:Connect(function()
		playUiClick()
		value = not value
		refreshVisual()
		onChanged(value)
	end)
end

local function addSlider(parent: Instance, label: string, minVal: number, maxVal: number, defaultVal: number, decimals: number, onChanged: (number) -> ())
	local card = Instance.new("Frame")
	card.Name = label
	card.Size = UDim2.new(1, 0, 0, 54)
	card.BackgroundColor3 = THEME.BgCard
	card.Parent = parent
	round(card, 10)
	stroke(card, THEME.Stroke, 1)

	local function fmt(n: number): string
		local mult = 10 ^ decimals
		return tostring(math.round(n * mult) / mult)
	end

	local topLbl = Instance.new("TextLabel")
	topLbl.Size = UDim2.new(1, -20, 0, 18)
	topLbl.Position = UDim2.fromOffset(10, 4)
	topLbl.BackgroundTransparency = 1
	topLbl.Text = label .. ": " .. fmt(defaultVal)
	topLbl.TextColor3 = THEME.Text
	topLbl.TextXAlignment = Enum.TextXAlignment.Left
	topLbl.Font = Enum.Font.GothamMedium
	topLbl.TextSize = 12
	topLbl.Parent = card

	local track = Instance.new("Frame")
	track.Size = UDim2.new(1, -20, 0, 8)
	track.Position = UDim2.fromOffset(10, 32)
	track.BackgroundColor3 = THEME.BgMain
	track.Parent = card
	round(track, 4)

	local ratioInit = math.clamp((defaultVal - minVal) / math.max(maxVal - minVal, 0.0001), 0, 1)
	local fill = Instance.new("Frame")
	fill.Size = UDim2.new(ratioInit, 0, 1, 0)
	fill.BackgroundColor3 = THEME.Purple
	fill.Parent = track
	round(fill, 4)

	local sliding = false
	local function updateFromInput(inputPosX: number)
		local rel = math.clamp((inputPosX - track.AbsolutePosition.X) / math.max(track.AbsoluteSize.X, 1), 0, 1)
		fill.Size = UDim2.new(rel, 0, 1, 0)
		local rawVal = minVal + (maxVal - minVal) * rel
		local mult = 10 ^ decimals
		local finalVal = math.round(rawVal * mult) / mult
		topLbl.Text = label .. ": " .. fmt(finalVal)
		onChanged(finalVal)
	end

	track.InputBegan:Connect(function(input: InputObject)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			sliding = true
			updateFromInput(input.Position.X)
		end
	end)

	UserInputService.InputChanged:Connect(function(input: InputObject)
		if sliding and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			updateFromInput(input.Position.X)
		end
	end)

	UserInputService.InputEnded:Connect(function(input: InputObject)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			sliding = false
		end
	end)
end

local function addDropdown(parent: Instance, label: string, options: {string}, defaultOption: string, onSelected: (string) -> ())
	local current = defaultOption
	local expanded = false

	local wrapper = Instance.new("Frame")
	wrapper.Name = label
	wrapper.Size = UDim2.new(1, 0, 0, 40)
	wrapper.BackgroundColor3 = THEME.BgCard
	wrapper.ClipsDescendants = true
	wrapper.Parent = parent
	round(wrapper, 10)
	stroke(wrapper, THEME.Stroke, 1)

	local headerBtn = Instance.new("TextButton")
	headerBtn.Size = UDim2.new(1, 0, 0, 40)
	headerBtn.BackgroundTransparency = 1
	headerBtn.Text = ""
	headerBtn.Parent = wrapper

	local lbl = Instance.new("TextLabel")
	lbl.Size = UDim2.new(0.52, -10, 1, 0)
	lbl.Position = UDim2.fromOffset(10, 0)
	lbl.BackgroundTransparency = 1
	lbl.Text = label
	lbl.TextColor3 = THEME.Text
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Font = Enum.Font.GothamMedium
	lbl.TextSize = 12
	lbl.Parent = headerBtn

	local valLbl = Instance.new("TextLabel")
	valLbl.Size = UDim2.new(0.45, -10, 1, 0)
	valLbl.Position = UDim2.new(0.54, 0, 0, 0)
	valLbl.BackgroundTransparency = 1
	valLbl.Text = current .. " ▾"
	valLbl.TextColor3 = THEME.Blue
	valLbl.TextXAlignment = Enum.TextXAlignment.Right
	valLbl.Font = Enum.Font.GothamBold
	valLbl.TextSize = 11
	valLbl.Parent = headerBtn

	local listFrame = Instance.new("Frame")
	listFrame.Size = UDim2.new(1, -16, 0, #options * 28)
	listFrame.Position = UDim2.fromOffset(8, 42)
	listFrame.BackgroundTransparency = 1
	listFrame.Parent = wrapper

	local listLayout = Instance.new("UIListLayout")
	listLayout.Padding = UDim.new(0, 4)
	listLayout.Parent = listFrame

	for _, opt in ipairs(options) do
		local optBtn = Instance.new("TextButton")
		optBtn.Size = UDim2.new(1, 0, 0, 24)
		optBtn.BackgroundColor3 = THEME.BgMain
		optBtn.Text = opt
		optBtn.TextColor3 = THEME.Text
		optBtn.Font = Enum.Font.Gotham
		optBtn.TextSize = 11
		optBtn.Parent = listFrame
		round(optBtn, 6)

		optBtn.MouseButton1Click:Connect(function()
			playUiClick()
			current = opt
			valLbl.Text = current .. " ▾"
			expanded = false
			TweenService:Create(wrapper, TweenInfo.new(0.15), {Size = UDim2.new(1, 0, 0, 40)}):Play()
			onSelected(opt)
		end)
	end

	headerBtn.MouseButton1Click:Connect(function()
		playUiClick()
		expanded = not expanded
		local targetH = expanded and (46 + #options * 28) or 40
		TweenService:Create(wrapper, TweenInfo.new(0.16), {Size = UDim2.new(1, 0, 0, targetH)}):Play()
	end)
end

local function addTextInput(parent: Instance, label: string, placeholder: string, defaultText: string, onSubmit: (string) -> ())
	local card = Instance.new("Frame")
	card.Name = label
	card.Size = UDim2.new(1, 0, 0, 54)
	card.BackgroundColor3 = THEME.BgCard
	card.Parent = parent
	round(card, 10)
	stroke(card, THEME.Stroke, 1)

	local lbl = Instance.new("TextLabel")
	lbl.Size = UDim2.new(1, -20, 0, 18)
	lbl.Position = UDim2.fromOffset(10, 4)
	lbl.BackgroundTransparency = 1
	lbl.Text = label
	lbl.TextColor3 = THEME.Text
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Font = Enum.Font.GothamMedium
	lbl.TextSize = 12
	lbl.Parent = card

	local box = Instance.new("TextBox")
	box.Size = UDim2.new(1, -20, 0, 24)
	box.Position = UDim2.fromOffset(10, 24)
	box.BackgroundColor3 = THEME.BgMain
	box.PlaceholderText = placeholder
	box.Text = defaultText
	box.TextColor3 = THEME.Blue
	box.Font = Enum.Font.GothamBold
	box.TextSize = 12
	box.ClearTextOnFocus = false
	box.Parent = card
	round(box, 6)

	box.FocusLost:Connect(function()
		if box.Text ~= "" then
			onSubmit(box.Text)
		end
	end)
end

-- ============================================================================
-- FUNCIONES DE GRÁFICOS, EMOTES, APARIENCIA, EFECTOS, ESP Y AUDIO
-- ============================================================================
local function applyLightingPreset(preset: string)
	State.LightingPreset = preset
	for _, child in ipairs(Lighting:GetChildren()) do
		if child:GetAttribute("NovyxPreset") then
			child:Destroy()
		end
	end

	if preset == "Original" then
		Lighting.Ambient = State.OriginalLighting.Ambient
		Lighting.OutdoorAmbient = State.OriginalLighting.OutdoorAmbient
		Lighting.ColorShift_Top = State.OriginalLighting.ColorShift_Top
		Lighting.FogColor = State.OriginalLighting.FogColor
		Lighting.FogEnd = State.OriginalLighting.FogEnd
		Lighting.Brightness = State.OriginalLighting.Brightness
	elseif preset == "Noche Sangrienta" then
		Lighting.Brightness = 0.8
		Lighting.Ambient = Color3.fromRGB(65, 10, 18)
		Lighting.OutdoorAmbient = Color3.fromRGB(90, 12, 22)
		Lighting.FogColor = Color3.fromRGB(45, 5, 10)
		Lighting.FogEnd = 260
		local cc = Instance.new("ColorCorrectionEffect")
		cc:SetAttribute("NovyxPreset", true)
		cc.TintColor = Color3.fromRGB(255, 130, 145)
		cc.Contrast = 0.22
		cc.Parent = Lighting
	elseif preset == "Atardecer Fantástico" then
		Lighting.Brightness = 2.1
		Lighting.Ambient = Color3.fromRGB(120, 75, 155)
		Lighting.OutdoorAmbient = Color3.fromRGB(245, 145, 110)
		Lighting.FogColor = Color3.fromRGB(210, 125, 175)
		Lighting.FogEnd = 420
		local bloom = Instance.new("BloomEffect")
		bloom:SetAttribute("NovyxPreset", true)
		bloom.Intensity = 0.65
		bloom.Parent = Lighting
	elseif preset == "Sin Vida" then
		Lighting.Brightness = 1.1
		Lighting.Ambient = Color3.fromRGB(95, 95, 95)
		Lighting.OutdoorAmbient = Color3.fromRGB(110, 110, 110)
		Lighting.FogColor = Color3.fromRGB(120, 120, 120)
		Lighting.FogEnd = 220
		local cc = Instance.new("ColorCorrectionEffect")
		cc:SetAttribute("NovyxPreset", true)
		cc.Saturation = -0.95
		cc.Contrast = 0.15
		cc.Parent = Lighting
	elseif preset == "El Final" then
		Lighting.Brightness = 0.6
		Lighting.Ambient = Color3.fromRGB(48, 22, 92)
		Lighting.OutdoorAmbient = Color3.fromRGB(32, 16, 68)
		Lighting.FogColor = Color3.fromRGB(22, 12, 46)
		Lighting.FogEnd = 190
		local cc = Instance.new("ColorCorrectionEffect")
		cc:SetAttribute("NovyxPreset", true)
		cc.TintColor = Color3.fromRGB(190, 155, 255)
		cc.Contrast = 0.25
		cc.Parent = Lighting
	elseif preset == "Skype" then
		Lighting.Brightness = 2.4
		Lighting.Ambient = Color3.fromRGB(70, 165, 245)
		Lighting.OutdoorAmbient = Color3.fromRGB(110, 200, 255)
		Lighting.FogColor = Color3.fromRGB(165, 225, 255)
		Lighting.FogEnd = 550
	end
end

local function syncFloatingButtons()
	if not State.ShowFloatingButtons then
		openBtn.Visible = false
		quickEmoteBtn.Visible = false
		return
	end

	if State.AutoSelectButtons then
		local isTouch = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled
		openBtn.Visible = true
		quickEmoteBtn.Visible = isTouch
		return
	end

	if State.FloatingSelection == "Solo Open HUB" then
		openBtn.Visible = true
		quickEmoteBtn.Visible = false
	elseif State.FloatingSelection == "Solo Emote Rápido" then
		openBtn.Visible = false
		quickEmoteBtn.Visible = true
	else
		openBtn.Visible = true
		quickEmoteBtn.Visible = true
	end
end

local function resetFloatingPositions()
	openBtn.Position = UDim2.new(0, 16, 0.5, -20)
	quickEmoteBtn.Position = UDim2.new(0, 16, 0.5, 28)
	mainWindow.Position = UDim2.fromScale(0.5, 0.5)
end

local function playEmoteByName(emoteName: string)
	local char = player.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	local animator = hum and hum:FindFirstChildOfClass("Animator")
	local animId = EMOTE_IDS[emoteName]
	if animator and animId then
		local anim = Instance.new("Animation")
		anim.AnimationId = animId
		local track = animator:LoadAnimation(anim)
		track:Play()
	end
end

local function setCharacterInvisible(enabled: boolean)
	local char = player.Character
	if not char then return end

	if enabled then
		State.SavedTransparency = {}
		for _, desc in ipairs(char:GetDescendants()) do
			if desc:IsA("BasePart") and desc.Name ~= "HumanoidRootPart" then
				State.SavedTransparency[desc] = desc.Transparency
				desc.Transparency = 1
			elseif desc:IsA("Decal") then
				desc.Transparency = 1
			end
		end
	else
		for part, orig in pairs(State.SavedTransparency) do
			if part and part.Parent then
				part.Transparency = orig
			end
		end
		State.SavedTransparency = {}
	end
end

local function refreshCharacterEffects()
	local char = player.Character
	local root = char and char:FindFirstChild("HumanoidRootPart") :: BasePart?
	if not root then return end

	local oldParticles = root:FindFirstChild("NovyxParticles")
	if oldParticles then oldParticles:Destroy() end

	local oldAura = root:FindFirstChild("NovyxAura")
	if oldAura then oldAura:Destroy() end

	if State.ParticlesEnabled then
		local pe = Instance.new("ParticleEmitter")
		pe.Name = "NovyxParticles"
		pe.Color = ColorSequence.new(State.EffectColor)
		pe.Size = NumberSequence.new(0.45, 0)
		pe.Rate = 20
		pe.Lifetime = NumberRange.new(0.6, 1.1)
		pe.Speed = NumberRange.new(2, 5)
		pe.Parent = root
	end

	if State.AuraEnabled then
		local hl = Instance.new("Highlight")
		hl.Name = "NovyxAura"
		hl.Adornee = char
		hl.FillColor = State.EffectColor
		hl.OutlineColor = THEME.Blue
		hl.FillTransparency = 0.65
		hl.OutlineTransparency = 0.1
		hl.Parent = root
	end
end

-- ESP exclusivo sobre elementos de prueba en Workspace.TrainingDummies
local function refreshPracticeEsp()
	for _, obj in ipairs(Workspace:GetDescendants()) do
		if obj.Name == "NovyxPracticeEsp" then
			obj:Destroy()
		end
	end

	if not State.VisualEspEnabled then return end
	local dummiesFolder = Workspace:FindFirstChild("TrainingDummies")
	if not dummiesFolder then return end

	local char = player.Character
	local root = char and char:FindFirstChild("HumanoidRootPart") :: BasePart?

	for _, dummy in ipairs(dummiesFolder:GetChildren()) do
		if dummy:IsA("Model") then
			local dRoot = dummy:FindFirstChild("HumanoidRootPart") :: BasePart?
			if dRoot then
				local dist = root and math.floor((root.Position - dRoot.Position).Magnitude) or 0
				if dist <= State.VisualMaxDistance then
					if State.ObjectIndicators and (State.MarkerStyle == "Resaltado + Etiqueta" or State.MarkerStyle == "Solo Resaltado") then
						local hl = Instance.new("Highlight")
						hl.Name = "NovyxPracticeEsp"
						hl.Adornee = dummy
						hl.FillColor = State.VisualColor
						hl.OutlineColor = THEME.Text
						hl.FillTransparency = 0.7
						hl.Parent = dummy
					end

					if State.VisualInfoEnabled and (State.MarkerStyle == "Resaltado + Etiqueta" or State.MarkerStyle == "Solo Etiqueta") then
						local bb = Instance.new("BillboardGui")
						bb.Name = "NovyxPracticeEsp"
						bb.Size = UDim2.fromOffset(140, 30)
						bb.StudsOffset = Vector3.new(0, 3.2, 0)
						bb.AlwaysOnTop = true
						bb.Adornee = dRoot
						bb.Parent = dummy

						local lbl = Instance.new("TextLabel")
						lbl.Size = UDim2.fromScale(1, 1)
						lbl.BackgroundColor3 = THEME.BgMain
						lbl.BackgroundTransparency = 0.25
						lbl.Text = string.format("[NPC] %s (%dm)", dummy.Name, dist)
						lbl.TextColor3 = THEME.Text
						lbl.Font = Enum.Font.GothamBold
						lbl.TextSize = 11
						lbl.Parent = bb
						round(lbl, 6)
					end
				end
			end
		end
	end
end

local musicSound = Instance.new("Sound")
musicSound.Name = "NovyxMusicPlayer"
musicSound.Looped = true
musicSound.Volume = State.MusicVolume
musicSound.Parent = SoundService

local function syncMusicPlayer()
	local track = MUSIC_PLAYLIST[State.MusicTrackIndex] or MUSIC_PLAYLIST[1]
	musicSound.SoundId = track.SoundId
	musicSound.Volume = State.MusicVolume
	if State.MusicPlaying then
		musicSound:Play()
	else
		musicSound:Pause()
	end
end

-- ============================================================================
-- LAS 17 CATEGORÍAS COMPLETAS
-- ============================================================================
categories["INICIO"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "INICIO — PANEL PRINCIPAL")
		addInfoCard(parent, "Estado del HUB", HUB_META.HubName .. " (" .. HUB_META.Version .. ") • Activo")
		addInfoCard(parent, "Entorno de Pruebas", "Dummies activos: " .. tostring(countActiveDummies()) .. " | Racha: " .. tostring(State.Stats.Streak))
		addActionButton(parent, "Generar Dummy Ficticio de Prueba", "CREAR", function()
			spawnSamplePracticeDummy()
			refreshPracticeEsp()
		end)
		addActionButton(parent, "Restaurar Gráficos Originales", "RESTAURAR", function()
			applyLightingPreset("Original")
		end)
	end
}

categories["INFORMACIÓN"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "INFORMACIÓN DEL SISTEMA")
		addInfoCard(parent, "Nombre del HUB", HUB_META.HubName .. " (" .. HUB_META.HeaderTitle .. ")")
		addInfoCard(parent, "Versión", HUB_META.Version)
		addInfoCard(parent, "Información del Script", HUB_META.ScriptInfo)
		addInfoCard(parent, "Información de Actualizaciones", HUB_META.UpdatesInfo)
		addInfoCard(parent, "Créditos", HUB_META.Credits)
		addInfoCard(parent, "Información de la Comunidad", HUB_META.Community)
	end
}

categories["EXTRA SCRIPTS"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "EXTRA SCRIPTS AUTORIZADOS")
		addToggle(parent, "Herramienta — Etiquetas de Distancia", true, function(v)
			State.VisualInfoEnabled = v
			refreshPracticeEsp()
		end)
		addToggle(parent, "Herramienta — Indicadores de Práctica", true, function(v)
			State.DummyIndicators = v
		end)
		addActionButton(parent, "Generar 3 Dummies de Práctica", "GENERAR", function()
			for _ = 1, 3 do
				spawnSamplePracticeDummy()
			end
			refreshPracticeEsp()
		end)
	end
}

categories["GRÁFICOS"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "GRÁFICOS E ILUMINACIÓN")
		addDropdown(parent, "Iluminación / Ambiente", {
			"Original",
			"Noche Sangrienta",
			"Atardecer Fantástico",
			"Sin Vida",
			"El Final",
			"Skype",
		}, State.LightingPreset, function(chosen)
			applyLightingPreset(chosen)
		end)
		addActionButton(parent, "Noche Sangrienta", "APLICAR", function() applyLightingPreset("Noche Sangrienta") end)
		addActionButton(parent, "Atardecer Fantástico", "APLICAR", function() applyLightingPreset("Atardecer Fantástico") end)
		addActionButton(parent, "Sin Vida", "APLICAR", function() applyLightingPreset("Sin Vida") end)
		addActionButton(parent, "El Final", "APLICAR", function() applyLightingPreset("El Final") end)
		addActionButton(parent, "Skype", "APLICAR", function() applyLightingPreset("Skype") end)
		addActionButton(parent, "Restaurar Gráficos Originales", "RESTAURAR", function() applyLightingPreset("Original") end)
	end
}

categories["BOTONES"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "BOTONES FLOTANTES Y EMOTES")
		addToggle(parent, "Mostrar Botones Flotantes", State.ShowFloatingButtons, function(v)
			State.ShowFloatingButtons = v
			syncFloatingButtons()
		end)
		addDropdown(parent, "Seleccionar Botones Visibles", {"Ambos (Open + Emote)", "Solo Open HUB", "Solo Emote Rápido"}, State.FloatingSelection, function(v)
			State.FloatingSelection = v
			syncFloatingButtons()
		end)
		addToggle(parent, "Selección Automática (Móvil / PC)", State.AutoSelectButtons, function(v)
			State.AutoSelectButtons = v
			syncFloatingButtons()
		end)
		addActionButton(parent, "Restablecer Posiciones de Botones", "RESTABLECER", function()
			resetFloatingPositions()
		end)
		addDropdown(parent, "Selector de Emotes", {"Wave", "Dance", "Cheer", "Point"}, State.EmoteSelector, function(v)
			State.EmoteSelector = v
		end)
		addInfoCard(parent, "Mis Emotes Equipados", table.concat(State.EquippedEmotes, ", "))
		addToggle(parent, "Spam de Emotes (Entorno de Prueba)", State.EmoteSpamActive, function(v)
			State.EmoteSpamActive = v
		end)
	end
}

categories["AIMBOT"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "AIMBOT — CAMPO DE PRÁCTICA (SOLO DUMMIES)")
		addToggle(parent, "Campo de Práctica Activo", State.PracticeRange, function(v)
			State.PracticeRange = v
		end)
		addToggle(parent, "Indicadores Visuales de Objetivos", State.DummyIndicators, function(v)
			State.DummyIndicators = v
		end)
		addDropdown(parent, "Selector de Objetivos Ficticios", {"Más cercano", "Centrado en cámara"}, State.SelectedDummyMode, function(v)
			State.SelectedDummyMode = v
		end)
		addSlider(parent, "Configuración de Sensibilidad", 0.05, 1.50, State.Sensitivity, 2, function(v)
			State.Sensitivity = v
		end)
		addToggle(parent, "Indicador FOV de Entrenamiento", State.ShowTrainingFov, function(v)
			State.ShowTrainingFov = v
			fovCircle.Visible = v
		end)
		addSlider(parent, "Radio FOV de Entrenamiento (px)", 40, 260, State.TrainingFovRadius, 0, function(v)
			State.TrainingFovRadius = v
			fovCircle.Size = UDim2.fromOffset(v * 2, v * 2)
		end)
	end
}

categories["VISUAL"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "VISUAL — ESP DE ENTORNO DE PRUEBA")
		addToggle(parent, "ESP de Elementos Propios de Prueba", State.VisualEspEnabled, function(v)
			State.VisualEspEnabled = v
			refreshPracticeEsp()
		end)
		addToggle(parent, "Indicadores de Objetos", State.ObjectIndicators, function(v)
			State.ObjectIndicators = v
			refreshPracticeEsp()
		end)
		addToggle(parent, "Información Visual (Distancia / Nombre)", State.VisualInfoEnabled, function(v)
			State.VisualInfoEnabled = v
			refreshPracticeEsp()
		end)
		addDropdown(parent, "Personalización de Colores", {"Azul Eléctrico", "Morado Neón", "Esmeralda"}, "Azul Eléctrico", function(v)
			if v == "Morado Neón" then
				State.VisualColor = THEME.Purple
			elseif v == "Esmeralda" then
				State.VisualColor = THEME.Success
			else
				State.VisualColor = THEME.Blue
			end
			refreshPracticeEsp()
		end)
		addSlider(parent, "Distancia de Visualización (studs)", 25, 600, State.VisualMaxDistance, 0, function(v)
			State.VisualMaxDistance = v
			refreshPracticeEsp()
		end)
		addDropdown(parent, "Marcadores Configurables", {"Resaltado + Etiqueta", "Solo Resaltado", "Solo Etiqueta"}, State.MarkerStyle, function(v)
			State.MarkerStyle = v
			refreshPracticeEsp()
		end)
	end
}

categories["KILL ALL"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "KILL ALL — NPC FICTICIOS (ENTORNO PRIVADO)")
		addInfoCard(parent, "Alcance Seguro", "Solo afecta NPCs ficticios en Workspace.TrainingDummies.")
		addActionButton(parent, "Crear Dummy Ficticio de Prueba", "CREAR", function()
			spawnSamplePracticeDummy()
			refreshPracticeEsp()
		end)
		addActionButton(parent, "Eliminar Todos los NPC Ficticios", "EJECUTAR", function()
			eliminatePracticeDummies(nil)
		end)
		addToggle(parent, "Eliminación Automática de NPC Ficticios", State.AutoKillDummies, function(v)
			State.AutoKillDummies = v
		end)
	end
}

categories["INVISIBLE"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "INVISIBLE — PERSONAJE DE PRUEBA")
		addToggle(parent, "Efecto Visual de Invisibilidad", State.InvisibleEnabled, function(v)
			State.InvisibleEnabled = v
			setCharacterInvisible(v)
		end)
		addActionButton(parent, "Restaurar Apariencia Original", "RESTAURAR", function()
			State.InvisibleEnabled = false
			setCharacterInvisible(false)
		end)
	end
}

categories["TRAMPASO"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "TRAMPASO — MOVIMIENTO DE PRUEBA")
		addToggle(parent, "Speed Configurable", State.SpeedEnabled, function(v)
			State.SpeedEnabled = v
		end)
		addSlider(parent, "Ajuste de Velocidad (WalkSpeed)", 8, 80, State.WalkSpeedValue, 0, function(v)
			State.WalkSpeedValue = v
		end)
		addToggle(parent, "Salto de Prueba", State.JumpEnabled, function(v)
			State.JumpEnabled = v
		end)
		addSlider(parent, "Ajuste de Salto (JumpPower)", 20, 140, State.JumpPowerValue, 0, function(v)
			State.JumpPowerValue = v
		end)
		addToggle(parent, "Vuelo en Modo de Pruebas", State.FlyEnabled, function(v)
			State.FlyEnabled = v
		end)
		addSlider(parent, "Velocidad de Vuelo", 10, 120, State.FlySpeedValue, 0, function(v)
			State.FlySpeedValue = v
		end)
	end
}

categories["FARMEAR RACHA"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "FARMEAR RACHA — ESTADÍSTICAS DE PRÁCTICA")
		local killsCard = addInfoCard(parent, "Contador de Eliminaciones de NPC", tostring(State.Stats.Kills))
		local streakCard = addInfoCard(parent, "Contador de Rachas de Entrenamiento", tostring(State.Stats.Streak) .. " (Mejor: " .. tostring(State.Stats.BestStreak) .. ")")
		local sessionCard = addInfoCard(parent, "Estadísticas de Sesión", "Tiempo activo: " .. tostring(math.floor(os.clock() - State.Stats.SessionStart)) .. "s | Dummies: " .. tostring(countActiveDummies()))

		addActionButton(parent, "Actualizar Panel Ahora", "ACTUALIZAR", function()
			killsCard.Text = tostring(State.Stats.Kills)
			streakCard.Text = tostring(State.Stats.Streak) .. " (Mejor: " .. tostring(State.Stats.BestStreak) .. ")"
			sessionCard.Text = "Tiempo activo: " .. tostring(math.floor(os.clock() - State.Stats.SessionStart)) .. "s | Dummies: " .. tostring(countActiveDummies())
		end)
		addActionButton(parent, "Reiniciar Racha Actual", "REINICIAR", function()
			State.Stats.Streak = 0
			streakCard.Text = "0 (Mejor: " .. tostring(State.Stats.BestStreak) .. ")"
		end)
	end
}

categories["EFECTOS"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "EFECTOS VISUALES, PARTÍCULAS Y AURAS")
		addToggle(parent, "Partículas de Entrenamiento", State.ParticlesEnabled, function(v)
			State.ParticlesEnabled = v
			refreshCharacterEffects()
		end)
		addToggle(parent, "Aura de Personaje", State.AuraEnabled, function(v)
			State.AuraEnabled = v
			refreshCharacterEffects()
		end)
		addDropdown(parent, "Colores Personalizables", {"Morado Novyx", "Azul Cian", "Verde Neón"}, "Morado Novyx", function(v)
			if v == "Azul Cian" then
				State.EffectColor = THEME.Blue
			elseif v == "Verde Neón" then
				State.EffectColor = THEME.Success
			else
				State.EffectColor = THEME.Purple
			end
			refreshCharacterEffects()
		end)
		addActionButton(parent, "Desactivar Todos los Efectos", "APAGAR", function()
			State.ParticlesEnabled = false
			State.AuraEnabled = false
			refreshCharacterEffects()
		end)
	end
}

categories["ANIMACIONES"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "ANIMACIONES DE PRUEBA")
		addDropdown(parent, "Selector de Animaciones Propias", {"Idle Clásico", "Salto de Prueba", "Carrera Ágil"}, State.SelectedAnim, function(v)
			State.SelectedAnim = v
		end)
		addActionButton(parent, "Reproducir Animación", "REPRODUCIR", function()
			local char = player.Character
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			local animator = hum and hum:FindFirstChildOfClass("Animator")
			local id = ANIM_IDS[State.SelectedAnim]
			if animator and id then
				if State.ActiveAnimTrack then
					State.ActiveAnimTrack:Stop()
				end
				local anim = Instance.new("Animation")
				anim.AnimationId = id
				local track = animator:LoadAnimation(anim)
				track.Looped = true
				track:Play()
				State.ActiveAnimTrack = track
			end
		end)
		addActionButton(parent, "Detener Animación", "DETENER", function()
			if State.ActiveAnimTrack then
				State.ActiveAnimTrack:Stop()
				State.ActiveAnimTrack = nil
			end
		end)
		addActionButton(parent, "Restablecer Animación", "RESTABLECER", function()
			if State.ActiveAnimTrack then
				State.ActiveAnimTrack:Stop()
				State.ActiveAnimTrack = nil
			end
			State.SelectedAnim = "Idle Clásico"
		end)
	end
}

categories["SONIDOS"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "EFECTOS DE SONIDO")
		addToggle(parent, "Activar / Desactivar Sonidos", State.SoundEffectsEnabled, function(v)
			State.SoundEffectsEnabled = v
		end)
		addSlider(parent, "Control de Volumen SFX", 0, 1, State.SfxVolume, 2, function(v)
			State.SfxVolume = v
		end)
		addActionButton(parent, "Probar Efecto de Sonido", "PROBAR", function()
			playUiClick()
		end)
	end
}

categories["MÚSICA"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "REPRODUCTOR DE MÚSICA")
		local currentTrack = MUSIC_PLAYLIST[State.MusicTrackIndex] or MUSIC_PLAYLIST[1]
		local nowPlayingLbl = addInfoCard(parent, "Ahora Suena — Nombre de la Canción", currentTrack.Name)

		local function refreshTrackLabel()
			local t = MUSIC_PLAYLIST[State.MusicTrackIndex] or MUSIC_PLAYLIST[1]
			nowPlayingLbl.Text = t.Name .. (State.MusicPlaying and " (Reproduciendo)" or " (Pausado)")
		end

		addActionButton(parent, "Canción Anterior", "ANTERIOR", function()
			State.MusicTrackIndex = ((State.MusicTrackIndex - 2) % #MUSIC_PLAYLIST) + 1
			syncMusicPlayer()
			refreshTrackLabel()
		end)
		addActionButton(parent, "Canción Siguiente", "SIGUIENTE", function()
			State.MusicTrackIndex = (State.MusicTrackIndex % #MUSIC_PLAYLIST) + 1
			syncMusicPlayer()
			refreshTrackLabel()
		end)
		addActionButton(parent, "Reproducir Música", "PLAY", function()
			State.MusicPlaying = true
			syncMusicPlayer()
			refreshTrackLabel()
		end)
		addActionButton(parent, "Pausar Música", "PAUSA", function()
			State.MusicPlaying = false
			syncMusicPlayer()
			refreshTrackLabel()
		end)
		addActionButton(parent, "Detener Música", "STOP", function()
			State.MusicPlaying = false
			musicSound:Stop()
			refreshTrackLabel()
		end)
		addSlider(parent, "Control de Volumen Música", 0, 1, State.MusicVolume, 2, function(v)
			State.MusicVolume = v
			musicSound.Volume = v
		end)
	end
}

categories["ITEMS"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "CATÁLOGO DE OBJETOS DE PRUEBA")
		local itemNames = {}
		for k in pairs(ITEMS_CATALOG) do
			table.insert(itemNames, k)
		end
		table.sort(itemNames)

		local infoCard = addInfoCard(parent, "Información y Vista Previa", "")
		local function refreshItemInfo()
			local data = ITEMS_CATALOG[State.SelectedItem]
			if data then
				infoCard.Text = State.SelectedItem .. " | " .. data.Tipo .. " | " .. data.Vista
			end
		end
		refreshItemInfo()

		addDropdown(parent, "Selección de Objetos Autorizados", itemNames, State.SelectedItem, function(v)
			State.SelectedItem = v
			refreshItemInfo()
		end)
		addActionButton(parent, "Confirmar Selección de Objeto", "EQUIPAR", function()
			refreshItemInfo()
		end)
	end
}

categories["IDENTIDAD"] = {
	Builder = function(parent: Frame)
		addSectionHeader(parent, "IDENTIDAD Y PERFIL")
		addInfoCard(parent, "Información del Personaje", player.Name .. " (ID: " .. tostring(player.UserId) .. ")")
		addTextInput(parent, "Nombre Visible en Interfaz", "Escribe tu nombre visible...", State.CustomDisplayName, function(txt)
			State.CustomDisplayName = txt
			titleLabel.Text = HUB_META.HeaderTitle .. "  •  " .. State.CustomDisplayName
		end)
		addDropdown(parent, "Configuración de Colores del HUB", {"Morado + Azul", "Morado Intenso", "Azul Cian"}, State.AccentMode, function(v)
			State.AccentMode = v
		end)
		addTextInput(parent, "Personalización del Perfil", "Etiqueta de perfil...", State.ProfileTag, function(txt)
			State.ProfileTag = txt
		end)
	end
}

-- ============================================================================
-- NAVEGACIÓN, BUSCADOR EN TIEMPO REAL Y CONTROLES DE VENTANA
-- ============================================================================
local categoryOrder = {
	"INICIO",
	"INFORMACIÓN",
	"EXTRA SCRIPTS",
	"GRÁFICOS",
	"BOTONES",
	"AIMBOT",
	"VISUAL",
	"KILL ALL",
	"INVISIBLE",
	"TRAMPASO",
	"FARMEAR RACHA",
	"EFECTOS",
	"ANIMACIONES",
	"SONIDOS",
	"MÚSICA",
	"ITEMS",
	"IDENTIDAD",
}

local function openCategory(name: string)
	currentCategoryName = name
	clearContent()

	for catName, entry in pairs(categories) do
		local active = (catName == name)
		entry.Button.BackgroundColor3 = active and THEME.PurpleDark or THEME.BgCard
		entry.Button.TextColor3 = active and THEME.Text or THEME.TextMuted
	end

	local entry = categories[name]
	if entry then
		entry.Builder(contentArea)
	end
end

for idx, catName in ipairs(categoryOrder) do
	local catEntry = categories[catName]
	if catEntry then
		local btn = Instance.new("TextButton")
		btn.Name = "Nav_" .. catName
		btn.LayoutOrder = idx
		btn.Size = UDim2.new(1, 0, 0, 28)
		btn.BackgroundColor3 = THEME.BgCard
		btn.Text = catName
		btn.TextColor3 = THEME.TextMuted
		btn.Font = Enum.Font.GothamBold
		btn.TextSize = 11
		btn.Parent = navScroll
		round(btn, 8)

		catEntry.Button = btn
		btn.MouseButton1Click:Connect(function()
			playUiClick()
			searchBox.Text = ""
			openCategory(catName)
		end)
	end
end

searchBox:GetPropertyChangedSignal("Text"):Connect(function()
	local query = string.lower(searchBox.Text)

	for catName, catEntry in pairs(categories) do
		local matchCat = query == "" or string.find(string.lower(catName), query, 1, true) ~= nil
		catEntry.Button.Visible = matchCat
	end

	if query == "" then
		openCategory(currentCategoryName)
		return
	end

	clearContent()
	local holder = Instance.new("Frame")
	holder.Size = UDim2.new(1, 0, 0, 0)
	holder.AutomaticSize = Enum.AutomaticSize.Y
	holder.BackgroundTransparency = 1
	holder.Parent = contentArea

	local rLayout = Instance.new("UIListLayout")
	rLayout.Padding = UDim.new(0, 6)
	rLayout.Parent = holder

	addSectionHeader(holder, 'Resultados para "' .. searchBox.Text .. '"')

	for _, catName in ipairs(categoryOrder) do
		local catEntry = categories[catName]
		if catEntry then
			local tempFolder = Instance.new("Frame")
			catEntry.Builder(tempFolder)
			for _, item in ipairs(tempFolder:GetChildren()) do
				if item:IsA("GuiObject") and string.find(string.lower(item.Name), query, 1, true) then
					item.Parent = holder
				else
					item:Destroy()
				end
			end
			tempFolder:Destroy()
		end
	end
end)

openBtn.MouseButton1Click:Connect(function()
	playUiClick()
	mainWindow.Visible = not mainWindow.Visible
	if mainWindow.Visible then
		State.Minimized = false
		bodyFrame.Visible = true
		mainWindow.Size = UDim2.fromOffset(650, 400)
		openCategory(currentCategoryName)
	end
end)

quickEmoteBtn.MouseButton1Click:Connect(function()
	playUiClick()
	playEmoteByName(State.EmoteSelector)
end)

minBtn.MouseButton1Click:Connect(function()
	playUiClick()
	State.Minimized = not State.Minimized
	bodyFrame.Visible = not State.Minimized
	local targetSize = State.Minimized and UDim2.fromOffset(650, 44) or UDim2.fromOffset(650, 400)
	TweenService:Create(mainWindow, TweenInfo.new(0.18), {Size = targetSize}):Play()
end)

closeBtn.MouseButton1Click:Connect(function()
	playUiClick()
	mainWindow.Visible = false
end)

makeDraggable(mainWindow, topBar)
makeDraggable(openBtn, openBtn)
makeDraggable(quickEmoteBtn, quickEmoteBtn)

-- ============================================================================
-- BUCLE CONTINUO (MOVIMIENTO, VUELO, DUMMIES DE PRÁCTICA, EMOTES)
-- ============================================================================
local lastEmoteSpamTick = 0
local lastAutoKillTick = 0
local lastEspTick = 0

RunService.Heartbeat:Connect(function()
	local char = player.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	local root = char and char:FindFirstChild("HumanoidRootPart") :: BasePart?

	if hum and hum.Health > 0 then
		if State.SpeedEnabled then
			hum.WalkSpeed = State.WalkSpeedValue
		end
		if State.JumpEnabled then
			hum.UseJumpPower = true
			hum.JumpPower = State.JumpPowerValue
		end
	end

	-- Vuelo en modo de pruebas (compatible con joystick móvil y teclado PC)
	if State.FlyEnabled and root and hum then
		hum.PlatformStand = true
		local moveDir = hum.MoveDirection
		local upDown = 0
		if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
			upDown += 1
		end
		if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
			upDown -= 1
		end
		root.AssemblyLinearVelocity = (moveDir * State.FlySpeedValue) + Vector3.new(0, upDown * State.FlySpeedValue * 0.75, 0)
	elseif hum and hum.PlatformStand and not State.FlyEnabled then
		hum.PlatformStand = false
	end

	-- Spam de emotes en entorno de prueba
	if State.EmoteSpamActive and os.clock() - lastEmoteSpamTick >= 1.5 then
		lastEmoteSpamTick = os.clock()
		playEmoteByName(State.EmoteSelector)
	end

	-- Eliminación de NPCs ficticios en TrainingDummies
	if State.AutoKillDummies and os.clock() - lastAutoKillTick >= 0.75 then
		lastAutoKillTick = os.clock()
		eliminatePracticeDummies(nil)
	end

	-- Refresco ligero de ESP de entorno de prueba
	if State.VisualEspEnabled and os.clock() - lastEspTick >= 1.2 then
		lastEspTick = os.clock()
		refreshPracticeEsp()
	end
end)

player.CharacterAdded:Connect(function()
	task.wait(0.35)
	if State.InvisibleEnabled then
		setCharacterInvisible(true)
	end
	refreshCharacterEffects()
	refreshPracticeEsp()
end)

openCategory("INICIO")
