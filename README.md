# Train-track-crack-detection-system-using-Arduino-Uno-with-thingspeak-
Built a real time alert system when any crack has detected on a railway track. The data is shared and stored in a Cloud platform called thingspeak. Also integrated a message system through Telegram. 
#include <WiFi.h>
#include <HTTPClient.h>
#include <WiFiClientSecure.h>

// ─── AP CONFIG ─────────────────────────────
const char* ap_ssid = "ESP32_AP";
const char* ap_password = "12345678";

// ─── WIFI (INTERNET) ──────────────────────
const char* ssid = "Bharath";
const char* password = "Bharath@1234";

// ─── THINGSPEAK ───────────────────────────
String apiKey = "ODY3DRKL7CDEWQE8";

// ─── TELEGRAM ─────────────────────────────
const char* BOT_TOKEN = "8551639857:AAFW9Zd5S33lHKHRx2Ii_ePXEY19CL5AaZk";
const char* CHAT_ID  = "6899426616";

WiFiClientSecure telegramClient;

// ─── COMMANDS ─────────────────────────────
#define CMD_STOP   "STOP"
#define CMD_GO     "GO"

// ─── PIN ──────────────────────────────────
const int crackSensorPin = 4;

// ─── OBJECTS ──────────────────────────────
WiFiServer server(80);
WiFiClient esp01;

// ─── TIMER ────────────────────────────────
unsigned long lastThingSpeakTime = 0;
const unsigned long thingSpeakInterval = 15000;

// ─── STORE VALUES ─────────────────────────
int lastDistance = 0;
bool lastCrack = 0;
bool alertSent = false;  // prevent spam

// ──────────────────────────────────────────
void setup() {
  Serial.begin(115200);
  pinMode(crackSensorPin, INPUT_PULLUP);

  WiFi.mode(WIFI_AP_STA);

  // Start AP
  WiFi.softAP(ap_ssid, ap_password);
  Serial.println("[ESP32] AP Started");
  Serial.println(WiFi.softAPIP());

  // Connect to Internet
  WiFi.begin(ssid, password);
  Serial.print("Connecting WiFi");

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("\n[ESP32] Connected to Internet");
  Serial.println(WiFi.localIP());

  server.begin();
}

// ──────────────────────────────────────────
void loop() {

  // 1. ESP01 CONNECTION
  if (!esp01 || !esp01.connected()) {
    WiFiClient client = server.available();
    if (client) {
      esp01 = client;
      Serial.println("[ESP32] ESP01 Connected");
    }
  }

  // 2. RECEIVE DATA
  if (esp01 && esp01.available()) {
    String msg = esp01.readStringUntil('\n');
    msg.trim();

    Serial.print("[RX] ");
    Serial.println(msg);

    // FORMAT: $xx$
    if (msg.startsWith("$") && msg.endsWith("$")) {

      int distance = msg.substring(1, msg.length() - 1).toInt();
      lastDistance = distance;

      bool crack = digitalRead(crackSensorPin);
      lastCrack = crack;

      Serial.print("Distance: ");
      Serial.println(distance);

      Serial.print("Crack: ");
      Serial.println(crack);

      // ─── DECISION ───
      if (distance > 2 || crack == HIGH) {

        sendToESP01(CMD_STOP);

        // 🚨 TELEGRAM ALERT
        if (!alertSent) {
          String message = "🚨 Train crack detected!\n";
          message += "Location: Track Zone A\n";
          message += "Distance: " + String(distance);

          sendTelegram(message);
          alertSent = true;
        }

      } else {
        sendToESP01(CMD_GO);
        alertSent = false;
      }
    }
  }

  // 3. THINGSPEAK UPDATE
  if (millis() - lastThingSpeakTime >= thingSpeakInterval) {
    lastThingSpeakTime = millis();
    sendToThingSpeak(lastDistance, lastCrack);
  }
}

// ──────────────────────────────────────────
void sendToESP01(String data) {
  if (esp01 && esp01.connected()) {
    esp01.println(data);

    Serial.print("[TX] ");
    Serial.println(data);
  }
}

// ──────────────────────────────────────────
void sendToThingSpeak(int distance, bool crack) {

  if (WiFi.status() == WL_CONNECTED) {

    HTTPClient http;

    int crackValue = crack ? 1 : 0;

    String url = "http://api.thingspeak.com/update?api_key=" + apiKey +
                 "&field1=" + String(distance) +
                 "&field2=" + String(crackValue);

    http.begin(url);
    int httpResponseCode = http.GET();

    Serial.print("[ThingSpeak] ");
    Serial.println(httpResponseCode);

    http.end();
  }
}

// ──────────────────────────────────────────
void sendTelegram(String msg) {

  if (WiFi.status() == WL_CONNECTED) {

    telegramClient.setInsecure();

    HTTPClient http;

    String url = "https://api.telegram.org/bot" + String(BOT_TOKEN) + "/sendMessage";

    http.begin(telegramClient, url);
    http.addHeader("Content-Type", "application/json");

    String payload = "{\"chat_id\":\"" + String(CHAT_ID) +
                     "\",\"text\":\"" + msg + "\"}";

    int httpCode = http.POST(payload);

    Serial.print("[Telegram] ");
    Serial.println(httpCode);

    http.end();
  }
}
