# Activity 26: Serial Elevator Controller (3 Floors)

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-26`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Simulate a 3-floor elevator. Three LEDs show the current floor, a servo is the **door** (0° closed, 90° open), and a buzzer makes the arrival "ding".

| Key | Action |
|---|---|
| `1` `2` `3` | call / send the elevator to floor 1, 2, 3 |
| `o` | open the door (only when stopped at a floor) |
| `c` | close the door now |
| `e` | **EMERGENCY STOP** ON/OFF |
| `s` | status |

**Elevator rules:**

1. **Requests are remembered** (one flag per floor). Several floors can be requested at once, e.g. `31`.
2. **Direction logic (like a real elevator):** keep moving in the same direction while there are requests ahead. Only reverse when there are none left in that direction. The elevator going up from 1 with requests for 3 and 2 stops at **2 first**, then 3.
3. Moving one floor takes **2 s**. The floor LED of the floor being passed blinks while moving.
4. At each requested floor: stop, ding, open the door, wait **3 s**, close the door (the servo moves slowly), then continue.
5. **The elevator never moves while the door is open** (safety interlock).
6. **EMERGENCY:** stop immediately (even between floors), all LEDs blink, the buzzer beeps every second, and requests are ignored until `e` is sent again.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 3 | LEDs (floor 1, 2, 3) + 3 × 1 kΩ resistors |
| 1 | Micro Servo (door) |
| 1 | Piezo buzzer |

## Solution

* **Car states:** `IDLE`, `MOVING`, `DOOR_OPENING`, `DOOR_OPEN`, `DOOR_CLOSING`, `EMERGENCY`.
* `requests[3]` (bool), `floor` (0–2), `dir` (+1 up, −1 down, 0 none).
* `anyAhead(dir)` checks whether there is a request beyond the current floor in that direction. `chooseDirection()`: keep `dir` if `anyAhead(dir)`; otherwise try `-dir`; otherwise `dir = 0`.
* **MOVING:** after 2 s, `floor += dir`. If `requests[floor]` is set, clear it and start `DOOR_OPENING`.
* **Door:** the servo angle moves 2° every 20 ms. The open time is 3 s.
* **Emergency** saves the state and restores it when released. While moving, the half-finished floor trip restarts from the beginning.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Floor 1 LED | Anode | **D3** |
| Floor 2 LED | Anode | **D4** |
| Floor 3 LED | Anode | **D5** |
| LED cathodes | — | 1 kΩ → GND rail |
| Door servo | Signal | **D10** |
| Door servo | Power / Ground | 5V rail / GND rail |
| Piezo | + / − | **D9** / GND |

```
D3 ─► FLOOR1 ─[1k]─ GND     D10 ─► SERVO (door) signal
D4 ─► FLOOR2 ─[1k]─ GND     D9  ─► PIEZO ── GND
D5 ─► FLOOR3 ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 26 - Serial Elevator Controller
#include <Servo.h>

const int FLOOR_PINS[3] = {3, 4, 5};
const int SERVO_PIN = 10;
const int BUZZER_PIN = 9;

const int IDLE = 0, MOVING = 1, DOOR_OPENING = 2, DOOR_OPEN = 3, DOOR_CLOSING = 4, EMERGENCY = 5;
const char* NAMES[6] = {"IDLE", "MOVING", "DOOR OPENING", "DOOR OPEN", "DOOR CLOSING", "EMERGENCY"};

Servo door;
int state = IDLE, savedState = IDLE;
bool requests[3] = {false, false, false};
int floor_ = 0;
int dir = 0;
float doorAngle = 0;
unsigned long stateStart = 0, lastDoorStep = 0;

void go(int s) {
  state = s;
  stateStart = millis();
  Serial.print("[floor ");
  Serial.print(floor_ + 1);
  Serial.print("] ");
  Serial.println(NAMES[s]);
}

bool anyAhead(int d) {
  for (int f = floor_ + d; f >= 0 && f < 3; f += d) if (requests[f]) return true;
  return false;
}

void chooseDirection() {
  if (dir != 0 && anyAhead(dir)) return;
  if (anyAhead(1)) dir = 1;
  else if (anyAhead(-1)) dir = -1;
  else dir = 0;
}

