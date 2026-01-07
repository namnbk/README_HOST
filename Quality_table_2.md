# Building Your Custom Resolution Level Table with Hysteresis

## Your Input Data

You've calculated the **minimum bitrate** for good quality at each resolution: 

```
Level 0:  640 × 360   →  691 kbps
Level 1:  854 × 480   →  1229 kbps
Level 2:  960 × 540   →  1555 kbps
Level 3: 1280 × 720   →  2764 kbps
Level 4: 2048 × 1024  →  6291 kbps
Level 5: 2560 × 1440  →  11059 kbps
```

These represent the **minimum bitrate needed for acceptable quality** at each resolution.  This is our foundation. 

---

## Design Principles for Thresholds

### Principle 1: Upgrade Threshold > Minimum Bitrate

**Why? ** You want to ensure there's **headroom** before upgrading.  If you upgrade exactly at the minimum, any small network fluctuation immediately causes quality problems.

```
Upgrade Threshold = Minimum Bitrate × (1.1 to 1.2)
                  = 10-20% above minimum
```

### Principle 2: Downgrade Threshold < Minimum Bitrate of Current Level

**Why?** Give the current resolution a chance to recover before downgrading. Only downgrade when bitrate is **clearly insufficient**.

```
Downgrade Threshold = Minimum Bitrate × (0.7 to 0.85)
                    = 15-30% below minimum
```

### Principle 3: Stable Zone Should Be Meaningful

**Why?** The gap between thresholds should be large enough to prevent oscillation during normal network variance.

```
Stable Zone = Upgrade Threshold - Downgrade Threshold
            = Should be at least 15-25% of the minimum bitrate
```

---

## Calculating Your Thresholds

Let me walk through the calculation for each level:

### Level 0: 640 × 360 (Minimum = 691 kbps)

```
This is the LOWEST level, so: 
• Upgrade threshold:  N/A (you upgrade FROM this, not TO this)
• Downgrade threshold: This is the floor - set a minimum viable bitrate

Downgrade threshold = 691 × 0.65 ≈ 450 kbps
(Below this, video becomes unwatchable anyway)

To UPGRADE from Level 0 to Level 1:
• Need to reach Level 1's upgrade threshold (calculated below)
```

### Level 1: 854 × 480 (Minimum = 1229 kbps)

```
Upgrade threshold (to enter Level 1):
= 1229 × 1.15 = 1413 kbps → round to 1400 kbps

Downgrade threshold (to leave Level 1):
= 1229 × 0.75 = 922 kbps → round to 900 kbps

Stable zone:  1400 - 900 = 500 kbps (41% of minimum - good!)
```

### Level 2: 960 × 540 (Minimum = 1555 kbps)

```
Upgrade threshold: 
= 1555 × 1.15 = 1788 kbps → round to 1800 kbps

Downgrade threshold:
= 1555 × 0.75 = 1166 kbps → round to 1150 kbps

Stable zone: 1800 - 1150 = 650 kbps (42% of minimum - good!)
```

### Level 3: 1280 × 720 (Minimum = 2764 kbps)

```
Upgrade threshold:
= 2764 × 1.15 = 3179 kbps → round to 3200 kbps

Downgrade threshold:
= 2764 × 0.75 = 2073 kbps → round to 2100 kbps

Stable zone: 3200 - 2100 = 1100 kbps (40% of minimum - good!)
```

### Level 4: 2048 × 1024 (Minimum = 6291 kbps)

```
Upgrade threshold:
= 6291 × 1.15 = 7235 kbps → round to 7200 kbps

Downgrade threshold:
= 6291 × 0.75 = 4718 kbps → round to 4700 kbps

Stable zone: 7200 - 4700 = 2500 kbps (40% of minimum - good!)
```

### Level 5: 2560 × 1440 (Minimum = 11059 kbps)

```
This is the HIGHEST level, so: 
• Upgrade threshold: Must reach this to enter Level 5
• Downgrade threshold: N/A (this is the ceiling)

Upgrade threshold: 
= 11059 × 1.15 = 12718 kbps → round to 12700 kbps

Downgrade threshold:
= 11059 × 0.75 = 8294 kbps → round to 8300 kbps
```

