# ROBOT-EXTINCTEUR-BEN
le code Arduino du robot extincteur
#include <WiFi.h>
#include <Wire.h>
#include <Adafruit_MLX90640.h>

// === CONFIG WIFI ===
const char* ssid = "RobotFeu";
const char* password = "12345678";
WiFiServer server(80);

// === PINS ===
#define MOTOR_LEFT_IN1   25
#define MOTOR_LEFT_IN2   26
#define MOTOR_RIGHT_IN1  27
#define MOTOR_RIGHT_IN2  14
#define RELAY_PUMP_PIN   23
#define TRIG_PIN  5
#define ECHO_PIN  18
#define IR_LEFT_PIN   16
#define IR_RIGHT_PIN  17

// === THERMAL ===
Adafruit_MLX90640 mlx;
float frame[32*24];

// === AUTONOMOUS MODE FLAG ===
bool autonomousMode = false;

// === SEUILS ===
#define FIRE_TEMP_THRESHOLD 60.0
#define SAFE_TEMP_THRESHOLD 40.0
#define STOP_DISTANCE  20.0

// === HTML PAGE AVEC BOUTON AUTONOME ===
const char* htmlPage = R"rawliteral(
<!DOCTYPE HTML>
<html>
<head>
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Robot Feu - Contrôle & Thermique</title>
</head>
<body>
<h2>Contrôle Robot Feu</h2>
<form action="/CMD" method="get">
<button name="btn" value="AV" style="width:80px;height:80px;">Avancer</button><br><br>
<button name="btn" value="GA" style="width:80px;height:80px;">Gauche</button>
<button name="btn" value="AR" style="width:80px;height:80px;">Reculer</button>
<button name="btn" value="DR" style="width:80px;height:80px;">Droite</button><br><br>
<button name="btn" value="STOP" style="width:80px;height:80px;">Stop</button><br><br>
<button name="btn" value="PON" style="width:80px;height:60px;">Pompe ON</button>
<button name="btn" value="POFF" style="width:80px;height:60px;">Pompe OFF</button><br><br>
<button name="btn" value="AUTO" style="width:120px;height:60px;background:#0c0;color:#fff;">Autonome ON/OFF</button>
</form>
<h2>Image thermique (MLX90640)</h2>
<canvas id="thermal" width="320" height="240" style="border:1px solid #000;"></canvas>
<script>
function tempToColor(t) {
  var minT = 20, maxT = 80;
  var v = Math.max(0, Math.min(1, (t-minT)/(maxT-minT)));
  var r = Math.round(255*v);
  var g = 0;
  var b = Math.round(255*(1-v));
  return "rgb("+r+","+g+","+b+")";
}
function drawThermal(data) {
  var c = document.getElementById('thermal');
  var ctx = c.getContext('2d');
  var w = 32, h = 24;
  var cellW = c.width/w, cellH = c.height/h;
  for(var y=0;y<h;y++) {
    for(var x=0;x<w;x++) {
      var t = data[y*w+x];
      ctx.fillStyle = tempToColor(t);
      ctx.fillRect(x*cellW, y*cellH, cellW, cellH);
    }
  }
}
function fetchThermal() {
  fetch('/THERMAL').then(r=>r.json()).then(drawThermal);
}
setInterval(fetchThermal, 1000);
</script>
</body>
</html>
)rawliteral";

