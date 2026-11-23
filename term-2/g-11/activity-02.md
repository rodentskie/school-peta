# Activity 02: Serial RGB Color Mixer with Gamma Correction and EEPROM Memory

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-02`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Mix colors on an RGB LED by sending characters.

| Key | Action |
|---|---|
| `R` / `r` | red +16 / −16 |
| `G` / `g` | green +16 / −16 |
| `B` / `b` | blue +16 / −16 |
| `0`–`7` | preset colors: 0 black, 1 red, 2 green, 3 yellow, 4 blue, 5 magenta, 6 cyan, 7 white |
| `x` | swap red and blue |
| `c` | complementary color (255 − value for each channel) |
| `m` | save the color to EEPROM |
| `l` | load the color from EEPROM |
| `p` | print the color as `#RRGGBB` and as decimal values |
| `h` | help |

1. Values must stay in the range 0–255 (no wrap-around): `R` at 250 gives 255, not 10.
2. The human eye does not see brightness linearly. Apply **gamma correction** (γ = 2.2) before `analogWrite()`, so that the steps look even.
3. Print the new color after every change. `RRRR` in one line must apply four steps.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | RGB LED (common cathode) + 1 kΩ resistor on the cathode |

## Solution

* The **logical** color is stored in `int rgb[3]`. Commands change it with `adjust(ch, delta)`, which uses `constrain(v, 0, 255)`.
* **Gamma correction:** `out = pow(v / 255.0, 2.2) × 255`. For example, a logical 128 (50 %) becomes about 56. This makes the LED look half as bright, as the eye expects.
* **Preset colors** use the bits of the digit: bit 0 = red, bit 1 = green, bit 2 = blue. So `3` = `0b011` = red + green = yellow.
* **Hex printing:** each value is printed with 2 hex digits (add a leading `0` if the value < 16).
* **EEPROM:** addresses 0–2 store R, G, B, and address 3 stores the marker 0xA5, which means "valid data saved".

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
// G11 Activity 02 - Serial RGB Mixer with Gamma and EEPROM
#include <EEPROM.h>

const int PINS[3] = {9, 10, 11};       // R G B
const char NAMES[3] = {'R', 'G', 'B'};
const int STEP = 16;

int rgb[3] = {0, 0, 0};

void apply() {
  for (int i = 0; i < 3; i++) {
    int out = (int)(pow(rgb[i] / 255.0, 2.2) * 255.0 + 0.5);
    analogWrite(PINS[i], out);
  }
}

void printHex2(int v) {
  if (v < 16) Serial.print('0');
  Serial.print(v, HEX);
}

void printColor() {
  Serial.print("Color #");
  for (int i = 0; i < 3; i++) printHex2(rgb[i]);
  Serial.print("  (R=");
  Serial.print(rgb[0]);
  Serial.print(" G=");
  Serial.print(rgb[1]);
  Serial.print(" B=");
  Serial.print(rgb[2]);
  Serial.println(")");
}

void adjust(int ch, int delta) {
  rgb[ch] = constrain(rgb[ch] + delta, 0, 255);
}

void help() {
  Serial.println("--- RGB MIXER ---");
  Serial.println(" R/r G/g B/b : +16 / -16");
  Serial.println(" 0-7 presets | x swap R<->B | c complement");
  Serial.println(" m save | l load | p print | h help");
}

void handle(char c) {
  bool changed = true;
  if (c == 'R') adjust(0, STEP);
  else if (c == 'r') adjust(0, -STEP);
  else if (c == 'G') adjust(1, STEP);
  else if (c == 'g') adjust(1, -STEP);
  else if (c == 'B') adjust(2, STEP);
  else if (c == 'b') adjust(2, -STEP);
  else if (c >= '0' && c <= '7') {
    int n = c - '0';
    for (int i = 0; i < 3; i++) rgb[i] = ((n >> i) & 1) ? 255 : 0;
  }
  else if (c == 'x') { int t = rgb[0]; rgb[0] = rgb[2]; rgb[2] = t; }
  else if (c == 'c') { for (int i = 0; i < 3; i++) rgb[i] = 255 - rgb[i]; }
  else if (c == 'm') {
    for (int i = 0; i < 3; i++) EEPROM.update(i, rgb[i]);
    EEPROM.update(3, 0xA5);
    Serial.println("Saved to EEPROM");
    changed = false;
  }
  else if (c == 'l') {
    if (EEPROM.read(3) == 0xA5) {
      for (int i = 0; i < 3; i++) rgb[i] = EEPROM.read(i);
      Serial.println("Loaded from EEPROM");
    } else {
      Serial.println("Nothing saved yet");
      changed = false;
    }
  }
  else if (c == 'p') { printColor(); changed = false; }
  else if (c == 'h' || c == '?') { help(); changed = false; }
  else if (c == '\n' || c == '\r' || c == ' ') changed = false;
  else {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
    changed = false;
  }

  if (changed) {
    apply();
    printColor();
  }
}

void setup() {
  for (int i = 0; i < 3; i++) pinMode(PINS[i], OUTPUT);
  Serial.begin(9600);
  help();
  apply();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
}
```

## Assessment Questions

1. Send `R` four times. What happens to the RGB LED each time? What happens when you keep sending `R` after the red is already at maximum?
2. Send each digit from `0` to `7`. Which color appears for each one?
3. Send `3` and then `c`. What color do you see before and after?
4. Send `m`, stop the simulation, start it again and send `l`. What color appears? Why?
5. What will happen to the RGB LED if the resistor on the common (cathode) leg is removed? What happens if the wire to the green leg (D10) is disconnected?
