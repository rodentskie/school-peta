# Activity 21: Ultrasonic Range Finder with an LCD Bar Graph

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-21`

## Problem Statement

Make a digital measuring tape with an HC-SR04 and a 16×2 I2C LCD.

1. Line 1 shows the distance with 1 decimal, e.g. `Dist  123.4 cm`. If there is no echo, show `Out of range`.
2. Line 2 shows a **smooth bar graph**. The 16 character cells are divided into 5 columns each (**80 steps** total) using **5 custom characters**. The full bar = 200 cm (or 80 in).
3. Use a **median of 3** readings to remove false echoes.
4. **Short press:** switch units cm ↔ inches (1 in = 2.54 cm).
5. **Long press (≥ 1 s):** **HOLD**. Freeze the last value and show `HOLD` on line 1. A long press again releases it.
6. Update the display at most 5 times per second, without `lcd.clear()` (no flicker).

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Ultrasonic Distance Sensor (4-pin, HC-SR04) |
| 1 | LCD 16 × 2 (I2C) |
| 1 | Pushbutton |

## Solution

**Custom bar characters:** character *n* (1–5) has its first *n* pixel columns on, in all 8 rows. For example, character 3 has every row = `0b11100`. Character slots 1–5 are used (slot 0 is avoided because `lcd.print` treats byte 0 as the end of a string).

**Drawing the bar:**

```
steps = distance / fullScale * 80   (0..80)
full cells    = steps / 5           → write char 5
partial cell  = steps % 5           → write char 1..4
remaining     → spaces
```

**Median of 3:** sort three numbers with three compare-and-swap steps, and take the middle one.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| HC-SR04 | Power / Ground | 5V rail / GND rail |
| HC-SR04 | Trigger | **D7** |
| HC-SR04 | Echo | **D6** |
| Pushbutton | T1 / T2 | GND / **D2** |
| LCD (I2C) | GND / VCC | GND rail / 5V rail |
| LCD (I2C) | SDA / SCL | **A4** / **A5** |

```
HC-SR04: VCC─5V TRIG─D7 ECHO─D6 GND─GND
D2 ─► BUTTON ── GND
LCD: GND─GND VCC─5V SDA─A4 SCL─A5
```

## Code

```cpp
// Activity 21 - Ultrasonic Range Finder with LCD Bar Graph
#include <Adafruit_LiquidCrystal.h>

Adafruit_LiquidCrystal lcd(0);

const int TRIG_PIN = 7;
const int ECHO_PIN = 6;
const int BTN_PIN = 2;

bool inches = false;
bool hold = false;
float shownCm = -1;
unsigned long lastUpdate = 0;

int btnStable = HIGH, btnRaw = HIGH;
unsigned long btnChanged = 0, btnPressedAt = 0;
bool longDone = false;

void makeBarChars() {
  for (int n = 1; n <= 5; n++) {
    byte rowBits = 0;
    for (int c = 0; c < n; c++) rowBits |= (0b10000 >> c);
    byte ch[8];
    for (int r = 0; r < 8; r++) ch[r] = rowBits;
    lcd.createChar(n, ch);
  }
}

float measureCm() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  unsigned long us = pulseIn(ECHO_PIN, HIGH, 30000);
  if (us == 0) return -1;
  return us * 0.0343 / 2.0;
}

float median3(float a, float b, float c) {
  float t;
  if (a > b) { t = a; a = b; b = t; }
  if (b > c) { t = b; b = c; c = t; }
  if (a > b) { t = a; a = b; b = t; }
  return b;
}

void handleButton() {
  int r = digitalRead(BTN_PIN);
  unsigned long now = millis();
  if (r != btnRaw) { btnRaw = r; btnChanged = now; }
  if (now - btnChanged > 20 && r != btnStable) {
    btnStable = r;
    if (r == LOW) { btnPressedAt = now; longDone = false; }
    else if (!longDone) inches = !inches;
  }
  if (btnStable == LOW && !longDone && now - btnPressedAt >= 1000) {
    longDone = true;
    hold = !hold;
    Serial.println(hold ? "HOLD" : "LIVE");
  }
}

void drawBar(float cm) {
  int steps = 0;
  if (cm > 0) steps = (int)(cm / 200.0 * 80.0 + 0.5);
  if (steps > 80) steps = 80;
  lcd.setCursor(0, 1);
  int full = steps / 5;
  int part = steps % 5;
  for (int i = 0; i < 16; i++) {
    if (i < full) lcd.write(byte(5));
    else if (i == full && part > 0) lcd.write(byte(part));
    else lcd.print(' ');
  }
}

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  lcd.begin(16, 2);
  lcd.setBacklight(1);
  makeBarChars();
}

void loop() {
  unsigned long now = millis();
  handleButton();
  if (now - lastUpdate < 200) return;
  lastUpdate = now;

  if (!hold) {
    float a = measureCm();
    float b = measureCm();
    float c = measureCm();
    shownCm = median3(a, b, c);
  }

  lcd.setCursor(0, 0);
  if (shownCm < 0) {
    lcd.print("Out of range    ");
  } else {
    float v = inches ? shownCm / 2.54 : shownCm;
    lcd.print(hold ? "HOLD " : "Dist ");
    if (v < 100) lcd.print(' ');
    if (v < 10) lcd.print(' ');
    lcd.print(v, 1);
    lcd.print(inches ? " in  " : " cm  ");
  }
  drawBar(shownCm);

  Serial.print("distance_cm=");
  Serial.println(shownCm, 1);
}
```

## Assessment Questions

1. What happens to the bar on line 2 when the object moves farther away?
2. What changes on the LCD when you press the button quickly?
3. Hold the button for 1 second and then move the object. What happens? How do you go back?
4. What does the LCD show when the object is out of range?
5. If the Echo wire is disconnected, what will the LCD show? Why?
