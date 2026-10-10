#include <Keypad.h>
#include <Servo.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

#define RED_LED 2
#define GREEN_LED 13
#define SERVO_PIN 11
#define BUZZER_PIN 12
#define TRIG_PIN A1
#define ECHO_PIN A2

LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo lockServo;

// ============================
// KEYPAD
// ============================

const byte ROWS = 4;
const byte COLS = 4;

char keys[ROWS][COLS] = {
  {'1', '2', '3', 'A'},
  {'4', '5', '6', 'B'},
  {'7', '8', '9', 'C'},
  {'*', '0', '#', 'D'}
};

byte rowPins[ROWS] = {3, 4, 5, 6};
byte colPins[COLS] = {7, 8, 9, 10};

Keypad keypad = Keypad(
  makeKeymap(keys),
  rowPins,
  colPins,
  ROWS,
  COLS
);

// ============================
// PASSWORD
// ============================

String correctPIN = "0000";
String enteredPIN = "";
String newPIN = "";

bool changingPIN = false;

// ============================
// SYSTEM STATUS
// ============================

bool systemOn = false;
bool safeUnlocked = false;
bool leavingTimer = false;
bool personAwayWarning = false;

unsigned long leaveStartTime = 0;
unsigned long lastWarningBeep = 0;
unsigned long lastADCRead = 0;

const unsigned long OFF_DELAY = 5000;
const unsigned long WARNING_INTERVAL = 1000;
const unsigned long ADC_INTERVAL = 500;

const int DETECT_DISTANCE = 35;

int absentReadings = 0;
const int REQUIRED_ABSENT_READINGS = 3;

// ============================
// SETUP
// ============================

void setup() {

  Serial.begin(9600);

  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  lockServo.attach(SERVO_PIN);
  lockServo.write(0);

  digitalWrite(RED_LED, HIGH);
  digitalWrite(GREEN_LED, LOW);

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("SMART SAFE BOX");

  lcd.setCursor(0, 1);
  lcd.print("SYSTEM READY");

  Serial.println("SMART SAFE BOX READY");
  Serial.println("Password: 0000");
  Serial.println("Distance and ADC-scale monitor");

  delay(2000);

  lcd.clear();
  lcd.noBacklight();
}

// ============================
// MAIN LOOP
// ============================

void loop() {

  long distance = getDistance();

  // SERIAL MONITOR
  if (millis() - lastADCRead >= ADC_INTERVAL) {

    lastADCRead = millis();

    int adcValue = getADCValue(distance);

    Serial.print("Distance: ");

    if (distance < 0) {
      Serial.print("No echo");
    } else {
      Serial.print(distance);
      Serial.print(" cm");
    }

    Serial.print(" | ADC-scale value: ");
    Serial.println(adcValue);
  }

  // PERSON DETECTED WITHIN 35 CM
  if (distance > 0 && distance <= DETECT_DISTANCE) {

    absentReadings = 0;

    if (personAwayWarning) {

      personAwayWarning = false;
      noTone(BUZZER_PIN);

      if (safeUnlocked && !changingPIN) {

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("DOOR IS OPEN");

        lcd.setCursor(0, 1);
        lcd.print("# = LOCK");
      }
    }

    if (leavingTimer) {

      leavingTimer = false;

      if (systemOn && !safeUnlocked) {
        showPINScreen();
      }
    }

    // TURN SYSTEM ON
    if (!systemOn && !safeUnlocked) {

      systemOn = true;

      lcd.backlight();
      lcd.clear();

      lcd.setCursor(0, 0);
      lcd.print("PERSON DETECTED");

      lcd.setCursor(0, 1);
      lcd.print("ENTER PIN");

      tone(BUZZER_PIN, 1000, 100);
    }

    // ENTER PIN
    if (systemOn && !safeUnlocked && !changingPIN) {
      enterPIN();
    }

    // HANDLE UNLOCKED MODE
    if (safeUnlocked) {
      handleUnlockedMode();
    }
  }

  // PERSON NOT DETECTED
  else {

    if (distance > DETECT_DISTANCE || distance == -1) {
      if (absentReadings < REQUIRED_ABSENT_READINGS) {
        absentReadings++;
      }
    }

    // PERSON LEFT WHILE SAFE UNLOCKED
    if (safeUnlocked &&
        absentReadings >= REQUIRED_ABSENT_READINGS) {

      if (!personAwayWarning) {

        personAwayWarning = true;

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("PERSON LEFT");

        lcd.setCursor(0, 1);
        lcd.print("PRESS # TO LOCK");

        lastWarningBeep = millis();

        tone(BUZZER_PIN, 800, 300);
      }

      if (millis() - lastWarningBeep >= WARNING_INTERVAL) {

        lastWarningBeep = millis();
        tone(BUZZER_PIN, 800, 300);
      }
    }

    // START LEAVING TIMER WHEN LOCKED
    if (absentReadings >= REQUIRED_ABSENT_READINGS) {

      if (systemOn && !safeUnlocked && !leavingTimer) {

        leavingTimer = true;
        leaveStartTime = millis();

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("PERSON LEFT");

        lcd.setCursor(0, 1);
        lcd.print("WAIT 5 SECONDS");
      }
    }

    // SYSTEM OFF AFTER 5 SECONDS
    if (leavingTimer &&
        millis() - leaveStartTime >= OFF_DELAY) {

      systemOn = false;
      leavingTimer = false;
      enteredPIN = "";

      lcd.clear();
      lcd.noBacklight();

      noTone(BUZZER_PIN);

      absentReadings = 0;
    }
  }
}