---

## Your New Resolution Level Table

```c
typedef struct {
    UINT16 width;
    UINT16 height;
    UINT32 minBitrateKbps;      // Your calculated minimum for good quality
    UINT32 upgradeThresholdKbps;   // Bitrate must EXCEED this to upgrade TO this level
    UINT32 downgradeThresholdKbps; // Bitrate must DROP BELOW this to downgrade FROM this level
} ResolutionLevel;

static const ResolutionLevel LEVELS[] = {
    //    W       H     MinBitrate  Upgrade↑  Downgrade↓
    {   640,   360,       691,          0,        450 },   // Level 0: 360p (minimum level)
    {   854,   480,      1229,       1400,        900 },   // Level 1: 480p
    {   960,   540,      1555,       1800,       1150 },   // Level 2: 540p
    {  1280,   720,      2764,       3200,       2100 },   // Level 3: 720p
    {  2048,  1024,      6291,       7200,       4700 },   // Level 4: 2K
    {  2560,  1440,     11059,      12700,       8300 },   // Level 5: 1440p (maximum level)
};

#define NUM_RESOLUTION_LEVELS (sizeof(LEVELS) / sizeof(LEVELS[0]))
#define MIN_VIDEO_BITRATE_KBPS 450    // Absolute floor
#define MAX_VIDEO_BITRATE_KBPS 15000  // Absolute ceiling
```

---

## Visual Representation

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    Your Resolution Levels with Hysteresis                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   Bitrate (kbps)                                                                     │
│       │                                                                              │
│  15000┤                                                                    ┌────────│
│       │                                                                    │ 1440p  │
│  12700┤─────────────────────────────────────────────────────────────Upgrade┤        │
│       │                                                              ═════ │        │
│  11059┤.. ................................................ .... Min bitrate.. │........ │
│       │                                                              ═════ │        │
│   8300┤─────────────────────────────────────────────────────────Downgrade─┤        │
│       │                                                         ┌─────────┘        │
│   7200┤─────────────────────────────────────────────────Upgrade─┤ 2K               │
│       │                                                    ═════│                  │
│   6291┤................................................ Min. ...... │.................. │
│       │                                                    ═════│                  │
│   4700┤─────────────────────────────────────────────Downgrade───┤                  │
│       │                                              ┌──────────┘                  │
│   3200┤──────────────────────────────────────Upgrade─┤ 720p                        │
│       │                                         ═════│                             │
│   2764┤. ........ ................................ Min.. │. .... ........................ │
│       │                                         ═════│                             │
│   2100┤──────────────────────────────────Downgrade───┤                             │
│       │                                   ┌──────────┘                             │
│   1800┤───────────────────────────Upgrade─┤ 540p                                   │
│       │                              ═════│                                        │
│   1555┤...... ........................Min..│........ ................................│
│       │                              ═════│                                        │
│   1150┤───────────────────────Downgrade───┤                                        │
│       │                        ┌──────────┘                                        │
│   1400┤────────────────Upgrade─┤ 480p                                              │
│       │                   ═════│                                                   │
│   1229┤. .................. Min.. │... ................................................ │
│       │                   ═════│                                                   │
│    900┤────────────Downgrade───┤                                                   │
│       │             ┌──────────┘                                                   │
│    691┤. .... Min. .... │ 360p                                                         │
│       │        ═════│                                                              │
│    450┤──Downgrade──┤                                                              │
│       │             │                                                              │
│      0┤─────────────┘                                                              │
│       └──────────────────────────────────────────────────────────────────────────► │
│                                                                                      │
│   Legend:   ════ = Stable zone (no resolution change)                                │
│            ─── = Threshold boundary                                                 │
│            ...  = Minimum bitrate for good quality                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Threshold Summary Table

