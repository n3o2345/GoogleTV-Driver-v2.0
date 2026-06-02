# AndroidTV Control4 Driver v2.0

A Control4 DriverWorks driver for controlling Android TV / Google TV devices using the **Android TV Remote Service v2** protocol over TCP — no ADB or Developer Options required for standard operation.

> **Credit:** Driver created by **Greg Moerler**. This repository is a distribution of his work.

---

## Overview

This driver integrates Android TV and Google TV devices into a Control4 OS 3.x smart home system. It communicates over a native TLS-encrypted TCP connection (port 6466 for commands, port 6467 for pairing), enabling full remote control, power management, app launching, and real-time feedback — all without requiring ADB debugging or Developer Options on the TV.

The included **AndroidTV_Tool** Windows utility handles certificate extraction and RSA public key retrieval needed during the one-time pairing setup along with an app list tool (ADB needs to be enabled on device to use but can be turned back off once data is captured).

---

## Features

- **No ADB required** for pairing, control, app launching, or current-app feedback
- **TLS-encrypted communication** via Android TV Remote Service v2 (TCP 6466/6467)
- **Auto TLS negotiation** — tries TLS 1.3 first, falls back to SSLv2/3
- **Wake-on-LAN** support for powering on devices remotely
- **Real-time feedback**: current app, power status, model/vendor info, MAC address
- **App launching** — up to 20 configurable app URLs/package names
- **Full keymap remapping** — every button mappable to any Android keycode
- **Passthrough mode** for seamless mini-driver app switching
- **Room OFF behavior** — configurable: do nothing, pause, or turn off
- **Debug mode** for troubleshooting connection issues
- **Bundled certificate** — includes `AndroidTV.p12` so no manual cert generation is needed

---

## Contents

```
├── driver/
│   └── Android_v2.0.c4z         # Control4 driver package (install this)
└── AndroidTV_Tool/
    ├── TV_Toolkit.exe             # GUI tool for RSA key extraction
    ├── adb.exe                    # Android Debug Bridge
    ├── openssl.exe                # OpenSSL binary for cert inspection
    ├── AdbWinApi.dll              # ADB Windows runtime
    ├── AdbWinUsbApi.dll           # ADB USB runtime
    ├── libcrypto-4-x64.dll        # OpenSSL crypto library
    └── libssl-4-x64.dll           # OpenSSL SSL library
```

---

## Requirements

- **Control4 OS 3.x** or newer
- **Android TV Remote Service** enabled on the target device (enabled by default on most Android TV / Google TV devices)
- **Static IP address or DHCP reservation** for the Android TV device
- **Network access** from the Control4 controller to TCP ports **6466** and **6467**
- **Windows PC** (for one-time RSA key extraction with `AndroidTV_Tool`)

---

## Installation

