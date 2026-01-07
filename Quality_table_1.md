# Understanding Hysteresis in Resolution Switching

## What is Hysteresis?

**Hysteresis** means using **different thresholds for going up vs. going down**. It creates a "buffer zone" where the system stays stable and doesn't change. 

### Real-World Analogy:  Thermostat

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Thermostat Example                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   WITHOUT Hysteresis (Single threshold at 70°F):                             │
│   ──────────────────────────────────────────────                             │
│   Temperature: 69°F → Heater ON                                              │
│   Temperature: 70°F → Heater OFF                                             │
│   Temperature: 69°F → Heater ON                                              │
│   Temperature:  70°F → Heater OFF                                             │
│   ...  heater turns on/off every few seconds!  (BAD)                           │
│                                                                              │
│   ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│   WITH Hysteresis (Turn ON at 68°F, Turn OFF at 72°F):                       │
│   ────────────────────────────────────────────────────                       │
│   Temperature: 67°F → Heater ON                                              │
│   Temperature:  69°F → Heater stays ON (hasn't reached 72°F yet)              │
│   Temperature: 71°F → Heater stays ON (hasn't reached 72°F yet)              │
│   Temperature: 72°F → Heater OFF                                             │
│   Temperature: 70°F → Heater stays OFF (hasn't dropped to 68°F yet)          │
│   Temperature: 68°F → Heater ON                                              │
│   ...  heater changes state much less frequently (GOOD)                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## The Problem WITHOUT Hysteresis

Let's say you have a simple threshold:  **switch to 540p when bitrate ≥ 800 kbps**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              WITHOUT Hysteresis - Single Threshold at 800 kbps              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Time    Bitrate    Resolution    What Happens                              │
│   ────    ───────    ──────────    ─────────────────────────────────────     │
│   T1      750 kbps   480p          Normal, below threshold                   │
│   T2      785 kbps   480p          Still below 800                           │
│   T3      805 kbps   540p          ↑ Crossed 800!  UPGRADE to 540p            │
│   T4      795 kbps   480p          ↓ Dropped below 800!  DOWNGRADE to 480p    │
│   T5      802 kbps   540p          ↑ Crossed 800 again! UPGRADE to 540p      │
│   T6      798 kbps   480p          ↓ Below 800 again! DOWNGRADE to 480p      │
│   T7      801 kbps   540p          ↑ UPGRADE again!                           │
│   T8      799 kbps   480p          ↓ DOWNGRADE again!                        │
│                                                                              │
│   Problem: Resolution is flickering between 480p and 540p every second!      │
│            User sees constant quality changes - very annoying!               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Visual Timeline

```
Bitrate (kbps)
    │
820 ┤            ╱╲        ╱╲        ╱╲
    │           ╱  ╲      ╱  ╲      ╱  ╲
800 ┤─────────────────────────────────────── Single Threshold
    │         ╱    ╲    ╱    ╲    ╱    ╲
780 ┤        ╱      ╲  ╱      ╲  ╱      ╲
    │       ╱        ╲╱        ╲╱        ╲
760 ┤──────╱
    └──────┴────┴────┴────┴────┴────┴────┴────► Time
           T1   T2   T3   T4   T5   T6   T7   T8

Resolution:     480p 480p 540p 480p 540p 480p 540p 480p
                         ↑    ↓    ↑    ↓    ↑    ↓
                    Constant flickering!  (BAD)
```

---

## The Solution WITH Hysteresis

Use **two different thresholds**: 
- **Upgrade threshold**: 950 kbps (must exceed this to upgrade TO 540p)
- **Downgrade threshold**: 650 kbps (must fall below this to downgrade FROM 540p)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              WITH Hysteresis - Two Thresholds (650 and 950 kbps)            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Time    Bitrate    Resolution    What Happens                              │
│   ────    ───────    ──────────    ─────────────────────────────────────     │
│   T1      750 kbps   480p          Normal, in "stable zone"                  │
│   T2      785 kbps   480p          Still in stable zone (< 950)              │
│   T3      805 kbps   480p          Still in stable zone (< 950)              │
│   T4      795 kbps   480p          Still in stable zone                      │
│   T5      850 kbps   480p          Still in stable zone (< 950)              │
│   T6      920 kbps   480p          Still in stable zone (< 950)              │
│   T7      960 kbps   540p          ↑ Finally crossed 950! UPGRADE to 540p    │
│   T8      900 kbps   540p          Stay at 540p (still > 650)                │
│   T9      800 kbps   540p          Stay at 540p (still > 650)                │
│   T10     700 kbps   540p          Stay at 540p (still > 650)                │
│   T11     640 kbps   480p          ↓ Dropped below 650! DOWNGRADE to 480p    │
│                                                                              │
│   Result: Only 2 resolution changes instead of 6+                            │
│           Much more stable viewing experience!                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Visual Timeline

```
Bitrate (kbps)
    │
960 ┤                              ●───────╮
    │                             ╱        │
950 ┤ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─  Upgrade Threshold
    │                           ╱          │
900 ┤                          ╱           ╰───╮
    │         ╱╲    ╱╲        ╱                │
850 ┤        ╱  ╲  ╱  ╲      ╱                 │
    │       ╱    ╲╱    ╲    ╱                  ╰──╮
800 ┤──────╱            ╲  ╱                      │
    │                    ╲╱                       │
750 ┤                                             ╰──╮
    │                                                │
700 ┤                                                ╰─╮
    │                                                  │
650 ┤ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─  Downgrade Threshold
    │                                                  ●
600 ┤
    └──────┴────┴────┴────┴────┴────┴────┴────┴────┴────► Time
           T1   T2   T3   T4   T5   T6   T7   T8   T9  T10  T11

Resolution:    480p 480p 480p 480p 480p 480p 540p 540p 540p 540p 480p
                                             ↑                   ↓
                                      Only 2 changes! (GOOD)
```

---

## The "Stable Zone" Concept

The gap between the two thresholds creates a **stable zone** where no changes happen:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         The Stable Zone                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Bitrate                                                                    │
│   (kbps)                                                                     │
│      │                                                                       │
│  950 ┤ ════════════════════════════════════════  Upgrade Threshold           │
│      │                                                                       │
│      │         ┌─────────────────────────────┐                              │
│      │         │                             │                              │
│      │         │      STABLE ZONE            │                              │
│      │         │                             │                              │
│      │         │   If currently at 480p:      │                              │
│      │         │   → Stay at 480p            │                              │
│      │         │                             │                              │
│      │         │   If currently at 540p:     │                              │
│      │         │   → Stay at 540p            │                              │
│      │         │                             │                              │
│      │         │   No changes happen here!    │                              │
│      │         │                             │                              │
│      │         └─────────────────────────────┘                              │
│      │                                                                       │
│  650 ┤ ════════════════════════════════════════  Downgrade Threshold         │
│      │                                                                       │
│                                                                              │
│   Width of stable zone = 950 - 650 = 300 kbps                               │
│   Larger zone = more stability, but slower to react                         │
│   Smaller zone = faster reaction, but more flickering                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Complete Example: Walking Through a Session

Let's trace through a realistic 2-minute session:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Full Session Example                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Thresholds for 480p ↔ 540p transition:                                     │
│   • Upgrade to 540p when bitrate > 950 kbps                                  │
│   • Downgrade to 480p when bitrate < 650 kbps                                │
│                                                                              │
│   ═══════════════════════════════════════════════════════════════════════   │
│                                                                              │
│   Time   Bitrate   Current Res   Decision Logic                  Action      │
│   ─────  ───────   ───────────   ────────────────────────────   ──────────  │
│   0:00   500       480p          Starting point                 -           │
│   0:05   550       480p          550 < 950, stay                None        │
│   0:10   620       480p          620 < 950, stay                None        │
│   0:15   700       480p          700 < 950, stay                None        │
│   0:20   780       480p          780 < 950, stay                None        │
│   0:25   850       480p          850 < 950, stay                None        │
│   0:30   920       480p          920 < 950, stay                None        │
│   0:35   980       480p          980 > 950 ✓                    UPGRADE→540p│
│   0:40   950       540p          950 > 650, stay                None        │
│   0:45   900       540p          900 > 650, stay                None        │
│   0:50   820       540p          820 > 650, stay                None        │
│   0:55   750       540p          750 > 650, stay                None        │
│   1:00   680       540p          680 > 650, stay                None        │
│   1:05   630       540p          630 < 650 ✓                    DOWNGRADE→480p│
│   1:10   600       480p          In stable zone                 None        │
│   1:15   580       480p          580 < 950, stay                None        │
│   1:20   550       480p          550 < 950, stay                None        │
│   1:25   620       480p          620 < 950, stay                None        │
│   1:30   700       480p          700 < 950, stay                None        │
│   ...                                                                         │
│                                                                              │
│   Result: Only 2 resolution changes in 2 minutes                            │
│   Without hysteresis: Could have been 10+ changes                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Why Different Thresholds?

The key insight is:  **the threshold to enter a state should be different from the threshold to leave it**. 

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Why Two Thresholds Work                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                                                                              │
│   To UPGRADE to 540p:                                                        │
│   ───────────────────                                                        │
│   "Prove you can sustain high bitrate before I give you more pixels"        │
│   → Must reach 950 kbps (higher threshold)                                   │
│   → This ensures you have enough bandwidth for 540p quality                  │
│                                                                              │
│                                                                              │
│   To DOWNGRADE from 540p:                                                    │
│   ────────────────────────                                                   │
│   "I'll give you some room to recover before taking away pixels"            │
│   → Must drop below 650 kbps (lower threshold)                               │
│   → This prevents downgrade during brief network hiccups                     │
│                                                                              │
│                                                                              │
│   The GAP (950 - 650 = 300 kbps) is the "forgiveness zone"                  │
│   → Bitrate can fluctuate within this range without triggering changes      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Applying Hysteresis to Your Quality Ladder

Here's how to add hysteresis to each resolution band:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Quality Ladder with Hysteresis                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Resolution   Upgrade TO      Downgrade FROM     Stable Zone                │
│                (exceed this)   (fall below this)  (no change)                │
│   ──────────   ────────────    ────────────────   ────────────               │
│   240p         (minimum)       250 kbps           -                          │
│   360p         350 kbps        400 kbps           250-350 kbps               │
│   480p         600 kbps        650 kbps           400-600 kbps               │
│   540p         950 kbps        1200 kbps          650-950 kbps               │
│   720p         1800 kbps       2000 kbps          1200-1800 kbps             │
│   1080p        2800 kbps       (maximum)          2000-2800 kbps             │
│                                                                              │
│                                                                              │
│   Visual representation:                                                     │
│                                                                              │
│   Bitrate:   0    250   350  400   600  650   950  1200  1800  2000  2800    │
│             │     │     │    │     │    │     │     │     │     │     │     │
│   240p:     ├─────┤                                                          │
│   360p:           ├────┬────┤                                                │
│   480p:                     ├────┬────┤                                      │
│   540p:                               ├─────┬─────┤                          │
│   720p:                                           ├─────┬─────┤              │
│   1080p:                                                       ├──────►      │
│                   │    │    │    │    │     │     │     │     │             │
│                   ↓    ↑    ↓    ↑    ↓     ↑     ↓     ↑     ↓             │
│               Downgrade Upgrade                                              │
│               thresholds thresholds                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Simple Code Implementation

```c
typedef struct {
    UINT16 width;
    UINT16 height;
    UINT32 upgradeThreshold;    // Bitrate must EXCEED this to upgrade TO this level
    UINT32 downgradeThreshold;  // Bitrate must DROP BELOW this to downgrade FROM this level
} ResolutionLevel;

static const ResolutionLevel LEVELS[] = {
    //  W      H     Upgrade↑  Downgrade↓
    {  320,  240,       0,        250 },   // Level 0: 240p
    {  480,  360,     350,        400 },   // Level 1: 360p
    {  640,  480,     600,        650 },   // Level 2: 480p
    {  960,  540,     950,       1200 },   // Level 3: 540p
    { 1280,  720,    1800,       2000 },   // Level 4: 720p
    { 1920, 1080,    2800,          0 },   // Level 5: 1080p
};

INT32 determineResolutionLevel(UINT32 bitrateKbps, INT32 currentLevel) 
{
    // Check if we should UPGRADE (go to higher level)
    // Must exceed the upgrade threshold of the NEXT level
    if (currentLevel < 5) {
        if (bitrateKbps > LEVELS[currentLevel + 1].upgradeThreshold) {
            return currentLevel + 1;  // Upgrade! 
        }
    }
    
    // Check if we should DOWNGRADE (go to lower level)
    // Must fall below the downgrade threshold of CURRENT level
    if (currentLevel > 0) {
        if (bitrateKbps < LEVELS[currentLevel]. downgradeThreshold) {
            return currentLevel - 1;  // Downgrade!
        }
    }
    
    // Stay at current level (in stable zone or at limits)
    return currentLevel;
}
```

### Example Traces

```
Example 1: Upgrading from 480p to 540p
──────────────────────────────────────
currentLevel = 2 (480p)
bitrateKbps = 980

Check upgrade:  Is 980 > LEVELS[3].upgradeThreshold (950)?
               Yes!  980 > 950
               → Return 3 (upgrade to 540p)


Example 2: Staying in stable zone
─────────────────────────────────
currentLevel = 3 (540p)
bitrateKbps = 850

Check upgrade: Is 850 > LEVELS[4].upgradeThreshold (1800)?
               No, 850 < 1800
               
Check downgrade: Is 850 < LEVELS[3].downgradeThreshold (1200)?
                 No, 850 > 1200...  wait, this is wrong! 
                 
Actually:  Is 850 < 1200? Yes, but we need to check the
          downgrade threshold which is when to LEAVE 540p. 
          
Correction: The downgrade threshold for 540p is 1200 kbps. 
           850 < 1200, so we SHOULD downgrade. 

Let me fix the threshold values... 
```

---

## Corrected Threshold Design

I need to clarify the threshold logic:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Corrected Threshold Logic                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   The thresholds define the BOUNDARIES of each resolution's operating range │
│                                                                              │
│   Resolution   Min Bitrate        Max Bitrate        Range                   │
│                (downgrade below)  (upgrade above)                            │
│   ──────────   ────────────────   ───────────────    ──────────────────     │
│   240p         (none - minimum)   300 kbps           0 - 300 kbps           │
│   360p         250 kbps           550 kbps           250 - 550 kbps         │
│   480p         450 kbps           900 kbps           450 - 900 kbps         │
│   540p         750 kbps           1400 kbps          750 - 1400 kbps        │
│   720p         1200 kbps          2400 kbps          1200 - 2400 kbps       │
│   1080p        2000 kbps          (none - maximum)   2000+ kbps             │
│                                                                              │
│   Notice: Ranges OVERLAP! This is the hysteresis.                            │
│                                                                              │
│   Example: 480p range is 450-900, 540p range is 750-1400                    │
│            Overlap zone:  750-900 kbps                                        │
│            In this zone:  stay at whatever resolution you're currently at    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

```
Bitrate:   0    250   450   550   750   900   1200  1400  2000  2400  ∞
           │     │     │     │     │     │     │     │     │     │    │
           │     │     │     │     │     │     │     │     │     │    │
   240p:   ├─────────────────┤                                        
                 │           │                                        
   360p:          ├───────────────────┤                                
                       │             │                                
   480p:                ├─────────────────────┤                        
                             │               │                        
   540p:                      ├───────────────────────┤                
                                   │                 │                
   720p:                           ├─────────────────────────┤        
                                         │                   │        
  1080p:                                 ├────────────────────────────►
           │     │     │     │     │     │     │     │     │     │    │
           
           ════ Overlap zones (hysteresis) ════
           
           At 500 kbps: Could be 360p or 480p (depends on current state)
           At 800 kbps: Could be 480p or 540p (depends on current state)
           At 1300 kbps: Could be 540p or 720p (depends on current state)
```

---

## Summary

**Hysteresis = Using different thresholds for going up vs.  going down**

```
┌────────────────────────────────────────────────────────────────┐
│                 Key Takeaways                                  │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. WITHOUT hysteresis:                                         │
│     Single threshold → Rapid flickering when near threshold   │
│                                                                │
│  2. WITH hysteresis:                                           │
│     Two thresholds → Stable zone prevents flickering          │
│                                                                │
│  3. Rule of thumb:                                              │
│     Upgrade threshold > Downgrade threshold                   │
│     Gap between them = stability buffer                       │
│                                                                │
│  4. Larger gap = more stable but slower to react              │
│     Smaller gap = faster reaction but more changes            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```
