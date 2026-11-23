# Activity 26: LCD Stopwatch with Lap Memory

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-26`

## Problem Statement

Build a sports stopwatch with two buttons and a 16×2 I2C LCD.

1. Line 1 shows the running time as `MM:SS.cc` (minutes, seconds, hundredths).
2. **Button A (D2):** start / stop.
3. **Button B (D3):**
   * while **running**: record a **lap** (up to 5 laps are stored);
   * while **stopped**: a short press scrolls through the saved laps; a **long press** (≥ 1 s) resets everything.
4. Line 2 shows the latest (or selected) lap as `L2 00:12.34 +1.20`. The last number is the **difference** from the previous lap (+ slower / − faster).
5. Stopping and starting again continues from the stopped time; it does not restart from zero.
6. When the 6th lap is pressed, the oldest lap is dropped (FIFO), and the lap numbers continue counting.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | LCD 16 × 2 (I2C) |
| 2 | Pushbuttons |

## Solution

* **Time with pauses:** `elapsed = accumulated + (running ? millis() − startedAt : 0)`, the same pattern as the Pomodoro timer.
* **Lap times** are **split times**: the time of each lap alone = total at this lap − total at the previous lap.
* **Lap storage:** `laps[5]` with a FIFO shift. When full, shift all elements one place left and put the new one at the end. `lapNumber` keeps counting, so the display shows the real lap number.
* **Formatting:** `formatTime(ms, buf)` builds `MM:SS.cc` with `sprintf` and integers only (`%02lu`), because `sprintf` on the Arduino does not support `%f`.
* The difference (`+1.20`) is printed with a sign and two decimals using integer division and modulus.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Button A (start/stop) | T1 / T2 | GND / **D2** |
| Button B (lap/reset) | T1 / T2 | GND / **D3** |
| LCD (I2C) | GND / VCC | GND rail / 5V rail |
| LCD (I2C) | SDA / SCL | **A4** / **A5** |

```
D2 ─► BUTTON A ── GND
D3 ─► BUTTON B ── GND
LCD: GND─GND VCC─5V SDA─A4 SCL─A5
```

## Code

```cpp
// Activity 26 - LCD Stopwatch with Lap Memory
#include <Adafruit_LiquidCrystal.h>

Adafruit_LiquidCrystal lcd(0);

const int A_PIN = 2;
const int B_PIN = 3;
const int MAX_LAPS = 5;

bool running = false;
unsigned long startedAt = 0;
unsigned long accumulated = 0;

unsigned long laps[MAX_LAPS];     // split time of each stored lap (ms)
int lapNums[MAX_LAPS];            // real lap number of each stored lap
int lapCount = 0;
int lapNumber = 0;
unsigned long lastLapTotal = 0;
int viewIdx = -1;                 // -1 = latest

int aPrev = HIGH;
unsigned long aEdge = 0;
int bStable = HIGH, bRaw = HIGH;
unsigned long bChanged = 0, bPressedAt = 0;
bool bLongDone = false;
unsigned long lastDraw = 0;

unsigned long elapsed() {
  return accumulated + (running ? millis() - startedAt : 0);
}

void formatTime(unsigned long ms, char *buf) {
  unsigned long cs = ms / 10;                    // hundredths
  unsigned long m = cs / 6000;
  unsigned long s = (cs / 100) % 60;
  unsigned long c = cs % 100;
  sprintf(buf, "%02lu:%02lu.%02lu", m % 100, s, c);
}

void addLap() {
  unsigned long total = elapsed();
  unsigned long split = total - lastLapTotal;
  lastLapTotal = total;
  lapNumber++;
  if (lapCount == MAX_LAPS) {                    // FIFO: drop oldest
    for (int i = 0; i < MAX_LAPS - 1; i++) {
      laps[i] = laps[i + 1];
      lapNums[i] = lapNums[i + 1];
    }
    lapCount--;
  }
  laps[lapCount] = split;
  lapNums[lapCount] = lapNumber;
  lapCount++;
  viewIdx = -1;
  char buf[12];
  formatTime(split, buf);
  Serial.print("Lap ");
  Serial.print(lapNumber);
  Serial.print(": ");
  Serial.println(buf);
}

void resetAll() {
  running = false;
  accumulated = 0;
  lapCount = 0;
  lapNumber = 0;
  lastLapTotal = 0;
  viewIdx = -1;
  lcd.clear();
  Serial.println("RESET");
}

void drawLine2() {
  lcd.setCursor(0, 1);
  if (lapCount == 0) {
    lcd.print("A=run B=lap     ");
    return;
  }
  int i = (viewIdx < 0) ? lapCount - 1 : viewIdx;
  char buf[12];
  formatTime(laps[i], buf);
  lcd.print('L');
  lcd.print(lapNums[i] % 10);
  lcd.print(' ');
  lcd.print(buf);
  if (i > 0) {
    long diff = (long)laps[i] - (long)laps[i - 1];
    lcd.print(diff >= 0 ? '+' : '-');
    long a = abs(diff) / 10;                     // hundredths
    if (a > 999) a = 999;
    lcd.print(a / 100);
    lcd.print('.');
    if (a % 100 < 10) lcd.print('0');
    lcd.print(a % 100);
  }
  lcd.print("    ");
}

void setup() {
  pinMode(A_PIN, INPUT_PULLUP);
  pinMode(B_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  lcd.begin(16, 2);
  lcd.setBacklight(1);
  Serial.println("Stopwatch: A = start/stop, B = lap / view, hold B = reset");
}

void loop() {
  unsigned long now = millis();

  // Button A: start / stop
  int a = digitalRead(A_PIN);
  if (a != aPrev && now - aEdge > 40) {
    aEdge = now;
    aPrev = a;
    if (a == LOW) {
      if (running) {
        accumulated += now - startedAt;
        running = false;
        Serial.println("STOP");
      } else {
        startedAt = now;
        running = true;
        Serial.println("START");
      }
    }
  }

  // Button B: lap / view / reset
  int b = digitalRead(B_PIN);
  if (b != bRaw) { bRaw = b; bChanged = now; }
  if (now - bChanged > 20 && b != bStable) {
    bStable = b;
    if (b == LOW) {
      bPressedAt = now;
      bLongDone = false;
      if (running) { addLap(); bLongDone = true; }
    } else if (!bLongDone && !running && lapCount > 0) {
      viewIdx = (viewIdx < 0) ? 0 : viewIdx + 1;
      if (viewIdx >= lapCount) viewIdx = -1;
    }
  }
  if (bStable == LOW && !bLongDone && !running && now - bPressedAt >= 1000) {
    bLongDone = true;
    resetAll();
  }

  if (now - lastDraw >= 50) {
    lastDraw = now;
    char buf[12];
    formatTime(elapsed(), buf);
    lcd.setCursor(0, 0);
    lcd.print(running ? "RUN  " : "STOP ");
    lcd.print(buf);
    lcd.print("   ");
    drawLine2();
  }
}
```

## Assessment Questions

1. What happens when you press button A once, and then again?
2. What happens on line 2 when you press button B while the stopwatch is running?
3. Stop the stopwatch, then press A again. Does it start from zero or continue?
4. What happens when you hold B for 1 second while stopped? What happens while running?
5. If the LCD's power wire is disconnected, does the stopwatch still count? How could you check?
