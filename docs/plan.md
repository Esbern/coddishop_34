# Copenhagen Coffeeshop Shelter Simulation - Implementation Plan

Based on the clarified design in [concepts.md](concepts.md), this document outlines a phased implementation approach.

---

## **Phase 1: Weather Agent (Minimal MQTT Publisher)**

### Goal
Create a simple weather simulator that publishes rain events to MQTT on a timer.

### New Files
- `notebooks/agent_weather.ipynb`
  - Cell 1: Markdown (overview)
  - Cell 2: Import `simulated_city.config`, `simulated_city.mqtt`, `time`, `json`
  - Cell 3: Connect to MQTT using `MqttConnector` + `MqttPublisher`
  - Cell 4: Infinite loop alternating rain (10s) and sun (20s), publishing to `simulated_city/weather/rain`
  - Cell 5: Graceful shutdown (disconnect)

### Tests/Verification
```bash
# Terminal 1: Start a local MQTT broker (if not running)
mosquitto -v

# Terminal 2: Subscribe to weather topic to see messages
mosquitto_sub -h localhost -t "simulated_city/weather/rain"

# Terminal 3: Run the weather notebook
python -m jupyterlab notebooks/agent_weather.ipynb
```

Expected output in Terminal 2:
```json
{"raining": false, "duration_sec": 20}
{"raining": true, "duration_sec": 10}
{"raining": false, "duration_sec": 20}
...
```

### Investigation Before Next Phase
- **MQTT connection:** Does `MqttConnector` work with your local broker?
- **Timing accuracy:** Are the 10s/20s intervals accurate?
- **Message format:** Is the JSON payload readable?

### Dependencies
Already installed (from `pyproject.toml`):
- `paho-mqtt>=2.1`
- `jupyterlab>=4`

---

## **Phase 2: Configuration File Extension**

### Goal
Extend `config.yaml` to include simulation parameters (rain/sun duration, pedestrian names, coffeeshop locations).

### New/Modified Files
- **Modified:** `config.yaml`
  - Add `weather:` section (rain_duration, sun_duration)
  - Add `pedestrians:` section (count, names, colors)
  - Add `geography:` section (center_lat/lon, bounds)
  - Add `coffeeshops:` list with 5 locations
- **Modified:** `notebooks/agent_weather.ipynb`
  - Load config using `simulated_city.config.load_config()`
  - Replace hardcoded 10/20 values with `cfg.weather.rain_duration` / `cfg.weather.sun_duration`

### Tests/Verification
```bash
# Verify config loading
python -c "from simulated_city.config import load_config; cfg = load_config(); print(cfg.weather)"

# Re-run weather notebook, confirm it uses config values
python -m jupyterlab notebooks/agent_weather.ipynb
```

### Investigation Before Next Phase
- **Config structure:** Does `simulated_city.config.AppConfig` need a new dataclass for weather/pedestrians/coffeeshops?
- **Geography:** Are coffeeshop coordinates realistic (within Copenhagen bounding box)?
- **Fix latitude bug:** South bound in concepts.md says "65°S" — should be ~55.66°N

### Dependencies
Already installed:
- `PyYAML>=6.0`
- `python-dotenv>=1.0`

---

## **Phase 3: Single Pedestrian (No MQTT Yet)**

### Goal
Create a `Pedestrian` class with basic random walk logic, compute movement in a loop, print locations to console.

### New Files
- `notebooks/agent_pedestrians.ipynb`
  - Cell 1: Markdown (overview)
  - Cell 2: Import config, time, random, math
  - Cell 3: Define `Pedestrian` class with:
    - `__init__(name, color, start_lat, start_lon, bounds)`
    - `random_walk()` method (move in random direction by step_size)
    - `calculate_nearest_coffeeshop(coffeeshops)` method (Euclidean distance)
    - `move_toward_target(target_lat, target_lon, speed)` method
  - Cell 4: Instantiate ONE pedestrian (name="Alice", random start position)
  - Cell 5: Loop: call `random_walk()` every 0.5s, print position for 10 iterations
  - Cell 6: Test: manually call `calculate_nearest_coffeeshop()` with dummy coffeeshop list

