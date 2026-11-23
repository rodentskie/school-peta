# Activity 17: Keypad Door Lock with Code Change and Lockout

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-17`

## Problem Statement

Build an electronic door lock using a **4×4 keypad** and a **servo** as the bolt.

1. The user types a 4-digit code and presses **`#`** to submit. **`*`** clears the input.
2. Correct code → servo turns to 90° (unlocked) and the green LED lights. After **5 s** the door locks again automatically (0°).
3. Wrong code → the red LED flashes and an error beep sounds. After **3 wrong attempts in a row**, the keypad is **locked out for 30 s**. The red LED blinks and all keys are ignored.
4. **Change code:** press **`A`**, enter the current code + `#`, then the new code + `#`, then the new code again + `#`. If both entries match, save the new code in **EEPROM**.
5. Each key press gives a short click. The typed digits are shown as `*` on the Serial Monitor, never in plain text.
6. The default code is `2580` (used when the EEPROM is empty).

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Keypad 4×4 |
| 1 | Micro Servo |
| 2 | LEDs (green, red) + 2 × 1 kΩ resistors |
| 1 | Piezo buzzer |

## Solution

* **Keypad library:** `keypad.getKey()` returns the pressed character, or `NO_KEY` (0). Scanning rows and columns is handled by the library.
* **Input buffer:** `char input[5]` plus `inLen`. Digits are added up to 4. `#` submits the code.
* **Modes:** `NORMAL`, `CHG_OLD` (verify old code), `CHG_NEW1`, `CHG_NEW2`.
* **Timers:** `openUntil` (auto-lock) and `lockoutUntil` (penalty) are compared with `millis()`.
* **EEPROM:** addresses 0–3 hold the code digits, and address 4 holds the "valid" marker 42. Without the marker, the default code is used.

## Pin Diagram

| Keypad pin | Arduino |
|---|---|
| Row 1, Row 2, Row 3, Row 4 | **D9, D8, D7, D6** |
| Column 1, Column 2, Column 3, Column 4 | **D5, D4, D3, D2** |

| Other | Pin | Connects to |
|---|---|---|
| Micro Servo | Signal | **D10** |
| Micro Servo | Power / Ground | 5V rail / GND rail |
| Green LED | Anode | **A1** (used as digital output) |
| Red LED | Anode | **A2** (used as digital output) |
| LED cathodes | — | 1 kΩ → GND rail |
| Piezo | + / − | **D11** / GND |

```
KEYPAD R1..R4 ─► D9 D8 D7 D6     C1..C4 ─► D5 D4 D3 D2
SERVO signal ─► D10 (power from rails)
A1 ─► GREEN ─[1k]─ GND     A2 ─► RED ─[1k]─ GND
D11 ─► PIEZO ── GND
```

## Code

