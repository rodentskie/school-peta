# Activity 03: Non-Blocking Morse Code Transmitter

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-03`

## Problem Statement

Make a Morse code transmitter. Text typed in the **Serial Monitor** is converted to Morse code and sent with an LED and a buzzer.

Requirements:

1. Support letters **A–Z** and digits **0–9** (lowercase is converted to uppercase). Unknown symbols are skipped.
2. Use correct Morse timing: dot = 1 unit, dash = 3 units, gap inside a letter = 1 unit, gap between letters = 3 units, gap between words = 7 units (1 unit = 150 ms).
3. The transmitter must be **non-blocking**. New text can be typed while a message is still being sent, and it is added to the end of the queue.
4. Print each letter and its code to the Serial Monitor as it is sent, for example `S(...) O(---) S(...)`.
5. On power-up, automatically send `SOS`.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | LED + 1 kΩ resistor |
| 1 | Piezo buzzer |

## Solution

* A lookup table `MORSE[36]` stores the codes. Index 0–25 is `A`–`Z` and 26–35 is `0`–`9`.
* A `String queue` holds the characters waiting to be sent.
* The transmitter is a small state machine driven by `nextAt` (the time of the next change):
  * **key ON**: the element (dot or dash) is sounding. When its time ends, switch OFF and wait 1 unit, or 3 units if the letter is finished.
  * **key OFF**: when the gap ends, load the next element, letter, or word space and switch ON again.
* A space adds 4 more units to the 3-unit letter gap, giving the 7-unit word gap.
* Because `loop()` never waits, `Serial.read()` is checked continuously and new text can be added at any time.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LED | Anode | **D8** |
| LED | Cathode | 1 kΩ → GND rail |
| Piezo | + | **D9** |
| Piezo | − | GND rail |

```
D8 ──► LED(+) ── LED(−) ──[1kΩ]── GND
D9 ──► PIEZO(+) ── PIEZO(−) ───── GND
```

## Code

```cpp
// Activity 03 - Non-Blocking Morse Code Transmitter
const int LED_PIN = 8;
const int BUZZER_PIN = 9;
const unsigned int UNIT = 150;      // ms
const unsigned int TONE_HZ = 700;

const char* const MORSE[36] = {
  ".-", "-...", "-.-.", "-..", ".", "..-.", "--.", "....", "..", ".---",
  "-.-", ".-..", "--", "-.", "---", ".--.", "--.-", ".-.", "...", "-",
  "..-", "...-", ".--", "-..-", "-.--", "--..",
  "-----", ".----", "..---", "...--", "....-", ".....", "-....", "--...", "---..", "----."
};

String queue = "SOS";
unsigned int charIdx = 0;
const char* code = 0;       // code of the letter being sent (0 = none)
int elemIdx = 0;
bool keyOn = false;
unsigned long nextAt = 0;
String inputLine = "";

int lookup(char c) {
  if (c >= 'A' && c <= 'Z') return c - 'A';
  if (c >= '0' && c <= '9') return 26 + (c - '0');
  return -1;
}

void setKey(bool on) {
  keyOn = on;
  digitalWrite(LED_PIN, on ? HIGH : LOW);
  if (on) tone(BUZZER_PIN, TONE_HZ);
  else noTone(BUZZER_PIN);
}

void transmitter() {
  unsigned long now = millis();
  if ((long)(now - nextAt) < 0) return;     // still waiting

  if (keyOn) {                               // element finished
    setKey(false);
    elemIdx++;
    if (code[elemIdx] == '\0') {             // letter finished
      code = 0;
      charIdx++;
      nextAt = now + 3UL * UNIT;
    } else {
      nextAt = now + UNIT;
    }
    return;
  }

  if (code == 0) {                           // need a new letter
    if (charIdx >= queue.length()) {
      if (queue.length() > 0) {
        Serial.println(" [sent]");
        queue = "";
        charIdx = 0;
      }
      return;
    }
    char c = toupper(queue[charIdx]);
    if (c == ' ') {
      Serial.print(" / ");
      charIdx++;
      nextAt = now + 4UL * UNIT;            // 3 + 4 = 7 units word gap
      return;
    }
    int idx = lookup(c);
    if (idx < 0) { charIdx++; return; }     // skip unsupported symbol
    code = MORSE[idx];
    elemIdx = 0;
    Serial.print(c);
    Serial.print('(');
    Serial.print(code);
    Serial.print(") ");
  }

  setKey(true);
  nextAt = now + (code[elemIdx] == '-' ? 3UL * UNIT : UNIT);
}

void readSerial() {
  while (Serial.available() > 0) {
    char c = Serial.read();
    if (c == '\n' || c == '\r') {
      if (inputLine.length() > 0) {
        if (queue.length() > 0) queue += ' ';
        queue += inputLine;
        Serial.print("\n<queued: ");
        Serial.print(inputLine);
        Serial.println(">");
        inputLine = "";
      }
    } else {
      inputLine += c;
    }
  }
}

void setup() {
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Morse transmitter ready. Type text and press Enter.");
}

void loop() {
  readSerial();
  transmitter();
}
```

## Assessment Questions

1. What do you see and hear right after the simulation starts?
2. How can you tell the difference between a dot and a dash just by watching the LED?
3. What happens if you type a new word while the Arduino is still sending a message?
4. What happens when you type a symbol that is not a letter or number, like `?` or `@`?
5. What will happen to the LED if its resistor is removed? Will the buzzer still work?
