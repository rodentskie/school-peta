# Activity 16: Serial Command Interpreter with Background Tasks

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-16`

## Problem Statement

Real devices are often controlled by **text commands** sent over a serial port. Write a command interpreter that controls 2 LEDs and a buzzer.

| Command | Action |
|---|---|
| `HELP` | list all commands |
| `ON n` / `OFF n` | LED *n* (1 or 2) fully on / off |
| `TOGGLE n` | invert LED *n* |
| `PWM n v` | set LED *n* brightness to *v* (0–255) |
| `BLINK n times period` | blink LED *n* *times* times, *period* ms per blink, **in the background** |
| `BEEP freq ms` | play a tone |
| `STATUS` | show the state of both LEDs |

1. Commands are **case-insensitive** and end with Enter. Extra spaces are allowed.
2. Check the arguments. Print a clear error for an unknown command, a missing argument, or a value out of range (e.g. `ERROR: LED must be 1 or 2`).
3. `BLINK` must **not block**. Both LEDs can blink at the same time with different speeds while new commands are still accepted.
4. On start-up, run a short demo script: `HELP`, `BLINK 1 5 400`, `PWM 2 60`.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 2 | LEDs + 2 × 1 kΩ resistors |
| 1 | Piezo buzzer |

## Solution

1. **Reading a line:** characters are collected into `char line[40]` until `\n` or `\r`. Then `processLine()` is called. Lines that are too long are rejected.
2. **Tokenising:** the line is converted to uppercase, then `strtok(line, " ")` splits it into words. `atoi()` converts the number arguments.
3. **Dispatch:** `strcmp(cmd, "ON") == 0` and so on select the action.
4. **Background blinking:** each LED has a task structure: `togglesLeft`, `halfPeriod`, `nextAt` and `level`. `updateTasks()` runs every loop and toggles a LED when its time comes. This is **cooperative multitasking**.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LED 1 | Anode | **D9** (PWM) |
| LED 2 | Anode | **D10** (PWM) |
| LED cathodes | — | 1 kΩ → GND rail |
| Piezo | + / − | **D8** / GND |

```
D9  ─► LED1 ─[1k]─ GND
D10 ─► LED2 ─[1k]─ GND
D8  ─► PIEZO ───── GND
```

## Code

```cpp
// Activity 16 - Serial Command Interpreter
#include <string.h>

const int LED_PINS[2] = {9, 10};
const int BUZZER_PIN = 8;

struct LedTask {
  int togglesLeft;
  unsigned long halfPeriod;
  unsigned long nextAt;
  int level;              // current brightness 0..255
};
LedTask tasks[2] = {{0, 0, 0, 0}, {0, 0, 0, 0}};

char line[40];
int lineLen = 0;

void setLed(int i, int v) {
  tasks[i].level = v;
  analogWrite(LED_PINS[i], v);
}

void printHelp() {
  Serial.println("Commands:");
  Serial.println("  ON n | OFF n | TOGGLE n      (n = 1 or 2)");
  Serial.println("  PWM n value                  (0..255)");
  Serial.println("  BLINK n times period_ms");
  Serial.println("  BEEP freq ms");
  Serial.println("  STATUS | HELP");
}

// parse LED number; returns index 0..1 or -1 (prints error)
int ledArg(char *tok) {
  if (tok == NULL) { Serial.println("ERROR: missing LED number"); return -1; }
  int n = atoi(tok);
  if (n < 1 || n > 2) { Serial.println("ERROR: LED must be 1 or 2"); return -1; }
  return n - 1;
}

