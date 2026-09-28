--[[
    Onyx Hub - Steal an Egg
    AVISO: Este é um script educacional. Não me responsabilizo por banimentos.
    Você precisará descobrir os nomes corretos dos objetos e RemoteEvents do jogo.
]]

-- Configurações iniciais
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- Variáveis de controle
local isStealing = false
local antiBanEnabled = true
local antiKickEnabled = true
local currentEgg = nil
local hasEgg = false
local originalPosition = nil

-- ==========================================
-- 1. SISTEMA ANTI-BAN E ANTI-KICK (Básico)
-- ==========================================
-- Isso tenta impedir que o jogo te chute ou banir enquanto o script roda
local function setupAntiBan()
    if antiKickEnabled then
        -- Impede o kick (método antigo, pode não funcionar em todos os jogos)
        LocalPlayer.Kick = function() 
            warn("Tentativa de Kick bloqueada pelo Onyx Hub!") 
        end
    end
    
    if antiBanEnabled then
        -- Bloqueia a detecção de velocidade e teleporte (exemplo básico)
        LocalPlayer.CharacterAdded:Connect(function(char)
            char:WaitForChild("Humanoid").WalkSpeed = 16 -- Reseta a velocidade para não parecer suspeito
            char:WaitForChild("Humanoid").JumpPower = 50
        end)
    end
end

-- ==========================================
-- 2. INTERFACE (UI) BONITA
-- ==========================================
local function createUI()
    -- Cria a ScreenGui
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "OnyxHub"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

    -- Frame Principal (Design Moderno)
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0, 300, 0, 250)
    mainFrame.Position = UDim2.new(0.5, -150, 0.5, -125)
    mainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    mainFrame.BorderSizePixel = 0
    mainFrame.Active = true
    mainFrame.Draggable = true
    mainFrame.Parent = screenGui

    -- Arredondamento das bordas
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = mainFrame

    -- Título
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 40)
    title.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    title.Text = "Onyx Hub | Steal an Egg"
    title.TextColor3 = Color3.fromRGB(0, 255, 100)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 18
    title.Parent = mainFrame
    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 10)
    titleCorner.Parent = title

    -- Botão de Toggle (Instant Steal)
    local stealBtn = Instance.new("TextButton")
    stealBtn.Size = UDim2.new(0.8, 0, 0, 40)
    stealBtn.Position = UDim2.new(0.1, 0, 0.3, 0)
    stealBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    stealBtn.Text = "Instant Steal: OFF"
    stealBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    stealBtn.Font = Enum.Font.Gotham
    stealBtn.TextSize = 14
    stealBtn.Parent = mainFrame
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 8)
    btnCorner.Parent = stealBtn

    -- Status Label
    local statusLabel = Instance.new("TextLabel")
    statusLabel.Size = UDim2.new(0.8, 0, 0, 20)
    statusLabel.Position = UDim2.new(0.1, 0, 0.6, 0)
    statusLabel.BackgroundTransparency = 1
    statusLabel.Text = "Status: Pronto"
    statusLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    statusLabel.Font = Enum.Font.Gotham
    statusLabel.TextSize = 12
    statusLabel.Parent = mainFrame

    return stealBtn, statusLabel
end

-- ==========================================
-- 3. SISTEMA DE CARREGAMENTO (1% a 100%)
-- ==========================================
local function loadingSequence(statusLabel)
    for i = 1, 100 do
        statusLabel.Text = "Carregando... " .. i .. "%"
        task.wait(0.01) -- Rápido para não demorar
    end
    statusLabel.Text = "Status: Pronto"
end

-- ==========================================
-- 4. LÓGICA DE VOO E TELEPORTE (Instant Steal)
-- ==========================================

-- Função para encontrar o melhor ovo (Raridades: Secreto, Eterno, Divine)
local function findBestEgg()
    local bestEgg = nil
    local highestRarity = 0
    local rarityValues = {
        ["Secret"] = 3,
        ["Eternal"] = 2,
        ["Divine"] = 1
    }

    -- ATENÇÃO: Você precisa mudar "EggsFolder" para o nome real da pasta de ovos no jogo
    local eggsFolder = workspace:FindFirstChild("Eggs") 
    if not eggsFolder then return nil end

    for _, egg in pairs(eggsFolder:GetChildren()) do
        -- Verifica se é um ovo e se tem a raridade desejada
        -- Isso depende de como o jogo nomeia os ovos (ex: "DivineEgg", "SecretEgg")
        for rarityName, value in pairs(rarityValues) do
            if egg.Name:find(rarityName) then
                if value > highestRarity then
                    highestRarity = value
                    bestEgg = egg
                end
            end
        end
    end
    return bestEgg
