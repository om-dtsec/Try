# Enhanced Twin4 Docker Compose Configuration

```yaml
# Twin4 - Advanced FIWARE-based Smart Manufacturing Digital Twin
# Complete architecture with multiple PLCs, sensors, actuators, and external simulators

version: '3.8'

services:
  # =============================================================================
  # FIWARE CORE INFRASTRUCTURE
  # =============================================================================
  
  # MongoDB for Orion Context Broker
  mongo-db:
    image: mongo:4.4
    hostname: mongo-db
    container_name: twin4-mongo
    networks:
      - twin4-network
    command: --nojournal
    volumes:
      - mongo-db-data:/data/db
    restart: unless-stopped

  # Orion Context Broker - NGSI-v2
  orion:
    image: fiware/orion:3.10.1
    hostname: orion
    container_name: twin4-orion
    depends_on:
      - mongo-db
    networks:
      - twin4-network
    ports:
      - "1026:1026"
    command: -dbhost mongo-db -logLevel DEBUG -corsOrigin __ALL
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:1026/version"]
      interval: 30s
      timeout: 10s
      retries: 3

  # MQTT Broker - Mosquitto
  mosquitto:
    image: eclipse-mosquitto:2.0.15
    hostname: mosquitto
    container_name: twin4-mosquitto
    networks:
      - twin4-network
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./config/mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf
      - mosquitto-data:/mosquitto/data
      - mosquitto-logs:/mosquitto/log
    restart: unless-stopped

  # Primary IoT Agent UltraLight 2.0
  iot-agent-ul:
    image: fiware/iotagent-ul:2.4.0
    hostname: iot-agent-ul
    container_name: twin4-iot-agent-ul
    depends_on:
      - mongo-db
      - mosquitto
      - orion
    networks:
      - twin4-network
    ports:
      - "4041:4041"
      - "7896:7896"
    environment:
      - IOTA_CB_HOST=orion
      - IOTA_CB_PORT=1026
      - IOTA_NORTH_PORT=4041
      - IOTA_REGISTRY_TYPE=mongodb
      - IOTA_LOG_LEVEL=DEBUG
      - IOTA_TIMESTAMP=true
      - IOTA_CB_NGSI_VERSION=v2
      - IOTA_AUTOCAST=true
      - IOTA_MONGO_HOST=mongo-db
      - IOTA_MONGO_PORT=27017
      - IOTA_MONGO_DB=iotagentul
      - IOTA_HTTP_PORT=7896
      - IOTA_PROVIDER_URL=http://iot-agent-ul:4041
      - IOTA_MQTT_HOST=mosquitto
      - IOTA_MQTT_PORT=1883
      - IOTA_DEFAULT_RESOURCE=/iot/d
    restart: unless-stopped

  # Secondary IoT Agent for OPC-UA
  iot-agent-opcua:
    image: iotagent4fiware/iotagent-opcua:1.4.0
    hostname: iot-agent-opcua
    container_name: twin4-iot-agent-opcua
    depends_on:
      - mongo-db
      - orion
    networks:
      - twin4-network
    ports:
      - "4042:4041"
      - "4001:4001"
    environment:
      - IOTA_CB_HOST=orion
      - IOTA_CB_PORT=1026
      - IOTA_NORTH_PORT=4041
      - IOTA_REGISTRY_TYPE=mongodb
      - IOTA_LOG_LEVEL=DEBUG
      - IOTA_TIMESTAMP=true
      - IOTA_CB_NGSI_VERSION=v2
      - IOTA_AUTOCAST=true
      - IOTA_MONGO_HOST=mongo-db
      - IOTA_MONGO_PORT=27017
      - IOTA_MONGO_DB=iotagentopcua
      - IOTA_PROVIDER_URL=http://iot-agent-opcua:4041
    restart: unless-stopped

  # CrateDB for time-series storage
  crate-db:
    image: crate/crate:4.6.6
    hostname: crate-db
    container_name: twin4-crate
    ports:
      - "4200:4200"
      - "4300:4300"
    networks:
      - twin4-network
    command: crate -Cauth.host_based.enabled=false -Ccluster.name=twin4cluster -Chttp.cors.enabled=true -Chttp.cors.allow-origin="*"
    volumes:
      - crate-db-data:/data
    restart: unless-stopped

  # QuantumLeap for persisting time-series data
  quantumleap:
    image: orchestracities/quantumleap:0.8.3
    hostname: quantumleap
    container_name: twin4-quantumleap
    ports:
      - "8668:8668"
    depends_on:
      - crate-db
      - orion
    networks:
      - twin4-network
    environment:
      - CRATE_HOST=crate-db
      - CRATE_PORT=4200
      - LOGLEVEL=DEBUG
    restart: unless-stopped

  # Grafana for visualization
  grafana:
    image: grafana/grafana:9.1.0
    container_name: twin4-grafana
    ports:
      - "3000:3000"
    networks:
      - twin4-network
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./config/grafana/provisioning:/etc/grafana/provisioning
    restart: unless-stopped

  # =============================================================================
  # EXTERNAL SIMULATORS - REALISTIC INDUSTRIAL TOOLS
  # =============================================================================

  # SoftPLC - Industrial PLC Simulator
  soft-plc:
    image: fbarresi/softplc:latest-linux
    hostname: soft-plc
    container_name: twin4-softplc
    networks:
      - twin4-network
    ports:
      - "8080:8080"
      - "8443:443"
      - "102:102"
    environment:
      - DATA_PATH=/demodata
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # OpenPLC Runtime
  openplc-runtime:
    image: autonomylogic/openplc_v3:latest
    hostname: openplc-runtime
    container_name: twin4-openplc
    networks:
      - twin4-network
    ports:
      - "8081:8080"
      - "502:502"
    volumes:
      - ./simulators/external/openplc/programs:/workdir
    restart: unless-stopped
    privileged: true

  # =============================================================================
  # ENHANCED SIMULATORS WITH REALISTIC BEHAVIOR
  # =============================================================================

  # SCADA Simulator - Root level with multiple PLCs
  scada-simulator:
    build: 
      context: ./simulators/scada
      dockerfile: Dockerfile
    container_name: twin4-scada
    depends_on:
      - mosquitto
      - iot-agent-ul
      - orion
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - SERVICE_NAME=twin4factory
      - SERVICE_PATH=/
      - API_KEY=twin4apikey2025
      - DEVICE_ID=scada001
      - CONNECTED_PLCS=plc001,plc002,plc003
    restart: unless-stopped

  # Enhanced PLC1 Simulator - Paper Feeder with multiple sensors
  plc1-simulator:
    build: 
      context: ./simulators/plc/plc1
      dockerfile: Dockerfile
    container_name: twin4-plc1
    depends_on:
      - mosquitto
      - iot-agent-ul
      - scada-simulator
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - SERVICE_NAME=twin4factory
      - SERVICE_PATH=/
      - API_KEY=twin4apikey2025
      - DEVICE_ID=plc001
      - PLC_TYPE=paper_feeder
      - PARENT_DEVICE=scada001
      - CONNECTED_SENSORS=temp001,pressure001,vibration001
      - MODBUS_PORT=502
      - OPC_UA_PORT=4840
    ports:
      - "4840:4840"  # OPC-UA port
      - "50201:502"  # Modbus port
    restart: unless-stopped

  # Enhanced PLC2 Simulator - Printing Press with actuators
  plc2-simulator:
    build: 
      context: ./simulators/plc/plc2
      dockerfile: Dockerfile
    container_name: twin4-plc2
    depends_on:
      - mosquitto
      - iot-agent-ul
      - scada-simulator
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - SERVICE_NAME=twin4factory
      - SERVICE_PATH=/
      - API_KEY=twin4apikey2025
      - DEVICE_ID=plc002
      - PLC_TYPE=printing_press
      - PARENT_DEVICE=scada001
      - CONNECTED_SENSORS=temp002,camera001
      - CONNECTED_ACTUATORS=motor001,valve001
      - MODBUS_PORT=502
      - OPC_UA_PORT=4841
    ports:
      - "4841:4841"  # OPC-UA port
      - "50202:502"  # Modbus port
    restart: unless-stopped

  # New PLC3 Simulator - Process Control
  plc3-simulator:
    build: 
      context: ./simulators/plc/plc3
      dockerfile: Dockerfile
    container_name: twin4-plc3
    depends_on:
      - mosquitto
      - iot-agent-ul
      - scada-simulator
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - SERVICE_NAME=twin4factory
      - SERVICE_PATH=/
      - API_KEY=twin4apikey2025
      - DEVICE_ID=plc003
      - PLC_TYPE=process_control
      - PARENT_DEVICE=scada001
      - CONNECTED_SENSORS=temp003,pressure002
      - CONNECTED_ACTUATORS=motor002
      - MODBUS_PORT=502
      - OPC_UA_PORT=4842
    ports:
      - "4842:4842"  # OPC-UA port
      - "50203:502"  # Modbus port
    restart: unless-stopped

  # Enhanced RTU Simulator with more protocols
  rtu-simulator:
    build: 
      context: ./simulators/rtu
      dockerfile: Dockerfile
    container_name: twin4-rtu
    depends_on:
      - mosquitto
      - iot-agent-ul
      - scada-simulator
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - SERVICE_NAME=twin4factory
      - SERVICE_PATH=/
      - API_KEY=twin4apikey2025
      - DEVICE_ID=rtu001
      - PARENT_DEVICE=scada001
      - PROTOCOLS=MODBUS_RTU,MODBUS_TCP,DNP3
      - MODBUS_PORT=502
    ports:
      - "50301:502"  # Modbus port
    restart: unless-stopped

  # =============================================================================
  # SENSOR SIMULATORS
  # =============================================================================

  # Temperature Sensor 1
  temp-sensor-1:
    build: 
      context: ./simulators/sensors/temperature
      dockerfile: Dockerfile
    container_name: twin4-temp1
    depends_on:
      - mosquitto
      - iot-agent-ul
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - DEVICE_ID=temp001
      - SENSOR_TYPE=temperature
      - PARENT_DEVICE=plc001
      - MEASUREMENT_RANGE=0-100
      - ACCURACY=0.1
      - UPDATE_INTERVAL=5
    restart: unless-stopped

  # Temperature Sensor 2
  temp-sensor-2:
    build: 
      context: ./simulators/sensors/temperature
      dockerfile: Dockerfile
    container_name: twin4-temp2
    depends_on:
      - mosquitto
      - iot-agent-ul
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - DEVICE_ID=temp002
      - SENSOR_TYPE=temperature
      - PARENT_DEVICE=plc002
      - MEASUREMENT_RANGE=60-80
      - ACCURACY=0.1
      - UPDATE_INTERVAL=5
    restart: unless-stopped

  # Temperature Sensor 3
  temp-sensor-3:
    build: 
      context: ./simulators/sensors/temperature
      dockerfile: Dockerfile
    container_name: twin4-temp3
    depends_on:
      - mosquitto
      - iot-agent-ul
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - DEVICE_ID=temp003
      - SENSOR_TYPE=temperature
      - PARENT_DEVICE=plc003
      - MEASUREMENT_RANGE=20-40
      - ACCURACY=0.1
      - UPDATE_INTERVAL=5
    restart: unless-stopped

  # Pressure Sensor 1
  pressure-sensor-1:
    build: 
      context: ./simulators/sensors/pressure
      dockerfile: Dockerfile
    container_name: twin4-pressure1
    depends_on:
      - mosquitto
      - iot-agent-ul
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - DEVICE_ID=pressure001
      - SENSOR_TYPE=pressure
      - PARENT_DEVICE=plc001
      - MEASUREMENT_RANGE=0-10
      - ACCURACY=0.01
      - UPDATE_INTERVAL=3
    restart: unless-stopped

  # Pressure Sensor 2
  pressure-sensor-2:
    build: 
      context: ./simulators/sensors/pressure
      dockerfile: Dockerfile
    container_name: twin4-pressure2
    depends_on:
      - mosquitto
      - iot-agent-ul
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - DEVICE_ID=pressure002
      - SENSOR_TYPE=pressure
      - PARENT_DEVICE=plc003
      - MEASUREMENT_RANGE=0-15
      - ACCURACY=0.01
      - UPDATE_INTERVAL=3
    restart: unless-stopped

  # Vibration Sensor
  vibration-sensor:
    build: 
      context: ./simulators/sensors/vibration
      dockerfile: Dockerfile
    container_name: twin4-vibration1
    depends_on:
      - mosquitto
      - iot-agent-ul
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - DEVICE_ID=vibration001
      - SENSOR_TYPE=vibration
      - PARENT_DEVICE=plc001
      - MEASUREMENT_RANGE=0-50
      - ACCURACY=0.1
      - UPDATE_INTERVAL=2
    restart: unless-stopped

  # Vision Camera
  vision-camera:
    build: 
      context: ./simulators/sensors/camera
      dockerfile: Dockerfile
    container_name: twin4-camera1
    depends_on:
      - mosquitto
      - iot-agent-ul
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - DEVICE_ID=camera001
      - SENSOR_TYPE=camera
      - PARENT_DEVICE=plc002
      - FRAME_RATE=30
      - RESOLUTION=1920x1080
      - UPDATE_INTERVAL=10
    restart: unless-stopped

  # =============================================================================
  # ACTUATOR SIMULATORS
  # =============================================================================

  # Motor Actuator 1
  motor-actuator-1:
    build: 
      context: ./simulators/actuators/motor
      dockerfile: Dockerfile
    container_name: twin4-motor1
    depends_on:
      - mosquitto
      - iot-agent-ul
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - DEVICE_ID=motor001
      - ACTUATOR_TYPE=motor
      - PARENT_DEVICE=plc002
      - MAX_SPEED=3000
      - POWER_RATING=5000
      - UPDATE_INTERVAL=5
    restart: unless-stopped

  # Motor Actuator 2
  motor-actuator-2:
    build: 
      context: ./simulators/actuators/motor
      dockerfile: Dockerfile
    container_name: twin4-motor2
    depends_on:
      - mosquitto
      - iot-agent-ul
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - DEVICE_ID=motor002
      - ACTUATOR_TYPE=motor
      - PARENT_DEVICE=plc003
      - MAX_SPEED=1800
      - POWER_RATING=2200
      - UPDATE_INTERVAL=5
    restart: unless-stopped

  # Valve Actuator
  valve-actuator:
    build: 
      context: ./simulators/actuators/valve
      dockerfile: Dockerfile
    container_name: twin4-valve1
    depends_on:
      - mosquitto
      - iot-agent-ul
    networks:
      - twin4-network
    environment:
      - MQTT_BROKER=mosquitto
      - IOT_AGENT_URL=http://iot-agent-ul:4041
      - DEVICE_ID=valve001
      - ACTUATOR_TYPE=valve
      - PARENT_DEVICE=plc002
      - VALVE_TYPE=pneumatic
      - MAX_PRESSURE=10
      - UPDATE_INTERVAL=3
    restart: unless-stopped

  # =============================================================================
  # CUSTOM FIWARE AGENTS
  # =============================================================================

  # Custom Agent for Advanced Sensor Data Processing
  custom-agent-1:
    build: 
      context: ./agents/agent1
      dockerfile: Dockerfile
    container_name: twin4-agent1
    depends_on:
      - orion
      - quantumleap
    networks:
      - twin4-network
    environment:
      - ORION_URL=http://orion:1026
      - QUANTUMLEAP_URL=http://quantumleap:8668
      - SERVICE_NAME=twin4factory
      - SERVICE_PATH=/sensors
    restart: unless-stopped

  # Custom Agent for PLC-to-PLC Communication
  custom-agent-2:
    build: 
      context: ./agents/agent2
      dockerfile: Dockerfile
    container_name: twin4-agent2
    depends_on:
      - orion
      - quantumleap
    networks:
      - twin4-network
    environment:
      - ORION_URL=http://orion:1026
      - QUANTUMLEAP_URL=http://quantumleap:8668
      - SERVICE_NAME=twin4factory
      - SERVICE_PATH=/plcs
    restart: unless-stopped

  # =============================================================================
  # WEB INTERFACE
  # =============================================================================

  # Custom Dashboard
  web-dashboard:
    build: 
      context: ./web_interface
      dockerfile: Dockerfile
    container_name: twin4-dashboard
    depends_on:
      - orion
      - grafana
    networks:
      - twin4-network
    ports:
      - "8090:80"
    environment:
      - ORION_URL=http://orion:1026
      - GRAFANA_URL=http://grafana:3000
    restart: unless-stopped

networks:
  twin4-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16

volumes:
  mongo-db-data:
  crate-db-data:
  grafana-data:
  mosquitto-data:
  mosquitto-logs:
```

