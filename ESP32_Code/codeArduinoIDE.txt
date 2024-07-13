//contact:anloufamine@gmail.com


#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "espWIFI0";
const char* password = "01234567890";

const char* mqtt_server = "broker.hivemq.com";

const char* speed_topic = "scooter/speed5";
const char* feedback_topic = "scooter/feedback5";
const char* kp_topic = "scooter/kp5";
const char* ki_topic = "scooter/ki5";

const int motor_pin = 5; 
const int Vm = 296.5;

int speedValue;

WiFiClient espClient;
PubSubClient client(espClient);

float Kp = 0;
float Ki = 0; 
float integral = 0; // Variable pour stocker l'accumulation de l'erreur

const int capteur_pin = 33;

unsigned long previousMillis = 0; // Variable pour stocker le dernier moment où la lecture de la vitesse a été effectuée
const long interval = 0.1; 



float lireVitesse() {
  // Lire la tension du capteur et convertir en vitesse
  float tension_capteur = analogRead(capteur_pin) * (5.0 / 4095.0); // Conversion de la lecture analogique en tension (0-5V)
  float vitesseRetour = 0;
  if (tension_capteur < 0.1){
    vitesseRetour = 0;
  }
  else {
    vitesseRetour = 59 * tension_capteur + 1.55455;
  }
  return vitesseRetour;
}

void setup_wifi() {
  delay(10);
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  
  if (strcmp(topic, kp_topic) == 0) {
    // Convertir le payload en float pour Kp
    String payloadStr = "";
    for (int i = 0; i < length; i++) {
      payloadStr += (char)payload[i];
    }
    Kp = payloadStr.toFloat();
    Serial.print("New Kp value: ");
    Serial.println(Kp);
  } 
  else if (strcmp(topic, ki_topic) == 0) {
    // Convertir le payload en float pour Ki
    String payloadStr = "";
    for (int i = 0; i < length; i++) {
      payloadStr += (char)payload[i];
    }
    Ki = payloadStr.toFloat();
    Serial.print("New Ki value: ");
    Serial.println(Ki);
  }
  else {
    speedValue = atoi((char*)payload);
    if (speedValue > Vm){
      Serial.println("erreur MQTT");
    } 
    else {
      Serial.println(speedValue);
    }

    // Map speed value to PWM value
    int pwmValue = map(speedValue, 0, Vm, 0, 255); 

    // Control the motor using PWM
    analogWrite(motor_pin, pwmValue);
  }
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    
    if (client.connect("wa79rrbID00")) {
      Serial.println("connected");
      client.subscribe(speed_topic);
      client.subscribe(kp_topic);
      client.subscribe(ki_topic);
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  digitalWrite(35, HIGH);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  pinMode(motor_pin, OUTPUT);
  pinMode(35, OUTPUT);
  pinMode(capteur_pin, INPUT);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long currentMillis = millis(); // Temps actuel
  if (currentMillis - previousMillis >= interval) { 
    previousMillis = currentMillis; // Sauvegarder le temps actuel

    // Lire la vitesse
    float vitesseRetour = lireVitesse();

    // Envoyer la vitesse à MQTT
    client.publish(feedback_topic, String(vitesseRetour).c_str()); // Publier la vitesse sur MQTT

    // Afficher la vitesse sur le moniteur série
    Serial.print("Vitesse actuelle: ");
    Serial.println(vitesseRetour);

    float erreur = speedValue - vitesseRetour;
    integral += erreur * (interval / 1000.0); // Accumuler l'erreur 
    
    int pwmValue = map(speedValue, 0, Vm, 0, 255);
    
    int commandePWM = pwmValue + (Kp * erreur + Ki * integral);
    commandePWM = constrain(commandePWM, 0, 255);
    analogWrite(motor_pin, commandePWM);
  }
}
