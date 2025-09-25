# FIWARE Enhanced Industrial System Implementation
## Adding Advanced Features and Real-World Components

Your manager's feedback is spot-on! Let's enhance your system to use FIWARE's full potential with real device communication, multiple IoT Agents, and open-source industrial emulators.

## Phase 1: Add More Sensors Under PLCs (Immediate Enhancement)

### 1.1 Create Additional Sensor Devices

**Update your existing IoT Agent to handle more sensors:**

```powershell
# Add 3 more sensors under PLC001
$headers = @{ "fiware-service"="factory"; "fiware-servicepath"="/" }

# Sensor002 - Pressure sensor under PLC001
$sensor002 = @{
  devices = @(
    @{
      device_id = "sensor002"
      entity_name = "urn:ngsi-ld:Device:sensor002"
      entity_type = "Device"
      transport = "MQTT"
      apikey = "4jggokgpepnvsb2uv4s40d59ov"
      explicitAttrs = $true
      attributes = @(
        @{ object_id = "pr"; name = "pressure"; type = "Number" }
        @{ object_id = "te"; name = "temperature"; type = "Number" }
        @{ object_id = "se"; name = "sensor_health"; type = "Number" }
        @{ object_id = "pd"; name = "parent_device"; type = "Text" }
        @{ object_id = "st"; name = "status"; type = "Text" }
        @{ object_id = "ui"; name = "unique_id"; type = "Text" }
        @{ object_id = "hw"; name = "hardware_type"; type = "Text" }
        @{ object_id = "ip"; name = "ip_address"; type = "Text" }
      )
    }
  )
} | ConvertTo-Json -Depth 8

Invoke-RestMethod -Uri "http://localhost:4041/iot/devices" -Method Post -Headers $headers -ContentType "application/json" -Body $sensor002

# Sensor003 - Vibration sensor under PLC001
$sensor003 = @{
  devices = @(
    @{
      device_id = "sensor003"
      entity_name = "urn:ngsi-ld:Device:sensor003"
      entity_type = "Device"
      transport = "MQTT"
      apikey = "4jggokgpepnvsb2uv4s40d59ov"
      explicitAttrs = $true
      attributes = @(
        @{ object_id = "vi"; name = "vibration"; type = "Number" }
        @{ object_id = "te"; name = "temperature"; type = "Number" }
        @{ object_id = "se"; name = "sensor_health"; type = "Number" }
        @{ object_id = "pd"; name = "parent_device"; type = "Text" }
        @{ object_id = "st"; name = "status"; type = "Text" }
      )
    }
  )
} | ConvertTo-Json -Depth 8

Invoke-RestMethod -Uri "http://localhost:4041/iot/devices" -Method Post -Headers $headers -ContentType "application/json" -Body $sensor003

# Sensor004 - Flow sensor under PLC002
$sensor004 = @{
  devices = @(
    @{
      device_id = "sensor004"
      entity_name = "urn:ngsi-ld:Device:sensor004"
      entity_type = "Device"
      transport = "MQTT"
      apikey = "4jggokgpepnvsb2uv4s40d59ov"
      explicitAttrs = $true
      attributes = @(
        @{ object_id = "fl"; name = "flow_rate"; type = "Number" }
        @{ object_id = "te"; name = "temperature"; type = "Number" }
        @{ object_id = "pr"; name = "pressure"; type = "Number" }
        @{ object_id = "pd"; name = "parent_device"; type = "Text" }
        @{ object_id = "st"; name = "status"; type = "Text" }
      )
    }
  )
} | ConvertTo-Json -Depth 8

Invoke-RestMethod -Uri "http://localhost:4041/iot/devices" -Method Post -Headers $headers -ContentType "application/json" -Body $sensor004
```

### 1.2 Create New Sensor Simulators

**File: `simulators/sensor/sensor002_simulator.py`**
```python
import paho.mqtt.client as mqtt
import json
import time
import random
import threading

class PressureSensorSimulator:
    def __init__(self):
        self.client = mqtt.Client()
        self.client.on_connect = self.on_connect
        self.client.on_disconnect = self.on_disconnect
        
    def on_connect(self, client, userdata, flags, rc):
        print(f"Sensor002 (Pressure) connected with code {rc}")
        
    def on_disconnect(self, client, userdata, rc):
        print(f"Sensor002 (Pressure) disconnected with code {rc}")
        
    def generate_data(self):
        return {
            "pr": round(random.uniform(140.0, 160.0), 1),  # pressure in bar
            "te": round(random.uniform(68.0, 72.0), 1),    # temperature
            "se": random.randint(95, 100),                 # sensor_health
            "pd": "plc001",                                # parent_device
            "st": "Active",                                # status
            "ui": "PRESS-SENSOR-002",                      # unique_id
            "hw": "Siemens SITRANS P320",                  # hardware_type
            "ip": "192.168.1.102"                          # ip_address
        }
        
    def format_ultralight(self, data):
        return "|".join([f"{k}|{v}" for k, v in data.items()])
        
    def publish_loop(self):
        while True:
            try:
                data = self.generate_data()
                payload = self.format_ultralight(data)
                topic = "ul/4jggokgpepnvsb2uv4s40d59ov/sensor002/attrs"
                
                self.client.publish(topic, payload)
                print(f"Sensor002 Published: {payload}")
                
                time.sleep(6)  # Different interval from sensor001
                
            except Exception as e:
                print(f"Error in sensor002 publish loop: {e}")
                time.sleep(5)
                
    def run(self):
        try:
            self.client.connect("localhost", 1883, 60)
            self.client.loop_start()
            
            publish_thread = threading.Thread(target=self.publish_loop)
            publish_thread.daemon = True
            publish_thread.start()
            
            while True:
                time.sleep(1)
                
        except KeyboardInterrupt:
            print("Stopping sensor002 simulator...")
            self.client.loop_stop()
            self.client.disconnect()

if __name__ == "__main__":
    simulator = PressureSensorSimulator()
    simulator.run()
```

