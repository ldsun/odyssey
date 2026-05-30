# nuScenes — Token Walkthrough

nuScenes is built on **tokens** (UUID strings). Each token is the primary key of one row in a JSON table under `v1.0-mini/`. Rows **cross-reference** other rows by token — there are almost no file paths until you reach `sample_data.filename`.

**Mental model:** a small relational database in RAM (`NuScenes` devkit) + a blob store on disk (`samples/`, `sweeps/`).

---

## How lookup works

```python
from nuscenes import NuScenes

nusc = NuScenes(version="v1.0-mini", dataroot=str(NUSCENES_DATA_ROOT), verbose=False)

row = nusc.get("<table_name>", "<token>")   # one JSON row
```

---

## The six things you usually need

For one training instant:

1. A **continuous recording** (AV2: log)
2. **Ego state** at that instant
3. **Calibration** (sensor→ego extrinsics; camera intrinsics for cameras)
4. **Annotations** (3D boxes)
5. **Camera** raw data
6. **Lidar** raw data

In nuScenes you walk **top-down by token**, then fan out from one **`sample`** (keyframe).  
Two names for “recording”: **`log.json`** (drive session) and **`scene.json`** (~20 s clip cut from that session). For py123d / AV2 comparison, **`scene-0061` ≈ one AV2 log**.

---

## Vertical walk — token chain with row examples

Follow **`→`** = “look up this token in the next table”.  
Example rows are from **`v1.0-mini`**, `scene-0061`, keyframe 0.

```
log.json
  │  scene.log_token  (↑ parent session)
  ▼
scene.json
  │  scene.first_sample_token
  ▼
sample.json                    ← hub: one keyframe @ 2 Hz
  │
  ├─ sample.data["LIDAR_TOP"]
  │       ▼
  │   sample_data.json
  │       ├─ filename ──────────────────────► samples/LIDAR_TOP/….pcd.bin
  │       ├─ ego_pose_token ──► ego_pose.json
  │       └─ calibrated_sensor_token ──► calibrated_sensor.json ──► sensor.json
  │
  ├─ sample.data["CAM_FRONT"]
  │       ▼
  │   sample_data.json
  │       ├─ filename ──────────────────────► samples/CAM_FRONT/….jpg
  │       └─ calibrated_sensor_token ──► calibrated_sensor.json  (extrinsics + intrinsics)
  │
  └─ sample.anns[0]
          ▼
      sample_annotation.json
          └─ instance_token ──► instance.json ──► category.json
```

---

### Step 1 — `log.json` · recording session

The table of all drive sessions in the “database” — each row is one continuous recording; the ~20 s scenes in `scene.json` are cut from those sessions (`scene.log_token` points back here).

```
log.json        one drive session (can be minutes long)
  └── scene.json   ~20 s clip(s) cut from that drive
        └── sample.json   keyframes @ 2 Hz
```

`log.json` is the **top** of this tree. Each JSON pack (`v1.0-mini`, `v1.0-trainval`, …) has its own `log.json` listing only the logs in that split — not the entire nuScenes dataset (mini: **8** logs, **10** scenes).

```python
scene = nusc.scene[0]
log = nusc.get("log", scene["log_token"])
```

**Example row** (`nusc.get("log", scene["log_token"])`):

```json
{
  "token": "7e25a2c8ea1f41c5b0da1e69ecfa71a2",
  "logfile": "n015-2018-07-24-11-22-45+0800",
  "vehicle": "n015",
  "date_captured": "2018-07-24",
  "location": "singapore-onenorth",
  "map_token": "53992ee3023e5494b90c316c183be829"
}
```

| Field | Meaning |
|-------|---------|
| `logfile` | Raw data folder prefix (matches filenames under `samples/`) |
| `location` | City / map region |
| `map_token` | → `map.json` (HD map for this city) |

---

### Step 2 — `scene.json` · ~20 s clip

The table of all scenes in this split’s “database” — each row is one **~20 s clip** cut from a drive session (`log_token`). The row holds scene metadata plus pointers to the **first** and **last** annotated keyframe in `sample.json` (`first_sample_token`, `last_sample_token`). Keyframes in between are reached by walking `sample["next"]`, not from the scene row directly.

