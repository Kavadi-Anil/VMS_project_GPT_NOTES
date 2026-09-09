# NOP Vision Intelligence Project — Study Notes

## 1. Project in one sentence

**Video → Detection → Tracking → Zones → Events → Evidence → Dashboard**

The system takes a video, detects objects, keeps track of each object across frames, understands where objects move, identifies meaningful events, saves evidence, and presents results.

---

# 2. Core concepts

## Object Detection

Detection answers:

> **What objects are visible in this frame?**

The detector can return:
- Bounding box
- Confidence
- Object/class type

Example:

```text
Frame
  ↓
Detector
  ↓
Object 1: bbox + confidence
Object 2: bbox + confidence
```

### Bounding Box

A rectangle around an object.

Usually represented as:

```text
[x, y, width, height]
```

### Centroid

The center point of the bounding box.

The centroid can be used to determine which zone the object is inside.

---

# 3. Object Tracking

Detection works frame-by-frame. Tracking answers:

> **Is the object in this frame the same object I saw in the previous frame?**

Example:

```text
Frame 1: 📦 → ID 1
Frame 2:   📦 → ID 1
Frame 3:     📦 → ID 1
```

Tracking is important because events depend on an object's history.

---

# 4. Occlusion

**Occlusion = an object is temporarily hidden from the camera.**

Example:

```text
Frame 1:  📦
Frame 2:  🚚 📦  ← object hidden
Frame 3:  🚚    📦 ← visible again
```

A weak tracker may do:

```text
Frame 1 → ID 1
Hidden
Frame 3 → NEW ID 2
```

A stronger tracker tries to keep:

```text
Frame 1 → ID 1
Hidden → ID 1 maintained/predicted
Frame 3 → ID 1
```

The project uses Kalman prediction and occlusion coasting to help maintain tracks when detections temporarily disappear.

---

# 5. Dwell

**Dwell = how long one object remains in a particular zone.**

Example:

```text
Object enters QUEUE
        ↓
1 sec
2 sec
3 sec
4 sec
5 sec
        ↓
DWELL_THRESHOLD
```

If:

```json
"dwell_threshold_seconds": 5.0
```

then a dwell event can be triggered after the object remains long enough in the relevant zone.

### Real-world example

A person, vehicle, or package stays in a loading area for unusually long.

---

# 6. Queue

**Queue = multiple objects waiting in a defined area.**

Example:

```text
QUEUE ZONE

🚚
🚚
🚚
🚚

Queue occupancy = 4
```

If the configured threshold is exceeded, a queue alert/event can be generated.

### Dwell vs Queue

| Concept | Meaning |
|---|---|
| Dwell | How long **one object** stays |
| Queue | How many objects are **waiting together** |

---

# 7. Confidence

Confidence is primarily a **detector output**.

It represents the detector's score for how strongly it believes a detection matches the predicted object/class.

Important distinction:

```text
Confidence
  ↓
How strong is the detector's detection?

Tracking
  ↓
Which detection belongs to which existing object?

Hue
  ↓
Can color provide an additional matching clue?
```

## Starter detector confidence

The original baseline motion detector uses a fixed:

```python
"confidence": 0.50
```

This is **not a learned probability**. It is a baseline placeholder value.

A pretrained detector such as a YOLO/ONNX model can provide a model-generated confidence score.

Do not automatically interpret a score like `0.90` as a perfectly calibrated 90% probability of being correct; it is a model confidence score.

---

# 8. What if two objects have the same color?

Color/hue should be treated as an **auxiliary appearance cue**, not the primary identity mechanism.

Example:

```text
📦 RED       📦 RED
```

Hue cannot distinguish them.

The tracker can also use:

```text
Predicted position
+
Motion/velocity
+
Bounding-box information
+
Spatial distance
+
Hue/appearance when useful
```

The project's tracker uses Kalman prediction and Hungarian assignment, with hue gating as an additional matching constraint.

For difficult real-world scenes, stronger appearance features/embeddings or an appearance-based tracker could improve identity preservation.

### Good interview answer

