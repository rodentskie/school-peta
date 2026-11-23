# Activity 29: Environmental Data Logger with Statistics and Anomaly Detection

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-29`

## Problem Statement

Scientists use data loggers to record measurements over time. Build a logger for **temperature** (TMP36) and **light** (LDR).

1. Take a sample of both sensors **every second** and print it as a **CSV** line: `time_s,temp_c,light`.
2. Keep the last **30 samples** in ring buffers.
3. **Short press** of the button: print a statistics report for both channels: **minimum, maximum, mean and standard deviation** of the stored samples.
4. **Long press** (≥ 1 s): clear the buffers and restart the time counter.
5. **Anomaly detection:** once at least 10 samples are stored, if a new temperature differs from the mean by more than **2 standard deviations** (and by at least 1 °C), print `ALERT` and light the LED for 2 s.
6. The status LED gives a short blink at every sample (heartbeat).

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Temperature Sensor [TMP36] |
| 1 | Photoresistor (LDR) + 1 kΩ resistor |
| 1 | Pushbutton |
| 1 | LED + 1 kΩ resistor |

## Solution

**Statistics formulas** (n samples x₁ … xₙ):

* mean μ = (Σ xᵢ) / n
* standard deviation σ = √( Σ (xᵢ − μ)² / n )

**Ring buffer:** `temp[30]`, `light[30]`, `head`, `count`. `count` grows up to 30. After that, the oldest sample is overwritten.

**Anomaly (z-score):** z = |x − μ| / σ. A value with z > 2 is unusual: in normally distributed data, only about 5 % of values are that far from the mean. The extra rule "at least 1 °C" prevents false alarms when σ is very small.

The function `stats()` calculates all four values in one pass for min/max/sum, then a second pass for the variance. It is written once and used for both channels.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| TMP36 | Vs / Vout / GND | 5V / **A0** / GND |
| LDR | T1 / T2 | 5V / **A1** (+ 1 kΩ to GND) |
| Pushbutton | T1 / T2 | GND / **D2** |
| LED | Anode | **D8** (cathode → 1 kΩ → GND) |

```
TMP36: Vs─5V  Vout─A0  GND─GND
5V ─[LDR]─┬─► A1
          └─[1k]─ GND
D2 ─► BUTTON ── GND        D8 ─► LED ─[1k]─ GND
```

## Code

```cpp
// Activity 29 - Environmental Data Logger with Statistics
const int TMP_PIN = A0;
const int LDR_PIN = A1;
const int BTN_PIN = 2;
const int LED_PIN = 8;
const int N = 30;

float temp[N];
float light[N];
int head = 0;
int count = 0;
unsigned long t0 = 0;
unsigned long lastSample = 0;
unsigned long alertUntil = 0;

int btnStable = HIGH, btnRaw = HIGH;
unsigned long btnChanged = 0, btnPressedAt = 0;
bool longDone = false;

float sMin, sMax, sMean, sStd;

void stats(float *data) {
  sMin = 1e9;
  sMax = -1e9;
  float sum = 0;
  for (int i = 0; i < count; i++) {
    if (data[i] < sMin) sMin = data[i];
    if (data[i] > sMax) sMax = data[i];
    sum += data[i];
  }
  sMean = sum / count;
  float sq = 0;
  for (int i = 0; i < count; i++) sq += (data[i] - sMean) * (data[i] - sMean);
  sStd = sqrt(sq / count);
}

void printStats(const char *name, float *data) {
  stats(data);
  Serial.print(name);
  Serial.print(": min=");
  Serial.print(sMin, 2);
  Serial.print(" max=");
  Serial.print(sMax, 2);
  Serial.print(" mean=");
  Serial.print(sMean, 2);
  Serial.print(" std=");
  Serial.println(sStd, 3);
}

void report() {
  Serial.println("----- STATISTICS -----");
  if (count == 0) {
    Serial.println("no data");
    return;
  }
  Serial.print("samples: ");
  Serial.println(count);
  printStats("temp_c", temp);
  printStats("light ", light);
  Serial.println("----------------------");
}

void clearAll() {
  head = 0;
  count = 0;
  t0 = millis();
  Serial.println("# buffers cleared");
  Serial.println("time_s,temp_c,light");
}

float readTemp() {
  float mv = analogRead(TMP_PIN) * 5000.0 / 1024.0;
  return (mv - 500.0) / 10.0;
}

void handleButton() {
  int r = digitalRead(BTN_PIN);
  unsigned long now = millis();
  if (r != btnRaw) { btnRaw = r; btnChanged = now; }
  if (now - btnChanged > 20 && r != btnStable) {
    btnStable = r;
    if (r == LOW) { btnPressedAt = now; longDone = false; }
    else if (!longDone) report();
  }
  if (btnStable == LOW && !longDone && now - btnPressedAt >= 1000) {
    longDone = true;
    clearAll();
  }
}

void setup() {
  pinMode(BTN_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("# Data logger: short press = statistics, long press = clear");
  clearAll();
}

void loop() {
  unsigned long now = millis();
  handleButton();

  if (now - lastSample >= 1000) {
    lastSample = now;
    float t = readTemp();
    float l = analogRead(LDR_PIN);

    // anomaly check BEFORE adding the new value
    if (count >= 10) {
      stats(temp);
      float dev = fabs(t - sMean);
      if (dev > 2 * sStd && dev >= 1.0) {
        Serial.print("# ALERT: temp ");
        Serial.print(t, 2);
        Serial.print(" is ");
        Serial.print(sStd > 0 ? dev / sStd : 99, 1);
        Serial.println(" std from mean");
        alertUntil = now + 2000;
      }
    }

    temp[head] = t;
    light[head] = l;
    head = (head + 1) % N;
    if (count < N) count++;

    Serial.print((now - t0) / 1000);
    Serial.print(',');
    Serial.print(t, 2);
    Serial.print(',');
    Serial.println((int)l);
  }

  bool heartbeat = (now - lastSample) < 60;
  bool alert = (long)(alertUntil - now) > 0;
  digitalWrite(LED_PIN, (alert || heartbeat) ? HIGH : LOW);
}
```

## Assessment Questions

1. Calculate the mean and standard deviation of the temperatures 22, 23, 23, 24, 28 (°C). Is 28 an anomaly using the "2 standard deviations" rule?
2. Why does the program check for an anomaly **before** adding the new value to the buffer?
3. What does CSV stand for? Explain how you could copy the Serial Monitor output into a spreadsheet and draw a graph.
4. The statistics are calculated on the **stored** samples, which may not be in time order because of the ring buffer. Does the order matter for min, max, mean and standard deviation? Explain.
5. Add a third statistic, the **trend** (°C per minute), using the first and last samples in time order. Explain how to find the oldest sample in the ring buffer when it is full.
