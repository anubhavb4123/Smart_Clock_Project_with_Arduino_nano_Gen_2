# Smart_Clock_Project_with_Arduino_nano_Gen_2
#include <WiFi.h>
#include <UniversalTelegramBot.h>
#include <WiFiClientSecure.h>
#include <map>

// WiFi credentials
const char* ssid = "BAJPAI_2.4Ghz";
const char* password = "44444422";

// Telegram Bot Token and your chat ID
#define BOT_TOKEN "8024400126:AAEumRTzC-gFiB0nsxsjeIA5Wm_Qx0GvBl4"
#define ADMIN_CHAT_ID "1839775992"    // Replace with your own Telegram user ID
#define LOGIN_PASSWORD "4123"     // Password for guests

WiFiClientSecure secured_client;
UniversalTelegramBot bot(BOT_TOKEN, secured_client);

// Serial2 pin settings
#define RXD2 16
#define TXD2 17

// LED Pin
#define LED_PIN 2

// Sensor values
String serialData = "";
float latestTemp = 0, latestHumid = 0, latestVolt = 0;
int LastHour = 0;
int lowBatteryWarning = 0;
int lowBatteryPercentage = 0;
unsigned long lastCheckTime = 0;
const unsigned long checkInterval = 1000; // 1s

std::map<String, bool> guestLoggedIn;

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RXD2, TXD2);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  // WiFi setup
  WiFi.begin(ssid, password);
  secured_client.setInsecure();
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\n✅ Connected to WiFi");

  // Send startup message to admin only
  bot.sendChatAction(ADMIN_CHAT_ID, "typing");
  bot.sendMessage(ADMIN_CHAT_ID, "🚀 ESP32 is now online!", "");
}

void loop() {
  // Read serial data
  while (Serial2.available()) {
    char c = Serial2.read();
    if (c == '\n') {
      parseSensorData(serialData);
      serialData = "";
    } else {
      serialData += c;
    }
  }

  // Check Telegram updates
  if (millis() - lastCheckTime > checkInterval) {
    int newMsgCount = bot.getUpdates(bot.last_message_received + 1);
    while (newMsgCount) {
      handleMessages(newMsgCount);
      newMsgCount = bot.getUpdates(bot.last_message_received + 1);
    }
    lastCheckTime = millis();
  }

  // Send Low Battery Notification
  if (lowBatteryWarning == 1) {
    broadcastToLoggedInUsers("⚠️ Low Battery Warning! Current Percentage: " + String(lowBatteryPercentage));
    lowBatteryWarning = 0;
  }

  // Send Hour Notification
  if (LastHour != 0) {
    broadcastToLoggedInUsers("⏰ Hour: " + String(LastHour) + " Started");
    LastHour = 0;
  }
}

// Handle incoming messages
void handleMessages(int numNewMessages) {
  for (int i = 0; i < numNewMessages; i++) {
    String chat_id = bot.messages[i].chat_id;
    String text = bot.messages[i].text;

    bool isAdmin = (chat_id == ADMIN_CHAT_ID);
    bool isGuestLoggedIn = guestLoggedIn[chat_id];

    // Allow login
    if (text.startsWith("/login ")) {
      String pwd = text.substring(7);  // Extract password
      if (pwd == LOGIN_PASSWORD) {
        guestLoggedIn[chat_id] = true;
        bot.sendMessage(chat_id, "🔓 *Login successful!* You now have access.", "Markdown");
      } else {
        bot.sendMessage(chat_id, "❌ *Incorrect password.* Try again.", "Markdown");
      }
      continue;
    }

    // Logout
    if (text == "/logout") {
      guestLoggedIn[chat_id] = false;
      bot.sendMessage(chat_id, "🔒 You have been logged out.", "Markdown");
      continue;
    }

    // Admin access
    if (isAdmin) {
      processCommand(chat_id, text);
      continue;
    }

    // Guest access
    if (isGuestLoggedIn) {
      processCommand(chat_id, text);
    } else {
      bot.sendMessage(chat_id,
        "🚫 *Access Denied!*\nPlease login first using:\n`/login your_password`",
        "Markdown");
    }
  }
}

