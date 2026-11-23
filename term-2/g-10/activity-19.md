# Activity 19: LCD Digital Clock with Alarm and Button Setting

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-19`

## Problem Statement

Build a digital alarm clock on a 16×2 I2C LCD, using only `millis()` for time keeping (no RTC module).

1. Line 1 shows the time `HH:MM:SS` (24-hour). Line 2 shows the alarm, e.g. `Alarm 07:00 ON`.
2. The **MODE** button (D2) cycles the modes: RUN → SET HOUR → SET MINUTE → SET ALARM HOUR → SET ALARM MINUTE → RUN. The field being edited **blinks** (every 400 ms).
3. The **INC** button (D3) increases the blinking field. Holding it **auto-repeats** (after 500 ms, every 150 ms). Hours wrap at 24, minutes at 60.
4. In RUN mode, a **long press** of INC (≥ 1 s) turns the alarm ON or OFF.
5. When the time equals the alarm time (and the alarm is ON), play a beeping pattern for up to 20 s. Any button stops it.
6. For the demo, the clock starts at **06:59:50** with the alarm at **07:00**, so the alarm rings after about 10 s.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | LCD 16 × 2 (I2C) |
| 2 | Pushbuttons (MODE, INC) |
| 1 | Piezo buzzer |

## Solution

* **Time keeping:** store `secondsOfDay` (0–86399). Add 1 every time `millis() − lastTick ≥ 1000`, using `lastTick += 1000`. Adding exactly 1000 to `lastTick` (not `lastTick = millis()`) prevents slow drift. Hours = `s / 3600`, minutes = `(s / 60) % 60`, seconds = `s % 60`.
* **Setting:** changing the hour adds 3600 seconds (mod 86400). Changing the minute changes only the minute part, so the hour is kept.
* **Blinking field:** when the field matches the mode and `(millis() / 400) % 2 == 0`, print two spaces instead of the number.
* **Alarm:** checked once when `secondsOfDay` changes and `seconds == 0`.
* `print2()` prints numbers with a leading zero (`7` → `07`).

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LCD (I2C) | GND / VCC | GND rail / 5V rail |
| LCD (I2C) | SDA / SCL | **A4** / **A5** |
| MODE button | T1 / T2 | GND / **D2** |
| INC button | T1 / T2 | GND / **D3** |
| Piezo | + / − | **D9** / GND |

```
LCD: GND─GND  VCC─5V  SDA─A4  SCL─A5
D2 ─► MODE ── GND     D3 ─► INC ── GND     D9 ─► PIEZO ── GND
```

## Code

```cpp
// Activity 19 - LCD Digital Clock with Alarm
#include <Adafruit_LiquidCrystal.h>

Adafruit_LiquidCrystal lcd(0);

const int MODE_PIN = 2;
const int INC_PIN = 3;
const int BUZZER_PIN = 9;

const int RUN = 0;
const int SET_H = 1;
const int SET_M = 2;
const int SET_AH = 3;
const int SET_AM = 4;
int mode = RUN;

long secondsOfDay = 6L * 3600 + 59 * 60 + 50;   // 06:59:50
int alarmH = 7, alarmM = 0;
bool alarmOn = true;
bool ringing = false;
unsigned long ringStart = 0;
unsigned long lastTick = 0;
unsigned long lastDraw = 0;

// button state
int modePrev = HIGH;
unsigned long modeEdge = 0;
int incStable = HIGH, incRaw = HIGH;
unsigned long incChanged = 0, incPressedAt = 0, incNextRepeat = 0;
bool incLongDone = false;

void print2(int v) {
  if (v < 10) lcd.print('0');
  lcd.print(v);
}

void field(int v, bool editing) {
  bool hide = editing && ((millis() / 400) % 2 == 0);
  if (hide) lcd.print("  ");
  else print2(v);
}

void draw() {
  int h = secondsOfDay / 3600;
  int m = (secondsOfDay / 60) % 60;
  int s = secondsOfDay % 60;
  lcd.setCursor(0, 0);
  lcd.print("Time ");
  field(h, mode == SET_H);
  lcd.print(':');
  field(m, mode == SET_M);
  lcd.print(':');
  print2(s);
  lcd.print("   ");
  lcd.setCursor(0, 1);
  lcd.print("Alarm ");
  field(alarmH, mode == SET_AH);
  lcd.print(':');
  field(alarmM, mode == SET_AM);
  lcd.print(ringing ? " RING" : (alarmOn ? " ON  " : " OFF "));
}

