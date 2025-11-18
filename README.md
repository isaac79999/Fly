pcall(function()
    local Players = game:GetService("Players")
    local RunService = game:GetService("RunService")
    local LocalPlayer = Players.LocalPlayer
    local Camera = workspace.CurrentCamera

    local player = LocalPlayer
    local character = player.Character or player.CharacterAdded:Wait()
    local humanoid = character:WaitForChild("Humanoid")
    local rootPart = character:WaitForChild("HumanoidRootPart")

    -- Variáveis principais
    local flySpeed = 30
    local floatAmplitude = 4
    local floatSpeed = 0.5
    local riseHeight = 6
    local deadzone = 0.1

    -- Estado
    local flying = true
    local inputVector = Vector3.new()
    local sineTimer = 0
    local verticalBaseY = rootPart.Position.Y
    local targetVelocity = Vector3.new()

    -- GUI com ponto branco no centro da tela (do primeiro script)
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

    -- Motor6D helpers
    local function createMotor6D(part0, part1, c0, c1)
        local motor = Instance.new("Motor6D")
        motor.Part0 = part0
        motor.Part1 = part1
        motor.C0 = c0 or CFrame.new()
        motor.C1 = c1 or CFrame.new()
        motor.Parent = part0
        return motor
    end

    local function cleanOldMotors()
        for _, name in pairs({"FlyTorsoMotor", "FlyLeftArmMotor", "FlyRightArmMotor", "FlyRightLegMotor"}) do
            local m = rootPart:FindFirstChild(name) or character:FindFirstChild(name)
            if m then m:Destroy() end
        end
    end

    cleanOldMotors()

    local torso = character:FindFirstChild("UpperTorso") or character:FindFirstChild("Torso")
    local leftArm = character:FindFirstChild("LeftUpperArm") or character:FindFirstChild("Left Arm")
    local rightArm = character:FindFirstChild("RightUpperArm") or character:FindFirstChild("Right Arm")
    local rightLeg = character:FindFirstChild("RightUpperLeg") or character:FindFirstChild("Right Leg")

    local torsoMotor = createMotor6D(rootPart, torso)
    torsoMotor.Name = "FlyTorsoMotor"
    local leftArmMotor = createMotor6D(torso, leftArm)
    leftArmMotor.Name = "FlyLeftArmMotor"
    local rightArmMotor = createMotor6D(torso, rightArm)
    rightArmMotor.Name = "FlyRightArmMotor"
    local rightLegMotor = createMotor6D(torso, rightLeg)
    rightLegMotor.Name = "FlyRightLegMotor"

    -- Poses
    local function poseParado(sine)
        torsoMotor.C0 = CFrame.new()
        leftArmMotor.C0 = CFrame.new(-1.5, 0.5, 0)
        rightArmMotor.C0 = CFrame.new(1.5, 1, 0) * CFrame.Angles(math.rad(-90), math.rad(-20), math.rad(40))
        local legOffset = 0.05 * math.sin(sine * math.pi * 2)
        rightLegMotor.C0 = CFrame.new(0.4, -1 + legOffset, 0.1) * CFrame.Angles(math.rad(10 + legOffset * 10), 0, 0)
    end

    local function poseVoando(sine)
        torsoMotor.C0 = CFrame.Angles(math.rad(-45), 0, 0)
        leftArmMotor.C0 = CFrame.new(-1.5, 0.5, 0) * CFrame.Angles(math.rad(-90), math.rad(-30), math.rad(-10))
        rightArmMotor.C0 = CFrame.new(1.5, 0.5, 0) * CFrame.Angles(math.rad(-20), math.rad(20), math.rad(30))
        local legOffset = 0.1 * math.sin(sine * math.pi * 2)
        rightLegMotor.C0 = CFrame.new(0.5, -1 + legOffset, 0.3) * CFrame.Angles(math.rad(10 + legOffset * 15), 0, math.rad(10))
    end

    -- BodyMovers
    local bodyPos = Instance.new("BodyPosition", rootPart)
    bodyPos.MaxForce = Vector3.new(0, math.huge, 0)
    bodyPos.P = 3000
    bodyPos.D = 200

    local bodyVel = Instance.new("BodyVelocity", rootPart)
    bodyVel.MaxForce = Vector3.new(math.huge, 0, math.huge)
    bodyVel.P = 2000

    -- Começa o voo
    local function startFly()
        flying = true
        verticalBaseY = rootPart.Position.Y + riseHeight
        bodyPos.Position = Vector3.new(rootPart.Position.X, verticalBaseY, rootPart.Position.Z)
        bodyVel.Velocity = Vector3.new(0, 0, 0)
    end

    -- Para o voo e limpa motores
    local function stopFly()
        flying = false
        bodyPos:Destroy()
        bodyVel:Destroy()
        cleanOldMotors()
    end

    -- Atualiza direção baseado no personagem
    local function updateInputVector()
        inputVector = humanoid.MoveDirection
        inputVector = Vector3.new(inputVector.X, 0, inputVector.Z)
    end

    local function updateRotation()
        if inputVector.Magnitude > deadzone then
            local moveDir = rootPart.CFrame:VectorToWorldSpace(inputVector).Unit
            rootPart.CFrame = CFrame.new(rootPart.Position, rootPart.Position + moveDir)
            targetVelocity = moveDir * flySpeed
        else
            targetVelocity = targetVelocity:Lerp(Vector3.new(), 0.1)
        end
    end

    RunService.RenderStepped:Connect(function(dt)
        if flying then
            sineTimer = sineTimer + dt * floatSpeed

            updateInputVector()

            local floatYOffset = floatAmplitude * math.sin(sineTimer * math.pi * 2)
            verticalBaseY = verticalBaseY + (floatYOffset - (bodyPos.Position.Y - verticalBaseY)) * 0.1
            bodyPos.Position = Vector3.new(rootPart.Position.X, verticalBaseY + floatYOffset, rootPart.Position.Z)

            updateRotation()

            bodyVel.Velocity = Vector3.new(targetVelocity.X, 0, targetVelocity.Z)

            if targetVelocity.Magnitude > deadzone then
                poseVoando(sineTimer)
            else
                poseParado(sineTimer)
            end
        end
    end)

    startFly()
end)
