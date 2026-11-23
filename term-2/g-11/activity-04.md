# Activity 04: Serial Keyboard Music Box with Note Queue and Recorder

**Grade 11 · Term 2 · Difficulty: 8/10**
**Tinkercad design:** `g-11-activity-04`
**Control:** Serial Monitor (type characters and click **Send**)

## Problem Statement

Turn the computer keyboard into a piano. Every character sent over serial is a note or a command.

| Key | Action |
|---|---|
| `a s d f g h j k` | notes C D E F G A B C' (one octave) |
| `.` | rest (silence for one note length) |
| `1` `2` `3` `4` | note length 125 / 250 / 500 / 1000 ms |
| `+` / `-` | octave up / down (range −1 … +2) |
| `r` | start / stop recording |
| `p` | play the recording |
| `x` | stop and clear the queue |
| `?` | help |

1. Notes are **queued**. Sending `asdfghjk` plays the scale in order, one note after another, **without blocking** the program.
2. The queue is a **ring buffer of 64 characters**. If it is full, print `Queue full`.
3. A short gap of 30 ms between notes makes repeated notes sound separate.
4. While recording, every note and rest that is played is also saved (up to 100). `p` copies the recording into the queue.
5. The LED lights while a note is sounding.

## Components

| Qty | Component |
|---|---|
| 1 | Arduino Uno R3 |
| 1 | Breadboard Small |
| 1 | Piezo buzzer |
| 1 | LED + 1 kΩ resistor |

## Solution

* **Frequencies:** the base octave (C4…C5) is stored in `BASE[8]`. Octave shift *k* multiplies the frequency by 2ᵏ, using `<<` for up and `>>` for down.
* **Ring buffer:** `queue[64]`, `qHead` (next to play), `qTail` (next free slot), and `qCount`.
  * push: `queue[qTail] = c; qTail = (qTail + 1) % 64; qCount++`
  * pop: `c = queue[qHead]; qHead = (qHead + 1) % 64; qCount--`
* **Player state machine:** if not busy and the queue is not empty, pop a note, start `tone()`, and set `noteEnd = now + length − 30`. When `noteEnd` passes, stop the tone and wait 30 ms (`gapEnd`) before the next note.
* **Commands vs notes:** commands (`1`–`4`, `+`, `-`, `r`, `p`, `x`, `?`) act **immediately**, but notes and rests go into the queue.

## Pin Diagram

| Component | Pin | Connects to |
|---|---|---|
| Piezo | + | **D9** |
| Piezo | − | GND rail |
| LED | Anode | **D8** (cathode → 1 kΩ → GND) |

```
D9 ─► PIEZO(+)  PIEZO(−) ── GND
D8 ─► LED ─[1k]─ GND
```

## Code

```cpp
// G11 Activity 04 - Serial Keyboard Music Box
const int BUZZER_PIN = 9;
const int LED_PIN = 8;

const char KEYS[] = "asdfghjk";
const int BASE[8] = {262, 294, 330, 349, 392, 440, 494, 523};   // C4..C5
const int Q = 64;
const int REC_MAX = 100;

char queue[Q];
int qHead = 0, qTail = 0, qCount = 0;

char rec[REC_MAX];
int recLen = 0;
bool recording = false;

int octave = 0;            // -1 .. +2
int noteLen = 250;
bool busy = false;
unsigned long noteEnd = 0, gapEnd = 0;

int keyIndex(char c) {
  for (int i = 0; i < 8; i++) if (KEYS[i] == c) return i;
  return -1;
}

bool push(char c) {
  if (qCount >= Q) {
    Serial.println("Queue full");
    return false;
  }
  queue[qTail] = c;
  qTail = (qTail + 1) % Q;
  qCount++;
  return true;
}

char pop() {
  char c = queue[qHead];
  qHead = (qHead + 1) % Q;
  qCount--;
  return c;
}

int freqFor(int idx) {
  long f = BASE[idx];
  if (octave > 0) f <<= octave;
  if (octave < 0) f >>= -octave;
  return (int)f;
}

void help() {
  Serial.println("--- MUSIC BOX ---");
  Serial.println(" a s d f g h j k = C D E F G A B C   . = rest");
  Serial.println(" 1-4 length | + - octave | r record | p play | x stop | ? help");
}

void handle(char c) {
  if (keyIndex(c) >= 0 || c == '.') {
    push(c);
  } else if (c >= '1' && c <= '4') {
    noteLen = 125 << (c - '1');
    Serial.print("Note length ");
    Serial.print(noteLen);
    Serial.println(" ms");
  } else if (c == '+' || c == '-') {
    octave = constrain(octave + (c == '+' ? 1 : -1), -1, 2);
    Serial.print("Octave shift ");
    Serial.println(octave);
  } else if (c == 'r') {
    recording = !recording;
    if (recording) recLen = 0;
    Serial.print(recording ? "REC start" : "REC stop, notes: ");
    if (!recording) Serial.print(recLen);
    Serial.println();
  } else if (c == 'p') {
    Serial.print("Playing ");
    Serial.print(recLen);
    Serial.println(" recorded notes");
    recording = false;
    for (int i = 0; i < recLen; i++) if (!push(rec[i])) break;
  } else if (c == 'x') {
    qHead = qTail = qCount = 0;
    noTone(BUZZER_PIN);
    busy = false;
    Serial.println("Stopped");
  } else if (c == '?') {
    help();
  } else if (c != '\n' && c != '\r' && c != ' ') {
    Serial.print("Unknown '");
    Serial.print(c);
    Serial.println("'");
  }
}

void player() {
  unsigned long now = millis();
  if (busy) {
    if ((long)(now - noteEnd) >= 0) {
      noTone(BUZZER_PIN);
      digitalWrite(LED_PIN, LOW);
      busy = false;
      gapEnd = now + 30;
    }
    return;
  }
  if ((long)(now - gapEnd) < 0 || qCount == 0) return;

  char c = pop();
  if (recording && recLen < REC_MAX) rec[recLen++] = c;
  int idx = keyIndex(c);
  if (idx >= 0) {
    tone(BUZZER_PIN, freqFor(idx));
    digitalWrite(LED_PIN, HIGH);
    Serial.print(c);
  } else {
    Serial.print('.');
  }
  busy = true;
  noteEnd = now + noteLen - 30;
  if (qCount == 0) Serial.println();
}

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  help();
  const char *intro = "asdfghjk";
  for (int i = 0; intro[i]; i++) push(intro[i]);        // demo scale
}

void loop() {
  while (Serial.available() > 0) handle(Serial.read());
  player();
}
```

## Assessment Questions

1. What do you hear right after the simulation starts?
2. Send `asdf` in one line. Do you hear all the notes at the same time or one after another?
3. Send `1` and then `asdf`, then `4` and `asdf`. What is the difference? What changes when you send `+` before the notes?
4. Send `r`, play a few notes, send `r` again, then send `p`. What happens?
5. What will happen to the LED if its resistor is removed? If the buzzer is disconnected, does the LED still show the notes?
