-- Dash em semicírculo até as costas do player mais próximo
local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local lp = Players.LocalPlayer

local cfg = {
	Enabled = true,
	Key = Enum.KeyCode.E,
	ShiftLock = true,   -- gira a câmera junto (trava do shift)
	Duration = 0.40,    -- duração do dash com arco de 180° (s)
	MinDuration = 0.30,  -- duração mínima (s)
	Radius = 4,         -- distância final atrás do alvo (studs)
	Bulge = 0.4,        -- abertura do arco no meio (studs)
	MaxDist = 40,       -- distância máxima pra achar o alvo
}

local dashing, listening = false, false
local STEP = "DashCostasStep"

local function getParts(char)
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	local hum = char:FindFirstChildOfClass("Humanoid")
	if hrp and hum and hum.Health > 0 then return hrp, hum end
end

local function nearest()
	local hrp = getParts(lp.Character)
	if not hrp then return end
	local best, bestD
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= lp then
			local root = getParts(p.Character)
			if root then
				local d = (root.Position - hrp.Position).Magnitude
				if d <= cfg.MaxDist and (not bestD or d < bestD) then
					best, bestD = root, d
				end
			end
		end
	end
	return best
end

local function wrap(a)
	return (a + math.pi) % (2 * math.pi) - math.pi
end

-- gira a câmera (yaw) pra olhar o alvo, mantendo pitch e distância
local function faceCamera(rootPos, targetPos, camDist)
	local cam = workspace.CurrentCamera
	if not cam then return end
	local dir = Vector3.new(targetPos.X - rootPos.X, 0, targetPos.Z - rootPos.Z)
	if dir.Magnitude < 0.01 then return end
	dir = dir.Unit
	local pitch = math.asin(math.clamp(cam.CFrame.LookVector.Y, -1, 1))
	local look = Vector3.new(dir.X * math.cos(pitch), math.sin(pitch), dir.Z * math.cos(pitch))
	local focus = rootPos + Vector3.new(0, 1.5, 0)
	cam.CFrame = CFrame.lookAt(focus - look * math.max(camDist, 0.5), focus)
end

local function dash()
	if dashing or not cfg.Enabled then return end
	local hrp, hum = getParts(lp.Character)
	local target = nearest()
	if not hrp or not target then return end
	dashing = true

	local cam = workspace.CurrentCamera
	local focusStart = hrp.Position + Vector3.new(0, 1.5, 0)
	local camDist = cam and (cam.CFrame.Position - focusStart).Magnitude or 12

	local startOff = hrp.Position - target.Position
	local startAngle = math.atan2(startOff.X, startOff.Z)
	local startRadius = math.max(Vector3.new(startOff.X, 0, startOff.Z).Magnitude, 2)

	local look = target.CFrame.LookVector
	local firstDelta = wrap(math.atan2(-look.X, -look.Z) - startAngle)

	-- só trava o lado se estiver quase de frente (evita flip em ~180°)
	local lockSign = nil
	if math.abs(firstDelta) > 2.5 then
		lockSign = firstDelta >= 0 and 1 or -1
	end

	local duration = cfg.Duration
	local HOLD = 0.25 -- tempo extra mantendo a câmera virada após o dash

	local oldAutoRotate = hum.AutoRotate
	hum.AutoRotate = false
	local t0 = os.clock()

	local function finish()
		RunService:UnbindFromRenderStep(STEP)
		local _, h2 = getParts(lp.Character)
		if h2 then h2.AutoRotate = oldAutoRotate end
		dashing = false
	end

	-- roda DEPOIS da câmera do Roblox pra nossa rotação valer
	RunService:BindToRenderStep(STEP, Enum.RenderPriority.Camera.Value + 1, function()
		local h = getParts(lp.Character)
		if not h or not target.Parent then finish() return end

		local elapsed = os.clock() - t0
		local alpha = math.clamp(elapsed / duration, 0, 1)

		if alpha < 1 then
			local eased = alpha * alpha * (3 - 2 * alpha)

			local l = target.CFrame.LookVector
			local delta = wrap(math.atan2(-l.X, -l.Z) - startAngle)
			if lockSign then
				if lockSign > 0 and delta < 0 then delta += 2 * math.pi end
				if lockSign < 0 and delta > 0 then delta -= 2 * math.pi end
			end

			local angle = startAngle + delta * eased
			local radius = startRadius + (cfg.Radius - startRadius) * eased
			radius += math.sin(math.pi * alpha) * cfg.Bulge

			local pos = target.Position + Vector3.new(math.sin(angle), 0, math.cos(angle)) * radius
			pos = Vector3.new(pos.X, h.Position.Y, pos.Z)

			h.CFrame = CFrame.lookAt(pos, Vector3.new(target.Position.X, pos.Y, target.Position.Z))
			h.AssemblyLinearVelocity = Vector3.zero
			if cfg.ShiftLock then faceCamera(pos, target.Position, camDist) end
		else
			-- fase de hold: mantém personagem e câmera olhando pro alvo
			local p = h.Position
			h.CFrame = CFrame.lookAt(p, Vector3.new(target.Position.X, p.Y, target.Position.Z))
			if cfg.ShiftLock then faceCamera(p, target.Position, camDist) end
			if elapsed >= duration + (cfg.ShiftLock and HOLD or 0) then finish() end
		end
	end)