void processLine(char *s) {
  for (char *p = s; *p; p++) *p = toupper(*p);
  Serial.print("> ");
  Serial.println(s);

  char *cmd = strtok(s, " ");
  if (cmd == NULL) return;
  char *a1 = strtok(NULL, " ");
  char *a2 = strtok(NULL, " ");
  char *a3 = strtok(NULL, " ");

  if (strcmp(cmd, "HELP") == 0) {
    printHelp();
  } else if (strcmp(cmd, "ON") == 0 || strcmp(cmd, "OFF") == 0 || strcmp(cmd, "TOGGLE") == 0) {
    int i = ledArg(a1);
    if (i < 0) return;
    tasks[i].togglesLeft = 0;                       // cancel blinking
    int v;
    if (strcmp(cmd, "ON") == 0) v = 255;
    else if (strcmp(cmd, "OFF") == 0) v = 0;
    else v = tasks[i].level > 0 ? 0 : 255;
    setLed(i, v);
    Serial.println("OK");
  } else if (strcmp(cmd, "PWM") == 0) {
    int i = ledArg(a1);
    if (i < 0) return;
    if (a2 == NULL) { Serial.println("ERROR: missing value"); return; }
    int v = atoi(a2);
    if (v < 0 || v > 255) { Serial.println("ERROR: value must be 0..255"); return; }
    tasks[i].togglesLeft = 0;
    setLed(i, v);
    Serial.println("OK");
  } else if (strcmp(cmd, "BLINK") == 0) {
    int i = ledArg(a1);
    if (i < 0) return;
    if (a2 == NULL || a3 == NULL) { Serial.println("ERROR: BLINK n times period"); return; }
    int times = atoi(a2);
    long period = atol(a3);
    if (times < 1 || times > 100) { Serial.println("ERROR: times must be 1..100"); return; }
    if (period < 50 || period > 5000) { Serial.println("ERROR: period must be 50..5000"); return; }
    tasks[i].togglesLeft = times * 2;
    tasks[i].halfPeriod = period / 2;
    tasks[i].nextAt = millis();
    setLed(i, 0);
    Serial.println("OK (running in background)");
  } else if (strcmp(cmd, "BEEP") == 0) {
    if (a1 == NULL || a2 == NULL) { Serial.println("ERROR: BEEP freq ms"); return; }
    int f = atoi(a1);
    int ms = atoi(a2);
    if (f < 31 || f > 5000 || ms < 1 || ms > 3000) { Serial.println("ERROR: freq 31..5000, ms 1..3000"); return; }
    tone(BUZZER_PIN, f, ms);
    Serial.println("OK");
  } else if (strcmp(cmd, "STATUS") == 0) {
    for (int i = 0; i < 2; i++) {
      Serial.print("LED");
      Serial.print(i + 1);
      Serial.print(": level=");
      Serial.print(tasks[i].level);
      Serial.print(tasks[i].togglesLeft > 0 ? " (blinking, " : " (steady");
      if (tasks[i].togglesLeft > 0) {
        Serial.print(tasks[i].togglesLeft / 2);
        Serial.print(" left");
      }
      Serial.println(")");
    }
  } else {
    Serial.print("ERROR: unknown command '");
    Serial.print(cmd);
    Serial.println("' - type HELP");
  }
}

void updateTasks() {
  unsigned long now = millis();
  for (int i = 0; i < 2; i++) {
    if (tasks[i].togglesLeft > 0 && (long)(now - tasks[i].nextAt) >= 0) {
      setLed(i, tasks[i].level > 0 ? 0 : 255);
      tasks[i].togglesLeft--;
      tasks[i].nextAt = now + tasks[i].halfPeriod;
    }
  }
}

void readSerial() {
  while (Serial.available() > 0) {
    char c = Serial.read();
    if (c == '\n' || c == '\r') {
      if (lineLen > 0) {
        line[lineLen] = '\0';
        processLine(line);
        lineLen = 0;
      }
    } else if (lineLen < (int)sizeof(line) - 1) {
      line[lineLen++] = c;
    } else {
      Serial.println("ERROR: line too long");
      lineLen = 0;
    }
  }
}

void runScript(const char *cmd) {
  char buf[40];
  strncpy(buf, cmd, sizeof(buf) - 1);
  buf[sizeof(buf) - 1] = '\0';
  processLine(buf);
}

void setup() {
  pinMode(LED_PINS[0], OUTPUT);
  pinMode(LED_PINS[1], OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Command interpreter ready. Demo script:");
  runScript("help");
  runScript("blink 1 5 400");
  runScript("pwm 2 60");
}

void loop() {
  readSerial();
  updateTasks();
}
```

## Assessment Questions

1. What do the two LEDs do right after the simulation starts?
2. Type `ON 1`, then `off 1`. Does it matter if you use capital or small letters?
3. Type `BLINK 2 10 300`, then quickly type `ON 1`. Do both LEDs work at the same time?
4. What happens when you type a wrong command, for example `JUMP` or `ON 5`?
5. If LED 2 is moved from D10 to D7 (a pin without `~`), what happens when you type `PWM 2 60`?
