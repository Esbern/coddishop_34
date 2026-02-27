# Copenhagen Coffeeshop Shelter Simulation - Design Clarification

## Overview
A multi-agent simulation of pedestrians in Copenhagen city center seeking shelter in coffeeshops during simulated rainfall. Agents communicate via MQTT to coordinate behavior based on weather conditions.

---

## 1. Core Components (Technical Definition)
Each agent type runs in its own notebook. Individual agents (people) are instances of a `Pedestrian` class stored in a list inside the pedestrians notebook.
Pedestrians subscribe directly to MQTT messages from the Weather agent.
There is no rain sensor agent; the Weather agent publishes the weather state itself.


### **Trigger Agent: Weather Simulator**
**Purpose:** Generate time-series weather events that drive the simulation.

**Current understanding:**
- Alternates between rain (10 seconds) and sun (20 seconds)
- Publishes weather state to all agents
- No wind, temperature, or other factors; rain is a binary state



---

### **Observer Agent: Sensor Network**
REMOVED. The Weather agent publishes rain state directly to all agents via MQTT. No separate sensor agent is needed. 

---

### **Control Agent: Removed**
Pedestrians calculate the nearest coffeeshop themselves. When rain starts, each `Pedestrian` instance reads the persistent coffeeshop locations from config and computes the nearest target using Euclidean distance.


### **Response Agent(s): Pedestrian Movement**
**Purpose:** Move agents to target locations or wander randomly based on weather.

**Behavior:**
- Up to 5 pedestrians, each with:
  - Random starting position in Copenhagen center
  - Random name (from a predefined list)
  - Assigned color for mapping
- **When sunny:** walk randomly in a random direction
- **When rain starts:** calculate nearest coffeeshop using Euclidean distance, then move toward it
- **While sheltering:** stay at coffeeshop for 0–5 seconds after rain ends, then resume random walking
- Movement speed: 5 km/h
- Coffeeshop capacity: max 3 people per shop
- All agents run as `Pedestrian` class instances in a single notebook (`agent_pedestrians.ipynb`)
- Publish location and status to MQTT topics every 0.5 seconds


---

### **Visualization Agent: Dashboard**
**Purpose:** Display real-time state to user.

**Current understanding:**
- Shows pedestrian locations on a map of Copenhagen center
- Shows coffeeshop locations
- Darkens map theme when raining (visual feedback)
- Uses `anymap-ts` for interactive mapping
-the map updates on mqtt events using the methods described in docs/maplibre_anymap.md


## 2. MQTT Topic Structure

Based on the components above, here's a proposed topic hierarchy:

```
simulated_city/
├── weather/
│   └── rain              (Trigger → all agents)
│       └── {raining: true/false, duration_sec: N}
│
├── pedestrians/
│   ├── person_{id}/
│   │   ├── location       (Pedestrian → Dashboard)
│   │   │   └── {lat, lon, name, color}
│   │   └── status         (Pedestrian → Dashboard)
│   │       └── {activity: "walking"|"sheltering", target_coffeeshop_id}
│
├── coffeeshops/
│   └── location           (Config/static, published once at startup)
│       └── [{id, lat, lon, name}, ...]
│
└── dashboard/
    └── state              (all agents → Dashboard)
        └── snapshot of entire simulation state
```

**Publishing agents:**
- **Trigger (Weather):** `weather/rain`
- **Response (Pedestrians):** `pedestrians/person_{id}/location`, `pedestrians/person_{id}/status`
- **Dashboard (Visualization):** subscribes to all above, publishes nothing

---

## 3. Configuration Parameters

### **Required in `config.yaml`:**

```yaml
# Simulation timing (seconds)
weather:
  rain_duration: 10
  sun_duration: 20

# Pedestrians
pedestrians:
  count: 5
  names: ["Alice", "Bob", "Charlie", "Diana", "Eve"]
  colors: ["#FF5733", "#33FF57", "#3357FF", "#FF33F5", "#F5FF33"]
  
# Movement
movement:
  step_size: 0.0001  # degrees per step (lat/lon)
  random_walk_speed: 0.0001  # degrees per second during sun
  shelter_walk_speed: 0.0002  # degrees per second toward coffeeshop
  
# Geography - Copenhagen center coordinates
geography:
  center_lat: 55.6761
  center_lon: 12.5683
  bounds:  # rough bounding box for random walking
    north: 55.6900
    south: 55.6600
    east: 12.6000
    west: 12.5300

# Coffeeshops (5 locations in central Copenhagen)
coffeeshops:
  - id: 1
    name: "Kaffe A"
    lat: 55.6761
    lon: 12.5683
  - id: 2
    name: "Kaffe B"
    lat: 55.6800
    lon: 12.5700
  - id: 3
    name: "Kaffe C"
    lat: 55.6720
    lon: 12.5620
  - id: 4
    name: "Kaffe D"
    lat: 55.6850
    lon: 12.5750
  - id: 5
    name: "Kaffe E"
    lat: 55.6700
    lon: 12.5500
```