| What it holds | Examples |
|---------------|----------|
| **Metadata** | `name`, `description`, `nbr_samples` |
| **Parent link** | `log_token` → **Step 1** |
| **Head of timeline** | `first_sample_token` → first row in `sample.json` |
| **Tail** (convenience) | `last_sample_token` |

| What it does **not** hold |
|---------------------------|
| Array of all sample tokens — walk `sample["next"]` from `first_sample_token` |
| Sensor paths, ego pose, calibrations, boxes |

```python
scene = nusc.scene[0]   # scene-0061

# walk all annotated keyframes in this clip
sample_token = scene["first_sample_token"]
while sample_token:
    sample = nusc.get("sample", sample_token)
    sample_token = sample["next"] or None
```

**Example row** (`nusc.scene[0]`):

```json
{
  "token": "cc8c0bf57f984915a77078b10eb33198",
  "log_token": "7e25a2c8ea1f41c5b0da1e69ecfa71a2",
  "name": "scene-0061",
  "description": "Parked truck, construction, intersection, turn left, following a van",
  "nbr_samples": 39,
  "first_sample_token": "ca9a282c9e77460f8360f564131a8af5",
  "last_sample_token": "ed5fc18c31904f96a8f0dbb99ff069c0"
}
```

| Field | Follow with |
|-------|-------------|
| `log_token` | → **Step 1** `log.json` |
| `first_sample_token` | → **Step 3** `sample.json` (start of timeline) |
| `nbr_samples` | 39 keyframes @ 2 Hz in this clip |

---

### Step 3 — `sample.json` · keyframe hub

The table of all annotated keyframes in this split’s “database” — each row is one **annotated instant** @ **2 Hz** (~0.5 s apart). Rows for the same scene are chained with `prev` / `next`; start from `scene.first_sample_token` and walk forward. This is the **hub** row: it holds tokens to every sensor capture and every 3D box at that instant, not the raw files or poses themselves.

| What it holds | Examples |
|---------------|----------|
| **Timestamp** | `timestamp` (µs) |
| **Parent link** | `scene_token` → **Step 2** |
| **Time chain** | `prev`, `next` → neighbouring keyframes in the same scene |
| **Sensor refs** | `data` — map channel name → `sample_data` token (12 channels) |
| **Box refs** | `anns` — list of `sample_annotation` tokens |

| What it does **not** hold |
|---------------------------|
| File paths — follow `data[channel]` → **Step 4** `sample_data.json` |
| Ego pose or calibration — **Step 5** gives a token → **Step 6** |
| Box geometry — **Step 5** gives a token (`anns`) → **Step 6** |

```python
sample = nusc.get("sample", scene["first_sample_token"])
```

**Example row** (`nusc.get("sample", "ca9a282c9e77460f8360f564131a8af5")`):

```json
{
  "token": "ca9a282c9e77460f8360f564131a8af5",
  "timestamp": 1532402927647951,
  "prev": "",
  "next": "39586f9d59004284a7114a68825e8eec",
  "scene_token": "cc8c0bf57f984915a77078b10eb33198",
  "data": {
    "LIDAR_TOP": "9d9bf11f…",
    "CAM_FRONT": "e3d495d4…",
    "CAM_FRONT_LEFT": "fe542274…",
    "…": "… (12 channels total)"
  },
  "anns": ["ef63a697…", "… (69 box tokens)"]
}
```

| Field | Follow with |
|-------|-------------|
| `data["LIDAR_TOP"]` | → **Step 4** → **Step 5** (lidar) |
| `data["CAM_FRONT"]` | → **Step 4** → **Step 5** (camera) |
| `anns[i]` | → **Step 5** (box token) → **Step 6** |
| `next` | → next keyframe `sample.json` |

---

### Step 4 — `sample_data.json` · decipher `sample.data`

The table of all sensor captures in this split’s “database” — each row is **one sensor file** (lidar, camera, or radar) at one timestamp. This table **deciphers** the `data` block from **Step 3**: each token in `sample.data` is a primary key into this table; the lookup returns `channel`, **`filename`**, and further tokens (`ego_pose_token`, `calibrated_sensor_token`, …).

