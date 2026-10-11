# CycloidalActuator

# Overview
This actuator project is essentially a proving ground to see if I could build and implement technology used in industrial robotics in a hobby setting. The control hardware is designed to work with a range of brushless DC motors including gimbal motors and stepper motors, controlling them with a FOC (Field Oriented Control) algorithm to get maximum performance. It also implements a 12:1 cycloidal gearbox to reduce the output to a more usable speed while increasing available torque. The drive implements CANBus communication to allow multiple drives to be controlled with a 3 wire (including ground) robust data network that because of its differential signaling is resistant to interference, similar to how USB works. Overall I would call this prototype a success, but there are definitely some things I would like to fix and improve upon in future designs.

<br>

# Writeup

# Cycloidal Gearbox
The first version had its outer pins 3d printed into the inner surface of the gearbox casing, which led to a lot of friction and requiring significant amounts of grease. It also used an eccentric shaft pressed directly onto the motor shaft, which required that I get the motor very close to the gearbox so that the shaft could reach. One of the goals for the next version was to find a way to allow the motor shaft to be further back from the actual disks so that I had more space for rear bearing supports.

<br>

For my second revision, I decided to use ball bearing pins instead to hopefully reduce friction losses in the gearbox. The design eventually ended up using 56 ball bearings in the pins, because the 12:1 gearbox requires 13 pins, each with 4 bearings stacked on an M3 screw to achieve the 10mm bearing stack thickness. The bearings definitely helped with reducing friction, (especially with the spacer to let them spin easier). One of the biggest issues I ran into in V2 was eccentricity in the gearbox, which wasn't an issue in V1 since the input cams were mounted directly to the stepper motor's output shaft. In V2, the whole gearbox got longer to accomodate the screws that held the bearing pins in place, so the motor shaft was no longer long enough to connect all the drive cams. Instead, I bolted the cams and a 3D printed stepper motor attachment together with M2 screws to act as the main input shaft. However, this new design introduced the possibility that the input shaft was no longer perfectly aligned with the motor shaft, causing a periodic resistance in the gearbox rotation. Also, changing the discs in this version of the gearbox is much more difficult since it requires disassembling a large portion of the gearbox, so I was unable to properly tune the clearances of the discs inside the gearbox. However, with some tuning and more intentional usage of pins to align gearbox components, I think the gearbox could get far smoother. See images in the slideshow below depicting inner parts and assembly.

<br>

# Electronics

The original version was controlled by an ESP32 XIAO and a A4988 stepper driver, with an AS5600 I2C angle sensor for angle feedback. The motor control was done with the AccelStepper library and some custom code that I wrote to provide position feedback. The motor was just driven with regular open loop stepper control, and the relative setpoint was changed based on measured error. The goals for the next electronics prototype are to create a custom FOC control board that can both support BLDC gimbal motors and stepper motors with more precise FOC control which should give better torque control at lower speeds. 

<br>

The second version of the electronics included a custom PCB to control the stepper motor, and it has a STM32G431 microcontroller to run SimpleFOC, the library I use to do the FOC math and actually control the motor. To physically switch the electricity to the motor on and off, it features a DRV8316 triple half-H-bridge motor driver chip with integrated MOSFETS (electrical switches). The board also has a MT6816 14 bit magnetic angle sensor for sensing the angle of the motor (and its magnetic field) and allowing the FOC algorithm to properly calculate the required control outputs. The other chip on the board is a 1Mbps CAN transciever that allows the STM32 to communicate with other boards on CANBus. To get 3.3V for the STM32, the board uses the DRV8316's integrated buck converter to regulate its 20-26v input down to logic level without wasting too much as heat. The board also includes connectors to connect power, CANBus, and debug protocols like SWD and UART, used for programming and configuration.

<br>

Overall, the first version of the board performed well, with some minor issues that need to get fixed in later versions. One of the main issues was that the input power connector's footprint was backward, so to avoid breaking the board I had to wire the power backwards and connect 24v to the negative terminal and GND to the positive terminal. The other mistake is that the current sense input on the DRV8316 was not connected, so the current sense functionality of the board did not work. It's not the end of the world, since I can instead just approximate current using a mathematical model based on other motor parameters but I am losing efficiency and performance compared to proper current sensing. Other than that, the board works well, it is a 4 layer board with a power and ground plane each in the middle and signal planes on the outside. I ordered the board from JLCPCB and the components from LCSC, although with the amount of tarriffs I paid it probably would have been better to order from a US distributor like DigiKey, although they do not stock MagnaTek's angle sensors which are some of the more cost effective options.

