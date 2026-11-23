# Activity 05: Serial Traffic-Light Operator Console with Safety Interlocks

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-05`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

A traffic engineer controls a junction light remotely through a serial console.

| Key | Mode / action |
|---|---|
| `n` | **NORMAL**: automatic cycle green 5 s → yellow 2 s → red 5 s |
| `f` | **FLASH**: yellow blinks (night mode / fault) |
| `m` | **MANUAL**: operator controls the lights with `r`, `y`, `g` |
| `r` `y` `g` | (manual only) request red / yellow / green |
| `e` | **EMERGENCY**: all red + warning beeps for 10 s, then back to the previous mode |
| `s` | status (mode, current light, time in state) |
| `h` | help |

**Safety rules (interlocks) that the program must enforce, even in MANUAL mode:**

1. Green can **never** change directly to red. The program automatically inserts **yellow for 2 s** first.
2. Red to green is only allowed after red has been on for at least **3 s**. Otherwise, print `DENIED: red minimum 3 s (x.x s left)`.
3. `r`, `y`, `g` outside MANUAL mode print `Switch to manual first (m)`.
4. Every light change is logged with a timestamp.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 3 | LEDs (red, yellow, green) + 3 × 1 kΩ resistors |
| 1 | Piezo buzzer |

## Solution

* Two levels of state:
  * **mode**: NORMAL, FLASH, MANUAL, EMERGENCY;
  * **light**: RED, YELLOW, GREEN, OFF (`setLight()` is the only function that changes the LEDs and logs it).
* **Pending request:** a manual request `g → r` sets `pendingRed = true` and switches to yellow. After 2 s, the yellow is automatically followed by red.
* **Emergency** saves `prevMode`. When 10 s have passed, it restores that mode.
* In NORMAL mode, the time in the current light (`now − lightSince`) decides the next light.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Red LED | Anode | **D10** |
| Yellow LED | Anode | **D9** |
| Green LED | Anode | **D8** |
| LED cathodes | — | 1 kΩ → GND rail |
| Piezo | + / − | **D6** / GND |

```
D10 ─► RED    ─[1k]─ GND
D9  ─► YELLOW ─[1k]─ GND      D6 ─► PIEZO ── GND
D8  ─► GREEN  ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 05 - Serial Traffic-Light Console with Interlocks
const int RED_PIN = 10;
const int YELLOW_PIN = 9;
const int GREEN_PIN = 8;
const int BUZZER_PIN = 6;

const int NORMAL = 0, FLASH = 1, MANUAL = 2, EMERGENCY = 3;
const char* MODE_NAMES[] = {"NORMAL", "FLASH", "MANUAL", "EMERGENCY"};
const int L_OFF = 0, L_RED = 1, L_YELLOW = 2, L_GREEN = 3;
const char* LIGHT_NAMES[] = {"OFF", "RED", "YELLOW", "GREEN"};

int mode = NORMAL, prevMode = NORMAL;
int light = L_OFF;
unsigned long lightSince = 0, modeSince = 0;
bool pendingRed = false;

void stamp() {
  Serial.print("[");
  Serial.print(millis() / 1000.0, 1);
  Serial.print(" s] ");
}

void setLight(int l) {
  light = l;
  lightSince = millis();
  digitalWrite(RED_PIN, l == L_RED);
  digitalWrite(YELLOW_PIN, l == L_YELLOW);
  digitalWrite(GREEN_PIN, l == L_GREEN);
  if (mode != FLASH) {
    stamp();
    Serial.print("Light -> ");
    Serial.println(LIGHT_NAMES[l]);
  }
}

void setMode(int m) {
  if (m == EMERGENCY && mode != EMERGENCY) prevMode = mode;
  mode = m;
  modeSince = millis();
  pendingRed = false;
  noTone(BUZZER_PIN);
  stamp();
  Serial.print("MODE: ");
  Serial.println(MODE_NAMES[m]);
  if (m == NORMAL) setLight(L_GREEN);
  else if (m == EMERGENCY) setLight(L_RED);
  else if (m == FLASH) setLight(L_OFF);
}

// safe change used by MANUAL mode
void requestLight(int want) {
  unsigned long inState = millis() - lightSince;
  if (want == light) {
    Serial.println("Already in that state");
  } else if (want == L_RED && light == L_GREEN) {
    Serial.println("Interlock: GREEN -> YELLOW (2 s) -> RED");
    pendingRed = true;
    setLight(L_YELLOW);
  } else if (want == L_GREEN && light == L_RED && inState < 3000) {
    Serial.print("DENIED: red minimum 3 s (");
    Serial.print((3000 - inState) / 1000.0, 1);
    Serial.println(" s left)");
  } else {
    setLight(want);
  }
}

void status() {
  Serial.print("Mode=");
  Serial.print(MODE_NAMES[mode]);
  Serial.print("  Light=");
  Serial.print(LIGHT_NAMES[light]);
  Serial.print("  for ");
  Serial.print((millis() - lightSince) / 1000.0, 1);
  Serial.println(" s");
}

void help() {
  Serial.println("--- TRAFFIC CONSOLE ---");
  Serial.println(" n normal | f flash | m manual (r y g) | e emergency | s status | h help");
}

void handle(char c) {
  if (c == 'n') setMode(NORMAL);
  else if (c == 'f') setMode(FLASH);
  else if (c == 'm') setMode(MANUAL);
  else if (c == 'e') setMode(EMERGENCY);
  else if (c == 'r' || c == 'y' || c == 'g') {
    if (mode != MANUAL) Serial.println("Switch to manual first (m)");
    else requestLight(c == 'r' ? L_RED : (c == 'y' ? L_YELLOW : L_GREEN));
  }
  else if (c == 's') status();
  else if (c == 'h' || c == '?') help();
  else if (c != '\n' && c != '\r' && c != ' ') {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
  }
}

void setup() {
  pinMode(RED_PIN, OUTPUT);
  pinMode(YELLOW_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  help();
  setMode(NORMAL);
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();
  unsigned long t = now - lightSince;

  if (mode == NORMAL) {
    if (light == L_GREEN && t >= 5000) setLight(L_YELLOW);
    else if (light == L_YELLOW && t >= 2000) setLight(L_RED);
    else if (light == L_RED && t >= 5000) setLight(L_GREEN);
  }
  else if (mode == FLASH) {
    digitalWrite(YELLOW_PIN, ((now - modeSince) % 1000) < 500);
  }
  else if (mode == MANUAL) {
    if (pendingRed && light == L_YELLOW && t >= 2000) {
      pendingRed = false;
      setLight(L_RED);
    }
  }
  else if (mode == EMERGENCY) {
    unsigned long e = now - modeSince;
    if (e % 1000 < 100) tone(BUZZER_PIN, 1500);
    else noTone(BUZZER_PIN);
    if (e >= 10000) {
      Serial.println("Emergency over");
      setMode(prevMode == EMERGENCY ? NORMAL : prevMode);
    }
  }
}
```

## Assessment Questions

1. In MANUAL mode with the light GREEN, the operator sends `r` and then, 1 s later, `g`. Describe exactly what happens and what is printed.
2. Why are safety interlocks enforced **in the program** and not left to the operator? Give a real-world example.
3. Explain the purpose of `prevMode`. What would happen if the program always returned to NORMAL after an emergency?
4. The function `setLight()` is the only place where LEDs are changed. Why is this a good design?
5. Add a new interlock: in MANUAL mode, the light may not stay **green longer than 60 s**. After that, the program automatically changes to red (through yellow) and prints a warning. Write the code.
