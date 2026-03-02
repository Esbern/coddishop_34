# NeTEx Format Guide

NeTEx (Network and Timetable Exchange) is a universal XML standard for public transport data. This guide explains the structure, how the parser reads it, and how to extend the parser for new use cases.

---

## What is NeTEx?

NeTEx is a standard maintained by CEN (Comité Européen de Normalisation) for exchanging public transport network, route, and timetable data. It is designed to be:

- **Comprehensive**: Covers all public transport modes (rail, metro, bus, etc.)
- **Interoperable**: Used across European transport operators
- **Hierarchical**: Organized from large (operators) to small (individual stop visits)

Denmark's Rejseplanen publishes NeTEx data as a ZIP file containing multiple XML files, one per line per operator.

---

## File Structure

### Typical ZIP Layout

```
Rejseplanen+NeTEx.zip
├── DSB_000002/              # Operator: DSB (train operator)
│   ├── NX-PI-01_DK_NAP_LINE_DSB-000002-001_20260227.xml
│   ├── NX-PI-01_DK_NAP_LINE_DSB-000002-002_20260227.xml
│   └── ... (one file per line per date)
│
├── DSB_001022/              # Operator: DSB S-train (commuter rail)
│   ├── NX-PI-01_DK_NAP_LINE_DSB-001022-Blaa_20260227.xml
│   ├── NX-PI-01_DK_NAP_LINE_DSB-001022-Roed_20260227.xml
│   └── ...
│
└── MET_Metro/               # Operator: Copenhagen Metro
    ├── NX-PI-01_DK_NAP_LINE_MET-Metro-M1_20260227.xml
    ├── NX-PI-01_DK_NAP_LINE_MET-Metro-M2_20260227.xml
    └── ...
```

**Filename pattern**: `NX-PI-01_DK_NAP_LINE_{OPERATOR_ID}-{LINE_ID}_{DATE}.xml`

Each file is **self-contained** — it includes all stop locations and schedule information needed to simulate that line for that entire date.

---

## XML Structure

Here is a simplified view of a single NeTEx file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<PublicationDelivery xmlns="http://www.netex.org.uk/netex" version="1.2">
  <dataObjects>
    <CompositeFrame id="...">
      <conversions>
        <!-- ... -->
      </conversions>
      
      <FrameDefaults>
        <!-- Default XML namespace, CRS, etc. -->
      </FrameDefaults>
      
      <members>
        <!-- Operator metadata -->
        <Operator id="...">
          <Name>DSB</Name>
        </Operator>
        
        <!-- Line metadata -->
        <Line id="DSB_000002::Line:001::">
          <Name>IC 2</Name>
          <PublicCode>001</PublicCode>
          <TransportMode>rail</TransportMode>
          <Colour>...</Colour>
        </Line>
        
        <!-- Stop locations (geographic reference points) -->
        <ScheduledStopPoint id="DSB:StopPoint:61296">
          <Name>København H</Name>
          <Location>
            <Latitude>55.672734</Latitude>
            <Longitude>12.564911</Longitude>
          </Location>
        </ScheduledStopPoint>
        
        <!-- ... more stops ... -->
        
        <!-- Journey pattern (fixed sequence of stops) -->
        <ServiceJourney id="DSB_000002::ServiceJourney:...">
          <!-- Option A: calls/Call (Legacy format) -->
          <calls>
            <Call>
              <ScheduledStopPointRef ref="DSB:StopPoint:61296"/>
              <ArrivalTime>08:00:00</ArrivalTime>
              <DepartureTime>08:00:00</DepartureTime>
            </Call>
            <Call>
              <ScheduledStopPointRef ref="DSB:StopPoint:61297"/>
              <ArrivalTime>08:15:00</ArrivalTime>
              <DepartureTime>08:15:00</DepartureTime>
            </Call>
            <!-- ... more stops ... -->
          </calls>
          
          <!-- Option B: passingTimes/TimetabledPassingTime (Modern format) -->
          <passingTimes>
            <TimetabledPassingTime>
              <StopPointInJourneyPatternRef ref="..."/>
              <ArrivalTime>08:00:00</ArrivalTime>
              <DepartureTime>08:00:00</DepartureTime>
            </TimetabledPassingTime>
            <!-- ... more stops ... -->
          </passingTimes>
        </ServiceJourney>
        
        <!-- ... more journeys ... -->
      </members>
    </CompositeFrame>
  </dataObjects>
