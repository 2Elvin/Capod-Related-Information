# 🎯 Bluetooth Key Extraction Guide for Capod Project
### Read in: [English](README.md) | [简体中文](README.zh-CN.md)

<br>

## 📋 Purpose of This Guide
This guide provides a method to extract Bluetooth keys (IRK and ENC_KEY) from Apple devices for advanced settings in the [Capod](https://github.com/d4rken-org/capod) APP. The obtained security keys enable users to access advanced features and customization options in the Capod application.

<br>
<br>

## ⚙️ Environmental Requirements and Preparation
### Required Software Environment

  - **[VMware Workstation Player](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion)**
  - **[Ubuntu 24.04 LTS](https://ubuntu.com/download/desktop)**
  - **Python 3:** (Pre-installed with Ubuntu)

### Critical Prerequisites

  1. **Bluetooth must be completely disabled on Windows host**
  2. **USB controller must be properly configured in VMware (USB 3.0+)**
  3. **Bluetooth adapter must be successfully recognized in VMware**

<br>
<br>

## 🔧 Environment Configuration Steps
### 1. Verify Bluetooth Adapter Recognition

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
# Run in Ubuntu terminal, must see Bluetooth device information
lsusb | grep -i bluetooth
# Expected output: Bus 001 Device 004: ID 0a5c:21e8 Broadcom Corp. BCM20702A0 Bluetooth 4.0
```

### 2. Install Required Software Packages

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install bluez bluez-tools blueman python3 python3-pip libbluetooth-dev
pip3 install pybluez
```

### 3. Start Bluetooth Service

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
sudo systemctl enable bluetooth
sudo systemctl start bluetooth
sudo systemctl status bluetooth  # Confirm status is active (running)
```

<br>
<br>

## 📝 Script Deployment
### Code Source Statement
Python script code is sourced from GitHub open-source project [d4rken-org/capod](https://github.com/d4rken-org/capod), provided by user [@kavishdevar](https://github.com/kavishdevar).

### **Create Script File**

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
nano get_ble_keys.py
```

### Paste the Following Code

![Python](https://img.shields.io/badge/language-Python-blue)
```python
#!/usr/bin/env python3
import sys
import socket

PROXIMITY_KEY_TYPES = {
    0x01: "IRK",
    0x04: "ENC_KEY",
}

def parse_proximity_keys_response(data):
    if len(data) < 7 or data[4] != 0x31:
        return None
    key_count = data[6]
    keys = []
    offset = 7
    for _ in range(key_count):
        if offset + 3 >= len(data):
            break
        key_type = data[offset]
        key_length = data[offset + 2]
        offset += 4
        if offset + key_length > len(data):
            break
        key_bytes = data[offset:offset + key_length]
        keys.append((PROXIMITY_KEY_TYPES.get(key_type, f"TYPE_{key_type:02X}"), key_bytes))
        offset += key_length
    return keys

def hexdump(data):
    return " ".join(f"{b:02X}" for b in data)

def main():
    if len(sys.argv) != 2:
        print(f"Usage: {sys.argv[0]} <MAC>")
        sys.exit(1)

    bdaddr = sys.argv[1]
    PSM = 0x1001

    handshake = bytes.fromhex("00 00 04 00 01 00 02 00 00 00 00 00 00 00 00 00")
    key_req = bytes.fromhex("04 00 04 00 30 00 05 00")

    sock = socket.socket(socket.AF_BLUETOOTH, socket.SOCK_SEQPACKET, socket.BTPROTO_L2CAP)
    sock.connect((bdaddr, PSM))
    sock.send(handshake)
    sock.send(key_req)

    try:
        while True:
            pkt = sock.recv(1024)
            keys = parse_proximity_keys_response(pkt)
            if keys is not None:
                print("Proximity Keys:")
                for name, key_bytes in keys:
                    print(f"  {name}: {hexdump(key_bytes)}")
                break
    finally:
        sock.close()

if __name__ == "__main__":
    main()
```

### **Save and Set Permissions**

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
bash
# Save file: Ctrl+O -> Enter -> Ctrl+X
chmod +x get_ble_keys.py
```

<br>
<br>

## 🔍 Obtain Target Device MAC Address
### Method 1: Bluetooth Scanning

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
bluetoothctl scan on
# Wait for devices to appear, record MAC address (format: XX:XX:XX:XX:XX:XX)
# Press Ctrl+C to stop scanning
```

### Method 2: View Connected Devices

1. **Connect device to <code>Ubuntu Bluetooth</code>**
2. **Open: <code>System Settings</code> -> <code>Bluetooth</code>**
3. **Click on connected device name (not the switch)**
4. **View <code>MAC address</code> in popup interface**

<br>
<br>

## 🚀 Run Script and Parameter Explanation
### Run Command Details

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
sudo python3 get_ble_keys.py FC:XX:XX:XX:XX:XX
# Replace XX with the obtained MAC address and run directly
```

---

| Parameter             | Explanation                 | How to Obtain |
|-----------------------|-----------------------------|------------------------------------------------------------|
| **sudo**              | Administrator privileges    | Required for hardware access |
| **python3**           | Python interpreter          | Pre-installed with Ubuntu |
| **get_ble_keys.py**   | Script filename             | User-created script file |
| **FC:55:57:61:9B:D3** | Target device MAC address   | **Obtained via Bluetooth scanning or connected device** (replace with your actual address) |

<br>
<br>

## ✅ Expected Output Results
### Successful Output Example

![Text](https://img.shields.io/badge/language-Text-blue)
```text
Proximity Keys:
  IRK: C7 A2 09 96 8D E9 3B 0D E3 55 77 57 B5 E9 7A 32
  ENC_KEY: BB 5E E6 45 5E A9 EA 79 68 83 EE 40 B3 FB D8 E9
```

### Output Value Explanation

- **IRK** (Identity Resolving Key): Device identity resolution key for identifying privacy address devices
- **ENC_KEY** (Long Term Key): Long-term encryption key for secure communication

<br>
<br>

## ⚠️ Important Notes
### Legal Statement

- All software used are official free versions
- Python code is sourced from open-source projects, for educational research purposes only
- Please use only on your own devices or authorized devices

### Technical Requirements

- **Must ensure** Windows host Bluetooth is completely disabled
- **Must verify <code>lsusb | grep -i bluetooth</code>** can detect the device
- **MAC address must be accurate**, otherwise cannot connect to device

### Troubleshooting

- **Permission errors**: Confirm using **<code>sudo</code>**
- **Device not found**: Check if MAC address is correct
- **Connection failed**: Confirm device is in range and discoverable
- **No output**: Device may not support this protocol

<br>
<br>

## 🔄 Complete Operation Process

1. **Environment preparation**: Windows Bluetooth off -> VMware connect device -> Ubuntu verify recognition
1. **Software installation**: Install necessary dependency packages and Python libraries
1. **Script deployment**: Create and configure extraction script
1. **Target acquisition**: Scan or view connected devices to obtain MAC address
1. **Execution extraction**: Run script and record output results
1. **Environment restoration**: Disconnect device -> Restore Windows Bluetooth function

---

## Document Information:

- 🎯 **Primary purpose**: Provide Bluetooth keys for [Capod project](https://github.com/d4rken-org/capod) APP advanced settings
- 🐧 **Compatible systems**: **<code>[Ubuntu](https://ubuntu.com/download/desktop) 24.04</code>** LTS on VMware
- 📦 **Software sources**: All software are official free versions
- 💻 **Code source**: **<code>GitHub [d4rken-org/capod](https://github.com/d4rken-org/capod) project [@kavishdevar](https://github.com/kavishdevar).</code>**
- 🔬 **Usage purpose**: **<mark>Limited to educational research and authorized testing</mark>**
