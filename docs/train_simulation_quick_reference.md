# Train Simulation Quick Reference

## Running the Notebook

### Step-by-Step
1. Open `notebooks/visualize_trains.ipynb` in Jupyter
2. Click "Run All Cells" or execute cells in order
3. Select a date from the date picker
4. Click the **Start** button
5. Watch trains move on the map
6. Use Pause/Resume/Stop as needed

### Expected Output
- "✓ Dependencies imported successfully"
- "✓ Found NeTEx data: Rejseplanen+NeTEx.zip"
- "✓ Available operators: [DSB_000002, DSB_001022, ...]"
- "✓ Selected X train operator(s)"
- "✓ Loaded N train lines" (where N ≈ 30–50 main line trains)
- Date range (e.g., "2026-02-27 to 2026-03-04")

---

## Customization Examples

### Change Simulation Speed

Faster (more stretched):
```python
SECONDS_PER_SIM_HOUR = 5   # 1 sim hour = 5 real seconds (480× speedup)
```

Slower (more detailed):
```python
SECONDS_PER_SIM_HOUR = 30  # 1 sim hour = 30 real seconds (120× speedup)
```

### Show Station Markers

```python
SHOW_STATIONS_LAYER = True
```

Caution: This adds 1000+ gray dots and may slow down the map.

### Include S-train Lines

Modify `is_excluded_service()` to return `False`:
```python
def is_excluded_service(line_data: Dict) -> bool:
    # Only exclude metro
    transport_mode = (line_data.get("line_transport_mode") or "").lower()
    if transport_mode == "metro":
        return True
    return False
```

### Include Metro Lines

```python
def is_excluded_service(line_data: Dict) -> bool:
    return False  # Don't exclude anything
```

### Zoom to a Different Region

In `create_base_map_with_optional_stations()`:
```python
m = LiveMapLibreMap(
    center=(10.2, 56.5),  # Aarhus (lng, lat)
    zoom=9,
    height="650px",
    width="100%"
)
```

### Change Map Theme

After creating the map:
```python
m.add_basemap("CartoDB Positron")  # Dark map
# OR
m.add_basemap("OpenTopoMap")  # Terrain map
```

### Adjust Marker Update Rate

```python
UPDATE_INTERVAL_REAL_SEC = 0.5  # Update every 0.5 seconds (smoother but slower)
# OR
UPDATE_INTERVAL_REAL_SEC = 2    # Update every 2 seconds (faster but choppier)
```

---

## Troubleshooting

### "No train movement data for selected date"

**Causes:**
- Date is outside the valid range
- All journeys were filtered out
- Parser failed

**Solutions:**
```python
# Check valid date range (printed after Cell 5)
print(date_range['min'], "to", date_range['max'])

# Temporarily disable filtering to see all services
def is_excluded_service(line_data: Dict) -> bool:
    return False  # Include everything

# Check if parser found data
print(f"Lines loaded: {len(train_data)}")
for line, data in train_data.items():
    print(f"  Line {line}: {len(data['journeys'])} journeys")
```

### Trains are invisible or frozen

**Check:**
```python
# Print first few segments to verify they're being created
segments, used_lines = build_train_segments_for_date(date_picker.value)
print(f"Segments: {len(segments)}")
if segments:
    print(f"First segment: {segments[0]}")
```

**Verify times look reasonable:**
```python
# start_sec and end_sec should be between 0 and 86400 (seconds in 24 hours)
seg = segments[0]
print(f"Start: {seg['start_sec']}s ({seg['start_sec']//3600}:{(seg['start_sec']%3600)//60})")
print(f"End: {seg['end_sec']}s ({seg['end_sec']//3600}:{(seg['end_sec']%3600)//60})")
```

### Map doesn't display

**Try:**
```python
# Create a test map manually
test_map = LiveMapLibreMap(center=(12.5683, 55.6761), zoom=7)
test_map.add_basemap("OpenStreetMap.Mapnik")
test_map.add_marker(12.5683, 55.6761, name="test", color="#FF0000", popup="Test")
display(test_map)
```

### Station markers crowd the map

**Solution:**
```python
SHOW_STATIONS_LAYER = False
```

---

## Understanding Map Output

### Color Legend

Each train line gets a unique, stable color. To see the color for a line:

```python
line_number = "002"  # IC 2
color = get_line_color(line_number)
print(f"Line {line_number} color: {color}")
```

### Popup Information

Hover over a train marker to see:
- **Train line**: The line number (e.g., "002")
- **Destination**: Final stop of this journey
- **Position time**: "simulated" (not real time)

### Active Train Count

The status bar shows how many trains are between stops at the current simulated time. This varies:
- **Early morning** (00:00–06:00): 0–5 trains
- **Peak hours** (08:00–09:00, 16:00–18:00): 15–30 trains
- **Midday/evening** (10:00–15:00, 18:00–23:00): 5–20 trains
- **Late night** (23:00–00:00): 0–10 trains

---

## Performance Tips

### For Faster Playback on Slow Machines

```python
SECONDS_PER_SIM_HOUR = 20      # Faster playback
UPDATE_INTERVAL_REAL_SEC = 2   # Update every 2 seconds
SHOW_STATIONS_LAYER = False    # Disable station dots
```

### For Smoother Animation

```python
SECONDS_PER_SIM_HOUR = 10      # Normal speed
UPDATE_INTERVAL_REAL_SEC = 0.5 # Update every 0.5 seconds
SHOW_STATIONS_LAYER = False
```

### For Debugging (Step Through Time Manually)

```python
# Build segments once
segments, used_lines = build_train_segments_for_date(date_picker.value)

# Create map
m, _ = create_base_map_with_optional_stations(SHOW_STATIONS_LAYER)
display(m)

# Manually check a specific time (08:30)
sim_sec = 8 * 3600 + 30 * 60  # 30600 seconds = 08:30
active = collect_active_trains(segments, sim_sec)
print(f"Trains at 08:30: {len(active)}")
for train in active:
    print(f"  Line {train['line']} → {train['destination']}")
```

---

## Understanding the NeTEx Parser

### What Gets Parsed

From each XML file:
1. **Operator** name (e.g., "DSB", "DSB S-tog")
2. **Line** metadata: name, public code, transport mode
3. **Stops**: All station names and coordinates
4. **Service journeys**: Scheduled departures/arrivals for the day

### What Gets Filtered

By default, **excluded services**:
- Metro lines (mode="metro" or code starting with "M")
- S-train lines (operator contains "s-tog" or code in {A, B, C, E, F, H, Blå, Rød, ...})

Remaining: **Intercity trains** (IC), **regional trains**, **night trains**

### How to Verify Parsing

```python
# Check what was loaded
for line_num, data in train_data.items():
    print(f"Line {line_num}:")
    print(f"  Operator: {data['operator_name']}")
    print(f"  Code: {data['line_public_code']}")
    print(f"  Mode: {data['line_transport_mode']}")
    print(f"  Stops: {len(data['stops'])}")
    print(f"  Journeys: {len(data['journeys'])}")
```

---

## Reference Documentation

For in-depth guides, see:
- [docs/train_simulation.md](../docs/train_simulation.md) — Full architecture and usage
- [docs/netex_format.md](../docs/netex_format.md) — NeTEx XML structure and parser implementation

