--[[
	KRONOS — MECHA ANCIÃO DE GUERRA (steampunk / cyberpunk de engrenagens)
	LocalScript ÚNICO, 100% MOBILE-FIRST
	=======================================================================
	ONDE COLOCAR: StarterPlayer > StarterPlayerScripts > novo LocalScript (cole tudo).
	Não precisa de modelo, módulo, servidor nem asset: tudo é gerado por código.

	O QUE TEM
	  • MECHA GIGANTE (~78 studs, ~3x o anterior): bronze envelhecido, ferro, ferrugem, núcleo de energia,
	    olho com íris que escaneia, chaminés soltando fumaça, mochila-fornalha, guindaste com corrente,
	    radar giratório, canhão no braço direito, garra no esquerdo, ombreiras com espinhos.
	  • VIVO: 12 ENGRENAGENS girando (mais rápidas quando anda/turbo), ~12 PISTÕES hidráulicos que
	    estendem/encolhem conforme as juntas dobram, ~12 CABOS e CORRENTES com física (Verlet) que balançam,
	    anéis do reator girando, vapor nas juntas, vazamento de fumaça, luzes que pulsam.
	  • PERSONAGEM NOVO: o avatar do Roblox é escondido e você vira um OPERADOR MILITAR TECNOLÓGICO de 6 PARTES
	    (cabeça, tronco, 2 braços, 2 pernas) com animações próprias: parado, andar, correr, pular, cair,
	    pousar, ESCALAR e SENTADO pilotando.
	  • ESCALADA: o mecha fica estacionado. Chegue na escada (aparece ESCALAR), suba com o analógico
	    (cima/baixo), no topo toque ENTRAR: o operador passa pela sacada, as portas de vidro abrem, ele senta
	    na poltrona (visível pelo vidro), as portas fecham e os sistemas ligam.
	  • PILOTAR: andar pesado (passos fazem o chão tremer), torso gira para onde a câmera aponta,
	    canhão automático (FOGO) com calor, TURBO (chaminés em chamas), salto com jatos, pouso com impacto.
	  • SAIR: o mecha desliga, as portas abrem, o operador volta à sacada e você desce pela escada
	    (ou pula da sacada).

	CONTROLES
	  Celular : botões na tela (ESCALAR/ENTRAR/SAIR, FOGO, TURBO) + analógico + botão de pulo padrão
	  PC      : E interage | F fogo | Shift turbo | W/S sobe/desce a escada | Espaço pula/solta
	  Console : X interage | R2 fogo | L1 turbo

	LIMITAÇÃO (é só LocalScript): somente VOCÊ vê o mecha e o operador.
]]

--------------------------------------------------------------------------------
-- SERVIÇOS
--------------------------------------------------------------------------------
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ContextActionService = game:GetService("ContextActionService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer

--------------------------------------------------------------------------------
-- CONFIGURAÇÃO (ajuste aqui)
--------------------------------------------------------------------------------
local CFG = {
	Name = "KRONOS",
	Callsign = "K-01",

	Spawn = { Offset = Vector3.new(0, 0, -95), Fixed = nil }, -- Offset relativo ao seu 1º spawn; Fixed = CFrame fixo
	LadderRange = 9,       -- distância até a base da escada para poder escalar
	ClimbSpeed = 7,        -- studs/s na escada

	Keys = { Interact = Enum.KeyCode.E, Boost = Enum.KeyCode.LeftShift, Fire = Enum.KeyCode.F },
	Pad = { Interact = Enum.KeyCode.ButtonX, Boost = Enum.KeyCode.ButtonL1, Fire = Enum.KeyCode.ButtonR2 },

	-- HipHeight + RootSize.Y/2 = 36.5 (altura do quadril do mecha). O RootPart vira uma caixa alta de colisão.
	Pilot = { WalkSpeed = 38, SprintSpeed = 68, JumpPower = 112, HipHeight = 1.5, RootSize = Vector3.new(26, 70, 22), TurnRate = 1.5 },
	Boost = { Max = 100, Drain = 24, Regen = 14, RegenDelay = 1.3, MinToStart = 12 },
	Weapon = { Rate = 4.5, HeatPerShot = 7, Cool = 22, LockAt = 100, Unlock = 45, Speed = 600, Range = 1200 },

	Board = { Mount = 0.5, Door0 = 0.4, Walk0 = 1.2, WalkSpeed = 10, Sit = 0.6, PowerTime = 1.8 },
	Leave = { PowerDown = 1.2, Door0 = 0.4, Stand0 = 1.4, Stand = 0.5, WalkSpeed = 10, CloseTime = 1.0 },

	Anim = { Stride = 0.075, LegSwing = 0.5, Knee = 0.8, ArmSwing = 0.28, Lean = 0.08, Bob = 1.1, Sway = 0.035 },

	Camera = {
		Fov = 70, PilotFov = 76, BoostFovAdd = 12, MaxShake = 1.3,
		Pilot = { Min = 120, Max = 340, Offset = Vector3.new(0, 24, 0) },
		Climb = { Min = 16, Max = 100 },
	},

	ReduceMotion = false,  -- true = quase sem tremor/blur
	AutoQuality = true,    -- ajusta efeitos pelo FPS
	ShowButtonsOnDesktop = false,

	Colors = {
		Accent = Color3.fromRGB(255, 160, 50), Green = Color3.fromRGB(120, 255, 160),
		Dark = Color3.fromRGB(16, 12, 10), Danger = Color3.fromRGB(255, 70, 60), Cyan = Color3.fromRGB(80, 220, 255),
	},

	-- Sons: cole "rbxassetid://ID" (vazio = sem som, sem erro)
	Sounds = {
		HatchOpen = { Id = "", Volume = 1.2 }, HatchClose = { Id = "", Volume = 1.2 },
		PowerUp = { Id = "", Volume = 1.2 },   PowerDown = { Id = "", Volume = 1.2 },
		Online = { Id = "", Volume = 1.4 },    Clang = { Id = "", Volume = 1.2 },
		Step = { Id = "", Volume = 1.4 },      Land = { Id = "", Volume = 1.5 },
		Fire = { Id = "", Volume = 1.0 },      Boost = { Id = "", Volume = 1.0 },
		Rung = { Id = "", Volume = 0.6 },
	},
}

local PI = math.pi
local HALF_PI = PI / 2
local REST_PELVIS = CFrame.new(0, 36.5, 0)
local V3, CF, AN = Vector3.new, CFrame.new, CFrame.Angles

-- Geometria fixa (no espaço do mecha: origem no chão sob o centro, frente = -Z)
local LAD_BASE, LAD_TOP = V3(-6, 0, -34), V3(-6, 44, -25)
local PATH_LOCAL = { V3(-6, 47, -20.5), V3(0, 47, -14), V3(0, 47, -10), V3(0, 47, -8) } -- sacada -> porta -> poltrona
local SEAT_LOCAL = V3(0, 47.8, -6.2)

--------------------------------------------------------------------------------
-- UTILIDADES
--------------------------------------------------------------------------------
local function clamp01(x) return x < 0 and 0 or (x > 1 and 1 or x) end
local function smooth(x) x = clamp01(x) return x * x * (3 - 2 * x) end
local function easeOut(x) x = clamp01(x) return 1 - (1 - x) ^ 3 end
local function easeInOut(x)
	x = clamp01(x)
	if x < 0.5 then return 4 * x * x * x end
	return 1 - ((-2 * x + 2) ^ 3) / 2
end
local function damp(rate, dt) return 1 - math.exp(-rate * dt) end
local function wrap(a) return (a + PI) % (2 * PI) - PI end
local function yawOf(v) return math.atan2(-v.X, -v.Z) end
local function lerp(a, b, t) return a + (b - a) * t end
local function hash01(s)
	local h = 7
	for i = 1, #s do h = (h * 31 + string.byte(s, i)) % 1000003 end
	return (h % 997) / 997
end
local function bez(a, c, b, r) local k = 1 - r return a * (k * k) + c * (2 * k * r) + b * (r * r) end

local Actions = {} -- preenchido adiante

--------------------------------------------------------------------------------
-- QUALIDADE ADAPTATIVA
--------------------------------------------------------------------------------
local IS_MOBILE = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled
local DETAIL = IS_MOBILE and 1 or 2 -- 1 = menos peças/dentes/cabos (celular)
local TIERS = {
	{ part = 0.25, shake = 0.45, blur = 0, lights = false },
	{ part = 0.6, shake = 0.75, blur = 8, lights = true },
	{ part = 1.0, shake = 1.0, blur = 14, lights = true },
}
local tierIdx = IS_MOBILE and 2 or 3
local tierMax = IS_MOBILE and 2 or 3
do
	local ok, lvl = pcall(function() return settings().Rendering.QualityLevel.Value end)
	if ok and lvl > 0 then
		if lvl <= 4 then tierIdx = 1 elseif lvl <= 9 then tierIdx = math.min(tierIdx, 2) end
	end
	if CFG.ReduceMotion then tierIdx = 1 end
end
local function Q() return TIERS[tierIdx] end
do
	local ema, timer, since, good = 1 / 60, 0, 0, 0
	RunService.Heartbeat:Connect(function(dt)
		if not CFG.AutoQuality then return end
		dt = math.clamp(dt, 1 / 240, 0.25)
		ema += (dt - ema) * 0.05
		timer += dt
		since += dt
		if timer < 2 then return end
		timer = 0
		if since < 6 then return end
		local fps = 1 / ema
		if fps < 30 and tierIdx > 1 then tierIdx -= 1; since = 0; good = 0
		elseif fps > 56 and tierIdx < tierMax then
			good += 2
			if good >= 14 then tierIdx += 1; since = 0; good = 0 end
		else good = 0 end
	end)
end

--------------------------------------------------------------------------------
-- ESTADO
--------------------------------------------------------------------------------
local S = {
	ready = false, char = nil, hum = nil, root = nil, orig = nil, conns = {},
	-- mecha
	model = nil, N = nil, list = nil, fx = nil, O = nil, link = nil, ladder = nil,
	-- operador (6 partes)
	sold = nil, soldBase = CF(0, 0.5, 0), seatW = 0,
	-- fluxo: Free | Climb | Board | Pilot | Leave
	state = "Free", seq = nil, climb = nil,
	hatch = 0, hatchV = 0, hatchGoal = 0, power = 0, powerGoal = 0,
	boosting = false, firing = false, energy = CFG.Boost.Max, lastBoostEnd = 0,
	heat = 0, overheated = false, nextShot = 0, lastFire = -10,
	lastHrpCF = nil, nearLadder = false,
}
local M = { -- movimento/animação do MECHA
	phase = 0, lastStep = 0, spd = 0, air = 0, airTime = 0, lastVy = 0, grounded = true, wasGrounded = true,
	spring = 0, springV = 0, recoil = 0, recoilV = 0, aimW = 0, yawRate = 0, lastYaw = nil, hs = 0,
	camYaw = 0, camPitch = 0, torsoYaw = 0, headYaw = 0, rel = 0, aligning = false, ventTimer = 6, tick = 0,
	gearK = 0.06, eyeT = 0,
}
local FM = { -- movimento/animação do OPERADOR
	phase = 0, spd = 0, run = 0, air = 0, airTime = 0, lastVy = 0, grounded = true, wasGrounded = true,
	spring = 0, springV = 0, climbPh = 0, idleT = 0, override = nil, ladderS = 0,
}
local avatarParts = {}

--------------------------------------------------------------------------------
-- EMISSORES (criados uma vez, nunca no loop)
--------------------------------------------------------------------------------
local TEX_SPARK = "rbxasset://textures/particles/sparkles_main.dds"
local TEX_SMOKE = "rbxasset://textures/particles/smoke_main.dds"
local TEX_FIRE = "rbxasset://textures/particles/fire_main.dds"
local function nseq(a, b, c)
	if c then return NumberSequence.new({ NumberSequenceKeypoint.new(0, a), NumberSequenceKeypoint.new(0.5, b), NumberSequenceKeypoint.new(1, c) }) end
	return NumberSequence.new(a, b)
end
local function emitter(tex)
	local p = Instance.new("ParticleEmitter")
	p.Texture = tex; p.Enabled = false; p.Rate = 0; p.LockedToPart = false
	return p
end
local function newSpark()
	local p = emitter(TEX_SPARK)
	p.Color = ColorSequence.new(Color3.fromRGB(255, 250, 200), Color3.fromRGB(255, 140, 40))
	p.LightEmission = 1; p.Size = nseq(1.2, 0.6, 0)
	p.Lifetime = NumberRange.new(0.3, 0.7); p.Speed = NumberRange.new(25, 60)
	p.SpreadAngle = Vector2.new(180, 180); p.Acceleration = V3(0, -90, 0); p.Drag = 2
	p.Transparency = nseq(0, 0.2, 1); p.Rotation = NumberRange.new(0, 360)
	return p
end
local function newDust()
	local p = emitter(TEX_SMOKE)
	p.Color = ColorSequence.new(Color3.fromRGB(170, 158, 138))
	p.Size = nseq(6, 18, 30); p.Lifetime = NumberRange.new(1.0, 2.0); p.Speed = NumberRange.new(14, 34)
	p.SpreadAngle = Vector2.new(85, 85); p.EmissionDirection = Enum.NormalId.Top; p.Drag = 3
	p.Transparency = nseq(0.5, 0.75, 1); p.Rotation = NumberRange.new(0, 360); p.RotSpeed = NumberRange.new(-40, 40)
	return p
end
local function newSteam(big)
	local p = emitter(TEX_SMOKE)
	p.Color = ColorSequence.new(Color3.fromRGB(235, 238, 242))
	p.Size = big and nseq(2, 7, 12) or nseq(1, 3.5, 6)
	p.Lifetime = NumberRange.new(1.0, 1.8); p.Speed = NumberRange.new(10, 22)
	p.SpreadAngle = Vector2.new(14, 14); p.EmissionDirection = Enum.NormalId.Top; p.Drag = 1.5
	p.Transparency = nseq(0.4, 0.7, 1); p.Rotation = NumberRange.new(0, 360)
	return p
end
local function newSmoke() -- fumaça escura das chaminés
	local p = emitter(TEX_SMOKE)
	p.Color = ColorSequence.new(Color3.fromRGB(46, 44, 44), Color3.fromRGB(110, 108, 106))
	p.Size = nseq(3, 10, 18); p.Lifetime = NumberRange.new(2.5, 4.5); p.Speed = NumberRange.new(10, 18)
	p.SpreadAngle = Vector2.new(10, 10); p.EmissionDirection = Enum.NormalId.Top; p.Acceleration = V3(4, 5, 0)
	p.Transparency = nseq(0.25, 0.55, 1); p.Rotation = NumberRange.new(0, 360); p.RotSpeed = NumberRange.new(-20, 20)
	p:SetAttribute("BaseRate", 9)
	return p
end
local function newFlame()
	local p = emitter(TEX_FIRE)
	p.Color = ColorSequence.new(Color3.fromRGB(255, 235, 170), Color3.fromRGB(255, 90, 20))
	p.LightEmission = 1; p.Size = nseq(5, 3.5, 0.6)
	p.Lifetime = NumberRange.new(0.35, 0.65); p.Speed = NumberRange.new(40, 70)
	p.SpreadAngle = Vector2.new(6, 6); p.EmissionDirection = Enum.NormalId.Top
	p.Transparency = nseq(0, 0.3, 1)
	p:SetAttribute("BaseRate", 90)
	return p
end
local function burst(p, n)
	if p then p:Emit(math.max(1, math.floor(n * Q().part + 0.5))) end
end

--------------------------------------------------------------------------------
-- CÂMERA
--------------------------------------------------------------------------------
local Cam = {
	trauma = 0, flash = 0, fov = 70, fovT = 70, zMin = 0.5, zMax = 128, zMinT = 0.5, zMaxT = 128,
	dMin = 0.5, dMax = 128, off = Vector3.zero, offT = Vector3.zero, blurT = 0, aMin = nil, aMax = nil,
}
local blurFx = Instance.new("BlurEffect")
blurFx.Name = "MechaBlur"; blurFx.Size = 0; blurFx.Enabled = false; blurFx.Parent = Lighting
local ccFx = Instance.new("ColorCorrectionEffect")
ccFx.Name = "MechaFlash"; ccFx.Parent = Lighting

local function shake(a) Cam.trauma = math.min(1, Cam.trauma + a * (CFG.ReduceMotion and 0.15 or Q().shake)) end
local function flashScreen(a) Cam.flash = math.max(Cam.flash, a) end

local function camTargets()
	local C = CFG.Camera
	local st = S.state
	if st == "Pilot" or (st == "Board" and S.seq and S.seq.sealed) then
		Cam.fovT = C.PilotFov + (S.boosting and C.BoostFovAdd or 0)
		Cam.offT = C.Pilot.Offset
		Cam.zMinT, Cam.zMaxT = C.Pilot.Min, C.Pilot.Max
	elseif st == "Climb" or st == "Board" or st == "Leave" then
		Cam.fovT = C.Fov; Cam.offT = Vector3.zero
		Cam.zMinT, Cam.zMaxT = C.Climb.Min, C.Climb.Max
	else
		Cam.fovT = C.Fov; Cam.offT = Vector3.zero
		Cam.zMinT, Cam.zMaxT = Cam.dMin, Cam.dMax
	end
end

local function camRender(dt)
	dt = math.min(dt, 0.1)
	local cam = Workspace.CurrentCamera
	if not cam then return end
	Cam.fov += (Cam.fovT - Cam.fov) * damp(5, dt)
	if math.abs(cam.FieldOfView - Cam.fov) > 0.01 then cam.FieldOfView = Cam.fov end
	Cam.zMin += (Cam.zMinT - Cam.zMin) * damp(1.6, dt)
	Cam.zMax += (Cam.zMaxT - Cam.zMax) * damp(1.6, dt)
	if not Cam.aMin or math.abs(Cam.zMin - Cam.aMin) > 0.05 or math.abs(Cam.zMax - Cam.aMax) > 0.05 then
		local nMin, nMax = Cam.zMin, math.max(Cam.zMax, Cam.zMin + 4)
		pcall(function()
			if nMax > player.CameraMaxZoomDistance then
				player.CameraMaxZoomDistance = nMax; player.CameraMinZoomDistance = nMin
			else
				player.CameraMinZoomDistance = nMin; player.CameraMaxZoomDistance = nMax
			end
		end)
		Cam.aMin, Cam.aMax = nMin, nMax
	end
	local hum = S.hum
	if hum and hum.Parent then
		Cam.off = Cam.off:Lerp(Cam.offT, damp(2.2, dt))
		if (hum.CameraOffset - Cam.off).Magnitude > 0.01 then hum.CameraOffset = Cam.off end
	end
	if Cam.trauma > 0 then
		Cam.trauma = math.max(0, Cam.trauma - dt * 1.4)
		local sh = Cam.trauma * Cam.trauma * CFG.Camera.MaxShake
		local t = os.clock() * 26
		cam.CFrame = cam.CFrame * CF(math.noise(t, 0.3, 1.7) * sh * 0.4, math.noise(0.7, t, 7.3) * sh * 0.4, 0)
			* AN(0, 0, math.noise(t, t, 13.1) * sh * 0.02)
	end
	local bt = (Q().blur > 0 and not CFG.ReduceMotion) and Cam.blurT * Q().blur or 0
	blurFx.Size += (bt - blurFx.Size) * damp(10, dt)
	blurFx.Enabled = blurFx.Size > 0.2
	Cam.flash = math.max(0, Cam.flash - dt * 2.2)
	ccFx.Brightness = Cam.flash * 0.45
	ccFx.Contrast = Cam.flash * 0.25
end

--------------------------------------------------------------------------------
-- INTERFACE MOBILE
--------------------------------------------------------------------------------
local UI = { buttons = {}, built = false }
local function corner(p, r) local c = Instance.new("UICorner") c.CornerRadius = r c.Parent = p return c end
local function stroke(p, color, th, tr)
	local s = Instance.new("UIStroke")
	s.Color = color; s.Thickness = th; s.Transparency = tr or 0
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border; s.Parent = p
	return s
end
local function label(parent, font, color, xa)
	local l = Instance.new("TextLabel")
	l.BackgroundTransparency = 1; l.Font = font; l.TextColor3 = color; l.TextScaled = true
	l.TextXAlignment = xa or Enum.TextXAlignment.Center; l.Parent = parent
	return l
end
local function makeButton(gui, key, color)
	local b = Instance.new("TextButton")
	b.Name = "Btn_" .. key
	b.AutoButtonColor = false; b.Selectable = false; b.Active = true
	b.AnchorPoint = Vector2.new(1, 0); b.BackgroundColor3 = CFG.Colors.Dark; b.BackgroundTransparency = 0.2
	b.Text = ""; b.Parent = gui
	corner(b, UDim.new(1, 0))
	local st = stroke(b, color, 3, 0.05)
	local ring = Instance.new("Frame")
	ring.AnchorPoint = Vector2.new(0.5, 0.5); ring.Position = UDim2.fromScale(0.5, 0.5)
	ring.Size = UDim2.fromScale(0.82, 0.82); ring.BackgroundTransparency = 1; ring.Parent = b
	corner(ring, UDim.new(1, 0)); stroke(ring, color, 1, 0.6)
	local l = label(b, Enum.Font.GothamBlack, Color3.new(1, 1, 1))
	l.AnchorPoint = Vector2.new(0.5, 0.5); l.Position = UDim2.fromScale(0.5, 0.48); l.Size = UDim2.fromScale(0.76, 0.3)
	local hint = label(b, Enum.Font.GothamMedium, color)
	hint.AnchorPoint = Vector2.new(0.5, 0); hint.Position = UDim2.fromScale(0.5, 0.72); hint.Size = UDim2.fromScale(0.4, 0.14)
	local sc = Instance.new("UIScale"); sc.Parent = b
	local function press(down) TweenService:Create(sc, TweenInfo.new(0.08, Enum.EasingStyle.Quad), { Scale = down and 0.9 or 1 }):Play() end
	b.InputBegan:Connect(function(i) if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then press(true) end end)
	b.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then press(false) end end)
	local rec = { frame = b, label = l, hint = hint, stroke = st, enabled = true }
	UI.buttons[key] = rec
	return rec
