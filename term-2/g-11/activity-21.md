# Activity 21: Serial Smart-Home Controller with Scenes

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-21`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Control a model smart home from the Serial Monitor: 3 room lights (LEDs with dimming), a front door (servo lock) and a doorbell (buzzer).

| Key | Action |
|---|---|
| `1` `2` `3` | toggle the light in room 1 (living), 2 (kitchen), 3 (bedroom) |
| `!` `@` `#` | dim room 1 / 2 / 3: cycles 100 % → 50 % → 10 % → 100 % |
| `d` | lock / unlock the front door |
| `b` | ring the doorbell (ding-dong) |
| `m` | **Morning** scene: kitchen 100 %, living 50 %, bedroom off, door unlocked |
| `n` | **Night** scene: all lights off except bedroom 10 %, door locked |
| `v` | **Away** scene: everything off, door locked, lights "simulate presence" (random room on/off every few seconds) |
| `x` | all off (also stops Away mode) |
| `s` | status of everything |

1. Lights fade smoothly to their new brightness (no sudden jumps), about 0.5 s per change.
2. The door servo is at 0° when locked and 90° when unlocked.
3. In **Away** mode, every 2–5 s a random room toggles, to look as if someone is home. Any manual light command ends Away mode.
4. The doorbell plays two notes (E6, C6) without blocking.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 3 | LEDs (rooms) + 3 × 1 kΩ resistors |
| 1 | Micro Servo (door lock) |
| 1 | Piezo buzzer |

## Solution

* **Per-room arrays:** `on[3]`, `level[3]` (100/50/10 %), `current[3]` (PWM now) and `target[3]` (PWM wanted). Every 5 ms, `current` moves 3 PWM steps towards `target`.
* `applyRoom(i)` calculates `target[i] = on[i] ? level[i] × 255 / 100 : 0`.
* **Scenes** set several values at once and call `applyRoom()` for each room.
* **Away mode:** `nextAwayAt = now + random(2000, 5001)`. Each time it is reached, toggle `random(3)`.
* **Pin choice:** the Servo library uses Timer1, which disables PWM on D9 and D10. That is why the dimmable rooms use D5, D6 and D3.
* **Doorbell:** `bellStep` 0 → play E6, wait 400 ms → play C6, wait 600 ms → done.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Room 1 LED | Anode | **D5** (PWM) |
| Room 2 LED | Anode | **D6** (PWM) |
| Room 3 LED | Anode | **D3** (PWM) |
| LED cathodes | — | 1 kΩ → GND rail |
| Door servo | Signal | **D10** |
| Door servo | Power / Ground | 5V rail / GND rail |
| Piezo | + / − | **D8** / GND |

