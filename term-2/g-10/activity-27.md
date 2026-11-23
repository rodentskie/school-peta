# Activity 27: Light-Seeking Servo Tracker (Proportional Control)

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-27`

## Problem Statement

Solar panels earn more energy when they face the sun. Build a **sun tracker**: a servo turns a platform with two light sensors (left and right) until both see the same light.

1. Read the two LDR dividers on **A0 (left)** and **A1 (right)**, averaging 4 samples each.
2. **Calibrate** at start-up: measure the difference between the sensors for 1 s under equal light and store it as `offset`.
3. Error = `(left − right) − offset`.
4. Use **proportional control**: `angle += Kp × error`, with Kp = 0.02 and the step limited to ±3° per cycle. Keep the angle between 0° and 180°.
5. If |error| < **20** (dead-band), do not move. Light the **LOCK LED** to show the tracker is on target.
6. Every 20 ms, update the servo. Print `left, right, error, angle` every 250 ms in a format that works with the Serial Plotter.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 2 | Photoresistors (LDR) + 2 × 1 kΩ resistors |
| 1 | Micro Servo |
| 1 | LED + 1 kΩ resistor |

## Solution

**Control loop:**

```
 measure L, R ──► error = (L − R) − offset ──► step = clamp(Kp·error, ±3°)
        ▲                                               │
        └──────────── servo turns platform ◄── angle += step
```

* **Proportional control:** a big error causes a big correction and a small error a small correction. The servo moves fast when far from the target and slows down as it gets closer.
* **Dead-band:** prevents the servo from "hunting" (moving back and forth) because of tiny noise in the readings.
* **Step limit:** protects against sudden jumps when a shadow passes over one sensor.
* **Calibration:** two LDRs are never exactly the same. The offset removes the difference between them.
* The **sign** of the correction depends on how the sensors are mounted. If the tracker turns away from the light, change `DIRECTION` to −1.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Left LDR | T1 / T2 | 5V / **A0** (+ 1 kΩ to GND) |
| Right LDR | T1 / T2 | 5V / **A1** (+ 1 kΩ to GND) |
| Micro Servo | Signal | **D9** |
| Micro Servo | Power / Ground | 5V rail / GND rail |
| LOCK LED | Anode | **D8** (cathode → 1 kΩ → GND) |

```
5V ─[LDR L]─┬─► A0        5V ─[LDR R]─┬─► A1
            └─[1k]─ GND               └─[1k]─ GND
D9 ─► SERVO signal        D8 ─► LOCK LED ─[1k]─ GND
```

## Code

```cpp
// Activity 27 - Light-Seeking Servo Tracker
#include <Servo.h>

const int LEFT_PIN = A0;
const int RIGHT_PIN = A1;
const int SERVO_PIN = 9;
const int LOCK_PIN = 8;

const float KP = 0.02;
const float MAX_STEP = 3.0;
const int DEADBAND = 20;
const int DIRECTION = 1;          // change to -1 if it turns away from the light

Servo servo;
float angle = 90;
int offset = 0;
unsigned long lastCtrl = 0, lastPrint = 0;

int readAvg(int pin) {
  long s = 0;
  for (int i = 0; i < 4; i++) s += analogRead(pin);
  return s / 4;
}

void calibrate() {
  Serial.println("Calibrating (keep light equal)...");
  long sum = 0;
  int n = 0;
  unsigned long t0 = millis();
  while (millis() - t0 < 1000) {
    sum += readAvg(LEFT_PIN) - readAvg(RIGHT_PIN);
    n++;
    delay(10);
  }
  offset = sum / n;
  Serial.print("Offset = ");
  Serial.println(offset);
}

void setup() {
  pinMode(LOCK_PIN, OUTPUT);
  Serial.begin(9600);
  servo.attach(SERVO_PIN);
  servo.write((int)angle);
  calibrate();
}

void loop() {
  unsigned long now = millis();
  if (now - lastCtrl < 20) return;
  lastCtrl = now;

  int left = readAvg(LEFT_PIN);
  int right = readAvg(RIGHT_PIN);
  int error = (left - right) - offset;

  bool locked = abs(error) < DEADBAND;
  if (!locked) {
    float step = KP * error * DIRECTION;
    if (step > MAX_STEP) step = MAX_STEP;
    if (step < -MAX_STEP) step = -MAX_STEP;
    angle = constrain(angle + step, 0, 180);
    servo.write((int)(angle + 0.5));
  }
  digitalWrite(LOCK_PIN, locked ? HIGH : LOW);

  if (now - lastPrint >= 250) {
    lastPrint = now;
    Serial.print("left:");
    Serial.print(left);
    Serial.print(" right:");
    Serial.print(right);
    Serial.print(" error:");
    Serial.print(error);
    Serial.print(" angle:");
    Serial.println(angle, 1);
  }
}
```

## Assessment Questions

1. With Kp = 0.02, calculate the step for errors of 50, 120, 300 and −600. Which ones are limited by `MAX_STEP`?
2. What happens to the tracker's behaviour if Kp is (a) much smaller, e.g. 0.001, and (b) much larger, e.g. 0.5? Use the words *slow* and *oscillate* in your answer.
3. Explain why the dead-band is needed, using the idea of sensor noise.
4. How does the calibration offset make the tracker more accurate? What would happen if the calibration were done while one sensor is in a shadow?
5. Real solar trackers often stop moving at night. Add a **night mode**: if both sensors read below 100 for more than 5 s, slowly return the servo to 90° and switch off the LOCK LED. Write the code.
