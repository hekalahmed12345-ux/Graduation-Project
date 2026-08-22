#include <Servo.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <SoftwareSerial.h>

// =====================================================
// Bluetooth HC-05
// =====================================================
SoftwareSerial BT(A2, A3);  // HC-05 TXD -> A2 Arduino RX
                            // HC-05 RXD -> A3 Arduino TX

char btCommand = 'S';
bool manualMode = false;
int stopPressCount = 0;
const char* manualMotionText = "STOP";

// =====================================================
// LCD
// =====================================================
LiquidCrystal_I2C lcd(0x27, 16, 2);

unsigned long lastLcdUpdate = 0;
const int lcdInterval = 200;

// =====================================================
// Servos
// =====================================================
Servo scanServo;
Servo armBaseServo;
Servo armUpServo;
Servo armExtendServo;

// =====================================================
// Pins
// =====================================================
const int scanPin = 8;
const int basePin = 5;
const int upPin = 6;
const int extendPin = 7;

const int laserPin = 9;
const int buzzerPin = 12;

const int trigPin = 10;
const int echoPin = 11;

const int IN1 = 2;
const int IN2 = 3;
const int IN3 = 4;
const int IN4 = 13;

const int irLeftPin = A0;
const int irRightPin = A1;

const int IR_OBSTACLE_STATE = LOW;

// =====================================================
// Arm Positions
// =====================================================
const int baseHome = 77;
const int upHome = 20;
const int extendHome = 160;

const int upTarget = 90;
const int extendTarget = 90;

// =====================================================
// Scanner
// =====================================================
int scanAngle = 100;
int scanDirection = 1;

const int scanMin = 40;
const int scanMax = 170;
const int scanStep = 3;
const int scanDelay = 30;

unsigned long lastScanTime = 0;

// =====================================================
// Detection
// =====================================================
const int detectMin = 4;
const int detectMax = 35;

const int requiredConfirmations = 4;

int detectionCount = 0;
int detectedAngleSaved = 0;

long lastDistance = 999;
long engagedDistance = 999;

// =====================================================
// Arm Mapping
// =====================================================
const int baseMin = 40;
const int baseMax = 170;

int leftOffset = -25;
int middleOffset = -25;
int rightOffset = -40;

// =====================================================
// Smooth Arm Movement
// =====================================================
float baseCurrent = baseHome;
float upCurrent = upHome;
float extendCurrent = extendHome;

float baseTargetNow = baseHome;
float upTargetNow = upHome;
float extendTargetNow = extendHome;

const float armStep = 0.8;
const int armInterval = 5;

unsigned long lastArmMoveTime = 0;

// =====================================================
// Obstacle Avoidance
// =====================================================
unsigned long avoidStartTime = 0;
int avoidStep = 0;
int avoidTurnDirection = 1;

const unsigned long stopBeforeBackTime = 200;
const unsigned long backTime = 600;
const unsigned long turnTime = 650;
const unsigned long stopAfterTurnTime = 200;

// =====================================================
// System States
// =====================================================
enum SystemState {
  SCANNING,
  AIMING,
  LOCKED,
  RETURNING_HOME,
  AVOID_OBSTACLE
};

SystemState state = SCANNING;

// =====================================================
// Setup
// =====================================================
void setup() {

  Serial.begin(9600);
  BT.begin(9600);

  // Attach Servos
  scanServo.attach(scanPin);
  armBaseServo.attach(basePin);
  armUpServo.attach(upPin);
  armExtendServo.attach(extendPin);

  // Ultrasonic
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  // Laser & Buzzer
  pinMode(laserPin, OUTPUT);
  pinMode(buzzerPin, OUTPUT);

  // IR Sensors
  pinMode(irLeftPin, INPUT);
  pinMode(irRightPin, INPUT);

  // Motor Driver
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Initial Outputs
  digitalWrite(laserPin, LOW);
  digitalWrite(buzzerPin, LOW);

  stopCar();

  // Initial Servo Positions
  scanServo.write(scanAngle);

  armBaseServo.write(baseHome);
  armUpServo.write(upHome);
  armExtendServo.write(extendHome);

  // LCD
  lcd.init();
  lcd.backlight();
  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("EYE OF HORUS");

  lcd.setCursor(0, 1);
  lcd.print("SYSTEM READY");

  startupBeep();

  delay(1500);
}

