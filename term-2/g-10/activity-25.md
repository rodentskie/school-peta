# Activity 25: Tilt-Sensor Anti-Theft Alarm with a Sliding Time Window

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-25`

## Problem Statement

A bicycle alarm should not go off every time the wind shakes the bike a little, but it must react when someone is **really** moving it. Build an alarm using a tilt sensor:

1. Each change of the tilt switch (debounced 50 ms) is a **tilt event**.
2. A single event only gives a short **warning chirp** and a LED flash.
3. If **3 or more events happen within any 5-second window**, trigger the **ALARM**: siren and flashing LED for 10 s.
4. Use a **sliding window**. Store the times of the last events in a ring buffer and count how many are newer than `now − 5000`.
5. The **RESET button** stops the alarm immediately and clears the event history.
6. Print every event with its time and the current count in the window.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Tilt Sensor |
| 1 | LED + 1 kΩ resistor |
| 1 | Piezo buzzer |
| 1 | Pushbutton |

## Solution

* The tilt sensor is a switch. With `INPUT_PULLUP` it reads LOW when closed and HIGH when open. **Any** stable change counts as an event, so both tilting and returning count.
* **Ring buffer of timestamps:** `events[8]` and `head`. Each new event is written at `head`, then `head = (head + 1) % 8`.
* **Sliding window count:** loop over all 8 slots and count those where `t != 0 && now − t ≤ 5000`. Old events "slide out" of the window by themselves as time passes.
* **Alarm** lasts until `alarmUntil` or until the reset button is pressed.
* **Siren:** frequency alternates 900/1300 Hz every 250 ms.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Tilt Sensor | Terminal 1 | GND rail |
| Tilt Sensor | Terminal 2 | **D2** (INPUT_PULLUP) |
| LED | Anode | **D8** (cathode → 1 kΩ → GND) |
| Piezo | + / − | **D9** / GND |
| RESET button | T1 / T2 | GND / **D3** |

```
D2 ─► TILT SWITCH ── GND     D3 ─► RESET ── GND
D8 ─► LED ─[1k]─ GND         D9 ─► PIEZO ── GND
```

## Code

```cpp
// Activity 25 - Tilt Anti-Theft Alarm with Sliding Window
const int TILT_PIN = 2;
const int RESET_PIN = 3;
const int LED_PIN = 8;
const int BUZZER_PIN = 9;

const int BUF = 8;
const unsigned long WINDOW_MS = 5000;
const int TRIGGER_COUNT = 3;
const unsigned long ALARM_MS = 10000;

unsigned long events[BUF];
int head = 0;
unsigned long totalEvents = 0;

bool alarm = false;
unsigned long alarmUntil = 0;
unsigned long flashUntil = 0;

int tiltStable, tiltRaw;
unsigned long tiltChanged = 0;
int resetPrev = HIGH;
unsigned long resetEdge = 0;

void clearHistory() {
  for (int i = 0; i < BUF; i++) events[i] = 0;
}

int countInWindow(unsigned long now) {
  int n = 0;
  for (int i = 0; i < BUF; i++) {
    if (events[i] != 0 && now - events[i] <= WINDOW_MS) n++;
  }
  return n;
}

void onTiltEvent(unsigned long now) {
  events[head] = now;
  head = (head + 1) % BUF;
  totalEvents++;
  int n = countInWindow(now);

  Serial.print("[");
  Serial.print(now / 1000.0, 2);
  Serial.print(" s] tilt event #");
  Serial.print(totalEvents);
  Serial.print("  in last 5 s: ");
  Serial.println(n);

  if (!alarm && n >= TRIGGER_COUNT) {
    alarm = true;
    alarmUntil = now + ALARM_MS;
    Serial.println("*** THEFT ALARM ***");
  } else if (!alarm) {
    tone(BUZZER_PIN, 2500, 60);           // warning chirp
    flashUntil = now + 150;
  }
}

void setup() {
  pinMode(TILT_PIN, INPUT_PULLUP);
  pinMode(RESET_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  clearHistory();
  tiltStable = tiltRaw = digitalRead(TILT_PIN);
  Serial.println("Anti-theft alarm armed (3 tilts in 5 s = alarm)");
}

void loop() {
  unsigned long now = millis();

  // debounced tilt change detection
  int r = digitalRead(TILT_PIN);
  if (r != tiltRaw) { tiltRaw = r; tiltChanged = now; }
  if (now - tiltChanged > 50 && r != tiltStable) {
    tiltStable = r;
    onTiltEvent(now);
  }

  // reset button
  int b = digitalRead(RESET_PIN);
  if (b != resetPrev && now - resetEdge > 40) {
    resetEdge = now;
    resetPrev = b;
    if (b == LOW) {
      alarm = false;
      noTone(BUZZER_PIN);
      clearHistory();
      Serial.println("RESET - alarm cleared");
    }
  }

  if (alarm) {
    unsigned long t = now % 500;
    tone(BUZZER_PIN, t < 250 ? 900 : 1300);
    digitalWrite(LED_PIN, (now / 100) % 2);
    if ((long)(now - alarmUntil) >= 0) {
      alarm = false;
      noTone(BUZZER_PIN);
      clearHistory();
      Serial.println("Alarm timed out - re-armed");
    }
  } else {
    digitalWrite(LED_PIN, (long)(flashUntil - now) > 0 ? HIGH : LOW);
  }
}
```

## Assessment Questions

1. Events happen at t = 1.0 s, 4.0 s, 7.5 s, 8.0 s and 9.2 s. At which event does the alarm start? Show the window count at each event.
2. Explain the difference between a **sliding** window and a **fixed** window (e.g. counting events in 0–5 s, then 5–10 s). Give an example where a fixed window would miss a theft.
3. Why is the ring buffer size (8) larger than the trigger count (3)? What would happen if it were only 3?
4. What does `events[i] != 0` protect against in `countInWindow()`? Is there a rare case where it could be wrong?
5. Add a **sensitivity** setting using a potentiometer on A0 that changes `TRIGGER_COUNT` between 2 and 6. Write the code and explain how the user would know which value is set.
