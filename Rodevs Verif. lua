Discord: jubileuv2
Roblox: 27924383 👍

local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Framework = ReplicatedStorage.Framework
local Packages = Framework.Packages

local Assets = script.Assets

local Settings = require(script.Settings)

local Replicator = require(Packages.Replicator)
local Bouncy = require(Packages.Bouncy)

local Handler = Replicator.GetReplicatedTable()

local ToolName = "Grenade"
local ThrowCooldown = 0.35

local EnabledGrenades = {}
local Guaranteed = {}
local LastThrow = {}

local ClientGrenadesEnabled = false
local ClientSetupComplete = false
local ClientConnections = {}

--disconnecting through one helperhelps keep stuff easy 👍
--this matters cuz the listeners all have different lifetimes
local function DisconnectConnections(Connections : {RBXScriptConnection})
	for _, Connection in Connections do
		if Connection.Connected then
			Connection:Disconnect()
		end
	end

	table.clear(Connections)
end

--gets the humanoid through one path so every server/client validity check follows the same rules 👍
--also means I don't have to rewrite the same alive checks everywhere
local function GetAliveHumanoid(Character : Model?)
	if not Character then return nil end

	local Humanoid = Character:FindFirstChildOfClass("Humanoid")
	if not Humanoid then return nil end
	if Humanoid.Health <= 0 then return nil end

	return Humanoid
end

--only the horizontal camera direction controls aim because vertical launch power comes from Settings.Velocity 👍
--this stops the client camera y from also controlling how much upwards force the grenade gets
local function GetHorizontalDirection(Direction : Vector3, Fallback : Vector3)
	local Horizontal = Vector3.new(
		Direction.X,
		0,
		Direction.Z
	)

	if Horizontal.Magnitude <= 0.001 then
		Horizontal = Vector3.new(
			Fallback.X,
			0,
			Fallback.Z
		)
	end

	if Horizontal.Magnitude <= 0.001 then
		return nil
	end

	return Horizontal.Unit
end

--the settings velocity still decides how strong the throw is while the camera only decides where it goes 👍
--using the horizontal magnitude also means the old Vector3 velocity setting can still be used
local function GetThrowForce(Direction : Vector3)
	local ConfiguredHorizontal = Vector3.new(
		Settings.Velocity.X,
		0,
		Settings.Velocity.Z
	)

	local HorizontalSpeed = ConfiguredHorizontal.Magnitude
	local VerticalSpeed = Settings.Velocity.Y

	local HorizontalForce = Direction * HorizontalSpeed
	local VerticalForce = Vector3.new(
		0,
		VerticalSpeed,
		0
	)

	return HorizontalForce + VerticalForce
end

--guarantees player has tool 👍
--the permission and the actual tool are separate cuz the tool disappears on respawn while permission shouldn't
local function ToolGuaranteed(Player : Player, IsGuaranteed : boolean)
	if IsGuaranteed then
		local ExistingConnections = Guaranteed[Player]

		--tool granting is its own function because it has to happen both right now and after every respawn so plr doesn't lose their tools 👍
		local function GrantTool(Character : Model)
			local Backpack = Player:FindFirstChildOfClass("Backpack")
			if not Backpack then return end

			--checking both places matters because equipped tools are under Character and unequipped ones are in Backpack 👍
			--otherwise respawning or equipping at the wrong time could result in multiple grenade tools
			local Existing = Character:FindFirstChild(ToolName)
				or Backpack:FindFirstChild(ToolName)

			if Existing then return end

			local Tool = Assets.Grenade:Clone()
			Tool.Parent = Backpack
		end

		local Character = Player.Character
			or Player.CharacterAdded:Wait()

		--calling EnableForPlayer(true) again should repair a missing tool without making an extra conn 👍
		if ExistingConnections then
			GrantTool(Character)
			return
		end

		local Connections = {}

		Guaranteed[Player] = Connections

		GrantTool(Character)

		--the server permission survives respawns, so the tool has to be recreated for the new character 👍
		Connections[#Connections + 1] = Player.CharacterAdded:Connect(function(NewCharacter)
			GrantTool(NewCharacter)
		end)
	else
		local Connections = Guaranteed[Player]
		if not Connections then return end

		--disconnecting before destroying the tool makes sure a respawn during cleanup can't just give it back 👍
		DisconnectConnections(Connections)

		Guaranteed[Player] = nil

		local Character = Player.Character
		local Backpack = Player:FindFirstChildOfClass("Backpack")

		local CharacterTool = Character
			and Character:FindFirstChild(ToolName)

		local BackpackTool = Backpack
			and Backpack:FindFirstChild(ToolName)

		if CharacterTool then
			CharacterTool:Destroy()
		end

		if BackpackTool and BackpackTool ~= CharacterTool then
			BackpackTool:Destroy()
		end
	end