// =====================================================
// Main Loop
// =====================================================
void loop() {

  updateArmSmooth();
  checkBluetooth();

  if (manualMode) {

    manualControl();
    manualTargetingSystem();

    return;
  }

  if (state == SCANNING) {
    scanMode();
  }

  else if (state == AIMING) {
    aimingMode();
  }

  else if (state == LOCKED) {
    lockedMode();
  }

  else if (state == RETURNING_HOME) {
    returningHomeMode();
  }

  else if (state == AVOID_OBSTACLE) {
    avoidObstacleMode();
  }
}

// =====================================================
// Bluetooth
// =====================================================
void checkBluetooth() {

  while (BT.available()) {

    char command = BT.read();

    if (command == '\n' || command == '\r') {
      continue;
    }

    command = toupper(command);

    Serial.print("BT: ");
    Serial.println(command);

    // -------------------------------------------------
    // Movement Commands
    // -------------------------------------------------
    if (
      command == 'F' ||
      command == 'B' ||
      command == 'L' ||
      command == 'R'
    ) {

      btCommand = command;
      manualMode = true;
      stopPressCount = 0;

      if (state == AVOID_OBSTACLE) {
        state = SCANNING;
      }

      digitalWrite(laserPin, LOW);
      buzzerOff();

      if (state == SCANNING) {

        baseTargetNow = baseHome;
        upTargetNow = upHome;
        extendTargetNow = extendHome;
      }

      updateLCD("MANUAL CONTROL", "SURVEILLANCE ON");
    }

    // -------------------------------------------------
    // Stop Command
    // -------------------------------------------------
    else if (command == 'S') {

      if (!manualMode) {
        manualMode = true;
        stopPressCount = 1;
      }

      else {
        stopPressCount++;
      }

      btCommand = 'S';

      stopCar();

      manualMotionText = "STOP";

      if (stopPressCount >= 2) {

        stopPressCount = 0;
        returnToAutoMode();
      }

      else {
        updateLCD("MANUAL STOP", "SCAN ACTIVE");
      }
    }

    // -------------------------------------------------
    // Auto Mode
    // -------------------------------------------------
    else if (command == 'A') {

      returnToAutoMode();
    }
  }
}

// =====================================================
// Manual Hybrid Mode
// =====================================================
void manualControl() {

  if (
    state == AIMING ||
    state == LOCKED ||
    state == RETURNING_HOME
  ) {

    stopCar();
    return;
  }

  // Safety Stop
  if (isObstacleDetected() && btCommand == 'F') {

    stopCar();

    fastBeep();

    updateLCD(
      "MANUAL SAFETY",
      "OBSTACLE STOP"
    );

    return;
  }

  buzzerOff();

  // Forward
  if (btCommand == 'F') {

    manualMotionText = "FORWARD";
    moveCarForward();
  }

  // Backward
  else if (btCommand == 'B') {

    manualMotionText = "BACKWARD";
    moveCarBackward();
  }

  // Left
  else if (btCommand == 'L') {

    manualMotionText = "LEFT";
    turnCarLeft();
  }

  // Right
  else if (btCommand == 'R') {

    manualMotionText = "RIGHT";
    turnCarRight();
  }

  // Stop
  else if (btCommand == 'S') {

    manualMotionText = "STOP";
    stopCar();
  }
}

// =====================================================
// Manual Targeting System
// =====================================================
void manualTargetingSystem() {

  // ---------------------------------------------------
  // Scanning
  // ---------------------------------------------------
  if (state == SCANNING) {

    if (millis() - lastScanTime >= scanDelay) {

      lastScanTime = millis();

      scanServo.write(scanAngle);

      long distance = readDistance();

      lastDistance = distance;

      updateManualScanLCD(distance);

      if (
        distance >= detectMin &&
        distance <= detectMax
      ) {

        detectionCount++;

        detectedAngleSaved = scanAngle;

        if (detectionCount >= requiredConfirmations) {

          stopCar();

          updateLCD(
            "TARGET ACQUIRED",
            "MANUAL PAUSED"
          );

          int accurateAngle =
            findBestTargetAngle(detectedAngleSaved);

          startAiming(accurateAngle);

          detectionCount = 0;

          return;
        }
      }

      else {

        detectionCount = 0;
      }

      updateScanAngle();
    }
  }

  // ---------------------------------------------------
  // Aiming
  // ---------------------------------------------------
  else if (state == AIMING) {

    aimingMode();
  }

  // ---------------------------------------------------
  // Locked
  // ---------------------------------------------------
  else if (state == LOCKED) {

    lockedMode();
  }

  // ---------------------------------------------------
  // Returning Home
  // ---------------------------------------------------
  else if (state == RETURNING_HOME) {

    returningHomeMode();
  }
}

