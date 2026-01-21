# Race Timing Display & OBS Overlays

## Project Overview

A real-time race timing display system that connects to GetRaceResults.com live timing WebSocket feeds. Includes a full-featured timing display and multiple OBS overlays for streaming.

**WebSocket Source:** GetRaceResults.com live timing system
**Protocol:** SignalR WebSocket with LZString compression

---

## Files

| File | Description |
|------|-------------|
| `livetiming-display.html` | Full-featured timing display with leaderboard, messages, statistics |
| `combined-overlay.html` | All-in-one OBS overlay (flag + timer + messages + car) |
| `car-overlay.html` | Single car tracking overlay for OBS |
| `flag-overlay.html` | Race flag status overlay for OBS |
| `messages-overlay.html` | Race control messages overlay for OBS |

---

## WebSocket Protocol

### Connection URL Format
```
wss://livetiming.getraceresults.com/lt/connect?transport=webSockets&clientProtocol=1.5&_tk={token}&_gr=w&_tkdm={domain}&connectionToken={connectionToken}&tid={tid}
```

### Message Format
Messages are JSON with structure:
```json
{
  "M": [
    { "M": "method_name", "A": [args] }
  ]
}
```

Or in bulk init (array format):
```json
["method_name", arg1, arg2, ...]
```

### Key Message Types

| Method | Description |
|--------|-------------|
| `_` | Compressed bulk init (LZString UTF16) |
| `h_i` | Heat/session init |
| `h_h` | Heat/session update (flag, time, etc.) |
| `r_i` | Results init (headers + initial data) |
| `r_c` | Results change (cell updates) |
| `m_i` | Messages init |
| `m_c` | Message change (new message) |
| `t_i` | Tracker init (GPS positions) |
| `t_p` | Tracker position update |
| `a_i` | Statistics/analytics init |

### Bulk Init Decompression
```javascript
let decompressed = LZString.decompressFromUTF16(args[0]);
if (!decompressed) decompressed = LZString.decompress(args[0]);
const data = JSON.parse(decompressed);
// data is array of [method, args] pairs
```

---

## Data Structures

### Heat State (`h_i`, `h_h`)
```javascript
{
  n: "Race Name",           // Name
  f: 6,                     // Flag status (see flag codes)
  ll: 15,                   // Leader laps completed
  s: 123456789,             // Start time (microseconds since Jan 1, 2000)
  e: 123456789,             // End time
  r: 123456789,             // Race elapsed time (microseconds)
  lt: 3600000000,           // Race duration (microseconds)
  lr: 50                    // Total laps in race
}
```

### Flag Codes
| Code | Class | Text |
|------|-------|------|
| -1, 0 | not-started | NOT STARTED |
| 1 | not-started | READY |
| 2 | red | RED FLAG |
| 3 | yellow | YELLOW |
| 4 | fcy | FCY (Full Course Yellow) |
| 5 | checkered | FINISHED |
| 6 | green | GREEN |
| 7 | sc | SAFETY CAR |
| 8 | code60 | CODE 60 |

### Results Init (`r_i`)
```javascript
{
  l: {
    h: [                    // Headers array
      { n: "Position", p: null },
      { n: "Startnumber", p: null },
      { n: "Name", p: null },
      // ... more headers
    ]
  },
  r: [                      // Rows as flat cell array
    [rowIdx, colIdx, value, meta],
    [0, 0, "1", null],      // Row 0, Col 0 = "1" (position)
    [0, 1, "36", null],     // Row 0, Col 1 = "36" (car number)
    // ...
  ]
}
```

### Results Change (`r_c`)
```javascript
// Args is array of cell arrays
[
  [rowIdx, colIdx, value, meta],
  [0, 0, "2", null],        // Position changed to 2
  // ...
]
```

