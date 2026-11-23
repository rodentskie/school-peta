# Activity 30: Automatic Temperature-Controlled Fan with Manual Override

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-30`

## Problem Statement

Build a smart cooling fan (DC motor) that combines several ideas from this term.

1. **AUTO mode:** fan speed follows a **temperature curve**:

| Temperature | Fan |
|---|---|
| < 25 °C | OFF (turns ON at 25 °C, OFF only below 24 °C: hysteresis) |
| 25 °C – 35 °C | linear 40 % → 100 % |
| ≥ 35 °C | 100 % |
| ≥ 40 °C | 100 % **and** the warning LED flashes (overheat) |

2. **Kick-start:** when the fan starts from 0, run it at 100 % for 300 ms first, so it overcomes friction, then go to the target speed.
3. **MANUAL mode:** the potentiometer on A1 sets the speed (0–100 %). The temperature curve is ignored, **except** the overheat warning.
4. The **MODE button** switches AUTO ↔ MANUAL. The status LED is ON in AUTO mode and OFF in MANUAL mode, unless overheat flashing is active.
5. Print mode, temperature, fan % and PWM every second.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Temperature Sensor [TMP36] |
| 1 | DC Motor + NPN Transistor (BJT) + Diode + 1 kΩ base resistor |
| 1 | Potentiometer |
| 1 | Pushbutton |
| 1 | LED + 1 kΩ resistor |

## Solution

* **Speed curve:** `percent = 40 + (T − 25) × (100 − 40) / (35 − 25)`, clamped to 40–100. This is a straight line through (25 °C, 40 %) and (35 °C, 100 %).
* **Hysteresis:** the variable `fanOn` changes to true at ≥ 25 °C and to false only below 24 °C. This stops the fan switching on and off quickly near 25 °C.
* **Kick-start:** when the speed goes from 0 to more than 0, save `kickUntil = now + 300`. While `now < kickUntil`, output PWM 255.
* **Temperature smoothing:** exponential moving average with α = 0.2, sampled every 100 ms.
* **Motor driver:** the same NPN + flyback diode circuit as Activity 24 (PWM on D9).

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| TMP36 | Vs / Vout / GND | 5V / **A0** / GND |
| NPN transistor | Base | 1 kΩ → **D9** |
| NPN transistor | Collector | Motor terminal 2 + diode anode |
| NPN transistor | Emitter | GND |
| DC Motor | Terminal 1 | 5V |
| Diode | Cathode | 5V |
| Status LED | Anode | **D8** (cathode → 1 kΩ → GND) |
| MODE button | T1 / T2 | GND / **D2** |
| Potentiometer | T1 / Wiper / T2 | GND / **A1** / 5V |

```
TMP36 Vout ─► A0           POT wiper ─► A1
D9 ─[1k]─► NPN base        NPN collector ─ MOTOR ─ 5V  (diode across motor)
NPN emitter ─ GND          D2 ─► MODE ── GND    D8 ─► LED ─[1k]─ GND
```

## Code

```cpp
// Activity 30 - Automatic Temperature-Controlled Fan
const int TMP_PIN = A0;
const int POT_PIN = A1;
const int MOTOR_PIN = 9;
const int LED_PIN = 8;
const int BTN_PIN = 2;

const float T_ON = 25.0;
const float T_OFF = 24.0;
const float T_FULL = 35.0;
const float T_OVERHEAT = 40.0;
const int MIN_PERCENT = 40;

bool autoMode = true;
bool fanOn = false;
float temp = 0;
int percent = 0;
int lastPwm = 0;
unsigned long kickUntil = 0;
unsigned long lastSample = 0, lastPrint = 0;

int btnPrev = HIGH;
unsigned long btnEdge = 0;

float readC() {
  float mv = analogRead(TMP_PIN) * 5000.0 / 1024.0;
  return (mv - 500.0) / 10.0;
}

int autoPercent(float t) {
  if (!fanOn && t >= T_ON) fanOn = true;
  if (fanOn && t < T_OFF) fanOn = false;
  if (!fanOn) return 0;
  float p = MIN_PERCENT + (t - T_ON) * (100 - MIN_PERCENT) / (T_FULL - T_ON);
  return constrain((int)p, MIN_PERCENT, 100);
}

void setup() {
  pinMode(MOTOR_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  temp = readC();
  Serial.println("Smart fan - button toggles AUTO / MANUAL");
}

void loop() {
  unsigned long now = millis();

  // mode button
  int r = digitalRead(BTN_PIN);
  if (r != btnPrev && now - btnEdge > 40) {
    btnEdge = now;
    btnPrev = r;
    if (r == LOW) {
      autoMode = !autoMode;
      Serial.println(autoMode ? "MODE: AUTO" : "MODE: MANUAL");
    }
  }

  // smoothed temperature
  if (now - lastSample >= 100) {
    lastSample = now;
    temp += 0.2 * (readC() - temp);
  }

  if (autoMode) percent = autoPercent(temp);
  else percent = map(analogRead(POT_PIN), 0, 1023, 0, 100);

  int pwm = map(percent, 0, 100, 0, 255);
  if (lastPwm == 0 && pwm > 0) {
    kickUntil = now + 300;                 // kick-start
    Serial.println("Kick-start!");
  }
  lastPwm = pwm;
  if ((long)(kickUntil - now) > 0 && pwm > 0) analogWrite(MOTOR_PIN, 255);
  else analogWrite(MOTOR_PIN, pwm);

  bool overheat = temp >= T_OVERHEAT;
  if (overheat) digitalWrite(LED_PIN, (now / 150) % 2);
  else digitalWrite(LED_PIN, autoMode ? HIGH : LOW);

  if (now - lastPrint >= 1000) {
    lastPrint = now;
    Serial.print(autoMode ? "AUTO  " : "MANUAL");
    Serial.print("  T=");
    Serial.print(temp, 1);
    Serial.print("C  fan=");
    Serial.print(percent);
    Serial.print("%  pwm=");
    Serial.print(pwm);
    if (overheat) Serial.print("  OVERHEAT!");
    Serial.println();
  }
}
```

## Assessment Questions

1. Using the speed curve, calculate the fan percentage and the PWM value at 25 °C, 28 °C, 31.5 °C and 37 °C.
2. Draw a graph of fan % vs. temperature from 20 °C to 40 °C, **rising** and then **falling**. Mark where the hysteresis appears.
3. Why does a DC motor need a kick-start at low PWM values? What would happen at 40 % without it?
4. In MANUAL mode the overheat warning still works. Why is it important that safety features cannot be switched off by the user?
5. Replace the straight-line curve with a **lookup table** of 5 points (25→40 %, 28→55 %, 31→70 %, 34→90 %, 35→100 %) and **linear interpolation** between them. Write the function `int curvePercent(float t)`.
