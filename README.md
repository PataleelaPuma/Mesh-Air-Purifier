# Mesh-Air-Purifier
The Mesh Air Purifier is a smart air cleaning system that keeps indoor spaces healthier and more comfortable by using multiple air purifiers in the same room. Air purifiers and PM2.5 sensors are placed in different spots and work together as a team through Home Assistant.

### **Introduction**
The Mesh Air Purifier is a smart air cleaning system that keeps indoor spaces healthier and more comfortable. It solves this problem by using multiple air purifiers in the same room. Air purifiers and PM2.5 sensors are placed in different spots and work together as a team. These sensors send their readings to a smart controller in the system. The smart controller looks at all the information and determines which areas need more cleaning. It then tells the purifiers when to turn on or off, depending on how bad the air is in each part of the room. This way, the system uses less power and ensures the air is clean everywhere, not just in one spot. The Mesh Air Purifier is a great solution for homes, classrooms, and offices where clean air is important.

### **Objectives**
The objective of the Mesh Air Purifier is to improve indoor air quality smartly and efficiently, help save energy, and make sure that clean air is spread evenly throughout the space. 

### **Hardware Components**
_Air Purifier:_
- Sonoff
- AC Fan

_PM2.5 Sensor:_
- VINDRIKTNING (PM2.5 Sensor)
- ESP32

_Home Assistant:_
- Raspberry Pi 4

### **Overall System Diagram**

