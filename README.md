# Flight Viewer

A single-file, standalone 3-D flight viewer for paragliding / hang-gliding
tracks. Open `flight_viewer.html` in any modern browser, drop in an `.igc`,
`.gpx`, or `.kml` file, and fly the track over real 3-D terrain with
vario-coloured trail, altitude / vario / speed charts, flight statistics
with XC scoring, thermal detection and wind estimation, live HUD
(including AGL and glide ratio), multi-track comparison replay, OpenAir
airspace overlay, and full playback control. Settings and recent flights
persist locally. No build step, no server, no account.

**Try it online:**
<https://skywalker1905.github.io/paragliding_flight_3D_viewer/> — open
the page and drop in your own flight file. Nothing is uploaded; all
processing stays in your browser.

## Screenshots

Landing screen — waiting for a file:

![Landing screen](images/open_file.png)

Loaded flight — full GUI:

![Loaded GUI](images/GUI.png)

## Quick start

1. Open `flight_viewer.html` in any modern browser (Chrome, Edge, Firefox,
   Safari ≥ 16). Internet access is required (see [Online requirement](#online-requirement)).
2. Drag a flight file onto the window, or click **Open** in the bottom bar.
3. Press space to play / pause, scrub the timeline (hover for a time
   bubble), drag on the chart, or press `?` for all keyboard shortcuts.

The HTML contains zero embedded flight data — every open viewer starts on
the landing screen and waits for user input. Loading a new file always
loads cleanly over the previous one. Flights you load are remembered in
the browser's local IndexedDB and offered as one-tap **Recent flights**
on the landing screen (local only, never uploaded).

## Online requirement

`flight_viewer.html` is self-contained code, but it streams the following
resources from public CDNs and tile providers on first open and as you pan
around:

| Resource | Source | Purpose |
|----------|--------|---------|
| MapLibre GL JS 4.7 | `unpkg.com` | Map rendering engine |
| deck.gl 9.0 | `unpkg.com` | Track / pilot overlays, interleaved with MapLibre |
| Plotly 2.35 | `cdn.plot.ly` | Altitude / ground chart |
| `tz-lookup` 6.1 | `unpkg.com` | Coord → IANA timezone (used for DST-correct local time) |
| World Imagery (satellite basemap) | `services.arcgisonline.com` (Esri) | Visible map tiles |
| OpenStreetMap (street basemap) | `tile.openstreetmap.org` | Alternative map tiles |
| Terrarium DEM (3-D terrain + chart's ground line) | `s3.amazonaws.com/elevation-tiles-prod/terrarium/` (Mapzen / AWS Open Data, CC0) | Elevation, sampled directly at zoom 14 (~10 m / pixel) for ridge-accurate ground-altitude readouts |

If you open the viewer with no internet access, the map area will be blank
and the chart's ground trace will be empty. The track itself, the
playback, the HUD, and the spline geometry still work because they're
computed from the loaded `.igc` / `.gpx` / `.kml` alone. Timezone display
falls back to a longitude-only UTC offset (no DST) when `tz-lookup`
cannot load.

To replay a specific flight with **no network at all**, pre-download its
map with settings → **Offline map** while online and load the resulting
`.fvmap` file next time (see [Offline maps](#offline-maps)). Note the
JS libraries themselves still come from CDNs, so the browser must have
cached them or have been opened online at least once.

## Features

### Map and terrain
- Satellite (Esri World Imagery) or OSM street basemap, toggle in the
  drawer.
- Real 3-D terrain from the Mapzen / AWS **Terrarium** DEM, with
  adjustable exaggeration (1× – 3×, default 1×).
- Standard pan / rotate / pitch / zoom. **Swap L/R mouse** (on by
  default): left-drag rotates the 3-D view, right-drag pans — better
  for touchpads. Turn off to restore MapLibre's left-pan / right-rotate
  layout. Middle-wheel zoom is unchanged.
- Slower default scroll / trackpad zoom rates; map **+ / −** controls
  use instant zoom steps so they still work while **Follow pilot** is on
  (animated zoom would be cancelled by per-frame recentering).

### Track
- **Centripetal Catmull-Rom spline** (α = 0.5, Barry–Goldman evaluation
  in a local metre frame) through the original IGC fixes — the recorded
  points are never moved, only the connecting curve is smoothed. The
  centripetal parameterisation is overshoot- and cusp-free by
  construction, so thermal circles and long glides both come out fair
  with nothing to tune. A **Rounding** slider (0 – 100 %) blends between
  raw chords and the full spline.
- Per-vertex **vario colouring** (red = climb, blue = sink, green ≈ 0),
  with adjustable max climb / sink (default ±2 m/s), or **single
  colour** (default pure green).
- Adjustable pixel width (`Track width`) and optional metres-wide 3-D
  ribbon (`3-D thickness`) so the track reads as a solid body from any
  angle.
- The trail **ends exactly under the pilot marker** — the boundary
  fix-to-fix segment is re-sampled along the spline each frame so there
  is no overshoot, even on low-rate IGCs.
- **Track-ahead toggle** in the bottom bar: hides the unflown portion
  by default; click to show the full track at full opacity.
- **Track fade** slider (settings): seconds of fully-bright trail
  behind the pilot before fading to transparent. `0` = off (whole past
  stays bright). Step auto-scales to ~10 % of the current value; max
  stretches to the full flight duration.

### Spiral reconstruction
- **Model overlay** for thermal spirals. At 1 Hz sampling, a paraglider
  rotating faster than 0.5 turns/sec aliases into a zig-zag that no simple
  spline can recover.
- Synthesizes a smooth helix that passes *exactly* through every original
  GPS fix, using a sliding centroid to capture actual terrain-relative drift.
- **Auto mode**: detects spirals based on a continuous heading change threshold.
- **Manual mode**: lets you define the exact time window of a spiral.
- **Turns / sec**: slider to set the expected rotation rate, resolving the
  hidden integer turns between 1 Hz samples.

### Pilot marker
- Three shapes (settings **Pilot shape**):
  - **Cylinder** (default) — 3-D extruded marker: white inner puck,
    magenta outer annulus band, and red forward wedge. Opaque matte
    finish; constant on-screen pixel size at any zoom via `mPerPx`
    scaling (same scheme as Arrow / Wing).
  - **Arrow** — flat black-ringed disc with a triangular nose, always
    drawn on top of the track.
  - **Wing** — elongated ellipse canopy perpendicular to flight
    direction.
- **Pilot size** slider: 0.1 – 1× (default 0.5×).
- **Altitude line** (on by default): a thin vertical line from the pilot
  down to the terrain plus a soft shadow disc at its foot. Both are
  depth-tested (they hide behind ridges), so height above ground reads
  at a glance in the 3-D view.
- Heading follows the *smoothed* spline tangent, so the nose / wedge
  tracks the curve even during rapid thermal turns.

### Camera
- **Follow pilot** is **on by default** after load. The map recentres on
  the live marker every frame using an analytical ground-offset model
  (stable at high pitch, no pan-by oscillation).
- While following: **right-click drag** orbits around the pilot,
  **wheel / trackpad** zooms around the pilot, and **left-click** (or
  pan drag, depending on swap setting) exits follow. Toggle follow in
  settings, with the **⊕** map control, or **`C`**.
- **Fit track** (`R`) or the **Fit** button: top-down view containing
  the whole track (fitBounds is only reliable at pitch 0). Temporarily
  turns follow off. Manual Fit uses generous padding; the **opening
  view** uses a tighter fit plus a little extra zoom, then eases to
  65° pitch and re-enables follow.
- **Takeoff** button flies to the launch with a 3-D tilt and starts
  playback from the first fix.
- **Center on pilot** (`C` / ⊕) toggles follow mode (not a one-shot
  recenter).
- **Camera style** (settings): **Center** keeps the pilot centred with
  the heading under your control; **Chase** eases the camera bearing
  toward the direction of flight for a follow-cam feel; **Orbit**
  slowly circles the pilot while following (adjustable speed) to show
  the 3-D track from every side. Dragging always takes over manually.

### Chart
- Three metrics, switchable in the chart header: **Alt** (pilot altitude
  in blue + earth-brown ground altitude with soft fill — the visible gap
  is AGL), **Vario** (smoothed climb / sink), and **Speed** (ground
  speed).
- Ground altitude is fetched directly from Terrarium tiles at zoom 14
  (~10 m / pixel) — independent of the camera state, so ridge tops
  aren't undersampled the way they are with the map's render-time DEM.
- Y-axis range = **union** of pilot and ground extents, so the ground
  line stays in frame even when terrain rises above the pilot's path.
- X-axis is rendered in the **launch site's local clock time** with
  DST applied via `tz-lookup` + `Intl.DateTimeFormat`. Falls back to a
  longitude-only UTC offset when `tz-lookup` is unavailable.
- Click to seek, or **press and drag** (mouse or touch) to scrub
  continuously; the cursor line shows the current playback time.
- Draggable header, resizable corner (grip shown bottom-right).

### Flight statistics & thermals
- **Stats panel** (`I` or the bar-chart button): takeoff / landing time
  and altitude, airtime, max altitude (with time), max climb / sink,
  max speed, total ascent, track length, straight distance, max
  distance from takeoff — plus IGC header info (pilot, glider, site)
  when present.
- **Thermal detection**: sustained-circling segments (≥ 540° of
  rotation with altitude gain) are listed with duration, turns, gain
  and average climb — click one to jump straight to it. Thermals also
  appear as orange spans under the timeline, alongside green (takeoff)
  and blue (max altitude) ticks.
- **Wind estimation**: while circling, the mean drift of the track is
  fitted per thermal; the HUD shows the estimate nearest to the current
  playback time (speed, arrow, and the direction the wind blows *from*).

### HUD
- Live time (launch-local, DST-aware), altitude, **AGL** (from the DEM
  directly below the pilot), ground speed, vario, **glide ratio (L/D)**,
  **wind estimate**, cumulative distance.
- Draggable anywhere on screen.

### Playback
- Play / pause with `Space`, restart, scrub via the timeline; hovering
  or dragging the timeline shows a **time bubble** (local clock +
  elapsed).
- **Start at takeoff** (settings, on by default): playback begins at the
  detected takeoff instead of the minutes of standing around that IGC
  loggers usually record.
- Speed control 0.1× – 120×: drag the small slider, scroll the wheel
  (up = faster, 1 step / Shift = 10 steps), single-click the readout to
  cycle common presets (0.5 / 1 / 2 / 5 / 10 / 30 / 60 / 120×),
  double-click the readout to reset to 1×.
- Current and total time both shown in `HH:MM:SS`.
- Bottom playback bar is fully draggable.

### Comparison tracks
- Load up to six extra IGC / GPX / KML files (settings → **Comparison
  tracks**) and replay them alongside the main flight. Every track gets
  the same vario colouring and progressive reveal as the main one;
  pilots are told apart by their marker colours and name labels.
- **Align by Takeoff** starts every track at its own detected takeoff,
  synchronised to the main flight's takeoff (good for comparing
  different days); **Align by Clock** places tracks at their true
  recorded times (good for same-day / competition replays).
- **Pilot name labels** float above every marker in multi-track mode.
  Rename them in settings → Comparison tracks, or simply click a label
  on the map while paused and type (CJK names supported).

### Airspace overlay
- Load an **OpenAir** (`.txt`) airspace file (settings → **Airspace**).
  Zones are draped over the 3-D terrain, colour-coded by class
  (P/R/Q red, D amber, CTR blue, …). Click / tap a zone for its name,
  class and vertical limits. Supports `DP`, `DC`, `DB`, `DA` and
  `V X=/D=` records.

### Export & share
- **Screenshot**: save the current 3-D view as a JPEG (map + track
  only). Camera button in the transport bar, or settings → Export.
  Filenames carry a timestamp so repeated captures never collide.
- **Record video**: red-dot button in the transport bar (pulses while
  recording), click again to stop and save. The recording composites the
  3-D map **plus the Live HUD and the chart** whenever they're open —
  transport buttons, sliders and settings are excluded. H.264 MP4 is
  preferred so files open in QuickTime / stock players; browsers that
  can only encode VP8/VP9 fall back to `.webm` (open those with VLC).
  Hidden where unsupported.
- **Saving**: on desktop, recordings and screenshots download straight
  into the browser's download folder. On phones the system share sheet
  opens instead — tap *Save Video* / *Save Image* to put the file in
  the photo album (browsers cannot write there directly).
- **Copy link**: when served over HTTP(S), copies a URL with the current
  playback position (`#t=seconds`) embedded — opening it jumps straight
  to that moment.
- **URL parameters**: when the page is hosted on a web server,
  `?track=<url>` auto-loads a flight file on open and `&map=<url>`
  preloads an offline map bundle (same-origin or CORS-enabled URLs).

### Offline maps
- **Save map file** (settings → Offline map): downloads every basemap +
  terrain-DEM tile covering the loaded flight (zoom 8 up to 16, capped
  at ~2 400 tiles) and saves them as a single `.fvmap` file.
- Load a `.fvmap` through the normal **Open** button or drag-and-drop —
  before or after the flight file — and the same flight replays fully
  offline: basemap, 3-D terrain, AGL and the chart's ground line all
  come from the bundle.

### Settings drawer
Collapsible sections (open/closed state remembered):
- **Basics** — altitude source (GPS / baro), basemap, units, follow
  pilot, camera style (Center / Chase), start at takeoff, swap L/R
  mouse, Takeoff / Fit camera shortcuts.
- **Track & display** — terrain exaggeration, Alt offset (slider +
  numeric box + **Auto** DEM alignment), track width, 3-D thickness,
  smoothing subdivisions, rounding, KML sample rate.
- **Spiral reconstruction** — On/Off, Auto / Manual mode, turns / sec,
  detect threshold, manual range.
- **Trail & pilot** — track fade, colour mode + max climb / sink, pilot
  shape / size, altitude line.
- **Comparison tracks**, **Airspace**, **Offline map**, **Export &
  share** — see above.

Universal slider ergonomics: double-click / double-tap any slider to
reset it to its default; mouse-wheel nudges by `step` (Shift = ×10,
Ctrl = ×100). Click outside the drawer to close it. A **Reset all**
button in the drawer header restores every setting (and the saved
panel layout) to factory defaults.

Floating panels (HUD, stats, chart) automatically move out of each
other's way and snap to screen edges / neighbouring panels when
dragged close.

### Persistence
- All preferences (units, basemap, colours, speed, chart state, camera
  style, …) are saved to `localStorage` and restored on the next visit.
- Panel positions (HUD, stats, chart, bottom bar) and the chart's size
  survive reloads too.
- Recently loaded flights are kept in IndexedDB and listed on the
  landing screen for one-tap reload.

### Mobile
- Responsive portrait layout **and** a dedicated landscape layout for
  phones (compact single-row transport bar).
- Transport bar auto-hides during playback on touch devices
  (video-player style); tap anywhere to bring it back.
- **Screen wake lock** while playing, and a battery saver that drops
  the render loop to a slow poll while paused.
- Works in iOS WebViews (e.g. the Documents app) — fullscreen control
  and tap handling are adapted automatically.

## Keyboard shortcuts

| Key | Action |
|-----|--------|
| `Space` | Play / pause |
| `←` / `→` | Seek −10 s / +10 s (Shift = 60 s) |
| `↑` / `↓` | Playback speed up / down (Shift = 10 steps) |
| `Home` / `End` | Jump to start / end |
| `R` | Fit track (top-down) |
| `C` | Toggle follow pilot |
| `F` | Toggle map fullscreen |
| `O` | Open another file |
| `S` | Toggle the settings drawer |
| `I` | Toggle the flight-stats panel |
| `L` | Toggle the live HUD |
| `?` | Keyboard-shortcut overlay |
| `Esc` | Close overlay / settings / stats, exit fullscreen |

## Supported file formats

| Format | Notes |
|--------|-------|
| `.igc` | IGC paragliding / sailplane fixes (B records). GPS and baro altitude both supported. Header info (pilot, glider, site) is shown in the stats panel. Flights crossing UTC midnight are handled correctly. |
| `.gpx` | Standard `<trkpt>` track-points, with elevation. |
| `.kml` | Google Earth-style `<gx:Track>` / `<LineString>` (synthetic sample rate adjustable for time-less LineStrings). |
| OpenAir `.txt` | Airspace files, loaded separately via settings → Airspace. |
| `.fvmap` | Offline map bundle created by this viewer (settings → Offline map); load via Open / drag-and-drop. |

## Privacy

Flight files are processed entirely in the browser. They never leave
your machine; the viewer's only network traffic is the basemap / DEM /
library requests listed in [Online requirement](#online-requirement).
Preferences (`localStorage`) and recent flights (IndexedDB) are stored
locally in the browser and can be cleared with the site data.

## Browser support

Tested on recent Chrome, Edge, Firefox and Safari. WebGL 2 required.
Phones and tablets are fully supported (portrait + landscape layouts,
touch scrubbing, auto-hiding controls, wake lock).

## Performance notes

- Spline geometry is cached; only colour / alpha attributes update each
  frame. The per-frame fade and the dynamic trail-tip are essentially
  free.
- DEM ground-altitude sampling buckets fixes by tile so each Terrarium
  tile is fetched once and decoded once; a typical XC flight covers
  ~10 – 50 unique tiles (~500 KB – 2.5 MB total, downloaded in
  parallel and progressively painted into the chart).

## Credits

- Map rendering: [MapLibre GL JS](https://maplibre.org/) (BSD 3-Clause)
- Overlays: [deck.gl](https://deck.gl/) (MIT)
- Charts: [Plotly.js](https://plotly.com/javascript/) (MIT)
- Timezone lookup: [tz-lookup](https://github.com/darkskyapp/tz-lookup-oss) (CC0, archived 2020; viewer loads v6.1.25 from `unpkg.com`)
- Satellite basemap: © Esri World Imagery
- Street basemap: © OpenStreetMap contributors
- Terrain DEM: Mapzen / AWS Open Data Terrarium tiles (CC0)

## License

Released under the [MIT License](LICENSE). © 2026 skywalker1905.
