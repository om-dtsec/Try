# Setup Commands for Twin4 Project

## Windows 11 Setup Instructions

### 1. Create Directory Structure
```powershell
# Navigate to D: drive
D:
cd D:\

# Create main project directory
mkdir Twin4
cd Twin4

# Create all subdirectories
mkdir config
mkdir config\mosquitto
mkdir config\grafana
mkdir config\agents

mkdir simulators
mkdir simulators\plc
mkdir simulators\plc\plc1
mkdir simulators\plc\plc2
mkdir simulators\plc\plc3
mkdir simulators\sensors
mkdir simulators\sensors\temperature
mkdir simulators\sensors\pressure
mkdir simulators\sensors\vibration
mkdir simulators\sensors\camera
mkdir simulators\actuators
mkdir simulators\actuators\motor
mkdir simulators\actuators\valve
mkdir simulators\rtu
mkdir simulators\scada
mkdir simulators\external
mkdir simulators\external\softplc
mkdir simulators\external\openplc
mkdir simulators\external\openplc\programs

mkdir agents
mkdir agents\agent1
mkdir agents\agent2

mkdir data
mkdir data\smart_data_models
mkdir data\subscriptions

mkdir scripts

mkdir web_interface
mkdir web_interface\dashboard
mkdir web_interface\dashboard\css
mkdir web_interface\dashboard\js
mkdir web_interface\dashboard\components
```

### 2. Install Required Software
```powershell
# Install Docker Desktop for Windows
# Download from: https://www.docker.com/products/docker-desktop/

# Install Python 3.9+ 
# Download from: https://www.python.org/downloads/

# Install Git for Windows
# Download from: https://git-scm.com/download/win

# Verify installations
docker --version
python --version
git --version
```

### 3. Create Configuration Files

#### Mosquitto Configuration (config/mosquitto/mosquitto.conf)
```bash
# Create mosquitto config file
echo "listener 1883
listener 9001
protocol websockets
allow_anonymous true
log_type all
log_dest stdout" > config\mosquitto\mosquitto.conf
```

#### Main Docker Compose File
```powershell
# Copy the enhanced docker-compose.yml content to the root directory
# This file was provided in the enhanced-docker-compose.md
```

### 4. Create Python Virtual Environment
```powershell
# Create virtual environment
python -m venv twin4-env

# Activate virtual environment
twin4-env\Scripts\activate

# Install global requirements
pip install paho-mqtt requests flask python-opcua pymodbus asyncua
```

### 5. Generate All Simulator Files

#### Create Enhanced PLC Simulator Files
```powershell
# For each PLC directory (plc1, plc2, plc3), create:
# - plc_simulator.py (use the enhanced PLC simulator code)
# - requirements.txt
# - Dockerfile

# Create requirements.txt for PLCs
echo "paho-mqtt==1.6.1
requests==2.28.2
python-opcua==0.98.13
pymodbus==3.0.2
asyncua==1.0.4" > simulators\plc\plc1\requirements.txt

# Copy to other PLC directories
copy simulators\plc\plc1\requirements.txt simulators\plc\plc2\requirements.txt
copy simulators\plc\plc1\requirements.txt simulators\plc\plc3\requirements.txt

# Create basic Dockerfile for PLCs
echo "FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY plc_simulator.py .
CMD [\"python\", \"plc_simulator.py\"]" > simulators\plc\plc1\Dockerfile

# Copy to other directories
copy simulators\plc\plc1\Dockerfile simulators\plc\plc2\Dockerfile
copy simulators\plc\plc1\Dockerfile simulators\plc\plc3\Dockerfile
```

