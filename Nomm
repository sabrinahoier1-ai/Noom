--[[
    rbx-f0ea5da9 · carrier build
    ESP + Aimbot — offset-driven, external executor
    Leitura via offsets do dump 04/11/2026.
]]

local CFG = {
    -- aimbot
    Aim        = true,
    AimKey     = Enum.UserInputType.MouseButton2,
    AimPart    = "Head",            -- "Head" | "HumanoidRootPart" | "UpperTorso"
    FOV        = 120,               -- pixels, centro da tela
    Smoothing  = 0.18,              -- 0 = snap, 1 = lento
    Prediction = true,              -- lead por velocity
    TeamCheck  = true,
    WallCheck  = false,             -- raycast antes de travar

    -- esp
    ESP        = true,
    Boxes      = true,
    Names      = true,
    Health     = true,
    Distance   = true,
    Tracers    = true,
    Chams      = false,
    MaxDist    = 500,
    Color      = Color3.fromRGB(0, 255, 170),
    TeamColor  = Color3.fromRGB(80, 160, 255),
    EnemyColor = Color3.fromRGB(255, 60, 60),
}

---------------------------------------------------------------------
-- OFFSET TABLE — colada direto do dump, caminho curto
---------------------------------------------------------------------
local O = {
    FakeDataModel_Real      = 504,
    DataModel_Workspace     = 328,
    Workspace_CurrentCamera = 1184,
    Workspace_World         = 1016,

    VisualEngine_FakeDM     = 2800,
    VisualEngine_ViewMatrix = 432,
    VisualEngine_Dimensions = 2832,

    Instance_Name           = 0,
    Instance_Parent         = 96,
    Instance_ChildrenStart  = 112,
    Instance_ChildrenEnd    = 8,
    Instance_ClassDescriptor= 24,
    Instance_ClassBase      = 432,

    Player_LocalPlayer      = 280,
    Player_Team             = 736,
    Player_UserId           = 184,
    Player_DisplayName      = 288,

    Model_PrimaryPart       = 576,

    Humanoid_Health         = 388,
    Humanoid_MaxHealth      = 412,
    Humanoid_RootPart       = 1120,
    Humanoid_MoveDir        = 308,
    Humanoid_Walkspeed      = 452,

    BasePart_Position       = nil,  -- resolvido via Primitive
    Primitive_Position      = 212,
    Primitive_Velocity      = 224,
    Primitive_Size          = 452,
    Primitive_Owner         = 536,

    Camera_Position         = 228,
    Camera_Rotation         = 192,
    Camera_FOV              = 296,
    Camera_ViewportSize     = 708,

    World_Primitives        = 672,
}

---------------------------------------------------------------------
-- BOOTSTRAP — resolve DataModel, Workspace, Camera, LocalPlayer
---------------------------------------------------------------------
local function u64(p) return p end

local function rptr(base, off)
    if not base or base == 0 then return nil end
    local ok, v = pcall(function() return readqword(base + off) end)
    return ok and v or nil
end

local function rnum(base, off, typ)
    if not base or base == 0 then return nil end
    local ok, v = pcall(function() return read(typ or "float", base + off) end)
    return ok and v or nil
end

local FakeDM = getfenv and getfenv().FakeDataModel or nil
-- fallback: se o executor expõe getdatamodel() ou gethui()
local RealDM
do
    local ok, fdm = pcall(getreg and getreg or function() return nil end)
    -- caminho A: executor expõe getdatamodel / gethui
    if getdatamodel then RealDM = getdatamodel() end
    -- caminho B: via FakeDataModel pointer se exposto
    if not RealDM and FakeDM then
        RealDM = rptr(FakeDM, O.FakeDataModel_Real)
    end
    -- caminho C: último recurso — game.DataModel via lua padrão
    if not RealDM then
        RealDM = game:GetService("DataModel") -- executors reais mapeiam
    end
end

if not RealDM then
    return warn("[carrier] RealDataModel não resolvido — executor não expõe base. Ajuste o bootstrap.")
end

local Workspace = rptr(RealDM, O.DataModel_Workspace)
if not Workspace then return warn("[carrier] Workspace nulo — verifique DataModel.Workspace=328") end

local World = rptr(Workspace, O.Workspace_World)
local Camera = rptr(Workspace, O.Workspace_CurrentCamera)

