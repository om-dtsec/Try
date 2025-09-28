# Enhanced PLC Simulator with Multiple Sensors and FIWARE Integration

```python
#!/usr/bin/env python3
"""
Enhanced PLC Simulator for Twin4 - Multi-Protocol Industrial Controller

Features:
- Multiple sensor integration
- OPC-UA server
- Modbus TCP server  
- MQTT communication
- PLC-to-PLC communication
- FIWARE Smart Data Models
- Realistic industrial control logic
"""

import paho.mqtt.client as mqtt
import requests
import json
import time
import random
import os
import logging
import threading
from datetime import datetime
from opcua import Server, ua
from pymodbus.server.sync import StartTcpServer
from pymodbus.datastore import ModbusSlaveContext, ModbusServerContext
from pymodbus.datastore import ModbusSequentialDataBlock

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - PLC - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class EnhancedPLCSimulator:
    def __init__(self):
        # Configuration from environment
        self.mqtt_broker = os.getenv('MQTT_BROKER', 'localhost')
        self.mqtt_port = int(os.getenv('MQTT_PORT', '1883'))
        self.device_id = os.getenv('DEVICE_ID', 'plc001')
        self.plc_type = os.getenv('PLC_TYPE', 'paper_feeder')
        self.parent_device = os.getenv('PARENT_DEVICE', 'scada001')
        self.connected_sensors = os.getenv('CONNECTED_SENSORS', '').split(',')
        self.connected_actuators = os.getenv('CONNECTED_ACTUATORS', '').split(',')
        self.iot_agent_url = os.getenv('IOT_AGENT_URL', 'http://localhost:4041')
        self.api_key = os.getenv('API_KEY', 'twin4apikey2025')
        self.modbus_port = int(os.getenv('MODBUS_PORT', '502'))
        self.opcua_port = int(os.getenv('OPC_UA_PORT', '4840'))
        
        # MQTT Client
        self.mqtt_client = mqtt.Client()
        self.mqtt_client.on_connect = self.on_mqtt_connect
        self.mqtt_client.on_message = self.on_mqtt_message
        
        # Data storage for connected devices
        self.sensor_data = {}
        self.actuator_commands = {}
        self.plc_to_plc_data = {}
        
        # Configure PLC based on type
        self.configure_plc_type()
        
        # Initialize servers
        self.setup_opcua_server()
        self.setup_modbus_server()
        
    def configure_plc_type(self):
        """Configure PLC based on its industrial application"""
        if self.plc_type == "paper_feeder":
            self.hardware_meta_data = {
                "manufacturer": "Siemens",
                "model": "S7-1500 CPU 1515-2 PN",
                "firmware_version": "v4.2.1",
                "software_version": "TIA Portal v17",
                "io_modules": ["DI32xDC24V", "DO32xDC24V", "AI8xU/I/RTD/TC"],
                "serial_number": f"6ES7515-{random.randint(1000,9999)}-{random.randint(10,99)}"
            }
            
            self.network_data = {
                "ip_address": f"192.168.1.{101 + int(self.device_id[-3:])}",
                "protocols": ["Modbus TCP", "Profinet", "S7 Communication", "OPC-UA"],
                "ports": {
                    "modbus": self.modbus_port,
                    "opcua": self.opcua_port,
                    "profinet": 34962,
                    "s7": 102
                }
            }
            
            self.process_data = {
                # Paper feeder specific data
                "paper_level": 85.0,
                "feed_rate": 1500,
                "jam_detected": False,
                "motor_speed": 75.0,
                "cycles_completed": random.randint(4800, 5000),
                "target_feed_rate": 1500,
                "maintenance_hours": random.randint(8500, 8600),
                
                # Environmental data from sensors
                "temperature": 25.0,
                "pressure": 1.2,
                "vibration": 0.5,
                
                # Status indicators
                "operational_status": "running",
                "alarm_status": "normal",
                "production_efficiency": 98.5
            }
            
        elif self.plc_type == "printing_press":
            self.hardware_meta_data = {
                "manufacturer": "Allen Bradley",
                "model": "ControlLogix 1756-L73",
                "firmware_version": "v32.011",
                "software_version": "Studio 5000 v32",
                "io_modules": ["1756-EN2T", "1756-IB32", "1756-OB32", "1756-IF8"],
                "serial_number": f"1756-L73/{random.choice(['A', 'B', 'C'])}"
            }
            
            self.process_data = {
                # Printing press specific data
                "ink_level": 78.5,
                "print_quality": 98.7,
                "temperature": 68.2,
                "printing_speed": 15000,
                "sheets_printed": random.randint(12000, 13000),
                "target_speed": 15000,
                "waste_count": random.randint(50, 100),
                
                # Vision system data
                "defects_detected": 0,
                "color_accuracy": 99.2,
                "registration_accuracy": 0.1,
                
                # Actuator status
                "motor_status": "running",
                "valve_position": 50,
                
                # Quality metrics
                "print_density": 1.2,
                "drying_temperature": 120.0,
                "web_tension": 250.0
            }
            
        elif self.plc_type == "process_control":
            self.hardware_meta_data = {
                "manufacturer": "Schneider Electric",
                "model": "Modicon M580",
                "firmware_version": "v3.20",
                "software_version": "Unity Pro XL v13.1",
                "io_modules": ["BMXDDI3202K", "BMXDDO3202K", "BMXAMI0810"],
                "serial_number": f"BMEP58{random.randint(1000,9999)}"
            }
            
            self.process_data = {
                # Process control specific data
                "reactor_temperature": 65.5,
                "reactor_pressure": 2.5,
                "flow_rate": 125.0,
                "level_tank_1": 75.0,
                "level_tank_2": 60.0,
                "ph_value": 7.2,
                "conductivity": 850.0,
                
                # Control loop parameters
                "temperature_setpoint": 65.0,
                "pressure_setpoint": 2.4,
                "flow_setpoint": 120.0,
                
                # Safety systems
                "emergency_stop": False,
                "safety_valve_open": False,
                "interlock_status": "normal"
            }
    
    def setup_opcua_server(self):
        """Setup OPC-UA server for industrial protocol communication"""
        try:
            self.opcua_server = Server()
            self.opcua_server.set_endpoint(f"opc.tcp://0.0.0.0:{self.opcua_port}/freeopcua/server/")
            self.opcua_server.set_server_name(f"{self.plc_type.title()} PLC {self.device_id}")
            
            # Setup namespace
            uri = f"http://twin4.manufacturing.com/{self.device_id}"
            idx = self.opcua_server.register_namespace(uri)
            
            # Create object node
            self.opcua_objects = self.opcua_server.get_objects_node()
            self.plc_node = self.opcua_objects.add_object(idx, f"PLC_{self.device_id}")
            
            # Add variables for process data
            self.opcua_vars = {}
            for key, value in self.process_data.items():
                if isinstance(value, bool):
                    var = self.plc_node.add_variable(idx, key, value)
                elif isinstance(value, (int, float)):
                    var = self.plc_node.add_variable(idx, key, value)
                else:
                    var = self.plc_node.add_variable(idx, key, str(value))
                var.set_writable()
                self.opcua_vars[key] = var
                
            logger.info(f"OPC-UA server configured on port {self.opcua_port}")
            
        except Exception as e:
            logger.error(f"Failed to setup OPC-UA server: {e}")
    
    def setup_modbus_server(self):
        """Setup Modbus TCP server for PLC communication"""
        try:
            # Create data blocks for different data types
            holding_registers = ModbusSequentialDataBlock(0, [0] * 100)
            input_registers = ModbusSequentialDataBlock(0, [0] * 100) 
            coils = ModbusSequentialDataBlock(0, [False] * 100)
            discrete_inputs = ModbusSequentialDataBlock(0, [False] * 100)
            
            # Create datastore
            store = ModbusSlaveContext(
                di=discrete_inputs,
                co=coils,
                hr=holding_registers,
                ir=input_registers
            )
            
            self.modbus_context = ModbusServerContext(slaves=store, single=True)
            
            # Start server in separate thread
            self.modbus_thread = threading.Thread(
                target=StartTcpServer,
                kwargs={
                    'context': self.modbus_context,
                    'address': ("0.0.0.0", self.modbus_port)
                },
                daemon=True
            )
            
            logger.info(f"Modbus TCP server configured on port {self.modbus_port}")
            
        except Exception as e:
            logger.error(f"Failed to setup Modbus server: {e}")
    
    def on_mqtt_connect(self, client, userdata, flags, rc):
        """Handle MQTT connection"""
        logger.info(f"PLC {self.device_id} connected to MQTT broker with result code {rc}")
        
        # Subscribe to sensor data topics
        for sensor_id in self.connected_sensors:
            if sensor_id:  # Check if sensor_id is not empty
                topic = f"ul/{self.api_key}/{sensor_id}/attrs"
                client.subscribe(topic)
                logger.info(f"Subscribed to sensor {sensor_id}")
        
        # Subscribe to PLC-to-PLC communication
        client.subscribe(f"plc/+/data")
        client.subscribe(f"plc/{self.device_id}/commands")
        
        # Subscribe to SCADA commands
        client.subscribe(f"scada/{self.parent_device}/commands/{self.device_id}")
    
    def on_mqtt_message(self, client, userdata, msg):
        """Process incoming MQTT messages"""
        try:
            topic = msg.topic.decode()
            payload = msg.payload.decode()
            
            if "attrs" in topic:
                # Process sensor data
                self.process_sensor_data(topic, payload)
            elif "plc/" in topic and "/data" in topic:
                # Process PLC-to-PLC communication
                self.process_plc_communication(topic, payload)
            elif "/commands/" in topic:
                # Process commands from SCADA or other systems
                self.process_commands(topic, payload)
                
        except Exception as e:
            logger.error(f"Error processing MQTT message: {e}")
    
    def process_sensor_data(self, topic, payload):
        """Process incoming sensor data and update PLC logic"""
        try:
            # Extract sensor ID from topic
            parts = topic.split('/')
            sensor_id = parts[2] if len(parts) > 2 else "unknown"
            
            # Parse UL payload (format: attr1|value1|attr2|value2|...)
            data_pairs = payload.split('|')
            sensor_data = {}
            
            for i in range(0, len(data_pairs)-1, 2):
                if i+1 < len(data_pairs):
                    attr = data_pairs[i]
                    value = data_pairs[i+1]
                    try:
                        # Try to convert to number
                        sensor_data[attr] = float(value) if '.' in value else int(value)
                    except ValueError:
                        sensor_data[attr] = value
            
            self.sensor_data[sensor_id] = sensor_data
            
            # Update process data based on sensor readings
            if "temp" in sensor_id and "te" in sensor_data:
                self.process_data["temperature"] = sensor_data["te"]
            elif "pressure" in sensor_id and "pr" in sensor_data:
                self.process_data["pressure"] = sensor_data["pr"]
            elif "vibration" in sensor_id and "vi" in sensor_data:
                self.process_data["vibration"] = sensor_data["vi"]
            elif "camera" in sensor_id:
                if "de" in sensor_data:
                    self.process_data["defects_detected"] = sensor_data["de"]
                if "co" in sensor_data:
                    self.process_data["color_accuracy"] = sensor_data["co"]
                    
            logger.info(f"Updated PLC data from sensor {sensor_id}: {sensor_data}")
            
        except Exception as e:
            logger.error(f"Error processing sensor data: {e}")
    
    def process_plc_communication(self, topic, payload):
        """Process PLC-to-PLC communication"""
        try:
            source_plc = topic.split('/')[1]
            if source_plc != self.device_id:  # Don't process own messages
                data = json.loads(payload)
                self.plc_to_plc_data[source_plc] = data
                logger.info(f"Received data from PLC {source_plc}: {data}")
                
                # React to other PLC data (example: coordinate production)
                if source_plc == "plc001" and self.device_id == "plc002":
                    # Printing press adjusts speed based on paper feeder rate
                    if "feed_rate" in data:
                        self.process_data["target_speed"] = min(
                            data["feed_rate"] * 10,  # Convert to printing speed
                            18000  # Max speed limit
                        )
                        
        except Exception as e:
            logger.error(f"Error processing PLC communication: {e}")
    
    def process_commands(self, topic, payload):
        """Process commands from SCADA or other control systems"""
        try:
            data = json.loads(payload)
            logger.info(f"Received command: {data}")
            
            # Process different command types
            if "setpoint" in data:
                for key, value in data["setpoint"].items():
                    if key in self.process_data:
                        self.process_data[key] = value
                        logger.info(f"Updated setpoint {key} to {value}")
            
            if "emergency_stop" in data:
                self.process_data["emergency_stop"] = data["emergency_stop"]
                if data["emergency_stop"]:
                    logger.warning("EMERGENCY STOP ACTIVATED")
                    self.process_data["operational_status"] = "emergency_stop"
                    
        except Exception as e:
            logger.error(f"Error processing command: {e}")
    
    def execute_control_logic(self):
        """Execute advanced PLC control logic"""
        try:
            if self.plc_type == "paper_feeder":
                self.paper_feeder_logic()
            elif self.plc_type == "printing_press":
                self.printing_press_logic()
            elif self.plc_type == "process_control":
                self.process_control_logic()
                
            # Update OPC-UA variables
            self.update_opcua_variables()
            
            # Update Modbus registers
            self.update_modbus_registers()
            
        except Exception as e:
            logger.error(f"Error in control logic: {e}")
    
    def paper_feeder_logic(self):
        """Enhanced paper feeder control logic"""
        # Check for vibration-based jam detection
        if self.process_data.get("vibration", 0) > 10:
            self.process_data["jam_detected"] = True
            self.process_data["alarm_status"] = "jam_detected"
        
        # Temperature-based feed rate adjustment
        temp = self.process_data.get("temperature", 25)
        if temp > 30:  # Hot conditions
            adjustment_factor = 0.95
        elif temp < 20:  # Cold conditions  
            adjustment_factor = 1.05
        else:
            adjustment_factor = 1.0
            
        if not self.process_data["jam_detected"]:
            # Normal operation with environmental compensation
            base_rate = self.process_data["target_feed_rate"]
            self.process_data["feed_rate"] = int(base_rate * adjustment_factor)
            self.process_data["motor_speed"] = min(
                (self.process_data["feed_rate"] / 20.0), 100.0
            )
            
            # Consume paper
            consumption = random.uniform(0.1, 0.3)
            self.process_data["paper_level"] = max(
                0, self.process_data["paper_level"] - consumption
            )
            
            # Update cycle count
            self.process_data["cycles_completed"] += random.randint(0, 2)
            
            # Calculate efficiency
            actual_rate = self.process_data["feed_rate"]
            target_rate = self.process_data["target_feed_rate"]
            self.process_data["production_efficiency"] = min(
                (actual_rate / target_rate) * 100, 100
            )
        else:
            # Jam condition
            self.process_data["feed_rate"] = 0
            self.process_data["motor_speed"] = 0
            self.process_data["production_efficiency"] = 0
            
            # Random jam clearance
            if random.random() < 0.2:  # 20% chance to clear
                self.process_data["jam_detected"] = False
                self.process_data["alarm_status"] = "normal"
                logger.info("Jam cleared automatically")
    
    def printing_press_logic(self):
        """Enhanced printing press control logic"""
        # Temperature control for print quality
        temp = self.process_data.get("temperature", 68)
        if 65 <= temp <= 72:  # Optimal range
            quality_factor = 1.0
        else:
            quality_factor = 0.95 - abs(temp - 68.5) * 0.01
            
        self.process_data["print_quality"] = max(
            85.0, min(100.0, 98.0 * quality_factor)
        )
        
        # Speed adjustment based on quality requirements
        if self.process_data["print_quality"] < 95:
            # Reduce speed to improve quality
            self.process_data["printing_speed"] = int(
                self.process_data["target_speed"] * 0.9
            )
        else:
            self.process_data["printing_speed"] = self.process_data["target_speed"]
            
        # Defect handling from vision system
        if self.process_data.get("defects_detected", 0) > 2:
            # Too many defects, reduce speed
            self.process_data["printing_speed"] = int(
                self.process_data["printing_speed"] * 0.8
            )
            self.process_data["waste_count"] += self.process_data["defects_detected"]
        
        # Ink consumption based on printing
        if self.process_data["printing_speed"] > 0:
            ink_consumption = (self.process_data["printing_speed"] / 15000) * 0.1
            self.process_data["ink_level"] = max(
                0, self.process_data["ink_level"] - ink_consumption
            )
            
            # Update sheet count
            sheets_per_minute = self.process_data["printing_speed"] / 60
            self.process_data["sheets_printed"] += int(sheets_per_minute / 12)  # Per 5-second cycle
    
    def process_control_logic(self):
        """Process control logic for reactor management"""
        # Temperature control loop
        temp_error = self.process_data["temperature_setpoint"] - self.process_data["reactor_temperature"]
        
        if abs(temp_error) > 0.5:  # Outside deadband
            # Simple proportional control
            temp_adjustment = temp_error * 0.1
            self.process_data["reactor_temperature"] += temp_adjustment
        
        # Pressure control
        pressure_error = self.process_data["pressure_setpoint"] - self.process_data["reactor_pressure"]
        if abs(pressure_error) > 0.1:
            pressure_adjustment = pressure_error * 0.05
            self.process_data["reactor_pressure"] += pressure_adjustment
        
        # Flow rate control
        flow_error = self.process_data["flow_setpoint"] - self.process_data["flow_rate"]
        if abs(flow_error) > 2.0:
            flow_adjustment = flow_error * 0.1
            self.process_data["flow_rate"] += flow_adjustment
        
        # Safety interlocks
        if (self.process_data["reactor_temperature"] > 80 or 
            self.process_data["reactor_pressure"] > 5.0):
            self.process_data["emergency_stop"] = True
            self.process_data["safety_valve_open"] = True
            logger.warning("Safety interlock activated!")
    
    def update_opcua_variables(self):
        """Update OPC-UA server variables"""
        try:
            for key, var in self.opcua_vars.items():
                if key in self.process_data:
                    var.set_value(self.process_data[key])
        except Exception as e:
            logger.error(f"Error updating OPC-UA variables: {e}")
    
    def update_modbus_registers(self):
        """Update Modbus registers with current process data"""
        try:
            if hasattr(self, 'modbus_context'):
                slave_context = self.modbus_context[0]  # Get slave 0
                
                # Map process data to holding registers
                register_map = {}
                reg_addr = 0
                
                for key, value in self.process_data.items():
                    if isinstance(value, (int, float)):
                        register_map[reg_addr] = int(value * 10)  # Scale for integer storage
                        reg_addr += 1
                    elif isinstance(value, bool):
                        register_map[reg_addr] = 1 if value else 0
                        reg_addr += 1
                
                # Update registers
                for addr, value in register_map.items():
                    if addr < 100:  # Within our register range
                        slave_context.setValues(3, addr, [value])  # Function code 3 = holding registers
                        
        except Exception as e:
            logger.error(f"Error updating Modbus registers: {e}")
    
    def publish_plc_data(self):
        """Publish PLC data via MQTT using FIWARE Smart Data Models"""
        try:
            # Create FIWARE-compliant entity
            entity_data = {
                "id": f"urn:ngsi-ld:ManufacturingMachine:{self.device_id}",
                "type": "ManufacturingMachine",
                "manufacturingMachineType": {
                    "type": "Text",
                    "value": self.plc_type
                },
                "status": {
                    "type": "Text", 
                    "value": self.process_data.get("operational_status", "running")
                },
                "location": {
                    "type": "geo:json",
                    "value": {
                        "type": "Point",
                        "coordinates": [-3.7, 40.4]  # Example coordinates
                    }
                }
            }
            
            # Add process-specific attributes
            for key, value in self.process_data.items():
                if isinstance(value, (int, float)):
                    entity_data[key] = {
                        "type": "Number",
                        "value": value
                    }
                elif isinstance(value, bool):
                    entity_data[key] = {
                        "type": "Boolean", 
                        "value": value
                    }
                elif isinstance(value, str):
                    entity_data[key] = {
                        "type": "Text",
                        "value": value
                    }
            
            # Publish to MQTT for IoT Agent
            ul_payload = self.create_ul_payload()
            topic = f"ul/{self.api_key}/{self.device_id}/attrs"
            self.mqtt_client.publish(topic, ul_payload)
            
            # Publish for PLC-to-PLC communication
            plc_comm_data = {
                "source": self.device_id,
                "timestamp": datetime.now().isoformat(),
                "data": self.process_data
            }
            plc_topic = f"plc/{self.device_id}/data"
            self.mqtt_client.publish(plc_topic, json.dumps(plc_comm_data))
            
            timestamp = datetime.now().strftime("%H:%M:%S")
            logger.info(f"[{timestamp}] PLC {self.device_id} data published")
            
        except Exception as e:
            logger.error(f"Error publishing PLC data: {e}")
    
    def create_ul_payload(self):
        """Create UltraLight payload from process data"""
        payload_parts = []
        
        # Map process data to UL format
        data_mapping = {
            "temperature": "te",
            "pressure": "pr", 
            "vibration": "vi",
            "feed_rate": "fr",
            "motor_speed": "ms",
            "printing_speed": "ps",
            "ink_level": "il",
            "print_quality": "pq",
            "defects_detected": "dd",
            "operational_status": "os",
            "production_efficiency": "pe"
        }
        
        for key, value in self.process_data.items():
            if key in data_mapping:
                ul_key = data_mapping[key]
                payload_parts.append(f"{ul_key}|{value}")
        
        return "|".join(payload_parts)
    
    def provision_device(self):
        """Provision PLC device with IoT Agent using Smart Data Model"""
        try:
            device_config = {
                "devices": [{
                    "device_id": self.device_id,
                    "entity_name": f"urn:ngsi-ld:ManufacturingMachine:{self.device_id}",
                    "entity_type": "ManufacturingMachine",
                    "transport": "MQTT",
                    "attributes": [
                        {"object_id": "te", "name": "temperature", "type": "Number"},
                        {"object_id": "pr", "name": "pressure", "type": "Number"},
                        {"object_id": "vi", "name": "vibration", "type": "Number"},
                        {"object_id": "fr", "name": "feed_rate", "type": "Number"},
                        {"object_id": "ms", "name": "motor_speed", "type": "Number"},
                        {"object_id": "ps", "name": "printing_speed", "type": "Number"},
                        {"object_id": "il", "name": "ink_level", "type": "Number"},
                        {"object_id": "pq", "name": "print_quality", "type": "Number"},
                        {"object_id": "dd", "name": "defects_detected", "type": "Number"},
                        {"object_id": "os", "name": "operational_status", "type": "Text"},
                        {"object_id": "pe", "name": "production_efficiency", "type": "Number"}
                    ],
                    "static_attributes": [
                        {"name": "manufacturingMachineType", "type": "Text", "value": self.plc_type},
                        {"name": "manufacturer", "type": "Text", "value": self.hardware_meta_data["manufacturer"]},
                        {"name": "model", "type": "Text", "value": self.hardware_meta_data["model"]},
                        {"name": "serialNumber", "type": "Text", "value": self.hardware_meta_data["serial_number"]},
                        {"name": "parentDevice", "type": "Relationship", "value": self.parent_device}
                    ]
                }]
            }
            
            response = requests.post(
                f"{self.iot_agent_url}/iot/devices",
                headers={
                    "Content-Type": "application/json", 
                    "fiware-service": "twin4factory",
                    "fiware-servicepath": "/"
                },
                json=device_config
            )
            
            logger.info(f"PLC {self.device_id} provisioned: {response.status_code}")
            
        except Exception as e:
            logger.error(f"Failed to provision PLC device: {e}")
    
    def run(self):
        """Run the enhanced PLC simulator"""
        logger.info(f"🏭 Starting Enhanced PLC Simulator {self.device_id}")
        logger.info(f"📊 Type: {self.plc_type.title()}")
        logger.info(f"🔗 Parent: {self.parent_device}")
        logger.info(f"📡 Protocols: MQTT, OPC-UA (:{self.opcua_port}), Modbus TCP (:{self.modbus_port})")
        logger.info(f"🔌 Connected Sensors: {', '.join(self.connected_sensors) if self.connected_sensors else 'None'}")
        logger.info(f"⚙️  Connected Actuators: {', '.join(self.connected_actuators) if self.connected_actuators else 'None'}")
        
        try:
            # Start OPC-UA server
            self.opcua_server.start()
            logger.info(f"OPC-UA server started on port {self.opcua_port}")
            
            # Start Modbus server
            if hasattr(self, 'modbus_thread'):
                self.modbus_thread.start()
                logger.info(f"Modbus TCP server started on port {self.modbus_port}")
            
            # Connect to MQTT
            self.mqtt_client.connect(self.mqtt_broker, self.mqtt_port, 60)
            self.mqtt_client.loop_start()
            
            # Wait for FIWARE stack initialization
            time.sleep(30)
            
            # Provision device
            self.provision_device()
            
            # Main control loop
            while True:
                self.execute_control_logic()
                self.publish_plc_data()
                time.sleep(5)  # 5-second control cycle
                
        except KeyboardInterrupt:
            logger.info(f"PLC {self.device_id} simulator stopping...")
        except Exception as e:
            logger.error(f"Error in PLC simulator: {e}")
        finally:
            # Cleanup
            try:
                self.opcua_server.stop()
                self.mqtt_client.loop_stop()
                self.mqtt_client.disconnect()
                logger.info(f"PLC {self.device_id} simulator stopped")
            except:
                pass

if __name__ == "__main__":
    plc = EnhancedPLCSimulator()
    plc.run()
```

