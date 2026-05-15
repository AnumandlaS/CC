week-1
void setup() {
pinMode(13, OUTPUT);    // sets the digital pin 13 as output
}
void loop() {
digitalWrite(13, HIGH); // sets the digital pin 13 on
delay(1000);            // waits for a second
digitalWrite(13, LOW);  // sets the digital pin 13 off
delay(1000);            

week-2
int ldr=A0;//Set A0(Analog Input) for LDR.

int value=0;
void setup() { 
Serial.begin(9600); pinMode(3,OUTPUT);
}
void loop() {
value=analogRead(ldr);//Reads the Value of LDR(light). 
Serial.println("LDR value is :");//Prints the value of LDR to Serial Monitor.
  Serial.println(value);
if(value<300)
{
digitalWrite(3,HIGH);//Makes the LED glow in Dark.
}
else
{
digitalWrite(3,LOW);//Turns the LED OFF in Light.
}
}

week-3
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#define OLED_RESET 4 
Adafruit_SSD1306 display(OLED_RESET);
int greenLed = 2; 
int yellowLed = 3; 
int redLed = 4;
int analogValue = 0; 
float voltage = 0; 
int ledDelay = 1000;
 void setup()
{

display.begin(SSD1306_SWITCHCAPVCC, 0x3C); 
display.display();
display.clearDisplay();
 Serial.begin(9600); 
pinMode(greenLed, OUTPUT); 
pinMode(yellow Led, OUTPUT);  
pinMode(redLed,OUTPUT);
}

void loop()
{
analogValue = analogRead(A0);

voltage = 0.0048*analogValue; 
display.setTextSize(1); 
display.setTextColor(WHITE);
display.setCursor(0,0);
display.println(" Battery Voltage"); 
display.println("");
display.println(""); 
display.setTextSize(3); 
display.print(" ");
display.println(voltage); 
display.display(); 
delay (200);
display.clear Display(); 
Serial.println(voltage);
if( voltage >= 1.35 ) 
digitalWrite(greenLed, HIGH);
else if (voltage > 1.2 && voltage < 1.35) 
digitalWrite(yellowLed, HIGH);
else if( voltage <= 1.2) 
digitalWrite(redLed, HIGH);
delay(ledDelay); 
digitalWrite(redLed, LOW); 
digitalWrite(yellowLed, LOW); 
digitalWrite(greenLed, LOW);
}

week-4
#include <LiquidCrystal.h> 
LiquidCrystal lcd(12, 11, 5, 4, 3, 2); 
int randomnumber;

void setup()
{		
  lcd.begin(16, 2);
  randomSeed(7); 
  pinMode(8, INPUT);
}

void loop()
{
  lcd.setCursor(2, 0); 
  lcd.print("Dice value:");
  int DICEROLL = digitalRead(8); 
  if(DICEROLL==1)
  randomnumber = random(1,7); 
  lcd.setCursor(6, 1); 
  lcd.print(randomnumber);
}

week-5
#include <DHT.h>

// Define the DHT sensor type and pin
#define DHTPIN 8      // Digital pin connected to the DHT sensor
#define DHTTYPE DHT22  // DHT 22  (AM2302)

DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(9600);
  Serial.println("DHT22 Temperature and Humidity Sensor");
  dht.begin();
}

void loop() {
  // Delay between readings (adjust as needed)
  delay(2000);

  // Read temperature and humidity
  float humidity = dht.readHumidity();
  float temperature = dht.readTemperature();

  // Print the readings to the serial monitor
  Serial.print("Temperature: ");
  Serial.print(temperature);
  Serial.print(" *C");
  Serial.print(" Humidity: ");
  Serial.print(humidity);
  Serial.println(" %");
}

week-6
import RPi.GPIO as GPIO import time

GPIO.setwarnings(False) GPIO.setmode(GPIO.BOARD)
GPIO.setup(11, GPIO.IN)	#Read ouput from PIR motion sensor
GPIO.setup(3, GPIO.OUT)	#LED output pin
 
while True:

  i=GPIO.input(11)
  if i == 0:	#When output from motion sensor is LOW print (“No Person detected”,i)
  GPIO.output(3,0) time.sleep(0.1)
  elif i == 1:	#When output from motion sensor is HIGH print (“A Person is detected”,i)
  GPIO.output(3,1)	#Turn ON LED time.sleep(0.1)

week - 7
import RPi.GPIO as GPIO
import time
GPIO.setwarnings(False)
GPIO.setmode(GPIO.BOARD)
while True:
    GPIO.setup(10, GPIO.OUT)
    GPIO.output(10, GPIO.HIGH)
    time.sleep(2)
    GPIO.output(10, GPIO.LOW)
    time.sleep(2)
    
week-8
#Libraries
import RPi.GPIO as GPIO import time

#GPIOMode(BOARD/ BCM)
GPIO.setmode(GPIO.BCM)
#set GPIO Pins GPIO_TRIGGER=18 GPIO_ECHO=24

#set GPIO direction (IN / OUT) GPIO.setup(GPIO_TRIGGER,GPIO.OUT) GPIO.setup(GPIO_ECHO,GPIO.IN)



def distance():
  # set Trigger to HIGH GPIO.output(GPIO_TRIGGER,True)

  # set Trigger after 0.01ms to LOW time.sleep(0.00001)
  GPIO.output(GPIO_TRIGGER,False)

  StartTime = time.time() StopTime=time.time()

  # save StartTime
  while GPIO.input(GPIO_ECHO) == 0: StartTime=time.time()

  # save time of arrival
  while GPIO.input(GPIO_ECHO) == 1: StopTime=time.time()

  # time difference between start and arrival TimeElapsed=StopTime-StartTime #multiply with the sonicspeed(34300cm/s) #and divide by2, because there and back
  distance = (TimeElapsed * 34300) / 2
 
   return distance

if  __name__== '__main__':
	try:
      while True:
        dist=distance()
        print ("Measured Distance = %.1f cm" % dist) time.sleep(1)
        # Reset by pressing CTRL + C 
     except KeyboardInterrupt:
        print("Measurement stopped by User")
              GPIO.cleanup()

week - 9
from picamera import PiCamera from time import sleep

camera = PiCamera() camera.rotation = 180 camera.start_preview() sleep(5) camera.stop_preview()

#modify the image

from picamera import PiCamera from time import sleep
from picamera import PiCamera, Color

camera = PiCamera() camera.rotation = 180 camera.start_preview() camera.image_effect = 'watercolor'
camera.annotate_background = Color('yellow') camera.annotate_foreground = Color('blue') camera.annotate_text_size = 80 camera.annotate_text = "Hello KMIT"

for i in range(5): sleep(5)
camera.capture('/home/pi/Desktop/Rasp1%s.jpg' % i) camera.stop_preview()

#video

from picamera import PiCamera from time import sleep
from picamera import PiCamera, Color

camera = PiCamera() camera.rotation = 180 camera.start_preview()
 
camera.start_preview() camera.annotate_background = Color('yellow') camera.annotate_foreground = Color('blue') camera.annotate_text_size = 80


camera.annotate_text = " KMIT IT " camera.start_recording('/home/pi/Desktop/video.h264') sleep(5)
camera.stop_recording() camera.stop_preview()
          



