end
local function holdButton(rec, fn) -- segurar (rastreia o toque exato)
	local active
	rec.frame.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then active = i fn(true) end
	end)
	UserInputService.InputEnded:Connect(function(i) if active and i == active then active = nil fn(false) end end)
end
local function bar(parent, color)
	local f = Instance.new("Frame")
	f.BackgroundColor3 = CFG.Colors.Dark; f.BackgroundTransparency = 0.3; f.Parent = parent
	corner(f, UDim.new(1, 0)); stroke(f, color, 2, 0.35)
	local fill = Instance.new("Frame")
	fill.Size = UDim2.fromScale(1, 1); fill.BackgroundColor3 = color; fill.BorderSizePixel = 0; fill.Parent = f
	corner(fill, UDim.new(1, 0))
	return f, fill
end

local function buildUI()
	if UI.built then return end
	UI.built = true
	local gui = Instance.new("ScreenGui")
	gui.Name = "MechaUI"; gui.ResetOnSpawn = false; gui.IgnoreGuiInset = true
	gui.DisplayOrder = 50; gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	pcall(function() gui.ScreenInsets = Enum.ScreenInsets.DeviceSafeInsets end)
	gui.Parent = player:WaitForChild("PlayerGui")

	local bA = makeButton(gui, "Action", CFG.Colors.Accent)
	local bF = makeButton(gui, "Fire", CFG.Colors.Danger)
	local bB = makeButton(gui, "Boost", CFG.Colors.Green)
	bF.label.Text = "FOGO"; bB.label.Text = "TURBO"
	bA.hint.Text = IS_MOBILE and "" or "E"; bF.hint.Text = IS_MOBILE and "" or "F"; bB.hint.Text = IS_MOBILE and "" or "SHIFT"
	bA.frame.Activated:Connect(function() Actions.interact() end)
	holdButton(bF, function(on) Actions.fire(on) end)
	holdButton(bB, function(on) Actions.boost(on) end)

	local panel = Instance.new("Frame")
	panel.AnchorPoint = Vector2.new(0.5, 0); panel.BackgroundColor3 = CFG.Colors.Dark
	panel.BackgroundTransparency = 0.15; panel.Visible = false; panel.Parent = gui
	corner(panel, UDim.new(0, 8)); stroke(panel, CFG.Colors.Accent, 2, 0.15)
	local title = label(panel, Enum.Font.GothamBlack, CFG.Colors.Accent, Enum.TextXAlignment.Left)
	title.Position = UDim2.fromScale(0.04, 0.07); title.Size = UDim2.fromScale(0.74, 0.4)
	local pct = label(panel, Enum.Font.GothamBold, Color3.new(1, 1, 1), Enum.TextXAlignment.Right)
	pct.Position = UDim2.fromScale(0.78, 0.07); pct.Size = UDim2.fromScale(0.18, 0.4)
	local track = Instance.new("Frame")
	track.Position = UDim2.fromScale(0.04, 0.6); track.Size = UDim2.fromScale(0.92, 0.22)
	track.BackgroundColor3 = Color3.fromRGB(40, 32, 26); track.BorderSizePixel = 0; track.ClipsDescendants = true; track.Parent = panel
	corner(track, UDim.new(1, 0))
	local fill = Instance.new("Frame")
	fill.Size = UDim2.fromScale(0, 1); fill.BackgroundColor3 = CFG.Colors.Accent; fill.BorderSizePixel = 0; fill.Parent = track
	corner(fill, UDim.new(1, 0))
	local grad = Instance.new("UIGradient")
	grad.Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, Color3.fromRGB(190, 90, 20)), ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 220, 120)),
		ColorSequenceKeypoint.new(1, Color3.fromRGB(190, 90, 20)),
	})
	grad.Parent = fill
	UI.panel, UI.title, UI.pct, UI.fill, UI.grad = panel, title, pct, fill, grad

	local hb, hf = bar(gui, CFG.Colors.Danger)
	local eb, ef = bar(gui, CFG.Colors.Green)
	hb.AnchorPoint = Vector2.new(0.5, 0); eb.AnchorPoint = Vector2.new(0.5, 0)
	UI.heat, UI.heatFill, UI.energy, UI.energyFill = hb, hf, eb, ef

	local ch = Instance.new("Frame")
	ch.AnchorPoint = Vector2.new(0.5, 0.5); ch.Position = UDim2.fromScale(0.5, 0.5)
	ch.Size = UDim2.fromOffset(34, 34); ch.BackgroundTransparency = 1; ch.Visible = false; ch.Parent = gui
	corner(ch, UDim.new(1, 0))
	UI.crossStroke = stroke(ch, CFG.Colors.Accent, 2, 0.1)
	local dot = Instance.new("Frame")
	dot.AnchorPoint = Vector2.new(0.5, 0.5); dot.Position = UDim2.fromScale(0.5, 0.5); dot.Size = UDim2.fromOffset(4, 4)
	dot.BackgroundColor3 = CFG.Colors.Accent; dot.BorderSizePixel = 0; dot.Parent = ch
	corner(dot, UDim.new(1, 0))
	UI.cross = ch

	local tip = label(gui, Enum.Font.GothamMedium, Color3.fromRGB(240, 232, 215))
	tip.AnchorPoint = Vector2.new(0.5, 1); tip.TextScaled = false; tip.TextStrokeTransparency = 0.5
	UI.tip = tip
end

local function layoutUI()
	if not UI.built then return end
	local cam = Workspace.CurrentCamera
	local vp = cam and cam.ViewportSize or Vector2.new(800, 400)
	local size = math.clamp(math.min(vp.X, vp.Y) * 0.17, 58, 96)
	local margin, gap, top = size * 0.3, size * 0.2, vp.Y * 0.2
	local B, fs = UI.buttons, size * 1.15
	B.Action.frame.Size = UDim2.fromOffset(size, size)
	B.Fire.frame.Size = UDim2.fromOffset(fs, fs)
	B.Boost.frame.Size = UDim2.fromOffset(size, size)
	B.Action.frame.Position = UDim2.new(1, -margin, 0, top)
	B.Fire.frame.Position = UDim2.new(1, -margin, 0, top + size + gap)
	B.Boost.frame.Position = UDim2.new(1, -(margin + fs + gap), 0, top + size * 0.9)
	local pw = math.clamp(vp.X * 0.62, 230, 460)
	UI.panel.Size = UDim2.fromOffset(pw, math.clamp(vp.Y * 0.13, 46, 64)); UI.panel.Position = UDim2.new(0.5, 0, 0, 40)
	UI.heat.Size = UDim2.fromOffset(pw * 0.7, 8); UI.heat.Position = UDim2.new(0.5, 0, 0, 12)
	UI.energy.Size = UDim2.fromOffset(pw * 0.7, 8); UI.energy.Position = UDim2.new(0.5, 0, 0, 26)
	UI.tip.Size = UDim2.fromOffset(vp.X * 0.8, 22); UI.tip.TextSize = math.clamp(vp.Y * 0.035, 12, 18)
	UI.tip.Position = UDim2.new(0.5, 0, 1, -8)
end

local lastInput = UserInputService:GetLastInputType()
UserInputService.LastInputTypeChanged:Connect(function(t) lastInput = t end)
local function wantButtons()
	if CFG.ShowButtonsOnDesktop or lastInput == Enum.UserInputType.Touch then return true end
	return UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled and not UserInputService.GamepadEnabled
end
local function setButton(rec, show, enabled, text)
	rec.frame.Visible = show
	if text then rec.label.Text = text end
	if rec.enabled ~= enabled then
		rec.enabled = enabled
		rec.frame.BackgroundTransparency = enabled and 0.2 or 0.65
		rec.label.TextTransparency = enabled and 0 or 0.55
		rec.stroke.Transparency = enabled and 0.05 or 0.6
	end
