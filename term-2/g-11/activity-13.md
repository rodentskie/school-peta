# Activity 13: Serial Combination Safe with Servo Lock

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-13`
**Control:** Serial Monitor (type characters and click **Send**)

## Problem Statement

Build a safe whose lock (a servo) is opened by typing a code in the Serial Monitor.

| Key | Action |
|---|---|
| `0`–`9` | enter a digit (maximum 6) |
| `#` | submit the code |
| `*` | clear the input |
| `L` | lock the safe immediately |
| `C` | change the code (only while the safe is **open**) |
| `?` | status |

1. The code has 4–6 digits. The default is `1234`. The typed digits are echoed as `*`.
2. **Correct code:** servo turns to 90° (open), green LED on, a happy beep. The safe **auto-locks after 10 s** unless it is locked earlier with `L`.
3. **Wrong code:** red LED flashes and a low beep sounds. After **3 wrong codes**, the safe is blocked for 20 s, then 40 s after the next 3 wrong codes, then 80 s (the lockout **doubles** each time).
4. During a lockout, all keys print `BLOCKED - wait xx s`.
5. **Change code** (`C` while open): type the new code + `#`, then type it again + `#`. Only if both match is it saved in **EEPROM** (with a length byte).

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Micro Servo |
| 2 | LEDs (green, red) + 2 × 1 kΩ resistors |
| 1 | Piezo buzzer |

## Solution

* **Input buffer:** `char input[7]`, `inLen`.
* **Modes:** `NORMAL`, `NEW1` (first new code), `NEW2` (confirm).
* **Lockout:** `failCount` counts wrong codes, and `lockoutLevel` counts lockouts. The lockout length is `20000UL << lockoutLevel` (20 s, 40 s, 80 s …, capped at 320 s). A correct code resets both counters.
* **EEPROM layout:** address 0 = code length (4–6), addresses 1–6 = digits. If the length byte is not 4–6, use the default code.
* **Auto-lock:** `lockAt = now + 10000` when the safe opens.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Micro Servo | Signal | **D10** |
| Micro Servo | Power / Ground | 5V rail / GND rail |
| Green LED | Anode | **D8** |
| Red LED | Anode | **D7** |
| LED cathodes | — | 1 kΩ → GND rail |
| Piezo | + / − | **D9** / GND |

```
D10 ─► SERVO signal (power from rails)
D8 ─► GREEN ─[1k]─ GND     D7 ─► RED ─[1k]─ GND
D9 ─► PIEZO ── GND
```

## Code

