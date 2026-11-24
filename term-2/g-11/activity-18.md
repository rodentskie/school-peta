# Activity 18: Serial Basketball Scoreboard with Game Clock

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-18`
**Control:** Serial Monitor (type characters and click **Send**)

## Problem Statement

Build a scoreboard that a scorekeeper controls from a laptop.

| Key | Action |
|---|---|
| `1` `2` `3` | add 1 / 2 / 3 points to **HOME** |
| `7` `8` `9` | add 1 / 2 / 3 points to **AWAY** |
| `u` | undo the last score change |
| `g` | start / pause the game clock |
| `p` | next period (4 periods), resets the clock to 2:00 |
| `+` / `-` | adjust the clock ±10 s (only while paused) |
| `r` | reset the whole game |
| `?` | print the full status |

1. **LCD line 1:** `HOME 045-038 AWY` (AWAY is shortened to fit 16 characters). **LCD line 2:** `P2  01:37  RUN` (period, clock, RUN/PAUSE/END).
2. The game clock counts **down** from 2:00 (shortened for class). At 0:00 the **buzzer** sounds for 1.5 s and the clock stops. After period 4, show `FINAL` and the winner on the Serial Monitor.
3. The **undo** keeps a history of the last 10 score changes, so mistakes can be corrected.
4. The LED is on while the clock is running and blinks during the last 10 seconds.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | LCD 16 × 2 (I2C) |
| 1 | Piezo buzzer |
| 1 | LED + 1 kΩ resistor |

## Solution

* **Scores:** `int score[2]` (0 = HOME, 1 = AWAY).
* **Undo history:** two small arrays used as a **stack**: `histTeam[10]`, `histPts[10]` and `histLen`. `u` pops the last entry and subtracts it. When the stack is full, the oldest entry is shifted out.
* **Clock:** `remainingMs`. While running, `remainingMs` = `endTime − now`. Pause stores the remainder and resume calculates a new `endTime` (the same pattern as earlier timers).
* `formatScore()` uses `sprintf("%03d")` so the numbers keep their position on the LCD.
* The buzzer is a non-blocking `tone(pin, f, 1500)`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LCD (I2C) | GND / VCC | GND rail / 5V rail |
| LCD (I2C) | SDA / SCL | **A4** / **A5** |
| Piezo | + / − | **D9** / GND |
| LED | Anode | **D8** (cathode → 1 kΩ → GND) |

```
LCD: GND─GND VCC─5V SDA─A4 SCL─A5
D9 ─► PIEZO ── GND      D8 ─► LED ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 18 - Serial Basketball Scoreboard
#include <Adafruit_LiquidCrystal.h>

Adafruit_LiquidCrystal lcd(0);

const int BUZZER_PIN = 9;
const int LED_PIN = 8;
const unsigned long PERIOD_MS = 120000UL;     // 2:00
const int PERIODS = 4;
const int HIST = 10;

int score[2] = {0, 0};
int histTeam[HIST], histPts[HIST];
int histLen = 0;

int period = 1;
bool running = false;
bool ended = false;
unsigned long endTime = 0;
long remainingMs = PERIOD_MS;
unsigned long lastDraw = 0;

long remaining() {
  if (!running) return remainingMs;
  long r = (long)(endTime - millis());
  return r > 0 ? r : 0;
}

void draw() {
  char buf[17];
  sprintf(buf, "HOME %03d-%03d AWY", score[0], score[1]);
  lcd.setCursor(0, 0);
  lcd.print(buf);
  long r = remaining();
  int secs = (r + 999) / 1000;
  const char *st = ended ? "END  " : (running ? "RUN  " : "PAUSE");
  sprintf(buf, "P%d  %02d:%02d  %s", period, secs / 60, secs % 60, st);
  lcd.setCursor(0, 1);
  lcd.print(buf);
}

