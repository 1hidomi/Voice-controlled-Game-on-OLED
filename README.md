**A gesture-activated, voice-controlled pitch-training game with an automated mechanical enclosure.**

PitchPop is a real-time embedded game in which the player controls an on-screen ball by producing a vocal pitch within a target frequency range. The ball must pass through moving obstacles displayed on a 96×64 RGB OLED screen.

The project combines digital signal processing, interactive graphics, infrared presence detection, encoder-based motor control and a custom mechanical deployment mechanism into one self-contained system.

Developed for **COMP0207: Introduction to Electronics** at University College London.

## Repository Structure

```text
Voice-controlled Game on OLED/
├── CAD_PitchPop.zip
├── Report.pdf
├── Photos.zip
└── README.md
```

A complete demonstration should show:

* the motorised platform deploying;
* the OLED rotating into position;
* gesture-based game activation;
* voice-controlled ball movement;
* obstacle avoidance and score tracking;
* automatic mechanical retraction.

## Overview

PitchPop was designed as an interactive pitch-training system for users who find it difficult to sing a note accurately.

Instead of using buttons or a joystick, the game receives control input from a microphone. Audio samples are processed using a Fast Fourier Transform, allowing the microcontroller to estimate the dominant frequency in real time.

When a sufficiently loud sound is detected between approximately **350 Hz and 600 Hz**, the ball receives an upward impulse. Gravity continuously pulls the ball downward, so the player must repeatedly produce the correct pitch to navigate through gaps in the obstacles.

The system also includes a motorised enclosure:

1. A belt mechanism raises the gaming station.
2. A second motor rotates the OLED into its viewing position.
3. An infrared sensor waits for the player to start.
4. The game responds to vocal input.
5. After game over, the player can restart using the IR sensor.
6. If the game is not restarted, the mechanism retracts automatically.

## Key Features

* Real-time microphone sampling
* 256-point Fast Fourier Transform
* Hamming-window spectral analysis
* Frequency- and magnitude-based voice detection
* Voice-controlled ball physics
* Random obstacle generation
* Collision detection and score tracking
* Gesture/proximity-based game activation
* Dual DC motor actuation
* Quadrature encoder feedback
* Automatic deployment and retraction
* Finite-state-machine control architecture
* SPI-driven 96×64 RGB OLED graphics

## System Architecture

PitchPop uses an **Arduino Uno R4 WiFi** as the central controller.

The microcontroller:

* samples the microphone through its ADC;
* extracts the dominant audio frequency using FFT;
* updates game physics and collision logic;
* renders graphics to the OLED over SPI;
* measures the IR sensor's RC discharge time;
* sends motor commands to the Motoron controller over I²C;
* receives encoder feedback through interrupt-capable pins.

```text
                        ┌──────────────────────┐
                        │ Arduino Uno R4 WiFi  │
                        │   Main controller    │
                        └──────────┬───────────┘
                                   │
             ┌─────────────────────┼─────────────────────┐
             │                     │                     │
             ▼                     ▼                     ▼
     Microphone module      QTR IR sensor        SSD1331 OLED
        Analog ADC          RC pulse timing             SPI
             │                                           │
             └────────── FFT and game logic ─────────────┘
                                   │
                                   ▼
                         Motoron M3S550
                               I²C
                            ┌──────┴──────┐
                            ▼             ▼
                         Motor 1       Motor 2
                       Linear belt   Screen rotation
                            ▲             ▲
                         Encoder 1     Encoder 2
```

## Gameplay

The player controls a ball moving vertically on the OLED.

The obstacle travels from right to left:

```cpp
obsX -= 3;
```

The ball follows a simple gravity model:

```cpp
velocity += gravity;
ballY += velocity;
```

When an intentional sound within the configured pitch range is detected, the ball is pushed upward:

```cpp
velocity = -3.5;
```

A point is awarded whenever an obstacle leaves the display and is regenerated with a new random gap:

```cpp
if (obsX < -obsW) {
    obsX = 96;
    gapY = random(5, 35);
    score++;
}
```

The game ends when the ball horizontally overlaps an obstacle while lying outside its opening.

## Audio Processing

### Sampling

The microphone is connected to analog input `A0`.

The program collects 256 samples at a nominal sampling frequency of 10 kHz:

```cpp
#define SAMPLES       256
#define SAMPLING_FREQ 10000
```

Each sample is taken approximately every 100 microseconds:

```cpp
for (int i = 0; i < SAMPLES; i++) {
    int raw = analogRead(MIC_PIN);
    vReal[i] = raw;
    vImag[i] = 0;
    delayMicroseconds(100);
}
```

### DC-offset removal

The microphone waveform is biased around a non-zero voltage. The average sample value is therefore subtracted before spectral analysis:

```cpp
float avg = sum / SAMPLES;

for (int i = 0; i < SAMPLES; i++) {
    vReal[i] = (vReal[i] - avg) * SOFT_GAIN;
}
```

This reduces the DC component and stabilises pitch detection.

### FFT analysis

The program applies a Hamming window before computing the FFT:

```cpp
FFT.windowing(FFT_WIN_TYP_HAMMING, FFT_FORWARD);
FFT.compute(FFT_FORWARD);
FFT.complexToMagnitude();
```

The strongest non-DC frequency component is then located within the positive-frequency half of the spectrum.

### Trigger conditions

The ball only responds when both conditions are satisfied:

```cpp
absolutePeakFreq >= TARGET_MIN &&
absolutePeakFreq <= TARGET_MAX &&
absoluteMaxMag > NOISE_THRESHOLD
```

Default calibration values:

```cpp
#define SOFT_GAIN       12.0
#define NOISE_THRESHOLD 9500
#define TARGET_MIN      350
#define TARGET_MAX      600
```

A vocal trigger near **430 Hz** was used during prototype testing.

The dual frequency-and-magnitude constraint helps reject background noise and accidental low-level sounds.

## State Machine

The complete system is controlled by a finite state machine.

```text
BELT
  │
  ▼
ROTATING
  │
  ▼
WAITING_IR
  │  IR activation
  ▼
PLAYING
  │  collision
  ▼
GAME_OVER
  ├── IR activation ───────────────► PLAYING
  │
  └── timeout
         ▼
RETRACTING_MOTOR
         │
         ▼
RETRACTING_BELT
         │
         └─────────────────────────► BELT
```

| State              | Purpose                                                            |
| ------------------ | ------------------------------------------------------------------ |
| `BELT`             | Motor 1 raises or delivers the gaming platform                     |
| `ROTATING`         | Motor 2 rotates the OLED mechanism into position                   |
| `WAITING_IR`       | The system displays the start prompt and waits for proximity input |
| `PLAYING`          | Audio processing, physics, rendering and collision detection run   |
| `GAME_OVER`        | The final score is displayed and restart input is accepted         |
| `RETRACTING_MOTOR` | Motor 2 returns the screen mechanism                               |
| `RETRACTING_BELT`  | Motor 1 retracts the gaming platform                               |

Using explicit states keeps gameplay, sensing and mechanical motion separated and prevents conflicting motor commands.

## Mechanical System

PitchPop uses two independently controlled DC motors.

### Motor 1: linear deployment

Motor 1 drives a toothed-belt mechanism that raises the gaming platform.

The target displacement is approximately:

```text
d = 0.23 m
```

The displacement can be estimated from encoder counts using:

```text
d = (N / T) × 2πR
```

where:

* `N` is the accumulated encoder count;
* `T = 384` ticks per output-shaft revolution;
* `R = 0.025 m` is the effective pulley radius.

This produces a theoretical target of approximately:

```text
N ≈ 553 ticks
```

The implementation therefore stops Motor 1 after 553 encoder counts.

### Motor 2: screen rotation

Motor 2 controls the orientation of the OLED assembly.

The encoder-based angular relationship is:

```text
θ = N × 2π / T
```

The final code uses a configurable encoder target derived from the desired rotation angle and the motor's 384-tick output resolution.

### Encoder feedback

Both motors use quadrature encoders.

The A-channel edges trigger interrupt service routines, while the B channel determines direction:

```cpp
if (currentA != lastA_M2) {
    if (currentB != currentA) {
        encoderCountM2++;
    } else {
        encoderCountM2--;
    }
}
```

This provides position feedback that is more repeatable than estimating displacement from motor runtime alone.

## Infrared Interaction

The QTR-HD-09RC sensor is read using RC charge-and-decay timing.

The microcontroller:

1. configures the sensor pin as an output;
2. drives it high to charge the internal capacitor;
3. changes the pin to input mode;
4. measures how long the signal remains high.

```cpp
long readQTR(int pin) {
    pinMode(pin, OUTPUT);
    digitalWrite(pin, HIGH);
    delayMicroseconds(10);

    pinMode(pin, INPUT);

    long start = micros();

    while (
        digitalRead(pin) == HIGH &&
        micros() - start < 3000
    );

    return micros() - start;
}
```

A short discharge time indicates stronger reflected infrared light and therefore a nearby hand or object:

```cpp
if (readQTR(IR_PIN) < 800) {
    // Start or restart the game
}
```

## Hardware

| Component        | Model                        | Purpose                                |
| ---------------- | ---------------------------- | -------------------------------------- |
| Microcontroller  | Arduino Uno R4 WiFi          | Central processing and control         |
| OLED display     | Adafruit SSD1331             | Game graphics and user feedback        |
| Microphone       | SparkFun SEN-12642           | Real-time audio acquisition            |
| IR sensor        | Pololu QTR-HD-09RC           | Gesture and proximity detection        |
| Motor controller | Pololu Motoron M3S550        | Bidirectional motor control            |
| DC motors        | 2× DG01D-E with encoders     | Linear deployment and display rotation |
| Power supply     | 8 V DC supply                | Motor and system power                 |
| Mechanical parts | Custom 3D-printed components | Enclosure, mounts and screen support   |

## Pin Connections

### OLED display

| OLED signal | Arduino pin |
| ----------- | ----------: |
| CS          |         D10 |
| RESET       |          D9 |
| DC          |          D8 |
| SCLK        |         D13 |
| MOSI        |         D11 |
| VCC         |         5 V |
| GND         |         GND |

