# Enhanced FIWARE Digital Twin Implementation Guide

## Overview
This guide will help you transform your current Python-based simulator setup into a realistic industrial automation environment using actual industrial simulators and the full power of FIWARE ecosystem.

## Phase 1: Real Industrial Simulators Setup

### 1.1 OpenPLC Runtime Installation

**Download and Install OpenPLC:**
```bash
# For Linux/Raspberry Pi
git clone https://github.com/thiagoralves/OpenPLC_v3.git
cd OpenPLC_v3
./install.sh linux

# For Windows (use precompiled)
# Download from: https://autonomylogic.com/download/
```

**Configuration for FIWARE Integration:**
- Enable Modbus TCP on port 502
- Configure IP address: 192.168.1.10
- Set slave ID: 1

### 1.2 ModbusPal Modbus Simulator

**Download and Setup:**
```bash
# Download ModbusPal.jar
wget https://sourceforge.net/projects/modbuspal/files/ModbusPal.jar
java -jar ModbusPal.jar
```

**Configuration:**
- Create 5 Modbus slaves (IDs: 1-5)
- Each slave: 10 holding registers, 8 coils
- Auto-generate random values every 5 seconds

### 1.3 Factory I/O 3D Simulator

**Installation:**
- Download 30-day trial from https://factoryio.com
- Install on Windows machine
- Configure for Modbus TCP communication

**Sample Scenario Setup:**
- Conveyor belt with sensors
- Sorting system with 3 colors
- Tank filling station

### 1.4 OpenSCADA Installation

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install openscada

# Or download from: http://oscada.org/main/download/
```

## Phase 2: Multiple IoT Agents Configuration

### 2.1 Enhanced Docker Compose

**Create `docker-compose-enhanced.yml`:**
```yaml
version: '3.8'
services:
  # Multiple Orion Context Brokers
  orion-main:
    image: fiware/orion-ld:1.3.0
    hostname: orion-main
    container_name: fiware-orion-main
    depends_on:
      - mongo-db-main
    ports:
      - "1026:1026"
    command: -dbhost mongo-db-main -logLevel DEBUG
    healthcheck:
      test: curl --fail -s http://orion-main:1026/version || exit 1

  orion-secondary:
    image: fiware/orion:3.7.0
    hostname: orion-secondary
    container_name: fiware-orion-secondary
    depends_on:
      - mongo-db-secondary
    ports:
      - "1027:1026"
    command: -dbhost mongo-db-secondary -logLevel DEBUG

  # IoT Agent for Modbus
  iot-agent-modbus:
    image: iotagent4fiware/iotagent-modbus:latest
    hostname: iot-agent-modbus
    container_name: fiware-iot-agent-modbus
    depends_on:
      - mongo-db-iot
      - orion-main
    ports:
      - "4041:4041"
    environment:
      - IOTA_CB_HOST=orion-main
      - IOTA_CB_PORT=1026
      - IOTA_NORTH_PORT=4041
      - IOTA_REGISTRY_TYPE=mongodb
      - IOTA_MONGO_HOST=mongo-db-iot
      - IOTA_MONGO_PORT=27017
      - IOTA_MONGO_DB=iotagentmodbus
      - IOTA_SERVICE=factory
      - IOTA_SUBSERVICE=/modbus

  # IoT Agent for OPC-UA
  iot-agent-opcua:
    image: iotagent4fiware/iotagent-opcua:latest
    hostname: iot-agent-opcua
    container_name: fiware-iot-agent-opcua
    depends_on:
      - mongo-db-iot
      - orion-main
    ports:
      - "4042:4041"
    environment:
      - IOTA_CB_HOST=orion-main
      - IOTA_CB_PORT=1026
      - IOTA_NORTH_PORT=4041
      - IOTA_REGISTRY_TYPE=mongodb
      - IOTA_MONGO_HOST=mongo-db-iot

  # QuantumLeap for time-series data
  quantumleap:
    image: smartsdk/quantumleap:0.8.3
    hostname: quantumleap
    container_name: fiware-quantumleap
    ports:
      - "8668:8668"
    depends_on:
      - crate-db
    environment:
      - CRATE_HOST=crate-db
      - LOGLEVEL=DEBUG

  # CrateDB for time-series storage
  crate-db:
    image: crate:4.6.6
    hostname: crate-db
    container_name: db-crate
    ports:
      - "4200:4200"
      - "4300:4300"
    environment:
      - CRATE_HEAP_SIZE=1g

  # Databases
  mongo-db-main:
    image: mongo:4.4
    hostname: mongo-db-main
    container_name: db-mongo-main
    expose:
      - "27017"
    ports:
      - "27017:27017"
    volumes:
      - mongo-db-main:/data/db

  mongo-db-secondary:
    image: mongo:4.4
    hostname: mongo-db-secondary
    container_name: db-mongo-secondary
    expose:
      - "27017"
    ports:
      - "27018:27017"
    volumes:
      - mongo-db-secondary:/data/db