> "Color is only an auxiliary appearance cue. If two objects have the same hue, hue gating cannot distinguish them. The tracker primarily relies on motion prediction and spatial association, while stronger appearance features can be added for harder crossings."

---

# 9. NMS — Non-Maximum Suppression

Sometimes a detector produces several overlapping boxes for the same object.

Example:

```text
       ┌─────────┐
     ┌─────────────┐
       ┌─────────┐
          📦
```

NMS removes duplicate overlapping detections and keeps the best candidate.

```text
Many overlapping boxes
        ↓
       NMS
        ↓
One useful box
```

---

# 10. Kalman Filter

The Kalman Filter helps predict an object's next position based on previous motion.

Example:

```text
Frame 1: x=100
Frame 2: x=110
Frame 3: x=120
```

Prediction:

```text
Frame 4 ≈ x=130
```

This helps when:
- The object moves between frames
- A detection is temporarily missing
- The tracker needs a predicted position for association

The project describes an 8-state constant-velocity model:

```text
[cx, cy, w, h, vx, vy, vw, vh]
```

where:
- `cx, cy` = center position
- `w, h` = bounding-box size
- `vx, vy` = position velocity
- `vw, vh` = size-change velocity

---

# 11. Hungarian Algorithm

When there are multiple existing tracks and multiple new detections, the system must decide which detection belongs to which track.

The Hungarian algorithm solves the **global assignment problem**.

Conceptually:

```text
Existing tracks             New detections

ID 1  ───────────────────→  Detection A
ID 2  ───────────────────→  Detection B
ID 3  ───────────────────→  Detection C
```

The matching cost can consider distance/prediction and other cues.

Simple meaning:

> **Find the best overall matching between old tracks and new detections.**

---

# 12. Hue Gating

Hue represents the basic color family of an object.

The project can use hue as an additional matching clue.

Example:

```text
Track 1 = RED
Detection = BLUE
```

If the hue difference is too large, that match can be heavily penalized.

But:

> **If both objects have the same color, hue gating cannot distinguish them.**

Then motion, predicted position, geometry, and other appearance information become more important.

---

# 13. Zone

A zone is an important spatial area in the video.

Example:

```text
┌──────────────┐        ┌──────────────┐
│              │        │              │
│      A       │        │      B       │
│              │        │              │
└──────────────┘        └──────────────┘
```

A zone can be represented as a rectangle:

```text
[x1, y1, x2, y2]
```

or a polygon:

```text
[[x1,y1], [x2,y2], [x3,y3], ...]
```

The zone manager determines whether an object's position is inside a zone.

---

# 14. Event

An event is a **meaningful action or behavior** detected from tracking and zone history.

Examples:

```text
A_TO_B
B_TO_A
ZONE_ENTRY
ZONE_EXIT
DWELL_THRESHOLD
QUEUE_THRESHOLD
```

The event engine converts raw observations into meaningful events.

Example:

```text
ID 7 → A
ID 7 → A
ID 7 → B
        ↓
     A_TO_B
```

---

# 15. Stateful Event Engine

The event engine remembers an object's journey.

Example:

```text
Object 7:

A
 ↓
QUEUE
 ↓
B
```

The engine can understand that as:

```text
A → B transit
```

even though the object passed through an intermediate zone.

It also handles:
- Journey history
- Reversal handling
- Debouncing
- Dwell timers
- Queue logic

---

# 16. Debouncing

An object near a zone boundary may rapidly switch between:

```text
A
B
A
B
A
B
```

because of tiny movements or detection noise.

Without debouncing, the system could generate many false events.

Debouncing means:

> **Don't treat every tiny boundary fluctuation as a real event.**

Wait for meaningful movement/state change.

---

# 17. Evidence

Evidence is the **proof of an event**.

For:

```text
A_TO_B
```

the system can store:

```text
Track ID
Event type
Timestamp
Confidence
Snapshot
Metadata
```

Typical outputs:

```text
events.jsonl
events.csv
counts.csv
evidence/
    event_001.jpg
    event_002.jpg
```

The project's evidence writer validates JSONL records against the NOP evidence contract.

---

# 18. JSONL

