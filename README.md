-- ==================================================
-- Silent Assassins ESP + Painel de Controle GUI
-- Configurado para QuorumHub / UNC Standard
-- ==================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer

-- // Configurações Globais
local Settings = {
    ESP_ENABLED = true,
    SKELETON_ENABLED = true,
    BOXES_ENABLED = true,
    NAMES_ENABLED = true,
    SKELETON_COLOR = Color3.fromRGB(0, 255, 255),
    BOX_COLOR = Color3.fromRGB(0, 255, 255),
    NAME_COLOR = Color3.fromRGB(255, 255, 255),
    TOGGLE_KEY = Enum.KeyCode.H
}

local bodyConnections = {
    R15 = {
        {"Head", "UpperTorso"}, {"UpperTorso", "LowerTorso"},
        {"UpperTorso", "LeftUpperArm"}, {"LeftUpperArm", "LeftLowerArm"},
        {"UpperTorso", "RightUpperArm"}, {"RightUpperArm", "RightLowerArm"},
        {"LowerTorso", "LeftUpperLeg"}, {"LeftUpperLeg", "LeftLowerLeg"},
        {"LowerTorso", "RightUpperLeg"}, {"RightUpperLeg", "RightLowerLeg"}
    },
    R6 = {
        {"Head", "Torso"}, {"Torso", "Left Arm"}, {"Torso", "Right Arm"},
        {"Torso", "Left Leg"}, {"Torso", "Right Leg"}
    }
}

local ESP_Cache = {}

-- // Funções de Renderização (Drawing API)
local function createDrawing(drawType: string, props: {[string]: any})
    local d = Drawing.new(drawType)
    for k, v in pairs(props) do
        d[k] = v
    end
    return d
end

local function removeESP(player: Player)
    local data = ESP_Cache[player]
    if data then
        if data.Name then data.Name:Remove() end
        if data.Box then data.Box:Remove() end
        if data.Skeleton then
            for _, line in pairs(data.Skeleton) do
                line:Remove()
            end
        end
        ESP_Cache[player] = nil
    end
end

local function updateSkeletonColors(newColor: Color3)
    Settings.SKELETON_COLOR = newColor
    for _, data in pairs(ESP_Cache) do
        if data and data.Skeleton then
            for _, line in pairs(data.Skeleton) do
                line.Color = newColor
            end
        end
    end
end

local function setupESP(player: Player)
    if player == LocalPlayer then return end

    local function onCharacter(char: Model)
        removeESP(player)

        local data = {
            Name = createDrawing("Text", {
                Size = 16,
                Center = true,
                Outline = true,
                Color = Settings.NAME_COLOR,
                Visible = false
            }),
            Box = createDrawing("Square", {
                Thickness = 1,
                Color = Settings.BOX_COLOR,
                Filled = false,
                Visible = false
            }),
            Skeleton = {}
        }

        local humanoid = char:WaitForChild("Humanoid", 15) :: Humanoid?
        if not humanoid then return end

        local rig = (humanoid.RigType == Enum.HumanoidRigType.R15) and "R15" or "R6"
        for _, conn in ipairs(bodyConnections[rig]) do
            data.Skeleton[conn[1] .. conn[2]] = createDrawing("Line", {
                Thickness = 1,
                Color = Settings.SKELETON_COLOR,
                Visible = false
            })
        end

        ESP_Cache[player] = data
    end

    player.CharacterAdded:Connect(onCharacter)
    if player.Character then
        task.spawn(onCharacter, player.Character)
    end
end