volumes:
  mongo-db-main: ~
  mongo-db-secondary: ~
```

### 2.2 Multi-Service Configuration

**Create service provisioning scripts:**

```bash
#!/bin/bash
# provision-services.sh

# Provision Modbus service
curl -X POST \
  'http://localhost:4041/iot/services' \
  -H 'fiware-service: factory' \
  -H 'fiware-servicepath: /plc' \
  -H 'Content-Type: application/json' \
  -d '{
    "services": [
      {
        "apikey": "modbus-key-001",
        "cbroker": "http://orion-main:1026",
        "entity_type": "PLC",
        "resource": ""
      }
    ]
  }'

# Provision OPC-UA service
curl -X POST \
  'http://localhost:4042/iot/services' \
  -H 'fiware-service: factory' \
  -H 'fiware-servicepath: /scada' \
  -H 'Content-Type: application/json' \
  -d '{
    "services": [
      {
        "apikey": "opcua-key-001",
        "cbroker": "http://orion-main:1026",
        "entity_type": "SCADA",
        "resource": ""
      }
    ]
  }'
```

## Phase 3: Advanced FIWARE Features

### 3.1 Custom Context Providers

**Create `custom-context-provider.py`:**
```python
#!/usr/bin/env python3
"""
Custom Context Provider for complex industrial calculations
"""
import flask
import requests
import json
from datetime import datetime

app = flask.Flask(__name__)

class IndustrialContextProvider:
    def __init__(self):
        self.orion_url = "http://localhost:1026"
        
    def calculate_oee(self, entity_id):
        """Calculate Overall Equipment Effectiveness"""
        # Get entity data
        response = requests.get(f"{self.orion_url}/v2/entities/{entity_id}")
        if response.status_code != 200:
            return None
            
        entity = response.json()
        
        # OEE calculation logic
        availability = entity.get('availability', {}).get('value', 0)
        performance = entity.get('performance', {}).get('value', 0)
        quality = entity.get('quality', {}).get('value', 0)
        
        oee = (availability * performance * quality) / 10000  # Convert to percentage
        
        return {
            "oee": {
                "type": "Number",
                "value": round(oee, 2),
                "metadata": {
                    "timestamp": {
                        "type": "DateTime",
                        "value": datetime.utcnow().isoformat() + "Z"
                    },
                    "calculationType": {
                        "type": "Text",
                        "value": "real-time"
                    }
                }
            }
        }

@app.route('/v2/entities/<entity_id>', methods=['GET'])
def get_entity(entity_id):
    provider = IndustrialContextProvider()
    
    # Handle specific calculations based on entity type
    if entity_id.startswith('machine:'):
        result = provider.calculate_oee(entity_id)
        if result:
            return flask.jsonify(result)
    
    return flask.jsonify({"error": "Entity not found"}), 404

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3001, debug=True)
```

### 3.2 Inter-Device Communication Setup

**Create `device-communication-manager.py`:**
```python
#!/usr/bin/env python3
"""
Device Communication Manager for PLC-to-PLC communication
"""
import paho.mqtt.client as mqtt
import requests
import json
import time
import logging

