# 🛠️ PRAMAAN (BlueMesh) — Hardware Engineering Specification

This document details the embedded hardware design, microcontroller pinouts, communication protocols, radio power optimization, and bill of materials for the PRAMAAN BlueMesh system.

---

## 1. Subsystem Breakdown

The hardware ecosystem comprises three core physical devices:

```
┌─────────────────────────────────────────────────────────────┐
│                    1. Wearable Smart Band                   │
│   [ESP32-C3 / S3] <--> [Capacitive Fingerprint] <--> [LiPo] │
└──────────────────────────────┬──────────────────────────────┘
                               │ (ESP-NOW Encrypted Broadcast)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 2. Classroom Mesh Relay Node                │
│    [ESP32-WROOM-32D] <--> [Status LED] <--> [5V DC Wall]    │
└──────────────────────────────┬──────────────────────────────┘
                               │ (Dynamic Multi-Hop Relay)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                  3. Campus Gateway Master Sink              │
│    [ESP32 Master / Pi] <--> [4G LTE Module (EC200U)]        │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Wearable Biometric Wristband

### 2.1 Microcontroller: ESP32-C3 SuperMini / XIAO ESP32-S3
- **CPU:** 32-bit RISC-V single-core processor up to 160 MHz (or Dual-core Xtensa LX7 for S3)
- **Wireless:** 2.4 GHz Wi-Fi (802.11 b/g/n) + Bluetooth 5.0 (LE) + ESP-NOW
- **Cryptographic Hardware Acceleration:** AES-128/256, SHA-256, RSA, HMAC, Digital Signature peripheral
- **Ultra-Low-Power (ULP) Deep Sleep:** ~5 µA sleep current

### 2.2 Biometric Sensor: Micro Capacitive Touch Sensor
- **Model:** FPM10A / R307 / Grow Capacitive Sensor (AS608 compatible UART interface)
- **Resolution:** 508 DPI
- **False Acceptance Rate (FAR):** < 0.001%
- **False Rejection Rate (FRR):** < 0.1%
- **Enrollment Time:** < 0.2s
- **Interface:** Hardware Serial (UART RX/TX at 57600 baud)

### 2.3 Pin Connection Map (Wearable Band)

| Sensor Pin | ESP32-C3 Pin | Function |
| :--- | :--- | :--- |
| **VCC (3.3V)** | `3V3` | Power Supply |
| **GND** | `GND` | Common Ground |
| **TX** | `GPIO 20` (U0RXD) | Serial Data from Sensor to MCU |
| **RX** | `GPIO 21` (U0TXD) | Serial Data from MCU to Sensor |
| **WAKE / TOUCH_INT** | `GPIO 2` | Interrupt pin wakes ESP32 from Deep Sleep on finger contact |
| **STATUS_LED** | `GPIO 8` | RGB NeoPixel indicator (Blue: Mesh, Green: Match, Red: Error) |

### 2.4 Power Profiling & Battery Calculation
- **Battery:** 3.7V 300mAh Lithium-Polymer (LiPo) cell with TP4056 charge management.
- **Normal State (Deep Sleep):** $5\,\mu\text{A}$
- **Finger Contact Wake + Verification (350ms):** $38\,\text{mA}$
- **ESP-NOW Frame Burst (15ms at +20dBm):** $120\,\text{mA}$
- **Periodic Beacon (Every 60s during school hours):** $\text{Average Consumption} \approx 0.35\,\text{mAh/hour}$
- **Expected Battery Longevity:** **~28 to 35 school days** per charge cycle.

---

## 3. Classroom Mesh Relay Node

The Classroom Relay Node is a plug-and-forget device installed on classroom walls, powered by a standard 5V wall adapter.

### 3.1 Node Hardware Architecture
- **MCU:** ESP32-WROOM-32D (Dual-Core Xtensa LX6 @ 240MHz, 4MB SPI Flash, PCB antenna)
- **Power:** 5V/1A Micro-USB / Type-C wall brick with onboard AMS1117 3.3V LDO regulator.
- **Storage:** 1MB dedicated SPIFFS / LittleFS partition acting as an offline circular FIFO packet ring buffer.

### 3.2 Radio Parameters & Range Optimization
- **RF Protocol:** ESP-NOW Unicast & Broadcast over 2.4GHz IEEE 802.11 action frames.
- **Data Rate:** 1 Mbps (Long Range Mode enabled: PHY rate lowered for enhanced penetration through concrete walls).
- **Link Budget:** Output power +20 dBm; Receiver sensitivity -98 dBm.
- **Indoor Range:** Up to 45 meters through 2 standard masonry walls.
- **Outdoor / Corridor Range:** Up to 180 meters line-of-sight.

---

## 4. ESP-NOW Binary Packet Specification

Packets are optimized into a fixed 64-byte payload for minimal airtime and zero fragmentation:

```c
typedef struct __attribute__((packed)) {
    uint8_t  magic_byte;         // 0xBM (BlueMesh Header Identifier)
    uint8_t  packet_type;        // 0x01: Check-in, 0x02: Heartbeat, 0x03: Alert
    uint8_t  device_uuid[8];     // Unique 64-bit Hardware ID
    uint32_t timestamp_epoch;    // Rolling Unix Epoch (seconds)
    uint32_t nonce;              // Cryptographic Random Nonce
    uint8_t  hmac_signature[16]; // Truncated HMAC-SHA256 Token
    uint8_t  hop_count;          // Incremented at each mesh relay (0 to 10)
    uint8_t  origin_room_id[4];  // First listening node's Room Tag
    int8_t   rssi_origin;        // Signal strength at first reception point
    uint8_t  battery_level;      // Battery percentage (0 - 100%)
    uint8_t  checksum;           // XOR frame check sequence
    uint8_t  reserved[24];       // Reserved for future sensor expansion (Temp/IMU)
} BlueMeshPacket_t;
```

---

## 5. Central Edge Gateway & Cloud Bridge

### 5.1 Hardware Stack
- **Option A (Standalone Cellular):** ESP32-S3 connected to Quectel EC200U-CN 4G LTE modem via UART AT Commands.
- **Option B (Campus Ethernet):** Raspberry Pi 4 / ESP32-Ethernet (W5500 / LAN8720) directly plugged into administrative router.

### 5.2 Gateway Core Responsibilities
1. **Packet De-Duplication:** Maintains a sliding-window Bloom filter / LRU hash table of nonces received over the last 180 seconds to discard redundant mesh echoes.
2. **NTP Precision Time Sync:** Attaches high-precision GPS/NTP timestamps to incoming packet batches.
3. **TLS Encryption:** Wraps batched payloads in TLS 1.3 payloads for HTTPS/MQTT transmission to cloud endpoints.

---

## 6. Complete Bill of Materials (BOM) Cost Matrix

| Item | Component Description | Quantity per School (15 Rooms) | Unit Price (INR) | Total Cost (INR) |
| :--- | :--- | :--- | :--- | :--- |
| 1 | ESP32-C3 SuperMini Boards (Wearables) | 50 (Staff Pilot) | ₹220 | ₹11,000 |
| 2 | Capacitive Fingerprint Sensors | 50 | ₹480 | ₹24,000 |
| 3 | 300mAh LiPo Batteries + Straps | 50 | ₹135 | ₹6,750 |
| 4 | ESP32-WROOM-32D (Classroom Nodes) | 15 | ₹190 | ₹2,850 |
| 5 | 5V 1A Wall Power Adapters | 15 | ₹65 | ₹975 |
| 6 | Master Gateway Node + 4G Modem | 1 | ₹1,250 | ₹1,250 |
| 7 | Custom ABS/PLA 3D Cases | 66 units | ₹30 | ₹1,980 |
| **TOTAL** | **Full Campus Pilot Deployment** | — | — | **₹48,805 (~$580 USD)** |

---

<div align="center">
  <b>BlueMesh Engineering Specification • SIH 2026 Hardware Track</b>
</div>
