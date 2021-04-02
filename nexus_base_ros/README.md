# ROS wheelbase controller for the NEXUS Omni 4-Wheeled Mecanum Robot

This branch named *noetic* is currently under development for ROS Noetic. The project is now in the testing phase. The build seems to work fine so far.

![4WD Mecanum Wheel Mobile Arduino based Robotics Car](4WD_Mecanum_Wheel_Robotics_Car.jpg  "4WD Mecanum Wheel Mobile Arduino based Robotics Car")

## Required hardware
This wheel controller has been developed for the 4WD Mecanum Wheel Mobile Arduino Robotics Car 10011 from [NEXUS ROBOT](https://www.nexusrobot.com/product/4wd-mecanum-wheel-mobile-arduino-robotics-car-10011.html).

For teleoperation of the Nexus wheelbase a (wireless) game pad like the Logitech F710 is recommended. The base controller runs on a PC or a Raspberry Pi 3B with ROS [Noetic](http://wiki.ros.org/noetic/Installation/Ubuntu) (ROS-Base) installed. To install  ROS Noetic on a Rasberry Pi, it is recommended to start with [Ubuntu Server 20.04 for ARM](https://ubuntu.com/download/server/arm). Preferably install a lightweight desktop and a remote desktop server like X2GO.

Logitech F710 game pad | ROS Noetic
------------- | -----------
![Logitech F710](Logitech_F710.jpg  "Logitech F710") | ![ROS Noetic](Noetic.png  "ROS Noetic")

## Installing ROS and ROS dependencies
This project has been tested with *ROS Noetic*. The complete receipt to install *ROS Noetic* and the required ROS packages:

``` 
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-late>
sudo apt-key adv --keyserver 'hkp://keyserver.ubuntu.com:80' --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654
sudo apt update
sudo apt install ros-noetic-ros-base
sudo apt-get install ros-noetic-joy
sudo apt-get install ros-noetic-rosserial-arduino
sudo apt-get install ros-noetic-rosserial

// Add path in .bashrc
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
source ~/.bashrc

sudo apt install python3-rosdep python3-rosinstall python3-rosinstall-generator python3-wstool build-essential

sudo apt install python3-pip
sudo pip3 install -U catkin_tools

sudo rosdep init
rosdep update

//create a catkin workspace, for example named "ros_catkin_ws":
mkdir ~/ros_catkin_ws/src
cd ~/ros_catkin_ws /src
catkin_init_workspace
cd ~/ros_catkin_ws
catkin_make
```

Clone the *nexus-base-ros* package in the *ros_catkin_ws/src* directory (branch *noetic*).

`git clone https://github.com/MartinStokroos/nexus_base_ros.git`

The project uses an external C++ library. Clone the required PID_Library module in the `nexus_base_ros/lib` directory:

 `git submodule add https://github.com/MartinStokroos/PID_Controller.git`

## Building the package

Run `catkin_make`from the root of the workspace.

## Generating the ros_lib for the wheelbase Arduino
It is possible to compile the Arduino firmware with the catkin make proces without making use of the Arduino IDE. However, now it is done with the Arduino IDE.
The Arduino IDE should be installed and the `Arduino/libraries` directory must exist in your home folder.
Generate the *ros_lib* with the header files of the custom message types used in this project.

1. cd into `Arduino/libraries`
2. type: `rosrun rosserial_arduino make_libraries.py .`

In the newly made `ros_lib` , edit `ros.h` and reduce the number of buffers and downsize the buffer sizes to save some memory space:

```
#elif defined(__AVR_ATmega328P__)

  //typedef NodeHandle_<ArduinoHardware, 25, 25, 280, 280> NodeHandle;
  typedef NodeHandle_<ArduinoHardware, 8, 8, 128, 128> NodeHandle;
```
The Arduino firmware does not necessary have to be compiled on the Pi. The wheelbase firmware can be compiled and flashed onto the Arduino from another ROS PC workstation and then connected to the Pi.

## Flashing the firmware into the wheelbase Arduino board of the 10011 platform
1. download and include the *digitalWriteFast* library from: [digitalwritefast](https://code.google.com/archive/p/digitalwritefast/downloads)

2. clone and install the *PinChangeInt* library:

   `git clone https://github.com/MartinStokroos/PinChangeInt`

Program the *Nexus_Omni4WD_Rosserial.ino* sketch from the firmware folder into the Arduino  Duemilanove-328 based controller board from the 10011 platform.

**NOTE:** The Sonars cannot be used simultaneously with the serial interface and must be disconnected permanently!

## Description of the nodes
The picture below shows the rosgraph output from `rqt_graph`:

![rosgraph](rosgraph.png  "rosgraph")

* Topics

The *nexus_base_controller* node runs on the platform computer and listens to the *cmd_vel* topic and uses the x-, y and angular velocity inputs. The *cmd_vel* is from the geometry_msg Twist  type.
The *nexus_base_controller* node publishes odometry data based on the wheel encoder data, with the topic name *sensor_odom*.

*nexus_base* is a rosserial type node running on the wheelbase controller board. The code for this node can be found in the firmware directory. *nexus_base* listens to the *cmd_motor* topic and publishes the *wheel_vel* topic at a constant rate of 20Hz (default).
The data structure of the *cmd_motor* topic is a 4-dim. array of *Int16*'s. The usable range is from -255 to 255 and it represents the duty-cycle of the PWM signals to the motors. Negative values  reverses the direction of rotation.
The *wheel_vel* topic data format is also a 4-dim. array of *Int16*'s.

*wheel_vel* is the raw wheel speed and that is the number of encoder increments/decrements per sample interval (0.05s default).
Three seconds after receiving a speed (x,y and angular) of '0' the motors are set idle to save power by stopping the PWM.

*/joy* is the standard message topic from the joystick node. The *teleop_joy* node does the unit scaling and publishes the *cmd_vel* topic. It also contains the service clients of the ROS services *EmergencyStopEnable* and *ArmingEnable*.

* Services

Node *nexus_base* runs two ROS service servers. The first service is called *EmergencyStopEnable* and enables the emergency stop and the second service is called *ArmingEnable* and this service is to rearm the system.

* Block diagram of the Nexus base controller

The block diagram shows the internal structure of the *nexus_base_controller* node.

![Nexus base controller](base_controller_block_diagram.png  "Nexus base controller")

## Launching the example project
The tele-operation demo can be launched after connecting a game pad (tested with *Logitech F710*) and the wheel base USB-interface with the platform computer (small form factor PC or Raspberry Pi 3B). Check if the game pad is present by typing:

`ls -l /dev/input/js0`

Check if the wheel base USB interface is present: 

`ls -l /dev/ttyUSB0`

Launch:

`rosrun nexus_base_ros nexus_teleop_joy `

The left joystick handle steers the angular velocity when moved from left to right. The right joystick controls the x- and y-speed.
The red button enables the emergency stop. The green button is to rearm the robot after an emergency stop.

Some diagnostics:

```
$ rostopic list
/cmd_motor
/cmd_vel
/diagnostics
/joy
/joy/set_feedback
/rosout
/rosout_agg
/sensor_odom
/wheel_vel

$ rosservice list
/arming_enable
/base_controller/get_loggers
/base_controller/set_logger_level
/emergency_stop_enable
/joystick/get_loggers
/joystick/set_logger_level
/nexus_base/get_loggers
/nexus_base/set_logger_level
/rosout/get_loggers
/rosout/set_logger_level
/teleop_joy/get_loggers
/teleop_joy/set_logger_level

$ rostopic hz /wheel_vel 
subscribed to [/wheel_vel]
average rate: 20.010
	min: 0.047s max: 0.064s std dev: 0.00505s window: 40
```
## Auto-start from boot
The script to bring up the robot when booting the PC/RPi can be generated with the *robot_upstart* package.

`sudo apt-get install ros-melodic-robot-upstart`

Create the install script(s) (run from catkin workspace):

`rosrun robot_upstart install nexus_base_ros/launch/nexus_teleop_joy.launch`

`sudo systemctl daemon-reload && sudo systemctl start nexus`

One last thing to do, is giving permission for using the serial port at boot. 

```
cd /etc/udev/rules.d
sudo touch local.rules
```

edit local.rules

```
ACTION=="add", KERNEL=="dialout", MODE="0666"
ACTION=="add", KERNEL=="js0", MODE="0666"	//to be confirmed if really needed.
ACTION=="add", KERNEL=="ttyUSB0", MODE="0666"
```

The script can be enabled/disabled by:

`sudo systemctl nexus start`

`sudo system ctl nexus stop`

If needed, check the upstart log with:

`sudo journalctl -u nexus`

Uninstalling the script can be done with:

`rosrun robot_upstart uninstall nexus`

## Known issues
- ROS service calls do not work with rosserial 0.8.0. ROS Melodic comes with rosserial version 0.8.0. Rosserial 0.7.7 works. The workaround is:
Download [rosserial 0.7.7](https://repology.org/project/rosserial/packages) . Unzip and copy the following module from the rosserial-0.7.7 package into the catkin *src* directory and rebuild the project:

`rosserial_python` 

- Sometimes the wheels do not respond for a short moment when the robot is rearmed after an emergency stop.

