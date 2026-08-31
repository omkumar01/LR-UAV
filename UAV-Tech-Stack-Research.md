# Long-Range UAV System: Tech Stack Research & Implementation Guide

Based on the High-Level Design (HLD) document for the Long-Range UAV system over 4G/LTE, the architecture requires a mix of off-the-shelf industrial software and custom-built services. The environment introduces constraints like CGNAT, variable latency, and thermal issues, which strictly dictate the need for highly efficient, low-overhead, and performant technologies.

---

## 1. Off-the-Shelf Services & Packages (To Install/Configure)
These components are standard, highly optimized, and should **not** be built from scratch.

### 1.1 UAV Onboard Stack (Raspberry Pi CM4)
*   **OS:** Raspberry Pi OS Lite (64-bit). *Why:* No GUI overhead, native `systemd` for robust process management.
*   **VPN Agent:** WireGuard. *Why:* Runs in the Linux kernel (extremely low CPU usage), stateless (handles cellular tower handoffs and IP changes instantly), uses UDP (no TCP head-of-line blocking).
*   **MAVLink Router:** `mavlink-router` (C++). *Why:* Significantly more CPU-efficient than Python alternatives like MAVProxy. Handles UART to UDP routing with minimal latency.
*   **Hardware Monitor:** Telegraf. *Why:* Lightweight Go binary, minimal RAM usage.
*   **Local Broker:** Eclipse Mosquitto (MQTT). *Why:* C-based, tiny footprint, handles QoS 1 for buffering data during LTE drops.

### 1.2 Cloud Relay Infrastructure (Mumbai VPS)
*   **VPN Gateway:** WireGuard.
*   **WebRTC STUN/TURN:** Coturn. *Why:* Industry standard, C-based, high performance for NAT traversal.
*   **Telemetry Aggregation:** NATS (preferred over MQTT for cloud scale). *Why:* Go-based, millions of messages per second, perfect for fleet-scale MAVLink routing.
*   **Telemetry Translation:** `mavlink2rest` (Rust). *Why:* Blazing fast translation of binary MAVLink to JSON/WebSockets for the web dashboard.
*   **Time-Series DB:** TimescaleDB (PostgreSQL). *Why:* Optimized for time-series hardware metrics.

### 1.3 Ground Control Station
*   **Professional GCS:** QGroundControl (QGC). *Why:* Native MAVLink support, robust UI for professional operators.

---

## 2. Custom Software (To Create From Scratch)
These are the domain-specific services that must be developed to bridge the hardware, network, and user interfaces.

### 2.1 UAV Control API Service (Onboard)
*   **Role:** Translates high-level cloud commands (via gRPC) into local MAVLink commands.
*   **Recommended Tech Stack: Rust + MAVSDK-Rust + Tonic (gRPC)**
    *   *Analysis:* While C++ is traditional, Rust provides C-level performance with guaranteed memory safety. A segmentation fault in a C++ control API mid-flight could be catastrophic. Rust's `Tokio` runtime handles asynchronous UDP/gRPC traffic with incredibly low overhead.

### 2.2 Video Streaming Daemon (Onboard)
*   **Role:** Captures camera feed, hardware encodes it, and streams via WebRTC.
*   **Recommended Tech Stack: C++ + GStreamer + libdatachannel (WebRTC)**
    *   *Analysis:* Python is too slow for real-time video manipulation. C++ allows direct integration with GStreamer's C API for zero-copy DMA (Direct Memory Access) pipelines (`libcamerasrc` -> `v4l2h264enc`). `libdatachannel` is a lightweight C/C++ WebRTC implementation ideal for embedded Linux.

### 2.3 Watchdog / Health Manager (Onboard)
*   **Role:** Monitors LTE health and resets the Quectel modem via AT commands if frozen.
*   **Recommended Tech Stack: Go (Golang) or Rust**
    *   *Analysis:* Avoid Python or Bash for critical watchdogs to save RAM and avoid interpreter startup times. A small compiled Go or Rust binary is highly reliable, easily cross-compiled, and runs with <10MB of RAM.

### 2.4 Cloud Command API Backend (Cloud)
*   **Role:** Manages user Auth (JWT), RBAC, and routes commands from the Web Dashboard to the UAV over WireGuard.
*   **Recommended Tech Stack: Go (Golang) + Fiber + PostgreSQL**
    *   *Analysis:* Go excels at high-concurrency network routing and microservices. It is significantly faster and uses less memory than Python (FastAPI) or Node.js, making it highly cost-effective and performant as the fleet scales.

