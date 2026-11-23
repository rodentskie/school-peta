# Activity 04: 4-Bit Binary Counter with Debounce, Auto-Repeat and Long-Press Reset

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-04`

## Problem Statement

Build an up/down counter that shows its value (0–15) in **binary** on 4 LEDs.

1. The **UP** button adds 1 and the **DOWN** button subtracts 1. The value wraps around (15 + 1 → 0, 0 − 1 → 15).
2. Both buttons must be **debounced**, so one press changes the value exactly once.
3. **Holding** DOWN for more than 600 ms makes it auto-repeat every 200 ms.
4. **Holding** UP for 1.5 s resets the counter to 0. In this case the UP press must not also add 1.
5. Each time the value changes, print it in decimal, binary (4 digits) and hexadecimal, e.g. `9 = 0b1001 = 0x9`.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 4 | LEDs + 4 × 1 kΩ resistors |
| 2 | Pushbuttons |

## Solution

* Each button is stored in a `Button` structure (pin, stable level, last raw reading, time of last change, time pressed, flags). `updateButton()` debounces it and returns an **event code**:
  * `EV_PRESS`: the moment it is pressed.
  * `EV_RELEASE`: the moment it is released.
  * `EV_NONE`: nothing new.
* The UP button acts on **release**: a short press adds 1. While it is held, the code checks whether 1.5 s has passed and, if so, resets and marks `longFired` so the release does nothing.
* The DOWN button acts on **press**, then auto-repeats while held, using `nextRepeat`.
* `showValue()` uses bit masking: LED *i* is on when `(value >> i) & 1` is 1.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LED bit 0 (LSB) | Anode | **D4** |
| LED bit 1 | Anode | **D5** |
| LED bit 2 | Anode | **D6** |
| LED bit 3 (MSB) | Anode | **D7** |
| LED cathodes | — | 1 kΩ → GND rail |
| UP button | T1 / T2 | GND / **D2** |
| DOWN button | T1 / T2 | GND / **D3** |

```
D4 ─► LED b0 ─[1k]─ GND     D2 ─► UP   ─── GND
D5 ─► LED b1 ─[1k]─ GND     D3 ─► DOWN ─── GND
D6 ─► LED b2 ─[1k]─ GND
D7 ─► LED b3 ─[1k]─ GND
```

## Code

```cpp
// Activity 04 - 4-bit Binary Counter
// Tinkercad-compatible: the button update logic is a member function of
// Button, so the auto-prototype step never references Button before it exists.
const int LED_PINS[4] = {4, 5, 6, 7};   // bit0 .. bit3
const int UP_PIN = 2;
const int DOWN_PIN = 3;

const int EV_NONE = 0;
const int EV_PRESS = 1;
const int EV_RELEASE = 2;

struct Button {
  int pin;
  int stable;
  int lastRaw;
  unsigned long changedAt;
  unsigned long pressedAt;
  bool longFired;
  unsigned long nextRepeat;

  int update() {
    int raw = digitalRead(pin);
    unsigned long now = millis();
    if (raw != lastRaw) {
      lastRaw = raw;
      changedAt = now;
    }
    if (now - changedAt > 20 && raw != stable) {
      stable = raw;
      if (raw == LOW) {
        pressedAt = now;
        longFired = false;
        return EV_PRESS;
      }
      return EV_RELEASE;
    }
    return EV_NONE;
  }
};

Button upBtn = {UP_PIN, HIGH, HIGH, 0, 0, false, 0};
Button downBtn = {DOWN_PIN, HIGH, HIGH, 0, 0, false, 0};

int value = 0;

void printBinary(int v) {
  Serial.print("0b");
  for (int i = 3; i >= 0; i--) Serial.print((v >> i) & 1);
}

void showValue() {
  for (int i = 0; i < 4; i++) {
    digitalWrite(LED_PINS[i], ((value >> i) & 1) ? HIGH : LOW);
  }
  Serial.print(value);
  Serial.print(" = ");
  printBinary(value);
  Serial.print(" = 0x");
  Serial.println(value, HEX);
}

void changeBy(int d) {
  value = (value + d + 16) % 16;   // wrap-around in both directions
  showValue();
}

void setup() {
  for (int i = 0; i < 4; i++) pinMode(LED_PINS[i], OUTPUT);
  pinMode(UP_PIN, INPUT_PULLUP);
  pinMode(DOWN_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  Serial.println("Binary counter: UP=+1 (hold 1.5s = reset), DOWN=-1 (hold = repeat)");
  showValue();
}

void loop() {
  unsigned long now = millis();

  // ---- UP: act on release, long-press resets ----
  int ev = upBtn.update();
  if (upBtn.stable == LOW && !upBtn.longFired && now - upBtn.pressedAt >= 1500) {
    upBtn.longFired = true;
    value = 0;
    Serial.println("RESET");
    showValue();
  }
  if (ev == EV_RELEASE && !upBtn.longFired) changeBy(+1);

  // ---- DOWN: act on press, then auto-repeat ----
  ev = downBtn.update();
  if (ev == EV_PRESS) {
    changeBy(-1);
    downBtn.nextRepeat = now + 600;
  }
  if (downBtn.stable == LOW && (long)(now - downBtn.nextRepeat) >= 0) {
    changeBy(-1);
    downBtn.nextRepeat = now + 200;
  }
}
```

## Assessment Questions

1. What happens to the 4 LEDs each time you press and release the UP button?
2. What happens when all 4 LEDs are on and you press UP once more?
3. What happens when you **hold** the DOWN button for a few seconds? What happens when you hold UP for more than 1.5 seconds?
4. What will happen to the LED on D6 if its resistor is removed? Can you still read the number correctly?
5. If the LEDs on D4 and D7 are swapped on the breadboard, what will the LEDs show after the first UP press?
