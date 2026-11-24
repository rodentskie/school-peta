# Activity 11: Serial Morse Code Decoder with LCD Output

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-11`
**Control:** Serial Monitor (type characters and click **Send**)

## Problem Statement

This time the Arduino **receives** Morse code and translates it into text.

| Character | Meaning |
|---|---|
| `.` | dot |
| `-` | dash |
| `/` or space | end of a letter |
| `_` | end of a word (adds a space) |
| `<` | delete the last decoded letter |
| `#` | clear everything |

1. Every dot and dash is echoed with the LED and the buzzer (short beep for dot, long beep for dash) so the operator can hear what they sent.
2. When a letter ends, look the code up in a table (A–Z, 0–9). Add the letter to the message, or `?` if the code is unknown.
3. **LCD line 1:** the decoded message (last 16 characters). **LCD line 2:** the code of the letter currently being typed, e.g. `Code: .-.`.
4. Codes longer than 6 symbols are invalid. Print `Code too long` and discard them.
5. The decoded message is also printed on the Serial Monitor after every letter.

Example: sending `.../---/.../` shows `SOS`.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | LCD 16 × 2 (I2C) |
| 1 | LED + 1 kΩ resistor |
| 1 | Piezo buzzer |

## Solution

* **Current code:** `char code[7]` and `codeLen` collect dots and dashes.
* **Lookup:** loop through `MORSE[36]` and compare with `strcmp()`. Index 0–25 is `A`–`Z`, and 26–35 is `0`–`9`.
* **Message:** `char msg[41]` with `msgLen`. Line 1 shows the last 16 characters, so the text scrolls.
* **Echo:** `tone(pin, f, duration)` plays without blocking. The LED is switched off at `ledOffAt`.
* The LCD is redrawn only after a change, using a function that pads with spaces (no `clear()`, no flicker).

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LCD (I2C) | GND / VCC | GND rail / 5V rail |
| LCD (I2C) | SDA / SCL | **A4** / **A5** |
| LED | Anode | **D8** (cathode → 1 kΩ → GND) |
| Piezo | + / − | **D9** / GND |

```
LCD: GND─GND VCC─5V SDA─A4 SCL─A5
D8 ─► LED ─[1k]─ GND      D9 ─► PIEZO ── GND
```

## Code

```cpp
// G11 Activity 11 - Serial Morse Decoder with LCD
#include <Adafruit_LiquidCrystal.h>
#include <string.h>

Adafruit_LiquidCrystal lcd(0);

const int LED_PIN = 8;
const int BUZZER_PIN = 9;

const char* const MORSE[36] = {
  ".-", "-...", "-.-.", "-..", ".", "..-.", "--.", "....", "..", ".---",
  "-.-", ".-..", "--", "-.", "---", ".--.", "--.-", ".-.", "...", "-",
  "..-", "...-", ".--", "-..-", "-.--", "--..",
  "-----", ".----", "..---", "...--", "....-", ".....", "-....", "--...", "---..", "----."
};

char code[7];
int codeLen = 0;
char msg[41];
int msgLen = 0;
unsigned long ledOffAt = 0;

char decode(const char *c) {
  for (int i = 0; i < 36; i++) {
    if (strcmp(c, MORSE[i]) == 0) return i < 26 ? 'A' + i : '0' + (i - 26);
  }
  return '?';
}

void redraw() {
  lcd.setCursor(0, 0);
  int start = msgLen > 16 ? msgLen - 16 : 0;
  for (int i = 0; i < 16; i++) {
    int k = start + i;
    lcd.print(k < msgLen ? msg[k] : ' ');
  }
  lcd.setCursor(0, 1);
  lcd.print("Code: ");
  lcd.print(code);
  for (int i = codeLen; i < 10; i++) lcd.print(' ');
}

void addToMessage(char ch) {
  if (msgLen >= 40) {                 // drop oldest character
    for (int i = 0; i < 39; i++) msg[i] = msg[i + 1];
    msgLen = 39;
  }
  msg[msgLen++] = ch;
  msg[msgLen] = '\0';
  Serial.print("Message: ");
  Serial.println(msg);
}

void endLetter() {
  if (codeLen == 0) return;
  char ch = decode(code);
  Serial.print(code);
  Serial.print(" = ");
  Serial.println(ch);
  addToMessage(ch);
  codeLen = 0;
  code[0] = '\0';
}

void echo(bool dash) {
  digitalWrite(LED_PIN, HIGH);
  int ms = dash ? 300 : 100;
  tone(BUZZER_PIN, 700, ms);
  ledOffAt = millis() + ms;
}

void handle(char c) {
  if (c == '.' || c == '-') {
    if (codeLen >= 6) {
      Serial.println("Code too long - discarded");
      codeLen = 0;
      code[0] = '\0';
    } else {
      code[codeLen++] = c;
      code[codeLen] = '\0';
      echo(c == '-');
    }
  }
  else if (c == '/' || c == ' ') endLetter();
  else if (c == '_') { endLetter(); addToMessage(' '); }
  else if (c == '<') {
    if (msgLen > 0) msg[--msgLen] = '\0';
    Serial.print("Message: ");
    Serial.println(msg);
  }
  else if (c == '#') {
    msgLen = 0; msg[0] = '\0';
    codeLen = 0; code[0] = '\0';
    Serial.println("Cleared");
  }
  else if (c == '\n' || c == '\r') return;
  else {
    Serial.print("Use . - / _ < #  (got '");
    Serial.print(c);
    Serial.println("')");
    return;
  }
  redraw();
}

void setup() {
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  lcd.begin(16, 2);
  lcd.setBacklight(1);
  code[0] = '\0';
  msg[0] = '\0';
  Serial.println("Morse decoder: . dot, - dash, / end letter, _ space, < delete, # clear");
  const char *demo = ".../---/.../";
  for (int i = 0; demo[i]; i++) handle(demo[i]);
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  if (ledOffAt && (long)(millis() - ledOffAt) >= 0) {
    digitalWrite(LED_PIN, LOW);
    ledOffAt = 0;
  }
}
```

## Assessment Questions

1. What does the LCD show right after the simulation starts?
2. Send `.-` (without `/`). What do lines 1 and 2 of the LCD show? What happens when you then send `/`?
3. How is the sound different for `.` and `-`? What does the LED do?
4. Send `.-.-.-.-/`. What happens? What appears for a code that is not in the table, like `..--/`?
5. If the LCD's power wire is disconnected, does the decoder still work? Where can you still see the decoded message?