JSONL = JSON Lines.

Instead of one large JSON array, each line represents one event.

Example:

```json
{"track_id":1,"event_type":"A_TO_B"}
{"track_id":2,"event_type":"B_TO_A"}
```

Useful for event logs.

---

# 19. CSV

CSV is a spreadsheet-friendly format.

Example:

```text
event_type,count
A_TO_B,5
B_TO_A,2
DWELL_THRESHOLD,3
```

---

# 20. Analytics

Analytics calculates useful statistics such as:

### Dwell

```text
Object 1 = 3.2 sec
Object 2 = 7.4 sec
Object 3 = 5.8 sec
```

### Queue

```text
Current queue = 4
Peak queue = 7
```

### Velocity

```text
Object 1 = X pixels/sec
```

The project includes queue occupancy, dwell-time distributions and pixel velocity analytics.

---

# 21. Seven-Layer Architecture

The system follows this pipeline:

```text
1. VIDEO INGESTION
        ↓
2. DETECTION
        ↓
3. TRACKING
        ↓
4. ZONE / SPATIAL GEOMETRY
        ↓
5. EVENT REASONING
        ↓
6. EVIDENCE / EXPORT
        ↓
7. ANALYTICS / DASHBOARD
```

Easy way to remember:

> **READ → SEE → REMEMBER → LOCATE → UNDERSTAND → PROVE → SHOW**

---

# 22. Layer 1 — Video Ingestion

Technology:

```text
OpenCV VideoCapture
```

Responsibilities:
- Open video
- Read frames
- Get FPS
- Get resolution
- Track frame/time position

Conceptually:

```python
cap = cv2.VideoCapture(video_path)
ret, frame = cap.read()
```

The frame is passed to the next layer.

---

# 23. Layer 2 — Perception / Detection

Input:

```text
Frame
```

Output:

```text
Bounding boxes
+
Confidence
+
Object/class information
+
Visual features such as hue (where applicable)
```

The current project documentation describes:
- `AdaptiveDetector`
- `PretrainedDnnDetector`
- NMS
- color/saliency and MOG2-based detection
- dominant hue extraction

---

# 24. Layer 3 — Multi-Object Tracking

Input:

```text
Detections
```

Output:

```text
Track IDs
+
Predicted/updated bounding boxes
+
Motion history
```

Main ideas:
- Kalman prediction
- Hungarian assignment
- Track lifecycle
- Hue gating
- Occlusion coasting

---

# 25. Layer 4 — Spatial Geometry

Input:

```text
Track ID + position
```

Output:

```text
Current zone
```

Example:

```text
ID 7 → A
ID 8 → QUEUE
ID 9 → B
```

---

# 26. Layer 5 — Stateful Event Reasoning

Input:

```text
Track history + zone history + timestamps
```

Output:

```text
A_TO_B
B_TO_A
DWELL
QUEUE
etc.
```

This is the layer that turns movement into business meaning.

---

# 27. Layer 6 — Evidence / Export

Input:

```text
Confirmed event
```

Output:

```text
JSONL
CSV
Snapshot
Annotated evidence
```

Purpose:

> **Make the event traceable and reviewable.**

---

# 28. Layer 7 — Visual Review / Analytics

Outputs:
- Annotated video
- Event timeline
- Evidence snapshots
- Dwell statistics
- Queue graphs
- Dashboard

Streamlit is used for the interactive review dashboard.

---

# 29. End-to-End Data Flow Example

Imagine a package moves from Zone A to Zone B.

### Step 1 — Video

```text
Camera/video
 ↓
Frame
```

### Step 2 — Detection

```text
Detector sees:
📦
bbox = [100,200,50,50]
confidence = 0.90
```

### Step 3 — Tracking

```text
Tracker:
ID = 7
```

### Step 4 — Zone

```text
ID 7 → A
```

Next frames:

```text
ID 7 → A
ID 7 → A
ID 7 → B
```

### Step 5 — Event

```text
A_TO_B
```

### Step 6 — Evidence

```text
events.jsonl
counts.csv
event_001.jpg
```

### Step 7 — Dashboard

