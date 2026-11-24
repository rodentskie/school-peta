# Activity 12: Serial Color Reaction Quiz

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-12`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

A reaction game played with the keyboard. One of three LEDs (red, green, blue) lights up at random, and the player must send the matching key **as fast as possible**.

| Key | Meaning |
|---|---|
| `s` | start a new game (10 rounds) |
| `r` `g` `b` | answer: red, green, blue |
| `d` | difficulty: EASY (3 s to answer) → NORMAL (2 s) → HARD (1 s) |
| `q` | quit the current game |
| `h` | help |

1. Each round: wait a random time (1–3 s), light one random LED, and start a timer.
2. **Correct key** → short high beep, add points: `points = 1000 − reaction time in ms` (minimum 100).
3. **Wrong key** → low buzz, 0 points. **No answer** within the time limit → `TIMEOUT`, 0 points.
4. A key sent **before** the LED lights = `TOO EARLY!`, −200 points (the score can go negative).
5. After 10 rounds, print the total score, the number correct, the average reaction time of the correct answers, and the best reaction time.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 3 | LEDs (red, green, blue) + 3 × 1 kΩ resistors |
| 1 | Piezo buzzer |

## Solution

* **States:** `IDLE`, `WAITING` (random delay), `SHOWING` (LED on, timer running), `FINISHED`.
* `startRound()` picks `target = random(3)` and `waitUntil = now + random(1000, 3001)`.
* **Answer checking** happens in `handle(c)`, and the state decides the meaning:
  * in `WAITING` → too early;
  * in `SHOWING` → compare `c` with `KEYS[target]`.
* The time limit depends on the difficulty: `LIMITS[3] = {3000, 2000, 1000}`.
* Statistics: `correct`, `sumTime`, `bestTime` are updated only on correct answers.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Red LED | Anode | **D8** |
| Green LED | Anode | **D9** |
| Blue LED | Anode | **D10** |
| LED cathodes | — | 1 kΩ → GND rail |
| Piezo | + / − | **D11** / GND |

```
D8  ─► RED   ─[1k]─ GND
D9  ─► GREEN ─[1k]─ GND      D11 ─► PIEZO ── GND
D10 ─► BLUE  ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 12 - Serial Color Reaction Quiz
const int LED_PINS[3] = {8, 9, 10};
const char KEYS[3] = {'r', 'g', 'b'};
const char* COLOR_NAMES[3] = {"RED", "GREEN", "BLUE"};
const int BUZZER_PIN = 11;
const unsigned long LIMITS[3] = {3000, 2000, 1000};
const char* LEVELS[3] = {"EASY", "NORMAL", "HARD"};
const int ROUNDS = 10;

const int IDLE = 0, WAITING = 1, SHOWING = 2;
int state = IDLE;
int level = 1;
int round_ = 0;
int target = 0;
long score = 0;
int correct = 0;
unsigned long sumTime = 0, bestTime = 0;
unsigned long waitUntil = 0, shownAt = 0;

void allOff() {
  for (int i = 0; i < 3; i++) digitalWrite(LED_PINS[i], LOW);
}

void help() {
  Serial.println("--- REACTION QUIZ ---");
  Serial.println(" s start | r g b answer | d difficulty | q quit | h help");
}

void startRound() {
  round_++;
  allOff();
  target = random(3);
  waitUntil = millis() + random(1000, 3001);
  state = WAITING;
  Serial.print("Round ");
  Serial.print(round_);
  Serial.println("/10 - get ready...");
}

void finishGame() {
  state = IDLE;
  allOff();
  Serial.println("===== GAME OVER =====");
  Serial.print("Score: ");
  Serial.println(score);
  Serial.print("Correct: ");
  Serial.print(correct);
  Serial.println("/10");
  if (correct > 0) {
    Serial.print("Average reaction: ");
    Serial.print(sumTime / correct);
    Serial.println(" ms");
    Serial.print("Best reaction: ");
    Serial.print(bestTime);
    Serial.println(" ms");
  }
  Serial.println("Press s to play again");
}

void nextOrFinish() {
  allOff();
  if (round_ >= ROUNDS) finishGame();
  else startRound();
}

void answer(char c) {
  unsigned long rt = millis() - shownAt;
  if (c == KEYS[target]) {
    long pts = 1000 - (long)rt;
    if (pts < 100) pts = 100;
    score += pts;
    correct++;
    sumTime += rt;
    if (bestTime == 0 || rt < bestTime) bestTime = rt;
    tone(BUZZER_PIN, 1800, 80);
    Serial.print("CORRECT in ");
    Serial.print(rt);
    Serial.print(" ms  +");
    Serial.println(pts);
  } else {
    tone(BUZZER_PIN, 200, 300);
    Serial.print("WRONG - it was ");
    Serial.println(COLOR_NAMES[target]);
  }
  nextOrFinish();
}

void handle(char c) {
  if (c == '\n' || c == '\r' || c == ' ') return;
  if (c == 's') {
    score = 0; correct = 0; sumTime = 0; bestTime = 0; round_ = 0;
    Serial.print("New game - ");
    Serial.println(LEVELS[level]);
    startRound();
  } else if (c == 'd') {
    if (state != IDLE) { Serial.println("Change difficulty between games"); return; }
    level = (level + 1) % 3;
    Serial.print("Difficulty: ");
    Serial.println(LEVELS[level]);
  } else if (c == 'q') {
    if (state != IDLE) { Serial.println("Game stopped"); state = IDLE; allOff(); }
  } else if (c == 'h' || c == '?') {
    help();
  } else if (c == 'r' || c == 'g' || c == 'b') {
    if (state == WAITING) {
      score -= 200;
      tone(BUZZER_PIN, 150, 400);
      Serial.println("TOO EARLY! -200");
    } else if (state == SHOWING) {
      answer(c);
    } else {
      Serial.println("Press s to start");
    }
  } else {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
  }
}

void setup() {
  for (int i = 0; i < 3; i++) pinMode(LED_PINS[i], OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  randomSeed(analogRead(A0) + analogRead(A1));
  help();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();
  if (state == WAITING && (long)(now - waitUntil) >= 0) {
    digitalWrite(LED_PINS[target], HIGH);
    shownAt = now;
    state = SHOWING;
  } else if (state == SHOWING && now - shownAt > LIMITS[level]) {
    tone(BUZZER_PIN, 200, 300);
    Serial.println("TIMEOUT - 0 points");
    nextOrFinish();
  }
}
```

## Assessment Questions

1. Send `s`. What happens on the LEDs and in the Serial Monitor?
2. When the green LED lights, send `g`. What do you hear and see? What happens if you send `r` instead?
3. What happens if you send a key before any LED lights up?
4. Send `d` until HARD is selected, then play. What changes during the game?
5. What will happen to the blue LED if its resistor is removed? Could a player still win the blue rounds?