// Execute commands
void processCommand(String chat_id, String text) {
  if (text == "/status") {
    bot.sendChatAction(chat_id, "typing");
    String msg = "📊 *Sensor Readings:*\n";
    msg += "🌡️ Temp: " + String(latestTemp) + " °C\n";
    msg += "💧 Humidity: " + String(latestHumid) + " %\n";
    msg += "🔋 Voltage: " + String(latestVolt) + " V\n";
    msg += "🪫 Battery: " + String(lowBatteryPercentage) + " %\n";
    msg += "⏰ Last Hour: " + String(LastHour);
    bot.sendMessage(chat_id, msg, "Markdown");
  }
  else if (text == "/on") {
    digitalWrite(LED_PIN, HIGH);
    bot.sendMessage(chat_id, "💡 LED turned *ON*", "Markdown");
  }
  else if (text == "/off") {
    digitalWrite(LED_PIN, LOW);
    bot.sendMessage(chat_id, "💡 LED turned *OFF*", "Markdown");
  }
  else if (text == "/help") {
    bot.sendMessage(chat_id,
      "🤖 *Available Commands:*\n"
      "/status - Show current sensor data\n"
      "/on - Turn LED on\n"
      "/off - Turn LED off\n"
      "/login your_password - Login as guest\n"
      "/logout - Logout your session\n"
      "/whoami - Know your role",
      "Markdown");
  }
  else if (text == "/whoami") {
    if (chat_id == ADMIN_CHAT_ID) {
      bot.sendMessage(chat_id, "👑 You are the *Admin*", "Markdown");
    } else if (guestLoggedIn[chat_id]) {
      bot.sendMessage(chat_id, "🧑‍🚀 You are a *Logged-in Guest*", "Markdown");
    } else {
      bot.sendMessage(chat_id, "🕵️‍♂️ You are *not logged in*", "Markdown");
    }
  }
  else {
    bot.sendMessage(chat_id, "❓ Unknown command. Send /help to see available options.");
  }
}

// Parse sensor data from Serial2
void parseSensorData(String data) {
  Serial.println("Received: " + data);
  sscanf(data.c_str(), "%f,%f,%f,%d,%d,%d", &latestTemp, &latestHumid, &latestVolt, &LastHour, &lowBatteryWarning, &lowBatteryPercentage);
}

// Send message to all logged-in users
void broadcastToLoggedInUsers(String message) {
  bot.sendMessage(ADMIN_CHAT_ID, message, "");
  for (const auto& entry : guestLoggedIn) {
    if (entry.second) {
      bot.sendMessage(entry.first, message, "");
    }
  }
}
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <DHT.h>
#include <Adafruit_BMP085.h>
#include <Servo.h>

// WiFi and Telegram Bot
const char* ssid = "BAJPAI_2.4Ghz";  // YOUR_SSID
const char* password = "44444422"; // YOUR_PASSWORD
#define BOT_TOKEN "YOUR_TELEGRAM_BOT_TOKEN" // YOUR_TELEGRAM_BOT_TOKEN
#define CHAT_ID "YOUR_CHAT_ID" // YOUR_CHAT_ID

WiFiClientSecure client;
UniversalTelegramBot bot(BOT_TOKEN, client);
unsigned long lastTimeBotRan;

// Sensor Pins
#define DHTPIN 15
#define DHTTYPE DHT11
#define MQ135_PIN 34
#define RAIN_SENSOR_PIN 35
#define TRIG_PIN 5
#define ECHO_PIN 18
#define DOOR_SENSOR_PIN 32
#define PIR_PIN 33

// Relay Control
#define LIGHT_RELAY_PIN 25
#define FAN_RELAY_PIN 26

// Servo Lock
Servo lockServo;
#define SERVO_PIN 27

// DHT and BMP Setup
DHT dht(DHTPIN, DHTTYPE);
Adafruit_BMP085 bmp;

