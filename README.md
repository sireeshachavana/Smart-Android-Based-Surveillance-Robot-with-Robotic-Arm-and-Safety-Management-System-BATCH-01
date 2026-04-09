#define BLYNK_TEMPLATE_ID   "TMPL3pysIwKEL"
#define BLYNK_TEMPLATE_NAME "Robot"
#define BLYNK_AUTH_TOKEN    "z4YrYq5wt16OdNgyB2wK6J2cXfux5aRK"

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <Wire.h>
#include <Adafruit_PWMServoDriver.h>

char ssid[] = "Bandham437";
char pass[] = "logout437";

// --- Pin Definitions ---
#define ENA 13
#define IN1 12
#define IN2 14
#define ENB 25
#define IN3 27
#define IN4 26
#define TRIG 5
#define ECHO 18
#define MQ2_PIN 34 

// --- Rescue & Fire Pins ---
#define METAL_PIN 32   // Digital Input (PNP: HIGH = Metal)
#define FIRE_PIN 33    // Digital Input (Active LOW)
#define RELAY_PIN 15   // Digital Output (Active LOW)

Adafruit_PWMServoDriver pwm = Adafruit_PWMServoDriver();
BlynkTimer timer;

#define SERVOMIN  150
#define SERVOMAX  600
int joyX = 0, joyY = 0;
bool isRecoiling = false;
int gasThreshold = 1200; 

// --- Alert Locks ---
bool gasAlertSent = false; 
bool metalAlertSent = false;
bool fireAlertSent = false;
bool manualPumpOverride = false; // V10

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  pinMode(TRIG, OUTPUT); pinMode(ECHO, INPUT);
  ledcAttach(ENA, 20000, 8); 
  ledcAttach(ENB, 20000, 8);

  pinMode(METAL_PIN, INPUT);
  pinMode(FIRE_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, HIGH); // Pump OFF at boot

  pwm.begin();
  pwm.setPWMFreq(60);

  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  timer.setInterval(100L, checkDistance);     
  timer.setInterval(500L, checkRescueSensors); 
  timer.setInterval(1000L, checkGasSensor);   
}

void checkGasSensor() {
  int gasValue = analogRead(MQ2_PIN);
  Blynk.virtualWrite(V6, gasValue); 
  
  if (gasValue > gasThreshold && !gasAlertSent) {
    Blynk.logEvent("gas_alert", "EMERGENCY: Gas/Smoke Detected!");
    gasAlertSent = true; 
  } 
  if (gasValue < (gasThreshold - 200)) { 
    gasAlertSent = false; 
  }
}

void checkRescueSensors() {
  // PNP Sensor Logic: HIGH means metal is detected
  bool metalDetected = (digitalRead(METAL_PIN) == HIGH);
  Blynk.virtualWrite(V8, metalDetected ? 1 : 0); 
  
  if (metalDetected && !metalAlertSent) {
    Blynk.logEvent("metal_detected", "ALERT: Metal Object Located!");
    metalAlertSent = true;
  } else if (!metalDetected) {
    metalAlertSent = false;
  }

  // Fire Detection Logic: LOW means fire is detected
  bool fireDetected = (digitalRead(FIRE_PIN) == LOW);
  Blynk.virtualWrite(V9, fireDetected ? 1 : 0);

  // Pump triggers on fire OR manual override
  if (fireDetected || manualPumpOverride) {
    digitalWrite(RELAY_PIN, LOW); 
    if (fireDetected && !fireAlertSent) {
      Blynk.logEvent("fire_alert", "FIRE DETECTED! WATER PUMP ENGAGED!");
      fireAlertSent = true;
    }
  } else {
    digitalWrite(RELAY_PIN, HIGH); 
    fireAlertSent = false;
  }
}

BLYNK_WRITE(V10) {
  manualPumpOverride = param.asInt(); 
}

BLYNK_WRITE(V2) { pwm.setPWM(0, 0, map(param.asInt(), 0, 180, SERVOMIN, SERVOMAX)); }
BLYNK_WRITE(V3) { pwm.setPWM(1, 0, map(param.asInt(), 0, 180, SERVOMIN, SERVOMAX)); }
BLYNK_WRITE(V4) { pwm.setPWM(2, 0, map(param.asInt(), 0, 180, SERVOMIN, SERVOMAX)); }
BLYNK_WRITE(V5) { pwm.setPWM(3, 0, map(param.asInt(), 0, 180, SERVOMIN, SERVOMAX)); }

void checkDistance() {
  digitalWrite(TRIG, LOW); delayMicroseconds(2);
  digitalWrite(TRIG, HIGH); delayMicroseconds(10);
  digitalWrite(TRIG, LOW);
  long duration = pulseIn(ECHO, HIGH, 30000); 
  int distance = (duration == 0) ? 999 : (duration * 0.034 / 2);
  Blynk.virtualWrite(V7, distance);

  if (distance > 1 && distance <= 15 && !isRecoiling) {
    isRecoiling = true;
    drive(-255, -255); 
    timer.setTimeout(2000L, []() {
      drive(0, 0);
      isRecoiling = false;
    });
  }
}

BLYNK_WRITE(V0) { if(!isRecoiling) joyX = param.asInt(); }
BLYNK_WRITE(V1) { if(!isRecoiling) joyY = param.asInt(); }

void drive(int left, int right) {
  left = constrain(left, -255, 255);
  right = constrain(right, -255, 255);
  
  if (left > 40) { digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH); ledcWrite(ENA, abs(left)); }
  else if (left < -40) { digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW); ledcWrite(ENA, abs(left)); }
  else { digitalWrite(IN1, LOW); digitalWrite(IN2, LOW); ledcWrite(ENA, 0); }
  
  if (right > 40) { digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH); ledcWrite(ENB, abs(right)); }
  else if (right < -40) { digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW); ledcWrite(ENB, abs(right)); }
  else { digitalWrite(IN3, LOW); digitalWrite(IN4, LOW); ledcWrite(ENB, 0); }
}

void loop() {
  Blynk.run();
  timer.run();
  if (!isRecoiling) drive(joyY + joyX, joyY - joyX);
}
# Project Prototype
https://github.com/sireeshachavana/Smart-Android-Based-Surveillance-Robot-with-Robotic-Arm-and-Safety-Management-System/issues/1#issue-4231530344
# Robotic Arm
![Image](https://github.com/user-attachments/assets/d6144233-c04a-47ee-b7d2-3b46255df624)
# Block Diagram
![Image](https://github.com/user-attachments/assets/908c5778-bdab-40dd-bc6b-7bfae4acf16e)