</PublicationDelivery>
```

---

## Key Elements Explained

### Operator
```xml
<Operator id="DSB_001022">
  <Name>DSB S-tog</Name>
</Operator>
```

- **Purpose**: Identifies the transport operator
- **In simulation**: Used to filter specific operator types (e.g., "S-tog" = S-train operator)
- **Parser extracts**: `operator_name`

### Line
```xml
<Line id="DSB_000002::Line:001::">
  <Name>IC 2</Name>
  <PublicCode>001</PublicCode>
  <TransportMode>rail</TransportMode>
  <Colour>FF0000</Colour>
</Line>
```

- **Purpose**: Defines a fixed route with a name and number
- **In simulation**: Used to identify and color train lines, and to filter by mode
- **Parser extracts**: `line_number`, `line_public_code`, `line_name`, `line_transport_mode`

### ScheduledStopPoint
```xml
<ScheduledStopPoint id="DSB:StopPoint:61296">
  <Name>København H</Name>
  <Location>
    <Latitude>55.672734</Latitude>
    <Longitude>12.564911</Longitude>
  </Location>
</ScheduledStopPoint>
```

- **Purpose**: A named location where trains stop (station, platform, or halt)
- **In simulation**: Defines the geographic positions through which trains travel
- **Parser extracts**: `stops[stop_id] = {name, lat, lng}`

### ServiceJourney (with calls/Call)
```xml
<ServiceJourney id="DSB_000002::ServiceJourney:001_001">
  <calls>
    <Call>
      <ScheduledStopPointRef ref="DSB:StopPoint:61296"/>
      <ArrivalTime>08:00:00</ArrivalTime>
      <DepartureTime>08:00:00</DepartureTime>
    </Call>
    <Call>
      <ScheduledStopPointRef ref="DSB:StopPoint:61297"/>
      <ArrivalTime>08:15:00</ArrivalTime>
      <DepartureTime>08:15:00</DepartureTime>
    </Call>
  </calls>
</ServiceJourney>
```

- **Legacy format** (some older files)
- Each `<Call>` contains:
  - `ScheduledStopPointRef`: Reference to a stop by `id`
  - `ArrivalTime`: When the train arrives (HH:MM:SS or with timezone)
  - `DepartureTime`: When the train departs
- **In simulation**: Used to build journey segments (straight segments between stops)
- **Parser extracts**: `journeys[].stops[].{stop_id, time, sequence}`

### ServiceJourney (with passingTimes/TimetabledPassingTime)
```xml
<ServiceJourney id="DSB_001022::ServiceJourney:001_001">
  <passingTimes>
    <TimetabledPassingTime>
      <StopPointInJourneyPatternRef ref="DSB:SPJP:001_001_1"/>
      <ArrivalTime>06:00:00</ArrivalTime>
      <DepartureTime>06:00:00</DepartureTime>
    </TimetabledPassingTime>
    <TimetabledPassingTime>
      <StopPointInJourneyPatternRef ref="DSB:SPJP:001_001_2"/>
      <ArrivalTime>06:15:00</ArrivalTime>
      <DepartureTime>06:15:00</DepartureTime>
    </TimetabledPassingTime>
  </passingTimes>
</ServiceJourney>
```

- **Modern format** (DSB S-tog, metro, newer lines)
- Uses `StopPointInJourneyPatternRef` instead of direct stop references
- Requires a **mapping table** (StopPointInJourneyPattern → ScheduledStopPoint) elsewhere in the file
- **In simulation**: Same purpose as calls, but requires dereferencing

### StopPointInJourneyPattern (for mapping)
```xml
<StopPointInJourneyPattern id="DSB:SPJP:001_001_1">
  <ScheduledStopPointRef ref="DSB:StopPoint:61296"/>
