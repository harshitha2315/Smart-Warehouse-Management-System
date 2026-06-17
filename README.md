# Smart-Warehouse-Management-System
#include <WiFi.h>
#include <HTTPClient.h>
#include <SPI.h>
#include <MFRC522.h>
#include "HX711.h"

// -------- WIFI --------
const char* ssid = "C2-ENGG-MFO-002 7914";
const char* password = "12345678aaaa";

// -------- FIREBASE --------
String baseURL = "https://warehouseprojj-default-rtdb.firebaseio.com/warehouse/rice/";

// -------- RFID --------
#define SS_PIN 5
#define RST_PIN 22
MFRC522 rfid(SS_PIN, RST_PIN);

// -------- LOAD CELL --------
#define DT 4
#define SCK 2
HX711 scale;

// -------- BUZZER --------
#define BUZZER 15

// -------- VARIABLES --------
float lastWeight = 0;
unsigned long lastScanTime = 0;
int stock = 50;  // initial stock

// -------- SETUP --------
void setup() {
  Serial.begin(115200);

  // WiFi
  WiFi.begin(ssid, password);
  Serial.print("Connecting");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected");

  // RFID
  SPI.begin();
  rfid.PCD_Init();
  Serial.println("RFID Ready");

  // Load Cell
  scale.begin(DT, SCK);
  scale.set_scale(2280.f); // adjust later
  scale.tare();

  // Buzzer
  pinMode(BUZZER, OUTPUT);
}

// -------- LOOP --------
void loop() {

  // -------- RFID SCAN --------
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {

    String uid = "";
    for (byte i = 0; i < rfid.uid.size; i++) {
      uid += String(rfid.uid.uidByte[i], HEX);
    }
    uid.toUpperCase();

    Serial.print("UID: ");
    Serial.println(uid);

    // -------- MAP UID TO PRODUCT --------
    String product = "";

    if (uid == "52E1171") {
      product = "rice";
    }
    else if (uid == "F71A67C") {
      product = "sugar";
    }
    else if (uid == "32622722") {
      product = "oil";
    }

    // -------- SEND PRODUCT --------
    if (product != "") {
      sendToFirebase("lastScanned.json", "\"" + product + "\"");

      // reduce stock
      stock = stock - 1;
      sendToFirebase("stock.json", String(stock));

      // update scan time
      lastScanTime = millis();

      Serial.println("Product Scanned: " + product);
    }

    delay(1500);
  }

  // -------- READ WEIGHT --------
  float weight = scale.get_units();
  Serial.print("Weight: ");
  Serial.println(weight);

  sendToFirebase("weight.json", String(weight));

  // -------- THEFT DETECTION --------
  if (weight < lastWeight - 50) {   // sudden drop

    if (millis() - lastScanTime > 5000) {

      Serial.println("⚠ Unscanned Product!");

      sendToFirebase("alert.json", "\"unscanned product\"");

      digitalWrite(BUZZER, HIGH);
      delay(3000);
      digitalWrite(BUZZER, LOW);
    }
  }

  lastWeight = weight;

  // -------- LOW STOCK --------
  if (weight < 500) {
    sendToFirebase("request.json", "\"pending\"");
  }

  delay(2000);
}

// -------- FIREBASE FUNCTION --------
void sendToFirebase(String path, String value) {
  HTTPClient http;

  http.begin(baseURL + path);
  http.addHeader("Content-Type", "application/json");

  int code = http.PUT(value);

  Serial.print("HTTP: ");
  Serial.println(code);

  http.end();
}