```cpp
// G11 Activity 13 - Serial Combination Safe
#include <Servo.h>
#include <EEPROM.h>
#include <string.h>

const int SERVO_PIN = 10;
const int GREEN_PIN = 8;
const int RED_PIN = 7;
const int BUZZER_PIN = 9;

Servo bolt;
char code[7] = "1234";
char input[7];
char first[7];
int inLen = 0;

const int NORMAL = 0, NEW1 = 1, NEW2 = 2;
int mode = NORMAL;
bool isOpen = false;
unsigned long lockAt = 0;
int failCount = 0;
int lockoutLevel = 0;
unsigned long blockedUntil = 0;
unsigned long redUntil = 0;

void loadCode() {
  int len = EEPROM.read(0);
  if (len >= 4 && len <= 6) {
    for (int i = 0; i < len; i++) code[i] = EEPROM.read(1 + i);
    code[len] = '\0';
  }
}

void saveCode() {
  int len = strlen(code);
  EEPROM.update(0, len);
  for (int i = 0; i < len; i++) EEPROM.update(1 + i, code[i]);
}

void clearInput() {
  inLen = 0;
  input[0] = '\0';
}

void openSafe() {
  isOpen = true;
  lockAt = millis() + 10000;
  bolt.write(90);
  digitalWrite(GREEN_PIN, HIGH);
  tone(BUZZER_PIN, 1500, 100);
  failCount = 0;
  lockoutLevel = 0;
  Serial.println("OPEN - auto-lock in 10 s (L = lock now, C = change code)");
}

void lockSafe() {
  isOpen = false;
  mode = NORMAL;
  bolt.write(0);
  digitalWrite(GREEN_PIN, LOW);
  Serial.println("LOCKED");
}

void wrongCode() {
  failCount++;
  redUntil = millis() + 1000;
  tone(BUZZER_PIN, 200, 400);
  Serial.print("WRONG CODE (");
  Serial.print(failCount);
  Serial.println("/3)");
  if (failCount >= 3) {
    unsigned long len = 20000UL << lockoutLevel;
    if (lockoutLevel < 4) lockoutLevel++;
    failCount = 0;
    blockedUntil = millis() + len;
    Serial.print("BLOCKED for ");
    Serial.print(len / 1000);
    Serial.println(" s");
  }
}

void submit() {
  if (mode == NORMAL) {
    if (isOpen) Serial.println("Already open");
    else if (strcmp(input, code) == 0) openSafe();
    else wrongCode();
  } else if (mode == NEW1) {
    if (inLen < 4) Serial.println("New code needs 4-6 digits");
    else { strcpy(first, input); mode = NEW2; Serial.println("Repeat the new code then #"); }
  } else if (mode == NEW2) {
    if (strcmp(input, first) == 0) {
      strcpy(code, first);
      saveCode();
      Serial.println("Code changed and saved");
    } else {
      Serial.println("Codes do not match - not changed");
    }
    mode = NORMAL;
  }
  clearInput();
}

void status() {
  Serial.print(isOpen ? "Safe OPEN" : "Safe LOCKED");
  Serial.print(" | wrong attempts ");
  Serial.print(failCount);
  Serial.print(" | code length ");
  Serial.println(strlen(code));
}

void handle(char c) {
  if (c == '\n' || c == '\r' || c == ' ') return;
  long left = (long)(blockedUntil - millis());
  if (left > 0) {
    Serial.print("BLOCKED - wait ");
    Serial.print(left / 1000 + 1);
    Serial.println(" s");
    return;
  }
  if (c >= '0' && c <= '9') {
    if (inLen < 6) { input[inLen++] = c; input[inLen] = '\0'; }
    Serial.print("Input: ");
    for (int i = 0; i < inLen; i++) Serial.print('*');
    Serial.println();
  }
  else if (c == '#') submit();
  else if (c == '*') { clearInput(); Serial.println("Cleared"); }
  else if (c == 'L') { if (isOpen) lockSafe(); }
  else if (c == 'C') {
    if (!isOpen) Serial.println("Open the safe first");
    else { mode = NEW1; clearInput(); Serial.println("Type NEW code (4-6 digits) then #"); }
  }
  else if (c == '?') status();
  else {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
  }
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
  Serial.println("=== SAFE === type code then #  (* clear, ? status)");
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  unsigned long now = millis();
  if (isOpen && (long)(now - lockAt) >= 0) {
    Serial.print("Auto-");
    lockSafe();
  }
  bool blocked = (long)(blockedUntil - now) > 0;
  if (blocked) digitalWrite(RED_PIN, (now / 400) % 2);
  else digitalWrite(RED_PIN, (long)(redUntil - now) > 0 ? (now / 100) % 2 : LOW);
}
```

## Assessment Questions

1. Send `1234#`. What happens to the servo, the LEDs and the buzzer? What happens 10 seconds later?
2. Send a wrong code three times. What do the red LED and the Serial Monitor show? What happens if you type during this time?
3. Get blocked a second time. Is the waiting time the same as the first time?
4. Open the safe, send `C`, and change the code. Restart the simulation and try the old code and the new code. What happens?
5. If the servo's signal wire is disconnected, what does the Serial Monitor say when the correct code is entered? Is the safe really open?