**File: `simulators/sensor/sensor003_simulator.py`**
```python
import paho.mqtt.client as mqtt
import json
import time
import random
import threading

class VibrationSensorSimulator:
    def __init__(self):
        self.client = mqtt.Client()
        self.client.on_connect = self.on_connect
        
    def on_connect(self, client, userdata, flags, rc):
        print(f"Sensor003 (Vibration) connected with code {rc}")
        
    def generate_data(self):
        return {
            "vi": round(random.uniform(0.5, 3.0), 2),      # vibration in mm/s
            "te": round(random.uniform(67.0, 73.0), 1),    # temperature
            "se": random.randint(92, 100),                 # sensor_health
            "pd": "plc001",                                # parent_device
            "st": "Active",                                # status
        }
        
    def format_ultralight(self, data):
        return "|".join([f"{k}|{v}" for k, v in data.items()])
        
    def publish_loop(self):
        while True:
            try:
                data = self.generate_data()
                payload = self.format_ultralight(data)
                topic = "ul/4jggokgpepnvsb2uv4s40d59ov/sensor003/attrs"
                
                self.client.publish(topic, payload)
                print(f"Sensor003 Published: {payload}")
                
                time.sleep(7)  # Different interval
                
            except Exception as e:
                print(f"Error in sensor003 publish loop: {e}")
                time.sleep(5)
                
    def run(self):
        try:
            self.client.connect("localhost", 1883, 60)
            self.client.loop_start()
            
            publish_thread = threading.Thread(target=self.publish_loop)
            publish_thread.daemon = True
            publish_thread.start()
            
            while True:
                time.sleep(1)
                
        except KeyboardInterrupt:
            print("Stopping sensor003 simulator...")
            self.client.loop_stop()
            self.client.disconnect()

if __name__ == "__main__":
    simulator = VibrationSensorSimulator()
    simulator.run()
```

## Phase 2: Implement Device-to-Device Communication (FIWARE Commands)

### 2.1 Enhanced PLC with Command Processing

