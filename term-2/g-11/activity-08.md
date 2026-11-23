# Activity 08: Serial NeoPixel Ring Painter

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-08`
**Control:** Serial Monitor (type characters and click **Send**)

## Problem Statement

Make a small "paint program" for a 12-pixel NeoPixel ring, controlled with the keyboard.

| Key | Action |
|---|---|
| `a` / `d` | move the cursor counter-clockwise / clockwise (wraps around) |
| `r` `g` `b` `y` `w` `p` `o` | choose brush color: red, green, blue, yellow, white, purple, orange |
| `x` | paint the pixel under the cursor |
| `e` | erase the pixel under the cursor |
| `f` | fill the whole ring with the brush color |
| `c` | clear the canvas |
| `i` | invert all colors |
| `R` | rotate animation ON/OFF (the painting spins) |
| `+` / `-` | brightness up / down (10–150) |
| `v` | print the canvas as a text list |
| `h` | help |

1. The **cursor** is shown as a blinking pixel (white blink over the painted color) so the user can see where they are.
2. Commands can be combined in one line, e.g. `rxdxdxbx` paints red, red, red, blue in 4 neighboring pixels.
3. The painting is stored in an array `canvas[12]` (32-bit colors), separate from what is displayed. Rotation changes only the **view offset**, not the data.
4. `v` prints each pixel as `index: #RRGGBB`.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | NeoPixel Ring 12 |

## Solution

* **Model vs view:** `canvas[]` is the model (the data). `render()` builds the view: pixel `(i + offset) % 12` shows `canvas[i]`, and the cursor blinks on top. This separation makes rotation and the cursor easy.
* **Colors** are packed as `0xRRGGBB` in `uint32_t`. Extract the parts with shifts: `r = (c >> 16) & 0xFF`, and so on.
* **Invert:** `c = (~c) & 0xFFFFFF` for each pixel.
* **Cursor movement:** `cursor = (cursor + 1) % 12` and `(cursor + 11) % 12`. Adding 11 means "−1" without negative numbers.
* `render()` runs every 30 ms, so the blink and rotation are smooth and non-blocking.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| NeoPixel Ring | Input | **D6** |
| NeoPixel Ring | Power | 5V rail |
| NeoPixel Ring | Ground | GND rail |

```
D6 ─► RING DIN
5V rail ─► RING PWR     GND rail ─► RING GND
```

## Code

```cpp
// G11 Activity 08 - Serial NeoPixel Ring Painter
#include <Adafruit_NeoPixel.h>

const int RING_PIN = 6;
const int NUM = 12;
Adafruit_NeoPixel ring(NUM, RING_PIN, NEO_GRB + NEO_KHZ800);

uint32_t canvas[NUM];
uint32_t brush = 0xFF0000;
int cursor = 0;
int offset = 0;
bool rotating = false;
int brightness = 60;
unsigned long lastRender = 0, lastRotate = 0;

const char COLOR_KEYS[] = "rgbywpo";
const uint32_t COLORS[] = {0xFF0000, 0x00FF00, 0x0000FF, 0xFFFF00, 0xFFFFFF, 0x8000FF, 0xFF6000};
const char* COLOR_NAMES[] = {"red", "green", "blue", "yellow", "white", "purple", "orange"};

uint32_t toPixel(uint32_t c) {
  return ring.Color((c >> 16) & 0xFF, (c >> 8) & 0xFF, c & 0xFF);
}

void printHex6(uint32_t c) {
  Serial.print('#');
  for (int shift = 20; shift >= 0; shift -= 4) Serial.print((c >> shift) & 0xF, HEX);
}

void render() {
  bool blinkOn = (millis() / 250) % 2;
  for (int i = 0; i < NUM; i++) {
    int pix = (i + offset) % NUM;
    uint32_t c = canvas[i];
    if (i == cursor && blinkOn && !rotating) c = (c == 0) ? 0x303030 : 0xFFFFFF;
    ring.setPixelColor(pix, toPixel(c));
  }
  ring.show();
}

void help() {
  Serial.println("--- RING PAINTER ---");
  Serial.println(" a d move | r g b y w p o brush color | x paint | e erase");
  Serial.println(" f fill | c clear | i invert | R rotate | + - brightness | v view | h help");
}

void view() {
  for (int i = 0; i < NUM; i++) {
    Serial.print(i < 10 ? " " : "");
    Serial.print(i);
    Serial.print(": ");
    printHex6(canvas[i]);
    Serial.println(i == cursor ? "  <- cursor" : "");
  }
}

void handle(char c) {
  for (int k = 0; COLOR_KEYS[k]; k++) {
    if (c == COLOR_KEYS[k]) {
      brush = COLORS[k];
      Serial.print("Brush: ");
      Serial.println(COLOR_NAMES[k]);
      return;
    }
  }
  if (c == 'a') cursor = (cursor + NUM - 1) % NUM;
  else if (c == 'd') cursor = (cursor + 1) % NUM;
  else if (c == 'x') canvas[cursor] = brush;
  else if (c == 'e') canvas[cursor] = 0;
  else if (c == 'f') { for (int i = 0; i < NUM; i++) canvas[i] = brush; Serial.println("Filled"); }
  else if (c == 'c') { for (int i = 0; i < NUM; i++) canvas[i] = 0; Serial.println("Cleared"); }
  else if (c == 'i') { for (int i = 0; i < NUM; i++) canvas[i] = (~canvas[i]) & 0xFFFFFF; Serial.println("Inverted"); }
  else if (c == 'R') { rotating = !rotating; Serial.println(rotating ? "Rotate ON" : "Rotate OFF"); }
  else if (c == '+' || c == '-') {
    brightness = constrain(brightness + (c == '+' ? 20 : -20), 10, 150);
    ring.setBrightness(brightness);
    Serial.print("Brightness ");
    Serial.println(brightness);
  }
  else if (c == 'v') view();
  else if (c == 'h' || c == '?') help();
  else if (c != '\n' && c != '\r' && c != ' ') {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
    return;
  }
  if (c == 'a' || c == 'd') {
    Serial.print("Cursor ");
    Serial.println(cursor);
  }
}

void setup() {
  Serial.begin(9600);
  ring.begin();
  ring.setBrightness(brightness);
  for (int i = 0; i < NUM; i++) canvas[i] = 0;
  // small demo picture
  canvas[0] = 0xFF0000; canvas[4] = 0x00FF00; canvas[8] = 0x0000FF;
  help();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();
  if (rotating && now - lastRotate >= 150) {
    lastRotate = now;
    offset = (offset + 1) % NUM;
  }
  if (now - lastRender >= 30) {
    lastRender = now;
    render();
  }
}
```

## Assessment Questions

1. Right after the simulation starts, which pixels are lit? Where is the cursor and how can you see it?
2. Send `dddx`. What happens on the ring? Now send `bdx`. What changes?
3. Send `f`, then `c`. What does the ring show after each command?
4. Send `R`. What happens to the picture? Does the picture change or only move?
5. If the ring's Input wire is moved from D6 to D5, what will happen when you send commands? Will the Serial Monitor still answer?
