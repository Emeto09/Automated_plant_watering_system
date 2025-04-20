// Automated_plant_watering_system
#include <math.h>

// === Pin Configuration ===
const int pumpPin = 7;            // Digital pin controlling the pump
const int moisturePin = A0;       // Analog pin connected to the soil moisture sensor
const int tempPin = A1;           // Analog pin connected to NTC thermistor voltage divider

// === Soil Moisture Calibration ===
const int drySoil = 700;          // Sensor reading when soil is completely dry (calibrate)
const int wetSoil = 300;          // Sensor reading when soil is fully wet (calibrate)
const int moistureThreshold = 550;  // Moisture level to trigger watering
const int hysteresis = 50;        // Hysteresis to avoid frequent on/off switching

// === Temperature Sensor (NTC Thermistor) Configuration ===
const int seriesResistor = 10000;     // 10kΩ resistor in series with thermistor
const int thermistorNominal = 10000;  // 10kΩ at 25°C
const int betaCoefficient = 3950;     // Beta value from thermistor datasheet
const int temperatureThreshold = 23;  // Minimum temperature in °C to water sunflowers

bool pumpState = false;  // Track pump state to avoid redundant switching

void setup() {
  pinMode(pumpPin, OUTPUT);        // Set the pump pin as output
  Serial.begin(9600);              // Begin serial communication for debugging
}

void loop() {
  // === Read Soil Moisture Sensor ===
  int moistureRaw = analogRead(moisturePin); // Read raw analog value
  int moisturePercent = map(moistureRaw, drySoil, wetSoil, 0, 100); // Map to % scale
  moisturePercent = constrain(moisturePercent, 0, 100); // Clamp within 0–100%

  // === Read Temperature from Thermistor ===
  int adcValue = analogRead(tempPin);                  // Read analog voltage
  float voltage = adcValue * (5.0 / 1023.0);           // Convert ADC to voltage
  float resistance = (5.0 - voltage) * seriesResistor / voltage; // Calculate thermistor resistance

  // Apply Steinhart-Hart equation to estimate temperature in Celsius
  float steinhart = resistance / thermistorNominal;    
  steinhart = log(steinhart);                          
  steinhart /= betaCoefficient;                        
  steinhart += 1.0 / (25 + 273.15);                     // 25°C reference in Kelvin
  steinhart = 1.0 / steinhart;                         
  float temperatureC = steinhart - 273.15;              // Convert to Celsius

  // === Serial Output for Debugging ===
  Serial.print("Soil Moisture: ");
  Serial.print(moisturePercent);
  Serial.print("% (Raw: ");
  Serial.print(moistureRaw);
  Serial.println(")");

  Serial.print("Temperature: ");
  Serial.print(temperatureC, 1);
  Serial.println(" °C");

  // === Pump Control Logic ===
  if (moistureRaw > (moistureThreshold - hysteresis) && temperatureC > temperatureThreshold && !pumpState) {
    analogWrite(pumpPin, 128);  // Turn ON pump
    pumpState = true;
    Serial.println("Pump ON: Soil dry and temperature suitable for watering.");
  }
  else if ((moistureRaw < (moistureThreshold + hysteresis) || temperatureC <= temperatureThreshold) && pumpState) {
    analogWrite(pumpPin, 0);   // Turn OFF pump
    pumpState = false;
    Serial.println("Pump OFF: Soil moist enough or temperature too low.");
  }

  delay(100); // Wait 0.1 seconds before next reading
}