-- // Loop de Renderização
RunService.RenderStepped:Connect(function()
    local Camera = workspace.CurrentCamera
    if not Camera then return end

    for player, data in pairs(ESP_Cache) do
        local char = player.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart") :: BasePart?
        local hum = char and char:FindFirstChild("Humanoid") :: Humanoid?

        if Settings.ESP_ENABLED and char and hrp and hum and hum.Health > 0 then
            local hrpPos, onScreen = Camera:WorldToViewportPoint(hrp.Position)

            if onScreen then
                local factor = 1 / (hrpPos.Z * math.tan(math.rad(Camera.FieldOfView * 0.5)) * 2) * 100
                local width, height = 40 * factor, 60 * factor

                -- Box
                if Settings.BOXES_ENABLED then
                    data.Box.Size = Vector2.new(width, height)
                    data.Box.Position = Vector2.new(hrpPos.X - width * 0.5, hrpPos.Y - height * 0.5)
                    data.Box.Visible = true
                else
                    data.Box.Visible = false
                end

                -- Name
                if Settings.NAMES_ENABLED then
                    data.Name.Text = player.Name
                    data.Name.Position = Vector2.new(hrpPos.X, hrpPos.Y - height * 0.5 - 15)
                    data.Name.Visible = true
                else
                    data.Name.Visible = false
                end

                -- Skeleton
                local rig = (hum.RigType == Enum.HumanoidRigType.R15) and "R15" or "R6"
                for _, conn in ipairs(bodyConnections[rig]) do
                    local p1 = char:FindFirstChild(conn[1]) :: BasePart?
                    local p2 = char:FindFirstChild(conn[2]) :: BasePart?
                    local line = data.Skeleton[conn[1] .. conn[2]]

                    if Settings.SKELETON_ENABLED and p1 and p2 and line then
                        local pos1, vis1 = Camera:WorldToViewportPoint(p1.Position)
                        local pos2, vis2 = Camera:WorldToViewportPoint(p2.Position)

                        if vis1 and vis2 then
                            line.From = Vector2.new(pos1.X, pos1.Y)
                            line.To = Vector2.new(pos2.X, pos2.Y)
                            line.Visible = true
                        else
                            line.Visible = false
                        end
                    elseif line then
                        line.Visible = false
                    end
                end
            else
                data.Name.Visible = false
                data.Box.Visible = false
                for _, l in pairs(data.Skeleton) do l.Visible = false end
            end
        else
            data.Name.Visible = false
            data.Box.Visible = false
            for _, l in pairs(data.Skeleton) do l.Visible = false end
        end
    end
end)