end

--these tables use plr objects as keys cuzu it's easier to be cleaned when the player leaves 👍
--otherwise it'd have a mem leak because it wouldn't get removed
Players.PlayerRemoving:Connect(function(Player)
	EnabledGrenades[Player] = nil
	LastThrow[Player] = nil

	local Connections = Guaranteed[Player]

	if Connections then
		DisconnectConnections(Connections)
	end

	Guaranteed[Player] = nil
end)

--all explosion rays go through this so they always use exactly the same configured range 👍
local function DoRay(
	Start : Vector3,
	Direction : Vector3,
	Params : RaycastParams
)
	return workspace:Raycast(
		Start,
		Direction.Unit * Settings.ExplosionRange,
		Params
	)
end

--damage is done after raycasting instead of inside the ray loop 👍
--grenades like this gotta gather how many hits the plr had from rays and give them the big dmg
local function DamageHitCharacters(Hit : {[Model] : number})
	for Character, Count in Hit do
		local Humanoid = Character:FindFirstChildOfClass("Humanoid")

		if not Humanoid then continue end
		if Humanoid.Health <= 0 then continue end

		local ToTake = Count * Settings.DamagePerRay

		Humanoid:TakeDamage(
			math.round(ToTake)
		)
	end
end

--explode grenade and damage players 👍
local function ExplodeGrenade(Object : Model)
	if not Object then return end
	if not Object.Parent then return end

	local Start = Object:GetPivot().Position
	local Degrees = Settings.Degrees

	--the grenade itself needs to be ignored or rays could instantly hit the grenade instead of leaving 👍
	local Arguments = RaycastParams.new()

	Arguments.FilterType = Enum.RaycastFilterType.Exclude
	Arguments:AddToFilter(Object)

	--roblox Explosion is only being used for its explosion visual here 👍
	--all its built-in stuff are disabled because the ray system handles that the way I intended it to. (Closer = more dmg, farther = more damage), fragment grenade blablabla
	local Explosion = Instance.new("Explosion")

	Explosion.BlastPressure = 0
	Explosion.BlastRadius = 0
	Explosion.DestroyJointRadiusPercent = 0
	Explosion.ExplosionType = Enum.ExplosionType.NoCraters

	Explosion.Position = Start
	Explosion.Parent = workspace

	--each ray that can actually reach a player adds to how exposed they were to the explosion (or closer) 👍
	--walls naturally block rays, and players taking up more of the sampled area get hit by more rays, and players block other plrs, REALISM!
	local Hit : {[Model] : number} = {}

	--Degrees is basically a setting for the explosion, can be tweaked for performance too 👍
	--smaller steps = a lot more rays and better sampling, but the ray count increases really fast cuz circle
	local XStep = Degrees.X
	local YStep = Degrees.Y

	--a <=0 step would make the loop not valid, so someone bad setting settings shouldn't break the whole thing 👍
	if XStep <= 0 or YStep <= 0 then
		Object:Destroy()
		return
	end

	--keeping my original 0-360 thingy here also means the DamagePerRay tuning still behaves the same 👍
	for X = XStep, 360, XStep do
		for Y = YStep, 360, YStep do
			local Rotation = CFrame.fromEulerAnglesYXZ(
				math.rad(X),
				math.rad(Y),
				0
			)

			local Direction = Rotation.LookVector

			local Info = DoRay(
				Start,
				Direction,
				Arguments
			)

			if not Info then continue end

			local Instance = Info.Instance

			--raycasts can hit accessories or bodyparts so I walk upwards until I find their character model 
			--or else it wouldn't work properly

			local Character = Instance:FindFirstAncestorWhichIsA("Model")

			if not Character then continue end

			--only actual player characters count because this grenade system isn't supposed to damage random NPCs 👍
			local ValidPlayer = Players:GetPlayerFromCharacter(Character)

			if not ValidPlayer then continue end

			Hit[Character] = (Hit[Character] or 0) + 1
		end
	end

	DamageHitCharacters(Hit)

	Object:Destroy()
