# Activity 24: Serial Smart Window Blinds (Manual / Auto / Schedule)

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-24`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Motorised window blinds are driven by a servo: 0° = fully **closed**, 180° = fully **open**. A light sensor measures the sunlight outside.

| Key | Action |
|---|---|
| `o` / `c` | MANUAL: fully open / fully closed |
| `0`–`9` | MANUAL: open to 0 %, 11 % … 100 % (`9` = 100 %) |
| `a` | AUTO mode: blinds follow the light (see rules) |
| `p` | PRIVACY: close and ignore AUTO until `p` is sent again |
| `s` | status |
| `?` | help |

**AUTO rules:**

1. **Bright sun** (light > 800): close to 30 % to stop glare.
2. **Normal day** (300–800): open fully.
3. **Dark** (< 300): close fully (night).
4. A new position is only chosen after the light has stayed in the new zone for **3 s**, so a passing cloud does not move the blinds.

**Other rules:**

* The servo moves slowly (30 °/s), like a real motor.
* The LED shows the mode: ON = AUTO, slow blink = PRIVACY, OFF = MANUAL. The brightness (PWM) shows how open the blinds are.
* Any manual command (`o`, `c`, digits) switches AUTO off. PRIVACY overrides everything.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Micro Servo |
| 1 | Photoresistor (LDR) + 1 kΩ resistor |
| 1 | LED + 1 kΩ resistor |

## Solution

* **Zones:** `zoneOf(light)` returns 0 (dark), 1 (normal), 2 (bright).
* **3-second confirmation:** `candidate` and `candidateSince`. If the current zone differs from `candidate`, restart the timer. When the same zone has lasted 3 s, accept it as `zone` and set the target.
* **Servo motion:** `pos` moves towards `target` by 0.6° every 20 ms (30 °/s).
* **Priority:** PRIVACY > MANUAL command > AUTO.
* **LED on D5:** the Servo library disables PWM on D9 and D10, so the LED uses D5 for `analogWrite`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Micro Servo | Signal | **D9** |
| Micro Servo | Power / Ground | 5V rail / GND rail |
| LDR | T1 / T2 | 5V / **A0** (+ 1 kΩ to GND) |
| LED | Anode | **D5** (PWM) (cathode → 1 kΩ → GND) |

```
D9 ─► SERVO signal (power from rails)
5V ─[LDR]─┬─► A0
          └─[1k]─ GND
D5 ─► LED ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 24 - Serial Smart Window Blinds
#include <Servo.h>

const int SERVO_PIN = 9;
const int LDR_PIN = A0;
const int LED_PIN = 5;
const char* ZONES[3] = {"DARK", "NORMAL", "BRIGHT SUN"};
const int ZONE_TARGET[3] = {0, 180, 54};       // 0%, 100%, 30%

Servo blinds;
const int MANUAL = 0, AUTO = 1, PRIVACY = 2;
const char* MODES[3] = {"MANUAL", "AUTO", "PRIVACY"};
int mode = AUTO;
float pos = 90;
int target = 90;
int zone = -1;
int candidate = -1;
unsigned long candidateSince = 0, lastMove = 0, lastPrint = 0;

int zoneOf(int light) {
  if (light < 300) return 0;
  if (light > 800) return 2;
  return 1;
}

int percent() {
  return (int)(pos * 100 / 180 + 0.5);
}

void status() {
  Serial.print("Mode ");
  Serial.print(MODES[mode]);
  Serial.print(" | light ");
  Serial.print(analogRead(LDR_PIN));
  Serial.print(" (");
  Serial.print(zone >= 0 ? ZONES[zone] : "?");
  Serial.print(") | blinds ");
  Serial.print(percent());
  Serial.print("% -> ");
  Serial.print(target * 100 / 180);
  Serial.println("%");
}

void manual(int t) {
  if (mode == PRIVACY) { Serial.println("PRIVACY is on - send p first"); return; }
  mode = MANUAL;
  target = t;
  status();
}

void handle(char c) {
  if (c == 'o') manual(180);
  else if (c == 'c') manual(0);
  else if (c >= '0' && c <= '9') manual((c - '0') * 20);
  else if (c == 'a') {
    if (mode == PRIVACY) { Serial.println("PRIVACY is on - send p first"); return; }
    mode = AUTO;
    zone = -1;                       // force a fresh decision
    status();
  }
  else if (c == 'p') {
    if (mode == PRIVACY) { mode = MANUAL; Serial.println("Privacy OFF (manual)"); }
    else { mode = PRIVACY; target = 0; Serial.println("Privacy ON - blinds closed"); }
  }
  else if (c == 's') status();
  else if (c == '?' || c == 'h') Serial.println("o open | c close | 0-9 position | a auto | p privacy | s status");
  else if (c != '\n' && c != '\r' && c != ' ') { Serial.print("Unknown '"); Serial.print(c); Serial.println("'"); }
}

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  blinds.attach(SERVO_PIN);
  blinds.write((int)pos);
  Serial.println("Smart blinds ready (AUTO). Press ? for help.");
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();

  // AUTO decision with 3 s confirmation
  int z = zoneOf(analogRead(LDR_PIN));
  if (z != candidate) { candidate = z; candidateSince = now; }
  if (mode == AUTO && (zone == -1 || (candidate != zone && now - candidateSince >= 3000))) {
    zone = candidate;
    target = ZONE_TARGET[zone];
    Serial.print("AUTO: ");
    Serial.print(ZONES[zone]);
    Serial.print(" -> blinds ");
    Serial.print(target * 100 / 180);
    Serial.println("%");
  }

  // slow servo motion
  if (now - lastMove >= 20) {
    lastMove = now;
    float d = target - pos;
    if (d > 0.6) d = 0.6;
    if (d < -0.6) d = -0.6;
    pos += d;
    blinds.write((int)(pos + 0.5));
  }

  // LED: mode + opening
  int pwm = map(percent(), 0, 100, 20, 255);
  if (mode == MANUAL) pwm = 0;
  else if (mode == PRIVACY) pwm = ((now / 600) % 2) ? 40 : 0;
  analogWrite(LED_PIN, pwm);
}
```

## Assessment Questions

1. Right after the simulation starts (AUTO mode), what do the blinds do at normal light?
2. Make the light very bright. Do the blinds move immediately? What position do they go to? What happens if you make it dark?
3. Send `5`. What happens to the blinds and the LED? Does AUTO still react to light changes?
4. Send `p`, then `o`. What happens? How do you leave privacy mode?
5. If the LDR's 1 kΩ resistor is removed, which zone do you think the blinds will always think it is? What if the servo's power wire is disconnected?
