# Advanced Sensor Simulator with Realistic Industrial Behavior

```python
#!/usr/bin/env python3
"""
Enhanced Sensor Simulator for Twin4 - Realistic Industrial IoT Sensor

Features:
- Multiple sensor types (Temperature, Pressure, Vibration, Camera)
- Realistic noise and drift simulation
- Equipment-specific measurement ranges
- FIWARE Smart Data Models compliance
- Environmental factor simulation
- Calibration and maintenance cycles
- Alert generation for out-of-range values
"""

import paho.mqtt.client as mqtt
import requests
import json
import time
import random
import os
import logging
import math
from datetime import datetime, timedelta
import numpy as np

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - SENSOR - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class EnhancedSensorSimulator:
    def __init__(self):
        # Configuration from environment
        self.mqtt_broker = os.getenv('MQTT_BROKER', 'localhost')
        self.mqtt_port = int(os.getenv('MQTT_PORT', '1883'))
        self.device_id = os.getenv('DEVICE_ID', 'temp001')
        self.sensor_type = os.getenv('SENSOR_TYPE', 'temperature')
        self.parent_device = os.getenv('PARENT_DEVICE', 'plc001')
        self.measurement_range = os.getenv('MEASUREMENT_RANGE', '0-100').split('-')
        self.accuracy = float(os.getenv('ACCURACY', '0.1'))
        self.update_interval = int(os.getenv('UPDATE_INTERVAL', '5'))
        self.iot_agent_url = os.getenv('IOT_AGENT_URL', 'http://localhost:4041')
        self.api_key = os.getenv('API_KEY', 'twin4apikey2025')
        
        # Sensor characteristics
        self.min_value = float(self.measurement_range[0])
        self.max_value = float(self.measurement_range[1])
        self.current_value = (self.min_value + self.max_value) / 2
        
        # Industrial sensor metadata
        self.configure_sensor_metadata()
        
        # Simulation parameters
        self.base_value = self.current_value
        self.drift_rate = random.uniform(-0.001, 0.001)  # Gradual sensor drift
        self.noise_level = self.accuracy * 0.5
        self.last_calibration = datetime.now() - timedelta(days=random.randint(0, 180))
        self.calibration_interval = timedelta(days=180)  # 6 months
        self.operating_hours = random.randint(1000, 8000)
        
        # Environmental factors
        self.environmental_factors = {
            'ambient_temperature': 22.0,
            'humidity': 45.0,
            'vibration_level': 0.1,
            'electromagnetic_interference': 0.0
        }
        
        # MQTT Client
        self.mqtt_client = mqtt.Client()
        self.mqtt_client.on_connect = self.on_mqtt_connect
        
        # Simulation patterns based on sensor type
        self.setup_simulation_patterns()
        
    def configure_sensor_metadata(self):
        """Configure sensor metadata based on type and industrial standards"""
        
        sensor_specs = {
            "temperature": {
                "manufacturer": "Honeywell",
                "model": "STT3000",
                "type": "RTD Pt100",
                "response_time": "10s",
                "protection_rating": "IP67",
                "calibration_standard": "IEC 60751",
                "units": "°C",
                "resolution": 0.01,
                "stability": "±0.1°C/year"
            },
            "pressure": {
                "manufacturer": "Endress+Hauser", 
                "model": "Cerabar PMC21",
                "type": "Ceramic capacitive",
                "response_time": "100ms",
                "protection_rating": "IP68",
                "calibration_standard": "EN 837-1",
                "units": "bar",
                "resolution": 0.001,
                "stability": "±0.1%/year"
            },
            "vibration": {
                "manufacturer": "SKF",
                "model": "CMSS 2200",
                "type": "IEPE Accelerometer",
                "response_time": "1ms",
                "protection_rating": "IP67",
                "calibration_standard": "ISO 16063",
                "units": "mm/s RMS",
                "resolution": 0.01,
                "stability": "±2%/year"
            },
            "camera": {
                "manufacturer": "Cognex",
                "model": "In-Sight 7000",
                "type": "Machine Vision Camera",
                "response_time": "33ms",
                "protection_rating": "IP67",
                "calibration_standard": "ISO 9001",
                "units": "defects/frame",
                "resolution": "1920x1080",
                "stability": "±1%/year"
            }
        }
        
        self.sensor_metadata = sensor_specs.get(self.sensor_type, sensor_specs["temperature"])
        self.sensor_metadata["serial_number"] = f"{self.sensor_metadata['model']}-{random.randint(10000, 99999)}"
        self.sensor_metadata["installation_date"] = (datetime.now() - timedelta(days=random.randint(30, 1000))).isoformat()
        
    def setup_simulation_patterns(self):
        """Setup realistic simulation patterns for different sensor types"""
        
        if self.sensor_type == "temperature":
            # Temperature sensors show daily cycles and process-related changes
            self.daily_cycle_amplitude = (self.max_value - self.min_value) * 0.1
            self.process_cycle_period = 300  # 5 minutes for process cycles
            self.process_amplitude = (self.max_value - self.min_value) * 0.2
            
        elif self.sensor_type == "pressure":
            # Pressure sensors respond quickly to system changes
            self.system_pressure_base = (self.max_value - self.min_value) * 0.6 + self.min_value
            self.pressure_fluctuation_amplitude = (self.max_value - self.min_value) * 0.05
            self.pump_cycle_period = 120  # 2 minutes
            
        elif self.sensor_type == "vibration":
            # Vibration sensors detect mechanical issues
            self.normal_vibration = self.min_value + (self.max_value - self.min_value) * 0.1
            self.bearing_wear_factor = random.uniform(0.8, 1.2)
            self.imbalance_frequency = random.uniform(0.1, 0.3)
            
        elif self.sensor_type == "camera":
            # Vision systems detect defects based on production quality
            self.base_defect_rate = 0.02  # 2% normal defect rate
            self.quality_drift_period = 3600  # 1 hour cycles
            
    def on_mqtt_connect(self, client, userdata, flags, rc):
        """Handle MQTT connection"""
        logger.info(f"Sensor {self.device_id} connected to MQTT broker with result code {rc}")
        
        # Subscribe to calibration commands
        client.subscribe(f"sensor/{self.device_id}/calibrate")
        client.subscribe(f"sensor/{self.device_id}/commands")
        
    def simulate_sensor_value(self):
        """Generate realistic sensor values with industrial behavior patterns"""
        
        current_time = time.time()
        
        if self.sensor_type == "temperature":
            # Daily temperature cycle
            daily_cycle = math.sin(current_time / 86400 * 2 * math.pi) * self.daily_cycle_amplitude
            
            # Process temperature cycles (heating/cooling)
            process_cycle = math.sin(current_time / self.process_cycle_period * 2 * math.pi) * self.process_amplitude
            
            # Base temperature with cycles
            value = self.base_value + daily_cycle + process_cycle
            
            # Add equipment-specific behavior
            if "plc002" in self.parent_device:  # Printing press runs hotter
                value += 15
            elif "plc003" in self.parent_device:  # Process control needs precision
                value = self.base_value + daily_cycle * 0.5
                
        elif self.sensor_type == "pressure":
            # System pressure with pump cycles
            pump_cycle = math.sin(current_time / self.pump_cycle_period * 2 * math.pi)
            pressure_oscillation = pump_cycle * self.pressure_fluctuation_amplitude
            
            # Base system pressure
            value = self.system_pressure_base + pressure_oscillation
            
            # Add pressure drops during high demand
            if random.random() < 0.1:  # 10% chance of demand spike
                value *= 0.9  # Pressure drop
                
        elif self.sensor_type == "vibration":
            # Normal operational vibration
            base_vibration = self.normal_vibration * self.bearing_wear_factor
            
            # Add rotational imbalance
            imbalance = math.sin(current_time * self.imbalance_frequency * 2 * math.pi) * base_vibration * 0.3
            
            # Random mechanical events
            if random.random() < 0.02:  # 2% chance of vibration spike
                spike_magnitude = random.uniform(2, 5)
                value = base_vibration * spike_magnitude
            else:
                value = base_vibration + imbalance
                
        elif self.sensor_type == "camera":
            # Vision system defect detection
            # Quality varies with time (tool wear, material changes)
            quality_cycle = math.sin(current_time / self.quality_drift_period * 2 * math.pi)
            current_defect_rate = self.base_defect_rate * (1 + quality_cycle * 0.5)
            
            # Simulate defects per inspection
            defects = np.random.poisson(current_defect_rate * 100)  # Per 100 items
            value = max(0, defects)
            
            # Additional metrics for vision system
            self.color_accuracy = max(90, min(100, 98 + random.uniform(-1, 1)))
            self.registration_accuracy = max(0.05, min(0.5, 0.1 + random.uniform(-0.02, 0.02)))
            
        # Apply sensor drift over time
        drift_effect = self.drift_rate * self.operating_hours / 1000
        value += drift_effect
        
        # Apply measurement noise
        noise = random.gauss(0, self.noise_level)
        value += noise
        
        # Apply environmental effects
        if self.sensor_type == "temperature":
            # Ambient temperature affects readings
            ambient_effect = (self.environmental_factors['ambient_temperature'] - 22) * 0.1
            value += ambient_effect
            
        # Check calibration status
        time_since_calibration = datetime.now() - self.last_calibration
        if time_since_calibration > self.calibration_interval:
            # Add calibration error
            calibration_error = random.uniform(-self.accuracy, self.accuracy) * 2
            value += calibration_error
            
        # Clamp to sensor range
        value = max(self.min_value, min(self.max_value, value))
        
        # Update operating hours
        self.operating_hours += self.update_interval / 3600  # Convert seconds to hours
        
        return value
    
    def generate_sensor_alerts(self, value):
        """Generate alerts for out-of-range or abnormal sensor values"""
        alerts = []
        
        # Range alerts
        if value <= self.min_value * 1.05:  # Within 5% of minimum
            alerts.append({
                "type": "LOW_VALUE_WARNING",
                "message": f"Sensor {self.device_id} reading near minimum: {value:.2f}",
                "severity": "WARNING"
            })
        elif value >= self.max_value * 0.95:  # Within 5% of maximum
            alerts.append({
                "type": "HIGH_VALUE_WARNING", 
                "message": f"Sensor {self.device_id} reading near maximum: {value:.2f}",
                "severity": "WARNING"
            })
            
        # Calibration alerts
        time_since_calibration = datetime.now() - self.last_calibration
        if time_since_calibration > self.calibration_interval:
            alerts.append({
                "type": "CALIBRATION_DUE",
                "message": f"Sensor {self.device_id} calibration overdue by {time_since_calibration.days - 180} days",
                "severity": "INFO"
            })
            
        # Sensor-specific alerts
        if self.sensor_type == "vibration" and value > self.normal_vibration * 3:
            alerts.append({
                "type": "HIGH_VIBRATION_ALARM",
                "message": f"Dangerous vibration level detected: {value:.2f} mm/s",
                "severity": "CRITICAL"
            })
            
        if self.sensor_type == "temperature" and hasattr(self, 'previous_value'):
            rate_of_change = abs(value - self.previous_value) / self.update_interval
            if rate_of_change > 1.0:  # More than 1°C per second
                alerts.append({
                    "type": "RAPID_TEMPERATURE_CHANGE",
                    "message": f"Rapid temperature change detected: {rate_of_change:.2f} °C/s",
                    "severity": "WARNING"
                })
                
        return alerts
    
    def create_fiware_entity_data(self, value, alerts):
        """Create FIWARE-compliant sensor entity data"""
        
        # Base entity following Device data model
        entity_data = {
            "id": f"urn:ngsi-ld:Device:{self.device_id}",
            "type": "Device",
            "category": {
                "type": "Property",
                "value": ["sensor"]
            },
            "deviceType": {
                "type": "Property",
                "value": self.sensor_type
            },
            "name": {
                "type": "Property", 
                "value": f"{self.sensor_type.title()} Sensor {self.device_id}"
            },
            "serialNumber": {
                "type": "Property",
                "value": self.sensor_metadata["serial_number"]
            },
            "modelName": {
                "type": "Property",
                "value": self.sensor_metadata["model"]
            },
            "brandName": {
                "type": "Property",
                "value": self.sensor_metadata["manufacturer"]
            },
            "location": {
                "type": "GeoProperty",
                "value": {
                    "type": "Point",
                    "coordinates": [-3.7, 40.4]  # Example coordinates
                }
            },
            "parentDevice": {
                "type": "Relationship",
                "value": self.parent_device
            }
        }
        
        # Add sensor-specific measurements
        if self.sensor_type == "temperature":
            entity_data["temperature"] = {
                "type": "Property",
                "value": round(value, 2),
                "unitCode": "CEL",
                "accuracy": self.accuracy
            }
            
        elif self.sensor_type == "pressure":
            entity_data["pressure"] = {
                "type": "Property", 
                "value": round(value, 3),
                "unitCode": "BAR",
                "accuracy": self.accuracy
            }
            
        elif self.sensor_type == "vibration":
            entity_data["vibration"] = {
                "type": "Property",
                "value": round(value, 2),
                "unitCode": "H67",  # mm/s
                "accuracy": self.accuracy
            }
            
        elif self.sensor_type == "camera":
            entity_data["defectsDetected"] = {
                "type": "Property",
                "value": int(value)
            }
            entity_data["colorAccuracy"] = {
                "type": "Property",
                "value": round(self.color_accuracy, 1),
                "unitCode": "P1"  # Percentage
            }
            entity_data["registrationAccuracy"] = {
                "type": "Property",
                "value": round(self.registration_accuracy, 3),
                "unitCode": "MMT"  # Millimeter
            }
            
        # Add operational metadata
        entity_data["operatingHours"] = {
            "type": "Property",
            "value": round(self.operating_hours, 1)
        }
        
        entity_data["lastCalibration"] = {
            "type": "Property",
            "value": self.last_calibration.isoformat()
        }
        
        entity_data["batteryLevel"] = {
            "type": "Property", 
            "value": random.randint(85, 100),
            "unitCode": "P1"
        }
        
        # Add alerts if any
        if alerts:
            entity_data["alerts"] = {
                "type": "Property",
                "value": alerts
            }
            
        return entity_data
    
    def create_ul_payload(self, value):
        """Create UltraLight payload for IoT Agent"""
        
        # Map sensor types to UL attribute codes
        ul_mapping = {
            "temperature": "te",
            "pressure": "pr",
            "vibration": "vi",
            "camera": "de"  # defects
        }
        
        ul_code = ul_mapping.get(self.sensor_type, "va")  # Default to 'va' for value
        payload_parts = [f"{ul_code}|{value}"]
        
        # Add additional metrics for camera
        if self.sensor_type == "camera":
            payload_parts.append(f"co|{self.color_accuracy}")  # color accuracy
            payload_parts.append(f"ra|{self.registration_accuracy}")  # registration accuracy
            
        # Add operational data
        payload_parts.append(f"oh|{self.operating_hours}")  # operating hours
        payload_parts.append(f"bl|{random.randint(85, 100)}")  # battery level
        
        return "|".join(payload_parts)
    
    def provision_device(self):
        """Provision sensor device with IoT Agent"""
        try:
            # Define attributes based on sensor type
            attributes = [
                {"object_id": "oh", "name": "operatingHours", "type": "Number"},
                {"object_id": "bl", "name": "batteryLevel", "type": "Number"}
            ]
            
            if self.sensor_type == "temperature":
                attributes.append({"object_id": "te", "name": "temperature", "type": "Number"})
            elif self.sensor_type == "pressure":
                attributes.append({"object_id": "pr", "name": "pressure", "type": "Number"})
            elif self.sensor_type == "vibration":
                attributes.append({"object_id": "vi", "name": "vibration", "type": "Number"})
            elif self.sensor_type == "camera":
                attributes.extend([
                    {"object_id": "de", "name": "defectsDetected", "type": "Number"},
                    {"object_id": "co", "name": "colorAccuracy", "type": "Number"},
                    {"object_id": "ra", "name": "registrationAccuracy", "type": "Number"}
                ])
            
            device_config = {
                "devices": [{
                    "device_id": self.device_id,
                    "entity_name": f"urn:ngsi-ld:Device:{self.device_id}",
                    "entity_type": "Device",
                    "transport": "MQTT",
                    "attributes": attributes,
                    "static_attributes": [
                        {"name": "deviceType", "type": "Text", "value": self.sensor_type},
                        {"name": "manufacturer", "type": "Text", "value": self.sensor_metadata["manufacturer"]},
                        {"name": "model", "type": "Text", "value": self.sensor_metadata["model"]},
                        {"name": "serialNumber", "type": "Text", "value": self.sensor_metadata["serial_number"]},
                        {"name": "parentDevice", "type": "Relationship", "value": self.parent_device},
                        {"name": "protectionRating", "type": "Text", "value": self.sensor_metadata["protection_rating"]},
                        {"name": "accuracy", "type": "Number", "value": self.accuracy}
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
            
            logger.info(f"Sensor {self.device_id} provisioned: {response.status_code}")
            
        except Exception as e:
            logger.error(f"Failed to provision sensor device: {e}")
    
    def publish_sensor_data(self):
        """Publish sensor data to MQTT/FIWARE"""
        try:
            # Generate sensor reading
            value = self.simulate_sensor_value()
            
            # Generate alerts
            alerts = self.generate_sensor_alerts(value)
            
            # Create payloads
            ul_payload = self.create_ul_payload(value)
            
            # Publish UL payload for IoT Agent
            topic = f"ul/{self.api_key}/{self.device_id}/attrs"
            self.mqtt_client.publish(topic, ul_payload)
            
            # Publish alerts if any
            if alerts:
                alert_topic = f"alerts/{self.sensor_type}/{self.device_id}"
                self.mqtt_client.publish(alert_topic, json.dumps(alerts))
                
            timestamp = datetime.now().strftime("%H:%M:%S")
            logger.info(f"[{timestamp}] {self.sensor_type.title()} {self.device_id}: {value:.2f} {self.sensor_metadata['units']}")
            
            if alerts:
                for alert in alerts:
                    logger.warning(f"ALERT: {alert['message']}")
                    
            # Store previous value for rate calculations
            self.previous_value = value
            
        except Exception as e:
            logger.error(f"Error publishing sensor data: {e}")
    
    def run(self):
        """Run the enhanced sensor simulator"""
        logger.info(f"🌡️  Starting Enhanced {self.sensor_type.title()} Sensor {self.device_id}")
        logger.info(f"📊 Range: {self.min_value}-{self.max_value} {self.sensor_metadata['units']}")
        logger.info(f"⚡ Accuracy: ±{self.accuracy} {self.sensor_metadata['units']}")
        logger.info(f"🔗 Parent Device: {self.parent_device}")
        logger.info(f"🏭 Manufacturer: {self.sensor_metadata['manufacturer']} {self.sensor_metadata['model']}")
        logger.info(f"🔄 Update Interval: {self.update_interval} seconds")
        
        try:
            # Connect to MQTT
            self.mqtt_client.connect(self.mqtt_broker, self.mqtt_port, 60)
            self.mqtt_client.loop_start()
            
            # Wait for FIWARE initialization
            time.sleep(30)
            
            # Provision device
            self.provision_device()
            
            # Main sensing loop
            while True:
                self.publish_sensor_data()
                time.sleep(self.update_interval)
                
        except KeyboardInterrupt:
            logger.info(f"Sensor {self.device_id} stopping...")
        except Exception as e:
            logger.error(f"Error in sensor simulator: {e}")
        finally:
            try:
                self.mqtt_client.loop_stop()
                self.mqtt_client.disconnect()
                logger.info(f"Sensor {self.device_id} stopped")
            except:
                pass

if __name__ == "__main__":
    sensor = EnhancedSensorSimulator()
    sensor.run()
```