**File: `simulators/plc/enhanced_plc001_simulator.py`**
```python
import paho.mqtt.client as mqtt
import json
import time
import random
import threading
from datetime import datetime

class EnhancedPLC001Simulator:
    def __init__(self):
        self.client = mqtt.Client()
        self.client.on_connect = self.on_connect
        self.client.on_message = self.on_message
        
        # Data from child sensors
        self.sensor_data = {
            "sensor001": {"temperature": 70.0, "status": "unknown"},
            "sensor002": {"pressure": 150.0, "status": "unknown"},
            "sensor003": {"vibration": 1.5, "status": "unknown"}
        }
        
        # PLC processing results
        self.processed_data = {
            "avg_temperature": 70.0,
            "max_pressure": 150.0,
            "vibration_alarm": False,
            "system_health": 100,
            "hello_count": 0
        }
        
    def on_connect(self, client, userdata, flags, rc):
        print(f"Enhanced PLC001 connected with code {rc}")
        # Subscribe to commands and child sensor data
        client.subscribe("ul/4jggokgpepnvsb2uv4s40d59ov/plc001/cmd")
        client.subscribe("ul/4jggokgpepnvsb2uv4s40d59ov/sensor001/attrs")
        client.subscribe("ul/4jggokgpepnvsb2uv4s40d59ov/sensor002/attrs")
        client.subscribe("ul/4jggokgpepnvsb2uv4s40d59ov/sensor003/attrs")
        
    def on_message(self, client, userdata, msg):
        topic = msg.topic
        payload = msg.payload.decode()
        
        print(f"PLC001 Received: {topic} -> {payload}")
        
        # Process sensor data from children
        if "sensor001/attrs" in topic:
            self.process_sensor_data("sensor001", payload)
        elif "sensor002/attrs" in topic:
            self.process_sensor_data("sensor002", payload)
        elif "sensor003/attrs" in topic:
            self.process_sensor_data("sensor003", payload)
        elif "plc001/cmd" in topic:
            self.process_command(payload)
            
    def process_sensor_data(self, sensor_id, payload):
        """Process incoming sensor data and update local state"""
        try:
            # Parse UltraLight format: te|70.5|pr|145.2|...
            pairs = payload.split("|")
            data = {}
            for i in range(0, len(pairs)-1, 2):
                if i+1 < len(pairs):
                    key, value = pairs[i], pairs[i+1]
                    try:
                        data[key] = float(value)
                    except:
                        data[key] = value
                        
            # Update sensor data
            if sensor_id not in self.sensor_data:
                self.sensor_data[sensor_id] = {}
                
            if "te" in data:
                self.sensor_data[sensor_id]["temperature"] = data["te"]
            if "pr" in data:
                self.sensor_data[sensor_id]["pressure"] = data["pr"]
            if "vi" in data:
                self.sensor_data[sensor_id]["vibration"] = data["vi"]
            if "st" in data:
                self.sensor_data[sensor_id]["status"] = data["st"]
                
            # Recompute processed values
            self.compute_aggregated_data()
            
        except Exception as e:
            print(f"Error processing sensor data from {sensor_id}: {e}")
            
    def compute_aggregated_data(self):
        """Aggregate and process sensor data like a real PLC"""
        temperatures = []
        pressures = []
        vibrations = []
        
        for sensor_id, data in self.sensor_data.items():
            if "temperature" in data:
                temperatures.append(data["temperature"])
            if "pressure" in data:
                pressures.append(data["pressure"])
            if "vibration" in data:
                vibrations.append(data["vibration"])
                
        # Compute aggregated values
        if temperatures:
            self.processed_data["avg_temperature"] = round(sum(temperatures) / len(temperatures), 1)
            
        if pressures:
            self.processed_data["max_pressure"] = round(max(pressures), 1)
            
        if vibrations:
            max_vibration = max(vibrations)
            self.processed_data["vibration_alarm"] = max_vibration > 2.5
            
        # Compute system health based on all factors
        health = 100
        if self.processed_data["avg_temperature"] > 75:
            health -= 10
        if self.processed_data["max_pressure"] > 155:
            health -= 15
        if self.processed_data["vibration_alarm"]:
            health -= 20
            
        self.processed_data["system_health"] = max(health, 0)
        
    def process_command(self, payload):
        """Process commands from SCADA or other PLCs"""
        print(f"PLC001 processing command: {payload}")
        
    def generate_plc_data(self):
        """Generate PLC telemetry including processed sensor data"""
        return {
            "te": self.processed_data["avg_temperature"],
            "pr": round(random.uniform(142.0, 148.0), 1),  # PLC internal pressure
            "vi": round(random.uniform(1.0, 2.0), 1),       # PLC vibration
            "rp": random.randint(1480, 1520),               # RPM
            "po": random.randint(800, 900),                 # Power consumption
            "st": "Running",
            "md": "Auto",
            "sh": self.processed_data["system_health"],     # System health from sensors
            "mp": self.processed_data["max_pressure"],      # Max pressure from sensors
            "va": str(self.processed_data["vibration_alarm"]).lower(),  # Vibration alarm
            "sc": len([s for s in self.sensor_data.values() if s.get("status") == "Active"]),  # Sensor count
            "pd": "scada001",
            "cd": "sensor001,sensor002,sensor003",
        }
        
    def send_hello_to_plc002(self):
        """Send hello signal to PLC002 every 5 seconds"""
        hello_data = {
            "from": "plc001",
            "to": "plc002",
            "message": "hello",
            "timestamp": datetime.now().isoformat(),
            "count": self.processed_data["hello_count"]
        }
        
        payload = "|".join([f"{k}|{v}" for k, v in hello_data.items()])
        topic = "ul/4jggokgpepnvsb2uv4s40d59ov/plc002/cmd"
        
        self.client.publish(topic, payload)
        self.processed_data["hello_count"] += 1
        print(f"PLC001 -> PLC002 Hello #{self.processed_data['hello_count']}")
        
    def format_ultralight(self, data):
        return "|".join([f"{k}|{v}" for k, v in data.items()])
        
    def publish_loop(self):
        hello_timer = 0
        while True:
            try:
                # Publish PLC telemetry every 8 seconds
                data = self.generate_plc_data()
                payload = self.format_ultralight(data)
                topic = "ul/4jggokgpepnvsb2uv4s40d59ov/plc001/attrs"
                
                self.client.publish(topic, payload)
                print(f"PLC001 Published: processed sensors -> health:{data['sh']}%, max_pressure:{data['mp']}")
                
                # Send hello to PLC002 every 5 seconds
                hello_timer += 8
                if hello_timer >= 5:
                    self.send_hello_to_plc002()
                    hello_timer = 0
                
                time.sleep(8)
                
            except Exception as e:
                print(f"Error in PLC001 publish loop: {e}")
                time.sleep(5)
                
    def run(self):
        try:
            self.client.connect("localhost", 1883, 60)
            self.client.loop_start()
            
            publish_thread = threading.Thread(target=self.publish_loop)
            publish_thread.daemon = True
            publish_thread.start()
            
            while True:
                time.sleep(1)
                
        except KeyboardInterrupt:
            print("Stopping Enhanced PLC001 simulator...")
            self.client.loop_stop()
            self.client.disconnect()

if __name__ == "__main__":
    simulator = EnhancedPLC001Simulator()
    simulator.run()
```

## Phase 3: Add Second IoT Agent for Different Protocol

### 3.1 Update Docker Compose for Second IoT Agent

