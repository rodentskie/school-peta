# Activity 14: Serial Sound-Effects Generator

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-14`
**Control:** Serial Monitor (type a character and click **Send**)

## Problem Statement

Games and toys use sound effects made from changing frequencies. Make a sound-effects box played from the keyboard.

| Key | Effect |
|---|---|
| `1` | **Siren**: frequency rises and falls smoothly (600–1200 Hz) |
| `2` | **Laser**: fast falling sweep (2000 → 200 Hz in 200 ms), repeated |
| `3` | **Alarm**: two alternating tones (880 / 660 Hz) |
| `4` | **Heartbeat**: two short low beeps, then a pause |
| `5` | **Coin**: two quick high notes (once) |
| `6` | **Power-up**: rising staircase of 8 notes (once) |
| `x` | stop the sound |
| `+` / `-` | pitch up / down (×1.25 / ÷1.25, range 0.5×–2×) |
| `f` / `s` | effect speed faster / slower |
| `?` | help |

1. Effects 1–4 **loop** until stopped or replaced. Effects 5–6 play **once** and then stop by themselves.
2. All effects are generated **without `delay()`**. The frequency is calculated from the time since the effect started.
3. The LED **brightness follows the frequency** (PWM). Higher pitch means a brighter LED.
4. Print the effect name, pitch factor and speed whenever something changes.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Piezo buzzer |
| 1 | LED + 1 kΩ resistor |

## Solution

* Each effect is a function `freqAt(t)` that returns the frequency (Hz) at time `t` ms, or 0 for silence. For example:
  * siren: `900 + 300 × sin(2π t / 1500)`
  * laser: `2000 − (t % 300) × 9` for the first 200 ms of every 300 ms, else 0
  * staircase: note number `t / 80`, frequency `400 × 1.15ⁿ`
* `t` is scaled by the speed factor: `t = (now − start) × speed`.
* The final frequency is multiplied by the pitch factor and played with `tone()` only when it **changes**. Calling `tone()` too often causes clicks.
* **LED brightness:** `map(freq, 200, 2400, 10, 255)`.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Piezo | + | **D9** |
| Piezo | − | GND rail |
| LED | Anode | **D10** (PWM) (cathode → 1 kΩ → GND) |

```
D9  ─► PIEZO(+)  PIEZO(−) ── GND
D10 ─► LED ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 14 - Serial Sound-Effects Generator
const int BUZZER_PIN = 9;
const int LED_PIN = 10;
const char* NAMES[7] = {"silence", "Siren", "Laser", "Alarm", "Heartbeat", "Coin", "Power-up"};

int effect = 0;
unsigned long effectStart = 0;
float pitch = 1.0;
float speed = 1.0;
int lastFreq = -1;

// returns frequency in Hz (0 = silent, -1 = one-shot effect finished)
int freqAt(int e, unsigned long t) {
  if (e == 1) return 900 + (int)(300 * sin(2 * PI * (t % 1500) / 1500.0));
  if (e == 2) {
    unsigned long p = t % 300;
    return p < 200 ? 2000 - (int)(p * 9) : 0;
  }
  if (e == 3) return (t % 600) < 300 ? 880 : 660;
  if (e == 4) {
    unsigned long p = t % 1200;
    return (p < 80 || (p >= 200 && p < 280)) ? 150 : 0;
  }
  if (e == 5) {
    if (t < 80) return 988;
    if (t < 400) return 1319;
    return -1;
  }
  if (e == 6) {
    int n = t / 80;
    if (n >= 8) return -1;
    return (int)(400 * pow(1.15, n));
  }
  return 0;
}

void report() {
  Serial.print("Effect: ");
  Serial.print(NAMES[effect]);
  Serial.print(" | pitch x");
  Serial.print(pitch, 2);
  Serial.print(" | speed x");
  Serial.println(speed, 2);
}

void start(int e) {
  effect = e;
  effectStart = millis();
  lastFreq = -1;
  report();
}

void help() {
  Serial.println("--- SOUND FX ---");
  Serial.println(" 1 siren | 2 laser | 3 alarm | 4 heartbeat | 5 coin | 6 power-up");
  Serial.println(" x stop | + - pitch | f s speed | ? help");
}

void handle(char c) {
  if (c >= '1' && c <= '6') start(c - '0');
  else if (c == 'x') start(0);
  else if (c == '+') { pitch = min(2.0, pitch * 1.25); report(); }
  else if (c == '-') { pitch = max(0.5, pitch / 1.25); report(); }
  else if (c == 'f') { speed = min(4.0, speed * 1.5); report(); }
  else if (c == 's') { speed = max(0.25, speed / 1.5); report(); }
  else if (c == '?' || c == 'h') help();
  else if (c != '\n' && c != '\r' && c != ' ') {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
  }
}

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  help();
  start(1);                      // demo: siren
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());

  unsigned long t = (unsigned long)((millis() - effectStart) * speed);
  int f = freqAt(effect, t);
  if (f == -1) {                 // one-shot finished
    start(0);
    f = 0;
  }
  if (f > 0) f = (int)(f * pitch);

  if (f != lastFreq) {
    lastFreq = f;
    if (f > 0) tone(BUZZER_PIN, f);
    else noTone(BUZZER_PIN);
    analogWrite(LED_PIN, f > 0 ? constrain(map(f, 200, 2400, 10, 255), 10, 255) : 0);
  }
}
```

## Assessment Questions

1. What do you hear and see right after the simulation starts?
2. Send `2`, then `3`, then `4`. Describe how each sound is different. What does the LED do for each one?
3. Send `5` and `6`. Do they keep playing like the others? What happens when they end?
4. Send `+` three times, then `f` twice. What changes in the sound?
5. What will happen to the LED if its resistor is removed? If the buzzer is disconnected, can you still tell which effect is playing?
