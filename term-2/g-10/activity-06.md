# Activity 06: Multi-Mode LED Effects Engine

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-06`

## Problem Statement

Create a lighting "effects engine" for two PWM LEDs that can play four animations:

| Mode | Effect |
|---|---|
| 0 | **Breathe**: both LEDs fade in and out smoothly (sine wave) |
| 1 | **Heartbeat**: two quick pulses, then a pause (lub-dub) |
| 2 | **Candle**: random, flickering brightness like a flame |
| 3 | **Police**: LED 1 and LED 2 flash alternately, 3 flashes each |

1. A **short press** of the button switches to the next mode.
2. A **long press** (≥ 1 s) **freezes** or unfreezes the animation.
3. The potentiometer sets the **speed** from 0.25× to 4× using an exponential scale, so the middle position is 1×.
4. All animations must be computed from a time variable. No `delay()` is allowed.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 2 | LEDs + 2 × 1 kΩ resistors |
| 1 | Potentiometer |
| 1 | Pushbutton |

## Solution

* **Animation time:** instead of `millis()`, each effect uses `animTime`. Every loop, `animTime += elapsed × speed`. This makes the speed knob and the *freeze* function easy: freezing simply stops adding to `animTime`.
* **Exponential speed:** `speed = pow(2, (pot − 512) / 256.0)` gives 2⁻² = 0.25 at one end, 2⁰ = 1 in the middle, and 2² = 4 at the other end.
* **Breathe:** `(1 − cos(2πt/T)) / 2` goes smoothly from 0 to 1 and back.
* **Heartbeat:** the 1200 ms period is divided into time windows. Pulses are on at 0–120 ms and 250–370 ms.
* **Candle:** a "random walk". Every 40 ms, the brightness moves a random step towards a random target.
* **Police:** a 1200 ms period split into 12 slots of 100 ms. Slots 0–5 flash LED 1 on even slots, and slots 6–11 flash LED 2.
* **Button:** release under 1 s = next mode; held 1 s = toggle freeze (once per hold).

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LED 1 | Anode | **D9** (PWM) |
| LED 2 | Anode | **D10** (PWM) |
| LED cathodes | — | 1 kΩ → GND rail |
| Potentiometer | T1 / Wiper / T2 | GND / **A0** / 5V |
| Pushbutton | T1 / T2 | GND / **D2** |

```
D9  ─► LED1 ─[1k]─ GND          A0 ◄─ POT wiper (5V / GND ends)
D10 ─► LED2 ─[1k]─ GND          D2 ─► BUTTON ── GND
```

## Code

```cpp
// Activity 06 - Multi-Mode LED Effects Engine
const int LED1 = 9;
const int LED2 = 10;
const int POT_PIN = A0;
const int BTN_PIN = 2;
const int NUM_MODES = 4;
const char* MODE_NAMES[NUM_MODES] = {"Breathe", "Heartbeat", "Candle", "Police"};

int mode = 0;
bool frozen = false;
float animTime = 0;               // "virtual" milliseconds
unsigned long lastLoop = 0;

// button
int btnStable = HIGH, btnRaw = HIGH;
unsigned long btnChanged = 0, btnPressedAt = 0;
bool longDone = false;

// candle state
int candle = 150, candleTarget = 150;
unsigned long lastCandle = 0;

void handleButton() {
  int r = digitalRead(BTN_PIN);
  unsigned long now = millis();
  if (r != btnRaw) { btnRaw = r; btnChanged = now; }
  if (now - btnChanged > 20 && r != btnStable) {
    btnStable = r;
    if (r == LOW) {
      btnPressedAt = now;
      longDone = false;
    } else if (!longDone) {                       // short press released
      mode = (mode + 1) % NUM_MODES;
      animTime = 0;
      Serial.print("Mode -> ");
      Serial.println(MODE_NAMES[mode]);
    }
  }
  if (btnStable == LOW && !longDone && now - btnPressedAt >= 1000) {
    longDone = true;
    frozen = !frozen;
    Serial.println(frozen ? "FROZEN" : "RUNNING");
  }
}

void breathe(unsigned long t) {
  float x = (1.0 - cos(2.0 * PI * (t % 3000) / 3000.0)) / 2.0;
  int b = (int)(x * 255);
  analogWrite(LED1, b);
  analogWrite(LED2, b);
}

void heartbeat(unsigned long t) {
  unsigned long p = t % 1200;
  bool on = (p < 120) || (p >= 250 && p < 370);
  analogWrite(LED1, on ? 255 : 10);
  analogWrite(LED2, on ? 255 : 10);
}

void candleEffect(unsigned long t) {
  if (t - lastCandle >= 40 || t < lastCandle) {
    lastCandle = t;
    if (abs(candle - candleTarget) < 8) candleTarget = random(60, 256);
    candle += (candleTarget > candle) ? random(2, 12) : -random(2, 12);
    candle = constrain(candle, 0, 255);
  }
  analogWrite(LED1, candle);
  analogWrite(LED2, constrain(candle - 40, 0, 255));
}

void police(unsigned long t) {
  int slot = (t % 1200) / 100;          // 0..11
  bool on = (slot % 2 == 0);
  analogWrite(LED1, (slot < 6 && on) ? 255 : 0);
  analogWrite(LED2, (slot >= 6 && on) ? 255 : 0);
}

void setup() {
  pinMode(LED1, OUTPUT);
  pinMode(LED2, OUTPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  randomSeed(analogRead(A3));
  Serial.println("Effects engine. Short press = next mode, hold 1s = freeze.");
  Serial.print("Mode -> ");
  Serial.println(MODE_NAMES[mode]);
  lastLoop = millis();
}

void loop() {
  unsigned long now = millis();
  unsigned long dt = now - lastLoop;
  lastLoop = now;

  handleButton();

  float speed = pow(2.0, (analogRead(POT_PIN) - 512) / 256.0);   // 0.25x .. 4x
  if (!frozen) animTime += dt * speed;
  unsigned long t = (unsigned long)animTime;

  if (mode == 0) breathe(t);
  else if (mode == 1) heartbeat(t);
  else if (mode == 2) candleEffect(t);
  else police(t);
}
```

## Assessment Questions

1. Calculate the speed factor when the potentiometer reads 0, 256, 512, 768 and 1023.
2. Why does the program use `animTime` instead of `millis()` directly? Explain how this makes "freeze" possible.
3. Sketch a graph of brightness vs. time for one 1200 ms heartbeat period.
4. What is a "random walk"? Why does it look more like a real flame than `analogWrite(LED1, random(256))` every 40 ms?
5. Add a fifth mode, **"SOS strobe"**, that flashes both LEDs in the Morse pattern for SOS. Describe how you would compute the on/off state from `t`.