end

local function seqText(seq, tt)
	if seq.kind == "Board" then
		if tt < seq.walk0 then return "ABRINDO PORTAS DA CABINE..." end
		if tt < seq.walkEnd then return "ACESSANDO CABINE..." end
		if tt < seq.power0 then return "SELANDO CABINE..." end
		return "ACENDENDO CALDEIRAS E NÚCLEO..."
	end
	if tt < seq.stand0 then return "DESLIGANDO NÚCLEO..." end
	if tt < seq.walkEnd then return "SAINDO DA CABINE..." end
	return "TRAVANDO ESCOTILHAS..."
end

local function updateUI(t)
	if not UI.built then return end
	local st, seq = S.state, S.seq
	local touch = wantButtons()
	local B = UI.buttons
	local busy = seq ~= nil
	local pilot = st == "Pilot"
	local climbTop = st == "Climb" and S.climb and S.climb.s >= 0.97 and not S.climb.mount
	local text, show, enabled = "ESCALAR", false, true
	if st == "Free" then show = S.nearLadder and S.ready
	elseif st == "Climb" then show = true; text = climbTop and "ENTRAR" or "SOLTAR"; enabled = not (S.climb and S.climb.mount)
	elseif pilot then show = true; text = "SAIR" end
	setButton(B.Action, touch and show, enabled and not busy, text)
	setButton(B.Fire, touch and pilot, not S.overheated, S.overheated and "CALOR" or "FOGO")
	setButton(B.Boost, touch and pilot, S.energy >= CFG.Boost.MinToStart * 0.5, "TURBO")

	UI.heat.Visible, UI.energy.Visible, UI.cross.Visible = pilot, pilot, pilot
	if pilot then
		UI.heatFill.Size = UDim2.fromScale(clamp01(S.heat / CFG.Weapon.LockAt), 1)
		UI.heatFill.BackgroundColor3 = S.overheated and CFG.Colors.Danger or CFG.Colors.Accent
		UI.energyFill.Size = UDim2.fromScale(clamp01(S.energy / CFG.Boost.Max), 1)
		UI.crossStroke.Color = S.overheated and CFG.Colors.Danger or CFG.Colors.Accent
	end
	UI.panel.Visible = busy
	if busy then
		local tt = t - seq.t0
		local u = clamp01(tt / seq.total)
		UI.fill.Size = UDim2.fromScale(u, 1)
		UI.pct.Text = string.format("%d%%", math.floor(u * 100))
		UI.title.Text = seqText(seq, tt)
		UI.grad.Offset = Vector2.new(((t * 0.9) % 2) - 1, 0)
	end
	if st == "Climb" then
		UI.tip.Text = IS_MOBILE and "ANALÓGICO CIMA/BAIXO: SOBE/DESCE  •  PULO: SOLTAR" or "W/S: sobe/desce  •  ESPAÇO: soltar  •  E: entrar (no topo)"
		UI.tip.Visible = true
	elseif st == "Free" and not touch then
		UI.tip.Text = "Aproxime-se da escada do mecha  •  E: escalar"
		UI.tip.Visible = true
	elseif pilot and not touch then
		UI.tip.Text = "E: sair  •  F: fogo  •  SHIFT: turbo"
		UI.tip.Visible = true
	else
		UI.tip.Visible = false
	end
end

--------------------------------------------------------------------------------
-- SONS
--------------------------------------------------------------------------------
local function playSound(key, pv)
	local fx = S.fx
	local snd = fx and fx.sounds[key]
	if not snd then return end
	snd.PlaybackSpeed = 1 + (math.random() - 0.5) * (pv or 0)
	snd.TimePosition = 0
	snd:Play()
end

--------------------------------------------------------------------------------
-- OPERADOR MILITAR TECNOLÓGICO (6 PARTES + detalhes)
--------------------------------------------------------------------------------
local PLASTIC, METAL, NEON = Enum.Material.SmoothPlastic, Enum.Material.Metal, Enum.Material.Neon
local SG_DARK, SG_GRAY, SG_OLIVE = Color3.fromRGB(30, 34, 38), Color3.fromRGB(66, 72, 78), Color3.fromRGB(64, 74, 56)
local SG_BLACK = Color3.fromRGB(16, 18, 20)

