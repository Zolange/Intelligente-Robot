# Intelligente-Robot
Here ia a Robot project with Esp32
# Intelligente-Robot
Here ia a Robot project with Esp32
#include <WiFi.h>
#include <WebServer.h>
#include <ESP32PWM.h>  // Ajout de la bibliothèque PWM pour ESP32

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
  analogWrite(ENA, speedLeft);    // Utilisation de analogWrite au lieu de ledcWrite
  analogWrite(ENB, speedRight);   // Utilisation de analogWrite au lieu de ledcWrite

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
  analogWrite(ENA, 0);    // Utilisation de analogWrite au lieu de ledcWrite
  analogWrite(ENB, 0);    // Utilisation de analogWrite au lieu de ledcWrite
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

// Gestion des requêtes HTTP
void handleRoot() {
  String page = R"=====(
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Contrôle Robot - Pispa Tech Mali</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        
        body {
            background: linear-gradient(135deg, #1a2a6c, #b21f1f, #fdbb2d);
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            align-items: center;
            padding: 20px;
            color: white;
        }
        
        .container {
            width: 100%;
            max-width: 500px;
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 20px;
        }
        
        header {
            text-align: center;
            margin-bottom: 20px;
        }
        
        h1 {
            font-size: 2.2rem;
            margin-bottom: 10px;
            text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
            background: linear-gradient(90deg, #00bfff, #0080ff);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
        }
        
        .status {
            background-color: rgba(0, 0, 0, 0.3);
            padding: 8px 15px;
            border-radius: 20px;
            font-size: 0.9rem;
        }
        
        .control-panel {
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            grid-template-rows: repeat(3, 1fr);
            gap: 12px;
            width: 100%;
            aspect-ratio: 1/1;
            max-width: 350px;
            margin: 0 auto;
        }
        
        .btn {
            background: rgba(255, 255, 255, 0.2);
            border: 2px solid rgba(255, 255, 255, 0.3);
            border-radius: 15px;
            font-size: 1.2rem;
            font-weight: bold;
            color: white;
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: center;
            transition: all 0.2s;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
        }
        
        .btn:active, .btn.active {
            transform: scale(0.95);
            background: rgba(74, 111, 165, 0.8);
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
        }
        
        .btn-stop {
            grid-column: 2;
            grid-row: 2;
            background: rgba(255, 68, 68, 0.7);
        }
        
        .btn-stop:active, .btn-stop.active {
            background: rgba(255, 0, 0, 0.9);
        }
        
        .btn-forward {
            grid-column: 2;
            grid-row: 1;
        }
        
        .btn-backward {
            grid-column: 2;
            grid-row: 3;
        }
        
        .btn-left {
            grid-column: 1;
            grid-row: 2;
        }
        
        .btn-right {
            grid-column: 3;
            grid-row: 2;
        }
        
        .controls-bottom {
            display: flex;
            justify-content: center;
            align-items: center;
            gap: 15px;
            width: 100%;
            margin-top: 10px;
        }
        
        .switch-container {
            display: flex;
            align-items: center;
            gap: 10px;
            background: rgba(0, 0, 0, 0.2);
            padding: 10px 15px;
            border-radius: 25px;
        }
        
        .switch {
            position: relative;
            width: 60px;
            height: 30px;
        }
        
        .switch input {
            opacity: 0;
            width: 0;
            height: 0;
        }
        
        .slider {
            position: absolute;
            cursor: pointer;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background-color: #666;
            transition: .4s;
            border-radius: 34px;
        }
        
        .slider:before {
            position: absolute;
            content: "";
            height: 22px;
            width: 22px;
            left: 4px;
            bottom: 4px;
            background-color: white;
            transition: .4s;
            border-radius: 50%;
        }
        
        input:checked + .slider {
            background-color: #4CAF50;
        }
        
        input:checked + .slider:before {
            transform: translateX(30px);
        }
        
        .sensor-data {
            background: rgba(0, 0, 0, 0.3);
            padding: 15px;
            border-radius: 15px;
            width: 100%;
            text-align: center;
        }
        
        .sensor-value {
            font-size: 1.5rem;
            font-weight: bold;
            margin-top: 5px;
            color: #00bfff;
        }
        
        .key-info {
            margin-top: 20px;
            text-align: center;
            font-size: 0.9rem;
            opacity: 0.8;
        }
        
        .connection-status {
            display: flex;
            align-items: center;
            gap: 8px;
            margin-top: 5px;
        }
        
        .status-dot {
            width: 10px;
            height: 10px;
            border-radius: 50%;
            background-color: #4CAF50;
            animation: pulse 1.5s infinite;
        }
        
        @keyframes pulse {
            0% { opacity: 1; }
            50% { opacity: 0.5; }
           100% { opacity: 1; }
        }
        
        @media (max-width: 500px) {
            h1 {
                font-size: 1.8rem;
            }
            
            .btn {
                font-size: 1rem;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <header>
            <h1>Pispa Tech Mali - Robot Controller</h1>
            <div class="status">
                <div>Contrôle via Interface Web</div>
                <div class="connection-status">
                    <div class="status-dot"></div>
                    <span>Connecté</span>
                </div>
            </div>
        </header>
        
        <div class="control-panel">
            <button class="btn btn-forward" id="forward">↑<br>Avancer</button>
            <button class="btn btn-left" id="left">←<br>Gauche</button>
            <button class="btn btn-stop" id="stop">●<br>Stop</button>
            <button class="btn btn-right" id="right">→<br>Droite</button>
            <button class="btn btn-backward" id="backward">↓<br>Reculer</button>
        </div>
        
        <div class="controls-bottom">
            <div class="switch-container">
                <span>Moteurs:</span>
                <label class="switch">
                    <input type="checkbox" id="motorToggle">
                    <span class="slider"></span>
                </label>
                <span id="motorState">OFF</span>
            </div>
        </div>
        
        <div class="sensor-data">
            <div>Distance de l'obstacle:</div>
            <div class="sensor-value" id="distanceValue">-- cm</div>
        </div>
        
        <div class="key-info">
            Utilisez les touches ZQSD ou les flèches pour contrôler le robot
        </div>
    </div>

    <script>
        // Éléments DOM
        const forwardBtn = document.getElementById('forward');
        const backwardBtn = document.getElementById('backward');
        const leftBtn = document.getElementById('left');
        const rightBtn = document.getElementById('right');
        const stopBtn = document.getElementById('stop');
        const motorToggle = document.getElementById('motorToggle');
        const motorState = document.getElementById('motorState');
        const distanceValue = document.getElementById('distanceValue');
        
        // Variables d'état
        let motorEnabled = false;
        let obstacleDetected = false;
        let currentDirection = null;
        
        // Simulation des données du capteur
        function updateSensorData() {
            // Cette fonction simulerait la récupération des données du capteur
            const distance = Math.random() * 100;
            distanceValue.textContent = `${distance.toFixed(1)} cm`;
            
            // Simulation d'obstacle
            if (distance < 10 && motorEnabled && currentDirection === 'forward') {
                obstacleDetected = true;
                stopMotors();
                alert("Obstacle détecté! Arrêt du robot.");
            } else {
                obstacleDetected = false;
            }
        }
        
        // Mise à jour périodique des données du capteur
        setInterval(updateSensorData, 1000);
        
        // Fonctions de contrôle
        function moveForward() {
            if (!motorEnabled) return;
            currentDirection = 'forward';
            console.log("Avancer");
            forwardBtn.classList.add('active');
        }
        
        function moveBackward() {
            if (!motorEnabled) return;
            currentDirection = 'backward';
            console.log("Reculer");
            backwardBtn.classList.add('active');
        }
        
        function turnLeft() {
            if (!motorEnabled) return;
            currentDirection = 'left';
            console.log("Gauche");
            leftBtn.classList.add('active');
        }
        
        function turnRight() {
            if (!motorEnabled) return;
            currentDirection = 'right';
            console.log("Droite");
            rightBtn.classList.add('active');
        }
        
        function stopMotors() {
            currentDirection = null;
            console.log("Arrêt");
            removeActiveClasses();
            stopBtn.classList.add('active');
            setTimeout(() => stopBtn.classList.remove('active'), 300);
        }
        
        function removeActiveClasses() {
            forwardBtn.classList.remove('active');
            backwardBtn.classList.remove('active');
            leftBtn.classList.remove('active');
            rightBtn.classList.remove('active');
        }
        
        function toggleMotor() {
            motorEnabled = !motorEnabled;
            motorState.textContent = motorEnabled ? "ON" : "OFF";
            
            if (!motorEnabled) {
                stopMotors();
            }
            
            console.log(`Moteurs ${motorEnabled ? 'activés' : 'désactivés'}`);
        }
        
        // Gestionnaires d'événements pour les boutons
        forwardBtn.addEventListener('mousedown', moveForward);
        forwardBtn.addEventListener('touchstart', moveForward);
        forwardBtn.addEventListener('mouseup', stopMotors);
        forwardBtn.addEventListener('touchend', stopMotors);
        
        backwardBtn.addEventListener('mousedown', moveBackward);
        backwardBtn.addEventListener('touchstart', moveBackward);
        backwardBtn.addEventListener('mouseup', stopMotors);
        backwardBtn.addEventListener('touchend', stopMotors);
        
        leftBtn.addEventListener('mousedown', turnLeft);
        leftBtn.addEventListener('touchstart', turnLeft);
        leftBtn.addEventListener('mouseup', stopMotors);
        leftBtn.addEventListener('touchend', stopMotors);
        
        rightBtn.addEventListener('mousedown', turnRight);
        rightBtn.addEventListener('touchstart', turnRight);
        rightBtn.addEventListener('mouseup', stopMotors);
        rightBtn.addEventListener('touchend', stopMotors);
        
        stopBtn.addEventListener('click', stopMotors);
        motorToggle.addEventListener('change', toggleMotor);
        
        // Contrôle clavier
        document.addEventListener('keydown', (e) => {
            if (!motorEnabled) return;
            
            switch(e.key) {
                case 'ArrowUp':
                case 'z':
                case 'Z':
                    moveForward();
                    break;
                case 'ArrowDown':
                case 's':
                case 'S':
                    moveBackward();
                    break;
                case 'ArrowLeft':
                case 'q':
                case 'Q':
                    turnLeft();
                    break;
                case 'ArrowRight':
                case 'd':
                case 'D':
                    turnRight();
                    break;
                case ' ':
                    stopMotors();
                    break;
            }
        });
        
        document.addEventListener('keyup', (e) => {
            if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', 'z', 'Z', 's', 'S', 'q', 'Q', 'd', 'D'].includes(e.key)) {
                stopMotors();
            }
        });
        
        // Empêcher le défilement lors de l'utilisation de l'interface tactile
        document.addEventListener('touchstart', (e) => {
            if (e.target.tagName === 'BUTTON') {
                e.preventDefault();
            }
        }, { passive: false });
    </script>
</body>
</html>
)=====";

  server.send(200, "text/html", page);
}

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
  // Initialisation des broches moteurs
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENA, OUTPUT);
  pinMode(ENB, OUTPUT);
  
  // Initialisation du capteur à ultrasons
  pinMode(TRIGGER_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  // Configuration PWM simplifiée
  // Utilisation de analogWrite au lieu de ledcWrite pour la compatibilité
  analogWrite(ENA, 0);
  analogWrite(ENB, 0);
  
  stopMoteurs();
  
  // Initialisation série
  Serial.begin(115200);
  
  // Création du point d'accès
  WiFi.softAP(ssid, password);
  IPAddress IP = WiFi.softAPIP();
  Serial.print("Point d'accès actif. IP: ");
  Serial.println(IP);
  
  // Configuration du serveur web
  server.on("/", handleRoot);
  server.on("/forward", handleCommand);
  server.on("/backward", handleCommand);
  server.on("/left", handleCommand);
  server.on("/right", handleCommand);
  server.on("/stop", handleCommand);
  server.on("/motorToggle", handleCommand);
  server.on("/speed", handleCommand);
  
  server.begin();
  Serial.println("Serveur web actif!");
}

void loop() {
  server.handleClient();
  
  if (motorOn) {
    checkObstacle(); // Vérifie en permanence les obstacles uniquement si moteur activé
  }
  
  delay(10);
}