### 2.5 Web/Mobile Dashboard (Ground)
*   **Role:** UI for payload operators and observers to view video and basic telemetry.
*   **Recommended Tech Stack: Next.js (React) + TypeScript + Zustand + WebRTC API**
    *   *Analysis:* React ecosystem provides the best tools for complex, dynamic UIs. TypeScript ensures type safety for complex MAVLink JSON objects. WebRTC is natively supported in modern browsers for sub-250ms video.

---

## 3. Workflows for Custom Components

### 3.1 UAV Control API Workflow
1.  Initialize gRPC server listening on `10.8.0.2` (WireGuard IP).
2.  Initialize MAVSDK system and connect to `mavlink-router` via local UDP (`127.0.0.1:14540`).
3.  Receive authenticated gRPC command (e.g., `ExecuteRTL`) from Cloud API.
4.  Translate to MAVSDK `action.return_to_launch()`.
5.  Await acknowledgment from Flight Controller.
6.  Return gRPC success/failure response to Cloud API.

### 3.2 Video Streaming Daemon Workflow
1.  Initialize GStreamer pipeline hooking into the CSI camera.
2.  Route frames to CM4 hardware H.264 encoder (`v4l2h264enc`).
3.  Initialize WebRTC PeerConnection via `libdatachannel`.
4.  Listen for incoming WebRTC signaling (Offer/Answer/ICE candidates) via a lightweight WebSocket to the Cloud Server.
5.  Perform STUN/TURN handshake via Coturn.
6.  Inject H.264 NAL units from GStreamer into the WebRTC SRTP track.
7.  Monitor WebRTC Google Congestion Control (GCC) stats; dynamically adjust GStreamer encoder bitrate if LTE signal degrades.

### 3.3 Watchdog Workflow
1.  Run as a high-priority systemd daemon.
2.  Every 5 seconds, send an ICMP ping to the Cloud VPS IP (`10.8.0.1`).
3.  If 6 consecutive pings fail (30s), trigger recovery sequence.
4.  Open serial connection (`/dev/ttyUSB2`) to Quectel Modem.
5.  Send `AT+CFUN=1,1` to soft-reset the modem.
6.  Wait 60 seconds for LTE re-registration before resuming pings.

### 3.4 Cloud Command API Workflow
1.  Web Dashboard user logs in; API issues JWT.
2.  User submits a "Takeoff" command request.
3.  API validates JWT and checks PostgreSQL RBAC tables (is user a 'Pilot' for this specific UAV ID?).
4.  If authorized, API logs the command intent to the database for auditing.
5.  API opens a gRPC client connection to the specific UAV's WireGuard IP.
6.  API forwards command, waits for UAV response, and relays status back to the Web Dashboard via HTTP/WebSocket.

---

## 4. Minimum Viable Product (MVP) Phases

### MVP 1: The UAV Control API & Cloud Backend
*   **Goal:** Securely trigger an RTL (Return to Launch) command from the web.
*   **UAV API MVP:** A Rust binary that only exposes an `RTL` gRPC endpoint and uses MAVSDK to trigger RTL.
*   **Cloud API MVP:** A Go HTTP server with hardcoded authentication that exposes a `/trigger-rtl` endpoint, forwarding it to the UAV via gRPC.

### MVP 2: The Video Pipeline
*   **Goal:** Achieve 1-way sub-250ms video over the internet.
*   **Video Daemon MVP:** A C++ program that hardcodes a WebRTC connection to a specific viewer (bypassing complex signaling), taking a 720p 30fps stream from the Pi camera, encoding it, and streaming it via Coturn to a simple static HTML/JS page. No dynamic bitrate adjustment yet.

### MVP 3: Web Dashboard & Telemetry
*   **Goal:** Display real-time attitude and battery data in the browser.
*   **Dashboard MVP:** A React app that connects to the Cloud VPS `mavlink2rest` WebSocket. It parses the JSON and renders a basic textual artificial horizon (Roll/Pitch/Yaw) and a battery percentage progress bar.

### MVP 4: The Watchdog
*   **Goal:** Automatic modem recovery.
*   **Watchdog MVP:** A Go script that only pings the cloud and sends the `AT` reset command on failure. Logging is done simply to `stdout` (captured by `journalctl`).