### Tests/Verification
```bash
# Run the pedestrians notebook
python -m jupyterlab notebooks/agent_pedestrians.ipynb

# Expected console output:
# Alice at (55.6780, 12.5690)
# Alice at (55.6782, 12.5688)
# ...
```

### Investigation Before Next Phase
- **Movement model:** Is random walk step size realistic (should move ~1.4 m/s = 5 km/h)?
- **Bounds checking:** Does the pedestrian stay within the bounding box?
- **Distance calculation:** Is Euclidean distance formula correct for lat/lon?

### Dependencies
None (uses Python stdlib: `random`, `math`, `time`)

---

## **Phase 4: MQTT Integration for Pedestrian**

### Goal
Connect the pedestrian to MQTT: subscribe to weather, publish location and status.

### Modified Files
- **Modified:** `notebooks/agent_pedestrians.ipynb`
  - Cell 2: Add imports for `simulated_city.mqtt`, `json`
  - Cell 3: Update `Pedestrian` class:
    - Add `state` attribute: `"walking"` or `"sheltering"`
    - Add `target_coffeeshop_id` attribute
    - Add `on_weather_change(raining)` method to handle rain events
  - Cell 4: Connect to MQTT, subscribe to `simulated_city/weather/rain`
  - Cell 5: Set up callback to call `on_weather_change()` when weather message arrives
  - Cell 6: Main loop:
    - If sunny: call `random_walk()`
    - If raining and not at target: call `move_toward_target()`
    - Publish to `simulated_city/pedestrians/person_0/location` every 0.5s
    - Publish to `simulated_city/pedestrians/person_0/status` when state changes

### Tests/Verification
```bash
# Terminal 1: Run weather agent
jupyter notebook notebooks/agent_weather.ipynb

# Terminal 2: Subscribe to pedestrian topics
mosquitto_sub -h localhost -t "simulated_city/pedestrians/#" -v

# Terminal 3: Run pedestrian agent
jupyter notebook notebooks/agent_pedestrians.ipynb

# Expected: see location updates in Terminal 2, status changes when rain starts/stops
```

### Investigation Before Next Phase
- **Callback timing:** Does `on_weather_change()` fire immediately when weather changes?
- **Movement during rain:** Does the pedestrian reach the coffeeshop within 10 seconds?
- **State machine:** Are `"walking"` / `"sheltering"` states correct?

### Dependencies
Already installed (no new ones)

---

## **Phase 5: Multiple Pedestrians (5 Agents)**

### Goal
Expand to 5 pedestrian instances, publish to separate MQTT topics.

### Modified Files
- **Modified:** `notebooks/agent_pedestrians.ipynb`
  - Cell 4: Replace single pedestrian with list of 5:
    ```python
    pedestrians = [
        Pedestrian(name, color, random_start(), coffeeshops)
        for name, color in zip(cfg.pedestrians.names, cfg.pedestrians.colors)
    ]
    ```
  - Cell 6: Main loop iterates over `pedestrians` list:
    - Each publishes to `simulated_city/pedestrians/person_{i}/location`
    - Each publishes to `simulated_city/pedestrians/person_{i}/status`

### Tests/Verification
```bash
# Terminal 1: Run weather agent
# Terminal 2: Subscribe to all pedestrian topics
mosquitto_sub -h localhost -t "simulated_city/pedestrians/+/location"

# Terminal 3: Run pedestrians notebook (now with 5 agents)

# Expected: see 5 separate location streams updating every 0.5s
```

### Investigation Before Next Phase
- **Capacity logic:** If > 3 people move to the same coffeeshop, how do we handle it?
- **Performance:** Does 5 agents publishing every 0.5s cause lag?
- **Coffee wait time:** Should we add the 0–5 second random delay after rain stops?

### Dependencies
None

---

## **Phase 6: Dashboard Visualization**

### Goal
Create a dashboard notebook that subscribes to all MQTT topics and displays pedestrians + coffeeshops on a map using `anymap-ts`.