```
D5 ─► ROOM1 ─[1k]─ GND    D10 ─► SERVO signal
D6 ─► ROOM2 ─[1k]─ GND    D8  ─► PIEZO ── GND
D3 ─► ROOM3 ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 21 - Serial Smart-Home Controller
#include <Servo.h>

const int ROOM_PINS[3] = {5, 6, 3};   // not D9/D10: the Servo library uses their timer
const char* ROOMS[3] = {"Living", "Kitchen", "Bedroom"};
const int SERVO_PIN = 10;
const int BUZZER_PIN = 8;

Servo door;
bool on[3] = {false, false, false};
int level[3] = {100, 100, 100};
int current[3] = {0, 0, 0};
int target[3] = {0, 0, 0};
bool locked = true;
bool away = false;
unsigned long nextAwayAt = 0, lastFade = 0;
int bellStep = 0;
unsigned long bellAt = 0;

void applyRoom(int i) {
  target[i] = on[i] ? level[i] * 255 / 100 : 0;
}

void setDoor(bool lock) {
  locked = lock;
  door.write(lock ? 0 : 90);
}

void status() {
  for (int i = 0; i < 3; i++) {
    Serial.print(ROOMS[i]);
    Serial.print(": ");
    if (on[i]) { Serial.print(level[i]); Serial.print("%  "); }
    else Serial.print("off  ");
  }
  Serial.print("| Door ");
  Serial.print(locked ? "LOCKED" : "UNLOCKED");
  Serial.println(away ? " | AWAY mode" : "");
}

void stopAway() {
  if (away) { away = false; Serial.println("Away mode ended"); }
}

void scene(bool l1, int v1, bool l2, int v2, bool l3, int v3, bool lockDoor) {
  on[0] = l1; level[0] = v1;
  on[1] = l2; level[1] = v2;
  on[2] = l3; level[2] = v3;
  for (int i = 0; i < 3; i++) applyRoom(i);
  setDoor(lockDoor);
}

void handle(char c) {
  if (c >= '1' && c <= '3') {
    stopAway();
    int i = c - '1';
    on[i] = !on[i];
    applyRoom(i);
  } else if (c == '!' || c == '@' || c == '#') {
    stopAway();
    int i = (c == '!') ? 0 : (c == '@' ? 1 : 2);
    level[i] = (level[i] == 100) ? 50 : (level[i] == 50 ? 10 : 100);
    on[i] = true;
    applyRoom(i);
  } else if (c == 'd') {
    setDoor(!locked);
  } else if (c == 'b') {
    bellStep = 1;
    bellAt = millis();
    Serial.println("Ding-dong!");
    return;
  } else if (c == 'm') {
    stopAway();
    scene(true, 50, true, 100, false, 100, false);
    Serial.println("Scene: MORNING");
  } else if (c == 'n') {
    stopAway();
    scene(false, 100, false, 100, true, 10, true);
    Serial.println("Scene: NIGHT");
  } else if (c == 'v') {
    scene(false, 100, false, 100, false, 100, true);
    away = true;
    nextAwayAt = millis() + 2000;
    Serial.println("Scene: AWAY (presence simulation)");
  } else if (c == 'x') {
    stopAway();
    scene(false, level[0], false, level[1], false, level[2], locked);
    Serial.println("All lights off");
  } else if (c == 's') {
  } else {
    if (c != '\n' && c != '\r' && c != ' ') {
      Serial.println("1 2 3 lights | ! @ # dim | d door | b bell | m n v scenes | x off | s status");
    }
    return;
  }
  status();
}

void setup() {
  for (int i = 0; i < 3; i++) pinMode(ROOM_PINS[i], OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  randomSeed(analogRead(A0));
  door.attach(SERVO_PIN);
  setDoor(true);
  Serial.println("Smart home ready. Press s for status.");
  status();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();

  if (now - lastFade >= 5) {
    lastFade = now;
    for (int i = 0; i < 3; i++) {
      if (current[i] < target[i]) current[i] = min(target[i], current[i] + 3);
      else if (current[i] > target[i]) current[i] = max(target[i], current[i] - 3);
      analogWrite(ROOM_PINS[i], current[i]);
    }
  }

  if (away && (long)(now - nextAwayAt) >= 0) {
    int i = random(3);
    on[i] = !on[i];
    level[i] = 100;
    applyRoom(i);
    nextAwayAt = now + random(2000, 5001);
    Serial.print("(away) ");
    Serial.print(ROOMS[i]);
    Serial.println(on[i] ? " on" : " off");
  }

  if (bellStep == 1) { tone(BUZZER_PIN, 1319, 350); bellStep = 2; bellAt = now + 400; }
  else if (bellStep == 2 && (long)(now - bellAt) >= 0) { tone(BUZZER_PIN, 1047, 600); bellStep = 0; }
}
```

## Assessment Questions

1. Send `1`, then `!` twice. What happens to the room 1 light? Does it change suddenly or smoothly?
2. Send `m`, then `n`. Describe all the lights and the door after each scene.
3. Send `v` and watch for 20 seconds. What do the lights do? What happens when you send `2`?
4. Send `b` while the lights are fading. Does the doorbell stop the fading?
5. If room 3's LED is moved from D3 to D7 (a pin without `~`), what happens with the night scene's 10 % light? What if its resistor is removed?
