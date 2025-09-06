# Intelligente-Robot
Here ia a Robot project with Esp32
# Intelligente-Robot
Here ia a Robot project with Esp32
#include <WiFi.h>
#include <WebServer.h>

// === Définition des broches L298N ===
// Moteur A (Gauche)
#define IN1 25
#define IN2 26
#define ENA 13  // Broche PWM pour le moteur gauche

// Moteur B (Droite)
#define IN3 27
#define IN4 14
#define ENB 12  // Broche PWM pour le moteur droit

// Capteur à ultrasons
#define TRIGGER_PIN 32
#define ECHO_PIN 33

// Paramètres du point d'accès
const char* ssid = "RobotController";
const char* password = "12345678";

WebServer server(80);

// Variables pour le contrôle PWM
int speedLeft = 200;  // Vitesse par défaut (0-255)
int speedRight = 200; // Vitesse par défaut (0-255)

// Variables d'état
bool motorOn = false;
bool isForward = false;
bool isBackward = false;
bool isLeft = false;
bool isRight = false;
unsigned long lastObstacleTime = 0;

// Configuration des canaux PWM
const int pwmChannelLeft = 0;
const int pwmChannelRight = 1;
const int pwmFrequency = 5000;
const int pwmResolution = 8;

// === Fonctions de contrôle moteurs avec PWM ===
void updateMotors() {
  if (!motorOn) {
    stopMoteurs();
    return;
  }

  // Appliquer les vitesses PWM
  ledcWrite(pwmChannelLeft, speedLeft);
  ledcWrite(pwmChannelRight, speedRight);

  if (isForward && isLeft) {
    digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);
    digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH);
  }
  else if (isForward && isRight) {
    digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);
    digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH);
  }
  else if (isBackward && isRight) {
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
  }
  else if (isBackward && isLeft) {
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
  }
  else if (isForward) {
    digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);
    digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH);
  }
  else if (isBackward) {
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
  }
  else if (isLeft) {
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH);
  }
  else if (isRight) {
    digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
  }
  else {
    stopMoteurs();
  }
}

void stopMoteurs() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  ledcWrite(pwmChannelLeft, 0);
  ledcWrite(pwmChannelRight, 0);
}

void resetStates() {
  isForward = isBackward = isLeft = isRight = false;
}

// Fonction pour mesurer la distance avec le capteur à ultrasons
int measureDistance() {
  digitalWrite(TRIGGER_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIGGER_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIGGER_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  int distance = duration * 0.034 / 2;

  return distance;
}

bool checkObstacle() {
  int distance = measureDistance();
  if (distance > 0 && distance < 15 && millis() - lastObstacleTime > 1000) {
    resetStates();
    isBackward = true;
    updateMotors();
    lastObstacleTime = millis();
    delay(500); // recule 500ms
    isBackward = false;
    updateMotors();
    return true;
  }
  return false;
}

// Route pour la page HTML
void handleRoot() {
  String page = R"=====( 
  // === VOTRE HTML/JS RESTE LE MÊME ===
  )=====";

  server.send(200, "text/html", page);
}

void handleDistance() {
  int dist = measureDistance();
  server.send(200, "text/plain", String(dist));
}

// Commandes du robot
void handleCommand() {
  String command = server.uri();

  if (command == "/forward") {
    resetStates();
    isForward = true;
  }
  else if (command == "/backward") {
    resetStates();
    isBackward = true;
  }
  else if (command == "/left") {
    resetStates();
    isLeft = true;
  }
  else if (command == "/right") {
    resetStates();
    isRight = true;
  }
  else if (command == "/stop") {
    resetStates();
  }
  else if (command == "/motorToggle") {
    motorOn = !motorOn;
    server.send(200, "text/plain", motorOn ? "ON" : "OFF");
    return;
  }
  else if (server.hasArg("side") && server.hasArg("value")) {
    String side = server.arg("side");
    int value = server.arg("value").toInt();

    if (side == "left") {
      speedLeft = value;
    } else if (side == "right") {
      speedRight = value;
    }
    server.send(200, "text/plain", "OK");
    return;
  }

  updateMotors();
  server.send(200, "text/plain", "OK");
}

void setup() {
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENA, OUTPUT);
  pinMode(ENB, OUTPUT);

  pinMode(TRIGGER_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  // Configuration des canaux PWM
  ledcSetup(pwmChannelLeft, pwmFrequency, pwmResolution);
  ledcAttachPin(ENA, pwmChannelLeft);
  ledcSetup(pwmChannelRight, pwmFrequency, pwmResolution);
  ledcAttachPin(ENB, pwmChannelRight);

  ledcWrite(pwmChannelLeft, 0);
  ledcWrite(pwmChannelRight, 0);

  stopMoteurs();

  Serial.begin(115200);

  WiFi.softAP(ssid, password);
  IPAddress IP = WiFi.softAPIP();
  Serial.print("Point d'accès actif. IP: ");
  Serial.println(IP);

  server.on("/", handleRoot);
  server.on("/forward", handleCommand);
  server.on("/backward", handleCommand);
  server.on("/left", handleCommand);
  server.on("/right", handleCommand);
  server.on("/stop", handleCommand);
  server.on("/motorToggle", handleCommand);
  server.on("/speed", handleCommand);
  server.on("/distance", handleDistance); // Ajout route distance

  server.begin();
  Serial.println("Serveur web actif!");
}

void loop() {
  server.handleClient();
  if (motorOn) {
    checkObstacle();
  }
  delay(10);
}

  delay(10);
}