// =====================================================
// Return To Auto Mode
// =====================================================
void returnToAutoMode() {

  manualMode = false;
  btCommand = 'S';

  stopPressCount = 0;
  manualMotionText = "STOP";

  stopCar();

  detectionCount = 0;
  engagedDistance = 999;

  digitalWrite(laserPin, LOW);
  buzzerOff();

  baseTargetNow = baseHome;
  upTargetNow = upHome;
  extendTargetNow = extendHome;

  state = SCANNING;

  updateLCD(
    "AUTO MODE",
    "MISSION RESUMED"
  );
}

// =====================================================
// LCD
// =====================================================
void updateLCD(
  const char* line1,
  const char* line2
) {

  if (millis() - lastLcdUpdate < lcdInterval) {
    return;
  }

  lastLcdUpdate = millis();

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print(line1);

  lcd.setCursor(0, 1);
  lcd.print(line2);
}

// =====================================================
// Auto Scanning LCD
// =====================================================
void updateScanningLCD(long distance) {

  if (millis() - lastLcdUpdate < lcdInterval) {
    return;
  }

  lastLcdUpdate = millis();

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("AUTO: SCANNING");

  lcd.setCursor(0, 1);
  lcd.print("D:");

  if (distance == 999) {
    lcd.print("---");
  }

  else {
    lcd.print(distance);
  }

  lcd.print("cm A:");
  lcd.print(scanAngle);
}

// =====================================================
// Manual Scanning LCD
// =====================================================
void updateManualScanLCD(long distance) {

  if (millis() - lastLcdUpdate < lcdInterval) {
    return;
  }

  lastLcdUpdate = millis();

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("MANUAL:");
  lcd.print(manualMotionText);

  lcd.setCursor(0, 1);
  lcd.print("SCAN D:");

  if (distance == 999) {
    lcd.print("---");
  }

  else {
    lcd.print(distance);
  }

  lcd.print("cm");
}

// =====================================================
// Engaged Target LCD
// =====================================================
void updateEngagedLCD(long distance) {

  if (millis() - lastLcdUpdate < lcdInterval) {
    return;
  }

  lastLcdUpdate = millis();

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("TARGET ENGAGED");

  lcd.setCursor(0, 1);
  lcd.print("LOCK D:");

  if (distance == 999) {
    lcd.print("---");
  }

  else {
    lcd.print(distance);
  }

  lcd.print("cm");
}

// =====================================================
// Auto Scanning
// =====================================================
void scanMode() {

  buzzerOff();

  // Obstacle Detection
  if (isObstacleDetected()) {

    startAvoidObstacle();

    return;
  }

  moveCarForward();

  if (millis() - lastScanTime >= scanDelay) {

    lastScanTime = millis();

    scanServo.write(scanAngle);

    long distance = readDistance();

    lastDistance = distance;

    updateScanningLCD(distance);

    // Target Detection
    if (
      distance >= detectMin &&
      distance <= detectMax
    ) {

      detectionCount++;

      detectedAngleSaved = scanAngle;

      if (detectionCount >= requiredConfirmations) {

        stopCar();

        updateLCD(
          "TARGET ACQUIRED",
          "VERIFYING LOCK"
        );

        int accurateAngle =
          findBestTargetAngle(detectedAngleSaved);

        startAiming(accurateAngle);

        detectionCount = 0;

        return;
      }
    }

    else {

      detectionCount = 0;
    }

    updateScanAngle();
  }
}

// =====================================================
// Obstacle Avoidance - Start
// =====================================================
void startAvoidObstacle() {

  stopCar();

  digitalWrite(laserPin, LOW);
  buzzerOff();

  bool leftBlocked = leftObstacleDetected();
  bool rightBlocked = rightObstacleDetected();

  if (leftBlocked && !rightBlocked) {

    avoidTurnDirection = 1;
  }

  else if (rightBlocked && !leftBlocked) {

    avoidTurnDirection = -1;
  }

  else {

    avoidTurnDirection = 1;
  }

  avoidStep = 0;
  avoidStartTime = millis();

  updateLCD(
    "SAFETY ALERT",
    "OBSTACLE FOUND"
  );

  state = AVOID_OBSTACLE;
}