**What comes next?** Step 4 returns a row with mixed fields — some are **actual content** (`filename`), some are **more tokens** (`ego_pose_token`, …). **Step 5** splits the `sample.data` keys into **modality tables** (lidar, camera_front, …); **Step 6** resolves any remaining tokens.

```json
"data": {
  "LIDAR_TOP": "9d9bf11f…",
  "CAM_FRONT": "e3d495d4…",
  "CAM_FRONT_LEFT": "fe542274…",
  "…": "… (12 channels total)"
}
```

| Step 3 (`sample.data`) | Step 4 (`sample_data` row) |
|------------------------|----------------------------|
| `"LIDAR_TOP": "9d9bf11f…"` | `channel`: `"LIDAR_TOP"`, `filename`: `samples/LIDAR_TOP/….pcd.bin` |
| `"CAM_FRONT": "e3d495d4…"` | `channel`: `"CAM_FRONT"`, `filename`: `samples/CAM_FRONT/….jpg` |
| (12 keys, 12 tokens) | 12 **separate rows** in this table |

There is no separate JSON file per modality on disk — keyframes and sweeps share one `sample_data.json` table (`is_key_frame` tells you `samples/` vs `sweeps/`). **Step 5** splits that table by the keys in `sample.data` — one **modality table** per channel.

| What it holds | Examples |
|---------------|----------|
| **Which keyframe** (if annotated) | `sample_token` → back to Step 3 |
| **Which sensor** | `channel`, `sensor_modality` |
| **Raw file path** | `filename` → **first field that points at bytes on disk** |
| **Keyframe vs sweep** | `is_key_frame`: `true` → `samples/`; `false` → `sweeps/` |
| **Further tokens** | `ego_pose_token`, `calibrated_sensor_token` → **Step 5** (then **Step 6** if token) |

Repeat for every key in `sample["data"]` to resolve all 12 channels at one keyframe.

```python
token = sample["data"]["LIDAR_TOP"]
lidar_row = nusc.get("sample_data", token)
print(lidar_row["channel"])
print(lidar_row["filename"])
```

**Example row** (`nusc.get("sample_data", sample["data"]["LIDAR_TOP"])`):

```json
{
  "token": "9d9bf11fb0e144c8b446d54a8a00184f",
  "sample_token": "ca9a282c9e77460f8360f564131a8af5",
  "channel": "LIDAR_TOP",
  "is_key_frame": true,
  "filename": "samples/LIDAR_TOP/n015-2018-07-24-11-22-45+0800__LIDAR_TOP__1532402927647951.pcd.bin",
  "ego_pose_token": "9d9bf11fb0e144c8b446d54a8a00184f",
  "calibrated_sensor_token": "a183049901c24361a6b0b11b8013137c",
  "timestamp": 1532402927647951
}
```

| Field | Follow with |
|-------|-------------|
| `filename` | **Step 5** — actual content (sensor file path) |
| `ego_pose_token` | **Step 5** — token → **Step 6** |
| `calibrated_sensor_token` | **Step 5** — token → **Step 6** |

---

### Step 5 — modality tables

**Step 5 is the modality tables** — one table per key in **`sample.data`**, deciphered in **Step 4**:

| `sample.data` key | Modality table |
|-------------------|----------------|
| `LIDAR_TOP` | **lidar table** |
| `CAM_FRONT` | **camera_front table** |
| `CAM_FRONT_LEFT`, `CAM_FRONT_RIGHT`, … | **camera_* tables** *(6 cameras)* |
| `RADAR_FRONT`, `RADAR_FRONT_LEFT`, … | **radar_* tables** *(5 radars)* |

Each modality table has **one row at this keyframe** — the `sample_data` row you looked up with that token. That row tells you what is **content** you can use directly (e.g. `filename`) vs **tokens** to resolve in **Step 6**.