// ============================
// ULTRASONIC SENSOR
// ============================

long getDistance() {

  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);

  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);

  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000);

  if (duration == 0) {
    return -1;
  }

  return duration * 0.034 / 2;
}

// ============================
// DISTANCE TO ADC-SCALE VALUE
// ============================

int getADCValue(long distance) {

  if (distance <= 0 || distance > 100) {
    return 0;
  }

  return map(distance, 100, 1, 0, 1023);
}

// ============================
// PIN SCREEN
// ============================

void showPINScreen() {

  lcd.backlight();
  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("PERSON DETECTED");

  lcd.setCursor(0, 1);

  if (enteredPIN.length() == 0) {
    lcd.print("ENTER PIN");
  } else {
    for (int i = 0; i < enteredPIN.length(); i++) {
      lcd.print("*");
    }
  }
}

// ============================
// ENTER PIN
// ============================

void enterPIN() {

  char key = keypad.getKey();

  if (!key) return;

  if (key >= '0' && key <= '9') {

    if (enteredPIN.length() < 4) {

      enteredPIN += key;

      lcd.setCursor(0, 1);
      lcd.print("                ");
      lcd.setCursor(0, 1);

      for (int i = 0; i < enteredPIN.length(); i++) {
        lcd.print("*");
      }

      tone(BUZZER_PIN, 1000, 50);
    }
  }

  else if (key == '*') {

    enteredPIN = "";

    lcd.setCursor(0, 1);
    lcd.print("                ");
    lcd.setCursor(0, 1);
    lcd.print("ENTER PIN");

    tone(BUZZER_PIN, 700, 50);
  }

  else if (key == '#') {

    if (enteredPIN == correctPIN) {
      accessGranted();
    } else {
      accessDenied();
    }

    enteredPIN = "";
  }
}

// ============================
// UNLOCKED MODE
// ============================