### Common Column Mappings
| Index | Header Key | Description |
|-------|------------|-------------|
| 0 | position | Race position |
| 2 | startnumber | Car number |
| 3 | state | Car state (RUN, PIT, FIN) |
| 4 | name / currentdriver | Driver name |
| 10 | laps | Laps completed |
| 11 | diff | Gap to leader |
| 12 | hole | Alternative gap field |
| 13 | lastroundtime | Last lap time (microseconds) |
| 14 | fastestroundtime | Best lap time (microseconds) |
| 16 | sectortimes_1 | Sector 1 time (microseconds) |
| 17 | sectortimes_2 | Sector 2 time (microseconds) |
| 18 | sectortimes_3 | Sector 3 time (microseconds) |

### Time Format
All times are in **microseconds**:
- Lap time of 1:23.456 = `83456000`
- Sector time of 28.123 = `28123000`
- To convert: `seconds = value / 1000000`

### Validation
- Valid sector time: `> 5,000,000` and `< 300,000,000` (5s - 300s)
- Valid lap time: `> 30,000,000` and `< 600,000,000` (30s - 600s)
- MAX_INT64 placeholder: `9223372036854775807` (ignore these)

### Messages (`m_i`, `m_c`)
```javascript
{
  Id: "msg123",
  t: "Yellow flag sector 2",  // Text
  bc: 16776960,               // Background color (int)
  fc: 0                       // Foreground color (int)
}
```

Color conversion:
```javascript
function intToRgb(colorInt) {
  const r = (colorInt >> 16) & 255;
  const g = (colorInt >> 8) & 255;
  const b = colorInt & 255;
  return `rgb(${r},${g},${b})`;
}
```

### Tracker Position (`t_p`)
```javascript
[
  6,                    // [0] Race position
  "4",                  // [1] Car number
  1733000,              // [2] Track position value
  3611000,              // [3] Timing value
  1,                    // [4] Sector (1, 2, 3)
  42392,                // [5] Speed
  false,                // [6] In pit
  821005485644000       // [7] Timestamp
]
```

---

## OBS Overlays

### combined-overlay.html

All elements in one page for easy OBS setup.

**Layout:**
```
┌─────────────────────────────────────────────────────────────────┐
│ 🟢 GREEN        │      ⏱️ 0:45:23      │   [Race Message 1]    │
│ Race Name       │    Time Remaining     │   [Race Message 2]    │
│ (top-left)      │    (top-center)       │     (top-right)       │
│                                                                 │
│         ┌─────────────────────────────────────────────────────┐ │
│         │ P3 │ #36    │ Gap  │↑Ahead│↓Behind│Lap│S1 S2 S3│Cur │Last│
│         │    │ Driver │+5.234│ 2.34 │ +1.23 │15 │        │1:23│1:22│
│         └─────────────────────────────────────────────────────┘ │
│                       (bottom-center)                           │
└─────────────────────────────────────────────────────────────────┘
```

**URL Parameters:**
| Param | Description | Example |
|-------|-------------|---------|
| `car` | Car number to track | `36` |
| `ws` | WebSocket URL | `wss://...` |
| `msgs` | Max messages shown | `3` |

**Example:**
```
file:///path/to/combined-overlay.html?car=36&ws=wss://...&msgs=3
```

### car-overlay.html

**URL Parameters:**
| Param | Values |
|-------|--------|
| `car` | Car number |
| `ws` | WebSocket URL |
| `pos` | `top-left`, `top-center`, `top-right`, `bottom-left`, `bottom-center`, `bottom-right` |
| `style` | `full`, `compact`, `minimal` |

### flag-overlay.html

**URL Parameters:**
| Param | Values |
|-------|--------|
| `ws` | WebSocket URL |
| `pos` | Position (same as above) |
| `style` | `full`, `compact`, `minimal` |

### messages-overlay.html

**URL Parameters:**
| Param | Values |
|-------|--------|
| `ws` | WebSocket URL |
| `pos` | Position |
| `max` | Max messages (`1`, `3`, `5`, `10`) |
| `style` | `full`, `compact` |
| `time` | Show timestamps (`yes`/`no`) |

---

## OBS Setup

### Browser Source Settings
1. Add **Browser Source**
2. Set URL to local file or hosted URL with parameters
3. Width/Height suggestions:
   - Combined: 1920 x 1080
   - Car overlay: 900 x 150
   - Flag overlay: 400 x 80
   - Messages: 450 x 300