#### Create Sensor Simulator Files
```powershell
# Temperature sensor
echo "paho-mqtt==1.6.1
requests==2.28.2
numpy==1.24.3" > simulators\sensors\temperature\requirements.txt

echo "FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY sensor_simulator.py .
CMD [\"python\", \"sensor_simulator.py\"]" > simulators\sensors\temperature\Dockerfile

# Copy to other sensor directories
copy simulators\sensors\temperature\requirements.txt simulators\sensors\pressure\requirements.txt
copy simulators\sensors\temperature\requirements.txt simulators\sensors\vibration\requirements.txt
copy simulators\sensors\temperature\requirements.txt simulators\sensors\camera\requirements.txt

copy simulators\sensors\temperature\Dockerfile simulators\sensors\pressure\Dockerfile
copy simulators\sensors\temperature\Dockerfile simulators\sensors\vibration\Dockerfile
copy simulators\sensors\temperature\Dockerfile simulators\sensors\camera\Dockerfile
```

#### Create Actuator Simulator Files
```powershell
# Motor actuator
echo "paho-mqtt==1.6.1
requests==2.28.2" > simulators\actuators\motor\requirements.txt

echo "FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY motor_simulator.py .
CMD [\"python\", \"motor_simulator.py\"]" > simulators\actuators\motor\Dockerfile

# Copy to valve directory
copy simulators\actuators\motor\requirements.txt simulators\actuators\valve\requirements.txt
copy simulators\actuators\motor\Dockerfile simulators\actuators\valve\Dockerfile
```

### 6. Create FIWARE Agent Configuration Files

#### Agent 1 Configuration (config/agents/agent1.json)
```json
{
  "agent_type": "sensor_processor",
  "description": "Advanced sensor data processing agent",
  "subscriptions": [
    {
      "entity_type": "Device",
      "condition": "temperature,pressure,vibration",
      "notification_url": "http://custom-agent-1:8080/notify"
    }
  ],
  "processing_rules": {
    "temperature": {
      "threshold_high": 80,
      "threshold_low": 10,
      "alert_topic": "alerts/temperature"
    },
    "pressure": {
      "threshold_high": 10,
      "threshold_low": 0.5,
      "alert_topic": "alerts/pressure"
    }
  }
}
```

#### Agent 2 Configuration (config/agents/agent2.json)
```json
{
  "agent_type": "plc_coordinator",
  "description": "PLC-to-PLC communication coordinator",
  "subscriptions": [
    {
      "entity_type": "ManufacturingMachine",
      "condition": "operational_status,production_efficiency",
      "notification_url": "http://custom-agent-2:8080/notify"
    }
  ],
  "coordination_rules": {
    "production_line": {
      "entities": ["plc001", "plc002", "plc003"],
      "coordination_topic": "plc/coordination",
      "sync_interval": 10
    }
  }
}
```

### 7. Create Smart Data Models

#### Manufacturing Machine Model (data/smart_data_models/manufacturing_machine.json)
```json
{
  "@context": [
    "https://raw.githubusercontent.com/smart-data-models/dataModel.ManufacturingMachine/master/context.jsonld"
  ],
  "id": "urn:ngsi-ld:ManufacturingMachine:template",
  "type": "ManufacturingMachine",
  "manufacturingMachineType": {
    "type": "Property",
    "value": "PLC"
  },
  "serialNumber": {
    "type": "Property", 
    "value": "TEMPLATE-001"
  },
  "brandName": {
    "type": "Property",
    "value": "Industrial Systems"
  },
  "location": {
    "type": "GeoProperty",
    "value": {
      "type": "Point",
      "coordinates": [-3.7, 40.4]
    }
  },
  "status": {
    "type": "Property",
    "value": "working"
  },
  "temperature": {
    "type": "Property",
    "value": 25.0,
    "unitCode": "CEL"
  },
  "operationalStatus": {
    "type": "Property",
    "value": "running"
  }
}
```

### 8. Create Startup Scripts

