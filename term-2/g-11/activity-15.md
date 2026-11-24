# Activity 15: Serial-Set NeoPixel Ring Clock

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-15`
**Control:** Serial Monitor (type characters and click **Send**)

## Problem Statement

Use a 12-pixel NeoPixel ring as an **analog clock face**. The time is set from the Serial Monitor.

| Key | Action |
|---|---|
| `H` / `h` | hour +1 / −1 |
| `M` / `m` | minute +1 / −1 |
| `Z` | set seconds to 0 |
| `f` | fast mode ON/OFF (1 real second = 1 clock minute, for testing) |
| `t` | print the time `HH:MM:SS` |
| `b` | brightness cycle (low / medium / high) |
| `?` | help |

**Clock face rules:**

1. Pixel 0 is "12 o'clock". The 12 pixels are the 12 hour marks.
2. **Hour** hand: the pixel `hour % 12` is **red**.
3. **Minute** hand: pixel `minute / 5` is **green**.
4. **Second** hand: pixel `second / 5` is **blue**.
5. If hands are on the same pixel, the colors **add** (red + green = yellow, etc.).
6. The empty pixels at 12, 3, 6 and 9 o'clock show a very dim white marker, so the face is easy to read.

The time starts at `10:08:00`. The clock must keep counting without drift (use `lastTick += 1000`).

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | NeoPixel Ring 12 |

## Solution

* **Time** is stored as `long secs` (0–86399). `H` adds 3600 and `M` adds 60, both modulo 86400. Negative changes add `86400 − x` to avoid negative numbers.
* **Fast mode:** each tick adds 60 instead of 1.
* **Drawing:** start with three arrays `r[12], g[12], b[12]` = 0. Add 255 to the channel of each hand's pixel, `constrain` to 255, add the markers, then send them to the ring.
* The ring is redrawn only when the second changes (once per tick) or after a command.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| NeoPixel Ring | Input | **D6** |
| NeoPixel Ring | Power | 5V rail |
| NeoPixel Ring | Ground | GND rail |

```
D6 ─► RING DIN     RING PWR ─ 5V rail     RING GND ─ GND rail
```

## Code

```cpp
// G11 Activity 15 - Serial-Set NeoPixel Ring Clock
#include <Adafruit_NeoPixel.h>

const int RING_PIN = 6;
const int NUM = 12;
Adafruit_NeoPixel ring(NUM, RING_PIN, NEO_GRB + NEO_KHZ800);

long secs = 10L * 3600 + 8 * 60;     // 10:08:00
bool fast = false;
unsigned long lastTick = 0;
const byte LEVELS[3] = {20, 60, 150};
int level = 1;

void printTime() {
  int h = secs / 3600, m = (secs / 60) % 60, s = secs % 60;
  if (h < 10) Serial.print('0');
  Serial.print(h);
  Serial.print(':');
  if (m < 10) Serial.print('0');
  Serial.print(m);
  Serial.print(':');
  if (s < 10) Serial.print('0');
  Serial.println(s);
}

void draw() {
  int r[NUM], g[NUM], b[NUM];
  for (int i = 0; i < NUM; i++) { r[i] = 0; g[i] = 0; b[i] = 0; }
  int h = (secs / 3600) % 12;
  int m = (secs / 60) % 60;
  int s = secs % 60;
  r[h] += 255;
  g[m / 5] += 255;
  b[s / 5] += 255;
  for (int i = 0; i < NUM; i += 3) {
    if (r[i] == 0 && g[i] == 0 && b[i] == 0) { r[i] = 8; g[i] = 8; b[i] = 8; }
  }
  for (int i = 0; i < NUM; i++) {
    ring.setPixelColor(i, ring.Color(constrain(r[i], 0, 255), constrain(g[i], 0, 255), constrain(b[i], 0, 255)));
  }
  ring.show();
}

void adjust(long delta) {
  secs = (secs + delta + 86400L) % 86400L;
  printTime();
  draw();
}

void help() {
  Serial.println("--- RING CLOCK ---");
  Serial.println(" H/h hour +/- | M/m minute +/- | Z seconds=0");
  Serial.println(" f fast mode | t time | b brightness | ? help");
  Serial.println(" red = hour, green = minute, blue = second");
}

void handle(char c) {
  if (c == 'H') adjust(3600);
  else if (c == 'h') adjust(-3600);
  else if (c == 'M') adjust(60);
  else if (c == 'm') adjust(-60);
  else if (c == 'Z') adjust(-(secs % 60));
  else if (c == 'f') { fast = !fast; Serial.println(fast ? "FAST mode (1 s = 1 min)" : "Normal speed"); }
  else if (c == 't') printTime();
  else if (c == 'b') {
    level = (level + 1) % 3;
    ring.setBrightness(LEVELS[level]);
    draw();
    Serial.print("Brightness ");
    Serial.println(LEVELS[level]);
  }
  else if (c == '?') help();
  else if (c != '\n' && c != '\r' && c != ' ') {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
  }
}

void setup() {
  Serial.begin(9600);
  ring.begin();
  ring.setBrightness(LEVELS[level]);
  help();
  printTime();
  draw();
  lastTick = millis();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();
  if (now - lastTick >= 1000) {
    lastTick += 1000;
    secs = (secs + (fast ? 60 : 1)) % 86400L;
    draw();
    if (secs % 60 == 0 || fast) printTime();
  }
}
```

## Assessment Questions

1. Right after the simulation starts, which pixels are red, green and blue? Which pixels show a dim white marker?
2. Watch the ring for 10 seconds. Which color moves? How often does it move to the next pixel?
3. Send `H` a few times, then `M` a few times. Which hand moves for each command?
4. Send `f`. What changes? What color does a pixel turn when the hour and minute hands meet?
5. If the ring's Ground wire is disconnected, what happens to the clock face? Does the Serial Monitor still print the time with `t`?