// Motion Detection
bool motionActive = false;
bool wasRaining = false; // Rain state tracker
bool gasAlertSent = false; // Gas alert flag

void setup() {
  Serial.begin(115200);
  dht.begin();
  bmp.begin();
  lockServo.attach(SERVO_PIN);

  pinMode(MQ135_PIN, INPUT);
  pinMode(RAIN_SENSOR_PIN, INPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(DOOR_SENSOR_PIN, INPUT_PULLUP);
  pinMode(PIR_PIN, INPUT);

  pinMode(LIGHT_RELAY_PIN, OUTPUT);
  pinMode(FAN_RELAY_PIN, OUTPUT);

  digitalWrite(LIGHT_RELAY_PIN, HIGH);
  digitalWrite(FAN_RELAY_PIN, HIGH);

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  client.setInsecure();
  bot.sendMessage(CHAT_ID, "🤖 Smart Home Bot is Online!", "");
}

long readUltrasonicDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  return pulseIn(ECHO_PIN, HIGH);
}

void handleNewMessages(int numNewMessages) {
  for (int i = 0; i < numNewMessages; i++) {
    String text = bot.messages[i].text;
    String chat_id = bot.messages[i].chat_id;

    if (text == "/status") {
      float temp = dht.readTemperature();
      float hum = dht.readHumidity();
      int air = analogRead(MQ135_PIN);
      int rain = digitalRead(RAIN_SENSOR_PIN);
      float pressure = bmp.readPressure() / 100.0F;
      bool door = digitalRead(DOOR_SENSOR_PIN);

      String message = "🌡 Temp: " + String(temp) + " °C\n";
      message += "💧 Humidity: " + String(hum) + " %\n";
      message += "📈 Pressure: " + String(pressure) + " hPa\n";
      message += "🌫 Air Quality: " + String(air) + "\n";
      message += "🌧 Rain: " + String(rain == LOW ? "Yes" : "No") + "\n";
      message += "🚪 Door: " + String(door == LOW ? "Closed" : "Open") + "\n";
      message += "🕵️ PIR: " + String(motionActive ? "ACTIVE" : "OFF") + "";
      bot.sendMessage(chat_id, message, "");
    }
    else if (text == "/on light") {
      digitalWrite(LIGHT_RELAY_PIN, LOW);
      bot.sendMessage(chat_id, "💡 Light turned ON", "");
    }
    else if (text == "/off light") {
      digitalWrite(LIGHT_RELAY_PIN, HIGH);
      bot.sendMessage(chat_id, "💡 Light turned OFF", "");
    }
    else if (text == "/on fan") {
      digitalWrite(FAN_RELAY_PIN, LOW);
      bot.sendMessage(chat_id, "🌬 Fan turned ON", "");
    }
    else if (text == "/off fan") {
      digitalWrite(FAN_RELAY_PIN, HIGH);
      bot.sendMessage(chat_id, "🌬 Fan turned OFF", "");
    }
    else if (text == "/lock") {
      lockServo.write(0);
      bot.sendMessage(chat_id, "🔐 Gate Locked", "");
    }
    else if (text == "/unlock") {
      lockServo.write(90);
      bot.sendMessage(chat_id, "🔓 Gate Unlocked", "");
    }
    else if (text == "/waterlevel") {
      long duration = readUltrasonicDistance();
      float distance = duration * 0.034 / 2;
      float level = map(distance, 0, 30, 100, 0); // 30cm = max depth
      bot.sendMessage(chat_id, "🚰 Water Tank Level: " + String(level) + "%", "");
    }
    else if (text == "/motion on") {
      motionActive = true;
      bot.sendMessage(chat_id, "🟢 Motion detection ENABLED", "");
    }
    else if (text == "/motion off") {
      motionActive = false;
      bot.sendMessage(chat_id, "🔴 Motion detection DISABLED", "");
    }
  }
}

void loop() {
  if (millis() - lastTimeBotRan > 1000) {
    int numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    while (numNewMessages) {
      handleNewMessages(numNewMessages);
      numNewMessages = 
