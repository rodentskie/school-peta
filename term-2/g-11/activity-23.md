# Activity 23: Serial ASCII Oscilloscope (Two Channels)

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-23`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

An oscilloscope draws signals over time. Build a **text oscilloscope**: each line in the Serial Monitor is one sample, drawn as a bar of characters.

* **Channel 1 (A0):** potentiometer
* **Channel 2 (A1):** light sensor (LDR divider)

| Key | Action |
|---|---|
| `1` / `2` | show channel 1 / channel 2 |
| `b` | show **both** on one line (`*` for CH1, `o` for CH2, `#` where they overlap) |
| `+` / `-` | sample rate faster / slower (50–2000 ms) |
| `h` | **hold** (pause / resume drawing) |
| `t` | **trigger** ON/OFF: only start drawing when CH1 rises above the trigger level |
| `>` / `<` | trigger level up / down by 10 % |
| `n` | number mode: print raw values instead of bars |
| `?` | help |

1. Each bar line is 50 characters wide: `|` plus the bar, then the value in volts, e.g. `|*************************         2.51V`.
2. In trigger mode, after a **rising edge** (the previous sample was below the level and the current sample is at or above it), print 20 lines and then wait for the next edge.
3. The LED **lights while drawing** and is off while holding or waiting for a trigger.
4. Sampling must be non-blocking so keys work at any time.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Potentiometer |
| 1 | Photoresistor (LDR) + 1 kΩ resistor |
| 1 | LED + 1 kΩ resistor |

## Solution

* **Scaling:** `pos = value × 49 / 1023` gives a column from 0 to 49. Volts = `value × 5.0 / 1023`.
* **Drawing one line:** loop over the 50 columns. For CH1, print `*` while `col ≤ pos1`. In "both" mode, print the **marker** character only at the exact position of each channel, so two traces appear like on a real scope.
* **Trigger:** keep `prev1`. Detect the rising edge with `prev1 < level && v1 >= level`, then set `linesLeft = 20`.
* **Modes:** `view` (1, 2, 3 = both), `hold`, `trigger`, `numbers`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Potentiometer | T1 / Wiper / T2 | GND / **A0** / 5V |
| LDR | T1 / T2 | 5V / **A1** (+ 1 kΩ to GND) |
| LED | Anode | **D8** (cathode → 1 kΩ → GND) |

```
A0 ◄─ POT wiper (5V / GND ends)
5V ─[LDR]─┬─► A1
          └─[1k]─ GND
D8 ─► LED ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 23 - Serial ASCII Oscilloscope
const int CH1_PIN = A0;
const int CH2_PIN = A1;
const int LED_PIN = 8;
const int WIDTH = 50;

int view = 1;             // 1 = CH1, 2 = CH2, 3 = both
bool hold = false;
bool trigger = false;
bool numbers = false;
int trigLevel = 512;
unsigned int sampleMs = 200;
unsigned long lastSample = 0;
int prev1 = 0;
int linesLeft = 0;

void printVolts(int v) {
  Serial.print(v * 5.0 / 1023.0, 2);
  Serial.print('V');
}

void drawLine(int v1, int v2) {
  if (numbers) {
    Serial.print("CH1=");
    Serial.print(v1);
    Serial.print("  CH2=");
    Serial.println(v2);
    return;
  }
  int p1 = (long)v1 * (WIDTH - 1) / 1023;
  int p2 = (long)v2 * (WIDTH - 1) / 1023;
  Serial.print('|');
  for (int col = 0; col < WIDTH; col++) {
    char ch = ' ';
    if (view == 1 && col <= p1) ch = '*';
    else if (view == 2 && col <= p2) ch = 'o';
    else if (view == 3) {
      if (col == p1 && col == p2) ch = '#';
      else if (col == p1) ch = '*';
      else if (col == p2) ch = 'o';
      else if (trigger && col == (long)trigLevel * (WIDTH - 1) / 1023) ch = ':';
    }
    Serial.print(ch);
  }
  Serial.print(' ');
  if (view != 2) printVolts(v1);
  if (view == 3) Serial.print(" / ");
  if (view != 1) printVolts(v2);
  Serial.println();
}

void info() {
  Serial.print("[view ");
  Serial.print(view == 3 ? "both" : (view == 1 ? "CH1" : "CH2"));
  Serial.print(" | ");
  Serial.print(sampleMs);
  Serial.print(" ms | trigger ");
  if (trigger) { Serial.print("ON @ "); printVolts(trigLevel); }
  else Serial.print("OFF");
  Serial.println(hold ? " | HOLD]" : "]");
}

void handle(char c) {
  if (c == '1' || c == '2') view = c - '0';
  else if (c == 'b') view = 3;
  else if (c == '+') sampleMs = max(50, sampleMs / 2);
  else if (c == '-') sampleMs = min(2000, sampleMs * 2);
  else if (c == 'h') hold = !hold;
  else if (c == 't') { trigger = !trigger; linesLeft = 0; }
  else if (c == '>') trigLevel = min(1023, trigLevel + 102);
  else if (c == '<') trigLevel = max(0, trigLevel - 102);
  else if (c == 'n') numbers = !numbers;
  else if (c == '?') Serial.println("1 2 b view | + - rate | h hold | t trigger | > < level | n numbers");
  else return;
  info();
}

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("ASCII oscilloscope - turn the knob / change the light");
  info();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();
  if (now - lastSample < sampleMs) return;
  lastSample = now;

  int v1 = analogRead(CH1_PIN);
  int v2 = analogRead(CH2_PIN);
  bool draw = !hold;
  if (trigger && draw) {
    if (linesLeft == 0 && prev1 < trigLevel && v1 >= trigLevel) {
      linesLeft = 20;
      Serial.println("--- TRIGGER ---");
    }
    draw = linesLeft > 0;
    if (linesLeft > 0) linesLeft--;
  }
  prev1 = v1;
  digitalWrite(LED_PIN, draw);
  if (draw) drawLine(v1, v2);
}
```

## Assessment Questions

1. Turn the potentiometer slowly from one end to the other. What happens to the bars in the Serial Monitor?
2. Send `2` and change the light slider. Send `b`. How can you tell the two channels apart?
3. Send `h`. What happens to the drawing and the LED? Send `h` again.
4. Send `t`, then turn the knob from low to high. When does the drawing start, and when does it stop?
5. If the potentiometer's wiper wire to A0 is disconnected, what do you think channel 1 will show? What if the LDR's 1 kΩ resistor is removed?
