# Activity 18: Keypad Calculator with Operator Precedence on an I2C LCD

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-18`

## Problem Statement

Build a 4-function calculator using a **4×4 keypad** and a **16×2 I2C LCD**.

| Key | Meaning |
|---|---|
| `0`–`9` | digits |
| `A` | + |
| `B` | − |
| `C` | × |
| `D` | ÷ (integer division) |
| `#` | = (calculate) |
| `*` | clear |

1. Line 1 of the LCD shows the expression as it is typed, e.g. `12+3*4`. If it is longer than 16 characters, show the **last** 16.
2. Pressing `#` calculates the result **with operator precedence**: `×` and `÷` are evaluated before `+` and `−`. So `12+3*4` = **24**, not 60.
3. Line 2 shows `= result`. Handle the errors **division by zero** (`Error: DIV/0`) and **too many terms** (maximum 8 numbers).
4. After a result, typing an operator continues the calculation from the result (`24` → `24+1`). Typing a digit starts a new calculation.
5. Use 32-bit `long` numbers. Each key press gives a short click from the piezo.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Keypad 4×4 |
| 1 | LCD 16 × 2 (I2C) |
| 1 | Piezo buzzer |

## Solution

**Storing the expression:** two arrays: `nums[8]` (numbers) and `ops[7]` (operators). `cur` is the number being typed.

**Two-pass evaluation (operator precedence):**

1. **Pass 1 (× and ÷):** go through the operators from left to right. When `ops[i]` is `*` or `/`, replace `nums[i]` with `nums[i] op nums[i+1]` and remove `nums[i+1]` and `ops[i]` from the arrays (shift left).
2. **Pass 2 (+ and −):** now only + and − are left. Combine all of them from left to right.

Example: `12 + 3 * 4` → pass 1: `12 + 12` → pass 2: `24`.

**LCD:** the Tinkercad I2C LCD uses the `Adafruit_LiquidCrystal` library at address 0 (0x20). To avoid flicker, the whole display is redrawn only when something changes.

## Pin Diagram

| Keypad pin | Arduino |
|---|---|
| Row 1–4 | **D9, D8, D7, D6** |
| Column 1–4 | **D5, D4, D3, D2** |

| Other | Pin | Connects to |
|---|---|---|
| LCD (I2C) | GND / VCC | GND rail / 5V rail |
| LCD (I2C) | SDA | **A4** |
| LCD (I2C) | SCL | **A5** |
| Piezo | + / − | **D10** / GND |

```
KEYPAD R1..R4 ─► D9 D8 D7 D6   C1..C4 ─► D5 D4 D3 D2
LCD: GND─GND  VCC─5V  SDA─A4  SCL─A5
D10 ─► PIEZO ── GND
```

## Code

```cpp
// Activity 18 - Keypad Calculator with Operator Precedence
#include <Keypad.h>
#include <Adafruit_LiquidCrystal.h>

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
Adafruit_LiquidCrystal lcd(0);

const int BUZZER_PIN = 10;
const int MAXN = 8;

long nums[MAXN];
char ops[MAXN - 1];
int count = 0;           // numbers stored in nums[]
long cur = 0;
bool haveDigit = false;
bool showingResult = false;
long lastResult = 0;
String expr = "";
String line2 = "";

void redraw() {
  lcd.clear();
  lcd.setCursor(0, 0);
  if (expr.length() > 16) lcd.print(expr.substring(expr.length() - 16));
  else lcd.print(expr);
  lcd.setCursor(0, 1);
  lcd.print(line2);
}

void clearAll() {
  count = 0;
  cur = 0;
  haveDigit = false;
  showingResult = false;
  expr = "";
  line2 = "";
}

char opSymbol(char k) {
  if (k == 'A') return '+';
  if (k == 'B') return '-';
  if (k == 'C') return '*';
  return '/';
}

// returns false on error (message placed in line2)
bool evaluate(long &result) {
  long n[MAXN];
  char o[MAXN - 1];
  int c = count;
  for (int i = 0; i < c; i++) n[i] = nums[i];
  for (int i = 0; i < c - 1; i++) o[i] = ops[i];

  // pass 1: * and /
  int i = 0;
  while (i < c - 1) {
    if (o[i] == '*' || o[i] == '/') {
      if (o[i] == '/') {
        if (n[i + 1] == 0) { line2 = "Error: DIV/0"; return false; }
        n[i] = n[i] / n[i + 1];
      } else {
        n[i] = n[i] * n[i + 1];
      }
      for (int j = i + 1; j < c - 1; j++) n[j] = n[j + 1];
      for (int j = i; j < c - 2; j++) o[j] = o[j + 1];
      c--;
    } else {
      i++;
    }
  }
  // pass 2: + and -
  long r = n[0];
  for (int k = 0; k < c - 1; k++) {
    if (o[k] == '+') r += n[k + 1];
    else r -= n[k + 1];
  }
  result = r;
  return true;
}

void onDigit(char k) {
  if (showingResult) clearAll();                 // new calculation
  if (cur > 99999999L) return;                   // avoid overflow while typing
  cur = cur * 10 + (k - '0');
  haveDigit = true;
  expr += k;
}

void onOperator(char k) {
  if (showingResult) {                           // continue from result
    clearAll();
    cur = lastResult;
    haveDigit = true;
    expr = String(lastResult);
  }
  if (!haveDigit) return;                        // ignore "++" etc.
  if (count >= MAXN - 1) { line2 = "Error: too long"; return; }
  nums[count] = cur;
  ops[count] = opSymbol(k);
  count++;
  cur = 0;
  haveDigit = false;
  expr += opSymbol(k);
  line2 = "";
}

void onEquals() {
  if (!haveDigit || showingResult) return;
  nums[count++] = cur;
  long r;
  if (evaluate(r)) {
    lastResult = r;
    line2 = "= " + String(r);
    Serial.print(expr);
    Serial.print(" = ");
    Serial.println(r);
  } else {
    Serial.print(expr);
    Serial.print(" -> ");
    Serial.println(line2);
    lastResult = 0;
  }
  showingResult = true;
}

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  lcd.begin(16, 2);
  lcd.setBacklight(1);
  lcd.print("Calculator");
  lcd.setCursor(0, 1);
  lcd.print("A+ B- Cx D/ #=");
  Serial.println("Calculator: A=+ B=- C=x D=/ #== *=clear");
}

void loop() {
  char k = keypad.getKey();
  if (k == NO_KEY) return;
  tone(BUZZER_PIN, 1800, 25);

  if (k >= '0' && k <= '9') onDigit(k);
  else if (k == 'A' || k == 'B' || k == 'C' || k == 'D') onOperator(k);
  else if (k == '#') onEquals();
  else if (k == '*') clearAll();
  redraw();
}
```

## Assessment Questions

1. Evaluate these by hand using the program's two-pass method, and show the arrays after pass 1: (a) `8+6/2`  (b) `20-3*4+1`  (c) `7/2*2`.
2. Why does `7/2*2` give 6 on this calculator? What would a calculator using `float` show?
3. What is the largest value a `long` can hold on the Arduino Uno? What happens if a result is bigger?
4. Explain why `evaluate()` works on **copies** (`n[]`, `o[]`) of the arrays instead of changing `nums[]` and `ops[]` directly.
5. Add a **backspace** function on the `*` key (short press deletes the last character, long press clears everything). Which variables must be updated when the last character is a digit, and which when it is an operator?
