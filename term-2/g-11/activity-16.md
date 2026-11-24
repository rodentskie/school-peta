# Activity 16: Serial-Navigated LCD Menu System

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-16`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Many devices (printers, microwaves, routers) have a small screen with a **menu**. Build a 2-level menu on a 16×2 LCD, navigated with the keyboard like arrow keys.

| Key | Action |
|---|---|
| `w` | up |
| `s` | down |
| `d` | enter / select / increase value |
| `a` | back / decrease value |
| `q` | back to the main menu |

**Main menu** (line 1 shows `>` before the selected item, line 2 shows the next item):

| Item | Inside |
|---|---|
| LED Brightness | value 0–100 % in steps of 10 (`d` +10, `a` −10, `q` back). LED changes immediately. |
| LED Mode | choose `Steady` / `Blink` / `Breathe` with `w`/`s`, `d` confirms |
| Beep on Key | ON / OFF toggle with `d` |
| Backlight | ON / OFF toggle with `d` |
| About | shows `G11 Menu v1.0` and the uptime in seconds |

1. The selected item is marked with `>`. The menu **wraps** (down from the last item goes to the first).
2. Every key gives a short click if "Beep on Key" is ON.
3. LED modes work in the background: Blink 2 Hz, Breathe = smooth fade, both at the chosen brightness.
4. The menu is redrawn only when something changes.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | LCD 16 × 2 (I2C) |
| 1 | LED + 1 kΩ resistor |
| 1 | Piezo buzzer |

## Solution

* **Screen state:** `screen` = −1 for the main menu, or the index of the open item (0–4). `sel` is the selected main item and `modeSel` is the highlighted mode inside "LED Mode".
* `drawMain()` shows item `sel` with `>` on line 1 and item `(sel + 1) % 5` on line 2.
* `drawItem()` shows the title on line 1 and the value on line 2, e.g. `Value: 70%  [a-d+]`.
* Keys are handled by `handleMain(c)` or `handleItem(c)` depending on `screen`.
* **LED output** is calculated every loop from `ledMode` and `brightness`:
  * steady: `pwm = brightness × 255 / 100`
  * blink: `pwm` or 0, switching every 250 ms
  * breathe: `pwm × (1 − cos(2πt/2000)) / 2`

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LCD (I2C) | GND / VCC | GND rail / 5V rail |
| LCD (I2C) | SDA / SCL | **A4** / **A5** |
| LED | Anode | **D9** (PWM) (cathode → 1 kΩ → GND) |
| Piezo | + / − | **D8** / GND |

```
LCD: GND─GND VCC─5V SDA─A4 SCL─A5
D9 ─► LED ─[1k]─ GND      D8 ─► PIEZO ── GND
```

## Code

```cpp
// G11 Activity 16 - Serial-Navigated LCD Menu
#include <Adafruit_LiquidCrystal.h>

Adafruit_LiquidCrystal lcd(0);

const int LED_PIN = 9;
const int BUZZER_PIN = 8;
const int ITEMS = 5;
const char* MENU[ITEMS] = {"LED Brightness", "LED Mode", "Beep on Key", "Backlight", "About"};
const char* MODES[3] = {"Steady", "Blink", "Breathe"};

int screen = -1;          // -1 = main menu
int sel = 0;
int brightness = 50;      // %
int ledMode = 0;
int modeSel = 0;
bool beep = true;
bool backlight = true;

void line(int row, const char *text) {
  lcd.setCursor(0, row);
  int n = 0;
  while (text[n] && n < 16) { lcd.print(text[n]); n++; }
  while (n < 16) { lcd.print(' '); n++; }
}

void drawMain() {
  lcd.setCursor(0, 0);
  lcd.print('>');
  lcd.print(MENU[sel]);
  for (int i = strlen(MENU[sel]) + 1; i < 16; i++) lcd.print(' ');
  lcd.setCursor(0, 1);
  lcd.print(' ');
  int next = (sel + 1) % ITEMS;
  lcd.print(MENU[next]);
  for (int i = strlen(MENU[next]) + 1; i < 16; i++) lcd.print(' ');
}