// =====================================================
// Obstacle Avoidance Mode
// =====================================================
void avoidObstacleMode() {

  unsigned long now = millis();

  // ---------------------------------------------------
  // Step 0 - Stop
  // ---------------------------------------------------
  if (avoidStep == 0) {

    stopCar();

    updateLCD(
      "AUTO RECOVERY",
      "STOPPING"
    );

    fastBeep();

    if (now - avoidStartTime >= stopBeforeBackTime) {

      avoidStep = 1;
      avoidStartTime = now;
    }
  }

  // ---------------------------------------------------
  // Step 1 - Reverse
  // ---------------------------------------------------
  else if (avoidStep == 1) {

    moveCarBackward();

    updateLCD(
      "AUTO RECOVERY",
      "REVERSING"
    );

    fastBeep();

    if (now - avoidStartTime >= backTime) {

      avoidStep = 2;
      avoidStartTime = now;
    }
  }

  // ---------------------------------------------------
  // Step 2 - Turn
  // ---------------------------------------------------
  else if (avoidStep == 2) {

    if (avoidTurnDirection == 1) {

      turnCarRight();

      updateLCD(
        "AUTO RECOVERY",
        "TURN RIGHT"
      );
    }

    else {

      turnCarLeft();

      updateLCD(
        "AUTO RECOVERY",
        "TURN LEFT"
      );
    }

    slowBeep();

    if (now - avoidStartTime >= turnTime) {

      avoidStep = 3;
      avoidStartTime = now;
    }
  }

  // ---------------------------------------------------
  // Step 3 - Resume Scanning
  // ---------------------------------------------------
  else if (avoidStep == 3) {

    stopCar();

    buzzerOff();

    updateLCD(
      "PATH CLEARED",
      "SCAN RESUMING"
    );

    if (now - avoidStartTime >= stopAfterTurnTime) {

      detectionCount = 0;

      state = SCANNING;
    }
  }
}

// =====================================================
// Target Correction
// =====================================================
int findBestTargetAngle(int centerAngle) {

  int bestAngle = centerAngle;
  long bestDistance = 999;

  for (
    int a = centerAngle - 6;
    a <= centerAngle + 6;
    a += 3
  ) {

    int testAngle =
      constrain(a, scanMin, scanMax);

    scanServo.write(testAngle);

    delay(35);

    long d = readDistance();

    if (
      d >= detectMin &&
      d <= detectMax &&
      d < bestDistance
    ) {

      bestDistance = d;
      bestAngle = testAngle;
    }
  }

  scanServo.write(bestAngle);

  delay(40);

  engagedDistance = bestDistance;

  return bestAngle;
}

// =====================================================
// Aiming
// =====================================================
void startAiming(int detectedAngle) {

  stopCar();

  scanServo.write(detectedAngle);

  int baseTarget =
    map(
      detectedAngle,
      scanMin,
      scanMax,
      baseMin,
      baseMax
    );

  if (detectedAngle < 95) {

    baseTarget += leftOffset;
  }

  else if (detectedAngle > 105) {

    baseTarget += rightOffset;
  }

  else {

    baseTarget += middleOffset;
  }

  baseTarget =
    constrain(
      baseTarget,
      baseMin,
      baseMax
    );

  baseTargetNow = baseTarget;

  upTargetNow = upTarget;
  extendTargetNow = extendTarget;

  digitalWrite(laserPin, LOW);

  updateLCD(
    "TARGETING MODE",
    "ARM ALIGNING"
  );

  state = AIMING;
}

// =====================================================
// Aiming Mode
// =====================================================
void aimingMode() {

  stopCar();

  slowBeep();

  updateLCD(
    "AIMING SYSTEM",
    "LOCKING TARGET"
  );

  if (armAtTarget()) {

    digitalWrite(laserPin, HIGH);

    updateEngagedLCD(engagedDistance);

    state = LOCKED;
  }
}

// =====================================================
// Locked Mode
// =====================================================
void lockedMode() {

  stopCar();

  fastBeep();

  static unsigned long lastCheckTime = 0;

  if (millis() - lastCheckTime >= 100) {

    lastCheckTime = millis();

    long distance = readDistance();

    lastDistance = distance;

    updateEngagedLCD(distance);

    if (
      !(distance >= detectMin &&
        distance <= detectMax)
    ) {

      digitalWrite(laserPin, LOW);

      buzzerOff();

      updateLCD(
        "TARGET LOST",
        "RETURNING HOME"
      );

      baseTargetNow = baseHome;
      upTargetNow = upHome;
      extendTargetNow = extendHome;

      state = RETURNING_HOME;
    }
  }
}