**File: `docker-compose-enhanced.yml` (extends your existing)**
```yaml
version: '3.8'
networks:
  factory-network:
    driver: bridge

services:
  # Keep all your existing services...
  
  # Second IoT Agent for different protocol (JSON over HTTP)
  fiware-iot-agent-json:
    image: fiware/iotagent-json:1.25.0
    hostname: iot-agent-json
    container_name: fiware-iot-agent-json
    depends_on:
      - db-mongo
      - fiware-orion
    networks:
      - factory-network
    ports:
      - "4042:4041"
      - "7897:7896"
    environment:
      - IOTA_CB_HOST=orion
      - IOTA_CB_PORT=1026
      - IOTA_NORTH_PORT=4041
      - IOTA_REGISTRY_TYPE=mongodb
      - IOTA_MONGO_HOST=db-mongo
      - IOTA_MONGO_PORT=27017
      - IOTA_MONGO_DB=iotagentjson
      - IOTA_HTTP_PORT=7896
      - IOTA_PROVIDER_URL=http://iot-agent-json:4041
      - IOTA_DEFAULT_RESOURCE=/iot/json
      - IOTA_LOG_LEVEL=DEBUG

  # Add Node-RED for visual programming and device orchestration
  node-red:
    image: nodered/node-red:3.0.2
    container_name: node-red
    hostname: node-red
    networks:
      - factory-network
    ports:
      - "1880:1880"
    volumes:
      - node-red-data:/data
    environment:
      - TZ=Asia/Kolkata

  # Add Grafana for real-time dashboards
  grafana:
    image: grafana/grafana-oss:10.1.0
    container_name: grafana
    hostname: grafana
    networks:
      - factory-network
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

volumes:
  db-mongo: ~
  crate-db: ~
  node-red-data: ~
  grafana-storage: ~
```

### 3.2 Setup Second IoT Agent Service Group

```powershell
# Create service group for JSON IoT Agent
$headersJson = @{ "fiware-service"="manufacturing"; "fiware-servicepath"="/line2" }

$serviceGroupJson = @{
  services = @(
    @{
      apikey = "manufacturing_line2_2025"
      cbroker = "http://orion:1026"
      entity_type = "ManufacturingDevice"
      resource = "/iot/json"
      attributes = @()
      commands = @()
    }
  )
} | ConvertTo-Json -Depth 6

# Use port 4042 for the second IoT Agent
Invoke-RestMethod -Uri "http://localhost:4042/iot/services" -Method Post -Headers $headersJson -ContentType "application/json" -Body $serviceGroupJson
```

## Phase 4: Integrate Open-Source Industrial Emulators

### 4.1 Add OpenPLC Simulator (Real PLC Emulator)

**Update docker-compose-enhanced.yml:**
```yaml
  # OpenPLC Runtime - Real PLC Emulator
  openplc:
    image: openplcproject/openplc:latest
    container_name: openplc
    hostname: openplc
    networks:
      - factory-network
    ports:
      - "8080:8080"
      - "502:502"    # Modbus TCP
      - "44818:44818" # OPC UA
    volumes:
      - openplc-data:/workdir

  # ScadaBR - Open Source SCADA
  scadabr:
    image: scadabr/scadabr:latest
    container_name: scadabr
    hostname: scadabr
    networks:
      - factory-network
    ports:
      - "9090:8080"
    volumes:
      - scadabr-data:/opt/tomcat/webapps/ScadaBR/WEB-INF/db

volumes:
  openplc-data: ~
  scadabr-data: ~
```

### 4.2 Create Modbus IoT Agent for OpenPLC Integration

**Add to docker-compose-enhanced.yml:**
```yaml
  # IoT Agent for Modbus (to connect OpenPLC)
  fiware-iot-agent-modbus:
    image: fiware/iotagent-modbus:1.3.0
    hostname: iot-agent-modbus
    container_name: fiware-iot-agent-modbus
    depends_on:
      - db-mongo
      - fiware-orion
    networks:
      - factory-network
    ports:
      - "4043:4041"
    environment:
      - IOTA_CB_HOST=orion
      - IOTA_CB_PORT=1026
      - IOTA_NORTH_PORT=4041
      - IOTA_REGISTRY_TYPE=mongodb
      - IOTA_MONGO_HOST=db-mongo
      - IOTA_MONGO_PORT=27017
      - IOTA_MONGO_DB=iotagentmodbus
      - IOTA_PROVIDER_URL=http://iot-agent-modbus:4041
      - IOTA_LOG_LEVEL=DEBUG
```

### 4.3 Setup OpenPLC Device in FIWARE

```powershell
# Create service group for Modbus IoT Agent
$headersModbus = @{ "fiware-service"="factory"; "fiware-servicepath"="/realplc" }

$serviceGroupModbus = @{
  services = @(
    @{
      apikey = "real_plc_modbus_2025"
      cbroker = "http://orion:1026"
      entity_type = "RealPLC"
      resource = "/iot/modbus"
      attributes = @()
      commands = @()
    }
  )
} | ConvertTo-Json -Depth 6

Invoke-RestMethod -Uri "http://localhost:4043/iot/services" -Method Post -Headers $headersModbus -ContentType "application/json" -Body $serviceGroupModbus

# Provision OpenPLC as a device
$openPLCDevice = @{
  devices = @(
    @{
      device_id = "openplc001"
      entity_name = "urn:ngsi-ld:RealPLC:openplc001"
      entity_type = "RealPLC"
      apikey = "real_plc_modbus_2025"
      endpoint = "http://openplc:502"
      transport = "MODBUS"
      attributes = @(
        @{ object_id = "h40001"; name = "holding_register_1"; type = "Number"; address = "40001" }
        @{ object_id = "h40002"; name = "holding_register_2"; type = "Number"; address = "40002" }
        @{ object_id = "c00001"; name = "coil_1"; type = "Boolean"; address = "00001" }
        @{ object_id = "c00002"; name = "coil_2"; type = "Boolean"; address = "00002" }
      )
      commands = @(
        @{ name = "set_coil"; type = "command" }
        @{ name = "write_register"; type = "command" }
      )
    }
  )
} | ConvertTo-Json -Depth 8

Invoke-RestMethod -Uri "http://localhost:4043/iot/devices" -Method Post -Headers $headersModbus -ContentType "application/json" -Body $openPLCDevice
```

