# Activity 10: Serial-Controlled Sensor Dashboard (Temperature, Light, Distance)

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-10`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Build a measuring station that only reports when it is **asked**, like a laboratory instrument.

| Key | Action |
|---|---|
| `t` | read temperature (°C) |
| `l` | read light level (0–1023 and %) |
| `u` | read ultrasonic distance (cm) |
| `a` | read **all** sensors once |
| `r` | auto-report ON/OFF |
| `1`–`5` | auto-report interval 1–5 s |
| `c` | output format: **text** ↔ **CSV** |
| `m` | print min/max of every sensor since the last reset |
| `z` | reset min/max |
| `>` / `<` | temperature alarm threshold +1 / −1 °C |
| `h` | help |

1. The LED turns on when the temperature is **above the threshold** (default 30 °C).
2. In **CSV** mode, each report is one line `time_s,temp_c,light,dist_cm`, with a header line printed when CSV mode is switched on. Text mode is human-friendly, e.g. `T=24.5C  L=612 (60%)  D=123.4cm`.
3. If the ultrasonic sensor gets no echo, show `---` (text) or an empty field (CSV), and do not update its min/max.
4. Min/max are updated for every reading (manual and automatic).
5. Auto-report must be **non-blocking** so commands still work while reporting.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Temperature Sensor [TMP36] |
| 1 | Photoresistor (LDR) + 1 kΩ resistor |
| 1 | Ultrasonic Distance Sensor (4-pin, HC-SR04) |
| 1 | LED + 1 kΩ resistor |

## Solution

* **Sensor functions:** `readTemp()`, `readLight()` and `readDist()` each return one measurement. `readDist()` returns −1 for no echo.
* **Min/max structure:** `struct Range { float lo, hi; bool valid; }` with the member functions `add(v)` and `print(name, digits)`. One structure is used for all three sensors, so no code is repeated.
* **One report function** `report(bool t, bool l, bool d)` reads the requested sensors, updates min/max, and prints in the selected format. Single-sensor commands call it with only one flag set. In CSV mode, sensors that were not requested are left empty.
* **Auto-report** with `millis()` and `intervalMs`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| TMP36 | Vs / Vout / GND | 5V / **A0** / GND |
| LDR | T1 / T2 | 5V / **A1** (+ 1 kΩ to GND) |
| HC-SR04 | Power / Ground | 5V rail / GND rail |
| HC-SR04 | Trigger / Echo | **D7** / **D6** |
| Alarm LED | Anode | **D8** (cathode → 1 kΩ → GND) |

```
TMP36: Vs─5V  Vout─A0  GND─GND
5V ─[LDR]─┬─► A1
          └─[1k]─ GND
HC-SR04: VCC─5V TRIG─D7 ECHO─D6 GND─GND
D8 ─► LED ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 10 - Serial-Controlled Sensor Dashboard
const int TMP_PIN = A0;
const int LDR_PIN = A1;
const int TRIG_PIN = 7;
const int ECHO_PIN = 6;
const int LED_PIN = 8;

// Tinkercad-compatible: helpers are member functions, so the auto-prototype
// step never references Range before it is declared.
struct Range {
  float lo;
  float hi;
  bool valid;

  void add(float v) {
    if (!valid) { lo = hi = v; valid = true; return; }
    if (v < lo) lo = v;
    if (v > hi) hi = v;
  }

  void print(const char *name, int digits) {
    Serial.print(name);
    if (!valid) { Serial.println(": no data"); return; }
    Serial.print(": min ");
    Serial.print(lo, digits);
    Serial.print("  max ");
    Serial.println(hi, digits);
  }
};
Range rT = {0, 0, false}, rL = {0, 0, false}, rD = {0, 0, false};

bool autoReport = false;
bool csv = false;
unsigned long intervalMs = 2000;
unsigned long lastAuto = 0;
float threshold = 30.0;
float lastTemp = 0;

