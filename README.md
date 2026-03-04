# ERIP — Emotion Recognition In Players

A research pipeline for studying player emotions during gameplay in **Risk of Rain 2**. The project collects multimodal data — in-game telemetry, facial expressions, and heart rate — and merges them into a unified dataset for emotion recognition research.

---

## Project Structure

```
ERIP/
├── DATASET.csv                  # Final merged dataset
├── RoR2/                        # BepInEx game plugin (C#)
│   └── source/ScienceKit/       # Plugin source code
│       ├── ScienceKitPlugin.cs
│       ├── StatisticsPersistentManager.cs
│       ├── GameSimplifier.cs
│       └── Statistics/          # Individual statistic collectors
└── Python/                      # Data processing pipeline (Python)
    ├── data_convertation.py     # Step 1: Convert raw logs to CSV
    ├── image_preparation.py     # Step 2: Preprocess webcam frames
    ├── heart_rate_preparation.py# Step 2: Preprocess heart rate data
    ├── image_emotion.py         # Step 3: Extract emotion (valence/arousal)
    ├── data_joining.py          # Step 4: Merge all data into final CSV
    ├── erip_dataset.py          # PyTorch Dataset for image loading
    └── DataAggregation/         # Per-second aggregation modules
        ├── AxisAggregator.py
        ├── ButtonsAggregator.py
        ├── EmotionAggregator.py
        ├── EnemiesAggregator.py
        ├── HealthAggregator.py
        ├── ItemsAggregator.py
        └── StatsAggregator.py
```

---

## Components

### 1. ScienceKit — Risk of Rain 2 BepInEx Plugin (C#)

A BepInEx mod that hooks into Risk of Rain 2's event system and logs gameplay telemetry to CSV files during each run.

**Collected statistics (per run, timestamped in seconds):**

| File prefix      | Contents                                                       |
|------------------|----------------------------------------------------------------|
| `AxisInputs`     | Analog axis name, delta value, runtime                         |
| `ButtonInputs`   | Button name (Jump, Primary, Secondary, etc.), state, runtime   |
| `Items`          | Item index, whether it's equipment, added/removed, runtime     |
| `Kills`          | Killed entity ID/name, level, distance to player, runtime      |
| `Spawns`         | Spawned entity ID/name, level, distance to player, runtime     |
| `PlayerHealth`   | Absolute health, health fraction, runtime                      |
| `Stats`          | Average speed, DPS, runtime                                    |

Files are written to `Application.persistentDataPath/Statistics/<Type>/<Type>-<timestamp>.log`.

The plugin also includes a `GameSimplifier` that suppresses shrines, portals, duplicators, and drone spawns to reduce confounding variables during study sessions.

**Dependencies:** BepInEx, R2API, MMHOOK_RoR2

---

### 2. Python Data Processing Pipeline

A sequential pipeline that converts raw game logs, webcam footage, and heart rate recordings into a single merged CSV.

#### Step 1 — Convert raw logs to CSV

```bash
python data_convertation.py --path /path/to/session/folder
```

Parses pipe-delimited `.log` files and writes structured CSVs into a `processed/` subdirectory for each data type.

#### Step 2 — Preprocess webcam images

```bash
python image_preparation.py --path /path/to/images --padding 200 --format "*.png"
```

Detects and crops the player's face from each webcam frame using a CNN model (`face_recognition`), resizes to 256×256, and saves to a `processed/` folder.

#### Step 2 — Preprocess heart rate data

```bash
python heart_rate_preparation.py --path /path/to/heartrate.csv
```

Fills gaps in heart rate recordings to produce a continuous per-second time series.

#### Step 3 — Extract emotion from images

```bash
python image_emotion.py --path /path/to/processed/images --device cuda:0 --batch_size 32
```

Runs [EmoNet](https://github.com/face-debiasing/emonet) on the preprocessed face images and produces a `result.csv` with continuous **valence** and **arousal** values per frame.

#### Step 4 — Join all data into final dataset

```bash
python data_joining.py --path /path/to/session/folder
```

Aggregates all per-second processed CSVs and aligns them by `RunTime` and `RunDate` into a single `final.csv`.

**Final dataset columns (per second of each run):**

| Column(s)                                | Source                       |
|------------------------------------------|------------------------------|
| `RunTime`, `RunDate`                     | Common time key              |
| Axis input values                        | AxisAggregator               |
| `EnemyCount`, `KillRate`                 | EnemiesAggregator            |
| `AverageSpeed`, `DPS`                    | StatsAggregator              |
| `Health`, `HealthFraction`               | HealthAggregator             |
| `PrimaryHeld`, `JumpPressed`, etc.       | ButtonsAggregator            |
| Item acquisition counts                  | ItemsAggregator              |
| `Valence`, `Arousal`                     | EmotionAggregator (EmoNet)   |

---

## Requirements

### Plugin (C#)
- Risk of Rain 2
- BepInEx 5
- R2API

### Python pipeline
- Python 3.8+
- PyTorch
- pandas, scikit-image, tqdm
- `face_recognition`
- EmoNet (place pretrained weights at `Python/emonet/pretrained/emonet_8.pth`)

Install Python dependencies:
```bash
pip install torch torchvision pandas scikit-image tqdm face_recognition
```

---

## Data Collection Protocol

1. Install `ScienceKit.dll` into the BepInEx `plugins/` folder.
2. Start a recording session (webcam + optional heart rate monitor).
3. Play Risk of Rain 2 — telemetry is logged automatically each run.
4. Run the Python pipeline on the collected session folder to produce the merged dataset.