## Phase 5: Advanced Device Communication Patterns

### 5.1 Command Processing Between Devices

**File: `simulators/plc/plc002_command_handler.py`**
```python
import paho.mqtt.client as mqtt
import json
import time
import threading
from datetime import datetime

class PLC002CommandHandler:
    def __init__(self):
        self.client = mqtt.Client()
        self.client.on_connect = self.on_connect
        self.client.on_message = self.on_message
        
        self.hello_received_count = 0
        self.commands_processed = 0
        
    def on_connect(self, client, userdata, flags, rc):
        print(f"PLC002 Command Handler connected with code {rc}")
        # Subscribe to commands from PLC001 and SCADA
        client.subscribe("ul/4jggokgpepnvsb2uv4s40d59ov/plc002/cmd")
        
    def on_message(self, client, userdata, msg):
        topic = msg.topic
        payload = msg.payload.decode()
        
        print(f"PLC002 Received Command: {payload}")
        
        # Parse the command
        try:
            pairs = payload.split("|")
            cmd_data = {}
            for i in range(0, len(pairs)-1, 2):
                if i+1 < len(pairs):
                    key, value = pairs[i], pairs[i+1]
                    cmd_data[key] = value
                    
            # Process different command types
            if cmd_data.get("from") == "plc001" and cmd_data.get("message") == "hello":
                self.handle_hello_from_plc001(cmd_data)
            elif cmd_data.get("from") == "scada001":
                self.handle_scada_command(cmd_data)
                
        except Exception as e:
            print(f"Error processing command: {e}")
            
    def handle_hello_from_plc001(self, cmd_data):
        """Handle hello message from PLC001"""
        self.hello_received_count += 1
        print(f"PLC002: Received hello #{cmd_data.get('count')} from PLC001")
        
        # Send acknowledgment back
        ack_data = {
            "from": "plc002",
            "to": "plc001", 
            "message": "hello_ack",
            "original_count": cmd_data.get("count", 0),
            "timestamp": datetime.now().isoformat()
        }
        
        ack_payload = "|".join([f"{k}|{v}" for k, v in ack_data.items()])
        ack_topic = "ul/4jggokgpepnvsb2uv4s40d59ov/plc001/cmd"
        
        self.client.publish(ack_topic, ack_payload)
        print(f"PLC002: Sent hello acknowledgment to PLC001")
        
        # Update our status based on peer communication
        self.update_status_attributes()
        
    def handle_scada_command(self, cmd_data):
        """Handle commands from SCADA"""
        self.commands_processed += 1
        command_type = cmd_data.get("command", "unknown")
        
        print(f"PLC002: Processing SCADA command: {command_type}")
        
        # Simulate command processing
        if command_type == "start_production":
            # Send confirmation back to SCADA
            self.send_command_response("scada001", "production_started", cmd_data)
        elif command_type == "stop_production":
            self.send_command_response("scada001", "production_stopped", cmd_data)
            
    def send_command_response(self, target, response_type, original_cmd):
        """Send command response to another device"""
        response_data = {
            "from": "plc002",
            "to": target,
            "response": response_type,
            "original_cmd": original_cmd.get("command", ""),
            "timestamp": datetime.now().isoformat(),
            "status": "completed"
        }
        
        response_payload = "|".join([f"{k}|{v}" for k, v in response_data.items()])
        response_topic = f"ul/4jggokgpepnvsb2uv4s40d59ov/{target}/cmd"
        
        self.client.publish(response_topic, response_payload)
        print(f"PLC002: Sent response '{response_type}' to {target}")
        
    def update_status_attributes(self):
        """Update PLC002 status based on peer communications"""
        status_data = {
            "hr": self.hello_received_count,        # hello_received
            "cp": self.commands_processed,          # commands_processed
            "cs": "Connected" if self.hello_received_count > 0 else "Isolated",  # connection_status
            "lu": datetime.now().isoformat()        # last_update
        }
        
        status_payload = "|".join([f"{k}|{v}" for k, v in status_data.items()])
        status_topic = "ul/4jggokgpepnvsb2uv4s40d59ov/plc002/attrs"
        
        self.client.publish(status_topic, status_payload)
        
    def run(self):
        try:
            self.client.connect("localhost", 1883, 60)
            self.client.loop_start()
            
            # Keep the command handler running
            while True:
                time.sleep(1)
                
        except KeyboardInterrupt:
            print("Stopping PLC002 Command Handler...")
            self.client.loop_stop()
            self.client.disconnect()

if __name__ == "__main__":
    handler = PLC002CommandHandler()
    handler.run()
```

## Phase 6: Real-World Process Simulation

### 6.1 Color Sorting Process Implementation