```text
Event:
Object 7
A → B
Timestamp: XX sec
Evidence available
```

---

# 30. Configuration — What is it?

A configuration file is basically:

> **Settings/instructions for how the algorithms should operate for a particular video or scenario.**

Think:

```text
Python code:
"What should the system do?"

JSON config:
"What settings should it use?"
```

The config does **not** contain the core algorithm.

---

# 31. Default Configuration

Example:

```json
{
  "scenario_id": "GENERIC_VIDEO",
  "camera_id": "challenge_cam_01",
  "min_area": 1000,
  "max_area": 30000,
  "max_match_distance": 130,
  "max_coasting_frames": 60,
  "dwell_threshold_seconds": 5.0,
  "terminal_zones": ["A", "B"],
  "zones": {
    "A": [50, 80, 420, 640],
    "B": [860, 80, 1230, 640]
  }
}
```

## `scenario_id`

Identifies the scenario/configuration.

```text
GENERIC_VIDEO
```

means a general/default setup.

S01 uses:

```text
S01_BASIC_GOODS
```

meaning the configuration is intended for S01.

---

## `camera_id`

Identifies the source camera.

Example:

```text
challenge_cam_01
```

Useful when multiple cameras exist.

---

## `min_area`

Minimum detected region size.

If too small:

```text
Noise / tiny blob
        ↓
IGNORE
```

Increasing it can reduce small false detections.

---

## `max_area`

Maximum accepted detected region size.

Very large regions can be ignored if they are unlikely to be valid objects.

---

## `max_match_distance`

Tracking parameter.

It controls how far a detection can be from the predicted/existing track position and still be considered a possible match.

Example:

```text
Old position = x=300
New detection = x=350

Distance = 50
max_match_distance = 130

50 < 130
→ possible match
```

If movement is very large, the value may need adjustment.

---

## `max_coasting_frames`

Controls how long a track can survive while the object is temporarily not detected.

Example:

```text
Detected
 ↓
Hidden
 ↓
Hidden
 ↓
Detected again
```

With enough coasting, the tracker can maintain the original identity.

At 30 FPS:

```text
60 frames ≈ 2 seconds
```

---

## `dwell_threshold_seconds`

How long an object must remain in the relevant area before a dwell event is triggered.

```text
5.0
```

means 5 seconds.

---

## `terminal_zones`

These are the important end zones for completed journeys.

```json
"terminal_zones": ["A", "B"]
```

So:

```text
A → B
B → A
```

can be treated as completed terminal-zone movements.

---

## `zones`

Defines where the zones are located in the video.

Rectangle:

```text
[x1, y1, x2, y2]
```

Example:

```text
A = [50,80,420,640]
```

means approximately:

```text
left   = 50
top    = 80
right  = 420
bottom = 640
```

---

# 32. Why is S01 Config Different?

S01 has:

```json
{
  "scenario_id": "S01_BASIC_GOODS",
  "camera_id": "challenge_cam_01",
  "min_area": 1200,
  "max_area": 25000,
  "max_match_distance": 140,
  "max_coasting_frames": 65,
  "terminal_zones": ["A", "B"],
  "zones": {
    "A": [80, 120, 430, 620],
    "B": [850, 120, 1200, 620]
  }
}
```

The differences simply mean:

> **S01 uses slightly different parameters and zone coordinates than the generic configuration.**

For example:

```text
Default min_area = 1000
S01 min_area     = 1200
```

S01 is slightly stricter about small detections.

```text
Default max_area = 30000
S01 max_area     = 25000
```

S01 limits maximum detection size more.

```text
Default max_match_distance = 130
S01     max_match_distance = 140
```

S01 allows slightly more movement between frames for matching.

```text
Default coasting = 60
S01     coasting = 65
```

S01 allows slightly longer track persistence.

Most importantly, the **zone coordinates are different because the zone locations are configuration-specific**.

---

# 33. Config vs Code

This distinction is extremely important.

### Code

```text
tracker.py
```

contains the tracking algorithm.

### Config

```json
"max_match_distance": 140
```

tells that tracker the matching-distance setting.

