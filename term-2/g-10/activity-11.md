# Activity 11: Ultrasonic Parking Assistant with Median Filter

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-11`

## Problem Statement

Build a car parking assistant using the **HC-SR04 ultrasonic sensor**.

1. Measure the distance in cm. Take **5 readings** and use the **median**, which removes single wrong echoes.
2. Show the zone with LEDs and beeps:

| Distance | LEDs | Buzzer |
|---|---|---|
| > 100 cm | Green | silent |
| 50–100 cm | Yellow | beep every 600 ms |
| 20–50 cm | Red | beep every 200 ms |
| < 20 cm | Red | **continuous** tone (STOP!) |
| no echo | all LEDs blink | silent |

3. The beeping must be **non-blocking**. The sensor keeps measuring while the buzzer beeps.
4. Print the distance and zone every 300 ms.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Ultrasonic Distance Sensor (4-pin, HC-SR04) |
| 3 | LEDs (green, yellow, red) + 3 × 1 kΩ resistors |
| 1 | Piezo buzzer |

## Solution

* **Measuring:** send a 10 µs HIGH pulse on TRIG. The sensor answers with a pulse on ECHO whose length equals the sound's round trip. `pulseIn(ECHO, HIGH, 30000)` returns the time in µs, or 0 on timeout. Distance = `time × 0.0343 / 2` cm (speed of sound 343 m/s, halved for the trip back).
* **Median of 5:** store 5 readings, sort them with insertion sort, and take the middle element `[2]`. A median ignores one or two wild values, while an average would be pulled off by them.
* **Beep scheduler:** `beepInterval` depends on the zone. If `now − lastBeep ≥ interval`, start a 60 ms tone. `tone(pin, f, 60)` stops by itself, so nothing blocks.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| HC-SR04 | Power (Vcc) | 5V rail |
| HC-SR04 | Trigger | **D7** |
| HC-SR04 | Echo | **D6** |
| HC-SR04 | Ground | GND rail |
| Green LED | Anode | **D3** |
| Yellow LED | Anode | **D4** |
| Red LED | Anode | **D5** |
| LED cathodes | — | 1 kΩ → GND rail |
| Piezo | + / − | **D9** / GND |

```
HC-SR04: VCC─5V  TRIG─D7  ECHO─D6  GND─GND
D3 ─► GREEN  ─[1k]─ GND
D4 ─► YELLOW ─[1k]─ GND        D9 ─► PIEZO ── GND
D5 ─► RED    ─[1k]─ GND
```

## Code

```cpp
// Activity 11 - Ultrasonic Parking Assistant
const int TRIG_PIN = 7;
const int ECHO_PIN = 6;
const int GREEN_PIN = 3;
const int YELLOW_PIN = 4;
const int RED_PIN = 5;
const int BUZZER_PIN = 9;

unsigned long lastBeep = 0, lastPrint = 0;
float distance = -1;

float measureOnce() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  unsigned long us = pulseIn(ECHO_PIN, HIGH, 30000);
  if (us == 0) return -1;                 // no echo / out of range
  return us * 0.0343 / 2.0;
}

float measureMedian() {
  float v[5];
  for (int i = 0; i < 5; i++) {
    v[i] = measureOnce();
    delayMicroseconds(500);
  }
  // insertion sort
  for (int i = 1; i < 5; i++) {
    float key = v[i];
    int j = i - 1;
    while (j >= 0 && v[j] > key) {
      v[j + 1] = v[j];
      j--;
    }
    v[j + 1] = key;
  }
  return v[2];
}

void setLeds(int g, int y, int r) {
  digitalWrite(GREEN_PIN, g);
  digitalWrite(YELLOW_PIN, y);
  digitalWrite(RED_PIN, r);
}

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(YELLOW_PIN, OUTPUT);
  pinMode(RED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Parking assistant ready");
}

void loop() {
  unsigned long now = millis();
  distance = measureMedian();

  const char* zone;
  unsigned long interval = 0;            // 0 = no beeping
  bool continuous = false;

  if (distance < 0) {
    zone = "NO ECHO";
    bool b = (now / 300) % 2;
    setLeds(b, b, b);
  } else if (distance > 100) {
    zone = "SAFE";
    setLeds(HIGH, LOW, LOW);
  } else if (distance > 50) {
    zone = "CAUTION";
    setLeds(LOW, HIGH, LOW);
    interval = 600;
  } else if (distance > 20) {
    zone = "CLOSE";
    setLeds(LOW, LOW, HIGH);
    interval = 200;
  } else {
    zone = "STOP!";
    setLeds(LOW, LOW, HIGH);
    continuous = true;
  }

  if (continuous) {
    tone(BUZZER_PIN, 2200);
  } else if (interval > 0) {
    if (now - lastBeep >= interval) {
      lastBeep = now;
      tone(BUZZER_PIN, 1500, 60);
    }
  } else {
    noTone(BUZZER_PIN);
  }

  if (now - lastPrint >= 300) {
    lastPrint = now;
    Serial.print("Distance: ");
    if (distance < 0) Serial.print("---");
    else Serial.print(distance, 1);
    Serial.print(" cm  [");
    Serial.print(zone);
    Serial.println("]");
  }
}
```

## Assessment Questions

1. What happens to the LEDs and the buzzer as you move the object closer to the sensor?
2. How is the beeping different in the yellow zone and in the red zone?
3. What happens when the object is very close (less than 20 cm)?
4. What happens when the object is moved out of the sensor's range?
5. If the Trigger and Echo wires are swapped, what do you think the LEDs will show? Why?