void status() {
  Serial.print("Floor ");
  Serial.print(floor_ + 1);
  Serial.print(" | ");
  Serial.print(NAMES[state]);
  Serial.print(" | dir ");
  Serial.print(dir > 0 ? "UP" : (dir < 0 ? "DOWN" : "-"));
  Serial.print(" | requests:");
  for (int f = 0; f < 3; f++) if (requests[f]) { Serial.print(' '); Serial.print(f + 1); }
  Serial.println();
}

void handle(char c) {
  if (c == 'e') {
    if (state == EMERGENCY) {
      Serial.println("Emergency released");
      int s = savedState;
      go(s);                                      // restarts any half-finished trip
    } else {
      savedState = state;
      go(EMERGENCY);
      noTone(BUZZER_PIN);
    }
    return;
  }
  if (state == EMERGENCY) {
    if (c != '\n' && c != '\r' && c != ' ') Serial.println("EMERGENCY - send e to release");
    return;
  }
  if (c >= '1' && c <= '3') {
    int f = c - '1';
    if (f == floor_ && (state == IDLE || state == DOOR_OPEN || state == DOOR_OPENING)) {
      if (state == IDLE) go(DOOR_OPENING);
      else stateStart = millis();                 // keep the door open longer
    } else {
      requests[f] = true;
      Serial.print("Request: floor ");
      Serial.println(f + 1);
    }
  }
  else if (c == 'o') { if (state == IDLE || state == DOOR_CLOSING) go(DOOR_OPENING); }
  else if (c == 'c') { if (state == DOOR_OPEN) go(DOOR_CLOSING); }
  else if (c == 's') status();
  else if (c != '\n' && c != '\r' && c != ' ') Serial.println("1 2 3 call | o open | c close | e emergency | s status");
}

void showFloor(unsigned long now) {
  for (int f = 0; f < 3; f++) {
    bool on = (f == floor_);
    if (state == MOVING && f == floor_) on = (now / 250) % 2;
    if (state == EMERGENCY) on = (now / 300) % 2;
    digitalWrite(FLOOR_PINS[f], on);
  }
}

void setup() {
  for (int f = 0; f < 3; f++) pinMode(FLOOR_PINS[f], OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  door.attach(SERVO_PIN);
  door.write(0);
  Serial.println("Elevator ready at floor 1. Send 1 2 3 to call.");
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();
  unsigned long t = now - stateStart;

  if (state == IDLE) {
    chooseDirection();
    if (dir != 0) go(MOVING);
  }
  else if (state == MOVING && t >= 2000) {
    floor_ += dir;
    if (requests[floor_]) {
      requests[floor_] = false;
      tone(BUZZER_PIN, 1568, 300);                 // ding
      go(DOOR_OPENING);
    } else {
      go(MOVING);                                  // pass this floor
    }
  }
  else if (state == DOOR_OPEN && t >= 3000) go(DOOR_CLOSING);
  else if (state == EMERGENCY) {
    if (t % 1000 < 100) tone(BUZZER_PIN, 2500);
    else noTone(BUZZER_PIN);
  }

  // door motion (interlock: only in door states)
  if (now - lastDoorStep >= 20) {
    lastDoorStep = now;
    if (state == DOOR_OPENING) {
      doorAngle = min(90.0, doorAngle + 2);
      if (doorAngle >= 90) go(DOOR_OPEN);
    } else if (state == DOOR_CLOSING) {
      doorAngle = max(0.0, doorAngle - 2);
      if (doorAngle <= 0) go(IDLE);
    }
    door.write((int)doorAngle);
  }
  showFloor(now);
}
```

## Assessment Questions

1. Send `3`. What do the floor LEDs do while the elevator moves? What happens when it arrives?
2. With the elevator at floor 1, send `32`. At which floor does it stop first? Why?
3. Send `2` and, while the door is open, send `1`. Does the elevator leave immediately? What happens first?
4. Send `3`, then `e` while it is moving. What do the LEDs and buzzer do? What happens to the request when you send `e` again?
5. If the door servo's signal wire is disconnected, what does the Serial Monitor say at each floor? Why would this be dangerous in a real elevator?
