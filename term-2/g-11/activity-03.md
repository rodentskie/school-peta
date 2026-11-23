# Activity 03: Serial Servo Jog Controller with Slew-Rate Limit

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-03`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Control a servo (for example, a robot arm joint) from the keyboard, like a CNC "jog" panel.

| Key | Action |
|---|---|
| `a` / `d` | jog −1° / +1° |
| `A` / `D` | jog −10° / +10° |
| `0`–`9` | go to position: digit × 20° (`9` = 180°) |
| `c` | centre (90°) |
| `w` | sweep mode on/off (moves between 0° and 180° continuously) |
| `+` / `-` | movement speed ×2 / ÷2 (15–240 °/s) |
| `s` | status |
| `h` | help |

1. The servo must **never jump**. It moves towards the **target** at the selected speed (**slew-rate limit**), updated every 20 ms.
2. The target is limited to 0°–180°. If a command tries to go beyond a limit, print `LIMIT!` and flash the LED.
3. The LED is ON while the servo is moving, and OFF when it has reached the target.
4. Print `Target xx° (moving)` when the target changes, and `Arrived at xx°` when the servo arrives.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Micro Servo |
| 1 | LED + 1 kΩ resistor |

## Solution

* Two variables: `target` (where we want to go) and `pos` (where the servo is now, a `float`).
* **Slew limit:** every 20 ms, the maximum step is `speed × 0.020` degrees. `pos` moves by at most this step towards `target`.
* **Sweep mode:** when `pos` reaches the target, the target is changed to the other end (0 ↔ 180).
* **Limit check:** `setTarget(t)` clamps `t` to 0–180 and remembers whether clamping happened, which starts the LED flash.
* **Arrival detection:** the variable `moving` changes from true to false exactly once, so "Arrived" is printed only once.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Micro Servo | Signal | **D9** |
| Micro Servo | Power / Ground | 5V rail / GND rail |
| LED | Anode | **D8** (cathode → 1 kΩ → GND) |

```
D9 ─► SERVO signal   (servo power from + / − rails)
D8 ─► LED ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 03 - Serial Servo Jog Controller
#include <Servo.h>

const int SERVO_PIN = 9;
const int LED_PIN = 8;

Servo servo;
float pos = 90;
int target = 90;
float speed = 60;                 // degrees per second
bool sweep = false;
bool moving = false;
unsigned long lastUpdate = 0;
unsigned long limitFlashUntil = 0;

void setTarget(int t) {
  if (t < 0 || t > 180) {
    t = constrain(t, 0, 180);
    Serial.println("LIMIT!");
    limitFlashUntil = millis() + 600;
  }
  target = t;
  moving = true;
  Serial.print("Target ");
  Serial.print(target);
  Serial.println(" deg (moving)");
}

void help() {
  Serial.println("--- SERVO JOG ---");
  Serial.println(" a/d -1/+1 | A/D -10/+10 | 0-9 goto digit*20 | c centre");
  Serial.println(" w sweep | + - speed | s status | h help");
}

void status() {
  Serial.print("pos=");
  Serial.print(pos, 1);
  Serial.print(" target=");
  Serial.print(target);
  Serial.print(" speed=");
  Serial.print(speed, 0);
  Serial.print(" deg/s sweep=");
  Serial.println(sweep ? "ON" : "OFF");
}

void handle(char c) {
  if (c == 'a') setTarget(target - 1);
  else if (c == 'd') setTarget(target + 1);
  else if (c == 'A') setTarget(target - 10);
  else if (c == 'D') setTarget(target + 10);
  else if (c >= '0' && c <= '9') setTarget((c - '0') * 20);
  else if (c == 'c') setTarget(90);
  else if (c == 'w') {
    sweep = !sweep;
    Serial.println(sweep ? "Sweep ON" : "Sweep OFF");
    if (sweep) setTarget(pos < 90 ? 180 : 0);
  }
  else if (c == '+') { speed = min(240.0, speed * 2); status(); }
  else if (c == '-') { speed = max(15.0, speed / 2); status(); }
  else if (c == 's') status();
  else if (c == 'h' || c == '?') help();
  else if (c != '\n' && c != '\r' && c != ' ') {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
  }
}

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  servo.attach(SERVO_PIN);
  servo.write((int)pos);
  help();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());

  unsigned long now = millis();
  if (now - lastUpdate >= 20) {
    lastUpdate = now;
    float maxStep = speed * 0.020;
    float diff = target - pos;
    if (diff > maxStep) diff = maxStep;
    if (diff < -maxStep) diff = -maxStep;
    pos += diff;
    servo.write((int)(pos + 0.5));

    if (moving && fabs(target - pos) < 0.01) {
      moving = false;
      Serial.print("Arrived at ");
      Serial.print(target);
      Serial.println(" deg");
      if (sweep) setTarget(target == 180 ? 0 : 180);
    }
  }

  bool flash = (long)(limitFlashUntil - now) > 0;
  if (flash) digitalWrite(LED_PIN, (now / 80) % 2);
  else digitalWrite(LED_PIN, moving ? HIGH : LOW);
}
```

## Assessment Questions

1. Send `D` three times. How does the servo move? Does it jump or move smoothly?
2. Send `9`, then `0`. What does the LED do while the servo is moving and after it arrives?
3. The servo is at 180°. What happens when you send `D` again? What do the LED and Serial Monitor show?
4. Send `w`, then `+` and `-` a few times. What changes in the servo's movement?
5. If the servo's signal wire is disconnected from D9, what happens when you send commands? Does the Serial Monitor still say the servo is moving?