#### Main Startup Script (scripts/start_twin4.py)
```python
#!/usr/bin/env python3
"""
Twin4 Main Startup Script
Orchestrates the complete FIWARE-based digital twin system
"""

import subprocess
import time
import requests
import os
import sys

def check_docker():
    """Check if Docker is running"""
    try:
        result = subprocess.run(['docker', 'version'], 
                              capture_output=True, text=True)
        if result.returncode == 0:
            print("✅ Docker is running")
            return True
        else:
            print("❌ Docker is not running")
            return False
    except FileNotFoundError:
        print("❌ Docker is not installed")
        return False

def start_fiware_stack():
    """Start the FIWARE infrastructure"""
    print("🚀 Starting FIWARE infrastructure...")
    
    # Start core FIWARE services first
    subprocess.run(['docker-compose', 'up', '-d', 'mongo-db', 'orion', 'mosquitto'])
    print("⏳ Waiting for core services to initialize...")
    time.sleep(30)
    
    # Start IoT Agents
    subprocess.run(['docker-compose', 'up', '-d', 'iot-agent-ul', 'iot-agent-opcua'])
    time.sleep(20)
    
    # Start data persistence
    subprocess.run(['docker-compose', 'up', '-d', 'crate-db', 'quantumleap'])
    time.sleep(15)
    
    # Start visualization
    subprocess.run(['docker-compose', 'up', '-d', 'grafana'])
    
    print("✅ FIWARE infrastructure started")

def start_external_simulators():
    """Start external realistic simulators"""
    print("🏭 Starting external simulators...")
    
    # Start SoftPLC
    subprocess.run(['docker-compose', 'up', '-d', 'soft-plc'])
    time.sleep(10)
    
    # Start OpenPLC
    subprocess.run(['docker-compose', 'up', '-d', 'openplc-runtime'])
    time.sleep(10)
    
    print("✅ External simulators started")

def start_twin4_simulators():
    """Start Twin4 custom simulators"""
    print("⚙️  Starting Twin4 simulators...")
    
    # Start SCADA first
    subprocess.run(['docker-compose', 'up', '-d', 'scada-simulator'])
    time.sleep(10)
    
    # Start PLCs
    subprocess.run(['docker-compose', 'up', '-d', 
                   'plc1-simulator', 'plc2-simulator', 'plc3-simulator'])
    time.sleep(15)
    
    # Start RTU
    subprocess.run(['docker-compose', 'up', '-d', 'rtu-simulator'])
    time.sleep(5)
    
    # Start sensors
    subprocess.run(['docker-compose', 'up', '-d', 
                   'temp-sensor-1', 'temp-sensor-2', 'temp-sensor-3',
                   'pressure-sensor-1', 'pressure-sensor-2',
                   'vibration-sensor', 'vision-camera'])
    time.sleep(10)
    
    # Start actuators
    subprocess.run(['docker-compose', 'up', '-d',
                   'motor-actuator-1', 'motor-actuator-2', 'valve-actuator'])
    time.sleep(5)
    
    print("✅ Twin4 simulators started")

def start_custom_agents():
    """Start custom FIWARE agents"""
    print("🤖 Starting custom agents...")
    
    subprocess.run(['docker-compose', 'up', '-d', 'custom-agent-1', 'custom-agent-2'])
    time.sleep(10)
    
    print("✅ Custom agents started")

def start_web_interface():
    """Start web dashboard"""
    print("🌐 Starting web interface...")
    
    subprocess.run(['docker-compose', 'up', '-d', 'web-dashboard'])
    time.sleep(5)
    
    print("✅ Web interface started")

def check_system_health():
    """Check if all services are healthy"""
    print("🔍 Checking system health...")
    
    services = [
        ("Orion Context Broker", "http://localhost:1026/version"),
        ("IoT Agent UL", "http://localhost:4041/version"),
        ("QuantumLeap", "http://localhost:8668/version"),
        ("Grafana", "http://localhost:3000/api/health"),
        ("CrateDB", "http://localhost:4200/_sql"),
        ("SoftPLC", "http://localhost:8080/api/health"),
        ("OpenPLC", "http://localhost:8081"),
        ("Web Dashboard", "http://localhost:8090")
    ]
    
    for service_name, url in services:
        try:
            response = requests.get(url, timeout=5)
            if response.status_code < 400:
                print(f"✅ {service_name}: OK")
            else:
                print(f"⚠️  {service_name}: HTTP {response.status_code}")
        except Exception as e:
            print(f"❌ {service_name}: Not responding")
    
    print("\n🎯 Access Points:")
    print("📊 Grafana Dashboard: http://localhost:3000 (admin/admin)")
    print("🌐 Web Dashboard: http://localhost:8090")
    print("🏭 SoftPLC API: http://localhost:8080")
    print("🔧 OpenPLC Runtime: http://localhost:8081")
    print("📡 Orion Context Broker: http://localhost:1026")
    print("🔍 CrateDB Admin: http://localhost:4200")

def main():
    """Main startup orchestration"""
    print("🚀 Twin4 - Advanced FIWARE Digital Twin System")
    print("=" * 50)
    
    # Check prerequisites
    if not check_docker():
        print("Please install and start Docker Desktop")
        sys.exit(1)
    
    try:
        # Sequential startup for proper initialization
        start_fiware_stack()
        start_external_simulators() 
        start_twin4_simulators()
        start_custom_agents()
        start_web_interface()
        
        # Wait for all services to fully initialize
        print("⏳ Waiting for all services to initialize...")
        time.sleep(30)
        
        # Health check
        check_system_health()
        
        print("\n🎉 Twin4 system startup complete!")
        print("📚 Check logs: docker-compose logs -f [service-name]")
        print("🛑 Stop system: docker-compose down")
        
    except Exception as e:
        print(f"❌ Startup failed: {e}")
        print("🧹 Cleaning up...")
        subprocess.run(['docker-compose', 'down'])
        sys.exit(1)

if __name__ == "__main__":
    main()
```