</StopPointInJourneyPattern>
```

- **Purpose**: Maps a stop reference in a journey to an actual stop definition
- **In parser**: Built into a lookup table (`spjp_to_stop_id`) before parsing journeys

---

## How the Parser Works

### Step 1: Extract Operator and Line Metadata

```python
op_name_elem = root.find('.//nx:Operator/nx:Name', ns)
if op_name_elem is not None:
    result['operator_name'] = op_name_elem.text.strip()

line_elem = root.find('.//nx:Line', ns)
if line_elem is not None:
    public_code_elem = line_elem.find('nx:PublicCode', ns)
    result['line_public_code'] = public_code_elem.text.strip()
    # ... extract mode, name, etc.
```

Uses XPath-style searches with the NeTEx namespace (`nx`) to find top-level metadata.

### Step 2: Extract Stop Locations

```python
for stop_elem in root.findall('.//nx:ScheduledStopPoint', ns):
    stop_id = stop_elem.get('id', '')
    lat_elem = stop_elem.find('nx:Location/nx:Latitude', ns)
    lng_elem = stop_elem.find('nx:Location/nx:Longitude', ns)
    
    result['stops'][stop_id] = {
        'name': name,
        'lat': float(lat_elem.text),
        'lng': float(lng_elem.text),
    }
```

Builds a complete dictionary: `stop_id → {name, lat, lng}`. This is used later to dereference journey stops.

### Step 3: Build StopPointInJourneyPattern Lookup (for modern format)

```python
spjp_to_stop_id = {}
for spjp in root.findall('.//nx:StopPointInJourneyPattern', ns):
    spjp_id = spjp.get('id', '')
    stop_ref = spjp.find('nx:ScheduledStopPointRef', ns)
    if stop_ref is not None:
        stop_id = stop_ref.get('ref', '')
        spjp_to_stop_id[spjp_id] = stop_id
```

Creates a mapping for files using the modern format, so that `StopPointInJourneyPatternRef` can be resolved to actual stops.

### Step 4: Parse Service Journeys

**Variant A: calls/Call (Legacy)**

```python
call_nodes = journey.findall('nx:calls/nx:Call', ns)
for idx, call in enumerate(call_nodes):
    stop_point_ref = call.find('nx:ScheduledStopPointRef', ns)
    stop_id = stop_point_ref.get('ref', '')
    departure = call.find('nx:DepartureTime', ns)
    
    journey_stops.append({
        'stop_id': stop_id,
        'time': departure.text,
        'sequence': idx,
    })
```

**Variant B: passingTimes/TimetabledPassingTime (Modern)**

```python
passing_nodes = journey.findall('nx:passingTimes/nx:TimetabledPassingTime', ns)
for idx, passing in enumerate(passing_nodes):
    spjp_ref_elem = passing.find('nx:StopPointInJourneyPatternRef', ns)
    spjp_ref = spjp_ref_elem.get('ref', '')
    stop_id = spjp_to_stop_id.get(spjp_ref)  # Dereference using lookup
    
    departure = passing.find('nx:DepartureTime', ns)
    
    journey_stops.append({
        'stop_id': stop_id,
        'time': departure.text,
        'sequence': idx,
    })
```

The parser tries both formats and uses whichever one is present in the file.

### Step 5: Validate and Return

```python
if journey_stops:
    result['journeys'].append({
        'journey_id': journey_id,
        'stops': journey_stops,
    })

return result if result['line_number'] else None
```

Only returns data if a line number was successfully extracted. Validation ensures each journey has at least one stop.

---

## Time Format

NeTEx times are ISO 8601 strings, sometimes with timezone information:

| Format | Example | Notes |
|--------|---------|-------|
| HH:MM:SS | 08:15:30 | Basic format |
| T prefix | T08:15:30 | With date prefix (parsed out) |
| Timezone offset | 08:15:30+01:00 | Suffix after seconds |
| Timezone letter | 08:15:30Z | UTC (Z suffix) |

The parser handles these with `parse_time_to_seconds()`:

```python
def parse_time_to_seconds(raw_time: Optional[str]) -> Optional[int]:
    text = raw_time.strip()
    
    # Remove date prefix
    if "T" in text:
        text = text.split("T", 1)[1]
    
    # Remove timezone offsets (+ or -)
    if "+" in text:
        text = text.split("+", 1)[0]
    if "-" in text and text.count(":") >= 2:
        text = text.rsplit("-", 1)[0]
    
    # Remove Z suffix
    if text.endswith("Z"):
        text = text[:-1]
    
    # Convert HH:MM:SS to seconds since midnight
    parts = text.split(":")
    hours = int(parts[0])
    minutes = int(parts[1])
    seconds = int(parts[2]) if len(parts) > 2 else 0
    
    return hours * 3600 + minutes * 60 + seconds
