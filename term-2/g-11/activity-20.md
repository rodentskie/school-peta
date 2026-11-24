# Activity 20: Serial Parking Gate with Vehicle Detection

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-20`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

A car-park barrier (servo) is controlled by a guard through the Serial Monitor. An ultrasonic sensor watches the area **under the barrier**.

| Key | Action |
|---|---|
| `o` | open the barrier |
| `c` | close the barrier |
| `a` | AUTO mode ON/OFF: open when a car arrives (< 30 cm), close 3 s after it has left |
| `s` | status (gate position, distance, car count, mode) |
| `z` | reset the car counter |
| `?` | help |

**Rules:**

1. The barrier moves **slowly** (servo 0° = closed, 90° = open, at 45 °/s). The yellow LED blinks while it moves.
2. **Safety:** the barrier must **never close while a car is under it** (distance < 30 cm). If `c` is sent while a car is there, print `UNSAFE - vehicle under barrier` and keep it open. If a car appears **while closing**, the barrier must **reverse** and open again.
3. Green LED = open (cars may go), red LED = closed.
4. Count the cars: one car = the distance goes below 30 cm and then back above 30 cm while the gate is open.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Micro Servo |
| 1 | Ultrasonic Distance Sensor (4-pin, HC-SR04) |
| 3 | LEDs (red, yellow, green) + 3 × 1 kΩ resistors |

## Solution

* **Gate states:** `CLOSED`, `OPENING`, `OPEN`, `CLOSING`. The servo angle moves 0.9° every 20 ms (45 °/s).
* **Vehicle detection:** measure about every 100 ms. `carPresent = (d > 0 && d < 30)`.
* **Counting** uses the **falling edge** of `carPresent` (present → not present) while the gate is OPEN.
* **AUTO:** if `carPresent` and gate CLOSED → OPENING. If no car for 3 s (`lastSeen`) and gate OPEN → CLOSING.
* **Safety reverse:** in `CLOSING`, if `carPresent` → change to `OPENING` immediately.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Micro Servo | Signal | **D10** |
| Micro Servo | Power / Ground | 5V rail / GND rail |
| HC-SR04 | Power / Ground | 5V rail / GND rail |
| HC-SR04 | Trigger / Echo | **D7** / **D6** |
| Red LED | Anode | **D3** |
| Yellow LED | Anode | **D4** |
| Green LED | Anode | **D5** |
| LED cathodes | — | 1 kΩ → GND rail |

```
D10 ─► SERVO signal            HC-SR04: VCC─5V TRIG─D7 ECHO─D6 GND─GND
D3 ─► RED ─[1k]─ GND   D4 ─► YELLOW ─[1k]─ GND   D5 ─► GREEN ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 20 - Serial Parking Gate with Vehicle Detection
#include <Servo.h>

const int SERVO_PIN = 10;
const int TRIG_PIN = 7;
const int ECHO_PIN = 6;
const int RED_PIN = 3;
const int YELLOW_PIN = 4;
const int GREEN_PIN = 5;
const float CAR_CM = 30.0;

const int CLOSED = 0, OPENING = 1, OPEN = 2, CLOSING = 3;
const char* NAMES[4] = {"CLOSED", "OPENING", "OPEN", "CLOSING"};

Servo gate;
int state = CLOSED;
float angle = 0;
bool autoMode = false;
bool carPresent = false;
float distance = -1;
int cars = 0;
unsigned long lastMeasure = 0, lastMove = 0, lastSeen = 0;

float measureCm() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  unsigned long us = pulseIn(ECHO_PIN, HIGH, 25000);
  if (us == 0) return -1;
  return us * 0.0343 / 2.0;
}

void setState(int s) {
  if (s == state) return;
  state = s;
  Serial.print("Gate: ");
  Serial.println(NAMES[s]);
}

void requestClose() {
  if (carPresent) { Serial.println("UNSAFE - vehicle under barrier"); return; }
  if (state == OPEN || state == OPENING) setState(CLOSING);
}

void status() {
  Serial.print("Gate ");
  Serial.print(NAMES[state]);
  Serial.print(" (");
  Serial.print(angle, 0);
  Serial.print(" deg) | distance ");
  if (distance < 0) Serial.print("---");
  else Serial.print(distance, 1);
  Serial.print(" cm | cars ");
  Serial.print(cars);
  Serial.print(" | AUTO ");
  Serial.println(autoMode ? "ON" : "OFF");
}

void handle(char c) {
  if (c == 'o') { if (state == CLOSED || state == CLOSING) setState(OPENING); }
  else if (c == 'c') requestClose();
  else if (c == 'a') { autoMode = !autoMode; Serial.println(autoMode ? "AUTO ON" : "AUTO OFF"); }
  else if (c == 's') status();
  else if (c == 'z') { cars = 0; Serial.println("Counter reset"); }
  else if (c == '?' || c == 'h') Serial.println("o open | c close | a auto | s status | z reset count");
  else if (c != '\n' && c != '\r' && c != ' ') { Serial.print("Unknown '"); Serial.print(c); Serial.println("'"); }
}

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(RED_PIN, OUTPUT);
  pinMode(YELLOW_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);
  Serial.begin(9600);
  gate.attach(SERVO_PIN);
  gate.write(0);
  Serial.println("Parking gate: o open | c close | a auto | s status | z reset | ?");
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();

  // 1) vehicle detection
  if (now - lastMeasure >= 100) {
    lastMeasure = now;
    distance = measureCm();
    bool present = distance > 0 && distance < CAR_CM;
    if (present) lastSeen = now;
    if (carPresent && !present && state == OPEN) {
      cars++;
      Serial.print("Car passed. Total: ");
      Serial.println(cars);
    }
    if (present && !carPresent) Serial.println("Vehicle detected");
    carPresent = present;
  }

  // 2) safety + auto logic
  if (state == CLOSING && carPresent) {
    Serial.println("SAFETY: vehicle detected - reversing!");
    setState(OPENING);
  }
  if (autoMode) {
    if (carPresent && (state == CLOSED)) setState(OPENING);
    if (!carPresent && state == OPEN && now - lastSeen > 3000) setState(CLOSING);
  }

  // 3) move the barrier
  if (now - lastMove >= 20) {
    lastMove = now;
    if (state == OPENING) {
      angle = min(90.0, angle + 0.9);
      if (angle >= 90) setState(OPEN);
    } else if (state == CLOSING) {
      angle = max(0.0, angle - 0.9);
      if (angle <= 0) setState(CLOSED);
    }
    gate.write((int)angle);
  }

  // 4) lights
  bool moving = (state == OPENING || state == CLOSING);
  digitalWrite(YELLOW_PIN, moving && (now / 200) % 2);
  digitalWrite(GREEN_PIN, state == OPEN);
  digitalWrite(RED_PIN, state == CLOSED);
}
```

## Assessment Questions

1. Send `o`. How does the barrier move? Which LEDs are on while it moves and after it stops?
2. Move the object closer than 30 cm and send `c`. What happens? What happens if you move the object away and send `c` again?
3. Send `c` and, while the barrier is closing, move the object under it. What does the barrier do?
4. Send `a`, then move the object in and out a few times. What happens to the barrier and the car counter?
5. If the ultrasonic sensor's Echo wire is disconnected, could the barrier close on a car? Why is this dangerous?
