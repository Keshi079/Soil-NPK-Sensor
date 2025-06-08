# Soil-NPK-Sensor
#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <SoftwareSerial.h>

// Replace with your Wi-Fi credentials
const char* ssid = "HOTSPOT'S NAME";
const char* password = "password";

// ThingSpeak configuration
const char* server = "Your API link";
const char* apiKey = "Your API key";

// RS485 NPK sensor
#define RE 16
#define DE 5
SoftwareSerial mod(14, 12); // RX, TX

const byte nitro[] = {0x01, 0x03, 0x00, 0x1e, 0x00, 0x01, 0xe4, 0x0c};
const byte phos[]  = {0x01, 0x03, 0x00, 0x1f, 0x00, 0x01, 0xb5, 0xcc};
const byte pota[]  = {0x01, 0x03, 0x00, 0x20, 0x00, 0x01, 0x85, 0xc0};

byte values[11];

void setup() {
  Serial.begin(115200);
  mod.begin(9600);

  pinMode(RE, OUTPUT);
  pinMode(DE, OUTPUT);

  // Connect to WiFi
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("\nConnected to WiFi");
}

void loop() {
  byte val1 = nitrogen();
  delay(250);
  byte val2 = phosphorous();
  delay(250);
  byte val3 = potassium();
  delay(250);

  Serial.print("Nitrogen: "); Serial.println(val1); Serial.println(" mg/Kg");

  Serial.print("Phosphorous: "); Serial.println(val2); Serial.println(" mg/Kg");

  Serial.print("Potassium: "); Serial.println(val3); Serial.println(" mg/Kg");


  sendToThingSpeak(val1, val2, val3);
  delay(20000);  // ThingSpeak recommends at least 15s between updates
}

void sendToThingSpeak(byte N, byte P, byte K) {
  if (WiFi.status() == WL_CONNECTED) {
    WiFiClient client;
    HTTPClient http;

    String url = String(server) + "?api_key=" + apiKey +
                 "&field1=" + String(N) +
                 "&field2=" + String(P) +
                 "&field3=" + String(K);

    Serial.println("Sending data to ThingSpeak:");
    Serial.println(url);

    http.begin(client, url); // Correct way for ESP32
    int httpResponseCode = http.GET();

    Serial.print("HTTP Response Code: ");
    Serial.println(httpResponseCode);
    http.end();
  } else {
    Serial.println("WiFi not connected");
  }
}

byte nitrogen() {
  return readSensor(nitro);
}

byte phosphorous() {
  return readSensor(phos);
}

byte potassium() {
  return readSensor(pota);
}

byte readSensor(const byte command[]) {
  digitalWrite(DE, HIGH);
  digitalWrite(RE, HIGH);
  delay(10);

  mod.write(command, 8);
  digitalWrite(DE, LOW);
  digitalWrite(RE, LOW);
  delay(20);

  int i = 0;
  while (mod.available() && i < 7) {
    values[i++] = mod.read();
  }

  if (i >= 5) {
    return values[4];  // This is the NPK value
  } else {
    Serial.println("Sensor read error!");
    return 0;
  }
}