```

---

## Validity Dates

Each NeTEx file specifies a date range:

```xml
<ValidBetween>
  <FromDate>2026-02-27</FromDate>
  <ToDate>2026-03-04</ToDate>
</ValidBetween>
```

The parser extracts these and the simulation uses them to filter journeys:

```python
if parsed['valid_from'] and parsed['valid_to']:
    valid_from = datetime.strptime(parsed['valid_from'], "%Y-%m-%d").date()
    valid_to = datetime.strptime(parsed['valid_to'], "%Y-%m-%d").date()
    if not (valid_from <= selected_date <= valid_to):
        continue  # Skip this line for the selected date
```

This ensures you only animate trains that are actually scheduled for the selected day.

---

## Filtering by Transport Mode

NeTEx defines transport modes as strings:

```xml
<TransportMode>rail</TransportMode>
```

Common values:
- **rail**: Main line trains (intercity, regional)
- **metro**: Subway/underground rapid transit
- **bus**: Bus services
- **tram**: Trams/streetcars
- **coach**: Long-distance coach services

The simulation uses this to exclude certain modes:

```python
if line_data.get("line_transport_mode") == "metro":
    return True  # Exclude metro
```

---

## Extending the Parser

### Extract Additional Stop Information

To capture platform numbers or stop types:

```python
for stop_elem in root.findall('.//nx:ScheduledStopPoint', ns):
    stop_id = stop_elem.get('id', '')
    
    # Extract platform
    platform_elem = stop_elem.find('nx:PlatformRef', ns)
    platform = platform_elem.get('ref', '') if platform_elem else None
    
    result['stops'][stop_id] = {
        'name': name,
        'lat': float(lat_elem.text),
        'lng': float(lng_elem.text),
        'platform': platform,  # NEW
    }
```

### Extract Distance Information

For realistic train speeds, extract distance traveled between stops:

```python
for journey in root.findall('.//nx:ServiceJourney', ns):
    # Look for <Distance> elements
    distance_elem = journey.find('nx:Distance', ns)
    if distance_elem is not None:
        distance_m = float(distance_elem.text)
        # Use to calculate speed: speed_kmh = distance_m / time_duration / 1000
```

### Extract Additional Journey Attributes

Capture vehicle type or accessibility information:

```python
vehicle_type_elem = journey.find('nx:VehicleType', ns)
is_wheelchair_accessible = journey.find('nx:WheelchairAccess', ns)
```

---

## Common Issues and Solutions

### "Element not found"

NeTEx is flexible — not all files have all elements. Always use `.find()` with null checks:

```python
# Safe:
name_elem = stop_elem.find('nx:Name', ns)
name = name_elem.text if name_elem is not None else 'Unknown'

# Dangerous (crashes if nx:Name missing):
name = stop_elem.find('nx:Name', ns).text
```

### Namespace Errors

NeTEx requires the namespace prefix `nx:` when searching:

```python
# Wrong:
root.find('.//Line')  # Returns None

# Correct:
root.find('.//nx:Line', {'nx': 'http://www.netex.org.uk/netex'})
```

### Timezone Confusion

Times in NeTEx files may be in local operator time with a timezone offset. The parser assumes all times are in the same timezone for a given file and converts to seconds since midnight. If you need exact UTC times, parse and store the timezone separately.

---

## References

- **Official NeTEx Specification**: https://www.netex.org.uk/
- **CEN TC 278 WG3**: Standardization body
- **Rejseplanen NeTEx Data**: Available via Danish transport authority

For questions or contributions, see the project README.