**File: `processes/color_sorting_system.py`**
```python
import paho.mqtt.client as mqtt
import json
import time
import random
import threading
from datetime import datetime
from enum import Enum

class Color(Enum):
    RED = "red"
    BLUE = "blue" 
    GREEN = "green"
    YELLOW = "yellow"
    UNKNOWN = "unknown"

class ColorSortingSystem:
    def __init__(self):
        self.client = mqtt.Client()
        self.client.on_connect = self.on_connect
        
        # System state
        self.current_block = None
        self.sorted_counts = {color.value: 0 for color in Color}
        self.total_processed = 0
        self.system_running = True
        
        # Components
        self.conveyor_speed = 100  # %
        self.camera_active = True
        self.sorter_positions = {"red": 0, "blue": 90, "green": 180, "yellow": 270}
        
    def on_connect(self, client, userdata, flags, rc):
        print(f"Color Sorting System connected with code {rc}")
        
    def generate_block(self):
        """Generate a new block with random color"""
        colors = [Color.RED, Color.BLUE, Color.GREEN, Color.YELLOW, Color.UNKNOWN]
        weights = [0.25, 0.25, 0.25, 0.20, 0.05]  # Higher chance for known colors
        
        color = random.choices(colors, weights=weights)[0]
        
        block = {
            "id": f"BLOCK_{self.total_processed + 1:04d}",
            "color": color.value,
            "timestamp": datetime.now().isoformat(),
            "confidence": random.uniform(0.85, 0.99) if color != Color.UNKNOWN else random.uniform(0.30, 0.60),
            "size": random.choice(["small", "medium", "large"]),
            "position": 0  # Starting position on conveyor
        }
        
        return block
        
    def detect_color(self, block):
        """Simulate camera color detection"""
        camera_data = {
            "block_id": block["id"],
            "detected_color": block["color"],
            "confidence": block["confidence"],
            "detection_time": datetime.now().isoformat(),
            "camera_status": "active" if self.camera_active else "inactive",
            "image_quality": random.uniform(0.8, 1.0)
        }
        
        # Publish camera detection data
        camera_payload = "|".join([f"{k}|{v}" for k, v in camera_data.items()])
        camera_topic = "ul/4jggokgpepnvsb2uv4s40d59ov/camera001/attrs"
        
        self.client.publish(camera_topic, camera_payload)
        print(f"Camera detected: {block['id']} -> {block['color']} (confidence: {block['confidence']:.2f})")
        
        return camera_data
        
    def control_sorter(self, block, detection_result):
        """Control sorting mechanism based on detection"""
        target_position = self.sorter_positions.get(block["color"], 0)
        
        # Send command to sorting actuator (simulated as RTU001)
        sorter_command = {
            "block_id": block["id"],
            "target_color": block["color"],
            "target_position": target_position,
            "action": "sort",
            "timestamp": datetime.now().isoformat()
        }
        
        sorter_payload = "|".join([f"{k}|{v}" for k, v in sorter_command.items()])
        sorter_topic = "ul/4jggokgpepnvsb2uv4s40d59ov/rtu001/cmd"
        
        self.client.publish(sorter_topic, sorter_payload)
        
        # Update sorting statistics
        if block["color"] != "unknown":
            self.sorted_counts[block["color"]] += 1
        
        self.total_processed += 1
        
        print(f"Sorted {block['id']} to {block['color']} bin (position: {target_position}°)")
        
    def update_plc_control(self):
        """Send process status to PLC for overall control"""
        plc_data = {
            "cs": self.conveyor_speed,              # conveyor_speed
            "tp": self.total_processed,             # total_processed
            "rc": self.sorted_counts["red"],        # red_count
            "bc": self.sorted_counts["blue"],       # blue_count
            "gc": self.sorted_counts["green"],      # green_count
            "yc": self.sorted_counts["yellow"],     # yellow_count
            "uc": self.sorted_counts["unknown"],    # unknown_count
            "ef": round((self.total_processed - self.sorted_counts["unknown"]) / max(self.total_processed, 1) * 100, 1),  # efficiency
            "sr": "running" if self.system_running else "stopped",  # system_running
            "lt": datetime.now().isoformat()        # last_update
        }
        
        plc_payload = "|".join([f"{k}|{v}" for k, v in plc_data.items()])
        plc_topic = "ul/4jggokgpepnvsb2uv4s40d59ov/plc002/attrs"  # PLC002 controls sorting line
        
        self.client.publish(plc_topic, plc_payload)
        
    def update_scada_overview(self):
        """Send high-level metrics to SCADA"""
        scada_data = {
            "line_name": "Color_Sorting_Line_1",
            "total_blocks": self.total_processed,
            "sorting_efficiency": round((self.total_processed - self.sorted_counts["unknown"]) / max(self.total_processed, 1) * 100, 1),
            "throughput_per_hour": round(self.total_processed * 3600 / max(time.time() - self.start_time, 1), 0),
            "red_percentage": round(self.sorted_counts["red"] / max(self.total_processed, 1) * 100, 1),
            "blue_percentage": round(self.sorted_counts["blue"] / max(self.total_processed, 1) * 100, 1),
            "green_percentage": round(self.sorted_counts["green"] / max(self.total_processed, 1) * 100, 1),
            "yellow_percentage": round(self.sorted_counts["yellow"] / max(self.total_processed, 1) * 100, 1),
            "system_status": "operational" if self.system_running else "stopped",
            "last_update": datetime.now().isoformat()
        }
        
        scada_payload = "|".join([f"{k}|{v}" for k, v in scada_data.items()])
        scada_topic = "ul/4jggokgpepnvsb2uv4s40d59ov/scada001/attrs"
        
        self.client.publish(scada_topic, scada_payload)
        
    def process_loop(self):
        """Main processing loop"""
        self.start_time = time.time()
        
        while self.system_running:
            try:
                # Generate new block every 3-5 seconds
                block = self.generate_block()
                self.current_block = block
                
                print(f"\n--- Processing {block['id']} ---")
                
                # Step 1: Camera detection
                detection_result = self.detect_color(block)
                time.sleep(1)  # Processing time
                
                # Step 2: Sorting control
                self.control_sorter(block, detection_result)
                time.sleep(1)  # Mechanical sorting time
                
                # Step 3: Update PLC with process data
                self.update_plc_control()
                
                # Step 4: Update SCADA with overview (every 10 blocks)
                if self.total_processed % 10 == 0:
                    self.update_scada_overview()
                
                # Step 5: Print status
                print(f"Status: {self.total_processed} processed, efficiency: {((self.total_processed - self.sorted_counts['unknown']) / max(self.total_processed, 1) * 100):.1f}%")
                
                # Wait for next block
                time.sleep(random.uniform(3.0, 5.0))
                
            except Exception as e:
                print(f"Error in process loop: {e}")
                time.sleep(5)
                
    def run(self):
        try:
            self.client.connect("localhost", 1883, 60)
            self.client.loop_start()
            
            # Start processing in separate thread
            process_thread = threading.Thread(target=self.process_loop)
            process_thread.daemon = True
            process_thread.start()
            
            print("Color Sorting System started. Press Ctrl+C to stop.")
            
            while True:
                time.sleep(1)
                
        except KeyboardInterrupt:
            print("\nStopping Color Sorting System...")
            self.system_running = False
            self.client.loop_stop()
            self.client.disconnect()

if __name__ == "__main__":
    sorting_system = ColorSortingSystem()
    sorting_system.run()
```