class DeviceCommunicationManager:
    def __init__(self):
        self.mqtt_client = mqtt.Client()
        self.mqtt_client.on_connect = self.on_connect
        self.mqtt_client.on_message = self.on_message
        
        self.orion_url = "http://localhost:1026"
        self.devices = {
            'plc001': {'ip': '192.168.1.10', 'type': 'modbus'},
            'plc002': {'ip': '192.168.1.11', 'type': 'modbus'},
            'scada001': {'ip': '192.168.1.20', 'type': 'opcua'}
        }
        
    def on_connect(self, client, userdata, flags, rc):
        logging.info(f"Connected to MQTT broker with result code {rc}")
        # Subscribe to device communication topics
        client.subscribe("device/+/command")
        client.subscribe("device/+/heartbeat")
        
    def on_message(self, client, userdata, msg):
        try:
            topic_parts = msg.topic.split('/')
            device_id = topic_parts[1]
            message_type = topic_parts[2]
            
            payload = json.loads(msg.payload.decode())
            
            if message_type == "heartbeat":
                self.handle_heartbeat(device_id, payload)
            elif message_type == "command":
                self.handle_device_command(device_id, payload)
                
        except Exception as e:
            logging.error(f"Error processing message: {e}")
            
    def handle_heartbeat(self, device_id, payload):
        """Handle device heartbeat messages"""
        # Update device status in FIWARE
        entity_data = {
            "id": f"urn:ngsi-ld:Device:{device_id}",
            "type": "Device",
            "status": {
                "type": "Text",
                "value": "online"
            },
            "lastSeen": {
                "type": "DateTime",
                "value": payload.get('timestamp', time.time())
            }
        }
        
        # Send to Orion Context Broker
        requests.patch(
            f"{self.orion_url}/v2/entities/urn:ngsi-ld:Device:{device_id}/attrs",
            headers={'Content-Type': 'application/json'},
            json=entity_data
        )
        
    def send_plc_command(self, source_plc, target_plc, command_data):
        """Send command from one PLC to another through FIWARE"""
        # Publish command via MQTT
        self.mqtt_client.publish(
            f"device/{target_plc}/command",
            json.dumps({
                "source": source_plc,
                "timestamp": time.time(),
                "command": command_data
            })
        )
        
        logging.info(f"Command sent from {source_plc} to {target_plc}: {command_data}")

if __name__ == '__main__':
    logging.basicConfig(level=logging.INFO)
    manager = DeviceCommunicationManager()
    manager.mqtt_client.connect("localhost", 1883, 60)
    manager.mqtt_client.loop_forever()