### 9. Start the System

```powershell
# Make sure you're in the Twin4 directory
cd D:\Twin4

# Run the startup script
python scripts\start_twin4.py

# Alternative: Start manually with docker-compose
# docker-compose up -d

# Check running containers
docker ps

# View logs of specific service
docker-compose logs -f orion

# Access the web interfaces:
# - Grafana: http://localhost:3000 (admin/admin)
# - Web Dashboard: http://localhost:8090
# - SoftPLC: http://localhost:8080
# - OpenPLC: http://localhost:8081
# - CrateDB: http://localhost:4200
```

### 10. Verify System Operation

```powershell
# Check FIWARE Context Broker
curl http://localhost:1026/v2/entities

# Check IoT Agent status
curl http://localhost:4041/version

# View MQTT messages (install mosquitto clients)
mosquitto_sub -h localhost -t "ul/+/+/+"

# Check container logs
docker-compose logs -f plc1-simulator
docker-compose logs -f scada-simulator
```

### 11. Stop the System

```powershell
# Stop all services
docker-compose down

# Stop and remove volumes (complete cleanup)
docker-compose down -v

# Remove all containers and images (full cleanup)
docker system prune -a
```

## Expected Results:

1. **3 PLCs** with different industrial applications
2. **Multiple sensors** feeding data to PLCs  
3. **Actuators** controlled by PLCs
4. **SCADA** system coordinating all PLCs
5. **Real industrial protocols** (OPC-UA, Modbus, MQTT)
6. **External simulators** (SoftPLC, OpenPLC) for authentic experience
7. **FIWARE agents** processing and coordinating data
8. **Complete visualization** via Grafana and custom dashboard
9. **PLC-to-PLC communication** demonstrating industrial coordination
10. **Smart data models** following FIWARE standards

This setup provides a **complete industrial IoT ecosystem** that demonstrates the full potential of FIWARE in smart manufacturing while using realistic industrial tools and protocols.