### New Files
- `notebooks/dashboard.ipynb`
  - Cell 1: Markdown (overview)
  - Cell 2: Import `anymap_ts`, `simulated_city.mqtt`, `simulated_city.config`, `json`
  - Cell 3: Load config, read coffeeshop locations
  - Cell 4: Create map centered on Copenhagen (55.6761°N, 12.5683°E), zoom 14
  - Cell 5: Add coffeeshop markers (green, labeled)
  - Cell 6: Connect to MQTT, subscribe to:
    - `simulated_city/weather/rain`
    - `simulated_city/pedestrians/+/location`
    - `simulated_city/pedestrians/+/status`
  - Cell 7: Set up MQTT callbacks:
    - `on_weather`: toggle map theme dark/light
    - `on_location`: update/create pedestrian marker with color
    - `on_status`: update marker popup text (show sheltering status)
  - Cell 8: Keep map running (e.g., `time.sleep()` or Jupyter output)

### Tests/Verification
```bash
# Terminal 1: Run weather agent
# Terminal 2: Run pedestrians agent
# Terminal 3: Run dashboard notebook
jupyter notebook notebooks/dashboard.ipynb

# Expected:
# - Map shows 5 coffeeshops (green)
# - Map shows 5 pedestrians (colored markers)
# - Markers move in real time
# - Map goes dark when raining
```

### Investigation Before Next Phase
- **Map update rate:** Does the map lag with 5 markers updating every 0.5s?
- **Theme switching:** Does `add_basemap("CartoDB.DarkMatter")` replace the existing basemap?
- **Marker IDs:** Are we using stable marker IDs (e.g., `name="person_0"`) to update positions?

### Dependencies
Already installed:
- `anymap-ts[all]`
- `matplotlib>=3.8` (for anymap-ts backend)

---

## **Phase 7: Polish and Final Features**

### Goal
Add capacity constraints, coffee wait time, final cleanup.

### Modified Files
- **Modified:** `notebooks/agent_pedestrians.ipynb`
  - Add shared state for coffeeshop occupancy (dict: `{coffeeshop_id: [person_names]}`)
  - Modify `calculate_nearest_coffeeshop()` to skip full coffeeshops (capacity=3)
  - Add `coffee_wait_time` attribute (random 0–5 seconds) when rain stops
  - Add logic: don't leave coffeeshop until wait time expires
- **Modified:** `notebooks/dashboard.ipynb`
  - Display coffeeshop occupancy count in marker popups
  - Show rain timer in map title or legend

### Tests/Verification
```bash
# Full system test: run all 3 notebooks
# Verify:
# 1. If 4 people go to same coffeeshop, 4th person picks a different one
# 2. People stay at coffeeshop for 0–5 seconds after rain stops
# 3. Map shows "3/3 full" for occupied coffeeshops
```

### Investigation Before Next Phase
- **Shared state:** How do pedestrians know coffeeshop occupancy? Via MQTT or shared variable?
- **Edge cases:** What if all 5 coffeeshops are full?
- **Movement speed units:** Should we convert 5 km/h to degrees/second based on latitude?

### Dependencies
None

---

## **Summary of Notebook Architecture**

After all phases:

1. **`notebooks/agent_weather.ipynb`** — publishes `simulated_city/weather/rain` every 30s
2. **`notebooks/agent_pedestrians.ipynb`** — subscribes to weather, publishes 5 pedestrian locations + statuses
3. **`notebooks/dashboard.ipynb`** — subscribes to all topics, displays map with live updates

All three run concurrently in separate Jupyter tabs or terminals.

---

## **Key Investigation Questions Across All Phases**

1. **Movement speed:** Should we use meters/second (5 km/h) or degrees/step? Need to convert based on latitude.
2. **MQTT field names:** Standardize on `coffeeshop_id` (not `shelf_id` or `target_shelf_id`).
3. **South latitude bug:** Fix "65°S" to "55.66°N" in concepts.md and config.yaml.
4. **Capacity enforcement:** Should capacity be enforced locally (pedestrians track occupancy) or via a coordinator agent?
5. **Basemap switching:** Does anymap-ts support live basemap switching, or do we need to recreate the map?

---

## **Next Steps**

When ready to begin implementation:
1. Review this plan and concepts.md
2. Request Phase 1 implementation
3. After each phase: run tests, investigate, then request next phase