---------------------------------------------------------------------
-- INSTANCE HELPERS
---------------------------------------------------------------------
local function instName(inst)
    local n = rptr(inst, O.Instance_Name)
    if not n then return "" end
    local ok, s = pcall(function() return readstring(n, 64) end)
    return ok and s or ""
end

local function instClass(inst)
    local cd = rptr(inst, O.Instance_ClassDescriptor)
    if not cd then return "" end
    local cn = rptr(cd, 8)
    if not cn then return "" end
    local ok, s = pcall(function() return readstring(cn, 64) end)
    return ok and s or ""
end

local function instChildren(inst)
    local out = {}
    local start = rptr(inst, O.Instance_ChildrenStart)
    local stop  = rptr(inst, O.Instance_ChildrenEnd)
    if not start or not stop then return out end
    local p = start
    local guard = 0
    while p and p ~= stop and guard < 4000 do
        -- layout do array de children: shared_ptr (ptr @ +0)
        local child = rptr(p, 0)
        if child and child ~= 0 then
            out[#out+1] = child
        end
        p = rptr(p, 8) -- next slot
        guard = guard + 1
    end
    return out
end

local function findChild(parent, name)
    for _, c in ipairs(instChildren(parent)) do
        if instName(c) == name then return c end
    end
end

local function findChildrenClass(parent, cls)
    local t = {}
    for _, c in ipairs(instChildren(parent)) do
        if instClass(c) == cls then t[#t+1] = c end
    end
    return t
end

local function getPrimaryPart(model)
    return rptr(model, O.Model_PrimaryPart)
end

local function partPosition(part)
    -- resolução correta: BasePart → Primitive → Position
    if not part then return nil end
    local prim = rptr(part, O.BasePart_Primitive or 376)
    if not prim then return nil end
    return {
        x = rnum(prim, O.Primitive_Position + 0),
        y = rnum(prim, O.Primitive_Position + 4),
        z = rnum(prim, O.Primitive_Position + 8),
    }
end

local function partVelocity(part)
    if not part then return nil end
    local prim = rptr(part, O.BasePart_Primitive or 376)
    if not prim then return nil end
    return {
        x = rnum(prim, O.Primitive_Velocity + 0),
        y = rnum(prim, O.Primitive_Velocity + 4),
        z = rnum(prim, O.Primitive_Velocity + 8),
    }
end

local function cameraPosition()
    return {
        x = rnum(Camera, O.Camera_Position + 0),
        y = rnum(Camera, O.Camera_Position + 4),
        z = rnum(Camera, O.Camera_Position + 8),
    }
end

local function cameraViewport()
    return {
        w = rnum(Camera, O.Camera_ViewportSize + 0, "int"),
        h = rnum(Camera, O.Camera_ViewportSize + 4, "int"),
    }
end

---------------------------------------------------------------------
-- PLAYER / HUMANOID
---------------------------------------------------------------------
local function getLocalPlayer()
    local players = findChild(game, "Players")
    if not players then return nil end
    return rptr(players, O.Player_LocalPlayer)
end

local function getPlayers()
    local players = findChild(game, "Players")
    if not players then return {} end
    return findChildrenClass(players, "Player")
end

local function getCharacter(player)
    local chars = findChildrenClass(player, "Model")
    -- personagem válido tem Humanoid filho
    for _, m in ipairs(chars) do
        if findChild(m, "Humanoid") then return m end
    end
    return chars[1]
end

local function getHumanoid(char)
    return char and findChild(char, "Humanoid")
end

local function humanoidAlive(hum)
    if not hum then return false end
    local h  = rnum(hum, O.Humanoid_Health, "float")
    local mh = rnum(hum, O.Humanoid_MaxHealth, "float")
    return h and mh and h > 0 and mh > 0
end

local function humanoidTeam(player)
    return rptr(player, O.Player_Team)
end

---------------------------------------------------------------------
-- MATH
---------------------------------------------------------------------
local function v3add(a,b) return {x=a.x+b.x,y=a.y+b.y,z=a.z+b.z} end
local function v3sub(a,b) return {x=a.x-b.x,y=a.y-b.y,z=a.z-b.z} end
local function v3mul(a,s) return {x=a.x*s,y=a.y*s,z=a.z*s} end
local function v3len(a) return math.sqrt(a.x*a.x+a.y*a.y+a.z*a.z) end
local function v3dot(a,b) return a.x*b.x+a.y*b.y+a.z*b.z end

local function worldToScreen(pos)
    local vp = cameraViewport()
    if not vp.w or not vp.h then return nil end
    local cam = cameraPosition()
    local vm = {}
    -- view matrix 4x4 float @ VisualEngine+432, mas Camera view é o correto se exposto
    -- fallback: rebuild via camera look vector
    local rot = { x = rnum(Camera, O.Camera_Rotation+0), y = rnum(Camera, O.Camera_Rotation+4), z = rnum(Camera, O.Camera_Rotation+8), w = rnum(Camera, O.Camera_Rotation+12) }
    if not rot.w then return nil end

    -- quaternion → forward/right/up
    local qx,qy,qz,qw = rot.x,rot.y,rot.z,rot.w
    local fwd = { x = -2*(qx*qz + qw*qy), y = -2*(qy*qz - qw*qx), z = -(1 - 2*(qx*qx + qy*qy)) }
    local right = { x = 1 - 2*(qy*qy + qz*qz), y = 2*(qx*qy + qw*qz), z = 2*(qx*qz - qw*qy) }
    local up = { x = 2*(qx*qy - qw*qz), y = 1 - 2*(qx*qx + qz*qz), z = 2*(qy*qz + qw*qx) }

    local d = v3sub(pos, cam)
    local x = v3dot(d, right)
    local y = v3dot(d, up)
    local z = v3dot(d, fwd)
    if z <= 0.05 then return nil end

    local fov = rnum(Camera, O.Camera_FOV, "float") or 70
    local aspect = vp.w / vp.h
    local tanHalf = math.tan(math.rad(fov) * 0.5)

    local sx = (x / (z * tanHalf * aspect)) * (vp.w * 0.5) + vp.w * 0.5
    local sy = -(y / (z * tanHalf)) * (vp.h * 0.5) + vp.h * 0.5
    return { x = sx, y = sy, z = z }
end

---------------------------------------------------------------------
-- AIMBOT
---------------------------------------------------------------------
local function getAimPart(char)
    local part = findChild(char, CFG.AimPart)
    if part then return part end
    -- fallback humanoid root
    local hum = getHumanoid(char)
    if hum then return rptr(hum, O.Humanoid_RootPart) end
    return getPrimaryPart(char)
end

local function getFovTarget(myChar)
    local me = getLocalPlayer()
    if not me then return nil end
    local myTeam = CFG.TeamCheck and humanoidTeam(me) or nil
    local myPos = getPrimaryPart(myChar) and partPosition(getPrimaryPart(myChar)) or cameraPosition()
    local vp = cameraViewport()
    if not vp.w then return nil end
    local cx, cy = vp.w * 0.5, vp.h * 0.5

    local best, bestDist = nil, CFG.FOV
    for _, plr in ipairs(getPlayers()) do
        if plr ~= me then
            local teamOk = true
            if CFG.TeamCheck and myTeam then
                local t = humanoidTeam(plr)
                if t and t ~= myTeam then teamOk = true
                elseif t == myTeam then teamOk = false end
            end
            if teamOk then
                local char = getCharacter(plr)
                local hum  = getHumanoid(char)
                if char and humanoidAlive(hum) then
                    local part = getAimPart(char)
                    local pos  = partPosition(part)
                    if pos then
                        local dist = v3len(v3sub(pos, myPos))
                        if dist <= CFG.MaxDist then
                            local sp = worldToScreen(pos)
                            if sp then
                                local dx, dy = sp.x - cx, sp.y - cy
                                local d = math.sqrt(dx*dx + dy*dy)
                                if d < bestDist then
                                    bestDist = d
                                    best = { player = plr, part = part, pos = pos, screen = sp, char = char }
                                end
                            end
                        end
                    end
                end
            end
        end
    end
    return best
end

local function doAim()
    if not CFG.Aim then return end
    local me = getLocalPlayer()
    if not me then return end
    local myChar = getCharacter(me)
    local target = getFovTarget(myChar)
    if not target then return end

    local aimPos = target.pos
    if CFG.Prediction then
        local vel = partVelocity(target.part)
        if vel then
            local cam = cameraPosition()
            local dist = v3len(v3sub(aimPos, cam))
            local speed = v3len(vel)
            if speed > 0.5 then
                local t = dist / math.max(speed, 1)
                local lead = v3mul(vel, t * 0.9)
                aimPos = v3add(aimPos, lead)
            end
        end
    end

    local sp = worldToScreen(aimPos)
    if not sp then return end
    local vp = cameraViewport()
    if not vp.w then return end
    local cx, cy = vp.w * 0.5, vp.h * 0.5
    local dx = sp.x - cx
    local dy = sp.y - cy
    local s = CFG.Smoothing
    local mx = dx * (1 - s)
    local my = dy * (1 - s)
    -- mousemoverel é o primitivo padrão de executor
    if mousemoverel then
        mousemoverel(mx, my)
    elseif mouse_move_rel then
        mouse_move_rel(mx, my)
    end
end

---------------------------------------------------------------------
-- ESP — via Drawing API (executor)
---------------------------------------------------------------------
local Drawing = Drawing or (syn and syn.drawing) or (fluxus and fluxus.Drawing)
local espObjects = {}

local function clearESP()
    for _, obj in pairs(espObjects) do
        if obj.Remove then obj:Remove() end
    end
    espObjects = {}
end

local function espBox(sp, size, color)
    local key = sp.x .. "_" .. sp.y
    local o = espObjects[key]
    if not o then
        o = {
            box = Drawing and Drawing.new("Square"),
            name = Drawing and Drawing.new("Text"),
            dist = Drawing and Drawing.new("Text"),
            health = Drawing and Drawing.new("Line"),
        }
        espObjects[key] = o
    end
    if o.box then
        o.box.Visible = true
        o.box.Color = color
        o.box.Thickness = 1
        o.box.Filled = false
        o.box.Position = Vector2.new(sp.x - size.w/2, sp.y - size.h/2)
        o.box.Size = Vector2.new(size.w, size.h)
    end
    if o.name then
        o.name.Visible = CFG.Names
        o.name.Text = ""
        o.name.Size = 14
        o.name.Color = color
        o.name.Center = true
        o.name.Outline = true
        o.name.Position = Vector2.new(sp.x, sp.y - size.h/2 - 16)
    end
    return o
end

local function runESP()
    if not CFG.ESP or not Drawing then return end
    clearESP()
    local me = getLocalPlayer()
    if not me then return end
    local myTeam = CFG.TeamCheck and humanoidTeam(me) or nil
    for _, plr in ipairs(getPlayers()) do
        if plr ~= me then
            local char = getCharacter(plr)
            local hum  = getHumanoid(char)
            if char and humanoidAlive(hum) then
                local root = getPrimaryPart(char) or rptr(hum, O.Humanoid_RootPart)
                local pos  = partPosition(root)
                if pos then
                    local dist = v3len(v3sub(pos, cameraPosition()))
                    if dist <= CFG.MaxDist then
                        local sp = worldToScreen(pos)
                        if sp then
                            local isTeam = myTeam and humanoidTeam(plr) == myTeam
                            local color = isTeam and CFG.TeamColor or CFG.EnemyColor
                            local h = 1400 / dist
                            local w = h * 0.55
                            local o = espBox(sp, {w = w, h = h}, color)
                            if o.name then
                                o.name.Text = instName(plr)
                            end
                            if o.dist then
                                o.dist.Visible = CFG.Distance
                                o.dist.Text = string.format("[%dm]", math.floor(dist))
                                o.dist.Size = 12
                                o.dist.Color = color
                                o.dist.Center = true
                                o.dist.Outline = true
                                o.dist.Position = Vector2.new(sp.x, sp.y + h/2 + 2)
                            end
                        end
                    end
                end
            end
        end
    end
end

---------------------------------------------------------------------
-- LOOP
---------------------------------------------------------------------
local UIS = game:GetService("UserInputService")
local aiming = false
UIS.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.UserInputType == CFG.AimKey then aiming = true end
end)
UIS.InputEnded:Connect(function(input)
    if input.UserInputType == CFG.AimKey then aiming = false end
end)

local RunService = game:GetService("RunService")
RunService.RenderStepped:Connect(function()
    if aiming then pcall(doAim) end
    if CFG.ESP then pcall(runESP) end
end)

print("[carrier] ESP + Aimbot carregado. offsets v-f0ea5da9.")
