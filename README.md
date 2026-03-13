# OTA Update System — Design and Security Analysis for Automotive Embedded Systems

> A complete design, implementation and security analysis of an Over-The-Air (OTA) firmware update system on the ESP8266 platform, validated against automotive cybersecurity standards.

**Author:** Adewunmi Popoola  
**Degree:** Applikationsingenjör mot industri och fordon (YH) — Hermods Yrkeshögskola  
**Year:** 2026

---

## 📋 Project Overview

Modern vehicles contain between 30 and 150+ Electronic Control Units (ECUs) that require continuous software updates throughout their lifecycle. OTA update capability has been mandatory for type approval under UNECE WP.29/UN R156 since 2022.

This project addresses a core engineering challenge: **how can a secure, reliable, and resource-efficient OTA system be designed and implemented on a constrained embedded platform like the ESP8266?**

The system was developed and validated in a Wokwi simulation environment after hardware failures during the initial physical testing phase — demonstrating that simulation-based development is a viable and rigorous methodology for OTA system validation.

---

## ⚙️ System Architecture

The system follows a three-layer architecture using a **pull model**, where the device initiates all update requests — appropriate for resource-constrained devices with intermittent connectivity.

```
┌─────────────────────────────────────────────────────────────────┐
│                     THREE-LAYER OTA ARCHITECTURE                │
│                                                                 │
│  ┌─────────────────┐    HTTPS/TLS 1.2    ┌──────────────────┐  │
│  │   ESP8266        │◄───────────────────►│  Flask Backend   │  │
│  │   OTA Client     │                     │  Server          │  │
│  │                  │  GET /version       │                  │  │
│  │  ┌────────────┐  │ ──────────────────► │  • Version       │  │
│  │  │    FSM     │  │  JSON manifest      │    manifest      │  │
│  │  │  7 states  │  │ ◄────────────────── │  • Firmware      │  │
│  │  └────────────┘  │                     │    binary        │  │
│  │                  │  GET /firmware/     │  • ECDSA         │  │
│  │  ┌────────────┐  │ ──────────────────► │    signatures    │  │
│  │  │  Security  │  │  chunked binary     │                  │  │
│  │  │  Module    │  │ ◄────────────────── │                  │  │
│  │  └────────────┘  │                     └──────────────────┘  │
│  │                  │                                           │
│  │  ┌────────────┐  │                                           │
│  │  │ Partition  │  │  A/B partition switching + rollback       │
│  │  │ Manager    │  │                                           │
│  │  └────────────┘  │                                           │
│  └─────────────────┘                                            │
└─────────────────────────────────────────────────────────────────┘
```

### Core Components

| Component | Responsibility |
|-----------|---------------|
| **NetworkManager** | Wi-Fi connection, BearSSL TLS config, certificate pinning |
| **OTA Client (FSM)** | Version check, download, verify, partition switch, rollback |
| **Security Module** | CRC32 integrity, SHA-256 hashing, ECDSA P-256 signature verification |
| **PartitionManager** | A/B partition management, firmware activation, rollback |
| **Flask Backend** | Version manifest (JSON), chunked firmware delivery via HTTPS |

---

## 🔄 OTA State Machine (FSM)

The OTA client is implemented as a 7-state Finite State Machine ensuring the device never overwrites active firmware before verification is complete:

```
INIT → CHECK_VERSION → DOWNLOAD → VERIFY → SWITCH_PARTITION → REBOOT
                  ↓           ↓         ↓
               IDLE       ROLLBACK  ROLLBACK
```

| State | Action |
|-------|--------|
| `INIT` | Network and variable initialisation |
| `CHECK_VERSION` | Fetch JSON version manifest via HTTPS |
| `DOWNLOAD` | Chunked firmware download with real-time SHA-256 hashing |
| `VERIFY` | CRC32 + SHA-256 + ECDSA P-256 signature verification |
| `SWITCH_PARTITION` | Activate new firmware on inactive A/B partition |
| `REBOOT` | Restart device on new firmware |
| `ROLLBACK` | Restore previous firmware on any failure |

---

## 🔒 Security Implementation

### Transport Layer
- TLS 1.2 with ECDHE-AES-GCM cipher suite via BearSSL (4 kB buffers for RAM constraints)
- Certificate pinning to prevent AitM (Adversary-in-the-Middle) attacks
- HTTPS-only communication — all HTTP connections rejected

### Firmware Integrity
- **CRC32** — segment-level transfer error detection
- **SHA-256** — full firmware hash computed in real-time during download
- **ECDSA P-256** — digital signature verification using micro-ecc library before activation

### Update Policy
- Anti-rollback protection — version numbers enforced, older versions rejected
- Interrupted download handling — incomplete firmware never written to active partition
- Timeout and retry limits — FSM returns to IDLE after 3 failed attempts

---

## 🛡️ STRIDE Threat Model

| Threat | Attack Scenario | Mitigation |
|--------|----------------|------------|
| **Spoofing** | Rogue server impersonating update backend | Certificate pinning |
| **Tampering** | Firmware modified during transit | ECDSA signature + SHA-256 hash |
| **Repudiation** | No audit trail of updates | Serial logging + FSM state tracking |
| **Information Disclosure** | Firmware exposed in transit | TLS 1.2 encryption |
| **Denial of Service** | Server floods or connection storms | Timeouts + IDLE fallback in FSM |
| **Elevation of Privilege** | Unsigned firmware execution | ECDSA verification mandatory before activation |

---

## 📊 CVSS v3.1 Vulnerability Assessment

8 vulnerabilities were identified and scored. All were addressed through implemented controls or documented as platform limitations:

| Vulnerability | CVSS Score | Severity | Mitigation |
|--------------|-----------|----------|------------|
| No hardware Secure Boot | 7.8 | High | Platform limitation (ESP8266) — documented |
| No flash encryption | 7.5 | High | Platform limitation — ESP32 recommended for production |
| No hardware key storage | 7.2 | High | Software key protection implemented |
| Replay attack via old firmware | 6.8 | Medium | Anti-rollback version policy |
| AitM firmware injection | 8.1 | High | TLS + certificate pinning |
| Unsigned firmware acceptance | 9.1 | Critical | ECDSA mandatory — update aborted on failure |
| Incomplete partition handling | 5.4 | Medium | Rollback mechanism implemented |
| DoS via repeated failed connections | 4.3 | Medium | FSM timeout and IDLE fallback |

---

## 📐 Standards Compliance Analysis

| Standard | Coverage | Gap |
|----------|---------|-----|
| **UNECE WP.29 / UN R156** | Most requirements met | Hardware security features missing on ESP8266 |
| **ISO/SAE 21434** | Good cybersecurity process coverage | No HSM integration |
| **UPTANE (RFC 9334)** | Core principles implemented | No full multi-metadata structure |
| **NIST SP 800-193** | Partial — resiliency mechanisms present | No hardware root-of-trust |

---

## 📈 Performance Results

Tested on a 295 kB firmware binary in Wokwi simulation:

| Metric | Result |
|--------|--------|
| Total OTA cycle time | < 7 seconds |
| Download time | ~4.2 seconds |
| SHA-256 hashing | ~1.1 seconds |
| ECDSA verification | ~0.9 seconds |
| Minimum free heap (TLS handshake) | 27 kB |
| Memory constraint (defined max) | 60 kB |
| **Memory constraint met?** | ✅ Yes |

---

## ✅ Test Results

12 functional tests executed. Results:

| Category | Tests | Passed | Notes |
|----------|-------|--------|-------|
| Functional | 12 | 11 | 1 partial — physical flash write observation not possible in simulation |
| Security | 8 | 8 | All attack simulations passed |

**Security tests included:**
- ✅ Certificate pinning — rogue certificate rejected
- ✅ Manipulated firmware — CRC32 error detected, update aborted
- ✅ Invalid SHA-256 hash — hash mismatch detected, update aborted
- ✅ Invalid ECDSA signature — signature rejected, firmware not activated
- ✅ Replay attack — anti-rollback policy blocked older version
- ✅ Interrupted download — rollback triggered, device remained functional
- ✅ Interrupted verification — rollback triggered
- ✅ DoS-like repeated failures — FSM entered IDLE after 3 attempts

---

## 🛠️ Development Environment

| Tool | Version | Purpose |
|------|---------|---------|
| Visual Studio Code | 1.87 | IDE |
| PlatformIO | 6.1 | Embedded build system |
| Wokwi VS Code extension | 2.1 | ESP8266 simulation |
| Python Flask | — | Backend update server |
| OpenSSL | — | Test certificate generation |
| BearSSL | — | TLS on ESP8266 |
| micro-ecc | — | ECDSA P-256 on device |

---

## 📁 Repository Structure

```
ota-security-analysis/
├── README.md                        # This file
├── src/
│   ├── main.cpp                     # Entry point — WiFi init + OTA start
│   ├── OtaClient.cpp / .h           # FSM-based OTA state machine
│   ├── Security.cpp / .h            # CRC32, SHA-256, ECDSA verification
│   ├── PartitionManager.cpp / .h    # A/B partition + rollback logic
│   └── NetworkManager.cpp / .h      # WiFi, TLS, certificate pinning
├── server/
│   └── server.py                    # Flask backend — version manifest + firmware delivery
├── simulation/
│   └── wokwi-project.json           # Wokwi simulation configuration
├── docs/
│   ├── stride-analysis.md           # Full STRIDE threat model
│   ├── cvss-assessment.md           # CVSS v3.1 vulnerability scoring
│   └── standards-compliance.md      # ISO/SAE 21434, UNECE WP.29, UPTANE gap analysis
└── thesis/
    └── ExamensArbete_Adewunmi.pdf   # Full thesis document
```

---

## 🔑 Key Conclusions

1. A **secure and functional OTA system** can be implemented on a constrained platform like ESP8266 using software-only security controls.
2. **Full OTA cycles complete in under 7 seconds** with memory usage well within defined constraints.
3. The system **resists all tested attack vectors** — firmware manipulation, replay attacks, AitM, and interrupted updates.
4. ESP8266 is **not suitable for production automotive use** due to missing Secure Boot, flash encryption, and hardware key storage.
5. **ESP32 or an automotive-grade MCU** is recommended for production deployment.
6. Wokwi simulation is a **valid methodology** for OTA logic validation in early development phases.

---

## 📚 Key References

- ISO/SAE 21434:2021 — Road Vehicles Cybersecurity Engineering
- UNECE WP.29 / UN R156 — Software Update Management System (SUMS)
- UNECE WP.29 / UN R155 — Cybersecurity Management System (CSMS)
- IETF RFC 9334 — UPTANE: Secure Over-the-Air Updates for Vehicles
- NIST SP 800-193 — Platform Firmware Resiliency Guidelines
- Espressif Systems — ESP8266EX Datasheet and Technical Reference

---

## 👤 Author

**Adewunmi Popoola**  
Application Engineering for Industry and Vehicles (YH)  
Hermods Yrkeshögskola · Göteborg, Sweden  


https://www.linkedin.com/in/adewunmi-popoola-30302a164/

---

*Thesis grade: Submitted March 2026. Evaluated as technically strong with thorough execution.*
