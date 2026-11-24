# Activity 25: Serial Pattern Memory Game (4 LEDs)

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-25`
**Control:** Serial Monitor (type characters and click **Send**)

## Problem Statement

A memory game played through the Serial Monitor. The Arduino flashes a sequence on **4 LEDs** (numbered 1–4). The player types the sequence back as digits, e.g. `3142`.

| Key | Action |
|---|---|
| `1`–`4` | answer digits (can be typed in one line) |
| `s` | start a new game |
| `r` | replay the current sequence (costs 1 of your 2 replays per game) |
| `m` | mode: CLASSIC (each level adds one step) ↔ SCRAMBLE (a completely new sequence each level) |
| `?` | help |

1. Level *n* has a sequence of *n* + 2 steps (level 1 = 3 steps). Each LED has its own tone.
2. The sequence is shown with **no input allowed**. Digits typed during the show are ignored with the message `Wait for the pattern!`.
3. After the show, the player has **5 s per digit**. Each correct digit is echoed with its LED and tone.
4. A wrong digit or a timeout ends the game. Print the correct sequence, the level reached, and the best level since start-up.
5. Every 3 levels the show gets faster (flash time −20 %, at least 150 ms).

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 4 | LEDs + 4 × 1 kΩ resistors |
| 1 | Piezo buzzer |

## Solution

* **States:** `IDLE`, `SHOWING`, `INPUT`.
* **Non-blocking show:** `showIdx` and `showPhase` (0 = LED on, 1 = gap). The next change happens when `millis()` reaches `nextAt`.
* **Sequence:** `byte seq[20]`. In CLASSIC mode a new random step is appended. In SCRAMBLE mode the whole sequence is re-generated with the new length.
* **Input:** `inputIdx` counts correct digits. `deadline = now + 5000` is renewed after each correct digit.
* The flash time is `600 × 0.8^(level/3)` ms, with a minimum of 150 ms.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LED 1 | Anode | **D4** |
| LED 2 | Anode | **D5** |
| LED 3 | Anode | **D6** |
| LED 4 | Anode | **D7** |
| LED cathodes | — | 1 kΩ → GND rail |
| Piezo | + / − | **D9** / GND |

```
D4 ─► LED1 ─[1k]─ GND    D6 ─► LED3 ─[1k]─ GND
D5 ─► LED2 ─[1k]─ GND    D7 ─► LED4 ─[1k]─ GND
D9 ─► PIEZO ── GND
```

## Code

```cpp
// G11 Activity 25 - Serial Pattern Memory Game
const int LED_PINS[4] = {4, 5, 6, 7};
const int TONES[4] = {392, 523, 659, 784};
const int BUZZER_PIN = 9;
const int MAXLEN = 20;

const int IDLE = 0, SHOWING = 1, INPUT_ = 2;
int state = IDLE;
bool scramble = false;
byte seq[MAXLEN];
int len = 0;
int level = 0;
int bestLevel = 0;
int replays = 2;

int showIdx = 0, showPhase = 0;
unsigned long nextAt = 0;
unsigned int flashMs = 600;
int inputIdx = 0;
unsigned long deadline = 0;
unsigned long echoOffAt = 0;

void allOff() {
  for (int i = 0; i < 4; i++) digitalWrite(LED_PINS[i], LOW);
}

void startShow() {
  state = SHOWING;
  showIdx = 0;
  showPhase = 1;               // start with a short gap
  nextAt = millis() + 500;
  allOff();
}

void nextLevel() {
  level++;
  int newLen = min(level + 2, MAXLEN);
  if (scramble) {
    for (int i = 0; i < newLen; i++) seq[i] = random(4);
  } else {
    if (len == 0) for (int i = 0; i < newLen - 1; i++) seq[i] = random(4);
    seq[newLen - 1] = random(4);
  }
  len = newLen;
  flashMs = max(150, (int)(600 * pow(0.8, level / 3)));
  Serial.print("Level ");
  Serial.print(level);
  Serial.print(" - ");
  Serial.print(len);
  Serial.println(" steps. Watch...");
  startShow();
}