end

function Handler.SetupGrenades()
	--this function deals with LocalPlayer input/camera stuff so there is no reason to run it on the server 👍
	if not RunService:IsClient() then return end

	--if this ever gets called twice I don't want two sets of Activated connections throwing two grenades 👍
	if ClientSetupComplete then return end

	ClientSetupComplete = true

	local Player = Players.LocalPlayer

	local ToolConnections = {}
	local CharacterConnections = {}

	--tool connections only belong to whichever grenade is currently being handled cuz of what I said above 👍
	local function ClearToolConnections()
		DisconnectConnections(ToolConnections)
	end

	--character connections need to die when the character does or new respawns would slowly stack conns👍
	local function ClearCharacterConnections()
		DisconnectConnections(CharacterConnections)
	end

	--this is where the grenade tool gets input 👍
	--the client detects activation, but it still can't directly damage or check anything because that's smth server related
	local function HandleGrenade(Tool : Tool?)
		if not Tool then return end
		if not Tool:IsA("Tool") then return end
		if Tool.Name ~= ToolName then return end

		ClearToolConnections()

		ToolConnections[#ToolConnections + 1] = Tool.Activated:Connect(function()
			--this basic state just stops pointless requests, but the server still checks the real permission anyway 👍
			--gotta be secure yk

			if not ClientGrenadesEnabled then return end

			local Character = Player.Character
			local Humanoid = GetAliveHumanoid(Character)

			if not Humanoid then return end

			Handler.ThrowGrenade()
		end)
	end

	--equipping a tool parents it to Character, so childadded tells me when the grenade gets equipped 👍
	--knowing that childremoved should be really obvious, yeah it's the plr unequipping the grenade
	local function HandleCharacter(Character : Model)
		ClearToolConnections()
		ClearCharacterConnections()

		CharacterConnections[#CharacterConnections + 1] = Character.ChildAdded:Connect(function(Child : Instance)
			if not Child:IsA("Tool") then return end
			if Child.Name ~= ToolName then return end

			HandleGrenade(Child)
		end)

		CharacterConnections[#CharacterConnections + 1] = Character.ChildRemoved:Connect(function(Child : Instance)
			if not Child:IsA("Tool") then return end
			if Child.Name ~= ToolName then return end

			ClearToolConnections()
		end)

		--setup could happen while the grenade is already equipped, so waiting for it being added isn't enough in that edgecase👍
		local ExistingTool = Character:FindFirstChild(ToolName)

		if ExistingTool and ExistingTool:IsA("Tool") then
			HandleGrenade(ExistingTool)
		end
	end

	--handles the character that already exists when the module starts 👍
	local Character = Player.Character
		or Player.CharacterAdded:Wait()

	HandleCharacter(Character)

	--this connection is to survive respawns 👍
	ClientConnections[#ClientConnections + 1] = Player.CharacterAdded:Connect(function(NewCharacter)
		HandleCharacter(NewCharacter)
	end)
end

Handler.SetupGrenades()

--for reference, my Replicator is basically my networking wrapper so I don't need raw remote events everywhere 👍
--client only provides stuff that only the client knows, server handles important stuff

--[[Client and server]]