## Key Features of Enhanced Sensor Simulator:

### 1. **Realistic Industrial Sensor Behavior**
- **Manufacturer specifications**: Real sensor models from Honeywell, Endress+Hauser, SKF, Cognex
- **Calibration cycles**: 6-month calibration intervals with drift simulation
- **Environmental effects**: Ambient temperature, humidity, EMI influence readings
- **Sensor aging**: Gradual drift and accuracy degradation over operating hours

### 2. **Advanced Signal Simulation**
- **Daily cycles**: Temperature sensors show natural daily temperature variations
- **Process cycles**: Equipment-specific operational patterns (heating, cooling, pump cycles)
- **Mechanical patterns**: Vibration sensors detect imbalances, bearing wear
- **Quality patterns**: Vision systems show gradual quality drift over time

### 3. **Industrial Protocols & Standards**
- **Protection ratings**: IP67/IP68 for harsh industrial environments
- **Calibration standards**: IEC 60751, EN 837-1, ISO 16063 compliance
- **Response times**: Realistic sensor response characteristics
- **Resolution and stability**: Manufacturer-specified accuracy metrics

### 4. **Alert Generation System**
- **Range monitoring**: Warnings for near-minimum/maximum values
- **Rate of change**: Rapid temperature change detection
- **Calibration alerts**: Automatic notification for overdue calibrations
- **Critical alarms**: High vibration, dangerous temperature alerts

### 5. **FIWARE Smart Data Models**
- **Device model**: Standard FIWARE Device entity with proper relationships
- **Sensor-specific attributes**: Temperature, pressure, vibration, vision metrics
- **Metadata enrichment**: Manufacturer, model, serial number, installation date
- **Operational data**: Operating hours, battery level, calibration status

### 6. **Multi-Sensor Support**
- **Temperature sensors**: RTD Pt100 with thermal cycles and ambient effects
- **Pressure sensors**: Capacitive sensors with pump cycles and system pressure
- **Vibration sensors**: IEPE accelerometers with mechanical wear patterns
- **Vision cameras**: Machine vision with defect detection and quality metrics

This enhanced sensor simulator provides **authentic industrial IoT sensor behavior** while maintaining full FIWARE compatibility and generating realistic data patterns that mirror actual industrial environments.