![Overall System Diagram](https://github.com/PataleelaPuma/Mesh-Air-Purifier/blob/main/Pictures/Air%20purifier%20Overall%20Diagram.jpg?raw=true)

### **Mesh Logic and Air Quality-Based Fan Control** 

We designed a mesh-based logic model that considers PM2.5 values from multiple sensor nodes in a structured grid layout below.

![Image](https://github.com/PataleelaPuma/Mesh-Air-Purifier/blob/main/Pictures/Mesh%20Map.png?raw=true)

The environment was divided into a 4x4 grid, each cell contains the location (x,y) and ideally contains PM2.5 Values from the sensors + ESP32 sensor pair, Red cells represent the location of the PM2.5 sensor, and Green cells represent the location of the air purifier. Each sensor reports real-time PM2.5 concentrations to Home Assistant. The location of the fan is fixed near this grid, and its proximity to each sensor is a distance between its cell and the sensor’s cell.

**Air Purifier Fan Control Logic:**
To ensure effective localized air purification and minimize unnecessary fan operation, a region-based control logic was implemented for each air purifier by monitoring the PM2.5 values within a 1-cell region around it.
Control Region Definition

- For air purifiers located near the grid corners or edges (e.g., A1), a 2×2 region is considered (the purifier cell and its immediate neighbors).

![2×2 region](https://github.com/PataleelaPuma/Mesh-Air-Purifier/blob/main/Pictures/2x2%20Coverd%20Region.png?raw=true)

- For centrally located air purifiers (e.g., B2), a 3×3 region is considered, centered around the purifier cell.

![3×3 region](https://github.com/PataleelaPuma/Mesh-Air-Purifier/blob/main/Pictures/3x3%20Coverd%20Region.png?raw=true)

**Fan ON/OFF Conditions:**

- Turn ON: The fan is turned ON only when all PM2.5 values within the defined region exceed a predefined threshold, which is 35 μg/m3 in this project. This ensures that the surrounding air quality genuinely requires purification, avoiding false positives from isolated spikes.

- Turn OFF: The fan is turned OFF only when all PM2.5 values in the region fall below the threshold, which is 30 μg/m3 in this project. This ensures that residual pollutants are addressed before stopping.

This logic provides a form of spatial hysteresis, promoting system stability by preventing frequent toggling due to minor fluctuations. It also improves energy efficiency and filter life by targeting regions with genuinely poor air quality.

**Weighted PM2.5 Calculation:**
To determine whether the air purifier should activate, the system computes a weighted average PM2.5 index, where each sensor’s contribution is adjusted based on its distance to the fan. This approach gives more influence to sensors closer to the fan, under the assumption that they better represent the air quality in the fan’s effective zone.

_1. Sensor Data Collection_

- p1, p2, and p3 ( on cell A3,C1 and D4) are the real-time PM2.5 values from three sensors.

- The sensors are placed at known coordinates in a 2D grid (e.g., (2,0), (0,2), and (3,3)).

_2. Distance Calculation_

- We define the target point (e.g., (2,3)) and compute the Euclidean distance from each sensor to this point: 
- d_i=√(〖(x-x_i)〗^2+〖(y-y_i)〗^2 )+ε
- _(x,y) is the target point.
- (x_i, y_i) is the sensor’s coordinate.
- ε = 0.01 is added to prevent division by zero when the distance is very small._

_3. Weight Assignment_

- Weights are calculated using the inverse of the distance: w_i=1/d_i. This ensures that closer sensors have more weight in the final result.

_4. Interpolation Formula_

- The PM2.5 value at the target point is computed as a weighted average of sensor values:
- PM=((〖p_1*w〗_1)+(〖p_2*w〗_2)+(〖p_3*w〗_3))/(w_1+w_2+w_3 )
- _This formula yields a smoothed, location-specific air quality value based on spatial proximity to the actual sensor locations.
- Implementation
- The method is implemented using Jinja2 templating in Home Assistant, where real-time sensor values are pulled from entity states, and the entire calculation is done dynamically within an automation or dashboard card._


### **Software Required**

- _eWELink_: In this system, EWeLink is used exclusively to expose the Sonoff switch (AC fan control) to Home Assistant via add-on.

- _ESPHome_: ESPHome is an open-source firmware that simplifies the integration of ESP32/ESP8266 boards with Home Assistant. The ESP32 is programmed via YAML configuration and compiled into binary firmware, which is then uploaded over USB or OTA.

- _Home Assistant_: Home Assistant acts as the central hub for automation and monitoring. Once ESPHome is integrated: Sensor Integration: PM2.5 data appears as a sensor entity in Home Assistant (e.g., sensor.pm25_value). This data can be visualized on dashboards, logged, and further used in automation. Automation Logic: Home Assistant supports YAML-based automation rules. For example: Device Control: Home Assistant communicates with the Sonoff smart switch

### **Setup Procedure**
### **Hardware Section**

- PM2.5 Sensor Wiring Diagram:

![PM2.5 Sensor Wiring Diagram](https://github.com/PataleelaPuma/Mesh-Air-Purifier/blob/main/Pictures/Wiring%20PM2.5%20Sensor.png?raw=true)

- AC Fan & Sonoff Wiring Diagram:

![AC Fan & Sonoff Wiring Diagram](https://github.com/PataleelaPuma/Mesh-Air-Purifier/blob/main/Pictures/Wiring%20sonoff%20&%20ac%20fan.png?raw=true)

### **Software Section**
**Home Assistant Setup**
We began by installing Home Assistant Supervisor on a Raspberry Pi, which serves as the central automation hub.

Docker & Home Assistant Supervised Installation on Raspberry Pi (Source: https://www.omgthecloud.com/home-assistant-supervised/#google_vignette)
Docker is used to package and run an isolated environment for Home Assistant.

1. Initial command to update packages: 

- `sudo apt update`

- `sudo apt upgrade`

2. Install Prerequisites: 

- `sudo apt install apparmor bluez cifs-utils curl dbus jq libglib2.0-bin lsb-release network-manager nfs-common systemd-journal-remote systemd-resolved udisks2 wget -y`
The command sets up the system with tools for security, Bluetooth, network sharing, remote log collection, JSON processing, and system management—preparing the environment for further configuration or use.

3. Install Docker: 

- `curl -fsSL get.docker.com | sh`
- `sudo usermod -aG docker Admin`

4. Install OS Agent

- `wget https://github.com/home-assistant/os-agent/releases/download/1.6.0/os-agent_1.6.0_linux_aarch64.deb`
- `sudo apt install ./os-agent_1.6.0_linux_aarch64.deb`
These commands download and install the Home Assistant OS Agent on an ARM64 Linux system, enabling better hardware and system integration features within Home Assistant.

5. Install Home Assistant Supervised
- `wget https://github.com/home-assistant/supervised-installer/releases/latest/download/homeassistant-supervised.deb`

6. This command installs Home Assistant Supervised

- `sudo apt install ./homeassistant-supervised.deb`
Choose machine type: qemuarm-64

7. This command is used to watch the installation and background task progress.

- `sudo journalctl -f`

8. This command is used to check whether Docker/HomeAssistant is running.

- `sudo docker stats`

**Setting up Matter Devices with eWeLink**
1.	Open the [eWeLink app](https://ewelink.cc/ewelink-app-v5-6-supports-matter-th-sensor-and-matter-door-window-sensor/).
2.	Tap the + button in the top right corner.
3.	Tap Scan
4.	Scan the onboarding code you found earlier, or manually input the numeric code.
5.	The eWeLink app will guide you through the setup, including naming devices and assigning them to rooms.

**Connecting eWeLink to Home Assistant**
1.	Click on 'Settings' on the sidebar.
2.	In the options, select 'Add-ons'
3.	Click on the 'Add-on store' button on the bottom right of the screen.
4.	On the top right of the scree,n click on the three dots.
5.	Click 'Repositories'.
6.	Copy and paste the eWeLink add-on link
7.	https://github.com/CoolKit-Technologies/ha-addon
8.	Click the 'Add' button.
9.	Close the pop-up.
10.	The add-on is now available on the add-on store page.
11.	Click on the add-on and click install.
12.	After the installation, select the options you want
13.	Click on the 'Start' link to activate the add-on.

**Esp32 & Esp Home Integration Procedure**
1.	After wiring the ESP32 with the IKEA sensor board, we installed an add-on called ESP Home Builder in Home Assistant
2.	In ESP Home, add a new device -> ESP32 -> Name a device -> Select ESP32
3.	Edit the YAML file following the YAML script included in the repo (edit the pin to match your wiring)
4.	Click install manually to get the .bin file used for flashing ESP32. Wait until finish
5.	Enter https://web.esphome.io/ (web-based tool for flashing)
6.	Select the port to connect to ESP32
7.	Before clicking install and choosing the .bin file, Press and hold the boot button located at ESP32 (Doesn’t have to do this in some ESP32 models)
8.	Erase the status appear, hold the button until it’s gone, and wait until it finishes
9.	In the Settings tab, click Devices & services
10.	Then add ESP Home

### **Demo Video**
[Demo Video](https://youtu.be/HwIsHfZKcZ4)

Directed by:
6430252321 Pataleela Buranavichetkul |
6432058321 Thakdanai Srisuk |
6430146521 Thanadon Tanabodeejirapong