Similarly:

```text
event_engine.py
```

contains dwell/event logic.

```json
"dwell_threshold_seconds": 5.0
```

sets the dwell threshold.

And:

```text
zones.py
```

contains zone geometry logic.

```json
"A": [80,120,430,620]
```

defines where Zone A is.

---

# 34. If the Interviewer Says "Modify the Code"

Do **not** randomly change parameters.

First observe the problem, identify the likely cause, then make the smallest justified change.

## Case 1 — Too many small false detections

Problem:

```text
Background noise → detections
```

Possible config change:

```json
"min_area": 1800
```

Explanation:

> "I increased the minimum detection area because small background regions were being interpreted as objects."

---

## Case 2 — Fast objects lose IDs

Possible adjustment:

```json
"max_match_distance": 180
```

Explanation:

> "The objects move significantly between frames, so I increased the allowable association distance."

---

## Case 3 — Object disappears behind an obstacle

Possible adjustment:

```json
"max_coasting_frames": 90
```

Explanation:

> "I increased the coasting period so the tracker can maintain the identity during temporary occlusion."

---

## Case 4 — Dwell triggers too early

Change:

```json
"dwell_threshold_seconds": 10.0
```

if the actual requirement is 10 seconds.

---

## Case 5 — Zones are wrong

Modify:

```json
"zones": {
    "A": [...],
    "B": [...]
}
```

to match the actual areas in the new video.

---

# 35. What If the Interviewer Wants Actual CODE Modification?

If they say:

> "Your tracker is losing IDs during crossing. Fix it."

Don't simply increase a threshold without understanding the cause.

A stronger approach is:

> "The problem appears to be an association failure during close crossing. I'll improve the matching cost using predicted position and appearance/color information, then validate it on the dense-crossing scenario."

This demonstrates engineering reasoning.

---

# 36. Recommended Project Improvements

The existing system should be **upgraded modularly**, not blindly replaced.

## 1. Better semantic detection

Move from generic motion detection toward a detector that can understand:

```text
Person
Vehicle
Goods
Package
```

This makes the system more useful for real-world warehouse scenarios.

## 2. Better tracking

Improve robustness for:
- Occlusion
- Dense crossing
- Same-color objects
- ID switches
- Fast movement

Potential direction:

```text
Kalman + global assignment + appearance features
```

or an established multi-object tracker.

## 3. Stronger event intelligence

Add:

```text
A_TO_B
B_TO_A
DWELL
QUEUE
OCCUPANCY
RESTRICTED_ZONE
ZONE_ENTRY
ZONE_EXIT
```

## 4. Goods Counting + Manifest Reconciliation

Strong business-oriented extension:

```text
Expected bags = 120
Detected bags = 114
        ↓
Shortage = 6
        ↓
ALERT
```

This connects computer vision to a real warehouse business problem.

## 5. Stronger evidence

Add richer metadata and, if appropriate, tamper-evident mechanisms such as hash chaining.

Goal:

> Make every important event traceable back to visual evidence.

## 6. Event Search

Allow operators to search structured events:

```text
"Show A_TO_B events"
"Show dwell events above 5 seconds"
"Show events between frame 200 and 500"
```

Start with deterministic structured search before adding an LLM/VLM.

## 7. Multi-camera dashboard

Eventually:

```text
┌──────────┬──────────┬──────────┐
│ Camera 1 │ Camera 2 │ Camera 3 │
├──────────┼──────────┼──────────┤
│ Camera 4 │ Camera 5 │ Camera 6 │
└──────────┴──────────┴──────────┘
```

Useful for a real warehouse/control-room scenario.

---

# 37. Recommended Priority

Do not try to build the entire enterprise NOP Pro+ platform.

Prioritize:

```text
1. Detection quality
        ↓
2. Tracking quality
        ↓
3. Event correctness
        ↓
4. Evidence quality
        ↓
5. Goods counting / manifest reconciliation
        ↓
6. Evaluation + performance
        ↓
7. Dashboard/search
```

Accuracy and robustness are more important than adding many flashy features.

---

# 38. Submission Structure

Final submission should look roughly like:

