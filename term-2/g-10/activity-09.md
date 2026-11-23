# Activity 09: Temperature Monitor with Alarm Levels and Mute

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-09`

## Problem Statement

Build a temperature monitor for a server room using the **TMP36** sensor.

1. Convert the reading to °C: `mV = raw × 5000 / 1024`, `°C = (mV − 500) / 10`. Average 10 readings.
2. Show the level with three LEDs:
   * **Green**: below 25 °C (normal)
   * **Yellow**: 25 °C to below 35 °C (warm)
   * **Red** + buzzer alarm: 35 °C or more (hot). The alarm beeps 200 ms on / 200 ms off.
3. A **short press** switches the display unit between °C and °F (`°F = °C × 9/5 + 32`).
4. A **long press** (≥ 1 s) **mutes** the alarm for 30 s. The red LED keeps blinking while muted.
5. Print the temperature every second, without using `delay()`.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Temperature Sensor [TMP36] |
| 3 | LEDs (green, yellow, red) + 3 × 1 kΩ resistors |
| 1 | Pushbutton |
| 1 | Piezo buzzer |

## Solution

* `readTempC()` averages 10 readings and applies the TMP36 formula. The sensor gives 10 mV per °C with a 500 mV offset at 0 °C.
* `level` is 0, 1 or 2 depending on the temperature. Only the LED for the current level is lit.
* **Alarm pattern:** `(millis() / 200) % 2` toggles every 200 ms.
* **Mute:** save `muteUntil = millis() + 30000`. While `millis() < muteUntil` the buzzer stays silent, but the red LED keeps blinking. The mute is cancelled automatically if the temperature returns to normal.
* The button uses the short/long press pattern: release before 1 s = unit toggle, holding 1 s = mute.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| TMP36 | Vs (+Vs) | 5V rail |
| TMP36 | Vout | **A0** |
| TMP36 | GND | GND rail |
| Green LED | Anode | **D5** |
| Yellow LED | Anode | **D6** |
| Red LED | Anode | **D7** |
| LED cathodes | — | 1 kΩ → GND rail |
| Pushbutton | T1 / T2 | GND / **D2** |
| Piezo | + / − | **D9** / GND |

```
5V ─ TMP36(Vs)  TMP36(Vout) ─► A0  TMP36(GND) ─ GND
D5 ─► GREEN ─[1k]─ GND    D2 ─► BUTTON ── GND
D6 ─► YELLOW─[1k]─ GND    D9 ─► PIEZO ─── GND
D7 ─► RED   ─[1k]─ GND
```

## Code

```cpp
// Activity 09 - Temperature Monitor with Alarm Levels and Mute
const int TMP_PIN = A0;
const int GREEN_PIN = 5;
const int YELLOW_PIN = 6;
const int RED_PIN = 7;
const int BTN_PIN = 2;
const int BUZZER_PIN = 9;

const float WARM_C = 25.0;
const float HOT_C = 35.0;

bool showF = false;
unsigned long muteUntil = 0;
unsigned long lastPrint = 0;

int btnStable = HIGH, btnRaw = HIGH;
unsigned long btnChanged = 0, btnPressedAt = 0;
bool longDone = false;

float readTempC() {
  long sum = 0;
  for (int i = 0; i < 10; i++) sum += analogRead(TMP_PIN);
  float raw = sum / 10.0;
  float mv = raw * 5000.0 / 1024.0;
  return (mv - 500.0) / 10.0;
}

void handleButton() {
  int r = digitalRead(BTN_PIN);
  unsigned long now = millis();
  if (r != btnRaw) { btnRaw = r; btnChanged = now; }
  if (now - btnChanged > 20 && r != btnStable) {
    btnStable = r;
    if (r == LOW) { btnPressedAt = now; longDone = false; }
    else if (!longDone) {
      showF = !showF;
      Serial.println(showF ? "Unit: Fahrenheit" : "Unit: Celsius");
    }
  }
  if (btnStable == LOW && !longDone && now - btnPressedAt >= 1000) {
    longDone = true;
    muteUntil = now + 30000;
    Serial.println("Alarm MUTED for 30 s");
  }
}

void setup() {
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(YELLOW_PIN, OUTPUT);
  pinMode(RED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  Serial.println("Temperature monitor ready");
}

void loop() {
  unsigned long now = millis();
  handleButton();

  float c = readTempC();
  int level = (c >= HOT_C) ? 2 : (c >= WARM_C ? 1 : 0);
  if (level < 2) muteUntil = 0;                  // cancel mute when safe again

  bool blink = (now / 200) % 2;
  bool muted = (long)(muteUntil - now) > 0;

  digitalWrite(GREEN_PIN, level == 0 ? HIGH : LOW);
  digitalWrite(YELLOW_PIN, level == 1 ? HIGH : LOW);
  digitalWrite(RED_PIN, (level == 2 && blink) ? HIGH : LOW);

  if (level == 2 && blink && !muted) tone(BUZZER_PIN, 1800);
  else noTone(BUZZER_PIN);

  if (now - lastPrint >= 1000) {
    lastPrint = now;
    float shown = showF ? c * 9.0 / 5.0 + 32.0 : c;
    Serial.print("Temp: ");
    Serial.print(shown, 1);
    Serial.print(showF ? " F" : " C");
    Serial.print("  Level: ");
    Serial.print(level == 0 ? "NORMAL" : (level == 1 ? "WARM" : "HOT!"));
    if (muted) Serial.print("  (muted)");
    Serial.println();
  }
}
```

## Assessment Questions

1. The TMP36 outputs 750 mV. Calculate the temperature in °C and °F. What `analogRead()` value would you expect?
2. Why is `muteUntil` compared using `(long)(muteUntil - now) > 0` instead of `muteUntil > now`?
3. Explain why the mute is cancelled when the level drops below 2. What safety problem would happen if it were not?
4. Why is it better to light **only one** LED for the current level rather than a bar graph in this application?
5. Add **hysteresis** of 1 °C to the HOT level (alarm turns on at 35.0 °C but only off below 34.0 °C). Write the new code for calculating `level`.