| Level | Resolution | Min Bitrate | Upgrade↑ | Downgrade↓ | Stable Zone | Headroom |
|-------|------------|-------------|----------|------------|-------------|----------|
| 0 | 640×360 | 691 | - | 450 | - | - |
| 1 | 854×480 | 1229 | 1400 | 900 | 500 kbps | +14% |
| 2 | 960×540 | 1555 | 1800 | 1150 | 650 kbps | +16% |
| 3 | 1280×720 | 2764 | 3200 | 2100 | 1100 kbps | +16% |
| 4 | 2048×1024 | 6291 | 7200 | 4700 | 2500 kbps | +14% |
| 5 | 2560×1440 | 11059 | 12700 | 8300 | 4400 kbps | +15% |

---

## Why These Specific Numbers?

### Upgrade Thresholds (×1.15, +15% above minimum)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   Why 15% above minimum?                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. Network variance buffer                                                 │
│      └── Real networks fluctuate ±10% constantly                            │
│      └── 15% headroom absorbs normal jitter without quality drop            │
│                                                                              │
│   2. Encoder efficiency variance                                             │
│      └── Complex scenes need more bits than simple scenes                   │
│      └── 15% buffer handles scene complexity changes                        │
│                                                                              │
│   3. Conservative upgrade                                                    │
│      └── Better to stay at lower resolution with good quality               │
│      └── Than upgrade and immediately struggle                              │
│                                                                              │
│   Example:                                                                    │
│   720p needs 2764 kbps minimum                                               │
│   We require 3200 kbps to upgrade (15% headroom)                             │
│   This means:   even if bitrate drops 10%, still above minimum                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Downgrade Thresholds (×0.75, -25% below minimum)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   Why 25% below minimum?                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. Recovery opportunity                                                    │
│      └── Brief network dips shouldn't trigger downgrade                     │
│      └── Give time for congestion to clear                                  │
│                                                                              │
│   2. Avoid unnecessary quality disruption                                    │
│      └── Resolution changes are visually noticeable                         │
│      └── Only downgrade when clearly necessary                              │
│                                                                              │
│   3. Quality tolerance zone                                                  │
│      └── Video is still watchable at 75-100% of optimal bitrate             │
│      └── Below 75%, quality degradation becomes obvious                     │
│                                                                              │
│   Example:                                                                    │
│   720p minimum is 2764 kbps                                                  │
│   Downgrade threshold is 2100 kbps (75% of minimum)                          │
│   Between 2100-2764 kbps:  quality reduced but acceptable                    │
│   Below 2100 kbps:  clearly insufficient, must downgrade                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Overlap Zones (Hysteresis Bands)

```
┌───────────────────────────────────────────────────────────────────���─────────┐
│   Example: 540p ↔ 720p Transition                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   540p downgrade threshold: 1150 kbps                                        │
│   720p upgrade threshold:    3200 kbps                                        │
│   720p downgrade threshold: 2100 kbps                                        │
│                                                                              │
│   Bitrate:  1150 ────────────── 2100 ────────────── 3200                      │
│               │                  │                   │                       │
│               │    ┌─────────────┴───────────────┐  │                       │
│               │    │                             │  │                       │
│               │    │  If at 540p: stay at 540p   │  │                       │
│               │    │  If at 720p: stay at 720p   │  │                       │
│               │    │                             │  │                       │
│               │    │  OVERLAP ZONE (950 kbps)    │  │                       │
│               │    │                             │  │                       │
│               │    └─────────────────────────────┘  │                       │
│               │                                     │                       │
│          Below 1150:                           Above 3200:                   │
│          Must be at 360p                       Can be at 720p               │
│          (540p unsustainable)                  (proven bandwidth)           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Implementation Code

```c
typedef struct {
    UINT16 width;
    UINT16 height;
    UINT32 minBitrateKbps;
    UINT32 upgradeThresholdKbps;
    UINT32 downgradeThresholdKbps;
} ResolutionLevel;

