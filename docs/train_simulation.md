# Train Simulation and Visualization

This document explains the train simulation notebook (`visualize_trains.ipynb`), how it reads public transport data from the NeTEx format, and how it animates trains on an interactive map.

---

## Overview

The train simulation visualizes **real train schedules** from Denmark's Rejseplanen data (NeTEx format). It:

1. **Loads schedule data** from a ZIP file containing XML files
2. **Parses routes and stops** from the NeTEx XML structure  
3. **Builds movement segments** for each train journey (interpolatable between stops)
4. **Runs a discrete-time simulation loop** that updates train positions every second
5. **Displays trains on a map** using interactive MapLibre markers

The simulation runs in a single notebook with **interactive controls**: Start, Pause, Resume, and Stop buttons allow you to control playback without rerunning cells.

---

## Key Concepts

### Train, Line, and Journey

- **Line**: A fixed route with a name or number (e.g., "IC 102" for an intercity train, "A" for an S-train).
- **Journey**: A specific instance of a line on a given day (e.g., "IC 102 departing 08:00 from Copenhagen").
- **Segment**: A movement between two consecutive stops in a journey.

### Simulation Model

The simulation uses **linear interpolation** between stops:
- Each train moves between two consecutive stops over a fixed time interval
- Its position is calculated as: `position = start + (end - start) × (elapsed_time / total_time)`
- When a train is not between stops, it is not displayed

### Discrete Timestep Loop

- **Real duration**: 1 second per update
- **Simulated duration**: 10 simulated seconds per 1 real second (240× speedup)
- **Update interval**: Every 1 real second, the map refreshes with new train positions

This means a full 24-hour simulated day takes about **240 real seconds** (4 minutes) to display.

---

## Architecture

### Data Flow

```
NeTEx ZIP File
    ↓
Parser (Cell 3)
    ↓
train_data dictionary
    ├─ line_number: string (e.g., "002", "A")
    ├─ operator_name: string (e.g., "DSB", "DSB S-tog")
    ├─ line_public_code: string (e.g., "IC", "Blå")
    ├─ line_transport_mode: string (e.g., "rail", "metro")
    ├─ stops: dict (stop_id → {name, lat, lng})
    └─ journeys: list of {journey_id, stops [
         {stop_id, sequence, time}
       ]}
         ↓
Segment Builder (filters by date and excluded services)
    ↓
segments: list of {
    train_id, line, color, destination,
    start_sec, end_sec,
    start_lat, start_lng, end_lat, end_lng
}
    ↓
Simulation Loop (every 1 real second)
    ├─ Calculate simulated time (0–86400 seconds)
    ├─ Collect active trains (segments containing this time)
    ├─ Interpolate positions
    └─ Update map markers using LiveMapLibreMap.move_marker()
```

### Code Structure

#### Cell 1: Imports
Loads dependencies: pandas, zipfile, datetime, XML parser, and `LiveMapLibreMap` from `simulated_city`.

#### Cell 3: NeTEx Parser
Reads XML files and extracts:
- Line metadata (name, public code, operator, transport mode)
- Stop locations (latitude, longitude)
- Service journeys and their timetables

Supports two timing models:
- **Model A** (`calls/Call`): Legacy format with inline arrival/departure times
- **Model B** (`passingTimes/TimetabledPassingTime`): Modern DSB format

See [docs/netex_format.md](netex_format.md) for details on the XML structure.

#### Cell 5: Data Loader
Iterates through all NeTEx files in the ZIP and calls the parser. Merges data from multiple files for the same line.

#### Cell 7: Date Picker
Interactive widget to select a date within the data's valid range.

#### Cell 9: Simulation Helpers

**`is_excluded_service(line_data)`**
- Filters out S-trains and metro lines
- Uses operator name, line code, and transport mode to identify excluded services

**`parse_time_to_seconds(raw_time)`**
- Converts NeTEx time strings (ISO 8601 format with optional timezone) to seconds since midnight
- Handles edge cases like timezone suffixes

**`build_train_segments_for_date(selected_date)`**
- For the selected date, iterates through all lines and journeys
- Checks date validity (`valid_from` and `valid_to`)
- Applies the exclusion filter
- Builds interpolatable segments between consecutive stops

**`interpolate_train_position(segment, simulation_sec)`**
- Given a segment and a simulated time in seconds, returns the interpolated (lng, lat)
- Uses linear interpolation: `ratio = (simulation_sec - start_sec) / (end_sec - start_sec)`

**`collect_active_trains(segments, simulation_sec)`**
- Scans all segments and returns trains that are active at the current simulation time
- A train is active if `start_sec ≤ simulation_sec < end_sec`

**`create_base_map_with_optional_stations(show_stations_layer)`**
- Creates a `LiveMapLibreMap` centered on Copenhagen with OpenStreetMap base layer
- Optionally adds station markers for all stops (disabled by default to reduce clutter)