1. Copy `driver/Android_v2.0.c4z` to your Control4 driver directory or install via Composer.
2. Add the driver to your project and set the **network binding** to the Android TV device's IP address.
3. Proceed to [Pairing Setup](#pairing-setup).

---

## Pairing Setup

Pairing is a one-time process that exchanges RSA certificates with the device.

### Step 1 — Extract the Device Public Key

Use `AndroidTV_Tool/TV_Toolkit.exe` on a Windows PC connected to the same network:

1. Launch **TV_Toolkit.exe**
2. Enter the Android TV device IP address
3. Click **Get Public Key**
4. Copy the **Modulus**  and **Exponent** (typically `65537`)

Alternatively, using OpenSSL from a command prompt on any networked machine:

```sh
openssl s_client -connect <DEVICE_IP>:6467 </dev/null 2>/dev/null | openssl x509 -noout -text
```

Or via ADB (requires Developer Options and ADB debugging enabled on the TV):

```sh
adb connect <DEVICE_IP>:5555
adb shell "openssl s_client -connect 127.0.0.1:6467 2>/dev/null </dev/null | openssl x509 -noout -text"
```

### Step 2 — Enter Key Values in Composer

In the driver's Properties panel in Control4 Composer:

- **Device Public Key Modulus** → paste the colon-separated hex modulus
- **Device Public Key Exponent** → enter `65537` (or the value from the certificate)

### Step 3 — Pair

1. Click **Begin Pairing Android TV Device** in the driver actions
2. A PIN code will appear on the TV screen
3. Enter the PIN and click **Finish Pairing Android TV Device (Enter PIN)**
4. If the driver does not connect automatically, click **Connect To Command Interface**

---

## Configuration

### TLS Method

| Setting | Behavior |
|---|---|
| `Auto` *(default)* | Tries TLS 1.3 first; falls back to SSLv23 if offline |
| `tlsv13` | Forces TLS 1.3 only |
| `sslv23` | Forces SSLv23 negotiation |

### App Launching

Configure up to **20 app launch slots** in Properties using Android package names or deep links:

```
com.netflix.ninja
com.google.android.youtube.tv
market://launch?id=com.google.android.tv.youtube.tv
```

**Common package names:**

| App | Package Name |
|---|---|
| Netflix | `com.netflix.ninja` or `com.netflix.mediaclient` |
| YouTube | `com.google.android.youtube.tv` |
| YouTube TV | `com.google.android.tv.youtube.tv` |
| Disney+ | `com.disneyplus` |
| Hulu | `com.hulu.plus` |
| Prime Video | `com.amazon.amazonvideo.livingroom` |
| Plex | `com.plexapp.android` |
| Google Play Store | `com.android.vending` |
| Settings | `com.android.settings` |

### Passthrough Mode

- **On** *(default)*: Mini-app drivers remain selected after app launch and pass commands through directly
- **Off**: Driver reselects itself after launching a mini-app, routing commands back through this driver

### Room OFF Behavior

Configures what happens when the room is turned off:
- `Do Nothing` *(default)*
- `Pause`
- `Turn OFF`

### Key Mapping

Every standard Control4 remote button can be remapped to any Android keycode. Configurable keys include: Power, Guide, Page Up/Down, Directional pad, Enter, Channel Up/Down, Info, Menu, Cancel, DVR, Rewind, Fast Forward, Skip, Play, Pause, Record, Stop, Color buttons (Red/Green/Yellow/Blue), and numeric buttons 0–9.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Pairing does not start | Confirm TCP 6467 is reachable and the TV is awake |
| Driver shows offline after pairing | Run **Disconnect From Command Interface**, then **Connect To Command Interface** |
| Pairing fails with bad PIN | Re-extract the modulus/exponent from the device certificate and re-enter before retrying |
| App launch fails | Verify the package name or use a known Android TV deep link format |
| ADB connection refused | Enable Developer Options → ADB Debugging on the TV, then approve the prompt |
| TLS handshake fails | Switch **TLS Method** from Auto to `sslv23` or `tlsv13` explicitly |

---

## Driver Actions Reference

| Action | Description |
|---|---|
| Begin Pairing Android TV Device | Initiates the pairing handshake on TCP 6467 |
| Finish Pairing Android TV Device (Enter PIN) | Completes pairing using the PIN shown on screen |
| Connect To Command Interface | Establishes the command channel on TCP 6466 |
| Disconnect From Command Interface | Closes the command channel |
| Send WOL | Sends a Wake-on-LAN magic packet to the device |
| Test Command URL | Tests an app launch URL |
| Test KeyCode – Press Then Release | Tests a keycode with press+release behavior |
| Test KeyCode – Send Code | Sends a raw keycode |
| Backup Configuration | Outputs config to the LUA tab for copy/paste backup |

---

## Protocol Notes

The driver uses **Android TV Remote Service v2** — Google's native encrypted remote protocol:

- **TCP 6466** — command/control channel (TLS)
- **TCP 6467** — pairing channel (TLS, RSA certificate exchange)
- **Protobuf** message framing over the TLS socket
- **Wake-on-LAN** via UDP broadcast for power-on

ADB is only needed for optional package-list commands or if you prefer the ADB method of certificate extraction during setup.

---

## Credits

Driver authored by **Greg Moerler**.

Distributed with permission. All driver logic, certificate handling, and protocol implementation are his original work.