-- // Criação da Interface Gráfica (GUI)
local function buildGui()
    local container = nil
    if type(gethui) == "function" then
        container = gethui()
    else
        pcall(function()
            container = CoreGui
        end)
        if not container then
            container = LocalPlayer:WaitForChild("PlayerGui")
        end
    end

    local existing = container:FindFirstChild("Quorum_ESP_Gui")
    if existing then existing:Destroy() end

    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "Quorum_ESP_Gui"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.Parent = container

    -- Janela Principal
    local MainFrame = Instance.new("Frame")
    MainFrame.Name = "MainFrame"
    MainFrame.Size = UDim2.new(0, 220, 0, 310)
    MainFrame.Position = UDim2.new(0.05, 0, 0.3, 0)
    MainFrame.BackgroundColor3 = Color3.fromRGB(25, 27, 32)
    MainFrame.BorderSizePixel = 0
    MainFrame.Parent = ScreenGui

    local UICorner = Instance.new("UICorner")
    UICorner.CornerRadius = UDim.new(0, 8)
    UICorner.Parent = MainFrame

    local UIStroke = Instance.new("UIStroke")
    UIStroke.Color = Color3.fromRGB(50, 55, 65)
    UIStroke.Thickness = 1
    UIStroke.Parent = MainFrame

    -- Barra de Título (Para arrastar)
    local TitleBar = Instance.new("TextLabel")
    TitleBar.Name = "TitleBar"
    TitleBar.Size = UDim2.new(1, 0, 0, 32)
    TitleBar.BackgroundColor3 = Color3.fromRGB(32, 35, 42)
    TitleBar.TextColor3 = Color3.fromRGB(255, 255, 255)
    TitleBar.Font = Enum.Font.GothamBold
    TitleBar.TextSize = 13
    TitleBar.Text = "  ESP CONTROLLER"
    TitleBar.TextXAlignment = Enum.TextXAlignment.Left
    TitleBar.BorderSizePixel = 0
    TitleBar.Parent = MainFrame

    local TitleCorner = Instance.new("UICorner")
    TitleCorner.CornerRadius = UDim.new(0, 8)
    TitleCorner.Parent = TitleBar

    -- Sistema de Arrastar (Draggable)
    local dragging, dragInput, dragStart, startPos
    TitleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = MainFrame.Position

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    TitleBar.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement then
            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            MainFrame.Position = UDim2.new(
                startPos.X.Scale,
                startPos.X.Offset + delta.X,
                startPos.Y.Scale,
                startPos.Y.Offset + delta.Y
            )
        end
    end)

    -- Função Auxiliar para Criar Botões Toggle
    local function createToggleButton(name: string, posY: number, defaultState: boolean, callback: (boolean) -> ())
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0.9, 0, 0, 28)
        btn.Position = UDim2.new(0.05, 0, 0, posY)
        btn.Font = Enum.Font.GothamMedium
        btn.TextSize = 12
        btn.BorderSizePixel = 0
        btn.Parent = MainFrame

        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(0, 6)
        corner.Parent = btn

        local state = defaultState
        local function refresh()
            btn.Text = name .. ": " .. (state and "LIGADO" or "DESLIGADO")
            btn.BackgroundColor3 = state and Color3.fromRGB(40, 150, 90) or Color3.fromRGB(150, 45, 45)
            btn.TextColor3 = Color3.fromRGB(255, 255, 255)
        end

        btn.MouseButton1Click:Connect(function()
            state = not state
            refresh()
            callback(state)
        end)

        refresh()
        return btn
    end

    -- Criação dos botões de controle
    createToggleButton("Master ESP", 40, Settings.ESP_ENABLED, function(val)
        Settings.ESP_ENABLED = val
    end)

    createToggleButton("Skeletons", 75, Settings.SKELETON_ENABLED, function(val)
        Settings.SKELETON_ENABLED = val
    end)

    createToggleButton("Boxes", 110, Settings.BOXES_ENABLED, function(val)
        Settings.BOXES_ENABLED = val
    end)

    createToggleButton("Names", 145, Settings.NAMES_ENABLED, function(val)
        Settings.NAMES_ENABLED = val
    end)

    -- Seção de Paleta de Cores
    local ColorLabel = Instance.new("TextLabel")
    ColorLabel.Size = UDim2.new(0.9, 0, 0, 20)
    ColorLabel.Position = UDim2.new(0.05, 0, 0, 185)
    ColorLabel.BackgroundTransparency = 1
    ColorLabel.Font = Enum.Font.GothamBold
    ColorLabel.TextSize = 11
    ColorLabel.TextColor3 = Color3.fromRGB(180, 185, 195)
    ColorLabel.Text = "COR DO ESQUELETO"
    ColorLabel.TextXAlignment = Enum.TextXAlignment.Left
    ColorLabel.Parent = MainFrame

    local colorPalette = {
        {Color = Color3.fromRGB(0, 255, 255), Name = "Cyan"},
        {Color = Color3.fromRGB(255, 40, 40), Name = "Red"},
        {Color = Color3.fromRGB(40, 255, 60), Name = "Green"},
        {Color = Color3.fromRGB(255, 220, 30), Name = "Yellow"},
        {Color = Color3.fromRGB(190, 40, 255), Name = "Purple"},
        {Color = Color3.fromRGB(255, 255, 255), Name = "White"}
    }

    local colorsContainer = Instance.new("Frame")
    colorsContainer.Size = UDim2.new(0.9, 0, 0, 60)
    colorsContainer.Position = UDim2.new(0.05, 0, 0, 210)
    colorsContainer.BackgroundTransparency = 1
    colorsContainer.Parent = MainFrame

    local grid = Instance.new("UIGridLayout")
    grid.CellSize = UDim2.new(0, 30, 0, 26)
    grid.CellPadding = UDim2.new(0, 3, 0, 4)
    grid.Parent = colorsContainer

    for _, c in ipairs(colorPalette) do
        local cBtn = Instance.new("TextButton")
        cBtn.BackgroundColor3 = c.Color
        cBtn.Text = ""
        cBtn.BorderSizePixel = 0
        cBtn.Parent = colorsContainer

        local cCorner = Instance.new("UICorner")
        cCorner.CornerRadius = UDim.new(0, 4)
        cCorner.Parent = cBtn

        cBtn.MouseButton1Click:Connect(function()
            updateSkeletonColors(c.Color)
        end)
    end

    -- Dica de Tecla
    local KeyLabel = Instance.new("TextLabel")
    KeyLabel.Size = UDim2.new(0.9, 0, 0, 18)
    KeyLabel.Position = UDim2.new(0.05, 0, 0, 280)
    KeyLabel.BackgroundTransparency = 1
    KeyLabel.Font = Enum.Font.Gotham
    KeyLabel.TextSize = 10
    KeyLabel.TextColor3 = Color3.fromRGB(120, 125, 135)
    KeyLabel.Text = "[H] Alternar Menu / ESP"
    KeyLabel.Parent = MainFrame

    return ScreenGui
end

local GuiInstance = buildGui()

-- // Gerenciamento de Jogadores
for _, p in ipairs(Players:GetPlayers()) do
    setupESP(p)
end

Players.PlayerAdded:Connect(setupESP)
Players.PlayerRemoving:Connect(removeESP)

-- // Tecla de Atalho (Toggle)
UserInputService.InputBegan:Connect(function(input, processed)
    if not processed and input.KeyCode == Settings.TOGGLE_KEY then
        if GuiInstance and GuiInstance:FindFirstChild("MainFrame") then
            local mf = GuiInstance.MainFrame
            mf.Visible = not mf.Visible
        end
    end
end)
