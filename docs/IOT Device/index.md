# Detailed Documentation for IoT Gateways

This document provides a comprehensive overview of three IoT gateway devices suitable for energy management systems, focusing on how they emit data to the cloud/local systems, along with their respective pros and cons. These gateways are designed to connect Modbus-enabled renewable energy equipment (solar inverters, wind turbine controllers) and transmit data to your applications.

---

## 1. Moxa AIG-101 Series

### Overview
The Moxa AIG-101 is an industrial-grade IIoT gateway that connects Modbus RTU/ASCII/TCP devices to cloud platforms such as Azure, AWS, and any MQTT broker. It is built for harsh environments with a wide operating temperature range and robust certifications. It features two software-selectable serial ports, dual Ethernet ports, and optional LTE Cat 1 cellular connectivity (APAC model available). The device also supports Modbus TCP server mode for simultaneous local SCADA access.

### Data Emission Capabilities
- **Cloud Protocols**:
  - Generic MQTT client (v3.1.1) with TLS 1.0/1.2, QoS 0/1/2, username/password authentication.
  - Native Azure IoT Device SDK (supports MQTT, AMQP, WebSockets; authentication via symmetric key or X.509 certificate).
  - Native AWS IoT Core (supports MQTT with X.509 certificate authentication).
- **Local Access**: Modbus TCP server mode allows a local PC or SCADA system to read data from the gateway over Ethernet.
- **Data Buffering**: Store and forward mechanism ensures no data loss during network outages; data is stored in onboard 8 GB eMMC or microSD card.
- **Preprocessing**: Built-in data processing functions (e.g., averaging, scaling) can be configured without coding, reducing cloud load.
- **Remote Management**: Web console (HTTP/HTTPS), AIG QuickON tool, and diagnostic features (traffic monitoring, packet capture) enable remote troubleshooting.
- **Cellular Connectivity**:
  - LTE Cat 1 with APAC bands: 1 (2100 MHz), 3 (1800 MHz), 5 (850 MHz), 8 (900 MHz).
  - Dual SIM slots for redundancy.
  - SMA antenna connectors for LTE and GPS.
- **Serial Communication**: Two RS-232/422/485 ports (software-selectable) with support for up to 62 Modbus slaves per port.

### Pros
- **Industrial Durability**: Metal housing, -40°C to 70°C operating temperature, 5-year warranty, and extensive EMC/safety certifications.
- **Flexible Cloud Integration**: Direct support for Azure and AWS SDKs, plus generic MQTT.
- **Local SCADA Interface**: Modbus TCP server allows easy integration with on-site PCs.
- **High Scalability**: Can handle up to 1500 data tags and 62 Modbus devices per serial port.
- **Advanced Diagnostics**: Built-in tools for remote troubleshooting reduce maintenance costs.
- **Store and Forward**: Prevents data loss during network interruptions.
- **Expandable I/O**: Can integrate with Moxa ioLogik modules for additional I/O.

### Cons
- **No Built-in I/O**: Requires external modules for analog/digital inputs, increasing cost and complexity if such signals are needed.
- **Higher Cost**: Typically more expensive than consumer-grade gateways due to industrial build quality.
- **Configuration Complexity**: While user-friendly, it may require more initial setup compared to plug-and-play devices with bundled software.

- **Data sheet**: [Click Here](../static/moxa-aig-101-series-datasheet-v2.1.pdf).

- **[Product Link](https://www.moxa.com/en/products/industrial-computing/iiot-gateways/ready-to-deploy-iiot-gateways/aig-101-series#resources)**

## Distributor Details:

- Moxa India Industrial Networking Private Ltd.
- Address: 4A113, WeWork Galaxy, No.43, Residency Road, Ashok Nagar 560025, Bangalore, Karnataka, India
- Phone 91-80-4172-9088
- Fax 91-80-4132-1045
- Website https://www.moxa.com/

---

## 2. PowerArm E-Track 4G Edge

### Overview
The PowerArm E-Track 4G Edge is a compact IoT gateway designed to convert Modbus RTU/TCP data to GPRS/4G/LTE with built-in I/O capabilities. It features 12 pins for power, RS485, and multiple I/O lines, making it suitable for applications where local sensors need direct connection. It supports configurable transmission protocols including TCP, HTTP, HTTPS, and MQTT with SSL.

### Data Emission Capabilities
- **Cloud Protocols**:
  - Configurable TCP, HTTP, HTTPS, MQTT (with SSL).
  - HTTP methods: GET/POST.
  - Store and forward with 64 MB flash memory (up to 17,000 records).
- **Local Access**: Not explicitly detailed, but Ethernet port (RJ45) and Modbus TCP master capability suggest possible local connectivity.
- **I/O Integration**:
  - 2 digital inputs (0-24V)
  - 2 analog differential inputs (0-24V) – labeled ADI
  - 2 analog current inputs (4-20mA) – labeled AI+/AI-
  - 1 relay output (NO, COM)
  - All I/O can be read and transmitted along with Modbus data.
- **Cellular Connectivity**:
  - Supports LTE FDD/TDD, UMTS, EDGE, GPRS.
  - Bands not explicitly listed, but GSM specs include EGSM900, DCS1800, and LTE bands typical for global models.
  - Single SIM (no dual SIM mention).
- **Serial Communication**:
  - RS485 interface (D+, D-) with support for up to 16 Modbus slave devices.
  - Baud rates up to 115200 bps.
  - 2 KV surge protection on RS485.
- **Configuration**: Auto-discovery via IP utility, likely web-based.

### Pros
- **Integrated I/O**: Built-in analog and digital inputs eliminate the need for separate I/O modules, ideal for monitoring additional sensors (temperature, panel voltage, etc.).
- **Relay Output**: Can be used for local control (e.g., alarms, switching).
- **Wide Operating Temperature**: -20°C to +85°C, suitable for most outdoor environments in India.
- **Low Power Consumption**: Idle 0.4W, transmission ~1W at 12VDC.
- **Flexible Protocols**: Supports multiple transmission protocols including MQTT with SSL.
- **Surge Protection**: On RS485 line, enhancing reliability in electrical environments.

### Cons
- **Limited Memory**: 64 MB flash for data logging may be insufficient for high-frequency, long-term storage.
- **Plastic Casing**: Less robust than metal enclosures; may degrade faster in extreme conditions.
- **Cellular Bands Not Explicitly Listed**: Uncertainty about compatibility with all Indian LTE bands (though GSM bands are common). Recommended to verify with vendor.
- **No Dual SIM**: No redundancy for cellular connectivity.
- **Configuration Tools**: Less mature than Moxa’s; may require more manual setup.

- **Data Sheet**: [Click Here](../static/E-Track-%204G-Edge.pdf)

- **[Product Link](https://poweramr.in/datalogger)**

## Dealer Details

**Name**: Logics PowerAMR Pvt. Ltd,
**Address**: 1st Floor, A-55, Sector 16, Noida, Uttar Pradesh 201301,
**Mail**: info@poweramr.in, support@poweramr.in
**Phone**: +91-8076963066

---