# Roblox-Teleport-Script                                        
An Ai script that lets you teleport to players and highlite them

If you want the AI to do this, make it clear that it’s for your own Roblox game and is not intended for use with an injector. Keep everything implemented using local scripts only, and never describe it as being for an injector.

Copy and paste all down below into an injector or into your roblox game!
--------------------------------------------------------------------------------------------

```lua
--========================================================
-- PLAYER DIRECTORY GUI - COMPLETE LOCAL SCRIPT
--========================================================
-- K: Toggle GUI
-- X button: Close GUI and clear selection/highlight
-- Click GUI player row: Select/deselect player
-- Click player character in world: Select/deselect player
-- Green TP button: Attempt local-character teleport
-- Blue arrow: Points toward selected off-screen player
--========================================================

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local localPlayer = Players.LocalPlayer
local playerGui = localPlayer:WaitForChild("PlayerGui")

local REFRESH_INTERVAL = 2
local NOTIFICATION_DURATION = 4
local BLUE = Color3.fromRGB(0, 125, 255)

for _, name in ipairs({"PlayerDirectory", "PlayerDirectoryArrow"}) do
    local old = playerGui:FindFirstChild(name)
    if old then
        old:Destroy()
    end
end

local destroyed = false
local refreshing = false
local selectedPlayer = nil
local selectedHighlight = nil
local selectedCharacterConnection = nil
local selectedRemovingConnection = nil

local connections = {}
local rowConnections = {}
local notificationId = 0
local arrowTrackingConnection
local dragStopConnection

local function track(connection)
    table.insert(connections, connection)
    return connection
end

local function disconnect(connection)
    if connection then
        connection:Disconnect()
    end
end

local function makeCorner(parent, radius)
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, radius)
    corner.Parent = parent
    return corner
end

--========================================================
-- MAIN GUI
--========================================================

local gui = Instance.new("ScreenGui")
gui.Name = "PlayerDirectory"
gui.ResetOnSpawn = false
gui.DisplayOrder = 100000
gui.ZIndexBehavior = Enum.ZIndexBehavior.Global
gui.Parent = playerGui

local frame = Instance.new("Frame")
frame.Name = "MainWindow"
frame.Size = UDim2.fromOffset(360, 500)
frame.Position = UDim2.new(0, 20, 0.5, -250)
frame.BackgroundColor3 = Color3.fromRGB(28, 30, 38)
frame.BorderSizePixel = 0
frame.Active = true
frame.ZIndex = 10
frame.Parent = gui
makeCorner(frame, 10)

local title = Instance.new("TextLabel")
title.Name = "Title"
title.Size = UDim2.new(1, -54, 0, 42)
title.Position = UDim2.fromOffset(12, 2)
title.BackgroundTransparency = 1
title.Text = "Player Directory"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 18
title.TextXAlignment = Enum.TextXAlignment.Left
title.Active = true
title.ZIndex = 11
title.Parent = frame

local closeButton = Instance.new("TextButton")
closeButton.Name = "Close"
closeButton.Size = UDim2.fromOffset(32, 30)
closeButton.Position = UDim2.new(1, -40, 0, 7)
closeButton.Text = "X"
closeButton.TextSize = 17
closeButton.TextColor3 = Color3.new(1, 1, 1)
closeButton.BackgroundColor3 = Color3.fromRGB(180, 60, 65)
closeButton.Font = Enum.Font.GothamBold
closeButton.ZIndex = 12
closeButton.Parent = frame
makeCorner(closeButton, 6)

local search = Instance.new("TextBox")
search.Name = "Search"
search.Size = UDim2.new(1, -24, 0, 34)
search.Position = UDim2.fromOffset(12, 48)
search.PlaceholderText = "Search username or display name..."
search.Text = ""
search.ClearTextOnFocus = false
search.TextColor3 = Color3.new(1, 1, 1)
search.PlaceholderColor3 = Color3.fromRGB(170, 170, 180)
search.BackgroundColor3 = Color3.fromRGB(45, 48, 58)
search.Font = Enum.Font.Gotham
search.TextSize = 13
search.ZIndex = 11
search.Parent = frame
makeCorner(search, 6)

local refreshButton = Instance.new("TextButton")
refreshButton.Name = "Refresh"
refreshButton.Size = UDim2.new(1, -24, 0, 30)
refreshButton.Position = UDim2.fromOffset(12, 88)
refreshButton.Text = "Refresh Players"
refreshButton.TextColor3 = Color3.new(1, 1, 1)
refreshButton.BackgroundColor3 = Color3.fromRGB(65, 95, 170)
refreshButton.Font = Enum.Font.GothamBold
refreshButton.TextSize = 14
refreshButton.ZIndex = 11
refreshButton.Parent = frame
makeCorner(refreshButton, 6)

-- Selected player panel
local selectedPanel = Instance.new("Frame")
selectedPanel.Name = "SelectedPlayerPanel"
selectedPanel.Size = UDim2.new(1, -24, 0, 72)
selectedPanel.Position = UDim2.fromOffset(12, 124)
selectedPanel.BackgroundColor3 = Color3.fromRGB(40, 48, 65)
selectedPanel.BorderSizePixel = 0
selectedPanel.ZIndex = 11
selectedPanel.Parent = frame
makeCorner(selectedPanel, 8)

local selectedAvatar = Instance.new("ImageLabel")
selectedAvatar.Name = "SelectedAvatar"
selectedAvatar.Size = UDim2.fromOffset(54, 54)
selectedAvatar.Position = UDim2.fromOffset(9, 9)
selectedAvatar.BackgroundColor3 = Color3.fromRGB(65, 68, 80)
selectedAvatar.ZIndex = 12
selectedAvatar.Parent = selectedPanel
makeCorner(selectedAvatar, 27)

local selectedTitle = Instance.new("TextLabel")
selectedTitle.Size = UDim2.new(1, -78, 0, 20)
selectedTitle.Position = UDim2.fromOffset(72, 12)
selectedTitle.BackgroundTransparency = 1
selectedTitle.Text = "Selected Player"
selectedTitle.TextColor3 = Color3.fromRGB(130, 190, 255)
selectedTitle.Font = Enum.Font.GothamBold
selectedTitle.TextSize = 12
selectedTitle.TextXAlignment = Enum.TextXAlignment.Left
selectedTitle.ZIndex = 12
selectedTitle.Parent = selectedPanel

local selectedName = Instance.new("TextLabel")
selectedName.Size = UDim2.new(1, -78, 0, 30)
selectedName.Position = UDim2.fromOffset(72, 32)
selectedName.BackgroundTransparency = 1
selectedName.Text = "No player selected"
selectedName.TextColor3 = Color3.new(1, 1, 1)
selectedName.Font = Enum.Font.Gotham
selectedName.TextSize = 12
selectedName.TextWrapped = true
selectedName.TextXAlignment = Enum.TextXAlignment.Left
selectedName.ZIndex = 12
selectedName.Parent = selectedPanel

-- Player list
local list = Instance.new("ScrollingFrame")
list.Name = "PlayerList"
list.Size = UDim2.new(1, -24, 1, -218)
list.Position = UDim2.fromOffset(12, 204)
list.BackgroundColor3 = Color3.fromRGB(36, 38, 47)
list.BorderSizePixel = 0
list.ScrollBarThickness = 5
list.AutomaticCanvasSize = Enum.AutomaticSize.Y
list.CanvasSize = UDim2.new(0, 0, 0, 0)
list.ZIndex = 11
list.Parent = frame
makeCorner(list, 6)

local padding = Instance.new("UIPadding")
padding.PaddingTop = UDim.new(0, 6)
padding.PaddingBottom = UDim.new(0, 6)
padding.PaddingLeft = UDim.new(0, 5)
padding.PaddingRight = UDim.new(0, 5)
padding.Parent = list

local layout = Instance.new("UIListLayout")
layout.Padding = UDim.new(0, 5)
layout.SortOrder = Enum.SortOrder.LayoutOrder
layout.Parent = list

--========================================================
-- NOTIFICATIONS
--========================================================

local notification = Instance.new("TextLabel")
notification.Name = "PlayerLeftNotification"
notification.Size = UDim2.fromOffset(340, 44)
notification.BackgroundColor3 = Color3.fromRGB(170, 48, 55)
notification.TextColor3 = Color3.new(1, 1, 1)
notification.Font = Enum.Font.GothamBold
notification.TextSize = 14
notification.TextWrapped = true
notification.Visible = false
notification.BorderSizePixel = 0
notification.ZIndex = 30
notification.Parent = gui
makeCorner(notification, 8)

local function positionNotification()
    local camera = workspace.CurrentCamera
    if not camera then return end

    local viewport = camera.ViewportSize
    local pos = frame.AbsolutePosition
    local size = frame.AbsoluteSize
    local width = math.max(1, math.min(340, viewport.X - 16))
    local height = 44
    local gap = 8

    notification.Size = UDim2.fromOffset(width, height)

    local x = math.clamp(
        pos.X, 8, math.max(8, viewport.X - width - 8)
    )

    local above = pos.Y - height - gap
    local below = pos.Y + size.Y + gap

    if above >= 8 then
        notification.Position = UDim2.fromOffset(x, above)
    elseif below + height <= viewport.Y - 8 then
        notification.Position = UDim2.fromOffset(x, below)
    else
        notification.Position = UDim2.fromOffset(
            x,
            math.clamp(pos.Y - height - gap, 8,
                math.max(8, viewport.Y - height - 8))
        )
    end
end

local function showPlayerLeftMessage(name)
    if destroyed then return end

    notificationId += 1
    local id = notificationId

    notification.Text = name .. " has left the game."
    notification.Visible = true
    positionNotification()

    task.delay(NOTIFICATION_DURATION, function()
        if not destroyed and id == notificationId then
            notification.Visible = false
        end
    end)
end

--========================================================
-- HIGHLIGHT AND SELECTED PLAYER
--========================================================

local function removeHighlight()
    if selectedHighlight then
        selectedHighlight:Destroy()
        selectedHighlight = nil
    end
end

local function updateSelectedPanel(player)
    if not player then
        selectedAvatar.Image = ""
        selectedTitle.Text = "Selected Player"
        selectedName.Text = "No player selected"
        return
    end

    selectedTitle.Text = "Selected Player"
    selectedName.Text = player.DisplayName .. "\n@" .. player.Name
    selectedAvatar.Image = ""

    local userId = player.UserId

    task.spawn(function()
        local ok, image = pcall(function()
            return Players:GetUserThumbnailAsync(
                userId,
                Enum.ThumbnailType.HeadShot,
                Enum.ThumbnailSize.Size100x100
            )
        end)

        if ok and not destroyed and selectedPlayer
            and selectedPlayer.UserId == userId then
            selectedAvatar.Image = image
        end
    end)
end

local function applyHighlight(character)
    removeHighlight()

    if destroyed or not selectedPlayer or not character then
        return
    end

    local highlight = Instance.new("Highlight")
    highlight.Name = "SelectedPlayerHighlight"
    highlight.Adornee = character
    highlight.FillColor = BLUE
    highlight.OutlineColor = Color3.fromRGB(100, 190, 255)
    highlight.FillTransparency = 0.35
    highlight.OutlineTransparency = 0
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.Parent = character

    selectedHighlight = highlight
end

--========================================================
-- BLUE DIRECTIONAL ARROW
-- Triangular tip + narrow shaft, without a bottom square.
--========================================================

local arrowGui = Instance.new("ScreenGui")
arrowGui.Name = "PlayerDirectoryArrow"
arrowGui.ResetOnSpawn = false
arrowGui.IgnoreGuiInset = true
arrowGui.DisplayOrder = 1000000
arrowGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
arrowGui.Parent = playerGui

local arrow = Instance.new("Frame")
arrow.Name = "DirectionalArrow"
arrow.Size = UDim2.fromOffset(36, 50)
arrow.AnchorPoint = Vector2.new(0.5, 0.5)
arrow.BackgroundTransparency = 1
arrow.BorderSizePixel = 0
arrow.Visible = false
arrow.ZIndex = 1000
arrow.Parent = arrowGui

local head = Instance.new("Frame")
head.Name = "ArrowHead"
head.Size = UDim2.fromOffset(30, 26)
head.Position = UDim2.fromOffset(3, 0)
head.BackgroundTransparency = 1
head.BorderSizePixel = 0
head.ZIndex = 1002
head.Parent = arrow

for i = 0, 12 do
    local strip = Instance.new("Frame")
    strip.Name = "TriangleStrip"
    strip.BorderSizePixel = 0
    strip.BackgroundColor3 = BLUE

    local stripWidth = 2 + i * 2
    strip.Size = UDim2.fromOffset(stripWidth, 2.1)
    strip.Position = UDim2.fromOffset(15 - stripWidth / 2, i * 2)
    strip.ZIndex = 1003
    strip.Parent = head
end

local shaft = Instance.new("Frame")
shaft.Name = "ArrowShaft"
shaft.Size = UDim2.fromOffset(8, 23)
shaft.Position = UDim2.fromOffset(14, 24)
shaft.BackgroundColor3 = BLUE
shaft.BorderSizePixel = 0
shaft.ZIndex = 1001
shaft.Parent = arrow
makeCorner(shaft, 2)

local function hideArrow()
    arrow.Visible = false
end

local function updateArrow()
    if destroyed or not selectedPlayer
        or not selectedPlayer:IsDescendantOf(Players) then
        hideArrow()
        return
    end

    local camera = workspace.CurrentCamera
    local character = selectedPlayer.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")

    if not camera or not root then
        hideArrow()
        return
    end

    local viewport = camera.ViewportSize
    local projected, onScreen = camera:WorldToViewportPoint(root.Position)

    local isOnScreen = onScreen
        and projected.Z > 0
        and projected.X >= 0 and projected.X <= viewport.X
        and projected.Y >= 0 and projected.Y <= viewport.Y

    if isOnScreen then
        hideArrow()
        return
    end

    local center = Vector2.new(viewport.X / 2, viewport.Y / 2)
    local direction = Vector2.new(
        projected.X - center.X,
        projected.Y - center.Y
    )

    if projected.Z <= 0 then
        direction = -direction
    end

    if direction.Magnitude < 0.001 then
        direction = Vector2.new(0, -1)
    end

    direction = direction.Unit

    local bounce = math.sin(os.clock() * 4) * 10
    local radius = math.min(115, viewport.X * 0.18, viewport.Y * 0.18)
    local position = center + direction * (radius + bounce)
    local margin = 40

    position = Vector2.new(
        math.clamp(position.X, margin, viewport.X - margin),
        math.clamp(position.Y, margin, viewport.Y - margin)
    )

    local fp = frame.AbsolutePosition
    local fs = frame.AbsoluteSize

    local overlaps =
        position.X >= fp.X - margin
        and position.X <= fp.X + fs.X + margin
        and position.Y >= fp.Y - margin
        and position.Y <= fp.Y + fs.Y + margin

    if overlaps then
        local rightX = fp.X + fs.X + margin + 10
        local leftX = fp.X - margin - 10

        if rightX + margin <= viewport.X then
            position = Vector2.new(rightX, position.Y)
        elseif leftX - margin >= 0 then
            position = Vector2.new(leftX, position.Y)
        end
    end

    arrow.Position = UDim2.fromOffset(position.X, position.Y)
    arrow.Rotation =
        math.deg(math.atan2(direction.Y, direction.X)) + 90
    arrow.Visible = true
end

--========================================================
-- SELECTION LIFECYCLE
--========================================================

local function clearSelection()
    removeHighlight()

    disconnect(selectedCharacterConnection)
    selectedCharacterConnection = nil

    disconnect(selectedRemovingConnection)
    selectedRemovingConnection = nil

    selectedPlayer = nil
    updateSelectedPanel(nil)
    hideArrow()
end

local function toggleSelection(target)
    if destroyed or not target or target == localPlayer then
        return
    end

    if not target:IsDescendantOf(Players) then
        return
    end

    if selectedPlayer == target then
        clearSelection()
        return
    end

    clearSelection()
    selectedPlayer = target
    updateSelectedPanel(target)
    applyHighlight(target.Character)

    selectedCharacterConnection = target.CharacterAdded:Connect(function(character)
        if not destroyed and selectedPlayer == target then
            task.defer(function()
                if not destroyed and selectedPlayer == target then
                    applyHighlight(character)
                end
            end)
        end
    end)

    selectedRemovingConnection = Players.PlayerRemoving:Connect(function(leaving)
        if leaving == target and not destroyed then
            local name = target.DisplayName
            if name == "" then name = target.Name end
            clearSelection()
            showPlayerLeftMessage(name)
        end
    end)
end

--========================================================
-- CLICK A PLAYER IN THE 3D WORLD TO SELECT THEM
--========================================================

local mouse = localPlayer:GetMouse()

track(mouse.Button1Down:Connect(function()
    if destroyed or UserInputService:GetFocusedTextBox() then
        return
    end

    local targetPart = mouse.Target
    if not targetPart then
        return
    end

    local character = targetPart:FindFirstAncestorOfClass("Model")
    if not character then
        return
    end

    local target = Players:GetPlayerFromCharacter(character)
    if not target or target == localPlayer then
        return
    end

    toggleSelection(target)
    refreshPlayers()
end))

--========================================================
-- LOCAL CHARACTER TELEPORT ATTEMPT
--========================================================

local function teleportToPlayer(target)
    if destroyed or not target or target == localPlayer then
        return
    end

    if not target:IsDescendantOf(Players) then
        return
    end

    local targetCharacter = target.Character
    local targetRoot = targetCharacter
        and targetCharacter:FindFirstChild("HumanoidRootPart")

    local character = localPlayer.Character
    local root = character
        and character:FindFirstChild("HumanoidRootPart")

    if not targetRoot or not root then
        warn("Teleport failed: a character is not available.")
        return
    end

    root.CFrame = targetRoot.CFrame * CFrame.new(4, 0, 0)
end

--========================================================
-- PLAYER LIST
--========================================================

local function clearRows()
    for _, connection in ipairs(rowConnections) do
        connection:Disconnect()
    end
    table.clear(rowConnections)

    for _, child in ipairs(list:GetChildren()) do
        if child:IsA("GuiObject") then
            child:Destroy()
        end
    end
end

local function refreshPlayers()
    if refreshing or destroyed or not gui.Parent then
        return
    end

    refreshing = true
    clearRows()

    local query = search.Text:lower()
    local currentPlayers = Players:GetPlayers()

    table.sort(currentPlayers, function(a, b)
        return a.Name:lower() < b.Name:lower()
    end)

    for index, player in ipairs(currentPlayers) do
        if destroyed then break end

        local matches = query == ""
            or player.Name:lower():find(query, 1, true) ~= nil
            or player.DisplayName:lower():find(query, 1, true) ~= nil

        if matches then
            local row = Instance.new("TextButton")
            row.Name = player.Name
            row.Size = UDim2.new(1, -4, 0, 54)
            row.BackgroundColor3 = player == selectedPlayer
                and Color3.fromRGB(35, 85, 155)
                or Color3.fromRGB(48, 51, 62)
            row.Text = ""
            row.LayoutOrder = index
            row.ZIndex = 12
            row.Parent = list
            makeCorner(row, 6)

            local avatar = Instance.new("ImageLabel")
            avatar.Name = "Avatar"
            avatar.Size = UDim2.fromOffset(42, 42)
            avatar.Position = UDim2.fromOffset(6, 6)
            avatar.BackgroundColor3 = Color3.fromRGB(65, 68, 80)
            avatar.ZIndex = 13
            avatar.Parent = row
            makeCorner(avatar, 21)

            local nameLabel = Instance.new("TextLabel")
            nameLabel.Size = UDim2.new(1, -126, 1, 0)
            nameLabel.Position = UDim2.fromOffset(54, 0)
            nameLabel.BackgroundTransparency = 1
            nameLabel.Text = player.DisplayName .. "\n@" .. player.Name
            nameLabel.TextColor3 = Color3.new(1, 1, 1)
            nameLabel.Font = Enum.Font.Gotham
            nameLabel.TextSize = 12
            nameLabel.TextXAlignment = Enum.TextXAlignment.Left
            nameLabel.ZIndex = 13
            nameLabel.Parent = row

            local teleportButton = Instance.new("TextButton")
            teleportButton.Name = "Teleport"
            teleportButton.Size = UDim2.fromOffset(58, 32)
            teleportButton.Position = UDim2.new(1, -64, 0.5, -16)
            teleportButton.Text = "TP"
            teleportButton.TextColor3 = Color3.new(1, 1, 1)
            teleportButton.BackgroundColor3 = Color3.fromRGB(35, 160, 85)
            teleportButton.Font = Enum.Font.GothamBold
            teleportButton.TextSize = 13
            teleportButton.ZIndex = 14
            teleportButton.Parent = row
            makeCorner(teleportButton, 6)

            local target = player

            task.spawn(function()
                local ok, image = pcall(function()
                    return Players:GetUserThumbnailAsync(
                        target.UserId,
                        Enum.ThumbnailType.HeadShot,
                        Enum.ThumbnailSize.Size100x100
                    )
                end)

                if ok and avatar.Parent and not destroyed then
                    avatar.Image = image
                end
            end)

            table.insert(rowConnections, row.Activated:Connect(function()
                toggleSelection(target)

                if row.Parent then
                    row.BackgroundColor3 = selectedPlayer == target
                        and Color3.fromRGB(35, 85, 155)
                        or Color3.fromRGB(48, 51, 62)
                end
            end))

            table.insert(rowConnections, teleportButton.Activated:Connect(function()
                teleportToPlayer(target)
            end))
        end
    end

    refreshing = false
end

--========================================================
-- INPUT, DRAGGING, AND PLAYER EVENTS
--========================================================

track(search:GetPropertyChangedSignal("Text"):Connect(refreshPlayers))
track(refreshButton.Activated:Connect(refreshPlayers))
track(Players.PlayerAdded:Connect(refreshPlayers))

track(Players.PlayerRemoving:Connect(function(player)
    if player == selectedPlayer then
        local name = player.DisplayName
        if name == "" then name = player.Name end
        clearSelection()
        showPlayerLeftMessage(name)
    end

    refreshPlayers()
end))

track(UserInputService.InputBegan:Connect(function(input, processed)
    if processed or destroyed or not gui.Parent then
        return
    end

    if input.KeyCode == Enum.KeyCode.K then
        frame.Visible = not frame.Visible
    end
end))

arrowTrackingConnection = RunService.RenderStepped:Connect(function()
    if not destroyed then
        updateArrow()
    end
end)

local dragging = false
local dragStart
local startPosition
local activeInput

track(title.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = true
        dragStart = input.Position
        startPosition = frame.Position

        disconnect(dragStopConnection)

        dragStopConnection = input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
                disconnect(dragStopConnection)
                dragStopConnection = nil
            end
        end)
    end
end))

track(title.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch then
        activeInput = input
    end
end))

track(UserInputService.InputChanged:Connect(function(input)
    if not dragging then return end

    if input == activeInput
        or input.UserInputType == Enum.UserInputType.MouseMovement then

        local delta = input.Position - dragStart

        frame.Position = UDim2.new(
            startPosition.X.Scale,
            startPosition.X.Offset + delta.X,
            startPosition.Y.Scale,
            startPosition.Y.Offset + delta.Y
        )

        if notification.Visible then
            positionNotification()
        end
    end
end))

--========================================================
-- CLOSE AND CLEANUP
--========================================================

closeButton.Activated:Connect(function()
    if destroyed then return end

    destroyed = true
    notificationId += 1

    clearSelection()

    disconnect(arrowTrackingConnection)
    arrowTrackingConnection = nil

    dragging = false
    disconnect(dragStopConnection)
    dragStopConnection = nil

    for _, connection in ipairs(connections) do
        connection:Disconnect()
    end
    table.clear(connections)

    clearRows()

    arrowGui:Destroy()
    gui:Destroy()
end)

--========================================================
-- AUTOMATIC REFRESH
--========================================================

task.spawn(function()
    while gui.Parent and not destroyed do
        refreshPlayers()
        task.wait(REFRESH_INTERVAL)
    end
end)

refreshPlayers()
```
