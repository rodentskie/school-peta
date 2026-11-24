# Activity 22: Serial-Controlled Thermostat (Heat / Cool / Auto)

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-22`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Build a room thermostat. The **heater** is shown by a red LED and the **air-conditioner (cooler)** by a blue LED. The temperature comes from a TMP36.

| Key | Action |
|---|---|
| `+` / `-` | set-point +0.5 / −0.5 °C (range 16–30 °C) |
| `h` | mode HEAT |
| `c` | mode COOL |
| `a` | mode AUTO (heat or cool as needed) |
| `o` | mode OFF |
| `e` | ECO ON/OFF: widens the comfort band from ±0.5 °C to ±2 °C |
| `s` | status |

**Control rules (with hysteresis):**

* **HEAT:** heater ON when `T < set − band`, OFF when `T > set + band`.
* **COOL:** cooler ON when `T > set + band`, OFF when `T < set − band`.
* **AUTO:** uses both rules, but heater and cooler can **never** be on at the same time.
* **Minimum run / rest time 5 s:** once a device turns ON or OFF, it may not change again for 5 s. This protects real compressors.

1. Print a status line every 2 s: `T=22.8C set=23.0C mode=AUTO heat=ON cool=off`.
2. The buzzer gives one beep when a device switches.
3. Temperatures above 40 °C or below 5 °C are a **sensor fault**: switch everything OFF and blink both LEDs.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Temperature Sensor [TMP36] |
| 2 | LEDs (red = heater, blue = cooler) + 2 × 1 kΩ resistors |
| 1 | Piezo buzzer |

## Solution

* **Smoothed temperature:** EMA with α = 0.1 every 100 ms.
* **Device state:** `heatOn`, `coolOn`, `heatChanged`, `coolChanged` (time of last switch).
* `switchDevice(isHeater, newState)` only allows the change if `now − lastChanged ≥ 5000`. It also beeps and logs.
* **Decision:** first calculate the "wanted" states using the hysteresis rules. Then turn devices **off** first and **on** second, so both are never on together (break-before-make).
* **Fault detection** has priority over everything else.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| TMP36 | Vs / Vout / GND | 5V / **A0** / GND |
| Heater LED (red) | Anode | **D5** |
| Cooler LED (blue) | Anode | **D6** |
| LED cathodes | — | 1 kΩ → GND rail |
| Piezo | + / − | **D9** / GND |

```
TMP36: Vs─5V  Vout─A0  GND─GND
D5 ─► HEAT (red) ─[1k]─ GND
D6 ─► COOL (blue)─[1k]─ GND     D9 ─► PIEZO ── GND
```

## Code

```cpp
// G11 Activity 22 - Serial Thermostat
const int TMP_PIN = A0;
const int HEAT_PIN = 5;
const int COOL_PIN = 6;
const int BUZZER_PIN = 9;
const unsigned long MIN_TIME = 5000;

const int M_OFF = 0, M_HEAT = 1, M_COOL = 2, M_AUTO = 3;
const char* MODES[4] = {"OFF", "HEAT", "COOL", "AUTO"};

int mode = M_AUTO;
float setPoint = 23.0;
bool eco = false;
float temp = 0;
bool heatOn = false, coolOn = false;
unsigned long heatChanged = 0, coolChanged = 0;
unsigned long lastSample = 0, lastPrint = 0;
bool fault = false;

float readC() {
  float mv = analogRead(TMP_PIN) * 5000.0 / 1024.0;
  return (mv - 500.0) / 10.0;
}

void switchDevice(bool heater, bool want) {
  unsigned long now = millis();
  if (heater) {
    if (want == heatOn || now - heatChanged < MIN_TIME) return;
    heatOn = want;
    heatChanged = now;
    Serial.println(want ? ">> HEATER ON" : ">> heater off");
  } else {
    if (want == coolOn || now - coolChanged < MIN_TIME) return;
    coolOn = want;
    coolChanged = now;
    Serial.println(want ? ">> COOLER ON" : ">> cooler off");
  }
  tone(BUZZER_PIN, want ? 1500 : 900, 80);
}

void status() {
  Serial.print("T=");
  Serial.print(temp, 1);
  Serial.print("C set=");
  Serial.print(setPoint, 1);
  Serial.print("C mode=");
  Serial.print(MODES[mode]);
  Serial.print(eco ? " ECO" : "");
  Serial.print(" heat=");
  Serial.print(heatOn ? "ON" : "off");
  Serial.print(" cool=");
  Serial.print(coolOn ? "ON" : "off");
  Serial.println(fault ? "  SENSOR FAULT!" : "");
}

void handle(char c) {
  if (c == '+') setPoint = min(30.0, setPoint + 0.5);
  else if (c == '-') setPoint = max(16.0, setPoint - 0.5);
  else if (c == 'h') mode = M_HEAT;
  else if (c == 'c') mode = M_COOL;
  else if (c == 'a') mode = M_AUTO;
  else if (c == 'o') mode = M_OFF;
  else if (c == 'e') eco = !eco;
  else if (c == 's') { }
  else {
    if (c != '\n' && c != '\r' && c != ' ') Serial.println("+ - set | h heat | c cool | a auto | o off | e eco | s status");
    return;
  }
  status();
}

void setup() {
  pinMode(HEAT_PIN, OUTPUT);
  pinMode(COOL_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  temp = readC();
  heatChanged = coolChanged = millis() - MIN_TIME;
  Serial.println("Thermostat ready");
  status();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();

  if (now - lastSample >= 100) {
    lastSample = now;
    temp += 0.1 * (readC() - temp);
  }

  fault = (temp > 40.0 || temp < 5.0);
  if (fault) {
    heatOn = coolOn = false;
    bool b = (now / 300) % 2;
    digitalWrite(HEAT_PIN, b);
    digitalWrite(COOL_PIN, b);
  } else {
    float band = eco ? 2.0 : 0.5;
    bool wantHeat = heatOn, wantCool = coolOn;
    if (mode == M_HEAT || mode == M_AUTO) {
      if (temp < setPoint - band) wantHeat = true;
      if (temp > setPoint + band) wantHeat = false;
    } else wantHeat = false;
    if (mode == M_COOL || mode == M_AUTO) {
      if (temp > setPoint + band) wantCool = true;
      if (temp < setPoint - band) wantCool = false;
    } else wantCool = false;
    if (wantHeat && wantCool) wantHeat = wantCool = false;   // never both

    if (!wantHeat) switchDevice(true, false);                // break first
    if (!wantCool) switchDevice(false, false);
    if (wantHeat && !coolOn) switchDevice(true, true);       // then make
    if (wantCool && !heatOn) switchDevice(false, true);

    digitalWrite(HEAT_PIN, heatOn);
    digitalWrite(COOL_PIN, coolOn);
  }

  if (now - lastPrint >= 2000) {
    lastPrint = now;
    status();
  }
}
```

## Assessment Questions

1. Set the temperature slider below 22 °C. Which LED turns on? What happens when you raise the temperature above 24 °C?
2. Send `+` several times while the heater is off. When does the heater turn on? Does it switch on immediately after it was just turned off?
3. Send `e`. How does the thermostat react differently when the temperature moves a little around the set-point?
4. Send `h` (HEAT mode) and make the room very warm. Does the blue LED turn on? Why?
5. Set the temperature slider to above 40 °C. What do both LEDs do? What do you think would happen if the TMP36's GND wire were disconnected?
