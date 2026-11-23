# Activity 14: 7-Segment Countdown Timer with Pause and Alarm

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-14`

## Problem Statement

Build a kitchen-style countdown timer on a single 7-segment display.

1. In **SETTING** mode, the potentiometer selects the start value 1–9 seconds, shown live on the display.
2. A **short press** starts the countdown. Another short press **pauses** or **resumes** it. While paused, the display blinks.
3. During the last 3 seconds, a short beep sounds each second.
4. At **0**, the display flashes "0" and an alarm (two-tone) sounds until the button is pressed. Then the timer returns to SETTING mode.
5. A **long press** (≥ 1 s) at any time **resets** the timer to SETTING mode.
6. Timing must be **accurate**. The remaining time is calculated from `millis()` and an end time, not by counting `delay(1000)` calls.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | 7-Segment Display (common cathode) + 1 kΩ resistor |
| 1 | Potentiometer |
| 1 | Pushbutton |
| 1 | Piezo buzzer |

## Solution

States: `SETTING → RUNNING ⇄ PAUSED → ALARM → SETTING`

* When RUNNING, `remainingMs = endTime − millis()`. The displayed digit is `ceil(remainingMs / 1000)`, using integer math: `(remainingMs + 999) / 1000`.
* On **pause**, store `remainingMs`. On **resume**, calculate a new `endTime = millis() + remainingMs`. No time is lost or gained.
* The 3-2-1 beeps are triggered when the shown digit **changes** (`digit != lastDigit`) and is ≤ 3.
* The button logic separates short and long presses (release < 1 s = short; held ≥ 1 s = long, fired once).

## Pin Diagram

| 7-Segment pin | Arduino |
|---|---|
| a, b, c, d, e, f, g | **D2, D3, D4, D5, D6, D7, D8** |
| Common | 1 kΩ → GND rail |

| Other | Pin | Connects to |
|---|---|---|
| Potentiometer | T1 / Wiper / T2 | GND / **A0** / 5V |
| Pushbutton | T1 / T2 | GND / **D10** |
| Piezo | + / − | **D11** / GND |

```
D2..D8 ─► a..g   COMMON ─[1k]─ GND     A0 ◄─ POT wiper
D10 ─► BUTTON ── GND                   D11 ─► PIEZO ── GND
```

## Code

```cpp
// Activity 14 - 7-Segment Countdown Timer
const int SEG_PINS[7] = {2, 3, 4, 5, 6, 7, 8};   // a..g
const int POT_PIN = A0;
const int BTN_PIN = 10;
const int BUZZER_PIN = 11;

const byte DIGITS[10] = {
  0b0111111, 0b0000110, 0b1011011, 0b1001111, 0b1100110,
  0b1101101, 0b1111101, 0b0000111, 0b1111111, 0b1101111
};

const int SETTING = 0;
const int RUNNING = 1;
const int PAUSED = 2;
const int ALARM = 3;
int state = SETTING;

unsigned long endTime = 0;
long remainingMs = 0;
int lastDigit = -1;
unsigned long alarmStart = 0;

int btnStable = HIGH, btnRaw = HIGH;
unsigned long btnChanged = 0, btnPressedAt = 0;
bool longDone = false;

void showPattern(byte p) {
  for (int i = 0; i < 7; i++) digitalWrite(SEG_PINS[i], ((p >> i) & 1) ? HIGH : LOW);
}

// returns 0 = nothing, 1 = short press, 2 = long press
int readButton() {
  int r = digitalRead(BTN_PIN);
  unsigned long now = millis();
  int ev = 0;
  if (r != btnRaw) { btnRaw = r; btnChanged = now; }
  if (now - btnChanged > 20 && r != btnStable) {
    btnStable = r;
    if (r == LOW) { btnPressedAt = now; longDone = false; }
    else if (!longDone) ev = 1;
  }
  if (btnStable == LOW && !longDone && now - btnPressedAt >= 1000) {
    longDone = true;
    ev = 2;
  }
  return ev;
}

void toSet() {
  state = SETTING;
  noTone(BUZZER_PIN);
  Serial.println("SETTING mode - turn knob, press to start");
}

void setup() {
  for (int i = 0; i < 7; i++) pinMode(SEG_PINS[i], OUTPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  toSet();
}

void loop() {
  unsigned long now = millis();
  int ev = readButton();

  if (ev == 2) {                       // long press = reset from anywhere
    Serial.println("RESET");
    toSet();
    return;
  }

  if (state == SETTING) {
    int start = map(analogRead(POT_PIN), 0, 1023, 1, 9);
    showPattern(DIGITS[start]);
    if (ev == 1) {
      endTime = now + start * 1000UL;
      lastDigit = -1;
      state = RUNNING;
      Serial.print("START from ");
      Serial.println(start);
    }
  }
  else if (state == RUNNING) {
    remainingMs = (long)(endTime - now);
    if (remainingMs <= 0) {
      state = ALARM;
      alarmStart = now;
      Serial.println("TIME'S UP!");
      return;
    }
    int digit = (remainingMs + 999) / 1000;
    showPattern(DIGITS[digit]);
    if (digit != lastDigit) {
      lastDigit = digit;
      Serial.println(digit);
      if (digit <= 3) tone(BUZZER_PIN, 1000, 100);
    }
    if (ev == 1) {
      state = PAUSED;
      Serial.println("PAUSED");
    }
  }
  else if (state == PAUSED) {
    int digit = (remainingMs + 999) / 1000;
    if ((now / 300) % 2) showPattern(DIGITS[digit]);
    else showPattern(0);
    if (ev == 1) {
      endTime = now + remainingMs;       // resume exactly where we stopped
      state = RUNNING;
      Serial.println("RESUMED");
    }
  }
  else if (state == ALARM) {
    unsigned long t = now - alarmStart;
    showPattern((t % 400) < 200 ? DIGITS[0] : 0);
    tone(BUZZER_PIN, (t % 500) < 250 ? 880 : 660);
    if (ev == 1) toSet();
  }
}
```

## Assessment Questions

1. What does the display show when you turn the potentiometer before starting?
2. What happens after you press the button once? What happens when you press it again while counting?
3. What do you hear during the last 3 seconds and when the timer reaches zero?
4. What happens when you hold the button for more than 1 second?
5. If the potentiometer's wire to A0 is disconnected, what do you think the display will show before starting?
