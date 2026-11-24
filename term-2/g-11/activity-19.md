# Activity 19: Serial RGB Light-Show Sequencer

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-19`
**Control:** Serial Monitor (type characters and click **Send**)

## Problem Statement

Program a light show for an RGB LED by typing a **sequence of color letters**, like a stage-light controller.

| Character | Meaning |
|---|---|
| `r g b y c m w k` | add a step: red, green, blue, yellow, cyan, magenta, white, black (off) |
| `p` | play the sequence in a loop / stop |
| `f` | transition mode: **cut** (instant) ↔ **fade** (smooth blend) |
| `+` / `-` | step time shorter / longer (100–3000 ms) |
| `<` | delete the last step |
| `x` | clear the sequence |
| `l` | list the steps |

1. The sequence holds up to **24 steps**. Sending `rgb` adds three steps.
2. While playing, print the step number and color name each time it changes.
3. In **fade** mode, the LED blends **linearly** from the current color to the next color over the whole step time. In **cut** mode, it jumps.
4. Adding or deleting steps while playing must not crash the show. The player keeps the index in range.
5. Start with the demo sequence `rygcbm` playing in fade mode.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | RGB LED (common cathode) + 1 kΩ resistor on the cathode |

## Solution

* **Colors** are stored as 3 bytes in a table `PALETTE[8][3]`, with the letter in `KEYS = "rgbycmwk"`. The sequence stores only the palette **index** of each step (`byte seq[24]`), so it uses very little memory.
* **Player:** `stepIdx` and `stepStart`. When `now − stepStart ≥ stepMs`, go to `(stepIdx + 1) % seqLen`.
* **Linear blend (fade):** `k = (now − stepStart) / stepMs` (0 → 1),
  `out = from + (to − from) × k` for each channel. `from` is the current step's color and `to` is the next step's color.
* Every time the sequence changes, `stepIdx` is checked with `if (stepIdx >= seqLen) stepIdx = 0`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| RGB LED | Red | **D9** (PWM) |
| RGB LED | Green | **D10** (PWM) |
| RGB LED | Blue | **D11** (PWM) |
| RGB LED | Cathode | 1 kΩ → GND rail |

```
D9  ─► R ┐
D10 ─► G ├─ RGB LED ─ K ─[1k]─ GND
D11 ─► B ┘
```

## Code

```cpp
// G11 Activity 19 - Serial RGB Light-Show Sequencer
const int PINS[3] = {9, 10, 11};
const char KEYS[] = "rgbycmwk";
const char* NAMES[8] = {"red", "green", "blue", "yellow", "cyan", "magenta", "white", "off"};
const byte PALETTE[8][3] = {
  {255, 0, 0}, {0, 255, 0}, {0, 0, 255}, {255, 180, 0},
  {0, 255, 255}, {255, 0, 255}, {255, 255, 255}, {0, 0, 0}
};
const int MAXSTEPS = 24;

byte seq[MAXSTEPS];
int seqLen = 0;
bool playing = false;
bool fade = true;
unsigned int stepMs = 800;
int stepIdx = 0;
unsigned long stepStart = 0;

int keyIndex(char c) {
  for (int i = 0; i < 8; i++) if (KEYS[i] == c) return i;
  return -1;
}

void setRGB(int r, int g, int b) {
  analogWrite(PINS[0], r);
  analogWrite(PINS[1], g);
  analogWrite(PINS[2], b);
}

void list() {
  Serial.print("Sequence (");
  Serial.print(seqLen);
  Serial.print("): ");
  for (int i = 0; i < seqLen; i++) {
    Serial.print(KEYS[seq[i]]);
  }
  Serial.print("  | ");
  Serial.print(fade ? "fade" : "cut");
  Serial.print(" | step ");
  Serial.print(stepMs);
  Serial.println(" ms");
}

void announceStep() {
  Serial.print("Step ");
  Serial.print(stepIdx + 1);
  Serial.print(": ");
  Serial.println(NAMES[seq[stepIdx]]);
}

void handle(char c) {
  int k = keyIndex(c);
  if (k >= 0) {
    if (seqLen >= MAXSTEPS) { Serial.println("Sequence full"); return; }
    seq[seqLen++] = k;
    list();
    return;
  }
  if (c == 'p') {
    if (seqLen == 0) { Serial.println("Sequence is empty"); return; }
    playing = !playing;
    stepIdx = 0;
    stepStart = millis();
    Serial.println(playing ? "PLAY" : "STOP");
    if (playing) announceStep(); else setRGB(0, 0, 0);
  }
  else if (c == 'f') { fade = !fade; list(); }
  else if (c == '+') { stepMs = max(100, stepMs / 2); list(); }
  else if (c == '-') { stepMs = min(3000, stepMs * 2); list(); }
  else if (c == '<') { if (seqLen > 0) seqLen--; list(); }
  else if (c == 'x') { seqLen = 0; playing = false; setRGB(0, 0, 0); list(); }
  else if (c == 'l') list();
  else if (c != '\n' && c != '\r' && c != ' ') {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("' (colors: r g b y c m w k)");
  }
  if (seqLen == 0) playing = false;
  if (stepIdx >= seqLen) stepIdx = 0;
}

void setup() {
  for (int i = 0; i < 3; i++) pinMode(PINS[i], OUTPUT);
  Serial.begin(9600);
  Serial.println("RGB sequencer: r g b y c m w k add steps | p play | f fade/cut | + - speed | < del | x clear | l list");
  const char *demo = "rygcbm";
  for (int i = 0; demo[i]; i++) handle(demo[i]);
  handle('p');
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  if (!playing || seqLen == 0) return;

  unsigned long now = millis();
  if (now - stepStart >= stepMs) {
    stepStart = now;
    stepIdx = (stepIdx + 1) % seqLen;
    announceStep();
  }
  const byte *from = PALETTE[seq[stepIdx]];
  if (!fade) {
    setRGB(from[0], from[1], from[2]);
    return;
  }
  const byte *to = PALETTE[seq[(stepIdx + 1) % seqLen]];
  float k = (float)(now - stepStart) / stepMs;
  int out[3];
  for (int i = 0; i < 3; i++) out[i] = from[i] + (int)((to[i] - from[i]) * k);
  setRGB(out[0], out[1], out[2]);
}
```

## Assessment Questions

1. What does the RGB LED do right after the simulation starts? What does the Serial Monitor print?
2. Send `f`. How does the light show look different? Send `f` again.
3. Send `+` twice and then `-` three times. What changes?
4. Send `x`, then `rk` and `p`. What does the LED do now?
5. If the wire to the green leg (D10) is disconnected, which steps of the demo show will look wrong? What will yellow look like?