```

## Phase 4: Industrial Process Implementation

### 4.1 Color Sorting Process

**Create OpenPLC Ladder Logic for sorting:**

1. **Sensor Inputs:**
   - I0.0: Object detected sensor
   - I0.1: Red color sensor
   - I0.2: Green color sensor
   - I0.3: Blue color sensor

2. **Actuator Outputs:**
   - Q0.0: Conveyor motor
   - Q0.1: Red sorting valve
   - Q0.2: Green sorting valve
   - Q0.3: Blue sorting valve

3. **Ladder Logic Structure:**
```
|--[I0.0]--[Timer_1s]--[Move_Red]--[Q0.1]--|
|--[I0.1]--[Timer_1s]--[Move_Green]--[Q0.2]--|
|--[I0.2]--[Timer_1s]--[Move_Blue]--[Q0.3]--|
```

### 4.2 Process Monitoring Dashboard

**Create `process-monitor.html`:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Industrial Process Monitor</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        .dashboard { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; padding: 20px; }
        .widget { border: 1px solid #ccc; padding: 15px; border-radius: 5px; }
        .status-online { color: green; }
        .status-offline { color: red; }
    </style>
</head>
<body>
    <h1>Factory Digital Twin Dashboard</h1>
    
    <div class="dashboard">
        <div class="widget">
            <h3>PLC Status</h3>
            <div id="plc-status"></div>
        </div>
        
        <div class="widget">
            <h3>Production Rate</h3>
            <canvas id="productionChart"></canvas>
        </div>
        
        <div class="widget">
            <h3>Color Sorting Stats</h3>
            <canvas id="colorChart"></canvas>
        </div>
        
        <div class="widget">
            <h3>System Health</h3>
            <div id="system-health"></div>
        </div>
    </div>

    <script>
        // WebSocket connection to FIWARE
        const ws = new WebSocket('ws://localhost:8080/notifications');
        
        ws.onmessage = function(event) {
            const data = JSON.parse(event.data);
            updateDashboard(data);
        };
        
        function updateDashboard(data) {
            // Update PLC status
            const plcStatus = document.getElementById('plc-status');
            plcStatus.innerHTML = `
                <p>PLC001: <span class="status-${data.plc001?.status || 'offline'}">${data.plc001?.status || 'Offline'}</span></p>
                <p>PLC002: <span class="status-${data.plc002?.status || 'offline'}">${data.plc002?.status || 'Offline'}</span></p>
            `;
        }
        
        // Initialize charts
        const productionCtx = document.getElementById('productionChart').getContext('2d');
        const productionChart = new Chart(productionCtx, {
            type: 'line',
            data: {
                labels: [],
                datasets: [{
                    label: 'Items/Hour',
                    data: [],
                    borderColor: 'rgb(75, 192, 192)',
                    tension: 0.1
                }]
            }
        });
    </script>
</body>
</html>
```

## Phase 5: Deployment and Testing

### 5.1 Complete Startup Script

**Create `start-enhanced-system.sh`:**
```bash
#!/bin/bash

echo "Starting Enhanced FIWARE Digital Twin System..."

# Start containers
docker-compose -f docker-compose-enhanced.yml up -d

# Wait for services to be ready
echo "Waiting for services to start..."
sleep 30

# Provision services
./provision-services.sh

# Start custom components
python3 custom-context-provider.py &
python3 device-communication-manager.py &

# Start industrial simulators
echo "Starting industrial simulators..."
# ModbusPal
java -jar ModbusPal.jar -headless -config modbus-config.json &

# OpenPLC (if on same machine)
# sudo systemctl start openplc

echo "System startup complete!"
echo "Access points:"
echo "- Main Dashboard: http://localhost:3000"
echo "- Orion Context Broker: http://localhost:1026"
echo "- QuantumLeap: http://localhost:8668"
echo "- CrateDB Admin: http://localhost:4200"
```

### 5.2 Testing and Validation

**Create test scenarios:**

1. **Basic Connectivity Test:**
```bash
# Test Modbus connectivity
curl -X GET 'http://localhost:1026/v2/entities?type=PLC'

# Test OPC-UA connectivity  
curl -X GET 'http://localhost:1026/v2/entities?type=SCADA'
```

2. **Process Flow Test:**
```bash
# Simulate color sorting process
curl -X PATCH 'http://localhost:1026/v2/entities/urn:ngsi-ld:Machine:sorting001/attrs' \
  -H 'Content-Type: application/json' \
  -d '{
    "objectDetected": {"type": "Boolean", "value": true},
    "color": {"type": "Text", "value": "red"}
  }'
```

## Benefits of Enhanced Setup

1. **Realistic Environment:** Uses actual industrial protocols and simulators
2. **Learning Value:** Exposure to real OT technologies like Modbus, OPC-UA
3. **Scalability:** Multiple FIWARE components for different functions
4. **Industry Standards:** Follows IEC 61131-3 for PLC programming
5. **Full FIWARE Stack:** Utilizes advanced features like QuantumLeap, Cygnus
6. **Professional Development:** Skills transferable to real industrial projects

## Next Steps

1. Start with Phase 1 - Install real simulators
2. Test each component individually before integration
3. Use the provided scripts and configurations
4. Monitor system performance and logs
5. Expand with additional industrial processes
6. Implement machine learning for predictive maintenance

This enhanced setup will give you hands-on experience with real industrial automation tools while leveraging the full power of the FIWARE ecosystem!
