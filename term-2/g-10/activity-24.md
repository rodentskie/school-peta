# Activity 24: DC Motor Soft-Start Controller with Emergency Stop

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-24`

## Problem Statement

Large motors are damaged if they start or change speed too suddenly. Build a motor controller with **acceleration limiting** and a safety **emergency stop**.

1. The potentiometer sets the **target speed** (0–100 %).
2. The actual PWM output may change by at most **2 units every 20 ms** (about 2.5 s from 0 to full speed), both up and down (ramp).
3. Below 15 % the motor cannot turn. Any target under 15 % is treated as 0 (dead zone).
4. The **E-STOP button** stops the motor **immediately** (no ramp) and latches. The status LED blinks fast.
5. **Safety interlock:** after an E-STOP, the motor may only restart when the knob has been turned back to **0 %** and the button pressed again (re-arm). This prevents a surprise start at high speed.
6. Status LED: OFF = stopped, ON = running, fast blink = E-STOP, slow blink = waiting for knob to return to zero.
7. Print target, actual PWM and state every 500 ms.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | DC Motor |
| 1 | NPN Transistor (BJT) |
| 1 | Diode (flyback) |
| 2 | 1 kΩ resistors (base, LED) |
| 1 | Potentiometer |
| 1 | Pushbutton |
| 1 | LED |

## Solution

**Why a transistor?** An Arduino pin can only supply about 20–40 mA, but a motor needs much more. The NPN transistor acts as a switch. The small base current (through 1 kΩ) controls the large collector current through the motor. **PWM** on the base switches the motor on and off very fast, which changes its average speed.

**Why a diode?** A motor is a coil. When the current is switched off, it produces a high voltage spike. The flyback diode across the motor gives this spike a safe path and protects the transistor.

**Ramp:**

```
if (actual < target) actual = min(actual + 2, target)
if (actual > target) actual = max(actual - 2, target)
```

**States:** `RUN`, `ESTOP`, `WAIT_ZERO`. `ESTOP` → (button) → `WAIT_ZERO` → (knob < 3 %) → `RUN`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| NPN transistor | Base | 1 kΩ → **D9** (PWM) |
| NPN transistor | Collector | Motor terminal 2 and diode anode |
| NPN transistor | Emitter | GND rail |
| DC Motor | Terminal 1 | 5V rail |
| Diode | Cathode | 5V rail (across the motor) |
| Potentiometer | T1 / Wiper / T2 | GND / **A0** / 5V |
| E-STOP button | T1 / T2 | GND / **D2** |
| Status LED | Anode | **D8** (cathode → 1 kΩ → GND) |

```
 5V ──┬──────────┐
      │        (MOTOR)
  diode ▲        │
      └──────────┤ collector
 D9 ─[1kΩ]─ base ─┤ NPN
                 │ emitter
 GND ────────────┘
 A0 ◄─ POT    D2 ─► E-STOP ── GND    D8 ─► LED ─[1k]─ GND
```

## Code

```cpp
// Activity 24 - DC Motor Soft-Start Controller with E-STOP
const int MOTOR_PIN = 9;
const int POT_PIN = A0;
const int BTN_PIN = 2;
const int LED_PIN = 8;

const int RUN = 0;
const int ESTOP = 1;
const int WAIT_ZERO = 2;
const char* NAMES[3] = {"RUN", "E-STOP", "WAIT KNOB=0"};

int state = RUN;
int actual = 0;          // PWM 0..255 actually sent
int target = 0;
unsigned long lastRamp = 0, lastPrint = 0;

int btnPrev = HIGH;
unsigned long btnEdge = 0;

bool buttonPressed() {
  int r = digitalRead(BTN_PIN);
  bool p = false;
  if (r != btnPrev && millis() - btnEdge > 40) {
    btnEdge = millis();
    if (r == LOW) p = true;
    btnPrev = r;
  }
  return p;
}

void setup() {
  pinMode(MOTOR_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  Serial.println("Motor controller. Button = E-STOP / re-arm");
}

void loop() {
  unsigned long now = millis();
  int percent = map(analogRead(POT_PIN), 0, 1023, 0, 100);

  if (buttonPressed()) {
    if (state == RUN) {
      state = ESTOP;
      actual = 0;
      analogWrite(MOTOR_PIN, 0);              // immediate stop, no ramp
      Serial.println("!!! EMERGENCY STOP !!!");
    } else if (state == ESTOP) {
      state = WAIT_ZERO;
      Serial.println("Re-armed: turn knob to 0 to restart");
    }
  }

  if (state == WAIT_ZERO && percent < 3) {
    state = RUN;
    Serial.println("Knob at zero - RUN enabled");
  }

  if (state == RUN) {
    target = (percent < 15) ? 0 : map(percent, 0, 100, 0, 255);
  } else {
    target = 0;
    actual = 0;
  }

  if (now - lastRamp >= 20) {
    lastRamp = now;
    if (actual < target) actual = min(actual + 2, target);
    else if (actual > target) actual = max(actual - 2, target);
    analogWrite(MOTOR_PIN, actual);
  }

  if (state == ESTOP) digitalWrite(LED_PIN, (now / 100) % 2);
  else if (state == WAIT_ZERO) digitalWrite(LED_PIN, (now / 500) % 2);
  else digitalWrite(LED_PIN, actual > 0 ? HIGH : LOW);

  if (now - lastPrint >= 500) {
    lastPrint = now;
    Serial.print("knob=");
    Serial.print(percent);
    Serial.print("%  target=");
    Serial.print(target);
    Serial.print("  pwm=");
    Serial.print(actual);
    Serial.print("  state=");
    Serial.println(NAMES[state]);
  }
}
```

## Assessment Questions

1. Calculate how long the motor takes to ramp from 0 to 255 PWM with a step of 2 every 20 ms.
2. Explain the job of each part: (a) NPN transistor, (b) 1 kΩ base resistor, (c) flyback diode. What could happen if the diode were missing?
3. Why does the E-STOP set `actual = 0` directly instead of letting the ramp slow the motor down?
4. Describe a real-life accident that the "knob must return to zero" interlock prevents.
5. Add a **stall detector**: if the target has been above 50 % for 3 s but the motor is "not turning" (simulate with a second button on D3), stop the motor and enter a new state `STALL`. Draw the updated state diagram.
