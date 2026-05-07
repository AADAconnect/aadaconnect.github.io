# AADAconnect

**A cloud-ready IoT control platform for ESP32-based smart devices — built for real-world reliability, not demos.**

AADAconnect is a full-stack web application that lets users register, configure, monitor, and control smart plug devices in real time over MQTT — directly from a browser, with no native app required. It is designed from the ground up with a production-first mindset: hashed credentials, secure MQTT topic isolation per user, graceful offline handling, and a clean, mobile-first UI that works on any device.

---

## Table of Contents

- [Why AADAconnect](#why-aadaconnect)
- [Key Features](#key-features)
- [Project Structure](#project-structure)
- [How It Works — Web Side](#how-it-works--web-side)
  - [Architecture Diagram](#architecture-diagram)
  - [Page-by-Page Flow](#page-by-page-flow)
  - [MQTT Topic Architecture](#mqtt-topic-architecture)
  - [State & Data Flow](#state--data-flow)
- [Security Model](#security-model)
- [UI & Design System](#ui--design-system)
- [Roadmap](#roadmap)
- [Rights & Contact](#rights--contact)

---

## Why AADAconnect

Most open-source IoT dashboards are either too complex (requiring Node.js backends, databases, and server infrastructure) or too simple (single-device demos with no user separation, no persistence, and no real-world polish). AADAconnect sits in a different category entirely:

| Feature | Typical IoT Demo | AADAconnect |
|---|---|---|
| Hosting | Requires a server | Pure static files — GitHub Pages |
| Authentication | Plaintext or none | Hashed credentials, stored locally |
| Multi-device | Rarely | Yes — dynamic device list via MQTT |
| User isolation | No | Yes — per-user MQTT topic namespacing |
| Device provisioning | Manual / separate tool | Built-in WiFi setup flow |
| UI quality | Functional | Production-grade, mobile-first |
| Offline handling | Crashes | Graceful timeouts & status feedback |
| Backend required | Yes (usually) | No — device handles auth directly |

The result is a platform that can be deployed entirely on GitHub Pages, requires no backend server, and still delivers a secure, multi-user, real-time control experience.

---

## Key Features

**Device Management**
- Real-time ON/OFF control via MQTT
- Live online/offline detection with 5-second timeout fallback
- Quick-toggle directly from the dashboard grid
- Per-device control page with animated power button
- Rename devices and assign them to rooms

**Room Organization**
- Create custom rooms with emoji labels
- Assign multiple devices to a room
- Per-room online device count
- Filter devices by room, online status, or name

**Device Provisioning**
- Generate a new device ID from the dashboard
- Redirect to WiFi setup flow with credentials pre-loaded via `sessionStorage`
- Send SSID + password + hashed user credentials directly to the ESP32 access point at `192.168.4.1/prov`

**Dashboard**
- Four layout modes: Grid, List, Bento, Compact
- Search by device name or ID
- Sort by name, status, or custom order
- Light/dark theme with system-preference sync

**MQTT Integration**
- Connects over WSS to a cloud EMQX broker
- Live connection status indicator
- Subscribes to device list, per-device online status, and per-device power state independently
- Publishes control commands and status check requests

---

## Project Structure

```
aadaconnect.github.io/
│
├── index.html          # Entry point — redirects to login.html
├── login.html          # Authentication — hashes credentials, stores session
├── dashboard.html      # Main hub — device grid, rooms, MQTT status, provisioning
├── control.html        # Per-device control — power button, rename, room assign
├── wifi_setup.html     # Device provisioning — sends WiFi credentials to ESP32
│
└── README.md
```

---

## How It Works — Web Side

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USER'S BROWSER                               │
│                                                                     │
│  ┌──────────┐    ┌──────────────┐    ┌────────────┐                │
│  │ index    │───▶│  login.html  │───▶│ dashboard  │                │
│  │ .html    │    │              │    │   .html     │                │
│  └──────────┘    │ Hash email + │    │             │◀──────────┐   │
│                  │ password     │    │ Device grid │           │   │
│                  │ Store in     │    │ Rooms view  │           │   │
│                  │ localStorage │    │ MQTT status │           │   │
│                  │ + session    │    │             │           │   │
│                  └──────────────┘    └──────┬──────┘           │   │
│                                             │                  │   │
│                              ┌──────────────┼──────────────┐   │   │
│                              │              │              │   │   │
│                              ▼              ▼              │   │   │
│                    ┌──────────────┐  ┌────────────┐        │   │   │
│                    │ wifi_setup   │  │ control    │────────┘   │   │
│                    │   .html      │  │  .html     │            │   │
│                    │              │  │            │            │   │
│                    │ Reads device │  │ Power btn  │            │   │
│                    │ ID from      │  │ Rename     │            │   │
│                    │ sessionStore │  │ Room assign│            │   │
│                    └──────┬───────┘  └─────┬──────┘           │   │
│                           │               │                    │   │
└───────────────────────────┼───────────────┼────────────────────┘   │
                            │               │  MQTT (WSS)             │
                            │    ┌──────────▼──────────┐             │
                            │    │   EMQX Cloud Broker  │             │
                            │    │  (Asia Southeast 1)  │             │
                            │    └──────────┬───────────┘             │
                            │               │  MQTT                   │
                            │    ┌──────────▼───────────┐             │
                            └───▶│     ESP32 Device      │             │
                                 │  (Smart Plug / Switch)│             │
                                 └───────────────────────┘             │
                                                                        │
                         Theme sync via localStorage ──────────────────┘
```

---

### Page-by-Page Flow

#### 1. `index.html` — Entry Point
A lightweight redirect page. When a user opens the site, they are immediately sent to `login.html` via an HTTP `meta refresh`. The page displays a "Redirecting..." message as a fallback.

---

#### 2. `login.html` — Authentication

The login page handles all credential management without a backend server.

**How it works:**

1. User enters email and password.
2. Both values are **hashed client-side** (using a deterministic hash function) before ever being stored or used.
3. The resulting `hashed_email` and `hashed_password` are stored in `localStorage` as the account key.
4. On successful login, the full credential object (including hashes) is written to `sessionStorage` as `current_user`.
5. The user is redirected to `dashboard.html`.

**Account switching:** If a different email is entered and an account already exists in `localStorage`, a confirmation sheet is shown before overwriting the saved account.

**Remember Me:** When enabled, the account is persisted in `localStorage` and automatically restored on the next visit without requiring re-entry of credentials.

```
User Input (email + password)
        │
        ▼
   hashValue() ──▶ hashed_email, hashed_password
        │
        ▼
  localStorage  ──▶ persisted account (if Remember Me)
  sessionStorage ──▶ current_user (for this tab session)
        │
        ▼
  dashboard.html
```

---

#### 3. `dashboard.html` — Main Hub

This is the core of the web application. On load it:

1. Reads `current_user` from `sessionStorage` (redirects to login if missing).
2. Connects to the MQTT broker over WSS.
3. Subscribes to the user's **fetch topic** to receive the full device list from the ESP32 backend.
4. Subscribes to each device's **OF (online/offline)** and **status** topics.
5. Publishes `check_status` to each device's OF topic to trigger a response.
6. Renders devices in a grid with live online/offline and ON/OFF state.

**Navigating to a device:** Tapping a device card writes the device ID and a status cache snapshot to `sessionStorage`, then navigates to `control.html`.

**Adding a device:** Generates a new device ID (`AADA-XXXXXXXX`), writes a provisioning payload to `sessionStorage`, then navigates to `wifi_setup.html`.

---

#### 4. `control.html` — Device Control

Reads the `active_device` and `device_status_cache` from `sessionStorage` on load. Establishes its own independent MQTT connection.

**Power control flow:**
1. User taps the large power button.
2. Page publishes `ON` or `OFF` to the device's `/control` topic.
3. Page sets a visual "pending" state and starts a 5-second timeout.
4. Device confirms the change by publishing back to the `/status` topic.
5. If no confirmation arrives within 5 seconds, the UI reverts to the last known state.

The control page also handles device **rename** (stored in `localStorage` under the user's account) and **room assignment** (persisted similarly), both of which are reflected back on the dashboard without any server call.

---

#### 5. `wifi_setup.html` — Device Provisioning

Reads provisioning data from `sessionStorage` (set by dashboard when "Add Device" is tapped).

**Provisioning flow:**
1. User connects their phone/laptop to the ESP32's temporary WiFi access point (`AADA-Plug`).
2. User enters the home WiFi SSID and password in the form.
3. On submit, the page sends a `POST` request to `http://192.168.4.1/prov` — the ESP32's local HTTP server — with a JSON payload containing:
   - SSID and WiFi password
   - `hashed_email` and `hashed_password` (so the device knows which MQTT namespace to use)
   - `device_id`
4. On success, `sessionStorage` is cleared of provisioning data and the user is redirected back to the dashboard.

---

### MQTT Topic Architecture

All topics are namespaced per user using their hashed credentials. This means no user can ever accidentally see or control another user's devices — even on a shared broker — because the topic paths are cryptographically derived from their credentials.

```
smartplug/{hashed_email}/{hashed_password}/fetch
    └─ Subscribed by dashboard on connect
    └─ Device publishes full device list as JSON: { device_list: [...] }

smartplug/{hashed_email}/{hashed_password}/{device_id}/OF
    └─ Subscribed per device after device list is received
    └─ Dashboard publishes:  "check_status"
    └─ Device publishes:     online message string  OR  "offline"

smartplug/{hashed_email}/{hashed_password}/{device_id}/status
    └─ Subscribed per device
    └─ Device publishes:     "ON"  or  "OFF"

smartplug/{hashed_email}/{hashed_password}/{device_id}/control
    └─ Published by dashboard/control page
    └─ Payload:  "ON"  or  "OFF"
```

---

### State & Data Flow

| Data | Storage | Scope |
|---|---|---|
| Hashed credentials | `localStorage` | Permanent (until logout) |
| Current session user | `sessionStorage` | Tab session |
| Device names (renamed) | `localStorage` | Permanent, per account |
| Room assignments | `localStorage` | Permanent, per account |
| Room definitions | `localStorage` | Permanent, per account |
| Active device (to control) | `sessionStorage` | Single navigation |
| Device status cache | `sessionStorage` | Single navigation |
| Provisioning payload | `sessionStorage` | Single navigation |
| Theme preference | `localStorage` | Permanent |
| Device list | In-memory (MQTT) | Live session |
| Online/power state | In-memory (MQTT) | Live session |

---

## Security Model

AADAconnect uses a **zero-backend, device-authoritative** security model:

- **No secrets in frontend code.** Credentials are never hardcoded. The MQTT broker password is a shared service credential, not a user credential — actual user identity is encoded in the topic path.
- **Hashed credentials as topic namespaces.** User email and password are hashed before use, and this hash forms the MQTT topic prefix. No other user can subscribe to your topics without knowing your exact credentials.
- **Device-side validation.** The ESP32 firmware validates incoming commands. Unauthorized commands on a topic the device doesn't recognize are ignored.
- **No credential transmission to any AADA server.** During provisioning, credentials go directly from the browser to the ESP32 over the local network (`192.168.4.1`), never through a cloud service.
- **Session-scoped navigation data.** Provisioning payloads and device navigation state live in `sessionStorage`, which is cleared when the tab closes.

> **Note:** The current hash function is deterministic and client-side. A future update will replace it with a cryptographic hash (e.g., SHA-256 via SubtleCrypto) for stronger security. Token-based authentication is also planned.

---

## UI & Design System

The interface is built as a single cohesive design system shared across all pages:

- **Framework:** Vanilla HTML, CSS, JavaScript — no build step, no framework dependency
- **Font:** Inter (Google Fonts)
- **Theming:** CSS custom properties (`data-theme="dark"` / `"light"`) — full dark and light mode, synced across tabs via `localStorage`
- **Layout:** Mobile-first, phone-frame optimized, bottom navigation
- **Icons:** Font Awesome 6 (CDN)
- **MQTT client:** `mqtt.js` (CDN, WSS transport)
- **Design principles:** No glow effects, minimal chrome, production polish — the UI aims to feel like a native app, not a web form

---

## Roadmap

- [ ] SHA-256 credential hashing via SubtleCrypto API
- [ ] Token-based authentication with broker-side ACL enforcement
- [ ] OTA firmware update trigger from dashboard
- [ ] Progressive Web App (PWA) with offline caching
- [ ] Device capability reporting (firmware version, uptime, energy usage)
- [ ] Group device control (turn all devices in a room on/off)
- [ ] MQTT-based in-app messaging (AADAmeet integration)
- [ ] Push notifications for device state changes
- [ ] Advanced automation / scheduling

---

## Rights & Contact

This project is **not licensed for public reuse, redistribution, or commercial use** unless explicitly permitted by the maintainers. All rights are reserved by the **AADA / AADAconnect team**.

- **Organization:** AADAconnect
- **Platform:** GitHub Organization — [aadaconnect.github.io](https://aadaconnect.github.io)
- **Contact:** *(to be provided)*

> Security issues should **not** be disclosed publicly. Contact the team directly.
