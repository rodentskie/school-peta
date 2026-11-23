# Activity 09: Serial Terminal on a 16×2 LCD (Typewriter with Scrolling)

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-09`
**Control:** Serial Monitor (type text and click **Send**)

## Problem Statement

Turn the LCD into a small **terminal screen**. Every printable character sent from the Serial Monitor appears on the LCD, like a typewriter.

| Character | Action |
|---|---|
| printable (`A`, `7`, `!` …) | write at the cursor, then move the cursor right |
| `\|` | new line (go to the start of the next line) |
| `<` | backspace (delete the previous character) |
| `#` | clear the screen |
| `~` | cursor visible ON/OFF (blinking block) |
| `^` | backlight ON/OFF |
| `@` | print the screen contents to the Serial Monitor |

1. When the text reaches the end of line 2 (or a new line is requested on line 2), the screen **scrolls up**: line 2 moves to line 1, and line 2 becomes empty.
2. Backspace at the start of line 2 goes back to the end of line 1.
3. Keep a copy of the screen in a buffer `char screen[2][17]`. The LCD is always redrawn **from the buffer**, so the buffer and the display never disagree.
4. The Enter key (newline from the Serial Monitor) is ignored. Use `|` for a new line.
5. Show a welcome message at start-up.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | LCD 16 × 2 (I2C) |

## Solution

* **Screen buffer:** `screen[row][col]` holds characters, with spaces for empty cells. `row` and `col` are the cursor position.
* `putChar(c)`: if `col == 16`, call `newLine()` first, then `screen[row][col++] = c`.
* `newLine()`: if `row == 0`, then `row = 1`. Otherwise **scroll**: copy row 1 into row 0, fill row 1 with spaces, and stay on row 1. Set `col = 0`.
* `backspace()`: if `col > 0`, then `col−−`. Otherwise, if `row == 1`, go to row 0, col 15. Then write a space at the cursor.
* `redraw()` prints both rows from the buffer and places the LCD's own cursor at (col, row).
* Special characters are handled in a `switch`. Any other character from 32 to 126 (printable ASCII) is written.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LCD (I2C) | GND | GND rail |
| LCD (I2C) | VCC | 5V rail |
| LCD (I2C) | SDA | **A4** |
| LCD (I2C) | SCL | **A5** |

```
LCD GND ─ GND rail     LCD VCC ─ 5V rail
LCD SDA ─► A4          LCD SCL ─► A5
```

## Code

```cpp
// G11 Activity 09 - Serial Terminal on a 16x2 LCD
#include <Adafruit_LiquidCrystal.h>

Adafruit_LiquidCrystal lcd(0);

char screen[2][17];
int row = 0, col = 0;
bool cursorOn = true;
bool backlight = true;

void redraw() {
  for (int r = 0; r < 2; r++) {
    lcd.setCursor(0, r);
    lcd.print(screen[r]);
  }
  lcd.setCursor(col < 16 ? col : 15, row);
}

void clearScreen() {
  for (int r = 0; r < 2; r++) {
    for (int c = 0; c < 16; c++) screen[r][c] = ' ';
    screen[r][16] = '\0';
  }
  row = 0;
  col = 0;
}

void newLine() {
  if (row == 0) {
    row = 1;
  } else {
    for (int c = 0; c < 16; c++) {     // scroll up
      screen[0][c] = screen[1][c];
      screen[1][c] = ' ';
    }
  }
  col = 0;
}

void putChar(char ch) {
  if (col >= 16) newLine();
  screen[row][col] = ch;
  col++;
}

void backspace() {
  if (col > 0) {
    col--;
  } else if (row == 1) {
    row = 0;
    col = 15;
  } else {
    return;                            // already at top-left
  }
  screen[row][col] = ' ';
}

void dump() {
  Serial.println("+----------------+");
  for (int r = 0; r < 2; r++) {
    Serial.print('|');
    Serial.print(screen[r]);
    Serial.println('|');
  }
  Serial.println("+----------------+");
  Serial.print("cursor row=");
  Serial.print(row);
  Serial.print(" col=");
  Serial.println(col);
}

void handle(char c) {
  switch (c) {
    case '\n': case '\r': return;
    case '|': newLine(); break;
    case '<': backspace(); break;
    case '#': clearScreen(); break;
    case '~':
      cursorOn = !cursorOn;
      if (cursorOn) lcd.blink(); else lcd.noBlink();
      break;
    case '^':
      backlight = !backlight;
      lcd.setBacklight(backlight ? 1 : 0);
      break;
    case '@': dump(); return;
    default:
      if (c >= 32 && c <= 126) putChar(c);
      else return;
  }
  redraw();
}

void setup() {
  Serial.begin(9600);
  lcd.begin(16, 2);
  lcd.setBacklight(1);
  lcd.blink();
  clearScreen();
  const char *hello = "Hello G11!|Type in Serial...";
  for (int i = 0; hello[i]; i++) handle(hello[i]);
  Serial.println("LCD terminal ready.");
  Serial.println(" | newline  < backspace  # clear  ~ cursor  ^ backlight  @ dump");
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
}
```

## Assessment Questions

1. What does the LCD show right after the simulation starts?
2. Type a long sentence of more than 32 characters and send it. What happens when the text reaches the end of line 2?
3. Send `<` a few times. What happens to the text and the cursor? Send `#`. What happens?
4. Send `^` and then `~`. What changes on the LCD?
5. If the LCD's SDA and SCL wires are swapped, what do you think the LCD will show? Will `@` still print the screen in the Serial Monitor?