void gameOver(const char *why) {
  state = IDLE;
  allOff();
  tone(BUZZER_PIN, 180, 600);
  Serial.println(why);
  Serial.print("Correct sequence was: ");
  for (int i = 0; i < len; i++) Serial.print(seq[i] + 1);
  Serial.println();
  int reached = level - 1;
  if (reached > bestLevel) bestLevel = reached;
  Serial.print("Levels completed: ");
  Serial.print(reached);
  Serial.print("   Best: ");
  Serial.println(bestLevel);
  Serial.println("Press s to play again");
}

void handle(char c) {
  if (c == '\n' || c == '\r' || c == ' ') return;
  if (c == 's') {
    level = 0; len = 0; replays = 2;
    Serial.print("New game (");
    Serial.print(scramble ? "SCRAMBLE" : "CLASSIC");
    Serial.println(")");
    nextLevel();
  }
  else if (c == 'm') {
    if (state != IDLE) { Serial.println("Change mode between games"); return; }
    scramble = !scramble;
    Serial.println(scramble ? "Mode: SCRAMBLE" : "Mode: CLASSIC");
  }
  else if (c == 'r') {
    if (state != INPUT_) { Serial.println("Replay only while answering"); return; }
    if (replays == 0) { Serial.println("No replays left"); return; }
    replays--;
    Serial.print("Replay (");
    Serial.print(replays);
    Serial.println(" left)");
    inputIdx = 0;
    startShow();
  }
  else if (c >= '1' && c <= '4') {
    if (state == SHOWING) { Serial.println("Wait for the pattern!"); return; }
    if (state != INPUT_) { Serial.println("Press s to start"); return; }
    int k = c - '1';
    if (k != seq[inputIdx]) { gameOver("WRONG!"); return; }
    allOff();
    digitalWrite(LED_PINS[k], HIGH);
    tone(BUZZER_PIN, TONES[k], 150);
    echoOffAt = millis() + 200;
    inputIdx++;
    deadline = millis() + 5000;
    if (inputIdx == len) {
      Serial.println("Correct!");
      nextLevel();
    }
  }
  else if (c == '?' || c == 'h') Serial.println("s start | 1-4 answer | r replay | m mode");
  else { Serial.print("Unknown '"); Serial.print(c); Serial.println("'"); }
}

void setup() {
  for (int i = 0; i < 4; i++) pinMode(LED_PINS[i], OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  randomSeed(analogRead(A0) + analogRead(A3));
  Serial.println("PATTERN MEMORY - press s to start, then type the LED numbers (1-4)");
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();

  if (state == SHOWING && (long)(now - nextAt) >= 0) {
    if (showPhase == 1) {                    // gap finished -> next LED on
      if (showIdx >= len) {
        state = INPUT_;
        inputIdx = 0;
        deadline = now + 5000;
        Serial.println("Your turn: type the sequence");
        return;
      }
      digitalWrite(LED_PINS[seq[showIdx]], HIGH);
      tone(BUZZER_PIN, TONES[seq[showIdx]], flashMs);
      showPhase = 0;
      nextAt = now + flashMs;
    } else {                                 // LED on finished -> gap
      allOff();
      showIdx++;
      showPhase = 1;
      nextAt = now + flashMs / 2;
    }
  }

  if (state == INPUT_ && (long)(now - deadline) >= 0) gameOver("TIME OUT!");
  if (echoOffAt && (long)(now - echoOffAt) >= 0) { echoOffAt = 0; if (state == INPUT_) allOff(); }
}
```

## Assessment Questions

1. Send `s`. What do the LEDs and the buzzer do? What does the Serial Monitor ask you to do after that?
2. Type the correct sequence in one line. What happens next? Is the new pattern longer?
3. Type a digit while the pattern is still being shown. What message appears?
4. Type a wrong digit. What does the Serial Monitor show? What happens if you wait more than 5 seconds?
5. If the resistor of LED 3 is removed, what will happen to that LED? Could a player still finish a level that uses LED 3?
