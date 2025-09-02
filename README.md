# Dino-Char
Turns player = dino


local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local frame = script.Parent
local charactersFolder = ReplicatedStorage:WaitForChild("Characters")

local selecting = false

local function classifyAnimName(name)
	if not name then return nil end
	local lname = name:lower()
	if lname:find("idle") then return "idle" end
	if lname:find("walk") then return "walk" end
	if lname:find("run") then return "run" end
	if lname == "idle" or lname == "idleanim" then return "idle" end
	if lname == "walk" or lname == "walkanim" then return "walk" end
	if lname == "run" or lname == "runanim" then return "run" end
	return nil
end

local function swapToDino(dinoName)
	if selecting then return end
	selecting = true

	local template = charactersFolder:FindFirstChild(dinoName)
	if not template then
		warn("Dino template not found for:", dinoName)
		selecting = false
		return
	end

	local oldChar = player.Character or player.CharacterAdded:Wait()
	local oldHRP = oldChar:FindFirstChild("HumanoidRootPart") or oldChar.PrimaryPart

	local newChar = template:Clone()
	newChar.Name = player.Name
	newChar.Parent = workspace

	local newHRP = newChar:FindFirstChild("HumanoidRootPart") or newChar.PrimaryPart
	if newHRP and oldHRP then
		newChar:SetPrimaryPartCFrame(oldHRP.CFrame)
	end

	local humanoid = newChar:WaitForChild("Humanoid", 5)
	if not humanoid then
		warn("New dino has no Humanoid:", dinoName)
		newChar:Destroy()
		selecting = false
		return
	end

	local animator = humanoid:FindFirstChildOfClass("Animator")
	if not animator then
		animator = Instance.new("Animator")
		animator.Parent = humanoid
	end

	local animFolder = newChar:FindFirstChild("AnimSaves")
	local tracks = { idle = nil, walk = nil, run = nil }

	if animFolder then
		for _, anim in ipairs(animFolder:GetChildren()) do
			if anim:IsA("Animation") then
				local clonedAnim = anim:Clone()
				clonedAnim.Parent = humanoid
				local ok, track = pcall(function() return animator:LoadAnimation(clonedAnim) end)
				if ok and track then
					local which = classifyAnimName(anim.Name)
					if which then
						tracks[which] = track
					else
						tracks[anim.Name] = track
					end
				else
					warn("Failed to load animation:", anim, "for dino:", dinoName)
				end
			end
		end
	else
		warn("No AnimSaves folder in dino model:", dinoName)
	end

	local currentState = nil
	local runningConn
	local function switchTo(state)
		if state == currentState then return end
		if currentState and tracks[currentState] and tracks[currentState].IsPlaying then
			pcall(function() tracks[currentState]:Stop(0.2) end)
		end
		currentState = state
		local track = tracks[state]
		if track then
			pcall(function() track:Play(0.2, 1, 1) end)
		end
	end

	runningConn = humanoid.Running:Connect(function(speed)
		if speed <= 0.1 then
			switchTo("idle")
		else
			local walkSpeed = humanoid.WalkSpeed or 16
			if speed <= walkSpeed + 0.5 then
				switchTo("walk")
			else
				switchTo("run")
			end
		end
	end)

	spawn(function()
		local initialSpeed = 0
		local hrp = newChar:FindFirstChild("HumanoidRootPart")
		if hrp then initialSpeed = hrp.Velocity and Vector3.new(hrp.Velocity.X, 0, hrp.Velocity.Z).Magnitude or 0 end
		if initialSpeed <= 0.1 then
			switchTo("idle")
		elseif initialSpeed <= humanoid.WalkSpeed + 0.5 then
			switchTo("walk")
		else
			switchTo("run")
		end
	end)

	local function onCharRemoving()
		if runningConn then
			runningConn:Disconnect()
			runningConn = nil
		end
	end
	newChar.AncestryChanged:Connect(function()
		if not newChar:IsDescendantOf(game) then
			onCharRemoving()
		end
	end)
	humanoid.Died:Connect(onCharRemoving)

	player.Character = newChar
	frame.Visible = false

	selecting = false
end

for _, child in ipairs(frame:GetChildren()) do
	if child:IsA("ImageButton") then
		child.MouseButton1Click:Connect(function()
			swapToDino(child.Name)
		end)
	end
end