float readTemp() {
  long s = 0;
  for (int i = 0; i < 5; i++) s += analogRead(TMP_PIN);
  float mv = (s / 5.0) * 5000.0 / 1024.0;
  return (mv - 500.0) / 10.0;
}

int readLight() {
  return analogRead(LDR_PIN);
}

float readDist() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  unsigned long us = pulseIn(ECHO_PIN, HIGH, 30000);
  if (us == 0) return -1;
  return us * 0.0343 / 2.0;
}

void report(bool wantT, bool wantL, bool wantD) {
  float t = 0, d = -1;
  int l = 0;
  if (wantT) { t = readTemp(); lastTemp = t; rT.add(t); }
  if (wantL) { l = readLight(); rL.add(l); }
  if (wantD) { d = readDist(); if (d >= 0) rD.add(d); }

  if (csv) {
    Serial.print(millis() / 1000.0, 1);
    Serial.print(',');
    if (wantT) Serial.print(t, 2);
    Serial.print(',');
    if (wantL) Serial.print(l);
    Serial.print(',');
    if (wantD && d >= 0) Serial.print(d, 1);
    Serial.println();
  } else {
    if (wantT) { Serial.print("T="); Serial.print(t, 1); Serial.print("C  "); }
    if (wantL) {
      Serial.print("L="); Serial.print(l);
      Serial.print(" ("); Serial.print(map(l, 0, 1023, 0, 100)); Serial.print("%)  ");
    }
    if (wantD) {
      Serial.print("D=");
      if (d < 0) Serial.print("---");
      else { Serial.print(d, 1); Serial.print("cm"); }
    }
    Serial.println();
  }
}

void help() {
  Serial.println("--- SENSOR DASHBOARD ---");
  Serial.println(" t temp | l light | u distance | a all");
  Serial.println(" r auto on/off | 1-5 interval s | c text/CSV");
  Serial.println(" m min/max | z reset | > < threshold | h help");
}

void handle(char c) {
  if (c == 't') report(true, false, false);
  else if (c == 'l') report(false, true, false);
  else if (c == 'u') report(false, false, true);
  else if (c == 'a') report(true, true, true);
  else if (c == 'r') {
    autoReport = !autoReport;
    lastAuto = millis();
    Serial.println(autoReport ? "Auto-report ON" : "Auto-report OFF");
  }
  else if (c >= '1' && c <= '5') {
    intervalMs = (c - '0') * 1000UL;
    Serial.print("Interval ");
    Serial.print(c - '0');
    Serial.println(" s");
  }
  else if (c == 'c') {
    csv = !csv;
    if (csv) Serial.println("time_s,temp_c,light,dist_cm");
    else Serial.println("Text mode");
  }
  else if (c == 'm') {
    rT.print("Temp C ", 1);
    rL.print("Light  ", 0);
    rD.print("Dist cm", 1);
  }
  else if (c == 'z') {
    rT.valid = rL.valid = rD.valid = false;
    Serial.println("Min/max reset");
  }
  else if (c == '>' || c == '<') {
    threshold += (c == '>') ? 1 : -1;
    Serial.print("Alarm threshold ");
    Serial.print(threshold, 0);
    Serial.println(" C");
  }
  else if (c == 'h' || c == '?') help();
  else if (c != '\n' && c != '\r' && c != ' ') {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
  }
}

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  help();
  report(true, true, true);
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();
  if (autoReport && now - lastAuto >= intervalMs) {
    lastAuto = now;
    report(true, true, true);
  }
  digitalWrite(LED_PIN, lastTemp > threshold ? HIGH : LOW);
}
```

## Assessment Questions

1. Send `t`, `l` and `u` one at a time. What does each one show?
2. Send `r`. What happens in the Serial Monitor? Send `5` and then `1`. What changes?
3. Send `c`, then `a`. How does the output look different from text mode?
4. Raise the temperature slider above 30 °C and send `t`. What does the LED do? What happens to the LED if you send `>` several times?
5. If the 1 kΩ resistor of the light sensor is removed, what light values would you expect? If the Echo wire is disconnected, what does `u` show?
