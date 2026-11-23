# Activity 13: Electronic Dice on a 7-Segment Display with Statistics

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-13`

## Problem Statement

Build an electronic dice using a **7-segment display**.

1. When the button is pressed, show a **rolling animation**. The digits change quickly at first and then **slow down** (20 ms → about 350 ms between changes) over 15 steps, with a click sound on each step.
2. The final number is random **1–6**. It blinks 3 times and then stays on.
3. Control the segments with a **lookup table** of bit patterns (one byte per digit) and a single `showDigit()` function.
4. Keep **statistics**: count how many times each face appeared and print a table with the count and percentage after every roll.
5. Ignore button presses while a roll is in progress.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | 7-Segment Display (common cathode) + 1 kΩ resistor on the common pin |
| 1 | Pushbutton |
| 1 | Piezo buzzer |

## Solution

**Segment encoding:** each digit is one byte. Bit 0 = segment a, bit 1 = b … bit 6 = g:

```
   aaa
  f   b
   ggg
  e   c
   ddd
```

| Digit | Segments | Byte |
|---|---|---|
| 1 | b c | `0b0000110` |
| 2 | a b d e g | `0b1011011` |
| 3 | a b c d g | `0b1001111` |
| 4 | b c f g | `0b1100110` |
| 5 | a c d f g | `0b1101101` |
| 6 | a c d e f g | `0b1111101` |

`showDigit()` loops over the 7 bits and writes each one to its pin. If the display is common-anode, set `COMMON_ANODE` to `true` and the logic is inverted.

**Slowing animation:** the waiting time grows each step: `wait = 20 + step² × 1.5`. The roll is a state machine (`rolling`, `step`, `nextStepAt`), so `loop()` never blocks.

## Pin Diagram

| 7-Segment pin | Arduino |
|---|---|
| a | **D2** |
| b | **D3** |
| c | **D4** |
| d | **D5** |
| e | **D6** |
| f | **D7** |
| g | **D8** |
| Common | 1 kΩ → GND rail |

| Other | Pin | Connects to |
|---|---|---|
| Pushbutton | T1 / T2 | GND / **D10** |
| Piezo | + / − | **D11** / GND |

```
D2..D8 ─► segments a..g      COMMON ─[1k]─ GND
D10 ─► BUTTON ── GND          D11 ─► PIEZO ── GND
```

## Code

```cpp
// Activity 13 - Electronic Dice with Statistics
const int SEG_PINS[7] = {2, 3, 4, 5, 6, 7, 8};   // a b c d e f g
const int BTN_PIN = 10;
const int BUZZER_PIN = 11;
const bool COMMON_ANODE = false;

const byte DIGITS[10] = {
  0b0111111, 0b0000110, 0b1011011, 0b1001111, 0b1100110,
  0b1101101, 0b1111101, 0b0000111, 0b1111111, 0b1101111
};

bool rolling = false;
int step = 0;
const int STEPS = 15;
unsigned long nextStepAt = 0;
int result = 0;
int blinkCount = 0;

unsigned int counts[7] = {0};    // index 1..6 used
unsigned int totalRolls = 0;

int btnPrev = HIGH;
unsigned long btnEdge = 0;

void showPattern(byte p) {
  for (int i = 0; i < 7; i++) {
    bool on = (p >> i) & 1;
    if (COMMON_ANODE) on = !on;
    digitalWrite(SEG_PINS[i], on ? HIGH : LOW);
  }
}

void showDigit(int d) {
  showPattern(DIGITS[d]);
}

bool buttonPressed() {
  int r = digitalRead(BTN_PIN);
  bool pressed = false;
  if (r != btnPrev && millis() - btnEdge > 40) {
    btnEdge = millis();
    if (r == LOW) pressed = true;
    btnPrev = r;
  }
  return pressed;
}

void printStats() {
  Serial.print("Rolled: ");
  Serial.print(result);
  Serial.print("   (total rolls: ");
  Serial.print(totalRolls);
  Serial.println(")");
  for (int f = 1; f <= 6; f++) {
    Serial.print("  ");
    Serial.print(f);
    Serial.print(": ");
    Serial.print(counts[f]);
    Serial.print("  ");
    Serial.print(100.0 * counts[f] / totalRolls, 1);
    Serial.print("%  ");
    for (unsigned int k = 0; k < counts[f] && k < 30; k++) Serial.print('*');
    Serial.println();
  }
  Serial.println();
}

void setup() {
  for (int i = 0; i < 7; i++) pinMode(SEG_PINS[i], OUTPUT);
  pinMode(BTN_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  randomSeed(analogRead(A0) + analogRead(A1));
  showPattern(0b1000000);          // "-" while waiting
  Serial.println("Electronic dice - press the button to roll");
}

void loop() {
  unsigned long now = millis();

  if (!rolling && buttonPressed()) {
    rolling = true;
    step = 0;
    blinkCount = 0;
    nextStepAt = now;
  }

  if (rolling && (long)(now - nextStepAt) >= 0) {
    if (step < STEPS) {
      int face = random(1, 7);
      showDigit(face);
      tone(BUZZER_PIN, 900, 8);
      step++;
      nextStepAt = now + 20 + (unsigned long)(step * step * 1.5);
    } else if (step == STEPS) {
      result = random(1, 7);
      counts[result]++;
      totalRolls++;
      step++;
      nextStepAt = now;
    } else if (blinkCount < 6) {             // 3 blinks = 6 toggles
      if (blinkCount % 2 == 0) showPattern(0);
      else showDigit(result);
      blinkCount++;
      nextStepAt = now + 150;
    } else {
      showDigit(result);
      tone(BUZZER_PIN, 1500, 120);
      rolling = false;
      printStats();
    }
  }
}
```

## Assessment Questions

1. Describe what you see and hear when you press the button.
2. What happens if you press the button again while the dice is still rolling?
3. What does the display show before the first roll?
4. What will happen to the display if the resistor on the common pin is removed?
5. If the wire to segment g (D8) is disconnected, which numbers will look wrong? What will the number 6 look like?