4. Check "Shutdown source when not visible" (saves resources)

All overlays have **transparent backgrounds**.

---

## Main Display Features (livetiming-display.html)

### Leaderboard
- Position, car number, driver name
- Gap to leader
- Lap count
- Sector times with color coding
- Last lap / Best lap times
- State indicators (RUN, PIT, FIN)

### Tracked Car Panel
- Click any car to track
- Enlarged view with all timing data
- Gap trend indicator
- Track position animation

### Messages Panel
- Race control messages
- Color-coded by type
- Auto-scrolling

### Statistics
- Session leader
- Best lap times
- Sector records

### WebSocket Recorder
Built-in recording/replay system:

**Recording:**
```javascript
wsRecorder.startRecording()   // Start capture
wsRecorder.stopRecording()    // Stop and save
wsRecorder.downloadRecording() // Manual download
```

**Replay:**
```javascript
wsRecorder.loadRecording(file)  // Load JSON file
wsRecorder.startReplay(speed)   // 0.5x - 50x speed
wsRecorder.stopReplay()
```

**Auto-save:** Every 5 minutes to localStorage, auto-downloads if >4MB

**Recording format:**
```json
{
  "version": 1,
  "recordedAt": "2026-01-09T12:00:00Z",
  "duration": 300000,
  "messageCount": 1234,
  "messages": [
    { "t": 0, "d": "{raw websocket message}" },
    { "t": 150, "d": "{...}" }
  ]
}
```

---

## Implementation Notes

### Timer Interpolation
The race timer counts down locally between server updates:
```javascript
let currentElapsed = raceElapsed;
if (raceElapsed && lastElapsedUpdateTime) {
  const timeSinceUpdate = (Date.now() - lastElapsedUpdateTime) * 1000;
  currentElapsed = raceElapsed + timeSinceUpdate;
}
remaining = raceDuration - currentElapsed;
```

### Lap Timer Reset
Current lap timer resets when:
1. `lastroundtime` value changes (primary trigger)
2. Lap count increases (backup trigger)

### Gap Calculation
- **Gap to leader:** Car's `diff` field
- **Gap ahead:** `ourDiff - carAheadDiff`
- **Gap behind:** `carBehindDiff - ourDiff`

### Track Progress Animation
Estimates position based on elapsed time and sector times:
```javascript
if (elapsed < s1) {
  position = (elapsed / s1) * 33.3;  // Sector 1
} else if (elapsed < s1 + s2) {
  position = 33.3 + ((elapsed - s1) / s2) * 33.3;  // Sector 2
} else {
  position = 66.6 + ((elapsed - s1 - s2) / s3) * 33.3;  // Sector 3
}
```

---

## Dependencies

- **LZString** - For decompressing bulk init messages
  ```html
  <script src="https://cdnjs.cloudflare.com/ajax/libs/lz-string/1.4.4/lz-string.min.js"></script>
  ```

- **Google Fonts** - Orbitron, Roboto Mono, Inter

---

## Known Issues / Future Work

- [ ] Tracker GPS position data (`t_p`) parsed but not fully utilized
- [ ] Class-based filtering for multi-class races
- [ ] Pit stop counter
- [ ] Driver stint tracking
- [ ] Fuel/tire strategy visualization

---

## Example WebSocket URL

```
wss://livetiming.getraceresults.com/lt/connect?transport=webSockets&clientProtocol=1.5&_tk=84992738a8304d15a8f20722b7a27947&_gr=w&_tkdm=1041140&connectionToken=FWT09RJEOFAHDbxfUngcCCb1h6UM6Yy0rx%2BBv0jXaVEC5KnepwdkWeHqZh58rBQmrV1hEAOpLQbfzrSUbMLBjHrrRU0NjN1kSovhFcPmOY%2BQxqWcOvb4ua56RpUtVb1O&tid=6
```

Note: Connection tokens expire - get fresh URL from the live timing page.