## Key Enhancements in Twin4:

### 1. **Multiple Industrial Simulators**
- **SoftPLC**: Real industrial PLC simulator with Web API
- **OpenPLC**: Open-source PLC runtime with standard programming languages
- Enhanced Python simulators with industrial protocols

### 2. **Comprehensive Device Hierarchy**
- **1 SCADA system** managing multiple PLCs
- **3 PLCs** with different specializations:
  - PLC1: Paper feeder with vibration monitoring
  - PLC2: Printing press with vision and actuation
  - PLC3: Process control with pressure management
- **Multiple sensors**: Temperature (3), Pressure (2), Vibration (1), Camera (1)
- **Multiple actuators**: Motors (2), Valve (1)
- **Enhanced RTU** with multiple protocols

### 3. **Advanced FIWARE Integration**
- **Dual IoT Agents**: UltraLight + OPC-UA for maximum compatibility
- **Custom Agents**: Specialized data processing and PLC communication
- **Smart Data Models**: Following FIWARE standards
- **Multiple protocols**: MQTT, OPC-UA, Modbus TCP/RTU, DNP3

### 4. **Enhanced Communication**
- **PLC-to-PLC**: Direct communication via custom agents
- **Sensor-to-PLC**: Hierarchical data flow
- **SCADA oversight**: Central monitoring and control
- **Real-time subscriptions**: Event-driven data updates

### 5. **Professional Features**
- **Health checks**: Container monitoring
- **Restart policies**: High availability
- **Volume persistence**: Data retention
- **Network isolation**: Secure communication
- **Port mapping**: External access for testing

This architecture represents a **complete industrial IoT ecosystem** using FIWARE's full potential while integrating realistic industrial simulators for authentic learning and testing.