void increment() {
  if (mode == SET_H) secondsOfDay = (secondsOfDay + 3600L) % 86400L;
  else if (mode == SET_M) {
    long h = secondsOfDay / 3600;
    long m = (secondsOfDay / 60) % 60;
    long s = secondsOfDay % 60;
    m = (m + 1) % 60;
    secondsOfDay = h * 3600 + m * 60 + s;
  }
  else if (mode == SET_AH) alarmH = (alarmH + 1) % 24;
  else if (mode == SET_AM) alarmM = (alarmM + 1) % 60;
}

void stopRinging() {
  if (ringing) {
    ringing = false;
    noTone(BUZZER_PIN);
    Serial.println("Alarm stopped");
  }
}

void handleButtons() {
  unsigned long now = millis();

  // MODE: simple debounced edge
  int r = digitalRead(MODE_PIN);
  if (r != modePrev && now - modeEdge > 40) {
    modeEdge = now;
    modePrev = r;
    if (r == LOW) {
      if (ringing) stopRinging();
      else {
        mode = (mode + 1) % 5;
        Serial.print("Mode ");
        Serial.println(mode);
      }
    }
  }

  // INC: press / auto-repeat / long-press
  r = digitalRead(INC_PIN);
  if (r != incRaw) { incRaw = r; incChanged = now; }
  if (now - incChanged > 20 && r != incStable) {
    incStable = r;
    if (r == LOW) {
      incPressedAt = now;
      incLongDone = false;
      if (ringing) { stopRinging(); incLongDone = true; }
      else if (mode != RUN) { increment(); incNextRepeat = now + 500; }
    }
  }
  if (incStable == LOW && mode != RUN && (long)(now - incNextRepeat) >= 0) {
    increment();
    incNextRepeat = now + 150;
  }
  if (incStable == LOW && mode == RUN && !incLongDone && now - incPressedAt >= 1000) {
    incLongDone = true;
    alarmOn = !alarmOn;
    Serial.println(alarmOn ? "Alarm ON" : "Alarm OFF");
  }
}

void setup() {
  pinMode(MODE_PIN, INPUT_PULLUP);
  pinMode(INC_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  lcd.begin(16, 2);
  lcd.setBacklight(1);
  lastTick = millis();
  Serial.println("Clock started at 06:59:50, alarm 07:00");
}

void loop() {
  unsigned long now = millis();
  handleButtons();

  // time keeping without drift
  while (now - lastTick >= 1000) {
    lastTick += 1000;
    secondsOfDay = (secondsOfDay + 1) % 86400L;
    int h = secondsOfDay / 3600;
    int m = (secondsOfDay / 60) % 60;
    int s = secondsOfDay % 60;
    if (alarmOn && !ringing && h == alarmH && m == alarmM && s == 0) {
      ringing = true;
      ringStart = now;
      Serial.println("*** ALARM! ***");
    }
  }

  if (ringing) {
    unsigned long t = now - ringStart;
    if ((t % 1000) < 400 && (t % 200) < 100) tone(BUZZER_PIN, 2000);
    else noTone(BUZZER_PIN);
    if (t > 20000) stopRinging();
  }

  if (now - lastDraw >= 100) {
    lastDraw = now;
    draw();
  }
}
```

## Assessment Questions

1. Convert `secondsOfDay = 45296` to `HH:MM:SS`. Show your calculation.
2. Explain why `lastTick += 1000` is better than `lastTick = millis()` for keeping accurate time. How much drift could the second method cause in one day if each loop takes 3 ms?
3. Why does `increment()` for minutes rebuild `secondsOfDay` from h, m and s instead of just adding 60?
4. The `millis()` clock of an Arduino can be wrong by about 0.5 %. How many minutes per day is that? What hardware module would fix it?
5. Add a **snooze** function: pressing MODE while ringing stops the alarm but rings again after 5 minutes (only once). Which variables do you need?
