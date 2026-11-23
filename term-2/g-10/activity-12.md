# Activity 12: PIR Home Security System (Finite State Machine)

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-12`

## Problem Statement

Design a home alarm system with a **PIR motion sensor** and one **ARM/DISARM** button. It must behave like a real alarm panel:

| State | Behaviour |
|---|---|
| **DISARMED** | Green LED on. Motion is ignored. |
| **EXIT_DELAY** (10 s) | Green LED blinks and the buzzer chirps every second so the owner can leave. |
| **ARMED** | Red LED blinks slowly (100 ms every 2 s). Motion → ENTRY_DELAY. |
| **ENTRY_DELAY** (8 s) | Red LED blinks fast and the buzzer beeps so the owner can disarm. Timeout → ALARM. |
| **ALARM** | Siren (frequency sweeps 600–1600 Hz) and red LED flashing, for at most 30 s, then back to ARMED. |

* Pressing the button in **any** state except DISARMED returns to DISARMED. Pressing it in DISARMED starts EXIT_DELAY.
* Every state change is logged on the Serial Monitor with a timestamp in seconds, e.g. `[12.4 s] ARMED`.
* The number of alarms since power-up is counted and printed.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | PIR Sensor |
| 2 | LEDs (green, red) + 2 × 1 kΩ resistors |
| 1 | Pushbutton |
| 1 | Piezo buzzer |

## Solution

```
           press                10 s                 motion
DISARMED ────────► EXIT_DELAY ───────► ARMED ───────────────► ENTRY_DELAY
   ▲                   │ press           ▲  │ press                │ 8 s     │ press
   └───────────────────┴─────────────────┼──┴──────────────────────┼─────────┘
                                         └──── 30 s ──── ALARM ◄────┘
```

* `go(newState)` saves the time, prints the log, and resets outputs.
* Each state's behaviour is computed from `t = now − stateStart`, for example a chirp when `t % 1000 < 50`.
* **Siren:** the frequency rises from 600 to 1600 Hz over 1 s, then starts again: `600 + (t % 1000)`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| PIR Sensor | Signal | **D2** |
| PIR Sensor | Power / Ground | 5V rail / GND rail |
| ARM button | T1 / T2 | GND / **D3** |
| Green LED | Anode | **D5** |
| Red LED | Anode | **D6** |
| LED cathodes | — | 1 kΩ → GND rail |
| Piezo | + / − | **D9** / GND |

```
PIR: VCC─5V  OUT─D2  GND─GND       D3 ─► BUTTON ── GND
D5 ─► GREEN ─[1k]─ GND             D9 ─► PIEZO ─── GND
D6 ─► RED   ─[1k]─ GND
```

## Code

```cpp
// Activity 12 - PIR Home Security System
const int PIR_PIN = 2;
const int BTN_PIN = 3;
const int GREEN_PIN = 5;
const int RED_PIN = 6;
const int BUZZER_PIN = 9;

const int DISARMED = 0;
const int EXIT_DELAY = 1;
const int ARMED = 2;
const int ENTRY_DELAY = 3;
const int ALARM = 4;
const char* NAMES[] = {"DISARMED", "EXIT DELAY", "ARMED", "ENTRY DELAY", "ALARM!"};

const unsigned long EXIT_MS = 10000;
const unsigned long ENTRY_MS = 8000;
const unsigned long ALARM_MS = 30000;

int state = DISARMED;
unsigned long stateStart = 0;
int alarmCount = 0;

int btnPrev = HIGH;
unsigned long btnEdge = 0;

void go(int s) {
  state = s;
  stateStart = millis();
  noTone(BUZZER_PIN);
  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(RED_PIN, LOW);
  Serial.print("[");
  Serial.print(millis() / 1000.0, 1);
  Serial.print(" s] ");
  Serial.println(NAMES[s]);
  if (s == ALARM) {
    alarmCount++;
    Serial.print("   Total alarms: ");
    Serial.println(alarmCount);
  }
}

bool buttonPressed() {
  int r = digitalRead(BTN_PIN);
  bool pressed = false;
  if (r != btnPrev && millis() - btnEdge > 50) {
    btnEdge = millis();
    if (r == LOW) pressed = true;
    btnPrev = r;
  }
  return pressed;
}

void setup() {
  pinMode(PIR_PIN, INPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(RED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Security panel - press button to ARM");
  go(DISARMED);
}

void loop() {
  unsigned long now = millis();
  unsigned long t = now - stateStart;
  bool motion = digitalRead(PIR_PIN) == HIGH;

  if (buttonPressed()) {
    if (state == DISARMED) go(EXIT_DELAY);
    else go(DISARMED);
    return;
  }

  if (state == DISARMED) {
    digitalWrite(GREEN_PIN, HIGH);
  }
  else if (state == EXIT_DELAY) {
    digitalWrite(GREEN_PIN, (t % 500) < 250 ? HIGH : LOW);
    if (t % 1000 < 50) tone(BUZZER_PIN, 2500);
    else noTone(BUZZER_PIN);
    if (t >= EXIT_MS) go(ARMED);
  }
  else if (state == ARMED) {
    digitalWrite(RED_PIN, (t % 2000) < 100 ? HIGH : LOW);
    if (motion) {
      Serial.println("   Motion detected!");
      go(ENTRY_DELAY);
    }
  }
  else if (state == ENTRY_DELAY) {
    digitalWrite(RED_PIN, (t % 200) < 100 ? HIGH : LOW);
    if (t % 500 < 80) tone(BUZZER_PIN, 3000);
    else noTone(BUZZER_PIN);
    if (t >= ENTRY_MS) go(ALARM);
  }
  else if (state == ALARM) {
    digitalWrite(RED_PIN, (t % 300) < 150 ? HIGH : LOW);
    tone(BUZZER_PIN, 600 + (t % 1000));
    if (t >= ALARM_MS) go(ARMED);
  }
}
```

## Assessment Questions

1. List all the possible transitions **out of** the ARMED state, and the event that causes each one.
2. Why does the system have an EXIT_DELAY and an ENTRY_DELAY? What would happen without them?
3. Why does the ALARM state end after 30 s and return to ARMED instead of DISARMED?
4. The code uses `return;` right after handling the button. What problem does this prevent in the rest of `loop()`?
5. Add a **PIN code**: the system can only be disarmed during ENTRY_DELAY or ALARM by pressing the button exactly 3 times within 2 seconds. Describe the variables you need and write the logic.
