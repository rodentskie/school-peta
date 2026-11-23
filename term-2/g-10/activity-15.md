# Activity 15: Simon Memory Game with High Score in EEPROM

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-15`

## Problem Statement

Program a 3-color **memory game**. Each color has an LED, a button, and its own tone.

1. The game shows a random sequence of colors. Each round adds **one** new color to the end of the same sequence.
2. The player must repeat the whole sequence using the buttons. Each button lights its LED and plays its tone.
3. The playback speed **increases** with each level (shorter flashes).
4. If the player presses a wrong button, or waits more than **3 seconds**, the game is over. Play a "fail" sound and show the score (the number of completed rounds).
5. The **high score** is saved in **EEPROM** and survives power loss. Celebrate a new record with a melody.
6. The maximum sequence length is 32. Reaching it means the player wins.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 3 | LEDs (red, green, blue) + 3 × 1 kΩ resistors |
| 3 | Pushbuttons |
| 1 | Piezo buzzer |

## Solution

* `byte sequence[32]` holds the colors (0–2). It is filled with `random(3)` at the start of the game.
* **Show phase:** for i = 0 … level−1, flash `sequence[i]`. The flash time is `max(150, 500 − level × 30)` ms.
* **Input phase:** `waitForButton(timeout)` returns the color index of the next press (0–2), or −1 on timeout. It waits for the release too, so one press counts once.
* Compare each press with `sequence[i]`. Any mismatch → `gameOver()`.
* **EEPROM:** address 0 stores the high score. A fresh EEPROM contains 255, which is treated as 0.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Red LED | Anode | **D8** |
| Green LED | Anode | **D9** |
| Blue LED | Anode | **D10** |
| LED cathodes | — | 1 kΩ → GND rail |
| Red button | T1 / T2 | GND / **D2** |
| Green button | T1 / T2 | GND / **D3** |
| Blue button | T1 / T2 | GND / **D4** |
| Piezo | + / − | **D11** / GND |

```
D8  ─► RED   ─[1k]─ GND     D2 ─► BTN RED   ── GND
D9  ─► GREEN ─[1k]─ GND     D3 ─► BTN GREEN ── GND
D10 ─► BLUE  ─[1k]─ GND     D4 ─► BTN BLUE  ── GND
D11 ─► PIEZO ── GND
```

## Code

```cpp
// Activity 15 - Simon Memory Game
#include <EEPROM.h>

const int LED_PINS[3] = {8, 9, 10};
const int BTN_PINS[3] = {2, 3, 4};
const int TONES[3] = {262, 330, 392};      // C4, E4, G4
const char* COLORS[3] = {"RED", "GREEN", "BLUE"};
const int BUZZER_PIN = 11;
const int MAX_LEN = 32;
const unsigned long INPUT_TIMEOUT = 3000;

byte sequence[MAX_LEN];
int level = 0;
int highScore = 0;

void flashColor(int c, int ms) {
  digitalWrite(LED_PINS[c], HIGH);
  tone(BUZZER_PIN, TONES[c]);
  delay(ms);
  digitalWrite(LED_PINS[c], LOW);
  noTone(BUZZER_PIN);
}

// returns 0..2, or -1 on timeout
int waitForButton(unsigned long timeoutMs) {
  unsigned long start = millis();
  while (millis() - start < timeoutMs) {
    for (int c = 0; c < 3; c++) {
      if (digitalRead(BTN_PINS[c]) == LOW) {
        delay(20);                                    // debounce
        if (digitalRead(BTN_PINS[c]) != LOW) continue;
        digitalWrite(LED_PINS[c], HIGH);
        tone(BUZZER_PIN, TONES[c]);
        while (digitalRead(BTN_PINS[c]) == LOW) {}    // wait for release
        delay(20);
        digitalWrite(LED_PINS[c], LOW);
        noTone(BUZZER_PIN);
        return c;
      }
    }
  }
  return -1;
}

void playMelody(const int *notes, int n, int ms) {
  for (int i = 0; i < n; i++) {
    tone(BUZZER_PIN, notes[i], ms);
    delay(ms + 30);
  }
}

void newGame() {
  for (int i = 0; i < MAX_LEN; i++) sequence[i] = random(3);
  level = 1;
  Serial.println("\n=== NEW GAME ===");
  for (int k = 0; k < 2; k++) {                       // "get ready" flash
    for (int c = 0; c < 3; c++) digitalWrite(LED_PINS[c], HIGH);
    delay(200);
    for (int c = 0; c < 3; c++) digitalWrite(LED_PINS[c], LOW);
    delay(200);
  }
  delay(500);
}

void gameOver(int score) {
  const int fail[] = {300, 250, 200, 150};
  playMelody(fail, 4, 200);
  Serial.print("GAME OVER - score: ");
  Serial.println(score);
  if (score > highScore) {
    highScore = score;
    EEPROM.update(0, highScore);
    const int win[] = {523, 659, 784, 1047};
    playMelody(win, 4, 120);
    Serial.println("*** NEW HIGH SCORE! ***");
  }
  Serial.print("High score: ");
  Serial.println(highScore);
  Serial.println("Press any button to play again.");
  while (waitForButton(60000) < 0) {}
  newGame();
}

void setup() {
  for (int c = 0; c < 3; c++) {
    pinMode(LED_PINS[c], OUTPUT);
    pinMode(BTN_PINS[c], INPUT_PULLUP);
  }
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  randomSeed(analogRead(A0) ^ (analogRead(A1) << 3));
  highScore = EEPROM.read(0);
  if (highScore == 255) highScore = 0;                // erased EEPROM
  Serial.print("SIMON - High score: ");
  Serial.println(highScore);
  newGame();
}

void loop() {
  // 1) show the sequence
  int flashMs = max(150, 500 - level * 30);
  Serial.print("Level ");
  Serial.print(level);
  Serial.print(": ");
  for (int i = 0; i < level; i++) {
    Serial.print(COLORS[sequence[i]]);
    Serial.print(' ');
    flashColor(sequence[i], flashMs);
    delay(flashMs / 2);
  }
  Serial.println();

  // 2) read the player's answer
  for (int i = 0; i < level; i++) {
    int c = waitForButton(INPUT_TIMEOUT);
    if (c != sequence[i]) {
      if (c < 0) Serial.println("Too slow!");
      else {
        Serial.print("Wrong! You pressed ");
        Serial.print(COLORS[c]);
        Serial.print(", expected ");
        Serial.println(COLORS[sequence[i]]);
      }
      gameOver(level - 1);
      return;
    }
  }

  // 3) next level
  if (level == MAX_LEN) {
    Serial.println("YOU WIN! Perfect memory!");
    gameOver(level);
    return;
  }
  level++;
  delay(700);
}
```

## Assessment Questions

1. What happens on the LEDs at the start of a new game?
2. What happens when you repeat the sequence correctly? What happens when you press a wrong button?
3. What happens if you wait too long before pressing a button?
4. Does the game change as the level goes up? How can you tell?
5. What will happen if the green LED's resistor is removed? Can you still play the game?
