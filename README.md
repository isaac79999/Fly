pcall(function()
	local Players = game:GetService("Players")
	local RunService = game:GetService("RunService")
	local LocalPlayer = Players.LocalPlayer
	local Camera = workspace.CurrentCamera

	-- GUI com ponto branco no centro da tela
	local gui = Instance.new("ScreenGui")
	gui.Name = "FlyUI"
	gui.ResetOnSpawn = false
	gui.IgnoreGuiInset = true
	gui.DisplayOrder = math.huge
	gui.Parent = LocalPlayer:WaitForChild("PlayerGui")

	local dot = Instance.new("Frame")
	dot.Size = UDim2.new(0, 4, 0, 4)
	dot.Position = UDim2.new(0.5, -2, 0.5, -2)
	dot.BackgroundColor3 = Color3.new(1, 1, 1)
	dot.BorderSizePixel = 0
	dot.Name = "FlyDot"
	dot.Parent = gui

	-- Variáveis principais
	local speed = 50
	local fallSpeed = -4
	local flying = true

	-- Função segura para aplicar noclip
	local function applyNoclip()
		for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
			if part:IsA("BasePart") then
				part.CanCollide = false
			end
		end
	end

	-- Função principal de voo
	local function startFly()
		local char = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
		local hrp = char:WaitForChild("HumanoidRootPart")
		local humanoid = char:FindFirstChildOfClass("Humanoid")

		applyNoclip()

		RunService:BindToRenderStep("SecureFly", Enum.RenderPriority.Character.Value + 1, function()
			if not flying or not hrp or not humanoid or humanoid.Health <= 0 then
				RunService:UnbindFromRenderStep("SecureFly")
				return
			end

			-- Faz o personagem girar junto com a câmera
			hrp.CFrame = CFrame.new(hrp.Position, hrp.Position + Camera.CFrame.LookVector)

			local moveDir = humanoid.MoveDirection
			if moveDir.Magnitude > 0 then
				local dir = Camera.CFrame.LookVector.Unit * speed
				hrp.Velocity = Vector3.new(dir.X, dir.Y, dir.Z)
			else
				hrp.Velocity = Vector3.new(0, fallSpeed, 0)
			end

			applyNoclip()
		end)
	end

	-- Começa automaticamente após o personagem spawnar
	if LocalPlayer.Character then
		startFly()
	end
	LocalPlayer.CharacterAdded:Connect(function()
		task.wait(1.2)
		startFly()
	end)
end)