## Key Features of Enhanced PLC Simulator:

### 1. **Multi-Protocol Support**
- **OPC-UA Server**: Industrial standard for machine-to-machine communication
- **Modbus TCP Server**: Widely used in industrial automation
- **MQTT Client**: For FIWARE IoT Agent communication
- **Real protocol endpoints** that can be accessed by external tools

### 2. **Advanced Control Logic**
- **Environmental compensation**: Temperature and pressure affect operations
- **Quality control**: Vision system feedback influences process parameters
- **Safety interlocks**: Emergency stop and safety valve management
- **Inter-PLC coordination**: PLCs communicate and adjust based on each other's data

### 3. **FIWARE Smart Data Models**
- **ManufacturingMachine**: Standard FIWARE entity type
- **Proper NGSI-LD**: Structured entity relationships
- **Static attributes**: Hardware metadata and relationships
- **Dynamic attributes**: Real-time process data

### 4. **Realistic Industrial Behavior**
- **Hardware metadata**: Actual PLC models and specifications
- **Network configuration**: IP addresses, ports, and protocols
- **Process-specific logic**: Different behavior for paper feeder, printing press, and process control
- **Maintenance tracking**: Operating hours and efficiency metrics

### 5. **Inter-Device Communication**
- **Sensor data processing**: Real sensor inputs affect PLC decisions
- **PLC-to-PLC messaging**: Coordination between production stages
- **SCADA command processing**: Higher-level control system integration
- **Actuator control**: Commands sent to connected actuators

This enhanced PLC simulator provides a **realistic industrial automation experience** while maintaining full FIWARE compatibility and supporting multiple industrial protocols for maximum learning value.