void setup() {
  Serial.begin(115200);
  pinMode(MOTOR_LEFT_IN1, OUTPUT);
  pinMode(MOTOR_LEFT_IN2, OUTPUT);
  pinMode(MOTOR_RIGHT_IN1, OUTPUT);
  pinMode(MOTOR_RIGHT_IN2, OUTPUT);
  pinMode(RELAY_PUMP_PIN, OUTPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(IR_LEFT_PIN, INPUT);
  pinMode(IR_RIGHT_PIN, INPUT);
  stopMotors(); digitalWrite(RELAY_PUMP_PIN, LOW);

  WiFi.softAP(ssid, password);
  IPAddress IP = WiFi.softAPIP();
  Serial.print("Connectez-vous au WiFi: "); Serial.println(ssid);
  Serial.print("Page de contrôle: http://"); Serial.println(IP);
  server.begin();

  Wire.begin();
  if (!mlx.begin(0x33, &Wire)) {
    Serial.println("Erreur MLX90640 !");
    while (1);
  }
  mlx.setMode(MLX90640_CHESS);
  mlx.setResolution(MLX90640_ADC_18BIT);
  mlx.setRefreshRate(MLX90640_4_HZ);
}

void loop() {
  // Gestion serveur web et commandes
  WiFiClient client = server.available();
  if (client) {
    String req = client.readStringUntil('\r');
    client.flush();
    if (req.indexOf("GET / ") >= 0) {
      client.print("HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n");
      client.print(htmlPage);
    }
    if (req.indexOf("GET /CMD?btn=") >= 0) {
      if (req.indexOf("btn=AV") > 0) {autonomousMode = false; avancer();}
      else if (req.indexOf("btn=AR") > 0) {autonomousMode = false; reculer();}
      else if (req.indexOf("btn=GA") > 0) {autonomousMode = false; tournerGauche();}
      else if (req.indexOf("btn=DR") > 0) {autonomousMode = false; tournerDroite();}
      else if (req.indexOf("btn=STOP") > 0) {autonomousMode = false; stopMotors();}
      else if (req.indexOf("btn=PON") > 0) {digitalWrite(RELAY_PUMP_PIN, HIGH);}
      else if (req.indexOf("btn=POFF") > 0) {digitalWrite(RELAY_PUMP_PIN, LOW);}
      else if (req.indexOf("btn=AUTO") > 0) {autonomousMode = !autonomousMode; stopMotors();}
      client.print("HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n");
      client.print(htmlPage);
    }
    if (req.indexOf("GET /THERMAL") >= 0) {
      String j = "[";
      if (mlx.getFrame(frame) == 0) {
        for (int i = 0; i < 32 * 24; i++) {
          j += String(frame[i], 1);
          if (i != 32 * 24 - 1) j += ",";
        }
      } else {
        for (int i = 0; i < 32 * 24; i++) j += "0,";
        j += "0";
      }
      j += "]";
      client.print("HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n");
      client.print(j);
    }
    delay(1);
    client.stop();
  }

  // === MODE AUTONOME ===
  if (autonomousMode) {
    navigationAutonome();
  }
}

// === AUTONOMOUS NAVIGATION (évitement + feu) ===
void navigationAutonome() {
  static unsigned long lastAuto = 0;
  if (millis() - lastAuto < 200) return; // 1 action toutes les 200 ms
  lastAuto = millis();

  float fireTemp = lireTemperatureMax();
  if (fireTemp >= FIRE_TEMP_THRESHOLD) {
    // Approche du feu
    float distance = lireDistanceUltrasonique();
    if (distance > STOP_DISTANCE) {
      avancer();
    } else {
      stopMotors();
      digitalWrite(RELAY_PUMP_PIN, HIGH);
      // Tant que le feu n'est pas éteint
      unsigned long t0 = millis();
      while (lireTemperatureMax() >= SAFE_TEMP_THRESHOLD && millis()-t0<15000) {
        delay(400); // sécurité timeout 15s max
      }
      digitalWrite(RELAY_PUMP_PIN, LOW);
      delay(1200); // recule après extinction
      reculer();
      delay(900);
      stopMotors();
    }
  } else {
    // Évitement d'obstacles
    float distance = lireDistanceUltrasonique();
    bool irGauche = digitalRead(IR_LEFT_PIN);
    bool irDroite = digitalRead(IR_RIGHT_PIN);

    if (distance < STOP_DISTANCE) {
      stopMotors(); delay(100);
      reculer(); delay(700);
      stopMotors(); delay(120);
      tournerDroite(); delay(600);
      stopMotors(); delay(100);
    } else if (irGauche) {
      tournerDroite(); delay(350); stopMotors(); delay(80);
    } else if (irDroite) {
      tournerGauche(); delay(350); stopMotors(); delay(80);
    } else {
      avancer();
    }
  }
}

// === Fonctions moteurs ===
void avancer() {
  digitalWrite(MOTOR_LEFT_IN1, HIGH);  digitalWrite(MOTOR_LEFT_IN2, LOW);
  digitalWrite(MOTOR_RIGHT_IN1, HIGH); digitalWrite(MOTOR_RIGHT_IN2, LOW);
}
void reculer() {
  digitalWrite(MOTOR_LEFT_IN1, LOW);   digitalWrite(MOTOR_LEFT_IN2, HIGH);
  digitalWrite(MOTOR_RIGHT_IN1, LOW);  digitalWrite(MOTOR_RIGHT_IN2, HIGH);
}
void tournerGauche() {
  digitalWrite(MOTOR_LEFT_IN1, LOW);   digitalWrite(MOTOR_LEFT_IN2, HIGH);
  digitalWrite(MOTOR_RIGHT_IN1, HIGH); digitalWrite(MOTOR_RIGHT_IN2, LOW);
}
void tournerDroite() {
  digitalWrite(MOTOR_LEFT_IN1, HIGH);  digitalWrite(MOTOR_LEFT_IN2, LOW);
  digitalWrite(MOTOR_RIGHT_IN1, LOW);  digitalWrite(MOTOR_RIGHT_IN2, HIGH);
}
void stopMotors() {
  digitalWrite(MOTOR_LEFT_IN1, LOW);   digitalWrite(MOTOR_LEFT_IN2, LOW);
  digitalWrite(MOTOR_RIGHT_IN1, LOW);  digitalWrite(MOTOR_RIGHT_IN2, LOW);
}

// === Capteurs ===
float lireTemperatureMax() {
  if (mlx.getFrame(frame) == 0) {
    float maxTemp = -1000.0;
    for (int i = 0; i < 32*24; i++) {
      if (frame[i] > maxTemp) maxTemp = frame[i];
    }
    return maxTemp;
  } else {
    Serial.println("Erreur lecture MLX90640");
    return -1;
  }
}
float lireDistanceUltrasonique() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  float distance = duration * 0.0343 / 2.0;
  return distance;
}
