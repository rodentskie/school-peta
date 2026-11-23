# Activity 23: NeoPixel Pomodoro Focus Timer

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-23`

## Problem Statement

The **Pomodoro technique** splits study time into focus sessions and breaks. Build a visual Pomodoro timer on a 12-LED NeoPixel ring.

1. Sequence: **Work → Short break → Work → Short break → Work → Short break → Work → Long break**, then repeat.
2. For the classroom demo, time is scaled so that **1 "minute" = 1 second**: Work = 25 s, Short break = 5 s, Long break = 15 s.
3. The ring shows **progress** as a filling circle: red for work, green for short break, blue for long break. The leading pixel is dimmed **proportionally**, so the progress moves smoothly between pixels.
4. **Short press:** start / pause. While paused, the whole ring **breathes** slowly in the phase color.
5. **Long press (≥ 1 s):** reset to the first Work phase (stopped).
6. When a phase ends, play a chime (3 notes) and print the next phase and the completed work sessions.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | NeoPixel Ring 12 |
| 1 | Pushbutton |
| 1 | Piezo buzzer |

## Solution

* `PHASES[8]` holds the type of each step (0 = work, 1 = short break, 2 = long break), and `DURATION[3]` holds their lengths.
* **Elapsed time with pause:** `elapsed = accumulated + (running ? now − runStart : 0)`. On pause, the running time is added to `accumulated`.
* **Smooth progress:** `p = elapsed × 12 / duration` (a float from 0 to 12). Pixels below `floor(p)` are full. The next pixel gets `(p − floor(p)) × 255` brightness, and the rest are off.
* **Breathing:** brightness = `(1 − cos(2πt / 2000)) / 2`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| NeoPixel Ring | Input | **D6** |
| NeoPixel Ring | Power / Ground | 5V rail / GND rail |
| Pushbutton | T1 / T2 | GND / **D2** |
| Piezo | + / − | **D9** / GND |

```
D6 ─► RING DIN   (5V / GND from rails)
D2 ─► BUTTON ── GND        D9 ─► PIEZO ── GND
```

## Code

```cpp
// Activity 23 - NeoPixel Pomodoro Timer
#include <Adafruit_NeoPixel.h>

const int RING_PIN = 6;
const int NUM = 12;
const int BTN_PIN = 2;
const int BUZZER_PIN = 9;

Adafruit_NeoPixel ring(NUM, RING_PIN, NEO_GRB + NEO_KHZ800);

const byte PHASES[8] = {0, 1, 0, 1, 0, 1, 0, 2};
const unsigned long DURATION[3] = {25000UL, 5000UL, 15000UL};   // demo: 1 min = 1 s
const char* NAMES[3] = {"WORK", "SHORT BREAK", "LONG BREAK"};
const byte COLR[3] = {255, 0, 0};
const byte COLG[3] = {0, 255, 40};
const byte COLB[3] = {0, 0, 255};

int step = 0;
bool running = false;
unsigned long runStart = 0;
unsigned long accumulated = 0;
int sessionsDone = 0;

int btnStable = HIGH, btnRaw = HIGH;
unsigned long btnChanged = 0, btnPressedAt = 0;
bool longDone = false;

int phase() { return PHASES[step]; }

unsigned long elapsed() {
  return accumulated + (running ? millis() - runStart : 0);
}

void chime() {
  tone(BUZZER_PIN, 784, 120);
  delay(150);
  tone(BUZZER_PIN, 988, 120);
  delay(150);
  tone(BUZZER_PIN, 1319, 200);
}

void announce() {
  Serial.print("Phase ");
  Serial.print(step + 1);
  Serial.print("/8: ");
  Serial.print(NAMES[phase()]);
  Serial.print("  (work sessions done: ");
  Serial.print(sessionsDone);
  Serial.println(")");
}

void resetAll() {
  step = 0;
  running = false;
  accumulated = 0;
  sessionsDone = 0;
  Serial.println("RESET - press to start");
  announce();
}

void handleButton() {
  int r = digitalRead(BTN_PIN);
  unsigned long now = millis();
  if (r != btnRaw) { btnRaw = r; btnChanged = now; }
  if (now - btnChanged > 20 && r != btnStable) {
    btnStable = r;
    if (r == LOW) { btnPressedAt = now; longDone = false; }
    else if (!longDone) {
      if (running) {
        accumulated += now - runStart;
        running = false;
        Serial.println("PAUSED");
      } else {
        runStart = now;
        running = true;
        Serial.println("RUNNING");
      }
    }
  }
  if (btnStable == LOW && !longDone && now - btnPressedAt >= 1000) {
    longDone = true;
    resetAll();
  }
}

void drawProgress(float p, int ph) {
  int full = (int)p;
  float frac = p - full;
  for (int i = 0; i < NUM; i++) {
    float k = 0;
    if (i < full) k = 1;
    else if (i == full) k = frac;
    ring.setPixelColor(i, ring.Color(COLR[ph] * k, COLG[ph] * k, COLB[ph] * k));
  }
}

void drawBreathing(int ph, float progress) {
  float k = 0.1 + 0.9 * (1 - cos(2 * PI * (millis() % 2000) / 2000.0)) / 2;
  int lit = (int)progress;
  for (int i = 0; i < NUM; i++) {
    float kk = (i < lit) ? k : k * 0.15;
    ring.setPixelColor(i, ring.Color(COLR[ph] * kk, COLG[ph] * kk, COLB[ph] * kk));
  }
}

void setup() {
  pinMode(BTN_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  ring.begin();
  ring.setBrightness(80);
  Serial.println("Pomodoro timer (demo: 1 minute = 1 second)");
  resetAll();
  running = true;                 // auto-start for the demo
  runStart = millis();
}

void loop() {
  handleButton();
  int ph = phase();
  unsigned long dur = DURATION[ph];
  unsigned long e = elapsed();

  if (e >= dur) {
    if (ph == 0) sessionsDone++;
    step = (step + 1) % 8;
    accumulated = 0;
    runStart = millis();
    chime();
    announce();
    return;
  }

  float p = e * (float)NUM / dur;
  if (running) drawProgress(p, ph);
  else drawBreathing(ph, p);
  ring.show();
  delay(10);
}
```

## Assessment Questions

1. If 13.75 s of a 25 s work phase have passed, calculate `p`, the number of full pixels, and the brightness (0–255) of the leading pixel.
2. Explain how the variables `accumulated`, `runStart` and `running` work together to support pausing. Trace them for: start → 4 s → pause → 3 s → resume → 2 s.
3. In the real Pomodoro technique, how long does one full cycle of 8 phases last (in minutes)? What single change to the code makes the timer run in real minutes?
4. The `chime()` function uses `delay()`. Why is this acceptable here, but `delay()` inside `drawProgress()` would not be?
5. Add a **"skip"** feature: double-pressing the button (two presses within 400 ms) jumps to the next phase immediately. Describe how to detect a double press without confusing it with a single press.
