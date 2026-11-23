# Activity 06: Serial DC Motor Speed Controller with Ramp and Emergency Stop

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-06`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Control a DC motor (through an NPN transistor driver) from the Serial Monitor.

| Key | Action |
|---|---|
| `0`–`9` | target speed 0 %, 10 % … 90 % |
| `f` | full speed (100 %) |
| `+` / `-` | target +5 % / −5 % |
| `m` | ramp mode ON/OFF (soft start/stop) |
| `x` | **EMERGENCY STOP**: motor off instantly, commands locked |
| `u` | unlock after an emergency stop (only if the target is 0 %) |
| `s` | status |
| `h` | help |

1. In **ramp mode**, the actual speed changes by at most **1 % every 20 ms** (2 s from 0 to 100 %). Without ramp mode, the speed changes immediately.
2. Speeds from 1 % to 24 % are too low for the motor to turn. Raise them to **25 %** and print a note.
3. After `x`, every speed command prints `LOCKED - send 0 then u`. `u` only works when the target is 0 %. This is a safety rule: the motor must not restart at the old speed.
4. LED: ON = motor running, OFF = stopped, blinking = locked (emergency).
5. Print `Target xx% | Actual yy%` whenever the actual speed reaches the target.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | DC Motor |
| 1 | NPN Transistor (BJT) + 1 kΩ base resistor |
| 1 | Diode (flyback) |
| 1 | LED + 1 kΩ resistor |

## Solution

* `target` and `actual` are percentages (0–100). Each 20 ms, `actual` moves towards `target` (ramp mode) or jumps to it. Then `analogWrite(MOTOR, actual × 255 / 100)`.
* `setTarget()` is the only way to change the target. It checks the lock, the range (`constrain` 0–100) and the minimum speed rule.
* **Emergency stop:** `locked = true; target = actual = 0;` and PWM 0 immediately (no ramp).
* **Unlock rule:** `u` is accepted only when `target == 0` (the operator must send `0` first).
* The "reached" message uses a flag so it is printed only once per change.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| NPN transistor | Base | 1 kΩ → **D9** (PWM) |
| NPN transistor | Collector | Motor terminal 2 + diode anode |
| NPN transistor | Emitter | GND rail |
| DC Motor | Terminal 1 | 5V rail |
| Diode | Cathode | 5V rail |
| LED | Anode | **D8** (cathode → 1 kΩ → GND) |

```
 5V ──┬──────────┐
      │        (MOTOR)
 diode▲          │
      └──────────┤ C
 D9 ─[1kΩ]─── B ─┤ NPN
                 │ E
 GND ────────────┘           D8 ─► LED ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 06 - Serial DC Motor Controller
const int MOTOR_PIN = 9;
const int LED_PIN = 8;
const int MIN_RUN = 25;

int target = 0;
int actual = 0;
bool rampMode = true;
bool locked = false;
bool reportPending = false;
unsigned long lastRamp = 0;

void help() {
  Serial.println("--- MOTOR CONSOLE ---");
  Serial.println(" 0-9 speed x10% | f full | + - 5% | m ramp on/off");
  Serial.println(" x E-STOP | u unlock | s status | h help");
}

void status() {
  Serial.print("Target ");
  Serial.print(target);
  Serial.print("% | Actual ");
  Serial.print(actual);
  Serial.print("% | Ramp ");
  Serial.print(rampMode ? "ON" : "OFF");
  Serial.println(locked ? " | LOCKED" : "");
}

void setTarget(int t) {
  if (locked) {
    if (t == 0) {
      target = 0;
      Serial.println("Target 0% (now send u to unlock)");
    } else {
      Serial.println("LOCKED - send 0 then u");
    }
    return;
  }
  t = constrain(t, 0, 100);
  if (t > 0 && t < MIN_RUN) {
    Serial.print(t);
    Serial.print("% too low to turn - using ");
    Serial.print(MIN_RUN);
    Serial.println("%");
    t = MIN_RUN;
  }
  target = t;
  reportPending = true;
  Serial.print("New target ");
  Serial.print(target);
  Serial.println("%");
}

void emergencyStop() {
  locked = true;
  target = 0;
  actual = 0;
  analogWrite(MOTOR_PIN, 0);
  Serial.println("!!! EMERGENCY STOP - motor off, controls locked !!!");
}

void handle(char c) {
  if (c >= '0' && c <= '9') setTarget((c - '0') * 10);
  else if (c == 'f') setTarget(100);
  else if (c == '+') setTarget(target + 5);
  else if (c == '-') setTarget(target - 5 < MIN_RUN ? 0 : target - 5);
  else if (c == 'm') {
    rampMode = !rampMode;
    Serial.println(rampMode ? "Ramp mode ON" : "Ramp mode OFF");
  }
  else if (c == 'x') emergencyStop();
  else if (c == 'u') {
    if (!locked) Serial.println("Not locked");
    else if (target != 0) Serial.println("Send 0 first");
    else { locked = false; Serial.println("Unlocked"); }
  }
  else if (c == 's') status();
  else if (c == 'h' || c == '?') help();
  else if (c != '\n' && c != '\r' && c != ' ') {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
  }
}

void setup() {
  pinMode(MOTOR_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  help();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();

  if (now - lastRamp >= 20) {
    lastRamp = now;
    if (rampMode) {
      if (actual < target) actual++;
      else if (actual > target) actual--;
    } else {
      actual = target;
    }
    analogWrite(MOTOR_PIN, map(actual, 0, 100, 0, 255));
    if (reportPending && actual == target) {
      reportPending = false;
      status();
    }
  }

  if (locked) digitalWrite(LED_PIN, (now / 150) % 2);
  else digitalWrite(LED_PIN, actual > 0 ? HIGH : LOW);
}
```

## Assessment Questions

1. In ramp mode, how long does the motor take to go from 30 % to 80 %? What PWM value is sent at 80 %?
2. The motor runs at 60 %. The operator sends `x`, then `5`, then `0`, then `u`, then `7`. Write what the program prints after each character.
3. Why does the `-` command jump to 0 % when the result would be below 25 %, instead of rounding up to 25 % like `setTarget()`?
4. Explain the role of the flyback diode and the transistor. Why can't the motor be connected directly to D9?
5. Add a **timer** command `t`: the motor runs at the current target for 10 s and then ramps down to 0 % automatically. Write the code and describe how it interacts with `x`.
