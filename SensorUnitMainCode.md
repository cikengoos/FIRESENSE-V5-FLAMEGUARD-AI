import dht
from machine import Pin
import time
import network
import urequests
import json

# ========== CONFIGURATION ==========
temp_threshold = 40
humid_threshold = 30

SSID = "WIFI_SSID"
PASSWORD = "WIFI_PASSWORD"

# Your Flask server endpoint
DEVICE_SERIAL = "0000"
SERVER_URL = f"http://000.000.000.000:5000/api/telemetry/{DEVICE_SERIAL}"

# Pin setup
sensor = dht.DHT11(Pin(19))
digital_pin = Pin(11`, Pin.IN, Pin.PULL_DOWN)

# Global variables
temp_value = 0
hum_value = 0
sensor_success = False

# ========== FUNCTIONS ==========

def update_dht11_readings():
    """Read DHT11 and update global temp_value and hum_value variables"""
    global temp_value, hum_value, sensor_success
    
    try:
        sensor.measure()
        temp_value = sensor.temperature()  
        hum_value = sensor.humidity()    
        sensor_success = True
        return True
    except Exception as e:
        print("DHT11 error:", e)
        temp_value = 0
        hum_value = 0
        sensor_success = False
        return False

def digital_read():
    """Read the digital pin - returns 0 or 1"""
    return digital_pin.value()

def get_status_code():
    """
    Return 2-digit CC code with meanings:
    "00" = Offline - no sensor data
    "01" = Safe - normal operation
    "02" = Alert - sensor out of range
    "03" = Danger - manual override / critical
    """
    global temp_value, hum_value, sensor_success
    
    # OFFLINE (00) - No sensor data
    if not sensor_success:
        return "00"
    
    esp32 = digital_read()
    
    # DANGER (03) - Digital pin active (emergency button pressed)
    if esp32 == 1:
        return "03"
    
    # ALERT (02) - Temperature too high OR humidity too low
    if temp_value > temp_threshold or hum_value < humid_threshold:
        return "02"
    
    # SAFE (01) - Everything normal
    return "01"

def assemble_14digit():
    """
    Assemble 14-digit string: SSSS + CC + AA + TTT + HHH
    SSSS = Serial (4 digits)
    CC = Status code (2 digits)
    AA = Digital/Analog value (2 digits, currently digital_read)
    TTT = Temperature (3 digits)
    HHH = Humidity (3 digits)
    """
    global temp_value, hum_value, sensor_success
    
    serial = DEVICE_SERIAL
    cc = get_status_code()
    aa = f"{digital_read():02d}"  # Digital value as 2 digits (00 or 01)
    temp_3digit = f"{temp_value:03d}"
    hum_3digit = f"{hum_value:03d}"

    if not sensor_success:
        temp_3digit = "000"
        hum_3digit = "000"
    
    # 4 + 2 + 2 + 3 + 3 = 14 digits
    data_14digit = f"{serial}{cc}{aa}{temp_3digit}{hum_3digit}"
    return data_14digit

def send_data():
    """Send data to Flask server"""
    global temp_value, hum_value
    
    cc_value = get_status_code()
    digital_val = digital_read()
    
    # Status meanings for display
    status_meanings = {
        "00": "OFFLINE",
        "01": "SAFE",
        "02": "ALERT",
        "03": "DANGER"
    }
    
    # ai_value based on status (True if not SAFE)
    if cc_value == "01":
        ai_value = False
    else:
        ai_value = True
    
    payload = {
        "temp": temp_value,
        "humid": hum_value,
        "digital": digital_val,
        "status_code": cc_value,
        "ai_value": ai_value
    }
    
    # Assemble 14-digit string
    data_14digit = assemble_14digit()
    
    print(f"14-digit: {data_14digit}")
    print(f"Format: {data_14digit[:4]} {data_14digit[4:6]} {data_14digit[6:8]} {data_14digit[8:11]} {data_14digit[11:14]}")
    print(f"Serial: {data_14digit[:4]} | CC: {data_14digit[4:6]} ({status_meanings.get(cc_value, 'UNKNOWN')}) | "
          f"Digital: {data_14digit[6:8]} | Temp: {data_14digit[8:11]}°C | Hum: {data_14digit[11:14]}%")
    
    try:
        response = urequests.post(
            SERVER_URL,
            json=payload,
            headers={"Content-Type": "application/json"}
        )
        
        print(f"Response: {response.status_code}")
        if response.status_code == 200:
            print("✓ Data sent to server")
        else:
            print(f"✗ Server error: {response.status_code}")
        
        response.close()
        return True
        
    except Exception as e:
        print(f"Send error: {e}")
        return False

def connect_wifi():
    """Connect to WiFi network"""
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    
    if not wlan.isconnected():
        print("Connecting to WiFi...")
        wlan.connect(SSID, PASSWORD)
        
        for i in range(10):
            if wlan.isconnected():
                break
            time.sleep(1)
    
    if wlan.isconnected():
        print(f"WiFi connected! IP: {wlan.ifconfig()[0]}")
        return True
    else:
        print("WiFi failed")
        return False

def main():
    """Main loop"""
    if not connect_wifi():
        return
    
    print("Sensor started! Sending data every 3 seconds...")
    print("14-digit Format: SSSS CC AA TTT HHH")
    print("-" * 50)
    
    while True:
        if update_dht11_readings():
            print(f"✓ Sensor OK - Temp: {temp_value}°C, Hum: {hum_value}%")
        else:
            print("✗ Sensor failed - sending offline status")
        
        send_data()
        print("-" * 50)
        time.sleep(3)

# ========== RUN ==========
if __name__ == "__main__":
    main()

# FIRESENSE-V5-FLAMEGUARD-AI
Version 6