end

-- ================= GUI =================
local gui = Instance.new("ScreenGui")
gui.Name = "DashCostasGui"
gui.ResetOnSpawn = false
gui.Parent = lp:WaitForChild("PlayerGui")

local frame = Instance.new("Frame")
frame.Size = UDim2.fromOffset(230, 296)
frame.Position = UDim2.fromOffset(30, 200)
frame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
frame.Active, frame.Draggable = true, true
frame.Parent = gui
Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 8)

local function label(text, y)
	local l = Instance.new("TextLabel")
	l.Size = UDim2.new(1, -20, 0, 20)
	l.Position = UDim2.fromOffset(10, y)
	l.BackgroundTransparency = 1
	l.TextColor3 = Color3.new(1, 1, 1)
	l.Font = Enum.Font.GothamBold
	l.TextSize = 13
	l.TextXAlignment = Enum.TextXAlignment.Left
	l.Text = text
	l.Parent = frame
	return l
end

local function button(text, y)
	local b = Instance.new("TextButton")
	b.Size = UDim2.new(1, -20, 0, 28)
	b.Position = UDim2.fromOffset(10, y)
	b.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
	b.TextColor3 = Color3.new(1, 1, 1)
	b.Font = Enum.Font.Gotham
	b.TextSize = 13
	b.Text = text
	b.Parent = frame
	Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)
	return b
end

local function numberBox(name, y, key)
	label(name, y)
	local tb = Instance.new("TextBox")
	tb.Size = UDim2.fromOffset(70, 22)
	tb.Position = UDim2.new(1, -80, 0, y)
	tb.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
	tb.TextColor3 = Color3.new(1, 1, 1)
	tb.Font = Enum.Font.Gotham
	tb.TextSize = 13
	tb.Text = tostring(cfg[key])
	tb.Parent = frame
	tb.FocusLost:Connect(function()
		local n = tonumber(tb.Text)
		if n and n >= 0 then cfg[key] = n else tb.Text = tostring(cfg[key]) end
	end)
end

label("DASH COSTAS", 8)
local toggleBtn = button("Ativado: SIM", 32)
local keyBtn = button("Tecla: " .. cfg.Key.Name, 66)
local shiftBtn = button("Trava Shift: SIM", 100)
numberBox("Duração (s)", 140, "Duration")
numberBox("Duração mín. (s)", 170, "MinDuration")
numberBox("Raio (studs)", 200, "Radius")
numberBox("Abertura arco", 230, "Bulge")
numberBox("Dist. máx.", 260, "MaxDist")

toggleBtn.MouseButton1Click:Connect(function()
	cfg.Enabled = not cfg.Enabled
	toggleBtn.Text = "Ativado: " .. (cfg.Enabled and "SIM" or "NÃO")
end)

shiftBtn.MouseButton1Click:Connect(function()
	cfg.ShiftLock = not cfg.ShiftLock
	shiftBtn.Text = "Trava Shift: " .. (cfg.ShiftLock and "SIM" or "NÃO")
end)

keyBtn.MouseButton1Click:Connect(function()
	listening = true
	keyBtn.Text = "Aperte uma tecla..."
end)

UIS.InputBegan:Connect(function(input, processed)
	if listening and input.UserInputType == Enum.UserInputType.Keyboard then
		listening = false
		cfg.Key = input.KeyCode
		keyBtn.Text = "Tecla: " .. input.KeyCode.Name
		return
	end
	if processed then return end
	if input.KeyCode == cfg.Key then dash() end
end)
