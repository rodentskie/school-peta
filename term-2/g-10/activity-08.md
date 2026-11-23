# Activity 08: Smart Night-Light with Moving Average and Hysteresis

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-08`

## Problem Statement

A simple night-light turns on when it is dark. But a cheap one **flickers** when the light level is right at the threshold, and it reacts to every passing shadow. Build a smart version:

1. Read the light sensor (LDR voltage divider) on **A0** and filter it with a **16-sample moving average** (ring buffer). Take one sample every 20 ms.
2. Use **hysteresis**: switch the lamp ON when the average drops below **300**, and OFF only when it rises above **450**.
3. The lamp (D9) must **fade** in and out smoothly, changing by 1 PWM step every 8 ms.
4. When ON, the lamp brightness depends on how dark it is: darker means brighter, with a minimum of 80/255.
5. The status LED (D8) shows the logical state (ON/OFF), without fading.
6. Print `light`, `average`, `state` and `brightness` every 500 ms.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Photoresistor (LDR) + 1 kΩ resistor (voltage divider) |
| 2 | LEDs (lamp, status) + 2 × 1 kΩ resistors |

## Solution

**Moving average with a ring buffer:** keep the last 16 readings in an array and a running `sum`. For each new sample, subtract the oldest value, add the new one, and overwrite the oldest slot. Calculating the average is then simply `sum / 16`, with no loop needed.

**Hysteresis (Schmitt-trigger logic):**

```
         reading falls below 300          reading rises above 450
 OFF  ─────────────────────────────► ON ──────────────────────────► OFF
```

Between 300 and 450 the state **does not change**, so small changes cannot make it flicker.

**Fading:** `current` moves towards `target` by 1 step every 8 ms, so a full fade (0 → 255) takes about 2 s.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| LDR | Terminal 1 | 5V rail |
| LDR | Terminal 2 | **A0** and 1 kΩ → GND (divider) |
| Lamp LED | Anode | **D9** (PWM) |
| Status LED | Anode | **D8** |
| LED cathodes | — | 1 kΩ → GND rail |

```
5V ──[LDR]──┬──► A0          D9 ─► LAMP   ─[1k]─ GND
            └─[1kΩ]── GND     D8 ─► STATUS ─[1k]─ GND
```

## Code

```cpp
// Activity 08 - Smart Night-Light
const int LDR_PIN = A0;
const int LAMP_PIN = 9;
const int STATUS_PIN = 8;

const int N = 16;
const int ON_BELOW = 300;
const int OFF_ABOVE = 450;

int samples[N];
int idx = 0;
long sum = 0;

bool lampOn = false;
int current = 0;   // PWM value now
int target = 0;    // PWM value we fade towards

unsigned long lastSample = 0, lastFade = 0, lastPrint = 0;

void setup() {
  pinMode(LAMP_PIN, OUTPUT);
  pinMode(STATUS_PIN, OUTPUT);
  Serial.begin(9600);
  int first = analogRead(LDR_PIN);
  for (int i = 0; i < N; i++) samples[i] = first;   // pre-fill buffer
  sum = (long)first * N;
  Serial.println("Smart night-light: ON < 300, OFF > 450");
}

void loop() {
  unsigned long now = millis();

  // 1) sample every 20 ms into ring buffer
  if (now - lastSample >= 20) {
    lastSample = now;
    int r = analogRead(LDR_PIN);
    sum -= samples[idx];
    samples[idx] = r;
    sum += r;
    idx = (idx + 1) % N;
  }
  int avg = sum / N;

  // 2) hysteresis decision
  if (!lampOn && avg < ON_BELOW) {
    lampOn = true;
    Serial.println(">> DARK: lamp ON");
  } else if (lampOn && avg > OFF_ABOVE) {
    lampOn = false;
    Serial.println(">> BRIGHT: lamp OFF");
  }
  digitalWrite(STATUS_PIN, lampOn ? HIGH : LOW);

  // 3) brightness target: darker = brighter
  if (lampOn) target = constrain(map(avg, 0, ON_BELOW, 255, 80), 80, 255);
  else target = 0;

  // 4) smooth fade
  if (now - lastFade >= 8) {
    lastFade = now;
    if (current < target) current++;
    else if (current > target) current--;
    analogWrite(LAMP_PIN, current);
  }

  // 5) log
  if (now - lastPrint >= 500) {
    lastPrint = now;
    Serial.print("light=");
    Serial.print(analogRead(LDR_PIN));
    Serial.print(" avg=");
    Serial.print(avg);
    Serial.print(" state=");
    Serial.print(lampOn ? "ON" : "OFF");
    Serial.print(" pwm=");
    Serial.println(current);
  }
}
```

## Assessment Questions

1. What happens to the lamp LED when you make the light sensor dark in the simulation?
2. Does the lamp turn on suddenly or slowly? What about turning off?
3. Move the light slider just a little, back and forth, near the point where the lamp switches. Does the lamp flicker? Describe what you see.
4. If the 1 kΩ resistor connected to the light sensor is removed, how do you think the lamp will behave?
5. If the light sensor and its resistor swap places (sensor to GND, resistor to 5V), what will the lamp do when it gets dark?
