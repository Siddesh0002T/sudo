# 🏛️ BlueMesh — Deep Technical Architecture & Fraud Detection Engine

This document provides a comprehensive technical breakdown of the mesh routing logic, cryptographic proof protocol, fraud detection algorithms, and API specifications.

---

## 1. Mesh Routing & Self-Healing Protocol (ESP-NOW Multi-Hop)

### 1.1 Ad-Hoc Topology Formation
Classroom nodes operate in a connectionless peer-to-peer broadcast mode. Unlike standard Wi-Fi station mode or Bluetooth SIG Mesh with heavy provisioning overhead, BlueMesh uses an optimized **Lightweight Distance-Vector Protocol (LDVP)** tailored for ESP-NOW.

```
       [ Wearable Wristband ] (Broadcasts beacon on Ch 1)
                  |
         +--------+--------+
         | (RSSI: -58dBm)  | (RSSI: -82dBm)
         v                 v
   [ Node 101-A ]     [ Node 101-B ]
         |                 |
         +--------+--------+ (Selects path with lowest Hop Count & best RSSI)
                  v
             [ Node 102 ]
                  v
         [ Gateway Sink Node ]
```

### 1.2 Self-Healing & Reroute Logic
Every node maintains a lightweight neighbor table in RAM containing:
- `Neighbor_MAC`
- `Hop_To_Gateway`
- `Link_Quality_RSSI`
- `Last_Keepalive_Timestamp`

**Algorithm (Node Relay Decision):**
1. When a packet arrives, node checks if `Nonce` is in its recent 100-packet buffer. If yes, it drops the frame to stop loops.
2. If `Hop_Count >= MAX_HOPS (8)`, drop packet to avoid broadcast storms.
3. Node increments `Hop_Count` by 1 and forwards the frame to its designated upstream parent with lowest cost metric ($Cost = Hop \times 100 - RSSI$).
4. If upstream parent fails to ACK after 2 retries, the node dynamically switches to the next lowest cost neighbor.

---

## 2. Cryptographic Proof-of-Presence & Anti-Replay Architecture

```
[ Capacitive Fingerprint Sensor ]
               │
               ▼ 1:1 Match OK (Hardware Interrupt)
┌─────────────────────────────────────────────────────────────┐
│                       ESP32 Secure Core                     │
│  1. Retrieve Device Master Key (K_dev) from eFuse           │
│  2. Generate Nonce = CSPRNG()                               │
│  3. Read Rolling Epoch Timestamp (T_epoch)                  │
│  4. Token = HMAC-SHA256(K_dev, UUID || T_epoch || Nonce)   │
│  5. Encrypt Frame payload via AES-128-CCM                   │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼ (ESP-NOW Airframe)
┌─────────────────────────────────────────────────────────────┐
│                     Central Cloud Engine                    │
│  1. Lookup K_dev for Device UUID                            │
│  2. Verify |T_current - T_epoch| <= 90 seconds              │
│  3. Check Nonce not previously seen                         │
│  4. Compute expected HMAC and verify constant-time equality │
│  5. Append record to SHA-256 Chained Ledger Block           │
└─────────────────────────────────────────────────────────────┘
```

### 2.1 Why Traditional BLE Beacons Fail (And How BlueMesh Fixes It)
Standard BLE tags (iBeacon/Eddystone) broadcast a static UUID. An attacker with a smartphone app (e.g. nRF Connect) can record this UUID and broadcast it continuously from a dummy device inside the classroom while the person is outside.

**The BlueMesh Fix:**
Because the token changes every 60 seconds using a cryptographic rolling nonce and shared secret derived on initial enrollment, recorded packets become invalid after 90 seconds.

---

## 3. Real-Time Fraud Detection & Anomaly Rules Engine

The backend ingestion microservice evaluates all incoming heartbeats against continuous heuristic anomaly detection pipelines:

### Rule 1: "Mark & Leave" Staff Absenteeism Detector
- **Trigger:** Staff member registers verified biometric check-in at 08:30 AM.
- **Rule Formulation:**
  $$\text{DwellTime}(u) = \sum_{k=1}^{N} \Big(\text{LastSeen}(u, k) - \text{FirstSeen}(u, k)\Big)$$
  $$\text{If } \Big(\text{CurrentTime} - \text{LastSeen}(u)\Big) > T_{\text{threshold}} \ (e.g. \ 25\text{ mins}) \ \land \ \text{CampusStatus} == \text{ACTIVE}$$
- **Action:** Generates `FLAG_ABSENTEEISM_EARLY_DEPARTURE`. Deducts attendance proportion automatically unless justified by signed administrative leave ticket.

### Rule 2: "Ghost Student" Welfare Anomaly Detector
- **Trigger:** End-of-month scholarship entitlement calculation.
- **Rule Formulation:**
  A student profile is flagged if their verified attendance count over 30 days is zero or statistically inconsistent with normal classroom cohort density, yet marked manually in legacy registers.
- **Action:** Generates `FLAG_GHOST_RECORD_AUDIT`, freezes Direct Benefit Transfer (DBT) scholarship disbursement pending physical inspection.

### Rule 3: Geo-Teleportation & Impossible Speed Anomaly
- **Trigger:** Two consecutive beacons received from geographically incompatible rooms within an impossible timeframe:
  $$\Delta t = t_2 - t_1 < \frac{\text{Distance}(\text{Room}_1, \text{Room}_2)}{V_{\text{max\_human\_walk}}}$$
- **Action:** Flags simultaneous band cloning or antenna misconfiguration (`FLAG_RF_COLLISION`).

---

## 4. Immutable SHA-256 Audit Ledger Specification

Every attendance event creates an append-only block:

```json
{
  "block_height": 10492,
  "previous_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "timestamp": "2026-09-01T09:14:02.110Z",
  "event_type": "BIOMETRIC_CHECKIN",
  "payload": {
    "user_id": "STU-2026-081",
    "device_uuid": "00:E0:4C:81:92:AA",
    "room_id": "ROOM-101-A",
    "gateway_id": "GW-CAMPUS-01",
    "rssi": -62,
    "verified_proof": "HMAC_8f3a971c2b4c"
  },
  "admin_override": null,
  "current_hash": "9a31bc0891deca2540b6101f3014a069e2c6df41c5949219e2170362fe0281cb"
}
```

If an administrator modifies a record, the change is recorded as an `ADMIN_OVERRIDE` event containing the administrator's digital signature and justification, preserving full historical transparency.

---

## 5. REST & WebSocket API Schema

### 5.1 Gateway Ingestion Endpoint
`POST /api/v1/gateway/telemetry/batch`
```json
{
  "gateway_id": "GW-DELHI-SCH-104",
  "batch_timestamp": 1788261242,
  "packets": [
    {
      "uuid": "A4C1389B0012",
      "timestamp": 1788261240,
      "nonce": 9841203,
      "token": "a1f94bc8...",
      "room_id": "101",
      "rssi": -55
    }
  ]
}
```

### 5.2 Live WebSocket Heatmap Feed
`ws://server/ws/campus/live`
```json
{
  "type": "PRESENCE_UPDATE",
  "data": {
    "room_id": "101-A",
    "student_count": 38,
    "teacher_present": true,
    "active_users": ["STU-081", "STU-082", "STF-201"],
    "alerts": []
  }
}
```

---

<div align="center">
  <b>BlueMesh • Architecture Document • SIH 2026</b>
</div>
