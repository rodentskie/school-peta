# Activity 07: HSV Color Wheel with an RGB LED

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-07`

## Problem Statement

Mixing colors by setting red, green and blue directly is not intuitive. Artists use **Hue–Saturation–Value (HSV)** instead. Build a color picker:

1. The potentiometer selects the **hue** (0°–359°) around the color wheel: red → yellow → green → cyan → blue → magenta → red.
2. A **short press** of the button cycles the brightness (**value**) through 25 %, 50 %, 75 % and 100 %.
3. A **long press** (≥ 1 s) toggles **Auto-Rainbow** mode. The hue then rotates by itself (one full turn every 6 s) and the knob is ignored.
4. Write your own `hsvToRgb()` function using **integer math only** (no `float`).
5. Print hue, brightness and the resulting R, G, B values whenever they change.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | RGB LED (common cathode) + 1 kΩ resistor on the cathode |
| 1 | Potentiometer |
| 1 | Pushbutton |

## Solution

**HSV → RGB algorithm** (saturation fixed at 100 %):

1. Split the hue circle into 6 sectors of 60°: `sector = h / 60`.
2. Find the position inside the sector, scaled to 0–255: `f = (h % 60) * 255 / 60`.
3. One channel is at maximum, one is at zero, and the third rises (`f`) or falls (`255 − f`):

| sector | R | G | B |
|---|---|---|---|
| 0 | 255 | f | 0 |
| 1 | 255−f | 255 | 0 |
| 2 | 0 | 255 | f |
| 3 | 0 | 255−f | 255 |
| 4 | f | 0 | 255 |
| 5 | 255 | 0 | 255−f |

4. Multiply each channel by the brightness percentage: `c * v / 100`.

The button uses the same short/long press logic as earlier activities. Auto-Rainbow uses `hue = (millis() / (6000/360)) % 360`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| RGB LED | Red | **D9** (PWM) |
| RGB LED | Green | **D10** (PWM) |
| RGB LED | Blue | **D11** (PWM) |
| RGB LED | Cathode (common) | 1 kΩ → GND rail |
| Potentiometer | T1 / Wiper / T2 | GND / **A0** / 5V |
| Pushbutton | T1 / T2 | GND / **D2** |

```
D9  ─► R ┐
D10 ─► G ├─ RGB LED ── K ─[1k]─ GND       A0 ◄─ POT wiper
D11 ─► B ┘                                D2 ─► BUTTON ── GND
```

## Code

```cpp
// Activity 07 - HSV Color Wheel
const int R_PIN = 9;
const int G_PIN = 10;
const int B_PIN = 11;
const int POT_PIN = A0;
const int BTN_PIN = 2;

const int LEVELS[4] = {25, 50, 75, 100};
int levelIdx = 3;
bool autoRainbow = false;

int lastHue = -1, lastLevel = -1;

int btnStable = HIGH, btnRaw = HIGH;
unsigned long btnChanged = 0, btnPressedAt = 0;
bool longDone = false;

int outR, outG, outB;

// hue 0..359, value 0..100 (saturation = 100%), integer math only
void hsvToRgb(int h, int v) {
  int sector = h / 60;
  int f = (long)(h % 60) * 255 / 60;
  int r = 0, g = 0, b = 0;
  switch (sector) {
    case 0: r = 255;     g = f;       b = 0;       break;
    case 1: r = 255 - f; g = 255;     b = 0;       break;
    case 2: r = 0;       g = 255;     b = f;       break;
    case 3: r = 0;       g = 255 - f; b = 255;     break;
    case 4: r = f;       g = 0;       b = 255;     break;
    default: r = 255;    g = 0;       b = 255 - f; break;
  }
  outR = (long)r * v / 100;
  outG = (long)g * v / 100;
  outB = (long)b * v / 100;
}

void handleButton() {
  int r = digitalRead(BTN_PIN);
  unsigned long now = millis();
  if (r != btnRaw) { btnRaw = r; btnChanged = now; }
  if (now - btnChanged > 20 && r != btnStable) {
    btnStable = r;
    if (r == LOW) { btnPressedAt = now; longDone = false; }
    else if (!longDone) levelIdx = (levelIdx + 1) % 4;      // short press
  }
  if (btnStable == LOW && !longDone && now - btnPressedAt >= 1000) {
    longDone = true;
    autoRainbow = !autoRainbow;
    Serial.println(autoRainbow ? "AUTO RAINBOW ON" : "MANUAL (knob)");
  }
}

void setup() {
  pinMode(R_PIN, OUTPUT);
  pinMode(G_PIN, OUTPUT);
  pinMode(B_PIN, OUTPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  Serial.println("HSV color wheel: knob = hue, press = brightness, hold = auto");
}

void loop() {
  handleButton();

  int hue;
  if (autoRainbow) hue = (millis() / 16) % 360;          // 360 steps * 16 ms ~ 6 s
  else hue = map(analogRead(POT_PIN), 0, 1023, 0, 359);

  int v = LEVELS[levelIdx];
  hsvToRgb(hue, v);
  analogWrite(R_PIN, outR);
  analogWrite(G_PIN, outG);
  analogWrite(B_PIN, outB);

  // print only on meaningful change (avoid flooding the monitor)
  if (abs(hue - lastHue) >= 5 || v != lastLevel) {
    lastHue = hue;
    lastLevel = v;
    Serial.print("H=");
    Serial.print(hue);
    Serial.print(" V=");
    Serial.print(v);
    Serial.print("% -> R=");
    Serial.print(outR);
    Serial.print(" G=");
    Serial.print(outG);
    Serial.print(" B=");
    Serial.println(outB);
  }
}
```

## Assessment Questions

1. Using the table, calculate R, G, B for hue = 30°, 120°, 200° and 300° at 100 % brightness.
2. Why is `(long)` used in `(long)(h % 60) * 255 / 60`? What could go wrong on an Arduino Uno if it were removed? (Hint: what is the size of `int`?)
3. What color do you see at hue = 60° and at hue = 180°? Explain using additive color mixing.
4. Why does the program only print when the hue changes by 5° or more?
5. Add a **saturation** control: a second press type (double press) cycles saturation between 100 %, 60 % and 30 %. How does saturation change the formula? (Hint: a white part is added to every channel.)