```cpp
// Activity 17 - Keypad Door Lock
#include <Keypad.h>
#include <Servo.h>
#include <EEPROM.h>

const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1', '2', '3', 'A'},
  {'4', '5', '6', 'B'},
  {'7', '8', '9', 'C'},
  {'*', '0', '#', 'D'}
};
byte rowPins[ROWS] = {9, 8, 7, 6};
byte colPins[COLS] = {5, 4, 3, 2};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

const int SERVO_PIN = 10;
const int GREEN_PIN = A1;
const int RED_PIN = A2;
const int BUZZER_PIN = 11;

Servo bolt;
char code[5] = "2580";
char input[5];
char newCode[5];
int inLen = 0;
int wrongCount = 0;

const int NORMAL = 0;
const int CHG_OLD = 1;
const int CHG_NEW1 = 2;
const int CHG_NEW2 = 3;
int mode = NORMAL;

bool isOpen = false;
unsigned long openUntil = 0;
unsigned long lockoutUntil = 0;
unsigned long redFlashUntil = 0;

void loadCode() {
  if (EEPROM.read(4) == 42) {
    for (int i = 0; i < 4; i++) code[i] = EEPROM.read(i);
    code[4] = '\0';
  }
}

void saveCode() {
  for (int i = 0; i < 4; i++) EEPROM.update(i, code[i]);
  EEPROM.update(4, 42);
}

void clearInput() {
  inLen = 0;
  input[0] = '\0';
}

void prompt() {
  if (mode == NORMAL) Serial.println("Enter code then #");
  else if (mode == CHG_OLD) Serial.println("CHANGE: enter CURRENT code then #");
  else if (mode == CHG_NEW1) Serial.println("CHANGE: enter NEW code then #");
  else Serial.println("CHANGE: repeat NEW code then #");
}

void unlock() {
  isOpen = true;
  openUntil = millis() + 5000;
  bolt.write(90);
  digitalWrite(GREEN_PIN, HIGH);
  wrongCount = 0;
  Serial.println("ACCESS GRANTED - door open for 5 s");
}

void wrong() {
  wrongCount++;
  redFlashUntil = millis() + 800;
  tone(BUZZER_PIN, 300, 400);
  Serial.print("WRONG CODE (");
  Serial.print(wrongCount);
  Serial.println("/3)");
  if (wrongCount >= 3) {
    lockoutUntil = millis() + 30000;
    wrongCount = 0;
    tone(BUZZER_PIN, 200, 1000);
    Serial.println("LOCKED OUT for 30 seconds!");
  }
}

void submit() {
  bool full = (inLen == 4);
  bool match = full && strcmp(input, code) == 0;

  if (mode == NORMAL) {
    if (match) unlock();
    else wrong();
  } else if (mode == CHG_OLD) {
    if (match) { mode = CHG_NEW1; }
    else { mode = NORMAL; wrong(); }
  } else if (mode == CHG_NEW1) {
    if (full) { strcpy(newCode, input); mode = CHG_NEW2; }
    else { Serial.println("Code must have 4 digits"); }
  } else if (mode == CHG_NEW2) {
    if (full && strcmp(input, newCode) == 0) {
      strcpy(code, newCode);
      saveCode();
      Serial.println("CODE CHANGED and saved");
    } else {
      Serial.println("Codes do not match - not changed");
    }
    mode = NORMAL;
  }
  clearInput();
  prompt();
}

void setup() {
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(RED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  bolt.attach(SERVO_PIN);
  bolt.write(0);
  loadCode();
  clearInput();
  Serial.println("=== DOOR LOCK ===  (A = change code, * = clear)");
  prompt();
}

void loop() {
  unsigned long now = millis();

  // auto re-lock
  if (isOpen && (long)(now - openUntil) >= 0) {
    isOpen = false;
    bolt.write(0);
    digitalWrite(GREEN_PIN, LOW);
    Serial.println("Door locked");
  }

  bool lockedOut = (long)(lockoutUntil - now) > 0;
  if (lockedOut) digitalWrite(RED_PIN, (now / 250) % 2);
  else digitalWrite(RED_PIN, (long)(redFlashUntil - now) > 0 ? (now / 100) % 2 : LOW);

  char k = keypad.getKey();
  if (k == NO_KEY) return;
  if (lockedOut) {
    Serial.println("Keypad locked - please wait");
    return;
  }
  tone(BUZZER_PIN, 2000, 30);

  if (k >= '0' && k <= '9') {
    if (inLen < 4) {
      input[inLen++] = k;
      input[inLen] = '\0';
    }
    Serial.print("Input: ");
    for (int i = 0; i < inLen; i++) Serial.print('*');
    Serial.println();
  } else if (k == '*') {
    clearInput();
    Serial.println("Cleared");
  } else if (k == '#') {
    submit();
  } else if (k == 'A' && mode == NORMAL) {
    mode = CHG_OLD;
    clearInput();
    prompt();
  }
}
```

## Assessment Questions

1. How many different 4-digit codes are possible? With 3 attempts every 30 seconds, how long would it take in the worst case to try them all?
2. Explain why the digits are displayed as `*`. Name one other security weakness of this lock that a real product should fix.
3. Why does the program store the marker value **42** at EEPROM address 4? What would happen on a brand-new Arduino without it?
4. Trace the values of `mode`, `input` and `newCode` when a user presses: `A 2 5 8 0 # 1 1 1 1 # 1 1 1 1 #`.
5. Add an **alarm**: after 2 lockouts in a row, the red LED stays on and the lock can only be reset by holding `D` for 3 seconds. Explain which variables and checks you need.
