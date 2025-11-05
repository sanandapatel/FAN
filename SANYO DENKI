// --- Sanyo Denki 9G0612G101: Detect Each Revolution & Measure Period ---
// Wiring: Yellow -> D2 (with pull-up to 5V or use INPUT_PULLUP), Black -> GND, Red -> +12V
// Note: Do NOT power the fan from the Arduino. Use common ground between fan and Arduino.

const byte fanTachPin = 2;           // interrupt-capable pin on UNO/Nano
volatile byte pulseCount = 0;        // counts tach pulses (2 pulses = 1 rev)
volatile unsigned long turnCount = 0;
volatile unsigned long lastTurnMicros = 0;        // timestamp of last revolution (micros)
volatile unsigned long lastTurnIntervalMicros = 0; // period between last two revolutions
volatile bool turnFlag = false;      // set by ISR to signal a new revolution

// ISR: very small — update counters/timestamps and set flag
void countPulse() {
  pulseCount++;
  if (pulseCount >= 2) {             // 2 pulses = 1 revolution
    unsigned long now = micros();
    // compute interval since last revolution (if first revolution, lastTurnMicros==0)
    if (lastTurnMicros != 0) {
      lastTurnIntervalMicros = now - lastTurnMicros;
    } else {
      lastTurnIntervalMicros = 0;
    }
    lastTurnMicros = now;
    turnCount++;
    pulseCount = 0;
    turnFlag = true;
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(fanTachPin, INPUT_PULLUP); // use internal pull-up (or use external pull-up to 5V)
  attachInterrupt(digitalPinToInterrupt(fanTachPin), countPulse, FALLING);
  Serial.println("Fan revolution detector started");
}

void loop() {
  // If ISR signalled a new revolution, copy volatile data atomically and print.
  if (turnFlag) {
    noInterrupts();
    unsigned long localTurnCount = turnCount;
    unsigned long localInterval = lastTurnIntervalMicros;
    turnFlag = false;
    interrupts();

    Serial.print("Turn #: ");
    Serial.print(localTurnCount);

    if (localInterval > 0) {
      float ms = localInterval / 1000.0;
      float rpm = 60000000.0 / (float)localInterval; // 60,000,000 microseconds per minute
      Serial.print("  |  Period: ");
      Serial.print(ms, 2);
      Serial.print(" ms");
      Serial.print("  |  RPM: ");
      Serial.print(rpm, 1);
    } else {
      Serial.print("  |  (first revolution, interval unknown)");
    }
    Serial.println();
  }

  // Optional: do other tasks here. Avoid long blocking delays that could cause missed pulses.
}
