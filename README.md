# Design of A Solar Powered Intelligent Waste Management System 

## ⚙️ Project Overview

The **Solar-Powered Intelligent Waste Management System (SPIWMS)** is an embedded-mechanical system designed to improve waste disposal efficiency through:

* Automated lid operation (contactless use)
* Real-time waste level monitoring
* Mechanical waste compression
* Solar-powered energy autonomy
* GSM-based alert system

Unlike conventional smart bins that only monitor waste levels, this system **actively increases storage capacity** through a **motor-driven compression mechanism**, reducing the frequency of waste collection.


## ⚙️ Key Features

*  **Solar-powered system** (30W panel + 12V battery)
*  **Arduino-based control system**
*  **GSM alert when bin is full**
*  **Dual ultrasonic sensing**
*  **Motor-driven compression system**
*  **Time-based control (NO limit switches)**
*  **Visual status indicators (LEDs)** 🟢🔴

## ⚙️ System Architecture

<img width="1280" height="720" alt="WhatsApp Image 2026-05-02 at 11 33 23 (1)" src="https://github.com/user-attachments/assets/25b947d7-9150-4aef-ba4e-9fbfd7faf5ad" />

### Inputs:

* Ultrasonic Sensor 1 (User/Lid Detection)
* Ultrasonic Sensor 2 (Waste Level Detection)
* Reset Switch

### Processing Unit:

* Arduino UNO (Microcontroller)

### Outputs:

* Servo Motor (Lid Control)
* DC Motor via Motor Driver (Compression)
* GSM Module (SMS Alerts)
* LEDs (Status Indication)

### Power System:

* Solar Panel → Charge Controller → Battery
* Battery → Arduino + Motors


## ⚙️ Hardware Components

###  Core Components

* **Microcontroller:** Arduino UNO
* **Main Motor:** 775 DC Motor (12V, high torque)
* **Motor Driver:** H-Bridge (L298N)
* **Servo Motor:** SG90 (Lid control)
* **Sensors:** HC-SR04 Ultrasonic Sensors (×2)
* **Communication:** GSM Module (SIM800)
* **Power System:**

  * 30W Solar Panel
  * 12V 7Ah Battery
  * PWM Charge Controller

<img width="1280" height="720" alt="WhatsApp Image 2026-05-02 at 11 33 23" src="https://github.com/user-attachments/assets/054e2a53-324e-4a5c-96b4-d8e57333a0a4" />


## ⚙️ Mechanical System

###  Compression Mechanism

The compression system is **NOT an actuator**. It is:

> A **belt-and-pulley driven threaded rod mechanism** powered by a **775 DC motor**.

### Working Principle:

1. Motor rotates pulley
2. Belt transfers motion
3. Pulley rotates threaded rod
4. Thread converts rotation → linear motion
5. Metal plate moves **down (compression)** and **up (reset)**


## ⚙️ Control Strategy

❗ **No limit switches were used for this design**

Instead, motion is controlled using **time-based control**:

*  Downward motion → **8 seconds**
*  Upward motion → **8 seconds**

<img width="463" height="898" alt="WhatsApp Image 2026-05-02 at 11 55 06" src="https://github.com/user-attachments/assets/f483bdb3-2d5c-4faa-b0c1-65d3a8159177" />


This simplifies:

* Wiring
* Cost
* Mechanical complexity


## ⚙️ System Operation Flow

1. System continuously reads both ultrasonic sensors
2. If user detected → lid opens
3. If waste level exceeds threshold:

   * Compression is triggered
4. Compression runs for fixed time:

   * Down (8s) → Pause → Up (8s)
5. Compression count increases
6. If max compression reached AND still full:

   * System flags **FULL**
   * Sends SMS alert


## ⚙️ Engineering Calculations

### Pulley Ratio

$Ratio = \frac{50}{40} = 1.25$

### Lead Screw Force

$F = \frac{P}{2 \pi \tau}$

* Theoretical Force ≈ **3142 N**
* Actual (35% efficiency) ≈ **1100 N**


