# Activity 05: LED Bar-Graph Voltmeter with Peak Hold

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-05`

## Problem Statement

Make a 5-LED bar-graph meter, like the level meter on a music player, that shows the voltage from a potentiometer.

1. Read the potentiometer on **A0** and smooth the value with the **average of 8 samples**.
2. Convert the reading to volts (0.00–5.00 V) and show it as a bar of 0–5 LEDs. Each LED represents 1 V.
3. Add a **peak-hold** dot. The highest level reached stays lit (blinking) for 1.5 s, then falls by one LED every 300 ms until it meets the bar again.
4. Print the voltage (2 decimals), bar level and peak level every 250 ms, without using `delay()`.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Potentiometer (10 kΩ) |
| 5 | LEDs + 5 × 1 kΩ resistors |

## Solution

1. **Smoothing:** `readAverage()` adds 8 `analogRead()` values and divides by 8 to reduce noise.
2. **Scaling:** `volts = raw * 5.0 / 1023.0` and `level = round(volts)`, clamped to 0–5.
3. **Peak logic:**
   * If `level > peak`: `peak = level`, and save `peakTime`.
   * Otherwise, after the 1.5 s hold, decrease `peak` by one every 300 ms (`lastDecay`), but never below `level`.
4. **Display:** LED *i* (1–5) is ON if `i <= level`. The peak LED (`i == peak`, `peak > level`) blinks with `(millis() / 100) % 2`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Potentiometer | Terminal 1 | GND rail |
| Potentiometer | Wiper | **A0** |
| Potentiometer | Terminal 2 | 5V rail |
| LED 1 (lowest) | Anode | **D3** |
| LED 2 | Anode | **D4** |
| LED 3 | Anode | **D5** |
| LED 4 | Anode | **D6** |
| LED 5 (highest) | Anode | **D7** |
| LED cathodes | — | 1 kΩ → GND rail |

```
5V ──┐
    [POT]──wiper──► A0         D3..D7 ─► LED1..LED5 ─[1k]─ GND
GND ─┘
```

## Code

```cpp
// Activity 05 - Bar-Graph Voltmeter with Peak Hold
const int POT_PIN = A0;
const int LED_PINS[5] = {3, 4, 5, 6, 7};

const unsigned long HOLD_MS = 1500;
const unsigned long DECAY_MS = 300;
const unsigned long PRINT_MS = 250;

int peak = 0;
unsigned long peakTime = 0;
unsigned long lastDecay = 0;
unsigned long lastPrint = 0;

int readAverage() {
  long sum = 0;
  for (int i = 0; i < 8; i++) sum += analogRead(POT_PIN);
  return sum / 8;
}

void setup() {
  for (int i = 0; i < 5; i++) pinMode(LED_PINS[i], OUTPUT);
  Serial.begin(9600);
  Serial.println("Bar-graph voltmeter with peak hold");
}

void loop() {
  unsigned long now = millis();
  int raw = readAverage();
  float volts = raw * 5.0 / 1023.0;
  int level = (int)(volts + 0.5);          // round to nearest volt
  if (level > 5) level = 5;

  // ---- peak hold & decay ----
  if (level >= peak) {
    peak = level;
    peakTime = now;
    lastDecay = now;
  } else if (now - peakTime > HOLD_MS && now - lastDecay > DECAY_MS) {
    peak--;
    lastDecay = now;
  }

  // ---- display ----
  bool blinkOn = (now / 100) % 2;
  for (int i = 1; i <= 5; i++) {
    bool on = (i <= level);
    if (i == peak && peak > level) on = blinkOn;
    digitalWrite(LED_PINS[i - 1], on ? HIGH : LOW);
  }

  // ---- report ----
  if (now - lastPrint >= PRINT_MS) {
    lastPrint = now;
    Serial.print("V=");
    Serial.print(volts, 2);
    Serial.print("  level=");
    Serial.print(level);
    Serial.print("  peak=");
    Serial.print(peak);
    Serial.print("  [");
    for (int i = 1; i <= 5; i++) Serial.print(i <= level ? '#' : (i == peak ? '|' : '.'));
    Serial.println("]");
  }
}
```

## Assessment Questions

1. The ADC returns a value from 0 to 1023. Calculate the voltage for raw readings of 205, 512 and 900.
2. Why do we add `0.5` before converting `volts` to `int`? What would the level be for 2.7 V with and without the `+ 0.5`?
3. Explain why averaging 8 samples makes the bar steadier. What is the disadvantage of averaging 100 samples instead?
4. Trace the value of `peak` if the knob goes quickly from 5 V to 0 V. How long does it take until the peak LED disappears?
5. Change the meter so that LED 5 also blinks as a warning whenever the voltage is above 4.5 V. Write the lines you would add.
