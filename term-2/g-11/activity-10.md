# Activity 10: Serial Character Inspector (ASCII on LEDs)

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-10`
**Control:** Serial Monitor (type characters and click **Send**)

## Problem Statement

Every character a computer sends is really a number, its **ASCII code**. Build a "character inspector" that shows the code of each character you send.

1. For each character received, print a line like:
   `'A' -> dec 65 | hex 0x41 | bin 0100 0001 | UPPERCASE LETTER`
   The class is one of: UPPERCASE LETTER, lowercase letter, DIGIT, SPACE, PUNCTUATION/SYMBOL, CONTROL.
2. Show the 8-bit code on **4 LEDs** in two steps: first the **high nibble** (upper 4 bits) for 1 s, then the **low nibble** for 1 s, with a short blank gap between them.
3. Characters are **queued** (up to 32). Sending `Hi!` shows `H`, `i` and `!` one after another.
4. Special commands start with `\` (backslash): `\s` = toggle slow/fast display (1000 ms / 300 ms), `\c` = clear the queue, `\n` must **not** be treated as a command (the Enter key is ignored).
5. The display must be **non-blocking**. Characters typed while the LEDs are busy are still printed immediately.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 4 | LEDs + 4 × 1 kΩ resistors |

## Solution

* **Nibbles:** `high = c >> 4`, `low = c & 0x0F`. Each nibble is shown on the 4 LEDs (bit 0 → LED on D4 … bit 3 → LED on D7).
* **Classification** uses ASCII ranges: `'A'–'Z'`, `'a'–'z'`, `'0'–'9'`, 32 (space), 33–126 (other printable), and below 32 or 127 (control).
* **Queue:** a ring buffer `char q[32]` with head, tail and count.
* **Display state machine** with phases: 0 = idle, 1 = high nibble, 2 = gap, 3 = low nibble, 4 = gap. `phaseEnd` stores when the phase ends.
* **Backslash commands:** when `\` arrives, set `escape = true`. The next character is treated as a command instead of being displayed.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LED bit 0 | Anode | **D4** |
| LED bit 1 | Anode | **D5** |
| LED bit 2 | Anode | **D6** |
| LED bit 3 | Anode | **D7** |
| LED cathodes | — | 1 kΩ → GND rail |

```
D4 ─► LED b0 ─[1k]─ GND     D6 ─► LED b2 ─[1k]─ GND
D5 ─► LED b1 ─[1k]─ GND     D7 ─► LED b3 ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 10 - Serial Character Inspector
const int LED_PINS[4] = {4, 5, 6, 7};
const int QSIZE = 32;

char q[QSIZE];
int qHead = 0, qTail = 0, qCount = 0;
bool escape = false;
unsigned int showMs = 1000;

int phase = 0;              // 0 idle, 1 high, 2 gap, 3 low, 4 gap
unsigned long phaseEnd = 0;
char current = 0;

void showNibble(byte n) {
  for (int i = 0; i < 4; i++) digitalWrite(LED_PINS[i], (n >> i) & 1);
}

const char* classify(char c) {
  if (c >= 'A' && c <= 'Z') return "UPPERCASE LETTER";
  if (c >= 'a' && c <= 'z') return "lowercase letter";
  if (c >= '0' && c <= '9') return "DIGIT";
  if (c == ' ') return "SPACE";
  if (c > 32 && c < 127) return "PUNCTUATION/SYMBOL";
  return "CONTROL";
}

void printNibbleBits(byte n) {
  for (int i = 3; i >= 0; i--) Serial.print((n >> i) & 1);
}

void describe(char c) {
  byte v = (byte)c;
  Serial.print('\'');
  if (v >= 32 && v < 127) Serial.print(c);
  else Serial.print('?');
  Serial.print("' -> dec ");
  Serial.print(v);
  Serial.print(" | hex 0x");
  if (v < 16) Serial.print('0');
  Serial.print(v, HEX);
  Serial.print(" | bin ");
  printNibbleBits(v >> 4);
  Serial.print(' ');
  printNibbleBits(v & 0x0F);
  Serial.print(" | ");
  Serial.println(classify(c));
}

void enqueue(char c) {
  if (qCount >= QSIZE) {
    Serial.println("(queue full)");
    return;
  }
  q[qTail] = c;
  qTail = (qTail + 1) % QSIZE;
  qCount++;
}

void handle(char c) {
  if (c == '\n' || c == '\r') return;
  if (escape) {
    escape = false;
    if (c == 's') {
      showMs = (showMs == 1000) ? 300 : 1000;
      Serial.print("Display time per nibble: ");
      Serial.print(showMs);
      Serial.println(" ms");
    } else if (c == 'c') {
      qHead = qTail = qCount = 0;
      Serial.println("Queue cleared");
    } else {
      Serial.println("Unknown \\ command (use \\s or \\c)");
    }
    return;
  }
  if (c == '\\') {
    escape = true;
    return;
  }
  describe(c);
  enqueue(c);
}

void display() {
  unsigned long now = millis();
  if (phase != 0 && (long)(now - phaseEnd) < 0) return;

  if (phase == 0) {
    if (qCount == 0) return;
    current = q[qHead];
    qHead = (qHead + 1) % QSIZE;
    qCount--;
    showNibble((byte)current >> 4);
    phase = 1;
    phaseEnd = now + showMs;
  } else if (phase == 1) {
    showNibble(0);
    phase = 2;
    phaseEnd = now + 150;
  } else if (phase == 2) {
    showNibble((byte)current & 0x0F);
    phase = 3;
    phaseEnd = now + showMs;
  } else if (phase == 3) {
    showNibble(0);
    phase = 4;
    phaseEnd = now + 400;
  } else {
    phase = 0;
  }
}

void setup() {
  for (int i = 0; i < 4; i++) pinMode(LED_PINS[i], OUTPUT);
  Serial.begin(9600);
  Serial.println("Character inspector: type anything. \\s = speed, \\c = clear");
  handle('A');
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  display();
}
```

## Assessment Questions

1. Send `A`. What does the Serial Monitor print, and which LEDs light up first and second?
2. Send `Hi!`. Are the three characters printed at the same time? Are they shown on the LEDs at the same time?
3. Send `a` and then `A`. Which LED pattern is different between the two?
4. Send `\s` and then a few characters. What changes in the LEDs? What happens if you send `\x`?
5. What will happen to the LED on D7 if its resistor is removed? Which characters would be hard to read without that LED?
