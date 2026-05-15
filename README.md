#include <ESP8266WiFi.h>
#include <ESP_Mail_Client.h>

// ── WiFi CREDENTIALS ────────────────────
const char* ssid     = "Sahasra";
const char* password = "sahasra0207";

// ── GMAIL CREDENTIALS ────────────────────
#define SENDER_EMAIL    "sahasravishnubaktula02@gmail.com"
#define SENDER_PASSWORD ""    // 16-char App Password (no spaces)
#define SENDER_NAME     "VoltSafe AI"

// ── RECIPIENT EMAILS (nearby people) ────
const char* recipients[] = {
  "23.612ashmithmekala@gmail.com",
  "sahasrasahu406@gmail.com",
  "srividhyapyaram@gmail.com"
};
const int recipientCount = 3;

// ── PIN DEFINITIONS ─────────────────────
#define BUZZER_PIN   5    // D1
#define RED_LED_PIN  4    // D2
#define RELAY_PIN    14   // D5 → power control

// ── THRESHOLDS ───────────────────────────
const int WATER_THRESHOLD   = 300;   // water touches wire
const int CURRENT_THRESHOLD = 700;   // live current in water

// ── STATE FLAGS ──────────────────────────
bool alertSent  = false;
bool normalSent = true;

// ── SMTP SESSION ─────────────────────────
SMTPSession smtp;
Session_Config config;

// ─────────────────────────────────────────

void setup() {
  Serial.begin(9600);

  pinMode(BUZZER_PIN,  OUTPUT);
  pinMode(RED_LED_PIN, OUTPUT);
  pinMode(RELAY_PIN,   OUTPUT);

  // Active LOW relay — LOW = power ON, HIGH = power CUT
  digitalWrite(BUZZER_PIN,  LOW);    // buzzer OFF
  digitalWrite(RED_LED_PIN, LOW);    // red LED OFF
  digitalWrite(RELAY_PIN,   LOW);    // power ON at start (white LED ON)

  // Connect to WiFi
  Serial.print("Connecting to WiFi");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected!");
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());

  // Gmail SMTP config
  config.server.host_name = "smtp.gmail.com";
  config.server.port      = 465;
  config.login.email      = SENDER_EMAIL;
  config.login.password   = SENDER_PASSWORD;
  config.login.user_domain = "";

  Serial.println("VoltSafe AI Ready!");
  Serial.println("White LED ON = Safe, Waiting...");
}

void loop() {
  int val = analogRead(A0);

  Serial.print("Sensor: ");
  Serial.println(val);

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // PRIORITY 1 — LIVE CURRENT IN WATER
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  if (val > CURRENT_THRESHOLD) {

    digitalWrite(RELAY_PIN,   HIGH);   // ⚡ CUT POWER IMMEDIATELY
    digitalWrite(RED_LED_PIN, HIGH);   // red LED ON
    digitalWrite(BUZZER_PIN,  HIGH);   // buzzer ON — current = lethal danger

    if (!alertSent) {
      Serial.println("DANGER! LIVE CURRENT IN WATER! POWER CUT!");
      sendEmailToAll(
        "DANGER - VoltSafe Alert!",
        "<h2 style='color:red'>🚨 DANGER — Live Wire in Water!</h2>"
        "<p><b>Live electrical current detected in water.</b></p>"
        "<p>⚡ <b>Power has been SHUT DOWN immediately.</b></p>"
        "<p>📍 Location: <b>Thimmapur</b></p>"
        "<p style='color:red'><b>Act immediately! This is life-threatening.</b></p>"
        "<hr><p style='color:gray;font-size:12px'>VoltSafe AI System</p>"
      );
      alertSent  = true;
      normalSent = false;
    }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // PRIORITY 2 — WATER TOUCHES WIRE
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  } else if (val > WATER_THRESHOLD) {

    digitalWrite(RELAY_PIN,   HIGH);   // 💧 CUT POWER immediately
    digitalWrite(RED_LED_PIN, HIGH);   // red LED ON
    digitalWrite(BUZZER_PIN,  LOW);    // no buzzer for water alone

    if (!alertSent) {
      Serial.println("WARNING! Water on wire! Power cut.");
      sendEmailToAll(
        "WARNING - VoltSafe Alert!",
        "<h2 style='color:orange'>⚠️ WARNING — Water on Live Wire!</h2>"
        "<p>Water has been detected in contact with a live wire.</p>"
        "<p>🔌 <b>Power has been SHUT DOWN as precaution.</b></p>"
        "<p>📍 Location: <b>Nustulapur</b></p>"
        "<p><b>Please check and fix immediately!</b></p>"
        "<hr><p style='color:gray;font-size:12px'>VoltSafe AI System</p>"
      );
      alertSent  = true;
      normalSent = false;
    }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // SAFE — dry, no current
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  } else {

    digitalWrite(RELAY_PIN,   LOW);    // ✅ RESTORE POWER
    digitalWrite(RED_LED_PIN, LOW);    // red LED OFF
    digitalWrite(BUZZER_PIN,  LOW);    // buzzer OFF

    if (!normalSent) {
      Serial.println("Safe. Power restored.");
      sendEmailToAll(
        "SAFE - VoltSafe Update",
        "<h2 style='color:green'>✅ Area is Safe — Power Restored</h2>"
        "<p>The hazard has been cleared. Wire is dry.</p>"
        "<p>🔌 <b>Power has been RESTORED.</b></p>"
        "<p>📍 Location: <b>Subhash Nagar</b></p>"
        "<hr><p style='color:gray;font-size:12px'>VoltSafe AI System</p>"
      );
      normalSent = true;
      alertSent  = false;
    }
  }

  delay(300);
}

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// SEND EMAIL TO ALL RECIPIENTS
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
void sendEmailToAll(const char* subject, const char* htmlBody) {
  SMTP_Message message;

  message.sender.name  = SENDER_NAME;
  message.sender.email = SENDER_EMAIL;
  message.subject      = subject;

  for (int i = 0; i < recipientCount; i++) {
    message.addRecipient("", recipients[i]);
  }

  message.html.content          = htmlBody;
  message.html.charSet          = "utf-8";
  message.html.transfer_encoding = Content_Transfer_Encoding::enc_qp;

  if (!smtp.connect(&config)) {
    Serial.println("SMTP connect failed!");
    Serial.println(smtp.errorReason());
    return;
  }

  if (!MailClient.sendMail(&smtp, &message)) {
    Serial.println("Email send failed!");
    Serial.println(smtp.errorReason());
  } else {
    Serial.println("Email sent to all!");
  }

  smtp.closeSession();
}
