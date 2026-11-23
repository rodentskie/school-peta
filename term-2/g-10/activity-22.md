# Activity 22: NeoPixel Ring Animation Controller

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-22`

## Problem Statement

Program a 12-LED **NeoPixel ring** (addressable RGB LEDs) with four animations:

| Mode | Animation |
|---|---|
| 0 | **Rainbow wheel**: all 12 colors of the rainbow rotate around the ring |
| 1 | **Comet**: one bright dot runs around with a fading tail of 5 pixels |
| 2 | **Theater chase**: every third pixel is lit and the pattern shifts |
| 3 | **Color wipe**: the ring fills with a color one pixel at a time, then wipes with the next color |

1. The potentiometer sets the animation **speed** (frame interval 15–200 ms).
2. **Short press:** next animation. **Long press:** cycle global brightness (20 → 60 → 120 → 20 …).
3. Each animation is a **frame function** that draws one frame from a frame counter. No `delay()` and no blocking loops are allowed.
4. Print the mode name and brightness when they change.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | NeoPixel Ring 12 |
| 1 | Potentiometer |
| 1 | Pushbutton |

## Solution

* **Adafruit_NeoPixel** library: `ring.setPixelColor(i, ring.Color(r, g, b))` sets pixel *i* and `ring.show()` sends the data. `ring.setBrightness()` scales all colors.
* **Color wheel:** `wheel(pos)` turns 0–255 into a rainbow color (3 sectors of 85 steps).
* **Frame-based animation:** `frame` increases each time the interval passes. Every animation calculates its picture from `frame`:
  * Rainbow: pixel *i* gets `wheel((i × 256 / 12 + frame × 4) & 255)`.
  * Comet: head = `frame % 12`. Pixel at distance *d* behind the head gets brightness `255 >> (d × 1.5)`, giving an exponential fade.
  * Theater: pixel lit if `(i + frame) % 3 == 0`.
  * Wipe: `n = frame % 13` pixels lit with color number `(frame / 13) % 4`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| NeoPixel Ring | Input (DIN) | **D6** |
| NeoPixel Ring | Power / Ground | 5V rail / GND rail |
| Potentiometer | T1 / Wiper / T2 | GND / **A0** / 5V |
| Pushbutton | T1 / T2 | GND / **D2** |

```
D6 ─► RING DIN     RING 5V ─ 5V rail    RING GND ─ GND rail
A0 ◄─ POT wiper    D2 ─► BUTTON ── GND
```

## Code

```cpp
// Activity 22 - NeoPixel Ring Animation Controller
#include <Adafruit_NeoPixel.h>

const int RING_PIN = 6;
const int NUM = 12;
const int POT_PIN = A0;
const int BTN_PIN = 2;

Adafruit_NeoPixel ring(NUM, RING_PIN, NEO_GRB + NEO_KHZ800);

const char* NAMES[4] = {"Rainbow", "Comet", "Theater chase", "Color wipe"};
const byte LEVELS[3] = {20, 60, 120};
int mode = 0;
int levelIdx = 1;
unsigned long frame = 0;
unsigned long lastFrame = 0;

int btnStable = HIGH, btnRaw = HIGH;
unsigned long btnChanged = 0, btnPressedAt = 0;
bool longDone = false;

uint32_t wheel(byte pos) {
  pos = 255 - pos;
  if (pos < 85) return ring.Color(255 - pos * 3, 0, pos * 3);
  if (pos < 170) {
    pos -= 85;
    return ring.Color(0, pos * 3, 255 - pos * 3);
  }
  pos -= 170;
  return ring.Color(pos * 3, 255 - pos * 3, 0);
}

void rainbow() {
  for (int i = 0; i < NUM; i++) ring.setPixelColor(i, wheel((i * 256 / NUM + frame * 4) & 255));
}

void comet() {
  int head = frame % NUM;
  ring.clear();
  for (int d = 0; d < 6; d++) {
    int idx = (head - d + NUM) % NUM;
    int b = 255 >> (int)(d * 1.5);
    ring.setPixelColor(idx, ring.Color(b / 4, b / 2, b));
  }
}

void theater() {
  uint32_t c = wheel((frame * 2) & 255);
  for (int i = 0; i < NUM; i++) ring.setPixelColor(i, ((i + frame) % 3 == 0) ? c : 0);
}

void wipe() {
  const uint32_t colors[4] = {ring.Color(255, 0, 0), ring.Color(0, 255, 0),
                              ring.Color(0, 0, 255), ring.Color(255, 160, 0)};
  int n = frame % (NUM + 1);
  uint32_t now = colors[(frame / (NUM + 1)) % 4];
  uint32_t prev = colors[((frame / (NUM + 1)) + 3) % 4];
  for (int i = 0; i < NUM; i++) ring.setPixelColor(i, i < n ? now : prev);
}

void announce() {
  Serial.print("Mode: ");
  Serial.print(NAMES[mode]);
  Serial.print("  Brightness: ");
  Serial.println(LEVELS[levelIdx]);
}

void handleButton() {
  int r = digitalRead(BTN_PIN);
  unsigned long now = millis();
  if (r != btnRaw) { btnRaw = r; btnChanged = now; }
  if (now - btnChanged > 20 && r != btnStable) {
    btnStable = r;
    if (r == LOW) { btnPressedAt = now; longDone = false; }
    else if (!longDone) {
      mode = (mode + 1) % 4;
      frame = 0;
      announce();
    }
  }
  if (btnStable == LOW && !longDone && now - btnPressedAt >= 1000) {
    longDone = true;
    levelIdx = (levelIdx + 1) % 3;
    ring.setBrightness(LEVELS[levelIdx]);
    announce();
  }
}

void setup() {
  pinMode(BTN_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  ring.begin();
  ring.setBrightness(LEVELS[levelIdx]);
  ring.show();
  announce();
}

void loop() {
  handleButton();
  unsigned long interval = map(analogRead(POT_PIN), 0, 1023, 15, 200);
  unsigned long now = millis();
  if (now - lastFrame < interval) return;
  lastFrame = now;

  if (mode == 0) rainbow();
  else if (mode == 1) comet();
  else if (mode == 2) theater();
  else wipe();
  ring.show();
  frame++;
}
```

## Assessment Questions

1. Describe each of the 4 animations on the ring.
2. What happens to the animation when you turn the potentiometer?
3. What happens when you hold the button for more than 1 second?
4. If the ring's Input wire is moved from D6 to D5, what will happen to the ring?
5. What happens to the ring if its power wire is disconnected?