Handler:CreateFunction("Client-Server", {
	Name = "ThrowGrenade";
	Callback = "StoreLatest";
	Reference = script;

	Server = function(
		Player : Player,
		Direction : Vector3
	)
		--typechecking doesn't magically validate values sent 👍
		--so typeof is still needed here because an exploiter can send whatever they want
		if typeof(Direction) ~= "Vector3" then return end

		--permission lives on the server so calling ThrowGrenade from an exploit doesn't enable grenades 👍
		local Valid = EnabledGrenades[Player]

		if not Valid then return end

		local Character = Player.Character
		local Humanoid = GetAliveHumanoid(Character)

		if not Character then return end
		if not Humanoid then return end

		--Tool.Activated normally limits how fast this gets called, but exploiters don't have to use Tool.Activated 👍
		--having a second cooldown here means the server still decides the actual throw rate
		local Now = os.clock()

		local Cooldown = Settings.ThrowCooldown or ThrowCooldown

		local PreviousThrow = LastThrow[Player] or -math.huge

		if Now - PreviousThrow < Cooldown then
			return
		end

		local CharPivot = Character:GetPivot()

		--camera aim can point nearly straight up/down, which has almost no usable horizontal direction 👍
		--in that case the character LookVector acts as a safe fallback instead
		local HorizontalDirection = GetHorizontalDirection(
			Direction,
			CharPivot.LookVector
		)

		if not HorizontalDirection then return end

		--cooldown is only consumed once the request has actually passed every important validation 👍
		LastThrow[Player] = Now

		local Object = Assets.WorldModel.Object:Clone()

		local Offset = Settings.Offset

		--spawn offset stays relative to the character so rotating the player rotates the grenade spawn point too 👍
		local StartingCFrame = CharPivot
			* CFrame.new(Offset)

		--camera controls where the horizontal throw goes while Settings.Velocity controls how strong/high it is 👍
		--this fixes my old version where Direction got sent and modified but then literally wasn't used lol
		local Force = GetThrowForce(
			HorizontalDirection
		)

		Object:PivotTo(StartingCFrame)
		Object.Parent = workspace

		--Bouncy owns the projectile simulation so this script doesn't need another custom per-frame physics loop 👍
		--the grenade system can just worry about permissions, networking, spawning and exploding
		Bouncy.Launch(
			Object,
			Force,
			1
		)

		--the fuse runs on the server so a client can't change its own timer and decide when damage should happen 👍
		task.wait(
			Settings.TimeBeforeBoom
		)

		--something else could delete the grenade during the fuse and exploding a dead object would be stupid 👍
		if not Object.Parent then return end

		ExplodeGrenade(Object)
	end,

	Client = function()
		--CurrentCamera is client stuff, which is why aim direction has to originate here 👍
		--only direction gets sent though cuz client in this case should never choose stats
		local Camera = workspace.CurrentCamera

		if not Camera then return end

		local Direction = Camera.CFrame.LookVector

		return {
			Arguments = {
				Direction
			}
		}
	end,
})

--[[Client and server]]

Handler:CreateFunction("Server-Client", {
	Name = "EnableForPlayer";
	Callback = "StoreLatest";
	Reference = script;

	Server = function(
		Player : Player,
		Enabled : boolean
	)
		if typeof(Enabled) ~= "boolean" then
			return
		end

		--permission and the physical tool are changed together so inventory matches what the server actually allows 👍
		EnabledGrenades[Player] = Enabled

		--old cooldown state doesn't matter anymore once grenades are disabled 👍
		if not Enabled then
			LastThrow[Player] = nil
		end

		ToolGuaranteed(
			Player,
			Enabled
		)

		--only this player needs their local permission state because nobody else handles this player's grenade input 👍
		return {
			Arguments = {
				Enabled
			},

			FiredTo = {
				Player
			}
		}
	end,

	Client = function(Enabled : boolean)
		--this is only to prevent client from firing the remote for non-exploiter ppl 👍
		--server has its own var table and still verifies every throw
		ClientGrenadesEnabled = Enabled
	end,
})

return Handler :: typeof({
	--[[SERVER ONLY. Enables or disables grenade usage for a specific player.]]
	EnableForPlayer = function(
		Player : Player,
		Enabled : boolean
	)
	end,

	--[[CLIENT ONLY. Uses the player's camera direction to throw a grenade.]]
	ThrowGrenade = function()
	end,
})
