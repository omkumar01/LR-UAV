# **UAV 4G Control Architecture — System Diagram**

This document contains a comprehensive Mermaid diagram visualizing the entire long-range UAV communication and control system over 4G/LTE cellular networks, including CGNAT traversal, WireGuard VPN, MAVLink routing, WebRTC video streaming, and failsafe mechanisms.

## **Complete System Architecture Diagram**

```mermaid
graph TD
    %% ==================== UAV ONBOARD SYSTEM ====================
    subgraph UAV["UAV Onboard System"]
        direction TB

        subgraph FlightController["Flight Controller (Pixhawk H7)"]
            FC["ArduPilot H7<br/>Cube Orange+"]
            IMU["Triple-Redundant IMU"]
            Baro["Barometer"]
            FC --> IMU
            FC --> Baro
        end

        subgraph CompanionComputer["Companion Computer (Raspberry Pi CM4)"]
            direction TB
            RPi["Raspberry Pi CM4<br/>4GB RAM, 32GB eMMC<br/>Raspberry Pi OS Lite 64-bit"]
            Camera["Camera Module 3<br/>MIPI CSI-2"]
            Modem["Quectel EM06-E<br/>LTE Cat 6 Modem<br/>M.2 Key-B"]
            BuckConv["Buck Converter<br/>4A @ 3.3V"]
            UBEC["Isolated UBEC<br/>from Flight Battery"]

            RPi --> Camera
            RPi --> Modem
            UBEC --> BuckConv
            BuckConv --> Modem
            UBEC --> RPi
        end

        subgraph SystemdServices["systemd Managed Services"]
            WG["WireGuard Interface<br/>wg0 (10.8.0.2)"]
            MavRouter["mavlink-router<br/>UART + UDP Endpoints"]
            MavSDK["MAVSDK Control Agent<br/>gRPC API"]
            VideoDaemon["WebRTC Video Daemon<br/>GStreamer + v4l2h264enc"]
            Telegraf["Telegraf Agent<br/>System Metrics"]
            Watchdog["Health Watchdog<br/>Modem Reset Logic"]
            SSH["SSH Daemon<br/>wg0 only"]

            WG --> MavRouter
            MavRouter --> MavSDK
            MavRouter --> VideoDaemon
            VideoDaemon --> WG
            Telegraf --> WG
            Watchdog --> Modem
            SSH --> WG
        end

        %% Internal connections
        FC -- "MAVLink 2<br/>UART 921600bps" --> RPi
        RPi --> SystemdServices
    end

    %% ==================== CELLULAR NETWORK ====================
    subgraph CellularNetwork["Cellular Network & Internet"]
        direction TB
        CellTower["LTE Tower<br/>Jio / Airtel / Vi<br/>Bands 3/5/40"]
        CGNAT["Carrier-Grade NAT<br/>Private IP (10.x.x.x)"]
        InternetCloud["Public Internet"]

        Modem -- "LTE Cat 6<br/>Carrier Aggregation" --> CellTower
        CellTower --> CGNAT
        CGNAT --> InternetCloud
    end

    %% ==================== CLOUD RELAY INFRASTRUCTURE ====================
    subgraph CloudRelay["Cloud Relay Infrastructure<br/>Mumbai Data Center"]
        direction TB
        CloudVPS["Cloud VPS<br/>Ubuntu 22.04 LTS"]
        WGServer["WireGuard VPN Gateway<br/>UDP 51820"]
        Coturn["Coturn STUN/TURN Server<br/>WebRTC NAT Traversal"]
        TelemetryBroker["MQTT/NATS Broker<br/>Telemetry Aggregation"]
        APIServer["UAV C2 Backend API<br/>FastAPI / Go<br/>JWT + RBAC"]
        DB["PostgreSQL<br/>Audit Log"]
        Grafana["Grafana Dashboard<br/>TimescaleDB"]

        CloudVPS --> WGServer
        CloudVPS --> Coturn
        CloudVPS --> TelemetryBroker
        CloudVPS --> APIServer
        APIServer --> DB
        TelemetryBroker --> Grafana
    end

    %% ==================== GROUND CONTROL STATION ====================
    subgraph GCS["Ground Control Station"]
        direction TB
        GCSLaptop["Ruggedized Laptop<br/>WireGuard Client"]
        QGC["QGroundControl<br/>UDP 14550"]
        MobileApp["Mobile Web Dashboard<br/>React / Next.js"]
        BrowserViewer["Browser WebRTC Viewer"]

        GCSLaptop --> QGC
        GCSLaptop --> MobileApp
        MobileApp --> BrowserViewer
    end

    %% ==================== INTER-SYSTEM CONNECTIONS ====================
    %% UAV to Cellular
    Modem -- "4G/LTE Uplink<br/>1.5-3.5 Mbps" --> CellTower

    %% Cellular to Cloud (VPN tunnel)
    InternetCloud -- "WireGuard Tunnel<br/>Outbound UDP 51820<br/>PersistentKeepalive=25s" --> CloudVPS

    %% Cloud to GCS (VPN tunnel)
    CloudVPS -- "WireGuard Tunnel<br/>10.8.0.3" --> GCSLaptop

    %% WebRTC video path
    VideoDaemon -- "WebRTC SRTP<br/>STUN/TURN via Coturn" --> Coturn
    Coturn -- "SRTP UDP Stream" --> BrowserViewer

    %% Telemetry path
    MavRouter -- "Filtered MAVLink 2<br/>UDP 14550" --> QGC

    %% MQTT monitoring path
    Telegraf -- "MQTT QoS 1<br/>Buffered on reconnect" --> TelemetryBroker

    %% gRPC C2 path
    APIServer -- "gRPC / MAVSDK" --> MavSDK

    %% SSH management path
    SSH -- "SSH over WireGuard<br/>Key-based Auth" --> CloudVPS

    %% ==================== FAILSAFE LOGIC ====================
    subgraph FailsafeLogic["Failsafe & Recovery Logic"]
        direction LR
        HeartbeatLoss["FC Heartbeat Timeout<br/>> 3s"]
        NetworkDrop["Cellular Outage<br/>Ping > 30s"]
        RPiCrash["RPi Crash/Reboot<br/>UART Loss"]
        ModemCrash["Modem Hang<br/>AT Reset"]
        VideoCrash["Video Pipeline Crash<br/>systemd Restart"]

        HeartbeatLoss --> FC
        NetworkDrop --> Watchdog
        RPiCrash --> FC
        ModemCrash --> Modem
        VideoCrash --> VideoDaemon
    end

    %% ==================== LATENCY BUDGET ====================
    subgraph LatencyBudget["Latency Budget (Ideal Conditions)"]
        direction TB
        L1["Camera Capture to HW Encode: 15ms"]
        L2["WebRTC Packetization & Encryption: 5ms"]
        L3["4G Radio Air Interface: 40-120ms"]
        L4["Cloud VPS Routing: 10ms"]
        L5["GCS Download & Decode: 20ms"]
        LTotal["Total Glass-to-Glass: 90-170ms"]

        L1 --> L2 --> L3 --> L4 --> L5 --> LTotal
    end

    %% ==================== BANDWIDTH ESTIMATES ====================
    subgraph BandwidthEstimates["Bandwidth Estimates"]
        direction TB
        B1["Video (Adaptive): 500Kbps - 3Mbps"]
        B2["Telemetry (Downlink): ~20Kbps"]
        B3["Command (Uplink): <5Kbps"]
        B4["Monitoring/VPN Overhead: ~10Kbps"]
        BTotal["Total Uplink: 1.5-3.5Mbps"]

        B1 --> B2 --> B3 --> B4 --> BTotal
    end
end
```