### Self-Locking Condition

* Lead Angle = 2.85°
* Friction Angle = 8.53°

✔ Since friction > lead angle → **System is self-locking**

➡ Plate will NOT fall when motor stops


## ⚙️ Power Analysis

* Battery: 12V, 7Ah
* Average system consumption optimized via:

  * Sensor duty cycling
  * Time-based motor operation
 
<img width="768" height="1024" alt="WhatsApp Image 2026-05-02 at 11 49 08" src="https://github.com/user-attachments/assets/7439ceff-0fa2-449b-b35f-b72f3fd515e6" />


### Runtime:
Was not effectively determined during the course of the project


## ⚙️ Software Logic

### Key Behaviors:

* Lid opens when object < 20cm
* Compression triggers when bin level < 25cm
* Maximum compression cycles = 2
* SMS sent only once when full
* Reset button clears system state

<img width="1280" height="720" alt="WhatsApp Image 2026-05-02 at 11 49 10" src="https://github.com/user-attachments/assets/6a09e1b7-7780-4aeb-9ff4-78aedf887715" />


## ⚙️ Arduino Code

```cpp
#include <Servo.h>
#include <SoftwareSerial.h>

// ---------------- PIN CONFIG ----------------

// Ultrasonic 1 (LID / INCOMING WASTE)
const int trigPin1 = 3;
const int echoPin1 = 4;

// Ultrasonic 2 (BIN LEVEL)
const int trigPin2 = A1;
const int echoPin2 = A5;

// LEDs
const int greenLED = 5;
const int redLED = 6;

// Motor Driver
const int motorIn1 = 8;
const int motorIn2 = 7;
const int ENA = 9;

// Servo
const int servoPin = 10;

// Reset Switch
const int resetSwitch = 2;

// GSM
SoftwareSerial gsm(11, 13); // RX, TX

// ---------------- VARIABLES ----------------
Servo lidServo;

long duration;
float distance1; // for lid
float distance2; // for bin level

int compressionCount = 0;
const int maxCompression = 2;

bool isFull = false;
bool isCompressing = false;
bool smsSent = false;

// Thresholds
const int lidTriggerDistance = 20;   // cm (incoming waste)
const int levelTriggerDistance = 25; // cm (bin level)

// ---------------- SETUP ----------------
void setup() {

  pinMode(trigPin1, OUTPUT);
  pinMode(echoPin1, INPUT);

  pinMode(trigPin2, OUTPUT);
  pinMode(echoPin2, INPUT);

  pinMode(greenLED, OUTPUT);
  pinMode(redLED, OUTPUT);

  pinMode(motorIn1, OUTPUT);
  pinMode(motorIn2, OUTPUT);
  pinMode(ENA, OUTPUT);

  pinMode(resetSwitch, INPUT_PULLUP);

  lidServo.attach(servoPin);
  lidServo.write(0);

  Serial.begin(9600);
  gsm.begin(9600);

  digitalWrite(greenLED, HIGH);

  delay(5000); // stabilization
}

// ---------------- LOOP ----------------
void loop() {

  // RESET CHECK
  if (digitalRead(resetSwitch) == LOW) {
    resetSystem();
  }

  // READ SENSORS
  distance1 = getDistance(trigPin1, echoPin1);
  distance2 = getDistance(trigPin2, echoPin2);

  Serial.print("Lid Sensor: ");
  Serial.print(distance1);
  Serial.print(" | Bin Level: ");
  Serial.println(distance2);

  // ---------------- FULL STATE ----------------
  if (distance2 <= levelTriggerDistance && compressionCount >= maxCompression) {

    isFull = true;

    digitalWrite(greenLED, LOW);
    digitalWrite(redLED, HIGH);

    if (!smsSent) {
      sendSMS();
      smsSent = true;
    }

    return;
  }

  // ---------------- NORMAL STATE ----------------
  digitalWrite(greenLED, HIGH);
  digitalWrite(redLED, LOW);

  // ---------------- LID CONTROL ----------------
  if (distance1 <= lidTriggerDistance && !isCompressing && !isFull) {

    Serial.println("Opening Lid");

    lidServo.write(0);
    delay(5000);

    lidServo.write(90);
    delay(1500);
  }

  // ---------------- COMPRESSION ----------------
  if (distance2 <= levelTriggerDistance && !isCompressing && compressionCount < maxCompression) {
    runCompression();
  }
}

// ---------------- ULTRASONIC FUNCTION ----------------
float getDistance(int trigPin, int echoPin) {

  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);

  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  long duration = pulseIn(echoPin, HIGH);

  return duration * 0.034 / 2;
}

// ---------------- COMPRESSION FUNCTION ----------------
void runCompression() {

  isCompressing = true;

  Serial.println("Compression Running");

  // Ensure lid is closed
  lidServo.write(90);
  delay(500);

  analogWrite(ENA, 200);

  // DOWN (clockwise)
  digitalWrite(motorIn1, HIGH);
  digitalWrite(motorIn2, LOW);
  delay(8000);

  // STOP
  digitalWrite(motorIn1, LOW);
  digitalWrite(motorIn2, LOW);
  delay(2000);

  // UP (anticlockwise)
  digitalWrite(motorIn1, LOW);
  digitalWrite(motorIn2, HIGH);
  delay(8000);

  // STOP
  digitalWrite(motorIn1, LOW);
  digitalWrite(motorIn2, LOW);

  compressionCount++;

  Serial.print("Compression Count: ");
  Serial.println(compressionCount);

  isCompressing = false;
}

// ---------------- GSM ----------------
void sendSMS() {

  Serial.println("Sending SMS...");

  gsm.println("AT+CMGF=1");
  delay(1000);

  gsm.println("AT+CMGS=\"+XXXXXXXXXXX\"");
  delay(1000);

  gsm.print("Smart Bin Alert: Bin FULL after compression.");
  delay(100);

  gsm.write(26);
  delay(5000);

  Serial.println("SMS Sent");
}

// ---------------- RESET ----------------
void resetSystem() {

  Serial.println("System Reset");

  compressionCount = 0;
  isFull = false;
  isCompressing = false;
  smsSent = false;

  digitalWrite(redLED, LOW);
  digitalWrite(greenLED, HIGH);
}
```


