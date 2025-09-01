# 🎯 Capod项目蓝牙密钥提取指南
### 阅读语言：[English](README.md) | [简体中文](README.zh-CN.md)

<br>

## 📋 指南用途说明
本指南提供了从Apple设备提取蓝牙密钥（IRK和ENC_KEY）的方法，用于在[Capod](https://github.com/d4rken-org/capod)APP中进行高级设置。获取的安全密钥使用户能够在Capod应用中启用高级功能和自定义选项。

<br>
<br>

## ⚙️ 环境要求与准备
### 必需软件环境

  - **[VMware Workstation Player](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion)**
  - **[Ubuntu 24.04 LTS](https://ubuntu.com/download/desktop)**
  - **Python 3:** (Ubuntu系统自带)

### 关键前提条件

  1. **必须完全关闭Windows主机的蓝牙功能**
  2. **必须在VMware中正确配置USB控制器（USB 3.0+）**
  3. **必须确保蓝牙适配器在VMware中被成功识别**

<br>
<br>

## 🔧 环境配置步骤
### 1. 验证蓝牙适配器识别

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
# 在Ubuntu终端中运行，必须能看到蓝牙设备信息
lsusb | grep -i bluetooth
# 预期输出：Bus 001 Device 004: ID 0a5c:21e8 Broadcom Corp. BCM20702A0 Bluetooth 4.0
```

### 2. 安装必要软件包

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install bluez bluez-tools blueman python3 python3-pip libbluetooth-dev
pip3 install pybluez
```

### 3. 启动蓝牙服务

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
sudo systemctl enable bluetooth
sudo systemctl start bluetooth
sudo systemctl status bluetooth  # Confirm status is active (running)
```

<br>
<br>

## 📝 脚本部署
### 代码来源说明
Python脚本代码来源于GitHub开源项目 [d4rken-org/capod](https://github.com/d4rken-org/capod), 由用户 [@kavishdevar](https://github.com/kavishdevar)提供。

### **创建脚本文件**

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
nano get_ble_keys.py
```

### 粘贴以下代码

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

### **保存并设置权限**

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
bash
# 保存文件：Ctrl+O → Enter → Ctrl+X
chmod +x get_ble_keys.py
```

<br>
<br>

## 🔍 获取目标设备MAC地址
### 方法一：蓝牙扫描获取

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
bluetoothctl scan on
# 等待设备出现，记录MAC地址（格式：XX:XX:XX:XX:XX:XX）
# 按Ctrl+C停止扫描
```

### 方法二：查看已连接设备

1. **将设备连接到<code>Ubuntu蓝牙</code>**
2. **打开：<code>系统设置</code> → <code>蓝牙</code>**
3. **点击已连接设备的名称（不是开关）**
4. **在弹出界面中查看<code>MAC地址</code>**

<br>
<br>

## 🚀 运行脚本与参数解释
### 运行命令详情

![Bash](https://img.shields.io/badge/language-Bash-blue)
```bash
sudo python3 get_ble_keys.py FC:XX:XX:XX:XX:XX
# 检查确保运行的文件名和 Python 代码段里的文件名一致
# 把获取到的MAC地址替换XX直接运行即可
```

---

| 参数                  | 说明                        | 如何获取 |
|-----------------------|-----------------------------|---------------------------------------------------------------|
| **sudo**              | 管理员权限                   | 需要硬件访问权限 |
| **python3**           | Python解释器                 | Ubuntu系统自带 |
| **get_ble_keys.py**   | 脚本文件名                   | 用户创建的脚本文件 |
| **FC:55:57:61:9B:D3** | 目标设备MAC地址              | **通过蓝牙扫描或查看已连接设备获得** （替换为你的实际设备地址）  |

<br>
<br>

## ✅ 预期输出结果
### 成功输出示例

![Text](https://img.shields.io/badge/language-Text-blue)
```text
Proximity Keys:
  IRK: C7 A2 09 96 8D E9 3B 0D E3 55 77 57 B5 E9 7A 32
  ENC_KEY: BB 5E E6 45 5E A9 EA 79 68 83 EE 40 B3 FB D8 E9
```

### 输出值说明

- **IRK** (身份解析密钥)：用于识别隐私地址设备的设备身份解析密钥
- **ENC_KEY** (长期密钥)：用于安全通信的长期加密密钥

<br>
<br>

## ⚠️ 重要注意事项
### 法律声明

- 所有使用的软件均为官方免费版本
- Python代码来源于开源项目，仅用于教育研究目的
- 请仅在自己的设备或获得授权的设备上使用

### 技术要求

- **必须确保 <mark>Windows主机蓝牙完全关闭</mark>**
- **必须验证 <mark>lsusb | grep -i bluetooth</mark> 能够检测到设备**
- **<mark>MAC地址</mark> 必须准确，否则无法连接到设备**

### 故障排除

- **权限错误**：确认使用<code>**sudo**</code>
- **设备未找到**：检查MAC地址是否正确
- **连接失败**：确认设备在范围内且可被发现
- **无输出**：设备可能不支持此协议

<br>
<br>

## 🔄 完整操作流程

1. **环境准备**：Windows蓝牙关闭 → VMware连接设备 → Ubuntu验证识别
1. **软件安装**：安装必要的依赖包和Python库
1. **脚本部署**：创建和配置提取脚本
1. **目标获取**：扫描或查看已连接设备以获取MAC地址
1. **执行提取**：运行脚本并记录输出结果
1. **环境恢复**：断开设备连接 → 恢复Windows蓝牙功能

---

## 文档信息

- 🎯 **主要用途**：为[Capod](https://github.com/d4rken-org/capod) APP提供蓝牙密钥进行高级设置
- 🐧 **兼容系统**：在VMware上运行的<code>**[Ubuntu](https://ubuntu.com/download/desktop) 24.04 LTS**</code>
- 📦 **软件来源**：所有软件均为官方免费版本
- 💻 **代码来源**：<code>**GitHub [d4rken-org/capod](https://github.com/d4rken-org/capod)**</code>项目 <code>**[@kavishdevar](https://github.com/kavishdevar)**</code>
- 🔬 **使用目的**：<mark>**仅限于教育研究和授权测试**</mark>