## **MAVLink Routing Detail**

```mermaid
graph LR
    subgraph "Flight Controller"
        Ardu["ArduPilot Telemetry Stream"]
    end

    subgraph "RPi CM4 - mavlink-router"
        UART_In["UART In Buffer<br/>/dev/ttyAMA0:921600"]
        RouterCore{"MAVLink 2 Routing Engine<br/>ID-based Filtering"}
        UDP_Local1["UDP 127.0.0.1:14540<br/>Local MAVSDK Consumer"]
        UDP_Local2["UDP 127.0.0.1:14569<br/>Local Logger / mavlink2rest"]
        UDP_Remote["UDP 10.8.0.3:14550<br/>GCS over WireGuard"]
    end

    Ardu -- "921600 bps" --> UART_In
    UART_In --> RouterCore
    RouterCore -- "Local MAVSDK Consumer" --> UDP_Local1
    RouterCore -- "Local Logger" --> UDP_Local2
    RouterCore -- "Filtered Remote Stream<br/>SRx: ATT 5-10Hz, GPS 5Hz,<br/>SYS_STATUS 1-2Hz, PARAMS 0Hz" --> UDP_Remote
    UDP_Remote -- "4G/Internet via wg0" --> GCS["Ground Control Station"]
```

## **Command and Control Sequence**

```mermaid
sequenceDiagram
    participant Operator as "GCS / QGC"
    participant WG as "Cloud VPN (WireGuard)"
    participant Router as "mavlink-router (CM4)"
    participant FC as "ArduPilot (FC)"

    Operator->>WG: MAVLink COMMAND_LONG (Takeoff)
    WG->>Router: Forward UDP Packet
    Router->>FC: Route over UART
    FC-->>Router: COMMAND_ACK (Accepted)
    Router-->>WG: Forward UDP Packet
    WG-->>Operator: Acknowledgment Received

    Note over Operator,FC: Scenario: Cellular network drops during flight
    Operator-xWG: MAVLink HEARTBEAT (Lost)
    WG-xRouter:
    Router-xFC:
    Note over FC: 3-second heartbeat timeout exceeded
    FC->>FC: Trigger GCS Failsafe -> Execute RTL
```

## **Network & NAT Traversal Sequence**

