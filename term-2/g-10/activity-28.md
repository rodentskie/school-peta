# Activity 28: Mini Piano with Recording and Playback

**Grade 10 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-10-activity-28`

## Problem Statement

Build a 3-key piano that can **record** a melody and **play it back** with the original rhythm.

1. Three note buttons play **C5, E5 and G5** while they are held (the tone stops on release).
2. The **FUNCTION** button:
   * **Short press:** start or stop **recording**. The REC LED is on while recording.
   * **Long press (≥ 1 s):** **play back** the recording with the original timing (start times and note lengths).
3. Store up to **32 notes**. Each note records which key, when it started (ms after recording began), and how long it lasted.
4. If the memory is full, stop recording automatically and print a message.
5. During playback the REC LED blinks. Pressing any note button stops the playback.
6. Print a table of the recorded notes when recording stops.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 4 | Pushbuttons (3 notes + FUNCTION) |
| 1 | Piezo buzzer |
| 1 | LED (REC) + 1 kΩ resistor |

## Solution

* **Data structure:** an array of `struct Note { byte key; unsigned int start; unsigned int length; }`. Using `unsigned int` (16 bit) limits a recording to 65 s but halves the memory used.
* **Recording:** when a note key is pressed: `start = now − recStart`. When it is released: `length = now − pressTime`, then save the note.
* **Monophonic:** only one note at a time. The first key pressed wins until it is released.
* **Playback** is a non-blocking state machine: `playIdx`, `playStart`. At each loop:
  * if the current note's `start` time is reached → `tone()` on;
  * if `start + length` is reached → `noTone()`, next note.
* The short/long press of FUNCTION uses the same pattern as earlier activities.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Key C | T1 / T2 | GND / **D2** |
| Key E | T1 / T2 | GND / **D3** |
| Key G | T1 / T2 | GND / **D4** |
| FUNCTION button | T1 / T2 | GND / **D5** |
| Piezo | + / − | **D9** / GND |
| REC LED | Anode | **D8** (cathode → 1 kΩ → GND) |

```
D2 ─► KEY C ── GND     D5 ─► FUNCTION ── GND
D3 ─► KEY E ── GND     D9 ─► PIEZO ───── GND
D4 ─► KEY G ── GND     D8 ─► REC LED ─[1k]─ GND
```

## Code

```cpp
// Activity 28 - Mini Piano with Record & Playback
const int KEY_PINS[3] = {2, 3, 4};
const int FREQ[3] = {523, 659, 784};          // C5 E5 G5
const char NAMES[3] = {'C', 'E', 'G'};
const int FUNC_PIN = 5;
const int LED_PIN = 8;
const int BUZZER_PIN = 9;
const int MAX_NOTES = 32;

struct Note {
  byte key;
  unsigned int start;
  unsigned int length;
};
Note song[MAX_NOTES];
int noteCount = 0;

bool recording = false;
bool playing = false;
unsigned long recStart = 0;
int activeKey = -1;                 // key currently sounding (live)
unsigned long keyDownAt = 0;

int playIdx = 0;
bool playSounding = false;
unsigned long playStart = 0;

int fStable = HIGH, fRaw = HIGH;
unsigned long fChanged = 0, fPressedAt = 0;
bool fLongDone = false;

void printSong() {
  Serial.print("Recorded ");
  Serial.print(noteCount);
  Serial.println(" notes:");
  Serial.println("  #  key  start(ms)  length(ms)");
  for (int i = 0; i < noteCount; i++) {
    Serial.print("  ");
    Serial.print(i + 1);
    Serial.print("   ");
    Serial.print(NAMES[song[i].key]);
    Serial.print("    ");
    Serial.print(song[i].start);
    Serial.print("       ");
    Serial.println(song[i].length);
  }
}

void stopRecording() {
  recording = false;
  digitalWrite(LED_PIN, LOW);
  Serial.println("REC stopped");
  printSong();
}

void startPlayback() {
  if (noteCount == 0) {
    Serial.println("Nothing recorded");
    return;
  }
  if (recording) stopRecording();
  playing = true;
  playIdx = 0;
  playSounding = false;
  playStart = millis();
  Serial.println("PLAYBACK");
}

void stopPlayback() {
  playing = false;
  noTone(BUZZER_PIN);
  digitalWrite(LED_PIN, LOW);
}

void handleFunction() {
  int r = digitalRead(FUNC_PIN);
  unsigned long now = millis();
  if (r != fRaw) { fRaw = r; fChanged = now; }
  if (now - fChanged > 20 && r != fStable) {
    fStable = r;
    if (r == LOW) { fPressedAt = now; fLongDone = false; }
    else if (!fLongDone) {                         // short press
      if (playing) stopPlayback();
      if (recording) stopRecording();
      else {
        recording = true;
        noteCount = 0;
        recStart = now;
        digitalWrite(LED_PIN, HIGH);
        Serial.println("REC started");
      }
    }
  }
  if (fStable == LOW && !fLongDone && now - fPressedAt >= 1000) {
    fLongDone = true;
    startPlayback();
  }
}

void handleKeys() {
  unsigned long now = millis();
  if (activeKey < 0) {
    for (int k = 0; k < 3; k++) {
      if (digitalRead(KEY_PINS[k]) == LOW) {
        if (playing) stopPlayback();
        activeKey = k;
        keyDownAt = now;
        tone(BUZZER_PIN, FREQ[k]);
        break;
      }
    }
  } else if (digitalRead(KEY_PINS[activeKey]) == HIGH && now - keyDownAt > 30) {
    noTone(BUZZER_PIN);
    if (recording) {
      song[noteCount].key = activeKey;
      song[noteCount].start = keyDownAt - recStart;
      song[noteCount].length = now - keyDownAt;
      noteCount++;
      if (noteCount >= MAX_NOTES) {
        Serial.println("Memory full!");
        stopRecording();
      }
    }
    activeKey = -1;
  }
}

void handlePlayback() {
  if (!playing) return;
  unsigned long t = millis() - playStart;
  digitalWrite(LED_PIN, (millis() / 150) % 2);
  if (playIdx >= noteCount) {
    stopPlayback();
    Serial.println("Playback finished");
    return;
  }
  Note &n = song[playIdx];
  if (!playSounding && t >= n.start) {
    tone(BUZZER_PIN, FREQ[n.key]);
    playSounding = true;
  } else if (playSounding && t >= (unsigned long)n.start + n.length) {
    noTone(BUZZER_PIN);
    playSounding = false;
    playIdx++;
  }
}

void setup() {
  for (int k = 0; k < 3; k++) pinMode(KEY_PINS[k], INPUT_PULLUP);
  pinMode(FUNC_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(9600);
  // demo song so playback works immediately: C E G E C
  const byte demoKeys[5] = {0, 1, 2, 1, 0};
  for (int i = 0; i < 5; i++) {
    song[i].key = demoKeys[i];
    song[i].start = i * 400;
    song[i].length = 300;
  }
  noteCount = 5;
  Serial.println("Piano ready. FUNCTION: short = REC on/off, long = PLAY");
  startPlayback();
}

void loop() {
  handleFunction();
  handleKeys();
  handlePlayback();
}
```

## Assessment Questions

1. What do you hear right after the simulation starts?
2. What happens when you press and hold a note button? What happens if you press two note buttons at the same time?
3. How do you record a melody? What does the LED do while recording?
4. What happens when you hold the FUNCTION button for 1 second?
5. If the wire from the E button to D3 is disconnected, what happens when you press it? Can the other notes still be recorded?