void handleUnlockedMode() {

  char key = keypad.getKey();

  if (!key) return;

  // # = CLOSE AND LOCK
  if (key == '#') {
    lockSafe();
    return;
  }

  // B = CHANGE PASSWORD
  if (key == 'B' && !changingPIN) {

    changingPIN = true;
    newPIN = "";
    personAwayWarning = false;

    noTone(BUZZER_PIN);

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("CHANGE PASSWORD");

    lcd.setCursor(0, 1);
    lcd.print("NEW PIN:");

    tone(BUZZER_PIN, 1200, 100);
    return;
  }

  if (changingPIN) {

    if (key >= '0' && key <= '9') {

      if (newPIN.length() < 4) {

        newPIN += key;

        lcd.setCursor(0, 1);
        lcd.print("                ");
        lcd.setCursor(0, 1);
        lcd.print("NEW:");

        for (int i = 0; i < newPIN.length(); i++) {
          lcd.print("*");
        }

        tone(BUZZER_PIN, 1000, 50);
      }
    }

    else if (key == '*') {

      newPIN = "";

      lcd.setCursor(0, 1);
      lcd.print("                ");
      lcd.setCursor(0, 1);
      lcd.print("NEW PIN:");

      tone(BUZZER_PIN, 700, 50);
    }

    else if (key == '#') {

      if (newPIN.length() == 4) {

        correctPIN = newPIN;

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("PASSWORD CHANGED");

        lcd.setCursor(0, 1);
        lcd.print("SUCCESSFULLY");

        tone(BUZZER_PIN, 1200, 150);
        delay(200);
        tone(BUZZER_PIN, 1500, 150);
        delay(1500);

        changingPIN = false;
        newPIN = "";

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("DOOR IS OPEN");

        lcd.setCursor(0, 1);
        lcd.print("# = LOCK");
      }

      else {

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("INVALID PASSWORD");

        lcd.setCursor(0, 1);
        lcd.print("4 DIGITS REQUIRED");

        tone(BUZZER_PIN, 500, 300);
        delay(1200);

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("CHANGE PASSWORD");

        lcd.setCursor(0, 1);
        lcd.print("NEW PIN:");
      }
    }
  }
}

// ============================
// ACCESS GRANTED
// ============================

void accessGranted() {

  safeUnlocked = true;
  systemOn = true;

  leavingTimer = false;
  personAwayWarning = false;

  lcd.backlight();
  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("ACCESS GRANTED");

  lcd.setCursor(0, 1);
  lcd.print("DOOR UNLOCKED");

  tone(BUZZER_PIN, 1000, 150);
  delay(200);
  tone(BUZZER_PIN, 1500, 150);

  lockServo.write(90);

  digitalWrite(RED_LED, LOW);
  digitalWrite(GREEN_LED, HIGH);

  delay(1500);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("DOOR IS OPEN");

  lcd.setCursor(0, 1);
  lcd.print("# = LOCK");
}

// ============================
// ACCESS DENIED
// ============================

void accessDenied() {

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("ACCESS DENIED");

  lcd.setCursor(0, 1);
  lcd.print("WRONG PIN");

  digitalWrite(RED_LED, HIGH);
  digitalWrite(GREEN_LED, LOW);

  for (int i = 0; i < 3; i++) {
    tone(BUZZER_PIN, 500, 300);
    delay(400);
  }

  noTone(BUZZER_PIN);
  delay(500);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("TRY AGAIN");

  lcd.setCursor(0, 1);
  lcd.print("ENTER PIN");
}

// ============================
// LOCK SAFE
// ============================

void lockSafe() {

  personAwayWarning = false;
  noTone(BUZZER_PIN);

  lcd.backlight();
  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("DOOR CONFIRMED");

  lcd.setCursor(0, 1);
  lcd.print("LOCKING...");

  delay(500);

  lockServo.write(0);

  digitalWrite(GREEN_LED, LOW);
  digitalWrite(RED_LED, HIGH);

  tone(BUZZER_PIN, 700, 200);
  delay(500);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("SAFE LOCKED");

  lcd.setCursor(0, 1);
  lcd.print("SYSTEM READY");

  safeUnlocked = false;
  systemOn = false;
  leavingTimer = false;
  changingPIN = false;

  enteredPIN = "";
  newPIN = "";

  delay(1500);

  lcd.clear();
  lcd.noBacklight();

  absentReadings = 0;
}