local function buildSoldier(char, root)
	local model = Instance.new("Model")
	model.Name = "Operador"
	model.Parent = char
	local sn, list = {}, {}
	local base = root.CFrame * CF(0, 0.5, 0)
	local function mk(name, size, rest, color, mat)
		local p = Instance.new("Part")
		p.Name = name; p.Size = size; p.Color = color; p.Material = mat or PLASTIC
		p.CanCollide = false; p.CanTouch = false; p.CanQuery = false; p.Massless = true; p.Anchored = false
		p.TopSurface = Enum.SurfaceType.Smooth; p.BottomSurface = Enum.SurfaceType.Smooth
		p.CastShadow = size.Magnitude > 2
		p.CFrame = base * rest
		p.Parent = model
		return p
	end
	local function node(name, parent, size, rest, joint, color, mat)
		local p = mk(name, size, rest, color, mat)
		local n = { name = name, part = p, rest = rest, tx = 0, ty = 0, tz = 0 }
		local m = Instance.new("Motor6D")
		if parent then
			m.Name = name .. "Joint"; m.Part0 = sn[parent].part; m.Part1 = p
			m.C0 = sn[parent].rest:Inverse() * CF(joint); m.C1 = rest:Inverse() * CF(joint)
		else
			m.Name = "SoldierRoot"; m.Part0 = root; m.Part1 = p; m.C0 = CF(0, 0.5, 0); m.C1 = CFrame.new()
		end
		m.Parent = p
		n.motor, n.c0 = m, m.C0
		sn[name] = n; list[#list + 1] = n
		return n
	end
	local function det(nodeName, size, off, color, mat)
		local n = sn[nodeName]
		local p = mk(nodeName .. "_d", size, n.rest * off, color, mat)
		local w = Instance.new("WeldConstraint"); w.Part0, w.Part1, w.Parent = n.part, p, p
		return p
	end

	node("Torso", nil, V3(2, 2.2, 1.1), CFrame.new(), nil, SG_GRAY)
	det("Torso", V3(1.7, 1.5, 0.3), CF(0, 0.25, -0.7), SG_DARK, METAL)               -- placa peitoral
	det("Torso", V3(0.9, 0.1, 0.05), CF(0, 0.55, -0.86), CFG.Colors.Cyan, NEON)      -- faixa tech
	det("Torso", V3(0.5, 0.1, 0.05), CF(0, 0.3, -0.86), CFG.Colors.Cyan, NEON)
	for i = 0, 2 do det("Torso", V3(0.45, 0.55, 0.3), CF(-0.65 + i * 0.65, -0.6, -0.72), SG_OLIVE) end -- bolsas de munição
	det("Torso", V3(2.15, 0.35, 1.2), CF(0, -0.95, 0), SG_BLACK, METAL)              -- cinto
	det("Torso", V3(1.5, 1.7, 0.8), CF(0, 0.2, 0.95), SG_DARK, METAL)                -- mochila de energia
	det("Torso", V3(1.0, 0.2, 0.1), CF(0, 0.9, 1.36), CFG.Colors.Cyan, NEON)
	det("Torso", V3(0.08, 1.2, 0.08), CF(0.55, 1.0, 1.0), SG_BLACK)                   -- antena
	for _, sx in ipairs({ -1, 1 }) do det("Torso", V3(0.9, 0.5, 1.0), CF(sx * 1.4, 1.15, 0), SG_DARK, METAL) end -- ombreiras

	node("Head", "Torso", V3(1.1, 1.1, 1.1), CF(0, 1.75, 0), V3(0, 1.15, 0), SG_GRAY)
	det("Head", V3(1.3, 0.7, 1.3), CF(0, 0.35, 0), SG_DARK, METAL)                   -- capacete
	det("Head", V3(1.0, 0.25, 0.06), CF(0, 0.05, -0.58), CFG.Colors.Cyan, NEON)      -- visor
	det("Head", V3(0.9, 0.3, 0.5), CF(0, -0.5, -0.4), SG_BLACK)
	det("Head", V3(0.2, 0.2, 1.2), CF(0, 0.75, 0), SG_GRAY)
	for _, sx in ipairs({ -1, 1 }) do det("Head", V3(0.15, 0.4, 0.4), CF(sx * 0.62, 0, 0.05), SG_BLACK) end

	for _, sx in ipairs({ -1, 1 }) do
		local s = sx < 0 and "L" or "R"
		node(s .. "Arm", "Torso", V3(0.8, 2.0, 0.8), CF(sx * 1.4, 0, 0), V3(sx * 1.4, 1.0, 0), SG_DARK)
		det(s .. "Arm", V3(0.95, 0.9, 0.95), CF(0, -0.65, 0), SG_GRAY, METAL)        -- luva blindada
		det(s .. "Arm", V3(0.1, 0.6, 0.05), CF(sx * 0.48, -0.65, 0), CFG.Colors.Cyan, NEON)
		det(s .. "Arm", V3(0.7, 0.5, 0.7), CF(0, -1.1, 0), SG_BLACK)
		if sx < 0 then det(s .. "Arm", V3(0.55, 0.12, 0.4), CF(0, -0.55, -0.5), CFG.Colors.Cyan, NEON) end -- holograma no pulso
		node(s .. "Leg", "Torso", V3(0.95, 2.4, 1.0), CF(sx * 0.5, -2.3, 0), V3(sx * 0.5, -1.1, 0), SG_DARK)
		det(s .. "Leg", V3(1.0, 0.9, 1.1), CF(0, 0.7, 0), SG_GRAY, METAL)
		det(s .. "Leg", V3(0.9, 0.5, 0.4), CF(0, -0.35, -0.55), SG_GRAY, METAL)      -- joelheira
		det(s .. "Leg", V3(1.1, 0.9, 1.6), CF(0, -1.0, -0.2), SG_BLACK)              -- bota
		det(s .. "Leg", V3(0.5, 0.08, 0.05), CF(0, -1.2, 0.62), CFG.Colors.Cyan, NEON)
	end
	return { model = model, nodes = sn, list = list }
end

-- anima o operador: mistura 3 poses (pé, escalada, sentado)
local PF, PC, PS = {}, {}, {}
local function soldierAnimate(dt, t)
	local so = S.sold
	if not so then return end
	local spd, ph, run, air = FM.spd, FM.phase, FM.run, FM.air
	local sq = clamp01(FM.spring)

	-- POSE A PÉ
	local sL, cL = math.sin(ph), math.cos(ph)
	local amp = spd * (0.75 + 0.45 * run)
	local idleB = math.sin(t * 1.5) * 0.02
	local look = M.headYaw * 0
	PF[1], PF[2], PF[3] = 0.04, look, 0                                              -- cabeça
	PF[4], PF[5], PF[6] = -sL * amp * 0.95 + air * 2.2 + idleB, 0, -0.06             -- braço E
	PF[7], PF[8], PF[9] = sL * amp * 0.95 + air * 2.2 - idleB, 0, 0.06               -- braço D
	PF[10], PF[11], PF[12] = sL * amp * 1.0 + air * 0.55 + sq * 0.6, 0, 0            -- perna E
	PF[13], PF[14], PF[15] = -sL * amp * 1.0 + air * 0.2 + sq * 0.6, 0, 0            -- perna D
	PF[16] = -math.abs(sL) * 0.18 * amp - sq * 0.55                                  -- bob Y
	PF[17] = -(0.05 + 0.2 * run) * spd                                               -- inclinação
	PF[18] = sL * 0.05 * amp                                                         -- rolagem

	-- POSE ESCALADA
	local a = math.sin(FM.climbPh)
	PC[1], PC[2], PC[3] = 0.3, 0, 0
	PC[4], PC[5], PC[6] = 2.55 + a * 0.5, 0, -0.22
	PC[7], PC[8], PC[9] = 2.55 - a * 0.5, 0, 0.22
	PC[10], PC[11], PC[12] = 0.25 + 0.95 * math.max(0, -a), 0, 0
	PC[13], PC[14], PC[15] = 0.25 + 0.95 * math.max(0, a), 0, 0
	PC[16], PC[17], PC[18] = 0, 0.05, a * 0.05

	-- POSE SENTADO (pilotando)
	local busy = math.sin(t * 1.7) * 0.1 + M.spd * math.sin(t * 6) * 0.08
	PS[1], PS[2], PS[3] = M.camPitch * 0.3, S.state == "Pilot" and M.headYaw * 0.5 or 0, 0
	PS[4], PS[5], PS[6] = 1.0 + busy, 0, -0.1
	PS[7], PS[8], PS[9] = 1.0 - busy * 0.8, 0, 0.1
	PS[10], PS[11], PS[12] = 1.5, 0, -0.04
	PS[13], PS[14], PS[15] = 1.5, 0, 0.04
	PS[16], PS[17], PS[18] = 0, 0, 0

	local wc = (S.state == "Climb") and 1 or 0
	local ws = S.seatW
	local wf = 1 - wc - ws
	if wf < 0 then wf = 0 end
	local sum = wf + wc + ws
	local N = so.nodes
	local order = { N.Head, N.LArm, N.RArm, N.LLeg, N.RLeg }
	for j = 1, 5 do
		local o = (j - 1) * 3
		local x = (PF[o + 1] * wf + PC[o + 1] * wc + PS[o + 1] * ws) / sum
		local y = (PF[o + 2] * wf + PC[o + 2] * wc + PS[o + 2] * ws) / sum
		local z = (PF[o + 3] * wf + PC[o + 3] * wc + PS[o + 3] * ws) / sum
		order[j].motor.C0 = order[j].c0 * CFrame.fromEulerAnglesYXZ(x, y, z)
	end
	local by = (PF[16] * wf + PC[16] * wc + PS[16] * ws) / sum
	local pitch = (PF[17] * wf + PC[17] * wc + PS[17] * ws) / sum
	local roll = (PF[18] * wf + PC[18] * wc + PS[18] * ws) / sum
	N.Torso.motor.C0 = S.soldBase * CF(0, by, 0) * AN(pitch, 0, roll)
end

-- movimento do operador (a pé / durante sequências)
local function updateFootMotion(dt)
	local root, hum = S.root, S.hum
	local hs
	if FM.override then
		hs = FM.override
		FM.grounded = true
	else
		local vel = root.AssemblyLinearVelocity
		hs = math.sqrt(vel.X * vel.X + vel.Z * vel.Z)
		FM.grounded = hum.FloorMaterial ~= Enum.Material.Air
		if FM.grounded then
			if not FM.wasGrounded and FM.airTime > 0.25 then FM.springV = clamp01(-FM.lastVy / 60) * 5; playSound("Land", 0.1) end
			FM.airTime, FM.lastVy = 0, 0
		else
			FM.airTime += dt
			if vel.Y < FM.lastVy then FM.lastVy = vel.Y end
		end
		FM.wasGrounded = FM.grounded
	end
	FM.air += ((FM.grounded and 0 or 1) - FM.air) * damp(8, dt)
	FM.springV += (-FM.spring * 140 - FM.springV * 12) * dt
	FM.spring += FM.springV * dt
	FM.spd += (clamp01(hs / 14) * (hs > 0.4 and 1 or 0) - FM.spd) * damp(9, dt)
	FM.run += (clamp01((hs - 14) / 10) - FM.run) * damp(6, dt)
	if FM.grounded and hs > 0.4 then FM.phase += hs * dt * 1.0 end
end

--------------------------------------------------------------------------------
-- MECHA ANCIÃO (procedural, steampunk-cyberpunk)
-- Espaço do modelo: origem no CHÃO sob o centro, Y para cima, FRENTE = -Z
--------------------------------------------------------------------------------
local BRONZE, IRON, RUST = Color3.fromRGB(150, 100, 54), Color3.fromRGB(46, 44, 44), Color3.fromRGB(118, 60, 34)
local BRASS, STONE, BLACK = Color3.fromRGB(192, 148, 66), Color3.fromRGB(88, 84, 80), Color3.fromRGB(20, 20, 22)
local GLOW, EYE, GLASS = Color3.fromRGB(255, 110, 30), Color3.fromRGB(255, 50, 30), Color3.fromRGB(255, 180, 110)
local DIAMOND, GLASSM = Enum.Material.DiamondPlate, Enum.Material.Glass
local cylY, cylZ = AN(0, 0, HALF_PI), AN(0, HALF_PI, 0)

local dynFolder, cableFolder, tracerFolder
local pistons, cables = {}, {}
local rayParams = RaycastParams.new()
rayParams.FilterType = Enum.RaycastFilterType.Exclude
rayParams.IgnoreWater = true

local function buildMecha(O)
	local model = Instance.new("Model")
	model.Name = "MechaKronos"
	model.Parent = Workspace
	local N, list = {}, {}
	local fx = { sounds = {}, visors = {}, lamps = {}, stacks = {}, vents = {}, glowParts = {} }
	local count = 0

	local function tint(c, name)
		local k = 1 + (hash01(name) - 0.5) * 0.22 -- desgaste/ferrugem
		return Color3.new(math.clamp(c.R * k, 0, 1), math.clamp(c.G * k, 0, 1), math.clamp(c.B * k, 0, 1))
	end
	local function mk(name, shape, size, cf, color, mat)
		local p = Instance.new("Part")
		p.Name = name; p.Size = size; p.Anchored = true
		p.CanCollide = false; p.CanTouch = false; p.Massless = true
		p.TopSurface = Enum.SurfaceType.Smooth; p.BottomSurface = Enum.SurfaceType.Smooth
		p.Color = (mat == NEON or mat == GLASSM) and color or tint(color, name)
		p.Material = mat or METAL
		p.CastShadow = math.max(size.X, size.Y, size.Z) > 5
		if shape == "Cyl" then p.Shape = Enum.PartType.Cylinder elseif shape == "Ball" then p.Shape = Enum.PartType.Ball end
		p.CFrame = O * cf
		p.Parent = model
		return p
	end
	local function node(name, parentName, shape, size, rest, joint, color, mat, solid)
		local p = mk(name, shape, size, rest, color, mat)
		local n = { name = name, part = p, rest = rest, solid = solid, tx = 0, ty = 0, tz = 0, ax = 0, ay = 0, az = 0 }
		if parentName then
			local par = N[parentName]
			local m = Instance.new("Motor6D")
			m.Name = name .. "Joint"; m.Part0 = par.part; m.Part1 = p
			m.C0 = par.rest:Inverse() * CF(joint); m.C1 = rest:Inverse() * CF(joint)
			m.Parent = p
			n.motor, n.c0 = m, m.C0
		end
		N[name] = n; list[#list + 1] = n
		return n
	end
	local function det(nodeName, shape, size, off, color, mat)
		local n = N[nodeName]
		count += 1
		local p = mk(nodeName .. "_" .. count, shape, size, n.rest * off, color, mat)
		local w = Instance.new("WeldConstraint"); w.Part0, w.Part1, w.Parent = n.part, p, p
		return p
	end
	local function att(nodeName, name, pos)
		local a = Instance.new("Attachment"); a.Name = name; a.Position = pos; a.Parent = N[nodeName].part
		return a
	end
	-- cilindro com eixo x/y/z: length = comprimento, d = diâmetro
	local function cyl(nodeName, length, d, off, axis, color, mat)
		local rot = axis == "y" and cylY or (axis == "z" and cylZ or CFrame.new())
		return det(nodeName, "Cyl", V3(length, d, d), off * rot, color, mat)
	end
	-- engrenagem giratória (nó próprio, com dentes e raios)
	local function gear(parentName, name, center, axis, R, thick, rate)
		local rot = axis == "y" and cylY or (axis == "z" and cylZ or CFrame.new())
		local g = node(name, parentName, "Cyl", V3(thick, R * 2, R * 2), CF(center) * rot, center, BRASS, METAL)
		g.spinRate, g.spinAxis, g.ang = rate, axis, hash01(name) * PI
		local teeth = DETAIL > 1 and math.clamp(math.floor(R * 1.5), 6, 10) or 6
		for i = 0, teeth - 1 do
			det(name, "Block", V3(thick * 0.9, R * 0.26, R * 0.3), AN(i * 2 * PI / teeth, 0, 0) * CF(0, R * 1.08, 0), BRONZE)
		end
		det(name, "Cyl", V3(thick * 1.4, R * 0.5, R * 0.5), CF(), IRON)
		det(name, "Block", V3(thick * 0.6, R * 1.7, R * 0.14), CF(), IRON)
		det(name, "Block", V3(thick * 0.6, R * 0.14, R * 1.7), CF(), IRON)
		return g
	end
	local function tdet(shape, size, cf, color, mat) -- relativo ao CENTRO do tronco (0,53,0)
		return det("Torso", shape, size, CF(0, 0, -6.5) * cf, color, mat)
	end
	local function anchorAt(nodeName, off) return { n = N[nodeName], off = off } end

	-- PELVE --------------------------------------------------------------------
	node("Pelvis", nil, "Block", V3(30, 8, 20), REST_PELVIS, nil, IRON, METAL, true)
	det("Pelvis", "Block", V3(10, 7, 3), CF(0, -1, -11.5), BRONZE)
	det("Pelvis", "Block", V3(14, 6, 3), CF(0, 0, 11.5), RUST)
	det("Pelvis", "Block", V3(32, 1.5, 22), CF(0, 3.8, 0), IRON)
	for _, sx in ipairs({ -1, 1 }) do
		det("Pelvis", "Block", V3(2.5, 9, 12), CF(sx * 16, 0, 0), IRON)
		det("Pelvis", "Block", V3(7, 9, 1.5), CF(sx * 8, -6, -10.5) * AN(-0.3, 0, 0), BRONZE)
		cyl("Pelvis", 3, 5, CF(sx * 12, -3, -10), "z", STONE)
	end

	-- TRONCO (cavidade da cabine aberta na frente) -----------------------------
	node("Torso", "Pelvis", "Block", V3(38, 24, 13), CF(0, 53, 6.5), V3(0, 40.5, 0), BRONZE, METAL, true)
	tdet("Block", V3(11, 24, 13), CF(-13.5, 0, -6.5), BRONZE)           -- paredes da cavidade
	tdet("Block", V3(11, 24, 13), CF(13.5, 0, -6.5), BRONZE)
	tdet("Block", V3(16, 9, 13), CF(0, 7.5, -6.5), RUST)
	tdet("Block", V3(16, 3, 13), CF(0, -10.5, -6.5), IRON)
	tdet("Block", V3(36, 7, 3.5), CF(0, 9.5, -14) * AN(-0.35, 0, 0), BRONZE) -- cobertura do peito
	tdet("Block", V3(38, 2, 3), CF(0, 12.5, -12.5), IRON)
	for _, sx in ipairs({ -1, 1 }) do
		tdet("Block", V3(12, 22, 2.4), CF(sx * 13.5, 0, -14.2), sx < 0 and RUST or BRONZE)
		for i = 0, 3 do cyl("Torso", 1.2, 1.6, CF(sx * 13.5 + (i % 2 == 0 and -4 or 4), -8 + i * 5, -22.4), "z", BRASS) end -- rebites
		tdet("Block", V3(1.5, 20, 12), CF(sx * 19.5, 0, 2), IRON)       -- laterais
		cyl("Torso", 6, 11, CF(sx * 20.5, 9, -6.5), "x", IRON)           -- encaixe do ombro
		fx.lamps[#fx.lamps + 1] = cyl("Torso", 3, 4, CF(sx * 16.5, 10.5, -22.2), "z", Color3.fromRGB(255, 220, 160), NEON) -- faróis
	end
	-- interior da cabine
	tdet("Block", V3(16, 0.5, 13), CF(0, -9.25, -6.5), IRON)
	tdet("Block", V3(5, 2.2, 4.5), CF(0, -7.9, -7.5), BLACK)             -- poltrona
	tdet("Block", V3(5, 5, 1), CF(0, -5.5, -4.5), BLACK)
	tdet("Block", V3(7, 2.2, 3), CF(0, -7.8, -11.2), IRON)               -- console
	fx.cabin = tdet("Block", V3(14, 8, 0.3), CF(0, -4, -0.3), GLOW, NEON) -- painel quente ao fundo
	for i = -2, 2 do fx.glowParts[#fx.glowParts + 1] = tdet("Block", V3(0.9, 0.18, 1.0), CF(i * 1.3, -6.6, -11.8), i % 2 == 0 and CFG.Colors.Cyan or GLOW, NEON) end
	-- núcleo de energia (grade + esfera + anéis)
	cyl("Torso", 1.2, 11, CF(0, 7.5, -20.4), "z", BRASS)
	fx.core = tdet("Ball", V3(6.8, 6.8, 6.8), CF(0, 7.5, -14.3), GLOW, NEON)
	for i = -2, 2 do tdet("Block", V3(0.8, 9, 0.8), CF(i * 2, 7.5, -15.4), IRON) end
	-- mochila-fornalha
	tdet("Block", V3(28, 22, 12), CF(0, 0, 19), IRON)
	tdet("Block", V3(24, 18, 1.5), CF(0, 0, 25.7), RUST)
	for i = 0, 3 do tdet("Block", V3(20, 0.5, 0.5), CF(0, 6 - i * 2.4, 26.6), GLOW, NEON) end -- fendas incandescentes
	for _, sx in ipairs({ -1, 1 }) do
		cyl("Torso", 25, 5, CF(sx * 9, 19.5, 11.5), "y", STONE)            -- chaminés
		cyl("Torso", 1.4, 7, CF(sx * 9, 32.2, 11.5), "y", IRON)
		cyl("Torso", 1.2, 5.8, CF(sx * 9, 24, 11.5), "y", BRASS)
		cyl("Torso", 22, 2.2, CF(sx * 14, 0, 6.3), "y", BRASS)          -- tubulações
		fx.stacks[#fx.stacks + 1] = att("Torso", "Stack" .. sx, V3(sx * 9, 33.2, 11.5))
		fx.vents[#fx.vents + 1] = att("Torso", "Vent" .. sx, V3(sx * 14, 11.5, 6.3))
		fx.vents[#fx.vents + 1] = att("Torso", "VentB" .. sx, V3(sx * 18, -11, -6))
	end
	-- guindaste com corrente
	tdet("Block", V3(2, 12, 2), CF(-12, 18.5, 8), IRON)
	tdet("Block", V3(2, 2, 18), CF(-12, 24, 0), IRON)
	tdet("Block", V3(1, 8, 1), CF(-12, 20, -2) * AN(0.5, 0, 0), RUST)
	-- radar
	tdet("Block", V3(0.8, 6, 0.8), CF(14, 15, 8), IRON)

	-- engrenagens do tronco/quadril
	gear("Torso", "GearBackBig", V3(0, 55, 26.8), "z", 9, 1.4, 0.55)
	gear("Torso", "GearBackMid", V3(-14.5, 55, 26.8), "z", 5.5, 1.4, -0.9)
	gear("Torso", "GearBackSm", V3(13.8, 61, 26.8), "z", 5, 1.4, -0.98)
	gear("Torso", "GearSideL", V3(-20.6, 50, 14), "x", 5, 1.2, 0.8)
	gear("Torso", "GearSideR", V3(20.6, 50, 14), "x", 5, 1.2, -0.8)
	gear("Torso", "GearWinch", V3(-13.4, 72, 2), "x", 3.5, 1.2, 1.1)
	for _, sx in ipairs({ -1, 1 }) do gear("Pelvis", "GearHip" .. (sx < 0 and "L" or "R"), V3(sx * 18.4, 37, 0), "x", 4.5, 1.4, 0.9 * sx) end

	-- anéis do reator (giram sobre o núcleo)
	local r1 = node("Ring1", "Torso", "Cyl", V3(0.5, 10, 10), CF(0, 60.5, -15.6) * cylZ, V3(0, 60.5, -15.6), GLOW, NEON)
	r1.spinRate, r1.spinAxis, r1.ang = 1.6, "z", 0
	local r2 = node("Ring2", "Torso", "Cyl", V3(0.4, 8.2, 8.2), CF(0, 60.5, -16.3) * cylZ, V3(0, 60.5, -16.3), BRASS, METAL)
	r2.spinRate, r2.spinAxis, r2.ang = -2.4, "z", 0
	det("Ring2", "Block", V3(0.5, 1.2, 1.2), CF(0, 4.1, 0), RUST)
	-- radar giratório
	local dish = node("Dish", "Torso", "Cyl", V3(0.8, 9, 9), CF(14, 69.5, 8) * AN(0, 0, HALF_PI) * AN(0, 0, -0.5), V3(14, 69.5, 8), BRONZE, METAL)
	dish.spinRate, dish.spinAxis, dish.ang = 0.9, "y", 0
	det("Dish", "Block", V3(0.5, 0.5, 6), CF(0, 1.6, 0), IRON)

	-- PORTAS DE VIDRO (articuladas) -------------------------------------------
	for _, sx in ipairs({ -1, 1 }) do
		local nm = sx < 0 and "DoorL" or "DoorR"
		node(nm, "Torso", "Block", V3(0.8, 12, 0.8), CF(sx * 7.6, 50, -13.4), V3(sx * 7.6, 50, -13.4), BRONZE, METAL)
		det(nm, "Block", V3(8, 0.8, 0.8), CF(-sx * 4, 5.6, 0), BRONZE)
		det(nm, "Block", V3(8, 0.8, 0.8), CF(-sx * 4, -5.6, 0), BRONZE)
		det(nm, "Block", V3(8, 0.5, 0.6), CF(-sx * 4, 0.4, 0), IRON)
		det(nm, "Block", V3(0.8, 12, 0.8), CF(-sx * 8, 0, 0), BRONZE)
		det(nm, "Block", V3(7.2, 10.4, 0.2), CF(-sx * 4, 0, 0), GLASS, GLASSM).Transparency = 0.55
	end
	-- sacada
	local bal = { { 20, 1, 12, 0, -9.5, -19 } }
	tdet("Block", V3(20, 1, 12), CF(0, -9.5, -19), IRON, DIAMOND)
	for _, px in ipairs({ -10, -4, 10 }) do for _, pz in ipairs({ -13.3, -24.7 }) do
		if not (px == -4 and pz == -24.7) then tdet("Block", V3(0.6, 3.6, 0.6), CF(px, -7.2, pz), BRASS) end
	end end
	tdet("Block", V3(0.5, 0.5, 11.6), CF(10, -5.5, -19), BRASS)
	tdet("Block", V3(0.5, 0.5, 11.6), CF(-10, -5.5, -19), BRASS)
	tdet("Block", V3(6, 0.5, 0.5), CF(7, -5.5, -24.7), BRASS)
	tdet("Block", V3(12, 0.5, 0.5), CF(-2, -5.5, -24.7)  , BRASS)
	tdet("Block", V3(1, 9, 1), CF(-9, -14, -19) * AN(0.0, 0, 0.0), IRON)
	tdet("Block", V3(1, 10, 1), CF(9, -14.5, -19), IRON)

	-- CABEÇA (cúpula + olho) ----------------------------------------------------
	node("Head", "Torso", "Ball", V3(15, 15, 15), CF(0, 67, -3), V3(0, 64.5, -3), BRONZE, METAL)
	det("Head", "Block", V3(14, 2, 3), CF(0, 4.2, -6.5) * AN(0.35, 0, 0), IRON)
	cyl("Head", 1.2, 9, CF(0, -0.5, -7.1), "z", BRASS)
	fx.eye = det("Head", "Ball", V3(7, 7, 7), CF(0, -0.5, -6.5), EYE, NEON)
	for _, sx in ipairs({ -1, 1 }) do
		cyl("Head", 2, 6, CF(sx * 8, 0, 0), "x", IRON)
		det("Head", "Block", V3(0.4, 10, 0.4), CF(sx * 4, 11, 3), BLACK)
		fx.visors[#fx.visors + 1] = det("Head", "Block", V3(0.9, 0.9, 0.9), CF(sx * 4, 16.2, 3), EYE, NEON)
	end
	det("Head", "Block", V3(5, 3, 6), CF(0, 7.4, 1), RUST)
	node("Iris", "Head", "Block", V3(0.9, 5, 0.6), CF(0, 66.5, -12.9), V3(0, 66.5, -12.9), BLACK, METAL)
	fx.eyeAtt = att("Head", "EyeLight", V3(0, -0.5, -9))
	fx.headAtt = att("Head", "HeadTop", V3(0, 8, 0))

	-- BRAÇOS (esquerdo = -X) ----------------------------------------------------
	for _, sx in ipairs({ -1, 1 }) do
		local s = sx < 0 and "L" or "R"
		local u = "UpperArm" .. s
		node(u, "Torso", "Block", V3(10, 16, 11), CF(sx * 25, 54, 0), V3(sx * 25, 62, 0), BRONZE, METAL)
		det(u, "Ball", V3(17, 17, 17), CF(sx * 0.5, 8, 0), BRONZE)                 -- ombreira
		det(u, "Block", V3(2, 7, 2), CF(sx * 3, 17, 3) * AN(0, 0, -sx * 0.25), IRON) -- espinhos
		det(u, "Block", V3(2, 9, 2), CF(sx * 0, 18.5, -1), IRON)
		det(u, "Block", V3(2, 7, 2), CF(sx * -3, 17, -4) * AN(0, 0, sx * 0.25), IRON)
		det(u, "Block", V3(1.5, 12, 9), CF(sx * 5.8, 0, 0), RUST)
		cyl(u, 12, 7, CF(0, -8.6, 0), "x", IRON)                                     -- cotovelo
		gear("UpperArm" .. s, "GearElbow" .. s, V3(sx * 31.6, 46, 0), "x", 3.5, 1.2, 1.0 * sx)
		node("Forearm" .. s, u, "Block", V3(11, s == "R" and 14 or 16, 12), CF(sx * 25, s == "R" and 39 or 38, -1), V3(sx * 25, 46, 0), RUST, METAL)
		local f = "Forearm" .. s
		det(f, "Block", V3(9, 10, 2), CF(0, 0, -7), BRONZE)
		det(f, "Block", V3(1.5, 12, 10), CF(sx * 6, 0, 0), IRON)
		det(f, "Block", V3(12, 2, 13), CF(0, s == "R" and -7 or -8, 0), IRON)
	end
	-- garra esquerda
	node("HandL", "ForearmL", "Block", V3(10, 9, 10), CF(-25, 25.5, -1), V3(-25, 30, 0), IRON, METAL)
	for i = -1, 1 do det("HandL", "Block", V3(3, 9, 3.2), CF(i * 3.2, -6, -2.5) * AN(-0.35, 0, 0), BRONZE) end
	det("HandL", "Block", V3(3, 7, 3), CF(-5.5, -2, 3) * AN(0.2, 0, 0.5), BRONZE)
	-- canhão direito
	det("ForearmR", "Block", V3(9, 5, 9), CF(0, -10, 0), IRON)
	cyl("ForearmR", 12, 3.8, CF(0, -17, 0), "y", STONE)
	cyl("ForearmR", 1.4, 6, CF(0, -12.5, 0), "y", BRASS)
	cyl("ForearmR", 2.4, 5.4, CF(0, -23.6, 0), "y", BLACK)
	cyl("ForearmR", 5, 6.5, CF(6.5, -2, 2), "z", IRON)                                -- tambor de munição
	fx.muzzle = att("ForearmR", "Muzzle", V3(0, -25, 0))

	-- PERNAS -------------------------------------------------------------------
	for _, sx in ipairs({ -1, 1 }) do
		local s = sx < 0 and "L" or "R"
		local th, sh, ft = "Thigh" .. s, "Shin" .. s, "Foot" .. s
		node(th, "Pelvis", "Block", V3(14, 15, 15), CF(sx * 10, 25.5, 0), V3(sx * 10, 33, 0), RUST, METAL, true)
		det(th, "Block", V3(15, 3, 16), CF(0, 8.5, 0), BRONZE)
		det(th, "Block", V3(12, 12, 2.5), CF(0, 0, -8.5), BRONZE)
		det(th, "Block", V3(2.5, 13, 13), CF(sx * 8, 0, 0), IRON)
		det(th, "Block", V3(12, 5, 5), CF(0, -8, -5), IRON)
		cyl(th, 15, 2.6, CF(-3.5, 0, 8.6), "y", STONE); cyl(th, 15, 2.6, CF(3.5, 0, 8.6), "y", STONE)
		node(sh, th, "Block", V3(12, 14, 13), CF(sx * 10, 11.5, 0), V3(sx * 10, 18, 0), BRONZE, METAL, true)
		det(sh, "Block", V3(11, 10, 2), CF(0, 0.5, -7.2) * AN(0.15, 0, 0), BRONZE)
		det(sh, "Block", V3(10, 10, 4), CF(0, 1, 8), RUST)
		det(sh, "Block", V3(2, 10, 10), CF(sx * 7, 0, 0), IRON)
		for i = -1, 1 do det(sh, "Block", V3(1, 11, 1), CF(i * 3.5, 0.5, -8.4), IRON) end
		cyl(sh, 4, 2, CF(sx * 3, -5, 8.6), "y", STONE)
		fx.vents[#fx.vents + 1] = att(sh, "ShinVent" .. s, V3(sx * 6, 6, 8))
		gear(sh, "GearKnee" .. s, V3(sx * 19, 17.5, 0), "x", 4.5, 1.4, -0.85 * sx)
		node(ft, sh, "Block", V3(16, 4, 24), CF(sx * 10, 2, -3), V3(sx * 10, 4.5, 0), IRON, METAL, true)
		det(ft, "Block", V3(12, 6, 12), CF(0, 5, 3), BRONZE)
		det(ft, "Block", V3(14, 3, 6), CF(0, 0.8, -12.5) * AN(0.2, 0, 0), RUST)
		det(ft, "Block", V3(14, 5, 5), CF(0, 1.5, 10), RUST)
		det(ft, "Block", V3(16.4, 0.6, 24.4), CF(0, -1.8, 0), BLACK, DIAMOND)
		for i = -1, 1 do det(ft, "Block", V3(1.6, 1.6, 4), CF(i * 4.5, 1, -14.5), IRON) end
		fx["foot" .. s] = N[ft]
	end

	-- desancora tudo (menos a pelve); o conjunto fica fixo porque a raiz está ancorada
	local pelvis = N.Pelvis.part
	for _, d in ipairs(model:GetDescendants()) do
		if d:IsA("BasePart") and d ~= pelvis then d.Anchored = false end
	end
	model.PrimaryPart = pelvis
	local link = Instance.new("Motor6D"); link.Name = "PilotLink"; link.Part1 = pelvis; link.Parent = pelvis
	for _, n in ipairs(list) do if n.solid then n.part.CanCollide = true end end

	-- PISTÕES hidráulicos (dois cilindros telescópicos entre dois pontos dos nós)
	dynFolder = Instance.new("Folder"); dynFolder.Name = "MechaDyn"; dynFolder.Parent = Workspace
	local function dynPart(r, color)
		local p = Instance.new("Part")
		p.Shape = Enum.PartType.Cylinder; p.Size = V3(1, r * 2, r * 2)
		p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CanTouch = false; p.CastShadow = false
		p.Material = METAL; p.Color = color; p.Parent = dynFolder
		return p
	end
	local function addPiston(a, ao, b, bo, r)
		pistons[#pistons + 1] = { a = N[a], ao = ao, b = N[b], bo = bo, outer = dynPart(r, STONE), inner = dynPart(r * 0.62, BRASS) }
	end
	for _, sx in ipairs({ -1, 1 }) do
		local s = sx < 0 and "L" or "R"
		addPiston("Thigh" .. s, V3(0, 2, -8.4), "Shin" .. s, V3(0, 3, -8), 1.0)
		addPiston("Thigh" .. s, V3(sx * 3, 2, 8.4), "Shin" .. s, V3(sx * 3, 4, 8), 1.0)
		addPiston("Pelvis", V3(sx * 10, 2, -10.2), "Thigh" .. s, V3(0, 5, -8.8), 1.1)
		addPiston("UpperArm" .. s, V3(0, -2, 6.4), "Forearm" .. s, V3(0, 6, 7), 0.9)
		addPiston("Pelvis", V3(sx * 8, 3, -10.2), "Torso", V3(sx * 8, -10.5, -19), 1.3)
	end
	addPiston("Torso", V3(-4, 12, -14), "Head", V3(-4, -5, -6), 0.9)
	addPiston("Torso", V3(4, 12, -14), "Head", V3(4, -5, -6), 0.9)

	-- CABOS e CORRENTES (física Verlet)
	cableFolder = Instance.new("Folder"); cableFolder.Name = "MechaCables"; cableFolder.Parent = Workspace
	local CN = DETAIL > 1 and 9 or 6
	local function addCable(a, ao, b, bo, length, r, color)
		local c = { a = N[a], ao = ao, b = b and N[b], bo = bo, n = CN, L = length / (CN - 1), pts = {}, prev = {}, parts = {}, r = r }
		local st = c.a.part.CFrame * ao
		for i = 1, CN do
			local p = st + V3(0, -(i - 1) * c.L, 0)
			c.pts[i], c.prev[i] = p, p
		end
		for i = 1, CN - 1 do
			local p = Instance.new("Part")
			p.Shape = Enum.PartType.Cylinder; p.Size = V3(1, r * 2, r * 2)
			p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CanTouch = false; p.CastShadow = false
			p.Material = METAL; p.Color = color; p.Parent = cableFolder
			c.parts[i] = p
		end
		cables[#cables + 1] = c
	end
	for _, sx in ipairs({ -1, 1 }) do
		addCable("Pelvis", V3(sx * 12, 3, -9.5), nil, nil, 14, 0.45, IRON)
		addCable("Torso", V3(sx * 17, 10, -14.5), nil, nil, 16, 0.5, BLACK)
		addCable("Torso", V3(sx * 8, -11, 18.5), nil, nil, 18, 0.55, IRON)
		addCable("Torso", V3(sx * 13, 5, 17.5), "UpperArm" .. (sx < 0 and "L" or "R"), V3(0, 4, 6.2), 36, 0.5, BLACK)
		addCable("Head", V3(sx * 5, -3, 6), "Torso", V3(sx * 5, 10, -4.5), 9, 0.35, BLACK)
	end
	addCable("Torso", V3(-12, 23, -14.5), nil, nil, 24, 0.6, IRON)       -- corrente do guindaste
	addCable("Torso", V3(10, 0, 17), "ForearmR", V3(3, -3, 2), 42, 0.55, BLACK)

	return model, N, list, fx, link
end

-- escada inclinada (modelo separado: some ao pilotar, volta ao estacionar)
local function buildLadder(O)
	local m = Instance.new("Model")
	m.Name = "MechaLadder"
	local function mk(size, cf, color, mat)
		local p = Instance.new("Part")
		p.Size = size; p.Anchored = true; p.CanCollide = false; p.CanTouch = false; p.CanQuery = false
		p.Color = color; p.Material = mat or METAL; p.TopSurface = Enum.SurfaceType.Smooth; p.BottomSurface = Enum.SurfaceType.Smooth
		p.CFrame = O * cf; p.Parent = m
		return p
	end
	local base, top = LAD_BASE, LAD_TOP
	local axis = (top - base)
	local len = axis.Magnitude
	local mid = (base + top) / 2
	local look = CFrame.lookAt(mid, top)
	for _, sx in ipairs({ -1, 1 }) do
		mk(V3(0.7, 0.7, len), look * CF(sx * 1.5, 0, 0), BRASS)
	end
	local rungs = math.floor(len / 2)
	for i = 0, rungs do
		local p = base + axis * (i / rungs)
		mk(V3(3, 0.4, 0.4), CFrame.lookAt(p, p + axis), IRON)
	end
	mk(V3(6, 0.6, 6), CF(base.X, 0.3, base.Z), IRON, DIAMOND)                 -- base apoiada no chão
	for i = 1, 4 do                                                            -- travessas de reforço
		local p = base + axis * (i / 5)
		mk(V3(4.2, 0.3, 0.3), CFrame.lookAt(p, p + axis) * AN(0, 0, 0.8), RUST)
	end
	m.PrimaryPart = m:FindFirstChildWhichIsA("BasePart")
	m.Parent = Workspace
	return m, O:ToObjectSpace(m.PrimaryPart.CFrame)
end

--------------------------------------------------------------------------------
-- DINÂMICA: engrenagens, pistões, cabos
--------------------------------------------------------------------------------
local function stepPistons()
	for _, p in ipairs(pistons) do
		local A, B = p.a.part.CFrame * p.ao, p.b.part.CFrame * p.bo
		local d = B - A
		local L = d.Magnitude
		if L > 0.1 then
			local dir = d / L
			local l = L * 0.6
			local rot = AN(0, HALF_PI, 0)
			p.outer.Size = V3(l, p.outer.Size.Y, p.outer.Size.Z)
			p.inner.Size = V3(l, p.inner.Size.Y, p.inner.Size.Z)
			local c1, c2 = A + dir * (l / 2), B - dir * (l / 2)
			p.outer.CFrame = CFrame.lookAt(c1, c1 + dir) * rot
			p.inner.CFrame = CFrame.lookAt(c2, c2 + dir) * rot
		end
	end
end

local function resetCables()
	for _, c in ipairs(cables) do
		local st = c.a.part.CFrame * c.ao
		for i = 1, c.n do c.pts[i], c.prev[i] = st + V3(0, -(i - 1) * c.L, 0), st + V3(0, -(i - 1) * c.L, 0) end
	end
end

local function stepCables(dt)
	dt = math.min(dt, 1 / 30)
	local g = V3(0, -130, 0) * dt * dt
	local rot = AN(0, HALF_PI, 0)
	for _, c in ipairs(cables) do
		local pts, prev, n = c.pts, c.prev, c.n
		local A = c.a.part.CFrame * c.ao
		local B = c.b and (c.b.part.CFrame * c.bo) or nil
		for i = 2, n do
			local p = pts[i]
			local v = (p - prev[i]) * 0.985
			prev[i] = p
			pts[i] = p + v + g
		end
		for _ = 1, 3 do
			pts[1] = A
			if B then pts[n] = B end
			for i = 1, n - 1 do
				local p1, p2 = pts[i], pts[i + 1]
				local d = p2 - p1
				local m = d.Magnitude
				if m > 1e-4 then
					local k = (m - c.L) / m
					if i == 1 then pts[2] = p2 - d * k
					elseif i == n - 1 and B then pts[i] = p1 + d * k
					else local h = d * k * 0.5; pts[i] = p1 + h; pts[i + 1] = p2 - h end
				end
			end
		end
		pts[1] = A
		for i = 1, n - 1 do
			local p1, p2 = pts[i], pts[i + 1]
			local d = p2 - p1
			local l = d.Magnitude
			if l > 0.01 then
				local seg = c.parts[i]
				seg.Size = V3(l + 0.08, c.r * 2, c.r * 2)
				seg.CFrame = CFrame.lookAt((p1 + p2) * 0.5, p2) * rot
			end
		end
	end
end

--------------------------------------------------------------------------------
-- FX DO MECHA
--------------------------------------------------------------------------------
local function buildFX(fx, N)
	local anchor = Instance.new("Attachment"); anchor.Name = "FxAnchor"; anchor.Parent = N.Pelvis.part
	fx.anchor = anchor
	fx.spark = newSpark(); fx.spark.Parent = anchor
	fx.dust = newDust(); fx.dust.Parent = anchor
	fx.steam = {}
	for _, a in ipairs(fx.vents) do local s = newSteam(false); s.Parent = a fx.steam[#fx.steam + 1] = s end
	fx.smoke, fx.flame = {}, {}
	for _, a in ipairs(fx.stacks) do
		local s = newSmoke(); s.Parent = a; fx.smoke[#fx.smoke + 1] = s
		local f = newFlame(); f.Parent = a; fx.flame[#fx.flame + 1] = f
	end
	local function light(parent, color, range)
		local l = Instance.new("PointLight"); l.Color = color; l.Range = range; l.Brightness = 0; l.Parent = parent
		return l
	end
	fx.coreLight = light(N.Torso.part, GLOW, 60)
	fx.cabinLight = light(N.Torso.part, Color3.fromRGB(255, 190, 110), 24)
	fx.eyeLight = light(fx.eyeAtt, EYE, 40)
	fx.muzzleLight = light(fx.muzzle, Color3.fromRGB(255, 200, 120), 80)
	fx.coreLight.Brightness = 0
	for key, def in pairs(CFG.Sounds) do
		if def.Id and def.Id ~= "" then
			local snd = Instance.new("Sound")
			snd.Name = "Mecha_" .. key; snd.SoundId = def.Id; snd.Volume = def.Volume or 1
			snd.RollOffMode = Enum.RollOffMode.InverseTapered; snd.RollOffMaxDistance = 500
			snd.Parent = N.Torso.part
			fx.sounds[key] = snd
		end
	end
	local bb = Instance.new("BillboardGui")
	bb.Name = "MechaTag"; bb.Adornee = N.Head.part; bb.Size = UDim2.fromOffset(240, 64)
	bb.StudsOffset = V3(0, 26, 0); bb.AlwaysOnTop = true; bb.MaxDistance = 400
	bb.Parent = player:WaitForChild("PlayerGui")
	local t1 = label(bb, Enum.Font.GothamBlack, CFG.Colors.Accent)
	t1.Size = UDim2.fromScale(1, 0.5); t1.Text = CFG.Name .. "  " .. CFG.Callsign; t1.TextStrokeTransparency = 0.4
	local t2 = label(bb, Enum.Font.GothamMedium, Color3.new(1, 1, 1))
	t2.Position = UDim2.fromScale(0, 0.52); t2.Size = UDim2.fromScale(1, 0.38); t2.TextStrokeTransparency = 0.4
	fx.tag, fx.tagText = bb, t2

	tracerFolder = Instance.new("Folder"); tracerFolder.Name = "MechaTracers"; tracerFolder.Parent = Workspace
	fx.tracers = {}
	for i = 1, 8 do
		local p = Instance.new("Part")
		p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CanTouch = false; p.CastShadow = false
		p.Material = NEON; p.Color = Color3.fromRGB(255, 190, 90); p.Size = V3(1, 1, 12); p.Transparency = 1; p.Parent = tracerFolder
		fx.tracers[i] = { part = p, on = false }
	end
end

--------------------------------------------------------------------------------
-- GEOMETRIA / ESTADOS DO MECHA
--------------------------------------------------------------------------------
local function geom()
	local O = S.O
	local up = (O * LAD_TOP) - (O * LAD_BASE)
	S.u = up.Unit
	S.ladBase, S.ladTop = O * LAD_BASE, O * LAD_TOP
	local n = O.LookVector
	S.n = (n - S.u * n:Dot(S.u)).Unit
	S.path = {}
	for i, p in ipairs(PATH_LOCAL) do S.path[i] = O * p end
end

local function groundedY(pos, ignore)
	rayParams.FilterDescendantsInstances = ignore
	local r = Workspace:Raycast(pos + V3(0, 60, 0), V3(0, -200, 0), rayParams)
	return r and r.Position.Y or nil
end

local function setSolid(on)
	for _, n in ipairs(S.list) do if n.solid then n.part.CanCollide = on end end
end

local function setAvatarHidden()
	for i = #avatarParts, 1, -1 do
		local p = avatarParts[i]
		if not p.Parent then
			table.remove(avatarParts, i)
		elseif p:FindFirstAncestor("Operador") then
			-- Defesa extra: nunca ocultar peças do operador personalizado.
			table.remove(avatarParts, i)
		else
			p.LocalTransparencyModifier = 1
		end
	end
end

local function setBoost(on)
	if S.boosting == on then return end
	if on and (S.state ~= "Pilot" or S.seq or S.energy < CFG.Boost.MinToStart) then return end
	S.boosting = on
	if on then playSound("Boost") else S.lastBoostEnd = os.clock() end
	if S.state == "Pilot" then S.hum.WalkSpeed = on and CFG.Pilot.SprintSpeed or CFG.Pilot.WalkSpeed end
end

-- estaciona o mecha onde ele está agora
local function parkMecha()
	if not S.model then return end
	local cf = S.lastHrpCF or (S.root and S.root.CFrame) or S.O * REST_PELVIS
	local pos = cf.Position
	local yaw = yawOf(cf.LookVector)
	local gy = pos.Y - REST_PELVIS.Position.Y
	local gnd = groundedY(pos, { S.model, S.char or Workspace })
	if gnd and math.abs(gnd - gy) < 8 then gy = gnd end
	local O = CF(pos.X, gy, pos.Z) * AN(0, yaw, 0)
	S.link.C0 = CFrame.new()
	S.N.Pelvis.part.Anchored = true
	S.link.Part0 = nil
	S.model:PivotTo(O * REST_PELVIS)
	S.O = O
	setSolid(true)
	S.boosting, S.firing = false, false
	M.lastYaw = nil
	geom()
	if S.ladder then S.ladder.Parent = Workspace; S.ladder:PivotTo(O * S.ladPivot) end
	resetCables()
end

-- liga a pelve ao HRP (mecha passa a ser controlado)
local function linkPilot()
	local hum, root = S.hum, S.root
	local P = CFG.Pilot
	root.Anchored = true
	root.Size = P.RootSize
	hum.HipHeight = P.HipHeight
	root.CFrame = S.O * REST_PELVIS
	root.AssemblyLinearVelocity = Vector3.zero
	S.link.C0, S.link.C1 = CFrame.new(), CFrame.new()
	S.link.Part0 = root
	setSolid(false)
	S.N.Pelvis.part.Anchored = false
	root.Anchored = false
	hum.UseJumpPower = true
	hum.WalkSpeed, hum.JumpPower, hum.AutoRotate = P.WalkSpeed, P.JumpPower, false
	if S.ladder then S.ladder.Parent = nil end
	-- o operador vai para a poltrona (preso ao tronco do mecha)
	local N, so = S.N, S.sold
	local m = so.nodes.Torso.motor
	m.Part0 = N.Torso.part
	S.soldBase = N.Torso.rest:Inverse() * CF(SEAT_LOCAL)
	m.C0 = S.soldBase
	S.state, S.seatW = "Pilot", 1
	S.energy, S.heat, S.overheated = CFG.Boost.Max, 0, false
	M.lastYaw = nil; M.phase, M.spd = 0, 0
end

local function applyHumanStats()
	local hum, root, o = S.hum, S.root, S.orig
	hum.WalkSpeed, hum.UseJumpPower, hum.JumpPower, hum.JumpHeight = o.walk, o.useJP, o.jp, o.jh
	hum.AutoRotate = true
	root.Size = o.size
	hum.HipHeight = o.hip
end

local function unlinkSoldierToRoot(cf)
	local m = S.sold.nodes.Torso.motor
	m.Part0 = S.root
	S.soldBase = CF(0, 0.5, 0)
	m.C0 = S.soldBase
	S.root.CFrame = cf
end

--------------------------------------------------------------------------------
-- ESCALADA
--------------------------------------------------------------------------------
local function ladderPos(s) return S.ladBase:Lerp(S.ladTop, s) + S.u * 3.0 + S.n * 1.5 end
local function climbCF(s)
	local pos = ladderPos(s)
	return CFrame.fromMatrix(pos, S.u:Cross(S.n), S.u, S.n)
end

local function startClimb(fromPos, s)
	local root, hum = S.root, S.hum
	root.Anchored = true
	root.AssemblyLinearVelocity = Vector3.zero
	hum.WalkSpeed, hum.JumpPower, hum.AutoRotate = 0, 0, false
	S.state = "Climb"
	S.climb = { s = s, mount = 0, from = fromPos, fromCF = CFrame.new(fromPos) * AN(0, yawOf(root.CFrame.LookVector), 0), lastRung = 0 }
	FM.override = nil
end

local function endClimb(vel)
	local root = S.root
	root.Anchored = false
	applyHumanStats()
	if vel then root.AssemblyLinearVelocity = vel end
	S.state, S.climb = "Free", nil
end

local controls
task.spawn(function()
	local ok, res = pcall(function() return require(player:WaitForChild("PlayerScripts"):WaitForChild("PlayerModule", 15)):GetControls() end)
	if ok then controls = res end
end)

local function updateClimb(dt, t)
	local c = S.climb
	local root = S.root
	if not c then return end
	if c.mount then
		c.mount += dt
		local r = easeInOut(c.mount / 0.5)
		local target = climbCF(c.s)
		root.CFrame = c.fromCF:Lerp(target, r)
		FM.climbPh += 0
		if c.mount >= 0.5 then c.mount = nil end
		return
	end
	local mv = controls and controls:GetMoveVector() or Vector3.zero
	local v = -mv.Z
	if math.abs(v) < 0.15 then v = 0 end
	local ds = v * CFG.ClimbSpeed * dt / (S.ladTop - S.ladBase).Magnitude
	local old = c.s
	c.s = math.clamp(c.s + ds, 0, 1)
	FM.climbPh += math.abs(c.s - old) * (S.ladTop - S.ladBase).Magnitude / 2 * PI
	local rung = math.floor(FM.climbPh / PI)
	if rung ~= c.lastRung then c.lastRung = rung; if v ~= 0 then playSound("Rung", 0.2) end end
	root.CFrame = climbCF(c.s)
	-- desceu até o chão
	if c.s <= 0 and v < 0 then
		root.CFrame = CF(S.ladBase + S.n * 3 + V3(0, 3, 0)) * AN(0, yawOf(S.n * -1), 0)
		endClimb()
	end
end

--------------------------------------------------------------------------------
-- SEQUÊNCIAS: ENTRAR NA CABINE / SAIR
--------------------------------------------------------------------------------
local function once(seq, key, fn) if not seq.fired[key] then seq.fired[key] = true fn() end end

local function pathLens(pts)
	local cum, total = { 0 }, 0
	for i = 2, #pts do total += (pts[i] - pts[i - 1]).Magnitude; cum[i] = total end
	return cum, total
end
local function pathAt(pts, cum, d)
	d = math.clamp(d, 0, cum[#cum])
	for i = 2, #pts do
		if d <= cum[i] then
			local r = (d - cum[i - 1]) / math.max(cum[i] - cum[i - 1], 1e-4)
			return pts[i - 1]:Lerp(pts[i], r), (pts[i] - pts[i - 1]).Unit
		end
	end
	return pts[#pts], (pts[#pts] - pts[#pts - 1]).Unit
end

local function startBoard()
	if S.state ~= "Climb" or S.seq then return end
	local B = CFG.Board
	local cum, total = pathLens(S.path)
	local walkEnd = B.Walk0 + total / B.WalkSpeed
	local from = S.root.CFrame
	S.seq = {
		kind = "Board", t0 = os.clock(), fired = {}, cum = cum, len = total, from = from, sealed = false,
		walk0 = B.Walk0, walkEnd = walkEnd, power0 = walkEnd + 0.8, total = walkEnd + 0.8 + B.PowerTime + 0.1,
	}
	S.state = "Board"
	S.climb = nil
	FM.override = 0
end

local function startLeave()
	if S.state ~= "Pilot" or S.seq or S.hum.FloorMaterial == Enum.Material.Air then return end
	local L = CFG.Leave
	local root, hum = S.root, S.hum
	S.firing, S.boosting = false, false
	parkMecha() -- o mecha para e se "desliga" onde está
	root.Anchored = true
	root.Size = S.orig.size
	hum.HipHeight = S.orig.hip
	hum.WalkSpeed, hum.JumpPower = 0, 0
	root.CFrame = S.O * REST_PELVIS
	local cum, total = pathLens(S.path)
	local stand0 = L.Stand0
	local walk0 = stand0 + L.Stand
	local walkEnd = walk0 + total / L.WalkSpeed
	S.seq = { kind = "Leave", t0 = os.clock(), fired = {}, cum = cum, len = total, stand0 = stand0, walk0 = walk0, walkEnd = walkEnd,
		total = walkEnd + L.CloseTime }
	S.state = "Leave"
	S.powerGoal = 0
	FM.override = 0
	M.aimW = 0
end

local function updateSequence(t, dt)
	local seq = S.seq
	if not seq then return end
	local tt = t - seq.t0
	local fx, root, N = S.fx, S.root, S.N
	local so = S.sold.nodes.Torso.motor
	local pts, cum = S.path, seq.cum

	if seq.kind == "Board" then
		local B = CFG.Board
		-- 1) sobe da escada até a sacada
		if tt < B.Mount then
			local r = easeInOut(tt / B.Mount)
			root.CFrame = seq.from:Lerp(CFrame.lookAt(pts[1], pts[1] + (pts[2] - pts[1]) * V3(1, 0, 1)), r)
			FM.override = 0
		end
		-- 2) portas abrem
		if tt >= B.Door0 then once(seq, "open", function() S.hatchGoal = 1; playSound("HatchOpen", 0.05); shake(0.35); burst(fx.steam[1], 16) end) end
		-- 3) caminha: sacada -> porta -> poltrona
		if tt >= seq.walk0 and tt < seq.walkEnd then
			local d = (tt - seq.walk0) * B.WalkSpeed
			local pos, dir = pathAt(pts, cum, d)
			root.CFrame = CFrame.lookAt(pos, pos + V3(dir.X, 0, dir.Z))
			FM.override = B.WalkSpeed * 0.9
			FM.climbPh = 0
			once(seq, "seatrel", function() end)
		elseif tt >= seq.walkEnd and tt < seq.walkEnd + B.Sit then
			-- 4) vira, e senta (mistura da pose de pé para sentado)
			once(seq, "sit", function()
				seq.sitFrom = root.CFrame
				seq.sitTo = CFrame.lookAt(pts[4], pts[4] + S.O.LookVector)
			end)
			local r = smooth((tt - seq.walkEnd) / B.Sit)
			root.CFrame = seq.sitFrom:Lerp(seq.sitTo, r)
			S.seatW = r
			FM.override = 0
			local seatWorld = S.O * CF(SEAT_LOCAL)
			S.soldBase = CF(0, 0.5, 0):Lerp(root.CFrame:ToObjectSpace(seatWorld * AN(0, 0, 0)), r)
		elseif tt >= seq.walkEnd + B.Sit then
			once(seq, "linked", function()
				so.Part0 = N.Torso.part
				S.soldBase = N.Torso.rest:Inverse() * CF(SEAT_LOCAL)
				so.C0 = S.soldBase
				S.seatW = 1
				root.CFrame = S.O * REST_PELVIS
			end)
		end
		-- 5) portas fecham, sistemas ligam
		if tt >= seq.walkEnd + 0.3 then once(seq, "close", function() S.hatchGoal = 0; playSound("HatchClose", 0.05); shake(0.5) end) end
		if tt >= seq.walkEnd + 0.9 then
			once(seq, "sealed", function()
				seq.sealed = true
				playSound("Clang", 0.05); shake(0.8); burst(fx.dust, 14)
				for _, s in ipairs(fx.steam) do burst(s, 22) end
			end)
		end
		if tt >= seq.power0 then
			once(seq, "power", function() S.powerGoal = 1; playSound("PowerUp"); shake(0.4) end)
		end
		Cam.blurT = seq.sealed and (1 - clamp01((tt - seq.power0) / B.PowerTime)) * 0.8 or 0
		if tt >= seq.total then
			flashScreen(0.8); shake(1.0); playSound("Online")
			for _, s in ipairs(fx.steam) do burst(s, 28) end
			S.seq = nil
			S.power, S.powerGoal = 1, 1
			linkPilot()
			Cam.blurT = 0
		end
	else -- Leave
		local L = CFG.Leave
		if tt >= L.Door0 then once(seq, "open", function() S.hatchGoal = 1; playSound("HatchOpen", 0.05); shake(0.35); for _, s in ipairs(fx.steam) do burst(s, 12) end end) end
		once(seq, "down", function() playSound("PowerDown") end)
		if tt >= seq.stand0 and tt < seq.walk0 then
			once(seq, "stand", function()
				local seatWorld = S.O * CF(SEAT_LOCAL)
				local face = CFrame.lookAt(pts[4], pts[4] + S.O.LookVector)
				seq.rel = face:ToObjectSpace(seatWorld)
				unlinkSoldierToRoot(face)
			end)
			local r = smooth((tt - seq.stand0) / L.Stand)
			S.seatW = 1 - r
			S.soldBase = seq.rel:Lerp(CF(0, 0.5, 0), r)
			so.C0 = S.soldBase
		elseif tt >= seq.walk0 and tt < seq.walkEnd then
			once(seq, "stood", function() S.seatW = 0; S.soldBase = CF(0, 0.5, 0); so.C0 = S.soldBase end)
			local d = (tt - seq.walk0) * L.WalkSpeed
			local pos, dir = pathAt(pts, cum, seq.len - d)
			local face = -dir
			root.CFrame = CFrame.lookAt(pos, pos + V3(face.X, 0, face.Z))
			FM.override = L.WalkSpeed * 0.9
		elseif tt >= seq.walkEnd then
			once(seq, "walkdone", function() FM.override = 0; S.hatchGoal = 0; playSound("HatchClose", 0.05) end)
		end
		if tt >= seq.total then
			S.seq = nil
			startClimb(pts[1], 1) -- volta na sacada: desça pela escada ou pule
		end
	end
end

Actions.interact = function()
	if not S.ready or S.seq then return end
	if S.state == "Free" then
		if S.nearLadder then
			startClimb(S.root.Position, 0)
		end
	elseif S.state == "Climb" then
		local c = S.climb
		if c and not c.mount then
			if c.s >= 0.97 then startBoard()
			else
				local vel = S.n * 20 + V3(0, 14, 0)
				endClimb(vel)
			end
		end
	elseif S.state == "Pilot" then
		startLeave()
	end
end
Actions.boost = function(on) setBoost(on) end
Actions.fire = function(on) S.firing = on and S.state == "Pilot" and not S.seq end

UserInputService.JumpRequest:Connect(function()
	if S.state == "Climb" and S.climb and not S.climb.mount and S.ready then
		endClimb(S.n * 22 + V3(0, 16, 0))
	end
end)

--------------------------------------------------------------------------------
-- MOVIMENTO / ANIMAÇÃO DO MECHA
--------------------------------------------------------------------------------
local function onLand(impact)
	local fx = S.fx
	M.springV = impact * 6
	fx.anchor.WorldPosition = S.root.Position - V3(0, 36, 0)
	burst(fx.dust, math.floor(26 * impact))
	shake(0.7 * impact)
	playSound("Land", 0.06)
end
local function onStep(side)
	local fx = S.fx
	local ft = S.N[side == 1 and "FootL" or "FootR"].part
	fx.anchor.WorldPosition = ft.Position - V3(0, 2, 0)
	burst(fx.dust, 8)
	shake(0.5)
	playSound("Step", 0.08)
end

local function updateMotion(dt, t)
	local root, hum = S.root, S.hum
	local vel = root.AssemblyLinearVelocity
	local look = root.CFrame.LookVector
	local hs = math.sqrt(vel.X * vel.X + vel.Z * vel.Z)
	M.hs = hs
	local grounded = hum.FloorMaterial ~= Enum.Material.Air
	if grounded then
		if not M.wasGrounded and M.airTime > 0.35 then onLand(math.clamp(-M.lastVy / 90, 0.25, 1.6)) end
		M.airTime, M.lastVy = 0, 0
	else
		M.airTime += dt
		if vel.Y < M.lastVy then M.lastVy = vel.Y end
	end
	M.wasGrounded, M.grounded = grounded, grounded
	M.air += ((grounded and 0 or 1) - M.air) * damp(7, dt)
	M.springV += (-M.spring * 90 - M.springV * 9) * dt
	M.spring += M.springV * dt
	M.recoilV += (-M.recoil * 180 - M.recoilV * 14) * dt
	M.recoil += M.recoilV * dt
	M.spd += (clamp01(hs / CFG.Pilot.WalkSpeed) * (hs > 0.5 and 1 or 0) - M.spd) * damp(6, dt)
	if grounded and hs > 0.5 then M.phase += hs * dt * CFG.Anim.Stride end
	local idx = math.floor(M.phase / PI)
	if idx ~= M.lastStep then
		M.lastStep = idx
		if M.spd > 0.25 and grounded then onStep(idx % 2 == 0 and 2 or 1) end
	end
	local yaw = yawOf(look)
	if M.lastYaw then M.yawRate += (wrap(yaw - M.lastYaw) / math.max(dt, 1 / 240) - M.yawRate) * damp(8, dt) end
	M.lastYaw = yaw
	local cam = Workspace.CurrentCamera
	if cam then
		local cl = cam.CFrame.LookVector
		M.camYaw = yawOf(cl)
		M.camPitch = math.clamp(math.asin(math.clamp(cl.Y, -1, 1)), -0.6, 0.8)
		local rel = wrap(M.camYaw - yaw)
		M.rel = rel
		local ty = math.clamp(rel, -1.0, 1.0)
		M.torsoYaw += (ty - M.torsoYaw) * damp(7, dt)
		M.headYaw += (math.clamp(rel - ty, -0.6, 0.6) - M.headYaw) * damp(8, dt)
	end
end

local function setJ(n, x, y, z) n.tx, n.ty, n.tz = x, y, z end

local function animateMecha(dt, t)
	local N, A = S.N, CFG.Anim
	local p = S.power
	local slump = 1 - p
	local pil = S.state == "Pilot"
	local spd, air = pil and M.spd or 0, pil and M.air or 0
	local ph = M.phase
	local sL, cL = math.sin(ph), math.cos(ph)
	local sR, cR = -sL, -cL
	local breathe = math.sin(t * 1.1) * 0.012 * p
	local legA, kneeA, armA = A.LegSwing * spd, A.Knee * spd, A.ArmSwing * spd
	local sq = pil and math.clamp(M.spring, 0, 1.3) or 0
	local boostK = S.boosting and 1.6 or 1
	local aimW = M.aimW
	local jitter = math.sin(t * 33) * 0.012 * p * (1 - p) * 4

	local tL = legA * sL + air * 0.4 + sq * 0.3
	local tR = legA * sR + air * 0.15 + sq * 0.3
	local kL = -kneeA * math.max(0, cL) - air * 0.9 - sq * 0.9
	local kR = -kneeA * math.max(0, cR) - air * 0.55 - sq * 0.9
	setJ(N.ThighL, tL, 0, 0); setJ(N.ShinL, kL, 0, 0); setJ(N.FootL, -(tL + kL) * 0.85, 0, 0)
	setJ(N.ThighR, tR, 0, 0); setJ(N.ShinR, kR, 0, 0); setJ(N.FootR, -(tR + kR) * 0.85, 0, 0)

	setJ(N.Torso, breathe - A.Lean * spd * boostK - slump * 0.05 - sq * 0.05 - M.recoil * 0.05 + aimW * 0.04,
		(pil and M.torsoYaw or 0) * p - sL * 0.04 * spd, -sL * A.Sway * spd * 0.6 + jitter)
	local scan = (1 - p) * 0 + (pil and 0 or math.sin(t * 0.35) * 0.2 * p)
	setJ(N.Head, (pil and M.camPitch * 0.6 or 0) * p - slump * 0.45 + breathe, (pil and M.headYaw or 0) * p + scan, 0)

	setJ(N.UpperArmL, -armA * sL * boostK + air * 0.4 + breathe * 3 + slump * 0.05, 0, -0.04 - spd * 0.03)
	setJ(N.ForearmL, 0.2 + 0.3 * spd, 0, 0)
	setJ(N.HandL, 0.1, 0, 0)
	local aimPitch = math.clamp(M.camPitch, -0.7, 0.9)
	setJ(N.UpperArmR, -armA * 0.4 * sR + air * 0.35 + aimW * (HALF_PI + aimPitch) - M.recoil, 0, 0.04 + spd * 0.03 - aimW * 0.05)
	setJ(N.ForearmR, 0.06 * (1 - aimW), 0, 0)

	setJ(N.DoorL, 0, 1.9 * S.hatch, 0)
	setJ(N.DoorR, 0, -1.9 * S.hatch, 0)

	-- íris do olho escaneando (translação)
	M.eyeT += dt * (0.6 + p * 1.4)
	N.Iris.motor.C0 = N.Iris.c0 * CF(math.sin(M.eyeT * 1.3) * 2.2 * p + math.sin(M.eyeT * 0.4) * 0.8, 0, 0)

	local gk = M.gearK
	for _, n in ipairs(S.list) do
		if n.motor then
			if n.spinRate then
				n.ang += n.spinRate * dt * gk
				local a = n.ang
				local R = n.spinAxis == "x" and AN(a, 0, 0) or (n.spinAxis == "y" and AN(0, a, 0) or AN(0, 0, a))
				n.motor.C0 = n.c0 * R
			elseif n.name ~= "Iris" then
				local x, y, z = n.tx, n.ty, n.tz
				if math.abs(x - n.ax) > 1e-4 or math.abs(y - n.ay) > 1e-4 or math.abs(z - n.az) > 1e-4 then
					n.ax, n.ay, n.az = x, y, z
					n.motor.C0 = n.c0 * CFrame.fromEulerAnglesYXZ(x, y, z)
				end
			end
		end
	end

	if pil then
		local bobY = -math.abs(sL) * A.Bob * spd - sq * 2.4
		local roll = sL * A.Sway * spd + math.clamp(-M.yawRate * 0.012, -0.05, 0.05)
		S.link.C0 = CF(0, bobY, 0) * AN(-0.008 * spd * boostK, 0, roll)
	end
end

--------------------------------------------------------------------------------
-- ARMA (canhão)
--------------------------------------------------------------------------------
local function shoot(t)
	local fx = S.fx
	local origin = fx.muzzle.WorldPosition
	local cam = Workspace.CurrentCamera
	local vp = cam.ViewportSize
	local ray = cam:ViewportPointToRay(vp.X / 2, vp.Y / 2)
	rayParams.FilterDescendantsInstances = { S.char, S.model, tracerFolder, cableFolder, dynFolder }
	local res = Workspace:Raycast(ray.Origin, ray.Direction * CFG.Weapon.Range, rayParams)
	local hitPos = res and res.Position or (ray.Origin + ray.Direction * CFG.Weapon.Range)
	local d = hitPos - origin
	local dist = d.Magnitude
	if dist < 4 then return end
	for _, tr in ipairs(fx.tracers) do
		if not tr.on then tr.on, tr.from, tr.dir, tr.dist, tr.travel, tr.hit, tr.hitPos, tr.impact = true, origin, d.Unit, dist, 0, res ~= nil, hitPos, false break end
	end
	fx.anchor.WorldPosition = origin
	burst(fx.spark, 10)
	fx.muzzleLevel = 1
	M.recoilV += 4.5
	S.heat += CFG.Weapon.HeatPerShot
	S.lastFire = t
	shake(0.25)
	playSound("Fire", 0.12)
	if S.heat >= CFG.Weapon.LockAt then S.overheated = true; S.firing = false end
end

local function updateWeapon(dt, t)
	local W, fx = CFG.Weapon, S.fx
	if S.firing and S.state == "Pilot" and not S.seq and not S.overheated and t >= S.nextShot then
		S.nextShot = t + 1 / W.Rate
		shoot(t)
	end
	if not S.firing then S.heat = math.max(0, S.heat - W.Cool * dt) end
	if S.overheated and S.heat <= W.Unlock then S.overheated = false end
	M.aimW += (((S.firing or t - S.lastFire < 1.2) and S.state == "Pilot" and 1 or 0) - M.aimW) * damp(7, dt)
	for _, tr in ipairs(fx.tracers) do
		if tr.on then
			tr.travel += W.Speed * dt
			local head = math.min(tr.travel, tr.dist)
			local tail = math.max(0, head - 14)
			local hp, tp = tr.from + tr.dir * head, tr.from + tr.dir * tail
			tr.part.Size = V3(1, 1, math.max(0.3, head - tail))
			tr.part.CFrame = CFrame.lookAt((hp + tp) * 0.5, hp)
			tr.part.Transparency = 0
			if tr.travel >= tr.dist + 14 then tr.on = false; tr.part.Transparency = 1
			elseif tr.travel >= tr.dist and not tr.impact then
				tr.impact = true
				fx.anchor.WorldPosition = tr.hitPos
				burst(fx.spark, 12)
				if tr.hit then burst(fx.dust, 3) end
			end
		end
	end
	fx.muzzleLevel = math.max(0, (fx.muzzleLevel or 0) - dt * 12)
	fx.muzzleLight.Brightness = Q().lights and fx.muzzleLevel * 10 or 0
end

--------------------------------------------------------------------------------
-- CONTROLE DO PILOTO
--------------------------------------------------------------------------------
local function drive(dt)
	local hum, root = S.hum, S.root
	if S.state ~= "Pilot" then return end
	hum.AutoRotate = false
	local cur = yawOf(root.CFrame.LookVector)
	local md = hum.MoveDirection
	local target, rate
	local rel = M.rel
	if M.aligning and math.abs(rel) < 0.3 then M.aligning = false end
	if md.Magnitude > 0.1 then target, rate = yawOf(md), CFG.Pilot.TurnRate
	elseif math.abs(rel) > 0.95 then M.aligning = true end
	if not target and (M.aligning or (S.firing and math.abs(rel) > 0.6)) then target, rate = M.camYaw, CFG.Pilot.TurnRate * 0.75 end
	if target then
		local step = math.clamp(wrap(target - cur), -rate * dt, rate * dt)
		root.CFrame = CF(root.Position) * AN(0, cur + step, 0)
		root.AssemblyAngularVelocity = Vector3.zero
	end
	S.lastHrpCF = root.CFrame
end

--------------------------------------------------------------------------------
-- LOOP POR FRAME
--------------------------------------------------------------------------------
local function frame(dt)
	if not S.model then return end
	dt = math.min(dt, 0.1)
	local t = os.clock()
	local fx, hum, root = S.fx, S.hum, S.root
	local tier = Q()
	local alive = S.ready and hum and hum.Parent and root and root.Parent
	M.tick += 1

	-- molas das portas e do núcleo (levemente elásticas)
	S.hatchV += ((S.hatchGoal - S.hatch) * 38 - S.hatchV * 7) * dt
	S.hatch += S.hatchV * dt
	S.power += (S.powerGoal - S.power) * damp(1.1, dt)

	if alive then
		-- proximidade da escada
		if S.state == "Free" and not S.seq then
			local d = S.root.Position - S.ladBase
			S.nearLadder = Vector3.new(d.X, 0, d.Z).Magnitude <= CFG.LadderRange and math.abs(d.Y) < 14
		else
			S.nearLadder = false
		end
		if M.tick % 6 == 0 then setAvatarHidden() end

		if S.state == "Pilot" then
			updateMotion(dt, t)
			if S.boosting then
				S.energy = math.max(0, S.energy - CFG.Boost.Drain * dt)
				if S.energy <= 0 then setBoost(false) end
			elseif S.energy < CFG.Boost.Max and t - S.lastBoostEnd > CFG.Boost.RegenDelay then
				S.energy = math.min(CFG.Boost.Max, S.energy + CFG.Boost.Regen * dt)
			end
			updateWeapon(dt, t)
			drive(dt)
			M.ventTimer -= dt
			if M.ventTimer <= 0 then
				M.ventTimer = 4 + math.random() * 4
				if M.spd < 0.2 then for _, s in ipairs(fx.steam) do burst(s, 10) end end
			end
		else
			updateWeapon(dt, t)
			if S.state == "Climb" then updateClimb(dt, t) end
			updateSequence(t, dt)
			-- movimento do operador a pé
			if S.state ~= "Climb" then updateFootMotion(dt) end
		end
		FM.idleT += dt
		soldierAnimate(dt, t)
	end

	-- velocidade das engrenagens: lenta parado, rápida ao andar/turbo
	local gt = (0.06 + 0.94 * S.power) * (1 + 0.8 * M.spd + (S.boosting and 1.5 or 0))
	M.gearK += (gt - M.gearK) * damp(2, dt)

	animateMecha(dt, t)
	stepPistons()
	stepCables(dt)

	-- luzes, olho e núcleo pulsando
	local pulse = 0.5 + 0.5 * math.sin(t * 2.2)
	local glow = S.power
	fx.core.Color = Color3.fromRGB(120, 40, 10):Lerp(GLOW, clamp01(0.25 + glow * 0.75 + pulse * 0.15 * glow))
	fx.eye.Color = Color3.fromRGB(70, 14, 10):Lerp(EYE, clamp01(0.12 + glow * 0.88))
	if S.overheated then fx.eye.Color = Color3.fromRGB(255, 255, 255) end
	for _, v in ipairs(fx.visors) do v.Color = Color3.fromRGB(60, 20, 10):Lerp(EYE, math.sin(t * 3) > 0.6 and 1 or glow * 0.5) end
	for _, l in ipairs(fx.lamps) do l.Transparency = 1 - clamp01(glow * 1.2) end
	fx.cabin.Transparency = 1 - clamp01(math.max(S.hatch * 1.1, glow * 0.8))
	fx.coreLight.Brightness = tier.lights and (0.3 + glow * 1.6 + pulse * 0.4 * glow) or 0
	fx.cabinLight.Brightness = tier.lights and math.max(S.hatch * 2.5, glow * 0.8) or 0
	fx.eyeLight.Brightness = tier.lights and glow * 2.2 or 0

	-- fumaça das chaminés (mais forte ligado), chamas no turbo, vapor
	local smokeRate = (0.15 + glow * 0.85 * (1 + M.spd * 0.8)) * tier.part
	for _, s in ipairs(fx.smoke) do
		local on = tier.part > 0.2
		if s.Enabled ~= on then s.Enabled = on end
		s.Rate = s:GetAttribute("BaseRate") * smokeRate
	end
	local flameOn = alive and S.state == "Pilot" and S.boosting
	for _, f in ipairs(fx.flame) do
		if f.Enabled ~= (flameOn == true) then f.Enabled = flameOn == true end
		if flameOn then f.Rate = f:GetAttribute("BaseRate") * tier.part end
	end

	-- etiqueta 3D
	if M.tick % 12 == 0 then
		local txt = ""
		if alive then
			if S.state == "Pilot" then txt = "PILOTANDO"
			elseif S.state == "Free" then
				local dd = (S.root.Position - S.O.Position).Magnitude
				txt = S.nearLadder and "ESCALAR PELA ESCADA" or string.format("%d studs", math.floor(dd))
			else txt = "ACOPLANDO..." end
		end
		fx.tagText.Text = txt
		fx.tag.Enabled = S.state ~= "Pilot"
	end

	camTargets()
	if not S.seq then Cam.blurT = 0 end
	updateUI(t)
end

--------------------------------------------------------------------------------
-- SESSÃO DO PERSONAGEM
--------------------------------------------------------------------------------
local function spawnMecha(root)
	local O
	if CFG.Spawn.Fixed then
		O = CFG.Spawn.Fixed
	else
		local pos = (root.CFrame * CF(CFG.Spawn.Offset)).Position
		local gy = groundedY(pos, { S.char }) or (root.Position.Y - 3)
		local gp = V3(pos.X, gy, pos.Z)
		local toward = V3(root.Position.X, gy, root.Position.Z)
		O = (toward - gp).Magnitude > 0.5 and CFrame.lookAt(gp, toward) or CF(gp)
	end
	S.O = O
	S.model, S.N, S.list, S.fx, S.link = buildMecha(O)
	buildFX(S.fx, S.N)
	S.ladder, S.ladPivot = buildLadder(O)
	geom()
	resetCables()
end

local function releaseSession()
	S.ready = false
	for _, c in ipairs(S.conns) do c:Disconnect() end
	S.conns = {}
	if S.state == "Pilot" or S.state == "Leave" or S.state == "Board" then
		if S.link and S.link.Part0 then parkMecha() end
	end
	if S.sold then S.sold.model:Destroy(); S.sold = nil end
	S.state, S.seq, S.climb, S.boosting, S.firing = "Free", nil, nil, false, false
	S.hatchGoal, S.powerGoal = 0, 0
	S.seatW = 0
	S.soldBase = CF(0, 0.5, 0)
	avatarParts = {}
	FM.override = nil
	if S.ladder then S.ladder.Parent = Workspace end
	if S.N then S.N.Pelvis.part.Anchored = true end
end

local function setupCharacter(char)
	releaseSession()
	local hum = char:WaitForChild("Humanoid", 15)
	local root = char:WaitForChild("HumanoidRootPart", 15)
	if not hum or not root then return end
	if hum.RigType ~= Enum.HumanoidRigType.R15 then warn("[Mecha] Este sistema exige avatar R15.") return end
	S.char, S.hum, S.root = char, hum, root
	S.orig = { walk = 17, useJP = hum.UseJumpPower, jp = 50, jh = hum.JumpHeight, hip = hum.HipHeight, size = root.Size }
	S.orig.walk = hum.WalkSpeed

	-- esconde o avatar do Roblox e desliga o Animate padrão (economiza CPU)
	for _, d in ipairs(char:GetDescendants()) do
		if d:IsA("BasePart") or d:IsA("Decal") then table.insert(avatarParts, d) end
	end
	table.insert(S.conns, char.DescendantAdded:Connect(function(d)
		-- IMPORTANTE: o operador é filho do Character, mas não deve ser
		-- incluído na lista de peças do avatar que serão ocultadas.
		local isOperatorPart = d:FindFirstAncestor("Operador") ~= nil
		if not isOperatorPart and (d:IsA("BasePart") or d:IsA("Decal")) then
			table.insert(avatarParts, d)
		end
	end))
	hum.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None
	local an = char:FindFirstChild("Animate")
	if an then an.Disabled = true end
	local animator = hum:FindFirstChildOfClass("Animator")
	if animator then for _, tr in ipairs(animator:GetPlayingAnimationTracks()) do tr:Stop(0) end end
	setAvatarHidden()

	S.sold = buildSoldier(char, root)
	if not S.model then spawnMecha(root) end
	S.soldBase = CF(0, 0.5, 0)
	S.seatW = 0
	S.energy, S.heat, S.overheated = CFG.Boost.Max, 0, false

	table.insert(S.conns, hum.Died:Connect(function()
		S.firing, S.boosting = false, false
		if S.state == "Pilot" or S.state == "Board" or S.state == "Leave" then
			if S.link.Part0 then parkMecha() end
			S.seq, S.hatchGoal, S.powerGoal = nil, 0, 0
		end
	end))
	table.insert(S.conns, char.Destroying:Connect(function()
		if S.link and S.link.Part0 then parkMecha() end
	end))
	S.ready = true
end

--------------------------------------------------------------------------------
-- ENTRADA
--------------------------------------------------------------------------------
local function handler(name, state)
	if name == "MechaInteract" then
		if state == Enum.UserInputState.Begin then Actions.interact() end
	elseif name == "MechaBoost" then
		if state == Enum.UserInputState.Begin then Actions.boost(true)
		elseif state == Enum.UserInputState.End or state == Enum.UserInputState.Cancel then Actions.boost(false) end
	elseif name == "MechaFire" then
		if state == Enum.UserInputState.Begin then Actions.fire(true)
		elseif state == Enum.UserInputState.End or state == Enum.UserInputState.Cancel then Actions.fire(false) end
	end
	return Enum.ContextActionResult.Sink
end
local prio = Enum.ContextActionPriority.High.Value
ContextActionService:BindActionAtPriority("MechaInteract", handler, false, prio, CFG.Keys.Interact, CFG.Pad.Interact)
ContextActionService:BindActionAtPriority("MechaBoost", handler, false, prio, CFG.Keys.Boost, CFG.Pad.Boost)
ContextActionService:BindActionAtPriority("MechaFire", handler, false, prio, CFG.Keys.Fire, CFG.Pad.Fire)

--------------------------------------------------------------------------------
-- INICIALIZAÇÃO
--------------------------------------------------------------------------------
Cam.dMin, Cam.dMax = player.CameraMinZoomDistance, player.CameraMaxZoomDistance
Cam.zMin, Cam.zMax, Cam.zMinT, Cam.zMaxT = Cam.dMin, Cam.dMax, Cam.dMin, Cam.dMax

buildUI()
layoutUI()
local function watchViewport()
	local cam = Workspace.CurrentCamera
	if cam then cam:GetPropertyChangedSignal("ViewportSize"):Connect(layoutUI) end
end
watchViewport()
Workspace:GetPropertyChangedSignal("CurrentCamera"):Connect(function() watchViewport() layoutUI() end)

RunService:BindToRenderStep("MechaCamFX", Enum.RenderPriority.Camera.Value + 1, camRender)
RunService:BindToRenderStep("MechaFrame", Enum.RenderPriority.Camera.Value + 2, frame)

player.CharacterAdded:Connect(setupCharacter)
if player.Character then task.spawn(setupCharacter, player.Character) end
