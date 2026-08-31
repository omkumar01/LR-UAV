# Open-Source Alternatives Analysis: Long-Range UAV Systems

This document analyzes existing open-source and semi-open-source frameworks for long-range UAV telemetry and video over cellular networks (4G/LTE). It compares them against the requirements outlined in the High-Level Design (HLD) to identify missing features and architectural gaps.

---

## 1. Rpanion-server

**Overview:** 
Rpanion-server is a Node.js-based companion computer web interface designed specifically for Raspberry Pi and ArduPilot ecosystems. It acts as an onboard orchestrator.

### What it does well:
*   **MAVLink Routing:** Natively integrates `mavlink-router` for efficient telemetry splitting.
*   **Video Streaming:** Supports WebRTC out-of-the-box using GStreamer and the Mediasoup WebRTC server.
*   **Network Management:** Provides a GUI to connect to Wi-Fi and manage VPNs like ZeroTier.

### 🔴 Missing Features (Compared to our HLD):
1.  **No Cloud Relay Architecture:** Rpanion is strictly an *onboard* solution. It expects you to provide your own VPN server and ground station. It does not include a Cloud API backend or a centralized rendezvous server.
2.  **No Fleet Management or RBAC:** It is designed for a 1-to-1 relationship (one operator to one drone). There is no Role-Based Access Control (RBAC) to distinguish between "Pilots" and "Observers" via a JWT-authenticated cloud.
3.  **No Time-Series Observability:** While it shows current system stats, it lacks the Telegraf + Mosquitto + TimescaleDB architecture required to buffer metrics during cellular drops and perform post-flight predictive maintenance analysis.
4.  **No Abstracted Command API:** It passes raw MAVLink. It does not provide a safe, abstracted gRPC API (like our proposed MAVSDK integration) to prevent dangerous manual commands over high-latency networks.

---

## 2. UAVcast (Community Edition)

**Overview:**
UAVcast is a popular software suite for establishing 4G/LTE drone links. It handles modem dial-up, VPN connection, and video forwarding.

### What it does well:
*   **Modem Management:** Excellent built-in support for dialing various 4G LTE USB dongles and HATs (like Quectel and Huawei).
*   **Turnkey VPN:** Automates ZeroTier or WireGuard tunnel establishment.

### 🔴 Missing Features (Compared to our HLD):
1.  **Paywalled Advanced Features:** The most critical feature for BVLOS—low-latency WebRTC video streaming—is locked behind their paid "Pro" version. The free community edition relies on older, higher-latency protocols like RTSP or UDP streams.
2.  **Black-Box Architecture:** The software acts somewhat as a black box. Modifying their internal routing logic to add local endpoints (e.g., routing specific MAVLink messages to a local companion computer script) is difficult compared to writing a raw `mavlink-router.conf`.
3.  **Proprietary Web Dashboard:** If you want a web-based GCS, you are heavily pushed toward their proprietary cloud service rather than building your own custom Next.js dashboard.

---

## 3. Andruav / DroneEngage

**Overview:**
Andruav is an interconnected drone platform that uses cellular networks to link drones to a central web-based Ground Control Station. It provides both the onboard agent and the cloud server.

### What it does well:
*   **End-to-End System:** Provides the onboard agent, the cloud relay server, and the web GCS in one package.
*   **WebRTC Integration:** Natively uses WebRTC for video and data transmission.

### 🔴 Missing Features (Compared to our HLD):
1.  **Non-Standard Routing (No WireGuard/mavlink-router):** Andruav encapsulates MAVLink inside its own WebSockets/WebRTC data channels rather than using a standard flat WireGuard VPN and `mavlink-router`. This makes integrating standard tools like QGroundControl via simple UDP endpoints more complex.
2.  **Monolithic Architecture:** It is designed as an all-in-one ecosystem. Stripping out their Web GCS to use your own custom React dashboard requires significant reverse-engineering of their signaling server.
3.  **Hardware Watchdogs:** It lacks the low-level, hardware-specific watchdogs (like AT-command modem resets via serial) required for high-vibration, high-temperature industrial environments.
4.  **Custom Payload APIs:** It does not provide an easy bridge (like MAVSDK) for custom payloads on the companion computer to interact safely with the flight controller.

---

## Conclusion & Strategy

While these open-source tools are excellent for hobbyists or rapid prototyping, **none of them provide the complete, distributed, and highly secure architecture required for an industrial fleet.**

**Recommended Approach:**
Instead of adopting one of these monolithic frameworks and fighting its limitations, the best path forward is to build a **composite architecture**:
1.  Use the exact same underlying open-source binaries that these projects use (`mavlink-router`, `WireGuard`, `v4l2h264enc`, `Telegraf`).
2.  Write our own lightweight systemd services to manage them.
3.  Build the custom **Cloud Command API (Go)** and **Web Dashboard (Next.js)** entirely from scratch to guarantee total control over RBAC, fleet scaling, and data privacy.
