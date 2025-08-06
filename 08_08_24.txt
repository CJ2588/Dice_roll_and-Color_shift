#include <Adafruit_NeoPixel.h>

#define LED_PIN_70    2
#define LED_PIN_35    3
#define NUM_LEDS_70   70
#define NUM_LEDS_35   35
#define BRIGHTNESS    255
#define triggerPin    8      // Define the ultrasound sensor trigger pin
#define echoPin       9      // Define the ultrasound sensor echo pin
#define MIN_DISTANCE  0      // Define minimum distance to measure (in centimeters)
#define MAX_DISTANCE  9      // Define maximum distance to measure (in centimeters)
#define speakerPinLeft 10    // Define the speaker pin left
#define speakerPinRight 11   // Define the speaker pin right
#define baumerSensorPin A0   // Analog pin for Baumer sensor

const int numReadings = 10;  // Number of readings to average
int readings[numReadings];   // Array to store sensor readings
int index = 0;               // Index for the current reading
int total = 0;               // Total sum of readings

Adafruit_NeoPixel strip_70 = Adafruit_NeoPixel(NUM_LEDS_70, LED_PIN_70, NEO_GRB + NEO_KHZ800);
Adafruit_NeoPixel strip_35 = Adafruit_NeoPixel(NUM_LEDS_35, LED_PIN_35, NEO_GRB + NEO_KHZ800);

void setup() {
  Serial.begin(9600);
  strip_70.begin();
  strip_35.begin();
  strip_70.setBrightness(BRIGHTNESS);
  strip_35.setBrightness(BRIGHTNESS);
  strip_70.show(); // Initialize all pixels to 'off'
  strip_35.show(); // Initialize all pixels to 'off'

  pinMode(speakerPinLeft, OUTPUT);  // Set the left speaker pin as an output
  pinMode(speakerPinRight, OUTPUT); // Set the right speaker pin as an output
  pinMode(triggerPin, OUTPUT);      // Set trigger pin as an output
  pinMode(echoPin, INPUT);          // Set echo pin as an input
  pinMode(baumerSensorPin, INPUT);  // Set Baumer sensor pin as an input

  // Initialize the readings array to zero
  for (int i = 0; i < numReadings; i++) {
    readings[i] = 0;
  }
}

void loop() {
  delay(10);

  // Reading from the Baumer sensor
  int sensorValue = analogRead(baumerSensorPin);
  float voltage = sensorValue * (5.0 / 1023.0);    // Convert the analog value to voltage

  // Print the sensor value and voltage for debugging
  Serial.print("Sensor Value: ");
  Serial.print(sensorValue);
  Serial.print(", Voltage: ");
  Serial.println(voltage);

  int distance = map(sensorValue, 0, 1023, MIN_DISTANCE, MAX_DISTANCE); // Map the analog value to distance

  // Filter to avoid spikes in measurements
  total -= readings[index];
  readings[index] = distance;
  total += readings[index];
  index = (index + 1) % numReadings;

  // Calculate the average distance
  int distance_avg = total / numReadings;

  // Map the distance to frequency for sound distortion
  int freq_left = map(distance_avg, MIN_DISTANCE, MAX_DISTANCE, 100, 1000);  // Adjust frequency range
  int freq_right = map(distance_avg, MIN_DISTANCE, MAX_DISTANCE, 120, 900); // Adjust frequency range

  generatePWM(speakerPinLeft, freq_left);
  generatePWM(speakerPinRight, freq_right);

  // Set LED color based on distance
  setLEDColor(distance_avg);
}

void generatePWM(int pin, int frequency) {
  int period = 1000000 / frequency; // Period in microseconds
  int halfPeriod = period / 2;      // Half period for high/low cycle

  digitalWrite(pin, HIGH);
  delayMicroseconds(halfPeriod);
  digitalWrite(pin, LOW);
  delayMicroseconds(halfPeriod);
}

void setLEDColor(int distance) {
  // Map the distance to a value between 0 and 255
  int value = map(distance, MIN_DISTANCE, MAX_DISTANCE, 255, 0);

  // Calculate the color components
  int red = 255;
  int green = value;
  int blue = value;

  uint32_t color = strip_70.Color(red, green, blue);

  for (int i = 0; i < NUM_LEDS_70; i++) {
    strip_70.setPixelColor(i, color);
  }
  for (int i = 0; i < NUM_LEDS_35; i++) {
    strip_35.setPixelColor(i, color);
  }
  strip_70.show();
  strip_35.show();
}