**`update_train_markers(m, active_trains, previous_marker_ids)`**
- Updates train markers on the map in-place using `m.move_marker(marker_id, (lng, lat), ...)`
- Removes markers for trains that are no longer active
- Efficient alternative to remove/add pattern

#### Cell 11: Simulation Controls

Interactive buttons and status display:
- **Start**: Load data for selected date, create map, start simulation loop in background thread
- **Pause**: Pause the background simulation thread
- **Resume**: Resume the paused thread
- **Stop**: Stop the simulation

Global state variables manage the simulation:
- `simulation_running`: Boolean flag
- `simulation_paused`: Boolean flag for pause/resume
- `simulation_thread`: Background thread running the loop
- `simulation_map`: The `LiveMapLibreMap` instance
- `simulation_segments`: List of movement segments for the selected date
- `simulation_markers`: List of active marker IDs on the map

---

## Key Settings

```python
SECONDS_PER_SIM_HOUR = 10   # 1 simulated hour = 10 real seconds
UPDATE_INTERVAL_REAL_SEC = 1  # Update map every 1 real second
SHOW_STATIONS_LAYER = False   # Don't clutter map with station dots
```

Adjust these to change simulation speed or visual density.

---

## How to Use

1. **Run all cells** in order
2. **Select a date** from the date picker widget
3. **Click Start** to begin the simulation
4. Observe trains moving on the map from their departure time to arrival time
5. Use **Pause**, **Resume**, and **Stop** to control playback

The status bar shows:
- Current simulated time (HH:MM:SS)
- Number of active trains at that moment
- Log messages with segment and station counts

---

## Understanding the Map Output

**Map Features:**
- **Blue dots** (with colors per line): Train positions updated in real-time
- **Popup on hover**: Shows train line, destination, and "simulated" label
- **Station dots** (if enabled): Gray dots showing all train stops (disabled by default)
- **Background**: OpenStreetMap basemap (gray/muted for clarity)

**Color Assignment:**
Train lines are assigned colors using a hash-based algorithm. The same line always gets the same color across runs.

```python
def get_line_color(line_number: str) -> str:
    colors = ["#FF6B6B", "#4ECDC4", "#45B7D1", ...]  # 10 colors
    digest = hashlib.md5(line_number.encode("utf-8")).hexdigest()
    return colors[int(digest[:8], 16) % len(colors)]
```

---

## Customization

### Filter Different Services

Modify `is_excluded_service()` to include or exclude different line types:

```python
def is_excluded_service(line_data: Dict) -> bool:
    transport_mode = (line_data.get("line_transport_mode") or "").lower()
    
    # Example: exclude only metro
    if transport_mode == "metro":
        return True
    
    return False
```

### Adjust Simulation Speed

Change `SECONDS_PER_SIM_HOUR`:
- `5`: Faster (1 sim hour = 5 real seconds)
- `20`: Slower (1 sim hour = 20 real seconds)

### Show Station Layer

Set `SHOW_STATIONS_LAYER = True` to see all train stops as gray dots. Warning: This can make the map cluttered on dense routes.

### Change Map Center or Zoom

In `create_base_map_with_optional_stations()`, adjust the center coordinates:

```python
m = LiveMapLibreMap(
    center=(12.5683, 55.6761),  # (lng, lat) — Copenhagen
    zoom=7,  # 1–20 scale
    height="650px",
    width="100%"
)
```

---

## Troubleshooting

### "No train movement data for selected date"

**Cause**: The selected date has no journeys in the NeTEx data, or all journeys have been filtered out.

**Solution**: 
- Check that the date is within the valid range (displayed in the info box)
- Verify the parser loaded data without errors by reviewing the print output in Cell 5
- Temporarily comment out the `is_excluded_service()` filter to see if any trains appear

### Trains appear to be stuck or move very quickly

**Cause**: Simulation speed setting or interpolation issue.

**Solution**:
- Confirm `SECONDS_PER_SIM_HOUR` is set to 10 (or your intended value)
- Verify that segment times are being parsed correctly by inspecting a segment's `start_sec` and `end_sec`

### Station markers are hard to see

**Cause**: Too many station dots, or they are too small.

**Solution**:
- Set `SHOW_STATIONS_LAYER = False` (default)
- Or, modify the marker size/color in the `create_base_map_with_optional_stations()` function

---

## Next Steps

- **Extend the simulation**: Add other agents (weather, events, congestion) and publish train state to MQTT
- **Build a dashboard**: Subscribe to train positions and display them on a separate map
- **Analyze schedules**: Extract statistics (busiest hours, longest routes, etc.) from the segments
- **Real-time tracking**: Adapt the code to consume live MQTT feeds instead of pre-loaded files

For details on the NeTEx XML format, see [docs/netex_format.md](netex_format.md).
