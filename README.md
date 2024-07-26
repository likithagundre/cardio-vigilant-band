# cardio-vigilant-band

#code

#include <Wire.h>
#include <Adafruit_SSD1306.h>
#include <MAX30105.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 32
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

MAX30105 particleSensor;

const int blinkPin = 13;
const int ledPin = 6;

void setup()
{
  Serial.begin(9600);
  pinMode(blinkPin, OUTPUT);
  pinMode(ledPin, OUTPUT);
 
  if (!display.begin(SSD1306_I2C_ADDRESS, OLED_RESET))
  {
    Serial.println(F("SSD1306 allocation failed"));
    for (;;);
  }

  display.display(); // Clear the display buffer.
  display.setTextColor(SSD1306_WHITE); // Set text color to white
  display.setTextSize(1); // Set text size
  display.setCursor(0,0); // Set cursor position to (0,0)
  display.println(F("Heart Rate Monitor"));
  display.display();
 
  // Initialize the MAX30105 sensor
  if (!particleSensor.begin(Wire, I2C_SPEED_FAST))
  {
    Serial.println(F("MAX30105 was not found. Please check your connections."));
    for(;;);
  }
  particleSensor.setup();
  particleSensor.setPulseAmplitudeRed(0x0A); // Adjust the LED brightness
  particleSensor.setPulseAmplitudeGreen(0); // No green LED
}

void loop()
{
  static uint32_t lastSampleTime = 0;
  static int32_t lastBeat = 0;
  static float beatsPerMinute;
 
  int32_t irValue = particleSensor.getIR();
 
  if (checkForBeat(irValue))
  {
    // Calculate the inter-beat interval (IBI) in milliseconds
    uint32_t timeSinceLastBeat = millis() - lastSampleTime;
    lastSampleTime = millis();
   
    // Calculate heart rate in beats per minute (BPM)
    beatsPerMinute = 60 / (timeSinceLastBeat / 1000.0);
   
    if (beatsPerMinute > 40 && beatsPerMinute < 200)
    {
      Serial.print("Heart Rate: ");
      Serial.println(beatsPerMinute);
      display.clearDisplay();
      display.setCursor(0,0);
      display.println(F("Heart Rate:"));
      display.setCursor(0, 10);
      display.println(beatsPerMinute);
      display.display();
      digitalWrite(ledPin, HIGH); // Turn on LED when a heartbeat is detected
      delay(100);
      digitalWrite(ledPin, LOW); // Turn off LED
    }
  }
}

boolean checkForBeat(int32_t sample)
{
  static int32_t lastBeat = 0;
  static int32_t peak = 0;
  static int32_t trough = 0;
  static int32_t threshold = 100; // Adjust this threshold if needed
 
  boolean beatDetected = false;
 
  if (sample < threshold)
  {
    trough = sample;
  }
 
  if (sample > threshold && sample > peak)
  {
    peak = sample;
  }
 
  if (sample > threshold && sample < peak)
  {
    if ((peak - trough) > 20) // Heartbeat detected
    {
      lastBeat = millis();
      beatDetected = true;
    }
  }
 
  return beatDetected;
}
