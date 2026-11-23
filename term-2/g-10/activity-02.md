# Activity 02: Smart Pedestrian Crossing (Non-Blocking State Machine)

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-02`

## Problem Statement

Program a pedestrian crossing controller. The car lights (red, yellow, green) normally stay **green**. When a pedestrian presses the request button:

1. The request is **latched** (remembered) even if the button is released immediately.
2. The car light stays green for at least **5 s** in total, so cars are not stopped too often. Then it turns yellow for 2 s and red.
3. While the cars are red, the **WALK** LED is on for 5 s and the buzzer chirps every 500 ms (for visually impaired pedestrians).
4. The WALK LED then **flashes** for 3 s (warning), followed by a 1 s "all red" safety gap before the cars go green again.
5. Pressing the button while pedestrians are already crossing must **not** restart the cycle.
6. No `delay()` is allowed. Every timing must use `millis()`.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 4 | LEDs (red, yellow, green, green WALK) + 4 × 1 kΩ resistors |
| 1 | Pushbutton |
| 1 | Piezo buzzer |

## Solution

The controller is a **5-state machine**. Each state has a fixed duration, and the variable `stateStart` stores when the current state began.

```
CAR_GREEN --(request && ≥5 s)--> CAR_YELLOW --2 s--> WALK --5 s--> WALK_FLASH --3 s--> ALL_RED --1 s--> CAR_GREEN
```

* `enterState()` is the only place that changes the lights. This keeps the output logic in one location.
* The request flag `walkRequested` is set by the button at any time but cleared only when the WALK phase begins, so a press during WALK is ignored.
* The chirp and the flashing use `(now - stateStart) % period` to compute on/off without extra timers.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Red car LED | Anode | **D10** |
| Yellow car LED | Anode | **D9** |
| Green car LED | Anode | **D8** |
| WALK LED (green) | Anode | **D7** |
| All LED cathodes | — | 1 kΩ → GND rail |
| Pushbutton | T1 / T2 | GND / **D2** (INPUT_PULLUP) |
| Piezo | + / − | **D6** / GND |

```
D10 ─► RED    ─[1k]─ GND        D7 ─► WALK ─[1k]─ GND
D9  ─► YELLOW ─[1k]─ GND        D2 ─► BUTTON ───── GND
D8  ─► GREEN  ─[1k]─ GND        D6 ─► PIEZO ────── GND
```

## Code

```cpp
// Activity 02 - Smart Pedestrian Crossing
const int RED_PIN = 10;
const int YELLOW_PIN = 9;
const int GREEN_PIN = 8;
const int WALK_PIN = 7;
const int BTN_PIN = 2;
const int BUZZER_PIN = 6;

const int CAR_GREEN = 0;
const int CAR_YELLOW = 1;
const int WALK = 2;
const int WALK_FLASH = 3;
const int ALL_RED = 4;

const unsigned long MIN_GREEN_MS = 5000;
const unsigned long YELLOW_MS = 2000;
const unsigned long WALK_MS = 5000;
const unsigned long FLASH_MS = 3000;
const unsigned long ALL_RED_MS = 1000;

int state = CAR_GREEN;
unsigned long stateStart = 0;
bool walkRequested = false;
int crossings = 0;

void setLights(int r, int y, int g, int w) {
  digitalWrite(RED_PIN, r);
  digitalWrite(YELLOW_PIN, y);
  digitalWrite(GREEN_PIN, g);
  digitalWrite(WALK_PIN, w);
}

void enterState(int s) {
  state = s;
  stateStart = millis();
  if (s == CAR_GREEN) {
    setLights(LOW, LOW, HIGH, LOW);
    Serial.println("[CARS GREEN]");
  } else if (s == CAR_YELLOW) {
    setLights(LOW, HIGH, LOW, LOW);
    Serial.println("[CARS YELLOW]");
  } else if (s == WALK) {
    walkRequested = false;           // request served
    crossings++;
    setLights(HIGH, LOW, LOW, HIGH);
    Serial.print("[WALK] crossing #");
    Serial.println(crossings);
  } else if (s == WALK_FLASH) {
    Serial.println("[WALK FLASHING - finish crossing]");
  } else if (s == ALL_RED) {
    setLights(HIGH, LOW, LOW, LOW);
    noTone(BUZZER_PIN);
    Serial.println("[ALL RED safety gap]");
  }
}

void setup() {
  pinMode(RED_PIN, OUTPUT);
  pinMode(YELLOW_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(WALK_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  enterState(CAR_GREEN);
}

void loop() {
  unsigned long now = millis();
  unsigned long t = now - stateStart;

  // latch the request (ignored while pedestrians already have the light)
  if (digitalRead(BTN_PIN) == LOW && !walkRequested &&
      (state == CAR_GREEN || state == CAR_YELLOW || state == ALL_RED)) {
    walkRequested = true;
    Serial.println("Request received - please wait");
  }

  if (state == CAR_GREEN) {
    if (walkRequested && t >= MIN_GREEN_MS) enterState(CAR_YELLOW);
  }
  else if (state == CAR_YELLOW) {
    if (t >= YELLOW_MS) enterState(WALK);
  }
  else if (state == WALK) {
    // chirp 60 ms at the start of every 500 ms slot
    if (t % 500 < 60) tone(BUZZER_PIN, 2000);
    else noTone(BUZZER_PIN);
    if (t >= WALK_MS) enterState(WALK_FLASH);
  }
  else if (state == WALK_FLASH) {
    bool on = (t % 400) < 200;
    digitalWrite(WALK_PIN, on ? HIGH : LOW);
    if (on && t % 400 < 40) tone(BUZZER_PIN, 2500);
    else noTone(BUZZER_PIN);
    if (t >= FLASH_MS) enterState(ALL_RED);
  }
  else if (state == ALL_RED) {
    if (t >= ALL_RED_MS) enterState(CAR_GREEN);
  }
}
```

## Assessment Questions

1. What do the car lights do when nobody presses the button?
2. Describe what happens to all the lights and the buzzer, in order, after a pedestrian presses the button.
3. What happens if the button is pressed again while the WALK light is already on?
4. What will happen to the red LED if its resistor is removed? Will the other LEDs be affected?
5. If the buzzer is disconnected, does the crossing still work? Who would be affected in real life?