| Channel | What the row contains | Field | Step 5 |
|---------|----------------------|-------|--------|
| **`LIDAR_TOP`** | lidar file | `filename` | **content** — path to `.pcd.bin` |
| | ego pose | `ego_pose_token` | **token** → Step 6 |
| | calibration | `calibrated_sensor_token` | **token** → Step 6 |
| **`CAM_FRONT`** *(each camera similar)* | image file | `filename` | **content** — path to `.jpg` |
| | calibration | `calibrated_sensor_token` | **token** → Step 6 |
| | ego pose | `ego_pose_token` | **token** → Step 6 |
| **`RADAR_*`** *(5 radars)* | radar file | `filename` | **content** — same pattern |
| | ego pose, calibration | `ego_pose_token`, `calibrated_sensor_token` | **token** → Step 6 |

**3D boxes** are not in `sample_data` — they live on the **Step 3** row as `sample.anns[i]` (box tokens → Step 6).

```python
lidar_row = nusc.get("sample_data", sample["data"]["LIDAR_TOP"])
cam_row = nusc.get("sample_data", sample["data"]["CAM_FRONT"])
box_token = sample["anns"][0]
```

| Modality | Step 5 gives you | Type |
|----------|------------------|------|
| **Lidar file** | `lidar_row["filename"]` | **content** |
| **Camera file** | `cam_row["filename"]` | **content** |
| **Ego pose** | `lidar_row["ego_pose_token"]` | **token** → Step 6 |
| **Calibration** | `lidar_row["calibrated_sensor_token"]` | **token** → Step 6 |
| **3D boxes** | `box_token` from `sample.anns` | **token** → Step 6 |

**Sensor files are done at Step 5** — `filename` is a disk path, not a token. Open `NUSCENES_DATA_ROOT / filename` and read bytes.

```python
path = NUSCENES_DATA_ROOT / lidar_row["filename"]
# samples/LIDAR_TOP/n015-…__LIDAR_TOP__1532402927647951.pcd.bin

ego_token = lidar_row["ego_pose_token"]   # still a token — Step 6
```

---

### Step 6 — resolve tokens to actual content

For every modality that gave a **token** in Step 5, look it up once more. The row fields are the **actual content** — numbers you use directly.

| Modality | Token from Step 5 | Step 6 table | Actual content |
|----------|-------------------|--------------|----------------|
| **Ego pose** | `ego_pose_token` | `ego_pose.json` | `translation`, `rotation` |
| **Calibration** | `calibrated_sensor_token` | `calibrated_sensor.json` → `sensor.json` | sensor→ego extrinsics (`translation`, `rotation`); camera intrinsics when modality is camera |
| **3D boxes** | `sample.anns[i]` | `sample_annotation.json` | `translation`, `size`, `rotation`, `category_name` |

```python
ego = nusc.get("ego_pose", lidar_row["ego_pose_token"])
cal = nusc.get("calibrated_sensor", lidar_row["calibrated_sensor_token"])
ann = nusc.get("sample_annotation", sample["anns"][0])
```

#### Ego pose — example row

```json
{
  "token": "9d9bf11fb0e144c8b446d54a8a00184f",
  "timestamp": 1532402927647951,
  "translation": [411.30, 1180.89, 0.0],
  "rotation": [0.572, -0.0017, 0.0118, -0.8201]
}
```

Each sensor capture can have its own ego row (lidar vs cameras differ by a few ms).

#### Calibration — example rows

Every row in `calibrated_sensor.json` has **extrinsics** — `translation` and `rotation` transform sensor frame → ego frame (lidar, camera, and radar). Cameras also carry a **3×3 intrinsic** matrix in `camera_intrinsic` (empty `[]` for non-cameras).

Lidar (`calibrated_sensor.json` — extrinsics only):

```json
{
  "token": "a183049901c24361a6b0b11b8013137c",
  "sensor_token": "dc8b396651c05aedbb9cdaae573bb567",
  "translation": [0.94, 0.0, 1.84],
  "rotation": [0.7078, -0.0065, 0.0106, -0.7063],
  "camera_intrinsic": []
}
```

Camera (`calibrated_sensor.json` — extrinsics + intrinsics):

```json
{
  "token": "1d31c729b073425e8e0202c5c6e66ee1",
  "sensor_token": "725903f5b62f56118f4094b46a4470d8",
  "translation": [1.70, 0.02, 1.51],
  "rotation": [0.4998, -0.5030, 0.4995, -0.4974],
  "camera_intrinsic": [
    [1266.4, 0.0, 816.3],
    [0.0, 1266.4, 491.5],
    [0.0, 0.0, 1.0]
  ]
}
```