The display is operated using SPI.

### Microphone

| Microphone signal | Arduino pin |
| ----------------- | ----------: |
| AUDIO             |          A0 |
| VCC               |         5 V |
| GND               |         GND |

### IR sensor

| Sensor signal | Arduino pin |
| ------------- | ----------: |
| Sensor output |          A1 |
| VCC           |         5 V |
| GND           |         GND |

### Motor 1 encoder

| Encoder signal | Arduino pin |
| -------------- | ----------: |
| Channel A      |          D3 |
| Channel B      |         D12 |
| VCC            |         5 V |
| GND            |         GND |

### Motor 2 encoder

| Encoder signal | Arduino pin |
| -------------- | ----------: |
| Channel A      |          D2 |
| Channel B      |          D5 |
| VCC            |         5 V |
| GND            |         GND |

The Motoron M3S550 is mounted directly onto the Arduino and communicates using I²C.

## Software Dependencies

Install the following Arduino libraries:

* `Motoron`
* `Adafruit GFX Library`
* `Adafruit SSD1331 OLED Driver Library`
* `arduinoFFT`
* `SPI`
* `Wire`

The sketch should include:

```cpp
#include <Wire.h>
#include <SPI.h>
#include <Motoron.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1331.h>
#include "arduinoFFT.h"
```

## Building and Uploading

1. Clone the repository:

```bash
git clone https://github.com/YOUR_USERNAME/pitchpop.git
cd pitchpop
```

2. Open the Arduino sketch in the Arduino IDE.

3. Install the required libraries through the Library Manager.

4. Select **Arduino Uno R4 WiFi** as the target board.

5. Connect the hardware according to the pin tables.

6. Connect the motor system to the external 8 V supply.

7. Upload the sketch.

8. Open the Serial Monitor at:

```text
115200 baud
```

## Experimental Results

The completed prototype successfully integrated:

* real-time audio acquisition;
* FFT-based frequency extraction;
* OLED game rendering;
* IR-based user interaction;
* two-axis mechanical movement;
* encoder-based motor positioning.

During playtesting, users were able to complete multiple increasingly difficult obstacle sequences, with the strongest tester performance reaching approximately nine levels.

The 256-sample, 10 kHz FFT configuration provided a practical balance between spectral resolution and processing latency.

An 8 V motor supply was selected after testing showed that 6 V was insufficient to start and operate both motors reliably.

## Noise Mitigation

Several signal-processing problems were addressed during development.

### Electrical interference

A strong interference component near 3 kHz was observed during testing.

The final trigger logic ignores frequency components outside the intended vocal band.

### DC bias

Dynamic mean subtraction removes the microphone's baseline voltage before the FFT is calculated.

### Accidental triggering

A frequency threshold alone was insufficient because low-amplitude environmental signals could still produce spectral peaks.

The final system therefore requires:

* a dominant frequency between 350 Hz and 600 Hz;
* a magnitude greater than 9500.

## Limitations

### Impulse-based control

A valid pitch applies a fixed upward impulse rather than directly mapping pitch to the ball's vertical position.

This creates a perceptible control delay and makes precise movement more difficult.

### Blocking audio acquisition

The program gathers all 256 microphone samples inside the main game loop using `delayMicroseconds()`.

During this acquisition period, rendering and physics updates are paused.

### Mechanical alignment

The belt mechanism experienced some alignment error due to tolerances in the belt structure and 3D-printed components.

### Fixed movement targets

Motor positions are determined using fixed encoder counts.

The current design does not use physical limit switches or a homing procedure.

### Limited game progression

The current firmware uses a single recurring obstacle and does not yet contain a fully structured level-management system.

### Emergency stop

A push-button emergency stop was considered during development, but it was not physically integrated into the final prototype.

## Future Improvements

* Replace impulse control with direct pitch-to-position mapping
* Use timer interrupts for consistent audio sampling
* Process audio without blocking the game loop
* Add smoothing across consecutive FFT results
* Add adaptive noise-floor calibration
* Introduce multiple pitch targets
* Load note sequences from songs
* Add progressively faster obstacles
* Add a persistent high-score system
* Add physical limit switches
* Implement automatic motor homing
* Improve belt alignment and mechanical rigidity
* Add a hardware emergency-stop button
* Separate audio, motor and game logic into reusable modules
* Use the full QTR sensor array for more advanced gestures

## Applications

Although developed as a game, the interaction model could also be extended to:

* introductory pitch training;
* music education;
* vocal warm-up exercises;
* speech and voice rehabilitation;
* interactive installations;
* accessible hands-free games.

Custom note sequences could turn the current obstacle system into a song-practice tool in which the player follows a predefined melody.

## Contributors

* Celine Wang
* Jasmine Chen
* Yutong Liu

**Team 13 — COMP0207: Introduction to Electronics**

MEng Robotics and Artificial Intelligence
Department of Computer Science
University College London

## Academic Context

This project was completed as coursework for UCL COMP0207.

The repository is provided for demonstration, documentation and portfolio purposes. Current students should follow their institution's academic-integrity requirements before reusing any part of the implementation.

