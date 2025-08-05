### Here's the Arduino code which makes this possible! ###

Do note, certain lines have been modified to secure network details.

```
#include <Wire.h>
#include "SSD1306.h"
#include <ESP8266WiFi.h>  
#include "24s_PubSubClient.h"
#include "24s_WiFiManager.h"
#include <Math.h>
#include <Servo.h>
#include <Ultrasonic.h>


//MQTT Communication associated variables
char payload_global[100];                     
boolean flag_payload = false;                         

//MQTT Setting variables  
const char* mqtt_server = "****";
const char* MQusername = "****";
const char* MQpassword = "****";
const char* MQtopic = "****";
const int mqtt_port = ****;
const char* ssid = "****";
const char* password = "****";
int x, y; 

// OLED Display Setup
SSD1306 display(0x3C, D14, D15); // Adjust the pins according to your ESP8266 module

//WiFi Define
WiFiClient espClient;                         
PubSubClient client(espClient);             

#define motor1pin D0
#define motor2pin D2
Ultrasonic ultrasonic_front(D8, D5);
Ultrasonic ultrasonic_right(D9, D6);
Ultrasonic ultrasonic_left(D10, D7);

//motor stuff
Servo motor1; 
Servo motor2;
//motor 1 is left (slower)
int distance_front = 0;
int distance_right = 0;
int distance_left = 0;

int target = 0;
float targetDist = 0;

int next_target = 1;

double xP = 1;
double xY = 1;

      
void setup_wifi() { 
  delay(10);
  // We stargett by connecting to a Stevens WiFi network
  WiFi.begin(ssid, password);           
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");                        
  }
  randomSeed(micros());                       
}

void callback(char* topic, byte* payload, unsigned int length) {
  for (int i = 0; i < length; i++) {
    payload_global[i] = (char)payload[i];
  }
  payload_global[length] = '\0';              
  flag_payload = true;                        
}

void reconnect() {                                                                
  // Loop until we're reconnected
  while (!client.connected()) {
    // Create a random client ID
    String clientId = "ESP8266Client-";       
    clientId += String(random(0xffff), HEX);  
    // Attempt to connect                     
    if (client.connect(clientId.c_str(),MQusername,MQpassword)) {
      client.subscribe(MQtopic);             
    } else {
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void UpdateCoordinates() {
  //Robot location x,y from MQTT subscription variable testCollector
  client.loop();                              

  String payload(payload_global);              
  int testCollector[10];                      
  int count = 0;
  int prevIndex, delimIndex;
    
  prevIndex = payload.indexOf('[');           
  while( (delimIndex = payload.indexOf(',', prevIndex +1) ) != -1){
    testCollector[count++] = payload.substring(prevIndex+1, delimIndex).toInt();
    prevIndex = delimIndex;
  }
  delimIndex = payload.indexOf(']');
  testCollector[count++] = payload.substring(prevIndex+1, delimIndex).toInt();

  x = testCollector[0];
  y = testCollector[1];

  // OLED for x and y position
  char xPos[50];
  char yPos[50];

  sprintf(xPos, "%d", x);
  sprintf(yPos, "%d", y);
  display.drawString(0, 0, "X-Position: ");
  display.drawString(0, 30, "Y-Position: " );
  display.drawString(50,0, xPos);
  display.drawString(50,30, yPos);
  display.display();
  delay(60);
  display.clear();
}

void setup() {
  Serial.begin(115200);
  setup_wifi();
  // OLED Setup
  display.init();
  display.flipScreenVertically();
  display.drawString(0, 0, "Stevens Smart Robot: 1");
  display.setFont(ArialMT_Plain_16);
  display.display();                               
  delay(3000);
  Serial.println("Wemos POWERING UP ......... ");
  client.setServer(mqtt_server, mqtt_port);          //This 1883 is a TCP/IP port number for MQTT 
  client.setCallback(callback); 

  motor1.attach(motor1pin); 
  motor2.attach(motor2pin);
  motor1.write(90);
  motor2.write(90);
}

void stop(){
  motor1.write(90);
  motor2.write(90);
  delay(2000);
}

void right(double deg){
  motor1.write(180);
  motor2.write(0);
  delay(deg*2.60);
  motor1.write(90);
  motor2.write(90);
}

void left(double deg){
  motor1.write(0);
  motor2.write(180);
  delay(deg*2.60);
  motor1.write(90);
  motor2.write(90);
}

void UpdateSensors() {
  distance_front = ultrasonic_front.read(CM);
  delay(10);
  distance_right = ultrasonic_right.read(CM);
  delay(10);
  distance_left = ultrasonic_left.read(CM);
}

void loop() {
  //subscribe the data from MQTT server
  if (!client.connected()) {
    Serial.print("...");
    reconnect();
  }
  client.loop();                         
 
  String payload(payload_global);              
  int testCollector[10];                      
  int count = 0;
  int prevIndex, delimIndex;
   
  prevIndex = payload.indexOf('[');          
  while( (delimIndex = payload.indexOf(',', prevIndex +1) ) != -1){
    testCollector[count++] = payload.substring(prevIndex+1, delimIndex).toInt();
    prevIndex = delimIndex;
  }
  delimIndex = payload.indexOf(']');
  testCollector[count++] = payload.substring(prevIndex+1, delimIndex).toInt();
   
  int x, y;
  //Robot location x,y from MQTT subscription variable testCollector
  x = testCollector[0];
  y = testCollector[1];

  int TargetX[] = {930, 2050, 170, 150};
  int TargetY[] = {700, 700, 700, 180};
  
  if(x > 1 && x < 2400 && y > 1 && y < 900){

    targetDist = sqrt(sq(x-TargetX[target])+sq(y-TargetY[target]));
    Serial.println(targetDist);

    if (targetDist < 150){
      stop();

      if(target == 1 || target == 2 || target == 3 || target == 4){
        delay(2000);
      }
      else{
        delay(100);
      }
      target++;
      if (target != 3) {
        next_target++;
      }

      if(target > 3){
        exit(1);
      }
    }

    if(x != xP && y != xY){
      double targetdist[2] = {(TargetX[target]-x), (TargetY[target]-y)}; //target distance
      double directionaldist[2] = {(x-xP), (y-xY)}; //movement direction

      //Calculation Time
      double dotproduct = (targetdist[0]*directionaldist[0]) + (targetdist[1]*directionaldist[1]); //dot product
      double mvt = sqrt(sq(targetdist[0]) + sq(targetdist[1])); //Magnitude of targetget vector
      double mvm = sqrt(sq(directionaldist[0]) + sq(directionaldist[1])); //Magnitude of movement vector
      double degree = (180/M_PI)*acos(dotproduct/(mvt*mvm)); //Angle between vectors

      double crossproduct = ((x-xP)*(TargetY[target]-y)) - ((y-xY)*(TargetX[target]-x)); //left or right?

      if (crossproduct < 0){ //if Crossproduct is negative, turn right
        left(degree*.5);
      }
      else if(crossproduct > 0){ //if Crossproduct is positive, turn left
        right(degree*.5); 
      } else if (crossproduct == 0) {
        delay(100);
        UpdateSensors();
      }

      xP = x;
      xY = y;

      display.clear();                              
      String str_1 = "x: " + String(x);             // Converts the x value from 'int' to 'String'
      String str_2 = "y: " + String(y);             // Converts the y value from 'int' to 'String'
      
      String str_4 = "t: " + String(next_target); //next target

      display.drawString(0, 0, "Stats:");
      display.drawString(0, 15, str_1);
      display.drawString(0, 30, str_2);
      display.drawString(60, 15, str_4);
      display.display();
    }
  }

  distance_front = ultrasonic_front.read(CM);
  delay(10);
  distance_right = ultrasonic_right.read(CM);
  delay(10);
  distance_left = ultrasonic_left.read(CM);

  //Serial.println(distance_front);
  //Serial.println(distance_right);
  //Serial.println(distance_left);

  if (distance_front <= 12) {
    //stop, move backright slightly
      motor1.write(90);
      motor2.write(90);
      delay(1000);
      if (distance_right < 10 && distance_left < 10) {
        motor1.write(180);
        motor2.write(180);
        delay(50);
      } else if (distance_left < 10) {
        motor1.write(180);
        delay(100);
        motor1.write(180);
        motor2.write(180);
        delay(50);
      } else if (distance_right < 10) {
        motor2.write(180);
        delay(100);
        motor1.write(180);
        motor2.write(180);
        delay(50);
      } else {
        motor1.write(180);
        motor2.write(180);
        delay(150);
      }
      UpdateSensors();
  } else {
    // drives forward if no obstacles within 12 cm
    if(distance_left > 10){
      motor1.write(60);
      motor2.write(60);
      delay(100);
      UpdateSensors();
    }
    if(distance_left <= 10){
      motor1.write(60);
      motor2.write(90);
      delay(100);      
      UpdateSensors();
    }
    if(distance_right > 10){
      motor1.write(60);
      motor2.write(60);
      delay(100);
      UpdateSensors();
    }
    if(distance_right <= 10){
      motor1.write(90);
      motor2.write(60);
      delay(100);
      UpdateSensors();
    }
  }
}
```