##  Design Decisions

### Why Time-Based Instead of Limit Switches?

* Reduces hardware complexity
* Eliminates mechanical alignment issues
* Easier to debug and adjust
* Cost-effective

Trade-off:

* Less positional accuracy

---

## ⚙️ Challenges Encountered

* Belt slippage and snapping
* Incorrect initial power sizing
* Servo burnout due to previous poor understanding of the power source requirements
* Mechanical alignment issues
* Late-stage redesign of waste outlet

---

## ⚙️ Improvements for Future Work

* Add limit switches for precision
* Replace SLA battery with Li-ion
* Improve belt tensioning system
* Add IoT dashboard
* Integrate AI-based waste classification


## ⚙️ Project Status
`
> **Note:** This project is still under development and is being considered for future improvements and optimization, particularly in mechanical stability, energy efficiency, and system intelligence.


## 👩‍💻 Author

**Glory Akanbi**

Electronic & Electrical Engineering Student

Obafemi Awolowo University


Team Members-  Ogundimu Shakirat Abdulbaasit Olaore Saka Mukaram DANIEL GBOLAGUN Hadi Suleiman Abdul-lateef Rabiu Grace Oluyemi 
Supervisor- Dr. A.M. Jubril

## ⭐ Final Note

This project is more than a smart bin.
It is a **full integration of mechanical systems, embedded control, power electronics, and real-world engineering constraints**.

<img width="768" height="800" alt="WhatsApp Image 2026-04-24 at 09 03 36" src="https://github.com/user-attachments/assets/bcb993b2-c4e6-4c22-8204-9e3de5700ff7" />