// =====================================================
// Returning Home
// =====================================================
void returningHomeMode() {

  stopCar();

  buzzerOff();

  updateLCD(
    "SYSTEM RESET",
    "ARM TO HOME"
  );

  if (millis() - lastScanTime >= scanDelay) {

    lastScanTime = millis();

    scanServo.write(scanAngle);

    updateScanAngle();
  }

  if (armAtTarget()) {

    detectionCount = 0;
    engagedDistance = 999;

    state = SCANNING;
  }
}

// =====================================================
// Smooth Arm Movement
// =====================================================
void updateArmSmooth() {

  if (millis() - lastArmMoveTime < armInterval) {
    return;
  }

  lastArmMoveTime = millis();

  baseCurrent =
    moveSmoothValue(
      baseCurrent,
      baseTargetNow
    );

  upCurrent =
    moveSmoothValue(
      upCurrent,
      upTargetNow
    );

  extendCurrent =
    moveSmoothValue(
      extendCurrent,
      extendTargetNow
    );

  armBaseServo.write(round(baseCurrent));
  armUpServo.write(round(upCurrent));
  armExtendServo.write(round(extendCurrent));
}

// =====================================================
// Smooth Value Calculation
// =====================================================
float moveSmoothValue(
  float current,
  float target
) {

  if (abs(current - target) <= armStep) {
    return target;
  }

  if (current < target) {
    return current + armStep;
  }

  else {
    return current - armStep;
  }
}

// =====================================================
// Check Arm Target
// =====================================================
bool armAtTarget() {

  return
    abs(baseCurrent - baseTargetNow) < 1 &&
    abs(upCurrent - upTargetNow) < 1 &&
    abs(extendCurrent - extendTargetNow) < 1;
}

// =====================================================
// Scanner
// =====================================================
void updateScanAngle() {

  scanAngle += scanDirection * scanStep;

  if (scanAngle >= scanMax) {

    scanAngle = scanMax;
    scanDirection = -1;
  }

  if (scanAngle <= scanMin) {

    scanAngle = scanMin;
    scanDirection = 1;
  }
}

// =====================================================
// Ultrasonic Sensor
// =====================================================
long readDistance() {

  digitalWrite(trigPin, LOW);

  delayMicroseconds(2);

  digitalWrite(trigPin, HIGH);

  delayMicroseconds(10);

  digitalWrite(trigPin, LOW);

  long duration =
    pulseIn(
      echoPin,
      HIGH,
      30000
    );

  if (duration == 0) {
    return 999;
  }

  long distance =
    duration * 0.034 / 2;

  if (
    distance < 3 ||
    distance > 300
  ) {

    return 999;
  }

  return distance;
}

// =====================================================
// IR Sensors
// =====================================================
bool leftObstacleDetected() {

  return digitalRead(irLeftPin)
         == IR_OBSTACLE_STATE;
}

bool rightObstacleDetected() {

  return digitalRead(irRightPin)
         == IR_OBSTACLE_STATE;
}

bool isObstacleDetected() {

  return
    leftObstacleDetected() ||
    rightObstacleDetected();
}

// =====================================================
// Car Movement
// =====================================================
void moveCarForward() {

  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);

  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

void moveCarBackward() {

  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);

  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void turnCarLeft() {

  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);

  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

void turnCarRight() {

  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);

  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void stopCar() {

  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);

  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
}

// =====================================================
// Buzzer
// =====================================================
void startupBeep() {

  digitalWrite(buzzerPin, HIGH);

  delay(150);

  digitalWrite(buzzerPin, LOW);
}

void buzzerOff() {

  digitalWrite(buzzerPin, LOW);
}

// =====================================================
// Slow Beep
// =====================================================
void slowBeep() {

  static unsigned long lastBeepTime = 0;
  static bool buzzerState = false;

  if (millis() - lastBeepTime >= 400) {

    lastBeepTime = millis();

    buzzerState = !buzzerState;

    digitalWrite(
      buzzerPin,
      buzzerState
    );
  }
}

// =====================================================
// Fast Beep
// =====================================================
void fastBeep() {

  static unsigned long lastBeepTime = 0;
  static bool buzzerState = false;

  if (millis() - lastBeepTime >= 120) {

    lastBeepTime = millis();

    buzzerState = !buzzerState;

    digitalWrite(
      buzzerPin,
      buzzerState
    );
  }
}