## Phase 7: Setup and Deployment Scripts

### 7.1 Enhanced Start Script

**File: `start-enhanced-factory.ps1`**
```powershell
# Enhanced FIWARE Factory Startup Script
param(
    [switch]$IncludeRealPLC,
    [switch]$SkipHealthCheck
)

Write-Host "Starting Enhanced FIWARE Factory System..." -ForegroundColor Green

# Start Docker Compose with enhanced services
Write-Host "Starting Docker services..." -ForegroundColor Yellow
docker compose -f .\docker-compose-enhanced.yml up -d

if (!$SkipHealthCheck) {
    Write-Host "Waiting for services to start..." -ForegroundColor Yellow
    Start-Sleep -Seconds 30

    # Health check
    Write-Host "Performing health check..." -ForegroundColor Yellow
    $services = @(
        @{ Name="Orion"; URL="http://localhost:1026/version" }
        @{ Name="IoT Agent UL"; URL="http://localhost:4041/iot/about" }
        @{ Name="IoT Agent JSON"; URL="http://localhost:4042/iot/about" }
        @{ Name="QuantumLeap"; URL="http://localhost:8668/version" }
        @{ Name="CrateDB"; URL="http://localhost:4200" }
        @{ Name="Node-RED"; URL="http://localhost:1880" }
        @{ Name="Grafana"; URL="http://localhost:3000" }
    )

    if ($IncludeRealPLC) {
        $services += @{ Name="OpenPLC"; URL="http://localhost:8080" }
        $services += @{ Name="IoT Agent Modbus"; URL="http://localhost:4043/iot/about" }
    }

    foreach ($service in $services) {
        try {
            $response = Invoke-WebRequest $service.URL -UseBasicParsing -TimeoutSec 10
            Write-Host "  ✓ $($service.Name): OK" -ForegroundColor Green
        } catch {
            Write-Host "  ✗ $($service.Name): Failed" -ForegroundColor Red
        }
    }
}

Write-Host "Starting device simulators..." -ForegroundColor Yellow

# Start enhanced simulators
$simulators = @(
    "python simulators/sensor/sensor_simulator.py",
    "python simulators/sensor/sensor002_simulator.py", 
    "python simulators/sensor/sensor003_simulator.py",
    "python simulators/sensor/sensor004_simulator.py",
    "python simulators/plc/enhanced_plc001_simulator.py",
    "python simulators/plc/plc002_command_handler.py",
    "python simulators/scada/scada_simulator.py",
    "python simulators/rtu/rtu_simulator.py",
    "python processes/color_sorting_system.py"
)

foreach ($simulator in $simulators) {
    Start-Process -FilePath "powershell" -ArgumentList "-Command", $simulator -WindowStyle Minimized
    Write-Host "  Started: $simulator" -ForegroundColor Cyan
    Start-Sleep -Seconds 2
}

Write-Host "`nEnhanced FIWARE Factory System is running!" -ForegroundColor Green
Write-Host "Access points:" -ForegroundColor Yellow
Write-Host "  - Orion Context Broker: http://localhost:1026" -ForegroundColor White
Write-Host "  - IoT Agent UltraLight: http://localhost:4041" -ForegroundColor White  
Write-Host "  - IoT Agent JSON: http://localhost:4042" -ForegroundColor White
Write-Host "  - Node-RED: http://localhost:1880" -ForegroundColor White
Write-Host "  - Grafana: http://localhost:3000 (admin/admin)" -ForegroundColor White
Write-Host "  - CrateDB: http://localhost:4200" -ForegroundColor White