static const ResolutionLevel LEVELS[] = {
    //    W       H     MinBitrate  Upgrade↑  Downgrade↓
    {   640,   360,       691,          0,        450 },   // Level 0
    {   854,   480,      1229,       1400,        900 },   // Level 1
    {   960,   540,      1555,       1800,       1150 },   // Level 2
    {  1280,   720,      2764,       3200,       2100 },   // Level 3
    {  2048,  1024,      6291,       7200,       4700 },   // Level 4
    {  2560,  1440,     11059,      12700,       8300 },   // Level 5
};

#define NUM_LEVELS 6

INT32 determineResolutionLevel(UINT32 bitrateKbps, INT32 currentLevel)
{
    // Boundary checks
    if (currentLevel < 0) currentLevel = 0;
    if (currentLevel >= NUM_LEVELS) currentLevel = NUM_LEVELS - 1;
    
    // Check for UPGRADE (can we go to a higher level?)
    if (currentLevel < NUM_LEVELS - 1) {
        UINT32 upgradeThreshold = LEVELS[currentLevel + 1].upgradeThresholdKbps;
        if (bitrateKbps > upgradeThreshold) {
            DLOGI("Upgrade triggered:  %u kbps > %u threshold, Level %d -> %d",
                  bitrateKbps, upgradeThreshold, currentLevel, currentLevel + 1);
            return currentLevel + 1;
        }
    }
    
    // Check for DOWNGRADE (must we go to a lower level?)
    if (currentLevel > 0) {
        UINT32 downgradeThreshold = LEVELS[currentLevel]. downgradeThresholdKbps;
        if (bitrateKbps < downgradeThreshold) {
            DLOGI("Downgrade triggered: %u kbps < %u threshold, Level %d -> %d",
                  bitrateKbps, downgradeThreshold, currentLevel, currentLevel - 1);
            return currentLevel - 1;
        }
    }
    
    // Stay at current level (in stable zone)
    return currentLevel;
}
```

---

## Example Session Walkthrough

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   Session Example: Starting at 540p (Level 2)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Time   Bitrate    Check                              Result                │
│   ────   ───────    ─────────────────────────────────  ──────────────────   │
│   0:00   1600       Start at 540p (Level 2)            -                     │
│   0:05   1750       1750 < 3200 (upgrade to 720p)      Stay 540p             │
│   0:10   1900       1900 < 3200                        Stay 540p             │
│   0:15   2200       2200 < 3200                        Stay 540p             │
│   0:20   2800       2800 < 3200                        Stay 540p             │
│   0:25   3100       3100 < 3200                        Stay 540p             │
│   0:30   3300       3300 > 3200 ✓                      UPGRADE → 720p (L3)   │
│   0:35   3100       3100 > 2100 (downgrade threshold)  Stay 720p             │
│   0:40   2900       2900 > 2100                        Stay 720p             │
│   0:45   2500       2500 > 2100                        Stay 720p             │
│   0:50   2200       2200 > 2100                        Stay 720p             │
│   0:55   2000       2000 < 2100 ✓                      DOWNGRADE → 540p (L2) │
│   1:00   1800       1800 > 1150 (downgrade threshold)  Stay 540p             │
│   1:05   1500       1500 > 1150                        Stay 540p             │
│   1:10   1100       1100 < 1150 ✓                      DOWNGRADE → 480p (L1) │
│   1:15   1200       1200 < 1800 (upgrade to 540p)      Stay 480p             │
│   1:20   1500       1500 < 1800                        Stay 480p             │
│   1:25   1900       1900 > 1800 ✓                      UPGRADE → 540p (L2)   │
│                                                                              │
│   Total: 4 resolution changes in 1: 25                                        │
│   Without hysteresis: would have been 10+ changes                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

| Design Choice | Value | Rationale |
|---------------|-------|-----------|
| Upgrade threshold | Min × 1.15 | 15% headroom for network variance |
| Downgrade threshold | Min × 0.75 | 25% tolerance before forcing downgrade |
| Stable zone width | ~40% of minimum | Large enough to prevent oscillation |
| Minimum floor | 450 kbps | Absolute minimum for any video |
| Maximum ceiling | 15000 kbps | Reasonable upper bound |
