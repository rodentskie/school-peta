# Activity 10: Servo Position Memory with EEPROM

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-10`

## Problem Statement

A camera mount uses a servo motor. The operator wants to **save** favorite positions and **recall** them later, even after the power is switched off.

1. The potentiometer controls the servo angle (0°–180°) with **exponential smoothing** (`angle = angle + 0.2 × (target − angle)`), so the servo does not jitter.
2. **SAVE button (D2):** stores the current angle into the next of **3 memory slots** (slot 0 → 1 → 2 → 0 …) in **EEPROM**. The LED blinks *slot number + 1* times to confirm.
3. **RECALL button (D3):** moves the servo to the next saved slot at a limited speed of **60° per second** (slew-rate limit).
4. After a recall, the knob is **ignored** until it is turned by more than 10°. This stops the servo jumping back to the knob position.
5. On start-up, read the saved slots from EEPROM and print them. If a value is invalid (above 180), use 90.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Micro Servo |
| 1 | Potentiometer |
| 2 | Pushbuttons |
| 1 | LED + 1 kΩ resistor |

## Solution

* **EEPROM** keeps data without power. `EEPROM.read(addr)` and `EEPROM.update(addr, value)` read and write one byte. `update` only writes when the value is different, which extends the EEPROM's life. Slots use addresses 0–2, and address 3 stores the next slot index.
* **Modes:**
  * `KNOB` mode: `target` = knob angle.
  * `RECALL` mode: `target` = saved angle. Switch back to `KNOB` when the knob moves more than 10° from where it was at the moment of the recall.
* **Slew-rate limit:** in RECALL mode, the position changes by at most 1.2° every 20 ms (= 60°/s).
* **Blink confirmation** is non-blocking. `blinksLeft` counts LED toggles, handled every 150 ms.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Potentiometer | T1 / Wiper / T2 | GND / **A0** / 5V |
| SAVE button | T1 / T2 | GND / **D2** |
| RECALL button | T1 / T2 | GND / **D3** |
| LED | Anode | **D7** (cathode → 1 kΩ → GND) |
| Micro Servo | Signal | **D9** |
| Micro Servo | Power / Ground | 5V rail / GND rail |

```
A0 ◄─ POT wiper        D2 ─► SAVE   ── GND      D9 ─► SERVO signal
D7 ─► LED ─[1k]─ GND   D3 ─► RECALL ── GND      5V/GND ─► SERVO power
```

## Code

```cpp
// Activity 10 - Servo Position Memory with EEPROM
#include <Servo.h>
#include <EEPROM.h>

const int POT_PIN = A0;
const int SAVE_PIN = 2;
const int RECALL_PIN = 3;
const int LED_PIN = 7;
const int SERVO_PIN = 9;
const int SLOTS = 3;
const int ADDR_NEXT = 3;

Servo servo;
int saved[SLOTS];
int nextSave = 0;
int nextRecall = 0;

bool recallMode = false;
int knobAtRecall = 0;
float angle = 90;
float target = 90;

int blinksLeft = 0;
unsigned long lastBlink = 0, lastStep = 0;

int prevSave = HIGH, prevRecall = HIGH;
unsigned long lastSaveEdge = 0, lastRecallEdge = 0;

int knobAngle() {
  return map(analogRead(POT_PIN), 0, 1023, 0, 180);
}

void startBlinks(int n) {
  blinksLeft = n * 2;            // on + off per blink
  lastBlink = millis();
  digitalWrite(LED_PIN, LOW);
}

void saveSlot() {
  int a = (int)(angle + 0.5);
  saved[nextSave] = a;
  EEPROM.update(nextSave, a);
  Serial.print("Saved ");
  Serial.print(a);
  Serial.print(" deg in slot ");
  Serial.println(nextSave);
  startBlinks(nextSave + 1);
  nextSave = (nextSave + 1) % SLOTS;
  EEPROM.update(ADDR_NEXT, nextSave);
}

void recallSlot() {
  target = saved[nextRecall];
  recallMode = true;
  knobAtRecall = knobAngle();
  Serial.print("Recall slot ");
  Serial.print(nextRecall);
  Serial.print(" -> ");
  Serial.print(saved[nextRecall]);
  Serial.println(" deg");
  nextRecall = (nextRecall + 1) % SLOTS;
}

// true on a debounced HIGH->LOW edge
bool fell(int pin, int &prev, unsigned long &lastEdge) {
  int r = digitalRead(pin);
  bool edge = false;
  if (r != prev && millis() - lastEdge > 50) {
    lastEdge = millis();
    if (r == LOW) edge = true;
    prev = r;
  }
  return edge;
}

void setup() {
  pinMode(SAVE_PIN, INPUT_PULLUP);
  pinMode(RECALL_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  servo.attach(SERVO_PIN);

  for (int i = 0; i < SLOTS; i++) {
    int v = EEPROM.read(i);
    saved[i] = (v > 180) ? 90 : v;       // erased EEPROM reads 255
  }
  nextSave = EEPROM.read(ADDR_NEXT);
  if (nextSave >= SLOTS) nextSave = 0;

  Serial.print("Memory: ");
  for (int i = 0; i < SLOTS; i++) {
    Serial.print("[");
    Serial.print(i);
    Serial.print("]=");
    Serial.print(saved[i]);
    Serial.print("  ");
  }
  Serial.println();
  angle = target = knobAngle();
}

void loop() {
  unsigned long now = millis();

  if (fell(SAVE_PIN, prevSave, lastSaveEdge)) saveSlot();
  if (fell(RECALL_PIN, prevRecall, lastRecallEdge)) recallSlot();

  int k = knobAngle();
  if (recallMode && abs(k - knobAtRecall) > 10) {
    recallMode = false;
    Serial.println("Knob moved: manual control");
  }

  if (now - lastStep >= 20) {
    lastStep = now;
    if (recallMode) {
      float diff = target - angle;               // slew limited 1.2 deg / 20 ms
      if (diff > 1.2) diff = 1.2;
      if (diff < -1.2) diff = -1.2;
      angle += diff;
    } else {
      target = k;
      angle += 0.2 * (target - angle);           // exponential smoothing
    }
    servo.write((int)(angle + 0.5));
  }

  if (blinksLeft > 0 && now - lastBlink >= 150) {
    lastBlink = now;
    blinksLeft--;
    digitalWrite(LED_PIN, (blinksLeft % 2) ? HIGH : LOW);
  }
}
```

## Assessment Questions

1. What is the difference between `EEPROM.write()` and `EEPROM.update()`? Why does it matter that EEPROM cells can only be written about 100,000 times?
2. A fresh (erased) EEPROM byte reads **255**. Explain how the program protects itself against this value.
3. The slew limit is 1.2° per 20 ms. How long does the servo take to move from 10° to 150° in RECALL mode?
4. Compare **exponential smoothing** with the **slew-rate limit**. Describe how the servo motion looks different for each one when the target jumps from 0° to 180°.
5. Add a fourth action: holding **both** buttons for 2 seconds clears all slots to 90°. Write the code for this feature.