```mermaid
sequenceDiagram
    participant UAV as "UAV Modem (10.x.x.x)"
    participant ISP as "ISP CGNAT"
    participant VPS as "Cloud VPN (Public IP)"
    participant GCS as "Ground Station (10.8.0.3)"

    UAV->>ISP: Send UDP Packet (wg0, 10.8.0.2)
    ISP->>VPS: Translate IP & Forward (UDP 51820)
    VPS->>GCS: Route to GCS (WireGuard tunnel)

    Note over UAV,ISP: Tower Handoff / IP Change Occurs
    UAV--xISP: Connection Interrupted

    UAV->>ISP: Send new UDP Packet (New IP)
    ISP->>VPS: Forward from New IP
    Note over VPS: WireGuard stateless roaming detects new IP automatically
    VPS->>GCS: Route to GCS (Session maintained)
```

## **Video Streaming Pipeline**

```mermaid
graph LR
    Cam["MIPI CSI Camera<br/>Raspberry Pi Camera Module 3"] -- "NV12 Raw Frames" --> libcamera["libcamera API"]
    libcamera -- "GStreamer Pipeline" --> Encoder["v4l2h264enc HW Encoder<br/>H.264 Baseline Profile<br/>No B-frames"]
    Encoder -- "H.264 NAL Units" --> WebRTC["WebRTC C++ Daemon<br/>libdatachannel / MediaMTX"]
    WebRTC <-->|"STUN/TURN Negotiation"| CloudVPS["Cloud TURN Server<br/>Coturn"]
    CloudVPS <-->|"SRTP UDP Stream"| Viewer["GCS Browser/Mobile Viewer"]
```

## **Security Architecture**

```mermaid
graph TD
    subgraph "Defense in Depth Layers"
        direction TB

        subgraph NetworkLayer["1. Network Layer Security"]
            WG_Encrypt["WireGuard VPN<br/>ChaCha20-Poly1305<br/>All traffic encrypted"]
            NoPublicPorts["No public ports exposed<br/>SSH on wg0 only"]
        end

        subgraph MAVLinkLayer["2. MAVLink 2 Message Signing"]
            SigningKey["32-byte Secret Key<br/>Flashed via SETUP_SIGNING"]
            CryptoSign["Cryptographic Signature<br/>+ Incrementing Timestamp"]
            ReplayProtection["Rejects unsigned messages<br/>Rejects stale timestamps"]
        end

        subgraph AuthLayer["3. Authentication & RBAC"]
            JWT["JWT-based Authentication"]
            RBAC["Role-Based Access Control<br/>Observers (read-only)<br/>Pilots (command authority)"]
        end

        subgraph HostLayer["4. Host Security"]
            KeyAuth["SSH Key-based Auth Only"]
            UnattendedUpgrades["Unattended Security Upgrades"]
            NoWiFi["Wi-Fi Disabled<br/>Reduces interference"]
        end

        NetworkLayer --> MAVLinkLayer --> AuthLayer --> HostLayer
    end
```

## **Failure and Failsafe Matrix**

```mermaid
graph TD
    subgraph "Failure Detection & Recovery"
        direction TB

        subgraph FailureModes["Failure Modes"]
            F1["Loss of 4G Connectivity"]
            F2["High Network Latency/Jitter"]
            F3["Raspberry Pi Crash/Reboot"]
            F4["LTE Modem Crash"]
            F5["Power Brownout"]
            F6["Video Pipeline Failure"]
            F7["GPS Failure/Jamming"]
            F8["Cloud Server Outage"]
        end

        subgraph Detection["Detection Mechanisms"]
            D1["FC: Missing GCS Heartbeats > 3s"]
            D2["WebRTC GCC / MAVLink ping RTT"]
            D3["FC: Loss of UART traffic"]
            D4["Watchdog: Ping failure > 30s to VPS"]
            D5["FC voltage sensors / RPi dmesg"]
            D6["systemd: Process crash / exit code"]
            D7["FC EKF variance / satellite lock"]
            D8["VPN tunnel collapse / heartbeat lost"]
        end

        subgraph Fallback["Fallback Behavior"]
            B1["FC enters LOITER, then RTL after 10s"]
            B2["Video bitrate degrades 3Mbps -> 500Kbps"]
            B3["FC enters RTL, RPi watchdog reboots"]
            B4["AT reset command to modem"]
            B5["FC initiates immediate LAND"]
            B6["systemd auto-restart (Restart=always)"]
            B7["Dead reckoning / optical flow / altitude hold"]
            B8["Continue to last waypoint, then RTL"]
        end

        subgraph Recovery["Recovery Mechanisms"]
            R1["VPN reconnects when cellular returns"]
            R2["Bitrate scales up as RSRP/RSRQ improve"]
            R3["RPi boots, establishes VPN, resumes routing"]
            R4["Modem re-registers on cellular network"]
            R5["Operator investigates hardware"]
            R6["Daemon re-initializes libcamera"]
            R7["Operator uses video feed to assess"]
            R8["Redundant cloud servers / fleet scale"]
        end

        F1 --> D1 --> B1 --> R1
        F2 --> D2 --> B2 --> R2
        F3 --> D3 --> B3 --> R3
        F4 --> D4 --> B4 --> R4
        F5 --> D5 --> B5 --> R5
        F6 --> D6 --> B6 --> R6
        F7 --> D7 --> B7 --> R7
        F8 --> D8 --> B8 --> R8
    end
```
