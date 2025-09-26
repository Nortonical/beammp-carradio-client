local imgui = require('ui_imgui')
local radio = require('radio')  -- Assume shared streams in radio.lua or inline

-- Local state
local currentStation = 0
local currentVolume = 50
local personalSound = nil  -- For MP3 fallback
local vehicleSounds = {}  -- {vehID = soundHandle}
local showUI = false
local currentVehicle = nil
local screenWidth, screenHeight = getScreenSize()

-- Shared streams (inline for simplicity)
local streams = {
    {name = "Rock FM", path = "mods/radio/sounds/rock.ogg", song = "Guitar Riff - Band X"},
    {name = "Pop Hits", path = "mods/radio/sounds/pop.ogg", song = "Catchy Tune - Artist Y"},
    {name = "Chill Vibes", path = "mods/radio/sounds/chill.ogg", song = "Relaxing Beat - DJ Z"}
}

-- UI Render Hook
function onUiRender()
    if showUI then
        imgui.SetNextWindowSize(imgui.ImVec2(300, 200), imgui.Cond_FirstUseEver)
        if imgui.Begin("Radio Controls", imgui.BoolPtr(showUI)) then
            imgui.Text("Select Station:")
            for i, stream in ipairs(streams) do
                if imgui.Button(stream.name) then
                    currentStation = i
                    if currentVehicle then
                        MP.TriggerServerEvent("radio:changeStation", i)
                    end
                    updateLocalPlayback()
                end
            end
            local volChanged = imgui.SliderInt("Volume", currentVolume, 0, 100)
            if volChanged then
                if currentVehicle then
                    MP.TriggerServerEvent("radio:changeVolume", currentVolume)
                end
                updateLocalPlayback()
            end
            if imgui.Button("Toggle Personal MP3") then
                togglePersonalMP3()
            end
        end
        imgui.End()
    end

    -- On-Screen Display
    local veh = be:getPlayerVehicle(0)
    if veh then
        currentVehicle = veh
        local station = veh:getField("radio:station") or 0
        if station > 0 and streams[station] then
            imgui.SetNextWindowPos(imgui.ImVec2(10, screenHeight - 100))
            imgui.Begin("Radio Display", imgui.WindowFlags_NoTitleBar + imgui.WindowFlags_NoResize)
            imgui.Text(streams[station].name .. " - " .. streams[station].song)
            imgui.End()
        end
    end
end
extensions.hook("onUiRender", onUiRender)

-- Toggle UI (bind to key, e.g., R)
function toggleRadioUI()
    showUI = not showUI
end
extensions.hook("onKeyUp", function(key) if key == "r" then toggleRadioUI() end end)

-- Update local playback (inside vehicle or personal)
function updateLocalPlayback()
    if personalSound then
        Engine.Audio.stop(personalSound)
        personalSound = nil
    end
    if currentVehicle and currentStation > 0 then
        local path = streams[currentStation].path
        personalSound = Engine.Audio.playLooping("AudioVehicle", path, true)  -- 3D enabled
        Engine.Audio.setVolume(personalSound, currentVolume / 100)
        Engine.Audio.setMinDistance(personalSound, 5)  -- Close range loud
        Engine.Audio.setMaxDistance(personalSound, 50)  -- Falloff
    elseif currentStation > 0 then
        -- Personal 2D
        local path = streams[currentStation].path
        personalSound = Engine.Audio.playLooping("AudioGui", path, false)  -- 2D
        Engine.Audio.setVolume(personalSound, currentVolume / 100)
    end
end

-- Personal MP3 toggle (fallback if no vehicle)
function togglePersonalMP3()
    if currentStation > 0 then
        if personalSound then
            Engine.Audio.stop(personalSound)
            personalSound = nil
        else
            updateLocalPlayback()
        end
    end
end

-- Receive sync from server (for other players' vehicles)
MP.RegisterEvent("radio:syncStation", function(vehID, station)
    local veh = be:getObjectByID(tonumber(vehID))
    if veh and station > 0 and streams[station] then
        local path = streams[station].path
        if vehicleSounds[vehID] then
            Engine.Audio.stop(vehicleSounds[vehID])
        end
        local sound = Engine.Audio.playLooping("AudioVehicle", path, true)  -- 3D
        vehicleSounds[vehID] = sound
        Engine.Audio.setMinDistance(sound, 5)
        Engine.Audio.setMaxDistance(sound, 50)
    else
        if vehicleSounds[vehID] then
            Engine.Audio.stop(vehicleSounds[vehID])
            vehicleSounds[vehID] = nil
        end
    end
end)

MP.RegisterEvent("radio:syncVolume", function(vehID, vol)
    local sound = vehicleSounds[vehID]
    if sound then
        Engine.Audio.setVolume(sound, tonumber(vol) / 100)
    end
end)

-- Update 3D positions per frame
function onUpdate(dtReal, dtSim, dtRaw)
    for vehID, sound in pairs(vehicleSounds) do
        local veh = be:getObjectByID(vehID)
        if veh and Engine.Audio.isPlaying(sound) then
            local pos = veh:getPosition()
            Engine.Audio.setPosition(sound, pos)  -- Attach to vehicle
            -- Optional: Adjust volume based on player distance
            local playerPos = be:getPlayerVehicle(0):getPosition()
            local dist = (pos - playerPos):len()
            local attenVol = math.max(0, 1 - dist / 50)  -- Linear falloff
            Engine.Audio.setVolume(sound, attenVol * (veh:getField("radio:volume") or 50) / 100)
        else
            Engine.Audio.stop(sound)
            vehicleSounds[vehID] = nil
        end
    end
end
extensions.hook("onUpdate", onUpdate)

-- Vehicle enter/exit (poll in onUpdate or hook if available)
-- Handled via currentVehicle in UI

-- Cleanup on disconnect
MP.RegisterEvent("onPlayerLeft", function(playerID)
    local vehID = getPlayerVehicleID(playerID)  -- Assume helper func
    if vehicleSounds[vehID] then
        Engine.Audio.stop(vehicleSounds[vehID])
        vehicleSounds[vehID] = nil
    end
end)
