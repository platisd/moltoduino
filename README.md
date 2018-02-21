# Moltoduino
Add programmable cores to your project and enable Hardware In the Loop (HIL) testing.

![Stacked Moltoduinos](https://i.imgur.com/IfSTcNG.jpg)

## What?
Moltoduino is an Arduino shield that enables the stacking of multiple ATMega328P microcontrollers. The microcontroller pins are broken out in two ways:
* Outwards, to enable their arbitrary connection to different components or other pins via cables.
* Inwards, to enable their connection to the respective pin of the bottommost Arduino via jumpers.

The firmware on the Moltoduino cores can be easily flashed by flipping a switch that sets them in programming mode and then using the bottommost Arduino as an ISP programmer.

## Why?
Moltoduino is essentially a barebones Arduino UNO in the form of a shield that can be easily reprogrammed. So, what can you do with it?

* Parallelize time-consuming or time-critical functionality (e.g. reading sensor data, playing sound)
* Have a dedicated microcontroller for computationally intensive operations (e.g. Fast Fourier Transform)
* Enable Hardware In the Loop (HIL) testing
* Get additional resources, such as I/O pins, external interrupts, Serial port, I2C bus etc
* Save physical space needed for applications that require a more than one microcontrollers (e.g. by stacking them instead placing them next to each other)

## How?
Moltoduino was initially created with the purpose of providing a stackable pin extension solution for Arduino boards. During development, another use case emerged which involved the shield being used for HIL testing.

The picture below illustrates how the pins of the shield's ATMega328P are broken out, highlighting them with **green**. In **red** one can find the bottom Arduino's pins. It becomes apparent that pins that follow the Arduino numbering convention can be easily connected via jumpers (e.g. D1 with D1) regardless of which specific Arduino board is in place (i.e. Mega or Uno). Furthermore, the shield's pins are also broken on the sides of the board so to be accessible when multiple Moltoduinos or other shields are stacked on top.

![Moltoduino pin grouping](https://i.imgur.com/vwzHevR.jpg)

### Pin extension
Attaching a Moltoduino shield is more or less equivalent to placing an Arduino Uno on top of another Uno or Mega board. The cool part is that it doesn't take up any horizontal real estate. You program them to perform an individual task in parallel or connect the Arduinos together via Serial, I2C or otherwise to implement complex functionality.

The pins facing outward can interface with the external components while the inward facilitate connection to the bottom Arduino, especially via I2C or Serial.

### HIL testing
Moltoduino can be utilized to provide a _hobby-grade_ [HIL testing](http://www.hil-simulation.com/home/hil-testing.html) solution for your embedded project. Having two microcontrollers that are easily connected to each other it is possible to conduct HIL simulations. This is achieved by one microcontroller running the production code and the other running the HIL test which generates input for the system under test and reads its output.

Having a HIL test fixture allows you to automate system-level testing on your project. This can be a difficult and laborious process, where the tester would have to manually provide the environment input and check the output so to verify the system.

### Programming Moltoduino
1. Have all switches (there is one on each Moltoduino), in the "operation" position instead of "programming" mode
2. In the Arduino IDE, upload the ArduinoISP sketch to the "bottom" Arduino
3. Open the sketch to upload to the Nth Moltoduino on the Arduino IDE
4. Make sure that "Arduino as ISP" is chosen under Tools :arrow_right: Programmer
5. Choose the Arduino Uno as the target board, under Tools :arrow_right: Board :arrow_right: Arduino/Genuino UNO
6. Put the switch in the "programming" position, on the Moltoduino(s) you wish to program
7. (If not previously done) Burn the bootloader, under Tools :arrow_right: Burn bootloader
8. Upload sketch using programmer, under Sketch :arrow_right: Upload sketch using programmer
9. Put the switch back in the "operation" position, otherwise uploading to the bottom Arduino will not be possible

## Code examples
For the code examples the [Smartcar shield](http://plat.is/smartcar) library will be used. You can easily download it via your Arduino IDE library manager.

### Controlling a Smartcar via UART
In this scenario, a [Smartcar](http://plat.is/smartcar) is being controlled via UART (e.g. an HC-06 Bluetooth dongle) and we want to verify its behavior. The HIL simulation will be running on an Arduino Mega while the system under test firmware will be uploaded onto a Moltoduino that is stacked on top.

**System under test**

The following code should be straight forward. Depending on the UART input the Smartcar should change speed and direction. We are steering the Smartcar with a servo motor while the speed is controlled by a brushed motor. We will be sending commands via UART and reading the PWM steering signal as well as the brushed motor control signals in order to verify the Smartcar's intended behavior.

```cpp
#include <Smartcar.h>

const int SERVO_PIN = 3;
const int FORWARD_PIN = 8;
const int BACKWARD_PIN = 7;
const int THROTTLE_PIN = 6;
const int fSpeed = 70; //70% of the full speed forward
const int bSpeed = -70; //70% of the full speed backward
const int lDegrees = -50; //degrees to turn left
const int rDegrees = 50; //degrees to turn right

//initialize the car, that uses a servo motor for steering
//and a brushed DC motor to pins 8,7 (direction) and 6 (PWM) for throttling
Car car(useServo(SERVO_PIN), useDCMotor(FORWARD_PIN, BACKWARD_PIN, THROTTLE_PIN));

void setup() {
  Serial.begin(9600);
  car.begin(); //initialize the car using the encoders and the gyro
}

void loop() {
  handleInput();
}

void handleInput() { //handle serial input if there is any
  if (Serial.available()) {
    char input = Serial.read();
    switch (input) {
      case 'l': //turn counter-clockwise going forward
        car.setSpeed(fSpeed);
        car.setAngle(lDegrees);
        break;
      case 'r': //turn clock-wise
        car.setSpeed(fSpeed);
        car.setAngle(rDegrees);
        break;
      case 'f': //go ahead
        car.setSpeed(fSpeed);
        car.setAngle(0);
        break;
      case 'b': //go back
        car.setSpeed(bSpeed);
        car.setAngle(0);
        break;
      default: //if you receive something that you don't know, just stop
        car.setSpeed(0);
        car.setAngle(0);
    }
  }
}
```

**HIL simulation**

In this HIL test we need to simulate the input to the system (i.e. UART commands) and then observe its output (i.e. pin states and PWM signal) to verify production code's correct behavior. Moreover, jumpers are used to enable interaction between the system under test and the HIL simulation. The jumper placement is illustrated in the table below.

| HIL simulation | System Under Test | Purpose                           |
| :----:         |:----:             |:----:                             |
| TX             |  RX               | Transmit UART commands to the SUT |
| 3              |  3                | Read servo motor's PWM signal     |
| 7              |  7                | Read motor control pin            |
| 8              |  8                | Read motor control pin            |

```cpp
// Simulation pin configuration
const int STEERING_PIN = 3;
const int FORWARD_PIN = 8; // Car goes forward when set HIGH
const int BACKWARD_PIN = 7; // Car goes backward when set HIGH
const int THROTTLE_PIN = 6;
// UART commands
const char CAR_FORWARD = 'f';
const char CAR_BACKWARD = 'b';
const char CAR_LEFT = 'l';
const char CAR_RIGHT = 'r';
const char CAR_STOP = 's';
// Expected steering degrees
const int LEFT = -50;
const int RIGHT = 50;
const int STRAIGHT = 0;
// Other variables and structures
enum Throttle {
  FORWARD,
  BACKWARD,
  STOPPED,
};

bool isThrottle(Throttle expectedThrottle) {
  Throttle actualThrottle = STOPPED;
  if (digitalRead(FORWARD_PIN) && !digitalRead(BACKWARD_PIN)) {
    actualThrottle = FORWARD;
  } else if (!digitalRead(FORWARD_PIN) && digitalRead(BACKWARD_PIN)) {
    actualThrottle = BACKWARD;
  }

  return actualThrottle == expectedThrottle;
}

bool isSteering(int expectedAngle) {
  int pwm = pulseIn(STEERING_PIN, HIGH);
  // Map the pwm singal to a 0 to 180 scale
  int measuredAngle = map(pwm, 540, 2390, 0, 180);
  // Offset the angle by 90 to get an angle between -90 and 90
  measuredAngle -= 90;
  int absoluteDelta = expectedAngle > measuredAngle ?
                      expectedAngle - measuredAngle : measuredAngle - expectedAngle;
  // We are expecting some error in these measurements so let's define an acceptable error margin
  const int ERROR_MARGIN = 5;

  return absoluteDelta < ERROR_MARGIN;
}

void runSmartcarHIL(const char* testName, const char command, Throttle throttle, const int steering) {
  Serial.print("RUNNING: ");
  Serial.println(testName);
  // Send command
  Serial.print("Sending command: ");
  Serial.print(command);
  Serial.flush(); // Make sure we have finished sending
  // Wait a bit to make sure the command has been processed
  const unsigned long TEST_DELAY = 100;
  delay(TEST_DELAY);
  String result = isThrottle(throttle) && isSteering(steering) ? "PASSED" : "FAILED";
  Serial.print("\nRESULT: ");
  Serial.println(result);
  Serial.println("----");
}

bool goForward_test() {
  auto UARTcommand = CAR_FORWARD;
  auto expectedThrottle = FORWARD;
  auto expectedSteering = STRAIGHT;
  runSmartcarHIL(__func__, UARTcommand, expectedThrottle, expectedSteering);
}

bool goBackward_test() {
  auto UARTcommand = CAR_BACKWARD;
  auto expectedThrottle = BACKWARD;
  auto expectedSteering = STRAIGHT;
  runSmartcarHIL(__func__, UARTcommand, expectedThrottle, expectedSteering);
}

bool turnLeft_test() {
  auto UARTcommand = CAR_LEFT;
  auto expectedThrottle = FORWARD;
  auto expectedSteering = LEFT;
  runSmartcarHIL(__func__, UARTcommand, expectedThrottle, expectedSteering);
}

bool turnRight_test() {
  auto UARTcommand = CAR_RIGHT;
  auto expectedThrottle = FORWARD;
  auto expectedSteering = RIGHT;
  runSmartcarHIL(__func__, UARTcommand, expectedThrottle, expectedSteering);
}

bool stop_test() {
  auto UARTcommand = CAR_STOP;
  auto expectedThrottle = STOPPED;
  auto expectedSteering = STRAIGHT;
  runSmartcarHIL(__func__, UARTcommand, expectedThrottle, expectedSteering);
}

void setup() {
  pinMode(FORWARD_PIN, INPUT);
  pinMode(BACKWARD_PIN, INPUT);
  pinMode(STEERING_PIN, INPUT);
  Serial.begin(9600);
  Serial.println("====================");
  Serial.println("Starting HIL test suite");
  Serial.println("====================");

  goForward_test();
  goBackward_test();
  turnLeft_test();
  turnRight_test();
  stop_test();
}

void loop() {
}
```

## Bill Of Materials
| Quantity | Value             | Device                                    | Package                   | SKU       |
|----------|-------------------|-------------------------------------------|---------------------------|-----------|
| 1        | BC547             | TRANSISTOR_NPNBC547                       | TO-92-AMMO                | 305010028 |
| 1        | 100µF/16V  5X7    | DIP-CAP-ALUMINUM-47UF-16V(D5-H9MM)        | PC-D5.0MM                 | 302030033 |
| 1        | 16MHZ             | DIP-CRYSTAL-16MHZ-18PF-30PPM-50R(HC49US)  | HC49US                    | 306010037 |
| 1        | 1K                | DIP-RES-1K-5%-1/4W(PR-D2.3XL6.5MM)        | PR-D2.3XL6.5MM            | 301020000 |
| 2        | 22pf              | DIP-CERAMIC-DISC-22PF-50V-10%-NPO(D4.0MM) | CERAMIC-2.54              | 302010200 |
| 1        | 2x16p-2.54        | DIP-BLACK-MALE-HEADER-VERT(2X16P-2.54)    | H2X16-2.54                | 320020049 |
| 1        | 2x6p-2.54         | DIP-BLACK-MALE-HEADER(2X6P-2.54)          | H2X6-2.54                 | 320020001 |
| 1        | 3P-2.54-90D       | DIP-BLACK-FEMALE-HEADER(3P-2.54-90D)      | H3-2.54-90D-FEMALE        | 320030100 |
| 1        | 4.7K              | DIP-RES-4.7K-5%-1/4W(PR-D2.3XL6.5MM)      | PR-D2.3XL6.5MM            | 301020012 |
| 3        | 8p-2.54-90d       | DIP-BLACK-FEMALE-HEADER-R/A(8P-2.54-90D)  | H8-2.54-90D-FEMALE        | 320030076 |
| 1        | 74HC4066          | 74HC4066(DIP14)                           | DIP14-2.54-17.24X6.35MM   |           |
| 1        | Moltoduino shield | MOLTODUINO_SHIELD_PCB                     | ARDUINOR3_ICSP            |           |
| 1        | ATMEGA328P_PDIP   | ATMEGA328P_PDIP                           | DIL28-3                   | 310010051 |
| 1        | SK-12D01          | DIP-TOGGLE-SWITCH-ON-ON(3+2P-8.8X4.6MM)   | SW5-2.0-8.8X4.4X4.7MM-90D | 311030008 |
| 1        | TS-1101F          | DIP-BUTTON-FRONT-WHITE(2P-3.4X6MM)        | SW2-3.4X6.0X3.55MM-90D    | 311020025 |
| 1        | 10p-2.54          | F185-1110A1BSYA1                          | H10-2.54                  | 320030039 |
| 1        | 6p-2.54           | F185-1106A1BSYA2                          | H6-2.54                   | 320030012 |
| 2        | 8p-2.54           | F185-2108A1BSYA1                          | H8-2.54                   | 320030013 |
| 1        | F185-1206A0ASYC1  | F185-1206A0ASYC1                          | H2X3-2.54                 | 320030052 |