```text
candidate-solution/
│
├── README.md
├── REPORT.md
├── requirements.txt
│
├── src/
│
├── config/
│
├── output/
│   ├── annotated.mp4
│   ├── events.jsonl
│   ├── counts.csv
│   └── evidence/
│
└── tests/
```

### README

Explain:
- OS tested
- Python/runtime version
- Setup commands
- Run command
- Input format
- Output locations
- Hardware
- Limitations

### REPORT

Explain:
- Problem selected
- Architecture
- Detector
- Tracker
- Why those choices
- Evaluation method
- Measured accuracy
- FPS/processing time
- Failure cases
- Future improvements
- AI tools used
- Third-party components and licenses

**Do not invent benchmark numbers. Run the system and record actual results.**

---

# 39. Interview — Unseen Video Playbook

If the interviewer gives a new video:

## Step 1 — Inspect

Check:

```text
Resolution
FPS
Frame count
What is in the video?
Synthetic or real CCTV?
```

## Step 2 — Configure

Create an unseen-video config and adjust:

```text
Zones
Detection thresholds
Tracking parameters
Dwell threshold
```

## Step 3 — Run with display

Use the live display first so you can visually inspect:

```text
Bounding boxes
Track IDs
Trails
Zones
HUD
```

## Step 4 — Tune based on evidence

Examples:

```text
Too much noise
→ increase min_area

Fast objects
→ increase matching distance

Long occlusion
→ increase coasting frames

Real people/vehicles
→ use semantic DNN detector
```

## Step 5 — Show dashboard

Use the Streamlit dashboard to demonstrate:

```text
Annotated video
Event timeline
Evidence snapshots
Dwell analytics
Queue occupancy
```

---

# 40. Strong Interview Architecture Answer

If asked:

> **"Explain your architecture."**

You can say:

> "The system is a modular video analytics pipeline. First, OpenCV ingests the video frame by frame. The detector identifies objects and produces bounding boxes and confidence scores. The tracker assigns persistent IDs using motion prediction and global association. The zone manager determines the spatial region occupied by each track. The stateful event engine uses track and zone history to detect meaningful events such as A-to-B movement, reversals, dwell and queue behavior. Confirmed events are written as structured evidence with snapshots and CSV/JSONL outputs. Finally, analytics and the Streamlit dashboard provide visual review and operational metrics."

---

# 41. The Diagram to Memorize

```text
                    VIDEO
                      │
                      ▼
             ┌────────────────┐
             │ VIDEO INGESTION│
             │    OpenCV      │
             └───────┬────────┘
                     │ Frames
                     ▼
             ┌────────────────┐
             │   DETECTION    │
             │ Adaptive / DNN │
             └───────┬────────┘
                     │
              BBoxes + Confidence
                     │
                     ▼
             ┌────────────────┐
             │    TRACKING    │
             │ Kalman +       │
             │ Hungarian      │
             └───────┬────────┘
                     │
              Track IDs + Motion
                     │
                     ▼
             ┌────────────────┐
             │     ZONES      │
             │ A / B / QUEUE  │
             └───────┬────────┘
                     │
               Current Zone
                     │
                     ▼
             ┌────────────────┐
             │ EVENT ENGINE   │
             │ History/State  │
             └───────┬────────┘
                     │
            A→B / B→A / Dwell / Queue
                     │
                     ▼
             ┌────────────────┐
             │    EVIDENCE    │
             │ JSONL / CSV /  │
             │ Snapshots      │
             └───────┬────────┘
                     │
                     ▼
             ┌────────────────┐
             │   ANALYTICS &  │
             │   DASHBOARD    │
             └────────────────┘
```

## Final mental model

Remember:

```text
DETECT   = What do I see?
TRACK    = Which object is it?
ZONE     = Where is it?
EVENT    = What happened?
EVIDENCE = Can I prove it?
ANALYTICS = What does the data tell me?
DASHBOARD = How do I show it?
```

If you understand this flow, you understand the core architecture. Kalman, Hungarian, NMS, point-in-polygon, debouncing and hue gating are techniques used inside those stages.
