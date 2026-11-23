# Activity 01: Serial LED Console with Bit Operations

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-01`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Control 4 LEDs **only** through characters sent from the Serial Monitor. The 4 LEDs are stored as a 4-bit pattern (`byte leds`), and every command changes this pattern with **bitwise operators**.

| Key | Action |
|---|---|
| `1` `2` `3` `4` | toggle LED 1–4 (`^` XOR) |
| `a` | all ON |
| `o` | all OFF |
| `i` | invert all (`~` NOT) |
| `<` / `>` | rotate the pattern left / right (the bit that leaves one end enters the other) |
| `k` | start / stop the **Knight Rider** animation |
| `b` | start / stop the **binary counter** animation (0–15) |
| `+` / `-` | animation faster / slower (50–1000 ms) |
| `s` | print status: pattern in binary, animation, speed |
| `h` or `?` | help |

1. Several characters in one line (e.g. `13>>`) must run as several commands, in order.
2. Newline characters (`\n`, `\r`) are ignored. Unknown characters print `Unknown command 'x' - press h for help`.
3. Animations run **without `delay()`**. Any manual LED command (`1`–`4`, `a`, `o`, `i`, `<`, `>`) stops the running animation.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 4 | LEDs + 4 × 1 kΩ resistors |

## Solution

* **Bitwise operations on `leds`:**
  * toggle LED n: `leds ^= (1 << (n−1))`
  * invert: `leds = ~leds & 0x0F` (keep only 4 bits)
  * rotate left: `leds = ((leds << 1) | (leds >> 3)) & 0x0F`
  * rotate right: `leds = ((leds >> 1) | (leds << 3)) & 0x0F`
* `show()` writes bit *i* of `leds` to LED *i*.
* **Command dispatcher:** `Serial.read()` returns one character, and a `switch` statement selects the action.
* **Animations:** `anim` = 0 (none), 1 (Knight Rider) or 2 (counter). Every `speedMs`, `stepAnimation()` makes one step. Knight Rider moves a single bit back and forth using a direction variable.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LED 1 (bit 0) | Anode | **D4** |
| LED 2 (bit 1) | Anode | **D5** |
| LED 3 (bit 2) | Anode | **D6** |
| LED 4 (bit 3) | Anode | **D7** |
| All cathodes | — | 1 kΩ → GND rail |
| Arduino | 5V / GND | Breadboard + / − rails |

```
D4 ─► LED1 ─[1k]─ GND
D5 ─► LED2 ─[1k]─ GND
D6 ─► LED3 ─[1k]─ GND
D7 ─► LED4 ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 01 - Serial LED Console with Bit Operations
const int LED_PINS[4] = {4, 5, 6, 7};

byte leds = 0;               // bit0 = LED1 ... bit3 = LED4
int anim = 0;                // 0 none, 1 knight rider, 2 counter
int speedMs = 200;
unsigned long lastStep = 0;
int krPos = 0, krDir = 1;

void show() {
  for (int i = 0; i < 4; i++) digitalWrite(LED_PINS[i], (leds >> i) & 1);
}

void printPattern() {
  Serial.print("LEDs [4..1] = ");
  for (int i = 3; i >= 0; i--) Serial.print((leds >> i) & 1);
  Serial.println();
}

void help() {
  Serial.println("--- LED CONSOLE ---");
  Serial.println(" 1-4 toggle | a all on | o all off | i invert");
  Serial.println(" < > rotate | k knight rider | b binary counter");
  Serial.println(" + - speed  | s status | h help");
}

void stopAnim() {
  if (anim != 0) {
    anim = 0;
    Serial.println("(animation stopped)");
  }
}

void stepAnimation() {
  if (anim == 1) {
    leds = 1 << krPos;
    krPos += krDir;
    if (krPos == 3 || krPos == 0) krDir = -krDir;
  } else if (anim == 2) {
    leds = (leds + 1) & 0x0F;
  }
  show();
}

void handle(char c) {
  switch (c) {
    case '1': case '2': case '3': case '4':
      stopAnim();
      leds ^= (1 << (c - '1'));
      show(); printPattern();
      break;
    case 'a': stopAnim(); leds = 0x0F; show(); printPattern(); break;
    case 'o': stopAnim(); leds = 0x00; show(); printPattern(); break;
    case 'i': stopAnim(); leds = ~leds & 0x0F; show(); printPattern(); break;
    case '<':
      stopAnim();
      leds = ((leds << 1) | (leds >> 3)) & 0x0F;
      show(); printPattern();
      break;
    case '>':
      stopAnim();
      leds = ((leds >> 1) | (leds << 3)) & 0x0F;
      show(); printPattern();
      break;
    case 'k':
      anim = (anim == 1) ? 0 : 1;
      krPos = 0; krDir = 1;
      Serial.println(anim ? "Knight Rider ON" : "Knight Rider OFF");
      break;
    case 'b':
      anim = (anim == 2) ? 0 : 2;
      if (anim) leds = 0;
      Serial.println(anim ? "Binary counter ON" : "Binary counter OFF");
      break;
    case '+':
      speedMs = max(50, speedMs / 2);
      Serial.print("Speed: "); Serial.print(speedMs); Serial.println(" ms");
      break;
    case '-':
      speedMs = min(1000, speedMs * 2);
      Serial.print("Speed: "); Serial.print(speedMs); Serial.println(" ms");
      break;
    case 's':
      printPattern();
      Serial.print("Animation: ");
      Serial.println(anim == 0 ? "none" : (anim == 1 ? "knight rider" : "counter"));
      Serial.print("Speed: "); Serial.print(speedMs); Serial.println(" ms");
      break;
    case 'h': case '?': help(); break;
    case '\n': case '\r': case ' ': break;
    default:
      Serial.print("Unknown command '");
      Serial.print(c);
      Serial.println("' - press h for help");
  }
}

void setup() {
  for (int i = 0; i < 4; i++) pinMode(LED_PINS[i], OUTPUT);
  Serial.begin(9600);
  help();
  show();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  if (anim != 0 && millis() - lastStep >= (unsigned long)speedMs) {
    lastStep = millis();
    stepAnimation();
  }
}
```

## Assessment Questions

1. Send `1`, then `3`. Which LEDs are on? What happens when you send `1` again?
2. Send `a`, then `i`. What do the LEDs do after each command?
3. Send `k`, then `+` a few times. What changes? What happens to the animation when you send `2`?
4. What happens when you send several characters in one line, for example `1234`?
5. What will happen to the LED on D6 if its resistor is removed? Will the other LEDs still follow the commands?
