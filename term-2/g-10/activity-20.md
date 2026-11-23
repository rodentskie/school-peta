# Activity 20: LCD Min/Max Thermometer with Trend Arrows

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-20`

## Problem Statement

Build a weather-station style thermometer with an I2C LCD.

1. Read the **TMP36** every 250 ms and smooth it with an **exponential moving average** (`avg = avg + 0.1 × (new − avg)`).
2. Line 1: current temperature with 1 decimal and a **trend arrow** (custom LCD characters ↑ ↓ or =). The trend compares the temperature now with the value **5 seconds ago**, with a ±0.3 °C dead-band.
3. Line 2: the **minimum** and **maximum** since the last reset, e.g. `Lo 18.2 Hi 31.0`.
4. **Short press** of the button: reset min/max to the current temperature. **Long press** (≥ 1 s): switch °C ↔ °F.
5. A warning LED lights above **30 °C** and turns off below **29 °C** (hysteresis).

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Temperature Sensor [TMP36] |
| 1 | LCD 16 × 2 (I2C) |
| 1 | Pushbutton |
| 1 | LED + 1 kΩ resistor |

## Solution

* **Custom characters:** an LCD character is a 5×8 dot matrix. Each row is one byte (5 low bits). Arrow up and arrow down are defined in arrays and loaded with `lcd.createChar(0, up)` and `lcd.createChar(1, down)`, then shown with `lcd.write(byte(0))`.
* **Trend history:** a ring buffer `hist[20]` stores one value every 250 ms. 20 × 250 ms = 5 s, so the oldest entry is the value from 5 s ago.
* **Min/Max:** updated on each sample. A reset sets both to the current average.
* **Unit conversion** is only done for display. All logic stays in °C.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| TMP36 | Vs / Vout / GND | 5V / **A0** / GND |
| Pushbutton | T1 / T2 | GND / **D2** |
| LED | Anode | **D8** (cathode → 1 kΩ → GND) |
| LCD (I2C) | GND / VCC | GND rail / 5V rail |
| LCD (I2C) | SDA / SCL | **A4** / **A5** |

```
5V ─ TMP36 ─ GND, Vout ─► A0       D2 ─► BUTTON ── GND
D8 ─► LED ─[1k]─ GND               LCD: SDA─A4 SCL─A5 VCC─5V GND─GND
```

## Code

```cpp
// Activity 20 - LCD Min/Max Thermometer with Trend
#include <Adafruit_LiquidCrystal.h>

Adafruit_LiquidCrystal lcd(0);

const int TMP_PIN = A0;
const int BTN_PIN = 2;
const int LED_PIN = 8;
const int HIST = 20;                    // 20 x 250 ms = 5 s

byte arrowUp[8]   = {0b00100, 0b01110, 0b10101, 0b00100, 0b00100, 0b00100, 0b00100, 0b00000};
byte arrowDown[8] = {0b00100, 0b00100, 0b00100, 0b00100, 0b10101, 0b01110, 0b00100, 0b00000};

float avg = 0, tMin = 0, tMax = 0;
float hist[HIST];
int hIdx = 0;
bool showF = false;
bool warning = false;
unsigned long lastSample = 0;

int btnStable = HIGH, btnRaw = HIGH;
unsigned long btnChanged = 0, btnPressedAt = 0;
bool longDone = false;

float readC() {
  float mv = analogRead(TMP_PIN) * 5000.0 / 1024.0;
  return (mv - 500.0) / 10.0;
}

float shown(float c) {
  return showF ? c * 9.0 / 5.0 + 32.0 : c;
}

void printTemp(float c) {
  float v = shown(c);
  if (v >= 0 && v < 10) lcd.print(' ');
  lcd.print(v, 1);
}

void handleButton() {
  int r = digitalRead(BTN_PIN);
  unsigned long now = millis();
  if (r != btnRaw) { btnRaw = r; btnChanged = now; }
  if (now - btnChanged > 20 && r != btnStable) {
    btnStable = r;
    if (r == LOW) { btnPressedAt = now; longDone = false; }
    else if (!longDone) {
      tMin = tMax = avg;
      Serial.println("Min/Max reset");
    }
  }
  if (btnStable == LOW && !longDone && now - btnPressedAt >= 1000) {
    longDone = true;
    showF = !showF;
    lcd.clear();
  }
}

void setup() {
  pinMode(BTN_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  lcd.begin(16, 2);
  lcd.setBacklight(1);
  lcd.createChar(0, arrowUp);
  lcd.createChar(1, arrowDown);
  avg = readC();
  tMin = tMax = avg;
  for (int i = 0; i < HIST; i++) hist[i] = avg;
}

void loop() {
  unsigned long now = millis();
  handleButton();

  if (now - lastSample < 250) return;
  lastSample = now;

  avg += 0.1 * (readC() - avg);
  if (avg < tMin) tMin = avg;
  if (avg > tMax) tMax = avg;

  float old = hist[hIdx];               // value from 5 s ago
  hist[hIdx] = avg;
  hIdx = (hIdx + 1) % HIST;
  float diff = avg - old;

  if (!warning && avg > 30.0) warning = true;
  if (warning && avg < 29.0) warning = false;
  digitalWrite(LED_PIN, warning ? HIGH : LOW);

  // line 1: current + trend
  lcd.setCursor(0, 0);
  lcd.print("Temp ");
  printTemp(avg);
  lcd.print(showF ? "F " : "C ");
  if (diff > 0.3) lcd.write(byte(0));
  else if (diff < -0.3) lcd.write(byte(1));
  else lcd.print('=');
  lcd.print(warning ? " HOT" : "    ");

  // line 2: min / max
  lcd.setCursor(0, 1);
  lcd.print("Lo");
  printTemp(tMin);
  lcd.print(" Hi");
  printTemp(tMax);
  lcd.print("  ");

  static unsigned long lastLog = 0;
  if (now - lastLog >= 1000) {
    lastLog = now;
    Serial.print("T=");
    Serial.print(avg, 2);
    Serial.print(" min=");
    Serial.print(tMin, 1);
    Serial.print(" max=");
    Serial.print(tMax, 1);
    Serial.print(" trend=");
    Serial.println(diff, 2);
  }
}
```

## Assessment Questions

1. What does the LCD show on line 1 and on line 2?
2. Raise the temperature slider quickly. Which arrow appears? What is shown when the temperature stays the same?
3. What happens to the LED when the temperature goes above 30 °C? Does it switch off exactly at 30 °C when cooling down?
4. What does a short press of the button do? What does a long press do?
5. If the wire from the sensor's middle pin (Vout) to A0 is disconnected, what do you think the LCD will show?