end

-- Função de Voo Suave (Tween)
local function flyTo(targetPosition, statusLabel)
    local char = LocalPlayer.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    -- Desativa a física para o voo
    hrp.Anchored = true
    
    local distance = (hrp.Position - targetPosition).Magnitude
    local speed = 50 -- Velocidade não muito alta
    local timeToTravel = distance / speed

    local tweenInfo = TweenInfo.new(timeToTravel, Enum.EasingStyle.Linear)
    local tween = TweenService:Create(hrp, tweenInfo, {CFrame = CFrame.new(targetPosition)})
    
    tween:Play()
    statusLabel.Text = "Voando até o ovo..."
    tween.Completed:Wait()
    
    hrp.Anchored = false -- Reativa a física ao chegar
end

-- Função Principal do Script
local function startInstantSteal(statusLabel)
    if isStealing then return end
    isStealing = true
    statusLabel.Text = "Iniciando Instant Steal..."

    while isStealing do
        local char = LocalPlayer.Character
        if not char then break end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hrp then break end

        -- 1. Encontrar o melhor ovo
        local targetEgg = findBestEgg()
        if not targetEgg then
            statusLabel.Text = "Nenhum ovo raro encontrado."
            task.wait(1)
            continue
        end

        -- 2. Voar até o ovo
        local eggPos = targetEgg.Position + Vector3.new(0, 3, 0) -- Fica um pouco acima do ovo
        flyTo(eggPos, statusLabel)
        
        -- 3. Simular a coleta (Pegar o ovo na mão)
        -- Aqui você precisaria disparar o RemoteEvent real do jogo.
        -- Exemplo: game.ReplicatedStorage.Remotes.StealEgg:FireServer(targetEgg)
        statusLabel.Text = "Pegando o ovo..."
        task.wait(0.5) 
        hasEgg = true

        -- 4. Teletransportar para os ovos da galinha (Primeira Ilha)
        -- ATENÇÃO: Substitua "ChickenEggsLocation" pela posição real.
        local safeZonePos = Vector3.new(0, 50, 0) -- Exemplo de coordenada segura
        hrp.CFrame = CFrame.new(safeZonePos)
        statusLabel.Text = "Teleportado para a Zona Segura!"

        -- 5. Verificação de Guardião (Se for atacado, volta)
        -- Isso é um loop de verificação simples
        local startTime = tick()
        while hasEgg and (tick() - startTime) < 5 do -- Verifica por 5 segundos
            local humanoid = char:FindFirstChild("Humanoid")
            if humanoid and humanoid.Health < humanoid.MaxHealth then
                statusLabel.Text = "Ataque detectado! Voltando..."
                -- Volta para o ovo
                flyTo(eggPos, statusLabel)
                -- Tenta pegar de novo
                statusLabel.Text = "Tentando pegar novamente..."
                task.wait(0.5)
                -- (Aqui você dispararia o RemoteEvent novamente)
                break
            end
            task.wait(0.1)
        end
        
        hasEgg = false
        task.wait(1) -- Pequena pausa antes de procurar outro ovo
    end
    
    statusLabel.Text = "Status: Parado"
    isStealing = false
end

-- ==========================================
-- 5. INICIALIZAÇÃO
-- ==========================================
local stealButton, statusLabel = createUI()

-- Conecta o botão à função
stealButton.MouseButton1Click:Connect(function()
    if not isStealing then
        -- Ativa o script
        stealButton.Text = "Instant Steal: ON"
        stealButton.BackgroundColor3 = Color3.fromRGB(0, 150, 0)
        
        -- Roda o carregamento
        loadingSequence(statusLabel)
        
        -- Inicia a lógica
        setupAntiBan()
        startInstantSteal(statusLabel)
    else
        -- Desativa o script
        isStealing = false
        stealButton.Text = "Instant Steal: OFF"
        stealButton.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
        statusLabel.Text = "Status: Desativado"
    end
end)

print("Onyx Hub Carregado com Sucesso!")
