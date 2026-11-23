# Activity 01: Reaction Timer with False-Start Detection

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-01`

## Problem Statement

Build a reaction-time game. When the player presses the button, the game waits a **random** time (1.5–4 s), then turns on the LED and beeps. The player must press the button as fast as possible.

Requirements:

1. Measure the reaction time in milliseconds using `millis()`, with no `delay()` in the main loop.
2. If the player presses **before** the LED turns on, print `FALSE START` and play a low buzz.
3. If the player takes longer than 3 seconds, the round ends as `TOO SLOW`.
4. Keep track of the **best time**, **number of rounds**, and **average time**, and rate each result (Lightning / Good / Average / Slow).
5. Debounce the button in software.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | LED (red) + 1 kΩ resistor |
| 1 | Pushbutton |
| 1 | Piezo buzzer |

## Solution

The program is a **finite state machine** with three states:

| State | Meaning | Leaves when… |
|---|---|---|
| `IDLE` | waiting for player to start | button pressed → `WAITING` |
| `WAITING` | random delay running, LED off | button pressed → *false start* → `IDLE`; delay elapsed → `GO` |
| `GO` | LED on, timer running | button pressed → record time → `IDLE`; 3 s passed → *too slow* → `IDLE` |

* `random(1500, 4001)` picks the hidden delay and `randomSeed(analogRead(A5))` makes every power-up different.
* The button uses `INPUT_PULLUP` (pressed = `LOW`). The `buttonPressed()` function reports **one** event per press only after the signal has been stable for 25 ms (debouncing).
* Because there is no `delay()`, the loop runs thousands of times per second and the measured time is accurate to about 1 ms.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LED | Anode (+) | Arduino **D8** (via breadboard col 4) |
| LED | Cathode (−) | 1 kΩ resistor → GND rail |
| Pushbutton | Terminal 1 | GND rail |
| Pushbutton | Terminal 2 | Arduino **D2** (INPUT_PULLUP) |
| Piezo | + | Arduino **D9** |
| Piezo | − | GND rail |
| Arduino | 5V / GND | Breadboard + / − rails |

```
 Arduino D8 ──────────────► LED(+)  LED(−) ──[1kΩ]── GND
 Arduino D2 ──────────────► Button ─────────────────── GND
 Arduino D9 ──────────────► Piezo(+) Piezo(−) ──────── GND
 Arduino 5V ─► (+) rail      Arduino GND ─► (−) rail
```

## Code

```cpp
// Activity 01 - Reaction Timer with False-Start Detection
const int LED_PIN = 8;
const int BTN_PIN = 2;
const int BUZZER_PIN = 9;

const int IDLE = 0;
const int WAITING = 1;
const int GO = 2;

int state = IDLE;
unsigned long stateStart = 0;
unsigned long hiddenDelay = 0;

unsigned long bestTime = 0;   // 0 means "no record yet"
unsigned long totalTime = 0;
unsigned int validRounds = 0;
unsigned int falseStarts = 0;

// ----- debounced button -----
const unsigned long DEBOUNCE_MS = 25;
int lastReading = HIGH;
int stableState = HIGH;
unsigned long lastChange = 0;

bool buttonPressed() {
  int reading = digitalRead(BTN_PIN);
  if (reading != lastReading) {
    lastChange = millis();
    lastReading = reading;
  }
  if (millis() - lastChange > DEBOUNCE_MS && reading != stableState) {
    stableState = reading;
    if (stableState == LOW) return true;   // falling edge = new press
  }
  return false;
}

const char* rating(unsigned long ms) {
  if (ms < 200) return "Lightning!";
  if (ms < 300) return "Good";
  if (ms < 450) return "Average";
  return "Slow";
}

void printStats() {
  Serial.print("Rounds: ");
  Serial.print(validRounds);
  Serial.print(" | Best: ");
  Serial.print(bestTime);
  Serial.print(" ms | Average: ");
  if (validRounds > 0) Serial.print(totalTime / validRounds);
  else Serial.print("-");
  Serial.print(" ms | False starts: ");
  Serial.println(falseStarts);
  Serial.println("Press the button to play again.");
  Serial.println();
}

void startRound() {
  hiddenDelay = random(1500, 4001);
  stateStart = millis();
  state = WAITING;
  digitalWrite(LED_PIN, LOW);
  Serial.println("Get ready... wait for the light!");
}

void setup() {
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  Serial.begin(9600);
  randomSeed(analogRead(A5));
  Serial.println("=== REACTION TIMER ===");
  Serial.println("Press the button to start.");
}

void loop() {
  bool pressed = buttonPressed();
  unsigned long now = millis();

  if (state == IDLE) {
    if (pressed) startRound();
  }
  else if (state == WAITING) {
    if (pressed) {
      falseStarts++;
      tone(BUZZER_PIN, 150, 600);
      Serial.println("FALSE START! You pressed before the light.");
      state = IDLE;
      printStats();
    } else if (now - stateStart >= hiddenDelay) {
      digitalWrite(LED_PIN, HIGH);
      tone(BUZZER_PIN, 1200, 80);
      stateStart = now;
      state = GO;
    }
  }
  else if (state == GO) {
    if (pressed) {
      unsigned long rt = now - stateStart;
      digitalWrite(LED_PIN, LOW);
      validRounds++;
      totalTime += rt;
      if (bestTime == 0 || rt < bestTime) {
        bestTime = rt;
        Serial.println("*** NEW BEST TIME ***");
      }
      Serial.print("Reaction: ");
      Serial.print(rt);
      Serial.print(" ms -> ");
      Serial.println(rating(rt));
      state = IDLE;
      printStats();
    } else if (now - stateStart > 3000) {
      digitalWrite(LED_PIN, LOW);
      Serial.println("TOO SLOW! (more than 3000 ms)");
      state = IDLE;
      printStats();
    }
  }
}
```

## Assessment Questions

1. Why does the program use `millis()` instead of `delay()` while waiting for the player? What would go wrong with false-start detection if `delay(hiddenDelay)` were used?
2. Explain what `INPUT_PULLUP` does. Why does a pressed button read `LOW` in this circuit?
3. In `buttonPressed()`, what problem happens if the debounce check is removed? Describe what the player would see.
4. The statement `if (bestTime == 0 || rt < bestTime)` uses `0` as a special value. Suggest another way to show that no record exists yet.
5. Change the program so the round only counts if the reaction is faster than 1000 ms, and add a "streak" counter for consecutive "Good or better" results. Write the lines you would add.