void drawItem() {
  line(0, MENU[screen]);
  char buf[17];
  if (screen == 0) {
    sprintf(buf, "Value: %3d%% a-d+", brightness);
  } else if (screen == 1) {
    sprintf(buf, "%c%-8s w/s d=ok", modeSel == ledMode ? '*' : ' ', MODES[modeSel]);
  } else if (screen == 2) {
    sprintf(buf, "%s   d=toggle", beep ? "ON " : "OFF");
  } else if (screen == 3) {
    sprintf(buf, "%s   d=toggle", backlight ? "ON " : "OFF");
  } else {
    sprintf(buf, "v1.0 up %lus", millis() / 1000);
  }
  line(1, buf);
}

void redraw() {
  if (screen < 0) drawMain();
  else drawItem();
}

void handleMain(char c) {
  if (c == 'w') sel = (sel + ITEMS - 1) % ITEMS;
  else if (c == 's') sel = (sel + 1) % ITEMS;
  else if (c == 'd') {
    screen = sel;
    modeSel = ledMode;
  }
}

void handleItem(char c) {
  if (c == 'q') { screen = -1; return; }
  if (screen == 0) {
    if (c == 'd') brightness = min(100, brightness + 10);
    else if (c == 'a') brightness = max(0, brightness - 10);
  } else if (screen == 1) {
    if (c == 'w') modeSel = (modeSel + 2) % 3;
    else if (c == 's') modeSel = (modeSel + 1) % 3;
    else if (c == 'd') {
      ledMode = modeSel;
      Serial.print("LED mode: ");
      Serial.println(MODES[ledMode]);
    }
    else if (c == 'a') screen = -1;
  } else if (screen == 2 || screen == 3) {
    if (c == 'd') {
      if (screen == 2) beep = !beep;
      else { backlight = !backlight; lcd.setBacklight(backlight ? 1 : 0); }
    } else if (c == 'a') screen = -1;
  } else {
    if (c == 'a' || c == 'd') screen = -1;
  }
}

void handle(char c) {
  if (c != 'w' && c != 's' && c != 'a' && c != 'd' && c != 'q') {
    if (c != '\n' && c != '\r' && c != ' ') Serial.println("Use w s a d q");
    return;
  }
  if (beep) tone(BUZZER_PIN, 2000, 15);
  if (screen < 0) handleMain(c);
  else handleItem(c);
  redraw();
}

void updateLed() {
  int pwm = brightness * 255 / 100;
  unsigned long t = millis();
  if (ledMode == 1) pwm = ((t / 250) % 2) ? pwm : 0;
  else if (ledMode == 2) pwm = (int)(pwm * (1 - cos(2 * PI * (t % 2000) / 2000.0)) / 2);
  analogWrite(LED_PIN, pwm);
}

void setup() {
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  lcd.begin(16, 2);
  lcd.setBacklight(1);
  Serial.println("Menu: w up, s down, d enter/+, a back/-, q main menu");
  redraw();
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  updateLed();
  static unsigned long lastAbout = 0;
  if (screen == 4 && millis() - lastAbout >= 1000) {   // live uptime
    lastAbout = millis();
    drawItem();
  }
}
```

## Assessment Questions

1. What does the LCD show right after the simulation starts? Which item has the `>`?
2. Send `s` five times. What happens on the LCD after the last item?
3. Open "LED Brightness" with `d`. What happens to the LED when you send `d` and `a` several times? How do you go back?
4. Open "LED Mode", choose Breathe, and confirm. What does the LED do now, even after you leave the menu?
5. If the LED's resistor is removed, what will happen to the LED? If the LCD is disconnected, can a user still change the settings? Why would it be hard?