<br>

I assembled the PCB using solder paste and a stencil that I also got from JLC, which made it easy to precisely place all the tiny components and also not put too much solder on the pads of the board. Then, I placed the board on a temperature controlled hot plate to melt all the solder at once and secure the parts. The components on the back I hand soldered, though I should have used solder paste and hot air for the angle sensor as that would have helped with its alignment on the PCB. After assembly, I connected the board to an LED on a breadboard and uploaded some simple test code using a Raspberry Pi PicoProbe USB debugging tool to see if the board booted, and it did!

<br>

# Software Development
However, the harder part was the code, which I'll admit I used a lot of AI (both Claude and Codex) to write and implement. The core of the board's function is the <a href="simplefoc.com">simpleFOC library</a> which handles all of the math that computes the PWM duty cycles (voltages) on each motor phase (wire) to create the correct magnetic field to move the motor in the desired way. To achieve that, it uses continous feedback from the angle sensor on the PCB, as well as optionally current sense logic on boards where it works. Additionally, the code handles the CANBus communication with other boards using the simpleCANio library by simpleFOC, which handles interfacing with the hardware CAN peripheral on the STM32 and other microcontrollers like the ESP32.

<br>

The first step was calibration of all the hardware, which I did with a bunch of calibration code that measured multiple things like the kV of the motor (for current estimation) and the offset of the angle sensor (because of imperfect centering) that I could include in the code afterward for better functionality. It turns out there was significant angular offset of the angle sensor that could have been solved with better mounting and soldering of the chip on the board, but most of it was calibrated out using the CalibratedSensor class in simpleFOC.

<br>

The next step I did was get the board connected over CANBus to the CAN controller I built to demonstrate its functionality (documented below) which wasn't that bad using the simpleCANio library and a can message spec written with the help of Claude. CANBus uses 11 bit message ID's and 8 byte message payloads, with message priority based on the numerical value of the ID, lower numbers first. So, dedicating 6 bits to message type and 5 bits to node ID, I can order it so that the message type affects priority more. This is because the message type is the 6 most significant bits, so any command with any node ID will always have a value in between the current command with node 0 and the next command with node 0. (1xx is always between 100 and 200, for example)

<br>

There were a few bugs to work out though, the main one being that the node would drop off the bus and go offline whenever the enable signal was sent over CANBus. The reason for this issue was that since the STM32G431 is a single core processor, the simpleFOC math (that has to run quickly) was put on an interrupt timer, where the cpu would drop everything it was doing every time the timer went off and do the FOC math instead, then go back to what it was doing. The issue arose from the fact that the timer frequency was set too high (20kHz) so when simpleFOC was enabled and it started doing the math, the cpu didn't have time to do anything else and so it looked like it was offline. The fix was easy, just set it to 2kHz instead (which was still plenty of frequency) which allowed the cpu to have time to do other things like communicate on the CANBus.

# CANBus Tester
The final piece of the puzzle was the CANBus controller, which I built by hand soldering together an Adafruit QT PY ESP32-S3, an Adafruit CAN Pal for CANBus, an SSD1306 128x64 OLED display, as well as a button and a rotary encoder to control its menus, all integrated with two breadboard-like protoboards, laser cut plates, and M2 standoffs and screws. It uses USB power and connects via the CANBus connector on the motor driver PCB. Again, with the help of Codex, I wrote code that followed the same CAN spec as the driver and could send various commands over the CAN connection to the driver as well as read back status codes and errors.

<br>

# Source Files
CAD Design Files are in this repo in /src/ folder, 
Onshape CAD View: <a href="https://cad.onshape.com/documents/33648b7e54c4d4e2de7ace25/w/9aa3ebcb439778296254b397/e/8d79e07f044a38655a9d68e0?renderMode=0&uiState=6acadacc88189f0bb211b0ae">V1,</a>  <a href="https://cad.onshape.com/documents/a6bee5f61b98d1e1cb7f04f8/w/4900588bdb5d1e46219c141d/e/e800cc2db78500788dc58b32?renderMode=0&uiState=6acad9d343eb9e25a4ee513d">V2</a>
<br>
PCB Design Files: https://github.com/chairlesengi/MotorCAN
<br>
Driver Code (mostly for integrated test demo, not production ready and heavy AI use): https://github.com/chairlesengi/STMDRIVERCIRCLE
<br>
CANBus Tester Code (also mostly for integrated test, heavy AI use): https://github.com/chairlesengi/ESPMTRCONTROLLER