if ($IncludeRealPLC) {
    Write-Host "  - OpenPLC: http://localhost:8080 (openplc/openplc)" -ForegroundColor White
    Write-Host "  - IoT Agent Modbus: http://localhost:4043" -ForegroundColor White
}

Write-Host "`nPress Ctrl+C to stop all services." -ForegroundColor Yellow
```

### 7.2 Device Provisioning Script

**File: `provision-enhanced-devices.ps1`**
```powershell
# Enhanced Device Provisioning Script
Write-Host "Provisioning enhanced device fleet..." -ForegroundColor Green

$headers = @{ "fiware-service"="factory"; "fiware-servicepath"="/" }

# Provision additional sensors
Write-Host "Provisioning additional sensors..." -ForegroundColor Yellow

# [Include all the PowerShell commands from Phase 1.1 here]

Write-Host "Provisioning enhanced PLCs..." -ForegroundColor Yellow

# Enhanced PLC001 with command processing
$enhancedPLC001 = @{
  devices = @(
    @{
      device_id = "plc001"
      entity_name = "urn:ngsi-ld:ManufacturingMachine:plc001"
      entity_type = "ManufacturingMachine"
      transport = "MQTT"
      apikey = "4jggokgpepnvsb2uv4s40d59ov"
      explicitAttrs = $true
      attributes = @(
        @{ object_id = "te"; name = "temperature"; type = "Number" }
        @{ object_id = "pr"; name = "pressure"; type = "Number" }
        @{ object_id = "sh"; name = "system_health"; type = "Number" }
        @{ object_id = "mp"; name = "max_pressure"; type = "Number" }
        @{ object_id = "va"; name = "vibration_alarm"; type = "Boolean" }
        @{ object_id = "sc"; name = "sensor_count"; type = "Number" }
        @{ object_id = "hr"; name = "hello_received"; type = "Number" }
        @{ object_id = "cp"; name = "commands_processed"; type = "Number" }
        @{ object_id = "cs"; name = "connection_status"; type = "Text" }
        @{ object_id = "pd"; name = "parent_device"; type = "Text" }
        @{ object_id = "cd"; name = "child_devices"; type = "Text" }
      )
      commands = @(
        @{ name = "hello"; type = "command" }
        @{ name = "status_request"; type = "command" }
        @{ name = "emergency_stop"; type = "command" }
      )
    }
  )
} | ConvertTo-Json -Depth 8

try {
    Invoke-RestMethod -Uri "http://localhost:4041/iot/devices" -Method Post -Headers $headers -ContentType "application/json" -Body $enhancedPLC001
    Write-Host "  ✓ Enhanced PLC001 provisioned" -ForegroundColor Green
} catch {
    Write-Host "  ✗ Failed to provision Enhanced PLC001: $($_.Exception.Message)" -ForegroundColor Red
}

Write-Host "Enhanced device provisioning complete!" -ForegroundColor Green
```

### 7.3 Real-World Integration Commands

```powershell
# Setup Node-RED flows for device orchestration
Write-Host "Setting up Node-RED flows..." -ForegroundColor Yellow

# Create a flow that listens to Orion entities and triggers actions
$nodeRedFlow = @{
    flows = @(
        @{
            id = "factory-orchestration"
            type = "tab"
            label = "Factory Orchestration"
        },
        @{
            id = "orion-listener"
            type = "http in"
            z = "factory-orchestration"
            name = "Orion Notifications"
            url = "/orion/notify"
            method = "post"
        },
        @{
            id = "process-notification"
            type = "function"
            z = "factory-orchestration"
            name = "Process Entity Change"
            func = "if (msg.payload.data && msg.payload.data[0]) { msg.payload = msg.payload.data[0]; } return msg;"
        },
        @{
            id = "mqtt-command"
            type = "mqtt out"
            z = "factory-orchestration"
            name = "Send Command"
            topic = "ul/4jggokgpepnvsb2uv4s40d59ov/{{entityId}}/cmd"
            broker = "mqtt-broker"
        }
    )
} | ConvertTo-Json -Depth 10

# Note: This would be imported via Node-RED UI at http://localhost:1880

Write-Host "Node-RED setup complete. Access at: http://localhost:1880" -ForegroundColor Green
```

## Summary: What This Achieves

### Real-World Features Added:
1. **Multiple sensors per PLC** - More realistic industrial hierarchy
2. **Device-to-device communication** - PLCs talking to each other via FIWARE commands
3. **Process simulation** - Color sorting system with real workflow
4. **Multiple IoT Agents** - UltraLight, JSON, and Modbus protocols
5. **Open-source emulators** - OpenPLC and ScadaBR integration
6. **Visual programming** - Node-RED for device orchestration
7. **Real-time dashboards** - Grafana for monitoring

### FIWARE Components Utilized:
- **3 IoT Agents** (UltraLight, JSON, Modbus)
- **Command processing** between devices
- **Multiple service groups** for different protocols
- **Entity relationships** and hierarchical modeling
- **Historical persistence** with QuantumLeap
- **Real-time notifications** and subscriptions

### How to Deploy:
1. Update your docker-compose-complete.yml with the enhanced version
2. Run the provisioning scripts to add new devices
3. Start the enhanced simulators and process systems
4. Monitor via Grafana dashboards and Node-RED flows

This transforms your system from "just sending data" to a **real industrial automation platform** using FIWARE's full capabilities!