Channel name via `sensor.json`: `"channel": "LIDAR_TOP"`, `"modality": "lidar"`. Calibration is per **log × channel**.

#### 3D boxes — example row

```json
{
  "token": "ef63a697930c4b20a6b9791f423351da",
  "sample_token": "ca9a282c9e77460f8360f564131a8af5",
  "category_name": "human.pedestrian.adult",
  "translation": [373.26, 1130.42, 0.8],
  "size": [0.62, 0.67, 1.64],
  "rotation": […]
}
```

Optional: `ann["instance_token"]` → `instance.json` → `category.json` for track ID and class name. Annotations exist only on **keyframes**, not sweeps.

---

## Map your six goals → steps

| # | Goal | Step |
|---|------|------|
| 1 | Continuous recording | 1–2 |
| — | Decipher `sample.data` | 4 |
| 5 | Camera raw | **5** (`filename` = content) |
| 6 | Lidar raw | **5** (`filename` = content) |
| 2 | Ego state | 5 → **6** |
| 3 | Calibration | 5 → **6** |
| 4 | Annotations | 5 → **6** |

---

## End-to-end code

```python
scene = nusc.scene[0]
log = nusc.get("log", scene["log_token"])
sample = nusc.get("sample", scene["first_sample_token"])

lidar_row = nusc.get("sample_data", sample["data"]["LIDAR_TOP"])
cam_row = nusc.get("sample_data", sample["data"]["CAM_FRONT"])
ego = nusc.get("ego_pose", lidar_row["ego_pose_token"])
cal = nusc.get("calibrated_sensor", lidar_row["calibrated_sensor_token"])
ann = nusc.get("sample_annotation", sample["anns"][0])

print(log["location"])              # singapore-onenorth
print(ego["translation"])           # [411.30, 1180.89, 0.0]
print(cal["translation"])           # lidar mount in ego frame
print(ann["category_name"])         # human.pedestrian.adult
print(cam_row["filename"])          # samples/CAM_FRONT/….jpg
print(lidar_row["filename"])        # samples/LIDAR_TOP/….pcd.bin
```

---

## Walk the full clip in time

```python
sample_token = scene["first_sample_token"]
while sample_token:
    sample = nusc.get("sample", sample_token)
    t = sample["timestamp"] / 1e6
    n_boxes = len(sample["anns"])
    # … lidar_row, ego, ann, etc. for this keyframe …
    sample_token = sample["next"] or None
```

39 keyframes for `scene-0061`.

---

## Summary

| You need | Token hops |
|----------|------------|
| Drive session | `scene.log_token` → `log` |
| Clip (~20 s) | `scene.json` |
| Keyframe @ t | `first_sample_token`, then `sample.next` → `sample` |
| Decipher sensors | Step 4 (`sample_data`) |
| Lidar / camera / radar | Step **5** — `filename` is content |
| Ego pose | Step **5** token → Step **6** `ego_pose` |
| Calibration | Step **5** token → Step **6** `calibrated_sensor` |
| 3D boxes | Step **5** token (`anns`) → Step **6** `sample_annotation` |

**Key insight:** Steps **1–4** = token index. Step **5** = per modality, content or token. Step **6** = resolve remaining tokens to content.

---

## What you end up with

After Steps 1–6 you have the same **modalities** as any multi-modal public dataset — ego pose, calibration, boxes, camera JPEGs, lidar sweeps — reached here via JSON tokens instead of AV2-style feathers. Map is separate from this chain (per city, not per scene).

| Content | nuScenes (after Steps 1–6) | AV2 S0 |
|---------|------------------------------|--------|
| Ego pose | `ego_pose.json` via `sample_data` | `city_SE3_egovehicle.feather` |
| Calibration | `calibrated_sensor.json` | `calibration/*.feather` |
| Boxes | `sample_annotation.json` (global frame) | `annotations.feather` (ego frame) |
| Camera / lidar | `sample_data.filename` | `sensors/cameras/`, `sensors/lidar/` |
| Map | Per-city `maps/expansion/` | Per-log `map/log_map_archive_*.json` |