void addPoints(int team, int pts) {
  score[team] += pts;
  if (histLen == HIST) {
    for (int i = 0; i < HIST - 1; i++) { histTeam[i] = histTeam[i + 1]; histPts[i] = histPts[i + 1]; }
    histLen--;
  }
  histTeam[histLen] = team;
  histPts[histLen] = pts;
  histLen++;
  tone(BUZZER_PIN, 1500, 40);
  Serial.print(team == 0 ? "HOME +" : "AWAY +");
  Serial.println(pts);
}

void undo() {
  if (histLen == 0) { Serial.println("Nothing to undo"); return; }
  histLen--;
  score[histTeam[histLen]] -= histPts[histLen];
  Serial.print("Undo: ");
  Serial.print(histTeam[histLen] == 0 ? "HOME -" : "AWAY -");
  Serial.println(histPts[histLen]);
}

void status() {
  Serial.print("HOME ");
  Serial.print(score[0]);
  Serial.print(" - ");
  Serial.print(score[1]);
  Serial.print(" AWAY | Period ");
  Serial.print(period);
  Serial.print(" | ");
  Serial.print(remaining() / 1000);
  Serial.println(" s left");
}

void endPeriod() {
  running = false;
  remainingMs = 0;
  tone(BUZZER_PIN, 400, 1500);
  Serial.print("End of period ");
  Serial.println(period);
  if (period == PERIODS) {
    ended = true;
    Serial.print("FINAL: ");
    status();
    if (score[0] == score[1]) Serial.println("It's a TIE!");
    else Serial.println(score[0] > score[1] ? "HOME wins!" : "AWAY wins!");
  }
}

void handle(char c) {
  if (c == '\n' || c == '\r' || c == ' ') return;
  if (c >= '1' && c <= '3') addPoints(0, c - '0');
  else if (c >= '7' && c <= '9') addPoints(1, c - '6');
  else if (c == 'u') undo();
  else if (c == 'g') {
    if (ended || remainingMs == 0) { Serial.println("Period over - press p"); }
    else if (running) { remainingMs = remaining(); running = false; Serial.println("PAUSE"); }
    else { endTime = millis() + remainingMs; running = true; Serial.println("RUN"); }
  }
  else if (c == 'p') {
    if (period < PERIODS && !running) { period++; remainingMs = PERIOD_MS; Serial.print("Period "); Serial.println(period); }
    else Serial.println(running ? "Pause the clock first" : "Last period already");
  }
  else if (c == '+' || c == '-') {
    if (running) Serial.println("Pause the clock first");
    else remainingMs = constrain(remainingMs + (c == '+' ? 10000L : -10000L), 0L, (long)PERIOD_MS);
  }
  else if (c == 'r') {
    score[0] = score[1] = 0; histLen = 0; period = 1;
    running = false; ended = false; remainingMs = PERIOD_MS;
    Serial.println("New game");
  }
  else if (c == '?') status();
  else { Serial.print("Unknown '"); Serial.print(c); Serial.println("'"); return; }
  draw();
}

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  lcd.begin(16, 2);
  lcd.setBacklight(1);
  Serial.println("Scoreboard: 1 2 3 HOME | 7 8 9 AWAY | u undo | g clock | p period | + - | r reset | ?");
  draw();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();
  if (running && remaining() == 0) endPeriod();
  long r = remaining();
  if (running && r <= 10000) digitalWrite(LED_PIN, (now / 250) % 2);
  else digitalWrite(LED_PIN, running ? HIGH : LOW);
  if (now - lastDraw >= 200) { lastDraw = now; draw(); }
}
```

## Assessment Questions

1. What does the LCD show right after the simulation starts?
2. Send `3`, `2` and `9`. What does line 1 show? Now send `u` twice. What changes?
3. Send `g`. What happens to the clock and the LED? Send `+` while it is running. What happens?
4. Let the clock run to the last 10 seconds and then to 0:00. What do the LED and the buzzer do?
5. If the LED's resistor is removed, what happens to the LED? If the buzzer is disconnected, how would the players know the period ended?
