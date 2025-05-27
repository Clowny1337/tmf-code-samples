# TMF Developer Portfolio – Code Samples

Hey, this is a quick portfolio showing some examples of my work. It includes

- FiveM Lua scripting (client/server)
- Frontend UI with React and Tailwind
- Backend route using Node.js and MySQL




## FiveM Lua Scripts

### Example 1: Dynamic Vehicle Tuning System  
Allows players to upgrade vehicle parts, saves upgrades per vehicle, and syncs data between client and server.

**File:** `client/vehicle_tuning.lua`

```lua
local currentVehicle = nil
local tuningData = {}

-- Request tuning data from server for a vehicle
function RequestTuningData(vehicleNetId)
    TriggerServerEvent("tuning:requestData", vehicleNetId)
end

-- Apply tuning upgrades visually
function ApplyTuningUpgrades(upgrades)
    local veh = currentVehicle
    if not veh then return end

    for part, level in pairs(upgrades) do
        -- Example: adjust vehicle mods dynamically
        SetVehicleMod(veh, part, level, false)
    end
end

-- Listen for tuning data from server
RegisterNetEvent("tuning:sendData", function(data)
    tuningData = data
    ApplyTuningUpgrades(tuningData)
end)

-- Open tuning menu (simplified example)
function OpenTuningMenu()
    local veh = GetVehiclePedIsIn(PlayerPedId(), false)
    if veh == 0 then return end

    currentVehicle = veh
    local netId = NetworkGetNetworkIdFromEntity(veh)
    RequestTuningData(netId)
    
    -- Here you would open your NUI menu and let player modify tuningData
end

-- Save tuning upgrades to server
function SaveTuningUpgrades(upgrades)
    local netId = NetworkGetNetworkIdFromEntity(currentVehicle)
    TriggerServerEvent("tuning:saveData", netId, upgrades)
end



local vehicleTuningData = {}

RegisterNetEvent("tuning:requestData")
AddEventHandler("tuning:requestData", function(netId)
    local src = source
    local data = vehicleTuningData[netId] or {}
    TriggerClientEvent("tuning:sendData", src, data)
end)

RegisterNetEvent("tuning:saveData")
AddEventHandler("tuning:saveData", function(netId, upgrades)
    vehicleTuningData[netId] = upgrades

    -- Optionally persist to database here

    -- Broadcast to all clients to sync visual upgrades
    TriggerClientEvent("tuning:sendData", -1, upgrades)
end)

--




React + Tailwind (Frontend UI)
Example 2: Interactive Staff Dashboard with Live Player List
This React component shows a live player list that updates using WebSockets, with controls to kick or ban players.

File: web/components/StaffDashboard.jsx

import React, { useEffect, useState } from "react";

export default function StaffDashboard() {
  const [players, setPlayers] = useState([]);

  useEffect(() => {
    const socket = new WebSocket("wss://yourserver.example/ws");

    socket.onmessage = (event) => {
      const data = JSON.parse(event.data);
      if (data.type === "player_list") {
        setPlayers(data.players);
      }
    };

    return () => socket.close();
  }, []);

  const kickPlayer = (id) => {
    fetch(`/api/players/${id}/kick`, { method: "POST" });
  };

  const banPlayer = (id) => {
    fetch(`/api/players/${id}/ban`, { method: "POST" });
  };

  return (
    <div className="p-6 bg-gray-100 rounded-lg shadow">
      <h2 className="text-xl font-bold mb-4">Live Player List</h2>
      <ul>
        {players.map((p) => (
          <li key={p.id} className="flex justify-between items-center mb-2 p-2 bg-white rounded shadow">
            <span>{p.name} (ID: {p.id})</span>
            <div>
              <button onClick={() => kickPlayer(p.id)} className="mr-2 px-3 py-1 bg-yellow-400 rounded">Kick</button>
              <button onClick={() => banPlayer(p.id)} className="px-3 py-1 bg-red-600 text-white rounded">Ban</button>
            </div>
          </li>
        ))}
      </ul>
    </div>
  );
}

--


Node.js Backend (Express + MySQL)
Example 3: Secure Player Action API with JWT and Role Checks
Handles authenticated requests for kicking and banning players, validating tokens and user roles.

File: web/api/routes/admin.js


const express = require("express");
const router = express.Router();
const db = require("../db");
const jwt = require("jsonwebtoken");

// Middleware to verify JWT and role
function authenticateAdmin(req, res, next) {
  const token = req.headers.authorization?.split(" ")[1];
  if (!token) return res.status(401).json({ error: "Unauthorized" });

  jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
    if (err || !decoded.roles.includes("admin")) {
      return res.status(403).json({ error: "Forbidden" });
    }
    req.user = decoded;
    next();
  });
}

// Kick player endpoint
router.post("/players/:id/kick", authenticateAdmin, async (req, res) => {
  const playerId = req.params.id;

  try {
    // Example: Insert kick record into DB, emit event to server, etc.
    await db.query("INSERT INTO kicks (player_id, admin_id, date) VALUES (?, ?, NOW())", [playerId, req.user.id]);
    
    // Emit event to FiveM server (pseudo-code)
    // globalEmitter.emit("kickPlayer", playerId);

    res.json({ success: true, message: `Player ${playerId} kicked` });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Server error" });
  }
});

// Ban player endpoint
router.post("/players/:id/ban", authenticateAdmin, async (req, res) => {
  const playerId = req.params.id;

  try {
    await db.query("INSERT INTO bans (player_id, admin_id, date) VALUES (?, ?, NOW())", [playerId, req.user.id]);
    // Emit ban event to server here

    res.json({ success: true, message: `Player ${playerId} banned` });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Server error" });
  }
});

module.exports = router;
