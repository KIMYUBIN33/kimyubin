/*
 * Smart Chemical Box - Complete Integration
 * Hardware: Arduino Uno, RC522, I2C LCD, DHT11, Water Sensor, Servo, DS1302, Bluetooth
 * 
 * SETUP INSTRUCTIONS:
 * 1. Upload this code first to find your RFID card UID
 * 2. Open Serial Monitor (9600 baud)
 * 3. Tag your RFID card - copy the UID shown
 * 4. Replace "YOUR_CARD_UID_HERE" below with your actual UID
 * 5. Uncomment RTC setup lines (69-70) for first upload only
 * 6. Re-upload and enjoy!
 */

#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <SPI.h>
#include <MFRC522.h>
#include <DHT.h>
#include <Servo.h>
#include <ThreeWire.h>  
#include <RtcDS1302.h>

// === PIN DEFINITIONS ===
#define DHTPIN 2
#define DHTTYPE DHT11
#define SERVO_PIN 6
#define WATER_PIN A0
#define SS_PIN 10
#define RST_PIN 9

// RTC Pins: DAT=4, CLK=5, RST=3
ThreeWire myWire(4, 5, 3); 
RtcDS1302<ThreeWire> Rtc(myWire);

// === OBJECT INITIALIZATION ===
LiquidCrystal_I2C lcd(0x27, 16, 2); 
MFRC522 mfrc522(SS_PIN, RST_PIN);
DHT dht(DHTPIN, DHTTYPE);
Servo myservo;

// === SETTINGS ===
String masterTag = "0F B8 E2 29";  // ⚠️ CHANGE THIS! Example: "D3 F2 A1 88"
bool isLocked = true;
unsigned long lastSendTime = 0;

void setup() {
  Serial.begin(9600);
  
  // Initialize Hardware
  SPI.begin();
  mfrc522.PCD_Init();
  dht.begin();
  myservo.attach(SERVO_PIN);
  
  // Initialize RTC
  Rtc.Begin();
  
  // ⚠️ UNCOMMENT NEXT 2 LINES FOR FIRST UPLOAD ONLY, THEN COMMENT AGAIN!
  // RtcDateTime compiled = RtcDateTime(__DATE__, __TIME__);
  // Rtc.SetDateTime(compiled);

  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Chemical");
  lcd.setCursor(0, 1);
  lcd.print("Box Starting...");
  
  lockBox();
  delay(2000);
  lcd.clear();
  
  Serial.println("=== Smart Chemical Box Ready ===");
  Serial.println("Waiting for RFID card...");
}

void loop() {
  checkRFID();
  
  if (millis() - lastSendTime > 2000) {
    sendDataToPhone();
    updateLCD();
    lastSendTime = millis();
  }
}

// === RFID AUTHENTICATION ===
void checkRFID() {
  if (!mfrc522.PICC_IsNewCardPresent() || !mfrc522.PICC_ReadCardSerial()) {
    return;
  }
  
  String content = "";
  for (byte i = 0; i < mfrc522.uid.size; i++) {
    content.concat(String(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " "));
    content.concat(String(mfrc522.uid.uidByte[i], HEX));
  }
  content.toUpperCase();
  
  // Debug: Show card UID
  Serial.print("Card detected: ");
  Serial.println(content.substring(1));
  
  if (content.substring(1) == masterTag) {
    Serial.println("✓ Access GRANTED");
    unlockBox();
  } else {
    Serial.println("✗ Access DENIED");
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Access Denied!");
    lcd.setCursor(0, 1);
    lcd.print("Unknown Card");
    delay(2000);
    lcd.clear();
  }
  
  mfrc522.PICC_HaltA();
}

// === SERVO CONTROL ===
void unlockBox() {
  isLocked = false;
  myservo.attach(SERVO_PIN);
  myservo.write(80);  // Open position
  delay(600);
  myservo.detach();   // Stop signal to prevent jitter
  
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Access Granted!");
  lcd.setCursor(0, 1);
  lcd.print("Box OPEN");
  
  delay(5000);  // Stay open for 5 seconds
  lockBox();
}

void lockBox() {
  isLocked = true;
  myservo.attach(SERVO_PIN);
  myservo.write(10);  // Lock position
  delay(600);
  myservo.detach();   // Stop signal
  
  Serial.println("Box locked");
}

// === BLUETOOTH DATA TRANSMISSION ===
void sendDataToPhone() {
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  int waterVal = analogRead(WATER_PIN);
  
  // Check for sensor errors
  if (isnan(t) || isnan(h)) {
    Serial.println("ERROR|ERROR|ERROR|ERROR");
    return;
  }
  
  String waterStatus = (waterVal > 300) ? "LEAK" : "DRY";
  String lockStatus = (isLocked) ? "LOCKED" : "OPEN";
  
  // Format: TEMP|HUMIDITY|WATER|LOCK
  Serial.print(t);
  Serial.print("|");
  Serial.print(h);
  Serial.print("|");
  Serial.print(waterStatus);
  Serial.print("|");
  Serial.println(lockStatus);
}

// === LCD DISPLAY UPDATE ===
void updateLCD() {
  RtcDateTime now = Rtc.GetDateTime();
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  int waterVal = analogRead(WATER_PIN);
  
  lcd.clear();
  
  // Line 1: Temperature and Time
  lcd.setCursor(0, 0);
  if (!isnan(t)) {
    lcd.print("T:");
    lcd.print((int)t);
    lcd.print("C ");
  } else {
    lcd.print("T:ERR ");
  }
  
  // Display time HH:MM
  if (now.Hour() < 10) lcd.print("0");
  lcd.print(now.Hour());
  lcd.print(":");
  if (now.Minute() < 10) lcd.print("0");
  lcd.print(now.Minute());
  
  // Line 2: Status
  lcd.setCursor(0, 1);
  if (waterVal > 300) {
    lcd.print("!! LEAK ALERT !!");
  } else if (!isLocked) {
    lcd.print("Status: OPEN    ");
  } else {
    lcd.print("Status: SECURE  ");
  }
}