---

## 4. Identified Ambiguities and Assumptions

NOTE: The items below are still unresolved. Please answer them before coding.

### **Critical** (must clarify before coding)

| Ambiguity | Your Assumption? | Recommendation |
|-----------|------------------|-----------------|
| **Pedestrian → Coffeeshop routing:** Do people walk via street network or "as the crow flies"? | Decided | Euclidean distance; beeline path (direct calculation, no street network) |
| **Notebook structure** | DECIDED | Each agent type has one notebook. Individual pedestrians are `Pedestrian` class instances stored in a list in `agent_pedestrians.ipynb`. |
| **Observer/Sensor agent** | DECIDED | Removed. Weather agent publishes `weather/rain` directly. |
| **Control agent** | DECIDED | Removed. Pedestrian class calculates nearest coffeeshop itself when rain starts using persistent coffeeshop locations. |
| **Sheltering behavior after rain stops** | Decided | Stay at coffeeshop for random 0–5 seconds, then resume random walking. |
| **Coffeeshop capacity** | DECIDED | Max 3 people per shop. |

### **Important** (decide but not blocking)

| Question | Notes |
|----------|-------|

| **Movement units:** Are speeds in meters/second (5 km/h) or degrees/step? | Conflict: both appear in the doc. Decide one. |
| **MQTT field names:** Should payloads use `coffeeshop_id` or `target_shelf_id`/`shelf_id`? | Conflict: both appear. Choose one name. |
| **Unclear rule:** "even if peopel can make it to the coffyshope the move" | Needs rewrite: this is not clear enough to implement. |
| **Map style toggle:** Does map *cycle* through themes or does user pick? | Suggest: Auto-cycle (dark when raining, light when sunny) |
| **Dashboard pause/resume:** Can user pause the sim? | Suggest: Not in MVP; can add later |
| **Statistics logging:** Do we track sheltering events? | Suggest: Dashboard displays live count; no file logging |

---

## 5. Realistic Starting Values

### **Timing (already in README)**
- Rain: 10 seconds
- Sun: 20 seconds
- **Total cycle:** 30 seconds (repeating)
- **Recommended simulation duration:** Run 2–3 cycles (60–90 seconds) for testing

### **Pedestrian Count and Speed**
- **Count:** 5 (as stated)
- **Random walk speed:** ~55m per step (real Copenhagen blocks are ~100m), so `step_size = 0.0009°` per step
- **Sheltering speed:** 1.5× random walk speed to ensure they reach shelter in ~15 seconds (shorter than rain duration)
- **Step interval:** 0.5 seconds (publish location every 0.5s)

### **Geography**
- **Center:** Nyhavn/Amalienborg area (55.6761°N, 12.5683°E)
- **Bounding box:** ~1 km² (0.01° ≈ 1 km)
  - North: 55.685°N
  - South: TODO: fix this value (65°S is invalid for Copenhagen)
  - East: 12.578°E
  - West: 12.558°E
- **Coffeeshops:** Spread across the box, realistic spacing (~300–500m apart)

### **MQTT**
- **Broker:** `localhost:1883` (development, no auth)
- **Base topic:** `simulated_city/`
- **Message rate:** Update agents every 0.5–1.0 seconds

---

## 6. Recommended Next Step

**Before coding, decide:**

1. ✅ **Pedestrian model:** Random walk vs. picking a random destination once per phase?
2. ✅ **Notebook split:** Separate notebooks for Weather, Control, Pedestrians, Dashboard or fewer?
3. ✅ **Sheltering logic:** Do pedestrians subscribe to raw weather, or to Control agent's targets?
4. ✅ **Map interactivity:** Show just positions + shops, or also show paths/timers?

Once you clarify these, I can help you design the MQTT flow diagram and notebook architecture.

---

