# Activity 07: Serial Hexadecimal Display and Counter (7-Segment)

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-07`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Show hexadecimal digits on a 7-segment display and control a hex counter from the Serial Monitor.

| Key | Action |
|---|---|
| `0`–`9`, `A`–`F` | show that hex digit (uppercase letters) |
| `u` | auto count **up** (0 → F → 0 …) |
| `d` | auto count **down** |
| `p` | pause / resume counting |
| `+` / `-` | counting faster / slower (100–2000 ms) |
| `b` | blink the current digit ON/OFF |
| `x` | blank the display |
| `t` | segment **test** (light a → g one by one, then all) |
| `s` | status |
| `h` | help |

1. Use a lookup table of **16** segment patterns (one byte each, bit 0 = a … bit 6 = g). `A b C d E F` follow the usual 7-segment forms.
2. Lowercase letters are commands. Uppercase `A`–`F` (and digits) are digits. A lowercase `a`–`f` that is not a command must print a hint: `Use UPPERCASE for hex digits`.
3. After each change, print the value in **hex, decimal and binary**, e.g. `D = 13 = 0b1101`.
4. The segment test runs **without blocking**: it is a small animation of 8 steps of 250 ms.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | 7-Segment Display (common cathode) + 1 kΩ resistor |
| 1 | Piezo buzzer (click on change) |

## Solution

* `HEX_SEG[16]` holds the patterns. `showValue(v)` prints `HEX_SEG[v]` unless the display is blanked or in the "off" half of a blink.
* **Parsing a hex character:**
  * `'0'..'9'` → `c − '0'`
  * `'A'..'F'` → `c − 'A' + 10`
* **Modes:** `countDir` (−1, 0, +1), `paused`, `blinking`, `blank`, and `testStep` (−1 = no test).
* Counting uses `(value + dir + 16) % 16` for wrap-around in both directions.
* Lowercase `b` and `d` are already used as commands (blink, down). To keep one simple rule, **all** hex letters must be uppercase. If the user sends `a`, `c`, `e` or `f`, the program explains this.

## Pin Diagram

| 7-Segment pin | Arduino |
|---|---|
| a, b, c, d, e, f, g | **D2, D3, D4, D5, D6, D7, D8** |
| Common | 1 kΩ → GND rail |

| Other | Pin | Connects to |
|---|---|---|
| Piezo | + / − | **D11** / GND |

```
D2..D8 ─► segments a..g      COMMON ─[1k]─ GND
D11 ─► PIEZO ── GND
```

## Code

```cpp
// G11 Activity 07 - Serial Hex Display and Counter
const int SEG_PINS[7] = {2, 3, 4, 5, 6, 7, 8};   // a..g
const int BUZZER_PIN = 11;

const byte HEX_SEG[16] = {
  0b0111111, 0b0000110, 0b1011011, 0b1001111,   // 0 1 2 3
  0b1100110, 0b1101101, 0b1111101, 0b0000111,   // 4 5 6 7
  0b1111111, 0b1101111, 0b1110111, 0b1111100,   // 8 9 A b
  0b0111001, 0b1011110, 0b1111001, 0b1110001    // C d E F
};

int value = 0;
int countDir = 0;           // -1 down, 0 none, +1 up
bool paused = false;
bool blinking = false;
bool blank = false;
int speedMs = 500;
int testStep = -1;
unsigned long lastCount = 0, lastTest = 0;

void writePattern(byte p) {
  for (int i = 0; i < 7; i++) digitalWrite(SEG_PINS[i], (p >> i) & 1);
}

void report() {
  Serial.print(value, HEX);
  Serial.print(" = ");
  Serial.print(value);
  Serial.print(" = 0b");
  for (int i = 3; i >= 0; i--) Serial.print((value >> i) & 1);
  Serial.println();
}

void setValue(int v) {
  value = v;
  blank = false;
  tone(BUZZER_PIN, 1200, 15);
  report();
}

void help() {
  Serial.println("--- HEX DISPLAY ---");
  Serial.println(" 0-9 A-F show digit (UPPERCASE letters)");
  Serial.println(" u up | d down | p pause | + - speed | b blink");
  Serial.println(" x blank | t segment test | s status | h help");
}

void status() {
  Serial.print("value ");
  report();
  Serial.print("count: ");
  Serial.print(countDir == 0 ? "off" : (countDir > 0 ? "up" : "down"));
  Serial.print(paused ? " (paused)" : "");
  Serial.print("  speed ");
  Serial.print(speedMs);
  Serial.print(" ms  blink ");
  Serial.println(blinking ? "on" : "off");
}

void handle(char c) {
  if (c >= '0' && c <= '9') { countDir = 0; setValue(c - '0'); }
  else if (c >= 'A' && c <= 'F') { countDir = 0; setValue(c - 'A' + 10); }
  else if (c == 'u') { countDir = 1; paused = false; Serial.println("Counting UP"); }
  else if (c == 'd') { countDir = -1; paused = false; Serial.println("Counting DOWN"); }
  else if (c == 'p') { paused = !paused; Serial.println(paused ? "Paused" : "Resumed"); }
  else if (c == '+') { speedMs = max(100, speedMs / 2); status(); }
  else if (c == '-') { speedMs = min(2000, speedMs * 2); status(); }
  else if (c == 'b') { blinking = !blinking; Serial.println(blinking ? "Blink ON" : "Blink OFF"); }
  else if (c == 'x') { blank = true; countDir = 0; Serial.println("Display blank"); }
  else if (c == 't') { testStep = 0; lastTest = millis(); Serial.println("Segment test..."); }
  else if (c == 's') status();
  else if (c == 'h' || c == '?') help();
  else if (c == 'a' || c == 'c' || c == 'e' || c == 'f') Serial.println("Use UPPERCASE for hex digits");
  else if (c != '\n' && c != '\r' && c != ' ') {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
  }
}

void setup() {
  for (int i = 0; i < 7; i++) pinMode(SEG_PINS[i], OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  help();
  report();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();

  // segment test animation has priority
  if (testStep >= 0) {
    if (now - lastTest >= 250) {
      lastTest = now;
      testStep++;
      if (testStep > 8) {
        testStep = -1;
        Serial.println("Test done");
      }
    }
    if (testStep >= 0) {
      writePattern(testStep < 7 ? (1 << testStep) : 0x7F);
      return;
    }
  }

  if (countDir != 0 && !paused && now - lastCount >= (unsigned long)speedMs) {
    lastCount = now;
    setValue((value + countDir + 16) % 16);
  }

  bool off = blank || (blinking && (now / 300) % 2);
  writePattern(off ? 0 : HEX_SEG[value]);
}
```

## Assessment Questions

1. Send `7`, then `A`, then `F`. What does the display show for each one?
2. Send `a` (lowercase). What happens? Why?
3. Send `u`. What does the display do? What happens when it reaches `F`? What changes when you send `d`?
4. Send `t`. Describe what the display does. Which segment lights first?
5. What will happen to the display if the resistor on the common pin is removed? If the wire to segment a (D2) is disconnected, which digits will look wrong?
