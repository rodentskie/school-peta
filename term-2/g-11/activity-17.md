# Activity 17: Serial RPN Stack Calculator on LCD

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-17`
**Control:** Serial Monitor (type characters and click **Send**)

## Problem Statement

Scientific calculators made by HP use **Reverse Polish Notation (RPN)**. You type the numbers first, and then the operator. For example, `3 4 +` means 3 + 4. RPN needs no brackets: `(2 + 3) × 4` is typed `2 3 + 4 *`.

Build an RPN calculator that keeps a **stack of 8 numbers**.

| Character | Action |
|---|---|
| `0`–`9`, `.` | type a number |
| space or `,` | **push** the typed number onto the stack |
| `+ - * /` | pop two numbers, calculate, push the result |
| `^` | power (x^y) |
| `r` | square root of the top number |
| `n` | negate the top number |
| `x` | swap the two top numbers |
| `d` | drop (remove) the top number |
| `c` | clear the stack |

1. An operator typed right after a number first pushes that number automatically. So `3 4+` works.
2. **LCD line 1:** the second number on the stack (`y:`). **LCD line 2:** the top number (`x:`), or the number being typed with a `_` cursor.
3. **Errors** print a message and leave the stack unchanged: `Stack underflow` (not enough numbers), `Stack full`, `Divide by 0`, `Sqrt of negative`.
4. After each operation, print the whole stack in the Serial Monitor (top at the right).

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | LCD 16 × 2 (I2C) |
| 1 | Piezo buzzer |

## Solution

* **Stack:** `float stack[8]` and `sp` (the number of items). `push()` and `pop()` check for full and empty.
* **Typing:** characters are collected in `char entry[12]`. `atof()` converts the text to a float when it is pushed.
* **Binary operators** check `sp >= 2` first. Then pop `b` (top) and `a` (below it) and push `a op b`. The order matters for `−`, `/` and `^`.
* An error beep (low tone) sounds for errors, and a click for normal keys.
* Numbers are shown with 4 decimals, which fits 16 characters for normal values.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LCD (I2C) | GND / VCC | GND rail / 5V rail |
| LCD (I2C) | SDA / SCL | **A4** / **A5** |
| Piezo | + / − | **D8** / GND |

```
LCD: GND─GND VCC─5V SDA─A4 SCL─A5
D8 ─► PIEZO ── GND
```

## Code

```cpp
// G11 Activity 17 - Serial RPN Stack Calculator
#include <Adafruit_LiquidCrystal.h>

Adafruit_LiquidCrystal lcd(0);

const int BUZZER_PIN = 8;
const int MAXS = 8;

float stack[MAXS];
int sp = 0;
char entry[12];
int entryLen = 0;

void beepOk() { tone(BUZZER_PIN, 1800, 15); }
void beepErr() { tone(BUZZER_PIN, 200, 250); }

void error(const char *msg) {
  Serial.print("ERROR: ");
  Serial.println(msg);
  beepErr();
}

bool push(float v) {
  if (sp >= MAXS) { error("Stack full"); return false; }
  stack[sp++] = v;
  return true;
}

void printStack() {
  Serial.print("Stack: [ ");
  for (int i = 0; i < sp; i++) {
    Serial.print(stack[i], 4);
    Serial.print(' ');
  }
  Serial.println("]");
}

void showLine(int row, const char *label, bool has, float v) {
  lcd.setCursor(0, row);
  lcd.print(label);
  if (has) lcd.print(v, 4);
  lcd.print("              ");
}

void redraw() {
  if (sp >= 2) showLine(0, "y: ", true, stack[sp - 2]);
  else showLine(0, "y: ", false, 0);
  lcd.setCursor(0, 1);
  if (entryLen > 0) {
    lcd.print("> ");
    lcd.print(entry);
    lcd.print("_             ");
  } else {
    showLine(1, "x: ", sp >= 1, sp >= 1 ? stack[sp - 1] : 0);
  }
}

void commitEntry() {
  if (entryLen == 0) return;
  push(atof(entry));
  entryLen = 0;
  entry[0] = '\0';
}

void binaryOp(char op) {
  if (sp < 2) { error("Stack underflow"); return; }
  float b = stack[sp - 1], a = stack[sp - 2];
  float r;
  if (op == '+') r = a + b;
  else if (op == '-') r = a - b;
  else if (op == '*') r = a * b;
  else if (op == '/') {
    if (b == 0) { error("Divide by 0"); return; }
    r = a / b;
  } else r = pow(a, b);
  sp -= 2;
  push(r);
}

void handle(char c) {
  if (c == '\n' || c == '\r') return;
  if ((c >= '0' && c <= '9') || c == '.') {
    if (c == '.' && strchr(entry, '.')) return;   // only one decimal point
    if (entryLen < 10) { entry[entryLen++] = c; entry[entryLen] = '\0'; }
    beepOk();
    redraw();
    return;
  }
  if (c == ' ' || c == ',') { commitEntry(); beepOk(); }
  else if (c == '+' || c == '-' || c == '*' || c == '/' || c == '^') { commitEntry(); binaryOp(c); beepOk(); }
  else if (c == 'r') {
    commitEntry();
    if (sp < 1) error("Stack underflow");
    else if (stack[sp - 1] < 0) error("Sqrt of negative");
    else stack[sp - 1] = sqrt(stack[sp - 1]);
  }
  else if (c == 'n') { commitEntry(); if (sp < 1) error("Stack underflow"); else stack[sp - 1] = -stack[sp - 1]; }
  else if (c == 'x') {
    commitEntry();
    if (sp < 2) error("Stack underflow");
    else { float t = stack[sp - 1]; stack[sp - 1] = stack[sp - 2]; stack[sp - 2] = t; }
  }
  else if (c == 'd') { if (entryLen) { entryLen = 0; entry[0] = '\0'; } else if (sp > 0) sp--; }
  else if (c == 'c') { sp = 0; entryLen = 0; entry[0] = '\0'; Serial.println("Cleared"); }
  else { error("Unknown key"); return; }
  printStack();
  redraw();
}

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  lcd.begin(16, 2);
  lcd.setBacklight(1);
  entry[0] = '\0';
  Serial.println("RPN calculator. Example: 2 3+4*  -> 20");
  const char *demo = "2 3+4*";
  for (int i = 0; demo[i]; i++) handle(demo[i]);
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
}
```

## Assessment Questions

1. What does the LCD show right after the simulation starts? What calculation was done?
2. Send `c`, then `5 2-`. What result appears? Now send `c` and `2 5-`. Is the result the same? Why?
3. Send `c`, then `+`. What happens on the Serial Monitor and what do you hear?
4. Send `c`, then `9r` and then `n`. What does the LCD show after each command?
5. If the LCD's SDA wire is disconnected, does the calculator still work? Where can you still see the results?
